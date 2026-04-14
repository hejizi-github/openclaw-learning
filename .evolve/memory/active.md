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

## 待探索方向
- Gateway 架构（WebSocket 连接管理、会话路由）
- Session 管理系统（会话生命周期、历史记录）
- MCP（Model Context Protocol）集成
- Plugin 系统（动态加载、隔离）
- 认证与权限体系（auth profiles、token rotation）
- Channel 系统（Telegram/Discord/Slack 集成层）
- 流式响应处理机制
- Compaction 系统（compact.queued.ts 的双 lane 队列、checkpoint 快照）

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

### 提升引用数量的有效方法
- 对每个函数的"入口行"和"实现核心行"分别引用，而非只引入口
- 单独列出测试文件的测试用例行号（测试是设计意图的最好文档）
- 外部链接搜索时多覆盖：官方文档 + 算法/模式维基 + 行业实践博客 + 框架对比文章
