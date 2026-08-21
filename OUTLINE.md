# 《解剖 DeepSeek Harness：一切皆插件的 Agent 运行时》大纲（修订版）

**定位**：上一本《解剖 Claude Code》讲的是"一个 while 循环及其护栏"；这本讲的是"没有特权内核的插件运行时"——Cordis 底座、组合树、能力接缝。全 MIT 开源，每个结论都可对照官方仓库行号。

**源码仓库**：deepseek-ai/deepseek-harness（浅克隆到本地 `/Users/ouqin/Public/code/claude/deepseek-harness`）

**版本策略**：锁定当前 commit（`141eb6f`），逐行对照。API 和包名可能随 v0.x 迭代变化，但插拔原则（可逆副作用、接缝、事件域）是稳定的。变化的部分在书里用脚注标注"此接口在更新版本中已变化"并做简要说明。

---

## 第一部分：底座

### 第 1 章 Harness 是什么（Agent = Model + Harness）
- v0.1 开发者预览版的定位：`npx @deepseek-ai/dsh web` 能做什么、不能做什么
- 仓库全景：pnpm monorepo，50+ `@deepseek-ai/dsh-*` 包，核心 vs 工具 vs 可选的分布
- 与 Claude Code / OpenAI Agents SDK / LangChain 的定位差异

### 第 2 章 Cordis 内核（Context / Service / Event / Fiber）
- 三种贡献物：service（能力提供）、typed event（类型安全的事件域）、reversible effect（可逆副作用）
- 注册即副作用：`ctx.plugin(CapabilityProvider)` → `ctx.sandboxed(callback)` 自动回卷所有 effect
- Fiber 与异步上下文：`ctx` 上的 key 就是公共契约；无特权内核，任何一行都可以被 patch

### 第 3 章 组合树（Profile / Bundle / Patch）
- Profile 命名组合 → Bundle 分发格式 → `cordis.patch.yml` 分层覆盖
- `dsh --dump-config` 看真实启动树：`dsh-base` → `dsh-headless` / `dsh-web-app` 的叠加
- Settings 系统贯穿：`~/.dsh/settings.yaml` 热重载、`~/.dsh/.credentials.yaml` 凭据分离、每个插件的 settings schema
- 配置合并优先级：CLI flag > settings.yaml > bundle 默认 > profile 默认

## 第二部分：主循环

### 第 4 章 会话日志（Session：Append-Only Event Log）
- `SessionEvent` 不可变流的存储模式：任何操作（包括标题生成、hook）都 append 到同一条日志
- `deriveMessages()` = 日志 → 模型历史；fork / resume / 回放共用同一组事件作为起点
- 运行时断言："model-visible means logged"——模型所见的一切（包括 system prompt）都记录在那条日志里

### 第 5 章 主循环（Agent Loop：Turn / Step）
- Turn（一次用户输入=一个 turn）与 Step（turn 内每次 LLM 请求=一个 step）
- `agent/pre-step` → `agent/request` → `llm/stream` → `tool/call*` → `agent/turn-stopping` 完整时序
- Waterfall 事件（必须 `next()` 执行者）与普通事件的区别
- Inbox 的唤醒与等待语义：`ctx.inbox.makeFuture()` → 工具异步等待用户确认

### 第 6 章 工具系统（Tools：注册表与执行管线）
- 作用域工具注册表：`ctx.tools.register(scope, schema)` 以 scope 隔离
- `tools/pre-execute` → `execute` → `tools/post-execute` 三段管线；guard 审批与权限预设
- 模型侧工具声明如何从 schema 汇入 system prompt；带参工具与无参工具的批量组装

### 第 7 章 提示词与模型（System Prompt / LLM Adapter）
- Prompt 分节组装：system prompt 分段、tool schemas 注入、plan mode 前缀、skill catalog 加载
- `ctx.llm` 适配器接缝：provider 路由、`llm/stream` 事件透传、`llm/stream-error` 传播
- Moderation 检查层：`ctx.moderation.check()` 作为 pre-request/post-response 桩
- token-meter 计数与 dsh-timeout 截止时间写入

### 第 8 章 LLM Provider 适配器（llm-pi-ai 深度解读）
- Provider 声明形式：手写路由键 + `apiKeyEnv` + `baseURL` + `compat` 兼容开关
- Compat 层详解：`supportsDeveloperRole`（developer→system 降级）、`maxTokensField`（`max_tokens` vs `max_completion_tokens`）、`api`（`openai-completions`/`openai-responses` vs anthropic）、model 列表声明的 YAML 格式
- Attribution 机制：硬编码 `APP_IDENTITY`、`requestHeaders()` 中 "nothing can suppress attribution entirely"、归因头与配置 headers 的 case-insensitive 合并（proxy 实战中遇到的 x-stainless-* 指纹穿透问题）
- 休眠挂载（dormant）：settings.yaml 配置 provider → 实时激活，无需改 bundle
- Credentials 分离、多 key 轮转与 settings 热重载

## 第三部分：能力子系统

### 第 9 章 能力接缝（Capability Seam：FS / Shell / Sandbox）
- 三角色定义：Service Definition（接口契约）+ Provider（实现）+ Consumer（调用方）
- 文件系统与子进程共享同一执行世界：换一个 provider，Bash、PTY、workspace 整体迁移
- 实战：workspace write 访问模式（standard vs code mode 的读写围栏）

### 第 10 章 子代理（Subagent / Agent Team）
- 一个接口背后的多种 provider：子进程子代理、同产品委托 turn
- 实验性的 Agent Teams：roster（花名册）、task board（看板）、mailbox（邮箱）
- Goal/Ralph/Workflow 工具的多 agent 编排：goal 跨轮持久目标、ralph fresh-agent 循环、workflow 脚本化编排

### 第 11 章 上下文工程（Context / Compaction / Spill）
- Attachment（附件）与 context injection（上下文注入按钮，Web UI 里 @ 符号那个按钮）
- Compaction 压缩：什么时候触发、什么被裁减、损失了哪些信息
- Context spill 上下文溢出检测：token 计数与降级策略

### 第 12 章 持久化与存储（Store / Fork / Resume / Trajectory）
- Storage 插件化：内存 store、文件 store、自定义后端；session 序列化与反序列化
- Session query：从 session 事件流中检索
- Fork（从某条事件分叉）与 resume（从某条事件恢复）
- Trajectory 视图：单一事件流的搜索、回放、可视化

## 第四部分：实战与思想

### 第 13 章 运行形态（Web UI / Headless / Python SDK）
- 三种运行形态的 Canvas 选择：`dsh-web-app` bundle 提供 Web GUI（端口 3080）、`dsh-headless` profile 无端口无 UI、python SDK JSON-RPC over stdio
- 实际使用侧重点：headless 适合脚本化/CI、Web UI 适合交互式调试、SDK 适合集成到现有系统
- Model-agnostic 的校验回路：任何形态都可以接入相同的事件域和插件系统

### 第 14 章 写一个插件（Cookbook 实战）
- adding-a-tool：从注册 schema 到模型调用再到执行，完整的工具插件生命周期
- adding-an-llm-adapter：自定义 provider 适配器如何接入 `ctx.llm`
- 自定义 bundle 与 patch 覆盖官方行：实践 Cordis 的可逆副作用

### 第 15 章 设计思想与对比（Seam / 事件域 / 可逆副作用）
- 三层事件域的选择：durable session 事件（append-only）、live agent 事件（waterfall）、capability 事件（瞬时）
- 与 Claude Code 逐点对照：主循环形状（while loop vs event waterfall）、工具管线、子代理
- "一切皆插件"的代价：配置是代码，调试是读组合树
- Attribution 与 telemetry 的设计哲学：强制归因 vs 白箱策略

## 附录

- 附录 A：Mini Harness（300 行可运行的极简 Cordis + 循环，无 API key）
- 附录 B：事件总表（按事件域分组，producer / consumer 对照）
- 附录 C：配置目录速查（config catalog）

---

**固定体例**：每章 = 源码导读 + 真实 TypeScript 代码块 + 逐点平实解释 + 四道思考题；不写比喻。

**待定事项**：
1. 书名——沿用"解剖"系列，当前用的《解剖 DeepSeek Harness：一切皆插件的 Agent 运行时》待确认。
2. 也可用子标题单独命名，主标题保留《解剖 DeepSeek Harness》。封面设计偏向延续上一本风格。