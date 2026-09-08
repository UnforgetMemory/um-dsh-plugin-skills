# 事件 —— Events

> **来源**：<https://deepseek-harness.github.io/deepseek-harness/en/develop/cordis-tutorial/04-events>
> 本页为精炼导读（中英对照），完整细节以官方原文为准。

## TL;DR（一句话概括）

服务支持直接调用；**事件**让插件在不知道谁在监听的情况下宣布某事。Harness 用事件处理工具结果、模型请求、审批决策等交互。

## 核心概念

### 声明、发出、监听（Declare, emit, listen）

创建 `stats.ts`——计数并宣布每次变化：

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Context { stats: StatsService }
  interface Events {                       // 声明事件名 + 监听器签名
    'stats/report'(name: string, count: number): void
  }
}

export class StatsService extends Service {
  private counts = new Map<string, number>()
  constructor(ctx: Context) { super(ctx, 'stats') }

  bump(name: string) {
    const next = (this.counts.get(name) ?? 0) + 1
    this.counts.set(name, next)
    this.ctx.emit('stats/report', name, next)   // 发出事件
  }
}

export const name = 'stats'
export function apply(ctx: Context) { ctx.plugin(StatsService) }
```

- `interface Events` 合并是第 3 章 `interface Context` 合并的事件版孪生：声明事件名与监听器签名，使 `ctx.emit` 和 `ctx.on` 完全类型化；`namespace/action` 命名约定让扁平事件命名空间可读。

创建 `reporter.ts`：

```ts
import type { Context } from '@deepseek-ai/cordis'
import type {} from './stats.ts'   // 运行时导入空，仅为让 TS 看到声明合并

export const name = 'reporter'
export const inject = ['stats']

export function apply(ctx: Context) {
  ctx.on('stats/report', (name, count) => {
    console.log(`[stats] ${name} -> ${count}`)
  })
  ctx.stats.bump('tool_call')
  ctx.stats.bump('tool_call')
  ctx.stats.bump('prompt')
}
```

输出：

```
[stats] tool_call -> 1
[stats] tool_call -> 2
[stats] prompt -> 1
```

- `ctx.on()` 是 effect，监听器随插件消失——永远无需手动 `removeListener`。

### 派发模式（Dispatch modes）

`emit` 是五种派发模式之一。事件用哪种模式是其契约的一部分——决定监听器能否返回值、并发运行或互相短路：

| 模式 | 调用 | 语义 |
|------|------|------|
| emit | `ctx.emit(name, ...args)` | 同步广播；返回的 promise 与值不被 await/收集 |
| parallel | `await ctx.parallel(name, ...args)` | 所有监听器并发运行，一起 await |
| serial | `await ctx.serial(name, ...args)` | 监听器按序 await；首个非 `null`/`false`/`undefined` 返回胜出并停止其余 |
| bail | `ctx.bail(name, ...args)` | serial 的同步版 |
| waterfall | `ctx.waterfall(name, ...args, next)` | 环绕中间件，见下 |

- 每个 Harness 事件在其子系统页面文档化自己的模式。

### Waterfall：变换或短路（Waterfall: transform or short-circuit）

Waterfall 是支撑拦截的模式。每个监听器收到参数外加 `next()` 续体：可变换 `next()` 的返回，或不调 `next()` 直接返回而短路余下链——Cordis 文档称之 veto。创建 `waterfall-demo.ts`：

```ts
import type { Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Events {
    'demo/transform'(input: string, next: () => Promise<string>): Promise<string>
  }
}

export const name = 'waterfall-demo'

export function apply(ctx: Context) {
  // 监听器 1：包裹下游结果
  ctx.on('demo/transform', async (input, next) => {
    const downstream = await next()
    return downstream.toUpperCase()
  })

  // 监听器 2：由它决策时短路
  ctx.on('demo/transform', async (input, next) => {
    if (input.includes('blocked')) return '** blocked **'
    return next()
  })

  void (async () => {
    console.log(await ctx.waterfall('demo/transform', 'hello', async () => 'hello'))
    console.log(await ctx.waterfall('demo/transform', 'blocked words', async () => 'blocked words'))
  })()
}
```

输出：

```
HELLO
** BLOCKED **
```

- 第二行流程：监听器 1 先跑、调 `next()` 进入监听器 2；监听器 2 见 `blocked` 不调 `next()` 直接返回——最内层默认值（传给 `ctx.waterfall` 的函数）从不执行——监听器 1 返回途中把替换消息转大写。

随之而来的纪律：**只观察/标注的 waterfall 监听器必须调 `next()`**；不调 `next()` 返回是有意短路。日志监听器忘写 `next()` 会静默吞掉所有下游的默认行为。

- Harness 用 waterfall 处理协作插件可包裹或回答的决策：`agent/request` 让插件替换模型调用配置，`approval/request` 让策略代用户回答。

## 术语对照

| 英文术语 | 中文 | 说明 |
|----------|------|------|
| event | 事件 | 无需共享服务的广播通信 |
| emit | 发出/广播 | 同步广播派发 |
| listener | 监听器 | 经 `ctx.on` 注册的回调 |
| dispatch mode | 派发模式 | 事件契约，决定返回值/并发/短路语义 |
| waterfall | 瀑布/环绕中间件 | 可变换结果或短路（veto） |
| next() | 续体 | waterfall 中把控制传给下一监听器 |
| veto | 否决/短路 | 不调 next() 直接返回 |

## 陷阱与注意

- `import type {} from './stats.ts'` 运行时导入空，仅为让 TS 看到声明合并。
- `ctx.on()` 是 effect，无需手动 `removeListener`。
- **只观察的 waterfall 监听器必须调 `next()`**，否则静默吞掉下游默认行为——这是仓库的常设规则。

## 关联页面

- [服务](../cordis-tutorial/03-services.md) —— `interface Context` 合并的孪生来源
- [配置](../cordis-tutorial/05-config.md) —— 来自 cordis.yml 的插件选项
