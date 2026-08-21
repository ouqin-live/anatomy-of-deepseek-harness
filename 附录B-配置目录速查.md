# 附录 B 配置目录速查

本附录列出 dsh 的关键配置文件和目录路径。

## B.1 用户配置文件

| 路径 | 说明 |
|------|------|
| `~/.dsh/settings.yaml` | 用户级配置覆盖，优先级高于 bundle 默认值 |
| `~/.dsh/credentials.yaml` | 加密的 API Key 存储 |
| `~/.dsh/sessions/` | Session 持久化存储目录（当使用 storage-fs 时） |
| `~/.dsh/spill/` | Spill 溢出文件目录 |
| `~/.dsh/logs/` | 运行时日志目录（如果配置了文件日志） |

## B.2 项目级文件

| 路径 | 说明 |
|------|------|
| `cordis.patch.yml` | 项目根目录的 Patch 覆盖文件（运行时生效） |
| `.dsh/` | 工作区级别的 dsh 数据目录（如 workspace-specific sessions） |

## B.3 Bundle 和 Profile

| 路径 | 说明 |
|------|------|
| `bundles/@deepseek-ai/ds-agent-default/` | 默认 Agent bundle |
| `bundles/@deepseek-ai/ds-profile-standard/` | 标准 Profile |
| `bundles/@deepseek-ai/ds-profile-minimal/` | 最小 Profile（用于浏览器 IDE 插件） |

## B.4 Session 事件日志格式

当使用 `storage-fs` 时，session 日志存储在 `~/.dsh/sessions/<sessionId>.jsonl`。每行一个 JSON 对象，代表一个 `SessionEvent`：

```json
{"type":"turn/start","seq":0,"time":1712345678000,"data":{"turn":1}}
{"type":"user/message","seq":1,"time":1712345678005,"data":{...},"surfaceOp":"append"}
{"type":"step/start","seq":2,"time":1712345678010,"data":{"turn":1,"step":1}}
{"type":"request/header","seq":3,"time":1712345678015,"data":{...}}
{"type":"assistant/chunk","seq":4,"time":1712345678500,"data":{...}}
{"type":"assistant/message","seq":5,"time":1712345678800,"data":{...},"surfaceOp":"append"}
{"type":"step/end","seq":6,"time":1712345678810,"data":{"turn":1,"step":1}}
{"type":"turn/end","seq":7,"time":1712345678820,"data":{"turn":1,"reason":{"kind":"completed"}}}
```

## B.5 Credentials 格式

`~/.dsh/credentials.yaml` 示例：

```yaml
providers:
  opus:
    apiKey: sk-ant-...
    baseUrl: https://api.anthropic.com
  sonnet:
    apiKey: sk-ant-...
```

## B.6 关键环境变量

| 变量 | 说明 |
|------|------|
| `DSH_HOME` | 覆盖默认的 `~/.dsh` 路径 |
| `DSH_CONFIG` | 覆盖配置文件路径 |
| `DSH_LOG_LEVEL` | 日志级别（`debug`/`info`/`warn`/`error`） |
| `DSH_PORT` | Web UI 的监听端口（默认 3914） |

## B.7 常用 CLI 命令

```bash
dsh start --config cordis.yml          # 启动 dsh 实例
dsh tell "message"                     # 发送消息（headless 模式）
dsh session list                       # 列出所有 session
dsh session resume <id>                # 恢复指定 session
dsh inspect                            # 查看组合树状态
dsh dump-config                        # 打印当前完整配置
```