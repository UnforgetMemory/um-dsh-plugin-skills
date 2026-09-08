# LLM 适配器 —— LLM Adapters

> **来源**：<https://deepseek-harness.github.io/deepseek-harness/en/develop/practice/llm-adapter>
> 本页为精炼导读（中英对照），完整细节以官方原文为准。

## TL;DR（一句话概括）

LLM 适配器继承 `LlmAdapter` 并实现 `stream()`：把 Harness 的 provider 无关请求翻译成具体 provider 的 API 调用，再把响应翻译回 Harness 的 `StreamChunk` 块序列；通过 `ctx.llm.registerAdapter()` 注册即可接入新 provider。

## 核心概念
### 最小实现（Minimal implementation）
```ts
import type { Context } from '@deepseek-ai/cordis'
import Schema from '@deepseek-ai/schemastery'
import { LlmAdapter, type GenerateOptions, type StreamChunk } from '@deepseek-ai/dsh-llm'

class MyAdapter extends LlmAdapter {
  constructor(private readonly apiKey: string) { super() }
  async *stream(options: GenerateOptions): AsyncIterable<StreamChunk> {
    // 1. 把 options.messages 转成 provider 格式；2. 调用流式 API；3. 转成 StreamChunk
  }
}

export interface Config { apiKey: string; providers: string[] }
export const Config: Schema<Config> = Schema.object({
  apiKey: Schema.string().required(),
  providers: Schema.array(Schema.string()).required(),
})

export const name = 'my-llm-adapter'
export const inject = ['llm']

export function apply(ctx: Context, config: Config) {
  ctx.llm.registerAdapter(config.providers, new MyAdapter(config.apiKey))
}
```

### StreamChunk 协议（StreamChunk protocol）
`stream()` 按以下协议逐块产出：

```ts
async function* exampleChunks(): AsyncIterable<StreamChunk> {
  yield { type: 'block-start', index: 0, blockType: 'text' }            // 1. 块开始
  yield { type: 'text-delta', index: 0, text: 'Hello' }                 // 2. 文本增量
  yield { type: 'text-delta', index: 0, text: ' world' }
  yield { type: 'block-end', index: 0, block: { type: 'text', text: 'Hello world' } }  // 3. 块结束 + 完整块
  yield { type: 'block-start', index: 1, blockType: 'tool-call' }       // 4. 工具调用块
  yield { type: 'tool-call-delta', index: 1, id: brandString('call-123'), name: 'bash', argumentsDelta: '{"command":"ls"}' }
  yield { type: 'block-end', index: 1, block: { type: 'tool-call', id: brandString('call-123'), name: 'bash', arguments: '{"command":"ls"}' } }
  yield { type: 'usage', usage: { inputTokens: 100, outputTokens: 50 } } // 5. 用量
  yield { type: 'finish', reason: { kind: 'stop' } }                     // 6. 结束（或 { kind: 'tool-calls' }）
}
```

关键规则（Key rules）：
- 每个 `block-start` 必须有配对的 `block-end`。
- `index` 从 0 递增，标识内容块顺序。
- `tool-call-delta` 的 `argumentsDelta` 携带原始 JSON 文本，可一次或分多块传输。
- `finish` 是最后一块；`usage` 必须在 `finish` 之前产出。

### GenerateOptions
`stream()` 收到 `GenerateOptions` 类型（含 model、adapter 自有 reasoning-effort id、对话历史、system prompt、工具 schema、生成参数、停止序列、abort signal）；以 `@deepseek-ai/dsh-llm` 导出的 TS 类型为准，把支持的字段映射到 provider API，无法满足的字段抛带稳定 code 的 `LlmError`，不要静默丢弃。

覆写 `resolveModel(provider, model, signal?)`，在一次查找中返回精确的 provider/model 身份及可选 `context`、`reasoning` 元数据；尊重可选 signal 以便异步查找能收敛取消与清理。未覆写 `reasoning` 表示该模型无 reasoning-effort 能力。

### 注册适配器（Register an adapter）
`ctx.llm.registerAdapter(['my-provider'], adapter)` 的第一个参数列出该适配器处理的 provider 路由；`GenerateOptions.provider` 选择已注册适配器，`GenerateOptions.model` 传 adapter 自有的 model id（无需生命周期注册）。需要向选择器展示可选项时覆写 `listModels()`。

### 从 cordis.yml 使用（Use it from cordis.yml）
```yaml
- id: my-llm
  name: './src/my-llm-adapter.ts'
  config:
    apiKey: !!js process.env.MY_API_KEY
    providers: [my-provider]

- id: agent-loop
  name: '@deepseek-ai/dsh-agent-loop'
  config:
    agents:
      - id: main
        provider: my-provider
        model: my-model-v1
```

### 错误处理（Error handling）
适配器用带稳定 code 的 `LlmError` 抛出传输/协议错误；agent loop 保留错误与 code 用于诊断与策略，但不会自动转换普通 `Error`。每次 provider HTTP 请求必须合并 `attributionHeaders()` 并转发 `options.signal`：

```ts
import { attributionHeaders, LlmAdapter, LlmError, type GenerateOptions, type StreamChunk } from '@deepseek-ai/dsh-llm'

class HttpAdapter extends LlmAdapter {
  constructor(private readonly endpoint: string) { super() }
  async *stream(options: GenerateOptions): AsyncIterable<StreamChunk> {
    const response = await fetch(this.endpoint, {
      method: 'POST',
      headers: { 'content-type': 'application/json', ...attributionHeaders() },
      body: JSON.stringify({ model: options.model, messages: options.messages }),
      ...options.signal ? { signal: options.signal } : {},
    })
    if (!response.ok) throw new LlmError(`Provider API error: ${response.status}`, 'PROVIDER_HTTP_ERROR')
    yield { type: 'finish', reason: { kind: 'stop' } }   // 真实适配器会解析响应并产出完整块序列
  }
}
```

## 术语对照
| 英文术语 | 中文 | 说明 |
|----------|------|------|
| LlmAdapter | LLM 适配器基类 | 继承并实现 `stream()` 接入 provider |
| stream() | 流式生成 | 产出 `StreamChunk` 的异步迭代器 |
| StreamChunk | 流式块 | block-start / text-delta / block-end / usage / finish 等 |
| GenerateOptions | 生成选项 | provider 无关的生成请求 |
| registerAdapter | 注册适配器 | 把 provider 路由绑定到适配器 |
| resolveModel | 模型解析 | 返回精确 provider/model 身份与元数据 |
| LlmError | 适配器错误 | 带稳定 code 的错误 |
| attributionHeaders | 归因头 | 需合并进每次 HTTP 请求 |

## 陷阱与注意
- `block-start` 必须与 `block-end` 配对，`usage` 先于 `finish`，`finish` 是最后一块。
- 无法满足的字段要抛带稳定 code 的 `LlmError`，别静默丢弃；普通 `Error` 不会被 agent loop 自动转换。
- provider HTTP 请求务必合并 `attributionHeaders()` 并转发 `options.signal`。
- `argumentsDelta` 是原始 JSON 文本（非已解析对象），工具调用参数靠它拼装。

## 关联页面
- [三角色能力设计](../practice/index.md) —— LLM 是同一“能力接缝”的实例
- [用 Cordis 工具扩展运行中的 Agent](../practice/dynamic-cordis.md)
- [你的第一个插件](../basic/index.md) —— 前置基础
- 官方参考实现：[llm-deepseek](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm-deepseek/README.md)（OpenAI 兼容格式）、[llm-pi-ai](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm-pi-ai/README.md)（不同 API 格式）
