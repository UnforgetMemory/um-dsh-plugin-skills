# LLM 适配器规范（LlmAdapter + StreamChunk）

## 骨架

```ts
import Schema from '@deepseek-ai/schemastery'
import { LlmAdapter, type GenerateOptions, type StreamChunk } from '@deepseek-ai/dsh-llm'

class MyAdapter extends LlmAdapter {
  constructor(private apiKey: string) { super() }

  async *stream(options: GenerateOptions): AsyncIterable<StreamChunk> {
    // 1. options.messages → provider 格式
    // 2. 调流式 API（带 attributionHeaders() 与 options.signal）
    // 3. 响应 → StreamChunk 序列
  }
}

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

## StreamChunk 协议

```
block-start(0,text) → text-delta* → block-end(0,完整块)
block-start(1,tool-call) → tool-call-delta* → block-end(1,完整tool-call)
usage → finish
```

1. 每个 `block-start` 必配 `block-end`；`index` 从 0 递增。
2. `text-delta` 流式文本；`tool-call-delta` 的 `argumentsDelta` 携带原始 JSON 文本（可一次或分片），`id` 用 `brandString<ToolCallId>()`。
3. `finish` 是最后 chunk；`reason: { kind: 'stop' }` 或 `{ kind: 'tool-calls' }`（请求执行工具）。
4. `usage` 须在 `finish` 之前。

## GenerateOptions

- 含 model、adapter 自有 reasoning-effort id、对话历史、系统提示、工具 schema、生成参数、stop 序列、abort signal——以 `@deepseek-ai/dsh-llm` 导出类型为权威。
- 不能 honor 的字段抛 `LlmError(code)`（稳定 code），**禁止静默丢弃**。
- 可选 `resolveModel(provider, model, signal?)` 返回精确身份 + `context`/`reasoning` 元数据；`listModels()` 向选择器宣称模型。

## 注册与使用

```ts
ctx.llm.registerAdapter(['my-provider'], adapter)
```

```yaml
- id: my-llm
  name: './src/my-llm-adapter.ts'
  config:
    apiKey: !!js process.env.MY_API_KEY
    providers: [my-provider]
```

## 错误处理

- 传输/协议失败抛 `LlmError(code)`；agent loop 保留错误与 code，不自动转换普通 Error。
- 每个 provider HTTP 请求合并 `attributionHeaders()` 并转发 `options.signal`。

## 参考实现（仓库内对照）

- `packages/llm/llm-deepseek/`（OpenAI 兼容格式）
- `packages/llm/llm-pi-ai/`（异构格式）

## 易错

- 忘 `block-end` / 顺序错乱 → 流解析失败。
- 不支持字段却静默丢弃 → 违反契约。
- 不转发 signal → 取消/释放无法到 quiescence。

## 深挖

完整精读/边界案例/全量代码骨架 → $(System.Collections.Hashtable[llm-adapter.md])（技能内内容层）。官方原文兜底走外链。
