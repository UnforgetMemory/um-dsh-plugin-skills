# 配置编写规范（Config + Schemastery）

## 骨架

```ts
import type { Context } from '@deepseek-ai/cordis'
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

## 规则

1. `Config` 必须是 Schemastery schema（interface + 同名 schema 变量）。**禁止导出普通对象**——不实现 Standard Schema，加载失败。
2. **默认值放 schema 字段**（不放 apply 内兜底），让 `cordis.yml` 能覆盖。
3. 校验原语：`required()` / `default(v)` / `union(['a','b'])` / `Schema.array(...)`。
4. **加载时校验**：非法配置以可操作错误使加载失败（fail loudly），不在 apply 里静默纠错。
5. 运行态引用（服务等）走 `inject` 注入，不写进 schema。

## 设计原则

- 不硬编码可调值：两个部署可能设置不同的东西一律进配置。判据：`cordis.yml` 能否不改代码改掉它。
- 自包含约束进 schema（加载即失败）；跨服务约束留运行期。

## HMR

配置编辑 → 热替换（卸载旧实例、加载新实例）；旧注册是 effect 自动清理，无残留。

## 易错

- 导出普通对象当 Config → 加载失败。
- 默认值放 apply 内 → cordis.yml 无法覆盖。
- 非法配置静默兜底 → 违背 fail loudly。

## 深挖

完整精读/边界案例/全量代码骨架 → $(System.Collections.Hashtable[config-schema.md])（技能内内容层）。官方原文兜底走外链。
