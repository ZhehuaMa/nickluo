# Build and test entry points

The repository scripts default to the x86_64 development image
`drone-tracker-rk3576:humble-amd64-rknn2.3.2`. Select `--backend arm64` to use the existing
ARM64 image explicitly. A failed build or test never triggers an automatic backend switch.
The backend selects the container's tools; the output architecture depends on the entry point.

| Entry point | Default amd64 output | Explicit arm64 output |
|---|---|---|
| `drone_tracker/build.sh` | ARM64 tracker, compiled by an x86_64 cross-compiler | ARM64 tracker, compiled by ARM64 tools |
| Tracker SDK test helper | ARM64 tests, with architecture-aware supporting tools | ARM64 tests in the ARM64 image |
| Interceptor scenario scripts | Native x86_64 node and messages for software simulation | ARM64 node and messages for software simulation |

A cross-compiler runs on x86_64 and produces ARM64 programs. An ARM64 program still needs
an ARM64 machine or QEMU, which emulates ARM instructions. Keep native and target libraries,
generated Python modules, and build caches separate.

## Tracker builds and tests

From `/home/mzh/nickluo/drone_tracker`:

```bash
./build.sh
./build.sh --test
bash tests/tracking_regression/run_sdk_tests.sh --regression-only

# Explicit fallback; these commands retain any real failures.
./build.sh --backend arm64
bash tests/tracking_regression/prepare_test_tools.sh --backend arm64
./build.sh --backend arm64 --test
```

The default test workflow supplies the native FFmpeg command-line tools used to generate
synthetic video. Python regression runs all 53 unique cases exactly once: 52 host orchestration
and native-compiler cases use native Python; the ARM ROS library-loading case uses target
Python and the target library loader. Both groups share the existing 30-second CTest limit,
and failure or a skipped case fails the aggregate check. Native Python must not load generated
ARM extension modules. The ARM64 fallback's private
FFmpeg tools must be prepared separately. Routine build and test containers have no network.

Use `--artifacts-dir /absolute/path` to place output outside the source repository. Each
backend uses `ROOT/backends/BACKEND/{build,install,log}`. The `--clean` option backs up the
selected backend's old output before rebuilding. Legacy `.docker-artifacts/build`, `install`,
and `log` directories are preserved. Image overrides are checked against the selected backend.

The deployment packager retains its legacy install-directory default. Always select the
newly built install tree explicitly, so an old build cannot be packaged accidentally:

```bash
bash deployment/package.sh /tmp/drone-tracker-release \
  --install-root "$PWD/.docker-artifacts/backends/amd64/install/drone_tracker" \
  --model-root /absolute/path/to/models
```

For the fallback output, replace `amd64` with `arm64`. With a custom artifact root, use its
absolute install path instead. The dependency collector still runs in the preserved ARM64
image; cross-compilation has not migrated this separate packaging step. Packaging does not
establish that the binary has passed device acceptance.

## Interceptor simulation

From `/home/mzh/nickluo/drone_interceptor`:

```bash
bash prototype/run_scenario.sh nominal
bash prototype/run_all_scenarios.sh --vision-only
bash prototype/run_all_scenarios.sh
bash prototype/run_scenario.sh --backend arm64 nominal
```

The runners build the real adjacent `quadrotor_msgs` source and the enabled C++ node. Each
run uses a fresh result directory, a read-only source mount, ROS 2 Humble, and an isolated
container network. The default artifact root is `/tmp/nickluo-interceptor-scenarios`; set
`--artifacts-dir` to preserve results elsewhere. Build caches are separated by backend and
immutable image ID. The simulator and its unchanged configuration are staged into the result
directory because the simulator writes its CSV and summary beside its script.

`run_one.sh` delegates to the same runner. Leading `launch_arg:=value` overrides and simulator
arguments remain supported. `radar_reintercept_enable:=true` is supplied by the complete suite
for the existing reinterception case.

A scenario passes only when its process succeeds and its fresh summary reports both
`Intercepted: True` and `NaN commands: 0`. NaN means an invalid numeric command. The existing
interception threshold is unchanged. Each simulator process has a 90-second limit. The suite
retains every existing scenario and returns nonzero if any scenario fails.

The ARM64 fallback compiled successfully on the current host, but its nominal ROS simulation
still received no control commands under emulation. That result is a failure. The default
native runner passed the six vision cases; `radar_only` remains an unmet interception case.

## Other projects and validation boundaries

All five submodules support the default x86_64 development image. The tested recipes below
separate native x86_64 output, ARM64 cross-compilation, and explicit ARM64-image fallback:

- [apm_bridge builds and protocol tests](../apm_bridge/docs/build-and-test.md).
- [quadrotor_msgs interface generation and serialization](../quadrotor_msgs/README.md).
- [rk_hdmi_osd portable tests and full ARM64 stack/bridge builds](../rk_hdmi_osd/BUILDING.md).
- [Interceptor build architectures](../drone_interceptor/docs/DOCKER_BUILD.md).
- [Tracker cross-build workflow and portable checks](../drone_tracker/dockerfiles/README.amd64.md).

The [submodule audit](submodule-image-audit-20261015.md) records fresh cross-build and
runtime checks. Tracker dependency collection and interceptor bundle assembly retain their
documented ARM64-environment requirements. The x86_64 image builder also imports its target
SDK from the preserved ARM64 image.

Both images can compile the current supported ARM64 targets. Neither supplies the RK3576
kernel drivers or peripherals on this x86_64 host. Camera, NPU inference, HDMI, physical
gimbal, firmware links, and flight behavior require separate device verification. Neither
Jetson builds nor disabled acados/NMPC controller variants are covered by these commands.

The earlier image and test evidence remains in `amd64-build-validation-20261015.md`.
Do not reinterpret its failed runs as passing after a script or emulator change. New checks
must retain their image ID, commands, output, and actual exit status.

## Verification of the adapted workflow

The actual default `./build.sh --test` passed all 57 CTests on image
`sha256:025d0aa88983b0646a1b7a4d25cbc13dc96a6206e0148475bbfd5fad15057421`.
It also passed the mandatory native OS memory-limit check. The source test retains its
90-second limit, diagnostics retain 120 seconds, and Python retains its aggregate 30 seconds.
The complete run took 156.28 seconds for the 57 CTests. It used the pinned, patched image
emulator explicitly and the `cortex-a72` CPU model, matching the production CPU family.

The explicit ARM64 tracker production build passed in a separate cache. Its complete test
suite is not claimed as passing: the preserved image still uses the host's older emulation
on this machine. Its real OS memory-limit check remains enabled and exposes unsupported
emulation rather than counting it as native OS coverage.

The interceptor's six native vision scenarios passed through the adapted suite runner.
The final image also passed a fresh native nominal build and simulation. The original
`radar_only` case returns nonzero. The explicit ARM64 fallback compiled but its nominal
simulation still received no control commands and returned nonzero.

Detailed root causes, unchanged assertions, retained native OS-limit coverage, deliberate
failure controls, and remaining radar/width observations are recorded in
`native-default-test-investigation-20261015.md`. No production algorithm or acceptance
threshold was changed. Runtime evidence is retained under:

- `/tmp/nickluo-build-workflow-20261015/`: build helper, fallback, image, and full-suite logs.
- `/tmp/nickluo-native-default-investigation-20261015/`: focused proofs and failure controls.
- `/tmp/nickluo-interceptor-workflow-20261015/`: scenario builds, summaries, CSV, and exit codes.

Preserve these directories separately before cleaning `/tmp` if the full evidence is needed.
