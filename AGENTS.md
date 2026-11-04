# nickluo workspace instructions

## Repository boundaries

- Read [AGENTS.local.md](AGENTS.local.md) before local work. Read the selected repository's `AGENTS.md` and, when present, `AGENTS.local.md` before modifying it.
- Treat each submodule as an independent repository with its own history. Use [.gitmodules](.gitmodules) for repository paths and remotes.
- Keep root-repository changes in root-owned files and recorded submodule commits. Implement code, documentation, and configuration changes inside the repository that owns them.
- Split cross-repository work into separately reviewable changes. Validate and report each repository's changes independently.
- Do not commit, push, or update the root's recorded submodule commits without user authorization. Inspect uncommitted work and local branches before changing submodule versions; never force an update over local work.

## Repository responsibilities

| Repository | Responsibility |
|---|---|
| `apm_bridge/` | Connect the flight controller to ROS, relay telemetry and control commands, and drive the gimbal. |
| `drone_tracker/` | Capture images, detect and track targets, and publish observations and tracking state. |
| `drone_interceptor/` | Estimate target motion and calculate interception commands. |
| `quadrotor_msgs/` | Define shared ROS messages and generate their language interfaces. |
| `rk_hdmi_osd/` | Provide the RK3576 camera and on-screen display stack. |

For camera or display integration, check both `rk_hdmi_osd` and `drone_tracker`. Distinguish reference implementations from integrated code; verification in one repository does not establish acceptance in another.

## Sources of truth and change workflow

- Obtain implementation details from source, message definitions, launch configuration, and build scripts. Keep exact interface values, dependency conditions, and build commands in their owning repository's maintained documentation.
- Before changing behavior, architecture, or interfaces, inspect the owning repository's OpenSpec configuration and active changes. OpenSpec stores change plans and requirements alongside code. Continue a matching change and follow that repository's planning and implementation workflow.
- Follow user-wide language rules and repository-specific artifact requirements. Avoid duplicating language policy here.
- Keep migration history, environment snapshots, and validation results in dated reports. A report establishes only the revision, environment, and checks it records.

## Cross-repository interfaces

- For a ROS interface change, check every affected publisher, subscriber, and launch remapping, which can change the topic name used at runtime.
- Verify message types, fields, units, coordinate frames, quaternion ordering, timestamps, trigger meanings, and state codes at both ends. Check QoS, the reliability and queue settings that determine whether endpoints can communicate.
- Build shared messages from the actual `quadrotor_msgs` source for the consumers' target architecture. Do not substitute copied generated headers or runtime-only files for a development dependency.
- After changing a shared message, rebuild affected consumers and check the corresponding communication paths. For control units or coordinate conversions, verify the complete path from algorithm output to flight-controller conversion.

## Build and acceptance

- Follow each repository's build and test guides. Use [docs/build-and-test-workflow.md](docs/build-and-test-workflow.md) as a workspace navigation aid; confirm commands against current repository scripts.
- Keep generated build, install, and log outputs outside source directories. Keep outputs for different target architectures and build environments separate.
- Report compilation, tests without devices, simulation, and physical-device acceptance separately. Record the actual platform and inputs used, and identify checks blocked by missing hardware, SDKs, models, or recordings.
- Do not treat host or emulated success as evidence of camera, accelerator, display, timing, gimbal, firmware-link, or flight behavior on a target device.
