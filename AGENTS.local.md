# nickluo local development environment

## Host and build boundaries

- The remote development checkout is `/home/mzh/nickluo`. This x86_64 host is not an RK3576 board.
- Use Docker for the supported local build workflows. Read the selected repository's local instructions and build guide for current images, toolchains, and commands.
- Check the current tools, images, devices, and artifact paths before relying on them. Query versions and identities when needed; do not infer availability from an old report.
- Distinguish container architecture from output architecture. A cross-compiler runs on the host architecture and produces programs for another architecture. ARM64 execution on this host requires emulation; hardware acceptance requires the actual target.
- Keep native and target libraries, generated language interfaces, and build caches separate. Use the interpreter required by the selected ROS environment for its generators and Python interfaces; follow user-wide Python policy for other work.

## Device access

- Resolve device access from current SSH or other connection configuration. Do not infer access or deployment state from historical aliases and paths.
- Before device work, confirm the intended board, deployed revision, services, and relevant hardware topology. A development board's results do not establish production-board acceptance.
- Historical board snapshots and absolute symbolic links do not establish a deployment on this host. Preserve dated evidence in its original context.

## Outputs and evidence

- Treat temporary directories and build caches as disposable. A documented path or cache backup does not guarantee that evidence still exists.
- Before cleanup, copy required acceptance evidence to an explicitly retained location outside disposable caches. Include the source revision, commands, environment identity, model and input identities when relevant, results, and actual exit statuses.
- Store migration records, image inventories, version details, and investigation results in dated reports. Keep this file limited to local constraints and working rules.
