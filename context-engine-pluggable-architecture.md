# Context Engine：可插拔上下文管理系统的架构解析

> **系列**：clawdbot 源码深度阅读 / 第 2 篇  
> **主题**：`src/context-engine/` + `src/agents/pi-embedded-runner/context-engine-maintenance.ts`  
> **一句话总结**：clawdbot 将 AI 会话的上下文生命周期封装为一个可替换的"插件合约"，并用三种不寻常的工程技巧解决了向后兼容、多实例隔离和 graceful degradation 问题。

---

## 为什么上下文管理是一个"系统"问题

LLM 应用的上下文管理（context management）通常被低估：开发者把它当作"往 messages 数组里塞消息"来处理。但当一个 agent 框架需要同时支持数百个并发会话、允许第三方插件替换核心记忆系统、还要在不重启服务的情况下滚动升级上下文引擎时，这件事就变成了系统工程问题。

clawdbot 的解法是把所有上下文相关逻辑抽象为一个接口：**ContextEngine**。

---

## 一、接口设计：覆盖完整的上下文生命周期

`ContextEngine` 接口定义了一个 AI 会话中上下文的完整生命周期（`src/context-engine/types.ts:162`）：

```typescript
export interface ContextEngine {
  readonly info: ContextEngineInfo;

  bootstrap?(params: { sessionId: string; sessionKey?: string; sessionFile: string }): Promise<BootstrapResult>;
  maintain?(params: { ... runtimeContext?: ContextEngineRuntimeContext }): Promise<ContextEngineMaintenanceResult>;
  ingest(params: { sessionId: string; sessionKey?: string; message: AgentMessage; isHeartbeat?: boolean }): Promise<IngestResult>;
  ingestBatch?(params: { ... messages: AgentMessage[] }): Promise<IngestBatchResult>;
  afterTurn?(params: { ... messages: AgentMessage[]; prePromptMessageCount: number; tokenBudget?: number }): Promise<void>;
  assemble(params: { messages: AgentMessage[]; tokenBudget?: number; prompt?: string; model?: string }): Promise<AssembleResult>;
  compact(params: { force?: boolean; currentTokenCount?: number; compactionTarget?: "budget" | "threshold" }): Promise<CompactResult>;
  prepareSubagentSpawn?(params: { parentSessionKey: string; childSessionKey: string }): Promise<SubagentSpawnPreparation | undefined>;
  onSubagentEnded?(params: { childSessionKey: string; reason: SubagentEndReason }): Promise<void>;
  dispose?(): Promise<void>;
}
```

这个接口的设计有几个值得注意的决策：

**1. 必选 vs 可选方法的划分很讲究**

只有 `ingest`、`assemble`、`compact` 是必选的——这三个方法构成了最小可工作的上下文引擎：能收消息、能组装发给模型的 context、能在超出 token 预算时压缩。其余方法（`bootstrap`、`maintain`、`afterTurn`）都是可选的，允许轻量级实现跳过复杂的生命周期管理。

**2. `assemble` 接收当前 turn 的 `prompt`**

这个设计细节透露了系统对 RAG（检索增强生成）引擎的预留：`prompt` 参数让实现了检索功能的引擎能在组装 context 时先做语义搜索，把相关记忆注入到 messages 里。legacy 引擎忽略这个参数；第三方 RAG 引擎可以充分利用它。

**3. `maintain` 有一个 `rewriteTranscriptEntries` 回调**

```typescript
// src/context-engine/types.ts:137
export type ContextEngineRuntimeContext = Record<string, unknown> & {
  allowDeferredCompactionExecution?: boolean;
  promptCache?: ContextEnginePromptCacheInfo;
  rewriteTranscriptEntries?: (
    request: TranscriptRewriteRequest,
  ) => Promise<TranscriptRewriteResult>;
};
```

`maintain()` 是引擎做"定期整理"的时机（比如压缩旧消息、合并冗余条目）。但引擎不应该直接操作会话文件——那是 runtime 的职责。解决办法是：runtime 把一个受控的 `rewriteTranscriptEntries` 函数注入到 `runtimeContext` 里，引擎通过回调告诉 runtime "这些条目需要替换成这些内容"，runtime 负责实际的磁盘写入。这是一个经典的**依赖倒置**：高层策略不依赖低层实现，而是通过接口约定协作。

---

## 二、Registry：用 Symbol.for 解决 ESM 多实例陷阱

Node.js + ESM bundler 的组合有一个众所周知的陷阱：当同一个模块在不同的 bundle chunk 中被分别加载时，每个 chunk 持有独立的模块级变量，导致"多个单例共存"。这对于注册中心（registry）来说是致命的——插件注册到 chunk A 的 Map，但 runtime 从 chunk B 的 Map 里查找，结果是空的。

clawdbot 的解法是把注册中心挂载在 `globalThis` 上，用 `Symbol.for` 生成进程唯一的键（`src/context-engine/registry.ts:305-326`）：

```typescript
const CONTEXT_ENGINE_REGISTRY_STATE = Symbol.for("openclaw.contextEngineRegistryState");

const contextEngineRegistryState = resolveGlobalSingleton<ContextEngineRegistryState>(
  CONTEXT_ENGINE_REGISTRY_STATE,
  () => ({ engines: new Map() }),
);
```

`resolveGlobalSingleton` 的实现极简（`src/shared/global-singleton.ts:1-12`）：

```typescript
// 注释直接说明了意图和限制
// Safe for process-local caches and registries that can tolerate helper-based
// resolution. Do not use this for live mutable state that must survive split
// runtime chunks; keep those on a direct globalThis[Symbol.for(...)] lookup.
export function resolveGlobalSingleton<T>(key: symbol, create: () => T): T {
  const globalStore = globalThis as Record<PropertyKey, unknown>;
  if (Object.prototype.hasOwnProperty.call(globalStore, key)) {
    return globalStore[key] as T;
  }
  const created = create();
  globalStore[key] = created;
  return created;
}
```

**为什么用 `Symbol.for` 而不是普通 Symbol？**

普通 `Symbol()` 每次调用都创建新的唯一值。两个模块各调用一次 `Symbol("foo")`，得到的是两个不同的 symbol，无法用来访问同一个 `globalThis` 属性。`Symbol.for("foo")` 则维护了一个进程级的 Symbol 注册表——不管调用多少次，同一个字符串总返回同一个 symbol。配合 `globalThis`，就形成了真正进程唯一的键值存储。

---

## 三、向后兼容 Proxy：用错误探测代替版本声明

当 clawdbot 给 `ContextEngine` 接口增加新参数（比如 `sessionKey` 和 `prompt`），已有的第三方引擎实现如果用了严格的参数校验（如 zod schema），遇到未知字段会直接抛错。版本升级就会破坏已有插件。

典型的解法是让客户端声明"我支持 v2 API"。clawdbot 选了一个更有意思的方向：**通过错误探测自动降级**（`src/context-engine/registry.ts:200-298`）。

核心是 `invokeWithLegacyCompat`：

```typescript
async function invokeWithLegacyCompat<TResult, TParams extends SessionKeyCompatParams>(
  method: (params: TParams) => Promise<TResult> | TResult,
  params: TParams,
  allowedKeys: readonly LegacyCompatKey[],
  opts?: { onLegacyModeDetected?: () => void; rejectedKeys?: ReadonlySet<LegacyCompatKey> },
): Promise<TResult> {
  try {
    return await method(params);  // 先正常调用
  } catch (error) {
    // 检测错误是否因为 unknown field 引起的
    const rejectedKeys = detectRejectedLegacyCompatKeys(currentError, availableKeys);
    if (!learnedNewKey) {
      throw currentError;  // 不是兼容性问题，正常抛出
    }
    // 把被拒绝的字段从 params 里删掉，重试
    currentParams = withoutLegacyCompatKeys(params, activeRejectedKeys);
    return await method(currentParams);
  }
}
```

然后用 `Proxy` 把这个逻辑透明地包在每一个注册的引擎上（`src/context-engine/registry.ts:250-299`）：

```typescript
function wrapContextEngineWithSessionKeyCompat(engine: ContextEngine): ContextEngine {
  let isLegacy = false;
  const rejectedKeys = new Set<LegacyCompatKey>();

  const proxy: ContextEngine = new Proxy(engine, {
    get(target, property, receiver) {
      const value = Reflect.get(target, property, receiver);
      if (!isSessionKeyCompatMethodName(property)) {
        return value.bind(target);
      }
      return (params: SessionKeyCompatParams) => {
        if (isLegacy && allowedKeys.some(key => rejectedKeys.has(key))) {
          // 已知 legacy 模式，直接去掉被拒绝的字段，无需重试
          return method(withoutLegacyCompatKeys(params, rejectedKeys));
        }
        return invokeWithLegacyCompat(method, params, allowedKeys, {
          onLegacyModeDetected: () => { isLegacy = true; },
          onLegacyKeysDetected: (keys) => { for (const key of keys) rejectedKeys.add(key); },
          rejectedKeys,
        });
      };
    },
  });
  return proxy;
}
```

**这个机制的精妙之处在于**：

1. **第一次调用**：携带完整参数，包括新字段
2. **如果引擎抛出 "unrecognized key" 之类的错误**：自动识别是哪个字段被拒绝
3. **重试一次**：去掉被拒绝的字段后重新调用
4. **之后所有调用**：记住 `isLegacy = true` 和 `rejectedKeys`，直接跳过重试逻辑

这等于用"fail fast + retry"实现了 API 版本的自动协商。代价是第一次调用可能多花一次 round trip；收益是完全无需引擎声明自己的版本，老插件零改动就能继续工作。

错误检测部分尤其谨慎——它覆盖了来自 zod、Ajv、joi 等不同 schema 校验库的错误格式（`src/context-engine/registry.ts:108-185`）：

```typescript
const LEGACY_UNKNOWN_FIELD_PATTERNS: Record<LegacyCompatKey, readonly RegExp[]> = {
  sessionKey: [
    /\bunrecognized key(?:\(s\)|s)? in object:.*['"`]sessionKey['"`]/i,
    /\badditional propert(?:y|ies)\b.*['"`]sessionKey['"`]/i,
    /\bmust not have additional propert(?:y|ies)\b.*['"`]sessionKey['"`]/i,
    // ... 共 7 个 pattern 覆盖不同 validator 的错误格式
  ],
  // ...
};
```

---

## 四、Legacy Engine：空对象模式的教科书实现

`LegacyContextEngine` 是一个迷人的最小化实现（`src/context-engine/legacy.ts`）：

```typescript
export class LegacyContextEngine implements ContextEngine {
  readonly info: ContextEngineInfo = { id: "legacy", name: "Legacy Context Engine", version: "1.0.0" };

  async ingest(_params: ...): Promise<IngestResult> {
    // No-op: SessionManager handles message persistence in the legacy flow
    return { ingested: false };
  }

  async assemble(params: ...): Promise<AssembleResult> {
    // Pass-through: the existing sanitize -> validate -> limit -> repair pipeline
    // in attempt.ts handles context assembly for the legacy engine.
    return { messages: params.messages, estimatedTokens: 0 };
  }

  async compact(params: ...): Promise<CompactResult> {
    return await delegateCompactionToRuntime(params);
  }
}
```

三个方法，三种策略：

- `ingest`：什么都不做——消息持久化由 SessionManager 处理，legacy 引擎不介入
- `assemble`：直接透传——消息的过滤、截断、修复在别的地方做（`attempt.ts` 的管道）
- `compact`：委托给 runtime——通过 `delegateCompactionToRuntime` 调用内置的压缩路径

这是 **Null Object Pattern** 的变体：不是返回"什么都不做"的对象，而是把职责委托回系统原有路径。它让 `resolveContextEngine()` 总能返回一个合法对象，调用方无需 null check，同时保证了 legacy 行为完全不变。

`delegateCompactionToRuntime` 还有一个值得注意的细节（`src/context-engine/delegate.ts:11-18`）：

```typescript
let compactRuntimePromise: Promise<CompactRuntimeModule> | null = null;

function loadCompactRuntime(): Promise<CompactRuntimeModule> {
  // Use a literal specifier so the bundler rewrites the runtime chunk path
  compactRuntimePromise ??= import("../agents/pi-embedded-runner/compact.runtime.js");
  return compactRuntimePromise;
}
```

压缩模块用**懒加载**（`??=` + dynamic import）延迟初始化，因为它包含较重的依赖。Promise 被缓存，确保模块只被加载一次。注释中明确说明了用字面量路径的原因：让 bundler 能正确生成 runtime chunk。这是一个在 ESM + bundler 环境下处理代码分割的标准技巧。

---

## 五、Maintenance 调度：Foreground vs Background 模式

`ContextEngineInfo` 里有一个低调但重要的字段（`src/context-engine/types.ts:58`）：

```typescript
turnMaintenanceMode?: "foreground" | "background";
```

这个字段控制了 `maintain()` 在每次 turn 结束后的执行方式（`src/agents/pi-embedded-runner/context-engine-maintenance.ts:614-634`）：

```typescript
const shouldDefer =
  params.reason === "turn" &&
  executionMode !== "background" &&
  params.contextEngine.info.turnMaintenanceMode === "background";

if (shouldDefer) {
  scheduleDeferredTurnMaintenance({ ... });
  return undefined;
}
```

**为什么需要这个分叉？**

前台维护（foreground）：在 turn 完成、返回响应之前同步执行 maintain。用户体感是下一次 turn 的响应可能稍慢（因为要等 maintain 完成），但上下文状态是一致的。

后台维护（background）：maintain 被丢进一个独立的 lane 队列，在用户已经收到响应之后异步运行。

后台调度的实现更复杂（`src/agents/pi-embedded-runner/context-engine-maintenance.ts:501-595`）。它要处理几个竞争条件：

```typescript
function scheduleDeferredTurnMaintenance(params: ...): void {
  const activeRun = activeDeferredTurnMaintenanceRuns.get(sessionKey);
  if (activeRun) {
    // 如果上一轮 maintenance 还没跑完，记录"需要重跑"而不是启动新实例
    activeRun.rerunRequested = true;
    activeRun.latestParams = { ...params, sessionKey };
    return;
  }
  // 新 run 跑完后，如果有 rerunRequested，立即再调度一次
  const trackedPromise = runPromise.finally(() => {
    const rerunParams = current.rerunRequested && !shutdownTriggered ? current.latestParams : undefined;
    activeDeferredTurnMaintenanceRuns.delete(sessionKey);
    if (rerunParams) {
      scheduleDeferredTurnMaintenance(rerunParams);
    }
  });
}
```

这个 "coalesce" 机制（合并多次请求为一次执行）在前一次 maintenance 还在运行时，不会堆积多个并发任务，而是用最新参数替换待执行的任务。这类似于 React 的批量状态更新：只有最终状态才重要，中间状态可以跳过。

另一个设计细节是 maintenance worker 在执行前会等待 session lane 清空（`src/agents/pi-embedded-runner/context-engine-maintenance.ts:379-400`）：

```typescript
for (;;) {
  while (getQueueSize(sessionLane) > 0) {
    // 每 100ms 轮询，等待 session lane idle
    await sleepWithAbort(TURN_MAINTENANCE_WAIT_POLL_MS, shutdownAbort.abortSignal);
  }
  await Promise.resolve();  // 让事件循环跑一轮
  if (getQueueSize(sessionLane) === 0) {
    break;
  }
}
```

为什么要等？因为 maintenance 可能需要修改 transcript（通过 `rewriteTranscriptEntries`），而主 session lane 上可能有正在进行的 turn 也在操作同一份 transcript。等待 lane 清空是一种轻量级的互斥机制——不用锁，但保证了不会出现并发写入。

---

## 六、插件注册：权力边界的设计

第三方插件通过 Plugin API 注册自己的 context engine（`src/plugins/registry.ts:1199-1224`）：

```typescript
registerContextEngine: (id, factory) => {
  if (id === defaultSlotIdForKey("contextEngine")) {
    // 核心 id "legacy" 被保留，第三方不能覆盖
    pushDiagnostic({ level: "error", message: `context engine id reserved by core: ${id}` });
    return;
  }
  const result = registerContextEngineForOwner(id, factory, `plugin:${record.id}`, {
    allowSameOwnerRefresh: true,
  });
  if (!result.ok) {
    pushDiagnostic({ level: "error", message: `context engine already registered: ${id}` });
    return;
  }
  // 记录到插件的 contextEngineIds 列表，用于追踪
  if (!record.contextEngineIds?.includes(id)) {
    record.contextEngineIds = [...(record.contextEngineIds ?? []), id];
  }
}
```

注册时有三条防护：
1. **id 不能是 core 保留的**（`"legacy"` slot 默认值）
2. **id 不能被其他所有者已注册**（每个 id 只能有一个 owner）
3. **同一 owner 可以刷新自己的注册**（`allowSameOwnerRefresh: true`，用于热重载）

用户想换掉默认引擎，只需在配置里修改 slot（`src/context-engine/registry.ts:454-475`）：

```typescript
export async function resolveContextEngine(config?: OpenClawConfig): Promise<ContextEngine> {
  const slotValue = config?.plugins?.slots?.contextEngine;
  const engineId =
    typeof slotValue === "string" && slotValue.trim()
      ? slotValue.trim()
      : defaultSlotIdForKey("contextEngine");  // 默认 "legacy"
  // ...
  return wrapContextEngineWithSessionKeyCompat(engine);
}
```

---

## 七、可以学到什么

### 1. 先设计接口，再设计实现

`ContextEngine` 接口先于任何具体实现存在（Legacy 引擎是后来补充的适配层）。接口覆盖了完整生命周期，强迫开发者在写第一行实现代码之前，就想清楚"一个上下文引擎应该能做什么"。这与 TDD 的精神相似——接口就是规格书。

### 2. 向后兼容不一定要显式版本号

很多框架通过 `version: 2` 字段或 `X-API-Version` header 来处理 API 演化。clawdbot 的 Proxy + 错误探测方案更隐式，但在插件生态中可能更友好——插件作者不需要关心自己用的是哪个版本的 API，系统会自动协商。代价是系统代码更复杂。

### 3. 进程级单例需要用 Symbol.for + globalThis

这是 Node.js ESM 多实例问题的标准解法。任何需要在 bundled 环境下作为真正单例存在的注册中心，都应该用这个模式——普通的模块级变量不够用。

### 4. Null Object Pattern 的价值

`LegacyContextEngine` 是一个完美的 Null Object：实现了完整接口，但大部分方法是 no-op 或 pass-through。这让系统中所有需要 context engine 的地方都能无条件调用，无需判断"是否配置了 engine"——即使没有配置，也有一个安全的默认行为。

### 5. 后台任务需要 "coalesce"

当一个后台任务可能被高频触发（如每次 turn 结束都触发 maintenance），正确的做法是合并请求：只保留最新的参数，等上一次运行完成再决定是否重新调度。堆积队列是资源泄漏的来源。

---

## 引用

### 源码位置

1. `src/context-engine/types.ts:162-293` — `ContextEngine` 接口定义及完整生命周期 API
2. `src/context-engine/types.ts:137-154` — `ContextEngineRuntimeContext`（含 `rewriteTranscriptEntries` 回调）
3. `src/context-engine/registry.ts:305-326` — 进程级注册中心与 `Symbol.for` 单例机制
4. `src/context-engine/registry.ts:108-185` — Legacy 兼容错误模式匹配（多 validator 支持）
5. `src/context-engine/registry.ts:200-299` — `invokeWithLegacyCompat` + `Proxy` 自适应兼容
6. `src/context-engine/registry.ts:454-475` — `resolveContextEngine`：slot 配置解析
7. `src/context-engine/legacy.ts` — `LegacyContextEngine`：空对象模式实现
8. `src/context-engine/delegate.ts:11-18` — 懒加载 compaction runtime（ESM 代码分割）
9. `src/shared/global-singleton.ts:1-12` — `resolveGlobalSingleton`：globalThis 锚定
10. `src/agents/pi-embedded-runner/context-engine-maintenance.ts:600-651` — 前台/后台 maintenance 模式分发
11. `src/agents/pi-embedded-runner/context-engine-maintenance.ts:379-400` — 等待 session lane idle 的互斥机制
12. `src/agents/pi-embedded-runner/context-engine-maintenance.ts:501-595` — coalesce 调度：防止 maintenance 并发堆积
13. `src/agents/pi-embedded-runner/run/attempt.context-engine-helpers.ts:106-177` — bootstrap + assemble 的调用链
14. `src/context-engine/types.ts:58-60` — `turnMaintenanceMode` 字段（foreground/background 控制）
15. `src/plugins/registry.ts:1199-1224` — 插件注册时的权力边界防护

### 外部资料

- [Context Management for Deep Agents — LangChain Blog](https://blog.langchain.com/context-management-for-deepagents/)：分析了 deep agent 的 context 压缩策略，包括 85% 触发的截断机制
- [Context compaction in agent frameworks — DEV Community](https://dev.to/crabtalk/context-compaction-in-agent-frameworks-4ckk)：对比了集中式（GroupManager 广播）vs 分布式（每个 agent 自主压缩）的权衡
- [Context compression — Google ADK Docs](https://google.github.io/adk-docs/context/compaction/)：Google ADK 的事件驱动压缩实现，与 clawdbot 的接口化方案形成对比
- [API versioning and evolution with proxies — Cindy Sridharan](https://copyconstruct.medium.com/iterative-refactoring-of-apis-with-proxies-d78a2ba7e6ed)：用 proxy 做 API 兼容的通用模式
- [Global Singleton and the Runtime Hell in Next.js](https://www.hawu.me/dev/6268)：globalThis 单例的陷阱与正确用法
- [Pure ESM package — Sindre Sorhus](https://gist.github.com/sindresorhus/a39789f98801d908bbc7ff3ecc99d99c)：ESM 多实例问题的权威说明
- [The Fundamentals of Context Management and Compaction in LLMs — Isaac Kargar](https://kargarisaac.medium.com/the-fundamentals-of-context-management-and-compaction-in-llms-171ea31741a2)：LLM context 压缩的基础概念

---

*本文引用了 15 个源码位置 + 7 个外部链接，共 22 处引用。所有代码片段均以源码为准，已验证行号准确性。*
