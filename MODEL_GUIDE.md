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
5. [Encryption — Decrypting Camera Data](#encryption)
6. [API Reference](#api-reference)
7. [Camera Alarm — Direct Trigger](#camera-alarm)
8. [Heartbeat Mechanism](#heartbeat-mechanism)
9. [Error Handling & Edge Cases](#error-handling)
10. [Configuration](#configuration)
11. [Data Contracts](#data-contracts)
12. [Development & Testing](#development--testing)
13. [Deployment](#deployment)

---

## Your Role in the System

You are responsible for building a **Python service** that:

1. **Logs in** to the Laravel backend (email + password — admin registers your model)
2. **Fetches assigned cameras** from the Laravel backend (admin assigns cameras to your model)
3. **Decrypts camera credentials** (AES-256-CBC encrypted for security)
4. **Watches RTSP video streams** from Tapo C200 cameras
5. **Detects crimes/suspicious activity** using your AI model
6. **Triggers the camera alarm directly** (fast response)
7. **Reports alerts and crimes** back to the Laravel backend via webhook API
8. **Sends heartbeats** every 60 seconds to indicate healthy operation

```
┌──────────────────────────────────────────────────────┐
│                   Your Service                       │
│                                                      │
│  1. Login (POST /api/model/login)                    │
│  2. Fetch assigned cameras (GET /api/model/cameras)  │
│  3. Decrypt camera credentials (AES-256-CBC)         │
│  4. Watch RTSP streams (rtsp://user:pass@ip/stream2) │
│  5. Run AI detection on frames                       │
│  6. If suspicious → trigger alarm + POST /alert      │
│  7. If confirmed  → POST /api/model/crime            │
│  8. If cleared    → stop alarm + POST /alert/stop    │
│  9. Heartbeat every 60s (POST /api/model/heartbeat)  │
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
                    │    Laravel Backend  │
                    │   (API Server)      │
                    │                     │
                    │  - Creates crime    │
                    │  - Assigns officer  │
                    │  - Sends notifs     │
                    │  - Extracts scene   │
                    └─────┬───────┬───────┘
                          │       │
             Login + GET  │       │ POST alert/crime
             cameras      │       │
             (encrypted)  │       │
                          │       │
                    ┌─────┴───────┴───────┐
                    │   Your AI Model     │
                    │   (Python Service)  │
                    │                     │
                    │  - Logs in (Sanctum)│
                    │  - Decrypts creds   │
                    │  - Watches streams  │
                    │  - Runs detection   │
                    │  - Reports crimes   │
                    └─────┬───────────────┘
                          │
              RTSP stream │  + Direct alarm trigger
                          │
                    ┌─────┴───────────────┐
                    │   Tapo C200 Cameras │
                    │   (192.168.x.x)     │
                    └─────────────────────┘
```

---

## Complete Flow

### Step 1: Startup — Login & Fetch Assigned Cameras

```python
import base64
import json
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.primitives import padding

# Login to get Sanctum token
login_response = requests.post(
    f"{BACKEND_URL}/api/model/login",
    json={
        "email": MODEL_EMAIL,
        "password": MODEL_PASSWORD
    }
)
token = login_response.json()["data"]["token"]
headers = {"Authorization": f"Bearer {token}"}

# Fetch cameras assigned to this model by admin
response = requests.get(
    f"{BACKEND_URL}/api/model/cameras",
    headers=headers
)
cameras_raw = response.json()["data"]

# Decrypt camera credentials
cameras = []
for cam in cameras_raw:
    decrypted = decrypt_camera_data(cam["encrypted_data"], ENCRYPTION_KEY)
    cam_data = {
        **cam,
        **decrypted  # ip_address, rtsp_url, user_name, password, connection_ip, etc.
    }
    del cam_data["encrypted_data"]
    cameras.append(cam_data)
```

### Step 2: Watch RTSP Streams

```python
import cv2

# Each camera has decrypted rtsp_url ready to use
# Use stream2 (low quality) for AI processing — less bandwidth
rtsp_url = camera["rtsp_url"]
# Example: rtsp://mohamed:m0ohamed123456789@41.35.120.50:8554/stream2

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
        headers=headers,
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
    headers=headers,
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
    headers=headers,
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
            headers=headers,
            json={
                "camera_ids": [cam["id"] for cam in cameras]
            }
        )
        time.sleep(60)

# Run in background thread
heartbeat_thread = threading.Thread(target=send_heartbeat, daemon=True)
heartbeat_thread.start()
```

---

## API Authentication

Your model authenticates using **email + password** login, just like other users in the system. The admin registers your model instance from the dashboard and gives you:

- **Email** — your model's login email
- **Password** — your model's login password
- **Encryption Key** — AES-256-CBC key for decrypting camera data

### Login Flow

```python
# Step 1: Login to get a Sanctum Bearer token
response = requests.post(f"{BACKEND_URL}/api/model/login", json={
    "email": "model1@system.local",
    "password": "your_password_here"
})
token = response.json()["data"]["token"]

# Step 2: Use the token in all subsequent requests
headers = {"Authorization": f"Bearer {token}"}
```

### IP Verification

The backend verifies your IP address on **every request** (including login). Your model's IP must match the whitelisted `ip_address` configured by the admin. If your IP changes, ask the admin to update it.

**Error (403 — IP mismatch):**
```json
{
  "status": false,
  "message": "Request IP does not match the whitelisted IP address for this model"
}
```

---

## Encryption

### Why Encryption?

Camera credentials (usernames, passwords, RTSP URLs) are sensitive. To prevent interception, the backend encrypts them with AES-256-CBC before sending them to you. You decrypt them locally.

### Decryption Implementation

```python
import base64
import json
import hashlib
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.primitives import padding as sym_padding
from cryptography.hazmat.backends import default_backend

def decrypt_camera_data(encrypted_payload: str, encryption_key: str) -> dict:
    """
    Decrypt camera data encrypted by Laravel's AES-256-CBC.
    
    Laravel's encrypt() produces a base64-encoded JSON with:
    - iv: base64-encoded initialization vector
    - value: base64-encoded encrypted data
    - mac: HMAC for integrity verification
    """
    payload = json.loads(base64.b64decode(encrypted_payload))
    
    iv = base64.b64decode(payload["iv"])
    value = base64.b64decode(payload["value"])
    
    # Derive key (Laravel uses the raw key from config)
    key = base64.b64decode(encryption_key)
    
    # Decrypt
    cipher = Cipher(algorithms.AES(key), modes.CBC(iv), backend=default_backend())
    decryptor = cipher.decryptor()
    decrypted_padded = decryptor.update(value) + decryptor.finalize()
    
    # Remove PKCS7 padding
    unpadder = sym_padding.PKCS7(128).unpadder()
    decrypted = unpadder.update(decrypted_padded) + unpadder.finalize()
    
    return json.loads(decrypted)
```

### Decrypted Data Structure

```json
{
  "ip_address": "192.168.1.6",
  "connection_ip": "41.35.120.50",
  "connection_port": 8554,
  "rtsp_url": "rtsp://mohamed:m0ohamed123456789@41.35.120.50:8554/stream2",
  "user_name": "mohamed",
  "password": "m0ohamed123456789",
  "api_port": 8443,
  "onvif_port": 8080
}
```

### Key Management

The encryption key (`MODEL_ENCRYPTION_KEY`) is shared between the Laravel backend and your model. Store it securely in your environment variables — never hardcode it.

---

## API Reference

### Base URL: `{BACKEND_URL}/api/model`

---

### 1. POST `/login` — Authenticate Model

Login with your email and password to receive a Sanctum bearer token.

**Request:**
```http
POST /api/model/login
Content-Type: application/json

{
  "email": "model1@system.local",
  "password": "your_password"
}
```

**Response (200):**
```json
{
  "status": true,
  "message": "Login successful",
  "data": {
    "token": "1|abc123def456...",
    "model": {
      "id": 1,
      "name": "Model Instance 1",
      "email": "model1@system.local",
      "ip_address": "41.35.120.50"
    }
  }
}
```

**Error (401 — invalid credentials):**
```json
{
  "status": false,
  "message": "Invalid credentials"
}
```

**Error (403 — IP mismatch):**
```json
{
  "status": false,
  "message": "Request IP does not match the whitelisted IP address for this model"
}
```

**Important:**
- Your request IP must match the whitelisted `ip_address` the admin configured
- Save the token and include it as `Authorization: Bearer {token}` in all subsequent requests
- The token does not expire until you log out or the admin revokes it

---

### 2. GET `/cameras` — List Assigned Cameras

Fetch cameras that the admin has assigned to your model instance. Camera credentials are **encrypted**.

**Request:**
```http
GET /api/model/cameras
Authorization: Bearer {token}
```

**Response (200):**
```json
{
  "status": true,
  "data": [
    {
      "id": 1,
      "name": "Main Entrance",
      "encrypted_data": "eyJpdiI6Ik...(base64 encoded AES-256-CBC encrypted JSON)...",
      "lat": 30.0444,
      "lang": 31.2357,
      "storage_type": "none",
      "police_station_id": 1
    },
    {
      "id": 2,
      "name": "Back Gate",
      "encrypted_data": "eyJpdiI6Im...",
      "lat": 30.0450,
      "lang": 31.2360,
      "storage_type": "sd_card",
      "police_station_id": 1
    }
  ]
}
```

**Important:**
- Returns **only cameras assigned to your model** by the admin
- Only returns cameras where `is_active = true`
- The `encrypted_data` field must be decrypted using the `MODEL_ENCRYPTION_KEY` (see [Encryption](#encryption) section)
- After decryption, you get: `ip_address`, `connection_ip`, `connection_port`, `rtsp_url`, `user_name`, `password`, `api_port`, `onvif_port`
- Use `stream2` (low quality) for AI processing to save bandwidth
- `storage_type` tells you how the backend records evidence — you don't need to worry about this

---

### 3. POST `/heartbeat` — Send Heartbeat

Tell the backend your model is still alive and processing cameras.

**Request:**
```http
POST /api/model/heartbeat
Authorization: Bearer {token}
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
1. Mark your model as inactive (`is_active = false`)
2. Log the event

---

### 4. POST `/alert` — Report Suspicious Activity

Send when your model first detects something suspicious (before confirming it's a crime).

**Request:**
```http
POST /api/model/alert
Authorization: Bearer {token}
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

### 5. POST `/alert/stop` — Scene Ended

Report that the suspicious activity has ended.

**Request:**
```http
POST /api/model/alert/stop
Authorization: Bearer {token}
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

### 6. POST `/crime` — Report Confirmed Crime

Send when your model has confirmed with high confidence that a crime occurred.

**Request:**
```http
POST /api/model/crime
Authorization: Bearer {token}
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

The Tapo C200 cameras have an HTTP API for alarm control. Since you have the camera credentials from the API (after decryption):

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

## Camera Assignment (Admin-Managed)

### How It Works

Camera assignment is **controlled entirely by the admin** from the dashboard. You do NOT choose which cameras to process — the admin assigns them to your model instance.

```
Admin Dashboard
├── Registers Model Instance 1 (IP: 41.35.120.50)
│   └── Assigns cameras 1, 2, 3, 4, 5
├── Registers Model Instance 2 (IP: 192.168.1.101)
│   └── Assigns cameras 6, 7, 8, 9, 10
└── Registers Model Instance 3 (IP: 10.0.0.5)
    └── Assigns cameras 11, 12, 13, 14, 15
```

### What This Means for You

- `GET /cameras` returns **only the cameras assigned to your model** — no filtering needed
- You process ALL cameras returned — no claiming or releasing
- If the admin adds/removes cameras from your assignment, the next `GET /cameras` call reflects the change
- You can periodically re-fetch cameras (e.g., every 5 minutes) to pick up assignment changes

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
        """Call when camera assignments change (after re-fetching)"""
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
| > 120s (2 min) | Backend marks your model as inactive |
| After inactive | Admin is notified; cameras remain assigned but not being processed |

### Recovery

If your service crashes and restarts:
1. Login again (`POST /api/model/login`) to get a new token
2. Fetch your assigned cameras (`GET /api/model/cameras`) — they're still assigned to you
3. Decrypt credentials and resume processing
4. Heartbeat resumes — backend marks model as active again

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
                print(f"Camera {camera['id']} unreachable — skipping")
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
    print("Shutting down gracefully...")
    heartbeat_service.stop()
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
MODEL_EMAIL=model1@system.local
MODEL_PASSWORD=your_password_here
MODEL_ENCRYPTION_KEY=base64:your_aes_256_key_here

# Processing
ALERT_THRESHOLD=50.0          # Confidence % to trigger alert
CRIME_THRESHOLD=80.0          # Confidence % to confirm crime
HEARTBEAT_INTERVAL=60         # Seconds between heartbeats
CAMERA_REFRESH_INTERVAL=300   # Seconds between camera list refreshes

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
    MODEL_EMAIL = os.getenv("MODEL_EMAIL", "")
    MODEL_PASSWORD = os.getenv("MODEL_PASSWORD", "")
    ENCRYPTION_KEY = os.getenv("MODEL_ENCRYPTION_KEY", "")
    ALERT_THRESHOLD = float(os.getenv("ALERT_THRESHOLD", 50.0))
    CRIME_THRESHOLD = float(os.getenv("CRIME_THRESHOLD", 80.0))
    HEARTBEAT_INTERVAL = int(os.getenv("HEARTBEAT_INTERVAL", 60))
    CAMERA_REFRESH_INTERVAL = int(os.getenv("CAMERA_REFRESH_INTERVAL", 300))
    FRAME_SKIP = int(os.getenv("FRAME_SKIP", 5))
```

---

## Data Contracts

### Camera Object (from GET /cameras — before decryption)

```json
{
  "id": 1,
  "name": "Main Entrance",
  "encrypted_data": "eyJpdiI6Ik...(base64 AES-256-CBC encrypted)...",
  "lat": 30.0444,
  "lang": 31.2357,
  "storage_type": "none",
  "police_station_id": 1
}
```

### Camera Object (after decryption of `encrypted_data`)

```json
{
  "ip_address": "192.168.1.6",
  "connection_ip": "41.35.120.50",
  "connection_port": 8554,
  "rtsp_url": "rtsp://mohamed:m0ohamed123456789@41.35.120.50:8554/stream2",
  "user_name": "mohamed",
  "password": "m0ohamed123456789",
  "api_port": 8443,
  "onvif_port": 8080
}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | int | Unique camera ID (use in all API calls) |
| `name` | string | Human-readable camera name |
| `encrypted_data` | string | AES-256-CBC encrypted JSON containing credentials |
| `lat` | float | Camera latitude (for location context) |
| `lang` | float | Camera longitude |
| `storage_type` | string | `"none"`, `"sd_card"`, or `"cloud"` — you don't need to handle this |
| `police_station_id` | int | Which police station owns this camera |

**Decrypted fields:**
| Field | Type | Description |
|-------|------|-------------|
| `ip_address` | string | Camera's IP on the local network |
| `connection_ip` | string/null | Public IP for remote access (use this if set) |
| `connection_port` | int | RTSP port (default 554) |
| `rtsp_url` | string | **Complete RTSP URL** — connect directly with OpenCV |
| `user_name` | string | Camera login username |
| `password` | string | Camera login password |
| `api_port` | int | Camera API port |
| `onvif_port` | int | ONVIF port |

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
   # 1. Login
   TOKEN=$(curl -s -X POST -H "Content-Type: application/json" \
        -d '{"email": "model1@system.local", "password": "test_password"}' \
        http://localhost:8000/api/model/login | jq -r '.data.token')
   
   # 2. Fetch assigned cameras (encrypted)
   curl -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/model/cameras
   
   # 3. Send heartbeat
   curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
        -d '{"camera_ids": [1]}' http://localhost:8000/api/model/heartbeat
   
   # 4. Send alert
   curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
        -d '{"camera_id": 1, "confidence_score": 75.0, "alert_type": "suspicious"}' \
        http://localhost:8000/api/model/alert
   
   # 5. Report crime
   curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
        -d '{"camera_id": 1, "crime_type": "theft", "confidence_score": 92.5, "start_time": "2026-02-27T14:28:00Z", "end_time": "2026-02-27T14:32:00Z"}' \
        http://localhost:8000/api/model/crime
   
   # 6. Stop alert
   curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
        -d '{"camera_id": 1}' http://localhost:8000/api/model/alert/stop
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
cryptography                   # for AES-256-CBC decryption
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
Environment=MODEL_EMAIL=model1@system.local
Environment=MODEL_PASSWORD=your_password
Environment=MODEL_ENCRYPTION_KEY=base64:your_key
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

To handle more cameras, the admin registers multiple model instances from the dashboard and assigns cameras to each:

```bash
# Instance 1 — admin assigned cameras 1-5
MODEL_EMAIL=model1@system.local MODEL_PASSWORD=pass1 python main.py

# Instance 2 — admin assigned cameras 6-10
MODEL_EMAIL=model2@system.local MODEL_PASSWORD=pass2 python main.py

# Instance 3 — admin assigned cameras 11-15
MODEL_EMAIL=model3@system.local MODEL_PASSWORD=pass3 python main.py
```

Each instance has its own credentials and IP whitelist. The admin controls which cameras each instance processes.

---

## Skeleton Code

Here's a minimal starter structure for your service:

```python
# main.py
import time
import threading
import requests
import cv2
import json
import base64
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.primitives import padding as sym_padding
from cryptography.hazmat.backends import default_backend
from config import Config

class CrimeDetectionService:
    def __init__(self):
        self.config = Config()
        self.session = requests.Session()
        self.cameras = []
        self.running = True
        self.token = None
    
    def start(self):
        # 1. Login
        self.login()
        
        # 2. Fetch assigned cameras and decrypt credentials
        self.fetch_cameras()
        
        # 3. Start heartbeat
        self.start_heartbeat()
        
        # 4. Start processing each camera in own thread
        threads = []
        for camera in self.cameras:
            t = threading.Thread(target=self.process_camera, args=(camera,))
            t.daemon = True
            t.start()
            threads.append(t)
        
        # 5. Wait for all threads
        try:
            while self.running:
                time.sleep(1)
        except KeyboardInterrupt:
            self.shutdown()
    
    def login(self):
        resp = self.session.post(f"{self.config.BACKEND_URL}/api/model/login", json={
            "email": self.config.MODEL_EMAIL,
            "password": self.config.MODEL_PASSWORD
        })
        resp.raise_for_status()
        self.token = resp.json()["data"]["token"]
        self.session.headers["Authorization"] = f"Bearer {self.token}"
        print(f"✅ Logged in successfully")
    
    def decrypt_camera_data(self, encrypted_payload: str) -> dict:
        payload = json.loads(base64.b64decode(encrypted_payload))
        iv = base64.b64decode(payload["iv"])
        value = base64.b64decode(payload["value"])
        key = base64.b64decode(self.config.ENCRYPTION_KEY.replace("base64:", ""))
        
        cipher = Cipher(algorithms.AES(key), modes.CBC(iv), backend=default_backend())
        decryptor = cipher.decryptor()
        decrypted_padded = decryptor.update(value) + decryptor.finalize()
        
        unpadder = sym_padding.PKCS7(128).unpadder()
        decrypted = unpadder.update(decrypted_padded) + unpadder.finalize()
        return json.loads(decrypted)
    
    def fetch_cameras(self):
        resp = self.session.get(f"{self.config.BACKEND_URL}/api/model/cameras")
        raw_cameras = resp.json()["data"]
        
        self.cameras = []
        for cam in raw_cameras:
            decrypted = self.decrypt_camera_data(cam["encrypted_data"])
            cam_data = {**cam, **decrypted}
            del cam_data["encrypted_data"]
            self.cameras.append(cam_data)
            print(f"✅ Camera ready: {cam['name']} ({decrypted.get('connection_ip') or decrypted['ip_address']})")
        
        print(f"📷 Total cameras assigned: {len(self.cameras)}")
    
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
                    ids = [c["id"] for c in self.cameras]
                    self.session.post(f"{self.config.BACKEND_URL}/api/model/heartbeat",
                                     json={"camera_ids": ids})
                except:
                    pass
                time.sleep(self.config.HEARTBEAT_INTERVAL)
        
        t = threading.Thread(target=heartbeat_loop, daemon=True)
        t.start()
    
    def shutdown(self):
        print("Shutting down gracefully...")
        self.running = False

if __name__ == "__main__":
    service = CrimeDetectionService()
    service.start()
```

---

## Summary Checklist

- [ ] Set up Python project with dependencies (including `cryptography`)
- [ ] Implement login flow (POST /api/model/login)
- [ ] Implement camera fetching from Laravel API
- [ ] Implement AES-256-CBC decryption for camera credentials
- [ ] Connect to RTSP streams via OpenCV
- [ ] Integrate AI model for crime detection
- [ ] Implement confidence thresholds (alert vs. crime)
- [ ] Implement camera alarm (direct via pytapo or camera API)
- [ ] Implement alert endpoint (POST /alert)
- [ ] Implement crime reporting (POST /crime)
- [ ] Implement alert stop (POST /alert/stop)
- [ ] Implement heartbeat (every 60s)
- [ ] Handle graceful shutdown
- [ ] Handle camera disconnections (retry + skip)
- [ ] Implement periodic camera list refresh (pick up assignment changes)
- [ ] Test with real Tapo C200 cameras
- [ ] Test full flow: login → decrypt → detect → alarm → alert → crime → stop
- [ ] Optimize: frame skipping, threading, GPU acceleration
- [ ] Deploy as systemd service or Docker container
