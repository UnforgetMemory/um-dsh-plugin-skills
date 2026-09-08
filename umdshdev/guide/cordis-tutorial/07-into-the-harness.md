# 接入 Harness —— Into the Harness

> **来源**：<https://deepseek-harness.github.io/deepseek-harness/en/develop/cordis-tutorial/07-into-the-harness>
> 本页为精炼导读（中英对照），完整细节以官方原文为准。

## TL;DR（一句话概括）

本章用 Harness 的 `tools` 服务注册一个可被模型调用的工具，经 Harness 工具管线执行它并观察结果事件——全程无 key、不调模型。

## 核心概念

### 工具插件（A tool plugin）

创建 `greet-tool.ts`：

```ts
import type { Context } from '@deepseek-ai/cordis'
import { brandString } from '@deepseek-ai/dsh-brand'
import { defineTool } from '@deepseek-ai/dsh-tools'
import type { ToolCallId } from '@deepseek-ai/dsh-llm'

export const name = 'greet-tool'
export const inject = ['tools']   // 工具注册表就绪前保持 PENDING

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({   // register 返回的 disposer 挂到插件，卸载即注销
    name: 'greet',
    description: 'Greet the named person.',
    parameters: {
      name: { type: 'string', required: true, description: 'Who to greet' },
    },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args) {
      return `Hello, ${args.name}!`
    },
  }))

  // 驱动一次真实执行管线，代替模型；ToolCallId 标记提供者会下发的关联 id
  void (async () => {
    const result = await ctx.tools.execute({
      callId: brandString<ToolCallId>('demo-1'),
      name: 'greet',
      arguments: { name: 'Cordis' },
      signal: new AbortController().signal,
    })
    console.log('tool replied:', JSON.stringify(result.content))
  })()
}
```

- 每个模式都来自前几章：`inject: ['tools']`（第 3 章）在工具注册表存在前挂起插件；`ctx.tools.register(...)`（第 2 章）把注册 disposer 挂到插件，卸载即注销工具。`defineTool` 把 `parameters` 规范转成给模型看的 JSON Schema、推断 `args` 类型、在 `execute` 前校验模型提供的参数。工具返回 `output.schema` 声明的规范值；`output.render` 单独产出 Native 与持久化结果内容。

### 观察者插件（An observer plugin）

创建 `tool-logger.ts`——独立插件，经 Harness 的 `tools/result` 事件观察应用的每次工具调用：

```ts
import type { Context } from '@deepseek-ai/cordis'
import type {} from '@deepseek-ai/dsh-tools'   // 拉入声明合并，使事件与负载类型化

export const name = 'tool-logger'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.on('tools/result', (exec, result) => {
    const text = result.content
      .map(block => (block.type === 'text' ? block.text : ''))
      .join('')
    console.log(`[tool-logger] ${exec.name} -> ${text}`)
  })
}
```

- `import type {} from '@deepseek-ai/dsh-tools'` 拉入该包的声明合并，使 `'tools/result'` 及其负载类型化——第 4 章 `stats.ts` 导入的包级版本。

### 组合与运行（Compose and run）

```yaml
- name: '@deepseek-ai/dsh-system-prompt'
- name: '@deepseek-ai/dsh-tools'
- name: './tool-logger.ts'
- name: './greet-tool.ts'
```

- `@deepseek-ai/dsh-tools` 注入 `systemPrompt` 服务（工具要把 schema 贡献进系统提示），所以组合列出其提供者；缺它则 tools 插件按第 6 章所述保持 PENDING。

```sh
node --import tsx ../../vendor/cordis/bin.js
```

```
[tool-logger] greet -> Hello, Cordis!
tool replied: [{"type":"text","text":"Hello, Cordis!"}]
```

- logger 先触发：`tools/result` 在结果物化期间发出，早于 `execute` 的 promise 对调用者 resolve。两个插件互不知道对方存在——注册表服务与事件把二者连起来。

### 从这里走向完整 agent（From here to a full agent）

真正的 agent 就是这份组合再加插件：LLM 适配器、agent 主循环、持久化、应用入口。对比 base profile 层与 headless 层的 `cordis.patch.yml`——现在能读懂它们的条目了；用小 `--patch` overlay 加入你的 `greet-tool.ts`。

进一步去向：

- [构建工具](../basic/tool.md) —— 更多 `defineTool`，含展示与更丰富的 schema
- [三层能力设计](../practice/index.md) —— Harness 如何构建可替换能力
- 生成的 `cordis-surface` 区域（子系统页面）—— 你能 inject 与监听的一切
- 架构 —— 这些插件所在的系统地图

## 术语对照

| 英文术语 | 中文 | 说明 |
|----------|------|------|
| tools service | 工具服务 | Harness 的工具注册表/执行管线 |
| defineTool | 工具定义 | 把参数规范转成 JSON Schema 并推断类型 |
| ctx.tools.register | 工具注册 | 返回 disposer，卸载即注销 |
| ctx.tools.execute | 工具执行 | 走真实执行管线 |
| output.schema / output.render | 输出模式/渲染 | 规范返回值 / 产出展示内容 |
| tools/result event | 结果事件 | 结果物化期间发出 |
| ToolCallId | 调用关联 id | 提供者会下发的品牌化 id |

## 陷阱与注意

- `@deepseek-ai/dsh-tools` `inject` `systemPrompt`，缺该提供者则 tools 插件永久 PENDING——组合里要一起列。
- `tools/result` 在 `execute` 的 promise resolve 前发出——logger 先打印。
- `ctx.tools.register(...)` 的 disposer 挂到插件，卸载自动注销工具，无需手动清理。
- 两个插件互不知道对方——由注册表服务与事件连接，别在插件间直接引用对方。

## 关联页面

- [组合与 HMR](../cordis-tutorial/06-composition-and-hmr.md) —— 上一步的组合与诊断
- [构建工具](../basic/tool.md) —— 更多 defineTool 用法
