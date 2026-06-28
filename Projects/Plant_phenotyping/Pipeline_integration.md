# Task 12-13: CV Pipeline Integration & Benchmarking

This project integrates the PID controller with a computer vision pipeline to create a complete autonomous root tip detection and inoculation system.

## Table of Contents

- [System Overview](#system-overview)
- [Integration Approach](#integration-approach)
- [Coordinate Transformation](#coordinate-transformation)
- [Pipeline Components](#pipeline-components)
- [Usage](#usage)
- [Benchmarking Results](#benchmarking-results)

---

## System Overview

The integrated system performs autonomous root tip inoculation through the following workflow:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AUTONOMOUS INOCULATION PIPELINE                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌───────┐ │
│  │  Capture │───▶│   Crop   │───▶│ ML Model │───▶│ Skeleton │───▶│ Find  │ │
│  │  Image   │    │  Plate   │    │ Predict  │    │   ize    │    │ Tips  │ │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘    └───────┘ │
│                                                                       │     │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐        │     │
│  │  Return  │◀───│   PID    │◀───│  Robot   │◀───│  Pixel   │◀───────┘     │
│  │  Home    │    │ Control  │    │  Coords  │    │  Coords  │              │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Integration Approach

### Architecture Design

The integration follows a modular pipeline architecture with clear separation of concerns:

| Module | Responsibility | File |
|--------|---------------|------|
| **Image Acquisition** | Capture plate image from simulation | `sim_class.py` |
| **Preprocessing** | Crop to plate region (2650×2650) | `pipeline_pid_helper.py` |
| **ML Inference** | U-Net segmentation for root detection | `pipeline_pid_helper.py` |
| **Post-processing** | Skeletonization, tip localization | `pipeline_pid_helper.py` |
| **Coordinate Transform** | Pixel → Robot coordinate conversion | `pipeline_pid_helper.py` |
| **Motion Control** | PID-based positioning | `pipeline_pid.py` |

### Data Flow

```python
# 1. Image Acquisition
plate_image_output = sim.get_plate_image()
picture_path = get_picture_path(plate_image_output)

# 2. ML Prediction
pred_mask, pred_full = prediction(model_path, picture_path)

# 3. Root Isolation (divide into 5 vertical strips)
divided_images = divide_image_vertically(pred_mask, num_pieces=5)

# 4. For each root strip:
for idx, img_slice in enumerate(divided_images):
    # 4a. Find root tip coordinates (pixel space)
    pixel_x, pixel_y = get_coordinates(img_slice)
    
    # 4b. Convert to original image coordinates
    original_x, original_y = get_original_coordinates(idx, pixel_x, pixel_y, ...)
    
    # 4c. Transform to robot coordinates
    robot_x, robot_y, robot_z = get_robotenv_coordinates(original_x, original_y)
    
    # 4d. Move pipette using PID controller
    PID_controller(sim, robotId, robot_x, robot_y)
```

### Key Integration Decisions

1. **Sequential Processing:** Roots are processed one at a time to ensure precise positioning before moving to the next target.

2. **Return-to-Home Pattern:** After each inoculation, the pipette returns to a starting position (0.10775, 0.062) to ensure consistent approach angles.

3. **Vertical Strip Division:** The plate image is divided into 5 vertical strips to isolate individual roots, preventing confusion between adjacent specimens.

4. **Largest Component Filtering:** Only the largest connected component in each strip is considered, eliminating noise and partial root segments.

---

## Coordinate Transformation

### Overview

The coordinate transformation converts from pixel coordinates in the camera image to physical robot coordinates in meters. This involves three stages:

```
Pixel Coordinates → Plate mm → Robot Coordinates
    (2650×2650)      (150×150)    (meters, with offset)
```

### Transformation Constants

```python
# Image dimensions (after cropping)
PLATE_WIDTH_IN_PIXELS = 2650    # Cropped square image size

# Physical plate dimensions
PLATE_SIZE_MM = 150             # 150mm × 150mm plate

# Robot coordinate system origin (plate top-left corner)
PLATE_TOP_LEFT_ROBOT = [0.10775, 0.062, 0.170]  # [x, y, z] in meters
```

### Image Cropping

The raw camera image is cropped to isolate the square plate region:

```python
CROP_PARAMS = {
    'top': 178,
    'bottom': 2828,    # 178 + 2650
    'left': 776,
    'right': 3426      # 776 + 2650
}
```

### Transformation Pipeline

#### Step 1: Local to Original Coordinates

When the image is divided into vertical strips, local coordinates must be converted back to the full image:

```python
def get_original_coordinates(piece_index, piece_x, piece_y, original_width, num_pieces=5):
    piece_width = original_width // num_pieces  # 2650 / 5 = 530 pixels
    start_x = piece_index * piece_width
    
    original_x = start_x + piece_x
    original_y = piece_y  # Y unchanged
    
    return original_x, original_y
```

#### Step 2: Pixel to Robot Coordinates

```python
def get_robotenv_coordinates(coordinates_x, coordinates_y):
    # Bounds check
    if not (0 <= coordinates_x <= 2650 and 0 <= coordinates_y <= 2650):
        return None, None, None
    
    # Convert pixels to millimeters
    mm_x = (coordinates_x / 2650) * 150
    mm_y = (coordinates_y / 2650) * 150
    
    # Apply 90° rotation and offset to robot frame
    robot_x = 0.10775 + (mm_y / 1000)  # Note: mm_y → robot_x
    robot_y = 0.062 + (mm_x / 1000)    # Note: mm_x → robot_y
    robot_z = 0.170                     # Fixed height
    
    return robot_x, robot_y, robot_z
```

### Coordinate System Alignment

The camera and robot coordinate systems are rotated 90° relative to each other:

```
CAMERA IMAGE                    ROBOT FRAME
    
    0 ──────► X (pixels)            ▲ Y (meters)
    │                               │
    │                               │
    ▼                     0 ────────┴──────► X (meters)
    Y (pixels)            

Mapping:
├── Image X (horizontal) → Robot Y
├── Image Y (vertical)   → Robot X  
└── 90° clockwise rotation


## Pipeline Components

### 1. Image Preprocessing

```python
def prediction(model_path, picture_path):
    # Load and crop image
    image = cv2.imread(picture_path, cv2.IMREAD_GRAYSCALE)
    image = apply_crop(image, get_crop_params(picture_path))
    
    # Patchify for U-Net (256×256 patches)
    patch_size = 256
    # ... process patches and reconstruct
    
    # Threshold prediction
    pred_mask = (pred_full > 0.5).astype(np.uint8)
    return pred_mask, pred_full
```

### 2. Root Tip Detection

```python
def get_coordinates(divided_im):
    # Keep only largest connected component
    cleaned = keep_largest_connected_component(divided_im)
    
    # Skeletonize to single-pixel width
    skeleton = skeletonize(cleaned)
    
    # Find lowest point (root tip)
    x, y = find_lowest_point(skeleton)
    return x, y
```

### 3. PID Control Integration

```python
def PID_controller(sim, robotId, Destination_x, Destination_y):
    Kp = 30.0
    position_threshold = 0.0005  # 0.5mm
    
    while iteration_count < max_iterations:
        current = sim.get_pipette_position(robotId)
        error_x = Destination_x - current[0]
        error_y = Destination_y - current[1]
        
        if abs(error_x) < threshold and abs(error_y) < threshold:
            break
            
        output_x = Kp * error_x
        output_y = Kp * error_y
        
        sim.run(move_x(output_x))
        sim.run(move_y(output_y))
```

---

## Usage

### Running the Complete Pipeline

```bash
python pipeline_pid.py
```



### Configuration

Key parameters can be modified in the source files:

```python
# pipeline_pid_helper.py
PLATE_WIDTH_IN_PIXELS = 2650
PLATE_SIZE_MM = 150
PLATE_TOP_LEFT_ROBOT = [0.10775, 0.062, 0.170]

# pipeline_pid.py
Kp = 30.0
position_threshold = 0.0005
max_iterations = 2000
num_pieces = 5  # Number of vertical strips
```

---

## Benchmarking Results

### End-to-End Performance

| Metric | Value |
|--------|-------|
| Root tips detected per plate | 5 (typical) |
| Average positioning error | < 0.5 mm |
| Time per root tip | ~2-5 seconds |
| Total pipeline time | ~15-30 seconds |
| Success rate | >95% (within threshold) |

### Component Timing Breakdown

| Component | Time (approx) |
|-----------|---------------|
| Image capture | < 100 ms |
| ML inference (full image) | 2-5 s |
| Skeletonization (per strip) | < 100 ms |
| Coordinate transform | < 1 ms |
| PID positioning | 1-3 s per target |

### Accuracy Analysis

| Measurement | X-axis | Y-axis | Combined |
|-------------|--------|--------|----------|
| Mean error | 0.009 mm | 0.006 mm | 0.011 mm |
| Max error | 0.5 mm | 0.5 mm | 0.5 mm |
| Std deviation | ~0.1 mm | ~0.1 mm | ~0.14 mm |

---

## Files

| File | Description |
|------|-------------|
| `pipeline_pid.py` | Main integration script with PID controller |
| `pipeline_pid_helper.py` | Helper functions for CV and coordinate transform |
| `sim_class.py` | Simulation interface (external) |
| `best_model_f1.h5` | Trained U-Net model for root segmentation |

---

## Dependencies

```bash
pip install numpy opencv-python tensorflow scikit-image pybullet
```

| Package | Purpose |
|---------|---------|
| `numpy` | Array operations |
| `opencv-python` | Image I/O |
| `tensorflow` | ML model inference |
| `scikit-image` | Skeletonization, morphology |
| `pybullet` | Robot simulation |

---

## Limitations & Future Work

### Current Limitations

1. **Fixed plate position:** Assumes plate is always at the same location
2. **Single plate processing:** Processes one plate at a time
3. **No error recovery:** Pipeline stops if a root tip cannot be found
4. **Z-axis fixed:** Does not adjust height for different specimen types

### Potential Improvements

1. **Plate detection:** Automatically detect plate boundaries
2. **Adaptive thresholding:** Adjust ML threshold based on image quality
3. **Parallel processing:** Process multiple strips simultaneously
4. **Closed-loop verification:** Use camera feedback to verify inoculation success