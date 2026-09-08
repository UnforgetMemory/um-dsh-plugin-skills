# 插件与生命周期 —— Plugins and lifecycle

> **来源**：<https://deepseek-harness.github.io/deepseek-harness/en/develop/framework/>
> 本页为精炼导读（中英对照），完整细节以官方原文为准。

## TL;DR（一句话概括）

每个已加载插件拥有独立的 **Fiber**（生命周期作用域），按 `PENDING → LOADING → ACTIVE → UNLOADING → DISPOSED` 状态机运转；通过 `ctx` 注册的一切能力在卸载时自动清理。

## 核心概念

### Fiber 状态机（Fiber state machine）

```
PENDING → LOADING → ACTIVE
                 ↘ FAILED
ACTIVE → UNLOADING → DISPOSED
```

| 状态 | 含义 |
|------|------|
| PENDING | 已声明，但所需依赖尚未就绪 |
| LOADING | 依赖就绪，正在执行 `apply` |
| ACTIVE | 插件正在运行 |
| FAILED | `apply` 抛出了错误 |
| UNLOADING | 正在卸载并释放资源 |
| DISPOSED | 已完全卸载 |

### 依赖驱动加载（Dependency-driven loading）

带 `inject` 的插件会**等待所有必需服务就绪**后才加载：

```ts
export const inject = ['tools', 'llm']

export function apply(ctx: Context) {
  // ctx.tools 与 ctx.llm 在此已就绪
}
```

若某个必需服务消失（例如 provider 被替换），插件会自动卸载（ACTIVE → DISPOSED），服务恢复后再次加载。

### 自动清理（Automatic cleanup）

通过 `ctx` 注册的一切在插件卸载时自动撤销：

```ts
export function apply(ctx: Context) {
  // 事件监听：卸载时自动移除
  ctx.on('some-event', handler)

  // 自定义资源：返回的 disposer 在卸载时执行
  ctx.effect(() => {
    const connection = createConnection()
    return () => connection.close()
  })
}
```

框架追踪并回收这些操作：`ctx.on(event, handler)`（事件监听）、`ctx.tools.register(tool)`（工具注册）、`ctx.llm.registerAdapter(names, adapter)`（LLM 适配器注册）、`ctx.effect(() => cleanup)`（自定义资源）。

> 卸载时 disposer **按注册逆序**触发，但多个异步 disposer **并发执行、无串行完成保证**。顺序相关的清理应放进**同一个 `ctx.effect()` 返回的 disposer**，并在其中串行 `await` 各步骤。

### 嵌套上下文（Nested contexts）

`ctx.plugin()` 创建一个子 Fiber：继承父上下文，但拥有独立生命周期，随父插件一起卸载：

```ts
export function apply(ctx: Context) {
  ctx.plugin(childPlugin)   // 子插件拥有自己的 Fiber
}
```

### 手动销毁（Dispose semantics）

```ts
const fiber = ctx.plugin(myPlugin)
await fiber.dispose()   // 稍后手动销毁
```

`dispose` 保证：① 移除插件拥有的全部注册；② 递归卸载子插件；③ 返回的 Promise 在所有异步清理完成后 resolve。

### 热替换（HMR）

`cordis.yml` 加载 `@deepseek-ai/cordis-plugin-hmr` 后，编辑插件源文件触发：卸载旧插件并清理注册 → 加载新代码 → 运行新 `apply`。因注册会自清理，热替换不会残留旧实例的注册。

## 术语对照

| 英文术语 | 中文 | 说明 |
|----------|------|------|
| Fiber | 生命周期作用域 | 每个已加载插件拥有的状态机容器 |
| PENDING / LOADING / ACTIVE | 待定/加载中/运行中 | 插件主要生命周期状态 |
| FAILED / DISPOSED | 失败/已卸载 | `apply` 抛错 / 完全卸载 |
| inject | 注入声明 | 声明所需服务，就绪后才加载 |
| ctx.on() | 事件监听注册 | 卸载时自动移除 |
| ctx.effect() | 效果注册 | 注册带 disposer 的资源 |
| ctx.plugin() | 嵌套插件注册 | 创建子 Fiber |
| fiber.dispose() | 手动销毁 | 提前停止插件实例 |
| HMR | 热替换 | 编辑源文件后自动重载 |

## 陷阱与注意

- 卸载时 disposer **逆序**触发，异步 disposer **并发**执行且**无串行完成保证**；顺序相关清理必须放进同一个 `ctx.effect()` 内串行 `await`。
- 不要手动清理 `ctx` 注册的内容；只有 `ctx.effect()` 管理的资源需返回 disposer。
- 必需服务消失会自动卸载插件（不是报错），服务恢复后再加载。
- 需要服务时必须 `inject`，不要假设 `ctx.tools` 等已存在。

## 关联页面

- [事件系统](../framework/events.md) —— 插件间通信
- [服务与依赖](../framework/service.md) —— 向其他插件暴露能力
- [你的第一个插件](../basic/index.md) —— 插件与 `ctx` 基础
- [Cordis 教程](../cordis-tutorial/index.md) —— 相同生命周期逐步搭建
- 官方完整参考：<https://deepseek-harness.github.io/deepseek-harness/en/develop/framework/>
