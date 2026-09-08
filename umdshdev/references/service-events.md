# 服务与事件规范

## 服务（Service）

- 服务 = 一个插件暴露给其他插件的能力，挂载 `ctx`（如 `ctx.tools`/`ctx.llm`/`ctx.agents`）。
- **消费**：`export const inject = ['metrics']` → apply 运行时服务已就绪，否则等待。
- **提供**：`class MetricsService extends Service`，构造器 `super(ctx, 'metrics')`（名字即服务名）；`static inject` 可声明自身依赖。

```ts
export default class MetricsService extends Service {
  static inject = ['llm']
  constructor(ctx: Context) { super(ctx, 'metrics') }
  record(event: string, value: number) { /* ... */ }
}
```

- **类型**：`declare module '@deepseek-ai/cordis' { interface Context { metrics: MetricsService } }` 声明合并（只类型，不接线）。
- **可选依赖**：不 inject，用 `ctx.get('metrics')` 判空 `?.`。
- **服务消失**：依赖插件自动 dispose，回归时自动重载。
- **隔离**：`group: true` + `isolate: { shell: true }` 让插件组看到各自服务实例。

## 事件（Event）

- 插件间松耦合通信；命名 `namespace/action`（如 `agent/pre-step`、`tools/result`、`session/event`）。
- `ctx.on('evt', handler)` / `ctx.emit('evt', payload)`；监听器是 effect，卸载自动移除。

### 五种模式

| 模式 | 语义 | await | 终止 |
|------|------|-------|------|
| `emit` | 广播，返回值/返回 promise 均忽略 | 否 | 无 |
| `parallel` | 监听器并发执行、一并等待 | 是 | 无 |
| `serial` | 顺序执行 | 是 | 首个非 null/false/undefined 结果 |
| `bail` | serial 的同步版 | 否 | 首个非 null/false/undefined 结果 |
| `waterfall` | 处理链可包裹下游 | 是 | **必须调 `next()` 委托下游**，否则短路（veto） |

某事件用哪种模式是其契约的一部分（能否返回值/并发/短路），在子系统页面事件签名中记录。

### 类型化事件

```ts
declare module '@deepseek-ai/cordis' {
  interface Events {
    'my-plugin/ready': (payload: { id: string }) => void
  }
}
```

### 注意：session 事件 ≠ Cordis 事件

`turn/*`、`step/*`、`tool/call`、`tool/result`、`compaction/*` 是持久 session-event 类型（经 `session/event` 观察 `event.type`），不是同名 Cordis 事件。

## 易错

- waterfall 忘调 `next()` → 下游静默短路（是特性，但要刻意）。
- 把 session 事件当 Cordis 事件监听 → 收不到。
- 需要服务时不 inject、直接 ctx.xxx → 属性不存在。

## 深挖

完整精读/边界案例/全量代码骨架 → $(System.Collections.Hashtable[service-events.md])（技能内内容层）。官方原文兜底走外链。
