# 配置 —— Configuration

> **来源**：<https://deepseek-harness.github.io/deepseek-harness/en/develop/cordis-tutorial/05-config>
> 本页为精炼导读（中英对照），完整细节以官方原文为准。

## TL;DR（一句话概括）

每个 `cordis.yml` 条目可带 `config` 块，插件声明一个在 `apply` 运行前校验它的 schema；坏配置让加载以精确错误失败——插件绝不半配置启动。

## 核心概念

### 可配置插件（A configurable plugin）

创建 `config-demo.ts`：

```ts
import type { Context } from '@deepseek-ai/cordis'
import Schema from '@deepseek-ai/schemastery'

export const name = 'config-demo'

export interface Config {          // TS 接口：给消费者类型
  greeting: string
  targets: string[]
}

export const Config: Schema<Config> = Schema.object({   // 运行时 schema：给 Cordis 校验
  greeting: Schema.string().default('Hello'),
  targets: Schema.array(String).default(['world']),
})

export function apply(ctx: Context, config: Config) {
  for (const target of config.targets) {
    console.log(`${config.greeting}, ${target}!`)
  }
}
```

- 导出的 `Config` 同时是 TS 接口和同名运行时 schema——消费者拿类型，Cordis 拿校验器。本仓库用 [Schemastery](https://github.com/shigma/schemastery)；Cordis 接受任何 Standard Schema 校验器，所以导出普通对象当 `Config` 不生效。

配置它：

```yaml
- name: './config-demo.ts'
  config:
    targets: ['alpha', 'beta']
```

运行：

```
Hello, alpha!
Hello, beta!
```

- `greeting` 被省略，schema 默认值填上——`apply` 总是收到完整、已校验的 config。

### 响亮失败（Fail loud）

喂入非法值：

```yaml
- name: './config-demo.ts'
  config:
    targets: 'not-an-array'
```

```
ValidationError: invalid config:
  - $.targets expected array but got not-an-array (at targets)
```

- 插件 fiber 进入 FAILED，本教程 launcher 打印错误后以状态 1 退出。插件也应尽快拒绝 schema 合法但指向不存在资源/提供者的 config。

### 计算型配置值（Computed config values）

本仓库 loader 支持 `!!js` 标签，用于加载期必须计算的配置值：

```yaml
- name: './config-demo.ts'
  config:
    greeting: !!js process.env.DEMO_GREETING ?? 'Hello'
```

- `!!js` 只在 `config` 和条目的 `disabled` 字段内生效。`disabled: !!js ...` 在每次挂载决策时对 loader 上下文求值（本仓库扩展），使行能按平台/环境自我门控；其他元数据（`name`、`id`、`inject`…）保持静态，其中表达式就是普通真值数据。

## 术语对照

| 英文术语 | 中文 | 说明 |
|----------|------|------|
| config block | 配置块 | cordis.yml 条目上的 `config:` |
| schema | 校验模式 | 在 apply 前校验 config |
| default | 默认值 | 省略字段时填充 |
| ValidationError | 校验错误 | 坏配置触发的精确失败 |
| Schemastery | — | 本仓库使用的 schema 库 |
| Standard Schema | 标准模式 | Cordis 接受的校验器接口 |
| !!js tag | JS 标签 | 加载期计算配置值的标签 |

## 陷阱与注意

- `Config` 必须导出为**运行时 schema**（Schemastery / Standard Schema），普通对象不生效。
- 坏配置使 fiber FAILED、进程以非零码退出——**响亮失败**，插件绝不半配置启动。
- `!!js` 只在 `config` 与 `disabled` 字段内生效；其他元数据里它是普通数据。
- 也应拒绝 schema 合法但指向不存在资源/提供者的 config，越早越好。

## 关联页面

- [事件](../cordis-tutorial/04-events.md) —— 通信的上一章
- [组合与 HMR](../cordis-tutorial/06-composition-and-hmr.md) —— 把 cordis.yml 当作应用
