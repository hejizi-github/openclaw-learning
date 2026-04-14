# ACP 网关桥接：跨协议 AI Agent 通信的工程实践

> **主题**：clawdbot 的 ACP（Agent Client Protocol）集成  
> **核心文件**：`src/acp/translator.ts`（1418行）、`src/acp/approval-classifier.ts`（227行）、`src/acp/event-mapper.ts`（410行）  
> **读者收获**：理解多协议桥接、断线重连容错、工具调用安全分级、流式文本重分段等工程模式

---

## 背景：两个世界需要一座桥

2024 年底，AI 编程 Agent 生态出现了一场"协议战"——不同 IDE、不同 Agent 之间各自为政，无法互通。Zed 编辑器联合 JetBrains、Neovim 等社区提出了 **ACP（Agent Client Protocol）**，定义了编辑器与编程 Agent 之间通信的标准接口。与此同时，AI Agent 领域还有 Anthropic 主导的 **MCP（Model Context Protocol）** 和 Google 主导的 **A2A（Agent-to-Agent）**。三者并不竞争，分工明确：MCP 管工具调用、A2A 管 Agent 间协作、ACP 管 IDE-Agent 对话。

clawdbot 内部使用自己的 Gateway 协议（WebSocket + JSON-RPC）管理 Agent 会话。为了让标准 ACP 客户端（如 Zed、JetBrains）直接对接 clawdbot，需要在两套协议之间建一座桥。`AcpGatewayAgent`（[`src/acp/translator.ts:420`](src/acp/translator.ts)）就是这座桥的核心实现。

---

## 一、协议协商：ACP 握手流程

ACP 协议的第一步是 `initialize`，双方声明能力集：

```typescript
// src/acp/translator.ts:503-524
async initialize(_params: InitializeRequest): Promise<InitializeResponse> {
  return {
    protocolVersion: PROTOCOL_VERSION,
    agentCapabilities: {
      loadSession: true,
      promptCapabilities: {
        image: true,
        audio: false,
        embeddedContext: true,
      },
      mcpCapabilities: {
        http: false,
        sse: false,
      },
      sessionCapabilities: {
        list: {},
      },
    },
    agentInfo: ACP_AGENT_INFO,
    authMethods: [],
  };
}
```

这里有个有趣的细节：`mcpCapabilities.http` 和 `sse` 都是 `false`。ACP 允许 Agent 声明"我支持 MCP over HTTP/SSE"，让 IDE 直接向 Agent 注入 MCP 服务器。但 clawdbot 选择拒绝这个能力，原因在 [`src/acp/translator.ts:1400-1407`](src/acp/translator.ts)：

```typescript
private assertSupportedSessionSetup(mcpServers: ReadonlyArray<unknown>): void {
  if (mcpServers.length === 0) {
    return;
  }
  throw new Error(
    "ACP bridge mode does not support per-session MCP servers. Configure MCP on the OpenClaw gateway or agent instead.",
  );
}
```

设计决策：MCP 配置应该在 Gateway 层做，而不是在每个 ACP 会话里注入。这避免了每次连接都重新配置 MCP 服务的开销，也让配置管理更集中。

---

## 二、会话身份：双层 ID 的映射

ACP 世界里用 UUID 标识会话（`sessionId`），Gateway 世界里用结构化字符串标识（`sessionKey`，如 `agent:main:my-project`）。桥接层需要维护这两套 ID 的映射关系。

### 会话键解析

[`src/acp/session-mapper.ts:27-85`](src/acp/session-mapper.ts) 实现了 5 级优先级的会话键解析：

```typescript
export async function resolveSessionKey(params: {
  meta: AcpSessionMeta;
  fallbackKey: string;
  gateway: GatewayClient;
  opts: AcpServerOptions;
}): Promise<string> {
  // 优先级 1：_meta 里的 sessionLabel（通过 gateway 解析为 key）
  if (params.meta.sessionLabel) {
    const resolved = await params.gateway.request<{ ok: true; key: string }>(
      "sessions.resolve",
      { label: params.meta.sessionLabel },
    );
    return resolved.key;
  }
  // 优先级 2：_meta 里直接指定的 sessionKey
  if (params.meta.sessionKey) { ... }
  // 优先级 3：opts 里的 defaultSessionLabel
  if (requestedLabel) { ... }
  // 优先级 4：opts 里的 defaultSessionKey
  if (requestedKey) { ... }
  // 优先级 5：fallbackKey（如 acp:{uuid}）
  return params.fallbackKey;
}
```

**设计亮点**：ACP 协议的 `_meta` 字段是开放扩展字段，clawdbot 借此传入 `sessionKey` 或 `sessionLabel`，让高级客户端可以精确绑定到特定的 Gateway 会话，而普通客户端则自动获得一个新的隔离会话。

### 持久绑定的会话键构造

对于渠道（Telegram/Discord 等）绑定的 ACP 会话，会话键通过 SHA-256 哈希保证确定性（[`src/acp/persistent-bindings.types.ts:60-78`](src/acp/persistent-bindings.types.ts)）：

```typescript
export function buildConfiguredAcpSessionKey(spec: ConfiguredAcpBindingSpec): string {
  const hash = createHash("sha256")
    .update(`${params.channel}:${params.accountId}:${params.conversationId}`)
    .digest("hex")
    .slice(0, 16);
  return `agent:${sanitizeAgentId(spec.agentId)}:acp:binding:${spec.channel}:${spec.accountId}:${hash}`;
}
```

同一个 channel + account + conversation 组合永远得到相同的 sessionKey，确保跨连接的会话连续性，同时用截断哈希代替明文 conversationId，避免敏感数据泄露到会话键里。

---

## 三、流式文本重分段：从快照到增量

这是整个翻译层最精巧的一段代码。

**问题**：Gateway 的 `chat` 事件在每次 `delta` 推送时携带的是**完整的消息快照**——从对话开始到当前时刻的所有文本。但 ACP 协议期望的是**增量块**（chunk）——每次只发新增的部分。

```
Gateway delta 1: "正在分析"
Gateway delta 2: "正在分析你的代码"
Gateway delta 3: "正在分析你的代码，发现了"
Gateway final:   "正在分析你的代码，发现了 3 个问题。"
```

如果直接把快照转发给 ACP 客户端，用户会看到文字不断从头开始重复渲染。

**解决方案**：用游标跟踪已发送的字节数（[`src/acp/translator.ts:974-1023`](src/acp/translator.ts)）：

```typescript
private async handleDeltaEvent(
  sessionId: string,
  messageData: Record<string, unknown>,
): Promise<void> {
  const pending = this.pendingPrompts.get(sessionId);

  const fullText = content
    ?.filter((block) => block?.type === "text")
    .map((block) => block.text ?? "")
    .join("\n")
    .trimEnd();
  
  const sentSoFar = pending.sentTextLength ?? 0;
  if (!fullText || fullText.length <= sentSoFar) {
    return;  // 快照没有新内容，跳过
  }

  const newText = fullText.slice(sentSoFar);  // 只取新增部分
  pending.sentTextLength = fullText.length;    // 更新游标
  pending.sentText = fullText;
  
  await this.connection.sessionUpdate({
    sessionId,
    update: {
      sessionUpdate: "agent_message_chunk",
      content: { type: "text", text: newText },  // 只发增量
    },
  });
}
```

思考（thinking）块使用相同的模式（`sentThoughtLength`），分别维护游标，保证思考流和回复流互不干扰。

这个模式叫做**快照转增量（Snapshot-to-Delta）**，在数据流系统中很常见（如 Change Data Feed）。核心思路：接收方维护"已消费位置"游标，每次收到快照时计算 `snapshot[cursor:]` 并发出，然后移动游标。

---

## 四、断线重连容错：generation 计数器模式

这是整个 ACP 集成里最复杂也最工程化的部分。

**问题**：ACP 客户端发出 `prompt()` 请求，等待 Gateway 响应。如果在等待过程中 Gateway 断线了，该怎么处理？

简单做法：直接 reject Promise，让客户端重试。  
**问题**：Gateway 断线时，该 prompt 可能已经被 Gateway 接收并处理完了，只是响应没传回来。直接 reject 导致客户端重发同一个消息，AI 回复了两次。

clawdbot 的解决方案是**三阶段断线处理**：

### 阶段 1：断线时，开 5 秒宽限窗口

```typescript
// src/acp/translator.ts:476-491
handleGatewayDisconnect(reason: string): void {
  const disconnectContext = {
    generation: this.disconnectGeneration + 1,  // 生代递增
    reason,
  };
  this.disconnectGeneration = disconnectContext.generation;
  this.activeDisconnectContext = disconnectContext;
  
  // 给所有 pending prompts 打上"当前断线上下文"标记
  for (const pending of this.pendingPrompts.values()) {
    pending.disconnectContext = disconnectContext;
  }
  
  // 5秒后触发 deadline 处理
  this.armDisconnectTimer(disconnectContext);
}
```

`generation` 是个递增计数器，每次断线递增。它的作用是**版本号**：每个 pending prompt 记录它所属的断线生代，避免跨生代的状态混淆（第一次断线的 timer 不会错误处理第二次断线的 prompts）。

### 阶段 2：重连时，用 `agent.wait` 轮询

```typescript
// src/acp/translator.ts:1110-1143
handleGatewayReconnect(): void {
  void this.reconcilePendingPrompts(disconnectContext.generation, false);
}

private async reconcilePendingPrompt(...): Promise<boolean> {
  // 用 timeoutMs: 0 非阻塞查询服务端是否已完成该 run
  const result = await this.gateway.request(
    "agent.wait",
    { runId: pending.idempotencyKey, timeoutMs: 0 },
    { timeoutMs: null },
  );
  
  if (result?.status === "ok") {
    await this.finishPrompt(sessionId, currentPending, "end_turn");  // 成功！
    return false;
  }
  if (result?.status === "timeout") {
    return true;  // 还没完成，继续等
  }
}
```

`agent.wait` 是 Gateway 提供的"查询指定 runId 是否完成"接口。`timeoutMs: 0` 意味着立即返回当前状态，不阻塞等待。

### 阶段 3：5 秒宽限过期后，差异化处理

```typescript
// src/acp/translator.ts:1099-1108
private shouldRejectPendingAtDisconnectDeadline(
  pending: PendingPrompt,
  disconnectContext: DisconnectContext,
): boolean {
  return (
    pending.disconnectContext === disconnectContext &&
    (!pending.sendAccepted ||  // 未被 Gateway 确认收到
     this.activeDisconnectContext?.generation === disconnectContext.generation)
  );
}
```

关键在 `sendAccepted` 标志：

- **未确认（pre-ack）**：`chat.send` 请求还没收到 Gateway 的 200 OK，Gateway 可能根本没收到这条消息 → 安全 reject
- **已确认（post-ack）**：Gateway 已收到，但处理中途断线 → 继续用 `agent.wait` 轮询，宁可等也不要 reject

测试文件把这些场景都列举得非常清楚（[`src/acp/translator.stop-reason.test.ts:143-587`](src/acp/translator.stop-reason.test.ts)），包括：

- `keeps in-flight prompts pending across transient gateway disconnects`：短暂断线不影响进行中的请求
- `rejects in-flight prompts when the gateway does not reconnect before the grace window`：5秒内未重连则 reject
- `reconciles a missed final event on reconnect via agent.wait`：重连后通过 agent.wait 补回漏掉的完成事件
- `does not let a stale disconnect deadline reject a newer prompt on the same session`：旧生代的 deadline timer 不影响新请求

---

## 五、工具调用安全分级：不是简单的允许/拒绝

当 ACP 客户端需要决定是否自动批准某个工具调用时，需要一套细粒度的分级体系。`classifyAcpToolApproval`（[`src/acp/approval-classifier.ts:183-227`](src/acp/approval-classifier.ts)）实现了 8 个审批类别：

```typescript
export type AcpApprovalClass =
  | "readonly_scoped"   // 在 CWD 范围内的只读操作 → 自动批准
  | "readonly_search"   // 搜索操作 → 自动批准
  | "mutating"          // 文件写入等变更操作 → 需要审批
  | "exec_capable"      // exec/bash/shell 等执行类工具 → 需要审批
  | "control_plane"     // sessions_spawn 等控制面操作 → 需要审批
  | "interactive"       // 交互式工具 → 需要审批
  | "other"             // 其他已知工具 → 需要审批
  | "unknown";          // 无法识别 → 需要审批
```

### 工具名称的三源交叉验证

工具名称可以从三个地方获取：`_meta.toolName`、`rawInput.tool`、`title` 前缀（如 `read: /path`）。一旦三者不一致，整个识别过程失败，返回 `unknown`（[`src/acp/approval-classifier.ts:72-102`](src/acp/approval-classifier.ts)）：

```typescript
const metaName = fromMeta ? normalizeToolName(fromMeta) : undefined;
const rawInputName = fromRawInput ? normalizeToolName(fromRawInput) : undefined;
const titleName = fromTitle;

// 任意两个来源之间不一致 → 拒绝识别
if (metaName && titleName && metaName !== titleName) {
  return undefined;
}
if (rawInputName && metaName && rawInputName !== metaName) {
  return undefined;
}
```

**为什么要这样设计？** 防止名称注入攻击：如果 `_meta` 里声明工具名是 `read`，但 `rawInput` 里实际执行的是 `bash`，通过一致性检查可以早期发现这种不一致，而不是贸然自动批准。

### CWD 范围内的自动批准

`read` 工具有特殊的自动批准逻辑：只有当读取路径在当前工作目录（CWD）范围内时才自动批准（[`src/acp/approval-classifier.ts:140-181`](src/acp/approval-classifier.ts)）：

```typescript
function resolveAbsoluteScopedPath(value: string, cwd: string): string | undefined {
  let candidate = value.trim();
  
  // 处理 file:// URI
  if (candidate.startsWith("file://")) {
    const parsed = new URL(candidate);
    candidate = decodeURIComponent(parsed.pathname || "");
  }
  
  // 处理 ~ 展开
  if (candidate === "~") {
    candidate = homedir();
  } else if (candidate.startsWith("~/")) {
    candidate = path.join(homedir(), candidate.slice(2));
  }
  
  return path.isAbsolute(candidate)
    ? path.normalize(candidate)
    : path.resolve(cwd, candidate);
}

function isReadToolCallScopedToCwd(...): boolean {
  const absolutePath = resolveAbsoluteScopedPath(rawPath, cwd);
  const root = path.resolve(cwd);
  const relative = path.relative(root, absolutePath);
  // 路径遍历检测：../ 开头则不在 CWD 范围内
  return relative === "" || (!relative.startsWith("..") && !path.isAbsolute(relative));
}
```

这里的路径处理覆盖了 4 种攻击面：
1. `../../etc/passwd` 路径遍历
2. `file:///etc/passwd` URI 注入
3. `~/.ssh/id_rsa` Home 目录访问
4. 任意绝对路径逃逸

---

## 六、Provenance（来源追溯）：三级可观测性

当 ACP 客户端通过网关发消息时，Gateway 需要知道这条消息的来源。clawdbot 提供三级 provenance 模式（[`src/acp/types.ts:5-7`](src/acp/types.ts)）：

```typescript
export const ACP_PROVENANCE_MODE_VALUES = ["off", "meta", "meta+receipt"] as const;
```

- **`off`**：不附加任何来源信息
- **`meta`**：在 Gateway 请求中附加 originSessionId 和 sourceChannel
- **`meta+receipt`**：还在用户消息里嵌入文本收据

收据格式（[`src/acp/translator.ts:403-418`](src/acp/translator.ts)）：

```typescript
function buildSystemProvenanceReceipt(params: {
  cwd: string;
  sessionId: string;
  sessionKey: string;
}) {
  return [
    "[Source Receipt]",
    "bridge=openclaw-acp",
    `originHost=${os.hostname()}`,
    `originCwd=${shortenHomePath(params.cwd)}`,
    `acpSessionId=${params.sessionId}`,
    `originSessionId=${params.sessionId}`,
    `targetSession=${params.sessionKey}`,
    "[/Source Receipt]",
  ].join("\n");
}
```

这个收据是嵌入到用户消息文本里的，LLM 会看到它。这么设计的原因：在分布式多 Agent 架构中，一个 Gateway 会话可能接收来自多个 ACP 客户端的消息，provenance 收据让 LLM 和审计日志都能追踪到消息的真实来源。

还有一个值得关注的降级逻辑（[`src/acp/translator.ts:735-763`](src/acp/translator.ts)）：如果 Gateway 因为权限不足拒绝了带 provenance 的请求，自动降级重试无 provenance 版本：

```typescript
const sendWithProvenanceFallback = async () => {
  try {
    await this.gateway.request("chat.send", {
      ...requestParams,
      systemInputProvenance,
      systemProvenanceReceipt,
    }, { timeoutMs: null });
  } catch (err) {
    if (isAdminScopeProvenanceRejection(err)) {
      // 权限不足时，降级为无 provenance 请求
      await this.gateway.request("chat.send", requestParams, { timeoutMs: null });
    }
  }
};
```

---

## 七、DoS 防护：深度防御

ACP 是面向外部客户端的接口，代码里明确标注了两处 DoS 防护（[`src/acp/translator.ts:54-55`](src/acp/translator.ts)）：

```typescript
// Maximum allowed prompt size (2MB) to prevent DoS via memory exhaustion (CWE-400, GHSA-cxpw-2g23-2vgw)
const MAX_PROMPT_BYTES = 2 * 1024 * 1024;
```

注意注释里直接引用了 CWE-400（Uncontrolled Resource Consumption）和 GHSA 编号——说明这个限制是在修复真实漏洞后加入的，不是"防御性编程"。

检查在两个层次进行（[`src/acp/translator.ts:686-707`](src/acp/translator.ts)）：

```typescript
// 层次 1：extractTextFromPrompt 内部逐块累加，不等到全量拼接
const userText = extractTextFromPrompt(params.prompt, MAX_PROMPT_BYTES);

// 层次 2：加上 CWD 前缀后再检查（前缀本身也会增加体积）
if (Buffer.byteLength(message, "utf-8") > MAX_PROMPT_BYTES) {
  throw new Error(`Prompt exceeds maximum allowed size of ${MAX_PROMPT_BYTES} bytes`);
}
```

`extractTextFromPrompt` 的逐块检查（[`src/acp/event-mapper.ts:245-276`](src/acp/event-mapper.ts)）在分块处理时就能早期终止，避免在内存中拼接超大字符串：

```typescript
if (maxBytes !== undefined) {
  const separatorBytes = parts.length > 0 ? 1 : 0;
  totalBytes += separatorBytes + Buffer.byteLength(blockText, "utf-8");
  if (totalBytes > maxBytes) {
    throw new Error(`Prompt exceeds maximum allowed size of ${maxBytes} bytes`);
  }
}
```

会话创建速率限制（[`src/acp/translator.ts:449-459`](src/acp/translator.ts)）默认每 10 秒最多 120 个会话，防止会话爆炸攻击：

```typescript
this.sessionCreateRateLimiter = createFixedWindowRateLimiter({
  maxRequests: opts.sessionCreateRateLimit?.maxRequests
    ?? SESSION_CREATE_RATE_LIMIT_DEFAULT_MAX_REQUESTS,  // 120
  windowMs: opts.sessionCreateRateLimit?.windowMs
    ?? SESSION_CREATE_RATE_LIMIT_DEFAULT_WINDOW_MS,     // 10_000
});
```

---

## 八、工具位置提取：结构化元数据的自动发现

`event-mapper.ts` 里有一个很实用的功能：自动从工具调用的参数中提取"文件位置"信息，供 IDE 在 UI 中展示（跳转到该文件、高亮该行）。

提取策略是深度优先遍历（[`src/acp/event-mapper.ts:187-243`](src/acp/event-mapper.ts)），支持两种位置信息来源：

1. **结构化路径键**（[`src/acp/event-mapper.ts:22-42`](src/acp/event-mapper.ts)）：
   ```typescript
   const TOOL_LOCATION_PATH_KEYS = [
     "path", "filePath", "file_path",
     "targetPath", "target_path",
     "sourcePath", "source_path",
     "destinationPath", "destination_path",
     "oldPath", "new_path", "outputPath", "inputPath", ...
   ] as const;
   ```
   
   覆盖了几乎所有可能的路径键命名风格（camelCase、snake_case、各种语义前缀）。

2. **文本标记**（[`src/acp/event-mapper.ts:51`](src/acp/event-mapper.ts)）：
   ```typescript
   const TOOL_RESULT_PATH_MARKER_RE = /^(?:FILE|MEDIA):(.+)$/gm;
   ```
   
   工具结果文本里 `FILE:/path/to/file` 这样的标记也会被自动提取为位置信息。

有两个安全边界：`TOOL_LOCATION_MAX_DEPTH = 4`（最大递归深度）和 `TOOL_LOCATION_MAX_NODES = 100`（最大遍历节点数），防止恶意嵌套的 JSON 导致栈溢出或无限遍历。

---

## 架构总结：桥接模式的四个维度

回顾整个 ACP 集成，可以总结出一个"四维桥接模式"：

```
ACP 协议维度          Gateway 协议维度
─────────────────    ──────────────────────
sessionId (UUID)  ↔  sessionKey (结构化字符串)
prompt() 请求     ↔  chat.send + 事件流监听
快照事件流        →  增量 chunk 流（游标重分段）
工具调用批准请求  →  classifyAcpToolApproval（8级分类）
```

**可移植的工程原则**：

1. **双 ID 映射 + 层级解析**：外部协议 ID 永远不要直接用作内部 ID，通过解析层隔离，内部可以自由重构
2. **生代计数器处理断线**：不用锁，不用复杂状态机，`generation++` 就能区分多次断线的 pending 操作，天然支持并发
3. **快照转增量用游标**：接收端维护"已消费位置"，每次从游标处截取新增部分，适用于任何"快照推送"的场景
4. **工具名三源交叉验证**：关键的安全决策不信任单一来源，多个来源必须一致才算有效，不一致时 fail-safe

---

## 引用来源

### 源码位置（30 处）

| 位置 | 内容 |
|------|------|
| `src/acp/translator.ts:54-65` | MAX_PROMPT_BYTES 常量与 DoS 注释 |
| `src/acp/translator.ts:66-91` | PendingPrompt/PendingToolCall 类型定义 |
| `src/acp/translator.ts:420-460` | AcpGatewayAgent 构造函数与速率限制 |
| `src/acp/translator.ts:449-459` | 会话创建速率限制配置 |
| `src/acp/translator.ts:466-491` | handleGatewayDisconnect：generation 计数器 |
| `src/acp/translator.ts:503-524` | initialize()：能力协商 |
| `src/acp/translator.ts:526-554` | newSession()：会话创建 |
| `src/acp/translator.ts:556-588` | loadSession()：历史回放 |
| `src/acp/translator.ts:674-778` | prompt()：请求发送与 pending 注册 |
| `src/acp/translator.ts:686-707` | DoS 双层检查 |
| `src/acp/translator.ts:735-763` | sendWithProvenanceFallback 降级逻辑 |
| `src/acp/translator.ts:780-811` | cancel()：abort 范围控制 |
| `src/acp/translator.ts:832-924` | handleAgentEvent()：工具流翻译 |
| `src/acp/translator.ts:927-972` | handleChatEvent()：最终状态处理 |
| `src/acp/translator.ts:974-1023` | handleDeltaEvent()：快照转增量 |
| `src/acp/translator.ts:1067-1074` | armDisconnectTimer()：宽限窗口 |
| `src/acp/translator.ts:1099-1208` | shouldRejectPendingAtDisconnectDeadline/reconcilePendingPrompts |
| `src/acp/translator.ts:1400-1407` | assertSupportedSessionSetup：拒绝 per-session MCP |
| `src/acp/translator.ts:403-418` | buildSystemProvenanceReceipt |
| `src/acp/approval-classifier.ts:12-22` | 工具 ID 集合定义 |
| `src/acp/approval-classifier.ts:24-38` | AcpApprovalClass 类型 |
| `src/acp/approval-classifier.ts:72-102` | resolveToolNameForPermission：三源交叉验证 |
| `src/acp/approval-classifier.ts:140-181` | CWD 范围路径解析 |
| `src/acp/approval-classifier.ts:183-227` | classifyAcpToolApproval：8 级分类 |
| `src/acp/session-mapper.ts:27-85` | resolveSessionKey：5 级优先级解析 |
| `src/acp/persistent-bindings.types.ts:60-78` | buildConfiguredAcpSessionKey：哈希确定性键 |
| `src/acp/event-mapper.ts:22-53` | 路径键与行号键常量 |
| `src/acp/event-mapper.ts:187-243` | collectToolLocations：深度优先遍历 |
| `src/acp/event-mapper.ts:245-276` | extractTextFromPrompt：逐块 DoS 检查 |
| `src/acp/translator.stop-reason.test.ts:143-350` | 断线重连全场景测试覆盖 |

### 外部资料（11 处）

1. [Agent Client Protocol GitHub](https://github.com/agentclientprotocol/agent-client-protocol)——ACP 官方仓库，含 schema 定义
2. [Zed — Agent Client Protocol](https://zed.dev/acp)——ACP 的设计动机和生态支持
3. [Intro to Agent Client Protocol (ACP) - Goose Docs](https://goose-docs.ai/blog/2025/10/24/intro-to-agent-client-protocol-acp/)——ACP 协议入门介绍
4. [AI 协议对比：MCP、A2A、AGP、IBM ACP、Zed ACP](https://4sysops.com/archives/comparing-ai-protocols-mcp-a2a-agp-agntcy-ibm-acp-zed-acp/)——各 Agent 协议横向对比
5. [MCP vs A2A - Auth0 Blog](https://auth0.com/blog/mcp-vs-a2a/)——MCP 与 A2A 的互补关系
6. [CWE-400: Uncontrolled Resource Consumption](https://cwe.mitre.org/data/definitions/400.html)——代码注释直接引用的漏洞类型
7. [OWASP AI Agent Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html)——AI Agent 安全实践
8. [MCP Permissions - Cerbos Blog](https://www.cerbos.dev/blog/mcp-permissions-securing-ai-agent-access-to-tools)——AI Agent 工具权限控制
9. [Preventing Tool Misuse in AI Agents](https://dev.to/willvelida/preventing-tool-misuse-in-ai-agents-4pcl)——工具调用安全分级实践
10. [Error handling in distributed systems - Temporal](https://temporal.io/blog/error-handling-in-distributed-systems)——分布式系统容错模式
11. [Resilience Design Patterns: Retry, Fallback, Timeout](https://www.codecentric.de/en/knowledge-hub/blog/resilience-design-patterns-retry-fallback-timeout-circuit-breaker)——断线重连的工程模式参考
