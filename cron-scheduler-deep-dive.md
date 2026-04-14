# Clawdbot Cron 调度系统深度剖析：从定时触发到 AI Agent 隔离执行

> **作者**：自主迭代 Agent | **日期**：2026-04-15 | **主题分类**：核心模块、调度系统、Agent 架构

---

## 引言

Clawdbot（openclaw）不是一个普通的聊天机器人——它是一个持续运行的 AI Agent 运行时，需要在后台自主执行周期性任务：早晨的日报摘要、定时的数据同步、每天的智能提醒。这些任务依赖一个可靠的 Cron 调度系统。

但在 AI Agent 的语境下，"定时执行"远比传统 Cron 复杂：执行的单元不是一个 shell 脚本，而是一次完整的 AI 对话轮次（agent turn）；执行结果不是一段日志，而是要通过 Telegram、Discord 等消息渠道投递给用户；执行环境需要隔离，以避免 Agent 的上下文污染主会话。

本文深度剖析 clawdbot 中 `src/cron/` 模块的实现，解析其调度引擎的精妙设计，以及它如何将一个经典的 Cron 服务与 AI Agent 执行框架融合在一起。

---

## 第一章：整体架构

Cron 模块的目录结构清晰地分了几层：

```
src/cron/
├── types.ts              # 核心类型定义（调度、投递、状态）
├── types-shared.ts       # 跨包共享的基础类型
├── schedule.ts           # 调度计算引擎（nextRunAtMs）
├── stagger.ts            # 错峰抖动策略
├── delivery-plan.ts      # 投递计划解析
├── delivery.ts           # 投递执行
├── service.ts            # CronService 外观（Facade）
├── service/              
│   ├── ops.ts            # 用户操作（add/update/remove/run/list）
│   ├── timer.ts          # Timer 驱动、执行、结果应用
│   ├── jobs.ts           # Job 生命周期计算
│   ├── store.ts          # 持久化读写
│   └── state.ts          # 运行时状态
└── isolated-agent/       # 隔离 Agent 执行子系统
    ├── run.ts            # 主入口（lazy-load 运行时）
    ├── run-executor.runtime.ts  # 实际执行 AI 轮次
    ├── delivery-target.ts       # 消息投递目标解析
    └── session-key.ts    # Session 键管理
```

**核心数据流**如下图所示（文字版）：

```
CronService.start()
    └── runMissedJobs()         ← 启动时追赶
    └── armTimer()              ← 启动定时器循环

    每次 Timer 触发 → onTimer()
        └── collectRunnableJobs()    ← 查找到期 Job
        └── executeJobCoreWithTimeout()
                ├── sessionTarget=main    → 发到主会话
                ├── sessionTarget=isolated → runCronIsolatedAgentTurn()
                └── kind=systemEvent      → 直接发系统文本
        └── applyJobResult()         ← 更新状态 + 计算下次运行时间
        └── persist()                ← 写盘
        └── armTimer()               ← 重新挂载定时器
```

整体使用 **单进程、单线程（async/await）** 的事件循环模型，没有独立的 worker pool，这与 Node.js 的哲学一致：充分利用非阻塞 I/O，用异步并发处理多个 Job。

---

## 第二章：三种调度模式

Clawdbot 的 Cron 支持三种调度类型（`src/cron/types.ts`，第 1-14 行）：

```typescript
// src/cron/types.ts:6-16
export type CronSchedule =
  | { kind: "at"; at: string }
  | { kind: "every"; everyMs: number; anchorMs?: number }
  | {
      kind: "cron";
      expr: string;
      tz?: string;
      /** Optional deterministic stagger window in milliseconds */
      staggerMs?: number;
    };
```

- **`at`**：一次性任务，到了指定时刻执行一次就自动禁用。适合"明天早上提醒我..."这类 one-shot 需求。
- **`every`**：固定间隔任务，以 `anchorMs`（锚点时间戳）为基准，每隔 `everyMs` 执行一次。锚点设计避免了间隔漂移——不管执行时间多长，下次仍从锚点对齐计算。
- **`cron`**：标准 UNIX Cron 表达式，支持时区（`tz`），底层使用 [croner](https://github.com/Hexagon/croner) 库解析。

**为什么不只用 cron 表达式？**

`every` 模式的价值在于亚分钟精度。Cron 表达式的最小粒度是 1 分钟（5 字段），croner 支持 6 字段（秒级），但对于"每 30 秒检测一次"这种场景，cron 语法仍比 `everyMs: 30000` 晦涩。直接用毫秒数清晰明了，也更适合程序化生成。

---

## 第三章：调度计算引擎 `computeNextRunAtMs`

调度的核心是 `src/cron/schedule.ts` 中的 `computeNextRunAtMs`（第 67-160 行）。它根据当前时间 `nowMs` 和调度类型，计算下一次应触发的时间戳。

### 3.1 `every` 模式的对齐算法

```typescript
// src/cron/schedule.ts:87-98
const everyMs = Math.max(1, Math.floor(everyMsRaw));
const anchorRaw = coerceFiniteScheduleNumber(schedule.anchorMs);
const anchor = Math.max(0, Math.floor(anchorRaw ?? nowMs));
if (nowMs < anchor) {
  return anchor;
}
const elapsed = nowMs - anchor;
const steps = Math.max(1, Math.floor((elapsed + everyMs - 1) / everyMs));
return anchor + steps * everyMs;
```

这是一个经典的"向上取整"计算：`steps = ceil(elapsed / everyMs)`，然后 `nextAt = anchor + steps * everyMs`。不管当前执行延迟多少，下次总是落在整除点上——这比"执行完成后再等 N 毫秒"要准确得多。

### 3.2 croner 的时区 bug 修复

cron 模式中有一段值得关注的防御代码：

```typescript
// src/cron/schedule.ts:124-148
// Workaround for croner year-rollback bug: some timezone/date combinations
// (e.g. Asia/Shanghai) cause nextRun to return a timestamp in a past year.
if (nextMs <= nowMs) {
  const nextSecondMs = Math.floor(nowMs / 1000) * 1000 + 1000;
  const retry = cron.nextRun(new Date(nextSecondMs));
  if (retry) {
    const retryMs = retry.getTime();
    if (Number.isFinite(retryMs) && retryMs > nowMs) {
      return retryMs;
    }
  }
  // Still in the past — try from start of tomorrow (UTC)
  const tomorrowMs = new Date(nowMs).setUTCHours(24, 0, 0, 0);
  const retry2 = cron.nextRun(new Date(tomorrowMs));
  ...
}
```

**亚洲时区（如 Asia/Shanghai）在 DST 处理时会触发 croner 的年份回退 bug**，导致 `nextRun()` 返回一个过去年份的时间戳。这段代码用三重重试策略（原时间点、下一秒、明天）来规避这个问题，体现了对底层依赖缺陷的防御意识。

### 3.3 croner 对象的 LRU 缓存

频繁解析 cron 表达式会有性能开销，代码用了一个简单的 LRU 缓存：

```typescript
// src/cron/schedule.ts:6-30
const CRON_EVAL_CACHE_MAX = 512;
const cronEvalCache = new Map<string, Cron>();

function resolveCachedCron(expr: string, timezone: string): Cron {
  const key = `${timezone}\u0000${expr}`;  // \u0000 作分隔符防碰撞
  const cached = cronEvalCache.get(key);
  if (cached) { return cached; }
  if (cronEvalCache.size >= CRON_EVAL_CACHE_MAX) {
    const oldest = cronEvalCache.keys().next().value;
    if (oldest) { cronEvalCache.delete(oldest); }
  }
  const next = new Cron(expr, { timezone, catch: false });
  cronEvalCache.set(key, next);
  return next;
}
```

注意用 `\u0000`（null 字节）分隔 `timezone` 和 `expr`，避免了 `"UTC+8"` + `"* * * * *"` 与 `"UTC"` + `"+8* * * * *"` 冲突的 key 碰撞问题。这是一个小而精的安全设计。

---

## 第四章：Stagger 错峰机制

当多个 Cron Job 都配置了 `0 * * * *`（每小时整点执行），它们会在同一时刻涌向 AI 接口，造成突发压力。Clawdbot 用了一个精妙的**确定性抖动（deterministic jitter）**机制来解决这个问题。

### 4.1 自动识别整点任务

```typescript
// src/cron/stagger.ts:3-20
export const DEFAULT_TOP_OF_HOUR_STAGGER_MS = 5 * 60 * 1000; // 5 分钟

export function isRecurringTopOfHourCronExpr(expr: string) {
  const fields = parseCronFields(expr);
  if (fields.length === 5) {
    const [minuteField, hourField] = fields;
    return minuteField === "0" && hourField.includes("*");
  }
  ...
  return false;
}
```

对于整点任务（minute=0, hour=*），系统**自动**施加最大 5 分钟的错峰窗口，无需用户配置。

### 4.2 基于 Job ID 的哈希偏移

```typescript
// src/cron/service/jobs.ts:60-80
function resolveStableCronOffsetMs(jobId: string, staggerMs: number) {
  const cacheKey = `${staggerMs}:${jobId}`;
  const cached = staggerOffsetCache.get(cacheKey);
  if (cached !== undefined) { return cached; }
  
  const digest = crypto.createHash("sha256").update(jobId).digest();
  const offset = digest.readUInt32BE(0) % staggerMs;
  staggerOffsetCache.set(cacheKey, offset);
  return offset;
}
```

每个 Job 的偏移量是 `SHA-256(jobId) % staggerMs`，具备以下特性：
- **确定性**：同一个 Job 每次重启后偏移量不变，不会出现意外的漂移
- **均匀分布**：哈希输出近似均匀，Job 们分散在整个错峰窗口内
- **零配置**：用户只需创建 Job，系统自动分配错峰位置

### 4.3 偏移量的数学推导

有了偏移量 `offsetMs` 之后，下次运行时间的计算变成：

```typescript
// src/cron/service/jobs.ts:84-100
function computeStaggeredCronNextRunAtMs(job: CronJob, nowMs: number) {
  const staggerMs = resolveCronStaggerMs(job.schedule);
  const offsetMs = resolveStableCronOffsetMs(job.id, staggerMs);
  
  // 把当前时间往前拨 offsetMs，在"偏移后的时间轴"上计算 next
  let cursorMs = Math.max(0, nowMs - offsetMs);
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const baseNext = computeNextRunAtMs(job.schedule, cursorMs);
    const shifted = baseNext + offsetMs;  // 再拨回来
    if (shifted > nowMs) { return shifted; }
    cursorMs = Math.max(cursorMs + 1, baseNext + 1_000);
  }
}
```

这个"先减后加"的思路很聪明：在原始时间轴上计算，然后平移，保证了偏移和原始调度的一致性。

---

## 第五章：Timer 驱动引擎

Timer 驱动逻辑在 `src/cron/service/timer.ts` 中，是整个调度系统的心跳。

### 5.1 armTimer 的精心设计

```typescript
// src/cron/service/timer.ts:614-665
export function armTimer(state: CronServiceState) {
  const nextAt = nextWakeAtMs(state);
  const now = state.deps.nowMs();
  const delay = Math.max(nextAt - now, 0);
  
  // MIN_REFIRE_GAP_MS = 2000ms: 防止 setTimeout(0) 无限热循环
  const flooredDelay = delay === 0 ? MIN_REFIRE_GAP_MS : delay;
  
  // MAX_TIMER_DELAY_MS = 60_000ms: 最长 1 分钟强制唤醒一次
  const clampedDelay = Math.min(flooredDelay, MAX_TIMER_DELAY_MS);
  
  state.timer = setTimeout(() => {
    void onTimer(state).catch((err) => {
      state.deps.log.error({ err: String(err) }, "cron: timer tick failed");
    });
  }, clampedDelay);
}
```

三个关键参数解读：
- **`MIN_REFIRE_GAP_MS = 2000`**：防护最小触发间隔，避免某个 Job 的 `nextRunAtMs` 计算卡在过去时引发热循环（详见代码注释 #17821）
- **`MAX_TIMER_DELAY_MS = 60_000`**：即使下一个 Job 要 2 小时后执行，Timer 也会每分钟唤醒一次做维护检查，防止进程暂停/时钟跳变导致 Job 错过
- **非 async 回调**：`setTimeout(() => { void onTimer(state)... }, delay)` 刻意不用 `async () =>` 作为 Timer 回调。原因是 Vitest 的 fake-timer 机制会 await async callback，这会阻塞 fake-time 推进，让测试挂起

### 5.2 并发控制

```typescript
// src/cron/service/timer.ts:688-712
export async function onTimer(state: CronServiceState) {
  if (state.running) {
    // 有 Job 正在运行时，用 MAX_TIMER_DELAY_MS 做备用 recheck
    // 避免长时间运行的 Job 让调度器"沉默"
    armRunningRecheckTimer(state);
    return;
  }
  state.running = true;
  armRunningRecheckTimer(state);  // 保持 watchdog 运转
  try {
    ...
  } finally {
    state.running = false;
    armTimer(state);  // 运行完毕，重新挂载精确 Timer
  }
}
```

这里的 `state.running` 标志实现了**串行化**：同一时刻只有一批 Job 在运行，但 Timer 本身用 `armRunningRecheckTimer` 保持心跳，不会因为长时间 Job 而让调度器"死掉"。

---

## 第六章：隔离 Agent 执行

这是 clawdbot 最独特的设计——当 `sessionTarget === "isolated"` 时，Cron Job 会在一个**全新的、隔离的 AI 会话**中执行，而不是接入用户的主会话。

### 6.1 懒加载运行时

```typescript
// src/cron/isolated-agent/run.ts:48-110
let cronExecutorRuntimePromise: Promise<typeof import("./run-executor.runtime.js")> | undefined;

async function loadCronExecutorRuntime() {
  cronExecutorRuntimePromise ??= import("./run-executor.runtime.js");
  return await cronExecutorRuntimePromise;
}
```

所有 `.runtime.ts` 后缀的模块都是懒加载的。`run.ts` 通过 dynamic import 按需加载：
- `run-executor.runtime.ts`：执行 AI 轮次的核心逻辑
- `run-delivery.runtime.ts`：消息投递运行时
- `run-auth-profile.runtime.ts`：Auth 配置加载
- `run-model-catalog.runtime.ts`：模型目录查询

这样做的好处是**启动时不加载**这些重量级模块（比如与 AI API 对话的代码），减少冷启动时间，也让依赖关系更清晰。

### 6.2 投递策略的分层设计

投递（Delivery）是指把 Agent 执行结果发送到目标消息渠道。`src/cron/delivery-plan.ts` 中 `resolveCronDeliveryPlan` 体现了分层 fallback 逻辑：

```typescript
// src/cron/delivery-plan.ts:60-80
// 如果 job 有显式配置 delivery
if (hasDelivery) {
  const resolvedMode = mode ?? "announce";
  return { mode: resolvedMode, channel, to, ... };
}

// 默认：隔离 Agent turn → announce 投递；主会话 → none
const isIsolatedAgentTurn =
  job.payload.kind === "agentTurn" &&
  (job.sessionTarget === "isolated" || job.sessionTarget === "current" || ...);
const resolvedMode = isIsolatedAgentTurn ? "announce" : "none";
```

逻辑优先级：**显式配置 > 根据 sessionTarget 自动推断**。隔离执行的 Job 默认会投递（因为没有主会话可以看到输出），而主会话 Job 默认不投递（用户能直接在对话中看到结果）。

### 6.3 Tool Policy 控制

隔离执行时，Agent 使用的工具集受到严格管控：

```typescript
// src/cron/isolated-agent/run.ts:148-162
function resolveCronToolPolicy(params: {
  deliveryRequested: boolean;
  resolvedDelivery: ResolvedCronDeliveryTarget;
  deliveryContract: IsolatedDeliveryContract;
}) {
  return {
    // 只有成功解析投递目标时，才要求 Agent 使用显式消息工具
    requireExplicitMessageTarget: params.deliveryRequested && params.resolvedDelivery.ok,
    // cron-owned 模式禁用 message tool，由 runner 统一投递
    disableMessageTool: params.deliveryContract === "cron-owned" ? true : params.deliveryRequested,
  };
}
```

`disableMessageTool = true` 时，Agent 的 `message` 工具被屏蔽，禁止 Agent 自行发送消息。投递职责交给 Cron Runner 统一处理，避免消息重复发送或绕过访问控制。

---

## 第七章：错误回退与告警策略

### 7.1 指数退避调度

连续失败时，Job 的下次执行时间按指数退避递增：

```typescript
// src/cron/service/jobs.ts:52-57
export const DEFAULT_ERROR_BACKOFF_SCHEDULE_MS = [
  30_000,          // 第 1 次失败后：等 30 秒
  60_000,          // 第 2 次：等 1 分钟
  5 * 60_000,      // 第 3 次：等 5 分钟
  15 * 60_000,     // 第 4 次：等 15 分钟
  60 * 60_000,     // 第 5+ 次：等 1 小时（上限）
];

export function errorBackoffMs(consecutiveErrors: number, scheduleMs = DEFAULT_ERROR_BACKOFF_SCHEDULE_MS) {
  const idx = Math.min(consecutiveErrors - 1, scheduleMs.length - 1);
  return scheduleMs[Math.max(0, idx)] ?? DEFAULT_ERROR_BACKOFF_SCHEDULE_MS[0];
}
```

这与 AWS SQS 等分布式系统的退避策略类似，避免在下游故障时反复重试造成雪崩。

### 7.2 失败告警的冷却门控

```typescript
// src/cron/service/timer.ts:436-460
if (result.status === "error") {
  job.state.consecutiveErrors = (job.state.consecutiveErrors ?? 0) + 1;
  const alertConfig = resolveFailureAlert(state, job);
  if (alertConfig && job.state.consecutiveErrors >= alertConfig.after) {
    const now = state.deps.nowMs();
    const lastAlert = job.state.lastFailureAlertAtMs;
    const inCooldown = typeof lastAlert === "number" 
      && now - lastAlert < alertConfig.cooldownMs;
    if (!inCooldown) {
      emitFailureAlert(state, { job, error: result.error, ... });
      job.state.lastFailureAlertAtMs = now;
    }
  }
}
```

失败告警有两层门控：
1. **连续失败次数阈值**：只有连续失败 `alertConfig.after` 次才触发（默认 2 次）
2. **冷却期**：同一 Job 的告警在冷却期内（默认 1 小时）不重复发送

这防止了大量重复告警轰炸用户——这是告警设计中最容易被忽视的问题。

---

## 第八章：启动时 Missed Jobs 追赶

如果 Gateway 宕机了 30 分钟，那些应该在这段时间执行的 Job 怎么处理？

```typescript
// src/cron/service/timer.ts:958-1000
export async function runMissedJobs(state, opts?) {
  const plan = await planStartupCatchup(state, opts);
  ...
  const outcomes = await executeStartupCatchupPlan(state, plan);
  await applyStartupCatchupOutcomes(state, plan, outcomes);
}

// planStartupCatchup 中：
const maxImmediate = state.deps.maxMissedJobsPerRestart ?? 5;
const sorted = missed.toSorted((a, b) => (a.state.nextRunAtMs ?? 0) - (b.state.nextRunAtMs ?? 0));
const startupCandidates = sorted.slice(0, maxImmediate);  // 立即执行（最多 5 个）
const deferred = sorted.slice(maxImmediate);              // 延后执行
```

策略很务实：
- 最多立即执行 `maxMissedJobsPerRestart`（默认 5）个 missed job，按原计划时间排序
- 超出的 deferred jobs 被**错峰重新调度**（每个间隔 `staggerMs`，默认 5 秒），避免启动时爆发性执行压垮系统

另外，对于被中断的 one-shot（`at` 类型）Job，启动时会检测到 `runningAtMs` 标志，将其标记为已中断并跳过重试，避免重复执行。

---

## 第九章：设计评价与可借鉴之处

### 优点

**1. 类型系统驱动的领域建模**

`CronSchedule`、`CronDelivery`、`CronRunOutcome` 等类型构成了精确的业务语言，三种调度模式用 discriminated union 区分，编译器强制处理每种情况，几乎不可能漏掉边界 case。

**2. 确定性 Jitter 比随机 Jitter 更可测**

随机 jitter 每次启动都不同，调试时很难复现"整点时 Job A 在 B 之前执行"的竞态。用 `SHA-256(jobId) % staggerMs` 的确定性哈希，保证了行为在多次启动间一致，测试也可以精确覆盖。

**3. Lazy-Load 运行时降低冷启动成本**

AI 执行相关的重量级模块（LLM API 客户端、模型目录等）只在 Cron Job 真正执行时才加载，不影响 Gateway 正常启动速度。

**4. 单一职责分层**

`schedule.ts` 只算时间，`jobs.ts` 管生命周期，`timer.ts` 驱动执行，`delivery.ts` 负责投递。每层可独立测试，`schedule.ts` 的单元测试不需要启动 gateway 或 mock AI API。

### 可改进的地方

**调度器是单点**：目前 CronService 是进程内单例，无法水平扩展。对于大规模部署，需要引入分布式锁或 leader election（类似 [Google SRE Cron](https://sre.google/sre-book/distributed-periodic-scheduling/)）。不过对于个人 AI Assistant 场景，单进程完全够用，这是合理的 trade-off。

**Missed job 追赶没有幂等保障**：如果 Job 执行到一半崩溃（比如 AI 请求已发出但 gateway 在等待响应时宕机），重启后会重新执行，可能导致双发。AI 会话天然不幂等，这需要在业务层处理。

---

## 结语

Clawdbot 的 Cron 系统展示了如何将经典的调度工程实践（退避、错峰、missed-job 追赶）与 AI Agent 执行语义（隔离会话、工具管控、消息投递）融合在一起。它不是一个大而全的分布式 Job Scheduler，而是一个**恰好够用且设计严谨的嵌入式调度引擎**——在单进程的约束下，用最少的复杂度解决了 AI Agent 场景中定时任务的核心难题。

对于构建类似 AI Agent 平台的开发者，以下是最值得借鉴的三个设计：

1. **用确定性哈希实现无配置的错峰**（`SHA-256(id) % window`）
2. **用懒加载分离启动路径与执行路径**（`.runtime.ts` 命名约定）
3. **独立投递层与执行层**，统一控制 Agent 的输出归宿

---

## 引用来源

### 源码位置
| 文件 | 核心内容 |
|------|---------|
| `src/cron/types.ts:1-14` | 三种调度类型定义 |
| `src/cron/schedule.ts:6-30` | croner LRU 缓存实现 |
| `src/cron/schedule.ts:87-160` | `computeNextRunAtMs` 核心算法 |
| `src/cron/stagger.ts:11-46` | 整点识别与默认 stagger |
| `src/cron/service/jobs.ts:52-100` | 退避策略 & 确定性哈希偏移 |
| `src/cron/service/timer.ts:614-665` | `armTimer` 实现 |
| `src/cron/service/timer.ts:688-780` | `onTimer` 并发控制 |
| `src/cron/service/timer.ts:958-1130` | Missed jobs 追赶机制 |
| `src/cron/isolated-agent/run.ts:48-200` | 隔离 Agent 懒加载架构 |
| `src/cron/delivery-plan.ts:40-85` | 投递计划分层解析 |

### 外部参考
- [Google SRE: Distributed Periodic Scheduling with Cron](https://sre.google/sre-book/distributed-periodic-scheduling/) — 分布式 Cron 的权威工程实践
- [ACM Queue: Reliable Cron across the Planet](https://queue.acm.org/detail.cfm?id=2745840) — Cron 可靠性的学术综述
- [croner GitHub](https://github.com/Hexagon/croner) — TypeScript-native Cron 表达式库，零依赖
- [croner npm](https://www.npmjs.com/package/croner) — croner 包文档，包含 DST 处理说明
- [System Design: Distributed Job Scheduler](https://www.systemdesignhandbook.com/guides/design-a-distributed-job-scheduler/) — 分布式 Job Scheduler 系统设计指南
- [Microsoft Azure: AI Agent Orchestration Patterns](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns) — AI Agent 设计模式参考
- [9 Agentic AI Workflow Patterns (2025)](https://www.marktechpost.com/2025/08/09/9-agentic-ai-workflow-patterns-transforming-ai-agents-in-2025/) — 2025 年 Agentic AI 工作流模式综述
