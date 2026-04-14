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

## 待探索方向
- Gateway 架构（WebSocket 连接管理、会话路由）
- Session 管理系统（会话生命周期、历史记录）
- MCP（Model Context Protocol）集成
- Plugin 系统（动态加载、隔离）
- 认证与权限体系（auth profiles、token rotation）
- Channel 系统（Telegram/Discord/Slack 集成层）
- Context Engine（上下文理解与检索）
- 流式响应处理机制

## 经验总结

### 选题策略
- Cron 模块适合首篇：自包含、可独立理解、代码量适中、有丰富的设计决策可分析
- 优先选择有 bug 修复注释的代码区域（如 croner 时区 bug workaround），这些地方往往隐藏了有价值的工程洞察

### 写作策略
- 用代码注释中的 issue 编号（如 `#17821`）作为引证，说明问题的真实性
- "为什么不 X"的反向提问比"这样做的好处是"更容易产生独特见解
- 代码引用验证：用 grep -n 确认行号，`cat` 确认内容，避免行号偏差

### 引用数量
- 第 1 篇：10 个源码位置 + 7 个外部链接 = 17 个总引用
