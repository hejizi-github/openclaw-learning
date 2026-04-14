# Plugin 供应链安全：Clawdbot 的五层代码防御体系

> 每一个成熟的插件平台，都是一个潜在的供应链攻击面。

## 为什么插件安全如此困难

2025 年 9 月，一个被称为"Shai-Hulud"的蠕虫通过污染 18 个 npm 包，波及每周下载量超过 26 亿次的生态系统 [[CISA alert]](https://www.cisa.gov/news-events/alerts/2025/09/23/widespread-supply-chain-compromise-impacting-npm-ecosystem)。2026 年 3 月，Axios（每周 7000 万下载）遭 North Korean APT 组织 Sapphire Sleet 污染 [[Microsoft Security Blog]](https://www.microsoft.com/en-us/security/blog/2026/04/01/mitigating-the-axios-npm-supply-chain-compromise/)。这不是孤立事件，而是常态。

Clawdbot 是一个允许用户安装第三方 channel 插件和 skill 脚本的 AI agent 平台。每一个插件都是用户系统上的任意代码执行权。挑战在于：

- 插件需要运行时动态加载（不能完全静态分析）
- TypeScript 源码可以不经编译直接运行（Jiti）
- 恶意插件可以伪装成合法插件（依赖混淆、符号链接）
- 平台要对用户透明，不能做黑盒沙箱（影响可调试性）

Clawdbot 的解决方案不是银弹，而是**五层防御纵深（Defense in Depth）**——安装前扫描三层，运行时隔离两层，每层独立失效，组合覆盖不同攻击向量。

---

## 第一层：依赖黑名单（Dependency Denylist）

**核心文件**: `src/plugins/dependency-denylist.ts`

安装时最先检查的不是代码内容，而是依赖声明。已知危险的包名被硬编码进黑名单：

```typescript
// src/plugins/dependency-denylist.ts:1
const BLOCKED_INSTALL_DEPENDENCY_PACKAGE_NAMES = ["plain-crypto-js"] as const;
```

目前只有一个 `plain-crypto-js`，但机制已经完备。关键的设计不在于列表长度，而在于**多层路径穿透检测**：攻击者知道名字被拦截后，会通过三种方式绕过：

### 绕过手法 1：alias（别名）

在 `package.json` 的 `overrides` 字段声明：

```json
{
  "overrides": {
    "safe-looking-name": "npm:plain-crypto-js"
  }
}
```

Clawdbot 递归解析 `overrides` 对象树，提取所有 `npm:` 前缀的包名，和黑名单做 case-insensitive 比对 [dependency-denylist.ts:150-175]。

### 绕过手法 2：node_modules 符号链接

恶意插件可以在 `node_modules/safe-name` 创建一个符号链接，实际指向 `../plain-crypto-js/dist/index.js`。

Clawdbot 在 manifest 遍历时专门处理这个情况：

```typescript
// src/plugins/install-security-scan.runtime.ts:163-201
async function inspectNodeModulesSymlinkTarget(params: {
  rootRealPath: string;
  symlinkPath: string;
  symlinkRelativePath: string;
}): Promise<...> {
  const resolvedTargetPath = await fs.realpath(params.symlinkPath);

  // 必须在 install root 内，否则直接报错
  if (!isPathInside(params.rootRealPath, resolvedTargetPath)) {
    throw new Error(
      `manifest dependency scan found node_modules symlink target outside install root at ${params.symlinkRelativePath}`,
    );
  }
  // 再次对 realpath 路径做 blocked package 检测
  const blockedDirectoryFinding = findBlockedPackageDirectoryInPath({...});
  ...
}
```

关键洞察：`realpath` 解析符号链接后，重新对**真实路径**做一次黑名单检测。单层路径检测（只看符号链接名）是不够的。

### 绕过手法 3：深度嵌套 node_modules

vendored 的 transitive dependencies 里可能藏着危险包。代码注释里明确说明了设计意图：

```typescript
// src/plugins/install-security-scan.runtime.ts:319-320
// Intentionally walk vendored/node_modules trees so bundled transitive
// manifests cannot hide blocked packages from install-time policy checks.
```

遍历深度限制（默认 64 层、10,000 个目录）防止 DoS，但不影响正常插件：

```typescript
// src/plugins/install-security-scan.runtime.ts:233-237
const DEFAULT_PACKAGE_MANIFEST_TRAVERSAL_LIMITS: PackageManifestTraversalLimits = {
  maxDepth: 64,
  maxDirectories: 10_000,
  maxManifests: 10_000,
};
```

这三个上限都支持通过环境变量 `OPENCLAW_INSTALL_SCAN_MAX_DEPTH` 等覆盖，方便在特殊环境（如 monorepo）调整。

---

## 第二层：静态代码扫描（Skill Scanner）

**核心文件**: `src/security/skill-scanner.ts`

依赖黑名单只能拦截已知危险包，但恶意代码完全可以在一个"全新"包名下首次出现。第二层是基于**危险模式的静态代码扫描**。

Clawdbot 定义了两类规则，区别在于作用域：

### LINE_RULES：逐行扫描

适合能精确定位到行号的模式：

```typescript
// src/security/skill-scanner.ts:147-173
const LINE_RULES: LineRule[] = [
  {
    ruleId: "dangerous-exec",
    severity: "critical",
    message: "Shell command execution detected (child_process)",
    pattern: /\b(exec|execSync|spawn|spawnSync|execFile|execFileSync)\s*\(/,
    requiresContext: /child_process/,  // ← 关键！
  },
  {
    ruleId: "dynamic-code-execution",
    severity: "critical",
    message: "Dynamic code execution detected",
    pattern: /\beval\s*\(|new\s+Function\s*\(/,
  },
  {
    ruleId: "crypto-mining",
    severity: "critical",
    message: "Possible crypto-mining reference detected",
    pattern: /stratum\+tcp|stratum\+ssl|coinhive|cryptonight|xmrig/i,
  },
  {
    ruleId: "suspicious-network",
    severity: "warn",
    message: "WebSocket connection to non-standard port",
    pattern: /new\s+WebSocket\s*\(\s*["']wss?:\/\/[^"']*:(\d+)/,
  },
];
```

### SOURCE_RULES：全文上下文扫描

某些危险模式只有在**两个条件同时成立**时才真正危险：

```typescript
// src/security/skill-scanner.ts:177-205
const SOURCE_RULES: SourceRule[] = [
  {
    ruleId: "potential-exfiltration",
    severity: "warn",
    message: "File read combined with network send — possible data exfiltration",
    pattern: /readFileSync|readFile/,
    requiresContext: /\bfetch\b|\bpost\b|http\.request/i,  // 读文件 AND 发网络
  },
  {
    ruleId: "env-harvesting",
    severity: "critical",
    message: "Environment variable access combined with network send",
    pattern: /process\.env/,
    requiresContext: /\bfetch\b|\bpost\b|http\.request/i,  // 读环境变量 AND 发网络
  },
];
```

### requiresContext 的精妙之处

`dangerous-exec` 规则要求 `requiresContext: /child_process/`。这意味着：如果代码里有 `exec()` 但**没有** `import 'child_process'`（比如这只是一个叫 `exec` 的普通函数），规则不会触发。

这是在**false positive 和 false negative 之间做了显式权衡**。学术界对 JavaScript 恶意代码检测的研究表明，上下文感知分析能显著降低误报率 [[JStill]](https://www.cse.psu.edu/~sxz16/papers/JStill.pdf) [[JaSt]](https://swag.cispa.saarland/papers/fass2018jast.pdf)。Clawdbot 选择了基于关键字上下文的轻量方案，而非完整的 AST 分析，以牺牲一定精度换取扫描速度。

### 扫描性能：mtime 缓存

```typescript
// src/security/skill-scanner.ts:52-53
const FILE_SCAN_CACHE_MAX = 5000;
const DIR_ENTRY_CACHE_MAX = 5000;
```

每个文件的扫描结果按 `(filePath, size, mtimeMs, maxFileBytes)` 四元组缓存。当文件未修改（mtime 和 size 一致）时，直接返回缓存的 findings，避免重复扫描。在 `security audit --deep` 这样需要多次运行的场景下，这个缓存效果显著。

扫描上限：最多 500 个文件，每文件最大 1MB：

```typescript
// src/security/skill-scanner.ts:50-51
const DEFAULT_MAX_SCAN_FILES = 500;
const DEFAULT_MAX_FILE_BYTES = 1024 * 1024;
```

超出上限时不报错，而是截断——因为大型包里的核心文件通常在前几百个。

---

## 第三层：安装管道与 Hook 扩展

**核心文件**: `src/plugins/install-security-scan.runtime.ts`

这一层是两层原子扫描的编排器，也是**可扩展性**的设计所在。

### 扫描管道

每次安装触发三步检查（以 bundle 安装为例）：

```typescript
// src/plugins/install-security-scan.runtime.ts:652-705
export async function scanBundleInstallSourceRuntime(params: ...) {
  // Step 1: 依赖黑名单（如果命中，立即返回，不继续）
  const dependencyBlocked = await scanManifestDependencyDenylist({...});
  if (dependencyBlocked) {
    return dependencyBlocked;
  }

  // Step 2: 静态代码扫描
  const builtinScan = await scanDirectoryTarget({...});
  const builtinBlocked = resolveBuiltinScanDecision({...});

  // Step 3: before_install Hook（插件可扩展）
  const hookResult = await runBeforeInstallHook({
    builtinScan,  // ← 把内置扫描结果传给 hook
    ...
  });

  // Hook 优先：hook 说 block 就 block，否则看内置结论
  return hookResult?.blocked ? hookResult : builtinBlocked;
}
```

步骤 1 短路：依赖黑名单命中后直接返回，不浪费时间做代码扫描。

### before_install Hook：可扩展扫描策略

最有趣的是第三步。Clawdbot 把**内置扫描的完整结果**（包括所有 findings 的 severity/ruleId/message/evidence）传给 `before_install` hook：

```typescript
// src/plugins/install-security-scan.runtime.ts:609-648
async function runBeforeInstallHook(params: {
  builtinScan: BuiltinInstallScan;
  ...
}) {
  const hookRunner = getGlobalHookRunner();
  if (!hookRunner?.hasHooks("before_install")) {
    return undefined;  // 没有 hook，跳过
  }
  const { event, ctx } = createBeforeInstallHookPayload({
    builtinScan: params.builtinScan,  // 把扫描结果暴露给 hook
    ...
  });
  const hookResult = await hookRunner.runBeforeInstall(event, ctx);
  if (hookResult?.block) {
    // hook 决定拦截，使用 hook 提供的 reason
    return { blocked: { reason: hookResult.blockReason || "..." } };
  }
}
```

这意味着企业用户可以编写自定义安全策略（比如"只允许安装经过我们内部 GPG 签名的插件"），而不需要 fork Clawdbot 修改核心代码。内置扫描结果作为上下文传入，hook 可以基于它做增量决策（"这个 warn 级别的发现在我们的环境是 critical"）。

### 逃生舱口：dangerouslyForceUnsafeInstall

```typescript
// src/plugins/install-security-scan.runtime.ts:503-516
if (params.dangerouslyForceUnsafeInstall) {
  return undefined;  // 跳过拦截，但继续走 hook
}
```

这是一个**有意设计的逃生舱口（escape hatch）**，但不是静默的。强制安装时，必须记录警告：

```typescript
// src/plugins/install-security-scan.runtime.ts:542-548
if (params.dangerouslyForceUnsafeInstall && params.builtinScan.critical > 0) {
  logDangerousForceUnsafeInstall({
    findings: params.builtinScan.findings,
    logger: params.logger,
    targetLabel: params.targetLabel,
  });
}
```

设计原则：**不要让安全逃生舱口变成沉默的后门**。任何绕过行为都要留下可审计的记录。

---

## 第四层：运行时路径边界（Boundary Path）

**核心文件**: `src/infra/boundary-path.ts`, `src/infra/boundary-file-read.ts`

前三层保护安装阶段。但插件代码一旦被加载执行，加载过程本身也是攻击面——恶意插件可能通过构造的模块路径做**路径穿透（Path Traversal）**，读取插件沙箱外的文件。

OWASP 将路径穿透列为 CWE-22（Improper Limitation of a Pathname to a Restricted Directory）[[OWASP]](https://owasp.org/www-community/attacks/Path_Traversal) [[CWE-22]](https://cwe.mitre.org/data/definitions/22.html)。常见手法包括：

- `../../etc/passwd`（上层目录穿透）
- `plugin-root/safe.ts -> /etc/passwd`（符号链接穿透）
- `plugin-root/hardlink`（硬链接穿透）

Clawdbot 用 `openBoundaryFileSync` 统一处理这些攻击面：

```typescript
// src/infra/boundary-file-read.ts:69-92
export function openBoundaryFileSync(params: OpenBoundaryFileSyncParams): BoundaryFileOpenResult {
  const resolved = resolveBoundaryFilePathGeneric({
    absolutePath: params.absolutePath,
    resolve: (absolutePath) =>
      resolveBoundaryPathSync({
        absolutePath,
        rootPath: params.rootPath,         // 插件根目录
        rootCanonicalPath: params.rootRealPath,
        boundaryLabel: params.boundaryLabel,
        skipLexicalRootCheck: params.skipLexicalRootCheck,
      }),
  });
  ...
  return finalizeBoundaryFileOpen({
    resolved,
    rejectHardlinks: params.rejectHardlinks,  // 默认 true，拒绝硬链接
    ...
  });
}
```

返回值类型是一个 Result 类型（而非 throw）：

```typescript
// src/infra/boundary-file-read.ts:28-32
export type BoundaryFileOpenResult =
  | { ok: true; path: string; fd: number; stat: fs.Stats; rootRealPath: string }
  | { ok: false; reason: BoundaryFileOpenFailureReason; error?: unknown };
```

三类失败原因：
- `"path"` — 路径逃出了根目录边界
- `"validation"` — 内部验证错误（异步/同步混用等）
- `"io"` — 文件系统读取错误

`matchBoundaryFileOpenFailure` 提供模式匹配 API，调用方可以为每种失败情况提供不同处理逻辑 [boundary-file-read.ts:94-112]。

### 在 module-loader 中的应用

```typescript
// src/channels/plugins/module-loader.ts:73-101
export function loadChannelPluginModule(params: {
  modulePath: string;
  rootDir: string;
  boundaryRootDir?: string;
  ...
}): unknown {
  // 先过边界检查
  const opened = openBoundaryFileSync({
    absolutePath: params.modulePath,
    rootPath: params.boundaryRootDir ?? params.rootDir,
    boundaryLabel: params.boundaryLabel ?? "plugin root",
    rejectHardlinks: false,  // ← 注意：channel plugin 允许硬链接
    skipLexicalRootCheck: true,
  });
  if (!opened.ok) {
    throw new Error(
      `${params.boundaryLabel ?? "plugin"} module path escapes plugin root or fails alias checks`,
    );
  }
  const safePath = opened.path;
  fs.closeSync(opened.fd);

  // 再做 Jiti 加载
  return loadModule(safePath)(safePath);
}
```

注意 `rejectHardlinks: false`——channel 插件明确允许硬链接（注释 [module-loader.ts:85]），但文件范围仍受 boundary 限制。这是一个**有意识的权衡决策**，说明不同场景可以共用同一套边界 API，但配置不同策略。

---

## 第五层：Jiti 运行时加载器与 SDK 别名隔离

**核心文件**: `src/plugins/jiti-loader-cache.ts`, `src/plugins/sdk-alias.ts`

最后一层是加载机制本身。Clawdbot 允许插件以 TypeScript 源码形式存在，运行时通过 [Jiti](https://github.com/unjs/jiti) 直接执行（无需预编译）。Jiti 是 UnJS 团队维护的运行时 TypeScript/ESM 支持库，被 Nuxt、Tailwind CSS、ESLint 等广泛使用 [[npm]](https://www.npmjs.com/package/jiti)。

### 问题：SDK 版本冲突

当插件 `import { PluginSDK } from "@openclaw/sdk"` 时，如果插件本地安装了和宿主不同版本的 SDK，可能出现多个 SDK 实例并存的问题（类似 React 的"multiple React instances"问题）。

### 解决方案：别名映射

Clawdbot 在 Jiti 配置时注入 SDK 别名映射（alias map），强制插件对 SDK 的引用重定向到宿主的 SDK 路径：

```typescript
// src/plugins/jiti-loader-cache.ts:1-60
export function getCachedPluginJitiLoader(params: {
  cache: PluginJitiLoaderCache;
  modulePath: string;
  ...
  aliasMap?: Record<string, string>;
  tryNative?: boolean;
}) {
  const { tryNative, aliasMap } = resolvePluginLoaderJitiConfig({...});

  // 缓存键包含 aliasMap 哈希，不同别名配置各自独立缓存
  const cacheKey = createPluginLoaderJitiCacheKey({ tryNative, aliasMap });
  const scopedCacheKey = `${params.jitiFilename ?? params.modulePath}::${...cacheKey}`;

  const cached = params.cache.get(scopedCacheKey);
  if (cached) {
    return cached;  // 复用已有 Jiti 实例
  }

  const loader = createJiti(params.jitiFilename ?? params.modulePath, {
    ...buildPluginLoaderJitiOptions(aliasMap),  // 注入别名
    tryNative,
  });
  params.cache.set(scopedCacheKey, loader);
  return loader;
}
```

### SDK 根目录可信验证

在构建 alias map 之前，代码先验证宿主 SDK 根目录的可信标识：

```typescript
// src/plugins/sdk-alias.ts:54-74
function hasTrustedOpenClawRootIndicator(params: {
  packageRoot: string;
  packageJson: PluginSdkPackageJson;
}): boolean {
  const hasPluginSdkRootExport = Object.prototype.hasOwnProperty.call(
    packageExports, "./plugin-sdk"
  );
  if (!hasPluginSdkRootExport) {
    return false;
  }
  // 必须同时满足：有 cli-entry 导出 OR openclaw bin OR openclaw.mjs 入口文件
  const hasCliEntryExport = ...;
  const hasOpenClawBin = ...;
  const hasOpenClawEntrypoint = fs.existsSync(
    path.join(params.packageRoot, "openclaw.mjs")
  );
  return hasCliEntryExport || hasOpenClawBin || hasOpenClawEntrypoint;
}
```

只有同时具备 `./plugin-sdk` 导出**且**包含官方 CLI 入口标识的包，才会被信任为 SDK 根目录。防止攻击者创建一个假的 "openclaw SDK 包"，让插件的 SDK 别名映射到恶意代码。

### tryNative：优化加载路径

`tryNative: true` 时，Jiti 会先尝试用 Node.js 原生 ESM 加载，失败才降级到 Jiti 的 TypeScript transpile 路径。对于已编译的 `.js`/`.mjs` 文件，原生加载更快、更稳定。代码在 Windows 上还有特殊处理（`module-loader.ts:94-100`），尝试 `require()` 作为第三选择。

---

## 设计全景：防御纵深的层次分析

把五层拼起来，可以画出完整的防御图：

```
安装阶段
│
├── [Layer 1] 依赖黑名单 (dependency-denylist.ts)
│   └── 拦截：known-bad 包、alias 绕过、symlink 绕过、vendored 嵌套
│
├── [Layer 2] 静态代码扫描 (skill-scanner.ts)
│   └── 拦截：shell exec、eval/Function、crypto mining、数据外泄、凭证收集
│
├── [Layer 3] 安装管道编排 (install-security-scan.runtime.ts)
│   ├── 拦截：扫描失败 → 阻止安装
│   └── 扩展：before_install hook 支持自定义策略
│
运行时阶段
│
├── [Layer 4] 路径边界 (boundary-path.ts + boundary-file-read.ts)
│   └── 拦截：路径穿透、符号链接逃逸、硬链接（可配置）
│
└── [Layer 5] Jiti SDK 别名隔离 (jiti-loader-cache.ts + sdk-alias.ts)
    └── 拦截：SDK 版本冲突、伪造 SDK 根目录
```

每层的拦截对象不同，失效模式也不同——第一层无法拦截内联恶意代码（不依赖任何包），第二层无法拦截运行时动态生成的危险调用。这就是为什么需要纵深防御，而不是把所有逻辑堆进一个大扫描器。

---

## 扫描边界的诚实代价

这套系统也有明确的局限：

1. **动态分析缺失**：正则扫描只能看到静态文本，无法追踪运行时的 `eval(atob(encodedPayload))` 这类先解码后执行的模式。当然，`obfuscated-code` 规则尝试用 base64 长度 / hex 序列做启发式检测 [skill-scanner.ts:186-203]。

2. **500 文件上限**：大型插件会被截断扫描。这是性能和覆盖率之间有意识的权衡——`extensionUsesSkippedScannerPath` 为隐藏目录/node_modules 中的声明的 entry point 做了补偿性强制扫描（force scan）[install-security-scan.runtime.ts:740-745]。

3. **单一黑名单**：`plain-crypto-js` 是目前唯一的被封锁包。黑名单本质上是对**已知**威胁的响应，新型供应链攻击（首次出现的包）无法通过黑名单拦截，必须依赖第二层静态代码扫描。

4. **Jiti 不是 VM 沙箱**：Jiti 解决 TypeScript 加载和 SDK 别名问题，但插件代码和宿主进程共享同一 Node.js 运行时。真正的进程级沙箱需要额外的 Docker/seccomp 层，这属于另一套系统（`src/agents/sandbox/`）。

---

## 可以学到什么

### 1. 防御纵深不是冗余，而是正交

每层覆盖不同的威胁向量。依赖黑名单拦截 already-identified threats，静态扫描拦截 novel patterns，路径边界拦截 runtime exploitation。单一超级扫描器不如多个专注的轻量层。

### 2. false positive 是设计变量，不是缺陷

`requiresContext: /child_process/` 的设计明确接受"漏报部分不使用 child_process 的 eval"，换取"不误报自定义 exec 函数"。这个权衡应该是显式的，文档化的，可调整的。

### 3. 逃生舱口必须留，但不能沉默

`dangerouslyForceUnsafeInstall` 的设计哲学：**不要让安全机制成为工程师的敌人**，但一切绕过都必须可审计。强制安装一定要记录 findings，这比"根本没有逃生舱口，工程师自己 fork 代码绕过"安全得多。

### 4. Hook 是可扩展性的正确抽象

`before_install` hook 把**内置扫描结果**暴露给自定义策略，而不是让自定义策略接管整个扫描流程。这样既保证了基础安全基线（内置扫描总是跑），又允许企业根据自己的风险偏好做增量决策。

### 5. 结果类型优于异常

`BoundaryFileOpenResult = ok | failure` 强制调用方处理失败情况，而不是假设"文件一定存在且合法"。在安全关键路径上，Result 类型比 try/catch 更不容易被忽略。

---

## 源码索引

| 文件 | 行号 | 说明 |
|------|------|------|
| `src/plugins/dependency-denylist.ts` | 1 | 黑名单包名常量 |
| `src/plugins/dependency-denylist.ts` | 44-57 | case-insensitive 双集合查找 |
| `src/plugins/install-security-scan.runtime.ts` | 163-201 | symlink target 穿透检测 |
| `src/plugins/install-security-scan.runtime.ts` | 233-237 | 遍历深度/数量限制常量 |
| `src/plugins/install-security-scan.runtime.ts` | 319-321 | 注释：为何主动遍历 node_modules |
| `src/plugins/install-security-scan.runtime.ts` | 487-518 | `buildBlockedScanResult`：扫描决策函数 |
| `src/plugins/install-security-scan.runtime.ts` | 503-504 | dangerouslyForceUnsafeInstall 逃生舱口 |
| `src/plugins/install-security-scan.runtime.ts` | 542-548 | 强制安装时的强制日志 |
| `src/plugins/install-security-scan.runtime.ts` | 609-648 | `runBeforeInstallHook` + 内置结果传递 |
| `src/plugins/install-security-scan.runtime.ts` | 652-705 | `scanBundleInstallSourceRuntime` 三步管道 |
| `src/security/skill-scanner.ts` | 39-48 | SCANNABLE_EXTENSIONS 集合 |
| `src/security/skill-scanner.ts` | 50-53 | 扫描上限和缓存大小常量 |
| `src/security/skill-scanner.ts` | 147-173 | LINE_RULES 定义 |
| `src/security/skill-scanner.ts` | 177-205 | SOURCE_RULES 定义（含 requiresContext）|
| `src/security/skill-scanner.ts` | 221-258 | `scanSource` 主逻辑（LINE_RULES 迭代）|
| `src/security/skill-scanner.ts` | 263-308 | `scanSource` SOURCE_RULES 处理 |
| `src/security/scan-paths.ts` | 4-8 | `isPathInside`：路径边界检查 |
| `src/security/scan-paths.ts` | 35-42 | `extensionUsesSkippedScannerPath` |
| `src/infra/boundary-file-read.ts` | 28-32 | `BoundaryFileOpenResult` Result 类型 |
| `src/infra/boundary-file-read.ts` | 69-92 | `openBoundaryFileSync` 入口 |
| `src/infra/boundary-file-read.ts` | 94-112 | `matchBoundaryFileOpenFailure` 模式匹配 |
| `src/channels/plugins/module-loader.ts` | 73-101 | `loadChannelPluginModule`：边界检查 + Jiti |
| `src/plugins/jiti-loader-cache.ts` | 8-10 | Jiti 类型导出 |
| `src/plugins/jiti-loader-cache.ts` | 45-59 | 缓存键构造 + 复用逻辑 |
| `src/plugins/sdk-alias.ts` | 54-74 | `hasTrustedOpenClawRootIndicator` |
| `src/plugins/sdk-alias.ts` | 42-51 | `isSafePluginSdkSubpathSegment` 路径验证 |

---

## 延伸阅读

- [OWASP Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal) — 路径穿透攻击原理与防御
- [CWE-22: Improper Limitation of a Pathname](https://cwe.mitre.org/data/definitions/22.html) — 路径限制不当的标准分类
- [PortSwigger: File Path Traversal](https://portswigger.net/web-security/file-path-traversal) — 交互式漏洞实验
- [GitHub: unjs/jiti](https://github.com/unjs/jiti) — Jiti 运行时加载器官方文档
- [JStill: Mostly Static Detection of Obfuscated Malicious JavaScript](https://www.cse.psu.edu/~sxz16/papers/JStill.pdf) — 混淆恶意 JS 检测论文
- [JaSt: Fully Syntactic Detection of Malicious JavaScript](https://swag.cispa.saarland/papers/fass2018jast.pdf) — 语法级恶意 JS 检测
- [Hiding in Plain Site: Detecting JavaScript Obfuscation](https://www.kapravelos.com/publications/jsobf-imc20.pdf) — JS 混淆检测研究
- [CISA: npm Ecosystem Supply Chain Compromise 2025](https://www.cisa.gov/news-events/alerts/2025/09/23/widespread-supply-chain-compromise-impacting-npm-ecosystem) — 2025 年 npm 大规模供应链事件
- [Microsoft: Mitigating the Axios npm supply chain compromise](https://www.microsoft.com/en-us/security/blog/2026/04/01/mitigating-the-axios-npm-supply-chain-compromise/) — Axios 攻击事件分析
- [npm Security Cheat Sheet | OWASP](https://cheatsheetseries.owasp.org/cheatsheets/NPM_Security_Cheat_Sheet.html) — npm 安全最佳实践
- [npq: Safely install npm packages](https://github.com/lirantal/npq) — 预安装审计工具参考
