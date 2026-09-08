# 组合与热更新 —— Composition and HMR

> **来源**：<https://deepseek-harness.github.io/deepseek-harness/en/develop/cordis-tutorial/06-composition-and-hmr>
> 本页为精炼导读（中英对照），完整细节以官方原文为准。

## TL;DR（一句话概括）

迄今为止的每个能力都是插件，`cordis.yml` 选定应用的插件树；本章改组合、热更新插件，并诊断一个永不加载的插件。

## 核心概念

### 条目不止 name（Entries are more than a name）

配置条目接受 `name` 与 `config` 之外的元数据：

```yaml
- id: greeter          # 该条目的稳定身份
  name: './greeter.ts'
- id: consumer
  name: './consumer.ts'
  disabled: true       # 保留条目，但跳过挂载
```

- `id` 给条目稳定身份，让 loader 区分"编辑已有条目"与"删除 + 新增"；`disabled: true` 卸载插件但不删条目——翻回来插件（及所有 PENDING 其服务的插件）再次加载。
- 组（groups）嵌套一份条目子列表，作为整体加载/卸载；`isolate` 给组一个自己的服务名实例——两组可各看一个不同配置的 `shell` 提供者而不互相影响。

### 热模块替换（Hot module replacement）

卸载释放 effect（第 2 章）、加载遵循依赖（第 3 章），所以 HMR 能通过卸载 + 加载替换运行中的插件。`@deepseek-ai/cordis-plugin-hmr` 监听文件并在保存时照做。写 `cordis.yml`：

```yaml
- id: logger
  name: '@deepseek-ai/cordis-plugin-logger-console'
- id: timer
  name: '@deepseek-ai/cordis-plugin-timer'
- id: hmr
  name: '@deepseek-ai/cordis-plugin-hmr'
  config:
    root: ['.']
- id: hello
  name: './hello.ts'
```

- 两个支撑插件加入列表：HMR 经 Cordis logger 服务输出，没有 console exporter 就看不到消息；它 `inject` `timer` 服务做防抖——没有 `@deepseek-ai/cordis-plugin-timer` 就永远静默 PENDING。这份沉默是下节主题。
- 编辑 `hello.ts` 改日志消息并保存：

```
hello from my first plugin
2026-07-22 15:44:36 [I] hmr watching [ '.' ]
2026-07-22 15:44:39 [I] hmr reload plugin at hello.ts
hello from my EDITED plugin
```

- 旧实例卸载（effect 全部回卷）、新代码加载、`apply` 再跑。编辑 `cordis.yml` 本身也会被拾取：loader 按 `id` diff 条目，只挂载/卸载/重配置变化的部分——这就是上面条目带显式 `id` 的原因：没 `id` 的条目每次读取都生成新 id，于是任何配置编辑后都被当作"删除 + 新增"而重挂载，即使它自己的行没变。

### 诊断永不加载的插件（Diagnosing a plugin that never loads）

依赖驱动加载的反面：`inject` 点名了无人提供的服务的插件永远等待、不打印。没有错误——PENDING 是合法状态，因为提供者可能稍后被挂载。

创建 `diagnose.ts` 直接看状态：

```ts
import { FiberState, type Context } from '@deepseek-ai/cordis'

export const name = 'diagnose'

export function apply(ctx: Context) {
  setTimeout(() => {
    for (const runtime of ctx.registry.values()) {
      for (const fiber of runtime.fibers) {
        if (fiber.state === FiberState.PENDING) {
          console.log(`${fiber.name} is PENDING — a required service is missing`)
        }
      }
    }
  }, 500)
}
```

加上不可满足依赖的 `needs-timer.ts`：

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'needs-timer'
export const inject = ['timer']   // 无提供者

export function apply(ctx: Context) {
  console.log('needs-timer loaded')
}
```

```yaml
- name: './needs-timer.ts'
- name: './diagnose.ts'
```

运行输出：

```
needs-timer is PENDING — a required service is missing
```

- `inject: ['timer']` 无提供者；列表加上 `- name: '@deepseek-ai/cordis-plugin-timer'` 后插件就加载。插件既不做也不报时，检查它的 fiber 状态。不加 PENDING 过滤地迭代还会看到 loader 自己的插件（Loader、Include）是 ACTIVE fiber，因为插件也挂载配置文件本身。

## 术语对照

| 英文术语 | 中文 | 说明 |
|----------|------|------|
| entry | 条目 | cordis.yml 中的一行插件声明 |
| id | 稳定身份 | 让 loader 区分编辑与增删 |
| disabled | 禁用 | 保留条目但跳过挂载 |
| group / isolate | 组 / 隔离 | 整体加载卸载 / 组内独立服务实例 |
| HMR | 热模块替换 | 保存时卸载+加载替换插件 |
| FiberState | fiber 状态 | 枚举插件实例所处状态 |
| ctx.registry | 插件注册表 | 可枚举各 runtime 的 fibers |

## 陷阱与注意

- 无 `id` 的条目每次读取生成新 id，配置一编辑就被当作增删而重挂载——条目应带显式 `id`。
- HMR 经 logger 服务输出，缺 console exporter 看不到消息；它 `inject` `timer`，缺该服务则永久 PENDING。
- `disabled: true` 只跳过挂载，不删条目；翻回来即恢复。
- 插件静默时先查 fiber 状态——PENDING 通常意味着某个 `inject` 服务缺失。

## 关联页面

- [配置](../cordis-tutorial/05-config.md) —— 上一步的配置校验
- [接入 Harness](../cordis-tutorial/07-into-the-harness.md) —— 同样的模式对真实 Harness 服务
