# practice 分册 —— 实战

## 前置

read `references/core-concepts.md`（必须）；建议先过 basic + framework 分册。

## 适用

三角色能力设计 / 动态 Cordis 扩展 / LLM 适配器接入。

## 操作流程

| 场景 | 做什么 | 读什么 |
|------|--------|--------|
| 能力需可替换提供者 | 三角色：Service Definition（契约+类型）→ Provider（实现）→ Consumer（工具） | `references/capability-layering.md` |
| agent 动态挂/卸插件 | 启用 `@deepseek-ai/dsh-tool-cordis`：`pnpm dsh web --patch apps/cli/config/examples/cordis/cordis.yml` | 官方 tool-cordis README |
| 接入新 LLM provider | 扩展 `LlmAdapter` 实现 `stream()`，`ctx.llm.registerAdapter(['prov'], adapter)` | `references/llm-adapter.md` |

## 要点速记

### 三角色

- 不预拆分：角色需独立演进/替换才拆独立包；简单工具插件不需要。
- 换 Provider = 改 cordis.yml 一行；契约与工具不变。
- 默认值显式 resolve()，不藏 `?? default`。

### 动态 Cordis

- 临时插件在内存中挂载/卸载，卸载或进程退出即消失；可能影响同进程其他会话。
- 需要模型凭据。

### LLM 适配器

- `class extends LlmAdapter` + `async *stream(options)`；StreamChunk：block-start/delta/block-end、usage、finish。
- 不能 honor 的字段抛 LlmError(code)，禁止静默丢弃；HTTP 请求合并 attributionHeaders() + 转发 options.signal。
- 参考实现：`packages/llm/llm-deepseek/`、`packages/llm/llm-pi-ai/`。

## 易错

- 能力没到可替换规模就拆三角色包 → 空抽象。
- 动态插件当持久配置用 → 进程退出即丢。
- LLM 适配器不回发完整 chunk 序列 / 丢 signal → 流解析失败、取消失效。

## 按需转向

机制疑问回 `professions/framework.md`；基础语法回 `professions/basic.md`；入口路由 `SKILL.md`；深挖精读 → `guide/practice/`：`index.md`（三角色能力）· `dynamic-cordis.md`（动态 Cordis）· `llm-adapter.md`（LLM 适配器）。