# 插件配置 —— Plugin Configuration

> **来源**：<https://deepseek-harness.github.io/deepseek-harness/en/develop/basic/config>
> 本页为精炼导读（中英对照），完整细节以官方原文为准。

## TL;DR（一句话概括）

导出同名 `Config` 类型 + Schemastery schema，`cordis.yml` 里的 `config` 经 schema 校验、填默认值后作为第二个参数传入 `apply(ctx, config)`。

## 核心概念

### 定义 Config 类型（Define the Config type）

- 导出 `Config` interface 和同名 Schemastery schema，**默认值直接写在 schema 字段上**：

```ts
import Schema from '@deepseek-ai/schemastery'

export interface Config {
  greeting: string
  maxRetries: number
  verbose?: boolean
}

export const Config: Schema<Config> = Schema.object({
  greeting: Schema.string().default('Hello'),
  maxRetries: Schema.number().default(3),
  verbose: Schema.boolean().default(false),
})

export function apply(ctx: Context, config: Config) {
  console.log(config.greeting)  // 用户值或 schema 默认值
}
```

- 在 `cordis.yml` 中给插件行加 `config`：

```yaml
- insert:
    - id: hello
      name: './src/my-plugin.ts'
      config:
        greeting: 'Hi there'
        maxRetries: 5
```

> ⚠️ **不要**把 `Config` 导出为普通对象——它必须实现 Cordis 要求的 Standard Schema 接口。

### Schema 校验（Schema validation）

- 更强的校验用 Schemastery：`required()` / `default()` / `union(['fast', 'accurate'])` 等。

```ts
export const Config = Schema.object({
  apiKey: Schema.string().required(),
  timeout: Schema.number().default(30000),
  mode: Schema.union(['fast', 'accurate']).default('fast'),
})
// 加载时校验；非法配置以可操作的错误信息使加载失败
```

### 设计原则（Design principles）

1. **不硬编码可调值**：两个部署可能需要不同设置的东西，一律做成配置字段。判据：`cordis.yml` 能否不改代码就改掉这个值。
2. **非法配置快速失败（fail loudly）**：自包含约束写进 schema，让非法配置在加载时失败；涉及服务/已注册资源的引用走依赖注入（见 [services 教程](../framework/service.md)）。

### 与 HMR 协作（Work with HMR）

- 配置编辑会**热替换插件**：框架卸载旧实例、加载新实例。注册项是 effect 会自动清理，因此替换不会残留旧实例的注册。

## 术语对照

| 英文术语 | 中文 | 说明 |
|----------|------|------|
| Config schema | 配置模式 | Schemastery schema，兼具校验与默认值 |
| Standard Schema | 标准模式接口 | Cordis 要求的 schema 接口，普通对象不满足 |
| required / default / union | 必填/默认/联合 | schema 校验原语 |
| HMR | 热模块替换 | 配置编辑后自动卸载+重载插件 |

## 陷阱与注意

- `Config` 必须是 Schemastery schema 实例，纯对象会报错。
- 默认值放 schema（而非 apply 内兜底），保证 `cordis.yml` 可覆盖。
- HMR 后旧实例的注册自动清理，无需（也不应）手动清理。

## 关联页面

- [打包安装插件](./publish.md) —— 以可安装包形式分发
- [插件与生命周期](../framework/index.md) —— 完整插件生命周期
- [服务与依赖](../framework/service.md) —— 向其他插件提供服务