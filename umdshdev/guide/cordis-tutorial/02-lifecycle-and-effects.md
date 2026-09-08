# 生命周期与副作用 —— Lifecycle and Effects

> **来源**：<https://deepseek-harness.github.io/deepseek-harness/en/develop/cordis-tutorial/02-lifecycle-and-effects>
> 本页为精炼导读（中英对照），完整细节以官方原文为准。

## TL;DR（一句话概括）

Cordis 插件可被配置编辑、热更新、显式 dispose 或必需服务丢失而卸载；经 Cordis API 注册的一切都是 effect（副作用），随插件卸载自动撤销，API 之外管理的资源则需包进 `ctx.effect()`。

## 核心概念

### 副作用（Effects）

Cordis 不管理的资源（定时器、连接、watcher）用 `ctx.effect()` 包裹并返回 disposer。创建 `lifecycle.ts`：

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'lifecycle-demo'

function heartbeat(ctx: Context) {
  console.log('heartbeat plugin loading')
  ctx.effect(() => {
    const timer = setInterval(() => console.log('tick'), 200)
    return () => {            // 卸载时执行
      clearInterval(timer)
      console.log('heartbeat cleaned up')
    }
  })
}

export function apply(ctx: Context) {
  // 从代码挂载子插件，保留 fiber 以便稍后 dispose
  const fiber = ctx.plugin(heartbeat)
  // 演示定时器本身也是 effect：若本插件先被卸载，待执行回调被取消
  ctx.effect(() => {
    const timer = setTimeout(async () => {
      await fiber.dispose()
      console.log('disposed')
      process.exit(0)
    }, 700)
    return () => clearTimeout(timer)
  })
}
```

运行输出：

```
heartbeat plugin loading
tick
tick
tick
heartbeat cleaned up
disposed
```

三个要点：

- `ctx.plugin(heartbeat)` 从**代码**挂载函数为插件——与 YAML loader 对每个配置条目做的操作相同。函数插件无需 `apply` 方法：Cordis 直接调用函数，名字仅用于诊断；`apply` 方法只有对象形态才需要。返回值为 **fiber**（一个已加载插件实例的运行时句柄）。
- effect 主体在加载时运行，返回的 disposer 在卸载时运行；插件生命期资源永远不用自己调用 disposer。
- `fiber.dispose()` 等该插件所有清理（含异步 disposer）完成后 resolve，并递归卸载它挂载的所有子插件。

### fiber 状态机（The fiber state machine）

```
PENDING → LOADING → ACTIVE → UNLOADING → DISPOSED
                 ↘ FAILED
```

- **PENDING** — 已声明，但某必需服务（第 3 章）尚不可用
- **LOADING / ACTIVE** — `apply` 正在运行 / 已完成
- **FAILED** — `apply` 或配置校验抛错
- **UNLOADING / DISPOSED** — disposer 正在执行 / 全部拆除

### 哪些已经是 effect（What is already an effect）

内置注册 API 本身就是 effect，很少需要手写 `ctx.effect()`：

- `ctx.on(event, listener)` —— 卸载时移除监听器（第 4 章）
- `ctx.plugin(child)` —— 子插件随父插件 dispose
- 服务注册是 effect；Harness 注册表如 `ctx.tools.register(...)` 也把返回的 disposer 挂到调用插件上，自动回卷（第 7 章）

Cordis 不管理的资源：在 `ctx.effect()` 内获取并返回释放它的 disposer，Cordis 在卸载（含热更新）时调用。

一个顺序警告：disposer 按注册的**逆序**开始，但多个**异步** disposer **并发**运行；拆解步骤需按序时，放在同一个 disposer 里 await。

## 术语对照

| 英文术语 | 中文 | 说明 |
|----------|------|------|
| effect | 副作用/效果 | 注册 + 返回 disposer，卸载自动撤销 |
| disposer | 释放器 | 卸载时执行、回收资源的函数 |
| fiber | 实例句柄 | 一个已加载插件实例的运行时句柄 |
| ctx.plugin() | 代码挂载 | 从代码挂载插件，等价于 YAML 条目 |
| fiber.dispose() | 实例销毁 | 等所有清理完成后递归卸载 |
| PENDING / FAILED | 挂起 / 失败 | fiber 状态机中的两种关键状态 |

## 陷阱与注意

- 插件生命期资源**不要自己调用** disposer；Cordis 在卸载时调用。
- disposer 逆序开始，但**异步 disposer 并发运行**；需按序拆解时合并进一个 disposer。
- `ctx.on` / `ctx.plugin` / 服务注册都已是 effect，无需手动 `removeListener` / 清理。

## 关联页面

- [服务](../cordis-tutorial/03-services.md) —— 插件如何共享能力
- [组合与 HMR](../cordis-tutorial/06-composition-and-hmr.md) —— PENDING 是"插件为何不打印"的常见答案
