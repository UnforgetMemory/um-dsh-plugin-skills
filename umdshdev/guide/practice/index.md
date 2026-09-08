# 三角色能力设计 —— Three-role Capability Design

> **来源**：<https://deepseek-harness.github.io/deepseek-harness/en/develop/practice/>
> 本页为精炼导读（中英对照），完整细节以官方原文为准。

## TL;DR（一句话概括）

当一个能力（如 Bash 执行）足够通用、需要可替换的提供者时，Harness 把它拆成三个角色——服务定义（Service Definition）、服务提供者（Service Provider）、消费者（Consumer），三者经服务契约解耦，可独立演进、独立替换。

## 核心概念
### 三个角色（Three roles）
- **服务定义（Service Definition）**：定义 Cordis 服务及请求/结果类型。
- **服务提供者（Service Provider）**：实现具体行为（如本地执行命令）。
- **消费者（Consumer）**：把能力暴露为模型可调用的工具。

> 三者组成的**完整能力才是接缝（seam）**，单个角色不是接缝。只有在需要独立演进或替换时才拆到不同包；一个包也可以承担多个角色。

### Bash 示例（Bash example）
```
dsh-shell(定义) ──▶ dsh-bash-local(提供者)        dsh-tool-bash(消费者/工具)
      ▲────────────────────────────────────────────────┘
                            inject: ['shell']
```

### 拆分的收益（Benefits of the split）
- **可替换提供者（Replace providers）**：一个服务定义可有多个提供者，在 `cordis.yml` 里换一行即可，定义与工具不变：

```yaml
- name: '@deepseek-ai/dsh-bash-local'   # 本地执行；换成其他提供同一服务的包即可
```

- **独立演进（Evolve independently）**：定义契约很少变；提供者可独立优化性能/安全；消费者可独立改变向模型呈现能力的方式。
- **解耦依赖（Decouple dependencies）**：提供者依赖定义，消费者依赖定义，但提供者与消费者**互不依赖**。

### 教程：开发一个三角色能力（Tutorial）

#### 第 1 步：服务定义（Service Definition）
```ts
// packages/my-cap/my-cap/src/index.ts
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' { interface Context { myCap: MyCapService } }  // 让 ctx.myCap 可用

export abstract class MyCapService extends Service {
  constructor(ctx: Context) { super(ctx, 'myCap') }
  abstract execute(request: MyCapRequest): Promise<MyCapResult>
}
export interface MyCapRequest { input: string }
export interface MyCapResult { output: string }
```

#### 第 2 步：服务提供者（Service Provider）
```ts
// packages/my-cap/my-cap-local/src/index.ts
import type { Context } from '@deepseek-ai/cordis'
import { MyCapService, type MyCapRequest, type MyCapResult } from '@deepseek-ai/dsh-my-cap'

class MyCapLocal extends MyCapService {
  async execute(request: MyCapRequest): Promise<MyCapResult> {
    return { output: request.input.toUpperCase() }   // 本地提供者行为
  }
}
export const name = 'my-cap-local'
export function apply(ctx: Context) { ctx.plugin(MyCapLocal) }
```

#### 第 3 步：消费者（Consumer）
```ts
// packages/my-cap/tool-my-cap/src/index.ts
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'tool-my-cap'
export const inject = ['tools', 'myCap']   // 声明依赖

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'my_cap',
    description: 'Execute my capability.',
    parameters: { input: { type: 'string', required: true } },
    output: { schema: { type: 'string' }, render: (_a, v) => [{ type: 'text', text: v }] },
    async execute(args) {
      const result = await ctx.myCap.execute({ input: args.input })
      return result.output
    },
  }))
}
```

#### 在 cordis.yml 中组合
```yaml
- name: '@deepseek-ai/dsh-my-cap-local'
- name: '@deepseek-ai/dsh-tool-my-cap'
```

### 设计要点（Design points）
- **不要过早拆分**：只有角色需要独立演进时才拆包；一个简单的工具插件不需要。
- **服务定义拥有 Request/Result 类型**：提供者与消费者只依赖服务定义包。
- **显式优于隐式**：在显式的 `resolve(request): Spec` 步骤解析默认值，而不是把 `?? default` 藏在 `run()` 里。

## 术语对照
| 英文术语 | 中文 | 说明 |
|----------|------|------|
| Service Definition | 服务定义 | 定义 Cordis 服务及请求/结果类型 |
| Service Provider | 服务提供者 | 实现能力的具体行为 |
| Consumer | 消费者 | 把能力暴露为模型可调用工具 |
| capability | 能力 | 一个完整功能（如 Bash 执行） |
| seam | 接缝 | 能力的完整边界，可替换/演进的单元 |
| inject | 注入声明 | 声明所需服务，框架等待其就绪 |
| defineTool | 工具定义 | 声明模型可调用工具 |
| ctx.plugin() | 插件挂载 | 把一个 Service 类注册为插件 |

## 陷阱与注意
- 接缝是三个角色组成的完整能力，别把单个角色误当成接缝。
- 提供者与消费者只依赖服务定义包，不要让它们互相依赖。
- 简单工具插件不必强行三拆，拆包要基于“独立演进”的真实需要。
- Request/Result 类型必须由服务定义导出，避免提供者与消费者之间出现隐式耦合。

## 关联页面
- [LLM 适配器](../practice/llm-adapter.md) —— 实现一个 LLM 提供者（官方“下一步”）
- [用 Cordis 工具扩展运行中的 Agent](../practice/dynamic-cordis.md) —— 本组另一实践
- [你的第一个插件](../basic/index.md) —— 前置：基础插件路径
- [能力接缝参考（官方完整版）](https://deepseek-harness.github.io/deepseek-harness/en/reference/capability-seams) —— 内置能力家族与包链接
