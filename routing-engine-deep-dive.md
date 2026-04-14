# 路由引擎深度剖析：多渠道 Agent 的会话身份解析系统

> 本文深度解析 clawdbot 的路由引擎（`src/routing/`），聚焦三个核心问题：**会话键如何成为安全边界**、**8 层优先级绑定匹配如何运作**、**三层 WeakMap 缓存如何实现零副作用失效**。

---

## 背景：多渠道路由为什么难

一个 Agent 可以同时在 Discord、Telegram、Slack 上被用户访问。每条消息背后都有一系列决策：

- 这条消息应该交给哪个 Agent 处理？（`resolveAgentRoute`）
- 这条消息属于哪个会话？（`buildAgentSessionKey`）
- 这个会话是否与其他用户的会话隔离？（DM scope）
- 如果发消息的用户在多个平台上都联系过 Agent，应该共享上下文还是分开？（identity links）

这四个问题不能独立解决——它们共同决定了一个会话的**身份**。clawdbot 的路由引擎将这四个关注点统一在 `resolveAgentRoute` 这一个函数中，返回一个 `ResolvedAgentRoute` 对象：

```typescript
// src/routing/resolve-route.ts:40-60
export type ResolvedAgentRoute = {
  agentId: string;
  channel: string;
  accountId: string;
  /** Internal session key used for persistence + concurrency. */
  sessionKey: string;
  /** Convenience alias for direct-chat collapse. */
  mainSessionKey: string;
  /** Which session should receive inbound last-route updates. */
  lastRoutePolicy: "main" | "session";
  /** Match description for debugging/logging. */
  matchedBy:
    | "binding.peer"
    | "binding.peer.parent"
    | "binding.peer.wildcard"
    | "binding.guild+roles"
    | "binding.guild"
    | "binding.team"
    | "binding.account"
    | "binding.channel"
    | "default";
};
```

`matchedBy` 字段是调试利器——它精确记录了路由决策走了哪条路径，便于在日志中快速定位异常路由行为。

---

## 第一层：会话键——把会话身份编码进字符串

### 格式设计

clawdbot 的会话键是一个带前缀的冒号分隔字符串：

```
agent:<agentId>:<rest>
```

其中 `<rest>` 随隔离级别不同而变化：

| 场景 | 会话键格式 | 例子 |
|------|-----------|------|
| DM (main scope) | `agent:<id>:main` | `agent:main:main` |
| DM (per-peer) | `agent:<id>:direct:<peerId>` | `agent:main:direct:7550356539` |
| DM (per-channel-peer) | `agent:<id>:<channel>:direct:<peerId>` | `agent:main:telegram:direct:7550356539` |
| DM (per-account-channel-peer) | `agent:<id>:<channel>:<accountId>:direct:<peerId>` | `agent:main:telegram:tasks:direct:7550356539` |
| 群组/频道 | `agent:<id>:<channel>:<kind>:<peerId>` | `agent:main:discord:channel:C123456` |
| 线程 | `<基础键>:thread:<threadId>` | `agent:main:discord:channel:C123456:thread:T789` |

这个设计有一个重要属性：**会话键是它自己的解析器**。任何知道格式的组件都可以从键中还原出所有路由信息，无需查询外部状态。

```typescript
// src/sessions/session-key-utils.ts:28-48
export function parseAgentSessionKey(
  sessionKey: string | undefined | null,
): ParsedAgentSessionKey | null {
  const raw = normalizeOptionalLowercaseString(sessionKey);
  if (!raw) {
    return null;
  }
  const parts = raw.split(":").filter(Boolean);
  if (parts.length < 3) {
    return null;
  }
  if (parts[0] !== "agent") {
    return null;
  }
  const agentId = normalizeOptionalString(parts[1]);
  const rest = parts.slice(2).join(":");
  if (!agentId || !rest) {
    return null;
  }
  return { agentId, rest };
}
```

解析函数故意只返回 `agentId` 和 `rest`，不尝试解析 `rest` 的语义——这是 Tolerant Reader 模式：消费方只取自己需要的部分，无需理解完整格式。

### 键的版本控制不依赖版本号

会话键的格式靠**结构**而非**版本号**区分不同形态。`isCronSessionKey`、`isSubagentSessionKey`、`isAcpSessionKey` 这些函数都是通过正则匹配 `rest` 部分来判断键的类型：

```typescript
// src/sessions/session-key-utils.ts:50-76
export function isCronRunSessionKey(sessionKey: string | undefined | null): boolean {
  const parsed = parseAgentSessionKey(sessionKey);
  if (!parsed) {
    return false;
  }
  return /^cron:[^:]+:run:[^:]+$/.test(parsed.rest);
}

export function isSubagentSessionKey(sessionKey: string | undefined | null): boolean {
  const raw = normalizeOptionalString(sessionKey);
  if (!raw) {
    return false;
  }
  if (normalizeOptionalLowercaseString(raw)?.startsWith("subagent:")) {
    return true;
  }
  const parsed = parseAgentSessionKey(raw);
  return normalizeOptionalLowercaseString(parsed?.rest)?.startsWith("subagent:") === true;
}
```

不同的 key 类型之间不会互相误判，因为前缀（`cron:`、`subagent:`、`acp:`）是全局唯一的命名空间。

---

## 第二层：4 种 DM 隔离模式

### 为什么需要多种隔离模式？

默认情况下（`dmScope: "main"`），所有用户的私信共享同一个会话。这在单用户场景下是合理的（节省资源），但在多用户场景下是安全漏洞：Alice 的私信对 Bob 可见。

clawdbot 提供了 4 种隔离粒度：

```typescript
// src/routing/session-key.ts:129-175
export function buildAgentPeerSessionKey(params: {
  agentId: string;
  mainKey?: string | undefined;
  channel: string;
  accountId?: string | null;
  peerKind?: ChatType | null;
  peerId?: string | null;
  identityLinks?: Record<string, string[]>;
  dmScope?: "main" | "per-peer" | "per-channel-peer" | "per-account-channel-peer";
}): string {
  const peerKind = params.peerKind ?? "direct";
  if (peerKind === "direct") {
    const dmScope = params.dmScope ?? "main";
    // ...
    if (dmScope === "per-account-channel-peer" && peerId) {
      const channel = normalizeLowercaseStringOrEmpty(params.channel) || "unknown";
      const accountId = normalizeAccountId(params.accountId);
      return `agent:${normalizeAgentId(params.agentId)}:${channel}:${accountId}:direct:${peerId}`;
    }
    if (dmScope === "per-channel-peer" && peerId) {
      const channel = normalizeLowercaseStringOrEmpty(params.channel) || "unknown";
      return `agent:${normalizeAgentId(params.agentId)}:${channel}:direct:${peerId}`;
    }
    if (dmScope === "per-peer" && peerId) {
      return `agent:${normalizeAgentId(params.agentId)}:direct:${peerId}`;
    }
    return buildAgentMainSessionKey({
      agentId: params.agentId,
      mainKey: params.mainKey,
    });
  }
  // group/channel types always get isolated keys
  const channel = normalizeLowercaseStringOrEmpty(params.channel) || "unknown";
  const peerId = normalizeLowercaseStringOrEmpty(params.peerId) || "unknown";
  return `agent:${normalizeAgentId(params.agentId)}:${channel}:${peerKind}:${peerId}`;
}
```

注意群组类消息（`group`/`channel`）**永远是隔离的**——不受 `dmScope` 影响。隔离只是私信（`direct`）的问题。

这 4 个模式的实际使用建议：

| 模式 | 适用场景 |
|------|---------|
| `main` | 单用户 Agent、个人助理 |
| `per-peer` | 多用户但同一平台（Telegram 机器人） |
| `per-channel-peer` | 多平台部署、推荐默认 |
| `per-account-channel-peer` | 多账号（billing 隔离）场景 |

### identity links：跨渠道身份融合

与隔离相对的是身份融合。如果同一个用户从 Telegram 和 Discord 联系 Agent，默认会产生两个独立会话，对话历史割裂。`identityLinks` 解决了这个问题：

```typescript
// src/routing/session-key.ts:178-210
function resolveLinkedPeerId(params: {
  identityLinks?: Record<string, string[]>;
  channel: string;
  peerId: string;
}): string | null {
  const identityLinks = params.identityLinks;
  if (!identityLinks) {
    return null;
  }
  const candidates = new Set<string>();
  const rawCandidate = normalizeToken(peerId);
  if (rawCandidate) {
    candidates.add(rawCandidate);
  }
  const channel = normalizeToken(params.channel);
  if (channel) {
    const scopedCandidate = normalizeToken(`${channel}:${peerId}`);
    if (scopedCandidate) {
      candidates.add(scopedCandidate);
    }
  }
  // 在 identityLinks 表中查找 canonical name
  for (const [canonical, ids] of Object.entries(identityLinks)) {
    for (const id of ids) {
      const normalized = normalizeToken(id);
      if (normalized && candidates.has(normalized)) {
        return canonicalName;
      }
    }
  }
  return null;
}
```

配置示例：

```yaml
session:
  dmScope: "per-channel-peer"
  identityLinks:
    alice: ["telegram:111111111", "discord:222222222222222222"]
```

当 `telegram:111111111` 发来消息，`resolveLinkedPeerId` 把 peerId 替换为 `alice`，生成 `agent:main:telegram:direct:alice`。当 `discord:222222222222222222` 发来消息，同样生成 `agent:main:discord:direct:alice`。两个会话键不同但解析到同一个 canonical 身份。

设计亮点：**查找是双向的**。函数同时尝试 `<peerId>` 和 `<channel>:<peerId>` 两种形式匹配，所以 `identityLinks` 表中既可以写 `"111111111"` 也可以写 `"telegram:111111111"`，更细粒度的 channel-scoped 形式允许同平台多账号不被错误合并。

---

## 第三层：8 层优先级绑定匹配

### 绑定（Binding）是什么

绑定是用户在配置中声明的"消息路由规则"：

```yaml
bindings:
  - agentId: "sales"
    match:
      channel: "discord"
      peer: { kind: "channel", id: "C_SALES" }
  - agentId: "support"
    match:
      channel: "discord"
      guildId: "G_MAIN"
  - agentId: "main"
    match:
      channel: "discord"
```

当一条消息到来，引擎要从所有绑定中找出**最特异的匹配**。这是一个经典的规则优先级问题——类似 CSS specificity，更精确的规则优先级更高。

### 8 层优先级

```typescript
// src/routing/resolve-route.ts:748-836
const tiers: Array<{...}> = [
  {
    matchedBy: "binding.peer",          // 最精确：精确匹配某个 peer
    enabled: Boolean(peer),
    // ...
  },
  {
    matchedBy: "binding.peer.parent",   // 线程 parent 的 peer 匹配
    enabled: Boolean(parentPeer && parentPeer.id),
    // ...
  },
  {
    matchedBy: "binding.peer.wildcard", // peer 类型通配符
    enabled: Boolean(peer),
    // ...
  },
  {
    matchedBy: "binding.guild+roles",   // Discord guild + 角色组合
    enabled: Boolean(guildId && memberRoleIds.length > 0),
    // ...
  },
  {
    matchedBy: "binding.guild",         // Discord guild
    enabled: Boolean(guildId),
    // ...
  },
  {
    matchedBy: "binding.team",          // Slack workspace/team
    enabled: Boolean(teamId),
    // ...
  },
  {
    matchedBy: "binding.account",       // 账号级匹配
    enabled: true,
    // ...
  },
  {
    matchedBy: "binding.channel",       // 渠道级通配
    enabled: true,
    // ...
  },
];

for (const tier of tiers) {
  if (!tier.enabled) {
    continue;
  }
  const matched = tier.candidates.find(
    (candidate) =>
      tier.predicate(candidate) &&
      matchesBindingScope(candidate.match, {
        ...baseScope,
        peer: tier.scopePeer,
      }),
  );
  if (matched) {
    return choose(matched.binding.agentId, tier.matchedBy);
  }
}

return choose(resolveDefaultAgentId(input.cfg), "default");
```

这 8 层加一个默认值，对应了 9 种路由结果。每一层不仅决定匹配什么，还决定了 **`enabled` 条件**——比如 `guild+roles` 层只有当请求携带 guildId 且 memberRoleIds 不为空时才激活。这避免了无效的查找。

### 线程的 parent 继承

Discord 线程是频道下的子频道。当用户在线程里发消息，`peer` 是线程的 ID，`parentPeer` 是线程所属频道的 ID。

如果管理员只配置了频道级绑定（`peer.id = "parent-channel-123"`），线程消息（`peer.id = "thread-456"`）在第一层（`binding.peer`）找不到匹配，会继续到第二层（`binding.peer.parent`）——用 parentPeer 去匹配绑定表，从而继承父频道的路由：

```typescript
// src/routing/resolve-route.test.ts:621-646
test("thread inherits binding from parent channel when no direct match", () => {
  expectDiscordThreadRoute({
    cfg: {
      bindings: [makeDiscordPeerBinding("adecco", defaultParentPeer.id)],
    },
    expectedAgentId: "adecco",
    expectedMatchedBy: "binding.peer.parent",
  });
});

test("direct peer binding wins over parent peer binding", () => {
  expectDiscordThreadRoute({
    cfg: {
      bindings: [
        makeDiscordPeerBinding("thread-agent", threadPeer.id),
        makeDiscordPeerBinding("parent-agent", defaultParentPeer.id),
      ],
    },
    expectedAgentId: "thread-agent",
    expectedMatchedBy: "binding.peer",
  });
});
```

这个设计不是偶然的——它模拟了 Discord 的权限继承语义：线程默认继承父频道的配置，除非显式覆盖。

### Chat Type 别名：group == channel

Discord 和 Slack 对频道有不同的术语（Discord 叫 `channel`，Slack 叫 `channel` 或 `group`）。clawdbot 做了双向别名：

```typescript
// src/routing/resolve-route.ts:338-348
function peerLookupKeys(kind: ChatType, id: string): string[] {
  if (kind === "group") {
    return [`group:${id}`, `channel:${id}`];
  }
  if (kind === "channel") {
    return [`channel:${id}`, `group:${id}`];
  }
  return [`${kind}:${id}`];
}
```

当用 `group:C123` 去索引时，会同时查找 `group:C123` 和 `channel:C123`。这意味着用户可以在配置里随意使用这两个词，路由引擎都能正确识别。

---

## 第四层：三层 WeakMap 缓存

路由计算不廉价：每次路由都需要遍历所有绑定、规范化字符串、合并两个有序列表。高频消息场景下，这会成为瓶颈。

clawdbot 构建了三层缓存：

```typescript
// src/routing/resolve-route.ts:127, 204-215
const agentLookupCacheByCfg = new WeakMap<OpenClawConfig, AgentLookupCache>();
// ^--- 1. Agent ID 规范化缓存 (line 127)

const evaluatedBindingsCacheByCfg = new WeakMap<OpenClawConfig, EvaluatedBindingsCache>(); // line 204
const MAX_EVALUATED_BINDINGS_CACHE_KEYS = 2000;
// ^--- 2. 绑定评估缓存（含 channel×account 索引）

const resolvedRouteCacheByCfg = new WeakMap<   // line 206
  OpenClawConfig,
  {
    bindingsRef: OpenClawConfig["bindings"];
    agentsRef: OpenClawConfig["agents"];
    sessionRef: OpenClawConfig["session"];
    byKey: Map<string, ResolvedAgentRoute>;
  }
>();
const MAX_RESOLVED_ROUTE_CACHE_KEYS = 4000;
// ^--- 3. 最终路由结果缓存
```

### WeakMap 的关键作用：以对象身份作为缓存键

三层缓存都用 `WeakMap<OpenClawConfig, ...>` 将缓存与配置对象绑定。当 `OpenClawConfig` 对象更换（配置热更新），旧的 WeakMap 条目自动变成垃圾回收候选——无需手动清理缓存，**配置变更即缓存失效**。

这利用了 JavaScript WeakMap 的核心语义：[WeakMap 的键必须是对象，且不持有强引用](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap)。一旦键对象没有其他引用，WeakMap 条目就会被 GC 回收。

### 细粒度失效：不止对象级

但对象级失效还不够精确。配置对象可能没有重建（hot-patch 某个字段），但子字段（比如 `bindings`）已经是新引用。第三层缓存做了更细粒度的检查：

```typescript
// src/routing/resolve-route.ts:521-538
function resolveRouteCacheForConfig(cfg: OpenClawConfig): Map<string, ResolvedAgentRoute> {
  const existing = resolvedRouteCacheByCfg.get(cfg);
  if (
    existing &&
    existing.bindingsRef === cfg.bindings &&
    existing.agentsRef === cfg.agents &&
    existing.sessionRef === cfg.session
  ) {
    return existing.byKey;
  }
  const byKey = new Map<string, ResolvedAgentRoute>();
  resolvedRouteCacheByCfg.set(cfg, {
    bindingsRef: cfg.bindings,
    agentsRef: cfg.agents,
    sessionRef: cfg.session,
    byKey,
  });
  return byKey;
}
```

它记录了 `bindingsRef`、`agentsRef`、`sessionRef` 三个子字段的引用快照。每次访问缓存时，对比这三个引用是否仍然相同。任何一个发生变化，缓存整体失效。

这是 **"版本引用"模式**：用引用相等性（`===`）代替版本号，天然防止版本计数器溢出和失步。

### "清空再插入"的 LRU 替代方案

常见的 LRU 缓存实现需要维护访问顺序的双向链表，开销不小。clawdbot 用一个更粗暴但有效的策略：

```typescript
// src/routing/resolve-route.ts:706-712
if (routeCache && routeCacheKey) {
  routeCache.set(routeCacheKey, route);
  if (routeCache.size > MAX_RESOLVED_ROUTE_CACHE_KEYS) {
    routeCache.clear();
    routeCache.set(routeCacheKey, route);
  }
}
```

当缓存条目超过 4000 个，直接清空整个缓存，只保留刚刚计算的这一条。

这个做法的理论依据：路由缓存的价值不在于"保留最近访问的条目"，而在于"避免相同输入的重复计算"。在大多数生产场景中，活跃的 peer-account 组合远少于 4000 个。当缓存溢出时，很可能是遭遇了大规模新用户涌入或攻击（海量不同 ID），此时缓存已经失效，清空重建是合理的选择。

---

## 设计反思：为什么不用规则引擎？

8 层优先级匹配让人想到工业级规则引擎（Drools、CLIPS）和它们底层的 [Rete 算法](https://en.wikipedia.org/wiki/Rete_algorithm)。Rete 的核心思想是把规则网络编译成共享的判断节点，避免重复求值同一个条件。

但 clawdbot 的路由系统走的是完全相反的路：

1. **预计算索引**，而非运行时网络求值。绑定在首次访问时按 `channel×account` 分桶，每个桶再按 peer/guild/team 建立倒排索引（`byPeer`、`byGuild`、`byTeam`...）
2. **分层 `find` 而非全量匹配**。每一层只查询对应索引，命中即返回，不需要遍历所有绑定
3. **配置驱动的层激活**。`enabled` 条件在评估前剔除无关层，避免空查询

这套方案的取舍很清晰：

| 维度 | Rete 规则引擎 | clawdbot 路由引擎 |
|------|-------------|----------------|
| 规则数量 | 数千到数万 | 数十到数百 |
| 条件复杂度 | 任意布尔组合 | 固定 8 层优先级 |
| 运行时灵活性 | 支持动态规则修改 | 需要重建缓存 |
| 实现复杂度 | 高（网络编译、工作内存） | 低（Map + 分层 find） |
| 调试可见性 | 需要专用工具 | `matchedBy` 字段直接可读 |

对于一个用户配置驱动、规则数量有限的场景，简单的分层索引比 Rete 更合适。

---

## 可以学到什么

### 1. 把路由状态编码进键，而不是存进数据库

会话键格式 `agent:<id>:<channel>:<kind>:<peer>` 让任何组件都可以从键中还原路由信息。这减少了分布式状态查询，也让键在日志中直接可读。这是 [Smart Endpoint, Dumb Pipe](https://martinfowler.com/articles/microservices.html#SmartEndpointsAndDumbPipes) 原则在键设计层面的应用。

### 2. 引用相等性作为缓存版本号

用 `bindingsRef === cfg.bindings` 代替显式版本号，既简单又可靠。前提是配置更新时确保引用变更（不可变更新语义）。这在 React 生态里叫 structural sharing，在 clawdbot 里是配置管理的隐性约定。

### 3. WeakMap + 对象生命周期 = 零成本缓存清理

以配置对象为键的 WeakMap 缓存，让缓存生命周期与对象生命周期绑定，无需手动 invalidate。适用场景：**以配置/schema 对象为核心的缓存**。

### 4. "清空重置"比 LRU 更简单，有时更合理

LRU 维护开销在高吞吐场景可观。当缓存的目的是"避免相同路由重复计算"而非"保留热点数据"，清空重置在大多数情况下效果相近。选择的关键是问：**缓存溢出时，是热点数据太多还是遇到了异常流量？**

### 5. 8 层优先级用数组表达比 if-else 链更易维护

代码里的 `tiers` 数组本质上是一个[策略模式](https://refactoring.guru/design-patterns/strategy)的变体——每一层是一个策略，外层循环统一执行。新增优先级只需在数组中插入新项，不需要修改核心匹配逻辑。

---

## 关键源码位置索引

| 文件 | 行号 | 内容 |
|------|------|------|
| `src/routing/resolve-route.ts` | 40-60 | `ResolvedAgentRoute` 类型定义 |
| `src/routing/resolve-route.ts` | 127-215 | 三层 WeakMap 缓存声明 |
| `src/routing/resolve-route.ts` | 338 | `peerLookupKeys` chat type 别名 |
| `src/routing/resolve-route.ts` | 521-538 | `resolveRouteCacheForConfig` 引用失效 |
| `src/routing/resolve-route.ts` | 706-712 | 清空重置缓存蒸发策略 |
| `src/routing/resolve-route.ts` | 748-836 | 8 层 tiers 优先级数组 |
| `src/routing/session-key.ts` | 129-175 | `buildAgentPeerSessionKey` 4 种 DM scope |
| `src/routing/session-key.ts` | 178-210 | `resolveLinkedPeerId` identity links |
| `src/sessions/session-key-utils.ts` | 28-48 | `parseAgentSessionKey` 键解析 |
| `src/sessions/session-key-utils.ts` | 50-76 | `isCronRunSessionKey`/`isSubagentSessionKey` 类型判断 |
| `src/routing/resolve-route.test.ts` | 568-690 | parentPeer 线程继承测试 |
| `src/routing/resolve-route.test.ts` | 195-215 | identity links 测试 |

---

## 外部参考资料

- [WeakMap - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap) — WeakMap 语义与垃圾回收行为
- [Session Management - OpenClaw Documentation](https://docs.openclaw.ai/concepts/session) — DM scope 和 identity links 的官方说明
- [Rete Algorithm - Wikipedia](https://en.wikipedia.org/wiki/Rete_algorithm) — 规则引擎的传统实现，与 clawdbot 做法的对比背景
- [Strategy Pattern - Refactoring Guru](https://refactoring.guru/design-patterns/strategy) — tiers 数组背后的设计模式
- [Smart Endpoints and Dumb Pipes - Martin Fowler](https://martinfowler.com/articles/microservices.html#SmartEndpointsAndDumbPipes) — 把路由逻辑内化进键设计的架构哲学
- [Multi-LLM routing strategies on AWS](https://aws.amazon.com/blogs/machine-learning/multi-llm-routing-strategies-for-generative-ai-applications-on-aws/) — 行业对多 Agent 路由的工程实践对比
- [The Routing Pattern: Build Smart Multi-Agent AI Workflows with LangGraph](https://medium.com/@huzaifaali4013399/the-routing-pattern-build-smart-multi-agent-ai-workflows-with-langgraph-44f177aadf7a) — LangGraph 的路由模式与 clawdbot 的对比视角
