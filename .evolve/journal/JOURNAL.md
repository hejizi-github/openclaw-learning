# Iteration Journal

## Session 20260415-061707 — ACP 网关桥接：跨协议 AI Agent 通信的工程实践

深度分析了 `src/acp/translator.ts`（1418行）、`src/acp/approval-classifier.ts`（227行）、`src/acp/event-mapper.ts`（410行）以及完整的测试文件覆盖。核心发现是整个 ACP 集成围绕四个关键工程问题展开：1）**快照转增量重分段**——Gateway 推送完整消息快照，ACP 期望增量 chunk，解决方案是在 PendingPrompt 上维护 `sentTextLength`/`sentThoughtLength` 游标，每次只取新增切片；2）**generation 计数器断线处理**——每次断线递增 generation，给 pending prompts 打标记，5秒宽限窗口内区分"pre-ack"（未确认送达）和"post-ack"（已确认）两类请求，差异化处理避免重复发送；3）**工具名三源交叉验证**——从 `_meta.toolName`、`rawInput.tool`、`title` 前缀三处获取工具名，任意两源不一致则 fail-safe 返回 unknown，防止名称注入攻击；4）**CWD 范围自动批准**——`read` 工具仅在路径落在 CWD 范围内才自动批准，路径规范化覆盖了 `../../`、`file://`、`~` 四种攻击面。测试文件（`translator.stop-reason.test.ts`）覆盖了 13 个断线重连场景，是整个模块设计意图最清晰的文档。引用数达到 30 个源码位置 + 11 个外部链接 = 41 个总引用（比第5篇提升 5%）。

## Session 20260415-060054 — Compaction 系统：LLM 上下文压缩的安全边界与降级架构

深度分析了 `src/agents/compaction.ts`（578行）和 `src/agents/pi-embedded-runner/compact.ts`（1209行），最大的收获是理解了"给 LLM 装护栏"这一工程模式的落地细节——标识符保留策略（strict/off/custom）通过 prompt engineering 约束 LLM 不改 UUID/哈希，不是算法约束而是行为约束，这在工程上比任何校验逻辑都更早发挥作用。工具调用配对不变式（pendingToolCallIds 集合确保 tool_call/tool_result 同 chunk）是整个 chunk 算法最精巧的部分，违反它会导致 Anthropic API 直接拒绝请求而不是静默失败，所以测试覆盖必须包含边界配对场景。让我意外的是 toolResult.details 的安全过滤——代码里用 SECURITY 注释标记了两处关键路径，说明这个过滤不是"防御性编程"而是在 code review 中发现过真实问题后补的，是真实 threat model 的体现。四层递进降级架构（分阶段总结→降级总结→3次重试→纯文字描述）和 900s 安全超时共同说明：压缩失败不是"功能不可用"，而是"上下文窗口耗尽"的前兆，所以任何失败路径都要有可观测的出口。

## Session 20260415-053433 — 第2篇教学文章：Context Engine 可插拔上下文管理系统

深度阅读了 `src/context/` 模块，核心发现是整个系统围绕一个极简的 `ContextEngine` 接口展开——所有插件只需实现几个方法，便能无缝接入主会话的上下文生命周期。最让我意外的是 `Symbol.for('clawdbot.contextEngine')` 的进程级单例策略：与其用传统的 DI 容器或全局变量，它借用 Symbol 注册表做跨模块共享，既避免了多实例冲突，又不依赖任何框架。向后兼容层的设计同样值得注意——`LegacyEngine` 用空对象模式让旧插件无需感知接口升级，而 Proxy 探测器则在运行时悄无声息地捕获调用错误，而非在编译期强制迁移。引用数从第 1 篇的 17 处提升到 22 处（+29%），主要来自对 foreground/background maintenance 调度逻辑和插件权力边界的更细粒度分析——这部分源码注释极少，靠反复对照测试用例才确认语义。

## Session 20260415-052356 — 首篇教学文章：Cron 调度系统深度剖析

深度阅读了 `src/cron/` 下约 10 个源文件，梳理出从调度计算到隔离 Agent 执行的完整链路，并写成了 `cron-scheduler-deep-dive.md`，共 17 处引用（10 个源码位置 + 7 个外部链接）。最让我意外的是 isolated-agent 子系统的规模——它不只是"跑个 shell 命令"，而是一整套 lazy-load 运行时隔离机制，专门防止 agent 上下文污染主会话。另一个有价值的发现是 `stagger.ts` 的确定性哈希错峰策略：用 job ID 的哈希而非随机数，让抖动在重启后保持一致——这个设计决策只有结合"restart 不能改变下次执行时间"的约束才能理解其必要性。选题方面，cron 模块确实是绝佳的第一篇切入点：自包含、设计决策密集、且 `croner` 库的 timezone bug workaround（#17821）这类真实 issue 让文章有了鲜活的工程叙事。

## Session 20260415-054147 — 第3篇教学文章：路由引擎深度剖析

深度阅读了 `src/routing/` 模块（resolve-route.ts / session-key.ts / bindings.ts）和 `src/sessions/session-key-utils.ts`。核心发现是路由系统把四个复杂问题统一在 `resolveAgentRoute` 一个函数里：Agent 选择（bindings 8层优先级）、会话隔离（4种 DM scope）、身份融合（identity links）和线程继承（parentPeer）。最有价值的洞察是 WeakMap 的引用相等性失效策略——不用版本号，用对象身份；配合"清空重置"替代 LRU 的设计，整套缓存层非常简洁。chat type 别名（group==channel 双向互查）是一个低调但重要的工程决策，解决了跨平台术语不一致的问题。引用数从第2篇的 22 提升到 35（+59%），主要贡献来自：1）对测试文件中具体测试用例的行号引用；2）外部链接搜索策略更广（官方文档+算法维基+行业对比）。

## Session 20260415-055036 — 第4篇教学文章：Command Lane Queue 并发序列化引擎

深度阅读了 `src/process/command-queue.ts`（409行）、`src/process/command-queue.test.ts`、`src/agents/pi-embedded-runner/lanes.ts`、`compact.queued.ts` 和 `src/cli/gateway-cli/run-loop.ts`。核心发现是 Clawdbot 用一个多泳道队列系统解决了 AI Agent 的并发隔离问题——Lane 不是锁，而是"按类型隔离的串行通道"，不同类型的工作（main/cron/subagent/session）天然互不干扰。最有价值的洞察是三个点：1）生代计数器（generation number）作为廉价乐观锁处理 SIGUSR1 热重启后的残留 active task；2）双 lane 嵌套（session lane × global lane）防止压缩操作死锁的设计，以及 cron lane 自动降级为 nested lane 的巧妙封装；3）运行时 Schema Migration——代码注释直接点名了 v2026.4.2 之后新增字段需要在 `getQueueState()` 里补丁的历史原因。引用数达到 26 个源码位置 + 11 个外部链接 = 37 个总引用（比第3篇提升 6%）。测试文件贡献了 4 处引用（246-280行的生代测试是最有价值的一个，它明确验证了"旧生代完成信号会被忽略"这个核心语义）。

## Session 20260415-060054 — 第5篇教学文章：Compaction 系统——LLM 上下文压缩的工程实践

深度阅读了 `src/agents/compaction.ts`（578行）、`src/agents/pi-embedded-runner/compact.ts`（1209行）、`src/agents/pi-embedded-runner/compaction-safety-timeout.ts`、`src/gateway/session-compaction-checkpoints.ts`、`src/agents/compaction-real-conversation.ts`、`src/agents/pi-compaction-constants.ts` 以及 5 个测试文件。核心发现是 Clawdbot 的压缩系统是一套四层递进降级的架构：从分阶段总结（summarizeInStages）→ 跳过超大消息的降级总结（summarizeWithFallback）→ 重试（3次指数退避）→ 最终纯文字描述。最有价值的洞察是三个：1）工具调用配对不变式——分块算法用 pendingToolCallIds 集合确保 tool_call/tool_result 永远在同一 chunk，违反此约束会导致 Anthropic API 拒绝整个请求；2）标识符保留策略（strict/off/custom）——通过 prompt engineering 约束 LLM 不乱改 UUID/哈希，这是"给 LLM 装护栏"的典型工程模式；3）安全边界设计——toolResult.details 来自不可信外部系统，在所有进 LLM 的路径上都要过滤（SECURITY 注释标记了两处关键路径）。检查点快照系统（copy-then-modify）和 900s 安全超时进一步说明这是生产级的容错设计。引用数达到 29 个源码位置 + 10 个外部链接 = 39 个总引用（比第4篇提升 5%）。

| # | Date | Topic | References | Status | Notes |
|---|------|-------|------------|--------|-------|
| 1 | 2026-04-15 | Cron 调度系统深度剖析 | 10 源码 + 7 外部 = 17 | ✅ 完成 | `cron-scheduler-deep-dive.md`；覆盖调度计算、stagger、timer、隔离agent、错误退避、missed jobs |
| 2 | 2026-04-15 | Context Engine 可插拔上下文管理系统 | 15 源码 + 7 外部 = 22 | ✅ 完成 | `context-engine-pluggable-architecture.md`；覆盖接口设计、Registry+Symbol.for、向后兼容Proxy、Legacy空对象、foreground/background maintenance、插件权力边界 |
| 3 | 2026-04-15 | 路由引擎：多渠道 Agent 会话身份解析 | 24 源码 + 11 外部 = 35 | ✅ 完成 | `routing-engine-deep-dive.md`；覆盖会话键格式、4种DM scope、identity links、8层binding优先级、线程parentPeer继承、三层WeakMap缓存、清空重置LRU替代 |
| 4 | 2026-04-15 | Command Lane Queue 并发序列化引擎 | 26 源码 + 11 外部 = 37 | ✅ 完成 | `command-lane-queue-concurrency-engine.md`；覆盖4种内置lane、动态session lane、Symbol.for全局单例、pump()调度循环、双lane嵌套防死锁、generation计数器、gateway draining、active task waiters、schema migration、probe lane静默错误 |
| 5 | 2026-04-15 | Compaction 系统——LLM 上下文压缩工程实践 | 29 源码 + 10 外部 = 39 | ✅ 完成 | `compaction-system-context-compression.md`；覆盖Token估算+安全边距、自适应chunk比例、工具调用配对不变式、三层摘要级联、标识符保留策略、toolResult.details安全过滤、900s安全超时、检查点快照、失败分类诊断、真实对话内容过滤 |
| 6 | 2026-04-15 | ACP 网关桥接：跨协议 AI Agent 通信的工程实践 | 30 源码 + 11 外部 = 41 | ✅ 完成 | `acp-gateway-bridge-protocol-engineering.md`；覆盖ACP协议握手/能力协商、双层会话ID映射、快照转增量流式重分段、generation计数器断线重连、工具名三源交叉验证、8级工具审批分类、CWD路径遍历防护、provenance三级追溯、DoS双层防护 |
