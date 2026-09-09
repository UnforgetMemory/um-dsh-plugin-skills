# framework 分册 —— 机制参考

## 前置

read `references/core-concepts.md`（必须）；建议先过 basic 分册。

## 适用

插件生命周期 / Fiber 状态机 / 自动清理 / 事件系统 / 服务提供消费 / 依赖行为 / 服务隔离。

## 生命周期（Fiber 状态机）

```
PENDING → LOADING → ACTIVE → FAILED | UNLOADING → DISPOSED
```

- PENDING：依赖未就绪；LOADING：依赖就绪、apply 运行；ACTIVE：运行中；FAILED：apply 抛错；UNLOADING→DISPOSED：卸载清理。
- 依赖驱动：inject 缺失的服务 → PENDING；服务消失 → 自动卸载；回归 → 自动重载。
- dispose 保证：移除全部注册、递归卸载子插件、异步清理完成后 promise 才 resolve。
- HMR：编辑源文件 → 卸载旧实例 → 加载新代码 → 跑新 apply。

## 自动清理清单

| 注册 | 方式 | 卸载时 |
|------|------|--------|
| 事件监听 | `ctx.on(evt, h)` | 自动移除 |
| 工具 | `ctx.tools.register(t)` | 自动注销 |
| LLM 适配器 | `ctx.llm.registerAdapter(n, a)` | 自动注销 |
| 自定义资源 | `ctx.effect(() => cleanup)` | 返回函数执行 |

- disposer 按注册**逆序**开始，多异步 disposer 并发、无串行保证 → 顺序依赖的清理放同一个 ctx.effect() 内串行 await。
- 子上下文：`ctx.plugin(child)` 创建独立 Fiber 的子插件，随父卸载。

## 事件

五种模式：emit（广播）/ parallel（并发）/ serial（顺序 await）/ bail（同步短路）/ waterfall（处理链，**必须调 next()**，否则有意短路）。详见 `references/service-events.md`。

## 服务

消费 = inject；提供 = `class extends Service` + `super(ctx, 'name')` + 声明合并类型；可选依赖 = `ctx.get('svc')?.`；隔离 = `group: true` + `isolate: { shell: true }`。详见 `references/service-events.md`。

## 易错

- waterfall 忘调 next() → 下游静默短路。
- 把 session 事件当 Cordis 事件监听 → 收不到。
- 需要服务时不 inject、直接 ctx.xxx → 属性不存在。
- 多 disposer 有顺序依赖却分开 effect → 并发无保证。

## 按需转向

LLM 适配 → `professions/practice.md`；服务做能力契约 → `professions/practice.md`（三角色，见 `references/capability-layering.md`）；深挖精读 → `guide/framework/`：`index.md`（插件与生命周期）· `service.md`（服务与依赖）· `events.md`（事件系统）。