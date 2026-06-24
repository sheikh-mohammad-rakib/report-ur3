# UR3 Reinforcement Learning in Gazebo — Research Reports

A two-part reinforcement-learning research stack that trains a **PPO policy** to drive a simulated **UR3 robotic arm** in **Gazebo** toward a target while avoiding a spherical obstacle.

This repository contains the **reports** generated for a read-only review of the two source projects. The source code itself lives in sibling directories on the Desktop:

| Project | Role |
|---------|------|
| [`~/Desktop/rakib2009`](https://github.com/UniversalRobots/Universal_Robots_ROS2_GZ_Simulation) | ROS 2 colcon workspace — Gazebo simulation of UR robots (`ur_simulation_gz` v2.5.0, ROS 2 Jazzy) |
| `~/Desktop/rakib_RL` | Standalone Python workspace — Gymnasium + Stable-Baselines3 PPO trainer that drives the simulated UR3 through ROS 2 topics |

---

## Contents

| File | Description |
|------|-------------|
| [`final_report.md`](./final_report.md) | Meta-summary of this report directory |
| [`COMBINED_REPORT.md`](./COMBINED_REPORT.md) | Unified cross-workspace report (start here) |
| [`rakib2009_report.md`](./rakib2009_report.md) | Detailed report on the Gazebo simulation workspace |
| [`rakib_RL_report.md`](./rakib_RL_report.md) | Detailed report on the RL training workspace |
| [`terminal_output.md`](./terminal_output.md) | Cleaned transcript of the report-generation session |

---

## System Overview

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
│      └── Reward: +100 goal / -100 collision / -dist + 0.1*margin            │
│                                                                              │
│   train_agent.py                                                             │
│      └── PPO (MlpPolicy), 500 k timesteps, CheckpointCallback every 10 k   │
│                                                                              │
└──────────────────────────────┬───────────────────────────────────────────────┘
                               │
                               ▼
                       PPO checkpoints + TensorBoard logs
```

---

## Sim ↔ RL Contract

| Direction | Topic | Message type | Purpose |
|-----------|-------|--------------|---------|
| sim → RL  | `/joint_states` | `sensor_msgs/JointState` | Current joint positions (first 6 used) |
| RL → sim  | `/joint_trajectory_controller/joint_trajectory` | `trajectory_msgs/JointTrajectory` | 1-point trajectories, `time_from_start = 0.1 s` |
| sim → all | `/clock` | `rosgraph_msgs/Clock` | Bridged from `gz.msgs.Clock` — enables `use_sim_time` |

**Joint naming convention (must match exactly):**
`shoulder_pan_joint`, `shoulder_lift_joint`, `elbow_joint`, `wrist_1_joint`, `wrist_2_joint`, `wrist_3_joint`

**Home reset pose:** `[0.0, -π/2, 0.0, -π/2, 0.0, 0.0]` rad — matches the `initial_positions` default in `ur_gz.ros2_control.xacro`.

---

## Quick Start

```bash
# 1. Source the simulation workspace
source /opt/ros/jazzy/setup.bash
source ~/Desktop/rakib2009/install/setup.bash

# 2. Launch Gazebo + UR3 + controllers (terminal 1)
ros2 launch ur_simulation_gz ur_sim_control.launch.py \
    ur_type:=ur3 launch_rviz:=false gazebo_gui:=false

# 3. Train (terminal 2)
cd ~/Desktop/rakib_RL
python3 train_agent.py
# - Auto-saves checkpoint every 10 000 steps
# - set reset_num_timesteps=False to resume from latest checkpoint
# - Monitor: tensorboard --logdir ~/Desktop/rakib_RL/ur3_tensorboard
```

---

## Configuration Snapshot

### RL training (`train_agent.py`)
- Algorithm: **PPO** (`MlpPolicy`)
- Total timesteps: **500 000**
- Checkpoint frequency: **every 10 000 steps**
- TensorBoard log: `./ur3_tensorboard/`
- Final save: `ur3_ppo_final`

### RL environment (`ur3_env.py`)
| Parameter | Value |
|-----------|-------|
| Action space | `Box(-1, 1, (6,), float32)` |
| Observation space | `Box(-π, π, (6,), float32)` |
| Step duration | 0.1 s (real-time) |
| Max step per joint | 0.05 rad |
| Max steps / episode | 300 |
| Target position | `(0.3, -0.2, 0.1)` m |
| Obstacle center / radius | `(0.2, 0.0, 0.2)` m / 0.12 m |
| Goal tolerance | 0.05 m |
| Collision reward | −100 |
| Success reward | +100 |
| Step reward | `-distance + 0.1*max(0, min_dist - radius)` |

---

## Training Progress

9 PPO runs have been launched. The most recent run (**PPO_9**) is at **120 000 / 500 000 steps (24 %)** with 12 checkpoints on disk.

| Run | Date | Latest checkpoint | Notes |
|-----|------|-------------------|-------|
| PPO_1 → PPO_8 | 2026-06-18 → 2026-06-23 | — | Iterative hyper-parameter trials; PPO_4/6/7 aborted early |
| **PPO_9** | 2026-06-24 | **120 000 steps** | Currently active |

No `ur3_ppo_final.zip` yet — training has not completed the 500 k target.

---

## Build Status

| Stage | Outcome | Time |
|-------|---------|------|
| Configure (cmake) | Success | ~1.1 s |
| Build | Success (pure install/launch/urdf — no compile units) | <0.1 s |
| Install | Success (symlinked) | <0.1 s |
| Job end | `rc: 0` | — |
| ROS 2 distro | `jazzy` | — |
| Build date | 2026-06-17 15:44:58 | — |

---

## Risks and Recommendations

| # | Observation | Suggested fix |
|---|-------------|---------------|
| 1 | Obstacle geometry only checked in env's FK, not in Gazebo | Add a visible (non-collidable) sphere in a custom world |
| 2 | `time.sleep(0.1)` couples env to wall clock | Replace with simulated-time stepping via ROS clock subscription |
| 3 | Target/obstacle fixed across episodes (overfitting risk) | Add domain randomization in `reset()` |
| 4 | No torque/acceleration constraints in action mapping | Add soft penalty for `||dθ/dt||` exceeding threshold |
| 5 | Safety margin coefficient (0.1) is small | Tune or schedule alongside distance weight |
| 6 | Three of nine PPO runs (PPO_4/6/7) aborted early | Investigate Gazebo / controller logs |
| 7 | No launch test validates the env's topic contract | Add a test that publishes a known trajectory and asserts joint state |
| 8 | `ur_simulation_gz` is unmodified upstream code | Fork a thin overlay package if project-specific logging is needed |

See [`COMBINED_REPORT.md`](./COMBINED_REPORT.md) §8 for the full discussion.

---

## Repository Layout

```
report_ur3/
├── README.md              ← this file
├── final_report.md        ← meta-summary of this directory
├── COMBINED_REPORT.md     ← unified cross-workspace report
├── rakib2009_report.md    ← detailed sim-workspace report
├── rakib_RL_report.md     ← detailed RL-workspace report
└── terminal_output.md     ← cleaned session transcript
```

---

## License & Attribution

The simulation workspace wraps **Universal Robots'** [`ur_simulation_gz`](https://github.com/UniversalRobots/Universal_Robots_ROS2_GZ_Simulation) package, licensed under **BSD-3-Clause** (v2.5.0, released 2025-10-13).

The RL training workspace uses:
- [Gymnasium](https://gymnasium.farama.org/) (Apache-2.0 / MIT)
- [Stable-Baselines3](https://stable-baselines3.readthedocs.io/) (MIT)
- [ROS 2 Jazzy](https://docs.ros.org/en/jazzy/) (Apache-2.0)

---

## Generated

2026-06-24 — read-only review. No source files outside this directory were modified.
