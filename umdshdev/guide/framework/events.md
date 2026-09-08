# 事件系统 —— Event system

> **来源**：<https://deepseek-harness.github.io/deepseek-harness/en/develop/framework/events>
> 本页为精炼导读（中英对照），完整细节以官方原文为准。

## TL;DR（一句话概括）

事件是 Cordis 插件间的核心通信机制；Harness 大量使用它做松耦合扩展点，并提供 `emit`（广播）/ `parallel`（并发）/ `serial`（顺序）/ `bail`（同步短路）/ `waterfall`（管道）多种交互模式。

## 核心概念

### 事件模式（Event modes）

> Cordis 共 **五种**派发模式（emit/parallel/serial/bail/waterfall）。本参考页原文详列四种；`parallel`（并发执行、一并等待）见 [Cordis 教程第 4 章](../cordis-tutorial/04-events.md)。

#### emit —— 广播（基本用法）

基本用法即 `ctx.on('event', cb)` 监听 / `ctx.emit('event', payload)` 派发；所有监听器**同步**执行，返回值被忽略：

```ts
ctx.emit('my-plugin/ready', { id: 'worker-1' })

ctx.on('my-plugin/ready', ({ id }) => console.log(`${id} is ready`))
```

#### bail —— 短路

监听器按序执行；首个**非** `null`/`false`/`undefined` 的返回值成为最终结果：

```ts
const result = ctx.bail('some-check', input)

ctx.on('some-check', (input) => {
  if (shouldBlock(input)) return 'blocked'   // 返回该值 → 停止后续监听器
  // 返回 null/false/undefined → 继续下一个监听器
})
```

#### serial —— 顺序执行

监听器按**注册顺序**执行，异步结果被 `await`；首个非 `null`/`false`/`undefined` 结果停止后续执行：

```ts
await ctx.serial('setup-phase', context)
```

#### waterfall —— 管道

每个监听器可**包裹下游结果**形成处理链；监听器**必须调用 `next()`** 委托下游，省略则短路管道：

```ts
const output = await ctx.waterfall('my-plugin/transform', input, async () => input)

ctx.on('my-plugin/transform', async (_input, next) => {
  const downstream = await next()   // next() 是必需的
  return downstream.trim()
})
```

### 类型化事件（Typed events）

用 TS 声明合并获得类型安全：

```ts
import '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Events {
    'my-plugin/ready': (payload: { id: string }) => void
    'my-plugin/check': (input: string) => boolean | undefined
    'my-plugin/transform': (input: string, next: () => Promise<string>) => Promise<string>
  }
}
```

### Cordis 事件与 session 记录

Harness Cordis 事件用 `namespace/action` 命名，如 `agent/pre-step`、`agent/request`、`agent/request-error`、`tools/result`、`session/event`；subsystem 页面生成的 `cordis-surface` 区记录了完整签名与模式。

`turn/*`、`step/*`、`tool/call`、`tool/result`、`compaction/*` 是**持久 session 事件类型**，并非同名 Cordis 事件；要观察它们需监听 `session/event` 并检查 `event.type`。

### 事件监听器即 effect（Event listeners are effects）

`ctx.on()` 注册的监听器在插件卸载时自动移除，无需手动移除。

### 示例：日志插件（logging plugin）

```ts
import type { Context } from '@deepseek-ai/cordis'
import '@deepseek-ai/dsh-tools'

export const name = 'tool-logger'

export function apply(ctx: Context) {
  ctx.on('tools/result', (exec, result) => {
    console.log(`[tool] ${exec.name}(${JSON.stringify(exec.arguments)})`)
    const text = result.content.map(b => b.type === 'text' ? b.text : '').join('')
    console.log(`[tool result] ${text.slice(0, 100)}`)
  })
}
```

## 术语对照

| 英文术语 | 中文 | 说明 |
|----------|------|------|
| event | 事件 | 插件间通信机制 |
| ctx.on() / ctx.emit() | 监听 / 派发 | 事件基础用法 |
| emit（broadcast） | 广播 | 同步执行、忽略返回值 |
| bail | 短路 | 首个非空返回值即最终结果 |
| serial | 顺序执行 | 按注册序、异步 await |
| waterfall | 管道 | 监听器包裹下游，须调 next() |
| next() | 下游委托 | waterfall 中继续处理链 |
| declaration merging | 声明合并 | 用 `interface Events` 做类型安全 |
| session/event | 会话事件 | 观察持久 session 事件的入口 |

## 陷阱与注意

- waterfall 监听器**必须调用 `next()`**；省略会短路管道（这是拦截/网关的预期行为）。
- `bail` / `serial` 中返回 `null`/`false`/`undefined` 会继续，返回其他值即终止。
- `turn/*`、`step/*`、`tool/call`、`tool/result`、`compaction/*` 是持久 session 事件，**不是**同名 Cordis 事件；须监听 `session/event` 并检查 `event.type`。
- `ctx.on()` 监听器是 effect，随插件卸载自动清理。

## 关联页面

- [插件与生命周期](../framework/index.md) —— 监听器自动清理
- [服务与依赖](../framework/service.md) —— 通过服务暴露能力
- [能力分层](../practice/index.md) —— 在能力接口中理解事件
- [LLM 适配器](../practice/llm-adapter.md) —— 实现完整 LLM 后端
- 官方完整参考：<https://deepseek-harness.github.io/deepseek-harness/en/develop/framework/events>
