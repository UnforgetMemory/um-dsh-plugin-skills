# 三角色能力设计规范

## 模式

能力足够通用、需可替换实现时拆三角色；**独立演进/替换时才拆独立包**（简单工具插件不需要）：

```
Service Def(契约) ←─ Provider(实现) ──┐
     ▲                                 ├─ 互不依赖，只依赖 Def
     └────────── Consumer(工具) ───────┘
```

- **Service Definition**：定义 Cordis 服务 + Request/Result 类型。
- **Service Provider**：实现服务，执行真实行为。
- **Consumer**：把能力暴露为模型可调用工具。
- Provider 与 Consumer **互不依赖**，只依赖 Definition 包。

## 收益

1. 换 Provider：`cordis.yml` 换一行 → 换实现，契约与工具不变。
2. 独立演进：契约低频变化；Provider 单独优化性能/安全；Consumer 单独改呈现。
3. 解耦依赖：Provider→Def，Consumer→Def。

## 三个文件骨架

```ts
// 1) Service Definition（抽象类 + 类型）
export abstract class MyCapService extends Service {
  constructor(ctx: Context) { super(ctx, 'myCap') }
  abstract execute(request: MyCapRequest): Promise<MyCapResult>
}
declare module '@deepseek-ai/cordis' {
  interface Context { myCap: MyCapService }
}
```

```ts
// 2) Service Provider（实现抽象类）
class MyCapLocal extends MyCapService {
  async execute(request: MyCapRequest): Promise<MyCapResult> {
    return { output: request.input.toUpperCase() }
  }
}
export function apply(ctx: Context) { ctx.plugin(MyCapLocal) }
```

```ts
// 3) Consumer（defineTool 包一层）
export const inject = ['tools', 'myCap']
// execute: ctx.myCap.execute({ input: args.input }).then(r => r.output)
```

```yaml
# cordis.yml 组合
- name: '@deepseek-ai/dsh-my-cap-local'
- name: '@deepseek-ai/dsh-tool-my-cap'
```

## 设计要点

- **不预拆分**：只有角色需独立演进才拆包。
- **契约拥有 Request/Result 类型**：Provider 与 Consumer 只依赖 Definition 包。
- **显式 > 隐式**：默认值在显式 `resolve(request): Spec` 步骤解析，不藏进 `run()` 的 `?? default`。
- 内置参考：`dsh-shell`（Def）· `dsh-bash-local`（Prov）· `dsh-tool-bash`（Consumer）。

## 易错

- 过早拆包 → 空抽象层负担。
- Provider/Consumer 互相 import → 失去替换能力。

## 深挖

完整精读/边界案例/全量代码骨架 → $(System.Collections.Hashtable[capability-layering.md])（技能内内容层）。官方原文兜底走外链。
