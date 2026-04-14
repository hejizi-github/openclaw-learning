# 实时反馈架构：Clawdbot 如何跨平台可视化 Agent 状态

> **主题**：channels/status-reactions.ts · draft-stream-loop.ts · typing.ts · transport/stall-watchdog.ts  
> **核心问题**：当一个 AI Agent 思考几十秒时，怎样向用户准确、及时、不干扰地传达它的状态？

---

## 为什么这是一个比看起来更难的问题

给聊天机器人加"typing..."指示器看起来很简单——Telegram 有 `sendChatAction("typing")`，Discord 有 `channel.sendTyping()`，Slack 有 `rtm.sendTyping()`。但 AI Agent 的状态反馈有几个特殊困难：

**1. 状态是动态的，不只是"busy/idle"**  
Agent 在同一次回复里会经历：排队等待 → 思考 → 调用工具（可能是代码执行、网页搜索、文件读写）→ 再思考 → 压缩上下文 → 完成。用户看到同一个"typing..."无法区分 LLM 在推理还是在等待工具结果。

**2. 中间状态更新频率极高，平台 API 有速率限制**  
LLM 每输出一个 token 就可能触发一次 UI 更新，但 Telegram 的 `editMessageText` 最高允许 20 req/s（同一聊天 5 req/s），Discord 和 Slack 的速率限制更严格。

**3. 不同平台的反应（Reaction）模型完全不同**  
Slack 支持"先加再删"单独操作每个 reaction；Telegram 只能原子替换最新 reaction（`setMessageReaction`），没有删除接口；Discord 则两者皆有但 emoji 格式不同。

**4. 网络抖动会产生"僵尸状态"**  
如果打字指示器的停止调用因网络超时而没发出，平台会显示机器人永远在打字。GitHub 上有多个 issue 记录了这个问题：[#27177](https://github.com/openclaw/openclaw/issues/27177)、[#27219](https://github.com/openclaw/openclaw/issues/27219)、[#27493](https://github.com/openclaw/openclaw/issues/27493)。

Clawdbot 为解决这四个问题，构建了四个协同工作的子系统。

---

## 子系统一：状态反应控制器

`src/channels/status-reactions.ts`（415 行）是核心：它定义了一个与平台无关的 emoji 状态机。

### 1.1 状态语义和默认值

```typescript
// src/channels/status-reactions.ts:57-76
export const DEFAULT_EMOJIS: Required<StatusReactionEmojis> = {
  queued: "👀",
  thinking: "🤔",
  tool: "🔥",
  coding: "👨‍💻",
  web: "⚡",
  done: "👍",
  error: "😱",
  stallSoft: "🥱",
  stallHard: "😨",
  compacting: "✍",
};

export const DEFAULT_TIMING: Required<StatusReactionTiming> = {
  debounceMs: 700,
  stallSoftMs: 10_000,
  stallHardMs: 30_000,
  doneHoldMs: 1500,
  errorHoldMs: 2500,
};
```

这里有两个值得注意的设计决策：

- **stallSoft/stallHard 两级警报**：10 秒无状态变化换 🥱，30 秒换 😨。这不是纯粹的 UI 美观——它是面向用户的 SLO 警报。用户看到 😨 时知道 Agent 可能卡住了，可以考虑中断。这比 ACP 网关的传输层看门狗（抽象层更低）更早给出人类可读的信号。
- **done/error 有 `holdMs` 但不在 controller 里**：注释明确写"not used in controller, but exported for callers"。调用者负责决定"成功/失败表情保留多久再清除"，controller 本身只管状态跃迁。这是一个清晰的职责分离。

### 1.2 Promise 链序列化：防止并发竞态

状态反应控制器面对的最棘手问题是：`setReaction` 是异步的，如果两次状态更新并发执行，平台 API 的响应顺序是不确定的，可能导致"旧表情盖住新表情"。

Clawdbot 的解法只有一行：

```typescript
// src/channels/status-reactions.ts:178-181
function enqueue(fn: () => Promise<void>): Promise<void> {
  chainPromise = chainPromise.then(fn, fn);
  return chainPromise;
}
```

这个模式称为 **Promise 链序列化**（Promise Chain Serialization）——`chainPromise` 是一个不断延伸的 Promise 链，每次 `enqueue` 都把新操作接在链尾，确保所有 API 调用严格顺序执行，永远不会并发。

注意 `.then(fn, fn)` 而不是 `.then(fn).catch(fn)`：两个参数的写法保证即使前一个操作失败，链路也不会中断——下一个操作仍然会被执行。这是序列化队列不同于普通 Promise 链的关键：单个失败不能阻塞整条队列。

这与 [thoughtspile 的 Promise 序列化文章](https://thoughtspile.github.io/2018/06/20/serialize-promises/) 中描述的 mutex 模式完全一致，但 Clawdbot 的实现更简洁，没有任何外部依赖。

### 1.3 中间态防抖，终止态立即执行

```typescript
// src/channels/status-reactions.ts:259-299
function scheduleEmoji(
  emoji: string,
  options: { immediate?: boolean; skipStallReset?: boolean } = {},
): void {
  if (!enabled || finished) {
    return;
  }
  // 去重：emoji 没变就不发送（但刷新 stall timer）
  if (emoji === currentEmoji || emoji === pendingEmoji) {
    if (!options.skipStallReset) {
      resetStallTimers();
    }
    return;
  }

  pendingEmoji = emoji;
  clearDebounceTimer();

  if (options.immediate) {
    // 终止状态：立即进队
    void enqueue(async () => {
      await applyEmoji(emoji);
      pendingEmoji = "";
    });
  } else {
    // 中间状态：700ms 防抖
    debounceTimer = setTimeout(() => {
      debounceTimer = null;
      void enqueue(async () => {
        await applyEmoji(emoji);
        pendingEmoji = "";
      });
    }, timing.debounceMs);
  }

  if (!options.skipStallReset) {
    resetStallTimers();
  }
}
```

调用方式对比：

```typescript
// src/channels/status-reactions.ts:305-320
function setQueued(): void {
  scheduleEmoji(emojis.queued, { immediate: true }); // 排队等待 → 立即
}
function setThinking(): void {
  scheduleEmoji(emojis.thinking); // 思考 → 防抖 700ms
}
function setTool(toolName?: string): void {
  const emoji = resolveToolEmoji(toolName, emojis);
  scheduleEmoji(emoji); // 工具调用 → 防抖 700ms
}
```

**为什么中间态防抖，终止态不防抖？**

- 中间态（thinking/tool）是"连续更新"的信号。LLM 可能每 200ms 就切换一次工具，如果每次都立即更新 emoji，每分钟 300 次 API 调用会触发速率限制。防抖让快速切换合并为一次。
- 终止态（done/error/queued）是"用户需要立即知道"的信号。用户在等待结果，done 如果也防抖 700ms，体验会明显变差。立即更新是正确的选择。

这与 [Debounce vs Throttle 的经典分析](https://kettanaito.com/blog/debounce-vs-throttle) 一致：debounce 适合"只关心最终结果"，throttle 适合"需要稳定的中间更新"——而 Clawdbot 这里对中间态用 debounce、对终止态绕过 debounce，是对这个原则的精确应用。

### 1.4 Stall 定时器：两级升级

```typescript
// src/channels/status-reactions.ts:214-229
function resetStallTimers(): void {
  if (stallSoftTimer) {
    clearTimeout(stallSoftTimer);
  }
  if (stallHardTimer) {
    clearTimeout(stallHardTimer);
  }

  stallSoftTimer = setTimeout(() => {
    scheduleEmoji(emojis.stallSoft, { immediate: true, skipStallReset: true });
  }, timing.stallSoftMs);

  stallHardTimer = setTimeout(() => {
    scheduleEmoji(emojis.stallHard, { immediate: true, skipStallReset: true });
  }, timing.stallHardMs);
}
```

`skipStallReset: true` 是防止 stall 计时器递归刷新自身：stall 触发的状态更新不应该重置 stall 计时器，否则 stall 会永远被自己续期。

### 1.5 平台适配：双模式清除

```typescript
// src/channels/status-reactions.ts:350-378
async function clear(): Promise<void> {
  if (!enabled) return;
  clearAllTimers();
  finished = true;

  await enqueue(async () => {
    if (adapter.removeReaction) {
      // Discord/Slack 模式：逐个删除所有已知 emoji
      const emojisToRemove = Array.from(knownEmojis);
      for (const emoji of emojisToRemove) {
        try {
          await adapter.removeReaction(emoji);
        } catch (err) {
          if (onError) onError(err);
        }
      }
    } else {
      // Telegram 模式：不需要删除，下次 setReaction 会原子替换
      // 注释: "For platforms without removeReaction, set empty or just skip"
    }
    currentEmoji = "";
    pendingEmoji = "";
  });
}
```

`knownEmojis` 是在构造时从所有配置的 emoji 中构建的 Set（第 161-173 行）。clear 时遍历这个集合，逐个删除——即使其中大部分 emoji 从未被添加，`removeReaction` 对"不存在的 reaction"的调用也只是抛出 `no_reaction` 错误（在 Slack 测试 mock 中可见），被 `onError` 吞掉即可。

这个"尝试删除所有可能出现过的 emoji"的策略比"精确追踪当前 emoji"要健壮：即使因为并发或网络重试导致状态跟踪不精确，clear 也能保证最终干净。

### 1.6 工具名 emoji 分类

```typescript
// src/channels/status-reactions.ts:78-118
export const CODING_TOOL_TOKENS: string[] = [
  "exec", "process", "read", "write", "edit", "session_status", "bash",
];
export const WEB_TOOL_TOKENS: string[] = [
  "web_search", "web-search", "web_fetch", "web-fetch", "browser",
];

export function resolveToolEmoji(
  toolName: string | undefined,
  emojis: Required<StatusReactionEmojis>,
): string {
  const normalized = normalizeOptionalLowercaseString(toolName) ?? "";
  if (!normalized) return emojis.tool;
  if (WEB_TOOL_TOKENS.some((token) => normalized.includes(token))) return emojis.web;
  if (CODING_TOOL_TOKENS.some((token) => normalized.includes(token))) return emojis.coding;
  return emojis.tool;
}
```

用的是 `includes(token)` 而不是精确匹配——`"web-search"` 和 `"web_search_tool"` 都能命中 `"web_search"`。这是一个故意的模糊匹配策略：工具名由 Plugin 自由定义，无法穷举，只要包含关键词就分类。

---

## 子系统二：草稿流循环

`src/channels/draft-stream-loop.ts`（104 行）处理流式文本输出的节流和背压。

### 2.1 节流机制

```typescript
// src/channels/draft-stream-loop.ts:54-62
const schedule = () => {
  if (timer) {
    return; // 已有定时器，去重
  }
  const delay = Math.max(0, params.throttleMs - (Date.now() - lastSentAt));
  timer = setTimeout(() => {
    void flush();
  }, delay);
};
```

`delay = throttleMs - elapsed`：如果距离上次发送已经超过 `throttleMs`，delay 为 0，立即触发（`Math.max(0, ...)`）；否则等待剩余时间。这是一个"剩余配额"节流而不是"重置窗口"节流——每次发送后都从 `Date.now()` 重新计时，而不是从固定的窗口起点计时。

### 2.2 背压：`sent === false` 信号

```typescript
// src/channels/draft-stream-loop.ts:36-50
const current = params.sendOrEditStreamMessage(text).finally(() => {
  if (inFlightPromise === current) {
    inFlightPromise = undefined;
  }
});
inFlightPromise = current;
const sent = await current;
if (sent === false) {
  pendingText = text; // 恢复待发送文本，等下次 flush
  return;
}
lastSentAt = Date.now();
```

`sendOrEditStreamMessage` 返回 `false` 表示"现在不能发，请稍后重试"（例如消息 ID 尚未建立，或者发送中被取消）。这是一个手工实现的背压协议——与 [Node.js Stream 的 `write()` 返回 `false` 触发背压](https://nodejs.org/learn/modules/backpressuring-in-streams) 是同一个设计模式，只不过这里的"流"是消息发送，而不是字节缓冲区。

### 2.3 in-flight 跟踪

```typescript
// src/channels/draft-stream-loop.ts:65-79
update: (text: string) => {
  if (params.isStopped()) return;
  pendingText = text;
  if (inFlightPromise) {
    schedule(); // 有请求在途，延后调度
    return;
  }
  if (!timer && Date.now() - lastSentAt >= params.throttleMs) {
    void flush(); // 没有在途请求，配额已满，立即发
    return;
  }
  schedule(); // 否则按节流窗口调度
},
```

`update` 是"覆盖而不是追加"：新文本直接替换 `pendingText`，不排队。这反映了流式输出的特性——每次 `update` 都是最新的完整草稿，不需要保留中间版本。

---

## 子系统三：打字指示器三层架构

打字指示器系统由三个文件组合实现，每层负责一个独立关注点。

### 3.1 KeepaliveLoop：幂等的周期性心跳

`src/channels/typing-lifecycle.ts`（55 行）：

```typescript
// src/channels/typing-lifecycle.ts:16-26
const tick = async () => {
  if (tickInFlight) {
    return; // 上一次 tick 还没完成，跳过
  }
  tickInFlight = true;
  try {
    await params.onTick();
  } finally {
    tickInFlight = false;
  }
};
```

`tickInFlight` 标志保证同一时刻只有一个 tick 在执行——如果 API 调用比 keepalive 间隔（默认 3000ms）慢，不会堆积并发请求。这是一个经典的"幂等信号"模式：发送频率有上界，不关心单次是否慢。

Telegram 的 `sendChatAction("typing")` 默认有效期约 5 秒，超时后自动消失。3 秒的 keepalive 间隔确保在有效期内持续续期。

### 3.2 StartGuard：熔断器

`src/channels/typing-start-guard.ts`（63 行）实现了一个轻量级熔断器：

```typescript
// src/channels/typing-start-guard.ts:32-52
const run: TypingStartGuard["run"] = async (start) => {
  if (isBlocked()) return "skipped";
  try {
    await start();
    consecutiveFailures = 0;
    return "started";
  } catch (err) {
    consecutiveFailures += 1;
    params.onStartError?.(err);
    if (maxConsecutiveFailures && consecutiveFailures >= maxConsecutiveFailures) {
      tripped = true;
      params.onTrip?.(); // 通知 keepalive 停止继续 tick
      return "tripped";
    }
    return "failed";
  }
};
```

默认连续失败 2 次后熔断（`maxConsecutiveFailures: 2`，见 `typing.ts:17`）。熔断后 `onTrip` 回调会停止 keepalive 循环，不再反复尝试一个明显失败的 API 端点。

返回值有四种语义：`"started"` / `"skipped"` / `"failed"` / `"tripped"`。这比布尔值更精确，让调用方知道"是跳过了还是真的失败了"。这与 [opossum 等 Node.js circuit breaker 库](https://github.com/nodeshift/opossum) 的状态模型完全一致：`CLOSED`（正常）→ `OPEN`（熔断）。

### 3.3 TypingCallbacks：TTL 安全保障

`src/channels/typing.ts`（99 行）把 KeepaliveLoop 和 StartGuard 组合起来，并加入了 TTL 安全机制：

```typescript
// src/channels/typing.ts:50-69
const startTtlTimer = () => {
  if (maxDurationMs <= 0) return;
  clearTtlTimer();
  ttlTimer = setTimeout(() => {
    if (!closed) {
      console.warn(`[typing] TTL exceeded (${maxDurationMs}ms), auto-stopping typing indicator`);
      fireStop();
    }
  }, maxDurationMs); // 默认 60_000ms（60秒）
};
```

**TTL 的核心价值**：正常流程中，`onIdle` 回调会在 Agent 完成时停止打字指示器。但如果 Agent 崩溃、异常退出、或者 `onIdle` 因为某个 bug 没有被调用，没有 TTL 的打字指示器会永远存在（"zombie typing"）。60 秒 TTL 是一个 last-resort 安全网。

---

## 子系统四：传输层看门狗

`src/channels/transport/stall-watchdog.ts`（103 行）工作在比状态反应更低的抽象层：它监控的不是 Agent 状态，而是传输连接本身的活跃度。

```typescript
// src/channels/transport/stall-watchdog.ts:16-29
export function createArmableStallWatchdog(params: {
  label: string;
  timeoutMs: number;
  checkIntervalMs?: number;
  abortSignal?: AbortSignal;
  runtime?: RuntimeEnv;
  onTimeout: (meta: StallWatchdogTimeoutMeta) => void;
}): ArmableStallWatchdog {
  const timeoutMs = Math.max(1, Math.floor(params.timeoutMs));
  const checkIntervalMs = Math.max(
    100,
    Math.floor(params.checkIntervalMs ?? Math.min(5_000, Math.max(250, timeoutMs / 6))),
  );
  // ...
```

`checkIntervalMs` 自动计算为 `timeoutMs / 6`，上限 5000ms，下限 250ms。这个设计很实用：如果超时是 30 秒，检查间隔就是 5 秒；如果超时是 1 秒（测试中常见），检查间隔是 250ms 而不是 167ms，保留足够的精度同时不过度轮询。

### 4.1 arm/touch/disarm 三态协议

这是直接借鉴嵌入式系统[看门狗定时器](https://en.wikipedia.org/wiki/Watchdog_timer)的 API 设计：

```typescript
// src/channels/transport/stall-watchdog.ts:57-76
const arm = (atMs?: number) => {
  if (stopped) return;
  lastActivityAt = atMs ?? Date.now(); // 记录最后活跃时间，激活监控
  armed = true;
};

const touch = (atMs?: number) => {
  if (stopped) return;
  lastActivityAt = atMs ?? Date.now(); // 更新最后活跃时间（不改变 armed 状态）
};
```

- `arm`：开始监控（相当于嵌入式的"启动看门狗"）
- `touch`：喂狗（相当于嵌入式的"kick the dog"或"pet the watchdog"）
- `disarm`：停止监控（相当于"禁用看门狗"）

传输活跃时不断 `touch`，完成传输后 `disarm`，开始新传输时 `arm`。如果传输中途卡住，`touch` 停止更新，超过 `timeoutMs` 后触发 `onTimeout`。

```typescript
// src/channels/transport/stall-watchdog.ts:93
timer.unref?.(); // 后台定时器，不阻止进程正常退出
```

`timer.unref()` 确保看门狗的 setInterval 不会阻止 Node.js 进程退出——这与之前在 Command Lane Queue（`pruneTimer.unref()`）和 cron 调度器中看到的相同惯例。

---

## 四个子系统的协作视图

```
用户发送消息
    │
    ▼
[状态反应控制器] ──► setQueued()  👀 (立即)
    │
    ▼
Agent 开始处理
    │
    ├──► [打字指示器] ──► start() → keepalive 每 3s 续期
    │
    ├──► [状态反应] ──► setThinking() 🤔 (防抖 700ms)
    │
    │  ─── LLM 输出流 ───
    │        │
    │        ▼
    │  [草稿流循环] ──► throttle 250ms → sendOrEditStreamMessage()
    │
    ├──► [状态反应] ──► setTool("web_search") ⚡ (防抖)
    │                   stall 计时器重置
    │
    │  如果 10s 无状态变化 ──► setStallSoft() 🥱
    │  如果 30s 无状态变化 ──► setStallHard() 😨
    │
    ├──► [传输层看门狗] ──► 独立监控 WebSocket/HTTP 连接活跃度
    │                      如果传输卡住 ──► onTimeout → 重连
    │
    ▼
Agent 完成
    │
    ├──► [打字指示器] ──► fireStop() (onIdle)
    ├──► [草稿流循环] ──► flush() → 最后一次发送
    └──► [状态反应] ──► setDone() ✅ (立即) → 1500ms 后 clear()
```

**层次分工**：
- 状态反应 = 用户可见的语义状态（Agent 在做什么）
- 草稿流循环 = 用户可见的内容增量（Agent 说了什么）
- 打字指示器 = 平台原生的"活跃"信号（持续续期）
- 传输层看门狗 = 基础设施健康检查（连接是否还活着）

---

## 测试设计的工程价值

`status-reactions.slack-lifecycle.test.ts` 的一个测试用例特别值得关注：

```typescript
// src/channels/status-reactions.slack-lifecycle.test.ts:97-118
it("restoreInitial clears stall timers without re-adding queued emoji", async () => {
  const ctrl = createStatusReactionController({
    enabled: true,
    adapter,
    initialEmoji: "eyes",
    timing: { debounceMs: 0, stallSoftMs: 10, stallHardMs: 20 },
  });

  void ctrl.setQueued();
  await vi.advanceTimersByTimeAsync(1);
  expect(active.has("eyes")).toBe(true);

  await ctrl.restoreInitial();
  await vi.advanceTimersByTimeAsync(30); // 经过 stall 超时时间

  expect(adapter.setReaction).toHaveBeenCalledTimes(1); // 没有额外调用！
  expect(active.has(DEFAULT_EMOJIS.stallSoft)).toBe(false);
  expect(active.has(DEFAULT_EMOJIS.stallHard)).toBe(false);
});
```

这个测试验证了一个隐蔽的 bug 场景：`restoreInitial()` 不仅要把 emoji 还原，还要确保它清除了 stall 定时器。否则，在 `restoreInitial` 之后经过 stallSoftMs，系统仍然会发送 🥱——即使机器人已经回到了空闲状态。

这类测试（"操作结束后的副作用被正确清理"）是有状态机的系统里最重要、也最容易遗漏的测试维度。

---

## 引用索引

### 源码位置

| 文件 | 行号 | 内容 |
|------|------|------|
| `src/channels/status-reactions.ts` | 57-76 | DEFAULT_EMOJIS / DEFAULT_TIMING 常量定义 |
| `src/channels/status-reactions.ts` | 78-94 | CODING_TOOL_TOKENS / WEB_TOOL_TOKENS |
| `src/channels/status-reactions.ts` | 103-118 | resolveToolEmoji 分类函数 |
| `src/channels/status-reactions.ts` | 129-136 | createStatusReactionController 参数类型 |
| `src/channels/status-reactions.ts` | 161-173 | knownEmojis Set 构建（clear 用） |
| `src/channels/status-reactions.ts` | 178-181 | Promise 链序列化核心：`chainPromise.then(fn, fn)` |
| `src/channels/status-reactions.ts` | 214-229 | resetStallTimers：软/硬两级告警 |
| `src/channels/status-reactions.ts` | 234-254 | applyEmoji：set/remove 两阶段执行 |
| `src/channels/status-reactions.ts` | 259-299 | scheduleEmoji：防抖 vs 立即 判断 |
| `src/channels/status-reactions.ts` | 305-320 | setQueued/setThinking/setTool 入口 |
| `src/channels/status-reactions.ts` | 350-378 | clear：Discord removeReaction vs Telegram 原子替换 |
| `src/channels/draft-stream-loop.ts` | 20-52 | flush 函数：背压 `sent === false` 处理 |
| `src/channels/draft-stream-loop.ts` | 54-62 | schedule：剩余配额节流 |
| `src/channels/draft-stream-loop.ts` | 65-79 | update：覆盖语义 + in-flight 感知 |
| `src/channels/typing-lifecycle.ts` | 16-26 | tickInFlight 幂等保护 |
| `src/channels/typing-lifecycle.ts` | 29-47 | start/stop：start 幂等，stop 清 flag |
| `src/channels/typing.ts` | 25 | 默认 keepaliveIntervalMs = 3000 |
| `src/channels/typing.ts` | 27 | 默认 maxDurationMs = 60_000（TTL 安全网） |
| `src/channels/typing.ts` | 50-61 | startTtlTimer：60s 强制停止 |
| `src/channels/typing-start-guard.ts` | 19 | consecutiveFailures 计数器 |
| `src/channels/typing-start-guard.ts` | 32-52 | run：四状态返回值（started/skipped/failed/tripped） |
| `src/channels/typing-start-guard.ts` | 44-50 | 熔断触发：`tripped = true` + `onTrip()` |
| `src/channels/transport/stall-watchdog.ts` | 26-29 | checkIntervalMs 自动计算：timeoutMs/6 |
| `src/channels/transport/stall-watchdog.ts` | 57-62 | arm：激活监控，记录基准时间 |
| `src/channels/transport/stall-watchdog.ts` | 65-70 | touch：更新活跃时间（喂狗） |
| `src/channels/transport/stall-watchdog.ts` | 72-86 | check：超时判断和触发 |
| `src/channels/transport/stall-watchdog.ts` | 93 | timer.unref()：不阻止进程退出 |
| `src/channels/status-reactions.slack-lifecycle.test.ts` | 97-118 | 测试：restoreInitial 清理 stall 定时器 |
| `src/channels/status-reactions.slack-lifecycle.test.ts` | 43-73 | 测试：完整 queued→thinking→tool→done→clear 流程 |

### 外部参考

1. [Debounce vs Throttle: Definitive Visual Guide — kettanaito.com](https://kettanaito.com/blog/debounce-vs-throttle) — 防抖与节流的语义分析
2. [Advanced Promises Coordination: Serialization — thoughtspile](https://thoughtspile.github.io/2018/06/20/serialize-promises/) — Promise 链序列化模式
3. [Circuit Breaker Pattern in Node.js — Dev Community](https://dev.to/wallacefreitas/circuit-breaker-pattern-in-nodejs-and-typescript-enhancing-resilience-and-stability-bfi) — 熔断器模式详解
4. [opossum: Node.js circuit breaker — GitHub](https://github.com/nodeshift/opossum) — 工业级熔断器库
5. [Backpressuring in Streams — Node.js 官方文档](https://nodejs.org/learn/modules/backpressuring-in-streams) — `write()` 返回 `false` 背压模式
6. [Watchdog timer — Wikipedia](https://en.wikipedia.org/wiki/Watchdog_timer) — 看门狗定时器概念
7. [Watchdog Timer Best Practices — Interrupt/Memfault](https://interrupt.memfault.com/blog/firmware-watchdog-best-practices) — arm/touch/disarm 模式起源
8. [Telegram bot typing indicator 无法停止 — Issue #27177](https://github.com/openclaw/openclaw/issues/27177) — zombie typing 真实问题记录
9. [Typing keepalive loop 内存泄漏 — Issue #27493](https://github.com/openclaw/openclaw/issues/27493) — keepalive 未清理问题
10. [Discord typing indicator 10s 过期 — Issue #390](https://github.com/sipeed/picoclaw/issues/390) — 平台 TTL 限制的实际影响
11. [Slack RTM API typing indicator](https://tools.slack.dev/node-slack-sdk/rtm-api) — Slack 打字指示器 API 文档
12. [Backpressure in JavaScript — gaborkoos.com](https://blog.gaborkoos.com/posts/2026-01-06-Backpressure-in-JavaScript-the-Hidden-Force-Behind-Streams-Fetch-and-Async-Code/) — JS 背压机制原理

---

## 可以学到什么

**1. 对同一"慢慢来"需求，区分"用户感知层"和"基础设施层"**  
stall timer（10s/30s emoji）和 stall watchdog（传输超时）解决的是同一类问题，但面向不同受众：前者给用户看，后者触发重连。在同一系统里并存不是冗余，而是层次分明的告警机制。

**2. Promise 链序列化是异步 API 并发控制的最简单解法**  
`chainPromise = chainPromise.then(fn, fn)` 只需一行，不需要锁、队列或任何并发原语，就能保证所有 API 调用严格顺序执行。适用于任何"不能并发、但需要响应性"的 API 调用场景。

**3. 终止态和中间态用不同的"急迫性"处理**  
中间态快速变化，防抖合并；终止态用户等待，立即执行。这个分类思路可以推广到所有流式 UI 更新场景。

**4. TTL 安全网不是可选的，而是必要的**  
任何有"开始/结束"语义的系统（typing indicator、loading spinner、progress bar）都应该有最大持续时间限制。依赖"正常结束路径"被完整执行是一个不安全假设——进程崩溃、网络断开、代码 bug 都可能让结束信号丢失。

**5. 熔断器的价值在于"不让错误级联"**  
TypingStartGuard 在 API 持续失败时停止 keepalive，避免每次 tick 都产生错误日志。对于非关键功能（typing indicator）的 API 调用，熔断比重试更合适——用户看不到 typing 不会影响回复内容。
