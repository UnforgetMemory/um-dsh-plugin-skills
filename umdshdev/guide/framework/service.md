# 服务与依赖 —— Services and dependencies

> **来源**：<https://deepseek-harness.github.io/deepseek-harness/en/develop/framework/service>
> 本页为精炼导读（中英对照），完整细节以官方原文为准。

## TL;DR（一句话概括）

服务（service）是一个插件暴露给其他插件的能力，`inject` 声明插件所需的服务；提供方继承 `Service` 类并用 `super(ctx, '名称')` 注册，消费方用 `inject` 或 `ctx.get()` 取用。

## 核心概念

### 什么是服务（What is a service）

Harness 中 `tools`、`llm`、`agents` 都是服务——挂在 `ctx` 上的具名能力（`ctx.tools` = ToolRuntime 服务、`ctx.llm` = LLM 服务、`ctx.agents` = Agent 服务）。任何插件都能提供服务供其他插件消费。

### 消费服务（Consume a service）

用 `inject` 声明使用已有服务；`apply` 运行时所有 `inject` 声明的服务均已就绪（未就绪则**等待**而不是运行）：

```ts
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(/* ... */)   // ctx.tools 已存在且就绪
}
```

### 提供服务（Provide a service）

继承 `Service`，构造器用 `super(ctx, '服务名')` 注册；再用 TS 声明合并为 `ctx.<服务名>` 添加类型：

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Context { metrics: MetricsService }   // 类型声明
}

export default class MetricsService extends Service {
  static inject = ['llm']   // 服务本身也可依赖其他服务

  constructor(ctx: Context) {
    super(ctx, 'metrics')   // 'metrics' 即服务名
  }

  record(event: string, value: number) { /* 公开方法 */ }
}
```

加载后消费方 `inject: ['metrics']` 即可 `ctx.metrics.record('tool_call', 1)` 访问。

### 依赖行为（Dependency behavior）

```ts
export const inject = ['tools']        // 必需：服务缺失时插件不加载

export function apply(ctx: Context) {  // 可选：不写 inject，用 ctx.get() 查询
  const metrics = ctx.get('metrics')
  metrics?.record('plugin_loaded', 1)  // 可能为 undefined
}
```

运行中若必需服务消失（其提供者卸载）：① 依赖它的插件自动 dispose；② 服务恢复后再次加载——防止插件调用已不存在的服务。

### 服务隔离（Service isolation）

`cordis.yml` 用 `group` + `isolate` 让不同插件组看到同一服务的**独立实例**：

```yaml
- id: group-a
  name: '@deepseek-ai/cordis-plugin-group'
  group: true
  isolate: { shell: true }
  config:
    - name: '@deepseek-ai/dsh-bash-local'
      config: { timeoutMs: 5000 }
    - name: './src/plugin-a.ts'

- id: group-b   # 结构同 group-a，仅 timeoutMs 不同
  name: '@deepseek-ai/cordis-plugin-group'
  group: true
  isolate: { shell: true }
  config:
    - name: '@deepseek-ai/dsh-bash-local'
      config: { timeoutMs: 60000 }
    - name: './src/plugin-b.ts'
```

`plugin-a` 与 `plugin-b` 各自看到自己组内的 Bash 实例，互不影响。

### 内置 Harness 服务（Built-in Harness services）

仓库把每个服务的服务名、公开方法、源码位置生成到各自的 subsystem 页面；开发时用那些生成区域 + 服务的 TS 接口，**不要**维护第二份静态列表。

## 术语对照

| 英文术语 | 中文 | 说明 |
|----------|------|------|
| service | 服务 | 一个插件暴露给其他插件的能力 |
| inject | 注入声明 | 声明插件所需的必需服务 |
| Service（基类） | 服务基类 | 提供服务的继承基类 |
| super(ctx, name) | 注册服务名 | 构造器中将实例挂到 `ctx.<name>` |
| ctx.get(name) | 可选依赖查询 | 返回服务实例或 undefined |
| provider | 提供者 | 提供某服务的插件 |
| group | 插件组 | cordis.yml 中隔离的插件分组 |
| isolate | 隔离 | 让各分组看到独立服务实例 |

## 陷阱与注意

- 必需服务消失时依赖插件**自动卸载**（不是报错），恢复后再加载。
- 可选依赖不要写进 `inject`，用 `ctx.get('name')` 并处理 `undefined`（如 `metrics?.record(...)`）。
- 提供服务必须 `super(ctx, '服务名')` 注册名称，消费方才能以 `ctx.<服务名>` 访问。
- 服务名/公开方法以 subsystem 页面生成结果为准，不要自行维护静态列表。

## 关联页面

- [插件与生命周期](../framework/index.md) —— 依赖驱动的加载/卸载
- [事件系统](../framework/events.md) —— 插件间松耦合通信
- [能力分层](../practice/index.md) —— 把服务当作能力接口
- [你的第一个插件](../basic/index.md) —— 三种插件形态（含 Class 提供服务）
- 官方完整参考：<https://deepseek-harness.github.io/deepseek-harness/en/develop/framework/service>
