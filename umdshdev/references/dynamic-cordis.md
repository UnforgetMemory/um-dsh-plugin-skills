# 会话内动态插件规范（cordis_define 路径）

> 供入口「会话内动态插件」分支按需加载。事实源：本会话系统提示 Dynamic Cordis Plugins 规范 + `cordis-plugin-development` 技能 + host 服务目录实测签名。本页与 `basic.md`（仓库级源码开发）是**两条不同路径**，勿混用。

## 两条路径判定

| | 路径 A：仓库级源码开发 | 路径 B：会话内动态插件（本页） |
|---|---|---|
| 形态 | TS 模块 `export function apply(ctx)` | 纯 JS Plugin 对象，`cordis_define` 定义 |
| 加载 | cordis.yml + `pnpm dsh web --patch` | `cordis_run` 激活 |
| 前提 | deepseek-harness 源码 checkout | 当前 harness 进程（本会话即可） |
| 生命周期 | 文件 + 重启 | `cordis_stop`/`cordis_undefine`，进程退出即消失 |

## 最小插件（纯 JS）

```js
return {
  apply(ctx) {
    console.log('[hello] loaded')
  },
}
```

- `code.host` / `code.client` 都是**纯 JavaScript 函数体**，返回 Cordis Plugin。**不编译**：无 TS 类型、`as`、装饰器、`import`/`require`、JSX。
- Client 的 React 代码必须用 `React.createElement(...)`，禁止 `<Component />`。
- 不要假设 `process`/`Buffer`/`window`/`document`/`fetch`/原生定时器存在；先查 `Builtin.listBuiltins`。

## 访问服务（ctx.get 与 inject）

- **默认 `ctx.get('name')` 读可选服务并判空**：

```js
return {
  apply(ctx) {
    const service = ctx.get('serviceName')
    if (service === undefined) return
    service.someMethod()
  },
}
```

- **仅当硬依赖**才 `inject`（服务缺席时插件进入等待，出现后 Cordis 重激活）：

```js
return {
  inject: ['requiredService'],
  apply(ctx) {
    ctx.requiredService.someMethod()
  },
}
```

- 禁止未声明 inject 就访问 `ctx.x`（Guard 拒绝）；禁止为省判空滥用 inject。

## 生命周期（每个副作用必须可逆）

- `ctx.on()` 注册事件监听；`ctx.effect(() => ...)` 持有返回 disposer 的外部订阅；Cordis 的 Service/Tool/Slot/timer/theme API 返回的 disposer 都要保留。
- 全部属于当前 Fiber：stop / update / undefine 时自动移除。禁止在模块作用域或 `apply()` 外产生进程级/页面级副作用。
- 定时器是**服务**（名为 `timer`，非 Builtin）：用前查 `{ "service": "timer" }` 并 `inject: ['timer']`，然后 `ctx.timeout()` / `ctx.interval()`。

## Host 与 Client 分工

| 需求 | 平台 | 先查 |
|------|------|------|
| 文件/命令/进程/网络 | Host | fs/bash/subprocess/pty/web 服务 |
| 注册下一模型步可调用的工具 | Host | `harness`（Builtin）+ `Tool.listTools` |
| 页面主题/布局/当前页状态 | Client | Theme 令牌 + Client 服务 |
| 设置页/侧栏/输入区/覆盖层/工具卡 | Client | `Slots.listSubTree` |

- **Client→Host 通信**：Host 用 `harness.handle(method, handler)` 注册私有方法，Client 用 `host.call(method, args)` 调用；只传无损耗 JSON，禁止传函数/React 元素/类实例/Service 等运行态对象。不要用 `ctx.remote` 做包内私有通信。
- Client UI 必须注册进查询到的 Slot；`apply()` 不能直接返回 React 元素作为插件结果。

## 注册动态模型工具（Host）

```js
return {
  inject: ['tools'],
  apply(ctx) {
    ctx.tools.register({
      name: 'my_tool',
      description: '...',
      // 参数/结果 JSON 兼容；execute 拥有业务结果
    })
  },
}
```

- 工具注册必须属于当前插件 Fiber → stop/update 后自动移除。
- 先 `Tool.listTools` 查重与冲突；execute 的结果与 render 的展示分离。

## 版本与修复

- `pluginId` = 稳定插件实例；`packageId` = 不可变代码版本；`pluginRunId` = 一次激活尝试。
- 改代码 = 定义新 Package（`cordis_define` kind: existing），用 `update` 切版本；失败不自动回滚，`run` 到 current 回滚。
- 技术失败后：`cordis_inspect_self(pluginId, packageId)` 读源码与诊断 → 修正 → 定义新 Package → 重跑；用户拒绝审批后不得自动重试。
- 查看当前插件：`cordis_inspect_self` 无参列出。

## 易错

- 在 code 里写 TS/JSX/import → 客户端解析失败。
- `ctx.x` 未声明 inject → "service not declared"。
- `ctx.get('timer')` 未判空 / 未 inject → timer 不可用。
- 忘保留 Service/Slot 返回的 disposer → 卸载泄漏。
- Client apply 返回 React 元素 → 应注册进 Slot。
- 序列化 Service/Event 载荷/快照 → 只读需要的叶子字段，构造最小自有 JSON。

## 深挖

完整精读/边界案例/全量代码骨架 → `guide/practice/dynamic-cordis.md`（技能内内容层）。官方原文兜底走外链。
