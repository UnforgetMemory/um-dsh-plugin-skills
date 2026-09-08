# basic 分册 —— 插件入门（路径 A：仓库级源码开发）

> ⚠️ 本分册引导 **deepseek-harness 源码 checkout 内开发**（TS + cordis.yml + `pnpm dsh web --patch`）。若目标是**当前会话内动态创建插件**（cordis_define/cordis_run、纯 JS），走 `references/dynamic-cordis.md`——两条路径形态不兼容，勿混用。

## 前置

read `references/core-concepts.md`（必须）。

## 适用

第一个插件 / 写插件 / 加载进 Web UI / cordis.yml 注册 / apply 入门 / 三种插件形态。

## 操作流程

1. 建 scratch 项目 `scratch-plugin/src/`，写 `my-plugin.ts`：

```ts
import type { Context } from '@deepseek-ai/cordis'
export const name = 'hello-plugin'
export function apply(ctx: Context) {
  console.log('[hello-plugin] plugin loaded!')
}
```

2. 写 `scratch-plugin/cordis.yml` 插入插件行：

```yaml
- insert:
    - id: hello
      name: '/absolute/path/to/deepseek-harness/scratch-plugin/src/my-plugin.ts'
```

3. 启动：`pnpm dsh web --patch ./scratch-plugin/cordis.yml`，打开 http://127.0.0.1:3080，终端应打印加载日志。

4. 消费服务时声明依赖：`export const inject = ['tools']` → `ctx.tools` 就绪。

5. 需自定义清理：`ctx.effect(() => { ...; return () => cleanup })`。

## 自动清理

ctx 注册的一切（事件/工具/定时器）卸载时自动清理，不要手动 removeListener/clearInterval。需显式释放的资源用 ctx.effect() 返回 disposer。

## 易错

- patch 中插件路径必须绝对路径，否则模块解析失败。
- 用服务却不 inject → ctx.tools 不存在。
- 手动清理 ctx 注册物 → 多余且可能破坏框架账本。

## 按需转向

工具 → `references/tool-authoring.md`；配置 → `references/config-schema.md`；打包 → `references/publish-distribution.md`；机制 → framework 分册；Cordis 底层 → tutorial 分册；深挖精读 → `guide/basic/<页>.md`。