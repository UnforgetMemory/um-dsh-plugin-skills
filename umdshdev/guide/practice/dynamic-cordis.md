# 用 Cordis 工具扩展运行中的 Agent —— Extend a Running Agent with Cordis Tools

> **来源**：<https://deepseek-harness.github.io/deepseek-harness/en/develop/practice/dynamic-cordis>
> 本页为精炼导读（中英对照），完整细节以官方原文为准。

## TL;DR（一句话概括）

启用 `@deepseek-ai/dsh-tool-cordis` 后，agent 可以检查自己当前的 Cordis 进程，并在内存中挂载/卸载由模型编写的插件；这些临时插件在卸载或进程退出时消失，且可能影响同一进程中的其他会话。

## 核心概念

### 它能做什么（What it does）

启用 `@deepseek-ai/dsh-tool-cordis` 工具集后，agent 获得两个核心能力：

- **检查（inspect）** 当前 Cordis 进程——查看已加载插件与服务。
- **挂载/卸载（mount/unmount）** 由模型编写的插件——在内存中动态生效。

关键性质：

- **临时性**：插件在卸载或进程退出时即消失，不是持久化配置。
- **作用域**：临时插件可能影响同一进程中的其他会话。

### 运行方式（Run it）

用带内置 overlay 的浏览器界面启动：

```sh
pnpm dsh web --patch apps/cli/config/examples/cordis/cordis.yml
```

> 该命令需要模型凭据（model credential）。

### 完整契约（Full contract）

工具的参数（arguments）、生命周期（lifetime）、清理（cleanup）与安全（safety）契约，见官方 Cordis 工具参考（下方“关联页面”）。

## 术语对照

| 英文术语 | 中文 | 说明 |
|----------|------|------|
| Cordis tool | Cordis 工具 | 让 agent 检查/挂载运行时插件 |
| inspect | 检查 | 只读查看当前进程的服务与插件 |
| mount | 挂载 | 在内存中注册插件 |
| unmount | 卸载 | 从内存移除插件 |
| overlay / patch | 覆盖层/补丁 | 经 `--patch` 注入的配置层 |
| model-authored plugin | 模型编写的插件 | 由模型产出、动态挂载的插件 |

## 陷阱与注意

- 临时插件进程退出即消失，不要当作持久化配置。
- 动态插件可能影响同进程其他会话，注意副作用边界。
- 运行需要模型凭据，缺失时会失败。

## 关联页面

- [三角色能力设计](../practice/index.md) —— 静态能力的三角色拆分
- [LLM 适配器](../practice/llm-adapter.md) —— 接入新 LLM provider
- [你的第一个插件](../basic/index.md) —— 前置：基础插件路径
- [Cordis 工具参考（官方完整版）](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/extensions/tool-cordis/README.md) —— 工具参数、生命周期、清理与安全契约
