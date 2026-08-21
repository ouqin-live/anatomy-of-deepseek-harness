# 第 1 章 Harness 是什么（Agent = Model + Harness）

## 1.1 什么是 Agent

在 2024-2025 年间，社区对 Agent 的定义经历了从"LLM + 工具调用"到"LLM + 循环 + 护栏"的转变。一个 Agent 不仅仅是模型能够调用外部函数，而是：模型在一个包含工具、文件系统、子进程、代码执行、用户审批等基础设施的"环境"中自主地、循环地执行任务，直到得出可接受的输出。

Claude Code 是这个模式的典型代表。当用户输入"帮我排查这个 bug"，Claude Code 会：调用工具搜索代码 → 阅读文件 → 运行测试 → 分析栈跟踪 → 可能再写代码 → 再测试。这个过程有循环、有工具调用、有状态管理。它是一个 Agent。

dsh（DeepSeek Harness）是把这个模式**形式化为一个可组合、可测试的运行时**。它不是某个具体的 Agent 应用，而是一个构造 Agent 的框架。

## 1.2 拆成两半

如果将 Agent 视为输入输出函数，它可以被拆成两半：

```
Agent = Model + Harness
```

**Model** 指 LLM——能够在给定提示词和消息列表后输出文本和工具调用序列的推理引擎。模型可以是 Claude Opus 4、Claude Sonnet、DeepSeek 等任何提供 `POST /chat/completions` 接口的推理服务。

**Harness（马具、缰绳）** 指所有让模型能够与环境交互的东西：
- 工具注册、参数解析、执行调度、结果回写
- 文件系统访问（读、写、编辑）
- Shell/Bash 执行
- 子代理衍生
- 上下文管理（压缩、溢出）
- 会话持久化和恢复
- 审批策略
- 用户界面或 SDK 接入

dsh 的定位就是实现这个 Harness 层。Model 是可插拔的，Harness 是固定的。

## 1.3 三个核心原则

dsh 的代码库近年来有数万行提交，但最核心的设计原则只有三个：

**原则一：事件溯源（Event Sourcing）。** 每个模型请求、每个工具结果、每个轮次边界都作为不可变事件写入 session 日志。模型可见的历史消息从事件日志中算出来（`deriveMessages()`），而不是存储为一个可变数组。这使得回放（fork/resume）不需要外部快照。

**原则二：能力接缝（Capability Seam）。** 所有与环境交互的能力（文件读写、Shell 执行、子进程催生）都封装在 Cordis Service 中，通过 Typed Event 暴露钩子。每个"接缝"是一个可独立替换的插件。FS 接缝、Shell 接缝、Subagent 接缝、Approval 接缝——每个都有自己的事件边界（`fs/write-intent`、`tools/pre-execute`、`approval/request` 等）。插件在接缝边界上注册监听器，而不是修改主循环代码。

**原则三：没有特权内核（No Privileged Kernel）。** 主循环（`ReactLoopAgent`）本身也是一个插件。它是一个实现 `Agent` 接口的 `AgentLoop` Service。平时它充当 Agent 的默认实现，但完全可以通过另一个循环插件替换——比如用 `schedule` 插件实现定时驱动的 Agent 循环，或者用 `plan-mode` 插件实现计划驱动的执行。即使是闭环核心逻辑，框架也不把它放在不可替换的位置上。

## 1.4 仓库布局

dsh 的仓库采用 `pnpm` monorepo，包数超过 50 个。这些包按功能域分布在不同目录下：

```
packages/
├── core/           # 核心：session、agent-loop、agent、tools、system-prompt、llm、scope
├── preset/         # 预置配置：agent-presets、deployments
├── boot/           # 启动加载器：bootstrap、loader
├── fs/             # 文件系统接缝：fs、tool-fs、tool-str-replace-editor 等
├── terminal/       # Shell 接缝：terminal-bash
├── subagent/       # 子代理接缝：subagent、tool-subagent 等
├── sandbox/        # 沙箱接缝：sandbox-policy
├── context/        # 上下文注入：agent-instructions、session-reference、time-context 等
├── compaction/     # 上下文压缩：compaction、compaction-basic
├── storage/        # 持久化：storage-domain、storage-fs、storage-indexdb 等
├── session/        # 会话扩展：session-persistence、session-title、session-telemetry 等
├── interaction/    # 人机交互：commands、user-approval、permission-presets 等
├── plan/           # 计划模式：plan-mode
├── llm/            # LLM 扩展：llm-retry、token-meter 等
├── skill/          # 技能：skill、tool-skill 等
├── goal/           # 目标驱动：goal、goal-round-driver
├── workflow/       # 工作流：workflow、tool-workflow
├── jobs/           # 任务队列：tool-jobs
├── hooks/          # Hook 平台：hook-protocol、hooks-claude-code、hooks-codex
├── credit/         # 信用额度：credit-simple
├── extensions/     # 扩展基础：tool-cordis、cordis-host-runner
├── acp/            # Agent Communication Protocol
├── test-support/   # 测试工具：loader-smoke、llm-replay
└── settings/       # 配置系统：settings
```

53 个包（commit `141eb6f` 当时）。每个包的入口是一个继承 `Service` 的类，依赖通过 `ctx.inject()` 声明，通过 Cordis 自动装配。

## 1.5 如何使用

dsh 的使用者通过一个配置文件（如 `cordis.patch.yml`）声明要加载哪些插件，以及每个插件的配置。启动命令：

```bash
dsh start --config cordis.patch.yml
```

dsh 读取配置文件后，实例化一个 Cordis App（即组合根），按 `inject` 依赖顺序加载插件，构造 Service 实例。所有 Service 实例就位后，Session 和 Agent 被创建，用户可以通过内置的 Web UI（默认 `localhost:3914`）或 Python SDK 与之交互。

这个"声明式组合"的模型——不是写 main 函数直接实例化和组装代码，而是写一个 YAML 文件描述用哪些插件——是全书的起点，也是第 3 章的主题。

## 1.6 读源码的起点

本书所有源码引用均基于 commit `141eb6f`（2026 年 8 月）。dsh 仓库根目录下有 `docs/` 目录，包含架构总览、Cordis 入门、工具执行管线等若干篇精炼文档。建议读者先快速浏览 `docs/architecture.md` 获取全局图景，然后按本书章节顺序深入各子系统。

## 思考题

1. "Agent = Model + Harness" 这个等式中，Harness 的边界在哪里？Runtime、LLM adapter、tools 是否都属于 Harness？
2. "没有特权内核"对插件生态有什么影响？如果一个插件注册了与默认 AgentLoop 冲突的监听器，最终行为由谁决定？
3. dsh 的包数量超过 50，一个只跑在浏览器的 IDE 插件需要加载所有包吗？哪些包是可选的？
4. 事件溯源作为第一个原则，它与"能力接缝"之间是否存在矛盾——事件日志是中心化的，但能力接缝是去中心化的？