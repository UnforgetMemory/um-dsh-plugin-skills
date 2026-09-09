# 工具编写规范（defineTool）

## 骨架

```ts
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'greet-tool'
export const inject = ['tools']          // 必须：等工具注册表就绪

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'greet',                                // 工具名（模型可见）
    description: 'Greet someone by name.',         // 交代输入与意图
    parameters: {
      name: { type: 'string', required: true, description: 'The name to greet' },
    },
    output: {
      schema: { type: 'string' },                  // execute 返回的规范值类型
      render: (_args, value) => [{ type: 'text', text: value }],  // 规范值 → 模型可见内容
    },
    async execute(args) {
      return `Hello, ${args.name}!`                // 必须符合 output.schema
    },
  }))
}
```

## 规则

1. 必写 `inject: ['tools']`，否则 `ctx.tools` 未就绪。
2. `parameters` 定义入参；`defineTool` 据此推断并校验 `args`。
3. `execute` 返回规范值（canonical value），必须符合 `output.schema`。
4. `output.render` 只负责展示形态，不改规范值。
5. 注册是 effect：插件卸载自动注销。

## 进阶（详见官方 cookbook/adding-a-tool.md）

嵌套 schema、后台任务、策略钩子、PTC 模式、UI 卡片；工具结果经 `tools/result` 事件可被观察。

## 易错

- 忘 inject → `ctx.tools` 未定义。
- execute 返回值与 schema 不符 → 校验失败。
- 名称/描述含混 → 模型不调用。

## 深挖

完整精读/边界案例/全量代码骨架 → `guide/basic/tool.md`（技能内内容层）。官方原文兜底走外链。
