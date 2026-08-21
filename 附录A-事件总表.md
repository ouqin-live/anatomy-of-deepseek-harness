# 附录 A 事件总表

本附录列出了 dsh 中所有 Cordis Typed Event，按功能域分组。

## A.1 Session 事件

| 事件 | Mode | Dispatch | 说明 |
|------|------|----------|------|
| `session/created` | emit | `ctx.emit()` | Session 创建并 announce 后触发 |
| `session/event` | emit | `ctx.emit()` | 每次事件 append 到 session log 时触发 |
| `session/flush` | parallel | `ctx.parallel()` | 要求持久化/telemetry 刷入存储 |
| `session/disposed` | emit | `ctx.emit()` | Session 被销毁后触发 |

## A.2 Agent 事件

| 事件 | Mode | Dispatch | 说明 |
|------|------|----------|------|
| `agent/created` | emit | `ctx.emit()` | Agent 创建并注册后触发 |
| `agent/disposed` | emit | `ctx.emit()` | Agent 被销毁后触发 |
| `agent/status` | emit | `ctx.emit()` | 状态在 idle ↔ running 间切换 |
| `agent/inbox/inserted` | emit | `ctx.emit()` | 消息进入 inbox |
| `agent/inbox/claimed` | emit | `ctx.emit()` | 消息从 inbox 移出（开始处理） |
| `agent/inbox/discarded` | emit | `ctx.emit()` | 消息被丢弃（未处理） |
| `agent/session-start` | emit | `ctx.emit()` | Session 第一次可用时触发 |
| `agent/pre-step` | waterfall | `ctx.waterfall()` | Step 开始前，可拒绝或替代消息 |
| `agent/request` | waterfall | `ctx.waterfall()` | LLM 请求构建后，可修改请求配置 |
| `agent/request-error` | waterfall | `ctx.waterfall()` | LLM 请求失败，可决定重试 |
| `agent/turn-stopping` | serial | `ctx.serial()` | Turn 即将关闭 |
| `agent/error` | emit | `ctx.emit()` | 不可恢复错误 |

## A.3 Tool 事件

| 事件 | Mode | Dispatch | 说明 |
|------|------|----------|------|
| `tools/pre-execute` | waterfall | `ctx.waterfall()` | 工具执行前决策关卡 |
| `tools/execute` | waterfall | `ctx.waterfall()` | 工具实际执行（around-dispatch） |
| `tools/post-execute` | waterfall | `ctx.waterfall()` | 工具结果后处理 |
| `tools/code-dispatch-log` | waterfall | `ctx.waterfall()` | Code-mode 子分发的日志替换 |
| `tools/result` | emit | `ctx.emit()` | 最终结果冻结后通知 |
| `tools/change` | emit | `ctx.emit()` | 工具注册表变化 |

## A.4 System Prompt 事件

| 事件 | Mode | Dispatch | 说明 |
|------|------|----------|------|
| `system-prompt/assemble` | waterfall | `ctx.waterfall()` | 每次 assembly 后，可修改 assembly |
| `system-prompt/change` | emit | `ctx.emit()` | 有 section/context/variable 注册或卸载 |

## A.5 LLM 事件

| 事件 | Mode | Dispatch | 说明 |
|------|------|----------|------|
| `llm/stream` | waterfall | `ctx.waterfall()` | 每个模型流式请求 |
| `llm/adapters-updated` | emit | `ctx.emit()` | Adapter 注册表变化 |

## A.6 FS 事件

| 事件 | Mode | Dispatch | 说明 |
|------|------|----------|------|
| `fs/write-intent` | waterfall | `ctx.waterfall()` | 文件写入意图 |
| `fs/edit-intent` | waterfall | `ctx.waterfall()` | 文件编辑意图 |
| `fs/observed` | emit | `ctx.emit()` | 文件已读/已写通知 |

## A.7 Subagent 事件

| 事件 | Mode | Dispatch | 说明 |
|------|------|----------|------|
| `subagent/provider-added` | emit | `ctx.emit()` | Subagent provider 注册 |
| `subagent/provider-removed` | emit | `ctx.emit()` | Subagent provider 卸载 |
| `subagent/start` | emit | `ctx.emit()` | 子代理运行开始 |
| `subagent/end` | emit | `ctx.emit()` | 子代理运行结束 |

## A.8 Approval 事件

| 事件 | Mode | Dispatch | 说明 |
|------|------|----------|------|
| `approval/request` | waterfall | `ctx.waterfall()` | 工具执行审批请求 |

## A.9 Settings 事件

| 事件 | Mode | Dispatch | 说明 |
|------|------|----------|------|
| `settings/updated` | emit | `ctx.emit()` | 配置变更通知 |
| `settings/document-updated` | emit | `ctx.emit()` | 配置文件变更通知 |

## A.10 Workflow 事件

| 事件 | Mode | Dispatch | 说明 |
|------|------|----------|------|
| `workflow/start` | emit | `ctx.emit()` | 工作流开始 |
| `workflow/end` | emit | `ctx.emit()` | 工作流结束 |
| `workflow/agent-start` | emit | `ctx.emit()` | Agent 进入工作流 |
| `workflow/agent-end` | emit | `ctx.emit()` | Agent 完成工作流 |
| `workflow/phase` | emit | `ctx.emit()` | 工作流阶段变化 |
| `workflow/log` | emit | `ctx.emit()` | 工作流日志事件 |

## A.11 其他事件

| 事件 | Mode | Dispatch | 说明 |
|------|------|----------|------|
| `credentials/updated` | emit | `ctx.emit()` | Credentials 更新 |
| `command/change` | emit | `ctx.emit()` | 命令配置变化 |
| `goal/changed` | emit | `ctx.emit()` | 目标状态变化 |
| `domain/changed` | emit | `ctx.emit()` | 存储域变化 |
| `skills/change` | emit | `ctx.emit()` | 技能库变化 |