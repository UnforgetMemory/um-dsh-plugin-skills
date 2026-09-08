# 构建一个工具 —— Build a Tool

> **来源**：<https://deepseek-harness.github.io/deepseek-harness/en/develop/basic/tool>
> 本页为精炼导读（中英对照），完整细节以官方原文为准。

## TL;DR（一句话概括）

用 `defineTool`（来自 `@deepseek-ai/dsh-tools`）+ `ctx.tools.register` 注册一个工具，`parameters` 定义入参、`output` 定义返回值 schema 与渲染方式、`execute` 返回规范值。

## 核心概念

### 工具插件骨架

```ts
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'greet-tool'
export const inject = ['tools']          // 等工具注册表就绪

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'greet',
    description: 'Greet someone by name.',
    parameters: {
      name: { type: 'string', required: true, description: 'The name to greet' },
    },
    output: {
      schema: { type: 'string' },                    // execute 返回的规范值（canonical value）
      render: (_args, value) => [{ type: 'text', text: value }],  // 转成模型可见内容
    },
    async execute(args) {
      return `Hello, ${args.name}!`                  // 按 output.schema 返回
    },
  }))
}
```

### 运行与调用

```sh
pnpm dsh web --patch ./scratch-plugin/cordis.yml
```

打开 <http://127.0.0.1:3080> 后提问：`Use the greet tool to greet Ada.` → 模型调用 `greet`，工具结果为 `Hello, Ada!`。

### 各部分职责

| 字段 | 职责 |
|------|------|
| `inject: ['tools']` | 让 Cordis 等工具注册表就绪再执行 |
| `parameters` | 定义入参 schema，`defineTool` 据此推断并校验 `args` |
| `output.schema` | 声明 `execute` 返回的规范值（canonical value）类型 |
| `output.render` | 把规范值转为模型可见的内容（文本/卡片等） |
| `execute` | 实际执行逻辑，返回规范值 |

## 术语对照

| 英文术语 | 中文 | 说明 |
|----------|------|------|
| defineTool | 工具定义 DSL | `@deepseek-ai/dsh-tools` 提供的工具声明器 |
| tools registry | 工具注册表 | 全局工具服务 `ctx.tools` |
| canonical value | 规范值 | `execute` 按 `output.schema` 返回的权威结果 |
| render | 渲染 | 规范值 → 模型可见内容（text/UI） |

## 陷阱与注意

- 忘写 `inject: ['tools']`，`ctx.tools` 未就绪。
- `execute` 返回值必须符合 `output.schema`；`render` 只负责展示形态，不改规范值。
- 命名空间/描述要贴合模型可理解性（名称简短、描述交代输入与意图）。

## 关联页面

- [插件配置](./config.md) —— 让问候语可配置
- [工具创作参考](https://deepseek-harness.github.io/deepseek-harness/en/reference/cookbook/adding-a-tool) —— 嵌套 schema、规范值、后台任务、策略钩子、PTC 模式、UI 卡片
- [能力分层](../practice/index.md) —— 把可替换能力拆成 Service Definition / Provider / Consumer 包