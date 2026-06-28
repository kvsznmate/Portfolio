# Task 11: RL Controller for Robotic Pipette Positioning

This project implements a Reinforcement Learning (RL) controller using Proximal Policy Optimization (PPO) for precise pipette positioning in a PyBullet simulation environment, with comparison to a PID baseline controller.

## Table of Contents

- [Implementation Steps and Design Choices](#implementation-steps-and-design-choices)
- [Tuning Strategies](#tuning-strategies)
- [Libraries Used](#libraries-used)
- [Performance Metrics and Error Analysis](#performance-metrics-and-error-analysis)
- [Comparison: RL vs PID Controller](#comparison-rl-vs-pid-controller)
- [Best Hyperparameters](#best-hyperparameters)
- [Saved Model Weights](#saved-model-weights)

---

## Implementation Steps and Design Choices

### System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         RL CONTROL PIPELINE                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐           │
│  │  Observation  │───▶│   PPO Policy  │───▶│    Action     │           │
│  │   (9D state)  │    │  (MlpPolicy)  │    │  (3D velocity)│           │
│  └───────────────┘    └───────────────┘    └───────────────┘           │
│         ▲                                          │                    │
│         │            ┌───────────────┐             │                    │
│         └────────────│  Simulation   │◀────────────┘                    │
│                      │   (PyBullet)  │                                  │
│                      └───────────────┘                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Design Choice 1: Custom Gymnasium Environment

A custom environment (`OT2Env`) wraps the PyBullet simulation:

```python
class OT2Env(gym.Env):
    def __init__(self, render=False, max_steps=1000):
        super(OT2Env, self).__init__()
        
        self.sim = Simulation(num_agents=1, render=self.render)
        
        # Action space: 3D continuous velocities
        self.action_space = spaces.Box(
            low=np.array([-1, -1, -1]), 
            high=np.array([1, 1, 1]), 
            shape=(3,), 
            dtype=np.float32
        )
        
        # Observation space: 9D vector
        self.observation_space = spaces.Box(
            low=-np.inf, high=np.inf, 
            shape=(9,), dtype=np.float32
        )
```

### Design Choice 2: Observation Space (9D)

The observation vector provides complete spatial awareness:

```python
observation = np.concatenate([
    pipette_pos,       # Current position [x, y, z]     (3D)
    self.goal_position, # Goal position [x, y, z]       (3D)
    diff_vector        # Direction to goal [dx, dy, dz] (3D)
], axis=0)
```

| Component | Dimensions | Description |
|-----------|------------|-------------|
| `pipette_pos` | 3 | Current pipette tip position (m) |
| `goal_position` | 3 | Target position (m) |
| `diff_vector` | 3 | Vector from pipette to goal (m) |

### Design Choice 3: Action Space (3D Continuous)

Actions are continuous velocity commands in the range [-1, 1]:

```python
# Action format: [velocity_x, velocity_y, velocity_z]
sim_action = list(action) + [0]  # Append drop=0
self.sim.run([sim_action])
```

### Design Choice 4: Reward Function

The reward function balances goal-reaching with smooth control:

```python
# Distance penalty (main learning signal)
reward = -10.0 * distance

# Action penalty (encourages smooth movements)
reward -= 0.1 * np.linalg.norm(action)

# Success bonus (goal reached within 5mm)
if distance < 0.005:
    reward += 100.0
    terminated = True
```

| Reward Component | Value | Purpose |
|-----------------|-------|---------|
| Distance penalty | `-10.0 × distance` | Drive toward goal |
| Action penalty | `-0.1 × ‖action‖` | Smooth trajectories |
| Success bonus | `+100.0` | Encourage goal completion |

### Design Choice 5: Workspace Boundaries

Boundaries derived from environment exploration (Task 9):

```python
# Workspace limits (meters)
self.x_min, self.x_max = -0.1870, 0.2530
self.y_min, self.y_max = -0.1705, 0.2195
self.z_min, self.z_max = 0.1695, 0.2895
```

---

## Tuning Strategies

### PPO Hyperparameter Selection

The training configuration was chosen for stable learning on CPU:

```python
model = PPO(
    policy="MlpPolicy",
    env=env,
    learning_rate=3e-4,      # Standard PPO learning rate
    n_steps=2048,            # Steps before policy update
    batch_size=64,           # Mini-batch size
    n_epochs=10,             # Epochs per update
    gamma=0.99,              # Discount factor (long-horizon)
    gae_lambda=0.95,         # GAE parameter
    clip_range=0.2,          # PPO clipping
    device="cpu",
    tensorboard_log=f"runs/{run.id}"
)
```

### Hyperparameter Rationale

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `learning_rate` | 3e-4 | Standard stable rate for PPO |
| `n_steps` | 2048 | Sufficient trajectory length for credit assignment |
| `batch_size` | 64 | Balance between stability and speed |
| `n_epochs` | 10 | Multiple passes improve sample efficiency |
| `gamma` | 0.99 | High discount for goal-reaching tasks |
| `gae_lambda` | 0.95 | Standard GAE parameter |
| `clip_range` | 0.2 | Prevents large policy updates |

### Reward Tuning

The reward weights were tuned iteratively:

| Version | Distance Weight | Action Weight | Success Bonus | Result |
|---------|----------------|---------------|---------------|--------|
| Initial | -1.0 | 0 | +10 | Slow learning |
| v2 | -10.0 | 0 | +100 | Overshoots |
| **Final** | **-10.0** | **-0.1** | **+100** | **Smooth + accurate** |

### Training Monitoring

Training was tracked using Weights & Biases:

```python
run = wandb.init(
    project="ot2_rl_movement",
    name="v1_cpu_baseline",
    sync_tensorboard=True,
    config={
        "algorithm": "PPO",
        "total_timesteps": 1000000,
        "learning_rate": 3e-4,
        # ...
    }
)
```

---

## Libraries Used

### Core Dependencies

| Library | Version | Purpose |
|---------|---------|---------|
| `stable-baselines3` | 2.x | PPO algorithm implementation |
| `gymnasium` | 0.29+ | RL environment interface |
| `pybullet` | - | Physics simulation |
| `numpy` | - | Numerical operations |
| `torch` | 2.x | Neural network backend |
| `wandb` | - | Experiment tracking |
| `tensorflow` | 2.x | ML model for root detection |

### Installation

```bash
pip install stable-baselines3 gymnasium pybullet numpy torch wandb tensorflow
```

### Import Structure

```python
# RL Components
from stable_baselines3 import PPO
from stable_baselines3.common.monitor import Monitor
from wandb.integration.sb3 import WandbCallback
import gymnasium as gym
from gymnasium import spaces

# Simulation
from sim_class import Simulation

# Vision Pipeline
from tensorflow.keras.models import load_model
from skimage.morphology import skeletonize
```

---

## Performance Metrics and Error Analysis

### RL Controller Performance

| Metric | Value |
|--------|-------|
| Position Threshold | 1.0 mm |
| Goal Tolerance (training) | 5.0 mm |
| Max Iterations | 2000 |
| Training Timesteps | 1,000,000 |

### Controller Comparison

```python
# RL Controller usage
def RL_controller(env, model, obs, Destination_x, Destination_y, Destination_z):
    position_threshold = 0.001  # 1mm
    max_iterations = 2000
    
    while iteration_count < max_iterations:
        action, _states = model.predict(obs, deterministic=True)
        obs, rewards, terminated, truncated, info = env.step(action)
        
        if error < position_threshold:
            # Drop inoculum
            drop_action = np.array([0, 0, 0, 1])
            env.step(drop_action)
            break
```

### Error Analysis

| Controller | Threshold | Typical Error | Convergence |
|------------|-----------|---------------|-------------|
| **RL (PPO)** | 1.0 mm | < 1.0 mm | Variable |
| **PID (P-only)** | 0.5 mm | ~0.01 mm | Consistent |

### Training Metrics

| Metric | Value |
|--------|-------|
| Total timesteps | 1,000,000 |
| Episode length (trained) | ~50-200 steps |
| Success rate (final) | >90% within 5mm |

---

## Comparison: RL vs PID Controller

### Architecture Comparison

| Aspect | RL Controller (PPO) | PID Controller |
|--------|---------------------|----------------|
| **Control Type** | Learned policy | Handcrafted |
| **State Input** | 9D observation | 2D error |
| **Action Output** | 3D continuous | 2D velocity |
| **Parameters** | Neural network weights | 3 gains (Kp, Ki, Kd) |

### Performance Comparison

| Metric | RL Controller | PID Controller |
|--------|---------------|----------------|
| **Position Threshold** | 1.0 mm | 0.5 mm |
| **Typical Final Error** | < 1.0 mm | ~0.01 mm |
| **Convergence Speed** | Variable | Consistent |
| **3D Control** | Yes | No (X/Y only) |

### Development Comparison

| Aspect | RL Controller | PID Controller |
|--------|---------------|----------------|
| **Development Time** | Days (training) | Hours (tuning) |
| **Compute Required** | CPU/GPU intensive | Minimal |
| **Tuning Method** | Hyperparameter search | Manual iteration |
| **Iterations to Deploy** | 1M+ timesteps | 4 iterations |

### Pros and Cons

#### RL Controller (PPO)

**Pros:**
- Handles 3D positioning (X, Y, Z)
- Can learn complex behaviors
- Adapts to environment dynamics
- No explicit model required

**Cons:**
- Requires extensive training
- Less precise than PID
- Black-box decision making
- Training can be unstable

#### PID Controller

**Pros:**
- High precision (~0.01mm)
- Fast to develop and tune
- Interpretable parameters
- Consistent performance
- Minimal compute

**Cons:**
- Limited to 2D (X/Y)
- Fixed parameters
- Requires system knowledge
- No adaptation

### When to Use Each

| Scenario | Recommended Controller |
|----------|----------------------|
| High precision required (<0.1mm) | PID |
| 3D positioning needed | RL |
| Fast deployment | PID |
| Complex dynamics | RL |
| Limited compute | PID |
| Research/exploration | RL |

---

## Best Hyperparameters

### RL Controller (PPO)

```python
# Training Configuration
PPO_CONFIG = {
    "policy": "MlpPolicy",
    "learning_rate": 3e-4,
    "n_steps": 2048,
    "batch_size": 64,
    "n_epochs": 10,
    "gamma": 0.99,
    "gae_lambda": 0.95,
    "clip_range": 0.2,
    "device": "cpu"
}

# Environment Configuration
ENV_CONFIG = {
    "max_steps": 1000,
    "goal_tolerance": 0.005,  # 5mm for training
    "distance_weight": -10.0,
    "action_weight": -0.1,
    "success_bonus": 100.0
}

# Inference Configuration
INFERENCE_CONFIG = {
    "position_threshold": 0.001,  # 1mm
    "max_iterations": 2000,
    "deterministic": True
}
```

### PID Controller (for comparison)

```python
PID_CONFIG = {
    "Kp": 30.0,
    "Ki": 0.0,
    "Kd": 0.0,
    "dt": 0.1,
    "position_threshold": 0.0005,  # 0.5mm
    "max_iterations": 2000
}
```

---

## Saved Model Weights

### RL Model

| Property | Value |
|----------|-------|
| **File** | `models/model_v1_cpu.zip` |
| **Algorithm** | PPO |
| **Policy** | MlpPolicy |
| **Training Steps** | 1,000,000 |
| **Device** | CPU |

### Loading the Model

```python
from stable_baselines3 import PPO

# Load trained model
model_path = "models/model_v1_cpu.zip"
model = PPO.load(model_path)

# Use for inference
action, _states = model.predict(observation, deterministic=True)
```

### Model Architecture

```
MlpPolicy:
├── Feature Extractor: Flatten
├── Policy Network: MLP [64, 64]
├── Value Network: MLP [64, 64]
└── Action Distribution: Gaussian (continuous)
```

### Computer Vision Model

| Property | Value |
|----------|-------|
| **File** | `best_model_f1.h5` |
| **Architecture** | U-Net |
| **Input** | 256×256 grayscale patches |
| **Output** | Binary segmentation mask |

---

## Files

| File | Description |
|------|-------------|
| `rl_env.py` | Custom Gymnasium environment for OT2 robot |
| `rl_train.py` | PPO training script with W&B logging |
| `rl_pipeline.py` | Full pipeline: CV detection + RL control |
| `models/model_v1_cpu.zip` | Trained PPO model weights |
| `best_model_f1.h5` | U-Net model for root segmentation |

---

## Usage

### Training a New Model

```bash
python rl_train.py
```

This will:
1. Create the OT2 environment
2. Initialize PPO with configured hyperparameters
3. Train for 1M timesteps with W&B logging
4. Save model to `models/model_v1_cpu.zip`

### Running the Full Pipeline

```bash
python rl_pipeline.py
```

This will:
1. Load trained RL model
2. Capture plate image
3. Run ML prediction for root detection
4. Move pipette to each root tip using RL controller
5. Drop inoculum at each location

### Using the RL Controller Directly

```python
from stable_baselines3 import PPO
from rl_env import OT2Env

# Setup
env = OT2Env(render=False)
model = PPO.load("models/model_v1_cpu.zip")
obs, info = env.reset()

# Control loop
target = [0.15, 0.10, 0.17]  # x, y, z
env.goal_position = np.array(target)

while True:
    action, _ = model.predict(obs, deterministic=True)
    obs, reward, terminated, truncated, info = env.step(action)
    
    if info['distance'] < 0.001:  # 1mm threshold
        print("Goal reached!")
        break

env.close()
```