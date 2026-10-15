# Submodule compiler-image audit — 2026-10-15

A newly started subagent audited all five submodules and updated their active build instructions.
Every submodule can compile using the x86_64 development image. None inherently requires an
ARM64 container for compilation. Complete RK3576 tracker and display artifacts still require
ARM64 output and genuine target libraries.

The audited image is `drone-tracker-rk3576:humble-amd64-rknn2.3.2`, immutable ID
`sha256:025d0aa88983b0646a1b7a4d25cbc13dc96a6206e0148475bbfd5fad15057421`.
A cross-compiler runs on x86_64 but produces ARM64 programs. ARM64 programs need an ARM64
system or QEMU, which emulates ARM CPU instructions. Neither compilation nor CPU emulation
establishes correct operation of a physical peripheral.

## Results by repository

| Repository | Using the x86_64 image | Test scope and boundaries | Revised guide |
| --- | --- | --- | --- |
| `apm_bridge` | Native x86_64 or ARM64 cross-built bridge, gimbal harness and real message libraries | Virtual-serial protocol simulations support both architectures. Physical serial links, firmware and gimbals require device checks. | [Build and test](../apm_bridge/docs/build-and-test.md) |
| `drone_tracker` | Complete RK3576 ARM64 cross-build; selected portable tests can run natively | Full target suite uses explicit ARM emulation and architecture-matched helpers. Camera, NPU and HDMI paths require the board. Full native x86_64 RK3576 output is not supported. | [Compiler image and workflow](../drone_tracker/dockerfiles/README.amd64.md) |
| `drone_interceptor` | Native x86_64 simulation node or ARM64 cross-built enabled IMM-UKF node | Native ROS simulation works, with existing failed cases retained. ARM emulation has an unresolved ROS communication failure. Disabled NMPC/acados variants are outside this evidence. | [Docker builds](../drone_interceptor/docs/DOCKER_BUILD.md) |
| `quadrotor_msgs` | Native or ARM64 generated C/C++ libraries and Python extensions | Serialization and real consumer checks are supported. Python extensions require a matching interpreter architecture. No registered CTests exist. | [Message generation](../quadrotor_msgs/README.md) |
| `rk_hdmi_osd` | Native portable core with `OSD_WITH_HW=OFF`; complete ARM64 cross-built hardware stack and `osd_bridge` | Portable layout tests run natively or under ARM emulation. Full native x86_64 bridge output is correctly rejected. Camera/display checks require hardware. | [Build architectures](../rk_hdmi_osd/BUILDING.md) |

## Fresh verification during this audit

Source mounts were read-only. Outputs were fresh and outside source. Containers had no network.

- Clean ARM64 generation of the real `quadrotor_msgs` source succeeded. Generated shared
  message libraries were checked as ARM64 executable files.
- Both `apm_bridge` executables and the enabled interceptor IMM-UKF executable cross-compiled.
  The genuine ARM dynamic loader resolved every dependency, including the fresh message libraries.
- The unchanged gimbal protocol simulator passed all three models against the cross-built ARM
  harness under explicitly selected QEMU: C200T 483 Hz, C40D 480 Hz and C20S 488 Hz.
- Clean native message generation and a real `ControlCommand` serialization/deserialization
  round trip passed. This checks message encoding, not a publisher/subscriber connection.
- Clean native portable OSD build/test passed 1/1 `layout_snapshot` CTest. A clean complete ARM
  OSD stack and `osd_bridge` build succeeded; its ARM/QEMU layout test passed 1/1.
- Native x86_64 `osd_bridge` configuration was correctly rejected. Existing architecture
  guards remain intact. Only their comments or error explanations changed.

The ROS message, APM and enabled interceptor packages register no CTests. Zero discovered
tests are not reported as a regression pass. The prior tracker result on this same image,
57/57 CTests plus the mandatory native OS memory-limit check, was retained rather than rerun
for documentation changes. See the [test investigation](native-default-test-investigation-20261015.md).

The original noisy `radar_only` scenario, Python wrong-width auxiliary velocity criterion,
and emulated ARM ROS communication remain unresolved. No failed result was recast as passing.
No algorithms, assertions, thresholds or test deadlines were changed in this audit.

## Genuine ARM requirements retained

- Tracker `deployment/package.sh` collects dependencies in its hard-coded ARM64 image.
  It can consume cross-built ARM64 installations, but requires the chosen backend's absolute
  `--install-root` explicitly. Its legacy default was not redirected.
- Interceptor `make_bundle.sh` requires `uname -m=aarch64` and native `ldd` dependency inspection.
  Cross-compilation has not migrated that separate packaging procedure. The new cross-built
  output was not verified through complete bundle assembly and installation.
- The x86_64 image builder imports its target SDK from the preserved ARM64 image.
  `dockerfiles/build-image.sh` intentionally builds this fallback and SDK-source image.

Explicit fallback recipes, board runtime requirements and dated historical evidence were
preserved. Neither image supplies physical RK3576 devices or kernel drivers on this host.
No board, NPU, camera, HDMI, physical gimbal, flight controller or production deployment was
exercised. Jetson and disabled NMPC/acados paths remain unverified.

## Documentation changes and evidence

The subagent updated 33 submodule-owned files: current build guides, `AGENTS.md` and local
instructions, platform and deployment descriptions, benchmark recipes, Copilot/Claude
instructions, and CMake diagnostic text. Final review also corrected the tracker build-help
packaging example to include its output argument and an absolute installation path. Existing
unrelated changes were preserved. No commits, pushes or parent submodule pointers were made.

Raw evidence is under `/tmp/nickluo-submodule-image-audit-20261015/`:

- `AUDIT_REPORT.md`, `changed-files.json`, `final-file-sha256.json` and `before/` capture the
  subagent's report, original 33-file manifest, hashes and pre-audit snapshots. These precede
  the small final review corrections described above.
- `cross-ros.sh` and `cross-console.log` reproduce clean ARM message/consumer builds and
  dependency inspection.
- `check-doc-recipes.sh`, `finish-doc-recipes.sh`, their console logs and `logs/` record native
  messages, portable/full OSD builds, architecture guards and gimbal simulations.

One helper initially stopped after successful OSD builds because `rg` was absent inside the
image. The continuation used available `grep` and ran the remaining checks. This was a helper
dependency issue. Both logs remain available. Preserve raw evidence before cleaning `/tmp`.

Final whitespace checks passed in the root and all five submodules. Syntax checks passed
for the scripts whose comments or help text changed. Use the [workspace workflow](build-and-test-workflow.md)
for default and fallback entry points.
