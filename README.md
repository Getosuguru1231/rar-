# AI Drowning Prevention & Monitoring System

**YOLOv8-Pose + RDK X5 + GRU + WebRTC** — Real-time drowning detection for pool safety.

[![License](https://img.shields.io/badge/license-Apache%202.0-blue)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-RDK%20X5-orange)](https://developer.horizon.cc/)
[![Node](https://img.shields.io/badge/Nodehub-Deploy-green)]()

## Demo Video

<!-- Replace with your Bilibili embed link -->
```html
<iframe src="https://player.bilibili.com/player.html?bvid=YOUR_BVID" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" width="100%" height="480"></iframe>
```

## Overview

This system uses **YOLOv8-Pose** on the Horizon Robotics RDK X5 edge AI platform to detect human keypoints in real-time, then feeds temporal pose sequences to a **GRU classifier** to distinguish drowning from normal swimming with high accuracy.

### Key Features

- **Edge AI Deployment**: Optimized YOLOv8-Pose model running on RDK X5 BPU (hazard board platform), 3.8+ TOPS inference
- **Temporal Behaviour Recognition**: GRU (Gated Recurrent Unit) analyses 32-frame sliding windows of 17 keypoints to detect drowning patterns
- **Stereo Camera Pipeline**: 816×960 binocular dual-camera input, synchronized via ROS2 `/image_combine_raw` topic
- **Multi-Protocol Streaming**: MJPEG for compatibility, WebRTC for sub-200ms low-latency monitoring
- **Real-Time Dashboard**: Vue.js + ECharts web interface displaying live video feed, alert timeline, and system health
- **Alert System**: Multi-level risk classification with voice broadcast and SMS notification
- **Flask Video Server**: Lightweight standalone mode with Redis-based status reporting

### Architecture

```
Stereo Cameras (Binocular 816×960)
        │
        ▼
  ┌─────────────────┐
  │  RDK X5 BPU     │
  │  YOLO11m-Pose   │  ← 17 COCO keypoints per frame
  │  (hazard raw head)│
  └────────┬────────┘
           │ [55-dim feature vector]
           ▼
  ┌─────────────────┐
  │  BoxTracker     │  ← IoU-based person tracking
  │  (bytedance-like)│
  └────────┬────────┘
           │ [per-track deque: 32 frames]
           ▼
  ┌─────────────────┐
  │  GRU Classifier │  ← Temporal action recognition
  │  [32, 55] input │     4-class output: Swimming,
  │                  │     Idle, Drowning, Out
  └────────┬────────┘
           │ [probabilities every frame]
           ▼
  ┌─────────────────┐
  │  Alert & Stream │  ← Redis publish + MJPEG/WebRTC
  │  Dashboard       │     + Vue.js frontend
  └─────────────────┘
```

## Directory Structure

```
├── APP2_GRU.py                   # RDK X5 main application (YOLO-Pose + GRU + Flask)
├── backend/                      # PC-side backend (FastAPI)
│   ├── api/                      # REST API layer (alerts, cameras, video, config)
│   ├── components/               # Core components (alert manager, camera health, WebSocket)
│   ├── detection/                # YOLOv8 detection engine (multi-threaded, multi-camera)
│   ├── services/                 # Business services (alert, detection, LLM, video storage)
│   ├── webrtc/                   # WebRTC low-latency streaming module
│   ├── models/                   # Custom model loader
│   ├── configs/                  # Config files (system.yaml, dataset_config.yaml)
│   └── main.py                   # FastAPI application entry point
├── frontend/                     # Vue.js 3 web dashboard
│   ├── src/                      # Vue source (components, views, router)
│   └── package.json              # Frontend dependencies
├── docs/                         # Documentation & design specs
├── docker-compose.yml            # Docker deployment config
├── start_all.sh                  # One-click startup script
├── best.pt                       # YOLOv8 weight file (example)
├── requirements.txt              # Python dependencies
└── README.md                     # This file
```

## Quick Start

### Prerequisites

- Python 3.8+
- Node.js 18+
- Redis 7.0+
- OpenCV 4.8+

### 1. Clone & Install

```bash
git clone https://github.com/YOUR_USERNAME/yulin-drowning-prevention.git
cd yulin-drowning-prevention

# Backend
cd backend
pip install -r requirements.txt

# Frontend
cd ../frontend
npm install
```

### 2. Configure

Edit `backend/settings.py` to set your camera sources, Redis host, and database connection.

### 3. Run

```bash
# Development
bash start_all.sh

# Or start components individually:
bash start_backend.sh    # FastAPI server on port 8000
bash start_redis.sh      # Redis server
bash start_frontend_vue.sh  # Vue.js dev server on port 5173
```

### 4. Access

- **Dashboard**: http://localhost:5173
- **API Docs**: http://localhost:8000/docs
- **Health Check**: http://localhost:8000/api/health/health

## Model Details

### YOLOv8-Pose (Edge Deployment)

The RDK X5 version uses a custom `hobot_dnn` inference API with *rawhead* output format. Key parameters:

| Parameter | Value | Description |
|-----------|-------|-------------|
| `input_shape` | [1, 3, 640, 640] | Model input resolution |
| `conf_threshold` | 0.45 | Detection confidence threshold |
| `kpt_conf_threshold` | 0.30 | Keypoint confidence threshold |
| `N_KPT` | 17 | COCO keypoints |

### GRU Classifier

| Parameter | Value |
|-----------|-------|
| `SEQ_LEN` | 32 frames |
| `FEATURE_DIM` | 55 (17×3 + 4 bbox) |
| `num_classes` | 4 (Swimming, Idle in water, Drowning, Person out of water) |
| `hidden_size` | 128 |
| `num_layers` | 2 |

### Classification Labels

| Class ID | Label | Description |
|----------|-------|-------------|
| 0 | Swimming | Normal swimming behaviour |
| 1 | Idle in water | Person floating or standing still |
| 2 | Drowning | Suspected drowning — **triggers alarm** |
| 3 | Person out of water | Person has left the water |

## API Reference

See [API_DOCUMENTATION.md](API_DOCUMENTATION.md) for full endpoint reference.

### Key Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/health/health` | GET | System health check |
| `/api/alerts/list` | GET | Alert history list |
| `/api/alerts/drowning-timeline` | GET | Drowning event timeline |
| `/api/monitoring/stats` | GET | Real-time monitoring statistics |
| `/api/video/stream` | GET | MJPEG video stream |
| `/api/webrtc/signal` | WebSocket | WebRTC signaling |

## Performance

| Metric | Value |
|--------|-------|
| Inference FPS | 15-25 (RDK X5 BPU) |
| WebRTC Latency | <200ms |
| MJPEG Latency | 1-2s |
| GPU Memory | ~2GB |
| CPU Usage (backend) | 30-40% (4-core) |

## Acknowledgements

- Horizon Robotics RDK X5 platform
- Ultralytics YOLOv8/YOLO11
- ByteTrack tracker architecture
- COCO keypoint format

## License

This project is licensed under the Apache License 2.0 — see [LICENSE](LICENSE) for details.
