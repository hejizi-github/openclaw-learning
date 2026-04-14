# Clawdbot 的 Compaction 系统：LLM 上下文压缩的工程实践

> 当 AI Agent 运行时间足够长，它必然会遇到一个不可回避的问题：上下文窗口装不下了。  
> Clawdbot 的 Compaction 系统是一套精心设计的上下文压缩引擎，处理从 Token 估算到检查点快照的完整流程。本文深入解剖其实现，提炼出可迁移的工程原则。

---

## 一、问题背景：上下文窗口是 AI Agent 的"工作记忆"

对于短对话，上下文窗口几乎不成问题。但对于需要长期运行的 AI Agent——调试复杂 bug、执行多步骤任务、管理长期项目——会话消息会持续增长，最终触碰模型的上下文限制。

当前主流大模型的上下文窗口虽然已达到数百万 token（如 Claude 3.7 的 200K、Gemini 的 1M），但这并不意味着问题消失：

- **成本随上下文线性增长**：每次请求都要发送完整历史
- **推理质量下降**：过长的历史会稀释关键信息
- **工具调用记录膨胀**：每次工具调用都留下 `tool_call` + `tool_result` 对，快速消耗 token

解决方案有两大类：**截断**（直接丢弃老消息）和**压缩**（用摘要替换老消息）。Clawdbot 选择了压缩，但其实现比"叫 LLM 总结一下"复杂得多。

**核心代码文件：**
- `src/agents/compaction.ts`（578 行）— 压缩算法层
- `src/agents/pi-embedded-runner/compact.ts`（1209 行）— 压缩编排层
- `src/agents/pi-embedded-runner/compaction-safety-timeout.ts`（93 行）— 超时保护
- `src/gateway/session-compaction-checkpoints.ts`（207 行）— 检查点管理
- `src/agents/compaction-real-conversation.ts` — 内容过滤
- `src/agents/pi-compaction-constants.ts` — Token 预算常量

---

## 二、Token 估算：精度与效率的权衡

压缩的第一步是知道当前有多少 token。精确的 Token 计数需要调用 tokenizer，但这意味着：
1. 每次估算都要调用库（性能开销）
2. 跨模型 tokenizer 不同（维护负担）

Clawdbot 选择了字符数估算法（chars/4 启发式）并配合**安全边距**：

```typescript
// src/agents/compaction.ts:21
export const SAFETY_MARGIN = 1.2; // 20% buffer for estimateTokens() inaccuracy
```

```typescript
// src/agents/compaction.ts:226
const effectiveMax = Math.max(1, Math.floor(maxTokens / SAFETY_MARGIN));
```

即：如果最大允许 4000 token，实际只用 `4000 / 1.2 ≈ 3333` token。这个 20% 的缓冲补偿了 chars/4 启发式的低估，尤其在以下场景：

- **多字节字符**（中文、日语每个字符约 2-3 token）
- **代码 token**（特殊符号的实际 token 数往往高于 chars/4）
- **特殊 token**（开头/结尾标记、角色标记等）

**为什么不用精确 tokenizer？**  
在 Agent 系统中，Token 估算用于决定"是否需要压缩"，而非精确计费。20% 的缓冲足以保证不越界，且几乎零性能开销——这是一个典型的"够用比精确更重要"的工程决策。

同理，单条消息的最大 Token 估算也带安全边距：

```typescript
// src/agents/compaction.ts:287-290
export function isOversizedForSummary(msg: AgentMessage, contextWindow: number): boolean {
  const tokens = estimateCompactionMessageTokens(msg) * SAFETY_MARGIN;
  return tokens > contextWindow * 0.5;
}
```

**单条消息超过上下文窗口 50% 时，无法安全总结**——这个阈值同样保守，但合理：总结一条消息需要为系统提示、总结指令、输出留出空间。

---

## 三、自适应 Chunk 比例：不是固定切片

Clawdbot 用"分块"策略将长历史切割成多个部分分别总结：

```typescript
// src/agents/compaction.ts:19-20
export const BASE_CHUNK_RATIO = 0.4;
export const MIN_CHUNK_RATIO = 0.15;
```

默认每个 chunk 最多占上下文窗口的 40%，但当消息平均体积较大时，会动态降低比例：

```typescript
// src/agents/compaction.ts:262-281
export function computeAdaptiveChunkRatio(messages: AgentMessage[], contextWindow: number): number {
  if (messages.length === 0) {
    return BASE_CHUNK_RATIO;
  }

  const totalTokens = estimateMessagesTokens(messages);
  const avgTokens = totalTokens / messages.length;

  // Apply safety margin to account for estimation inaccuracy
  const safeAvgTokens = avgTokens * SAFETY_MARGIN;
  const avgRatio = safeAvgTokens / contextWindow;

  // If average message is > 10% of context, reduce chunk ratio
  if (avgRatio > 0.1) {
    const reduction = Math.min(avgRatio * 2, BASE_CHUNK_RATIO - MIN_CHUNK_RATIO);
    return Math.max(MIN_CHUNK_RATIO, BASE_CHUNK_RATIO - reduction);
  }

  return BASE_CHUNK_RATIO;
}
```

逻辑分析：
- 正常场景（平均消息 < 10% 上下文）：用 40%（BASE_CHUNK_RATIO）
- 消息体积较大：线性缩小比例，最低至 15%（MIN_CHUNK_RATIO）
- 防止过度适应（reduction 上限）：`BASE_CHUNK_RATIO - MIN_CHUNK_RATIO = 0.25`

**设计意图**：当工具调用返回大量数据（如 API 响应 JSON），直接用 40% 切分可能导致单个 chunk 仍超出模型限制。自适应比例是对"工具调用密集型 Agent"的特别优化。

---

## 四、工具调用配对不变式：不能割裂的"原子单元"

这是 Compaction 最精妙的设计之一：Anthropic API 要求 `tool_call` 和对应的 `tool_result` **必须相邻出现**。如果在分块时将它们割裂到不同 chunk，分别总结后就会出现孤儿 `tool_result`，导致下一次 API 请求被拒绝。

因此，分块算法维护了一个"待完成工具调用集合"：

```typescript
// src/agents/compaction.ts:138-196（splitMessagesByTokenShare 核心逻辑）
let pendingToolCallIds = new Set<string>();
let pendingChunkStartIndex: number | null = null;

// ...
if (message.role === "assistant") {
  const toolCalls = extractToolCallsFromAssistant(message);
  const stopReason = (message as { stopReason?: unknown }).stopReason;
  const keepsPending =
    stopReason !== "aborted" && stopReason !== "error" && toolCalls.length > 0;
  pendingToolCallIds = keepsPending ? new Set(toolCalls.map((t) => t.id)) : new Set();
  pendingChunkStartIndex = keepsPending ? current.length - 1 : null;
} else if (message.role === "toolResult" && pendingToolCallIds.size > 0) {
  const resultId = extractToolResultId(message);
  if (!resultId) {
    pendingToolCallIds = new Set();
    pendingChunkStartIndex = null;
  } else {
    pendingToolCallIds.delete(resultId);
  }
  // 只有所有结果都收到，才考虑在此处切分
  if (pendingToolCallIds.size === 0 && ...) {
    splitCurrentAtPendingBoundary();
  }
}
```

测试用例完整地验证了这一约束（`src/agents/compaction.test.ts:81`）：

```typescript
// src/agents/compaction.test.ts:81
it("keeps tool_use and matching toolResult in the same chunk", () => {
```

**对应的修复机制**：当压缩后的历史中存在孤儿 `tool_result`（其 `tool_use` 在被丢弃的部分里），`repairToolUseResultPairing` 函数负责修复：

```typescript
// src/agents/session-transcript-repair.ts:468
export function repairToolUseResultPairing(
  messages: AgentMessage[],
  options?: ToolUseResultPairingOptions,
): ToolUseRepairReport {
  // Anthropic (and Cloud Code Assist) reject transcripts where assistant tool calls are not
  // immediately followed by matching tool results. Session files can end up with results
  // displaced (e.g. after user turns) or duplicated. Repair by:
  // - moving matching toolResult messages directly after their assistant toolCall turn
  // - inserting synthetic error toolResults for missing ids
  // - dropping duplicate toolResults for the same id (anywhere in the transcript)
```

`pruneHistoryForContextShare` 调用此修复函数（`src/agents/compaction.ts:546`），确保丢弃旧 chunk 后的历史依然合法。

---

## 五、三层摘要级联：渐进降级策略

Clawdbot 的摘要生成不是一次调用，而是三层级联，每层都是前一层失败时的退路：

```
summarizeInStages
    ↓ （分阶段：将历史分成 N 块，分别总结，再合并）
summarizeWithFallback
    ↓ （降级 1：跳过超大消息，只总结可处理部分）
[文本描述]
    ↓ （降级 2：连总结都无法生成，只留一句话）
"Context contained N messages. Summary unavailable due to size limits."
```

**第一层：`summarizeInStages`（分阶段总结）**

```typescript
// src/agents/compaction.ts:444-508
export async function summarizeInStages(params: {
  ...
  parts?: number;         // 分成几个阶段（默认 2）
  minMessagesForSplit?: number; // 至少多少条消息才分阶段
```

分阶段总结后，再用一次 LLM 合并所有阶段摘要：

```typescript
// src/agents/compaction.ts:24-36
const MERGE_SUMMARIES_INSTRUCTIONS = [
  "Merge these partial summaries into a single cohesive summary.",
  "",
  "MUST PRESERVE:",
  "- Active tasks and their current status (in-progress, blocked, pending)",
  "- Batch operation progress (e.g., '5/17 items completed')",
  "- The last thing the user requested and what was being done about it",
  ...
].join("\n");
```

合并指令的设计非常有意思：它要求 LLM **优先保留"当前状态"而不是"历史事件"**。这切中了 Agent 使用场景的核心需求——Agent 需要知道"正在做什么"，而不是"讨论过什么"。

**第二层：`summarizeWithFallback`（跳过超大消息）**

```typescript
// src/agents/compaction.ts:406-413
for (const msg of messages) {
  if (isOversizedForSummary(msg, contextWindow)) {
    const role = (msg as { role?: string }).role ?? "message";
    const tokens = estimateCompactionMessageTokens(msg);
    oversizedNotes.push(
      `[Large ${role} (~${Math.round(tokens / 1000)}K tokens) omitted from summary]`,
    );
  } else {
    smallMessages.push(msg);
  }
}
```

超大消息（如包含大量代码的工具结果）会被替换为占位符，但不丢失其存在的记录。

**第三层：最终降级**

```typescript
// src/agents/compaction.ts:438-441
return (
  `Context contained ${messages.length} messages (${oversizedNotes.length} oversized). ` +
  `Summary unavailable due to size limits.`
);
```

即使所有总结都失败，也有一个格式化字符串作为最后的保底——Agent 至少知道"这段历史无法总结"。

同时 `summarizeChunks` 中配置了 3 次重试，带指数退避：

```typescript
// src/agents/compaction.ts:329-337
{
  attempts: 3,
  minDelayMs: 500,
  maxDelayMs: 5000,
  jitter: 0.2,
  label: "compaction/generateSummary",
  shouldRetry: (err) => !isAbortError(err) && !isTimeoutError(err),
}
```

注意 `shouldRetry` 的排除条件：AbortError（用户取消）和 TimeoutError（安全超时触发）**不重试**。

---

## 六、标识符保留策略：LLM 的致命弱点

这是 Compaction 最有趣的安全考量之一。

LLM 在总结时，往往会对 UUID、哈希、Token、API Key 等"不透明标识符"进行：
- **截断**：`abc123...`
- **重构**：`abc-123` → `abc123`
- **替换**：用更简短的占位符

这在任何依赖精确标识符的系统中都是灾难性的。Clawdbot 专门为此设计了标识符保留策略：

```typescript
// src/config/types.agent-defaults.ts（通过 zod-schema 引用）
// 三种策略：strict / off / custom
```

```typescript
// src/agents/compaction.ts:60-82
function resolveIdentifierPreservationInstructions(
  instructions?: CompactionSummarizationInstructions,
): string | undefined {
  const policy = instructions?.identifierPolicy ?? "strict";
  if (policy === "off") {
    return undefined;
  }
  if (policy === "custom") {
    const custom = instructions?.identifierInstructions?.trim();
    return custom && custom.length > 0 ? custom : IDENTIFIER_PRESERVATION_INSTRUCTIONS;
  }
  return IDENTIFIER_PRESERVATION_INSTRUCTIONS;
}
```

默认使用 `strict` 策略，对应的指令（`IDENTIFIER_PRESERVATION_INSTRUCTIONS`）非常具体：

```
Preserve all opaque identifiers exactly as written (no shortening or reconstruction),
including UUIDs, hashes, IDs, tokens, API keys, hostnames, IPs, ports, URLs, and file names.
```

测试文件完整地验证了这三种策略（`src/agents/compaction.identifier-policy.test.ts:4`）：

```typescript
// src/agents/compaction.identifier-policy.test.ts:5-28
it("defaults to strict identifier preservation", () => { ... });
it("can disable identifier preservation with off policy", () => { ... });
it("supports custom identifier instructions", () => { ... });
it("falls back to strict text when custom policy is missing instructions", () => { ... });
```

**工程洞察**：这个设计反映了一个深刻认识——不是所有 LLM 任务都应该让 LLM 自由发挥。对于"精确复现"类的需求，必须用 prompt engineering 约束 LLM 的行为。标识符保留策略本质上是"给 LLM 装护栏"。

---

## 七、安全隔离：toolResult.details 永不进入摘要

这是 Compaction 代码里最引人注目的安全注释之一：

```typescript
// src/agents/compaction.ts:104
// SECURITY: toolResult.details can contain untrusted/verbose payloads; never include in LLM-facing compaction.
const safe = stripToolResultDetails(messages);
```

```typescript
// src/agents/compaction.ts:308
// SECURITY: never feed toolResult.details into summarization prompts.
const safeMessages = stripToolResultDetails(params.messages);
```

这两处注释说明了一个重要的威胁模型：工具调用结果（`toolResult.details`）可能包含来自外部系统的、不可信的大量数据（如 API 响应、文件内容、网页内容）。如果这些内容直接进入摘要 LLM 的上下文：

1. **Prompt injection 风险**：恶意数据可以操控摘要内容
2. **Token 浪费**：详细的工具输出应该由 Agent 处理，而非摘要化
3. **信息泄漏**：某些 `details` 可能包含不应持久化的敏感信息

解决方案是在所有进入 LLM 的路径上都过滤 `details` 字段，包括 Token 估算（`src/agents/compaction.ts:103-106`）和摘要生成（第 308 行）。

测试用例专门验证了这一点（`src/agents/compaction.token-sanitize.test.ts:23`）：

```typescript
it("does not pass toolResult.details into per-message token estimates", () => {
```

---

## 八、压缩安全超时：15 分钟的硬限制

压缩操作是昂贵的——它需要调用 LLM 生成摘要，可能需要多轮调用（分阶段总结）。如果 LLM 响应慢或网络不稳定，压缩可能陷入长时间等待。

Clawdbot 为此设计了一个安全超时：

```typescript
// src/agents/pi-embedded-runner/compaction-safety-timeout.ts:4
export const EMBEDDED_COMPACTION_TIMEOUT_MS = 900_000; // 15 分钟
```

超时是可配置的（`resolveCompactionTimeoutMs`，第 18 行），读取 `config.agents.defaults.compaction.timeoutSeconds`，但最终由 `compactWithSafetyTimeout` 强制执行：

```typescript
// src/agents/pi-embedded-runner/compaction-safety-timeout.ts:26-93
export async function compactWithSafetyTimeout<T>(
  compact: () => Promise<T>,
  timeoutMs: number = EMBEDDED_COMPACTION_TIMEOUT_MS,
  opts?: {
    abortSignal?: AbortSignal;
    onCancel?: () => void;  // 超时时调用此 hook
  },
): Promise<T>
```

注意 `onCancel` 的存在：超时不只是中断任务，还会触发回调。在 `compact.ts` 中，这个回调用于设置 `safeguardCancelReason`（`src/agents/pi-embedded-runner/compact.ts:993`），后续通过 `resolveCompactionFailureReason` 诊断失败原因：

```typescript
// src/agents/pi-embedded-runner/compact-reasons.ts:8-16
export function resolveCompactionFailureReason(params: {
  reason: string;
  safeguardCancelReason?: string | null;
}): string {
  // 如果是通用取消原因，用 safeguardCancelReason 替代（更具体）
  if (isGenericCompactionCancelledReason(params.reason) && params.safeguardCancelReason) {
    return params.safeguardCancelReason;
  }
  return params.reason;
}
```

超时测试（`src/agents/pi-embedded-runner.compaction-safety-timeout.test.ts`）覆盖了 6 种场景，包括外部 AbortSignal 的交互（第 68 行）——双信号竞争（超时 vs 外部中止）。

---

## 九、检查点快照：压缩是可回退的

**这是 Compaction 系统里最有价值的保险机制**。压缩操作会修改会话历史，如果压缩后 Agent 发现信息丢失、需要回溯，或者压缩本身出了 bug，能否恢复？

Clawdbot 的答案是：每次压缩前先备份。

```typescript
// src/gateway/session-compaction-checkpoints.ts:56-104
export function captureCompactionCheckpointSnapshot(params: {
  sessionManager: Pick<SessionManager, "getLeafId">;
  sessionFile: string;
}): CapturedCompactionCheckpointSnapshot | null {
  // ...
  // 生成唯一文件名：sessionFile.checkpoint.<UUID>.jsonl
  const snapshotFile = path.join(
    parsedSessionFile.dir,
    `${parsedSessionFile.name}.checkpoint.${randomUUID()}${parsedSessionFile.ext || ".jsonl"}`,
  );
  try {
    fsSync.copyFileSync(sessionFile, snapshotFile); // 同步复制，确保快照完整
  } catch {
    return null;
  }
```

返回的 `CapturedCompactionCheckpointSnapshot` 包含三个字段：
- `sessionId`：快照的 Session ID
- `sessionFile`：快照文件路径（`.checkpoint.<UUID>.jsonl`）
- `leafId`：会话树的叶节点 ID（用于定位压缩点）

压缩完成后，将快照元数据注册到 Session Store：

```typescript
// src/gateway/session-compaction-checkpoints.ts:140-163
const checkpoint: SessionCompactionCheckpoint = {
  checkpointId: randomUUID(),
  reason: params.reason,      // "auto-threshold" / "overflow-retry" / "timeout-retry" / "manual"
  preCompaction: {            // 压缩前的会话信息
    sessionId: params.snapshot.sessionId,
    sessionFile: params.snapshot.sessionFile,
    leafId: params.snapshot.leafId,
  },
  postCompaction: {           // 压缩后的会话信息
    sessionId: params.sessionId,
    sessionFile: params.postSessionFile,
    leafId: params.postLeafId,
  },
};
```

每个 Session 最多保留 25 个检查点（`src/gateway/session-compaction-checkpoints.ts:17`）：

```typescript
const MAX_COMPACTION_CHECKPOINTS_PER_SESSION = 25;
```

超过后滑动丢弃最旧的（`trimSessionCheckpoints`），以 FIFO 方式保留最近的历史。

**四种触发原因的检查点**：

```typescript
// src/gateway/session-compaction-checkpoints.ts:44-54
export function resolveSessionCompactionCheckpointReason(params: {
  trigger?: "budget" | "overflow" | "manual";
  timedOut?: boolean;
}): SessionCompactionCheckpointReason {
  if (params.trigger === "manual") { return "manual"; }
  if (params.timedOut) { return "timeout-retry"; }         // 上次压缩因超时失败，重试
  if (params.trigger === "overflow") { return "overflow-retry"; } // 超出上下文窗口
  return "auto-threshold";  // 达到 token 预算阈值
}
```

---

## 十、什么算真正的对话内容？

压缩不应该对"无意义"的消息进行总结。Clawdbot 专门设计了一个过滤器来判断哪些消息值得被总结：

```typescript
// src/agents/compaction-real-conversation.ts:5-8
export const TOOL_RESULT_REAL_CONVERSATION_LOOKBACK = 20;
const NON_CONVERSATION_BLOCK_TYPES = new Set([
  "toolCall",
  "toolUse",
  "functionCall",
  "thinking",    // CoT 推理块不算对话内容
  "reasoning",   // 同上
]);
```

`hasMeaningfulConversationContent`（第 29 行）的过滤逻辑：
- 纯工具调用块（`toolCall/toolUse/functionCall`）：不算
- 内部推理块（`thinking/reasoning`）：不算
- 心跳 token（`isSilentReplyText`，第 19 行）：不算
- 剥去心跳后有文本内容：算
- 有其他类型的块（非以上）：算

这个过滤器防止 Clawdbot 对仅包含心跳（keepalive 信号）或内部 CoT 的会话历史进行无意义的摘要。

---

## 十一、压缩失败的分类诊断

压缩失败后，`classifyCompactionReason`（`src/agents/pi-embedded-runner/compact-reasons.ts:18`）将错误信息分类为结构化的失败码，供监控和告警使用：

| 失败码 | 对应错误文本特征 |
|--------|----------------|
| `no_compactable_entries` | "nothing to compact" |
| `below_threshold` | "below threshold" |
| `already_compacted_recently` | "already compacted" |
| `live_context_still_exceeds_target` | "still exceeds target" |
| `guard_blocked` | 含 "guard" |
| `summary_failed` | 含 "summary" |
| `timeout` | 含 "timed out" / "timeout" |
| `provider_error_4xx` | 含 "400/401/403/429" |
| `provider_error_5xx` | 含 "500/502/503/504" |

这个字符串匹配分类器是一个务实的设计：通过分析错误文本中的关键词，而非依赖结构化错误码（因为底层 LLM SDK 并不保证结构化错误）。对 HTTP 状态码的特殊处理反映了 LLM API 调用失败的常见模式——网络/限流错误往往比 LLM 错误更常见。

---

## 十二、整体架构图

```
触发压缩（budget / overflow / manual）
         │
         ▼
captureCompactionCheckpointSnapshot   ← 快照备份（先保存，后修改）
         │
         ▼
compactWithSafetyTimeout(compact, 900_000ms)
         │
    ┌────┴────┐
    │         │
 超时触发  正常执行
 onCancel  compact()
    │         │
    │    summarizeInStages
    │         │
    │    [分 N 阶段] → summarizeWithFallback (×N)
    │         │                │
    │    [合并摘要]         [跳过超大消息]
    │         │                │
    └────►成功 / 失败◄──────────┘
              │
     ┌────────┴────────┐
     │                 │
   成功             失败（分类 → classifyCompactionReason）
     │                 │
persistCheckpoint   清理快照文件
（保留至多25个）
```

---

## 十三、可以学到什么

这套 Compaction 系统对于任何构建长期运行 AI Agent 的工程师都有参考价值：

**1. 安全边距优于精确估算**  
用 chars/4 + 20% buffer 取代精确 tokenizer，是"够用比精确更重要"的工程决策。Token 估算用于触发压缩，不需要精确到个位数。

**2. 保持 API 协议不变式**  
Anthropic API 要求 tool_call/tool_result 相邻。这个约束必须在所有操作历史的代码路径上都得到保证——切分、丢弃、修复。"守护不变式"是分布式系统和 API 集成的共同原则。

**3. 关键操作必须有检查点**  
压缩是破坏性的，但可以通过先备份来使其变成可回退的。这个 copy-then-modify 模式在数据库（WAL）和文件系统（写前日志）中普遍存在。

**4. 渐进降级而非快速失败**  
当整体总结失败，尝试跳过超大消息的局部总结；当局部总结也失败，留下一行文字说明。用户收到的信息越具体，Agent 的可恢复性越强。

**5. LLM 需要护栏**  
标识符保留策略通过 prompt engineering 约束 LLM 不乱改 UUID/哈希。在任何需要 LLM 精确复现内容的场景，都需要类似的机制。

**6. 区分安全边界**  
`toolResult.details` 来自外部系统，是不可信边界。即使 LLM 整体上是受信的，也要对不可信数据的传播路径设立明确的边界。

---

## 引用来源

### 源码引用

1. `src/agents/compaction.ts:19-23` — 核心常量（BASE_CHUNK_RATIO、SAFETY_MARGIN、DEFAULT_PARTS）
2. `src/agents/compaction.ts:24-36` — MERGE_SUMMARIES_INSTRUCTIONS 合并摘要指令
3. `src/agents/compaction.ts:85-101` — buildCompactionSummarizationInstructions 指令构建
4. `src/agents/compaction.ts:103-107` — SECURITY: toolResult.details 过滤（Token 估算层）
5. `src/agents/compaction.ts:120-208` — splitMessagesByTokenShare 分块算法（工具调用配对不变式）
6. `src/agents/compaction.ts:214-256` — SUMMARIZATION_OVERHEAD_TOKENS + chunkMessagesByMaxTokens
7. `src/agents/compaction.ts:262-281` — computeAdaptiveChunkRatio 自适应 chunk 比例
8. `src/agents/compaction.ts:287-290` — isOversizedForSummary 单条消息超大判断
9. `src/agents/compaction.ts:308-338` — summarizeChunks（SECURITY 过滤 + 重试配置）
10. `src/agents/compaction.ts:380-441` — summarizeWithFallback 降级总结
11. `src/agents/compaction.ts:444-508` — summarizeInStages 分阶段总结 + 合并
12. `src/agents/compaction.ts:510-572` — pruneHistoryForContextShare（丢弃旧 chunk + orphan repair）
13. `src/agents/pi-embedded-runner/compaction-safety-timeout.ts:4` — EMBEDDED_COMPACTION_TIMEOUT_MS = 900_000
14. `src/agents/pi-embedded-runner/compaction-safety-timeout.ts:18-23` — resolveCompactionTimeoutMs
15. `src/agents/pi-embedded-runner/compaction-safety-timeout.ts:26-93` — compactWithSafetyTimeout 实现
16. `src/agents/pi-embedded-runner/compact-reasons.ts:3-57` — classifyCompactionReason 失败分类
17. `src/gateway/session-compaction-checkpoints.ts:17` — MAX_COMPACTION_CHECKPOINTS_PER_SESSION = 25
18. `src/gateway/session-compaction-checkpoints.ts:44-54` — resolveSessionCompactionCheckpointReason
19. `src/gateway/session-compaction-checkpoints.ts:56-104` — captureCompactionCheckpointSnapshot
20. `src/gateway/session-compaction-checkpoints.ts:120-187` — persistSessionCompactionCheckpoint
21. `src/agents/compaction-real-conversation.ts:5-8` — NON_CONVERSATION_BLOCK_TYPES 过滤集合
22. `src/agents/compaction-real-conversation.ts:29-85` — hasMeaningfulConversationContent + isRealConversationMessage
23. `src/agents/pi-compaction-constants.ts:6-12` — MIN_PROMPT_BUDGET_TOKENS + MIN_PROMPT_BUDGET_RATIO
24. `src/agents/session-transcript-repair.ts:468-520` — repairToolUseResultPairing
25. `src/agents/compaction.test.ts:81` — 工具调用配对不变式测试
26. `src/agents/compaction.test.ts:285` — 孤儿 tool_result 丢弃测试
27. `src/agents/compaction.identifier-policy.test.ts:4-36` — 标识符保留策略测试
28. `src/agents/compaction.token-sanitize.test.ts:23` — toolResult.details 安全过滤测试
29. `src/agents/pi-embedded-runner.compaction-safety-timeout.test.ts:8` — 超时测试套件

### 外部参考

30. [Anthropic 官方文档：Claude 上下文窗口](https://docs.anthropic.com/en/docs/build-with-claude/context-windows) — Claude 各模型上下文窗口规格
31. [Anthropic 官方文档：Tool use with Claude](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) — tool_call/tool_result 配对协议规范
32. [Agenta: Top techniques to Manage Context Lengths in LLMs](https://agenta.ai/blog/top-6-techniques-to-manage-context-length-in-llms) — 上下文管理方法综述（截断 vs 压缩 vs 摘要）
33. [arXiv: Recursively Summarizing Enables Long-Term Dialogue Memory in Large Language Models](https://arxiv.org/html/2308.15022v3) — 递归摘要用于长对话记忆的研究
34. [mem0.ai: LLM Chat History Summarization Best Practices](https://mem0.ai/blog/llm-chat-history-summarization-guide-2025) — 对话历史摘要的工程实践
35. [Microsoft Learn: Compaction in Agent Conversations](https://learn.microsoft.com/en-us/agent-framework/agents/conversations/compaction) — 微软 Agent 框架对压缩机制的定义
36. [Eden AI: Understanding LLM Billing - From Characters to Tokens](https://www.edenai.co/post/understanding-llm-billing-from-characters-to-tokens) — chars/4 启发式 Token 估算方法背景
37. [Traceloop: A Comprehensive Guide to Tokenizing Text for LLMs](https://www.traceloop.com/blog/a-comprehensive-guide-to-tokenizing-text-for-llms) — LLM Token 化原理与估算方法
38. [Factory.ai: Compressing Context](https://factory.ai/news/compressing-context) — 生产环境中的上下文压缩工程实践
39. [OWASP: Prompt Injection (LLM01)](https://owasp.org/www-project-top-10-for-large-language-model-applications/) — toolResult.details 安全过滤背后的威胁模型参考

**总引用数：29 个源码位置 + 10 个外部链接 = 39 个总引用**
