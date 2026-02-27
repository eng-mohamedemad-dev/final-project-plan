# AI Model Developer Guide — Crime Prediction System

> هذا الملف يحتوي على كل ما يحتاجه مطور الـ AI Model لفهم دوره في النظام والتكامل مع الـ Backend.
>
> This file contains everything the AI Model developer needs to understand their role in the system and integrate with the Backend.

---

## Table of Contents

1. [Your Role in the System](#your-role-in-the-system)
2. [Architecture Overview](#architecture-overview)
3. [Complete Flow](#complete-flow)
4. [API Authentication](#api-authentication)
5. [API Reference](#api-reference)
6. [Camera Alarm — Direct Trigger](#camera-alarm)
7. [Load Balancing](#load-balancing)
8. [Heartbeat Mechanism](#heartbeat-mechanism)
9. [Error Handling & Edge Cases](#error-handling)
10. [Configuration](#configuration)
11. [Data Contracts](#data-contracts)
12. [Development & Testing](#development--testing)
13. [Deployment](#deployment)

---

## Your Role in the System

You are responsible for building a **Python service** that:

1. **Fetches cameras** from the Laravel backend
2. **Watches RTSP video streams** from Tapo C200 cameras
3. **Detects crimes/suspicious activity** using your AI model
4. **Triggers the camera alarm directly** (fast response)
5. **Reports alerts and crimes** back to the Laravel backend via webhook API
6. **Sends heartbeats** every 60 seconds to indicate healthy operation

```
┌──────────────────────────────────────────────────────┐
│                   Your Service                        │
│                                                      │
│  1. Fetch cameras (GET /api/model/cameras)           │
│  2. Claim cameras (POST /api/model/cameras/{id}/claim)│
│  3. Watch RTSP streams (rtsp://user:pass@ip/stream2) │
│  4. Run AI detection on frames                       │
│  5. If suspicious → trigger alarm + POST /alert      │
│  6. If confirmed  → POST /api/model/crime            │
│  7. If cleared    → stop alarm + POST /alert/stop    │
│  8. Heartbeat every 60s (POST /api/model/heartbeat)  │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**You do NOT need to:**
- Handle user authentication
- Send notifications (backend does this)
- Manage crime records (backend does this)
- Record video evidence (backend handles this)
- Assign officers (backend does this)

---

## Architecture Overview

```
                    ┌─────────────────────┐
                    │    Laravel Backend   │
                    │   (API Server)      │
                    │                     │
                    │  - Creates crime    │
                    │  - Assigns officer  │
                    │  - Sends notifs     │
                    │  - Extracts scene   │
                    └─────┬───────┬───────┘
                          │       │
              GET cameras │       │ POST alert/crime
              Claim       │       │
                          │       │
                    ┌─────┴───────┴───────┐
                    │   Your AI Model     │
                    │   (Python Service)  │
                    │                     │
                    │  - Watches streams  │
                    │  - Runs detection   │
                    │  - Reports crimes   │
                    └─────┬───────────────┘
                          │
              RTSP stream │  + Direct alarm trigger
                          │
                    ┌─────┴───────────────┐
                    │   Tapo C200 Cameras  │
                    │   (192.168.x.x)     │
                    └─────────────────────┘
```

---

## Complete Flow

### Step 1: Startup — Fetch & Claim Cameras

```python
# On startup, fetch available (unclaimed) cameras
response = requests.get(
    f"{BACKEND_URL}/api/model/cameras",
    headers={"X-Model-Api-Key": API_KEY}
)
cameras = response.json()["data"]
# Returns max 10 unclaimed, active cameras

# Claim each camera (so other model instances don't process it)
for camera in cameras:
    requests.post(
        f"{BACKEND_URL}/api/model/cameras/{camera['id']}/claim",
        headers={"X-Model-Api-Key": API_KEY}
    )
```

### Step 2: Watch RTSP Streams

```python
import cv2

# Each camera comes with a full RTSP URL
# Use stream2 (low quality) for AI processing — less bandwidth
rtsp_url = camera["rtsp_url"]
# Example: rtsp://mohamed:m0ohamed123456789@192.168.1.6:554/stream2

cap = cv2.VideoCapture(rtsp_url)
while cap.isOpened():
    ret, frame = cap.read()
    if ret:
        # Run your AI detection model on the frame
        prediction = model.predict(frame)
        process_prediction(prediction, camera)
```

### Step 3: Detect Suspicious Activity → Send Alert

When your model starts detecting something suspicious (confidence rising):

```python
# Confidence rising above threshold → alert
if prediction.confidence > ALERT_THRESHOLD:
    # 1. Trigger camera alarm DIRECTLY (faster than going through backend)
    trigger_camera_alarm(camera)
    
    # 2. Notify backend
    requests.post(
        f"{BACKEND_URL}/api/model/alert",
        headers={"X-Model-Api-Key": API_KEY},
        json={
            "camera_id": camera["id"],
            "confidence_score": prediction.confidence,
            "alert_type": "suspicious"
        }
    )
```

### Step 4: Confirm Crime → Report to Backend

When your model confirms a crime (high confidence, sustained detection):

```python
requests.post(
    f"{BACKEND_URL}/api/model/crime",
    headers={"X-Model-Api-Key": API_KEY},
    json={
        "camera_id": camera["id"],
        "crime_type": "theft",         # or: assault, vandalism, break_in, suspicious_activity
        "confidence_score": 92.5,
        "start_time": "2026-02-27T14:28:00Z",  # When suspicious activity started
        "end_time": "2026-02-27T14:32:00Z"      # When crime was confirmed
    }
)
```

**After calling this, the backend automatically:**
1. Extracts the crime scene video evidence
2. Creates a crime record
3. Finds the nearest available officer (within 5km)
4. Assigns the officer
5. Sends push notification to officer + police station
6. Broadcasts real-time event to dashboards

### Step 5: Scene Ended → Stop Alert

When your model determines the threat is over:

```python
# 1. Stop camera alarm DIRECTLY
stop_camera_alarm(camera)

# 2. Notify backend
requests.post(
    f"{BACKEND_URL}/api/model/alert/stop",
    headers={"X-Model-Api-Key": API_KEY},
    json={
        "camera_id": camera["id"]
    }
)
```

### Step 6: Heartbeat (Continuous — Every 60s)

```python
import threading

def send_heartbeat():
    while running:
        requests.post(
            f"{BACKEND_URL}/api/model/heartbeat",
            headers={"X-Model-Api-Key": API_KEY},
            json={
                "camera_ids": [cam["id"] for cam in claimed_cameras]
            }
        )
        time.sleep(60)

# Run in background thread
heartbeat_thread = threading.Thread(target=send_heartbeat, daemon=True)
heartbeat_thread.start()
```

---

## API Authentication

All API endpoints are secured with an API key. Include it in every request:

```
Header: X-Model-Api-Key: {your_api_key}
```

The API key will be stored in the backend's `.env` file as `MODEL_API_KEY`. The backend developer will share this key with you.

**You do NOT use Sanctum tokens or login.** The model API key is a simple static key for service-to-service authentication.

---

## API Reference

### Base URL: `{BACKEND_URL}/api/model`

---

### 1. GET `/cameras` — List Available Cameras

Fetch unclaimed, active cameras ready for processing.

**Request:**
```http
GET /api/model/cameras
X-Model-Api-Key: {key}
```

**Response (200):**
```json
{
  "status": true,
  "data": [
    {
      "id": 1,
      "name": "Main Entrance",
      "ip_address": "192.168.1.6",
      "connection_ip": "41.35.120.50",
      "connection_port": 8554,
      "rtsp_url": "rtsp://mohamed:m0ohamed123456789@41.35.120.50:8554/stream2",
      "user_name": "mohamed",
      "password": "m0ohamed123456789",
      "lat": 30.0444,
      "lang": 31.2357,
      "storage_type": "none",
      "police_station_id": 1
    },
    {
      "id": 2,
      "name": "Back Gate",
      "ip_address": "192.168.1.7",
      "connection_ip": null,
      "connection_port": 554,
      "rtsp_url": "rtsp://admin:pass123@192.168.1.7:554/stream2",
      "user_name": "admin",
      "password": "pass123",
      "lat": 30.0450,
      "lang": 31.2360,
      "storage_type": "sd_card",
      "police_station_id": 1
    }
  ]
}
```

**Important:**
- Returns **max 10** cameras per request
- Only returns cameras where `is_active = true` AND `model_ip_address IS NULL` (unclaimed)
- `rtsp_url` is a **complete, ready-to-use** RTSP URL — connect directly with OpenCV
- Use `stream2` (low quality) for AI processing to save bandwidth
- `storage_type` tells you how the backend records evidence — you don't need to worry about this

---

### 2. POST `/cameras/{id}/claim` — Claim a Camera

Tell the backend this model instance is processing this camera.

**Request:**
```http
POST /api/model/cameras/1/claim
X-Model-Api-Key: {key}
```

**Response (200):**
```json
{
  "status": true,
  "message": "Camera claimed successfully"
}
```

**Error (409 — already claimed):**
```json
{
  "status": false,
  "message": "Camera is already claimed by another model instance"
}
```

**Error (422 — limit reached):**
```json
{
  "status": false,
  "message": "Maximum camera limit reached (10). Release cameras before claiming new ones."
}
```

---

### 3. DELETE `/cameras/{id}/release` — Release a Camera

Release a camera so other model instances can pick it up.

**Request:**
```http
DELETE /api/model/cameras/1/release
X-Model-Api-Key: {key}
```

**Response (200):**
```json
{
  "status": true,
  "message": "Camera released successfully"
}
```

**When to release:**
- Camera stream is unreachable (camera offline)
- Your service is shutting down gracefully
- You want to switch to different cameras

---

### 4. POST `/heartbeat` — Send Heartbeat

Tell the backend your model is still alive and processing cameras.

**Request:**
```http
POST /api/model/heartbeat
X-Model-Api-Key: {key}
Content-Type: application/json

{
  "camera_ids": [1, 2, 3, 5, 8]
}
```

**Response (200):**
```json
{
  "status": true,
  "message": "Heartbeat received"
}
```

**CRITICAL:** Send heartbeat every **60 seconds**. If the backend doesn't receive a heartbeat for **2 minutes**, it will:
1. Release all cameras claimed by your model instance
2. Make them available for other model instances
3. Log the event

---

### 5. POST `/alert` — Report Suspicious Activity

Send when your model first detects something suspicious (before confirming it's a crime).

**Request:**
```http
POST /api/model/alert
X-Model-Api-Key: {key}
Content-Type: application/json

{
  "camera_id": 1,
  "confidence_score": 65.5,
  "alert_type": "suspicious"
}
```

**Response (200):**
```json
{
  "status": true,
  "message": "Alert received"
}
```

**When to call:**
- Confidence rises above your alert threshold (e.g., 50%)
- You can send multiple alerts as confidence changes
- This is a **preliminary** warning — the police station sees "⚠️ Suspicious activity at Camera X"

**Note:** You should trigger the camera alarm **directly** BEFORE calling this endpoint — direct alarm is faster.

---

### 6. POST `/alert/stop` — Scene Ended

Report that the suspicious activity has ended.

**Request:**
```http
POST /api/model/alert/stop
X-Model-Api-Key: {key}
Content-Type: application/json

{
  "camera_id": 1
}
```

**Response (200):**
```json
{
  "status": true,
  "message": "Alert stopped"
}
```

**Note:** Stop the camera alarm **directly** BEFORE calling this endpoint.

---

### 7. POST `/crime` — Report Confirmed Crime

Send when your model has confirmed with high confidence that a crime occurred.

**Request:**
```http
POST /api/model/crime
X-Model-Api-Key: {key}
Content-Type: application/json

{
  "camera_id": 1,
  "crime_type": "theft",
  "confidence_score": 92.5,
  "start_time": "2026-02-27T14:28:00Z",
  "end_time": "2026-02-27T14:32:00Z"
}
```

**Response (201):**
```json
{
  "status": true,
  "message": "Crime reported successfully",
  "data": {
    "crime_id": 15,
    "status": "pending",
    "severity": "critical"
  }
}
```

**Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `camera_id` | integer | ✅ | Camera that detected the crime |
| `crime_type` | string | ✅ | Type of crime detected |
| `confidence_score` | float | ✅ | 0-100, your model's confidence |
| `start_time` | ISO 8601 | ✅ | When the suspicious activity started |
| `end_time` | ISO 8601 | ✅ | When the crime was confirmed |

**Valid `crime_type` values:**
| Value | Description |
|-------|-------------|
| `theft` | Stealing / stealing attempt |
| `assault` | Physical attack |
| `vandalism` | Property damage |
| `break_in` | Breaking into building/vehicle |
| `suspicious_activity` | General suspicious behavior |
| `robbery` | Armed robbery |
| `trespassing` | Unauthorized entry |
| `fighting` | Physical altercation between people |

**Severity is auto-calculated from confidence_score:**
| Confidence | Severity |
|-----------|----------|
| < 40% | `low` |
| 40-59% | `medium` |
| 60-79% | `high` |
| ≥ 80% | `critical` |

**start_time and end_time are CRITICAL** — the backend uses these timestamps to extract the right video segment from the camera recording. If the timestamps are wrong, the evidence video will be wrong.

---

## Camera Alarm

### Why Direct?

Speed matters. If you route the alarm through the backend (Model → Backend → Camera), it adds 200-500ms+ of latency. By triggering the alarm directly, the response time is < 50ms.

### How to Trigger

The Tapo C200 cameras have an HTTP API for alarm control. Since you already have the camera credentials from the `/cameras` API:

#### Option A: Use the Existing Backend Camera API (Simpler)

The backend has a camera control API (already built). You can use it:

```python
# Start alarm — use connection_ip if set, otherwise ip_address
effective_ip = camera.get("connection_ip") or camera["ip_address"]

requests.post(
    f"{BACKEND_URL}/api/camera/alarm",
    json={
        "ip": effective_ip,
        "username": camera["user_name"],
        "password": camera["password"],
        "action": "start",
        "duration": 30,       # seconds
        "type": "both",       # sound + light
        "volume": "high"
    }
)

# Stop alarm
requests.post(
    f"{BACKEND_URL}/api/camera/alarm",
    json={
        "ip": effective_ip,
        "username": camera["user_name"],
        "password": camera["password"],
        "action": "stop"
    }
)
```

#### Option B: Use pytapo Directly (Fastest — No Backend Dependency)

Install pytapo in your Python environment:

```bash
pip install pytapo
```

```python
from pytapo import Tapo

# Initialize camera connection (same credentials from API)
tapo = Tapo(
    host=camera["ip_address"],
    user=camera["user_name"],
    password=camera["password"],
    cloudPassword=camera.get("tapo_password")
)

# Start alarm
tapo.startManualAlarm()

# Stop alarm
tapo.stopManualAlarm()
```

#### Option C: Use ONVIF (Alternative)

```python
from onvif import ONVIFCamera

cam = ONVIFCamera(
    camera["ip_address"], 2020,
    camera["user_name"],
    camera["password"]
)
# Trigger alarm via ONVIF events...
```

**Recommendation:** Use Option A for simplicity. Use Option B for fastest response time.

---

## Load Balancing

### Why?

Each model instance handles a **maximum of 10 cameras**. If you have 30 cameras, you need 3 model instances.

### How It Works

```
Model Instance 1 (192.168.1.100)
├── Claims cameras 1-10
├── Sets model_ip_address = "192.168.1.100"
└── Processes 10 streams simultaneously

Model Instance 2 (192.168.1.101)
├── Fetches cameras → only gets 11-20 (1-10 are claimed)
├── Claims cameras 11-20
├── Sets model_ip_address = "192.168.1.101"
└── Processes 10 streams simultaneously

Model Instance 3 (192.168.1.102)
├── Fetches cameras → only gets 21-30
├── Claims cameras 21-30
└── Processes remaining streams
```

### Per-Instance Logic

```python
MAX_CAMERAS = 10

# Fetch only unclaimed cameras
cameras = get_available_cameras()

# Claim up to MAX_CAMERAS
for camera in cameras[:MAX_CAMERAS]:
    success = claim_camera(camera["id"])
    if success:
        claimed_cameras.append(camera)

print(f"Claimed {len(claimed_cameras)} cameras")
```

### API Enforcement

- `GET /cameras` only returns cameras where `model_ip_address IS NULL`
- `POST /cameras/{id}/claim` rejects if you already have 10 cameras claimed
- `POST /cameras/{id}/claim` rejects if camera is already claimed by someone else

---

## Heartbeat Mechanism

### Purpose

The heartbeat tells the backend "I'm still alive and processing these cameras." Without it, the backend assumes your model crashed and releases the cameras.

### Implementation

```python
import threading
import time

class HeartbeatService:
    def __init__(self, api_client, interval=60):
        self.api_client = api_client
        self.interval = interval
        self.running = False
        self.camera_ids = []
    
    def start(self, camera_ids):
        self.camera_ids = camera_ids
        self.running = True
        self.thread = threading.Thread(target=self._run, daemon=True)
        self.thread.start()
    
    def stop(self):
        self.running = False
    
    def update_cameras(self, camera_ids):
        """Call when cameras change (claim/release)"""
        self.camera_ids = camera_ids
    
    def _run(self):
        while self.running:
            try:
                self.api_client.post("/api/model/heartbeat", json={
                    "camera_ids": self.camera_ids
                })
            except Exception as e:
                print(f"Heartbeat failed: {e}")
                # Don't crash — retry next cycle
            time.sleep(self.interval)
```

### What Happens if Heartbeat Stops

| Missed Duration | Backend Action |
|----------------|----------------|
| 0-60s | Nothing — normal gap between heartbeats |
| 60-120s | Backend starts monitoring (warning logged) |
| > 120s (2 min) | Backend releases ALL cameras claimed by your instance |
| After release | Cameras become available for other model instances |

### Recovery

If your service crashes and restarts:
1. The backend already released your cameras (after 2 min timeout)
2. Just call `GET /cameras` again — your previously claimed cameras are now available
3. Claim them again and resume

---

## Error Handling

### Network Failures

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

# Setup retry logic
session = requests.Session()
retry = Retry(total=3, backoff_factor=0.5, status_forcelist=[500, 502, 503])
session.mount("http://", HTTPAdapter(max_retries=retry))

# All API calls auto-retry on server errors
```

### Camera Stream Failures

```python
def watch_camera(camera):
    cap = cv2.VideoCapture(camera["rtsp_url"])
    
    consecutive_failures = 0
    MAX_FAILURES = 10
    
    while True:
        ret, frame = cap.read()
        
        if not ret:
            consecutive_failures += 1
            if consecutive_failures >= MAX_FAILURES:
                print(f"Camera {camera['id']} unreachable — releasing")
                release_camera(camera["id"])
                break
            time.sleep(1)
            cap = cv2.VideoCapture(camera["rtsp_url"])  # Reconnect
            continue
        
        consecutive_failures = 0
        # Process frame...
```

### Graceful Shutdown

```python
import signal

def shutdown_handler(signum, frame):
    print("Shutting down — releasing all cameras...")
    heartbeat_service.stop()
    for camera in claimed_cameras:
        try:
            release_camera(camera["id"])
        except:
            pass  # Best effort
    sys.exit(0)

signal.signal(signal.SIGTERM, shutdown_handler)
signal.signal(signal.SIGINT, shutdown_handler)
```

---

## Configuration

### Environment Variables

```bash
# Backend API
BACKEND_URL=http://192.168.1.50:8000
MODEL_API_KEY=your_secret_api_key_here

# Processing
MAX_CAMERAS=10
ALERT_THRESHOLD=50.0          # Confidence % to trigger alert
CRIME_THRESHOLD=80.0          # Confidence % to confirm crime
HEARTBEAT_INTERVAL=60         # Seconds between heartbeats

# Camera
RTSP_TRANSPORT=tcp            # tcp or udp (tcp is more reliable)
FRAME_SKIP=5                  # Process every N-th frame (performance)
```

### Sample Config File

```python
# config.py
import os

class Config:
    BACKEND_URL = os.getenv("BACKEND_URL", "http://localhost:8000")
    API_KEY = os.getenv("MODEL_API_KEY", "")
    MAX_CAMERAS = int(os.getenv("MAX_CAMERAS", 10))
    ALERT_THRESHOLD = float(os.getenv("ALERT_THRESHOLD", 50.0))
    CRIME_THRESHOLD = float(os.getenv("CRIME_THRESHOLD", 80.0))
    HEARTBEAT_INTERVAL = int(os.getenv("HEARTBEAT_INTERVAL", 60))
    FRAME_SKIP = int(os.getenv("FRAME_SKIP", 5))
```

---

## Data Contracts

### Camera Object (from GET /cameras)

```json
{
  "id": 1,
  "name": "Main Entrance",
  "ip_address": "192.168.1.6",
  "rtsp_url": "rtsp://user:pass@192.168.1.6:554/stream2",
  "user_name": "mohamed",
  "password": "m0ohamed123456789",
  "lat": 30.0444,
  "lang": 31.2357,
  "storage_type": "none",
  "police_station_id": 1
}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | int | Unique camera ID (use in all API calls) |
| `name` | string | Human-readable camera name |
| `ip_address` | string | Camera's IP on the local network |
| `rtsp_url` | string | **Complete RTSP URL** — connect directly with OpenCV |
| `user_name` | string | Camera login username |
| `password` | string | Camera login password |
| `lat` | float | Camera latitude (for location context) |
| `lang` | float | Camera longitude |
| `storage_type` | string | `"none"`, `"sd_card"`, or `"cloud"` — you don't need to handle this |
| `police_station_id` | int | Which police station owns this camera |

### Alert Request

```json
{
  "camera_id": 1,
  "confidence_score": 65.5,
  "alert_type": "suspicious"
}
```

### Crime Report Request

```json
{
  "camera_id": 1,
  "crime_type": "theft",
  "confidence_score": 92.5,
  "start_time": "2026-02-27T14:28:00Z",
  "end_time": "2026-02-27T14:32:00Z"
}
```

### Heartbeat Request

```json
{
  "camera_ids": [1, 2, 3, 5, 8]
}
```

---

## Development & Testing

### Without Real Cameras

During development, you can test without real cameras:

1. **Use a local video file as fake RTSP stream:**
   ```bash
   # Use VLC to stream a local video as RTSP
   vlc --sout '#rtp{sdp=rtsp://localhost:8554/test}' crime_video.mp4 --loop
   ```
   Then connect to `rtsp://localhost:8554/test` in your code.

2. **Use the backend's seed data:**
   The backend will have fake cameras in the database. You can claim them and test the API flow even without real streams.

3. **Manual testing flow:**
   ```bash
   # 1. Fetch cameras
   curl -H "X-Model-Api-Key: test_key" http://localhost:8000/api/model/cameras
   
   # 2. Claim a camera
   curl -X POST -H "X-Model-Api-Key: test_key" http://localhost:8000/api/model/cameras/1/claim
   
   # 3. Send heartbeat
   curl -X POST -H "X-Model-Api-Key: test_key" -H "Content-Type: application/json" \
        -d '{"camera_ids": [1]}' http://localhost:8000/api/model/heartbeat
   
   # 4. Send alert
   curl -X POST -H "X-Model-Api-Key: test_key" -H "Content-Type: application/json" \
        -d '{"camera_id": 1, "confidence_score": 75.0, "alert_type": "suspicious"}' \
        http://localhost:8000/api/model/alert
   
   # 5. Report crime
   curl -X POST -H "X-Model-Api-Key: test_key" -H "Content-Type: application/json" \
        -d '{"camera_id": 1, "crime_type": "theft", "confidence_score": 92.5, "start_time": "2026-02-27T14:28:00Z", "end_time": "2026-02-27T14:32:00Z"}' \
        http://localhost:8000/api/model/crime
   
   # 6. Stop alert
   curl -X POST -H "X-Model-Api-Key: test_key" -H "Content-Type: application/json" \
        -d '{"camera_id": 1}' http://localhost:8000/api/model/alert/stop
   
   # 7. Release camera
   curl -X DELETE -H "X-Model-Api-Key: test_key" http://localhost:8000/api/model/cameras/1/release
   ```

### With Real Tapo C200 Cameras

1. Connect camera to local network
2. Set up camera using Tapo app (get username/password)
3. Camera RTSP stream will be at: `rtsp://{user}:{pass}@{camera_ip}:554/stream2`
4. Add camera to the backend database (police station app or seeder)
5. Your model fetches it via API and processes the stream

### Performance Tips

- **Frame skipping:** Don't process every frame — process every 3rd or 5th frame
- **Resolution:** Use `stream2` (low quality, ~640x360) for detection, not `stream1` (1080p)
- **Batch processing:** If using GPU, batch frames from multiple cameras
- **Threading:** Use one thread per camera stream, dedicated detection thread
- **OpenCV optimization:** Compile with CUDA support for GPU acceleration

```python
# Example: process every 5th frame
frame_count = 0
while cap.isOpened():
    ret, frame = cap.read()
    frame_count += 1
    
    if frame_count % 5 != 0:  # Skip 4 out of 5 frames
        continue
    
    prediction = model.predict(frame)
    # ...
```

---

## Deployment

### Requirements

```
Python >= 3.10
OpenCV (opencv-python or opencv-python-headless)
requests
pytapo (if using direct alarm)
your AI model dependencies (torch, tensorflow, etc.)
```

### Running as a Service

```bash
# Option 1: systemd service
sudo nano /etc/systemd/system/crime-model.service

[Unit]
Description=Crime Detection AI Model
After=network.target

[Service]
User=model
WorkingDirectory=/opt/crime-model
Environment=BACKEND_URL=http://192.168.1.50:8000
Environment=MODEL_API_KEY=your_key
ExecStart=/opt/crime-model/venv/bin/python main.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target

# Enable and start
sudo systemctl enable crime-model
sudo systemctl start crime-model
```

```bash
# Option 2: Docker
FROM python:3.10
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "main.py"]
```

### Multiple Instances

To scale beyond 10 cameras, run multiple instances:

```bash
# Instance 1 — handles cameras 1-10
MODEL_API_KEY=key1 python main.py

# Instance 2 — handles cameras 11-20 (automatically gets different cameras)
MODEL_API_KEY=key2 python main.py

# Instance 3 — handles cameras 21-30
MODEL_API_KEY=key3 python main.py
```

Each instance automatically gets different cameras because the `GET /cameras` endpoint only returns unclaimed cameras.

---

## Skeleton Code

Here's a minimal starter structure for your service:

```python
# main.py
import time
import threading
import requests
import cv2
from config import Config

class CrimeDetectionService:
    def __init__(self):
        self.config = Config()
        self.session = requests.Session()
        self.session.headers["X-Model-Api-Key"] = self.config.API_KEY
        self.claimed_cameras = []
        self.running = True
    
    def start(self):
        # 1. Fetch and claim cameras
        self.fetch_and_claim_cameras()
        
        # 2. Start heartbeat
        self.start_heartbeat()
        
        # 3. Start processing each camera in own thread
        threads = []
        for camera in self.claimed_cameras:
            t = threading.Thread(target=self.process_camera, args=(camera,))
            t.daemon = True
            t.start()
            threads.append(t)
        
        # 4. Wait for all threads
        try:
            while self.running:
                time.sleep(1)
        except KeyboardInterrupt:
            self.shutdown()
    
    def fetch_and_claim_cameras(self):
        resp = self.session.get(f"{self.config.BACKEND_URL}/api/model/cameras")
        cameras = resp.json()["data"]
        
        for cam in cameras[:self.config.MAX_CAMERAS]:
            try:
                self.session.post(f"{self.config.BACKEND_URL}/api/model/cameras/{cam['id']}/claim")
                self.claimed_cameras.append(cam)
                print(f"✅ Claimed camera: {cam['name']} ({cam['ip_address']})")
            except Exception as e:
                print(f"❌ Failed to claim {cam['name']}: {e}")
    
    def process_camera(self, camera):
        cap = cv2.VideoCapture(camera["rtsp_url"])
        frame_count = 0
        
        while self.running and cap.isOpened():
            ret, frame = cap.read()
            if not ret:
                time.sleep(1)
                cap = cv2.VideoCapture(camera["rtsp_url"])
                continue
            
            frame_count += 1
            if frame_count % self.config.FRAME_SKIP != 0:
                continue
            
            # ========================================
            # YOUR AI MODEL GOES HERE
            # prediction = your_model.predict(frame)
            # ========================================
            
            # Example logic:
            # if prediction.confidence > self.config.ALERT_THRESHOLD:
            #     self.send_alert(camera, prediction)
            # if prediction.confidence > self.config.CRIME_THRESHOLD:
            #     self.report_crime(camera, prediction)
        
        cap.release()
    
    def send_alert(self, camera, prediction):
        self.session.post(f"{self.config.BACKEND_URL}/api/model/alert", json={
            "camera_id": camera["id"],
            "confidence_score": prediction.confidence,
            "alert_type": "suspicious"
        })
    
    def report_crime(self, camera, prediction):
        self.session.post(f"{self.config.BACKEND_URL}/api/model/crime", json={
            "camera_id": camera["id"],
            "crime_type": prediction.crime_type,
            "confidence_score": prediction.confidence,
            "start_time": prediction.start_time.isoformat(),
            "end_time": prediction.end_time.isoformat()
        })
    
    def start_heartbeat(self):
        def heartbeat_loop():
            while self.running:
                try:
                    ids = [c["id"] for c in self.claimed_cameras]
                    self.session.post(f"{self.config.BACKEND_URL}/api/model/heartbeat",
                                     json={"camera_ids": ids})
                except:
                    pass
                time.sleep(60)
        
        t = threading.Thread(target=heartbeat_loop, daemon=True)
        t.start()
    
    def shutdown(self):
        print("Shutting down...")
        self.running = False
        for cam in self.claimed_cameras:
            try:
                self.session.delete(f"{self.config.BACKEND_URL}/api/model/cameras/{cam['id']}/release")
                print(f"Released camera: {cam['name']}")
            except:
                pass

if __name__ == "__main__":
    service = CrimeDetectionService()
    service.start()
```

---

## Summary Checklist

- [ ] Set up Python project with dependencies
- [ ] Implement camera fetching from Laravel API
- [ ] Implement camera claiming/releasing
- [ ] Connect to RTSP streams via OpenCV
- [ ] Integrate AI model for crime detection
- [ ] Implement confidence thresholds (alert vs. crime)
- [ ] Implement camera alarm (direct via pytapo or camera API)
- [ ] Implement alert endpoint (POST /alert)
- [ ] Implement crime reporting (POST /crime)
- [ ] Implement alert stop (POST /alert/stop)
- [ ] Implement heartbeat (every 60s)
- [ ] Handle graceful shutdown (release cameras)
- [ ] Handle camera disconnections (retry + release)
- [ ] Test with real Tapo C200 cameras
- [ ] Test full flow: detect → alarm → alert → crime → stop
- [ ] Optimize: frame skipping, threading, GPU acceleration
- [ ] Deploy as systemd service or Docker container
