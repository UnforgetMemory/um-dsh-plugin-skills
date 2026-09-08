# DSH 官方开发指南 · 蒸馏精读（中英对照）

> 蒸馏自 DeepSeek Harness 官方英文文档 <https://deepseek-harness.github.io/deepseek-harness/en/develop/>
> 覆盖 `/en/develop/` 全部 **18 个页面**：basic(4) · cordis-tutorial(8) · framework(3) · practice(3)。
> 每页一文件镜像命名；正文为中文精炼讲解 + 英文术语对照 + 关键代码骨架。
> 原始官方 Markdown 按 umpp 规范存于共享目录区域 `.um.agents/memory/raw-source/` 供对照；完整细节以官方原文为准。

> 🧩 **自包含技能体系**：本精读已作为内容层并入开发技能 [`umdshdev/`](umdshdev/README.md)（SKILL.md 主路由 + professions + references + guide 精读层，随技能整体安装），详见其 README。

## 阅读导览

| 组 | 内容 | 前置 |
|----|------|------|
| [basic](umdshdev/guide/basic/index.md) | 上手：第一个插件 → 工具 → 配置 → 打包发布 | 无 |
| [cordis-tutorial](umdshdev/guide/cordis-tutorial/index.md) | 底层 Cordis 框架逐章教程（生命周期/服务/事件/配置/组合/HMR/深入 harness） | 无（scratch 目录起手，无需 API key） |
| [framework](umdshdev/guide/framework/index.md) | 参考：插件生命周期 · 事件系统 · 服务与依赖 | basic |
| [practice](umdshdev/guide/practice/index.md) | 进阶实践：三角色能力设计 · 动态 Cordis · LLM 适配器 | basic + framework/service |

推荐路径：`basic → framework → practice`；`cordis-tutorial` 可并行/按需深入。

## 目录树

```
umdshdev/guide/
├── basic/
│   ├── index.md        # 你的第一个插件（apply/ctx/effect/inject/三种形态）
│   ├── config.md       # 插件配置（Config schema、校验、HMR）
│   ├── publish.md      # 打包安装（bundle/profile/层顺序/git 安装陷阱）
│   └── tool.md         # 构建工具（defineTool 骨架）
├── cordis-tutorial/
│   ├── index.md        # 教程总览
│   ├── 01-first-plugin.md        # 第一个插件
│   ├── 02-lifecycle-and-effects.md  # 生命周期与效果
│   ├── 03-services.md            # 服务
│   ├── 04-events.md              # 事件
│   ├── 05-config.md              # 配置
│   ├── 06-composition-and-hmr.md # 组合与热重载
│   └── 07-into-the-harness.md    # 深入 Harness
├── framework/
│   ├── index.md        # 插件与生命周期（Fiber 状态机/自动清理/嵌套/HMR）
│   ├── events.md       # 事件系统（emit/bail/serial/waterfall/类型化事件）
│   └── service.md      # 服务与依赖（提供/消费/隔离/内置服务）
└── practice/
    ├── index.md        # 三角色能力设计（Service Definition/Provider/Consumer）
    ├── dynamic-cordis.md  # 用 Cordis 工具扩展运行中的 agent
    └── llm-adapter.md  # LLM 适配器（StreamChunk 协议）
```

## 核心心智模型（蒸馏地图）

- **一切皆插件（plugin）**：导出 `apply(ctx)` 的模块；`ctx` 注册能力、自动清理；`inject` 声明依赖、等就绪。
- **一切注册皆 effect**：`ctx.on / tools.register / llm.registerAdapter / effect` 随 Fiber 卸载自动回收，HMR 因此无残留。
- **配置 = Schema**：Schemastery schema 校验 + 填默认值 + 错误信息；可调值必须进配置，不许硬编码。
- **分发 = bundle + profile**：bundle 贡献层、profile 决定组合顺序；后层逐行覆盖、patch 整体替换 config。
- **能力 = 三角色**：Service Definition（契约）← Provider（实现）与 Consumer（工具呈现）只依赖契约、互不依赖。
- **组合 = cordis.yml**：patch 覆盖成层；`isolate` 可隔离服务实例；`group` 组织插件组。

## 使用说明

- 每个文件都是「精炼导读」：中文讲解为主、英文术语与代码骨架保留，可直接引用原文档链接。
- 需要完整细节（嵌套 schema、PTC 模式、事件签名、内置服务清单）时，跟随各页「关联页面」跳官方原文。
- 来源：`.um.agents/memory/raw-source/` 下的机器抓取原稿仅作对照，不是交付物。

## 元信息

- 蒸馏日期范围：以本仓库 git 历史为准（创建此文档时执行）。
- 官方站版本：以抓取时的 deepseek-harness `master` 为准。
- 全部改写均为本地文件，未推送任何远程。
