# Cordis 教程 —— Cordis Tutorial

> **来源**：<https://deepseek-harness.github.io/deepseek-harness/en/develop/cordis-tutorial/>
> 本页为精炼导读（中英对照），完整细节以官方原文为准。

## TL;DR（一句话概括）

Cordis 是 DeepSeek Harness 底层的插件框架——一个微型运行时，工具、LLM 适配器、文件访问乃至 agent 主循环本身，都是挂载进共享 Context 的插件；本教程在仓库 scratch 目录里逐章构建可运行示例，最终把插件接入真实 Harness 服务，全程无需 API key。

## 核心概念

### 教程定位（What this is）

- 受众是 agent 开发者，不要求深厚的 TypeScript 经验；每章给出确切命令与预期输出。
- 三条阅读路径：**Cordis primer** 是浓缩概念速览；**本教程**是动手走查（scratch 目录起手）；**cordis-surface 区域**（子系统页面）与 **Cordis core API** 页面是穷尽的 API 参考。
- 要写 Harness 自身的插件（从 `cordis.yml` 加载、经 Web UI 驱动而非下面的 launcher），从「你的第一个 Harness 插件」起手。

### 环境准备（Setup）

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install

mkdir -p tmp/cordis-tutorial   # tmp/ 已 gitignore，改动不进版本控制
cd tmp/cordis-tutorial
```

- 每章从该目录运行同一条命令：

```sh
node --import tsx ../../vendor/cordis/bin.js
```

- 这个单文件 launcher 创建根 `Context`、挂载 Loader 插件，让它加载当前目录的 `./cordis.yml`；`--import tsx` 让 Node 免构建直接运行配置指向的 TS 文件。

### 章节地图（Chapters）

1. 第一个插件 —— 插件是一个函数，loader 挂载它
2. 生命周期与副作用 —— Cordis 管理的注册在插件卸载时被撤销
3. 服务 —— 在 `ctx` 上暴露能力并用 `inject` 依赖它
4. 事件 —— 类型化事件、广播派发、waterfall 短路
5. 配置 —— 来自 `cordis.yml` 的校验配置，坏输入大声失败
6. 组合与 HMR —— 配置文件即插件树、热更新、诊断永不加载的插件
7. 接入 Harness —— 对真实 Harness 服务注册可被模型调用的工具

### TypeScript 要点（TypeScript notes）

- **类型注解**：只描述值、不改变运行时行为（`ctx: Context`、`who: string`、`string[]`）。
- **`import type { Context } from '@deepseek-ai/cordis'`**：只导入类型信息，运行时消失，不为插件文件增加运行时依赖。
- **声明合并**：`declare module '@deepseek-ai/cordis' { ... }` 向 Cordis 已声明的接口追加条目（如新 `ctx.greeter` 属性、事件名）；不产生任何运行时接线，服务/事件由插件另行提供。

## 术语对照

| 英文术语 | 中文 | 说明 |
|----------|------|------|
| plugin | 插件 | 挂载进共享 Context 的模块 |
| Context (ctx) | 上下文 | 插件注册一切贡献的通道 |
| Loader | 加载器 | 读取 cordis.yml 并挂载插件的插件 |
| launcher | 启动器 | 创建根 Context 的单文件 `bin.js` |
| cordis.yml | 组合配置 | 声明插件树（id + name + config） |
| declaration merging | 声明合并 | `declare module` 向既有接口追加类型 |
| scratch directory | 草稿目录 | 每章构建示例的 `tmp/cordis-tutorial` |

## 陷阱与注意

- 本教程在 `tmp/cordis-tutorial` 内操作，`tmp/` 已被 gitignore，改动不进版本控制。
- `import type` 只含类型信息，运行时不存在；仅为注解需要 `Context` 的插件不增加运行时依赖。
- 声明合并不产生运行时接线，必须由插件单独提供该服务或发出该事件。

## 关联页面

- [你的第一个 Harness 插件](../basic/index.md) —— 从 cordis.yml 加载、经 Web UI 驱动的 Harness 插件写法
- [你的第一个插件](../cordis-tutorial/01-first-plugin.md) —— 插件是一个函数，loader 挂载它
