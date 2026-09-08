# 你的第一个插件 —— Your First Plugin

> **来源**：<https://deepseek-harness.github.io/deepseek-harness/en/develop/basic/>
> 本页为精炼导读（中英对照），完整细节以官方原文为准。

## TL;DR（一句话概括）

Harness 插件 = 一个导出 `apply(ctx)` 函数的 TypeScript 模块；框架加载时调用它，插件通过 `ctx` 注册一切能力（事件监听/工具/定时器），卸载时自动清理。

## 核心概念

### 什么是插件（What is a plugin）

- 插件是导出 `apply` 函数的 TS 模块，框架调用时传入 `ctx`（Context）对象，插件经 `ctx` 注册能力。
- 最小完整配置：

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'my-plugin'          // 插件名
export function apply(ctx: Context) {
  // 在这里注册能力
}
```

### 加载进 Web UI（三个步骤）

1. 项目根建 `scratch-plugin/src/`，写 `my-plugin.ts`（如上模板，加一行 `console.log`）。
2. 写 `scratch-plugin/cordis.yml`（Web overlay）插入本地插件：

```yaml
- insert:
    - id: hello
      name: '/absolute/path/to/deepseek-harness/scratch-plugin/src/my-plugin.ts'
```

> ⚠️ 插件路径**必须是绝对路径**——patch 文件只贡献配置，不会改变模块解析的 profile 目录。

3. 启动：`pnpm dsh web --patch ./scratch-plugin/cordis.yml`，打开 <http://127.0.0.1:3080>，终端应打印 `[hello-plugin] plugin loaded!`。

### 自动清理（Automatic cleanup）

- 通过 `ctx` 注册的一切（事件监听、工具、定时器）在插件卸载时**自动清理**，无需手动 `removeListener` / `clearInterval`。
- 需要显式释放的资源（如网络连接）用 `ctx.effect()` 提供 disposer，返回的函数在卸载时执行：

```ts
export function apply(ctx: Context) {
  ctx.effect(() => {
    const timer = setInterval(() => console.log('heartbeat'), 5000)
    return () => clearInterval(timer)   // 卸载时执行
  })
}
```

### 声明依赖（Declare dependencies）

- 消费 `tools` / `llm` 等服务时，用 `inject` 声明；框架会**等所有必需服务就绪后才加载插件**：

```ts
export const name = 'my-tool-plugin'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(/* ... */)   // ctx.tools 此时已就绪
}
```

### 三种插件形态（Three plugin forms）

| 形态 | 适用 | 说明 |
|------|------|------|
| Function 函数形式 | 大多数情况 | `export function apply(ctx)` |
| Object 对象形式 | 同函数 | `export default { name, inject, apply }` |
| Class 类形式 | 需要向其他插件提供服务时 | `class MyService extends Service`，构造器做同步初始化、`super(ctx, 'myService')` |

## 术语对照

| 英文术语 | 中文 | 说明 |
|----------|------|------|
| plugin | 插件 | 导出 `apply` 的 TS 模块 |
| apply(ctx) | 应用函数 | 框架加载插件时的入口 |
| ctx (Context) | 上下文 | 注册能力的通道 |
| inject | 注入声明 | 声明所需服务，框架等待其就绪 |
| ctx.effect() | 效果/副作用注册 | 注册带 disposer 的资源，卸载自动回收 |
| overlay / patch | 覆盖层/补丁 | 通过 `--patch` 提供的配置层 |
| cordis.yml | 组合配置文件 | 声明插件行（id + name + config） |

## 陷阱与注意

- patch 中插件路径必须**绝对路径**，否则模块解析失败。
- 不要手动清理 `ctx` 注册的内容——框架自动做；只有 `ctx.effect()` 管理的资源需提供 disposer。
- 插件需要服务时必须 `inject`，不要在 `apply` 里假设 `ctx.tools` 已存在。

## 关联页面

- [构建工具](./tool.md) —— 工具定义 DSL
- [插件配置](./config.md) —— 接收用户配置
- [Cordis 教程](../cordis-tutorial/index.md) —— 底层插件框架（无需 API key，scratch 目录起手）