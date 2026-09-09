# 打包与安装插件 —— Package and Install a Plugin

> **来源**：<https://deepseek-harness.github.io/deepseek-harness/en/develop/basic/publish>
> 本页为精炼导读（中英对照），完整细节以官方原文为准。

## TL;DR（一句话概括）

把插件打成 npm 包（**bundle**，声明 `dsh.bundle` 补丁层），装进 **profile**（声明 `dsh.profile` 的有序组合）——二者由同一 `package.json` 承载但答案不同：bundle 答「贡献什么」，profile 答「按什么顺序组合谁」。

## 核心概念

### 两个概念、两类 manifest（Two concepts, two manifests）

| | bundle（包） | profile（档案） |
|---|---|---|
| 是什么 | npm 包，携带一个配置层 | `$DSH_HOME/profiles/<name>` 目录，描述一个可运行组合 |
| manifest | `dsh.bundle` | `dsh.profile` |
| 回答的问题 | 这个包贡献什么？ | 哪些 bundle 按什么顺序组合？ |
| 谁用 | 作者分发 | 用户 `dsh --profile <name>` 启动 |

> 「一个东西不可能既是 bundle 又是 profile」。

### Bundle 清单（The bundle manifest）

```
hello-plugin/
├── package.json       # 声明 dsh.bundle
├── cordis.patch.yml   # 本 bundle 被列出时应用的层
└── index.js           # patch 行引用的插件模块
```

```json
{
  "name": "dsh-hello-plugin",
  "version": "0.1.0",
  "type": "module",
  "main": "index.js",
  "files": ["index.js", "cordis.patch.yml"],
  "dsh": { "bundle": { "patch": "./cordis.patch.yml" } }
}
```

patch 中插件行按**包名**引用（而非相对源码路径），让 Node 解析到已安装代码：

```yaml
- insert:
    - id: hello
      name: dsh-hello-plugin
```

> 💡 没有 `dsh.bundle` 的包也能安装，但只是普通依赖：`dsh plugin` 会警告且不激活任何层——这种格式用于被插件 import 的库。

### Profile 清单（The profile manifest）

- 两个文件：`package.json`（含 `dsh.profile.bundles` 有序列表 + pnpm 管理的依赖）和 `cordis.patch.yml`（用户自己的补丁层，在所有 bundle 层之后应用）。
- **从不手写 profile manifest**：`dsh plugin` 自动创建与维护。

### 安装进 profile（Install into a profile）

```sh
dsh plugin --profile demo add ./hello-plugin
# 首次使用会初始化 profile（首个 bundle 为 @deepseek-ai/dsh-base），
# pnpm 链接 checkout，dsh 自动把 bundle 追加进 dsh.profile.bundles

dsh --profile demo --dump-config   # 只验证层，不启动（应看到 "# == dsh-hello-plugin"）
dsh --profile demo                 # 启动
dsh plugin --profile demo remove dsh-hello-plugin   # 同时移除依赖与层
```

## 加载顺序（The loading order）

有效配置按顺序叠加（后面的层逐行胜出，patch 整体替换 `config` 值、不深合并）：

1. profile 的 `dsh.profile.bundles` 列表中每个 bundle patch，按列表顺序（`@deepseek-ai/dsh-base` 最先）
2. profile 自己的 `cordis.patch.yml`
3. 主目录级 `$DSH_HOME/cordis.patch.yml`（机器本地偏好，所有 profile 共享）
4. 每个 `--patch <path>` overlay（按 argv 顺序）

对 bundle 作者的推论：
- 你的 patch 可以按 `id` 覆盖更早层的行（如 `dsh-web-app` 覆盖 `dsh-base`），但必须**重述行所需的每个键**，而非只改一个。
- 用户可以在自己的 `cordis.patch.yml` 覆盖你的行，所以配置默认值应取用户大概率保留的值，其余交给 schema。
- box 内 bundle 名始终从 dsh 安装本体解析；pnpm 只管 box 外包。

### 给 surface bundle 自己的命令行

- bundle 定义可运行应用：挂载普通 provider 插件（`inject = ['cmdlineArgs']`，用 `@deepseek-ai/dsh-cmdline` 的 `parseCmdline` 建自己的 commander 程序），在 action 里提供 app 自有服务。
- 需要该参数的插件行 `inject` 此 provider 的服务，并在 `!!js` config 里读取（deployment 值作 fallback）：

```yaml
- id: my-app
  name: '@example/my-app'
  inject: [myAppStartup]
  config:
    port: !!js ctx.myAppStartup.port ?? 8080
```

- `--help` 时 provider 不发布服务，这些行永不激活；Loader 只装配一次组合，等每行的常规注入就绪后才对注入的上下文求值 `!!js` config。

## 从 GitHub 安装：构建脚本陷阱（The build-script catch）

- `dsh plugin --profile demo add github:you/hello-plugin` 可免注册表安装，但**git 安装取到的是源码而非构建产物**：`build` 脚本不会运行，TS 包缺 `lib/` 输出会加载失败。

两条腿缺一不可：

1. **作者**：提供 `prepare` 脚本（pnpm 在 git 安装后运行），从源码构建发布入口，**自包含**（不假设 dev-only 上下文，如兄弟 monorepo checkout）。示例：<https://github.com/deepseek-harness/turtle-ui>。
2. **用户**：放行构建——pnpm ≥10 默认拒绝运行 git 依赖的 `prepare`，首次 `add` 会失败；把 pnpm 打印的包 key 写进 profile 的 `pnpm-workspace.yaml`：

```yaml
allowBuilds:
  dsh-hello-plugin: true
```

> ⚠️ 这个放行 = **允许在安装时于你机器上执行该包代码**（在 agent 沙箱之外）。只放行信得过的源码，并 pin 提交（`github:you/hello-plugin#<sha>`）防静默漂移。

不想让用户放行？分发构建产物（两种都不用构建权限）：
- 发 npm：`pnpm publish` 时带上已构建的 `lib/`，用户 `dsh plugin add your-package`。
- 发 tarball：`pnpm pack` 产物，用户 `dsh plugin add ./hello-plugin-0.1.0.tgz`。

## 术语对照

| 英文术语 | 中文 | 说明 |
|----------|------|------|
| bundle | 包/束 | 携带配置层的 npm 包，声明 `dsh.bundle` |
| profile | 档案/组合 | `$DSH_HOME/profiles/<name>` 的可运行组合，声明 `dsh.profile` |
| patch / overlay | 补丁/覆盖层 | 配置层，后层逐行覆盖前层 |
| layer order | 层顺序 | bundles → profile.patch → home.patch → --patch |
| prepare script | 准备脚本 | git 安装后由 pnpm 运行的构建钩子 |
| allowBuilds | 构建放行 | pnpm ≥10 要求的 git 依赖 prepare 白名单 |

## 陷阱与注意

- 路径为相对路径时按 profile 目录解析；patch 不做深合并，覆盖行必须重述全部键。
- git 安装不带构建产物——`prepare`（作者）+ `allowBuilds`（用户）缺一即失败。
- `--patch` overlay 不是另一层 profile 层；app 参数经 app 自有服务解析，不加补丁层。
- 「装了但没反应」先看警告 `dsh: warning: <pkg> declares no dsh.bundle — installed as a plain dependency…`：包未声明 bundle manifest，对账（reconcile）时只作普通依赖。机制（CLI 行为参考）：每次 `dsh plugin` 成功运行后 `dsh.profile.bundles` 与已安装依赖对账——获得声明的 `update` 自动入层、无声明的保持普通依赖并一次性警告、已移除的移出层栈；bundle 成员变更须重启 profile 生效。作者修复 = 补 `dsh.bundle` 声明 + `cordis.patch.yml`（+ npm 发布时 `files` 含二者）。

## 关联页面

- [插件与生命周期](../framework/index.md) —— 完整插件生命周期
- [CLI 行为参考](https://github.com/deepseek-ai/deepseek-harness/blob/master/apps/cli/reference/README.md) —— 层优先级、flags、profile 机制细节（仓库内文档，未发布到文档站）