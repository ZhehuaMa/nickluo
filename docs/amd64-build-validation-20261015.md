# amd64 build-image validation — 2026-10-15

The delivered image runs native x86_64 build tools. RK3576 programs remain genuine ARM64 targets. Cross-compilation means the x86_64 compiler produces ARM64 executables; QEMU runs selected ARM64 tests without the board.

`CTest` is CMake's test runner. ELF checks inspect an executable's target machine architecture. The gimbal simulation uses a pseudo-terminal (PTY), a virtual serial port that replaces the physical serial connection for this test.

The Python ABI, or binary interface, determines which generated extension modules the target interpreter can load. ROS type support is the generated code that converts messages for transport. NaN means an invalid “not a number” value; those control commands are counted separately from interception success.

Image rebuild instructions and the local package-snapshot provenance are in `/home/mzh/nickluo/drone_tracker/dockerfiles/README.amd64.md`. The original ARM64 production image remains unchanged.

All five repositories and `osd_bridge` compiled within their supported architecture. Final native checks passed 35 portable CTests plus one diagnostic CTest. The complete ARM/QEMU tracker run passed 54/57 CTests. A target-Python rerun passed all 53 Python cases, while diagnostic and source tests remain unmet. Actual C++ ROS simulations met the interception criterion in 13/14 scenarios.

## Image and source boundaries

- Delivered tag: `drone-tracker-rk3576:humble-amd64-rknn2.3.2`.
- Final immutable image ID: `sha256:156c071fcff2385947d0adaea176b29be1b3fb110d2969ebddfda39f2d9d99fd` (`linux/amd64`).
- Initial image ID: `sha256:7b63829372e2cf10af24f619d619b01b858e7458538c352905a0a4f7d5c1c520`.
- All validation containers used `--platform linux/amd64`, `--network none`, read-only source mounts, and fresh external build/install/log directories. No validation command changed production source, committed files, pushed branches, or updated submodule pointers.
- Full native and ARM builds were repeated from clean directories after the final image incorporated the proven sysroot repairs. No runtime sysroot repair was applied to that final image.

## Per-project results

| Project | Compilation in final image | Executed checks |
|---|---|---|
| `quadrotor_msgs` | Native x86_64 and cross ARM64; generated from actual source | Native consumers linked successfully. ARM Python imported both messages and their real type support. Clean ARM extension filenames use `cpython-310-aarch64-linux-gnu`. No registered CTests. |
| `apm_bridge` | Native x86_64, both installed executables verified | Existing PTY firmware simulation passed C200T, C40D, C20S; measured 492, 496, 498 packets/s. No registered colcon tests. |
| `drone_interceptor` | Native x86_64 IMM-UKF node, verified ELF architecture | Python algorithm simulation: 7/7 interceptions and zero NaN commands. Existing real C++ ROS scenarios: 13/14 met interception criterion; `radar_only` failed at minimum range 2.08 m. No registered colcon tests. |
| `drone_tracker` | Complete ARM64 node, OSD, recording, diagnostics, benchmarks, regression targets | Native portable subsets: OSD 9/9, algorithms 9/9, camera 16/16 CTests. Additional native diagnostics 1/1. Complete ARM/QEMU suite and focused results below. |
| `rk_hdmi_osd` | Native portable core; genuine ARM64 hardware libraries and standalone executables | Native layout snapshot 1/1. ARM/QEMU layout snapshot 1/1. |
| `rk_hdmi_osd/osd_bridge` | Complete ARM64 bridge linked against the cross-built real message package | Installed node verified ARM aarch64. No registered CTests. |

The three native ROS packages built together through colcon in dependency order. Colcon reports zero registered tests for them; compilation is not counted as a test pass. The portable native total is 35 CTests, plus one additional diagnostic CTest. These tests do not require cameras or flight hardware.

Native full configurations of RK3576 `drone_tracker` and `osd_bridge` were also attempted. Both correctly reject x86_64 targets. Their ARM64 checks were preserved, and full builds used the actual AArch64 compiler and target libraries.

## Complete ARM/QEMU suite

The untouched final-image tracker CTest run passed 54/57 tests in 290.08 seconds. It retained these three failures:

| Test | Default result | Focused evidence |
|---|---|---|
| `diagnostic_tests` | 120-second timeout after `cross-process confirmation` failed | The bundled QEMU 6.2 reports `Unknown syscall 434`. Target headers identify syscall 434 as `pidfd_open`, which the real worker requires to monitor process lifetime. The same existing test passes natively (1/1). No production behavior was weakened to bypass the requirement. |
| `tracking_regression_python` | 52/53 Python cases passed; real middleware identity case failed | Native `ldd` cannot resolve an ARM executable. Running all 53 unchanged cases under real ARM Python, with a wrapper that asks the genuine target loader for ARM dependencies and delegates native ELF to normal `ldd`, passed 53/53. This includes actual middleware loading and rejection of a missing middleware selector. |
| `tracking_regression_source_tests` | 90-second timeout | Process evidence shows the ARM FFmpeg child still encoding the 1,200-frame 1920×1200 FFV1 fixture after 64 seconds. The focused native-FFmpeg retry failed in 37.48 seconds at the memory-limit injection, as detailed below. |

The initial-image run passed 53/57. It additionally failed `osd_presenter_tests` with `new interval missing`. The clean final-image run passed that test in 0.80 seconds. This is retained as a nonpersistent timing observation, not hidden or counted as a persistent product defect.

The separate standalone ARM OSD build included hardware targets. Its portable layout test passed 1/1 under QEMU; this did not exercise hardware. `osd_bridge` registers zero CTests; its successful CTest invocation is not counted as a test pass.

The source-test retry used native `ffmpeg version 4.4.2-0ubuntu0.22.04.1`. Its SHA-256 is `36d94a605d612e4090d1b8aec889d0c0801c6eafb1593c90f5c0dfd2e2966a45`; full version/configuration and both CLI hashes are retained in `final/logs/native-ffmpeg-*`. The test remained ARM64 and retained its 90-second limit. It reached `allocationFailure()` and reported `resource-limited preparation incorrectly succeeded`, followed by `allocation failure did not cleanly reject preparation`. That test sets `RLIMIT_AS`, the process address-space limit, to the size read from `/proc/self/statm` plus 128 MiB. The intended allocation rejection was not observed under QEMU. This is unresolved and needs confirmation on native ARM64; no memory limit, assertion, production code, or acceptance threshold was changed. Intentional malformed-video diagnostics earlier in that test belong to its existing negative test, not an additional reported failure.

## Simulations and their limits

The seven-scenario Python algorithm harness passed its interception/NaN verdict. Its additional velocity-error observation was 6/7 below 3 m/s; `wrong_wguess` ended with 8.76 m/s velocity error. This harness is a Python model, separate from the real C++ node.

All six vision and eight radar scenarios were executed against the compiled C++ ROS node. Each scenario ran in an isolated container communication domain and wrote its own summary/CSV outside source. Thirteen reported `Intercepted: True`; `radar_only` reported `False`, minimum range 2.08 m, no vision frames, and zero NaN commands. All fourteen had zero NaN commands. A simulation process exiting zero does not establish interception success; the counts above come from summary contents.

These simulations used the initial image ID. Final-image native executables and libraries were rebuilt and tested separately. The final image repairs change only cross-build/sysroot setup. No controller or simulator source was changed between these runs. Some scenarios later reopen range after entering the interception threshold, so these results do not establish stable capture or physical-flight safety.

## Evidence and reproduction

All paths below are relative to `/tmp/nickluo-amd64-validation-20261015`.

- `final-native/logs/`: final native colcon build, ELF checks, gimbal simulation, 35 portable CTests, and explicit native architecture rejections.
- `final/logs/`: final full ARM builds, default CTest output, focused native diagnostic result, target-Python regression result, and native fixture-tool retry. `final-python-abi-console.log` records clean ARM message extension names, ELF architecture, and type-support imports.
- `sim/prototype/*_summary.txt` and `*.csv`: actual C++ scenario outcomes.
- `logs/interceptor-algorithm-sim.log`: seven-scenario Python model and its numeric observations. Optional plotting was unavailable because matplotlib is not installed.
- `native.sh`, `final-cross.sh`, `native-diagnostics.sh`, `arm-python-regression.sh`, `retry-regression-source.sh`: external build/test scripts. Mount this directory read-only as `/scripts`, the workspace read-only as `/src`, and a fresh writable output directory as `/out` in the final immutable image. `native.sh` expects an empty output directory. `final-cross.sh` also needs the workspace mounted read-only at `/home/mzh/nickluo` so existing Git metadata resolves.
- `simulations.sh` and `radar-simulations.sh`: adapted external scenario runners. They use ROS Humble and the external install prefix instead of the repository runner's historical Jazzy/workstation paths. The first log contains one invalid `radar_approach` invocation, a harness setup error; the intended eight valid radar scenarios were subsequently run by `radar-simulations.sh`.
- Initial-image build/setup attempts remain in top-level logs. The delivered image includes the proven absolute target Python path, target BLAS/LAPACK symlink repairs, target loader prefix, and ARM Python extension ABI setting.
- `final/logs/final-default-LastTest.log` preserves the complete default CTest record before the focused retry updated CTest's temporary log. `final/logs/native-fixture-source-retry.log` preserves the focused failure. `source-encoding-process.log` records the original encoder bottleneck.

All task-owned `nickluo-amd64-*` validation containers exited and were removed by Docker's `--rm`. A final filtered container check found none. Outputs were retained, and unrelated Docker containers/images were not removed.

## Unverified hardware scope

No RK3576 board connection, NPU model inference, live camera, HDMI output, physical gimbal, Betaflight link, or flight-controller loop was tested. The full ARM builds prove compilation/linking; QEMU tests cover their specific no-device paths. Jetson builds and disabled acados/NMPC variants were not exercised. No production deployment was performed.
