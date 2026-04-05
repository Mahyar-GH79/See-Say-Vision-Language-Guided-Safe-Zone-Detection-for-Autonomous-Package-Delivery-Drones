# See&Say: Vision–Language Guided Safe Zone Detection for Autonomous Package Delivery Drones

<p align="center">
  <img src="assets/teaser.png" alt="See&Say Overview" width="800"/>
</p>

<p align="center">
  <a href="https://urvis-workshop.github.io/">
    <img src="https://img.shields.io/badge/CVPR%202026-URVIS%20Workshop-blue?style=flat-square" alt="CVPR 2026 URVIS Workshop"/>
  </a>
  <img src="https://img.shields.io/badge/Code-Coming%20Soon-orange?style=flat-square" alt="Code Coming Soon"/>
  <img src="https://img.shields.io/badge/Dataset-Coming%20Soon-orange?style=flat-square" alt="Dataset Coming Soon"/>
</p>

---

> **Accepted at the [CVPR 2026 Workshop on Unified Robotic Vision with Cross-Modal Sensing and Alignment (URVIS)](https://urvis-workshop.github.io/)**

---

## Overview

Autonomous drone delivery is rapidly maturing, but reliably identifying **safe package drop zones** in cluttered urban and suburban environments remains an open challenge. Existing methods rely either on geometric depth analysis *or* semantic segmentation — but not both, leaving critical gaps in safety reasoning.

**See&Say** bridges this gap by combining:

- 🗺️ **Monocular depth gradients** for geometric flatness estimation
- 🔍 **Open-vocabulary segmentation** (DINO-X) for semantic hazard detection
- 🧠 **Vision–Language Model (VLM) reasoning** (GPT-o3) for iterative refinement and temporal safety assessment
- 🎯 **Preference-guided alternative zone selection** when the primary drop-pad is unsafe

The system operates entirely on monocular RGB input — no LiDAR or additional sensors required.

---

## How It Works

<p align="center">
  <img src="assets/pipeline.png" alt="See&Say Pipeline" width="800"/>
</p>

See&Say processes batches of **5 consecutive RGB frames** and their corresponding monocular depth maps through a multi-stage pipeline:

1. **Depth Estimation** — Depth-Anything V2 generates per-frame normalized depth maps.
2. **Gradient-Based Flatness** — A Gaussian-smoothed gradient filter identifies non-flat, geometrically unsafe regions.
3. **Open-Vocabulary Hazard Detection** — DINO-X segments hazardous objects using a prompt vocabulary initialized with common aerial-view categories (e.g., *person, car, truck*).
4. **Initial Safety Map Fusion** — The geometric mask and semantic mask are combined via a pixel-wise logical union.
5. **VLM Prompt Refinement (Agent 1)** — A VLM (GPT-o3) analyzes the 5-frame RGB+depth context and the initial safety overlay to:
   - Decide whether the **primary drop-pad is safe**
   - Provide a **scene description** and **future prediction**
   - **Refine the DINO-X prompt list** by adding scene-specific hazards (e.g., *coiled garden hose*, *wooden deck*, *soccer ball*)
6. **Refined Safety Map** — DINO-X re-runs with the updated prompts, and the final safety map is produced.
7. **Alternative Drop Zone Ranking (Agent 2)** — When the primary pad is unsafe, a second VLM agent ranks candidate zones (on a hexagonal grid) according to **user-defined preferences** (geometric or semantic).

---

## Key Results

### Primary Drop-Pad Safety Assessment

| Method | Success Rate | Reasoning |
|--------|:-----------:|:---------:|
| DINO-X (VisDrone categories) | 0.483 | ✗ |
| DINO-X (Complete categories) | 0.916 | ✗ |
| **See&Say — Single Frame** | **0.958** | ✓ |
| **See&Say — 5 Frames (Ours)** | **0.975** | ✓ |

Temporal context across 5 frames consistently boosts safety decision accuracy by leveraging motion cues and short-term dynamics.

---

### Safety Map Quality (IoU & Per-Pixel Metrics)

| Method | IoU ↑ | Dice/F1 ↑ | Recall ↑ | Accuracy ↑ | Bal. Acc. ↑ |
|--------|:-----:|:---------:|:--------:|:----------:|:-----------:|
| Gradient | 0.610 | 0.736 | 0.618 | 0.813 | 0.802 |
| RTDETR+SAM2+Gradient | 0.619 | 0.744 | 0.628 | 0.817 | 0.806 |
| YOLOv8+SAM2+Gradient | 0.619 | 0.744 | 0.627 | 0.817 | 0.806 |
| DINO-X+Gradient | 0.619 | 0.744 | 0.627 | 0.817 | 0.806 |
| Vanilla VLM (GPT-o3) | 0.535 | 0.687 | 0.684 | 0.746 | 0.732 |
| **See&Say (Ours)** | **0.797** | **0.880** | **0.868** | **0.886** | **0.880** |

See&Say achieves the highest IoU, Dice/F1, Recall, Accuracy, and Balanced Accuracy — demonstrating superior hazard coverage and overlap with human-annotated ground truth.

---

### Alternative Drop Zone Evaluation (AP & ROC-AUC)

| Method | AP @ η=95% ↑ | ROC @ η=95% ↑ | MAE @ η=95% ↓ |
|--------|:-----------:|:------------:|:------------:|
| Gradient | 0.741 | 0.724 | 0.151 |
| DINO-X (flat ground) | 0.614 | 0.649 | 0.122 |
| Vanilla VLM (GPT-o3) | 0.633 | 0.560 | 0.149 |
| **See&Say (Ours)** | **0.969** | **0.933** | **0.036** |

See&Say achieves the lowest MAE and highest AP/ROC-AUC across **all** safety thresholds (η ∈ {0.95, 0.90, 0.85, 0.80}).

---

### Human Preference Evaluation

See&Say was evaluated by two independent reviewers for preference adherence (score 1–3) across three scenes with geometric and semantic preferences:

| Scene | Preference | Avg. R1 | Avg. R2 | Cohen's κ |
|-------|-----------|:-------:|:-------:|:---------:|
| Scene 1 | Close to the parking door | 2.25 | 2.12 | 0.810 |
| Scene 1 | Avoid organic/vegetation zones | 2.12 | 2.12 | 1.000 |
| Scene 2 | Top-right corner of the scene | 3.00 | 3.00 | 1.000 |
| Scene 2 | Furthest from all detected people | 2.88 | 2.75 | 0.600 |
| Scene 3 | Closest flat zone to H-pad | 2.75 | 2.62 | 0.714 |
| Scene 3 | Away from activity | 2.75 | 2.75 | 0.333 |

The VLM reasoning component reliably internalizes a diverse range of human preferences, with particularly strong performance on spatially explicit and semantic constraints.

---

## Dataset

We curated a custom dataset of **3 real-world suburban scenes** (Fairfax, VA, USA) covering front and back yards with:

- 🧍 Dynamic human activity and moving objects
- 🌦️ Varying lighting (shadow, direct sunlight)
- 🌿 Diverse surfaces (grass, pavement, decks, stairs, flower beds)
- 🅗 Physical drop pads marked with an "H" symbol

**120 frames** (24 batches × 5 frames) were manually annotated by a human expert for pixel-level safety maps.

---

## Citation

If you find this work useful, please consider citing:

```bibtex
@inproceedings{seesay2026,
  title     = {See\&Say: Vision Language Guided Safe Zone Detection for Autonomous Package Delivery Drones},
  author    = {Anonymous},
  booktitle = {Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR) Workshops},
  year      = {2026}
}
```

---

## 🚧 Code & Dataset — Coming Soon

The full codebase, dataset, and all supplementary materials (including VLM prompts and hyperparameter configurations) will be publicly released with the camera-ready version of the paper.

Stay tuned!

---

<p align="center">
  Accepted at the <a href="https://urvis-workshop.github.io/">CVPR 2026 URVIS Workshop</a>
</p>
