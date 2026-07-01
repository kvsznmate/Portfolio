# Autonomous Root Tip Inoculation System

An end-to-end computer vision and robotics pipeline for automated inoculation of plant root tips in a PyBullet simulation environment.

---

## Project Overview

This project automates the inoculation of plant root tips growing on petri dishes. A camera captures an image of the dish, a deep learning model identifies and segments the roots, and a robotic pipette autonomously navigates to the tip of each root to deliver inoculum — all without human intervention.

The system bridges computer vision, coordinate geometry, and robot control into a single deployable pipeline.

![pybullet simulation](PID_controller.gif)

---

## Business Objective

In plant biology research, inoculating root tips with bacteria or fungi is a labour-intensive, manual process that requires trained technicians and introduces human variability. Automating this workflow enables:

- **Higher throughput**  process multiple plates without operator fatigue
- **Reproducibility**  consistent inoculation position across all specimens
- **Scalability**  run experiments in parallel or overnight unattended
- **Data quality**  sub-millimetre precision reduces experimental error

---

## The objectives

I was responsible for the full development of this pipeline, including:

- **Computer vision pipeline**  integrating and train a U-Net model to segment root structures from grayscale images of petri dishes, followed by skeletonisation and tip localisation to find the exact end point of each root
- **Coordinate transformation**  designing the pixel-to-robot coordinate mapping, accounting for the 90° rotational offset between the camera frame and the robot frame, and scaling from a 2650×2650 pixel image to physical millimetre coordinates
- **PID controller**  iteratively developing and tuning a proportional controller for precise 2D pipette positioning, achieving a typical error of ~0.05 mm against a 0.5 mm tolerance threshold
- **RL controller (experimental)**  implementing reinforcement learning agent using Stable-Baselines3 as an alternative controller, capable of 3D positioning; compared against the PID baseline and evaluated for production suitability
- **Full pipeline integration**  connecting all components into a single autonomous loop: image capture → ML inference → tip detection → coordinate transform → robot control → inoculation → return home

---


![predicted root tips](task_5_inference_overlay.png)
Predicted root tips (green) overlaid on the original image of a petri dish.


## How It Works

```
Image Capture → Crop Plate → U-Net Segmentation → Skeletonisation → Find Root Tips
                                                                           │
Return Home ← PID Control ← Robot Coordinates ← Pixel-to-Robot Transform ┘
```

The plate image is divided into 5 vertical strips to isolate individual roots. For each strip, the largest connected component is extracted, skeletonised, and the lowest point (root tip) is identified. Pixel coordinates are converted to robot-frame metres and the PID controller moves the pipette to within 0.5 mm of the target before dispensing.

---

## Controllers

### PID Controller (Production)

A pure proportional controller (P-only) was chosen for the final system after 4 tuning iterations. The integral and derivative terms were disabled  the simulation provides clean, noise-free feedback, making them unnecessary and a source of instability.

| Parameter | Value |
|-----------|-------|
| Kp | 30.0 |
| Position threshold | 0.5 mm |
| Typical error | ~0.01 mm |
| Max iterations | 2000 |

### RL Controller (Experimental)

A PPO agent was trained for 1 million timesteps as an alternative approach. While it demonstrated 3D positioning capability and >90% success within 5 mm, it was less precise and less consistent than the PID controller for this task.

| Metric | RL (PPO) | PID |
|--------|----------|-----|
| Threshold | 1.0 mm | 0.5 mm |
| Typical error | < 1.0 mm | ~0.01 mm |
| 3D control | ✓ | ✗ |
| Training required | 1M steps | None |

---

## Tech Stack

| Category | Tools |
|----------|-------|
| Simulation | PyBullet |
| Deep Learning | TensorFlow / Keras (U-Net) |
| RL | Stable-Baselines3, Gymnasium |
| Image Processing | OpenCV, scikit-image |
| Experiment Tracking | Weights & Biases |
| Language | Python |

---

## Results

| Metric | Value |
|--------|-------|
| Root tips detected per plate | 5 |
| Mean positioning error | ~0.05 mm |
| Success rate | > 95% |
| Total pipeline time | 15–30 seconds |

---



## Repository Structure

```
├── pipeline_pid.py          # Main pipeline script
├── pipeline_pid_helper.py   # CV + coordinate transform utilities
├── rl_env.py                # Custom Gymnasium environment
├── rl_train.py              # PPO training script
├── rl_pipeline.py           # Full pipeline with RL controller
├── sim_class.py             # PyBullet simulation interface
├── best_model_f1.h5         # Trained U-Net weights
└── models/
    └── model_v1_cpu.zip     # Trained PPO model weights
```
