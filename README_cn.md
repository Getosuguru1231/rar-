# AI溺水防控监测系统

**YOLOv8-Pose + RDK X5 + GRU + WebRTC** — 面向游泳池安全的实时溺水检测系统。

[![License](https://img.shields.io/badge/license-Apache%202.0-blue)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-RDK%20X5-orange)](https://developer.horizon.cc/)

## 演示视频

<!-- 请将 BV 号替换为你的 B站视频 -->
<iframe src="https://player.bilibili.com/player.html?bvid=BV_XXXXXXXXX" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" width="100%" height="480"></iframe>

## 项目概述

基于地平线 **RDK X5** 边缘AI计算平台的溺水防控系统。YOLOv8-Pose 逐帧提取 17 个 COCO 人体关键点，IoU 跟踪器维持跨帧人员 ID 一致，**GRU**（门控循环单元）分析 32 帧连续姿态序列，分类为四类行为：正常游泳、水中静止、溺水、已离开水域。

### 核心功能

- **边缘AI推理**：YOLOv8-Pose 配合自定义 raw-head 解码器部署在 RDK X5 BPU，15–25 FPS
- **时序行为识别**：2层 GRU 分类器处理 `[32, 55]` 姿态序列，连续 ≥3 帧溺水概率超阈值触发告警
- **双目摄像头输入**：816 × 960 双目相机，通过 ROS2 `image_combine_raw` 话题同步采集
- **多协议视频流**：MJPEG（高兼容性）+ WebRTC（<200ms 低延迟）
- **实时监控大屏**：Vue.js 3 + ECharts，展示实时视频、告警时间线、系统健康状态
- **多级告警**：高/中/低三级风险分类，支持语音播报和短信通知
- **Redis 状态总线**：检测结果和告警标识发布至 Redis，供监控大屏和外部服务消费

### 系统架构

```
┌─────────────────────────────────────────┐
│  双目摄像头 (816 × 960)                  │
└────────────────┬────────────────────────┘
                 │ ROS2 /image_combine_raw
                 ▼
┌─────────────────────────────────────────┐
│  RDK X5 BPU                             │
│  ┌─────────────────────────────────────┐│
│  │ YOLOv8-Pose (raw-head 解码器)       ││
│  │ → 17关键点 × 3 + 4bbox = 55维特征   ││
│  └───────────────┬─────────────────────┘│
│                  ▼                       │
│  ┌─────────────────────────────────────┐│
│  │ BoxTracker (IoU 匹配)               ││
│  │ → 每人独立 track_id                ││
│  └───────────────┬─────────────────────┘│
│                  ▼                       │
│  ┌─────────────────────────────────────┐│
│  │ GRU 分类器 [32, 55]                 ││
│  │ → 4类 Softmax 概率                  ││
│  └───────────────┬─────────────────────┘│
└──────────────────┼──────────────────────┘
                   │
     ┌─────────────┼──────────────┐
     ▼             ▼              ▼
  Redis       MJPEG/WebRTC    Flask API
  (状态)      (视频流)          (REST)
```

## 目录结构

```
├── APP2_GRU.py                     # RDK X5 主程序 (YOLO + GRU + Flask)
├── backend/                        # PC 端后端 (FastAPI)
│   ├── api/                        # REST API (告警/摄像头/视频/监控)
│   ├── components/                 # 告警管理、摄像头健康、WebSocket、Redis池
│   ├── detection/                  # YOLOv8 检测引擎 (多线程/多摄像头)
│   ├── services/                   # 业务层 (告警、LLM、视频存储)
│   ├── webrtc/                     # WebRTC 低延迟信令与推流
│   ├── configs/                    # system.yaml、dataset_config.yaml
│   ├── models/                     # 自定义模型加载器
│   └── main.py                     # FastAPI 应用入口
├── frontend/                       # Vue.js 3 监控大屏 (Element Plus, ECharts)
│   ├── src/components/             # Vue 组件 (视频/告警/图表)
│   └── src/composables/            # Composition API 钩子
├── gru_training/                   # ★ GRU 训练流水线
│   ├── src/
│   │   ├── pretrack_videos.py      # YOLO-Pose + ByteTrack → 关键点CSV
│   │   ├── build_gru_dataset.py    # events.csv + 关键点 → X.npy / y.npy
│   │   ├── train_gru.py            # GRU 训练 (含分层划分)
│   │   ├── evaluate.py             # 准确率 / F1 / 混淆矩阵
│   │   ├── generate_events.py      # 从关键点自动生成 events.csv
│   │   ├── check_dataset.py        # 数据集完整性检查
│   │   ├── infer_video.py          # 端到端视频推理
│   │   ├── config.py               # 全局超参数与路径
│   │   └── models/gru_classifier.py  # GRU 模型定义 (PyTorch)
│   ├── data/annotations/           # 标注模板 (events_template.csv)
│   └── requirements.txt
├── docs/                           # 技术文档
├── docker-compose.yml              # Docker 部署配置
├── start_all.sh / start_all.bat    # 一键启动脚本
└── README.md / README_cn.md
```

## 快速开始

### 环境要求

- **RDK X5**：地平线 BPU，含 `hobot_dnn` SDK
- **PC 后端**：Python 3.8+, Node.js 18+, Redis 7.0+

### 1. 克隆仓库

```bash
git clone https://github.com/Getosuguru1231/drowning-code-v2.1.git
cd drowning-code-v2.1
```

### 2. RDK X5 部署

```bash
# 将模型文件放入项目根目录
cp yolo8m-pose_x5_rawhead.bin .
cp best_gru.bin .

# 设置环境变量
export POOL_POSE_MODEL=yolo8m-pose_x5_rawhead.bin
export POOL_GRU_MODEL=best_gru.bin
export REDIS_HOST=127.0.0.1
export ENABLE_DSI_DISPLAY=0

python3 APP2_GRU.py
```

### 3. PC 后端

```bash
cd backend
pip install -r requirements.txt
python main.py    # FastAPI → http://localhost:8000
```

### 4. 前端监控大屏

```bash
cd frontend
npm install
npm run dev       # Vite → http://localhost:5173
```

### 5. GRU 训练流水线（可选）

```bash
cd gru_training
pip install -r requirements.txt

# 第一步：YOLO-Pose + ByteTrack 预处理
python src/pretrack_videos.py --input data/raw_videos --output data/pretrack --model yolov8m-pose.pt

# 第二步：生成标注模板
python src/generate_events.py

# 第三步：人工标注 events.csv，然后构建数据集
python src/build_gru_dataset.py --events data/annotations/events.csv --keypoints data/pretrack

# 第四步：训练 GRU
python src/train_gru.py --x data/processed/X.npy --y data/processed/y.npy --epochs 50

# 第五步：评估
python src/evaluate.py --x data/processed/X.npy --y data/processed/y.npy --model outputs/models/best_gru.pth
```

## 模型详情

### YOLO-Pose（边缘端）

| 参数 | 值 |
|------|-----|
| 模型 | `yolo8m-pose_x5_rawhead.bin` |
| 输入形状 | [1, 3, 640, 640] |
| 输出头 | 9（3 尺度 × box/score/keypoint） |
| 检测置信度阈值 | 0.45 |
| NMS IoU 阈值 | 0.35 |
| 关键点置信度阈值 | 0.30 |

### GRU 分类器

| 参数 | 值 |
|------|-----|
| 序列长度 | 32 帧（30FPS 下约 1 秒） |
| 特征维度 | 55（17×3 关键点 + 4 边界框） |
| 隐层大小 | 128 |
| 层数 | 2（堆叠） |
| 类别数 | 4 |
| 溺水阈值 | 0.50 |
| 连续告警帧数 | 3 |

### 行为分类标签

| ID | 标签 | 说明 |
|----|------|------|
| 0 | Swimming | 正常游泳 — 周期性肢体运动 |
| 1 | Idle in water | 水中静止 / 缓慢漂浮 |
| 2 | **Drowning** | 疑似溺水 — **触发告警** |
| 3 | Person out of water | 人员已离开水域 |

### 单帧特征设计（55维）

| 维度 | 内容 |
|------|------|
| [0 … 50] | 17 个 COCO 关键点 × (x, y, 置信度) — 归一化至 [0, 1] |
| [51 … 54] | 边界框 (中心x, 中心y, 宽, 高) — 归一化至 [0, 1] |

## API 参考

完整接口文档见 [API_DOCUMENTATION.md](API_DOCUMENTATION.md)。

### 主要接口

| 端点 | 方法 | 说明 |
|------|------|------|
| `/video` | GET | MJPEG 视频流 |
| `/api/detection` | GET | 实时检测结果 (JSON) |
| `/api/status` | GET | 系统状态 (摄像头/模型/FPS) |
| `/api/alerts/list` | GET | 告警历史列表 |
| `/api/alerts/drowning-timeline` | GET | 溺水事件时间线 |
| `/api/video_feed` | WebSocket | 低延迟帧推送 |
| `/api/webrtc/signal` | WebSocket | WebRTC 信令 |

## 性能指标

| 指标 | 数值 |
|------|------|
| YOLO 推理 (RDK X5 BPU) | 15–25 FPS |
| GRU 推理 (RDK X5 CPU) | > 60 窗口/秒 |
| WebRTC 延迟 | < 200 ms |
| MJPEG 延迟 | 1–2 秒 |
| Flask API 响应 | < 10 ms |

## 开源协议

Apache License 2.0 — 详见 [LICENSE](LICENSE)。

## 致谢

- 地平线 RDK X5 平台 & `hobot_dnn` SDK
- Ultralytics YOLOv8-Pose
- COCO 关键点标准格式
- Redis、FastAPI、Vue.js 开源社区
