# 第 2 章 Cordis 内核（Context / Service / Event / Fiber）

## 2.1 Cordis 解决什么问题

dsh 有 50 多个 npm 包。如果不用框架，组装它们需要写一个 main 函数：

```typescript
// 伪代码——没有框架的组装方式
import { Session } from './packages/core/session'
import { ToolRuntime } from './packages/core/tools'
import { AgentLoop } from './packages/core/agent-loop'

const session = new Session()
const tools = new ToolRuntime(session)
const loop = new AgentLoop(session, tools)
```

这段代码有三个问题：

1. **包 A 依赖包 B，但包 A 必须在代码中显式 new B**。如果 B 的构造函数签名改了，所有 new B 的地方都要改。超过 50 个包互相引用，手动排实例化顺序难以维护。
2. **无法在运行时替换实现**。测试环境想用 mock 替换真实的 tools 实例，必须写两个 main 函数或者 switch-case。
3. **无法热重载**。改了 tools 包的代码，必须重启整个进程。

Cordis 解决这三个问题的方式是定义一个**插件协议**：每个包是一个插件，插件之间通过一个中介对象（Context）发现彼此，而不是直接 import。

## 2.2 Context——插件的容器

Cordis 应用启动时，创建一个 `Context` 实例。每个插件收到自己的 Context：

```typescript
import { Context } from '@deepseek-ai/cordis'

export function apply(ctx: Context) {
  // ctx 是这个插件的 Context
  // 插件在这里注册自己的功能
}
```

Context 上有两样东西：**Service**（其他插件提供的功能）和 **Event**（插件之间通信的通道）。

Context 不是全局单例。一个 Cordis App 中有一个根 Context，每加载一个插件，派生出一个子 Context。子 Context 可以访问父级的所有 Service，但父级不知道子级的存在。这个层级关系在后面 Agent 隔离的部分会用到。

## 2.3 Service——插件暴露功能的方式

如果一个插件想对外提供能力（如"管理所有工具"），它就把自己注册为一个 Service。

Service 是一个继承 `Service` 基类的类，构造函数调用 `super(ctx, 名字)`：

```typescript
import { Context, Service } from '@deepseek-ai/cordis'

// 先声明 ctx 上有一个 tools 属性
declare module '@deepseek-ai/cordis' {
  interface Context {
    tools: ToolRuntime
  }
}

// 然后定义 Service 类
export class ToolRuntime extends Service {
  private tools = new Map()

  constructor(ctx: Context) {
    super(ctx, 'tools')  // 注册为 ctx.tools
  }

  register(name: string, fn: Function) {
    this.tools.set(name, fn)
  }

  get(name: string) {
    return this.tools.get(name)
  }
}
```

`super(ctx, 'tools')` 等价于：`ctx.provide('tools', this)`。Cordis 把 `ToolRuntime` 实例绑定到当前 Context 的 `tools` 属性上。

其他插件通过 `ctx.tools` 拿到这个实例：

```typescript
export function apply(ctx: Context) {
  // ctx.tools 在调用 apply 之前就已经就绪
  ctx.tools.register('read_file', readFileFn)
}
```

这个插件不需要 `import` `ToolRuntime` 类。它只知道 `ctx.tools` 上有 `register()` 方法，但不知道这个实例是 `ToolRuntime` 还是 `MockToolRuntime`。这个解耦是关键：依赖是声明式的，而不是硬编码的 import。

声明依赖的方式是 `inject`：

```typescript
export const inject = ['tools']  // 告诉 Cordis：加载本插件前先有 tools

export function apply(ctx: Context) {
  // 这里 ctx.tools 一定可用
  ctx.tools.register('search', searchFn)
}
```

如果 `inject` 声明了 `['tools', 'llm']`，Cordis 加载器会先确保 `ctx.tools` 和 `ctx.llm` 都已就绪，再调用 `apply`。如果某个依赖加载失败或不存在，Cordis 抛出启动错误。

## 2.4 Event——插件间通信的通道

Service 之间需要通信。"工具执行完毕了，通知其他插件"。Cordis 的事件系统解决这个问题。

事件在 TypeScript 中通过 declaration merging 声明。Service 包在自己的代码中往 `Events` 接口追加事件签名：

```typescript
declare module '@deepseek-ai/cordis' {
  interface Events {
    'tools/result'(exec: ToolExecution, result: ToolResult): void
  }
}
```

这行代码声明了一个名为 `'tools/result'` 的事件，payload 是 `(exec, result)`。

其他插件可以监听这个事件：

```typescript
export function apply(ctx: Context) {
  ctx.on('tools/result', (exec, result) => {
    console.log(`工具 ${exec.name} 执行完毕，结果长度 ${result.content.length}`)
  })
}
```

Service 内部触发事件：

```typescript
export class ToolRuntime extends Service {
  execute(name: string, args: unknown) {
    const result = this.tools.get(name)!(args)
    this.ctx.emit('tools/result', { name }, { content: result })  // 通知所有监听器
    return result
  }
}
```

Cordis 提供了四种事件调度方式：

| 模式 | 方法 | 阻塞 | 返回值 | 语义 |
|------|------|------|--------|------|
| emit | `ctx.emit()` | 否 | void | 通知式广播，监听器各自处理，不等待 |
| waterfall | `ctx.waterfall()` | 是 | 变换后的值 | 洋葱中间件模式，监听器接收上一个的返回值 |
| parallel | `ctx.parallel()` | 是 | void | 所有监听器并行执行，等待所有完成 |
| serial | `ctx.serial()` | 是 | void | 监听器按注册顺序串行执行 |

dsh 中最常用的两种：

- **emit**：纯通知。事件发出后，监听器各自处理，不阻塞派发方，也不返回值。`'session/event'`（session 追加了事件，通知持久化插件和 UI 插件）是 emit 的代表。
- **waterfall**：中间件链。监听器按注册顺序依次执行，每个监听器调用 `await next()` 把控制权移交给下一个监听器（或最内层的兜底回调），并拿到它的返回值；监听器修改结果后作为自己的返回值，最外层监听器的返回值就是 `ctx.waterfall()` 的结果。不调 `next()` 就短路。`'tools/pre-execute'`（每个监听器决定是否批准工具执行）是 waterfall 的代表。

Waterfall 的典型调用方式：

```typescript
declare module '@deepseek-ai/cordis' {
  interface Events {
    'my-filter'(input: string, next: (v: string) => Promise<string>): Promise<string>
  }
}

// 派发方
const result = await ctx.waterfall('my-filter', '原始文本', (text) => text)

// 监听器（注册在别处）
ctx.on('my-filter', async (text, next) => {
  const downstream = await next()  // 下游监听器或兜底回调的返回值
  return downstream + '（追加）'    // 修改后作为返回值传给上游
})
```

调用 `ctx.waterfall('my-filter', '原始文本', inner)` 时，Cordis 收集所有已注册的 `'my-filter'` 监听器，按注册顺序依次执行。第一个监听器收到 `(text, next)`；它调用 `await next()` 后，Cordis 执行下一个监听器（或最后传入的 inner），每个监听器的返回值逐层返回，最外层监听器的返回值就是 `ctx.waterfall()` 的结果。

需要注意：`next` 在 Cordis 的实现中是一个**无参函数**（`vendor/cordis/src/events.ts:234-243`），监听器传给 `next(value)` 的参数会被丢弃，下一个监听器收到的仍是原始参数。修改的传递依赖返回值（以及共享对象引用的原地修改），而不是 `next` 的参数。

## 2.5 Effect——可逆副作用

插件需要注册各种东西（事件监听器、工具、prompt section）。当插件卸载时，这些注册必须被撤销——否则残留的监听器会继续运行，产生错误或内存泄漏。

Cordis 的 `ctx.effect()` 解决这个问题。它的机制是：注册副作用时传入一个回调，回调**同步执行**并返回一个 disposer 函数。当插件卸载时，Cordis 调用所有 disposer，按注册顺序的逆序：

```typescript
ctx.effect(() => {
  // 第一步：安装副作用
  const handler = (exec, result) => console.log('工具完成:', exec.name)
  ctx.on('tools/result', handler)    // 注册事件监听器

  // 第二步：返回 disposer——卸载时执行的清理代码
  return () => {
    ctx.off('tools/result', handler) // 移除事件监听器
  }
}, 'my-plugin.log-tools()')
```

effect 在整个 dsh 中普遍使用：

- `ctx.tools.register(tool)` 内部调用了 `ctx.effect()`——disposer 从工具表中移除该工具。
- `ctx.systemPrompt.section(section)` 内部调用了 `ctx.effect()`——disposer 删除该 section。
- 即使插件注册了 10 个工具，卸载时也是逆序依次移除。

HMR（热重载）时的工作流程：插件代码变化 → Cordis 卸载旧插件（撤销所有 effect）→ 加载新插件（重新执行所有 effect）。整个过程不需要开发者手动管理。

## 2.6 Fiber——因果跟踪与回滚

Fiber 是 Cordis 的异步上下文跟踪机制。当一个操作由事件触发，在它的异步执行期间派生的所有子操作属于同一个 fiber。如果 fiber 被取消，所有子操作都收到取消信号。

可以理解为：每次 `ctx.emit()` 或 `ctx.waterfall()` 创建一个 fiber，在这个调用链中通过 `ctx.effect()` 注册的所有副作用都关联到该 fiber。如果 fiber 中途抛出异常，Cordis 回滚所有已执行的 effect——已注册的监听器被移除，已追加的 session 事件被撤销。

dsh 中 fiber 的具体用途：

- **Session 事务的原子性**。`session.append()` 涉及写入事件数组和触发 `session/event` 通知。如果通知过程中某个监听器抛出异常，Cordis 回滚该 fiber——已 append 的事件从日志中移除。保证了 session 日志的 append-only 语义：事件要么完整可见，要么完全不存在。
- **Waterfall 的取消**。Agent 被 cancel 时，fiber 取消信号传播到当前 waterfall 链中尚未执行的监听器，它们不会被执行。

## 2.7 Cordis 在 dsh 中的角色

Cordis 不是 dsh 的"工具库"，它是 dsh 的组合和扩展模型。每一行约定都是基于 Cordis 的 Context、Service、Event、Effect 这四个概念：

- 每个插件是一个函数或 Service 类。入口就是 `apply(ctx)`。
- Service 通过 Context 相互发现。不 import 实现类。
- 事件用 TypeScript 声明，dispatch/listen 两端类型匹配。
- 副作用可逆，支持热重载。

dsh 的开发者在 Cordis 之上几乎没有再抽象一层——SystemPrompt Service、ToolRuntime Service、SubagentRuntime Service 都是 Cordis Service。它们用 `ctx.on()` 监听其他 Service 的事件，用 `ctx.emit()`/`ctx.waterfall()` 发布自己的事件。整个系统是纯 Cordis 的。

## 思考题

1. `waterfall` 事件中，如果一个监听器修改了传递给 `next()` 的参数，后续监听器收到的参数是原始值还是修改后的值？
2. `ctx.effect()` 的 disposer 可以抛出异常吗？如果抛出，Cordis 的回滚机制会如何处理？
3. 为什么 dsh 选择 Cordis 而不是直接使用 Node.js 自带的 EventEmitter？EventEmitter 不支持什么？
4. Fiber 取消如何与 long-running 的 LLM 流式请求交互？如果 fiber 在流式响应中间被取消，stream 是否会被关闭？