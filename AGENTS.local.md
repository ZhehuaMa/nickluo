# nickluo 远程开发机约定

- 当前开发机为 `mzh`，通过 `ssh mzh` 从原工作站登录；宿主机为 Ubuntu 22.04.5 LTS、Linux x86_64，CPU 为 Intel Core Ultra 9 285K，24 个逻辑 CPU，内存约 62 GiB。
- 工作区根目录为 `/home/mzh/nickluo`。五个子仓库保留各自 Git 历史和未提交修改；逐仓库检查差异，不跨仓库提交，不自动提交或推送。
- Docker is available to `mzh`. Build/test scripts now default to `drone-tracker-rk3576:humble-amd64-rknn2.3.2` (`linux/amd64`). Select `--backend arm64` for the preserved `drone-tracker-rk3576:humble-arm64-rknn2.3.2` fallback. This x86_64 host runs ARM64 containers through QEMU; it is not an ARM64 machine or an RK3576 board.
- 使用 ARM64 容器时显式传入 `--platform linux/arm64`。容器内有 ROS 2 Humble、GCC 11.4.0、CMake 3.22.1、Python 3.10.12；RKNN Runtime 2.3.2 库 SHA-256 为 `d31fc19c85b85f6091b2bd0f6af9d962d5264a4e410bfb536402ec92bac738e8`。
- The host provides ROS 2 Humble, GCC 11.4.0, CMake 3.22.1, and system Python 3.10.12. Tracker builds remain ARM64 with either the amd64 cross-compiler backend or the ARM64 fallback. Interceptor scenario scripts build native binaries for their selected image architecture. Conda base is now available at `/home/mzh/miniconda3/bin/python` (Python 3.14.6); prefer it for general host Python work. ROS Humble checks use the matching Python 3.10 environment, normally inside the image.
- `~/.codex/language_norms/cpp_norms.md` 已存在于远程开发机。修改 C++ 前读取它。
- 远程开发机未配置 `firefly1`、`firefly2`、`firefly-production` SSH 别名。本次迁移没有复制 SSH 私钥、修改 SSH 配置或连接开发板。需要板端验证时先确认远程机的访问配置；现有容器检查不能替代板端验收。

## 迁移与验证

- 2026-10-15 从 `/Users/mazhehua/projects/nickluo` 完整复制到本目录，包含隐藏文件、Git 元数据、未提交文件、模型和 `.docker-artifacts/`。原工作站源码保持原状。
- 当前镜像与旧镜像 `roc-rk3576-ros2-build:humble-arm64` 已传入 Docker；旧镜像只用于历史复现，不作为当前发布入口。
- 镜像归档、迁移校验和文档修改前备份放在 `/home/mzh/nickluo-migration-20261015/`，不属于任何源码仓库。
- 活动文档使用当前工作区路径。历史报告保留原始机器、日期、路径和验收结论。报告中的 `/Users/mazhehua/projects/nickluo/` 前缀对应当前 `/home/mzh/nickluo/`，不要把旧报告解释为本机刚完成的验证。
- 历史证据中指向 `/home/hzhy` 或 `/opt/drone_tracker` 的绝对符号链接属于板端快照。保留原链接，不代表远程开发机具有相应部署。
- Migrated build caches may contain old workstation paths. Preserve them and use the new per-backend cache directories. The earlier plain `./build.sh --test` configuration failure is historical: the adapted entry point now supplies fixture-tool paths and target-aware test launchers. The explicit ARM64 backend still needs its private FFmpeg tools prepared first. Migration-era evidence remains in `/home/mzh/nickluo-validation-20261015/`; current workflow and investigation evidence is recorded below.
- `drone_tracker/.git/worktrees/add-nanotrackv3-benchmark-validation/gitdir` 保留指向原工作站外部 worktree 的历史引用。该外部目录不在本次复制范围内；本次未裁剪记录，使用该 worktree 前须另行恢复或修复。主工作区不依赖该外部目录。
- Git 的 `core.fsmonitor` 已在六个远程仓库中设为 `false`。fsmonitor 是文件变化监视器；四个从原工作站复制的运行时 socket 节点已删除，防止远程 Git 尝试使用失效进程。备份与清单见迁移记录目录。
- 逐仓库读取对应 `AGENTS.md` 与 `AGENTS.local.md`。本文件只记录开发环境，不改变接口或生产板规格。

## Initial x86_64 image evidence (2026-10-15)

- The first completed `linux/amd64` image validation used `sha256:156c071fcff2385947d0adaea176b29be1b3fb110d2969ebddfda39f2d9d99fd`. Later workflow repairs may update the normal image tag; retain this ID when interpreting the original evidence. Use `--platform linux/amd64` for the native image.
- Native builds produce x86_64 binaries. RK3576 targets use the native ARM64 cross-compiler and `/opt/toolchains/aarch64-humble.cmake`. A cross-compiler runs on this x86_64 host but produces ARM64 binaries. Target ROS and hardware libraries are isolated under `/opt/arm64-sysroot`.
- For ARM64 test subprocesses, export `QEMU_LD_PREFIX=/opt/arm64-sysroot` in the container. Keep native and ARM64 build/install directories separate. QEMU results do not establish camera, NPU, HDMI, timing, or flight behavior.
- The existing ARM64 production image remains available and unchanged. The new image does not change the existing production build or deployment entry points.
- Registry downloads stalled during this build. The delivered image therefore uses an explicitly named local Ubuntu/ROS package snapshot. Rebuild commands and package provenance are documented in `drone_tracker/dockerfiles/README.amd64.md`.
- Image-build evidence is under `/tmp/nickluo-amd64-image-20261015/`; its small provenance files are also preserved in the ignored `drone_tracker/.docker-artifacts/amd64-image-20261015/provenance/` directory. Project validation outputs are under `/tmp/nickluo-amd64-validation-20261015/`. Preserve these directories when retaining the evidence; `/tmp` may be cleaned by the operating system.
- The per-project results and unresolved test failures are recorded in `docs/amd64-build-validation-20261015.md`. The default ARM/QEMU tracker suite passed 54/57 checks; do not describe it as fully passing. The target-Python rerun passed all 53 Python cases. Diagnostics under QEMU and the source memory-limit test remain unmet; the separate C++ ROS simulation suite met the interception criterion in 13/14 scenarios.

## Current build and test entry points

- The current native image ID is `sha256:025d0aa88983b0646a1b7a4d25cbc13dc96a6206e0148475bbfd5fad15057421`. It adds a separately installed, pinned QEMU 8.2.2 with the verified descriptor-truncation repair. The original distribution emulator and host registration are unchanged. The earlier `156c...` image is retained under the `humble-amd64-rknn2.3.2-before-qemu-repair-20261015` tag.
- Read `docs/build-and-test-workflow.md` for default commands, explicit fallback, output architectures, and packaging paths. Older ARM64-only command examples above describe fallback or historical validation.
- Tracker `./build.sh --test` and `tests/tracking_regression/run_sdk_tests.sh` now share backend selection and supply their test-tool paths. The earlier plain `--test` configuration failure is historical. Keep new caches under `.docker-artifacts/backends/BACKEND/` or the selected external artifact root; preserve legacy caches.
- Packaging must explicitly select `--install-root` from the chosen backend. The packager's legacy default has not been redirected to a newly built tree.
- The all-submodule architecture audit is recorded in `docs/submodule-image-audit-20261015.md`. All five repositories support the x86_64 image, including genuine ARM64 cross-builds. Use their revised local build guides. Tracker dependency collection and interceptor bundle assembly still require their documented ARM64 environments; the amd64 image builder also imports its SDK from the preserved ARM64 image.
- Interceptor scenario scripts use ROS 2 Humble containers and external output directories. Six default native vision scenarios passed; explicit emulated ARM64 nominal simulation and the native `radar_only` case remain failures. Do not replace a failed verdict with a successful process exit or another backend.
- The adapted default tracker `./build.sh --test` passed 57/57 CTests plus the mandatory native OS memory-limit check on image `025d...`. Python regression runs all 53 unique cases exactly once: 52 native host-tool cases and one genuine ARM library-loading case, within the original aggregate timeout. No tests are skipped. The earlier 54/57 result remains historical evidence for image `156c...`.
- Current investigation details and remaining simulation limitations are in `docs/native-default-test-investigation-20261015.md`. The descriptor-leak and Python failure controls still return failure. No production controller thresholds, test deadlines, host emulator registration, or physical deployment were changed.
