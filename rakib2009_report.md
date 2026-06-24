# Report: `~/Desktop/rakib2009`

**Generated:** 2026-06-24
**Type:** ROS 2 colcon workspace — Gazebo simulation of Universal Robots manipulators
**Purpose:** Provides a working Gazebo (gz) simulation of UR robots (UR3/UR5/UR10/e-series/UR7e/UR12e/UR16e/UR8long/UR15/UR18/UR20/UR30) for use with `ros2_control` and MoveIt, consumed by the `rakib_RL` RL training pipeline.

---

## 1. Workspace Overview

| Property | Value |
|----------|-------|
| Root path | `~/Desktop/rakib2009` |
| Build type | `colcon` (ament_cmake + python) |
| Source | `src/ur_simulation_gz` (a checked-out upstream Universal Robots repo) |
| Build dir | `build/ur_simulation_gz/` |
| Install dir | `install/ur_simulation_gz/` |
| Log dir | `log/build_2026-06-17_15-44-58/` (latest symlinked) |
| ROS 2 distro (detected at build time) | `jazzy` (per build env) |
| Last build outcome | **Successful** (`rc: 0`, completed in ~1.6 s for the `ur_simulation_gz` package) |
| Underlying source package | Universal_Robots_ROS2_GZ_Simulation `ros2` branch, version 2.5.0 (released 2025-10-13) |
| License | BSD-3-Clause |
| Maintainers | Felix Exner (Universal Robots), Rune Søe-Knudsen, Universal Robots A/S |
| Build tool | CMake 3.5+; ament_cmake |
| Build flags used | `--symlink-install` (configurable via `AMENT_CMAKE_SYMLINK_INSTALL=1`) |

---

## 2. Source Tree (`src/ur_simulation_gz/`)

```
src/ur_simulation_gz/
├── CMakeLists.txt                  (top-level aggregator, NOT used in ROS 2 build)
├── LICENSE                         (BSD-3-Clause)
├── README.md
├── CHANGELOG.rst
├── ci_status.md
├── CONTRIBUTING.md
├── .clang-format
├── .pre-commit-config.yaml
├── .github/                        (dependabot.yml, mergify.yml, workflows/)
├── doc_requirements.txt
├── ur_simulation_gz.humble.repos
├── ur_simulation_gz.jazzy.repos
├── ur_simulation_gz.kilted.repos
├── ur_simulation_gz.lyrical.repos
├── ur_simulation_gz.rolling.repos
└── ur_simulation_gz/               ← actual ROS 2 ament package
    ├── CMakeLists.txt              (find_package ament_cmake; installs config/launch/urdf)
    ├── package.xml                 (build_type = ament_cmake, version 2.5.0)
    ├── CHANGELOG.rst
    ├── config/
    │   └── ur_controllers.yaml
    ├── launch/
    │   ├── ur_sim_control.launch.py
    │   └── ur_sim_moveit.launch.py
    ├── urdf/
    │   ├── ur_gz.urdf.xacro
    │   └── ur_gz.ros2_control.xacro
    ├── test/
    │   ├── test_common.py
    │   ├── test_description.py     (parametrized xacro→urdf→check_urdf over 14 ur_types)
    │   └── test_gz.py              (FollowJointTrajectory integration test, ~60 s)
    └── doc/
        ├── conf.py
        ├── index.rst
        ├── installation.rst
        ├── usage.rst
        ├── migration_notes.rst
        ├── migration/{jazzy,lyrical}.rst
        └── resources/{test_world.sdf, ur_controllers_test.yaml, *.png}
```

### 2.1 Package metadata (`package.xml`)
- **Name:** `ur_simulation_gz`
- **Version:** 2.5.0
- **Build tool:** `ament_cmake`
- **Runtime dependencies (`<exec_depend>`):** `forward_command_controller`, `gz_ros2_control`, `joint_state_publisher`, `joint_state_broadcaster`, `joint_trajectory_controller`, `launch`, `launch_ros`, `ros_gz_bridge`, `ros_gz_sim`, `rviz2`, `ur_controllers`, `ur_description`, `ur_moveit_config`, `urdf`, `xacro`.
- **Test deps:** `ament_cmake_pytest`, `launch_testing_ament_cmake`, `launch_testing_ros`, `liburdfdom-tools`, `xacro`.

### 2.2 `CMakeLists.txt`
- Defines option `UR_SIM_INTEGRATION_TESTS` (default `OFF`).
- Installs `config/`, `launch/`, `urdf/` directories into `share/ur_simulation_gz/`.
- If `BUILD_TESTING` is on: registers `test/test_description.py` (always) and `test/test_gz.py` (only if `UR_SIM_INTEGRATION_TESTS=ON`, 180 s timeout).

---

## 3. Launch Files

### 3.1 `ur_sim_control.launch.py` (≈ 318 lines)
The primary entry point. Brings up:
- `robot_state_publisher`
- `rviz2` (gated by `launch_rviz`)
- `joint_state_broadcaster` spawner
- `joint_trajectory_controller` (default `initial_joint_controller`) — stopped vs. started gated by `activate_joint_controller`
- Gazebo (`ros_gz_sim/gz_sim.launch.py`) — GUI or headless
- `ros_gz_sim/create` — spawns UR robot entity
- `ros_gz_bridge/parameter_bridge` — bridges `/clock@rosgraph_msgs/msg/Clock[gz.msgs.Clock`

**Key launch arguments:**

| Arg | Default | Purpose |
|-----|---------|---------|
| `ur_type` | `ur5e` | Selects UR model (choices: `ur3, ur5, ur10, ur3e, ur5e, ur7e, ur10e, ur12e, ur16e, ur8long, ur15, ur18, ur20, ur30`) |
| `safety_limits` | `true` | Enable UR safety controller |
| `safety_pos_margin` | `0.15` | Joint safety position margin |
| `safety_k_position` | `20` | Spring constant for safety controller |
| `controllers_file` | `ur_controllers.yaml` | Path to controller YAML |
| `tf_prefix` | `""` | Useful for multi-robot setups |
| `activate_joint_controller` | `true` | Auto-start the trajectory controller |
| `initial_joint_controller` | `joint_trajectory_controller` | First controller to spawn |
| `description_file` | `urdf/ur_gz.urdf.xacro` | URDF/XACRO description |
| `launch_rviz` | `true` | Launch RViz? |
| `rviz_config_file` | `ur_description/rviz/view_robot.rviz` | RViz config |
| `gazebo_gui` | `true` | Gazebo with GUI? |
| `world_file` | `empty.sdf` | World file (path or Gazebo world library name) |

The robot description is built at launch time by running `xacro` with the `simulation_controllers` parameter pointing to the YAML.

### 3.2 `ur_sim_moveit.launch.py` (≈ 156 lines)
- Wraps `ur_sim_control.launch.py` (with `launch_rviz=false`) and additionally launches the `ur_moveit_config/ur_moveit.launch.py` MoveIt node.
- Adds arg `launch_servo` (default `false`) and `moveit_launch_file` (default `ur_moveit.launch.py`).
- Same `ur_type` choices; `tf_prefix` is **not** propagated here.

### 3.3 `ur_controllers.yaml` (76 lines)
The controller manager configuration. Defines:

- `controller_manager.update_rate: 500` Hz
- Controllers (in this order):
  - `joint_state_broadcaster` (`joint_state_broadcaster/JointStateBroadcaster`)
  - `io_and_status_controller` (`ur_controllers/GPIOController`)
  - `speed_scaling_state_broadcaster` (`ur_controllers/SpeedScalingStateBroadcaster`)
  - `joint_trajectory_controller` (`joint_trajectory_controller/JointTrajectoryController`) — primary motion controller
  - `forward_velocity_controller`
  - `forward_position_controller`

**`joint_trajectory_controller` config:**
- Joints: all 6 UR joints.
- Command interfaces: `position` only.
- State interfaces: `position`, `velocity`.
- `state_publish_rate: 100.0` Hz.
- `action_monitor_rate: 20.0` Hz.
- `allow_partial_joints_goal: false`.
- Per-joint tolerances: `trajectory: 0.2 s, goal: 0.1 rad` (uniform).
- `stopped_velocity_tolerance: 0.2`.

The `forward_*_controllers` operate the same 6 joints on `velocity` / `position` interfaces respectively.

---

## 4. Robot Description

### 4.1 `urdf/ur_gz.urdf.xacro`
Top-level xacro that:
- Includes `ur_description/urdf/ur_macro.xacro` (the actual UR robot macros).
- Includes the ros2_control xacro (`ur_gz.ros2_control.xacro`).
- Adds a fixed `world` link and a 5×5 ground plane (visual + collision) at z = -0.01 m.
- Inserts the `<gazebo>` plugin block:
  ```xml
  <plugin filename="gz_ros2_control-system"
          name="gz_ros2_control::GazeboSimROS2ControlPlugin">
    <parameters>$(arg simulation_controllers)</parameters>
    <ros><namespace>$(arg ros_namespace)</namespace></ros>
  </plugin>
  ```
- Invokes `<xacro:ur_ros2_control .../>` to register the hardware interface.

**XACRO args (with defaults):**
`name=ur`, `ur_type=ur5x` (intentionally invalid default — must be overridden),
`tf_prefix=""`, `joint_limit_params`, `kinematics_params`, `physical_params`, `visual_params` (all default to `ur_description/config/<ur_type>/...yaml`),
`safety_limits=false`, `safety_pos_margin=0.15`, `safety_k_position=20`, `simulation_controllers=""`, `ros_namespace=""`.

### 4.2 `urdf/ur_gz.ros2_control.xacro`
Defines the macro `ur_ros2_control(name, tf_prefix, transmission_hw_interface, initial_positions)`:
- Creates a single `<ros2_control name="${name}" type="system">` block.
- Hardware plugin: `gz_ros2_control/GazeboSimSystem` (the gz-side hardware interface that proxies to Gazebo Sim's joint APIs).
- Default `initial_positions` = `{shoulder_pan: 0.0, shoulder_lift: -1.57, elbow: 0.0, wrist_1: -1.57, wrist_2: 0.0, wrist_3: 0.0}` (this matches the "Home" pose used in `ur3_env.py:reset`).
- Pulls in joint control description via `<xacro:ur_joint_control_description tf_prefix initial_positions/>`.

---

## 5. Tests

| File | Type | What it does |
|------|------|--------------|
| `test_common.py` | Library | Helper classes (`_ServiceInterface`, `ActionInterface`) to wait for and call ROS 2 services / actions in tests. |
| `test_description.py` | Parametrized pytest (always on) | For each of 14 `ur_type`s × 2 prefixes (`""`, `my_ur_`): runs `xacro` → URDF → `check_urdf` and asserts the resulting URDF is valid. |
| `test_gz.py` | Launch test (only with `UR_SIM_INTEGRATION_TESTS=ON`) | Spins up `ur_sim_control.launch.py` headlessly (`ur_type:=ur5e`, GUI off, RViz off) and sends a 3-waypoint `FollowJointTrajectory` action over ~12 s. Asserts the goal is accepted and `error_code == SUCCESSFUL` within a 60 s timeout. |

The `test_gz.py` test exercises the **same** `/joint_trajectory_controller/follow_joint_trajectory` action interface used by the RL environment, but at the action level (not raw topic) — the RL env bypasses the action wrapper for tighter 0.1 s step control.

---

## 6. Build Artifacts

### 6.1 `build/ur_simulation_gz/`
Standard ament_cmake build tree with `CMakeCache.txt`, `CMakeFiles/`, `colcon_command_prefix_build.sh`, `colcon_command_prefix_build.sh.env`, `CTestConfiguration.ini`, `CTestCustom.cmake`, and the per-`ament_cmake_*` subfolders. Marker file `colcon_build.rc` contains `0` (success exit code).

### 6.2 `install/ur_simulation_gz/`
Standard ament install:
- `share/ur_simulation_gz/cmake/` — `ur_simulation_gzConfig.cmake`, `ur_simulation_gzConfig-version.cmake`.
- `share/ur_simulation_gz/config/ur_controllers.yaml` (symlink).
- `share/ur_simulation_gz/launch/{ur_sim_control,ur_sim_moveit}.launch.py` (symlinks).
- `share/ur_simulation_gz/urdf/{ur_gz.ros2_control,ur_gz.urdf}.xacro` (symlinks).
- `share/ur_simulation_gz/environment/`, `share/ur_simulation_gz/hook/`.
- `package.xml` (symlink), `local_setup.{bash,sh,zsh,dsv}` (symlinks), `package.{bash,sh,dsv,ps1,zsh}`.
- (All symlinks point to the original files in `src/ur_simulation_gz/ur_simulation_gz/`.)

### 6.3 `log/build_2026-06-17_15-44-58/`
- `events.log` (29.6 KB) — full colcon event log (JobQueued → cmake → build → install → JobEnded rc=0).
- `logger_all.log` (21.4 KB) — colcon logger output.
- Build time: ≈ 1.5 s end-to-end (configure 1.1 s + build + install).
- Environment used: `ROS_DISTRO=jazzy`, `COLCON_PREFIX_PATH=/home/mie/ur3_isaac_ws/install`, `AMENT_PREFIX_PATH=/opt/ros/jazzy`.

---

## 7. Supported UR Models

`ur3, ur5, ur10, ur3e, ur5e, ur7e, ur10e, ur12e, ur16e, ur8long, ur15, ur18, ur20, ur30`

(Same set parametrized in both launch files, in `test_description.py`, and in the `ur_type` choices of `ur_gz.urdf.xacro`.)

---

## 8. Documentation Summary

- **`installation.rst`** — two install paths: apt binary (`ros-${ROS_DISTRO}-ur-simulation-gz`) or source build into a colcon workspace.
- **`usage.rst`** — examples for both launchers; covers custom description, `tf_prefix`, and custom world (`test_world.sdf`) usage.
- **`migration_notes.rst`** — Iron→Jazzy and Kilted→Lyrical migration guides (included from sub-files).
- **`ci_status.md`** — full CI matrix (Humble, Jazzy, Kilted, Lyrical, Rolling) with binary, semi-binary, and buildfarm status badges.
- **`CHANGELOG.rst`** — versions 2.0.0 → 2.5.0, including UR18 (2.5.0), UR8long (2.4.0), UR15 (2.3.0), UR7e/UR12e (2.2.0).

---

## 9. How to Launch

```bash
source /opt/ros/jazzy/setup.bash
source ~/Desktop/rakib2009/install/setup.bash

# Basic Gazebo + RViz with UR3 (RL use case)
ros2 launch ur_simulation_gz ur_sim_control.launch.py ur_type:=ur3

# Or with MoveIt
ros2 launch ur_simulation_gz ur_sim_moveit.launch.py ur_type:=ur3

# Quick sanity: send a move command
ros2 run ur_robot_driver example_move.py
```

---

## 10. Key Observations & Caveats

1. **Upstream clone** — this is a near-pristine copy of the Universal Robots `ros2` branch (no apparent local edits besides initial commit). The CHANGELOG shows version 2.5.0, but the build is the head of the local branch.
2. **No `obj`/`world`/sphere geometry present** — for the RL obstacle-avoidance task, a custom Gazebo world with the obstacle sphere (or the hard-coded geometric check in `ur3_env.py`) is required. The `doc/resources/test_world.sdf` is an example custom world, not the RL one.
3. **`gz_ros2_control` is the bridge** — Gazebo Sim → ros2_control → `joint_trajectory_controller` (position command interface). This is what makes the `JointTrajectory` topic work for `ur3_env.py`.
4. **`/clock` is bridged** — `use_sim_time: true` is set on `robot_state_publisher` to keep all nodes on simulated time.
5. **Build is reproducible** — the build is `colcon build --symlink-install` against Jazzy; rebuild is safe and incremental (the existing build tree can be reused).

---

## 11. Relationship to `rakib_RL`

`rakib2009` is the **simulation provider** for `rakib_RL`:

| RL needs | Provided by `rakib2009` |
|----------|-------------------------|
| Gazebo running a UR3 | `ur_sim_control.launch.py ur_type:=ur3` |
| `/joint_states` topic | `joint_state_broadcaster` |
| `/joint_trajectory_controller/joint_trajectory` topic | `joint_trajectory_controller` (position command) |
| `use_sim_time` | `/clock` bridge from `gz_sim` |
| UR3 DH parameters (used in env's FK) | Provided externally by `ur_description` package (not in this repo) |

The RL env in `rakib_RL/ur3_env.py` does not depend on this package at the Python level — it only depends on the running ROS 2 topics at runtime.
