# tutorial 分册 —— Cordis 教程路线

## 前置

read `references/core-concepts.md`（必须）。自 scratch 目录起手，无需 API key；可插入 basic/framework/practice 任意阶段。

## 环境准备

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install
mkdir -p tmp/cordis-tutorial   # tmp/ 已 gitignore，改动不进版本控制
cd tmp/cordis-tutorial
```

每章统一启动：`node --import tsx ../../vendor/cordis/bin.js`（建根 Context、挂 Loader 读 `./cordis.yml`；`--import tsx` 免构建跑 TS）。

## 章节路线（逐章动手）

| 章 | 主题 | 要点 |
|----|------|------|
| 0 | 环境准备 | 如上；launcher 机制 |
| 1 | 第一个插件 | 插件 = 导出 `apply(ctx)` 的函数；`name` 可选诊断元数据；cordis.yml 条目**并发**启动、文件位置不保证顺序；三种形态（Function/Object/Class） |
| 2 | 生命周期与 effect | 经 Cordis API 的注册都是 effect 自动撤销；Cordis 不管的资源（定时器/连接）用 `ctx.effect()` 包住返回 disposer；`ctx.plugin(child)` 返回 Fiber，`fiber.dispose()` 递归清理；状态机 PENDING→LOADING→ACTIVE→UNLOADING→DISPOSED（失败 ↘FAILED） |
| 3 | 服务 | 提供：`class X extends Service` + `super(ctx,'name')`；`declare module` 声明合并只补类型不接线；消费：`inject` 声明硬依赖（决定启动时机）；服务消失→卸载、回归→重载；可选依赖 `ctx.get('name')` |
| 4 | 事件 | `interface Events` 声明合并 + `ctx.emit`/`ctx.on` 类型化；命名 `namespace/action`；**五种派发**：emit/parallel/serial/bail/waterfall；waterfall 只观察必须调 `next()`，否则短路；harness 决策点 `agent/request`、`approval/request` |
| 5 | 配置 | 条目带 `config`；导出同名 Config interface + Schemastery Schema，apply 前校验，坏配置大声失败（Fiber→FAILED，launcher 退出码 1）；`apply(ctx, config)` 第二参恒为完整校验配置；**禁止导出普通对象当 Config**；`!!js` 只在 config 与 disabled 内计算加载期值 |
| 6 | 组合与 HMR | `id` 稳定身份（区分「改」与「删+增」）；`disabled: true` 保留条目跳过挂载；groups 分组、isolate 组内独立服务；HMR 需 `@deepseek-ai/cordis-plugin-hmr` + `logger-console` + `timer`（缺了永远 PENDING）；诊断：遍历 `ctx.registry` 找 `FiberState.PENDING` |
| 7 | 接入 Harness | `inject: ['tools']` + `ctx.tools.register(defineTool({...}))` 注册模型可调工具；defineTool 转 JSON Schema、推断 args、execute 前校验、返回 output.schema 规范值 + output.render 展示；观察 `ctx.on('tools/result')`（execute resolve 前发出）；组合需列 `@deepseek-ai/dsh-tools` + `@deepseek-ai/dsh-system-prompt` |

## 最小代码骨架

```ts
// hello.ts —— 插件 = apply(ctx) 函数（章 1）
import type { Context } from '@deepseek-ai/cordis'
export const name = 'hello'
export function apply(ctx: Context) {
  console.log('hello from my first plugin')
}
```

```ts
// 自定义清理（章 2）：ctx.effect 返回 disposer，卸载时调用
ctx.effect(() => {
  const timer = setInterval(() => console.log('tick'), 200)
  return () => clearInterval(timer)
})
```

## 易错

- tmp/ 已 gitignore：scratch 改动不进版本控制，放心实验。
- 声明合并只类型不接线：提供方仍要 super(ctx,'name') 注册、事件仍要 ctx.emit。
- session 事件 ≠ Cordis 事件：DSH 会话事件与 ctx.on/ctx.emit 是两套体系。
- 条目并发启动：依赖决定顺序，文件位置不保证先后。
- PENDING 是合法状态（服务可能后挂）：「静默无输出」先查 fiber 状态与拼写。
- 忘记 next() 吞掉 waterfall 下游：只观察/注解的监听器必须 next()。
- !!js 只在 config 与 disabled 生效；name/id/inject 里是普通数据。

## 按需转向

入口：`SKILL.md`；公共前置：`references/core-concepts.md`；机制深潜：`professions/framework.md`；接入实战：`professions/practice.md`；深挖精读：`guide/cordis-tutorial/`——`index.md`（总览）+ `01-first-plugin.md` … `07-into-the-harness.md`（逐章一文件）；官方原文：<https://deepseek-harness.github.io/deepseek-harness/en/develop/cordis-tutorial/>。