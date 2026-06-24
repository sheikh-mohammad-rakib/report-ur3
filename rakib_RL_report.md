# Report: `~/Desktop/rakib_RL`

**Generated:** 2026-06-24
**Type:** Reinforcement Learning Training Workspace (UR3 Robot)
**Purpose:** Trains a PPO (Proximal Policy Optimization) agent to control a simulated UR3 robotic arm in Gazebo for an obstacle-avoidance reaching task.

---

## 1. Workspace Overview

| Property | Value |
|----------|-------|
| Root path | `~/Desktop/rakib_RL` |
| Top-level files | `train_agent.py`, `ur3_env.py` |
| Top-level dirs | `models/`, `ur3_tensorboard/`, `__pycache__/`, `.vscode/` |
| Version control | git (2 commits on branch — latest: `6836511 trainning running`) |
| Python framework | Gymnasium + Stable-Baselines3 |
| Middleware | ROS 2 (rclpy) — bridges to Gazebo sim |
| Simulation target | Gazebo (Universal Robots UR3, controlled via `joint_trajectory_controller`) |
| Training goal | 500,000 timesteps |
| Checkpoints saved | Every 10,000 steps |

---

## 2. Source Files

### 2.1 `train_agent.py` (43 lines)
- **Role:** Training entry point. Instantiates the env, builds the PPO model, sets up auto-checkpointing, runs `model.learn()`, and saves the final model.
- **Algorithm:** PPO with `MlpPolicy`.
- **Key parameters:**
  - `tensorboard_log="./ur3_tensorboard/"`
  - `CheckpointCallback(save_freq=10000, save_path='./models/checkpoints/', name_prefix='ur3_ppo_checkpoint')`
  - `total_timesteps=500000`
  - `reset_num_timesteps=False` (continues from prior checkpoint if resumed)
- **Final save name:** `ur3_ppo_final`
- **Import critical note (in code):** `from ur3_env import UR3ObstacleEnv` — the env class name must be matched exactly.

### 2.2 `ur3_env.py` (151 lines)
- **Role:** Gymnasium RL environment wrapping the ROS 2 ↔ Gazebo UR3 bridge.
- **Classes:**
  1. `UR3ROSNode(Node)` — ROS 2 node `ur3_rl_bridge`
     - **Subscribes:** `/joint_states` (`sensor_msgs/msg/JointState`)
     - **Publishes:** `/joint_trajectory_controller/joint_trajectory` (`trajectory_msgs/msg/JointTrajectory`)
     - Maintains `current_joint_angles` (numpy, 6-DOF).
  2. `UR3ObstacleEnv(gym.Env)` — the RL environment.
- **Spaces:**
  - **Action:** `Box(-1.0, 1.0, shape=(6,), float32)` — normalized deltas.
  - **Observation:** `Box(-π, π, shape=(6,), float32)` — current joint angles.
- **Robot parameters:**
  - `max_joint_step = 0.05` rad per 0.1 s step.
  - Step duration: `0.1 s` (waited in `step()` via `time.sleep`).
  - Episode horizon: `max_steps = 300`.
- **Scenario coordinates:**
  - **Target:** `(0.3, -0.2, 0.1)` m.
  - **Obstacle center:** `(0.2, 0.0, 0.2)` m, **radius:** `0.12` m.
- **Forward kinematics:** Standard DH-table for UR3 (6 rows). Returns all 7 joint positions (base + 6 link origins) for collision checking.
- **Reward shaping:**
  | Event | Reward | Done? |
  |-------|--------|-------|
  | Collision with sphere (any link < radius) | `-100.0` | terminate |
  | End-effector within `0.05 m` of target | `+100.0` | terminate |
  | Otherwise | `-distance_to_goal + 0.1 * safety_margin` | — |
  | `current_step >= 300` | — | truncate |
- **Reset behavior:** Sends the arm to a "Home" pose `[0, -π/2, 0, -π/2, 0, 0]` over 3 s, then `time.sleep(3.5)` to ensure physical arrival before the next episode.

---

## 3. Training Artifacts

### 3.1 `models/checkpoints/`
12 PPO checkpoints recorded (most recent first):

| File | Approx. Steps | Last Modified |
|------|---------------|---------------|
| `ur3_ppo_checkpoint_120000_steps.zip` | 120 k | 2026-06-24 14:29 |
| `ur3_ppo_checkpoint_110000_steps.zip` | 110 k | 2026-06-24 14:08 |
| `ur3_ppo_checkpoint_100000_steps.zip` | 100 k | 2026-06-24 13:48 |
| `ur3_ppo_checkpoint_90000_steps.zip` | 90 k | 2026-06-24 13:29 |
| `ur3_ppo_checkpoint_80000_steps.zip` | 80 k | 2026-06-24 13:10 |
| `ur3_ppo_checkpoint_70000_steps.zip` | 70 k | 2026-06-24 12:51 |
| `ur3_ppo_checkpoint_60000_steps.zip` | 60 k | 2026-06-24 12:32 |
| `ur3_ppo_checkpoint_50000_steps.zip` | 50 k | 2026-06-24 12:13 |
| `ur3_ppo_checkpoint_40000_steps.zip` | 40 k | 2026-06-24 11:54 |
| `ur3_ppo_checkpoint_30000_steps.zip` | 30 k | 2026-06-24 11:35 |
| `ur3_ppo_checkpoint_20000_steps.zip` | 20 k | 2026-06-24 11:16 |
| `ur3_ppo_checkpoint_10000_steps.zip` | 10 k | 2026-06-24 10:57 |

- File size: ~146–149 KB per checkpoint.
- **Status:** Training reached at least 120 k of the 500 k target (24 %).
- No `ur3_ppo_final.zip` present yet (training not complete).

### 3.2 `ur3_tensorboard/`
Nine PPO run directories — represents iterative experiments / hyper-parameter trials.

| Run | Created | Events file size |
|-----|---------|------------------|
| `PPO_1` | 2026-06-18 | 4.4 KB |
| `PPO_2` | 2026-06-21 | 1.4 KB |
| `PPO_3` | 2026-06-21 | 2.6 KB |
| `PPO_4` | 2026-06-21 | 0.1 KB (tiny — likely aborted) |
| `PPO_5` | 2026-06-23 | 8.2 KB |
| `PPO_6` | 2026-06-23 | 0.2 KB (tiny) |
| `PPO_7` | 2026-06-23 | 0.1 KB (tiny) |
| `PPO_8` | 2026-06-23 | 1.0 KB |
| `PPO_9` | 2026-06-24 | 44.5 KB (largest — most recent active run; 3 event files) |

- The directory naming convention `PPO_<n>` is a fresh sub-folder for each new training invocation.
- `PPO_9` is the currently active run (matches the latest checkpoints at 120 k steps).

---

## 4. ROS 2 ↔ Simulation Interface

The RL env requires a running Gazebo + `ur_simulation_gz` stack (provided by the `rakib2009` workspace). Communication contract:

| Direction | Topic | Type | Notes |
|-----------|-------|------|-------|
| Sub | `/joint_states` | `sensor_msgs/JointState` | First 6 positions used |
| Pub | `/joint_trajectory_controller/joint_trajectory` | `trajectory_msgs/JointTrajectory` | 6 joints named: `shoulder_pan_joint, shoulder_lift_joint, elbow_joint, wrist_1_joint, wrist_2_joint, wrist_3_joint` |

Each `step()`:
1. Reads current joint state from ROS topic.
2. Computes new target = current + action × 0.05 (clipped to ±π).
3. Sends a 1-point trajectory with `time_from_start = 0.1 s`.
4. Sleeps 100 ms, then re-reads the state for the next observation.
5. Computes FK → distance to goal + collision check.
6. Returns `(obs, reward, terminated, truncated, info)`.

---

## 5. Key Observations & Risks

1. **No obstacle spheres in this repo** — the obstacle definition is hard-coded in the env (`obstacle_center`/`obstacle_radius`) but the actual Gazebo spheres (if any) are not present in this directory. The comment says "Scenario coordinates matching the Gazebo spheres" — these must be set up in the Gazebo world file.
2. **Synchronous sleep** — `time.sleep(0.1)` in `step()` couples wall-clock to env step. The env cannot be vectorized without removing this.
3. **Open-loop action safety** — no joint-limit / velocity / acceleration check before publishing; relies entirely on the controller's safety_limits (`safety_limits=true`, `safety_pos_margin=0.15`, `safety_k_position=20`) in the URDF launch args.
4. **Collision check is geometric only** — checks sphere-line proximity; ignores the obstacle's surface/box. Acceptable for spherical obstacles only.
5. **No goal randomization** — target and obstacle are fixed per episode; risk of policy overfitting.
6. **No curriculum** — `max_steps = 300` and target tolerance `0.05 m` are fixed.

---

## 6. How to Resume / Continue

```bash
cd ~/Desktop/rakib_RL
source /opt/ros/jazzy/setup.bash
source ~/Desktop/rakib2009/install/setup.bash    # gazebo sim package
# In one terminal — bring up Gazebo + UR3
ros2 launch ur_simulation_gz ur_sim_control.launch.py ur_type:=ur3
# In another — resume training (reset_num_timesteps=False preserves counter)
python3 train_agent.py
```

The script auto-resumes from the latest `ur3_ppo_checkpoint_*` due to `reset_num_timesteps=False`.
