# Command Lane Queue：Clawdbot 的并发序列化引擎

> **系列文章第 4 篇** — 本文深入剖析 `src/process/command-queue.ts` 这个隐藏在 Clawdbot 核心的并发基础设施，揭示它如何用"泳道"隔离不同类型的异步工作、如何在进程热重启后优雅地恢复状态，以及一个被内联注释称为"Schema migration"的运行时补丁技巧。

---

## 1. 为什么 AI Agent 需要串行化基础设施

在传统的 Web 服务器中，每个请求通常是独立的——并发 100 个请求就真正并行处理 100 个。但 Clawdbot 是一个多渠道 AI Agent 平台，它的"请求"不是独立的：

- 同一 **session** 里的两次 `/compact` 指令必须串行（否则两个压缩器同时操作同一个 JSONL session 文件会损坏数据）
- 同一个 **cron 任务** 不能同时运行两个实例
- 压缩操作需要持有 **session 写锁**，不能在锁内再触发另一个压缩（死锁）
- **SIGUSR1 热重启**时，正在执行的 Agent turn 不能被直接杀死，必须等待 drain

Clawdbot 的解法是一个多 Lane 并发队列：每种类型的工作跑在自己的"泳道"里，泳道内部严格串行，泳道之间可以并行。

---

## 2. Lane 分类：四种内置泳道 + 动态 session 泳道

Lane 的名称定义在 `src/process/lanes.ts`：

```typescript
// src/process/lanes.ts:1-6
export const enum CommandLane {
  Main = "main",
  Cron = "cron",
  Subagent = "subagent",
  Nested = "nested",
}
```

每种 lane 对应一类工作：

| Lane | 用途 | 典型任务 |
|------|------|---------|
| `main` | 主 Agent 会话处理 | stdin 指令处理、auto-reply 工作流 |
| `cron` | 定时任务执行 | 调度触发的 Agent 会话运行 |
| `subagent` | 子 Agent 执行 | 被主 Agent 派生的子任务 |
| `nested` | 防死锁的嵌套操作 | 见第 5 节 |
| `session:${key}` | 会话级串行化 | 压缩、写锁操作（动态创建） |

动态 session lane 的命名规则在 `src/agents/pi-embedded-runner/lanes.ts`：

```typescript
// src/agents/pi-embedded-runner/lanes.ts:3-6
export function resolveSessionLane(key: string) {
  const cleaned = key.trim() || CommandLane.Main;
  return cleaned.startsWith("session:") ? cleaned : `session:${cleaned}`;
}
```

任何不以 `session:` 开头的 key 都会被自动加前缀。这确保了 session lane 在名称空间上是隔离的，不会与内置 lane 冲突。

---

## 3. 全局单例：又一次 Symbol.for + globalThis

与 [Context Engine 注册表](context-engine-pluggable-architecture.md)如出一辙，Command Queue 也用 `Symbol.for` 共享进程级单例状态：

```typescript
// src/process/command-queue.ts:68-87
const COMMAND_QUEUE_STATE_KEY = Symbol.for("openclaw.commandQueueState");

function getQueueState() {
  const state = resolveGlobalSingleton(COMMAND_QUEUE_STATE_KEY, () => ({
    gatewayDraining: false,
    lanes: new Map<string, LaneState>(),
    activeTaskWaiters: new Set<ActiveTaskWaiter>(),
    nextTaskId: 1,
  }));
  // Schema migration: the singleton may have been created by an older code
  // version (e.g. v2026.4.2) that did not include `activeTaskWaiters`.  After
  // a SIGUSR1 in-process restart the new code inherits the stale object via
  // `resolveGlobalSingleton` because the Symbol key already exists on
  // globalThis.  Patch the missing field so all downstream consumers see a
  // valid Set instead of `undefined`.
  if (!state.activeTaskWaiters) {
    state.activeTaskWaiters = new Set<ActiveTaskWaiter>();
  }
  return state;
}
```

这里有一个被内联注释点名的关键工程问题：**运行时 Schema Migration**。

当 SIGUSR1 触发进程热重启时，新代码在同一个 Node.js 进程中加载——`globalThis` 上的旧 Symbol key 还在，`resolveGlobalSingleton` 会直接返回旧对象，不走工厂函数。如果新版本的状态结构多了字段（比如 `activeTaskWaiters` 是 v2026.4.2 之后新增的），旧对象里就没有这个字段，所有访问它的代码都会读到 `undefined`。

修复策略简单粗暴但有效：在每次 `getQueueState()` 调用时检查并修补缺失的字段。这是一个典型的**在线 schema 迁移**模式——不需要版本号，不需要迁移脚本，只需要在访问点做防御性检查。

**对比**：数据库有 `ALTER TABLE ... ADD COLUMN`，运行时 singleton 的等价操作就是 `if (!state.field) state.field = defaultValue`。

---

## 4. `pump()` 循环：事件循环友好的调度器

Lane 的核心机制是 `drainLane` + `pump()` 两层结构：

```typescript
// src/process/command-queue.ts:155-225
function drainLane(lane: string) {
  const state = getLaneState(lane);
  if (state.draining) {
    // 已经有 pump 在跑，不重复启动
    return;
  }
  state.draining = true;

  const pump = () => {
    try {
      while (state.activeTaskIds.size < state.maxConcurrent && state.queue.length > 0) {
        const entry = state.queue.shift() as QueueEntry;
        // ...
        const taskId = getQueueState().nextTaskId++;
        state.activeTaskIds.add(taskId);
        void (async () => {
          try {
            const result = await entry.task();
            completeTask(state, taskId, taskGeneration);
            notifyActiveTaskWaiters();
            pump(); // 递归调用：任务完成后尝试继续抽取
            entry.resolve(result);
          } catch (err) {
            // ...
            pump(); // 错误也要继续抽取
            entry.reject(err);
          }
        })();
      }
    } finally {
      state.draining = false;
    }
  };

  pump();
}
```

几个关键设计决策：

**① `draining` 标志防止重入**：如果已经有一个 `pump()` 在运行，新的 `drainLane` 调用直接返回。这避免了同一 lane 被并发驱动。

**② `maxConcurrent` 支持可调并发度**：默认是 1（严格串行），但可以通过 `setCommandLaneConcurrency()` 调高（`src/process/command-queue.ts:235-239`）。这为将来的并行化留了口子，而不需要重新设计队列结构。

**③ `pump()` 在任务完成后递归**：这是事件循环友好的关键——不用 `setInterval` 轮询，也不用阻塞等待，而是"任务完成→触发下一个"的链式推进。每次 `pump()` 调用都是同步的，只是注册了一个新的 async task 到事件循环，不会 block。

**④ 等待告警**：任务在队列里等待超过 `warnAfterMs`（默认 2000ms）会触发 `onWait` 回调和诊断日志（`src/process/command-queue.ts:169-183`）。这让运维可以及时发现 lane 拥塞。

---

## 5. 双 Lane 压缩：死锁预防的工程巧思

Clawdbot 的压缩（compaction）操作需要同时满足两个约束：
- **同一个 session 的压缩必须串行**（session lane 保证）
- **整个系统同时只运行一个压缩**（global lane 保证，因为 LLM API 调用昂贵且有速率限制）

`compactEmbeddedPiSession` 把这两个约束嵌套实现：

```typescript
// src/agents/pi-embedded-runner/compact.queued.ts:40-53
export async function compactEmbeddedPiSession(params) {
  const sessionLane = resolveSessionLane(params.sessionKey?.trim() || params.sessionId);
  const globalLane = resolveGlobalLane(params.lane);
  const enqueueGlobal =
    params.enqueue ?? ((task, opts) => enqueueCommandInLane(globalLane, task, opts));
  return enqueueCommandInLane(sessionLane, () =>     // 外层：session 串行
    enqueueGlobal(async () => {                       // 内层：global 串行
      // 实际压缩逻辑
    }),
  );
}
```

**关键问题：为什么这个嵌套不会死锁？**

如果内层用的 `globalLane` 也是 `cron`，就会死锁：外层已经占用了 cron lane 槽位，内层又要等 cron lane，形成循环依赖。

解决方案在 `resolveGlobalLane`：

```typescript
// src/agents/pi-embedded-runner/lanes.ts:8-15
export function resolveGlobalLane(lane?: string) {
  const cleaned = lane?.trim();
  // Cron jobs hold the cron lane slot; inner operations must use nested to avoid deadlock.
  if (cleaned === CommandLane.Cron) {
    return CommandLane.Nested;
  }
  return cleaned ? cleaned : CommandLane.Main;
}
```

当外层是 cron lane 时，内层自动降级为 `nested` lane。`nested` 是一个专门为此设计的"溢出 lane"——它没有业务语义，只是一个独立的串行通道，保证内层不会等外层已经持有的 lane。

这是一个非常优雅的设计：**死锁预防逻辑完全封装在 `resolveGlobalLane` 里，调用方无需感知**。

---

## 6. 生代计数器（generation）：处理 SIGUSR1 残留状态

SIGUSR1 热重启是 Clawdbot 的关键运维能力——不需要重启进程，只需要发信号就能重新加载代码。但这带来了一个棘手问题：

> 旧代码正在执行的任务的 finally 块**可能永远不会运行**，导致 `activeTaskIds` 里留着永远不会被删除的 task ID，lane 从此永久阻塞。

Clawdbot 的解法是**生代计数器（generation）**：

```typescript
// src/process/command-queue.ts:341-357
export function resetAllLanes(): void {
  const queueState = getQueueState();
  queueState.gatewayDraining = false;
  const lanesToDrain: string[] = [];
  for (const state of queueState.lanes.values()) {
    state.generation += 1;          // ← 生代号加 1
    state.activeTaskIds.clear();    // ← 清除旧的 active task
    state.draining = false;
    if (state.queue.length > 0) {
      lanesToDrain.push(state.lane);
    }
  }
  // 重启后立即 drain 有排队工作的 lane
  for (const lane of lanesToDrain) {
    drainLane(lane);
  }
  notifyActiveTaskWaiters();
}
```

任务完成时，会检查自己的 `taskGeneration` 是否与当前 lane 的 `state.generation` 相符：

```typescript
// src/process/command-queue.ts:115-121
function completeTask(state: LaneState, taskId: number, taskGeneration: number): boolean {
  if (taskGeneration !== state.generation) {
    return false;   // 旧生代的完成信号，忽略
  }
  state.activeTaskIds.delete(taskId);
  return true;
}
```

这是一个"廉价版乐观锁"：不需要真正的锁，只需要在任务完成时验证生代号。旧生代的任务完成信号会被安静地丢弃，不影响新生代的状态。

注意 `resetAllLanes` 还刻意**保留了 queue 里的排队任务**（注释也解释了原因：这些是"用户挂起的工作，重启后应继续执行"），只清除了 active 状态，然后立即触发 drain 让这些任务继续运行。

这个设计在 `src/cli/gateway-cli/run-loop.ts:247-254` 被精确地调用时机：

```typescript
// src/cli/gateway-cli/run-loop.ts:247-254
const onIteration = createRestartIterationHook(() => {
  // After an in-process restart (SIGUSR1), reset command-queue lane state.
  // Interrupted tasks from the previous lifecycle may have left `active`
  // counts elevated (their finally blocks never ran), permanently blocking
  // new work from draining. This must happen here — at the restart
  // coordinator level — rather than inside individual subsystem init
  // functions, to avoid surprising cross-cutting side effects.
  resetAllLanes();
});
```

注释里的一句话值得品味：**"this must happen here — at the restart coordinator level — rather than inside individual subsystem init functions"**。把生代重置放在重启协调器里，而不是各子系统的初始化里，避免了"时序依赖"陷阱——如果每个子系统各自 reset，可能在系统还没完全初始化时就开始 drain，导致不可预期的副作用。

---

## 7. Gateway Draining：优雅关闭的快失败策略

当 Gateway 准备重启时，需要阻止新任务进入队列，同时等待已有任务完成。Clawdbot 用了一个简单但有效的两阶段策略：

**阶段一：标记 draining，快速拒绝新任务**

```typescript
// src/process/command-queue.ts:229-233
export function markGatewayDraining(): void {
  getQueueState().gatewayDraining = true;
}

// src/process/command-queue.ts:250-254
export function enqueueCommandInLane<T>(lane: string, task: () => Promise<T>): Promise<T> {
  const queueState = getQueueState();
  if (queueState.gatewayDraining) {
    return Promise.reject(new GatewayDrainingError());
  }
  // ...
}
```

新任务在入队时就被拒绝，抛出 `GatewayDrainingError`。调用方可以捕获这个特定错误，向用户返回"系统重启中"的提示，而不是让请求挂起。

**阶段二：等待 active 任务完成**

```typescript
// src/cli/gateway-cli/run-loop.ts:153-181
markGatewayDraining();
const activeTasks = getActiveTaskCount();
// ...
if (activeTasks > 0 || activeRuns > 0) {
  const [tasksDrain, runsDrain] = await Promise.all([
    waitForActiveTasks(restartDrainTimeoutMs),
    waitForActiveEmbeddedRuns(restartDrainTimeoutMs),
  ]);
  if (!tasksDrain.drained || !runsDrain.drained) {
    // drain 超时，强制终止
    abortEmbeddedPiRun(undefined, { mode: "all" });
  }
}
```

`waitForActiveTasks` 用 Observer 模式实现（见下节），等到所有 active task 完成或超时。

这个两阶段策略（标记→等待）是[优雅关闭](https://blog.risingstack.com/graceful-shutdown-node-js-kubernetes/)的经典形式，在 Kubernetes 里也是标准做法：先停止接受新连接（`markGatewayDraining`），再等待已有连接关闭（`waitForActiveTasks`），最后强制退出（`abortEmbeddedPiRun`）。

---

## 8. Active Task Waiters：事件驱动的"等待所有任务完成"

`waitForActiveTasks` 的实现是一个精巧的 Observer 模式：

```typescript
// src/process/command-queue.ts:381-408
export function waitForActiveTasks(timeoutMs: number): Promise<{ drained: boolean }> {
  const queueState = getQueueState();
  const activeAtStart = new Set<number>();
  for (const state of queueState.lanes.values()) {
    for (const taskId of state.activeTaskIds) {
      activeAtStart.add(taskId);  // 记录"调用时刻"的所有 active task ID
    }
  }

  if (activeAtStart.size === 0) {
    return Promise.resolve({ drained: true });
  }

  return new Promise((resolve) => {
    const waiter: ActiveTaskWaiter = {
      activeTaskIds: activeAtStart,  // 只等这些 task，新入队的不算
      resolve,
    };
    waiter.timeout = setTimeout(() => {
      resolveActiveTaskWaiter(waiter, { drained: false });
    }, timeoutMs);
    queueState.activeTaskWaiters.add(waiter);
    notifyActiveTaskWaiters();
  });
}
```

每次任务完成时，`notifyActiveTaskWaiters` 被调用，它检查每个 waiter 关注的 task ID 集合是否已全部消失：

```typescript
// src/process/command-queue.ts:146-153
function notifyActiveTaskWaiters(): void {
  const queueState = getQueueState();
  for (const waiter of Array.from(queueState.activeTaskWaiters)) {
    if (waiter.activeTaskIds.size === 0 || !hasPendingActiveTasks(waiter.activeTaskIds)) {
      resolveActiveTaskWaiter(waiter, { drained: true });
    }
  }
}
```

设计亮点：**`activeAtStart` 是调用时刻的快照**，不是动态跟踪。这意味着：
- `waitForActiveTasks` 只等它调用时正在跑的任务，不等之后新入队的任务
- 语义清晰："等待当前这批活跃任务"，而非"等待系统空闲"

这种"瞬时快照等待"的语义在 draining 场景下是正确的：配合 `markGatewayDraining`（阻止新任务进入），等待快照中的任务就等价于"等待所有任务"。

---

## 9. Probe Lane：静默错误的特殊通道

在 `pump()` 的错误处理里，有一段针对 probe lane 的特殊逻辑：

```typescript
// src/process/command-queue.ts:201-208
const isProbeLane = lane.startsWith("auth-probe:") || lane.startsWith("session:probe-");
if (!isProbeLane && !isExpectedNonErrorLaneFailure(err)) {
  diag.error(
    `lane task error: lane=${lane} durationMs=${Date.now() - startTime} error="${String(err)}"`,
  );
} else if (!isProbeLane) {
  diag.debug(`lane task interrupted: ...`);
}
```

`auth-probe:` 和 `session:probe-` 这两类 lane 的错误会被静默——不记录 error 级别的日志。

为什么？Probe 操作的本质是**探测性试探**，失败是正常的预期结果（比如检测某个 auth 配置是否有效，失败就返回"无效"）。如果每次探测失败都产生 error 日志，运维会被大量噪音干扰。

这个设计体现了一个原则：**日志的严重级别应该反映业务语义，而不是技术层面的异常**。probe 在技术上抛出了异常，但在业务上是正常路径，所以不该记 error。

---

## 10. 两类错误：清理 vs 拒绝

Command Queue 定义了两种专门的错误类型，语义截然不同：

```typescript
// src/process/command-queue.ts:13-29
export class CommandLaneClearedError extends Error {
  constructor(lane?: string) {
    super(lane ? `Command lane "${lane}" cleared` : "Command lane cleared");
    this.name = "CommandLaneClearedError";
  }
}

export class GatewayDrainingError extends Error {
  constructor() {
    super("Gateway is draining for restart; new tasks are not accepted");
    this.name = "GatewayDrainingError";
  }
}
```

- **`CommandLaneClearedError`**：lane 被主动清空（`clearCommandLane`），还在队列里等待的任务被拒绝。这是"强制取消"语义——管理员决定清空这个 lane（比如会话切换）。
- **`GatewayDrainingError`**：网关正在 draining，新来的任务被拒绝。这是"系统级阻断"语义——整个系统正在关闭，不再接受工作。

通过命名不同的错误类型，调用方可以用 `instanceof` 精确区分这两种情况，给用户不同的反馈（"操作已取消" vs "系统重启中，请稍后重试"）。

---

## 11. 与同类方案的对比

### 对比 p-queue 库

[p-queue](https://github.com/sindresorhus/p-queue) 是 Node.js 生态最流行的 Promise 队列库，功能更丰富（优先级、并发动态调整、idle 事件）。Clawdbot 为什么不用它？

最可能的原因：**进程级 singleton 共享需要手工控制**。p-queue 是一个类，每次 `new PQueue()` 得到独立实例。在 ESM 多入口/分块打包的环境里，不同 chunk 会各自 new 出不同的 queue 实例，无法共享状态。Clawdbot 的 `Symbol.for` + `globalThis` 方案则保证所有 chunk 共享同一个 queue 状态——这是 p-queue 无法直接提供的。

当然，代价是缺乏 p-queue 的优先级队列、pause/resume 等高级功能。对于 Clawdbot 的场景（lane 间并行、lane 内串行），这些高级功能暂时不需要。

### 对比 Actor 模型

[Actor 模型](https://en.wikipedia.org/wiki/Actor_model)（Erlang/Akka）中，每个 Actor 有自己的消息邮箱，消息串行处理，Actor 间并行。Clawdbot 的 lane 系统在概念上非常接近 Actor 模型：

- lane = Actor 邮箱
- 任务 = 消息
- lane 内串行 = Actor 保证消息顺序处理

主要区别是：Actor 系统中 Actor 有身份（如 Akka 的 `ActorRef`），可以相互发消息；Clawdbot 的 lane 只是匿名队列，不存在 Actor 间通信。这是一个刻意的简化——Clawdbot 不需要 Actor 间通信，只需要隔离不同类型的工作。

另一个区别是**lane 是动态创建**的（`session:${key}`），而 Actor 通常是在系统启动时预先定义的。这让 Clawdbot 能为每个会话自动创建隔离通道，不需要预先知道有多少个并发会话。

---

## 12. 这套系统隐含的工程原则

**① 用命名空间隔离而非锁**：每个 session 有自己的 lane，根本不需要互相竞争锁。锁只是在 lane 内部隐式存在（`maxConcurrent=1` 的效果）。

**② 生代号是廉价的乐观锁**：不需要 mutex，只需要一个递增的整数。任务完成时校验生代号，过期的信号安静丢弃。这比真正的锁轻量得多。

**③ 运行时 Schema Migration 是实用主义**：在无法停服升级的系统里，代码可以通过"检查+补丁"的方式向前兼容。这不优雅，但直接有效。

**④ 把语义复杂度收进工厂函数**：`resolveGlobalLane` 把"cron lane 的内层操作必须用 nested"这个约束藏在一个工厂函数里，调用方不需要知道这条规则。复杂性被推进了函数边界，而不是散落在调用点。

**⑤ 日志级别应反映业务语义**：probe lane 的失败不是 error，是 probe 的正常结果。日志严重级别应由业务含义决定，而非技术层面的异常状态。

---

## 参考资料

### 源码位置

| 文件 | 行号 | 内容 |
|------|------|------|
| `src/process/command-queue.ts` | 13-29 | `CommandLaneClearedError` / `GatewayDrainingError` 定义 |
| `src/process/command-queue.ts` | 36-52 | `QueueEntry` / `LaneState` 数据结构 |
| `src/process/command-queue.ts` | 66-67 | `Symbol.for("openclaw.commandQueueState")` 进程级共享 key |
| `src/process/command-queue.ts` | 68-87 | 全局 singleton 获取 + Schema Migration 补丁 |
| `src/process/command-queue.ts` | 97-113 | `getLaneState`：lazy 创建 lane |
| `src/process/command-queue.ts` | 115-121 | `completeTask`：生代号校验 |
| `src/process/command-queue.ts` | 146-153 | `notifyActiveTaskWaiters`：Observer 通知 |
| `src/process/command-queue.ts` | 155-225 | `drainLane` + `pump()` 核心调度循环 |
| `src/process/command-queue.ts` | 201-208 | probe lane 静默错误逻辑 |
| `src/process/command-queue.ts` | 231-233 | `markGatewayDraining` |
| `src/process/command-queue.ts` | 235-239 | `setCommandLaneConcurrency` |
| `src/process/command-queue.ts` | 250-253 | `enqueueCommandInLane` 的 draining 快速拒绝 |
| `src/process/command-queue.ts` | 298-310 | `clearCommandLane`：强制取消队列任务 |
| `src/process/command-queue.ts` | 341-357 | `resetAllLanes`：热重启后生代递增 + 保留队列任务 |
| `src/process/command-queue.ts` | 381-408 | `waitForActiveTasks`：Active Task Waiter 实现 |
| `src/process/command-queue.test.ts` | 93-95 | 测试：`resetAllLanes` 在无 lane 时是安全的 |
| `src/process/command-queue.test.ts` | 246-280 | 测试：`resetAllLanes` 后立即 drain 队列，旧生代任务完成被忽略 |
| `src/process/command-queue.test.ts` | 285-310 | 测试：`waitForActiveTasks` 忽略调用之后才入队的任务（快照语义） |
| `src/process/command-queue.test.ts` | 361 | 测试：`markGatewayDraining` 后新入队任务抛出 `GatewayDrainingError` |
| `src/process/lanes.ts` | 1-6 | `CommandLane` 枚举 |
| `src/shared/global-singleton.ts` | 4-11 | `resolveGlobalSingleton`：`globalThis` 上的 Symbol key 单例工厂 |
| `src/agents/pi-embedded-runner/lanes.ts` | 3-15 | `resolveSessionLane` / `resolveGlobalLane` |
| `src/agents/pi-embedded-runner/compact.queued.ts` | 36-53 | 双 lane 嵌套压缩 + 死锁预防注释 |
| `src/agents/pi-embedded-runner/compact.queued.ts` | 47-52 | session lane × global lane 嵌套实现 |
| `src/cli/gateway-cli/run-loop.ts` | 153-181 | `markGatewayDraining` + `waitForActiveTasks` 调用时序 |
| `src/cli/gateway-cli/run-loop.ts` | 247-254 | `resetAllLanes` 在重启协调器中的调用位置 + 理由注释 |

### 外部参考资料

1. [Node.js Event Loop 官方文档](https://nodejs.org/learn/asynchronous-work/event-loop-timers-and-nexttick) — 理解 pump() 为什么是事件循环友好的调度方式
2. [Don't Block the Event Loop — Node.js Learn](https://nodejs.org/learn/asynchronous-work/dont-block-the-event-loop) — 理解 async task 串行化对 Node.js 性能的影响
3. [Event loop: microtasks and macrotasks — javascript.info](https://javascript.info/event-loop) — 深入理解 Promise 任务队列的执行顺序
4. [p-queue — sindresorhus/p-queue on GitHub](https://github.com/sindresorhus/p-queue) — 主流 Promise 队列库，本文对比参照
5. [Actor model — Wikipedia](https://en.wikipedia.org/wiki/Actor_model) — Actor 模型与 lane 系统的概念对比
6. [CSP vs Actor model for concurrency — dev.to](https://dev.to/karanpratapsingh/csp-vs-actor-model-for-concurrency-1cpg) — CSP 与 Actor 模型的异同，帮助定位 lane 系统在并发模型谱系中的位置
7. [Actor Model in Distributed Systems — GeeksforGeeks](https://www.geeksforgeeks.org/system-design/actor-model-in-distributed-systems/) — Actor 模型在分布式系统中的实现
8. [Graceful shutdown with Node.js and Kubernetes — RisingStack](https://blog.risingstack.com/graceful-shutdown-node-js-kubernetes/) — Gateway draining 策略的行业实践参照
9. [NodeJS Graceful Shutdown: A Beginner's Guide — dev.to](https://dev.to/yusadolat/nodejs-graceful-shutdown-a-beginners-guide-40b6) — SIGTERM/SIGINT 信号处理的基础
10. [Design Principles and Patterns for Highly Concurrent Applications — Baeldung](https://www.baeldung.com/concurrency-principles-patterns) — 生代计数器等并发模式的系统性介绍
11. [Epoch-based Optimistic Concurrency Control — arXiv](https://arxiv.org/html/2602.21566v1) — 生代/epoch 计数器在分布式并发控制中的学术背景
