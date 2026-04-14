# Security Audit 自检架构：一个系统如何审计自身

> 系列文章第 8 篇 | 源码路径：`src/security/audit.ts`（1384 行）及 `src/security/audit-*.ts` 系列

---

## 前言：为什么一个 AI Agent 需要审计自己？

大多数安全审计工具是"外部"的：一个独立程序扫描目标系统，输出报告。clawdbot 选择了不同的路径——把安全审计内嵌进运行时本身，让系统在每次启动时（也可随时按需调用）对自身配置做一次全面体检。

这不是小功能。`src/security/` 目录下有 **70+ 文件**，`audit.ts` 一个文件就有 1384 行，周围还有数十个专项检查器。本文剖析这套自检架构的设计哲学和工程实现，重点回答三个问题：

1. 系统怎么做到"审计自己"而不是审计别人？
2. 1384 行代码覆盖了哪些安全域，背后的结构是什么？
3. 有哪些值得借鉴的工程决策？

---

## 一、数据结构：Finding 是一等公民

先看最小单元。所有审计结果都归约为 `SecurityAuditFinding`：

```typescript
// src/security/audit.types.ts:3-9
export type SecurityAuditFinding = {
  checkId: string;
  severity: SecurityAuditSeverity;  // "info" | "warn" | "critical"
  title: string;
  detail: string;
  remediation?: string;
};
```

这个类型设计有几个值得注意的地方：

**`checkId` 是可查询的命名空间**。它不是自由文本，而是遵循 `domain.sub_domain.specific_check` 的层级点号命名：

| checkId | 含义 |
|---|---|
| `gateway.bind_no_auth` | 网关绑定到非回环地址但没有认证 |
| `gateway.control_ui.allowed_origins_wildcard` | Control UI 允许了通配符 Origin |
| `tools.exec.security_full_configured` | 配置了完全执行信任 |
| `fs.state_dir.perms_world_writable` | 状态目录全局可写 |
| `models.small_params` | 小参数量模型带工具暴露 |
| `summary.attack_surface` | 攻击面总览（info 级） |

这个命名空间有两个作用：一是让代码里的 `if (finding.checkId === "gateway.bind_no_auth")` 精确匹配，二是让运维人员能在日志中快速搜索特定检查类型。OWASP 的 [Security Misconfiguration（A05:2021）](https://owasp.org/Top10/2021/A05_2021-Security_Misconfiguration/) 指出，可以机器解析的结构化发现报告是有效响应的前提——clawdbot 的 checkId 设计正是这个思路的实现。

**`remediation` 是 first-class 字段，而非附注**。每个 critical/warn 发现都带有 `remediation` 字符串，告诉用户具体要改什么配置行。这区别于"你有问题，自己查文档"的审计报告风格，更接近 AWS Security Hub 的 [Remediation as Code](https://docs.aws.amazon.com/prescriptive-guidance/latest/vulnerability-management/remediate-security-findings.html) 理念——发现和修复步骤打包在一起交付。

聚合结果是 `SecurityAuditReport`：

```typescript
// src/security/audit.types.ts:17-30
export type SecurityAuditReport = {
  ts: number;
  summary: SecurityAuditSummary;  // { critical, warn, info }
  findings: SecurityAuditFinding[];
  deep?: {
    gateway?: {
      attempted: boolean;
      url: string | null;
      ok: boolean;
      error: string | null;
      close?: { code: number; reason: string } | null;
    };
  };
};
```

`summary` 是三个计数器的快照，由一个简单的线性扫描计算：

```typescript
// src/security/audit.ts:164-178
function countBySeverity(findings: SecurityAuditFinding[]): SecurityAuditSummary {
  let critical = 0;
  let warn = 0;
  let info = 0;
  for (const f of findings) {
    if (f.severity === "critical") {
      critical += 1;
    } else if (f.severity === "warn") {
      warn += 1;
    } else {
      info += 1;
    }
  }
  return { critical, warn, info };
}
```

---

## 二、架构：三层审计流水线

`runSecurityAudit`（audit.ts:1273）是入口，它把所有收集器组织成三个层次：

### 层一：同步配置检查（non-deep）

这层只读配置，不做 I/O，速度极快。所有检查器被 `audit.nondeep.runtime.ts` 统一导出：

```typescript
// src/security/audit.nondeep.runtime.ts:1-29（全文）
export {
  collectAttackSurfaceSummaryFindings,
  collectSmallModelRiskFindings,
} from "./audit-extra.summary.js";

export {
  collectExposureMatrixFindings,
  collectGatewayHttpNoAuthFindings,
  collectGatewayHttpSessionKeyOverrideFindings,
  collectHooksHardeningFindings,
  collectLikelyMultiUserSetupFindings,
  collectMinimalProfileOverrideFindings,
  collectModelHygieneFindings,
  collectNodeDangerousAllowCommandFindings,
  collectNodeDenyCommandPatternFindings,
  collectSandboxDangerousConfigFindings,
  collectSandboxDockerNoopFindings,
  collectSecretsInConfigFindings,
  collectSyncedFolderFindings,
} from "./audit-extra.sync.js";

export {
  collectSandboxBrowserHashLabelFindings,
  collectIncludeFilePermFindings,
  collectPluginsTrustFindings,
  collectStateDeepFilesystemFindings,
  collectWorkspaceSkillSymlinkEscapeFindings,
  readConfigSnapshotForAudit,
} from "./audit-extra.async.js";
```

这个桶文件（barrel export）让 `runSecurityAudit` 能用 `auditNonDeep.collectXxx()` 统一调用，同时隐藏了实现分散在三个文件的细节。

### 层二：文件系统检查（包含异步 I/O）

当 `includeFilesystem === true`（默认值）时，执行：

- **状态目录权限检查**：`fs.state_dir.perms_world_writable`（critical）→ 他人能写入 Agent 状态
- **配置文件权限检查**：`fs.config.perms_world_readable`（critical）→ Token/密钥泄漏
- **包含文件权限检查**：扫描所有 `include:` 引用的配置文件
- **Docker 沙盒标签验证**：检查 sandbox 镜像的安全 hash epoch（`SANDBOX_BROWSER_SECURITY_HASH_EPOCH`）
- **Workspace Skill 符号链接逃逸检查**：`audit-workspace-skill-escape.test.ts` 有 27 个测试用例
- **Plugin 信任级别检查**：`collectPluginsTrustFindings`
- **深度代码安全扫描**：仅 `deep=true` 时执行

文件系统检查使用了对称权限处理逻辑。以 symlink 为例：

```typescript
// src/security/audit.ts:261-262
const skipReadablePermWarnings = configPerms.isSymlink;
if (configPerms.isSymlink) {
```

如果配置文件是符号链接，readable 权限警告会被跳过（因为 symlink 的权限意义不同），但 writable 警告保留。这种"知道你在处理什么类型"的细节防护在主流安全审计框架里常被忽略。

### 层三：深度网络探针（live probe）

仅 `deep === true` 时执行，向本地 Gateway 发起实际 WebSocket 连接：

```typescript
// src/security/audit.ts:1370-1380
const deepProbeResult = context.deep
  ? await maybeProbeGateway({
      cfg,
      env,
      timeoutMs: context.deepTimeoutMs,
      probe: context.probeGatewayFn ?? (await loadGatewayProbeDeps()).probeGateway,
      explicitAuth: context.deepProbeAuth,
    })
  : undefined;
```

这是一次真实连接测试：如果 Gateway 配置了 Token 认证，这里会尝试连接并返回 `ok: true/false`。用于验证"配置里说有 Token，实际 Gateway 也能正常接受认证"。`deepTimeoutMs` 默认 5 秒（最小 250ms），防止 audit 被 Gateway 超时拖死。

---

## 三、动态懒加载：审计不应拖慢启动

`audit.ts` 顶部定义了六个模块级 Promise 缓存：

```typescript
// src/security/audit.ts:104-122
let channelPluginsModulePromise: Promise<...> | undefined;
let auditNonDeepModulePromise: Promise<...> | undefined;
let auditChannelModulePromise: Promise<...> | undefined;
let pluginRegistryLoaderModulePromise: Promise<...> | undefined;
let pluginMetadataRegistryLoaderModulePromise: Promise<...> | undefined;
let gatewayProbeDepsPromise: Promise<{...}> | undefined;
```

每个对应一个 `loadXxx()` 函数，使用 `??=` 操作符做惰性初始化：

```typescript
// src/security/audit.ts:129-132
async function loadAuditNonDeepModule() {
  auditNonDeepModulePromise ??= import("./audit.nondeep.runtime.js");
  return await auditNonDeepModulePromise;
}
```

这个模式的效果：
- **首次调用**：触发 `import()`，开始加载模块，Promise 缓存到变量
- **后续调用**：直接 `await` 已有 Promise，如果模块已加载则立即 resolve
- **并发调用**：多个并发调用等待同一个 Promise，不会触发多次加载

为什么要这样做？Gateway 深度探针依赖模块（`gateway/call.js`, `gateway/probe.js`）在常规启动路径中不需要，如果启动时就全量 import，会增加冷启动延迟。对于频繁调用的 `runSecurityAudit`（每次 UI 刷新都可能触发），这个差异是可感知的。

同样的 `??=` 懒加载模式出现在 `audit-extra.async.ts:62-68`：

```typescript
// src/security/audit-extra.async.ts:62-68
let skillsModulePromise: Promise<typeof import("../agents/skills.js")> | undefined;
let configModulePromise: Promise<typeof import("../config/config.js")> | undefined;

function loadSkillsModule() {
  skillsModulePromise ??= import("../agents/skills.js");
  return skillsModulePromise;
}
```

这是一个系统性工程决策，不是随机优化。

---

## 四、上下文敏感的严重性：同一漏洞在不同环境有不同风险

clawdbot 的审计规则里，许多 severity 不是硬编码的，而是根据运行时配置动态计算。这是区别于传统静态规则扫描的重要特征。

**例一：通配符 Origin 的严重性取决于网关是否对外暴露**

```typescript
// src/security/audit.ts:456-467
if (controlUiAllowedOrigins.includes("*")) {
  const exposed = bind !== "loopback";
  findings.push({
    checkId: "gateway.control_ui.allowed_origins_wildcard",
    severity: exposed ? "critical" : "warn",
    ...
  });
}
```

`bind === "loopback"`（只监听本地）时，通配符 Origin 是 warn（可接受的本地测试配置）；一旦 `bind` 是任何其他值（0.0.0.0 或具体 IP），立即升为 critical。理由：通配符 Origin 在对外暴露时会完全关闭 DNS rebinding 防护。

DNS rebinding 攻击的工作原理是：攻击者控制 DNS 服务器，让恶意域名先解析到攻击者服务器，再切换到 `127.0.0.1`；浏览器在 Origin 检查通过后，会执行跨域请求访问本地 Gateway。Host 头和 Origin 的严格匹配是对抗这类攻击的核心防线（参考：[GitHub Security Blog: DNS Rebinding Attacks](https://github.blog/security/application-security/dns-rebinding-attacks-explained-the-lookup-is-coming-from-inside-the-house/)、[Stanford CS: Protecting Browsers from DNS Rebinding](https://crypto.stanford.edu/dns/dns-rebinding.pdf)）。

**例二：HTTP 危险工具重新开放的严重性取决于 Tailscale Funnel**

```typescript
// src/security/audit.ts:389-402
if (reenabledOverHttp.length > 0) {
  const extraRisk = bind !== "loopback" || tailscaleMode === "funnel";
  findings.push({
    checkId: "gateway.tools_invoke_http.dangerous_allow",
    severity: extraRisk ? "critical" : "warn",
    ...
  });
}
```

只开了 Tailscale Serve（tailnet 内可访问）时是 warn；开了 Funnel（公网可访问）时是 critical。

**例三：小模型风险的严重性取决于工具暴露**

`collectSmallModelRiskFindings` 检查参数量 ≤300B 的模型，但严重性取决于该模型是否能使用 web_search/web_fetch/browser：

```typescript
// src/security/audit-extra.summary.ts:281-293
findings.push({
  checkId: "models.small_params",
  severity: hasUnsafe ? "critical" : "info",
  ...
});
```

同样是 "使用了小模型" 这件事，如果关掉了所有 Web 工具且运行在沙盒里，只是 info；否则是 critical。这体现了 [AI Agent 安全研究](https://www.enkryptai.com/blog/small-models-big-problems-why-your-ai-agents-might-be-sitting-ducks) 的共识：小模型的问题不是模型本身，而是它被赋予了不匹配其安全对齐能力的工具权限。

---

## 五、Plugin 安全审计扩展点

`collectPluginSecurityAuditFindings`（audit.ts:681）是整个审计系统最有设计感的部分之一：

```typescript
// src/security/audit.ts:681-750（精简版）
async function collectPluginSecurityAuditFindings(
  context: AuditExecutionContext,
): Promise<SecurityAuditFinding[]> {
  let collectors = getActivePluginRegistry()?.securityAuditCollectors ?? [];
  
  // 如果注册表中没有，尝试从元数据加载（不触发完整 plugin 初始化）
  if (collectors.length === 0) {
    const snapshot = (await loadPluginMetadataRegistryLoaderModule())
      .loadPluginMetadataRegistrySnapshot({ ... });
    collectors = snapshot.securityAuditCollectors ?? [];
  }
  
  const collectorResults = await Promise.all(
    collectors.map(async (entry) => {
      try {
        return await entry.collector({ config, sourceConfig, env, stateDir, configPath });
      } catch (err) {
        return [{
          checkId: `plugins.${entry.pluginId}.security_audit_failed`,
          severity: "warn" as const,
          title: "Plugin security audit collector failed",
          detail: `${entry.pluginId}: ${String(err)}`,
        }];
      }
    }),
  );
  return collectorResults.flat();
}
```

三个设计决策值得拆解：

**1. 两级注册表加载**：先查 `getActivePluginRegistry()`（运行时注册表，已加载的 plugin），再查元数据注册表（只加载元数据，不执行 plugin 代码）。这样 audit 时不需要完整初始化所有 plugin，减少副作用。

**2. 错误隔离**：每个 plugin collector 的错误都被 catch，降级为一个 `warn` 发现，不会让一个 plugin 的 bug 导致整个审计崩溃。失败本身也变成了一个可见的 finding（`plugins.xxx.security_audit_failed`）。

**3. 并发执行**：`Promise.all` 并行运行所有 plugin 的 collector，保持审计速度不因 plugin 数量线性增长。

这是 OWASP 在 [AI Agent Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html) 中提倡的"defense in depth"原则在框架层的实现：核心系统和 plugin 各自能扩展安全检查，但 plugin 的失败不损害核心审计的完整性。

---

## 六、Exec 安全审计：最复杂的检查域

`collectExecRuntimeFindings`（audit.ts:804-1110）是 audit.ts 里最长的单一函数，覆盖执行表面的多个维度：

### 6.1 跨域交叉检查：channel policy × exec trust

```typescript
// src/security/audit.ts:891-902
if (openExecSurfacePaths.length > 0 && execEnabledScopes.length > 0) {
  findings.push({
    checkId: "security.exposure.open_channels_with_exec",
    severity: fullExecScopes.length > 0 ? "critical" : "warn",
    title: "Open channels can reach exec-enabled agents",
    detail:
      `Open DM/group access detected at:\n${openExecSurfacePaths...}\n` +
      `Exec-enabled scopes:\n${execEnabledScopes...}`,
  });
}
```

`collectOpenExecSurfacePaths`（audit.ts:1112）用递归 DFS 遍历 `cfg.channels` 的整个对象树，找出所有 `groupPolicy === "open"` 或 `dmPolicy === "open"` 的路径：

```typescript
// src/security/audit.ts:1112-1145
function collectOpenExecSurfacePaths(cfg: OpenClawConfig): string[] {
  const channels = asNullableRecord(cfg.channels);
  const hits = new Set<string>();
  const seen = new WeakSet<object>();
  const visit = (value: unknown, scope: string) => {
    const record = asNullableRecord(value);
    if (!record || seen.has(record)) return;
    seen.add(record);
    if (record.groupPolicy === "open") hits.add(`${scope}.groupPolicy`);
    if (record.dmPolicy === "open")    hits.add(`${scope}.dmPolicy`);
    for (const [key, nested] of Object.entries(record)) {
      if (key === "groups" || key === "accounts" || key === "dms") {
        visit(nested, `${scope}.${key}`); continue;
      }
      if (asNullableRecord(nested)) visit(nested, `${scope}.${key}`);
    }
  };
  for (const [channelId, channelValue] of Object.entries(channels)) {
    visit(channelValue, `channels.${channelId}`);
  }
  return Array.from(hits).toSorted();
}
```

`seen` 用 `WeakSet` 防止对象循环引用导致的无限递归——这个细节在 JSON 配置树里通常不会触发，但防御性地包含体现了代码的严谨性。

然后把"开放频道"和"启用了 exec 的 agent"做**笛卡尔积**检查：任何人都能发消息的频道 + 能执行任意命令的 agent = critical。这种跨域交叉审计是单域检查无法覆盖的，也是 Self-Audit 架构的核心价值：系统知道自己的所有配置，能做全局视角的推理。

### 6.2 safeBinTrustedDirs 的目录分类

```typescript
// src/security/audit.ts:959-996
const classifyRiskySafeBinTrustedDir = (entry: string): string | null => {
  const normalized = path.resolve(raw).replace(/\\/g, "/").toLowerCase();
  if (normalized === "/tmp" || normalized.startsWith("/tmp/") || ...) {
    return "temporary directory is mutable and easy to poison";
  }
  if (normalized === "/usr/local/bin" || normalized === "/opt/homebrew/bin" || ...) {
    return "package-manager bin directory (often user-writable)";
  }
  if (normalized.startsWith("/users/") || normalized.startsWith("/home/") || ...) {
    return "home-scoped bin directory (typically user-writable)";
  }
  return null;
};
```

这段代码对可信二进制目录做白名单式风险分类：`/tmp` 是可变的（易被污染），`/usr/local/bin` 是包管理器目录（用户可写），`~/.*` 是用户主目录（完全用户控制）。只有 `/bin`、`/usr/bin` 这类系统目录才不会触发警告。理由是：safe bins 机制依赖"可信目录中的可信二进制"的假设——如果目录本身是用户可写的，攻击者可以在里面放一个和真实 binary 同名的恶意文件。

---

## 七、AuditExecutionContext：无状态收集器的有状态上下文

所有收集器都接收同一个 `AuditExecutionContext` 对象：

```typescript
// src/security/audit.ts:84-102
type AuditExecutionContext = {
  cfg: OpenClawConfig;
  sourceConfig: OpenClawConfig;      // 未经合并的原始配置（用于区分"配置里有"还是"从环境变量来"）
  env: NodeJS.ProcessEnv;
  platform: NodeJS.Platform;
  includeFilesystem: boolean;
  includeChannelSecurity: boolean;
  deep: boolean;
  deepTimeoutMs: number;
  stateDir: string;
  configPath: string;
  execIcacls?: ExecFn;               // Windows 权限检查（DI）
  execDockerRawFn?: ExecDockerRawFn;  // Docker 标签检查（DI）
  probeGatewayFn?: ProbeGatewayFn;    // Gateway 探针（DI）
  plugins?: ReturnType<typeof listChannelPlugins>;  // Channel plugin（DI）
  configSnapshot: ConfigFileSnapshot | null;
  codeSafetySummaryCache: Map<string, Promise<unknown>>;  // 跨调用共享缓存
  deepProbeAuth?: { token?: string; password?: string };
};
```

注意两点：

**`sourceConfig` vs `cfg`**：`cfg` 是完全解析后的配置（包含默认值、环境变量覆盖），`sourceConfig` 是用户写的原始配置文件内容。`collectGatewayConfigFindings` 同时接收两者，用 `sourceConfig` 判断"这个 Token 是用户写死在配置文件里的还是来自环境变量"——前者是真正的风险（Token 进了版本库），后者是正常做法。

**依赖注入的 `execIcacls`/`execDockerRawFn`/`probeGatewayFn`**：这三个字段在测试中被替换为 Mock，让针对 Windows ACL 检查、Docker 标签检查、Gateway 探针的测试完全可控。这是 clawdbot 一贯的 DI 风格在安全审计模块的体现。

---

## 八、Fix 系统：审计与修复的对称设计

`fix.ts` 提供了自动修复能力，但范围被刻意限制：

```typescript
// src/security/fix.ts:13-41
export type SecurityFixChmodAction = {
  kind: "chmod"; path: string; mode: number;
  ok: boolean; skipped?: string; error?: string;
};
export type SecurityFixIcaclsAction = {
  kind: "icacls"; path: string; command: string;
  ok: boolean; skipped?: string; error?: string;
};
export type SecurityFixResult = {
  ok: boolean;
  stateDir: string; configPath: string;
  configWritten: boolean;
  changes: string[]; actions: SecurityFixAction[]; errors: string[];
};
```

`SecurityFixAction` 只有两种：`chmod`（POSIX 权限修改）和 `icacls`（Windows ACL 修改）。系统不尝试自动修改 OpenClaw 配置文件的内容，因为配置修改有业务语义，需要用户确认。这个边界划定体现了自动化安全修复的基本原则：只修复"结构性错误"（文件权限），不触碰"语义决策"（安全策略配置）。

`safeChmod`（fix.ts:49）在执行 chmod 前会 `lstat`：如果目标是符号链接，直接返回 `skipped: "symlink"`——因为 chmod 符号链接只改 symlink 本身的权限而非目标文件，修了也没用，不如明确跳过。

---

## 九、危险标志的显式枚举

`dangerous-config-flags.ts` 维护了一个危险配置标志的枚举列表：

```typescript
// src/security/dangerous-config-flags.ts:13-36（精简）
export function collectEnabledInsecureOrDangerousFlags(cfg: OpenClawConfig): string[] {
  const enabledFlags: string[] = [];
  if (cfg.gateway?.controlUi?.allowInsecureAuth === true)
    enabledFlags.push("gateway.controlUi.allowInsecureAuth=true");
  if (cfg.gateway?.controlUi?.dangerouslyAllowHostHeaderOriginFallback === true)
    enabledFlags.push("gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true");
  if (cfg.gateway?.controlUi?.dangerouslyDisableDeviceAuth === true)
    enabledFlags.push("gateway.controlUi.dangerouslyDisableDeviceAuth=true");
  if (cfg.hooks?.gmail?.allowUnsafeExternalContent === true)
    enabledFlags.push("hooks.gmail.allowUnsafeExternalContent=true");
  if (cfg.tools?.exec?.applyPatch?.workspaceOnly === false)
    enabledFlags.push("tools.exec.applyPatch.workspaceOnly=false");
  ...
}
```

这些标志有一个共同特征：它们的存在本身就是风险，但都是合法的调试/逃生选项。设计者没有禁用它们，而是让它们"永远被看到"——每次 audit 都会把这些标志列出来，让运维人员意识到当前有哪些防护被主动关闭了。

这和上一篇文章里 `dangerouslyForceUnsafeInstall` 的哲学一致：逃生舱口必须留着，但必须留痕。

---

## 十、可以学到什么

### 1. 把审计内嵌进运行时，而非外挂

外部扫描工具需要被人为触发，往往在事故后才用。内嵌审计在系统启动时自动运行，问题在第一次配置错误时就被发现，不需要等待外部 CI/安全扫描流水线。代价是增加了系统复杂度，值不值取决于安全要求的等级。

### 2. `checkId` 命名空间让审计结果可编程

自由文本的警告消息只能给人看，结构化的 checkId 能被代码解析：可以过滤、可以统计、可以和其他系统对接（如告警规则引擎）。为审计发现定义稳定的标识符是把安全审计从"运维流程"提升为"系统能力"的关键步骤。

### 3. `remediation` 和 `finding` 捆绑交付

把修复步骤写进发现结构，而不是让用户自己查文档，大幅降低了"发现问题但不知道怎么修"的摩擦。这在团队环境中尤为重要——收到告警的人不一定是配置这个系统的人。

### 4. 上下文敏感的严重性优于静态阈值

`severity: exposed ? "critical" : "warn"` 这个模式比静态规则更精准。同一个配置选项，在不同的网络暴露场景下有本质不同的风险等级。减少噪音（降低 false positives）和减少漏报（保持 false negatives 低）是两个矛盾目标，上下文敏感的计算是平衡两者的有效手段。

### 5. 跨域交叉检查是单域检查无法覆盖的

"开放频道 + exec 信任"这类跨域检查，只有系统了解自身所有配置时才能做。这是 Self-Audit 架构的核心价值：知道自己的整体状态，能做全局视角的安全推理，而不是每个模块只看自己的部分。

---

## 源码引用索引

| 文件 | 行号 | 内容 |
|---|---|---|
| `audit.types.ts` | 1-30 | SecurityAuditFinding/Summary/Report 类型定义 |
| `audit.ts` | 54-82 | SecurityAuditOptions（公开入口参数） |
| `audit.ts` | 84-102 | AuditExecutionContext（内部执行上下文） |
| `audit.ts` | 104-161 | 六个动态懒加载 Promise 缓存 |
| `audit.ts` | 164-178 | countBySeverity |
| `audit.ts` | 187-316 | collectFilesystemFindings（状态目录 + 配置文件权限） |
| `audit.ts` | 318-650 | collectGatewayConfigFindings（约 20 项 Gateway 检查） |
| `audit.ts` | 389-402 | context-sensitive severity：HTTP 危险工具 |
| `audit.ts` | 456-467 | context-sensitive severity：通配符 Origin |
| `audit.ts` | 681-750 | collectPluginSecurityAuditFindings（plugin 扩展点） |
| `audit.ts` | 804-1110 | collectExecRuntimeFindings（exec 安全面） |
| `audit.ts` | 1112-1145 | collectOpenExecSurfacePaths（DFS 遍历 channels） |
| `audit.ts` | 1273-1384 | runSecurityAudit（主编排器） |
| `audit.nondeep.runtime.ts` | 1-29 | 桶文件：同步/异步收集器统一导出 |
| `audit-extra.summary.ts` | 177-205 | collectAttackSurfaceSummaryFindings |
| `audit-extra.summary.ts` | 207-297 | collectSmallModelRiskFindings（300B 阈值 + 工具暴露） |
| `audit-extra.async.ts` | 62-68 | 文件系统检查中的懒加载 |
| `audit-deep-code-safety.ts` | 11-33 | collectDeepCodeSafetyFindings（guard by params.deep） |
| `dangerous-config-flags.ts` | 13-50 | collectEnabledInsecureOrDangerousFlags |
| `fix.ts` | 13-41 | SecurityFixAction/Result 类型（chmod + icacls） |
| `fix.ts` | 49-80 | safeChmod（symlink guard） |

---

## 外部参考

1. [OWASP A05:2021 Security Misconfiguration](https://owasp.org/Top10/2021/A05_2021-Security_Misconfiguration/) — 结构化安全配置检查的行业背景
2. [OWASP A02:2025 Security Misconfiguration](https://owasp.org/Top10/2025/A02_2025-Security_Misconfiguration/) — 最新版分类（2025年已上升为 A02）
3. [OWASP AI Agent Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html) — AI Agent 安全的通用原则
4. [AWS: Remediate Security Findings](https://docs.aws.amazon.com/prescriptive-guidance/latest/vulnerability-management/remediate-security-findings.html) — 结构化修复步骤的工程实践
5. [GitHub Security Blog: DNS Rebinding Attacks Explained](https://github.blog/security/application-security/dns-rebinding-attacks-explained-the-lookup-is-coming-from-inside-the-house/) — Gateway Host-header 检查的攻击背景
6. [Stanford: Protecting Browsers from DNS Rebinding Attacks (PDF)](https://crypto.stanford.edu/dns/dns-rebinding.pdf) — DNS rebinding 防御的学术基础
7. [Enkrypt AI: Small Models, Big Problems](https://www.enkryptai.com/blog/small-models-big-problems-why-your-ai-agents-might-be-sitting-ducks) — 小模型 + 工具权限的安全风险研究
8. [Semiconductor Engineering: Small Language Models Create New Security Risks](https://semiengineering.com/small-language-models-create-new-security-risks/) — SLM 安全对齐弱点
9. [Snyk: Code Security Audit Guide](https://snyk.io/articles/code-security-audit/) — 代码安全审计作为持续过程的方法论
10. [SentinelOne: Vulnerability Remediation Guide](https://www.sentinelone.com/cybersecurity-101/cybersecurity/vulnerability-remediation/) — 漏洞修复流程的分类与优先级
11. [arXiv: Security Concerns for Large Language Models Survey](https://arxiv.org/html/2505.18889v1) — LLM 安全威胁综述

---

*文章所有代码引用均基于 `src/security/` 目录源码，行号经 `grep -n` 验证。*
