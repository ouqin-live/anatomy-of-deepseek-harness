# 附录 C Mini Harness

本附录提供一个简化版的"Mini Harness"实现，它展示了 dsh 核心设计（事件溯源 + Agent Loop + 工具系统）的最小形式。不依赖 Cordis，不依赖外部包，仅用标准 TypeScript。

## C.1 事件日志

```typescript
type SessionEvent =
  | { type: 'user/message'; content: string }
  | { type: 'assistant/message'; content: string; toolCalls?: ToolCall[] }
  | { type: 'tool/call'; name: string; args: Record<string, unknown> }
  | { type: 'tool/result'; content: string }

class Session {
  events: SessionEvent[] = []

  append(event: SessionEvent): void {
    this.events.push(event)
  }

  /** 简单的投影：直接返回所有事件（无 surface/replace 逻辑） */
  deriveMessages(): { role: 'user' | 'assistant' | 'tool'; content: string }[] {
    const msgs: { role: 'user' | 'assistant' | 'tool'; content: string }[] = []
    for (const event of this.events) {
      switch (event.type) {
        case 'user/message':
          msgs.push({ role: 'user', content: event.content })
          break
        case 'assistant/message':
          msgs.push({ role: 'assistant', content: event.content })
          break
        case 'tool/result':
          msgs.push({ role: 'tool', content: event.content })
          break
      }
    }
    return msgs
  }
}
```

## C.2 工具系统

```typescript
type ToolExecute = (args: Record<string, unknown>) => Promise<string>

interface ToolDefinition {
  name: string
  description: string
  parameters: Record<string, unknown>
  execute: ToolExecute
}

class ToolRegistry {
  private tools = new Map<string, ToolDefinition>()

  register(tool: ToolDefinition): void {
    this.tools.set(tool.name, tool)
  }

  get(name: string): ToolDefinition | undefined {
    return this.tools.get(name)
  }
}
```

## C.3 Agent Loop

```typescript
interface LlmResponse {
  content: string
  toolCalls?: { name: string; args: Record<string, unknown> }[]
}

// 一个只返回固定响应的 LLM（测试用）
class MockLlm {
  async stream(prompt: string): Promise<LlmResponse> {
    return { content: `You said: ${prompt}` }
  }
}

class Agent {
  session = new Session()
  tools = new ToolRegistry()
  llm = new MockLlm()

  async turn(userInput: string): Promise<void> {
    this.session.append({ type: 'user/message', content: userInput })

    let step = 0
    while (true) {
      step++
      const messages = this.session.deriveMessages()
      const systemPrompt = 'You are a helpful assistant.'

      const response = await this.llm.stream(systemPrompt + '\n' + JSON.stringify(messages))

      this.session.append({
        type: 'assistant/message',
        content: response.content,
        toolCalls: response.toolCalls,
      })

      if (!response.toolCalls || response.toolCalls.length === 0) break

      for (const call of response.toolCalls) {
        this.session.append({ type: 'tool/call', name: call.name, args: call.args })
        const tool = this.tools.get(call.name)
        if (tool) {
          const result = await tool.execute(call.args)
          this.session.append({ type: 'tool/result', content: result })
        } else {
          this.session.append({ type: 'tool/result', content: `Error: tool ${call.name} not found` })
        }
      }
    }
  }
}
```

## C.4 使用示例

```typescript
async function main() {
  const agent = new Agent()

  agent.tools.register({
    name: 'add',
    description: 'Add two numbers',
    parameters: { a: 'number', b: 'number' },
    execute: async (args) => {
      const result = (args.a as number) + (args.b as number)
      return `Result: ${result}`
    },
  })

  // 模拟 LLM 返回工具调用
  const originalStream = agent.llm.stream.bind(agent.llm)
  let callCount = 0
  agent.llm.stream = async () => {
    callCount++
    if (callCount === 1) {
      return { content: '', toolCalls: [{ name: 'add', args: { a: 1, b: 2 } }] }
    }
    return { content: 'The result is 3.' }
  }

  await agent.turn('What is 1 + 2?')

  console.log('Session events:', agent.session.events.length)
  console.log('Messages:', agent.session.deriveMessages())
}

main()
```

这个 Mini Harness 在约 100 行代码中实现了 dsh 的核心思想：事件日志记录所有交互，Agent Loop 在 turn/step 之间循环，工具注册系统供模型调用。dsh 的完整实现在此基础上增加了 Cordis 插件系统、表面层、Compaction、能力接缝等多个层次。

读者可以从这个 Mini Harness 开始，逐步增加持久化、事件回放、并行工具执行等功能，以理解 dsh 各个子系统为什么要存在。