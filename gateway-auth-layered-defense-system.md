# Gateway 认证体系：六种身份模式与分层防御设计

> **源码路径**：`src/gateway/auth.ts`（640 行）、`src/gateway/auth-rate-limit.ts`（236 行）、`src/gateway/credentials.ts`（336 行）、`src/gateway/auth-token-resolution.ts`（85 行）、`src/gateway/rate-limit-attempt-serialization.ts`（38 行）、`src/security/secret-equal.ts`（10 行）、`src/gateway/server-shared-auth-generation.ts`（94 行）、`src/gateway/device-auth.ts`（54 行）

---

## 为什么一个密码不够用

大多数系统的认证方案是：有一个密码，对就放行，不对就拒绝。这在简单场景下运作良好，但一个面向多种接入场景的 AI Agent 网关需要回答的问题要复杂得多：

- 本地 CLI 工具访问：需要认证吗？还是本机就可以信任？
- 从 Tailscale VPN 接入的私有设备：用密码还是用身份？
- 部署在 Kubernetes 中，前面有 Nginx 反向代理：谁来做认证？
- 移动 App 设备绑定（pairing）之后的访问：密码变了怎么办？
- 浏览器 WebSocket 连接的控制界面：需要和 HTTP API 一样的认证吗？

Clawdbot 的 Gateway 认证体系不是"一个密码"，而是六种身份模式、四级凭证解析链、三个独立安全机制的组合——而且每一个设计决策背后都有可以清楚说明的工程理由。

---

## 1. 六种认证模式的架构层级

```typescript
// src/gateway/auth.ts:29–67
export type GatewayAuthResult = {
  ok: boolean;
  method?:
    | "none"
    | "token"
    | "password"
    | "tailscale"
    | "device-token"
    | "bootstrap-token"
    | "trusted-proxy";
  user?: string;
  reason?: string;
  rateLimited?: boolean;
  retryAfterMs?: number;
};
```

这六种模式不是并列的，而是按**信任来源**分为三个层级：

### 层级一：网络位置信任（not really auth）
**`none` 模式**：网关直接放行所有请求，不做认证。设计用途：内部部署在完全隔离的网络（如 Kubernetes 内部服务网格），外围安全由网络边界保证，网关层不重复验证。

### 层级二：共享秘钥认证
**`token` 模式**和**`password` 模式**：经典的共享秘钥认证。两者的差异在于 HTTP 协议语义——token 通过 Bearer 头传递，password 通过 HTTP Basic Auth 或 WebSocket 连接参数传递。

```typescript
// src/gateway/auth.ts:231–304
export function resolveGatewayAuth(params: { ... }): ResolvedGatewayAuth {
  // ...
  let mode: ResolvedGatewayAuth["mode"];
  if (authOverride?.mode !== undefined) {
    mode = authOverride.mode;
    modeSource = "override";
  } else if (authConfig.mode) {
    mode = authConfig.mode;
    modeSource = "config";
  } else if (password) {
    mode = "password";
    modeSource = "password";
  } else if (token) {
    mode = "token";
    modeSource = "token";
  } else {
    mode = "token";
    modeSource = "default";
  }
```

注意 `modeSource` 字段——它记录了模式是从哪里解析出来的（override/config/password/token/default）。这不只是日志用途，它是可观测性的关键：当配置出问题时，`modeSource` 让你立即知道是哪层配置生效了。

### 层级三：第三方身份信任
**`tailscale` 模式**、**`trusted-proxy` 模式**：把身份断言的责任外包给可信的第三方系统——前者是 Tailscale VPN 网络，后者是前端反向代理。这两种模式的共同点是：网关本身不验证密码，而是验证第三方系统提供的身份声明是否可信。

**`device-token` 和 `bootstrap-token`**：设备绑定工作流的产物，用于 Android 节点/移动端设备与 Gateway 完成配对（pairing）后的长期访问。

---

## 2. 凭证解析优先链

在共享秘钥模式下，凭证从哪里来？这比看起来复杂。

```typescript
// src/gateway/auth-token-resolution.ts:12–85
export async function resolveGatewayAuthToken(params: {
  cfg: OpenClawConfig;
  env: NodeJS.ProcessEnv;
  explicitToken?: string;
  envFallback?: GatewayAuthTokenEnvFallback;  // "never" | "no-secret-ref" | "always"
}): Promise<{
  token?: string;
  source?: GatewayAuthTokenResolutionSource;  // "explicit" | "config" | "secretRef" | "env"
  secretRefConfigured: boolean;
  unresolvedRefReason?: string;
}>
```

解析顺序：
1. **`explicit`**：调用方直接传入（CLI 的 `--token` 参数）
2. **`config`**：配置文件的 `gateway.auth.token` 字段（明文字符串）
3. **`secretRef`**：配置文件中的秘密引用（`{ ref: "vault://...", provider: "hashicorp-vault" }`）
4. **`env`**：环境变量 `OPENCLAW_GATEWAY_TOKEN`

返回值中的 `source` 字段把"用了哪个凭证"变成了可追踪的字符串——这是设计 multi-source 配置系统时一个容易被忽略的细节：**当系统从多个来源组合配置时，需要记录每个字段实际生效的来源**，否则当凭证来自意料之外的地方时，调试会变得极其痛苦。

秘密引用（secretRef）的处理还有一个有趣的设计：

```typescript
// src/gateway/credentials.ts:30–47
export class GatewaySecretRefUnavailableError extends Error {
  readonly code = GATEWAY_SECRET_REF_UNAVAILABLE_ERROR_CODE;
  readonly path: string;

  constructor(path: string) {
    super([
      `${path} is configured as a secret reference but is unavailable in this command path.`,
      "Fix: set OPENCLAW_GATEWAY_TOKEN/OPENCLAW_GATEWAY_PASSWORD, pass explicit --token/--password,",
      "or run a gateway command path that resolves secret references before credential selection.",
    ].join("\n"));
  }
}
```

当 secretRef 无法解析（比如密钥管理服务不可用）时，代码不是悄悄降级到空值，而是抛出一个带有明确修复建议的结构化错误。**当安全配置无法解析时，拒绝启动并明确说明原因，远比悄悄降级到不安全状态要正确。**

---

## 3. Tailscale WHOIS 双重验证：为什么标头不够

Tailscale 认证模式看起来很简单：客户端通过 Tailscale VPN 连接，请求头里带着 `tailscale-user-login`。但 Clawdbot 不只检查这个头，还会做 WHOIS 二次验证：

```typescript
// src/gateway/auth.ts:198–229
async function resolveVerifiedTailscaleUser(params: {
  req?: IncomingMessage;
  tailscaleWhois: TailscaleWhoisLookup;
}): Promise<{ ok: true; user: TailscaleUser } | { ok: false; reason: string }> {
  const tailscaleUser = getTailscaleUser(req);  // 从 header 获取 login
  if (!isTailscaleProxyRequest(req)) {
    return { ok: false, reason: "tailscale_proxy_missing" };
  }
  const clientIp = resolveTailscaleClientIp(req);
  const whois = await tailscaleWhois(clientIp);  // 向 Tailscale daemon 查询实际 IP 的用户
  if (normalizeLogin(whois.login) !== normalizeLogin(tailscaleUser.login)) {
    return { ok: false, reason: "tailscale_user_mismatch" };
  }
  return { ok: true, user: { login: whois.login, ... } };
}
```

验证逻辑分三步：
1. **`isTailscaleProxyRequest`**：确认请求来自 loopback（127.0.0.1/::1）并且带有 Tailscale 代理头（x-forwarded-for + x-forwarded-proto + x-forwarded-host 三头同时存在）
2. **`resolveTailscaleClientIp`**：从转发链中解析出原始客户端 IP
3. **WHOIS 比对**：用真实 IP 查 Tailscale daemon（`tailscale whois <ip>`），得到经过 Tailscale 控制面验证的身份，与请求头中的 login 做比较

为什么不直接信任标头？因为 HTTP 标头是客户端可以伪造的。即使 Tailscale 代理会注入这些头，也存在这样的攻击面：一个直接连接 Gateway（绕过 Tailscale 代理）的本地进程可以构造出相同的标头格式。WHOIS 验证则通过 Tailscale daemon 的本地 Unix Socket 查询，不走 HTTP，无法被伪造。

还有一个微妙之处：

```typescript
// src/gateway/auth.ts:573–591
if (
  allowTailscaleHeaderAuth &&
  auth.allowTailscale &&
  !localDirect &&         // 本机直连不走 Tailscale 路径
  !hasExplicitSharedSecretAuth(connectAuth)  // 有显式 token/password 时不降级到 Tailscale
) {
  const tailscaleCheck = await resolveVerifiedTailscaleUser({ ... });
```

条件 `!localDirect` 表示：如果请求来自本地直连（没有任何转发头），就不走 Tailscale 认证路径。这是因为 Tailscale 认证只在 WS Control UI 面（即 `ws-control-ui`）启用（见下文认证面分离），而直连本机不需要 Tailscale 的身份断言。

---

## 4. Trusted Proxy 模式：把认证外包的安全约束

在 Nginx/Traefik 做 SSO 认证的部署场景中，用 `trusted-proxy` 模式是最自然的：反向代理已经完成了认证，只需要把用户身份通过 HTTP 头传给 Gateway。

```typescript
// src/gateway/auth.ts:375–417
function authorizeTrustedProxy(params: {
  req?: IncomingMessage;
  trustedProxies?: string[];
  trustedProxyConfig: GatewayTrustedProxyConfig;
}): { user: string } | { reason: string } {
  const remoteAddr = req.socket?.remoteAddress;
  if (!isTrustedProxyAddress(remoteAddr, trustedProxies)) {
    return { reason: "trusted_proxy_untrusted_source" };
  }
  if (isLoopbackAddress(remoteAddr)) {
    return { reason: "trusted_proxy_loopback_source" };
  }
  // 验证所有 requiredHeaders 都存在
  for (const header of requiredHeaders) {
    if (!headerValue(req.headers[...header])) {
      return { reason: `trusted_proxy_missing_header_${header}` };
    }
  }
  // 提取用户身份
  const user = userHeaderValue.trim();
  // 可选的用户白名单过滤
  if (allowUsers.length > 0 && !allowUsers.includes(user)) {
    return { reason: "trusted_proxy_user_not_allowed" };
  }
  return { user };
}
```

注意这个有意义的细节：`isLoopbackAddress(remoteAddr)` 会返回失败。乍看奇怪——本机 IP 不是很安全吗？实际上这是在防范一种误配置：如果有人把 `trusted-proxy` 模式和本机直连结合使用，那意味着任何本地进程都可以伪造任意用户身份头——这是一个危险的信任扩展。通过显式拒绝 loopback 来源的 trusted-proxy 请求，代码强制要求该模式只在有明确网络拓扑的场景下使用。

另一个细节是 `trusted-proxy` 和 `token` 是互斥的：

```typescript
// src/gateway/auth.ts:362–368
if (auth.mode === "trusted-proxy") {
  if (auth.token) {
    throw new Error(
      "gateway auth mode is trusted-proxy, but a shared token is also configured; " +
      "remove gateway.auth.token / OPENCLAW_GATEWAY_TOKEN because " +
      "trusted-proxy and token auth are mutually exclusive",
    );
  }
}
```

这个互斥性不是技术限制，而是设计约束：在 `trusted-proxy` 模式下，Gateway 不持有任何共享秘钥，不处于任何秘钥泄露的风险中。两者并存意味着 Gateway 实际上有两套认证路径，一旦 token 泄露，trusted-proxy 的隔离性就被打穿了。

---

## 5. 滑动窗口速率限制器：三个设计决策

```typescript
// src/gateway/auth-rate-limit.ts:99–236
export function createAuthRateLimiter(config?: RateLimitConfig): AuthRateLimiter {
  const maxAttempts = config?.maxAttempts ?? 10;      // 默认窗口内最多 10 次失败
  const windowMs   = config?.windowMs   ?? 60_000;   // 默认滑动窗口 1 分钟
  const lockoutMs  = config?.lockoutMs  ?? 300_000;  // 默认锁定 5 分钟
  const exemptLoopback = config?.exemptLoopback ?? true;

  const entries = new Map<string, RateLimitEntry>();

  // 关键设计：允许 GC，不阻止进程退出
  const pruneTimer = setInterval(() => prune(), pruneIntervalMs);
  if (pruneTimer?.unref) {
    pruneTimer.unref();  // 不持有 Node.js 事件循环
  }
```

**决策一：`unref()` 让 prune 定时器不阻止进程退出。**  
这是一个 Node.js 特有的细节：`setInterval` 默认会让 Node.js 进程保持运行（不退出），即使所有业务逻辑都已结束。`unref()` 解除了这个引用，让 prune 定时器成为"后台任务"——如果进程应该退出，这个定时器不会阻止它。

**决策二：滑动窗口用时间戳数组而非计数器。**

```typescript
// src/gateway/auth-rate-limit.ts:139–144
function slideWindow(entry: RateLimitEntry, now: number): void {
  const cutoff = now - windowMs;
  entry.attempts = entry.attempts.filter((ts) => ts > cutoff);
}
```

每次失败都记录时间戳到数组，检查时过滤掉窗口外的记录，计算当前窗口内的失败次数。相比固定窗口计数器，滑动窗口的优点是避免了"窗口重置瞬间的突击攻击"（在计数器归零前瞬间发送大量请求）。代价是内存：每次失败都要存一个时间戳，不过由于 `maxAttempts` 默认只有 10 次，单个 IP 最多存 10 个数字，开销可以忽略。

**决策三：四个独立的速率限制 scope。**

```typescript
// src/gateway/auth-rate-limit.ts:38–41
export const AUTH_RATE_LIMIT_SCOPE_DEFAULT        = "default";
export const AUTH_RATE_LIMIT_SCOPE_SHARED_SECRET  = "shared-secret";
export const AUTH_RATE_LIMIT_SCOPE_DEVICE_TOKEN   = "device-token";
export const AUTH_RATE_LIMIT_SCOPE_HOOK_AUTH      = "hook-auth";
```

Rate limiter 的 Map key 是 `${scope}:${ip}`，而不只是 ip。这意味着：一个设备的 device-token 被暴力破解不会影响它的 shared-secret 认证配额，反之亦然。否则一个针对 device-token 端点的攻击可以把整个 IP 的共享密钥认证也锁住——这是一个"无差别误伤"的设计缺陷。

---

## 6. 时序攻击防护：为什么需要 SHA256 再比较

```typescript
// src/security/secret-equal.ts
import { createHash, timingSafeEqual } from "node:crypto";

export function safeEqualSecret(
  provided: string | undefined | null,
  expected: string | undefined | null,
): boolean {
  if (typeof provided !== "string" || typeof expected !== "string") {
    return false;
  }
  const hash = (s: string) => createHash("sha256").update(s).digest();
  return timingSafeEqual(hash(provided), hash(expected));
}
```

`timingSafeEqual` 保证比较操作总是花费相同时间，无论在哪个位置出现差异——这防止了攻击者通过测量响应时间来推断正确的字节序列（时序攻击）。

但这里还有一个更微妙的细节：为什么先 SHA256 再比较，而不是直接 `timingSafeEqual(Buffer.from(provided), Buffer.from(expected))`？

因为 `timingSafeEqual` 要求两个 Buffer 的**长度相同**，否则直接抛出异常——而异常本身就是信息泄露（"你的 token 长度和正确 token 不一样"）。SHA256 把任意长度的输入变成固定 32 字节的摘要，让长度差异不再可见。

这是一个很小但很精确的安全细节：在密码学代码中，你需要假设攻击者可以观察到你程序的所有外部可观测行为，包括执行时间和错误模式。

---

## 7. 认证面分离：HTTP 和 WS Control UI 为什么不同

```typescript
// src/gateway/auth.ts:74
export type GatewayAuthSurface = "http" | "ws-control-ui";

// src/gateway/auth.ts:419–421
function shouldAllowTailscaleHeaderAuth(authSurface: GatewayAuthSurface): boolean {
  return authSurface === "ws-control-ui";
}

// src/gateway/auth.ts:624–640
export async function authorizeHttpGatewayConnect(params): Promise<GatewayAuthResult> {
  return authorizeGatewayConnect({ ...params, authSurface: "http" });
}
export async function authorizeWsControlUiGatewayConnect(params): Promise<GatewayAuthResult> {
  return authorizeGatewayConnect({ ...params, authSurface: "ws-control-ui" });
}
```

Tailscale 标头认证只在 `ws-control-ui` 面启用，在 `http` 面**显式禁用**。理由在注释里写得很清楚：

```typescript
// src/gateway/auth.ts:83–86
/**
 * Explicit auth surface. HTTP keeps Tailscale forwarded-header auth disabled.
 * WS Control UI enables it intentionally for tokenless trusted-host login.
 */
authSurface?: GatewayAuthSurface;
```

"tokenless trusted-host login"——控制界面（WS）场景下，用户从自己的 Tailscale 网络访问，他们的身份由 Tailscale VPN 保证，不需要额外记住一个 token。这是良好的用户体验设计：用 VPN 身份替代密码。

而 HTTP API 则要求显式凭证，因为 HTTP 请求来源更广，无法安全地假设每个 HTTP 请求都来自可信的 Tailscale 节点。

这是认证体系中的**最小权限原则**在认证面维度的应用：每个接入点只开放它真正需要的认证方式，而不是统一开放所有方式。

---

## 8. 共享令牌世代与热轮转

密码或 token 更新时，所有已建立的 WebSocket 连接怎么办？

```typescript
// src/gateway/server-shared-auth-generation.ts:15–31
export function disconnectStaleSharedGatewayAuthClients(params: {
  clients: Iterable<SharedGatewayAuthClient>;
  expectedGeneration: string | undefined;
}): void {
  for (const gatewayClient of params.clients) {
    if (!gatewayClient.usesSharedGatewayAuth) {
      continue;
    }
    if (gatewayClient.sharedGatewaySessionGeneration === params.expectedGeneration) {
      continue;
    }
    try {
      gatewayClient.socket.close(4001, "gateway auth changed");
    } catch { /* ignore */ }
  }
}
```

每个使用共享密钥（token/password）的 WebSocket 会话在连接时都会记录当前的"世代"（generation，即配置快照的版本标识符）。当密钥轮转发生时，系统遍历所有活跃连接，关闭那些使用旧世代的连接（WebSocket 关闭码 4001，原因 "gateway auth changed"）。

使用 device-token 的连接（移动设备、节点）不在这个机制里，因为它们有独立的轮转机制（见 `server.device-token-rotate-authz.test.ts`）。

这个设计解决了一个真实的安全窗口问题：如果 token 泄露后你立即更新配置，但不踢掉所有已连接的会话，泄露的凭证仍然可以被用于维持连接——只是无法建立新连接。通过主动关闭旧世代连接，轮转成为了真正有效的安全操作。

---

## 9. TOCTOU 防护：并发序列化 Tailscale 验证

```typescript
// src/gateway/rate-limit-attempt-serialization.ts
export async function withSerializedRateLimitAttempt<T>(params: {
  ip: string | undefined;
  scope: string | undefined;
  run: () => Promise<T>;
}): Promise<T> {
  const key = buildSerializationKey(params.ip, params.scope);
  const previous = pendingAttempts.get(key) ?? Promise.resolve();
  let releaseCurrent!: () => void;
  const current = new Promise<void>((resolve) => { releaseCurrent = resolve; });
  const tail = previous.catch(() => {}).then(() => current);
  pendingAttempts.set(key, tail);

  await previous.catch(() => {});
  try {
    return await params.run();
  } finally {
    releaseCurrent();
    if (pendingAttempts.get(key) === tail) {
      pendingAttempts.delete(key);
    }
  }
}
```

Tailscale WHOIS 验证是异步操作——它需要查询本地 Tailscale daemon。这会产生一个 TOCTOU（time-of-check-time-of-use）窗口：如果同一个 IP 在 WHOIS 查询进行中发送多个认证请求，它们可能并发地通过"check"阶段，即使其中有失败应该触发速率限制。

这个函数用一个 `pending` Map 将同一 `{scope, ip}` 的请求串行化：下一个请求等待上一个完成后才开始。这是用 Promise chaining 实现的读写锁（针对单个 key 的串行执行）。

更妙的是它的清理逻辑：`if (pendingAttempts.get(key) === tail)` 用**引用相等性**检查当前 tail 是否仍然是最新的——如果有新请求在等待期间入队，tail 就不再是最新的了，Map 里的引用已经被更新。这里复用了在路由引擎中看到的同一个模式：用对象引用作为廉价的版本标识符。

---

## 全局引用索引

### 源码引用

| 位置 | 内容 |
|------|------|
| `src/gateway/auth.ts:29–67` | GatewayAuthResult 类型，六种 method 定义 |
| `src/gateway/auth.ts:112–116` | hasExplicitSharedSecretAuth，显式凭证检测 |
| `src/gateway/auth.ts:139–160` | isLocalDirectRequest，本机直连检测 |
| `src/gateway/auth.ts:162–178` | getTailscaleUser，从标头提取用户信息 |
| `src/gateway/auth.ts:191–196` | isTailscaleProxyRequest，三头一致验证 |
| `src/gateway/auth.ts:198–229` | resolveVerifiedTailscaleUser，WHOIS 二次验证 |
| `src/gateway/auth.ts:231–304` | resolveGatewayAuth，模式解析（modeSource 字段） |
| `src/gateway/auth.ts:329–368` | assertGatewayAuthConfigured，互斥约束断言 |
| `src/gateway/auth.ts:375–417` | authorizeTrustedProxy，信任代理模式 |
| `src/gateway/auth.ts:419–421` | shouldAllowTailscaleHeaderAuth，认证面分离判断 |
| `src/gateway/auth.ts:449–471` | authorizeTokenAuth，token 认证（含 rate limit 集成） |
| `src/gateway/auth.ts:473–505` | authorizeGatewayConnect，并发序列化入口 |
| `src/gateway/auth.ts:508–621` | authorizeGatewayConnectCore，核心认证分支逻辑 |
| `src/gateway/auth.ts:624–640` | authorizeHttpGatewayConnect / authorizeWsControlUiGatewayConnect |
| `src/gateway/auth-rate-limit.ts:38–41` | 四个速率限制 scope 常量 |
| `src/gateway/auth-rate-limit.ts:99–113` | createAuthRateLimiter，含 unref() 技巧 |
| `src/gateway/auth-rate-limit.ts:139–144` | slideWindow，滑动窗口实现 |
| `src/gateway/auth-rate-limit.ts:145–176` | check 函数，lockout 检测 |
| `src/gateway/auth-rate-limit.ts:178–203` | recordFailure，lockout 触发 |
| `src/gateway/auth-rate-limit.ts:210–222` | prune，内存清理 |
| `src/gateway/auth-token-resolution.ts:12–85` | resolveGatewayAuthToken，四级优先链 |
| `src/gateway/credentials.ts:30–47` | GatewaySecretRefUnavailableError |
| `src/security/secret-equal.ts:1–10` | safeEqualSecret，SHA256 + timingSafeEqual |
| `src/gateway/rate-limit-attempt-serialization.ts:14–38` | withSerializedRateLimitAttempt，TOCTOU 防护 |
| `src/gateway/server-shared-auth-generation.ts:15–31` | disconnectStaleSharedGatewayAuthClients |
| `src/gateway/server-shared-auth-generation.ts:55–68` | setCurrentSharedGatewaySessionGeneration，世代更新 |
| `src/gateway/device-auth.ts:20–54` | buildDeviceAuthPayload v2/v3，管道分隔字段格式 |
| `src/gateway/auth-mode-policy.ts:7–26` | hasAmbiguousGatewayAuthModeConfig，双凭证互斥检测 |

### 外部引用

| 资源 | 链接 |
|------|------|
| 滑动窗口速率限制算法详解 | [arpitbhayani.me](https://arpitbhayani.me/blogs/sliding-window-ratelimiter/) |
| 速率限制算法对比（GeeksforGeeks） | [geeksforgeeks.org](https://www.geeksforgeeks.org/system-design/rate-limiting-algorithms-system-design/) |
| Tailscale Identity 概念文档 | [tailscale.com/docs/concepts/tailscale-identity](https://tailscale.com/docs/concepts/tailscale-identity) |
| OpenClaw Tailscale 集成文档 | [docs.openclaw.ai/gateway/tailscale](https://docs.openclaw.ai/gateway/tailscale) |
| Trusted Proxy Auth 官方文档 | [docs.openclaw.ai/gateway/trusted-proxy-auth](https://docs.openclaw.ai/gateway/trusted-proxy-auth) |
| X-Forwarded-For 安全问题 | [research.securitum.com](https://research.securitum.com/x-forwarded-for-header-security-problems/) |
| HMAC + Nonce 防重放攻击 | [linkedin.com](https://www.linkedin.com/advice/0/how-do-you-prevent-replay-attacks-when-using-hmac-authentication) |
| 时序攻击与常数时间比较 | [codahale.com](https://codahale.com/a-lesson-in-timing-attacks/) |
| 防止时序攻击：双重 HMAC 策略 | [paragonie.com](https://paragonie.com/blog/2015/11/preventing-timing-attacks-on-string-comparison-with-double-hmac-strategy) |
| OWASP 认证 Cheat Sheet | [owasp.org](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) |
| Node.js timingSafeEqual 文档 | [nodejs.org/api/crypto](https://nodejs.org/api/crypto.html#cryptotimingsafeequala-b) |
| Tailscale AI Gateway Identity | [workos.com](https://workos.com/blog/tailscale-ai-gateway-agents-identity) |

---

## 可以学到什么

### 1. 认证模式的分层而非平铺
不要把所有认证方式塞进一个函数，按**信任来源**（网络位置 / 共享秘钥 / 第三方身份）分层。每一层的设计约束不同，混在一起会产生微妙的安全漏洞（如 loopback + trusted-proxy 的误配置）。

### 2. 配置来源可追踪
多来源配置系统中，每个生效的配置项都应该记录它的来源（`modeSource: "config" | "override" | "env" | ...`）。这是可观测性的基础，没有它，生产环境调试会变成猜谜游戏。

### 3. 异步验证需要考虑并发窗口
任何异步验证（如 Tailscale WHOIS）都会产生 TOCTOU 窗口，需要串行化同一 key 的并发请求。用 Promise chaining 构建的"Promise mutex"是 Node.js 中处理这类问题的干净方案。

### 4. 速率限制 scope 要独立
不同的认证端点（shared-secret / device-token / hook-auth）应该有独立的失败计数。共享计数会导致一个端点的攻击误伤其他端点，形成无意的 DoS 向量。

### 5. 秘密比较必须在长度泄露和时序泄露两个维度都保护
`timingSafeEqual` 防止时序攻击，但不防止长度信息泄露（长度不同时会抛异常）。在比较前用哈希统一长度，是一个成熟的处理模式。

### 6. 密钥轮转必须主动踢连接
密钥更新后不踢旧连接，等同于给泄露的密钥一个"维持已有连接"的豁免权。轮转的语义是"所有用这个密钥的访问都应该结束"，这包括已建立的长连接。

---

*文章使用的所有代码引用均已与源码核对，行号基于 2026-04-15 版本的源码快照。*
