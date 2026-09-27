# dsh-dual-axis

**English summary.** Two packages that add a **dual-axis access mode** to DeepSeek Harness `0.1.7-rc.2`: the
file sandbox's read and write ranges are chosen **per session**, each one `deny | workspace | all | custom`,
with a reusable rule-group library and absolute allow/deny path lists on top of a `custom` base. The host
half (`@t4r71/dsh-dual-axis`) owns the per-session axis pair, the read-path tool fence, the `/axis` command
and the model-facing range paragraph; the client half (`@t4r71/dsh-dual-axis-ui`) renders the Plugins-page
configuration row and the two read/write dropdowns above the composer. Installing the bundle deliberately
disables the upstream `ui-permission` row — see **卸载** below and the host package's README. MIT.

> 正文为中文。两个包分别位于 [`packages/dual-axis`](packages/dual-axis)（宿主半边）与
> [`packages/dual-axis-ui`](packages/dual-axis-ui)（客户端半边）。

## 这是什么

给 DeepSeek Harness 的双轴访问模式：**按会话**控制文件读/写范围，支持**可复用规则组**与**自定义放行/排除路径**。

- 每条轴取值四选一：`deny`（全禁）、`workspace`（工作区内）、`all`（全盘）、`custom`（在某个底座上叠加绝对路径的放行/排除条目）。
- 规则组（`groups`）是一份命名的规则片段库，会话按 id 引用；`defaultGroups` 决定新会话开局引用哪几条。
- 界面：插件页上本包那一行的配置表单（两轴 + 规则组），以及输入框上方的读/写两个下拉；另有 `/axis <轴>:<值>` 命令。
- 轴对按会话保存在设置命名空间 `dual-axis-sessions` 里（会话 id → 轴对），**不写进会话日志**。

## 兼容性与稳定性：先读这一段

- 面向 **DeepSeek Harness `0.1.7-rc.2`**。两个包的 `peerDependencies` 全部钉在这个版本上。
- 本组合包挂在若干 **pre-stable 的内部接缝**上，不是稳定 API：`settings.describe()`、`configForms`、`settings/document-updated`、`plugins.row.config` 槽、`session/event`。
- **升级 DSH 版本后本组合包可能整体失效。** 症状包括但不限于：插件页不出现本包那一行、输入框上方没有下拉、`/axis` 报错、客户端卡在 “Loading plugins”。失效是预期内的，不是缺陷报告的前提。
- 当前只标定 `0.1.7-rc.2`；换版本请自行重测，见 **已知边界**。

## 安装

下面按**一次真实安装**的实际做法写（这次发布验证所用的 profile 目录），每一步都注明依据文件与行。占位符
`<profile>` 指 `$DSH_HOME/profiles/<名字>`。

### 1. 拿到两个 tgz

`npm pack` 两个包目录，或直接用 npm 上发布的同名包。装 `file:` 依赖时，Windows 路径在 `package.json` 里写成**正斜杠**（例如 `file:M:/…/t4r71-dsh-dual-axis.tgz`）。

### 2. 把两个包装进 profile

编辑 `<profile>/package.json`：**两个包都进 `dependencies`**，但 **`dsh.profile.bundles` 里只列宿主半边**：

```json
{
  "dependencies": {
    "@t4r71/dsh-dual-axis": "file:<绝对路径>/t4r71-dsh-dual-axis.tgz",
    "@t4r71/dsh-dual-axis-ui": "file:<绝对路径>/t4r71-dsh-dual-axis-ui.tgz"
  },
  "dsh": {
    "profile": {
      "bundles": [
        "@deepseek-ai/dsh-base",
        "@deepseek-ai/dsh-web-app",
        "@t4r71/dsh-dual-axis"
      ]
    }
  }
}
```

为什么客户端半边不进 `bundles`：它没有 `dsh.bundle.patch`，不是一个补丁层；它只需要作为依赖存在，好让宿主补丁里
`dual-axis-ui` 那一行能被解析到。它的宿主半边是一个空插件，这一行存在的唯一理由是 0.1.7 的客户端扫描只处理有
Loader 行的包。

本次安装的实际取值：两条 `file:` 依赖写在 `dependencies` 里（Windows 路径用**正斜杠**，形如 `file:M:/…/t4r71-dsh-dual-axis.tgz`），`dsh.profile.bundles` 写成 `["@deepseek-ai/dsh-base", "@deepseek-ai/dsh-web-app", "@t4r71/dsh-dual-axis"]`。

### 3. 在 profile 目录装依赖

```powershell
cd <profile>
pnpm install --prefer-offline
```

依据：该 profile 目录下存在 `pnpm-lock.yaml`，且 `node_modules/@t4r71/` 里是两个符号链接，分别指向
`.pnpm\@t4r71+dsh-dual-axis@file+…` 与 `.pnpm\@t4r71+dsh-dual-axis-ui@fil…`（`file:` 依赖装出来的样子）。

也可以让 CLI 代跑：`dsh plugin --profile desktop add <spec>` 会把参数原样转发给 profile 目录里的 pnpm
（上游 `apps/cli/src/args.ts:171-182`，`apps/cli/src/plugin.ts:12-24`）。**这次安装没有走这条路径**，
上面两条命令与两份文件才是实际发生的事。

### 4. Loader 行从哪来

**不用手写。** 验证部署的 profile 里，被设置页编辑的那两行（`dual-axis` 与 `dual-axis-sessions`）不是手写进 profile 的：
宿主包在自己的 `package.json` 里声明了 `dsh.bundle.patch: ./cordis.patch.yml`
（[`packages/dual-axis/package.json`](packages/dual-axis/package.json) 的 `dsh.bundle` 字段）；
profile 组装时，按 `dsh.profile.bundles` 的顺序把每个 bundle 的补丁叠在一个空条目表上，之后才是 profile 自己的
`cordis.patch.yml`，最后是启动参数带的 `--patch` 层
（上游 `packages/boot/app-boot/src/profile.ts:1-13`、`:905-934`）。

于是宿主那份补丁
（[`packages/dual-axis/cordis.patch.yml`](packages/dual-axis/cordis.patch.yml)）会自动插入四行，并把上游一行关掉：

| 行 id | 包 / 入口 | 作用 |
| --- | --- | --- |
| `dual-axis` | `@t4r71/dsh-dual-axis` | 宿主本体：设置行、`/axis`、模型可见段落、新会话种子 |
| `dual-axis-read-guard` | `@t4r71/dsh-dual-axis/read-guard` | 读轴的工具派发层围栏 |
| `dual-axis-sessions` | `@t4r71/dsh-dual-axis/session-store` | 每会话轴对的权威存储 |
| `dual-axis-ui` | `@t4r71/dsh-dual-axis-ui` | 纯客户端半边 |
| `ui-permission` | `@deepseek-ai/dsh-client-ui-permission-presets` | **`disabled: true`**（取代关系，见下） |

**profile 自己的 `cordis.patch.yml` 是可选的覆盖层。** 验证部署在里面用 `id:` 命中 `dual-axis` 与
`dual-axis-sessions`，把两轴取值、规则组库与 `defaultGroups` 换成自己的（
验证部署的 `<profile>/cordis.patch.yml` 53-163 行）。这种条目是**整段 config 覆盖**已存在的行；
按 id 找不到目标行时补丁引擎只打一条 warning 并跳过；`name` 与目标行不符时同样警告并跳过
（上游 `vendor/include/src/index.ts:104-124`，name 校验在 `:115-118`）。不写这两行也能跑，用的是补丁里插的默认值
（读轴 `all`、写轴 `workspace`、无规则组；见 [`packages/dual-axis/src/config.ts`](packages/dual-axis/src/config.ts) 的 `Config` 文档）。

### 5. 重启宿主

重启进程即可（profile 的条目树在启动时组装）。启动后应当看到：插件页出现本包那一行、且它的配置表单可编辑并落盘；
输入框上方出现读/写两个下拉。客户端那一行的键是 `@t4r71/dsh-dual-axis#dual-axis`
（[`packages/dual-axis-ui/src/client/row/index.ts`](packages/dual-axis-ui/src/client/row/index.ts) 的 `DUAL_AXIS_ROW_KEY`）——
**键错了页面会静默不渲染**，所以两边的行 id 与包名都不能改。

### 本节的依据

| 结论 | 依据 |
| --- | --- |
| profile 依赖与 bundles 列表 | 验证部署的 `<profile>/package.json`（依赖 4-6 行、`dsh.profile` 19-27 行） |
| 装依赖的实际命令与结果 | 该目录的 `pnpm-lock.yaml` 与 `node_modules/@t4r71/` 符号链接 |
| bundle 补丁怎么被叠上去 | 上游 `packages/boot/app-boot/src/profile.ts` 模块文档（1-13 行）与 `loadProfile`（905-934 行） |
| 插了哪四行、关了哪一行 | [`packages/dual-axis/cordis.patch.yml`](packages/dual-axis/cordis.patch.yml) |
| 覆盖层语义 | 上游 `vendor/include/src/index.ts:104-124` |
| 覆盖层的实际取值 | 验证部署的 `<profile>/cordis.patch.yml` 53-163 行 |

## 卸载，以及「想恢复官方权限 UI 就必须卸载本组合包」

1. 从 `<profile>/package.json` 的 `dsh.profile.bundles` 里删掉 `@t4r71/dsh-dual-axis`；
2. 从 `dependencies` 里删掉两个包；
3. 在 profile 目录重跑 `pnpm install`（或 `dsh plugin --profile <名字> remove @t4r71/dsh-dual-axis @t4r71/dsh-dual-axis-ui`）；
4. 重启。

**官方权限 UI 不会自己回来，除非把本组合包卸掉。** 本组合包的客户端半边注册的是上游那三个界面面本身——同一个
`conversation.input.permission` 单占槽、同一条 `settings.general.item` 行（id `permission`、order `-20`）、
同一个 `/permission` 命令装饰——因此补丁必须先把上游 `ui-permission` 行 `disabled: true`。两条独立的单占规则
（槽单占、locale 命名空间按 `(ns, locale)` 单占）都会在第二次注册时抛错，fiber 落进 FAILED，客户端卡在
“Loading plugins”。`disabled: false` 改回去只会把这个冲突重新引爆。

完整声明（取代了哪些面、哪两条机制冲突、行为变成什么、怎么拿回官方界面）在宿主包的 README：
[`packages/dual-axis/README.md`](packages/dual-axis/README.md) 的 “⚠ This bundle disables an official UI row” 一节。

## 已知边界

以下是本组合包**已经查明的边界**，照实列出，不做美化。

1. **本组合包会禁用上游 `ui-permission` 行。** 原因是单占槽 + locale 命名空间冲突，技术不可行共存，不是可以靠配置绕开的降级（详见上一节）。
2. **读轴 = 工具层围栏 + 提示词。** `read-guard` 只拦 `read`、`read_image`、`grep`、`glob` 四个读入口；命令工具、子进程、脚本不走这一层。**Windows 上不可能内核化**，因此读轴永远不是一道硬边界。
3. **命令工具可以写「写轴自定义排除」的路径。** 写轴的**基准档**（`deny` / `workspace` / `all`）会镜像到会话沙箱模式，在工具层之下还有一道内核边界；`custom` 的 `allow` / `deny` **条目**只有工具层，命令工具绕得过去。
4. **旧构建写过的会话不救。** 轴上线的构建之前已经写过的会话，其日志里没有轴记录，本包不为它们补写。
5. **子代理继承只实测过进程内一次性 spawn。** 没有收窄接口。
6. **收窄显示最多滞后 10 秒**（客户端跟随组定义刷新）。
7. **仅 Windows 验证过。**
8. **`@deepseek-ai/dsh-settings` 的 `./schema` 在 npm 版缺失**，影响测试（见 **开发** 一节的 `TSX_TSCONFIG_PATH`）。
9. **换 tgz 之后必须删掉 `node_modules` 重装**：`file:` 依赖按路径哈希认，覆盖同名 tgz 不会重新解包。

更细的边界（`DualAxisFileSystem` 未挂载、写轴不与读轴求交、无记录的会话如何取种子等）见
[`packages/dual-axis/README.md`](packages/dual-axis/README.md) 的 “Known Limitations and Deferred Work”。

## 开发

### 装依赖

```powershell
pnpm install
```

工作区由仓库根的 [`pnpm-workspace.yaml`](pnpm-workspace.yaml) 定义（`packages/*` 与 `packages/*/*`），其中
`allowBuilds` 必须先记录带安装脚本的依赖（`koffi`、`esbuild`），否则 pnpm 以 `ERR_PNPM_IGNORED_BUILDS` 退出 1。
实测环境：Node v24.18.0、pnpm 12.5.1。

### 构建

```powershell
pnpm --filter @t4r71/dsh-dual-axis run build      # tsc -b && tsdown
pnpm --filter @t4r71/dsh-dual-axis-ui run build   # tsc -b tsconfig.json && tsdown
```

先 `tsc` 出 `lib/types`，再由 `tsdown` 打包运行时的 `lib/*.js`。两个包都从**自己的 `node_modules`** 解析
`@deepseek-ai/*`，不依赖任何 monorepo：[`packages/dual-axis/tsconfig.base.json`](packages/dual-axis/tsconfig.base.json)
不声明任何 `paths`。

### 单测

```powershell
cd packages/dual-axis
$env:TSX_TSCONFIG_PATH = 'tsconfig.runtime.json'   # 必须，相对于当前目录
pnpm test

cd ../dual-axis-ui
pnpm test
```

`TSX_TSCONFIG_PATH` 指向的 [`packages/dual-axis/tsconfig.runtime.json`](packages/dual-axis/tsconfig.runtime.json) 只做一件事：
把 `@deepseek-ai/dsh-settings/schema` 映射到已安装依赖里的 `lib/types/schema.js`。上游那个包发布了这个文件却没在
`exports` 里声明 `./schema` 子路径，Node 按裸子路径解析不到它。上游补上 `./schema` 之后这个映射就可以删。

**基线（本仓库干净克隆实测）：宿主 117/117，客户端 54/54。**

### 仓库结构

```
packages/dual-axis/      宿主半边：设置行、轴存储、读轴围栏、/axis、模型可见段落、bundle 补丁
packages/dual-axis-ui/   客户端半边：插件页配置行 + 输入框上方两个下拉
pnpm-workspace.yaml      工作区定义与 allowBuilds
LICENSE                  MIT（T4R71）
```

`packages/dual-axis/tests/seed-parity.spec.ts` 是两半之间唯一的一致性门：它按**相对路径**导入客户端半边的纯模块
`packages/dual-axis-ui/src/client/permission/session-axes-seed.ts`。该路径按本仓库的 `packages/` 布局写；把包目录搬回
别的布局（例如 `packages/bundle/dual-axis`）时，那一行导入要跟着改，否则这条 spec 会在导入阶段失败。

## License

MIT，见 [`LICENSE`](LICENSE)，署名 T4R71。两个包的 `package.json` 同为 `"license": "MIT"`。
