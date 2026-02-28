# Flutter Developer Guide — Crime Prediction System

> هذا الملف يحتوي على كل ما يحتاجه مطور Flutter لبناء تطبيقين: **تطبيق الضابط** و **تطبيق مركز الشرطة**.
>
> This file contains everything the Flutter developer needs to build two apps: **Officer App** and **Police Station App**.

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Apps to Build](#apps-to-build)
3. [Tech Stack (Mobile)](#tech-stack-mobile)
4. [Authentication](#authentication)
5. [Officer App — Screens & Features](#officer-app)
6. [Police Station App — Screens & Features](#police-station-app)
7. [API Endpoints Reference](#api-endpoints-reference)
8. [Real-time Integration (Pusher)](#real-time-integration-pusher)
9. [Push Notifications (Firebase)](#push-notifications-firebase)
10. [Chat System](#chat-system)
11. [GPS Location Tracking](#gps-location-tracking)
12. [Error Handling](#error-handling)
13. [Data Models](#data-models)
14. [Important Notes](#important-notes)

---

## System Overview

This is a **Crime Prediction System** that uses AI cameras to detect crimes. The system has 3 user types:

| User Type | Platform | Auth Guard |
|-----------|----------|------------|
| **Admin** | Web dashboard (Livewire) — NOT your responsibility | `admin` |
| **Police Station** | Flutter mobile app | `police-station` |
| **Officer** | Flutter mobile app | `officer` |

**You are building 2 Flutter apps** (or 1 app with 2 modes):
1. **Officer App** — Used by police officers in the field
2. **Police Station App** — Used by police station managers

### How the System Works

```
1. AI cameras detect suspicious activity
2. Backend creates a crime record and finds the nearest officer
3. Officer gets a push notification
4. Officer accepts/declines the crime
5. Officer arrives at the scene and resolves it
6. Police station monitors everything in real-time on a map
```

---

## Apps to Build

### Option A: Two Separate Apps
- `officer_app/` — Officer-only features
- `police_station_app/` — Station-only features

### Option B: One App with Guard Selection (Recommended)
- Single codebase with a login screen that determines the app mode based on which guard the user logs in with
- Shared code: auth, notifications, profile, chat base
- Different home screens based on user type

---

## Tech Stack (Mobile)

| Package | Purpose |
|---------|---------|
| `dio` | HTTP client for API calls |
| `pusher_channels_flutter` | Real-time WebSocket events |
| `firebase_messaging` | Push notifications |
| `firebase_core` | Firebase initialization |
| `google_maps_flutter` | Maps for crime locations + officer tracking |
| `geolocator` | GPS location tracking |
| `shared_preferences` or `flutter_secure_storage` | Store auth token + encryption key |
| `provider` / `riverpod` / `bloc` | State management (your choice) |
| `file_picker` | Excel file upload |
| `video_player` | Play crime scene evidence videos |
| `flutter_vlc_player` or `media_kit` | RTSP live stream playback (camera live view) |
| `encrypt` / `pointycastle` | AES-256-CBC decryption of camera data |

---

## Authentication

### Base URL

```
https://{server-ip}/api/v1
```

All API requests (except login) must include the auth token in the header:

```
Authorization: Bearer {token}
```

### Login Flow

Both apps use the **same login endpoint** with different guard values:

```
POST /api/v1/{guard}/login
```

- Officer app: `POST /api/v1/officer/login`
- Police Station app: `POST /api/v1/police-station/login`

**Request:**
```json
{
  "email": "officer@example.com",
  "password": "password123"
}
```

**Response (200):**
```json
{
  "status": true,
  "message": "Login successful",
  "data": {
    "token": "1|abc123def456...",
    "encryption_key": "aB3cD4eF5gH6iJ7kL8mN9oP0qR1sT2uV3wX4yZ5aB6cD7eF8gH9iJ0kL1mN2oP3",
    "user": {
      "id": 1,
      "name": "Ahmed Mohamed",
      "email": "officer@example.com",
      "phone": "01012345678",
      "avatar": "/storage/avatars/officer_1.jpg"
    }
  }
}
```

**Store the token AND encryption_key** securely using `flutter_secure_storage`. The encryption key is used to decrypt sensitive camera data (RTSP URLs, credentials) received from the API. The key is regenerated on every login, so always use the latest one.

### Logout

```
POST /api/v1/{guard}/logout
Authorization: Bearer {token}
```

Revokes the token on the server side.

### System Language (IMPORTANT)

The app language is controlled by the admin from the dashboard. The Flutter app must fetch the system language on startup (before login) and use it for the entire UI.

```
GET /api/v1/settings/language
```

**Response:**
```json
{
  "status": true,
  "data": {
    "language": "ar"
  }
}
```

**Important Notes:**
- This endpoint requires **NO authentication** — it's public
- Call this on every app startup to get the latest language
- The user has NO option to change the language in the app — it's admin-controlled
- Supported values: `ar` (Arabic, RTL) and `en` (English, LTR)
- Cache the language locally for offline use, but refresh on every startup

### Encryption — Decrypting Camera Data

Some API responses contain encrypted sensitive data (camera credentials, RTSP URLs). The Flutter app must decrypt this data using the `encryption_key` received at login.

**Algorithm:** AES-256-CBC

**Encrypted data format:** Base64-encoded string containing `IV:CIPHERTEXT` (16-byte IV prepended to ciphertext, then Base64-encoded)

**Decryption steps:**
1. Base64-decode the `encrypted_data` field
2. Extract the first 16 bytes as the IV
3. Decrypt the remaining bytes using AES-256-CBC with the `encryption_key` and IV
4. Parse the decrypted string as JSON

**Dart example using `encrypt` package:**
```dart
import 'package:encrypt/encrypt.dart';
import 'dart:convert';

Map<String, dynamic> decryptCameraData(String encryptedData, String encryptionKey) {
  final decoded = base64.decode(encryptedData);
  final iv = IV(decoded.sublist(0, 16));
  final cipherText = Encrypted(decoded.sublist(16));
  final key = Key.fromUtf8(encryptionKey.substring(0, 32)); // AES-256 needs 32 bytes
  final encrypter = Encrypter(AES(key, mode: AESMode.cbc));
  final decrypted = encrypter.decrypt(cipherText, iv: iv);
  return jsonDecode(decrypted);
}
```

**When decrypted, camera data contains:**
```json
{
  "ip_address": "192.168.1.6",
  "connection_ip": "41.35.100.50",
  "connection_port": 8554,
  "rtsp_url": "rtsp://user:pass@41.35.100.50:8554/stream2",
  "user_name": "admin",
  "password": "camera123",
  "api_port": 8443,
  "onvif_port": 8080
}
```

### Error Response Format (All Endpoints)

```json
{
  "status": false,
  "message": "Validation error",
  "errors": {
    "email": ["The email field is required."],
    "password": ["The password is incorrect."]
  }
}
```

---

## Officer App

### Screen List

| # | Screen | Description |
|---|--------|-------------|
| 1 | **Login** | Email + password login |
| 2 | **Home / Crime List** | List of crimes assigned to this officer or within 5km |
| 3 | **Crime Detail** | Full crime info + scene video + location on map |
| 4 | **Map View** | Map showing nearby crime locations |
| 5 | **Profile** | View/edit name, email, phone, password, avatar upload |
| 6 | **Notifications** | List of all notifications |
| 7 | **Chat** | Chat with police station |
| 8 | **Status Toggle** | Change availability (available/busy/offline) |
| 9 | **Statistics** | Crime stats (daily/weekly/monthly) |

### Screen Details

#### 1. Home / Crime List

- Fetch crimes assigned to the officer OR within 5km of current location
- Each crime card shows: crime type, severity (color-coded), status, camera name, time, distance
- Pull-to-refresh + pagination
- Real-time: new crimes appear automatically via Pusher

**API:** `GET /api/v1/officer/crimes`

**Query params:**
- `page` — pagination
- `status` — filter by status (optional)

**Response:**
```json
{
  "status": true,
  "data": [
    {
      "id": 1,
      "crime_type": { "id": 1, "name_en": "Theft", "name_ar": "سرقة" },
      "status": "assigned",
      "severity": "high",
      "confidence_score": 85.5,
      "crime_date": "2026-02-27T14:30:00Z",
      "camera": {
        "id": 3,
        "name": "Main Entrance",
        "lat": 30.0444,
        "lang": 31.2357,
        "location_name": "Cairo Station"
      },
      "officer": { "id": 5, "name": "Ahmed" },
      "scene": {
        "sence_path": "/storage/crimes/1/evidence.mp4",
        "thumbnail_path": "/storage/crimes/1/thumbnail.jpg",
        "start_date": "2026-02-27T14:28:00Z",
        "end_date": "2026-02-27T14:32:00Z"
      },
      "distance_km": 2.3,
      "created_at": "2026-02-27T14:30:00Z"
    }
  ],
  "meta": { "current_page": 1, "last_page": 3, "total": 25 }
}
```

#### 2. Crime Detail

- Shows full crime info, map with crime location pin, evidence video player
- Action buttons based on status:
  - `assigned` → "Accept" / "Decline" buttons
  - `in_progress` → "I Arrived" button
  - `visited` → "Mark Resolved" button

**Actions:**

```
PUT /api/v1/officer/crimes/{id}/accept
→ No body needed. Status changes to in_progress.

PUT /api/v1/officer/crimes/{id}/arrive
→ No body needed. Status changes to visited. officer_arrive_time is recorded.

PUT /api/v1/officer/crimes/{id}/no-visit
→ Body: { "no_visit_reason": "False alarm - no suspicious activity found" }
→ Status changes to not_visited.

PUT /api/v1/officer/crimes/{id}/resolve
→ Body: { "notes": "Suspect apprehended" } (optional)
→ Status changes to resolved.
```

#### 3. GPS Location Tracking (Background)

**CRITICAL**: The officer app must send GPS updates to the server.

- Update location every time the officer moves **100 meters** or more
- Use background location service (even when app is minimized)
- This is used to find the nearest officer when a crime is detected

```
POST /api/v1/officer/location
Body: { "latitude": 30.0444, "longitude": 31.2357 }
```

#### 4. Status Toggle

- Officer can toggle between: `available`, `busy`, `offline`
- Only `available` + `is_on_shift: true` officers receive crime assignments
- Show current status prominently (green/yellow/red indicator)

```
PUT /api/v1/officer/status
Body: { "status": "available" }
```

#### 5. Chat (with Police Station)

- Officer can only chat with their assigned police station
- Simple 1-to-1 chat interface

```
GET  /api/v1/officer/chat          → Get message history (paginated)
POST /api/v1/officer/chat          → Send message
Body: { "message": "I'm on my way", "type": "text" }
```

#### 6. Statistics

```
GET /api/v1/officer/statistics?period=weekly
```

---

## Police Station App

### Screen List

| # | Screen | Description |
|---|--------|-------------|
| 1 | **Login** | Email + password login |
| 2 | **Dashboard** | Overview stats cards + recent crimes |
| 3 | **Officers List** | All officers with status indicators |
| 4 | **Officer Create/Edit** | Add/edit officer form |
| 5 | **Officers Import** | Upload Excel file to bulk create officers |
| 6 | **Officers Map** | Real-time map showing all officer locations |
| 7 | **Cameras List** | All cameras with status (active/inactive) |
| 8 | **Camera Create/Edit** | Add/edit camera form |
| 9 | **Cameras Import** | Upload Excel to bulk create cameras |
| 10 | **Camera Live View** | Live RTSP stream from camera using flutter_vlc_player/media_kit |
| 11 | **Camera Control** | Full camera control panel (alarm, PTZ, settings, recordings) |
| 12 | **Crimes List** | All crimes with filters (status, date, officer, camera) |
| 13 | **Crime Detail** | Full crime info + evidence + assigned officer |
| 14 | **Crime Manual Assignment** | Assign crime to specific officer |
| 15 | **Chat Conversations** | List of chats with officers |
| 16 | **Chat Thread** | Chat with specific officer |
| 17 | **Profile** | View/edit station profile (name, email, phone, address, city, governorate, avatar) |
| 18 | **Notifications** | List of all notifications |
| 19 | **Statistics** | Crime stats by period |

### Screen Details

#### 1. Dashboard

```
GET /api/v1/police-station/dashboard
```

**Response:**
```json
{
  "status": true,
  "data": {
    "total_officers": 20,
    "available_officers": 12,
    "total_cameras": 15,
    "active_cameras": 13,
    "total_crimes": 150,
    "pending_crimes": 3,
    "crimes_today": 5,
    "recent_crimes": [ ... ],
    "crimes_by_status": {
      "pending": 3,
      "assigned": 2,
      "in_progress": 4,
      "visited": 1,
      "resolved": 120,
      "not_visited": 10,
      "escalated": 5,
      "false_alarm": 5
    }
  }
}
```

#### 2. Officers CRUD

```
GET    /api/v1/police-station/officers              → List (paginated, searchable)
POST   /api/v1/police-station/officers              → Create
PUT    /api/v1/police-station/officers/{id}          → Update
DELETE /api/v1/police-station/officers/{id}          → Delete
```

**Create/Update Body:**
```json
{
  "name": "Ahmed Mohamed",
  "email": "ahmed@police.gov.eg",
  "phone": "01012345678",
  "password": "securepass123",
  "national_id": "29901011234567",
  "rank": "Lieutenant",
  "badge_number": "B-1234",
  "is_on_shift": true
}
```

**Reset Officer Password:**
```
PUT /api/v1/police-station/officers/{id}/reset-password
Body: { "password": "newSecurePassword" }
```

**Officers on Map:**
```
GET /api/v1/police-station/officers/locations
```

Returns all officers with their latest GPS coordinates. Use to:
- Show officers on a Google Map
- Color-coded by status (green = available, yellow = busy, red = offline)
- Update in real-time via Pusher (`OfficerLocationUpdated` event)

#### 3. Officers Import (Excel)

```
POST /api/v1/police-station/officers/import
Content-Type: multipart/form-data
Body: file={excel_file}
```

**Excel format (columns):**
| name | email | phone | password | national_id | rank | badge_number |
|------|-------|-------|----------|-------------|------|-------------|
| Ahmed | ahmed@police.gov.eg | 01012345678 | pass123 | 29901011234567 | Lieutenant | B-1234 |

**Response:**
```json
{
  "status": true,
  "message": "Import completed",
  "data": {
    "total_rows": 20,
    "success_count": 18,
    "error_count": 2,
    "errors": [
      { "row": 5, "message": "Email already exists" },
      { "row": 12, "message": "Phone number is invalid" }
    ]
  }
}
```

#### 4. Cameras CRUD

```
GET    /api/v1/police-station/cameras              → List
POST   /api/v1/police-station/cameras              → Create
PUT    /api/v1/police-station/cameras/{id}          → Update
DELETE /api/v1/police-station/cameras/{id}          → Delete
POST   /api/v1/police-station/cameras/import        → Excel import
```

**Create/Update Body:**
```json
{
  "name": "Main Entrance Camera",
  "ip_address": "192.168.1.6",
  "user_name": "admin",
  "password": "camera123",
  "tapo_email": "user@email.com",
  "tapo_password": "tapopass",
  "location_name": "Building A Entrance",
  "lat": 30.0444,
  "lang": 31.2357,
  "storage_type": "sd_card",
  "is_active": true,
  "connection_ip": "41.35.120.50",
  "connection_port": 8554,
  "api_port": 8443,
  "onvif_port": 8020
}
```

> **Note on connection fields:** If the camera is on the same local network as the server, you can leave `connection_ip` as null and `connection_port` as 554 (defaults). If the camera is on a remote network (e.g., deployed on a street with different WiFi), set `connection_ip` to the router's public IP and `connection_port` to the forwarded RTSP port.

#### 5. Camera Live View (RTSP Stream)

The police station can view live camera feeds directly in the Flutter app using RTSP streaming.

```
GET /api/v1/police-station/cameras/{id}/live
```

**Response:**
```json
{
  "status": true,
  "data": {
    "camera_id": 3,
    "camera_name": "Main Entrance",
    "encrypted_stream_data": "BASE64_AES_ENCRYPTED_STRING"
  }
}
```

**Decrypted `encrypted_stream_data` contains:**
```json
{
  "rtsp_url": "rtsp://admin:camera123@41.35.100.50:8554/stream1",
  "rtsp_url_sub": "rtsp://admin:camera123@41.35.100.50:8554/stream2"
}
```

- Use `stream1` for high-quality view, `stream2` for lower bandwidth
- **Decrypt** the `encrypted_stream_data` using the `encryption_key` from login (see Encryption section above)
- Use `flutter_vlc_player` or `media_kit` package to play RTSP:

```dart
// Using flutter_vlc_player
VlcPlayerController.network(
  decryptedData['rtsp_url_sub'], // Use stream2 for mobile
  hwAcc: HwAcc.full,
  autoPlay: true,
  options: VlcPlayerOptions(),
);

// Using media_kit
final player = Player();
player.open(Media(decryptedData['rtsp_url_sub']));
```

**Important:** RTSP streaming is supported on **Android and iOS only** — NOT on web.

#### 6. Camera Control

Police stations have full camera control via these endpoints:

```
POST /api/v1/police-station/cameras/{camera}/control/alarm        → on|off|start|stop|status|config
POST /api/v1/police-station/cameras/{camera}/control/privacy      → on|off|status
POST /api/v1/police-station/cameras/{camera}/control/led          → on|off|status
POST /api/v1/police-station/cameras/{camera}/control/motion       → on|off|status
POST /api/v1/police-station/cameras/{camera}/control/person       → on|off|status
POST /api/v1/police-station/cameras/{camera}/control/move         → PTZ direction
POST /api/v1/police-station/cameras/{camera}/control/goto-preset  → Go to preset
POST /api/v1/police-station/cameras/{camera}/control/motor        → move|calibrate
POST /api/v1/police-station/cameras/{camera}/control/autotrack    → on|off|status
POST /api/v1/police-station/cameras/{camera}/control/cruise       → on|off|status
POST /api/v1/police-station/cameras/{camera}/control/preset       → list|set|save|delete
POST /api/v1/police-station/cameras/{camera}/control/recordings   → list|get
POST /api/v1/police-station/cameras/{camera}/control/stream-url   → Get RTSP URL
POST /api/v1/police-station/cameras/{camera}/control/basic-info   → Device info
POST /api/v1/police-station/cameras/{camera}/control/status       → Full status
POST /api/v1/police-station/cameras/{camera}/control/daynight     → set|status|config
POST /api/v1/police-station/cameras/{camera}/control/flip         → on|off|status
POST /api/v1/police-station/cameras/{camera}/control/ldc          → on|off|status
POST /api/v1/police-station/cameras/{camera}/control/osd          → set|status
POST /api/v1/police-station/cameras/{camera}/control/sdcard       → status|format
POST /api/v1/police-station/cameras/{camera}/control/reboot       → Reboot camera
```

#### 7. Crimes

```
GET  /api/v1/police-station/crimes                 → List with filters
GET  /api/v1/police-station/crimes/{id}             → Detail
POST /api/v1/police-station/crimes/{id}/assign      → Manual assignment
```

**Filters (query params):**
- `status` — filter by crime status
- `severity` — filter by severity
- `camera_id` — filter by camera
- `officer_id` — filter by assigned officer
- `date_from`, `date_to` — date range
- `page` — pagination

**Manual Assignment:**
```json
POST /api/v1/police-station/crimes/{id}/assign
Body: { "officer_id": 5 }
```

#### 8. Chat

```
GET  /api/v1/police-station/chat/conversations     → List all officer chats
GET  /api/v1/police-station/chat/{officer_id}       → Messages with officer
POST /api/v1/police-station/chat/{officer_id}       → Send message
```

---

## API Endpoints Reference

### Base URL: `/api/v1`

### Shared Endpoints (both apps use same pattern, replace `{guard}`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/{guard}/login` | Login (returns token + encryption_key) |
| POST | `/{guard}/logout` | Logout |
| GET | `/{guard}/profile` | Get profile |
| PUT | `/{guard}/profile` | Update profile |
| GET | `/{guard}/notifications` | List notifications (paginated) |
| PUT | `/{guard}/notifications/{id}/read` | Mark notification as read |
| GET | `/{guard}/notifications/unread-count` | Get unread count |
| POST | `/{guard}/firebase-token` | Register FCM token |
| DELETE | `/{guard}/firebase-token` | Remove FCM token |
| GET | `/{guard}/dashboard` | Dashboard stats |
| GET | `/{guard}/statistics` | Statistics by period |

### Public Endpoints (no auth required)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/settings/language` | Get system language (ar/en) |

### Officer-Only Endpoints (`{guard}` = `officer`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/officer/crimes` | Nearby crimes (within 5km) |
| GET | `/officer/crimes/{id}` | Crime detail |
| PUT | `/officer/crimes/{id}/accept` | Accept crime assignment |
| PUT | `/officer/crimes/{id}/arrive` | Mark arrived at scene |
| PUT | `/officer/crimes/{id}/no-visit` | Decline with reason |
| PUT | `/officer/crimes/{id}/resolve` | Resolve crime |
| POST | `/officer/location` | Update GPS location |
| PUT | `/officer/status` | Update availability status |
| GET | `/officer/chat` | Chat messages with station |
| POST | `/officer/chat` | Send message to station |
| POST | `/officer/cameras/{camera}/control/alarm` | Trigger/stop camera alarm (start\|stop only) |
| POST | `/officer/cameras/{camera}/control/move` | PTZ camera movement |
| POST | `/officer/cameras/{camera}/control/goto-preset` | Go to camera preset |
| POST | `/officer/cameras/{camera}/control/preset` | List presets only |
| POST | `/officer/cameras/{camera}/control/recordings` | List/get recordings |
| POST | `/officer/cameras/{camera}/control/stream-url` | Get RTSP stream URL |

### Police Station-Only Endpoints (`{guard}` = `police-station`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/police-station/officers` | List officers |
| POST | `/police-station/officers` | Create officer |
| PUT | `/police-station/officers/{id}` | Update officer |
| DELETE | `/police-station/officers/{id}` | Delete officer |
| POST | `/police-station/officers/import` | Excel import |
| GET | `/police-station/officers/locations` | Officers GPS for map |
| GET | `/police-station/cameras` | List cameras |
| POST | `/police-station/cameras` | Create camera |
| PUT | `/police-station/cameras/{id}` | Update camera |
| DELETE | `/police-station/cameras/{id}` | Delete camera |
| POST | `/police-station/cameras/import` | Excel import |
| GET | `/police-station/crimes` | List crimes (filterable) |
| GET | `/police-station/crimes/{id}` | Crime detail |
| POST | `/police-station/crimes/{id}/assign` | Assign officer |
| GET | `/police-station/chat/conversations` | Chat list |
| GET | `/police-station/chat/{officer}` | Messages with officer |
| POST | `/police-station/chat/{officer}` | Send message |
| PUT | `/police-station/officers/{id}/reset-password` | Reset officer password |
| GET | `/police-station/cameras/{id}/live` | Get encrypted live stream URL |
| POST | `/police-station/cameras/{camera}/control/*` | Full camera control (21 endpoints) |

---

## Real-time Integration (Pusher)

### Setup

Use `pusher_channels_flutter` package. Credentials will be provided by the backend developer.

```dart
// Initialize Pusher
final pusher = PusherChannelsFlutter.getInstance();
await pusher.init(
  apiKey: "YOUR_PUSHER_KEY",
  cluster: "YOUR_CLUSTER",
  authEndpoint: "https://{server}/broadcasting/auth", // For private channels
  auth: PusherAuth(headers: {"Authorization": "Bearer $token"}),
);
await pusher.connect();
```

### Channels to Subscribe

#### Officer App:
```dart
// Private channel for this specific officer
await pusher.subscribe(channelName: "private-officer.{officer_id}");
```

#### Police Station App:
```dart
// Private channel for this specific station
await pusher.subscribe(channelName: "private-police-station.{station_id}");

// Presence channel — see which officers are online
await pusher.subscribe(channelName: "presence-police-station.{station_id}.officers");
```

### Events to Listen For

| Event Name | Channel | Data | Action |
|------------|---------|------|--------|
| `CrimeDetected` | `private-police-station.{id}`, `private-officer.{id}` | Crime object | Show new crime alert, add to list |
| `CrimeStatusUpdated` | `private-police-station.{id}` | `{crime_id, status, officer_id}` | Update crime card status |
| `OfficerLocationUpdated` | `private-police-station.{id}` | `{officer_id, lat, lng}` | Move officer pin on map |
| `SuspiciousActivityAlert` | `private-police-station.{id}` | `{camera_id, confidence}` | Show warning banner |
| `CrimeEscalated` | `private-police-station.{id}`, `private-officer.{id}` | `{crime_id}` | Show escalation alert |
| `ChatMessageSent` | `private-chat.{conversation_key}` | Message object | Add message to chat UI |
| `NewNotification` | `private-officer.{id}`, `private-police-station.{id}` | Notification object | Show notification badge, increment count |

---

## Push Notifications (Firebase)

### Setup

1. Add `firebase_messaging` and `firebase_core` to your Flutter project
2. Get `google-services.json` (Android) and `GoogleService-Info.plist` (iOS) from Firebase console
3. Request notification permissions on app start

### Register FCM Token

After login AND every time the FCM token refreshes:

```
POST /api/v1/{guard}/firebase-token
Body: {
  "token": "fcm_device_token_here",
  "device_type": "android"  // or "ios"
}
```

### Remove Token on Logout

```
DELETE /api/v1/{guard}/firebase-token
Body: { "token": "fcm_device_token_here" }
```

### Notification Payload Structure

```json
{
  "notification": {
    "title": "🚨 Crime Detected",
    "body": "Theft detected at Camera: Main Entrance"
  },
  "data": {
    "type": "crime_detected",
    "crime_id": "15",
    "click_action": "OPEN_CRIME_DETAIL"
  }
}
```

### Notification Types

| `data.type` | Description | Action on Tap |
|-------------|-------------|---------------|
| `crime_detected` | New crime assigned to officer | Navigate to Crime Detail |
| `crime_assigned` | Crime assigned (for station) | Navigate to Crime Detail |
| `crime_escalated` | Crime escalated | Navigate to Crime Detail |
| `suspicious_activity` | Preliminary alert | Show alert dialog |
| `crime_status_updated` | Officer accepted/arrived/etc. | Refresh crime list |
| `new_chat_message` | New chat message | Navigate to Chat |

### Handle Notifications

```dart
// Foreground — show in-app banner
FirebaseMessaging.onMessage.listen((message) {
  // Show local notification or in-app banner
});

// Background — user taps notification
FirebaseMessaging.onMessageOpenedApp.listen((message) {
  // Navigate to appropriate screen based on data.type
});

// Terminated — user taps notification
FirebaseMessaging.instance.getInitialMessage().then((message) {
  // Navigate to appropriate screen
});
```

---

## Chat System

### How It Works

- **Officer** chats only with their police station (1-to-1)
- **Police Station** chats with any of their officers (1-to-many conversations)
- Messages are delivered in real-time via Pusher
- Push notification sent when recipient's app is in background
- Supports: text messages only (no file/image attachments)

### Chat Conversation Key

Each conversation has a unique key used as the Pusher channel name:

- Station ↔ Officer: `station_{station_id}_officer_{officer_id}`

Subscribe to: `private-chat.station_1_officer_5`

### Message Object

```json
{
  "id": 1,
  "message": "Report to sector 5 immediately",
  "sender": { "id": 2, "name": "Station Commander", "type": "police_station" },
  "is_read": false,
  "created_at": "2026-02-27T14:30:00Z"
}
```

### Sending Messages

```json
POST /api/v1/officer/chat
// or
POST /api/v1/police-station/chat/{officer_id}

Body: {
  "message": "I'm on my way to the scene"
}
```

### Read Receipts

Messages have `is_read` and `read_at` fields. When user opens a chat, call the backend to mark messages as read (endpoint TBD — may be automatic on fetch).

---

## GPS Location Tracking

### Officer App Only

The officer app **must** track GPS location in the background and send updates when the officer moves 100+ meters.

### Implementation

```dart
// Use geolocator package
// Settings:
// - distanceFilter: 100 (meters) — only trigger when moved 100m
// - accuracy: LocationAccuracy.high
// - Background mode: enabled

Geolocator.getPositionStream(
  locationSettings: LocationSettings(
    accuracy: LocationAccuracy.high,
    distanceFilter: 100, // Only fire when moved 100m
  ),
).listen((Position position) {
  // Send to server
  api.post('/api/v1/officer/location', body: {
    'latitude': position.latitude,
    'longitude': position.longitude,
  });
});
```

### Important:
- Request location permission (always/when-in-use)
- Handle permission denied gracefully
- Show location permission rationale
- On Android: use foreground service for background tracking
- Battery optimization: 100m filter prevents excessive API calls

---

## Error Handling

### Standard Error Response

All API errors follow this format:

```json
{
  "status": false,
  "message": "Error description in user's language",
  "errors": {
    "field_name": ["Specific field error"]
  }
}
```

### HTTP Status Codes

| Code | Meaning | Action |
|------|---------|--------|
| 200 | Success | Parse response |
| 201 | Created | Resource created successfully |
| 401 | Unauthorized | Token expired → redirect to login |
| 403 | Forbidden | No permission for this action |
| 404 | Not Found | Resource doesn't exist |
| 422 | Validation Error | Show field errors |
| 429 | Too Many Requests | Rate limited — retry after delay |
| 500 | Server Error | Show generic error message |

### Token Expiration

If you get a `401` response, the token is expired or invalid:
1. Clear stored token and encryption key
2. Redirect to login screen
3. Show "Session expired, please login again"

Note: Tokens have a **7-day expiry**. After 7 days, the user must login again (which also regenerates the encryption key).

---

## Data Models

### Crime Status Flow

```
pending → assigned → in_progress → visited → resolved
                  ↘ not_visited
                  ↘ escalated → assigned (to next officer)
                  ↘ false_alarm
```

### Crime Severity (Color Coding)

| Severity | Color | Confidence Score |
|----------|-------|-----------------|
| `low` | 🟢 Green | < 40% |
| `medium` | 🟡 Yellow | 40-59% |
| `high` | 🟠 Orange | 60-79% |
| `critical` | 🔴 Red | ≥ 80% |

### Officer Status

| Status | Meaning | Color |
|--------|---------|-------|
| `available` | Ready for assignments | 🟢 Green |
| `busy` | Currently handling a crime | 🟡 Yellow |
| `offline` | Not available | 🔴 Red |

---

## Important Notes

### 1. Language Support (Server-Controlled)
- The app language is controlled by the **admin** from the dashboard — NOT by the user
- On app startup (before login), call `GET /api/v1/settings/language` to get the system language
- Supported languages: Arabic (`ar`, RTL) and English (`en`, LTR)
- Crime types have both `name_en` and `name_ar` — display based on system language
- Error messages come from the server in the configured system language
- Cache the language locally for offline use, refresh on every startup
- Do NOT add any language toggle or language picker in the app UI

### 2. Pagination
- All list endpoints support pagination: `?page=1`
- Response includes `meta.current_page`, `meta.last_page`, `meta.total`
- Implement infinite scroll or numbered pagination

### 3. Search & Filters
- Officer list: search by name or email
- Crime list: filter by status, severity, date range, camera, officer
- Pass filters as query params: `?status=pending&severity=high`

### 4. Media URLs
- Scene evidence path: prepend base URL → `https://{server}{sence_path}`
- Avatar paths: prepend base URL → `https://{server}{avatar_path}`

### 5. Date/Time Format
- All dates from API are in ISO 8601 format: `2026-02-27T14:30:00Z`
- Display in local timezone
- Use relative time for recent items ("5 minutes ago", "yesterday")

### 6. Offline Handling
- Cache critical data locally (officer list, recent crimes)
- Show cached data when offline with "offline" indicator
- Queue actions (location updates, status changes) and sync when back online

### 7. Testing
- Backend will provide seed data (test users) for development
- Use these test credentials during development:
  - Will be shared separately once auth is implemented

### 8. Excel Import Template
- The backend will provide Excel template files for officers and cameras import
- Important: the first row must be headers
- Validate file type before upload (.xlsx, .xls, .csv)

---

## Development Order (Suggested)

Follow this order to align with backend development:

| Week | Tasks |
|------|-------|
| Week 1 | Project setup, login/auth screens, connect to backend auth API |
| Week 2 | Officer: Crime list + detail + actions. Station: Officers CRUD + Cameras CRUD |
| Week 3 | Officer: Map + GPS tracking. Station: Crimes list + detail + manual assignment |
| Week 4 | Officers map (station), Excel import UI, Statistics screens |
| Week 5 | Pusher integration (real-time), Firebase push notifications |
| Week 6 | Chat UI + real-time messages, Profile pages, Notifications list |
| Week 7 | Polish: dark mode, animations, loading states, error handling |
| Week 8 | Testing, bug fixes, performance optimization |
