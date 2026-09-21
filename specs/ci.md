# CI 质量门禁与多平台自动打包

## 目的与范围

仓库历史上没有入库 CI 配置（内部流水线被剥离，仅代码里残留约定：`scripts/prepare-prebuilds.mjs:58` 引用的 `.gitlab/ci/00-workflow.yml`、`scripts/ci/ci-repo-hygiene.mjs` 均不存在）。本 spec 定义补齐后的 CI 面：

1. 质量门禁：push 到 `main` 与 pull request 时跑类型检查、lint 与架构检查。
2. 自动打包：push `v*` tag 与手动触发时，构建 Desktop 三平台安装包，产物作为 CI artifact 保留，并把安装包发布到本仓库的 GitHub Release。

非目标：不接入 macOS 代码签名/公证（`ZCODE_ENABLE_MAC_SIGN` 保持关闭）、不接入 Windows 代码签名（不注入 `CSC_LINK` / `CSC_KEY_PASSWORD`）、不上传到本仓库以外的分发源或 CDN。`electron-builder.config.js:756` 的 `publish` 仍指向 `http://localhost:8081` 占位，自动更新走服务端 manifest provider，**不**由 GitHub Release 提供。

## 触发与所有者

| Workflow                  | 触发                                 | 唯一所有者                                | 职责                                        |
| ------------------------- | ------------------------------------ | ----------------------------------------- | ------------------------------------------- |
| `.github/workflows/ci.yml`| `push: main`、`pull_request`         | `verify` job                              | 只读校验，不产出任何工件                    |
| `.github/workflows/package-desktop.yml` | `push: tags v*`、`workflow_dispatch` | `verify` → `package-*`（并行）→ `release` | 门禁通过后三平台出包，发布到 GitHub Release |

打包 workflow 自带 `verify` 前置 job，门禁不依赖 `ci.yml` 的执行结果，避免「PR 门禁绿了但 tag 打包跳过检查」的分叉路径。三个 `package-*` job 互相独立且并行，任一失败即整体失败，`release` 不执行——部分发布比红色构建更难排查，因此不做 `if: always()` 的部分交付。

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

## 打包矩阵

每个平台一个 job，各跑唯一入口 `scripts/bundle.mjs`（经根 `package.json` 的 `bundle:desktop` 脚本）。不拆解它内部步骤（prepare → build → electron-builder → 运行时依赖机械校验 → 体积审计）。该脚本是打包链路的唯一所有者，已有镜像 fallback 与瞬时网络失败重试；workflow 复制它的步骤会制造第二条写入路径。

| Job                   | Runner          | 命令                                  | 输出目录           | 目标产物                                              |
| --------------------- | --------------- | ------------------------------------- | ------------------ | ----------------------------------------------------- |
| `package-macos-arm64` | `macos-14`      | `pnpm bundle:desktop -- --os mac --arch arm64`   | `dist-macos-arm64` | `.dmg`、`.zip`                                        |
| `package-win-x64`     | `windows-latest`| `pnpm bundle:desktop -- --os win --arch x64`    | `dist-win-x64`     | `.exe`（NSIS）                                       |
| `package-linux-x64`   | `ubuntu-latest` | `pnpm bundle:desktop -- --os linux --arch x64`  | `dist-linux-x64`   | `.AppImage`、`.deb`、`.rpm`、`.pkg.tar.zst`          |

**架构选型**：Windows 与 Linux 只出 x64。`packages/desktop/scripts/desktop-native-package-policy.mjs:1` 的 `SUPPORTED_DESKTOP_PLATFORM_KEYS` 与 `scripts/native-search-tools-config.mjs:193` 的 `ENABLED_NATIVE_SEARCH_PLATFORM_KEYS` 都覆盖 `*-arm64`，工具链具备能力；但 CI 矩阵按主流桌面基线收敛到 x64，macOS 沿用现状只出 arm64。arm64 变体需要单独加 job，不属于本次范围。

**超时**：macOS 沿用 60 分钟；Windows 与 Linux 给 90 分钟。Linux job 要串行产出四个安装包格式，Windows 首轮还要下载 NSIS 工具链，两者的 cache miss 成本高于 macOS 的 dmg-builder。

### 环境变量契约

三个 job 共享同一套约定，只有平台相关的四个变量不同：

| 变量                       | mac                          | win                          | linux                        | 依据                                                             |
| -------------------------- | ---------------------------- | ---------------------------- | ---------------------------- | ---------------------------------------------------------------- |
| `ZCODE_TARGET_OS`          | `mac`                        | `win`                        | `linux`                      | `packages/desktop/scripts/target-platform.mjs:38`                |
| `ZCODE_TARGET_ARCH`        | `arm64`                      | `x64`                        | `x64`                        | 同上                                                             |
| `ZCODE_DESKTOP_DIST_DIR`   | `dist-macos-arm64`           | `dist-win-x64`               | `dist-linux-x64`             | 按平台/架构隔离输出；`.gitignore:6` 已忽略 `packages/desktop/dist-*/` |
| `ZCODE_SKIP_REMOTE_ASSETS` | `1`                          | `1`                          | `1`                          | 见下节                                                           |
| `ZCODE_ENV`                | `test`                       | `test`                       | `test`                       | 见下节                                                           |
| `ZCODE_ENABLE_MAC_SIGN`    | 未设置                       | 不适用                       | 不适用                       | 关闭签名                                                         |
| `ZCODE_PREVIEW_IDENTITY`   | 未设置                       | 未设置                       | 未设置                       | 身份由 `ZCODE_ENV` 推导                                          |

`ELECTRON_CACHE` 与 `ELECTRON_BUILDER_CACHE` 一律指向 `${{ github.workspace }}` 下的工作区内目录，与 `.github/actions/setup-repo` 的缓存路径一致。Windows job 使用反斜杠分隔（`${{ github.workspace }}\.electron-cache`），Linux/macOS 使用正斜杠。

**跳过 remote assets**：`prepare:remote-assets`（`scripts/prepare-prebuilds.mjs`）会下载 Node 运行时并打包跨平台远端 agent 二进制，组件源缺失且未配置 `ZCODE_DEPS_BASE_URL` / `INTRANET_MACHINE_HOST` 时会 skip（`scripts/prepare-prebuilds.mjs:963`）。桌面安装包不需要 mock-cdn：`electron-builder.config.js` 的 `extraResources` 只引用 `bundled-agents/<platform>/glm`（由 `prepare:agent-bundle` 本地生成，agent 无原生 NAPI 插件）与 `bundled-tools/<platform>/{ripgrep,bfs,ugrep}`（由 `prepare:native-search` 解包仓库内置归档），二者都在 `localRuntimeScripts` 中，不受该开关影响。三平台所需的归档都已随仓库分发（`apps/zcode-cli/dependencies/native-search/`），且 SHA-256 与 `SHA256SUMS` 一致，不需要任何下载源：`linux-x64` 为 `bfs`/`ugrep`（`x86_64-unknown-linux-gnu`）加 `ripgrep`（`x86_64-unknown-linux-musl`），`win32-x64` 为 `ugrep`/`ripgrep`（`x86_64-pc-windows-msvc`）——Windows 不出 `bfs`，见 `scripts/native-search-tools-config.mjs:193` 起的平台键开关。`packages/desktop/scripts/prepare-runtime-assets.mjs:56-59` 的注释已记录原 CI 因无条件执行该步骤而逼近 1 小时上限，故 CI 显式跳过。

**后端环境设为 test**：不设 `ZCODE_ENV` 时 `desktop-product-identity.mjs` 推导为 test，产物使用 `ZCode Preview` 身份并带 `_TEST` 文件名后缀。这是刻意的 fail-safe：CI 未配置任何签名身份，产物不能顶着正式 `ZCode` 身份流出被误认为发布包。需要生产身份出包时显式设置 `ZCODE_ENV=production`，并在同一 job 注入对应平台的签名凭据与 `ZCODE_ENABLE_MAC_SIGN=1`。

### Windows 平台前提

- 未注入 `CSC_LINK` / `CSC_KEY_PASSWORD` 时，electron-builder 产出**未签名** NSIS 安装包，job 不因此失败。这是有意的：CI 没有代码签名证书。
- 不需要额外系统工具。`packages/desktop/build/icon.ico`、`icon_installer.ico`、`installer.nsh` 已入库；`build/installer.nsh` 由 electron-builder 对 NSIS target 自动包含。
- `node-pty` 使用 `packages/desktop/native/prebuilds/win32-x64/` 的预编译产物（`node-pty-rebuild.mjs` 对 win32 走 prebuild 复用路径），不触发源码编译。
- `ZCODE_ENABLE_WINDOWS_BROWSER_IMPORT` 未设置，`prepare:browser-import-helper` 按 `prepare-runtime-assets.mjs:19` 跳过，不扩大签名面。

### Linux 平台前提

`ubuntu-latest` 当前解析为 `ubuntu-24.04`。`linux.target` 配置了四种格式（`electron-builder.config.js:695`），各自的输出工具都由 electron-builder 从 electron-builder-binaries 自下载（fpm `1.17.0-ruby-3.4.3`、appimage `12.0.1`），但它们依赖若干**主机**二进制，这些不在 ubuntu-24.04 默认镜像里：

| apt 包              | 提供的二进制 | 服务的目标 | 缺失时的失败信号                                    |
| ------------------- | ------------ | ---------- | --------------------------------------------------- |
| `binutils`          | `ar`         | `deb`      | `Need executable 'ar' to convert dir to deb`        |
| `rpm`               | `rpmbuild`   | `rpm`      | `Need executable 'rpmbuild' to convert dir to rpm`  |
| `xz-utils`          | `xz`         | `rpm`      | `xz: not found`（`FpmTarget.js` 的错误提示分支）    |
| `libarchive-tools`  | `bsdtar`     | `pacman`   | `Process failed: /bin/sh failed (exit code 127)`    |
| `zstd`              | `zstd`       | `pacman`   | `tar failed (exit code 2)`，`tar --zstd` 无压缩器  |
| `dpkg-dev`、`fakeroot` | `dpkg-deb` 等 | `deb`     | 防御性安装，实测未成为瓶颈                          |

electron-builder 自身在 `app-builder-lib/out/targets/FpmTarget.js:255` 与 `:260` 就打印了 `rpmbuild` / `xz` 的缺失提示；`ar`、`bsdtar`、`zstd` 三条需要实测定位（见「实测记录」）。Linux job 必须先装齐这组包，否则四个目标里第一个失败会中断整个 job。

## Release 交付

`release` job 在三平台打包全部成功后执行，唯一所有者是 `release` job 的 `Publish GitHub Release` step：

1. `actions/download-artifact@v4` 按 `pattern: desktop-*` 把三个平台的 artifact 下载到 `artifacts/<artifact-name>/`。
2. 只把安装包扩展名（`dmg`、`zip`、`exe`、`AppImage`、`deb`、`rpm`、`zst`）的文件复制进 `release-assets/`。**`builder-debug.yml` 留在 workflow artifact 里不进 Release**：它是构建过程诊断，不是交付物，且三个平台同名会互相覆盖。
3. `gh release create "$tag" --target "$GITHUB_SHA" --generate-notes` 创建 Release，附件经 bash 数组传入，保证 `ZCode Preview-..._TEST.dmg` 这类含空格的文件名不被 shell 拆词。

**权限边界**：workflow 顶层 `permissions: contents: read`，只有 `release` job 提升为 `contents: write`。三个打包 job 不需要任何写权限，无法改动远端仓库状态。

**幂等**：同一 tag 重跑时先 `gh release delete --cleanup-tag=false` 再重建，只删 Release 对象、保留 git tag。Release 由 CI 全权持有，重跑必须能刷新，否则一次失败会永久留下半成品。

**只发布 tag 触发的运行**：`github.ref_type != 'tag'` 时（即 `workflow_dispatch` 从分支触发）跳过 Release 创建，安装包仍保留在 workflow artifacts 中。分支上没有版本 tag，不臆造 tag 也不覆盖已发布的 Release；需要手动产出 Release 时推一个 tag 即可。

**不发布的内容**：`*.blockmap`、`latest-mac.yml`、`latest-linux.yml`、`linux-unpacked/`、`win-unpacked/`。差分更新清单与 unpacked 目录属于构建中间态，自动更新 feed 不由 GitHub Release 提供（见「目的与范围」），把它们放进 Release 会让人误以为自动更新已接好。

## 不变量

- workflow 不写入仓库源码树；所有中间产物落在 `dist-<os>-<arch>/`（已 gitignore）与工作区内缓存目录。
- CI 不刷新架构基线（`pnpm architecture:baseline:update` 仅在评审过的变更中由人执行）。
- CI 不产出签名产物：`mac.identity` 在 `ZCODE_ENABLE_MAC_SIGN != 1` 时为 `null`（`electron-builder.config.js:670`），Windows 不注入 `CSC_LINK`。
- artifact 与 Release 附件的命名由 `electron-builder.config.js:236` 的 `buildDesktopArtifactName` 决定，workflow 不改写产物文件名，也不做跨平台重命名。
- Release 附件集合是产物文件名的子集（按扩展名筛选），不新增、不合并、不重打包。

## 缓存

三个缓存目录与 `.gitignore:26-29` 预留位一一对应：

| 目录                      | 环境变量                    | 内容                                           |
| ------------------------- | --------------------------- | ---------------------------------------------- |
| `.pnpm-store`             | `pnpm config set store-dir` | pnpm 内容寻址存储                              |
| `.electron-cache`         | `ELECTRON_CACHE`            | Electron 运行时 zip                            |
| `.electron-builder-cache` | `ELECTRON_BUILDER_CACHE`    | dmg-builder、app-builder-lib、fpm、appimage 等 builder 辅助包 |

缓存 key 以 `pnpm-lock.yaml` 与 Node 版本为输入；缓存 miss 不应导致 job 失败（脚本自带镜像 fallback 与重试）。Linux 的 `apt-get install` 不进缓存，单次成本约数秒，不值得为此引入一层 apt cache。

## 失败语义

- `verify` 任一命令非零退出 → job 失败，三个 `package-*` 因 `needs` 不执行。
- 任一 `package-*` 失败 → `release` 因 `needs` 不执行，不产出部分 Release。
- 打包脚本自身已内建 fail-closed 校验：缺运行时依赖（`bundle.mjs:638` 起的 `verifyPackagedRuntimeDependencies`）、越界 native 资源、`node-pty` 预编译产物缺失、运行时依赖未进 `app.asar` 均直接抛错，不允许「产物存在但无法启动」的安装包流出。
- artifact 上传使用 `if-no-files-found: error`，且上传 step 失败会让 job 失败。产物未保留等价于本次打包没有交付，必须 fail-closed 暴露；重跑该 job 时缓存命中，边际成本只剩 electron-builder 阶段。
- `release` 的 `Stage release assets` 在筛出 0 个安装包时显式失败，不创建空 Release。

## 验收场景

| 场景                                           | 期望                                                                                               |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| PR 修改 `packages/*/src`                       | `ci.yml` 的 `verify` 运行三项检查并给出通过/失败结论                                               |
| 类型检查不通过                                 | `verify` 失败，日志指向 `pnpm typecheck`                                                           |
| push tag `v3.14.0`                             | 先 `verify`，再并行出三个平台安装包，全部成功后创建 GitHub Release，附件含 7 个安装包（mac 2 + win 1 + linux 4） |
| 手动 `workflow_dispatch`（选分支）             | 出包与上传 artifact 与 tag 触发一致，但跳过 Release 创建并打印说明；不臆造 tag                     |
| 同一 tag 重跑                                  | 删除既有 Release（保留 git tag）后重建，附件刷新为本次运行产物                                     |
| 三个平台中任一打包失败                         | 整个 workflow 红色，不创建 Release                                                                 |
| 打包脚本因缺依赖失败                           | 该 job 失败，不上传 artifact                                                                       |
| 设置 `ZCODE_ENABLE_MAC_SIGN=1` 但无 Apple 凭据 | 未显式设置该变量，故不进入签名分支；若日后打开，缺身份时 `electron-builder.config.js:211` 提前抛错 |

## 实测记录

在干净工作区按 CI 相同的环境变量实跑 `pnpm bundle:desktop -- --os mac --arch arm64`（macOS arm64 宿主）：

- 产物：`ZCode Preview-3.14.0-mac-arm64_TEST.dmg`（177 MiB）与同名 `.zip`（169 MiB），附带 `.blockmap`、`builder-debug.yml`、`latest-mac.yml`，全部落在 `packages/desktop/dist-macos-arm64/`，由 `.gitignore:6` 覆盖。
- 体积审计：`.zip` 169.0 MiB，上限 500 MiB，通过。
- 签名：日志为 `skipped macOS code signing  reason=identity explicitly is set to null`，与未开启 `ZCODE_ENABLE_MAC_SIGN` 的预期一致。
- 门禁：`pnpm typecheck`、`pnpm lint`（70 warnings / 0 errors）、`pnpm architecture:check`（全量与 `--changed` 两种语义）均通过。
- Node 版本：本地宿主为 v25.2.1，`apps/zcode-cli` 的 `engines.node` 要求 `24.14.0`，打包过程持续输出 engine warning 但仍成功。这印证 workflow 必须按 `mise.toml` 固定 24.14.0，而不是取 runner 默认版本。

Linux 主机工具链在 `docker run --platform linux/amd64 ubuntu:24.04` 容器内验证（electron-builder `26.8.1`、electron `41.0.3`、npm 安装的 pnpm `10.33.2`，与 `mise.toml` 一致），用一个最小 Electron 应用跑 `electron-builder --linux --x64`：

- 首轮失败：`deb` 目标报 `Need executable 'ar' to convert dir to deb`，装 `binutils` 后通过；`AppImage` 在同一轮已成功（114 MiB），说明 electron 运行时与 appimage 工具链下载不受影响。
- 第二轮四种格式全部产出：`.AppImage`（114 MiB）、`.deb`（89 MiB，文件名后缀 `-amd64`）、`.rpm`（79 MiB，`-x86_64`）、`.pacman`（81 MiB，未覆盖 `pacman.artifactName` 时为 `.pacman`；仓库配置改成 `.pkg.tar.zst`）。
- 确认 `fpm-1.17.0-ruby-3.4.3-linux-amd64` 与 `appimage-12.0.1` 从 `registry.npmmirror.com/-/binary/electron-builder-binaries/` 下载，镜像 fallback 生效。
- 构建 AppImage 不需要 fuse，运行时库只在**运行**时依赖；打包阶段只下载工具链。

原生搜索资产在本地按 `resolveNativeSearchPrebuiltPlan` 解析目标平台后逐文件核对（macOS arm64 宿主）：

- `linux-x64` 出 3 个工具（`bfs`、`ugrep`、`ripgrep`），`win32-x64` 出 2 个（Windows 不出 `bfs`），`linux-arm64` 出 3 个；全部归档文件存在，SHA-256 与 `apps/zcode-cli/dependencies/native-search/SHA256SUMS` 逐项一致。
- `ripgrep` 走 `NATIVE_SEARCH_OFFICIAL_RIPGREP_ASSETS`（`scripts/native-search-tools-config.mjs:111`），Linux 用的是 **musl** 归档（`x86_64-unknown-linux-musl`），不是 glibc；`bfs`/`ugrep` 走 producer 归档（gnu）。`prepare:native-search` 因此完全离线可跑。

**未验证项**（如实记录）：

- Windows runner 未实测。本机无法运行 Windows 容器，`windows-latest` 上的 `pnpm install`、NSIS 工具链下载、`patch-nsis-install-section.mjs` 对 `node_modules` 内模板的修补与还原都只在 macOS 宿主上做过静态核对。
- 三平台全量 `pnpm bundle:desktop` 未在 GitHub runner 上跑过（首次运行需推 `v*` tag 或 `workflow_dispatch`）。已验证的是宿主工具链层（Linux）与打包脚本的入口/参数解析路径，不是端到端。

## 与本地开发的边界

- `pnpm verify:pre-push` 仍用于本地提交前快速检查，保留 `--changed` 语义。
- CI 是产物与门禁事实的唯一来源；本地打包结果不入库、不作为发布依据。
