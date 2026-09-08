# 你的第一个插件 —— Your First Plugin

> **来源**：<https://deepseek-harness.github.io/deepseek-harness/en/develop/cordis-tutorial/01-first-plugin>
> 本页为精炼导读（中英对照），完整细节以官方原文为准。

## TL;DR（一句话概括）

Cordis 插件模块 named-export 一个 `apply` 函数；加载时 Cordis 用 **Context（`ctx`）** 调用它，插件经 `ctx` 注册自己贡献的一切——本页写出第一个插件并用 `cordis.yml` 组合运行。

## 核心概念

### 写插件（Write the plugin）

在 `tmp/cordis-tutorial` 创建 `hello.ts`：

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'hello'   // 可选：诊断用的显示元数据

export function apply(ctx: Context) {
  console.log('hello from my first plugin')
}
```

### 组合应用（Compose the app）

launcher 从配置组装应用，创建 `cordis.yml`：

```yaml
- name: './hello.ts'   # name 是模块标识符：相对路径或 npm 包名
```

- 文件是插件条目列表，loader 挂载每一个条目。
- 条目**并发启动**，列表位置不保证加载顺序——顺序来自服务依赖（`inject`，见第 3 章），而非文件位置。

### 运行（Run it）

```sh
node --import tsx ../../vendor/cordis/bin.js
# hello from my first plugin
```

发生的事：① launcher 创建根 `Context` 并挂载 Loader；② Loader 读 `cordis.yml`、解析 `./hello.ts` 并挂载为子插件；③ Cordis 调用你的 `apply(ctx)`。文件中没有框架引导代码——插件只描述自己贡献什么，`cordis.yml` 负责组合应用。

### 另外两种插件形态（The two other plugin shapes）

函数是最常见形态，Cordis 共接受三种：

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

// 1. 函数插件（你刚写的）
export function apply(ctx: Context) {}

// 2. 对象插件：带 apply 方法的对象
export const objectPlugin = {
  name: 'object-plugin',
  apply(ctx: Context) {},
}

// 3. 类插件：Service 子类（第 3 章详述）
export class MyService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'myTutorialService')
  }
}
```

- 需要暴露服务前一直用函数形态即可。

### 故意破坏它（Try breaking it）

让 `apply` 抛错：

```ts
export function apply(ctx: Context) {
  throw new Error('apply exploded')
}
```

- 再次运行：进程随错误崩溃——加载失败的插件是**响亮失败**，不是被跳过的条目。
- 一个值得早知的坑：模块**无法解析**（路径/包名拼错）的条目经 Cordis logger 服务报告而**不崩溃进程**，boot 时该报告可能在 console exporter 开始监听前丢失——新加的条目看似无反应时，先检查拼写。

## 术语对照

| 英文术语 | 中文 | 说明 |
|----------|------|------|
| plugin | 插件 | named-export `apply` 的模块 |
| apply(ctx) | 应用函数 | 框架加载插件时的入口 |
| ctx (Context) | 上下文 | 注册一切贡献的通道 |
| name export | 名称导出 | 可选显示元数据，标注诊断 |
| cordis.yml | 组合配置 | 插件条目列表 |
| loader | 加载器 | 读取并挂载配置条目 |
| function/object/class plugin | 函数/对象/类插件 | 三种插件形态 |

## 陷阱与注意

- 条目并发启动，**列表位置不保证加载顺序**；顺序来自 `inject` 依赖而非文件顺序。
- `apply` 抛错是响亮失败，进程直接崩溃。
- 模块**解析失败**（拼写错误）经 logger 报告、不崩溃进程，且 boot 时报告可能丢失——"看似没反应"先查拼写。

## 关联页面

- [生命周期与副作用](../cordis-tutorial/02-lifecycle-and-effects.md) —— 插件卸载时发生什么
- [服务](../cordis-tutorial/03-services.md) —— 何时该用类形态暴露服务
