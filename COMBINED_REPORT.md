# Combined Report: `rakib_RL` + `rakib2009`

**Generated:** 2026-06-24
**Author context:** Read-only review of two Desktop directories. No files were edited.
**Workspace root:** `/home/mie/Desktop/New Folder`
**Current working directory:** `/home/mie/Desktop/New Folder` (empty before this report).

---

## 1. Executive Summary

The two directories form a **two-part reinforcement-learning research stack for a UR3 robotic arm** in Gazebo simulation:

| Directory | Role | Type | State |
|-----------|------|------|-------|
| `~/Desktop/rakib2009` | **Simulation provider** | ROS 2 colcon workspace wrapping Universal Robots' `ur_simulation_gz` (Gazebo Sim) | Built and installed; one successful colcon build on 2026-06-17 |
| `~/Desktop/rakib_RL`   | **Learning agent**     | Standalone Python (Gymnasium + Stable-Baselines3) training script + custom env bridging to ROS 2 | Training in progress — at least **120 000 of 500 000** PPO timesteps completed across **9 PPO runs** |

`rakib2009` exposes the UR3 in Gazebo with `ros2_control`. `rakib_RL` consumes that simulation through ROS 2 topics to train a PPO policy that drives the arm to a target while avoiding a spherical obstacle.

---

## 2. Combined System Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           rakib2009 (sim)                                    │
│                                                                              │
│  Gazebo Sim  ──►  gz_ros2_control plugin  ──►  joint_trajectory_controller  │
│       ▲                       │                          │                   │
│       │                       │                          ▼                   │
│   /clock bridge       robot_state_publisher       /joint_states             │
│  (ros_gz_bridge)          (TF tree)                   (publish)              │
│                                                                              │
└──────────────────────────────┬───────────────────────────────────────────────┘
                               │  ROS 2 topics
                               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                          rakib_RL (learner)                                  │
│                                                                              │
│   UR3ROSNode (ur3_rl_bridge)                                                 │
│      ├── subscribes:  /joint_states                                          │
│      └── publishes:  /joint_trajectory_controller/joint_trajectory          │
│                                                                              │
│   UR3ObstacleEnv (Gymnasium)                                                 │
│      ├── action:     Box(-1, 1, 6)  (6-DOF joint deltas)                    │
│      ├── observation: Box(-π, π, 6) (joint angles)                          │
│      ├── FK via DH table → Cartesian pose                                   │
│      ├── Reward: +100 goal / -100 collision / -dist + 0.1*margin            │
│      └── Home reset pose: [0, -π/2, 0, -π/2, 0, 0]                           │
│                                                                              │
│   train_agent.py                                                             │
│      └── PPO (MlpPolicy), 500 k timesteps, CheckpointCallback every 10 k     │
│                                                                              │
└──────────────────────────────┬───────────────────────────────────────────────┘
                               │
                               ▼
                ┌──────────────────────────────┐
                │  models/checkpoints/         │
                │  ur3_ppo_checkpoint_*.zip   │
                │  ur3_ppo_final.zip (TBD)     │
                └──────────────────────────────┘
                               │
                               ▼
                ┌──────────────────────────────┐
                │  ur3_tensorboard/PPO_*/      │
                │  TensorBoard event files    │
                └──────────────────────────────┘
```

---

## 3. Contract Between the Two Workspaces

| Direction | Topic / Interface | raktype | Used by |
|-----------|------------------|---------|---------|
| sim → RL  | `/joint_states` (sensor_msgs/JointState) | Topic, first 6 positions | `UR3ROSNode.joint_state_callback` |
| RL → sim  | `/joint_trajectory_controller/joint_trajectory` (trajectory_msgs/JointTrajectory) | Topic, 1-point trajectories, `time_from_start=0.1 s` | `UR3ObstacleEnv.step` |
| sim → all | `/clock` (rosgraph_msgs/Clock) | Topic, bridged from `gz.msgs.Clock` | Provides `use_sim_time=true` |

### Joint naming convention (must match exactly)
`shoulder_pan_joint`, `shoulder_lift_joint`, `elbow_joint`, `wrist_1_joint`, `wrist_2_joint`, `wrist_3_joint`

### Control parameters that must agree
- `joint_trajectory_controller` accepts **position** commands (not velocity/torque) at 500 Hz update rate.
- Each RL step corresponds to a 1-point trajectory with `time_from_start = 0.1 s`.
- Joint safety limits: `safety_limits=true`, `safety_pos_margin=0.15`, `safety_k_position=20` (set in launch).
- Reset "Home" pose: `[0.0, -1.57, 0.0, -1.57, 0.0, 0.0]` rad — matches the `initial_positions` default in `ur_gz.ros2_control.xacro`.

---

## 4. Configuration Snapshot

### 4.1 RL training (`rakib_RL/train_agent.py`)
| Param | Value |
|-------|-------|
| Algorithm | PPO (`MlpPolicy`) |
| Total timesteps | 500 000 |
| Checkpoint freq | 10 000 |
| TensorBoard log | `./ur3_tensorboard/` |
| Checkpoint path | `./models/checkpoints/` |
| Final save | `ur3_ppo_final` |
| Resume mode | `reset_num_timesteps=False` |

### 4.2 RL environment (`rakib_RL/ur3_env.py`)
| Param | Value |
|-------|-------|
| Action space | `Box(-1, 1, (6,), float32)` |
| Observation space | `Box(-π, π, (6,), float32)` |
| Step duration | 0.1 s (real-time, `time.sleep(0.1)`) |
| Max step per joint | 0.05 rad |
| Max steps / episode | 300 |
| Target position | `(0.3, -0.2, 0.1)` m |
| Obstacle center | `(0.2, 0.0, 0.2)` m |
| Obstacle radius | 0.12 m |
| Goal tolerance | 0.05 m |
| Reward: collision | -100 |
| Reward: success | +100 |
| Reward: step | `-distance_to_goal + 0.1*max(0, min_dist - radius)` |
| Home reset pose | `[0, -π/2, 0, -π/2, 0, 0]` rad |
| Reset settle time | 3.5 s |

### 4.3 Gazebo sim (`rakib2009/.../ur_sim_control.launch.py`)
| Arg | Value (default) |
|-----|-----------------|
| `ur_type` | `ur5e` (must be overridden to `ur3` for the RL use) |
| `safety_limits` | `true` |
| `safety_pos_margin` | `0.15` |
| `safety_k_position` | `20` |
| `world_file` | `empty.sdf` |
| `gazebo_gui` | `true` |
| `initial_joint_controller` | `joint_trajectory_controller` |
| `activate_joint_controller` | `true` |
| `launch_rviz` | `true` |
| Controller update rate | 500 Hz |
| JTC state publish rate | 100 Hz |

### 4.4 URDF (`rakib2009/.../ur_gz.ros2_control.xacro`)
| Param | Value |
|-------|-------|
| Hardware plugin | `gz_ros2_control/GazeboSimSystem` |
| gz plugin | `gz_ros2_control::GazeboSimROS2ControlPlugin` |
| Initial positions | shoulder_pan=0, shoulder_lift=-π/2, elbow=0, wrist_1=-π/2, wrist_2=0, wrist_3=0 |
| Ground plane | 5×5 m at z=-0.01 m, white 50%-alpha |

---

## 5. Training Progress Snapshot

| Run | Date | Events file | Latest checkpoint |
|-----|------|-------------|-------------------|
| PPO_1 | 2026-06-18 | 4.4 KB | — |
| PPO_2 | 2026-06-21 | 1.4 KB | — |
| PPO_3 | 2026-06-21 | 2.6 KB | — |
| PPO_4 | 2026-06-21 | 0.1 KB (aborted) | — |
| PPO_5 | 2026-06-23 | 8.2 KB | — |
| PPO_6 | 2026-06-23 | 0.2 KB (aborted) | — |
| PPO_7 | 2026-06-23 | 0.1 KB (aborted) | — |
| PPO_8 | 2026-06-23 | 1.0 KB | — |
| PPO_9 | 2026-06-24 | 44.5 KB (active) | 120 000 steps |

- **Best run progress:** PPO_9 reached 120 000 steps (24 % of 500 k).
- **Checkpoint cadence:** every 10 000 steps; file size ≈ 146–149 KB.
- **No `ur3_ppo_final.zip` yet** — training has not completed the 500 k target.
- **Multiple short PPO_4/6/7 runs** suggest crashes/early restarts being investigated.

---

## 6. Build Status (`rakib2009`)

| Stage | Outcome | Time |
|-------|---------|------|
| Configure (cmake) | Success | ~1.1 s |
| Build | Success (no compile units — pure install/launch/urdf) | <0.1 s |
| Install | Success (symlinked) | <0.1 s |
| Job end | `rc: 0` | — |
| ROS 2 distro | `jazzy` | — |
| Build flags | `--symlink-install` | — |
| Build date | 2026-06-17 15:44:58 | — |
| Build dir | `~/Desktop/rakib2009/build/ur_simulation_gz/` | — |
| Install dir | `~/Desktop/rakib2009/install/ur_simulation_gz/` | — |

---

## 7. End-to-End Run Procedure

```bash
# --- 1. Source the sim workspace ---
source /opt/ros/jazzy/setup.bash
source ~/Desktop/rakib2009/install/setup.bash

# --- 2. Launch Gazebo + UR3 + controllers (terminal 1) ---
ros2 launch ur_simulation_gz ur_sim_control.launch.py ur_type:=ur3 launch_rviz:=false gazebo_gui:=false

# --- 3. Train (terminal 2) ---
cd ~/Desktop/rakib_RL
python3 train_agent.py
# - Auto-saves checkpoint every 10 000 steps.
# - set reset_num_timesteps=False in train_agent.py so it resumes from the latest checkpoint.
# - Monitor: tensorboard --logdir ~/Desktop/rakib_RL/ur3_tensorboard
```

For visualisation, add the obstacle and target in the Gazebo world (or rely on the geometric checks inside `ur3_env.py` which require no Gazebo geometry).

---

## 8. Risks, Gaps, and Recommendations

| # | Observation | Impact | Suggested fix |
|---|-------------|--------|---------------|
| 1 | Obstacle geometry is only checked in the env's FK, not in Gazebo | The RL sees correct collisions but the visualization will not show the arm penetrating the obstacle | Place a visible (non-collidable) sphere at `(0.2, 0, 0.2)` r=0.12 in `test_world.sdf` or a custom world |
| 2 | `time.sleep(0.1)` couples env to wall clock | Cannot vectorize; slows training | Replace with simulated-time step using a ROS clock subscription, or run multiple sims in parallel |
| 3 | Target/obstacle fixed across episodes | Policy may overfit | Add `numpy.random.uniform` perturbation in `reset()` and freeze the obstacle for curriculum phases |
| 4 | No torque / acceleration constraints in the action mapping | Aggressive actions clipped only by `safety_limits` downstream | Add soft penalty for `||dθ/dt||` exceeding a threshold |
| 5 | Reward is dense but the safety margin coefficient is small (0.1) | May not learn to "graze" the obstacle | Tune or schedule the safety weight alongside distance weight |
| 6 | Three of nine PPO runs (PPO_4, _6, _7) are tiny | Likely crashes on launch | Check Gazebo / controller logs; possibly caused by port conflicts or `rclpy.ok()` checks during restart |
| 7 | No explicit test in the sim workspace validates the env's topic contract | A future Gazebo upgrade could break the bridge | Add a launch test that publishes a known trajectory and asserts the joint state matches within tolerance |
| 8 | `ur_simulation_gz` is unmodified upstream code | Fine for the RL use, but no project-specific logging in the sim | If metrics are needed, fork a thin overlay package or use ROS 2 topic recording |

---

## 9. Files of Interest (quick index)

### `rakib_RL/`
- `train_agent.py` — main training entry
- `ur3_env.py` — env + ROS bridge
- `models/checkpoints/ur3_ppo_checkpoint_120000_steps.zip` — most recent saved policy
- `ur3_tensorboard/PPO_9/events.out.tfevents.1782275916.mie-MS-7D42.12133.0` — most recent active training log

### `rakib2009/src/ur_simulation_gz/ur_simulation_gz/`
- `launch/ur_sim_control.launch.py` — Gazebo + UR3 + controllers bring-up
- `launch/ur_sim_moveit.launch.py` — same + MoveIt
- `urdf/ur_gz.urdf.xacro` — robot description + ground plane + gz plugin
- `urdf/ur_gz.ros2_control.xacro` — `gz_ros2_control` hardware interface
- `config/ur_controllers.yaml` — controller manager + JTC + forward cmd controllers
- `test/test_description.py` — 14 × 2 xacro/urdf validation
- `test/test_gz.py` — FollowJointTrajectory integration test

### `rakib2009/install/ur_simulation_gz/` and `rakib2009/log/`
- All symlinked install artifacts and the build event log for reproducibility.

---

## 10. Cross-References to Other Reports

- **Detailed sim report:** `rakib2009_report.md` (this folder)
- **Detailed RL report:** `rakib_RL_report.md` (this folder)

---

**End of combined report.**
