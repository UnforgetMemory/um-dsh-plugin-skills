---
name: umdshdev
description: DSH (DeepSeek Harness) plugin development skill system. Routes by trigger words to four professions with on-demand reference loading for token efficiency. Use when developing DSH plugins, tools, config schemas, bundles, LLM adapters, or in-session dynamic Cordis plugins.
---

# umdshdev — DSH 开发技能统一入口

DSH（DeepSeek Harness）插件开发技能体系。本技能只做**路由**：按触发词选一个分册执行，分册内按需 read references。禁止全读，禁止整份加载。

## 路由表（按触发词/意图选**一个**分册，禁止全读）

| 触发词 / 意图 | 分册 |
|---------------|------|
| 写插件 / 第一个插件 / 加载进 Web UI / cordis.yml 注册 / apply / ctx 入门 | `professions/basic.md`（路径 A：仓库级源码开发） |
| 会话内动态插件 / cordis_define / cordis_run / 动态工具 / Host-Client 插件 | `references/dynamic-cordis.md`（路径 B：当前进程内定义+激活） |
| Cordis 教程 / 底层框架 / 生命周期 / 服务 / 事件 / 配置 / 组合 / HMR | `professions/tutorial.md` |
| 插件生命周期 / 自动清理 / Fiber / 事件系统 / 服务与依赖 / inject | `professions/framework.md` |
| 三角色能力 / 动态 Cordis / LLM 适配器 / 生产实践 | `professions/practice.md` |

裁决：多触发词命中 → 表序靠前分册优先；不确定 → 问用户，禁止自行裁定。

## 公共前置（每会话一次）

read `references/core-concepts.md`（必须，建立 plugin/effect/inject/schema 元认知），随后按路由进入分册。

## References（按需加载，禁止一次性全读）

| 场景 | 文件 |
|------|------|
| 会话内动态插件（cordis_define） | `references/dynamic-cordis.md` |
| 写工具 | `references/tool-authoring.md` |
| 写配置 | `references/config-schema.md` |
| 打包发布 | `references/publish-distribution.md` |
| 服务/事件机制 | `references/service-events.md` |
| LLM 适配 | `references/llm-adapter.md` |
| 能力拆分 | `references/capability-layering.md` |
| 深挖精读（全文/边界案例/完整代码） | `guide/`（技能内内容层，按需 read 对应 `<组>/<页>.md`） |

## 纪律

1. 只路由不决策；选定分册后按其步骤执行。
2. references 只在被分册引用时 read；禁止把 references 全文内联进分册或入口。
3. 分册禁止互相内联规则；共享知识一律进 references，一处生效。
4. 本技能不改项目文件；commit/push/release 一律按 um 技能人工确认协议执行。
5. 未知 API 先查本技能内容（references/guide）或官方文档，禁止编造。