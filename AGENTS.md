# nickluo 多仓库工作约定

## 修改范围

- 根仓库 `nickluo` 的修改只应发生在根仓库自身的文件或记录中，不得修改任何子仓库的文件。根仓库记录的子模块提交版本属于根仓库修改；子模块目录内的代码、文档和配置属于对应子仓库修改。
- 一项需求同时涉及根仓库和子仓库时，拆分为各仓库独立的修改，分别实现、验证和交付。例如：修改 `apm_bridge/` 的代码时，在该子仓库中独立完成；更新根仓库文档或其记录的 `apm_bridge` 提交版本时，在根仓库中另行完成。

## 目录与职责

本目录是 Git 根仓库，通过 5 个 submodule（子模块，即保留独立历史的子仓库）汇集同一 GitHub 作者的项目。根仓库通过 `.gitmodules` 记录各子模块的路径和远端地址，并记录各子模块使用的提交版本。进入对应子目录执行代码相关的 Git 操作，按仓库分别检查和交付变更。修改前读取该仓库的 `AGENTS.md`，以及存在时的 `AGENTS.local.md`（本地工具和环境约定）。

- 首次获取完整源码使用 `git clone --recurse-submodules <根仓库地址>`；已克隆根仓库时使用 `git submodule update --init --recursive` 初始化子模块。
- 子仓库修改独立完成并提交后，若需求包含更新根仓库记录的子模块提交版本，再在根仓库中暂存该子目录。例如：在 `apm_bridge/` 提交后，返回根目录执行 `git add apm_bridge`。两部分修改分别交付，提交和推送遵循用户当次授权。
- 切换根仓库版本后，需要匹配其记录的子模块版本时执行 `git submodule update --init --recursive`。执行前检查子模块的未提交修改及本地分支，不使用 `--force` 覆盖本地工作。

| 子目录 | 职责 |
|---|---|
| `apm_bridge/` | 通过 custom-link 自定义二进制协议连接 Betaflight 飞控；发布遥测和触发消息，接收控制指令，并驱动云台。 |
| `drone_tracker/` | 采集相机或视频画面，检测和跟踪目标，发布目标框、图像和跟踪状态；包含 Jetson 与 RK3576 平台实现。 |
| `drone_interceptor/` | 根据视觉和机体运动信息估计目标状态，计算拦截控制指令。 |
| `quadrotor_msgs/` | 定义共享 ROS 2 消息，包含 `ControlCommand` 和 `LowLevelFeedback`；构建时生成各语言的接口代码。 |
| `rk_hdmi_osd/` | RK3576 相机与 OSD（叠加在画面上的文字和图形）显示栈；内含独立 ROS 2 包 `osd_bridge/`。 |

涉及相机或 HDMI 显示时，核对 `rk_hdmi_osd/` 与 `drone_tracker/src/platform/rk3576/`、`drone_tracker/src/osd/` 的对应实现。区分参考实现与已集成代码，不把一个仓库的验证结果当作另一个仓库的验收结果。

## 跨仓库通信

`apm_bridge`、`drone_tracker`、`drone_interceptor` 通过 ROS 2 topic（具名的消息发布与订阅通道）协作。下表记录核心链路；完整接口以各节点源码和 launch 启动配置为准。

| 话题 | 消息类型 | 发布方 → 订阅方 |
|---|---|---|
| `/mavros/imu/data` | `sensor_msgs/msg/Imu` | `apm_bridge` → `drone_interceptor` |
| `/fpv/gimbal` | `sensor_msgs/msg/Imu` | `apm_bridge` → `drone_tracker`、`drone_interceptor` |
| `/fpv/tracker_trigger` | `std_msgs/msg/Header` | `apm_bridge` → `drone_tracker`、`drone_interceptor` |
| `/fpv/target_bbox` | `vision_msgs/msg/Detection2D` | `drone_tracker` → `drone_interceptor` |
| `/fpv/tracker_state` | `std_msgs/msg/Int8` | `drone_tracker` → `apm_bridge` |
| `/fpv/control_command` | `quadrotor_msgs/msg/ControlCommand` | `drone_interceptor` → `apm_bridge` |
| `/fpv/mission_state` | `std_msgs/msg/Bool` | `drone_interceptor` → `apm_bridge` |

- 修改话题名称、类型、字段、单位、时间戳或状态编码时，同时检查发布方、订阅方和 launch 中的 remapping（话题名称重映射）。`apm_bridge` 源码中的 `~/control_command` 由 launch 映射到 `/fpv/control_command`，直接运行可执行文件时不能假定该映射已生效。
- 保持触发消息的 `Header.frame_id` 取值 `ON`、`OFF`、`MISSION`，并核对各消费者对它们的处理。保持跟踪状态编码：`Detecting=0`、`TargetFound=1`、`Tracking=2`、`TargetLost=3`。
- 检查 QoS（消息传输的可靠性和队列策略）是否兼容。例如桥接节点以 `best_effort` 发布 IMU，订阅端不能要求 `reliable`，否则无法建立这条通信链路。
- 保持 `ControlCommand.bodyrates` 的单位为 rad/s，`collective_thrust` 为质量归一化推力，单位 m/s²。归一化油门走独立的 raw 接口，不能混用。核对消息注释及节点中的坐标系转换，不直接互换 ROS 四元数 `[x,y,z,w]` 与算法内部 `[w,x,y,z]`。

## 依赖与构建边界

- 将 `quadrotor_msgs/` 作为三个节点仓库的共享源码依赖纳入 ROS 2 工作空间。先构建它，或与消费者一起交给 `colcon` 按依赖顺序构建；不要复制生成的消息头文件或用手写替代消息包。
- 区分系统集成依赖与具体构建选项：`apm_bridge`、`drone_interceptor` 直接要求 `quadrotor_msgs`。当前 `drone_tracker` 仅在 `DRONE_TRACKER_BUILD_OSD=ON` 且 `DRONE_TRACKER_OSD_LOW_LEVEL_FEEDBACK=ON` 时要求它，`package.xml` 尚未声明该可选依赖。启用此选项时显式准备消息包，不能只依赖 `--packages-up-to drone_tracker` 自动发现它。
- 共享消息包不在标准 rosdep 索引中。`rosdep` 用于解析系统依赖，不能替代此处的源码构建；发布包中的运行时文件也不能替代开发安装文件。
- 使用 ROS 2 Humble 与匹配平台的 SDK（硬件开发库和工具）。主要 C++ 目标使用 C++17；修改 C++ 时读取并遵守 `~/.codex/language_norms/cpp_norms.md`。
- `apm_bridge` 的运行依赖是支持 custom-link 的 Betaflight 固件及串口或 TCP 链路。源码使用 `/mavros/*` 话题名，不代表必须另外启动 MAVROS 节点。
- `drone_tracker` 构建默认平台为 `jetson`；RK3576 构建显式指定 `DRONE_TRACKER_PLATFORM=rk3576`。平台 SDK、模型和本地构建入口遵循该仓库约定，不用主机编译替代硬件验收。
- 当前 `drone_interceptor/CMakeLists.txt` 只启用 `interceptor_node_relative_imm_ukf`，无需 acados。启用 NMPC（根据模型预测未来运动的控制器）变体时，再准备 acados、blasfeo、hpipm，并先运行对应 `prototype/drone_model_*.py` 生成 C 求解器。不要将当前构建通过解释为这些变体也已验证。

## 修改与验证

- 涉及功能、架构或接口变更时，检查相关仓库的 `openspec/changes/` 和 `openspec/config.yaml`。OpenSpec 是随代码维护的变更提案、设计、规格和任务；继续匹配的现有变更，并遵守所在仓库规定的实施顺序。文档使用简体中文，说明术语和可观察结果。
- 按仓库现有入口验证：`apm_bridge/AGENTS.md` 的 Docker 构建、`drone_interceptor/docs/DOCKER_BUILD.md` 的依赖构建，以及 `drone_tracker/build.sh` 和本地约定。将容器生成的 `build/`、`install/`、`log/` 放在源码目录之外。
- 修改 `quadrotor_msgs/msg/*.msg` 后，重新构建使用这些消息的消费者，并检查对应话题链路。修改控制单位或坐标系时，验证从算法输出到桥接转换的完整路径。
- `drone_tracker` 测试构建需开启 `DRONE_TRACKER_BUILD_TESTS=ON`。`rk_hdmi_osd` 的可移植显示核心可用 `cmake -S rk_hdmi_osd -B /tmp/nickluo-osd-build -DOSD_WITH_HW=OFF` 构建，再执行 `cmake --build /tmp/nickluo-osd-build` 与 `ctest --test-dir /tmp/nickluo-osd-build --output-on-failure`；该检查不覆盖相机、HDMI 或 ROS 桥接硬件路径。
- 交付时区分编译、无设备测试、仿真与实机验证。记录实际验证的平台和范围，并列明缺少硬件、SDK、模型或录像导致的验证缺口。
