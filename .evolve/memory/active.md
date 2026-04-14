# Active Memory

## 已覆盖主题
1. **Cron 调度系统**（`cron-scheduler-deep-dive.md`）
   - 三种调度模式：at/every/cron
   - 调度计算引擎（schedule.ts）
   - 确定性 Stagger 错峰机制
   - Timer 驱动引擎（armTimer/onTimer）
   - 隔离 Agent 执行（isolated-agent 子系统）
   - 错误退避 & 失败告警
   - 启动时 missed jobs 追赶

2. **Context Engine 可插拔上下文管理**（`context-engine-pluggable-architecture.md`）
   - ContextEngine 接口：完整生命周期 API（ingest/assemble/compact/maintain/bootstrap）
   - Registry + Symbol.for + globalThis：解决 ESM 多实例陷阱
   - 向后兼容 Proxy：用错误探测代替版本声明
   - LegacyContextEngine：空对象模式 + 委托压缩
   - foreground/background maintenance：lane 队列 + coalesce 调度
   - 插件注册权力边界（owner 机制）

3. **路由引擎：多渠道 Agent 会话身份解析**（`routing-engine-deep-dive.md`）
   - 会话键格式 `agent:<id>:<rest>` 作为可解析的分布式身份标识
   - 4 种 DM 隔离模式：main/per-peer/per-channel-peer/per-account-channel-peer
   - identity links：跨渠道身份融合（canonical name 映射）
   - 8 层优先级绑定匹配 tiers 数组（peer→parent-peer→wildcard→guild+roles→guild→team→account→channel）
   - 线程 parentPeer 继承机制（模拟 Discord 权限继承语义）
   - chat type 别名：group == channel 双向互查
   - 三层 WeakMap 缓存（agentLookup/evaluatedBindings/resolvedRoute）
   - 引用相等性作为缓存版本号（bindingsRef/agentsRef/sessionRef 对比）
   - 清空重置 LRU 替代方案（超过 MAX 直接 clear）

4. **Command Lane Queue 并发序列化引擎**（`command-lane-queue-concurrency-engine.md`）
   - 4 种内置 lane（main/cron/subagent/nested）+ 动态 session lane
   - `Symbol.for("openclaw.commandQueueState")` 进程级全局单例
   - `pump()` 事件循环友好调度循环（任务完成→递归触发下一个）
   - 双 lane 嵌套（session × global）防止压缩死锁；cron lane 自动降级 nested
   - generation 生代计数器：廉价乐观锁，处理 SIGUSR1 热重启残留 active task
   - Gateway Draining：markGatewayDraining → waitForActiveTasks → abortEmbeddedPiRun 三阶段关闭
   - Active Task Waiters：Observer 模式，快照语义（不等调用后新入队的任务）
   - 运行时 Schema Migration：getQueueState() 里内联字段补丁（v2026.4.2 兼容）
   - probe lane（auth-probe:/session:probe-）静默错误，不产生 error 级日志

5. **Compaction 系统——LLM 上下文压缩工程实践**（`compaction-system-context-compression.md`）
   - Token 估算：chars/4 启发式 + 1.2x 安全边距（SAFETY_MARGIN）
   - 自适应 chunk 比例：BASE_CHUNK_RATIO=0.4，当消息平均体积 > 10% 上下文时线性降至 MIN_CHUNK_RATIO=0.15
   - 工具调用配对不变式：pendingToolCallIds 集合确保 tool_call/tool_result 永远在同一 chunk
   - 三层摘要级联：summarizeInStages → summarizeWithFallback → 文字描述降级
   - 合并摘要指令优先保留"当前状态"而非"历史事件"（Agent 需知道自己在做什么）
   - 标识符保留策略（strict/off/custom）：通过 prompt engineering 约束 LLM 不乱改 UUID/哈希
   - SECURITY 双重过滤：toolResult.details 在 Token 估算和摘要生成两处路径都被剥离
   - 安全超时：EMBEDDED_COMPACTION_TIMEOUT_MS = 900_000ms（15分钟），支持 onCancel hook
   - 检查点快照：copy-then-modify，max 25 个/session，preCompaction + postCompaction 记录
   - 四种检查点原因：auto-threshold / overflow-retry / timeout-retry / manual
   - 真实对话过滤：NON_CONVERSATION_BLOCK_TYPES 排除 thinking/reasoning/toolCall 等块
   - 失败分类诊断：classifyCompactionReason 将错误文本关键词映射为结构化失败码

6. **ACP 网关桥接：跨协议 AI Agent 通信**（`acp-gateway-bridge-protocol-engineering.md`）
   - ACP 协议握手/能力协商：声明支持能力集（image/embeddedContext），拒绝 per-session MCP
   - 双层会话 ID 映射：ACP sessionId (UUID) ↔ Gateway sessionKey（结构化字符串），5 级优先级解析
   - 快照转增量重分段：`sentTextLength`/`sentThoughtLength` 游标，每次只取新增切片发送
   - generation 计数器断线处理：宽限窗口 5s，pre-ack vs post-ack 差异化处理，agent.wait 轮询恢复
   - 工具名三源交叉验证（_meta/rawInput/title 三源一致才接受），防名称注入
   - 8 级工具审批分类（readonly_scoped/readonly_search/mutating/exec_capable/control_plane/interactive/other/unknown）
   - CWD 路径遍历防护：覆盖 ../、file://、~/ 四种攻击面
   - provenance 三级（off/meta/meta+receipt），admin scope 拒绝自动降级重试
   - DoS 双层防护：逐块字节计数 + 全量二次检查，速率限制 120 会话/10s

## 待探索方向
- Gateway 架构（WebSocket 连接管理、会话路由）
- Session 管理系统（会话生命周期、历史记录）
- MCP（Model Context Protocol）集成
- Plugin 系统（动态加载、隔离）
- 认证与权限体系（auth profiles、token rotation）
- Channel 系统（Telegram/Discord/Slack 集成层）
- 流式响应处理机制（draft-stream-loop.ts）

## 经验总结

### 选题策略
- Cron 模块适合首篇：自包含、可独立理解、代码量适中、有丰富的设计决策可分析
- Context Engine 适合第二篇：接口 + 注册 + 兼容层的三层结构，能同时讲清楚设计模式和工程实践
- 优先选择有 bug 修复注释的代码区域（如 croner 时区 bug workaround），这些地方往往隐藏了有价值的工程洞察
- 带有多层设计模式的模块（Strategy + Registry + Proxy + Null Object）比单一模式的模块更值得深挖

### 写作策略
- 用代码注释中的 issue 编号（如 `#17821`）作为引证，说明问题的真实性
- "为什么不 X"的反向提问比"这样做的好处是"更容易产生独特见解
- 代码引用验证：用 grep -n 确认行号，`cat` 确认内容，避免行号偏差
- 在文章末尾用"可以学到什么"章节提炼可迁移的工程原则，让文章超越"clawdbot 源码解析"而具有普遍价值

### 引用数量
- 第 1 篇：10 个源码位置 + 7 个外部链接 = 17 个总引用
- 第 2 篇：15 个源码位置 + 7 个外部链接 = 22 个总引用（引用数量提升 29%）
- 第 3 篇：24 个源码位置 + 11 个外部链接 = 35 个总引用（引用数量提升 59%）
- 第 4 篇：26 个源码位置 + 11 个外部链接 = 37 个总引用（引用数量提升 6%）
- 第 5 篇：29 个源码位置 + 10 个外部链接 = 39 个总引用（引用数量提升 5%）
- 第 6 篇：30 个源码位置 + 11 个外部链接 = 41 个总引用（引用数量提升 5%）

### 提升引用数量的有效方法
- 对每个函数的"入口行"和"实现核心行"分别引用，而非只引入口
- 单独列出测试文件的测试用例行号（测试是设计意图的最好文档）
- 外部链接搜索时多覆盖：官方文档 + 算法/模式维基 + 行业实践博客 + 框架对比文章
