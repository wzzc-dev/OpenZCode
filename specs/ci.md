# CI 质量门禁与自动打包

## 目的与范围

仓库历史上没有入库 CI 配置（内部流水线被剥离，仅代码里残留约定：`scripts/prepare-prebuilds.mjs:58` 引用的 `.gitlab/ci/00-workflow.yml`、`scripts/ci/ci-repo-hygiene.mjs` 均不存在）。本 spec 定义补齐后的最小 CI 面：

1. 质量门禁：push 到 `main` 与 pull request 时跑类型检查、lint 与架构检查。
2. 自动打包：push `v*` tag 与手动触发时，构建 Desktop macOS arm64 安装包并作为 CI artifact 保留。

非目标：不接入 macOS 签名/公证（`ZCODE_ENABLE_MAC_SIGN` 保持关闭）、不上传外部分发源、不覆盖 win/linux 矩阵与 CLI 发行包。

## 触发与所有者

| Workflow                                      | 触发                                 | 唯一所有者                       | 职责                          |
| --------------------------------------------- | ------------------------------------ | -------------------------------- | ----------------------------- |
| `.github/workflows/ci.yml`                    | `push: main`、`pull_request`         | `verify` job                     | 只读校验，不产出任何工件      |
| `.github/workflows/package-desktop-macos.yml` | `push: tags v*`、`workflow_dispatch` | `verify` → `package-macos-arm64` | 门禁通过后出包并上传 artifact |

打包 workflow 自带 `verify` 前置 job，门禁不依赖 `ci.yml` 的执行结果，避免「PR 门禁绿了但 tag 打包跳过检查」的分叉路径。

工具链解析、缓存与依赖安装的唯一所有者是 `.github/actions/setup-repo`。**它不包含 `actions/checkout`**，checkout 必须由调用它的 job 自己先执行。GitHub 在展开 local composite action 之前就要读取它的 `action.yml`，此时工作区尚未 checkout，把 checkout 放在 action 内部会直接报 `Can't find 'action.yml' ... Did you forget to run actions/checkout before running your local action?`，且该 step 永远没有执行机会。这是平台约束而非风格选择，无法通过调整 step 顺序消除；相应地 `fetch-depth` 由 job 层的 checkout 决定，action 不再暴露该输入。

打包 workflow 的 `verify` job 是 reusable workflow 引用，`ci.yml` 内部自带 checkout，因此调用方无需重复。

## 工具链

版本以 `mise.toml` 为唯一事实来源，workflow 不得自行硬编码另一套版本：

- Node `24.14.0`（`mise.toml` `[tools] node`）
- pnpm `10.33.2`（`mise.toml` `[tools] pnpm`，与根 `package.json` 的 `packageManager` 一致）

## 门禁阶段

`verify` job 依次执行，任一失败即中断：

```bash
pnpm install --frozen-lockfile
pnpm typecheck
pnpm lint
pnpm architecture:check
```

架构检查必须全量执行（不带 `--changed`）。`scripts/architecture/index.mjs:317` 的 `changedFilesFromGit` 比较的是工作区与 `HEAD` 的差异加未跟踪文件；CI checkout 是干净树，该集合恒为空，`--changed` 在 CI 中退化成空检查。全量检查覆盖所有模块且当前耗时亚秒级，因此 CI 采用全量语义。`--changed` 仍保留给本地 pre-push 快速反馈，两条路径不共享假设。

## 打包阶段

`package-macos-arm64` job 运行在 `macos-14`（Apple Silicon），执行唯一入口：

```bash
pnpm bundle:desktop -- --os mac --arch arm64
```

不拆解 `scripts/bundle.mjs` 内部步骤（prepare → build → electron-builder → 运行时依赖机械校验 → 体积审计）。该脚本是打包链路的唯一所有者，已有镜像 fallback 与瞬时网络失败重试；workflow 复制它的步骤会制造第二条写入路径。

### 环境变量契约

| 变量                       | 取值               | 依据                                                             |
| -------------------------- | ------------------ | ---------------------------------------------------------------- |
| `ZCODE_TARGET_OS`          | `mac`              | `packages/desktop/scripts/target-platform.mjs:38`                |
| `ZCODE_TARGET_ARCH`        | `arm64`            | 同上                                                             |
| `ZCODE_DESKTOP_DIST_DIR`   | `dist-macos-arm64` | 按架构隔离输出；`.gitignore:6` 已忽略 `packages/desktop/dist-*/` |
| `ZCODE_SKIP_REMOTE_ASSETS` | `1`                | 见下节                                                           |
| `ZCODE_ENV`                | `test`             | 见下节                                                           |
| `ZCODE_ENABLE_MAC_SIGN`    | 未设置             | 关闭签名                                                         |
| `ZCODE_PREVIEW_IDENTITY`   | 未设置             | 身份由 `ZCODE_ENV` 推导                                          |

**跳过 remote assets**：`prepare:remote-assets`（`scripts/prepare-prebuilds.mjs`）会下载 Node 运行时并打包跨平台远端 agent 二进制，组件源缺失且未配置 `ZCODE_DEPS_BASE_URL` / `INTRANET_MACHINE_HOST` 时会 skip（`scripts/prepare-prebuilds.mjs:963`）。桌面安装包不需要 mock-cdn：`electron-builder.config.js` 的 `extraResources` 只引用 `bundled-agents/<platform>/glm`（由 `prepare:agent-bundle` 本地生成）与 `bundled-tools/<platform>/ripgrep`（由 `prepare:native-search` 解包仓库内置归档），二者都在 `localRuntimeScripts` 中，不受该开关影响。`packages/desktop/scripts/prepare-runtime-assets.mjs:56-59` 的注释已记录原 CI 因无条件执行该步骤而逼近 1 小时上限，故 CI 显式跳过。

**后端环境设为 test**：不设 `ZCODE_ENV` 时 `desktop-product-identity.mjs` 推导为 test，产物使用 `ZCode Preview` 身份并带 `_TEST` 文件名后缀。这是刻意的 fail-safe：CI 未配置签名身份，产物不能顶着正式 `ZCode` 身份流出被误认为发布包。需要生产身份出包时显式设置 `ZCODE_ENV=production`，并在同一 job 注入 Apple 凭据与 `ZCODE_ENABLE_MAC_SIGN=1`。

## 不变量

- workflow 不写入仓库源码树；所有中间产物落在 `dist-macos-arm64/`（已 gitignore）与工作区外缓存目录。
- CI 不刷新架构基线（`pnpm architecture:baseline:update` 仅在评审过的变更中由人执行）。
- CI 不产出签名产物；`mac.identity` 在 `ZCODE_ENABLE_MAC_SIGN != 1` 时为 `null`（`electron-builder.config.js:670`）。
- artifact 命名由 `electron-builder.config.js:236` 的 `buildDesktopArtifactName` 决定，workflow 不改写产物文件名。

## 缓存

三个缓存目录与 `.gitignore:26-29` 预留位一一对应：

| 目录                      | 环境变量                    | 内容                                           |
| ------------------------- | --------------------------- | ---------------------------------------------- |
| `.pnpm-store`             | `pnpm config set store-dir` | pnpm 内容寻址存储                              |
| `.electron-cache`         | `ELECTRON_CACHE`            | Electron 运行时 zip                            |
| `.electron-builder-cache` | `ELECTRON_BUILDER_CACHE`    | dmg-builder、app-builder-lib 等 builder 辅助包 |

缓存 key 以 `pnpm-lock.yaml` 与 Node 版本为输入；缓存 miss 不应导致 job 失败（脚本自带镜像 fallback 与重试）。

## 失败语义

- `verify` 任一命令非零退出 → job 失败，`package-macos-arm64` 因 `needs` 不执行。
- 打包脚本自身已内建 fail-closed 校验：缺运行时依赖（`bundle.mjs:638` 起的 `verifyPackagedRuntimeDependencies`）、越界 native 资源、`node-pty` 预编译产物缺失、运行时依赖未进 `app.asar` 均直接抛错，不允许「产物存在但无法启动」的安装包流出。
- artifact 上传使用 `if-no-files-found: error`，且上传 step 失败会让 job 失败。产物未保留等价于本次打包没有交付，必须 fail-closed 暴露；重跑该 job 时缓存命中，边际成本只剩 electron-builder 阶段。

## 验收场景

| 场景                                           | 期望                                                                                               |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| PR 修改 `packages/*/src`                       | `ci.yml` 的 `verify` 运行三项检查并给出通过/失败结论                                               |
| 类型检查不通过                                 | `verify` 失败，日志指向 `pnpm typecheck`                                                           |
| push tag `v3.14.0`                             | `package-desktop-macos.yml` 先 `verify` 后打包，产出 `.dmg` 与 `.zip` 并上传 artifact              |
| 手动 `workflow_dispatch`                       | 行为与 tag 触发一致，使用当前分支 HEAD                                                             |
| 打包脚本因缺依赖失败                           | job 失败，不上传 artifact                                                                          |
| 设置 `ZCODE_ENABLE_MAC_SIGN=1` 但无 Apple 凭据 | 未显式设置该变量，故不进入签名分支；若日后打开，缺身份时 `electron-builder.config.js:211` 提前抛错 |

## 实测记录

在干净工作区按 CI 相同的环境变量实跑 `pnpm bundle:desktop -- --os mac --arch arm64`（macOS arm64 宿主）：

- 产物：`ZCode Preview-3.14.0-mac-arm64_TEST.dmg`（177 MiB）与同名 `.zip`（169 MiB），附带 `.blockmap`、`builder-debug.yml`、`latest-mac.yml`，全部落在 `packages/desktop/dist-macos-arm64/`，由 `.gitignore:6` 覆盖。
- 体积审计：`.zip` 169.0 MiB，上限 500 MiB，通过。
- 签名：日志为 `skipped macOS code signing  reason=identity explicitly is set to null`，与未开启 `ZCODE_ENABLE_MAC_SIGN` 的预期一致。
- 门禁：`pnpm typecheck`、`pnpm lint`（70 warnings / 0 errors）、`pnpm architecture:check`（全量与 `--changed` 两种语义）均通过。
- Node 版本：本地宿主为 v25.2.1，`apps/zcode-cli` 的 `engines.node` 要求 `24.14.0`，打包过程持续输出 engine warning 但仍成功。这印证 workflow 必须按 `mise.toml` 固定 24.14.0，而不是取 runner 默认版本。

## 与本地开发的边界

- `pnpm verify:pre-push` 仍用于本地提交前快速检查，保留 `--changed` 语义。
- CI 是产物与门禁事实的唯一来源；本地打包结果不入库、不作为发布依据。
