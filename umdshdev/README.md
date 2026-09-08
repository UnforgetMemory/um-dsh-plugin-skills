# umdshdev 技能体系

DSH 开发技能的**自包含技能包**（给 agent 看的中文指令与规范，按需加载），内置内容层。

## 两层结构（同包随行）

| 层 | 位置 | 内容 | 谁看 |
|----|------|------|------|
| **技能层** | `SKILL.md` + `professions/` + `references/` | 纯中文指令：入口路由 + 分册 + 规范 | agent（按需加载） |
| **内容层** | `guide/` | 18 页蒸馏精读（双语） | 需要深挖的 agent / 人 |

> `guide/` 是技能包内资源目录（DSH 目录 bundle 规范允许任意资源子目录），add/复制安装时**随技能整体带走**，自包含、不依赖源仓库与网络。

## 结构

```
umdshdev/
├── SKILL.md             # 入口：只路由（触发词 → 分册/references）※DSH 规范入口名
├── professions/         # 分册：操作流程（一次只读一个）
│   ├── basic.md         # 插件入门（路径 A：仓库级源码开发）
│   ├── tutorial.md      # Cordis 教程路线（8 章）
│   ├── framework.md     # 生命周期/事件/服务
│   └── practice.md      # 三角色/动态Cordis/LLM适配
├── references/          # 规范：按需 read（一次只读被引用的）
│   ├── core-concepts.md
│   ├── dynamic-cordis.md   # 路径 B：会话内动态插件（cordis_define）
│   ├── tool-authoring.md
│   ├── config-schema.md
│   ├── publish-distribution.md
│   ├── service-events.md
│   ├── llm-adapter.md
│   └── capability-layering.md
└── guide/               # 内容层：18 页精读（深挖用，按需 read）
    ├── basic/           # 4 页
    ├── cordis-tutorial/ # 8 页
    ├── framework/       # 3 页
    └── practice/        # 3 页
```

## 安装

把 `umdshdev/` 整体复制到技能搜索路径（用户级 `C:\Users\um\.agents\skills\umdshdev\` 或项目级 `<项目>\.agents\skills\umdshdev\`）。DSH 生态：放入扫描根即被发现（add）；改文件即更新（update），无独立 CLI。

## 使用纪律（token 节俭）

1. 一次任务：入口 + 1 分册 + 公共前置 core-concepts（必读）+ 0~2 业务 references，不整份加载。
2. 需要全文/深挖时按需 read `guide/<组>/<页>.md`（技能内资源，随包可得）；官方原文兜底走各页外链。
3. 每个文件软上限 ≈ 100 行，超限先拆职责。

## 与 um 技能

um = 工程链路（计划/提交/发布/审查/分析）通用入口；umdshdev = DSH 开发知识专用入口。二者独立；涉及 commit/push/release 按 um 技能人工确认协议。
