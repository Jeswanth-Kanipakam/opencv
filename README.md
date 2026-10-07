# Omokai — Digital Twin Training Simulation

## Project status

**Current validated milestone:** GPS-denied indoor visual reconstruction pipeline using smartphone imagery + HLoc/SfM.

The project started by investigating phone-based room reconstruction with ARCore and the `rumahku` Gaussian-splat pipeline. That approach successfully produced a capture dataset and Gaussian PLY output, but the rendered result was not reliable enough for the training-simulation use case.

The current approach separates **geometric reconstruction/localization** from **visual rendering**:

> Smartphone capture → visual feature matching → SfM reconstruction → metric alignment → future Gaussian Splatting → visual relocalization → target/proximity alert

The SfM stage is currently validated. Gaussian Splatting training remains the next unfinished stage.

---

## 1. Original objective

The client requirement is a low-cost, scalable training simulation for public-service environments.

The intended workflow is:

1. Use a normal smartphone to capture a room.
2. Build a digital representation of that room.
3. Allow a supervisor to store a target/hazard location.
4. Have a trainee enter the physical room without reliable GPS.
5. Relocalize the phone against the stored room representation.
6. Convert the phone pose into the room coordinate system.
7. Trigger a signal/beep when the trainee approaches the stored target.

GPS is not suitable as the primary indoor localization mechanism, so the key technical problem is **GPS-denied visual/spatial relocalization**.

A perfect CAD-quality model is not required for the MVP.

---

# 2. Approach 1 — ARCore + Rumahku

## Why this approach was investigated

`rumahku` provides a phone-first pipeline using:

- Android
- ARCore camera tracking
- ARCore feature points
- phone image capture
- Gaussian Splat reconstruction

Repository:

https://github.com/teapotlaboratories/rumahku

The Pixel 9a was used for the real room capture.

## What was successfully achieved

A real indoor room was captured using the Pixel 9a.

Dataset:

- 152 JPEG frames
- 1920 × 1080 images
- ARCore camera poses
- ARCore feature points
- `transforms.json`
- `seed.ply`
- `seed_features.ply`
- generated `rumahku_6000.ply`

The capture itself worked reliably enough to prove the phone-based acquisition concept.

## Problem encountered

The generated Gaussian-splat PLY was readable and contained approximately 291,685 Gaussian records, but rendering/inspection did not produce a sufficiently useful room representation.

The output appeared as a poor/smooth color representation rather than a convincing reconstruction of the room.

Several diagnostics were attempted, including camera-convention checks and filtering experiments, but the visual quality did not become reliable enough.

## Decision

Instead of continuing to debug the entire Rumahku reconstruction stack, the project was changed to a more modular pipeline.

The important lesson was:

> ARCore capture is useful, but ARCore poses should not be treated as the only source of reconstruction geometry.

---

# 3. Approach 2 — HLoc + SuperPoint + SuperGlue + SfM

## Why the approach changed

The new approach first establishes a reliable geometric reconstruction using established visual SfM techniques.

Pipeline:

```text
Room images
   ↓
SuperPoint feature extraction
   ↓
SuperGlue feature matching
   ↓
Geometric verification
   ↓
COLMAP / pycolmap SfM
   ↓
Sparse 3D reconstruction
   ↓
Camera poses
```

This separates the difficult geometry problem from the later rendering problem.

## Feature extraction

SuperPoint was used with:

- maximum keypoints: 4096
- NMS radius: 3
- grayscale images
- resize maximum: 1600 px

All 152 images successfully produced features.

## Feature matching

SuperGlue was used on sequential image pairs.

Current successful run:

- 152 input images
- 1,465 sequential pairs
- overlap window: 10
- SuperGlue matching completed successfully

## SfM reconstruction

HLoc/pycolmap was then used to construct the sparse 3D model.

### Validated result

| Metric | Result |
|---|---:|
| Input images | 152 |
| Registered images | **142** |
| 3D points | **29,666** |
| Observations | **214,175** |
| Mean track length | **7.22** |
| Mean observations/image | **1,508** |
| Mean reprojection error | **1.234 px** |

The reconstruction is therefore a genuine successful SfM result rather than only a feature-matching experiment.

The saved model is under:

```text
room_001/sfm/
```

---

# 4. SfM → Nerfstudio conversion

The next step was to prepare the validated SfM reconstruction for Gaussian Splatting.

The COLMAP/pycolmap camera poses were converted into Nerfstudio's `transforms.json` representation.

The conversion handled the camera transform convention explicitly:

```text
COLMAP camera-from-world
        ↓
rotation transpose
        ↓
camera-to-world transform
```

The reconstructed camera model was:

- `SIMPLE_RADIAL`
- image size: 1920 × 1080
- focal length: approximately 1410 px
- principal point: (960, 540)

The resulting dataset contains:

- 142 registered images
- 142 camera poses
- valid `transforms.json`

Directory:

```text
room_001/nerfstudio_data/
├── transforms.json
└── images/
    └── 142 registered images
```

The dataset was validated before attempting Gaussian Splat training.

---

# 5. Approach 3 — Nerfstudio / Splatfacto

## Intended next stage

The plan is now:

```text
Validated SfM
     ↓
Nerfstudio transforms.json
     ↓
Splatfacto
     ↓
Gaussian Splat room model
     ↓
visual inspection
     ↓
localization / proximity prototype
```

A Tesla T4 Colab environment was tested with:

- PyTorch 2.11.0 + CUDA 13.0
- CUDA available
- gsplat import successful
- Nerfstudio import successful

However, the first Nerfstudio installation was intentionally attempted with dependency resolution disabled and therefore lacked many runtime dependencies.

A second installation attempt revealed package-version conflicts in the Colab environment. Training was therefore **not completed**.

This is an environment/setup issue, not evidence that the SfM dataset is invalid.

The SfM reconstruction and converted dataset remain the current validated baseline.

---

# 6. Localization work

The final application requires more than a pretty 3D model.

The target workflow is:

```text
Reference room
     ↓
stored room coordinate frame
     ↓
stored target coordinate
     ↓

trainee enters room
     ↓
phone camera
     ↓
visual relocalization
     ↓
phone pose in room coordinates
     ↓
distance to target
     ↓
beep / proximity feedback
```

A visual localization prototype was also explored using the reconstructed SfM map.

An earlier held-out-frame experiment produced:

- 543 2D→3D correspondences
- 398 PnP RANSAC inliers
- 73.30% inlier ratio
- mean reprojection error approximately 1.87 px

This indicates that the reconstructed map can already support a meaningful GPS-denied localization experiment.

---

# 7. Why the current architecture is better

The major change is separating the project into independent stages:

### Capture

ARCore is used because it provides practical smartphone tracking and capture support.

### Geometry

HLoc + SuperPoint + SuperGlue + SfM establish a visual 3D coordinate system.

### Rendering

Gaussian Splatting is treated as a separate stage for creating a visually useful digital twin.

### Localization

The sparse/dense map can then be used for visual relocalization.

### Application

The Android application only needs the resulting room coordinate system and target location to calculate proximity.

This makes debugging much easier because a failure in rendering does not invalidate the geometric reconstruction.

---

# 8. Current workspace structure

Recommended shared structure:

```text
Omokai_Digital_Twin_Training_Simulation/
│
├── README.md
│
├── docs/
│   ├── approach-history.md
│   ├── architecture.md
│   ├── reconstruction.md
│   └── localization.md
│
├── colab/
│   ├── 01_capture_and_dataset.ipynb
│   ├── 02_hloc_sfm.ipynb
│   └── 03_sfm_to_nerfstudio.ipynb
│
├── data/
│   └── README.md
│
└── results/
    ├── sfm/
    └── nerfstudio_data/
```

Large datasets and trained models should normally remain in Google Drive rather than being committed to Git.

---

# 9. What Amarth should check

The most useful next validation is:

1. Download/open the SfM reconstruction.
2. Confirm that the 142-image reconstruction loads.
3. Inspect the sparse 3D points and camera trajectory.
4. Verify the generated `transforms.json`.
5. Run Gaussian Splatting using a clean, compatible environment.
6. Compare the resulting visual model with the original Rumahku result.
7. Continue toward visual relocalization and target proximity.

---

# 10. Current status

### Completed

- [x] Real room capture using Pixel 9a
- [x] 152-frame dataset
- [x] ARCore capture pipeline tested
- [x] Rumahku Gaussian reconstruction tested
- [x] Rumahku output inspected
- [x] SuperPoint feature extraction
- [x] SuperGlue matching
- [x] HLoc SfM reconstruction
- [x] 142/152 images registered
- [x] 29,666 sparse 3D points
- [x] 1.234 px mean reprojection error
- [x] SfM → Nerfstudio pose conversion
- [x] 142-image Nerfstudio dataset validated
- [x] Initial GPS-denied visual localization experiment

### In progress / next

- [ ] Clean Nerfstudio/Splatfacto environment
- [ ] Train Gaussian Splat model
- [ ] Inspect/render the dense room model
- [ ] Improve visual relocalization robustness
- [ ] Store supervisor target coordinates
- [ ] Implement phone-side proximity/direction feedback
- [ ] Android integration
- [ ] End-to-end physical-room test

---

# 11. Important interpretation

The project should currently be described as:

> **A validated smartphone indoor-capture and GPS-denied visual SfM reconstruction pipeline, with Gaussian Splatting and Android proximity localization as the next integration stages.**

It should **not** yet be described as a finished digital-twin training system.

The strongest current technical result is the successful 142-image SfM reconstruction and its use as a basis for visual localization.

---

## External references

- Rumahku: https://github.com/teapotlaboratories/rumahku
- HLoc: https://github.com/cvg/Hierarchical-Localization
- InLoc: https://openaccess.thecvf.com/content_cvpr_2018/html/Taira_InLoc_Indoor_Visual_CVPR_2018_paper.html
- 3D Gaussian Splatting / model-city reference: https://github.com/rakeshsuthar6322/3d_gaussian_splatting_for_model_city

