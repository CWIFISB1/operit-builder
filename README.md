# operit-builder

Operit APK 专用构建器。源码不入库，构建时 clone；**产物直接发布到 Release，不走 Actions artifact**。

## 为什么单独建这个仓库

| 问题 | 原仓库 `CWIFISB1/Operit` | 本仓库 |
|---|---|---|
| artifact 配额 | 每次 421MB × N，配额爆过 | **完全不用 artifact**，产物进 Release |
| 源码污染 | workflow 改动混在业务提交里 | 构建逻辑独立 |
| runner 磁盘 | 中间产物堆积 | 每轮空闲实例，构建前释放磁盘 |
| 依赖来源 | Google Drive（`subpack.zip` 返回 2406 字节病毒拦截页） | **Release 镜像优先**，Drive 仅兜底 |
| 触发参数 | `gradle_task` | `repo` / `ref` / `gradle_task` / `publish_release` |

## 依赖镜像

预构建依赖放在 `CWIFISB1/operit-build` 的 `prebuilt-deps` Release：

- `libs.zip` 14.5 MB
- `subpack.zip` 33.6 MB
- `jniLibs.zip` 2.2 MB
- `ffmpeg-kit-local.aar` 14.6 MB

下载逻辑：先 `curl -fsSL --retry 3` 打镜像 → 非空直接用；失败才 `gdown` 走原 Google Drive。

## 用法

`Actions → Build APK → Run workflow`

| 参数 | 说明 |
|---|---|
| `repo` | 默认 `CWIFISB1/Operit` |
| `ref` | 分支或 commit |
| `gradle_task` | 默认 `assembleDebug` |
| `publish_release` | 默认开，成功后建 Release 并挂 APK |
| `release_tag` | 留空自动 `build-<run_number>` |

产物地址：
`https://github.com/CWIFISB1/operit-builder/releases`

## 需要 Secrets

| 名 | 用途 |
|---|---|
| `GH_PAT` | clone 私有源码、拉 terminal 子模块、建 Release、上传产物 |

## 关键排障记录

| 报错 | 根因 | 修法 |
|---|---|---|
| `BUILD FAILED in 54s` | runner 默认 JDK 25 与 AGP 不兼容 | 固定 temurin 21 |
| `No matching variant of :terminal` | NDK 被删，`:terminal` CMake 配置失败 | 不删 NDK，只在 sdkmanager 里装 |
| `No variants exist.` | `terminal` 是 submodule，未递归拉取 | `git submodule update --init --recursive` |
| `Permission denied (publickey)` | submodule 用 `git@github.com:` 地址 | `url.insteadOf` 重写为 HTTPS + token |
| `Missing liboperit_ripgrep.so` | 需 Rust 交叉编译 | `cargo build --target aarch64-linux-android`，NDK 25.1.8937393 |
| `Missing ffmpeg-kit-local.aar` | 被 `.gitignore /app/libs/*` 排除 | 从镜像 Release 下载 |
| `Artifact storage quota has been hit` | artifact 配额满（6–12 小时才重算） | **本仓库不用 artifact**，改走 Release |
