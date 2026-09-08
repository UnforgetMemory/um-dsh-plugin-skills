# 打包发布规范（bundle / profile / 层顺序）

## 两个概念

| | bundle（包） | profile（档案） |
|---|---|---|
| 是什么 | npm 包，携带一个配置层 | `$DSH_HOME/profiles/<name>` 可运行组合 |
| manifest | `dsh.bundle`（patch 路径） | `dsh.profile`（有序 bundles 列表） |
| 回答 | 贡献什么？ | 哪些包按什么顺序组合？ |
| 使用 | 作者分发 | 用户 `dsh --profile <name>` 启动 |

一个东西不可能既是 bundle 又是 profile。无 `dsh.bundle` 的包可安装但只作普通依赖，不激活层。

## Bundle 最小结构

```
hello-plugin/
├── package.json       # 声明 dsh.bundle
├── cordis.patch.yml   # 被列出时应用的层
└── index.js           # patch 行引用的插件模块
```

```json
{ "dsh": { "bundle": { "patch": "./cordis.patch.yml" } } }
```

```yaml
- insert:
    - id: hello
      name: dsh-hello-plugin   # 按包名引用（Node 解析已安装代码）
```

## 安装与组合

```sh
dsh plugin --profile demo add ./hello-plugin     # 初始化 profile 并 append bundle
dsh --profile demo --dump-config                # 只验证层不启动
dsh plugin --profile demo remove dsh-hello-plugin
```

**层顺序（后层逐行胜出，patch 整体替换 config、不深合并）**：
1. profile 的 `dsh.profile.bundles`（列表顺序，`@deepseek-ai/dsh-base` 最先）
2. profile 自己的 `cordis.patch.yml`
3. `$DSH_HOME/cordis.patch.yml`（机器本地）
4. 每个 `--patch <path>` overlay（argv 顺序）

**推论**：覆盖更早层行必须**重述全部键**；用户的 profile patch 总能覆盖你的行 → 默认值取用户大概率保留的值。

## surface bundle 自带命令行

- provider 插件 `inject = ['cmdlineArgs']`，用 `@deepseek-ai/dsh-cmdline` 的 `parseCmdline`；`--help` 时不发布服务 → 依赖行不激活。
- 依赖行 `!!js` config 读服务值 + 兜底：`port: !!js ctx.myAppStartup.port ?? 8080`。

## git 安装陷阱

- `dsh plugin add github:you/hello-plugin` 取**源码非构建产物**：`build` 不运行，TS 包缺 `lib/` → 加载失败。
- 作者：提供**自包含 `prepare` 脚本**（参考 turtle-ui 仓库）。
- 用户：pnpm ≥10 需在 profile `pnpm-workspace.yaml` 放行 `allowBuilds: dsh-hello-plugin: true`，否则首次 add 失败。
- 放行 = 允许安装时执行该包代码（sandbox 之外）→ 只放行可信源码 + pin 提交 `#<sha>`。
- 不想放行 → 分发构建产物：npm（`pnpm publish` 带 `lib/`）或 tarball（`pnpm pack` → `dsh plugin add ./x.tgz`）。

## 易错

- 相对源码路径在 profile 解析下失效 → 用包名或绝对路径。
- 覆盖行只写改动键 → 其余键被整体替换丢失。

## 深挖

完整精读/边界案例/全量代码骨架 → $(System.Collections.Hashtable[publish-distribution.md])（技能内内容层）。官方原文兜底走外链。
