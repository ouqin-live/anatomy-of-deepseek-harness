# 解剖 DeepSeek Harness：一切皆插件的 Agent 运行时

一本逐行对照源码、拆解 [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) 架构的技术书。

**定位**：《解剖 Claude Code》讲的是"一个 while 循环及其护栏"；这本讲的是"没有特权内核的插件运行时"——Cordis 底座、组合树、能力接缝。

## 目录

| 章节 | 主题 |
| --- | --- |
| 第一部分：底座 | |
| [第 1 章 Harness 是什么](第01章-Harness是什么.md) | Agent = Model + Harness；仓库全景与定位差异 |
| [第 2 章 Cordis 内核](第02章-Cordis内核.md) | Context / Service / Event / Fiber；可逆副作用 |
| [第 3 章 组合树](第03章-组合树.md) | Profile / Bundle / Patch；配置合并优先级 |
| 第二部分：主循环 | |
| [第 4 章 会话日志](第04章-会话日志.md) | Session：Append-Only Event Log |
| [第 5 章 主循环](第05章-主循环.md) | Agent Loop：Turn / Step；Waterfall 事件 |
| [第 6 章 工具系统](第06章-工具系统.md) | Tools：注册表与执行管线 |
| [第 7 章 提示词与模型](第07章-提示词与模型.md) | System Prompt / LLM Adapter |
| [第 8 章 LLM Provider 适配器](第08章-LLM-Provider适配器.md) | llm-pi-ai 深度解读 |
| 第三部分：能力子系统 | |
| [第 9 章 能力接缝](第09章-能力接缝.md) | 能力边界的插件化设计 |
| [第 10 章 子代理](第10章-子代理.md) | Subagent |
| [第 11 章 上下文工程](第11章-上下文工程.md) | 上下文管理与组装 |
| [第 12 章 持久化与存储](第12章-持久化与存储.md) | Storage / Session 持久化 |
| [第 13 章 运行形态](第13章-运行形态.md) | Web / Headless / Python SDK |
| [第 14 章 写一个插件](第14章-写一个插件.md) | 从零实现插件 |
| [第 15 章 设计思想与对比](第15章-设计思想与对比.md) | 与 Claude Code / OpenAI Agents SDK / LangChain 对比 |
| 附录 | |
| [附录 A 事件总表](附录A-事件总表.md) | 全量事件一览 |
| [附录 B 配置目录速查](附录B-配置目录速查.md) | `~/.dsh` 目录结构 |
| [附录 C Mini-Harness](附录C-Mini-Harness.md) | 最小可运行实现 |

## 源码对照约定

- 源码仓库：deepseek-ai/deepseek-harness（浅克隆）
- 版本策略：锁定 commit `141eb6f`，逐行对照；变化的部分用脚注标注"此接口在更新版本中已变化"
- 每个结论都可对照官方仓库行号

## License

MIT
