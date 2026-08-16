---
title: "Roboracer: Autonomous Racing via Optimal Planning and Reinforcement Learning"
date: 2026-06-08
draft: false
tags:
  - Reinforcement Learning
  - Autonomous Driving
  - Path Planning
  - ROS 2
  - Robotics
summary: "An integrated F1TENTH autonomous racing planner combining offline TUM-based raceline optimization with online PPO-based Deep Reinforcement Learning (DRL) controllers."
showToc: true
TocOpen: true
math: true
---

### Project Overview
**Roboracer** is a modular autonomy stack developed for the **F1TENTH** racing platform inside the AutoDRIVE simulator. This project was a collaborative effort completed with **three teammates**. 

My contribution was the **Planning Module**, which contains:
1. **An Offline Global Planner**: Computes the optimal raceline using curvature-constrained minimum-time optimization.
2. **An Online Local Planner (Waypoint-Following DRL)**: Tracks the optimal global raceline.
3. **An Online Local Planner (Generalized DRL)**: Drives track-agnostically on unseen layouts.

---

### Demonstration
Here is the final system navigating the F1TENTH track in the AutoDRIVE simulator:

{{< youtube ZMOKa8dJni0 >}}

---

### 1. Offline Global Planning (Trajectory Optimization)
The global planner calculates a minimum-time path offline based on the **TUM Global Raceline Optimization** framework. From a track occupancy grid, it extracts boundary data and solves an optimization problem to generate a list of waypoints:

$$ \mathbf{w}_i = [x_i, y_i, \psi_i, \kappa_i, v_{\text{ref}, i}] $$

Where $(x_i, y_i)$ are coordinates, $\psi_i$ is yaw heading, $\kappa_i$ is curvature, and $v_{\text{ref}, i}$ is the maximum kinematically feasible speed.

---

### 2. Waypoint-Following DRL Controller
Using the optimized waypoints, I trained a **Proximal Policy Optimization (PPO)** agent to track the raceline under high-speed slip conditions.

#### Observation & Action Spaces
*   **Observation Space (50D)**: 36-beam downsampled LiDAR scan, vehicle speed, IMU telemetry (orientation, angular velocity, linear acceleration), and IPS position coordinates.
*   **Action Space (2D)**: Steering command $\delta \in [-1, 1]$ and throttle/braking command $a \in [-1, 1]$.

#### Reward Function
The agent is trained using a shaped reward function:

$$ R_{\text{total}} = R_{\text{speed}} + R_{\text{tracking}} - R_{\text{smooth}} + R_{\text{crash}} $$

*   **Dynamic Speed Tracking ($R_{\text{speed}}$)**: The target speed is mapped to the path's forward curvature:
    $$ v_{\text{target}} = \text{clip}(v_{\text{max}} - c \cdot \kappa_{\text{max}}, v_{\text{min}}, v_{\text{max}}) $$
    where $v_{\text{max}} = 4.0\text{ m/s}$, $v_{\text{min}} = 2.2\text{ m/s}$, and $c = 2.0$.
*   **Path Tracking ($R_{\text{tracking}}$)**: Rewards progress along waypoints while exponentially penalizing cross-track error ($d_{\perp}$).
*   **Steering Regularization ($R_{\text{smooth}}$)**: Penalizes jerky steering changes ($\Delta \delta$) to encourage smooth driving.
*   **Collision Penalty ($R_{\text{crash}}$)**: A penalty of $-30.0$ if the LiDAR distance drops below $0.25\text{m}$.

#### Waypoint-Following Demo
Below is a demonstration of the waypoint-following agent in action:

![Waypoint-Following RL Controller](/images/ppo_wp.gif)

---

### 3. Generalized Track-Agnostic DRL Controller
To make the agent generalize to unseen tracks, I developed a coordinate-free model that drives reactively using local sensors instead of global waypoints.

#### Observation & Action Spaces
*   **Observation Space (83D)**: 72-beam LiDAR scan, vehicle speed, and IMU telemetry. No global position (IPS) coordinates are used.
*   **Action Space (2D)**: Steering command $\delta \in [-1, 1]$ and throttle/braking command $a \in [-1, 1]$.

#### Reward Function
The agent is trained to maximize speed while maintaining safety:

$$ R_{\text{total}} = R_{\text{speed}} + R_{\text{wall}} - R_{\text{smooth}} - R_{\text{reverse}} + R_{\text{crash}} $$

*   **Speed Maximization ($R_{\text{speed}}$)**: Positive reward based on forward velocity ($3.0 \cdot v_{\text{ego}}$), with a $-2.0$ penalty if speed falls below $0.5\text{ m/s}$.
*   **Wall Avoidance ($R_{\text{wall}}$)**: Calibrated proximity penalties scaling up as LiDAR readings drop below safety thresholds ($0.55\text{m}$, $0.35\text{m}$, and $0.25\text{m}$).
*   **Smoothness & Reverse ($R_{\text{smooth}}, R_{\text{reverse}}$)**: Penalizes steering fluctuations and continuous reverse driving.

#### Generalized Controller Demo
Below is the generalized agent navigating an unseen track:

![Generalized Track-Agnostic RL Controller](/images/rl_general.gif)

---

### 4. ROS 2 Node Integration & Synchronization
To prevent observation lag during training, the Gymnasium `step()` loop runs synchronously with the simulator. It publishes the steering and throttle controls, resets a message flag, and spins the node until a fresh LiDAR scan is received:

```python
def step(self, action):
    # 1. Publish steering & throttle
    self.steering_publisher.publish(Float32(data=float(action[0])))
    self.throttle_publisher.publish(Float32(data=float(action[1])))
    self.new_lidar_received = False
    
    # 2. Synchronize: Spin until the next LiDAR callback runs
    while not self.new_lidar_received:
        rclpy.spin_once(self.node, timeout_sec=0.001)
        
    # 3. Return observation and reward
    reward = self._calculate_reward(action)
    return self.latest_observation.copy(), reward, self.is_done, False, {}
```

---

### Links & Code
*   **Repository**: [GitHub - mk04366/roboracer-autodrive](https://github.com/mk04366/roboracer-autodrive)
*   **My Planner Code**:
    *   [Waypoint RL Node](https://github.com/mk04366/roboracer-autodrive/blob/master/planner/rl_control/rl_control/rl_agent_node.py)
    *   [Generalized RL Node](https://github.com/mk04366/roboracer-autodrive/blob/master/planner/rl_control/rl_control/rl_general.py)
    *   [Offline Trajectory Optimizer](https://github.com/mk04366/roboracer-autodrive/tree/master/planner/global-planning)
