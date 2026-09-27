# Changelog

本仓库的两个包同版本发布，条目按日期倒序。版本号跟随所面向的 DeepSeek Harness 版本。

## 2026-09-27 — 0.1.7-rc.2（首版）

首次公开发布：[`@t4r71/dsh-dual-axis`](packages/dual-axis)（宿主半边）与
[`@t4r71/dsh-dual-axis-ui`](packages/dual-axis-ui)（客户端半边）。

### 功能

- **按会话的双轴**：读轴与写轴各自取 `deny | workspace | all | custom`；`custom` 在某个底座上叠加绝对路径的
  `allow` / `deny` 条目。
- **可复用规则组**：设置行维护一份命名规则片段库（`groups`），`defaultGroups` 决定新会话开局引用哪几条；组定义改动
  会作用到引用它的会话。
- **每会话的轴对存储**：设置命名空间 `dual-axis-sessions`（会话 id → 轴对），整份 `axes` 字段一次 `replace`，
  按 revision 重试；四次都冲突则抛 `DUAL_AXIS_AXES_CONFLICT`。轴对不写进会话日志。
- **读轴围栏**：`read-guard` 在工具派发层拦 `read` / `read_image` / `grep` / `glob` 四个入口。
- **模型可见的取值段落**：会话组装时按已解析的范围渲染（段落名 `sandbox:dual-axis`），中途改轴在下一次请求生效。
- **`/axis <轴>:<值>` 命令**：按会话改轴，并冻结该会话的轴对。
- **客户端半边**：插件页的配置行（两轴 + 规则组，一次原子保存）与输入框上方的读/写两个下拉；行键
  `@t4r71/dsh-dual-axis#dual-axis`。
- **安装即取代官方权限行**：补丁禁用上游 `ui-permission` 行（单占槽与 locale 命名空间两处冲突，技术不可行共存）。
  想拿回官方界面必须卸载本组合包，见根 README。
- **bundle 补丁**：`dsh.bundle.patch` 声明的补丁插入四个 Loader 行并关掉上游那一行，profile 无需手写 Loader 条目。

### 兼容性

- 面向 DeepSeek Harness `0.1.7-rc.2`（两个包的 `peerDependencies` 全部钉在该版本）。
- 插件挂在 pre-stable 内部接缝上（`settings.describe()`、`configForms`、`settings/document-updated`、
  `plugins.row.config` 槽、`session/event`），升级可能失效。

### 仓库

- 两个包的源码快照（不含 `lib/`、`node_modules/`、`*.tsbuildinfo`、`pnpm-lock.yaml`、`*.tgz`）、根 README、
  本 CHANGELOG、MIT LICENSE、pnpm 工作区配置。
- 构建与单测在只有这两个包的干净目录里实测通过（宿主 117/117，客户端 54/54）。
