# 核心概念（公共前置）

DSH 插件开发的元概念。所有分册的前置，会话内读一次。

## 六条心智模型

1. **插件 = `apply(ctx)` 函数模块**：框架加载时调用 `apply`，插件经 `ctx` 注册能力（事件/工具/定时器/服务），卸载时全部自动清理。
2. **一切注册皆 effect**：`ctx.on()` / `ctx.tools.register()` / `ctx.llm.registerAdapter()` / `ctx.effect()` 都随 Fiber 卸载自动回收。需显式释放的资源（连接等）用 `ctx.effect(() => { ...; return () => cleanup })` 提供 disposer。
3. **依赖用 inject 声明**：`export const inject = ['tools']`；框架等全部必需服务就绪后才加载插件。服务消失 → 插件自动卸载，回归 → 自动重载。
4. **配置 = Schemastery Schema**：导出 `Config` 类型 + 同名 schema，默认值写 schema 字段；`cordis.yml` 的 `config` 经校验+填默认后作 `apply(ctx, config)` 第二参。禁止导出普通对象当 Config。
5. **组合 = cordis.yml**：插件行（id + name + config）；patch/overlay 按层叠加，后层逐行覆盖，patch 整体替换 config（不深合并）。
6. **能力 = 三角色**：Service Definition（契约）← Provider（实现）、Consumer（工具呈现）只依赖契约互不依赖，可换 Provider。

## 三种插件形态

| 形态 | 写法 | 适用 |
|------|------|------|
| Function | `export function apply(ctx)` | 大多数 |
| Object | `export default { name, inject, apply }` | 同函数 |
| Class | `export default class X extends Service` | 提供服务给其他插件 |

## 关键术语

| 术语 | 含义 |
|------|------|
| apply(ctx) | 插件入口函数 |
| ctx / Context | 注册能力的上下文 |
| inject | 声明所需服务，等其就绪 |
| effect | 可自动清理的注册 |
| Fiber | 插件实例的生命周期作用域 |
| schema / Config | 配置校验 + 默认值 |
| patch / overlay | 配置层，后层覆盖前层 |

## 按需转向

会话内动态插件 → dynamic-cordis.md；写工具 → tool-authoring.md；配置 → config-schema.md；打包 → publish-distribution.md；服务/事件 → service-events.md；LLM → llm-adapter.md；能力拆分 → capability-layering.md

## 深挖

完整精读/边界案例/全量代码骨架 → `guide/basic/index.md`（第一个插件）· `guide/framework/index.md`（插件与生命周期）（技能内内容层）。官方原文兜底走外链。
