# 服务 —— Services

> **来源**：<https://deepseek-harness.github.io/deepseek-harness/en/develop/cordis-tutorial/03-services>
> 本页为精炼导读（中英对照），完整细节以官方原文为准。

## TL;DR（一句话概括）

**服务**是一个插件提供、其他插件经 `ctx` 消费的命名能力；消费者按名字（如 `'tools'`）而非导入提供者来引用它，于是配置可在不改消费者的前提下换提供者。

## 核心概念

### 提供服务（Provide a service）

创建 `greeter.ts`：

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {   // 类型声明合并（仅编译期，不产生代码）
  interface Context {
    greeter: GreeterService
  }
}

export class GreeterService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'greeter')   // 运行时：以名字 'greeter' 注册实例
  }

  greet(who: string) {
    return `Hello, ${who}!`
  }
}

export const name = 'greeter'

export function apply(ctx: Context) {
  ctx.plugin(GreeterService)   // Service 子类本身就是插件
}
```

两块协同：

- **运行时**：`super(ctx, 'greeter')` 以名字 `greeter` 注册实例，此后任何插件都能以 `ctx.greeter` 访问它；注册是 effect——卸载提供者即移除服务。
- **编译期**：`declare module '@deepseek-ai/cordis'` 是 TS 声明合并，把 `greeter` 加进 `Context` 接口，使 `ctx.greeter` 处处通过类型检查；不产生代码，没有它服务运行时照常工作，只是消费者失去类型安全。

### 用 `inject` 消费（Consume a service with inject）

创建 `consumer.ts`：

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'consumer'
export const inject = ['greeter']   // 声明硬依赖

export function apply(ctx: Context) {
  console.log(ctx.greeter.greet('world'))   // 此时 ctx.greeter 保证就绪
}
```

- `inject` 列出本插件所需服务；Cordis 让插件保持 PENDING 直到每个服务存在，因此 `apply` 内 `ctx.greeter` 保证就绪。`cordis.yml` 的加载顺序无关——依赖而非文件顺序决定插件何时启动。

```yaml
- name: './greeter.ts'
- name: './consumer.ts'
# 输出：Hello, world!
```

- 交换两行重跑：输出相同。删掉 `./greeter.ts`：consumer 停在 PENDING 且不打印——不崩溃、不半运行。PENDING fiber 也不维持 Node 事件循环，故无其他运行内容的组合会静默退出码 0（第 6 章讲如何诊断该状态）。

### 依赖在加载后被跟踪（Dependencies are tracked after load）

`inject` 不是一次性的 boot 检查。运行中若必需服务消失（提供者被卸载/热替换），每个依赖它的插件也一并卸载，服务回来时再加载。配合 effect（第 2 章），防止运行中的消费者保留对不可用服务的引用：依赖消失时其自身注册被回卷。这也是服务替换在配置里可行的原因——卸载 `dsh-bash-local`、挂载另一个 `shell` 提供者，每个 `inject: 'shell'` 的插件都干净地对着新实现重启。

### 可选依赖（Optional dependencies）

`inject` 用于硬需求；可无此能力也能活的插件跳过 `inject`，在使用点探测：

```ts
export function apply(ctx: Context) {
  // 无提供者时为 undefined；插件仍运行
  const greeter = ctx.get('greeter')
  console.log(greeter?.greet('maybe') ?? 'no greeter available')
}
```

### 命名（Naming）

服务名处于每个应用的单一扁平命名空间。给自己的服务加前缀/命名空间加以区分（Harness 占用 `tools`、`llm` 等普通名）；子系统页面的 `cordis-surface` 区域列出 Harness 注册的每个名字。

## 术语对照

| 英文术语 | 中文 | 说明 |
|----------|------|------|
| service | 服务 | 一个插件提供、其他插件消费的命名能力 |
| provider | 提供者 | 注册服务实现的插件 |
| consumer | 消费者 | 经 `inject` 依赖服务的插件 |
| inject | 注入声明 | 硬依赖列表，框架等待其就绪 |
| Service subclass | 服务子类 | 类形态插件，`super(ctx, name)` 注册 |
| declaration merging | 声明合并 | 把属性加进 Context 接口，编译期生效 |
| ctx.get() | 可选获取 | 探测可选服务，无提供者返回 undefined |

## 陷阱与注意

- 服务**注册是 effect**：卸载提供者即移除服务。
- `inject` 是**持续跟踪**，不是 boot 时一次性检查；依赖消失消费者也被卸载。
- 硬依赖用 `inject`，可选项用 `ctx.get()` 探测，别在 `apply` 里假设 `ctx.greeter` 已存在。
- 服务名是**扁平命名空间**，自定义服务要加前缀避免撞 Harness 的 `tools`/`llm`。

## 关联页面

- [生命周期与副作用](../cordis-tutorial/02-lifecycle-and-effects.md) —— 服务注册如何作为 effect 回卷
- [事件](../cordis-tutorial/04-events.md) —— 无需共享服务的通信
