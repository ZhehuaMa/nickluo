# Investigation of unmet validation checks

This work separates emulator defects, test synchronization defects, and genuine estimator/controller limitations. Production algorithms, interfaces, and acceptance thresholds were preserved. The baseline image is `sha256:156c071fcff2385947d0adaea176b29be1b3fb110d2969ebddfda39f2d9d99fd`.

A file descriptor is a process's open handle to a file, socket, or event. `SCM_RIGHTS` is the Unix-socket mechanism used here to transfer those handles between processes. Linux must close received handles that do not fit the recipient's message buffer; `MSG_CTRUNC` reports that the extra message data was truncated. The diagnostic tests require exact descriptor cleanup, not merely successful message delivery.

## Results established independently

| Observation | Root cause and correction | Evidence |
|---|---|---|
| Diagnostic handshake fails under QEMU 6.2 | Its ARM syscall emulation reports syscall 434, `pidfd_open`, as unsupported. QEMU 8.2 executes the real process-instance monitoring operation. No production fallback was added. | Previous `diagnostic-arm-syscalls.log`; pinned Ubuntu QEMU 8.2 package; explicit modern main/worker/inspector routes. |
| Diagnostic descriptor-count check first reports 13 → 11 | Healthy directory scans temporarily open two traversal descriptors. A one-second bounded observation establishes the healthy minimum before malformed requests. Exact post-request equality and each original one-second cleanup bound remain. | Instrumented external logs and `Storage::protect()` implementation. |
| Modern QEMU then reports 11 → 19 at malformed scenario 9 | This is an actual emulator leak. The guest cannot see eight descriptors installed into QEMU when ancillary data is truncated. A small Linux-user syscall patch closes only descriptors not delivered to the guest and preserves the truncation flag. | Native socket-contract probe passes. Unpatched ARM probe leaks at a 15-byte control buffer. Patched genuine ARM probe passes all seven capacities. Three strict ARM diagnostic runs pass. |
| Source memory-limit injection reports success under emulation | QEMU deliberately ignores guest changes to address-space/data/stack limits while returning success. Replace that injection in the target test with a real OpenCV allocation exception. Keep the original OS-limit test separately on native Linux. | Native full source test passes 32.72 s; native original OS-limit check passes 5.83 s. Injection rejects the fourth actual cache allocation, verifies exactly three preceding allocations are freed, and preserves failure/evidence/no-clock assertions. |
| Full source test exceeds 90 s using QEMU's default CPU | The default `max` model exposes features absent on RK3576. Selecting its actual Cortex-A72 model makes the same full ARM test finish within its unchanged 90-second limit. Guest library CPU dispatch is a likely performance mechanism, not an independently established diagnosis. | `root/logs/source-a72-test.log` and `source-a72-status.log`: all source phases complete and exit 0. Native FFmpeg 4.4.2 generates only the fixture; the decoder and tested source remain ARM64. |
| Intermittent OSD interval/log assertions | A fake backend announces output before the presenter finishes bookkeeping. Shutdown or another control event can race that bookkeeping. The mission test also jumped its clock by 3 s across a 1,000 ms stale limit, allowing a legitimate stale transition. | Observe public completed-frame counters before associated control/shutdown. Keep the original following-frame 200 ms bound, share existing two-second deadlines from before submit, retain the three-second mission duration and 1,000 ms stale limit, and send completed heartbeat frames. Corrected strict ARM test passes 20/20 repeats. |

QEMU's deliberate resource-limit handling is visible in the [official v8.2.2 syscall source](https://github.com/qemu/qemu/blob/v8.2.2/linux-user/syscall.c). The package version and checksum were checked against [Ubuntu's package record](https://packages.ubuntu.com/noble/amd64/qemu-user-static/download). These sources support emulator behavior; native and actual ARM test outcomes above come from retained local logs.

## Descriptor correctness proof

The external socket probe sends sixteen real descriptors. It tests control-buffer capacities 0, 15, 16, 20, 24, 32, and 80 bytes. After every receive, the probe closes guest-visible descriptors and requires the exact pre-request count. It also requires the Linux truncation flag when sixteen descriptors cannot fit.

- Native Linux: all seven cases pass.
- Unpatched QEMU 8.2: the 15-byte case increases descriptors from 7 to 10 and fails.
- Patched QEMU: all seven genuine ARM cases pass with counts unchanged.
- Deliberately retaining a received descriptor makes the patched probe fail with exit 1.
- Three complete genuine ARM diagnostic test runs pass. They retain all twelve malformed cases twice, healthy-session continuation, inspector rejection checks, reconnect, and exact cleanup requirements.
- A separately compiled ARM worker that deliberately retains a rejected descriptor makes that same diagnostic test fail with `rejected descriptor leak` and exit 1.

No descriptor categories were filtered from the check. No host `binfmt_misc` registration was changed. Native launcher scripts explicitly select the investigated QEMU for the ARM test, worker, and inspector, because the host's registered interpreter otherwise remains QEMU 6.2.

## Emulator provenance

- Official source archive: `https://archive.ubuntu.com/ubuntu/pool/main/q/qemu/qemu_8.2.2+ds.orig.tar.xz`.
- Source size: 39,678,456 bytes.
- Official source SHA-256: `5e692dd20950a819ff6550cd3974e00319a93f47c0a15f808c58df1a9e4d2478`.
- Verified archive: `/tmp/nickluo-qemu-source-fetch-20261015/qemu_8.2.2+ds.orig.tar.xz`.
- Patch: `qemu-close-truncated-rights.patch`, SHA-256 `a2838c7f61ec52a29a59e5622a2d69a488a7f5a58d761d0276188702a42ad4e9`.
- Original `linux-user/syscall.c` SHA-256: `0538eba5231df22295cf56dddc376b5f2ed7ab02ca4eb05687e036a7cb1e9a62`.
- Patched source-file SHA-256: `23fff7afa16410aa977880073329e81dd519243cc7753967996c341114a5f26e`.
- Independently built static binary: `qemu-build/qemu-aarch64`, SHA-256 `cc2fe25c77da3a8cbf0d2792dd426f2de1283d3872fd43ae11c1af0ea5ce54ea`.
- Build helpers: Meson 1.2.3, Ninja wheel 1.11.1.1, tomli 2.0.1, installed only into this investigation's external helper directory.
- Configure used only `aarch64-linux-user`, static linking, and disabled system/tools/docs/guest-agent/plugins/capstone/slirp/fdt/gcrypt/nettle/download/install-blobs. The Debian-source archive omits firmware, and the user-mode emulator requires none.
- Static linking emits glibc user-database/NSS warnings. The build host and validation image both use Ubuntu 22.04; the actual binary was executed inside the image during all positive and negative controls.

Full configure/build/provenance output and `patched-qemu-proof.sh` are beside this report. The image agent owns the reproducible repository preparation script, tracked patch, final image build, and full ordinary-entrypoint validation. A separately built recipe binary can have a different hash because build paths are embedded; its own provenance and affected checks must be retained.

## Remaining simulation limitations

The original native C++ `radar_only` scenario remains a failure: minimum distance 2.0807 m, zero vision frames, zero NaN commands. Its source says it characterizes radar-only approach without a visual terminal phase. It uses 10 Hz radar with at least 2 m position standard deviation per axis, additional range-dependent noise, and 50–100 ms delay. The unchanged uniform interception criterion is 1 m.

An external diagnostic input variant sets only radar position/range/velocity noise to zero. It preserves the same C++ controller, geometry, duration, detector suppression, latency, and 1 m criterion. It intercepts at 9.91 s with minimum distance 0.69 m. This supports measurement precision as a major contributor; it is not a replacement pass for the original noisy case. A radar-only engagement also never arms the vision-terminal capture state in the current node. No controller gains, capture gates, or thresholds were tuned.

The Python `wrong_wguess` model observation also remains real. At the original width guess 1.5 m and true width 0.8 m, final velocity error is 8.756 m/s. Matching only the diagnostic initial width guess to 0.8 m reduces error to 0.221 m/s. Removing measurement noise while keeping the wrong width still yields 8.980 m/s. The model stops after interception at 3.58 s in the original case. These controlled inputs support a persistent size/range/velocity scale initialization error rather than random noise. The model's interception criterion passes, but its auxiliary velocity-convergence observation does not.

All variants and their hashes are recorded in `root/logs/simulation-analysis.json` and `root/radar-diagnostic/input-variant.json`. Neither experiment changes the production controller or the original scenario verdict.

## Changed repository scope and verification gaps

Only these C++ test files were changed by this investigation: `src/diagnostics/tests/diagnostic_test.cpp`, `src/osd/tests/presenter_test.cpp`, and `tests/tracking_regression/tests/realtime_source_test.cpp`. The shared workflow agent owns CMake registrations and emulator preparation/image changes. Existing OpenSpec planning explicitly records the strict test corrections and retained native OS-limit coverage.

`git diff --check` passes for these files. No configured clang-tidy or clangd executable was available, so those static checks were not performed. No RK3576, camera, HDMI, NPU, physical gimbal, or flight-controller deployment was accessed. These are native Linux and emulated ARM checks, not hardware acceptance.

## Independent final-image verification

The final frozen image is `sha256:025d0aa88983b0646a1b7a4d25cbc13dc96a6206e0148475bbfd5fad15057421`, `linux/amd64`. Its own installed QEMU 8.2.2 binary at `/opt/qemu-arm64/bin/qemu-aarch64` has SHA-256 `da420e56e1d0199413c081067d647936af23123b9eefdb859b1facb92cccaddc`. The independent check used that binary, without mounting an emulator replacement. Its manifest confirms the same pinned source and patch hashes above.

`final-image-proof.sh` passed all seven real ARM descriptor-buffer cases and three complete strict ARM diagnostic repetitions. The intentionally leaking probe and intentionally leaking ARM worker both failed with the required exit 1; the worker test reported `rejected descriptor leak`. This confirms that the delivered image repairs the emulator defect while the diagnostic test still detects an actual persistent leak. Evidence is under `root/final-image-logs/`, including the copied image manifest, executable hashes, architecture records, and each positive/negative log. The final whole-workflow run is owned separately by the image/workflow agent.

The corrected OSD repetitions are recorded in `root/logs/osd-deadlines-status.log`: 20/20 pass with the following-frame deadline established before submit and limited to 200 ms. Earlier prototype repetitions with a two-second following-frame allowance are superseded and are not the evidence for this result.

The stale task-owned radar ROS launch was identified by container name, exact image, command, and source/output mounts before stopping it. No `nickluo-investigation-*` container remained after the final checks. All outputs were retained.

## Host supervision versus target-library Python checks

The fresh ordinary workflow then exposed an additional upstream QEMU 8.2 limitation: `PR_SET_CHILD_SUBREAPER` and `PR_GET_CHILD_SUBREAPER` are explicitly unimplemented in its Linux-user `do_prctl` switch and return `EINVAL`. Three Python controller tests correctly failed before supervision began. This does not invalidate the ARM diagnostic descriptor proof; it is a separate syscall requirement in host orchestration. No emulator subreaper patch or production supervisor fallback was added.

A child subreaper is a supervisor that adopts descendant processes when their original parent exits. These tests use the real Linux operation to verify that failed or interrupted runs leave no orphan process behind.

Inspection of all 53 regression Python tests found one case that loads actual ARM ROS libraries: `DynamicIdentityTests.test_actual_ros_middleware_and_missing_selector`. The other 52 exercise portable data rules, native Linux process supervision, mocks, or fresh native `cc` loader fixtures. The implemented coordinator runs those 52 with native Python and the middleware case with genuine ARM Python under the selected QEMU. It retains every discovered test identifier and the existing aggregate 30-second CTest limit. This is a host/target split, not a claim that QEMU supports ARM Python subreaper operations. The middleware probe's `sys.executable` children explicitly select the chosen QEMU rather than rely on host binfmt; the independent trace verifies both children.

The independent final coordinator check is complete on image `025d0aa8…`: all 53 discovered unique test IDs executed exactly once, with 52 native and one genuine ARM case. There were no skips, duplicates, or missing tests. The whole positive invocation took 8.82 seconds under the original aggregate 30-second bound. A copied native test with a deliberate assertion failure returned 1. A copied ARM middleware test with a deliberate assertion failure also returned 1 after the native partition passed. Repository test files were not changed for these negative controls.

`python-controls/inventory-verification.json` records the parsed discovery/execution comparison. `root/final-image-logs/python-positive.log`, `python-native-failure.log`, `python-target-failure.log`, and `python-control-status.log` retain outcomes. `python-qemu-invocations.log` proves the ARM main interpreter, real target loader, and both ctypes child probes explicitly reenter the image-installed selected QEMU. The complete reproducible control script and isolated test copies are under `python-controls/`.

The first full ordinary final-image run retained its genuine 56/57 result: only the incorrectly ARM-routed host Python partition failed. The full source test passed in 71.37 seconds within its unchanged 90-second bound. That failed run is preserved separately from the corrected run.

The actual corrected `./build.sh --test` run on image `025d...` passed all 57 CTests in 156.28 seconds. It also passed the mandatory native OS memory-limit check, 1/1. The Python coordinator passed all 53 cases in 8.74 seconds; the complete ARM source test passed in 71.26 seconds, retaining its original 90-second limit. No test deadlines, assertions, production algorithms, or acceptance thresholds were loosened. Full commands, output identities, fallback coverage, and strict OpenSpec validation are recorded in `/home/mzh/nickluo/drone_tracker/openspec/changes/archive/2026-10-15-default-amd64-rk3576-build-workflow/verification.md`.
