# VisionGuard — Video-Based Intrusion Detection System

> A Python + OpenCV system that detects unauthorized entry into a restricted zone in pre-recorded video footage and logs all intrusion events with timestamps and snapshots.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Project Structure](#project-structure)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Usage](#usage)
- [Parameters](#parameters)
- [Output](#output)
- [How It Works](#how-it-works)
- [Sample Results](#sample-results)

---

## Overview

VisionGuard solves a real-world security problem: **How do you automatically detect when someone enters a restricted area in recorded footage without expensive hardware or cloud services?**

The system uses **MOG2 background subtraction** (an adaptive Gaussian mixture model) to separate moving objects from the static scene. When a detected object overlaps with a user-defined **Region of Interest (ROI)**, an intrusion event is recorded.

This project was built as part of a computer vision course capstone (BYOP).

---

## Features

- Background subtraction using OpenCV's MOG2 algorithm
- User-defined Region of Interest (ROI) — works with any zone in any video
- Contour filtering to suppress noise (lighting changes, small animals, leaves)
- Bounding boxes drawn around detected intruders
- Timestamped intrusion log saved as CSV
- JPEG snapshot saved for every intrusion event
- Fully annotated output video with HUD overlay
- Configurable sensitivity (threshold, min area, cooldown)
- Built-in sample video generator for testing

---

## Project Structure

```
VisionGuard/
├── src/
│   ├── detector.py          # Main detection pipeline
│   ├── utils.py             # Drawing, logging, ROI overlap helpers
│   ├── generate_sample.py   # Synthetic test video generator
│   └── report.py            # Log reader and Markdown report generator
├── sample_data/             # Place your input videos here
├── output/
│   ├── annotated_output.mp4 # Processed video with annotations
│   ├── intrusion_log.csv    # Event log (frame, timestamp, objects, snapshot)
│   └── snapshots/           # JPEG frames captured at each alert
├── requirements.txt
└── README.md
```

---

## Installation

**Prerequisites:** Python 3.8 or higher

```bash
# 1. Clone the repository
git clone https://github.com/YOUR_USERNAME/VisionGuard.git
cd VisionGuard

# 2. (Recommended) Create a virtual environment
python -m venv venv
source venv/bin/activate        # On Windows: venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt
```

---

## Quick Start

**Step 1 — Generate a test video** (if you don't have real footage):

```bash
python src/generate_sample.py --output sample_data/test_video.mp4
```

**Step 2 — Run detection:**

```bash
python src/detector.py \
    --input sample_data/test_video.mp4 \
    --roi 200 200 240 200
```

**Step 3 — View results:**

```
output/
├── annotated_output.mp4       # Open this in any video player
├── intrusion_log.csv          # Open in Excel / any text editor
└── snapshots/                 # JPEG images of each intrusion
```

**Step 4 — Generate a summary report:**

```bash
python src/report.py --log output/intrusion_log.csv --output output/report.md
```

---

## Usage

### With your own video

```bash
python src/detector.py \
    --input /path/to/your/footage.mp4 \
    --roi X Y WIDTH HEIGHT
```

**Finding your ROI coordinates:**
Open the video in VLC or any player, pause on a frame, and note the pixel coordinates of the zone you want to monitor. The ROI is defined as:
- `X Y` — top-left corner of the zone
- `WIDTH HEIGHT` — dimensions of the zone

Example for a doorway at pixel position (300, 150) that is 120px wide and 250px tall:
```bash
--roi 300 150 120 250
```

### Adjusting sensitivity

```bash
# More sensitive (detects smaller / slower movement)
python src/detector.py --input video.mp4 --roi 200 200 240 200 \
    --min-area 400 --threshold 30

# Less sensitive (ignore minor movement, focus on large objects)
python src/detector.py --input video.mp4 --roi 200 200 240 200 \
    --min-area 1500 --threshold 70
```

---

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--input` | required | Path to input video file |
| `--roi X Y W H` | required | Region of Interest coordinates |
| `--output-dir` | `output/` | Directory for results |
| `--min-area` | `800` | Minimum contour area (pixels²) to consider as motion |
| `--threshold` | `50` | MOG2 variance threshold (higher = less sensitive) |
| `--history` | `500` | Frames used to model the background |
| `--overlap` | `0.3` | Minimum fraction of box overlapping ROI to trigger alert |
| `--cooldown` | `3.0` | Seconds between consecutive alert log entries |

---

## Output

### `annotated_output.mp4`
The original video with:
- Green ROI border (turns red during intrusion)
- Orange bounding boxes around detected objects
- HUD showing frame count and total events
- "!! ALERT !!" banner when intrusion is active

### `intrusion_log.csv`
```
frame,timestamp,num_detections,snapshot
312,2024-01-15 10:23:41,1,intrusion_000312.jpg
387,2024-01-15 10:23:44,1,intrusion_000387.jpg
```

### `snapshots/`
JPEG images captured at the moment of each alert, with all annotations visible.

---

## How It Works

```
Input Video
    │
    ▼
Frame Extraction (OpenCV VideoCapture)
    │
    ▼
Background Subtraction (MOG2)
    │  Builds an adaptive model of the "empty" scene
    │  Returns a foreground mask highlighting moving objects
    ▼
Morphological Cleanup
    │  Fills holes, removes specks from the binary mask
    ▼
Contour Detection + Size Filter
    │  Finds edges of moving blobs, discards noise below min_area
    ▼
ROI Overlap Check
    │  Each contour's bounding box is tested against the ROI
    │  Uses Intersection over Box-Area (IoBA) with configurable threshold
    ▼
Intrusion Decision
    │  If any box overlaps the ROI → intrusion event
    ▼
Output: Annotated Video + CSV Log + Snapshots
```

**Background Subtraction (MOG2):** Maintains a per-pixel Gaussian mixture model. Pixels that differ from the model are classified as foreground (moving objects). Shadow pixels are detected separately and excluded.

**Morphological Operations:** `MORPH_CLOSE` fills small gaps in detected blobs; `MORPH_OPEN` removes isolated noise pixels.

**IoBA vs IoU:** Standard Intersection over Union (IoU) shrinks when a small object is inside a large ROI. IoBA divides by the *box area* instead, ensuring a fully-enclosed small object always triggers.

---

## Sample Results

After running on the generated test video:

```
[ALERT] Frame   203 | 1 object(s) in ROI | snapshot: intrusion_000203.jpg
[ALERT] Frame   278 | 1 object(s) in ROI | snapshot: intrusion_000278.jpg
[DONE] Processed 375 frames.
[DONE] Total intrusion events : 2
[DONE] Annotated video        : output/annotated_output.mp4
[DONE] Intrusion log          : output/intrusion_log.csv
```

---

## Dependencies

| Library | Version | Purpose |
|---------|---------|---------|
| `opencv-python` | ≥ 4.8 | Frame reading, background subtraction, drawing |
| `numpy` | ≥ 1.24 | Array operations for mask processing |

---

## License

MIT License — free to use, modify, and distribute.
