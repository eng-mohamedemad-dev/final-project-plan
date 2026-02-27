# Plan: Crime Prediction System — Full Architecture & Implementation Plan

## TL;DR

Build a multi-guard crime prediction system where AI models monitor Tapo C200 cameras, detect crimes, and alert nearby officers and police stations. The system has 3 user types (admins, officers, police_stations) each with separate auth, dashboards, and mobile APIs. The existing camera control API (35+ endpoints with Tapo/ONVIF integration) serves as the foundation. We'll add domain models matching the ERD, multi-guard Sanctum authentication, an admin Livewire dashboard, a RESTful API for the Flutter mobile app, Pusher for real-time updates, and Firebase for push notifications.

**Key Assumptions:**
- The AI model runs as a separate server — it fetches cameras from Laravel, watches streams via RTSP, and calls a Laravel webhook API when it detects a crime. The `model_ip_address` field on cameras is used to "claim" a camera so other model instances don't process it.
- **Load balancing**: Each model instance handles a maximum of **10 cameras**. When a model claims cameras, it sets its `model_ip_address`. Other model instances calling the API only receive unclaimed cameras. This distributes the processing load across multiple model servers.
- **Camera alarm**: The AI model triggers the camera alarm **directly** (since it already has RTSP access to the camera). This is faster than routing through Laravel. The model calls the camera's alarm API immediately upon detection.
- **Scene recording — dual strategy**:
  - If camera has storage (SD card or cloud) → `storage_type = 'sd_card'` or `'cloud'` → Laravel fetches the recording from the camera using the existing recording API endpoints.
  - If camera has NO storage → `storage_type = 'none'` → Laravel runs a **continuous ffmpeg recording service** that records RTSP segments (1 minute each) to the server. When a crime is reported, Laravel finds the relevant segments by timestamp, merges them into a single evidence file in `storage/app/crimes/`, and cleans up old segments.
- `sence_path` = file path to the recorded video evidence (either fetched from camera or assembled from ffmpeg segments).
- **RTSP URL**: The full RTSP URL (e.g. `rtsp://user:pass@192.168.1.6:554/stream2`) is returned as a complete field when serving cameras to the model, so the model connects directly without constructing URLs.
- **Remote cameras (different networks)**: Cameras deployed on the street (different WiFi networks) need port forwarding on their router to be accessible. The camera model stores both `ip_address` (for display) and `connection_ip` + `connection_port` (for actual RTSP/API access). The `rtsp_url` accessor uses `connection_ip:connection_port`. For cameras on the same LAN, `connection_ip` = `ip_address` and `connection_port` = `554`. For remote cameras, `connection_ip` = public IP of the router + port forwarded to camera RTSP port.
- Officers see crimes within 5km of their GPS location.
- Firebase tokens are polymorphic for push notifications to all 3 user types.
- Camera `stream_path` stores RTSP stream URLs (stream1 = high quality for recording, stream2 = low quality for AI processing).
- Crime `status` values: `pending`, `assigned`, `in_progress`, `visited`, `not_visited`, `resolved`, `escalated`, `false_alarm`.
- Crime `severity` levels: `low`, `medium`, `high`, `critical` — based on AI model confidence score.

---

## Crime Detection Flow (End-to-End)

```
1. AI Model fetches unclaimed cameras from Laravel (GET /api/model/cameras)
   - Receives max 10 cameras with full RTSP URLs (stream2 / low quality)
   - Claims them (POST /api/model/cameras/{id}/claim) → sets model_ip_address
   - Sends heartbeat every 60s (POST /api/model/heartbeat)

2. Model watches camera RTSP stream, detects suspicious activity (confidence rises)

3. Model triggers camera alarm DIRECTLY (faster than routing through Laravel)
   - Model has camera credentials from step 1
   - Uses camera alarm API directly: POST http://{camera_ip}/alarm/start

4. Model sends POST /api/model/alert → Laravel:
   - camera_id, confidence_score, alert_type (green→yellow→red)
   - Laravel sends preliminary "⚠️ Suspicious activity" notification to police station
   - Laravel logs alert in activity_logs

5. Model confirms crime POST /api/model/crime → Laravel:
   - camera_id, start_time, end_time, crime_type, confidence_score
   - Laravel checks camera storage_type:
     a. If 'sd_card' or 'cloud' → fetches recording from camera using existing API
     b. If 'none' → finds matching ffmpeg segments on server, merges into single file:
        ffmpeg -i "concat:seg1.mkv|seg2.mkv" -c:v copy -c:a aac -b:a 64k output.mp4
   - Saves evidence to storage/app/crimes/{crime_id}/
   - Creates Scene record (with video path + auto-generated thumbnail)
   - Creates Crime record (status: pending, severity based on confidence)
   - Finds nearest available officer (Haversine, same police station, within 5km)
   - Assigns officer → status: assigned
   - Sends Firebase push notification to officer + police station
   - Broadcasts Pusher event to police station + admin dashboard

6. Officer receives notification in Flutter app:
   - Sees crime details, location on map, evidence video
   - Accepts → status: in_progress
   - Arrives → status: visited, officer_arrive_time recorded
   - OR declines → status: not_visited, no_visit_reason required

7. If officer doesn't respond within X minutes → ESCALATION:
   - Auto-assign to next nearest officer
   - Notify police station of escalation
   - status: escalated

8. Police station sees real-time updates on dashboard map

9. After resolution → status: resolved

10. If false alarm → status: false_alarm (feeds back to improve model)

11. Model sends POST /api/model/alert/stop when threat cleared
    - Model stops camera alarm directly
```

---

## Phase 1: Database Schema & Models

1. Create the `admins` migration and model at `app/Models/Admin.php` with fields: `id`, `name`, `email`, `phone`, `password`, timestamps. Use `Authenticatable` base class. Add `HasFactory`, `Notifiable`, `HasApiTokens` traits.

2. Create the `police_stations` migration and model at `app/Models/PoliceStation.php` with fields: `id`, `name`, `email`, `phone`, `password`, timestamps. Use `Authenticatable`. Relationships: `hasMany(Officer)`, `hasMany(Camera)`, `hasMany(Crime)` (through cameras), `morphMany(FirebaseToken)`, `morphMany(Notification)`.

3. Create the `officers` migration and model at `app/Models/Officer.php` with fields: `id`, `police_station_id` (FK), `name`, `email`, `phone`, `password`, `latitude` (decimal 10,7), `longitude` (decimal 10,7), `last_location_update` (timestamp, nullable), `status` (enum: `available`, `busy`, `offline`, default `offline`), `is_on_shift` (boolean, default false), timestamps. Use `Authenticatable`. Relationships: `belongsTo(PoliceStation)`, `hasMany(Crime)`, `morphMany(FirebaseToken)`, `morphMany(Notification)`.

4. Create the `cameras` migration and model at `app/Models/Camera.php` with all ERD fields: `id`, `name`, `police_station_id` (FK), `ip_address`, `user_name`, `password`, `tapo_email`, `tapo_password`, `stream_path` (JSON for stream1/stream2), `location_name`, `lat` (decimal 10,7), `lang` (decimal 10,7), `model_ip_address` (nullable), `is_active` (boolean, default false), `storage_type` (enum: `sd_card`, `cloud`, `none`, default `none` — determines recording strategy), `connection_ip` (nullable — public/VPN IP for remote cameras; if null, falls back to `ip_address`), `connection_port` (integer, default 554 — RTSP port, can be a forwarded port for remote cameras), `api_port` (integer, default 443 — Tapo HTTPS API port, can be forwarded), `onvif_port` (integer, default 2020 — ONVIF port, can be forwarded), timestamps. Relationships: `belongsTo(PoliceStation)`, `hasMany(Crime)`. Add unique constraint on `ip_address`. Add accessor `rtsp_url` that builds the full RTSP URL using `connection_ip` (or `ip_address` as fallback) and `connection_port`: `rtsp://{user_name}:{password}@{effective_ip}:{connection_port}/stream2`. Add accessor `effective_ip` that returns `connection_ip ?? ip_address`. Add accessor `effective_api_url` that returns `https://{effective_ip}:{api_port}`.

5. Create the `crime_types` migration and model at `app/Models/CrimeType.php` with fields: `id`, `name_en`, `name_ar`, `description`, `default_severity` (enum: low/medium/high/critical), timestamps. Seed with common crime types (theft, assault, vandalism, break-in, suspicious_activity, etc.).

6. Create the `scenses` migration and model at `app/Models/Scene.php` with fields: `id`, `start_date`, `end_date`, `crime_id` (FK, nullable initially), `sence_path`, `thumbnail_path` (nullable — extracted frame for quick preview), timestamps. Relationship: `belongsTo(Crime)`.

7. Create the `crimes` migration and model at `app/Models/Crime.php` with fields: `id`, `officer_id` (FK, nullable), `camera_id` (FK), `scene_id` (FK, nullable), `crime_type_id` (FK, nullable), `crime_date`, `status` (enum: pending/assigned/in_progress/visited/not_visited/resolved/escalated/false_alarm), `severity` (enum: low/medium/high/critical), `confidence_score` (decimal 5,2 — from AI model, 0-100%), `officer_arrive_time` (nullable), `no_visit_reason` (nullable text), `escalation_count` (integer, default 0), `response_time_minutes` (integer, nullable — auto-calculated), timestamps. Relationships: `belongsTo(Officer)`, `belongsTo(Camera)`, `belongsTo(CrimeType)`, `hasOne(Scene)`. Scopes: `pending()`, `active()`, `byRadius($lat, $lng, $km)`.

8. Create the `firebase_tokens` migration and model at `app/Models/FirebaseToken.php` with polymorphic fields: `id`, `token`, `tokenable_name`, `tokenable_id`, `device_type` (enum: android/ios/web), timestamps. Add `morphTo()` relationship. Unique constraint on `token`.

9. Create the `notifications` table using Laravel's built-in database notification migration with customization. Fields: `id` (uuid), `type`, `notifiable_type`, `notifiable_id`, `data` (JSON), `is_read` (boolean, default false), `read_at` (nullable), timestamps. Polymorphic `morphTo()`.

10. Create the `settings` migration and model at `app/Models/Setting.php` with fields: `id`, `key` (unique), `value` (text, nullable), `group` (string — e.g. firebase, system, notification), timestamps. Helper methods: `Setting::get('key', 'default')`, `Setting::set('key', 'value')`.

11. Create the `activity_logs` migration and model at `app/Models/ActivityLog.php` with fields: `id`, `loggable_type`, `loggable_id` (polymorphic — who did it), `action` (string — e.g. `crime.assigned`, `officer.location_updated`), `description` (text), `properties` (JSON — extra data), `ip_address`, timestamps. For audit trail and accountability.

12. Create factories and seeders for all models. Seed with realistic test data including:
    - 2 admins
    - 5 police stations (with Egyptian city names)
    - 20 officers (4 per station, with random GPS coordinates)
    - 15 cameras (3 per station, with realistic IP addresses)
    - 10 crime types
    - 30 sample crimes with varying statuses and severities

---

## Phase 2: Backend Architecture & Clean Code Conventions

This section defines the architecture rules that **every developer must follow** to keep the codebase clean, maintainable, and consistent.

### 2.1 Controller Rules — Thin Controllers

Controllers are **entry points only**. They must NOT contain business logic. Their job:
1. Accept the request (already validated by FormRequest)
2. Call the appropriate Service
3. Return the response (using API Resource or view)

```php
// ✅ GOOD — thin controller
class CrimeController extends Controller
{
    public function store(StoreCrimeRequest $request, CrimeService $crimeService): JsonResponse
    {
        $crime = $crimeService->createFromModelReport($request->validated());

        return $this->success(new CrimeResource($crime), __('messages.crime.created'), 201);
    }
}

// ❌ BAD — fat controller with inline logic
class CrimeController extends Controller
{
    public function store(Request $request): JsonResponse
    {
        $request->validate([...]);           // validation in controller
        $scene = Scene::create([...]);       // business logic in controller
        $crime = Crime::create([...]);       // more logic
        $officer = Officer::where(...);      // query logic
        // ... 50 more lines
    }
}
```

### 2.2 Service Layer — All Business Logic Here

Each domain has its own Service class in `app/Services/`. Services contain the actual business logic and are injected into controllers via constructor injection or method injection.

```
app/Services/
  Auth/             AuthService.php (shared auth logic)
  Crime/            CrimeService.php
  Officer/          OfficerService.php
  PoliceStation/    PoliceStationService.php
  Camera/           CameraService.php, CameraRecordingService.php, CameraHealthService.php
  Scene/            SceneExtractorService.php
  Notification/     FirebaseService.php, NotificationService.php
  Escalation/       EscalationService.php
  Import/           ExcelImportService.php
  Location/         NearestOfficerService.php
  Report/           ReportService.php
  Settings/         SettingsService.php
  Chat/             ChatService.php
```

### 2.3 FormRequest Classes — All Validation External

NO inline validation in controllers. Every endpoint gets its own FormRequest:

```
app/Http/Requests/
  Auth/               LoginRequest.php, UpdateProfileRequest.php
  Officer/            StoreOfficerRequest.php, UpdateOfficerRequest.php
  PoliceStation/      StorePoliceStationRequest.php, UpdatePoliceStationRequest.php
  Camera/             StoreCameraRequest.php, UpdateCameraRequest.php
  Crime/              StoreCrimeRequest.php, UpdateCrimeStatusRequest.php, NoVisitRequest.php
  Import/             ImportExcelRequest.php
  Chat/               SendMessageRequest.php
  Model/              AlertRequest.php, CrimeReportRequest.php, ClaimCameraRequest.php
  Settings/           UpdateSettingsRequest.php
```

All inherit from `BaseRequest` (already exists) which uses `ApiResponse` trait for consistent validation error format.

### 2.4 Eloquent API Resources — All Responses Formatted

Never return raw models in API responses. Always use Resources:

```
app/Http/Resources/
  AdminResource.php
  OfficerResource.php
  PoliceStationResource.php
  CameraResource.php (hides passwords, shows rtsp_url only to model API)
  CrimeResource.php (includes scene, officer, camera relations)
  SceneResource.php
  CrimeTypeResource.php
  NotificationResource.php
  ChatMessageResource.php
  ChatConversationResource.php
  DashboardResource.php
  ModelCameraResource.php (special resource for AI model — includes rtsp_url, credentials)
```

### 2.5 Observers — Automatic Side Effects

Use Observers for automatic actions triggered by model events. Never put these in controllers.

```
app/Observers/
  CrimeObserver.php
    - created()  → log in activity_logs, broadcast CrimeDetected event
    - updated()  → if status changed: broadcast CrimeStatusUpdated, calculate response_time

  OfficerObserver.php
    - updated()  → if location changed: broadcast OfficerLocationUpdated event
    - updated()  → if status changed to 'offline': release any assigned pending crimes

  CameraObserver.php
    - updated()  → if is_active changed: start/stop ffmpeg recording (for storage_type='none')
    - updated()  → if storage_type changed: manage recording service accordingly
    - deleting() → stop recording, release from model, cleanup recordings

  PoliceStationObserver.php
    - deleting() → cascade: deactivate cameras, unassign officers (prevent orphans)
```

### 2.6 Background Jobs — Heavy Operations Off the Request Cycle

Any operation that takes > 1 second goes into a queued job:

```
app/Jobs/
  SendCrimeNotification.php       → sends FCM + Pusher (called after crime assignment)
  ExtractCrimeScene.php           → fetches/merges video evidence (can take 10-30s)
  ProcessExcelImport.php          → processes uploaded Excel in background
  CheckCameraHealth.php           → pings camera, updates status
  GenerateReport.php              → generates PDF/Excel reports
  CleanupOldRecordings.php        → deletes old ffmpeg segments
  SendChatNotification.php        → sends FCM for new chat message
```

All jobs implement `ShouldQueue` and use Redis driver.

### 2.7 Livewire Components — Logic Separate from View

Every Livewire component has **two files**: a PHP class (logic) and a Blade view (template). They are never in the same file.

```
// PHP class — logic only, no HTML
app/Livewire/Admin/Dashboard.php
app/Livewire/Admin/PoliceStations/Index.php
app/Livewire/Admin/PoliceStations/Create.php
app/Livewire/Admin/PoliceStations/Edit.php
app/Livewire/Admin/Settings/Index.php
app/Livewire/Admin/Profile.php
app/Livewire/Admin/Notifications.php
app/Livewire/Admin/Auth/Login.php
app/Livewire/Admin/Chat/Index.php
app/Livewire/Admin/Chat/Conversation.php

// Blade views — presentation only, minimal logic
resources/views/livewire/admin/dashboard.blade.php
resources/views/livewire/admin/police-stations/index.blade.php
resources/views/livewire/admin/police-stations/create.blade.php
resources/views/livewire/admin/police-stations/edit.blade.php
resources/views/livewire/admin/settings/index.blade.php
resources/views/livewire/admin/profile.blade.php
resources/views/livewire/admin/notifications.blade.php
resources/views/livewire/admin/auth/login.blade.php
resources/views/livewire/admin/chat/index.blade.php
resources/views/livewire/admin/chat/conversation.blade.php
```

Blade views should contain **only**:
- HTML structure + Tailwind classes
- `wire:model`, `wire:click`, etc. directives
- Minimal `@if` / `@foreach` for rendering
- **No** PHP logic, complex expressions, or queries

### 2.8 Shared Routes Pattern — `/{guard}/...`

Common functionality shared between admins, police stations, and officers uses a **dynamic guard segment** in the URL instead of duplicating controllers:

```php
// routes/api.php

// Shared auth routes — one controller handles all 3 guards
Route::prefix('v1/{guard}')->where(['guard' => 'admin|police-station|officer'])->group(function () {
    Route::post('/login', [AuthController::class, 'login']);
    Route::post('/logout', [AuthController::class, 'logout'])->middleware('auth:sanctum');

    Route::middleware('auth:sanctum')->group(function () {
        Route::get('/profile', [ProfileController::class, 'show']);
        Route::put('/profile', [ProfileController::class, 'update']);
        Route::get('/notifications', [NotificationController::class, 'index']);
        Route::put('/notifications/{id}/read', [NotificationController::class, 'markAsRead']);
        Route::get('/notifications/unread-count', [NotificationController::class, 'unreadCount']);
        Route::post('/firebase-token', [FirebaseTokenController::class, 'store']);
        Route::delete('/firebase-token', [FirebaseTokenController::class, 'destroy']);
        Route::get('/dashboard', [DashboardController::class, 'index']);
        Route::get('/statistics', [StatisticsController::class, 'index']);
    });
});

// Guard-specific routes
Route::prefix('v1/police-station')->middleware(['auth:sanctum', 'guard:police_station'])->group(function () {
    Route::apiResource('officers', OfficerController::class);
    Route::post('officers/import', [OfficerController::class, 'import']);
    Route::get('officers/locations', [OfficerController::class, 'locations']);
    Route::apiResource('cameras', CameraController::class);
    Route::post('cameras/import', [CameraController::class, 'import']);
    Route::get('crimes', [CrimeController::class, 'index']);
    Route::get('crimes/{crime}', [CrimeController::class, 'show']);
    Route::post('crimes/{crime}/assign', [CrimeController::class, 'assign']);
    // Chat
    Route::get('chat/conversations', [ChatController::class, 'conversations']);
    Route::get('chat/{officer}', [ChatController::class, 'messages']);
    Route::post('chat/{officer}', [ChatController::class, 'send']);
});

Route::prefix('v1/officer')->middleware(['auth:sanctum', 'guard:officer'])->group(function () {
    Route::get('crimes', [OfficerCrimeController::class, 'index']);
    Route::get('crimes/{crime}', [OfficerCrimeController::class, 'show']);
    Route::put('crimes/{crime}/accept', [OfficerCrimeController::class, 'accept']);
    Route::put('crimes/{crime}/arrive', [OfficerCrimeController::class, 'arrive']);
    Route::put('crimes/{crime}/no-visit', [OfficerCrimeController::class, 'noVisit']);
    Route::put('crimes/{crime}/resolve', [OfficerCrimeController::class, 'resolve']);
    Route::post('location', [LocationController::class, 'update']);
    Route::put('status', [StatusController::class, 'update']);
    // Chat
    Route::get('chat', [OfficerChatController::class, 'messages']);
    Route::post('chat', [OfficerChatController::class, 'send']);
});

Route::prefix('v1/admin')->middleware(['auth:sanctum', 'guard:admin'])->group(function () {
    Route::apiResource('police-stations', AdminPoliceStationController::class);
    Route::post('police-stations/import', [AdminPoliceStationController::class, 'import']);
    Route::get('settings', [SettingsController::class, 'index']);
    Route::put('settings', [SettingsController::class, 'update']);
    // Chat
    Route::get('chat/conversations', [AdminChatController::class, 'conversations']);
    Route::get('chat/{policeStation}', [AdminChatController::class, 'messages']);
    Route::post('chat/{policeStation}', [AdminChatController::class, 'send']);
});
```

**Shared Controllers** resolve the guard dynamically:

```php
// app/Http/Controllers/Api/V1/Shared/AuthController.php
class AuthController extends Controller
{
    public function login(LoginRequest $request, string $guard, AuthService $authService): JsonResponse
    {
        $token = $authService->login($guard, $request->validated());

        return $this->success(['token' => $token], __('messages.auth.login_success'));
    }
}
```

### 2.9 Directory Structure Summary

```
app/
├── Http/
│   ├── Controllers/
│   │   ├── Api/
│   │   │   └── V1/
│   │   │       ├── Shared/              ← auth, profile, notifications, firebase-token, dashboard, stats
│   │   │       │   ├── AuthController.php
│   │   │       │   ├── ProfileController.php
│   │   │       │   ├── NotificationController.php
│   │   │       │   ├── FirebaseTokenController.php
│   │   │       │   ├── DashboardController.php
│   │   │       │   └── StatisticsController.php
│   │   │       ├── Admin/               ← admin-only: police stations CRUD, settings, chat
│   │   │       │   ├── PoliceStationController.php
│   │   │       │   ├── SettingsController.php
│   │   │       │   └── ChatController.php
│   │   │       ├── PoliceStation/        ← station-only: officers, cameras, crimes, chat
│   │   │       │   ├── OfficerController.php
│   │   │       │   ├── CameraController.php
│   │   │       │   ├── CrimeController.php
│   │   │       │   └── ChatController.php
│   │   │       ├── Officer/             ← officer-only: crimes, location, status, chat
│   │   │       │   ├── CrimeController.php
│   │   │       │   ├── LocationController.php
│   │   │       │   ├── StatusController.php
│   │   │       │   └── ChatController.php
│   │   │       └── Model/               ← AI model: cameras, heartbeat, alerts, crimes
│   │   │           └── AiModelController.php
│   │   └── Controller.php               ← base controller with ApiResponse trait
│   ├── Requests/                        ← all validation here, never inline
│   │   ├── Auth/
│   │   ├── Officer/
│   │   ├── PoliceStation/
│   │   ├── Camera/
│   │   ├── Crime/
│   │   ├── Chat/
│   │   ├── Import/
│   │   ├── Model/
│   │   └── Settings/
│   ├── Resources/                       ← all API response formatting
│   └── Middleware/
│       ├── ModelApiKeyMiddleware.php
│       └── ResolveGuardMiddleware.php   ← resolves {guard} from URL to auth guard
├── Services/                            ← all business logic
│   ├── Auth/
│   ├── Crime/
│   ├── Officer/
│   ├── PoliceStation/
│   ├── Camera/
│   ├── Scene/
│   ├── Notification/
│   ├── Escalation/
│   ├── Import/
│   ├── Location/
│   ├── Report/
│   ├── Settings/
│   └── Chat/
├── Observers/                           ← automatic model event side effects
│   ├── CrimeObserver.php
│   ├── OfficerObserver.php
│   ├── CameraObserver.php
│   └── PoliceStationObserver.php
├── Events/                              ← broadcasting events
├── Jobs/                                ← background processing
├── Models/                              ← Eloquent models
├── Livewire/                            ← Livewire component PHP classes
│   └── Admin/
├── Traits/                              ← shared traits (ApiResponse, etc.)
├── Enums/                               ← PHP enums
│   ├── CrimeStatus.php
│   ├── CrimeSeverity.php
│   ├── OfficerStatus.php
│   ├── StorageType.php
│   └── DeviceType.php
└── Helpers/                             ← utility classes
```

### 2.10 Enums — Type-Safe Constants

Use PHP 8.4 backed enums instead of string constants for all status/type fields:

```php
namespace App\Enums;

enum CrimeStatus: string
{
    case Pending = 'pending';
    case Assigned = 'assigned';
    case InProgress = 'in_progress';
    case Visited = 'visited';
    case NotVisited = 'not_visited';
    case Resolved = 'resolved';
    case Escalated = 'escalated';
    case FalseAlarm = 'false_alarm';
}
```

Cast them in models via `casts()` method.

---

## Phase 3: Multi-Guard Authentication

13. Configure 3 authentication guards in `config/auth.php`: `admin`, `police_station`, `officer` — each with its own Eloquent provider pointing to the respective model.

14. Configure Sanctum to support all 3 guards for API token authentication. Each guard uses Sanctum's `HasApiTokens` trait on the model.

15. Create **shared** `AuthController` (single controller, dynamic guard via URL segment `/{guard}`):
    - `POST /api/v1/{guard}/login` — resolves guard from URL, authenticates, returns Sanctum token
    - `POST /api/v1/{guard}/logout` — logout, revoke token
    - All shared routes under `/{guard}` prefix: profile, notifications, firebase-token, dashboard

16. Create `ResolveGuardMiddleware` — maps URL segment to auth guard:
    - `admin` → `admin` guard + `Admin` model
    - `police-station` → `police_station` guard + `PoliceStation` model
    - `officer` → `officer` guard + `Officer` model

17. For the web admin dashboard, use session-based auth with the `admin` guard. Create Livewire login component at the admin route.

18. Register middleware in `bootstrap/app.php` for guard-based route protection.

---

## Phase 3: Admin Dashboard (Livewire + Tailwind)

15. Create admin layout at `resources/views/layouts/admin.blade.php` with sidebar navigation, top bar with profile dropdown, notifications bell icon.

16. **Admin Login Page** — Livewire component at `/admin/login` with email/password form.

17. **Admin Dashboard/Metrics Page** — Livewire component at `/admin/dashboard`:
    - Total police stations, officers, cameras, crimes count cards
    - Crimes by status chart (pending, assigned, visited, etc.)
    - Recent crimes list
    - Active cameras count
    - Crime trends over time (line chart)

18. **Police Stations CRUD** — Livewire component at `/admin/police-stations`:
    - List with search, filter, pagination
    - Create/Edit modal form
    - **Excel upload**: bulk import police stations from Excel file (use `maatwebsite/excel` or `openspout/openspout` package)
    - Delete with confirmation
    - Show detail page with related officers and cameras

19. **System Settings** — Livewire component at `/admin/settings`:
    - Firebase notification keys configuration (store in DB settings table or `.env`)
    - System language toggle (ar/en)
    - Pusher configuration
    - General system settings

20. **Admin Profile** — Livewire component at `/admin/profile`:
    - Edit name, email, phone, password
    - Profile photo upload (optional)

21. **Notifications Page** — Livewire component at `/admin/notifications`:
    - List all notifications with read/unread status
    - Mark as read functionality
    - Real-time new notification badge via Pusher

---

## Phase 4: Police Station API (for Flutter Mobile App)

22. Create **guard-specific controllers** (thin — delegate to services) with these API endpoints (all under `api/v1/police-station/` with `auth:sanctum` + `guard:police_station` middleware):
    - `OfficerController` → Officers CRUD: `GET/POST/PUT/DELETE /officers`, `POST /officers/import` (Excel upload)
    - `CameraController` → Cameras CRUD: `GET/POST/PUT/DELETE /cameras`, `POST /cameras/import` (Excel upload)
    - `CrimeController` → Crimes: `GET /crimes` (filters by status, date, officer, camera), `GET /crimes/{id}`, `POST /crimes/{id}/assign`
    - `ChatController` → Chat with officers: `GET /chat/conversations`, `GET /chat/{officer}`, `POST /chat/{officer}`
    - **Officer Tracking**: `GET /officers/locations` (real-time officer positions for map display)
    - **Shared routes** (profile, notifications, dashboard, firebase-token) handled by shared controllers under `/{guard}` prefix

23. Create Eloquent API Resources for: `OfficerResource`, `CameraResource`, `CrimeResource`, `SceneResource`, `NotificationResource`, `PoliceStationResource`, `DashboardResource`, `ChatMessageResource`, `ChatConversationResource`.

24. Create Form Request classes in dedicated subdirectories: `Officer/StoreOfficerRequest`, `Camera/StoreCameraRequest`, `Import/ImportExcelRequest`, `Chat/SendMessageRequest`, etc.

---

## Phase 5: Officer API (for Flutter Mobile App)

25. Create **guard-specific controllers** (thin — delegate to services) under `api/v1/officer/` with `auth:sanctum` + `guard:officer`:
    - `CrimeController` → Crimes: `GET /crimes` (filtered by 5km radius), `GET /crimes/{id}`, actions: `accept`, `arrive`, `no-visit`, `resolve`
    - `LocationController` → `POST /location` (update lat/lng — triggered every 100m from Flutter)
    - `StatusController` → `PUT /status` (update availability status)
    - `ChatController` → Chat with police station: `GET /chat`, `POST /chat`
    - **Shared routes** (profile, notifications, dashboard, firebase-token) handled by shared controllers under `/{guard}` prefix

---

## Phase 6: AI Model Integration API

26. Create `AiModelController` with endpoints for the AI model to interact with Laravel:
    - `GET /api/model/cameras` — list cameras with `is_active = true` and `model_ip_address IS NULL` (unclaimed cameras), **max 10 per request** (enforced by API). Returns:
      ```json
      {
        "cameras": [
          {
            "id": 1,
            "name": "Main Entrance",
            "ip_address": "192.168.1.6",
            "rtsp_url": "rtsp://mohamed:m0ohamed123456789@192.168.1.6:554/stream2",
            "user_name": "mohamed",
            "password": "m0ohamed123456789",
            "lat": 30.0444,
            "lang": 31.2357,
            "storage_type": "none",
            "police_station_id": 1
          }
        ]
      }
      ```
    - `POST /api/model/cameras/{id}/claim` — model sets its IP in `model_ip_address` to claim a camera. **Rejects if model already has 10 cameras claimed.**
    - `DELETE /api/model/cameras/{id}/release` — model releases a camera
    - `POST /api/model/heartbeat` — model sends periodic heartbeat (every 60s) with list of camera IDs it's monitoring. If heartbeat missed for 2+ minutes, Laravel releases claimed cameras (via scheduled task) so other model instances can pick them up.
    - `POST /api/model/alert` — model sends initial alert (confidence rising, suspicious activity). Payload: `{camera_id, confidence_score, alert_type: "suspicious"}`. **Note: Model triggers camera alarm DIRECTLY** (not through Laravel — faster response time). Laravel:
      1. Sends preliminary notification to police station ("⚠️ Suspicious activity detected at Camera X")
      2. Broadcasts Pusher event to police station + admin dashboard
      3. Logs the alert in `activity_logs`
    - `POST /api/model/crime` — model confirms crime with scene timestamps. Payload: `{camera_id, start_time, end_time, crime_type, confidence_score}`. Laravel:
      1. Checks camera `storage_type`:
         - `sd_card` / `cloud` → **Fetches recording from camera** via existing recording API
         - `none` → **Finds matching ffmpeg segments** on server by timestamp range, merges into single MP4:
           ```bash
           # Find segments that overlap with crime window
           # Merge: ffmpeg -i "concat:seg1|seg2" -c:v copy -c:a aac -b:a 64k output.mp4
           ```
      2. Saves evidence to `storage/app/crimes/{crime_id}/evidence.mp4`
      3. Generates thumbnail from first frame for quick preview
      4. Creates `Scene` record with video path + thumbnail
      5. Creates `Crime` record (status: `pending`, severity based on confidence: <40% low, <60% medium, <80% high, >=80% critical)
      6. Finds nearest available officer (Haversine, same police station, within 5km, status: `available`, `is_on_shift: true`)
      7. If officer found: assigns → status: `assigned`, sends FCM push + Pusher event
      8. If no officer found: notifies police station for manual assignment
      9. Sends FCM push notification to police station
      10. Broadcasts Pusher event to police station dashboard + admin dashboard
      11. Logs everything in `activity_logs`
    - `POST /api/model/alert/stop` — model reports scene ended (confidence back to green). **Model stops camera alarm directly.** Laravel logs the event.

27. Secure model endpoints with a shared API key (stored in `.env` as `MODEL_API_KEY`) via `ModelApiKeyMiddleware`, since the model isn't a "user" — just an authenticated service.

28. Create a **scheduled task** (every 2 minutes) to check model heartbeats — release cameras from models that haven't sent heartbeat, mark them as available for other instances.

---

## Phase 7: Real-time & Notifications

29. Create Laravel Events & Broadcasting:
    - `CrimeDetected` event — broadcast to police station + admin private channels
    - `CrimeStatusUpdated` event — broadcast when officer accepts/arrives/declines
    - `OfficerLocationUpdated` event — broadcast officer GPS for map tracking
    - `CrimeEscalated` event — broadcast when crime is escalated due to no response
    - `SuspiciousActivityAlert` event — broadcast preliminary alert from model
    - `NewNotification` event — broadcast to specific user channel

30. Configure broadcasting channels in `routes/channels.php`:
    - `private police-station.{id}` — for police station real-time updates
    - `private officer.{id}` — for officer notifications
    - `private admin.{id}` — for admin notifications
    - `presence police-station.{id}.officers` — presence channel to see which officers are online

31. Create Firebase notification service at `app/Services/FirebaseService.php`:
    - Send push notifications via Firebase Cloud Messaging HTTP v1 API
    - Use service account credentials (stored securely, path in settings)
    - Methods: `sendToDevice($token, $title, $body, $data)`, `sendToMultiple($tokens, ...)`, `sendToTopic($topic, ...)`
    - Notification types: `crime_detected`, `crime_assigned`, `crime_escalated`, `suspicious_activity`, `officer_assigned`

32. Create a `SendCrimeNotification` queued job that handles sending both Pusher events and Firebase push notifications. Use Redis queue for background processing.

33. Create an **Escalation Service** at `app/Services/EscalationService.php`:
    - Scheduled task runs every minute
    - Checks crimes with status `assigned` where `updated_at` > X minutes (configurable, default 5 min)
    - If officer hasn't accepted: reassign to next nearest available officer
    - Increment `escalation_count` on the crime
    - After 3 escalations with no response: notify police station for manual intervention
    - Log all escalations in `activity_logs`

---

## Phase 8: Excel Import Feature

32. Add `openspout/openspout` or `maatwebsite/excel` package for Excel parsing. Create import classes:
    - `PoliceStationImport` — validates and creates police stations from Excel
    - `OfficerImport` — validates and creates officers (requires `police_station_id`)
    - `CameraImport` — validates and creates cameras (requires `police_station_id`)
    - Each import should validate rows, skip duplicates (by email), and return import results (success count, error rows)

---

## Phase 9: Geolocation & Officer Assignment

36. Create a `NearestOfficerService` at `app/Services/NearestOfficerService.php`:
    - Uses Haversine formula or PostgreSQL `earth_distance` extension to find officers within radius
    - Input: camera lat/lng + radius (default 5km, configurable via settings)
    - Filters: officer must belong to same police station as camera, officer `status = available`, `is_on_shift = true`
    - Returns ordered list of nearest officers (not just one — needed for escalation)
    - Fallback: if no officer in same station within radius, expand search or notify station

37. Add spatial query support — since using PostgreSQL, optionally use PostGIS extension or raw Haversine SQL for distance calculations.

38. **Response Time Tracking**: When officer status changes to `visited`, auto-calculate `response_time_minutes` = difference between `crime.created_at` and `crime.officer_arrive_time`. Store for analytics.

---

## Phase 10: Camera Health Monitoring & Recording Service

39. Create a **scheduled task** (every 5 minutes) that checks all active cameras:
    - Ping camera IP via existing `/camera/status` endpoint
    - If camera is unreachable: mark `is_active = false`, notify police station
    - If camera comes back online: mark `is_active = true`, notify police station
    - Log status changes in `activity_logs`

40. Create `CameraHealthCheck` job — queued job that performs the actual check for each camera. Dispatch in batches to avoid overwhelming the network.

41. Create **CameraRecordingService** at `app/Services/CameraRecordingService.php`:
    - Manages continuous ffmpeg recording for cameras with `storage_type = 'none'`
    - Each camera gets its own ffmpeg process recording RTSP stream in 1-minute segments:
      ```bash
      ffmpeg -rtsp_transport tcp -use_wallclock_as_timestamps 1 \
        -i "rtsp://{user}:{pass}@{ip}:554/stream2" \
        -map 0:v -c:v copy -map 0:a -c:a copy \
        -f segment -segment_time 60 -reset_timestamps 1 -strftime 1 \
        "storage/app/recordings/{camera_id}/%Y-%m-%d_%H-%M-%S.mkv"
      ```
    - Tracks ffmpeg process PIDs in Redis for management (start/stop/restart)
    - **Auto-start** on camera activation (`is_active = true` + `storage_type = 'none'`)
    - **Auto-stop** on camera deactivation or when `storage_type` changes
    - Artisan command: `php artisan camera:recording {start|stop|restart|status} {camera_id|all}`

42. Create **SceneExtractor** service at `app/Services/SceneExtractorService.php`:
    - Input: `camera_id`, `start_time`, `end_time`
    - If `storage_type = 'sd_card'` or `'cloud'`: fetch from camera via existing recording API
    - If `storage_type = 'none'`: find overlapping ffmpeg segments in `storage/app/recordings/{camera_id}/`
      1. List all `.mkv` segments whose timestamp falls within `[start_time - 30s, end_time + 30s]`
      2. Create concat file list for ffmpeg
      3. Merge and convert: `ffmpeg -f concat -i list.txt -c:v copy -c:a aac -b:a 64k output.mp4`
      4. Extract thumbnail: `ffmpeg -i output.mp4 -vframes 1 -q:v 2 thumbnail.jpg`
    - Output: `{video_path, thumbnail_path}`
    - Save to `storage/app/crimes/{crime_id}/`

43. Create **recording cleanup scheduled task** (daily at 3 AM):
    - Delete ffmpeg segments older than configurable retention period (default 7 days)
    - Delete orphaned crime evidence older than configurable period (default 90 days)
    - Report storage usage in admin dashboard

---

## Phase 11: Reports & Analytics

41. Create `ReportService` at `app/Services/ReportService.php`:
    - Generate PDF/Excel reports for:
      - **Daily/Weekly/Monthly crime summary** — total crimes, by type, by severity, by status
      - **Officer performance report** — response times, crimes handled, acceptance rate
      - **Camera utilization report** — crimes detected per camera, uptime
      - **Police station comparison report** — for admin dashboard
    - Use `barryvdh/laravel-dompdf` for PDF generation
    - Schedulable: auto-generate daily report and email to admin

42. Admin dashboard widgets for analytics:
    - Average response time (overall, per station, per officer)
    - Crime heatmap (based on camera locations)
    - Officer availability timeline
    - False alarm rate (helps evaluate AI model accuracy)
    - Crime trends over time with severity breakdown

---

## Phase 12: Settings & Configuration

43. Create a `settings` table with key-value pairs for system configuration:
    - `firebase_server_key`, `firebase_project_id`, `firebase_credentials_path`
    - `system_language` (ar/en)
    - `notification_radius_km` (default 5)
    - `crime_auto_assign` (boolean, default true)
    - `escalation_timeout_minutes` (default 5)
    - `max_escalation_attempts` (default 3)
    - `camera_health_check_interval_minutes` (default 5)
    - `model_heartbeat_timeout_minutes` (default 2)
    - `officer_location_update_distance_meters` (default 100)
    - `recording_segment_duration_seconds` (default 60 — ffmpeg segment length for cameras with no storage)
    - `recording_retention_days` (default 7 — how long to keep ffmpeg segments)
    - `crime_evidence_retention_days` (default 90 — how long to keep crime evidence files)
    - `max_cameras_per_model` (default 10 — load balancing limit)

44. Create `Setting` model and `SettingsService` for easy access: `Setting::get('key', 'default')`, `Setting::set('key', 'value')`. Cache settings in Redis for performance.

---

## Phase 13: Testing

45. Write Pest feature tests for:
    - Multi-guard authentication (admin, police_station, officer login/logout/profile)
    - Police station CRUD operations + Excel import
    - Officer CRUD operations + Excel import
    - Camera CRUD operations + Excel import
    - Crime creation and assignment flow (full end-to-end)
    - Nearest officer calculation with edge cases (no officers, all busy, out of range)
    - Escalation timeout behavior
    - API resource response structure
    - Notification sending (mock Firebase)
    - Camera health check
    - Model API key authentication
    - Settings CRUD
    - Report generation
    - Scene extraction (both storage types)
    - Camera recording service management
    - Model load balancing (10 camera limit)

---

## Phase 14: Real-time Chat System

46. Create `chat_messages` migration and model at `app/Models/ChatMessage.php`:
    - Fields: `id`, `conversation_key` (string, indexed — e.g. `admin_1_station_3` or `station_2_officer_5`), `sender_type` (polymorphic), `sender_id`, `receiver_type` (polymorphic), `receiver_id`, `message` (text), `type` (enum: text/image/file), `file_path` (nullable), `is_read` (boolean, default false), `read_at` (nullable), timestamps.
    - Relationships: `morphTo()` for sender and receiver
    - Scopes: `forConversation($key)`, `unread()`

47. Chat communication channels:
    - **Admin ↔ Police Station**: Admin can chat with any police station. One conversation per pair.
    - **Police Station ↔ Officer**: Police station can chat with officers belonging to it. One conversation per pair.
    - **NOT**: Officer ↔ Officer, Admin ↔ Officer (communication goes through the station hierarchy)

48. Chat API endpoints (already defined in shared routes above):
    - Police Station: `GET /chat/conversations`, `GET /chat/{officer}`, `POST /chat/{officer}`
    - Officer: `GET /chat` (messages with their station), `POST /chat`
    - Admin: `GET /chat/conversations`, `GET /chat/{policeStation}`, `POST /chat/{policeStation}`

49. Real-time chat via Pusher:
    - `ChatMessageSent` event → broadcast to private channel `chat.{conversation_key}`
    - Flutter app listens on channel for instant message delivery
    - Admin dashboard Livewire component updates in real-time
    - Unread badge count on chat icon

50. Chat features:
    - Text messages + image/file attachments
    - Read receipts (is_read + read_at)
    - Unread count per conversation
    - Message history with pagination (cursor-based for performance)
    - Firebase push notification for new messages when app is in background (`SendChatNotification` job)
    - Admin Livewire chat components: `Chat/Index.php` (conversation list) + `Chat/Conversation.php` (message view)

---

## Phase 15: Additional Professional Features (Optional Enhancements)

51. **Crime Statistics API** for the Flutter app:
    - `GET /api/v1/{guard}/statistics?period=daily|weekly|monthly`
    - Returns: total crimes, by type, by severity, response time avg, resolved %, false alarm %
    - Used by both police station and officer apps for their metrics pages
    - **Shared controller** — works for all guard types via `/{guard}/statistics`

52. **Multi-language Push Notifications**:
    - Store notification templates in both Arabic and English
    - Send push notification in the user's preferred language (stored in profile)
    - System language setting as default, per-user override

53. **Camera Live View Proxy** (for admin dashboard & police station app):
    - Endpoint: `GET /api/v1/police-station/cameras/{id}/live`
    - Proxy RTSP stream as HLS/DASH for browser/mobile playback
    - Use ffmpeg to transcode RTSP → HLS on-demand
    - Security: only authenticated users of the owning police station can view

54. **Officer Shift Management**:
    - Police station can set shift schedules for officers
    - Auto-toggle `is_on_shift` based on schedule
    - Officers outside shift hours don't receive crime assignments
    - Shift override for emergencies

55. **Audit Trail & Compliance**:
    - Complete activity log for all system actions
    - Exportable audit reports for legal compliance
    - Tamper-proof evidence chain (hash evidence files, store hash in crime record)

---

## Verification

- Run `php artisan test --compact` after each phase to ensure no regressions
- Test auth flows: login with each guard type, verify token-based and session-based auth
- Test crime flow end-to-end: simulate model calling `/api/model/crime` → verify crime created → officer notified → officer accepts → status updated → police station sees real-time update
- Test Excel imports with valid and invalid data files
- Test geolocation: seed officers at known coordinates, verify nearest officer selection
- Verify Pusher events fire correctly using Laravel Telescope
- Run `vendor/bin/pint --dirty --format agent` after every PHP change

---

## Decisions

- **Architecture**: Clean architecture — thin controllers, service layer for logic, FormRequests for validation, Resources for responses, Observers for side effects, Jobs for heavy work
- **Shared Routes**: Common endpoints (auth, profile, notifications, dashboard) use `/{guard}` URL pattern with a single shared controller set. Guard-specific routes are separate.
- **Auth Strategy**: Multi-guard with separate tables (not roles on a single users table) — matches the ERD and allows separate login flows and different fields per user type
- **Real-time**: Pusher for both web dashboard and mobile (including chat)
- **Firebase**: For mobile push notifications only (Pusher handles real-time data sync)
- **Chat**: Hierarchical — Admin ↔ Police Station ↔ Officers. Real-time via Pusher + FCM for background.
- **Excel**: Will recommend `maatwebsite/excel` (most popular, well-documented with Laravel) or `openspout` (lightweight)
- **Geolocation**: PostgreSQL Haversine formula (no PostGIS needed for basic radius queries)
- **API Versioning**: `/api/v1/` prefix for all mobile API routes for future compatibility
- **Crime Assignment**: Automatic — system finds nearest officer and assigns. Officer can decline with reason. Auto-escalation after timeout.
- **Model Auth**: Simple API key middleware (not Sanctum) since the AI model is a service, not a user
- **Scene Recording**: Dual strategy — camera with storage uses its own API, camera without storage uses server-side ffmpeg recording (1-min segments, merged on demand)
- **Camera Alarm**: Triggered **directly by the AI model** (not through Laravel) for fastest response time. Model already has camera credentials from the claim API.
- **RTSP URL**: Full URL returned in camera API response for direct model connection. Format: `rtsp://{user}:{pass}@{effective_ip}:{connection_port}/stream2`. Uses `connection_ip` (public/VPN IP) if set, otherwise falls back to `ip_address` (local LAN).
- **Camera Networking**: Cameras on remote networks (different WiFi) require port forwarding. The camera model stores `connection_ip`, `connection_port`, `api_port`, `onvif_port` — these can differ from local LAN values. For same-network cameras, these fields are either null (use defaults) or match local values. For remote cameras, `connection_ip` = router public IP, ports = forwarded ports. This keeps the system flexible for both dev (same LAN) and production (remote) without code changes.
- **Model Load Balancing**: Each model instance handles max 10 cameras, enforced by API. Load distributed automatically via claim mechanism.
- **Enums**: PHP 8.4 backed enums for all status/type fields — type-safe, auto-cast in models
- **Caching**: Redis for settings, session, queue, and cache. All configurable values cached.
- **Queue Driver**: Redis — for notifications, escalation checks, camera health checks, report generation, scene extraction, chat notifications

---

## Architecture Diagram

```
                           ┌──────────────────────────────────────────────┐
                           │           Laravel Backend (PHP 8.4)          │
                           │                                              │
                           │  ┌─────────────┐  ┌──────────────────────┐  │
                           │  │ Scene        │  │ Recording Service    │  │
                           │  │ Extractor    │  │ (ffmpeg for cameras  │  │
                           │  │ (merge segs) │  │  without storage)    │  │
                           │  └─────────────┘  └──────────────────────┘  │
                           │                                              │
                           └────────┬──────────────┬──────────────────────┘
                                    │              │
┌─────────────┐   webhook           │              │  Sanctum API    ┌─────────────────┐
│  AI Model   │ ──────────────────► │              │ ◄──────────────►│  Flutter Mobile  │
│  (Python)   │   /api/model/*      │              │    + FCM        │  (Officers +     │
│  max 10 cam │                     │              │                 │   Police Station)│
└──────┬──────┘                     │              │                 └─────────────────┘
       │                            │              │
       │ RTSP stream         Pusher │              │
       │ + direct alarm             │              │
       │                            ▼              │
┌──────┴──────┐              ┌─────────────────┐   │
│ Tapo C200   │              │ Admin Dashboard │   │
│ Cameras     │◄─────────────│ (Livewire)      │   │
│ (alarm API) │  camera API  └─────────────────┘   │
└─────────────┘                                    │
       ▲                                           │
       │  ffmpeg RTSP recording (if no storage)    │
       └───────────────────────────────────────────┘
```

---

## File Structure (New Files)

```
app/
├── Models/              Admin, PoliceStation, Officer, Camera, Crime, Scene,
│                        CrimeType, FirebaseToken, Setting, ActivityLog, ChatMessage
│
├── Enums/               CrimeStatus, CrimeSeverity, OfficerStatus, StorageType, DeviceType, MessageType
│
├── Http/
│   ├── Controllers/
│   │   └── Api/V1/
│   │       ├── Shared/          AuthController, ProfileController, NotificationController,
│   │       │                    FirebaseTokenController, DashboardController, StatisticsController
│   │       ├── Admin/           PoliceStationController, SettingsController, ChatController
│   │       ├── PoliceStation/   OfficerController, CameraController, CrimeController, ChatController
│   │       ├── Officer/         CrimeController, LocationController, StatusController, ChatController
│   │       └── Model/           AiModelController
│   ├── Requests/                Organized by domain: Auth/, Officer/, Camera/, Crime/, Chat/,
│   │                            Import/, Model/, Settings/, PoliceStation/
│   ├── Resources/               AdminResource, OfficerResource, CameraResource, CrimeResource,
│   │                            SceneResource, NotificationResource, ChatMessageResource,
│   │                            ChatConversationResource, DashboardResource, ModelCameraResource
│   └── Middleware/              ModelApiKeyMiddleware, ResolveGuardMiddleware
│
├── Services/                    Organized by domain:
│   ├── Auth/                    AuthService
│   ├── Crime/                   CrimeService
│   ├── Officer/                 OfficerService
│   ├── PoliceStation/           PoliceStationService
│   ├── Camera/                  CameraService, CameraRecordingService, CameraHealthService
│   ├── Scene/                   SceneExtractorService
│   ├── Notification/            FirebaseService, NotificationService
│   ├── Escalation/              EscalationService
│   ├── Import/                  ExcelImportService
│   ├── Location/                NearestOfficerService
│   ├── Report/                  ReportService
│   ├── Settings/                SettingsService
│   └── Chat/                    ChatService
│
├── Observers/                   CrimeObserver, OfficerObserver, CameraObserver, PoliceStationObserver
│
├── Events/                      CrimeDetected, CrimeStatusUpdated, OfficerLocationUpdated,
│                                CrimeEscalated, SuspiciousActivityAlert, ChatMessageSent
│
├── Jobs/                        SendCrimeNotification, ProcessExcelImport, CheckCameraHealth,
│                                ExtractCrimeScene, GenerateReport, CleanupOldRecordings,
│                                SendChatNotification
│
├── Livewire/Admin/              Auth/Login, Dashboard, PoliceStations/{Index,Create,Edit},
│                                Settings/Index, Profile, Notifications,
│                                Chat/{Index,Conversation}
│
├── Console/Commands/            CheckModelHeartbeats, EscalateStaleCrimes, CheckCameraHealth,
│                                ManageCameraRecording, CleanupRecordings
│
├── Traits/                      ApiResponse (existing)
├── Helpers/                     TapoHelper, OnvifHelper (existing)
└── Exceptions/                  ApiExceptionHandler (existing)

resources/views/
├── layouts/                     admin.blade.php
├── livewire/admin/              dashboard, police-stations/{index,create,edit},
│                                settings/index, profile, notifications, auth/login,
│                                chat/{index,conversation}
└── components/                  Shared Blade components (tables, modals, charts, etc.)

database/
├── migrations/                  12+ new migrations for domain tables
├── factories/                   Factory for each model
└── seeders/                     Seeder for each model + DatabaseSeeder
```

---

## API Versioning & Route Organization

All API routes are organized into separate files under `routes/api/` for maintainability and versioning:

```
routes/
├── api.php                         ← Central importer (no inline routes)
├── web.php                         ← Livewire admin dashboard
└── api/
    ├── camera.php                  ← Camera hardware control (unversioned)
    └── v1/
        ├── shared.php              ← Auth, profile, notifications — /{guard}/...
        ├── admin.php               ← Admin-specific routes
        ├── police-station.php      ← Police station-specific routes
        ├── officer.php             ← Officer-specific routes
        └── model.php               ← AI model integration routes
```

### Versioning Strategy

- **Versioned routes** (`/api/v1/...`): All client-facing endpoints (mobile apps, admin dashboard). When breaking changes are needed, create a `v2/` folder and add new route files without touching `v1/`.
- **Unversioned routes** (`/api/camera/...`, `/api/model/...`): Internal service routes for camera hardware control and AI model communication. These are consumed only by internal systems and don't need versioning.

### How `api.php` Works

```php
// Camera control API (unversioned — internal hardware control)
require __DIR__.'/api/camera.php';

// Versioned API routes (v1)
Route::prefix('v1')->group(function () {
    require __DIR__.'/api/v1/shared.php';
    require __DIR__.'/api/v1/admin.php';
    Route::prefix('police-station')->group(fn () => require __DIR__.'/api/v1/police-station.php');
    Route::prefix('officer')->group(fn () => require __DIR__.'/api/v1/officer.php');
});

// AI Model routes (unversioned — internal service)
Route::prefix('model')->group(fn () => require __DIR__.'/api/v1/model.php');
```

### Adding a New Version

1. Create `routes/api/v2/` with only the changed route files.
2. Add `Route::prefix('v2')->group(...)` in `api.php`.
3. Old mobile apps continue hitting `/api/v1/`, new versions use `/api/v2/`.

---

## Tech Stack Summary

| Layer | Technology | Purpose |
|---|---|---|
| **Backend** | PHP 8.4 + Laravel 12 | Core API & business logic |
| **Database** | PostgreSQL | Primary database with spatial queries |
| **Cache/Queue** | Redis (predis) | Background jobs, caching, sessions |
| **Admin Dashboard** | Livewire 4 + Tailwind CSS v4 + Blaze | Reactive admin panel |
| **Real-time** | Pusher | WebSocket events for dashboards & mobile |
| **Push Notifications** | Firebase Cloud Messaging | Mobile push notifications |
| **API Auth** | Laravel Sanctum v4 | Token-based auth for mobile apps |
| **Camera Control** | pytapo + ONVIF (existing) | Tapo C200 camera integration |
| **AI Model** | Python (separate service) | Crime detection via RTSP streams |
| **Mobile App** | Flutter | Officers & Police Stations apps |
| **Testing** | Pest v4 | Feature & unit tests |
| **Code Style** | Laravel Pint | Automated formatting |
| **Monitoring** | Laravel Telescope | Dev debugging & request inspection |

---

## API Endpoints Summary (for Mobile Team Reference)

### Shared Routes (all guards via `/{guard}` prefix)
| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/v1/{guard}/login` | Login, returns Sanctum token |
| POST | `/api/v1/{guard}/logout` | Logout, revoke token |
| GET | `/api/v1/{guard}/profile` | Get profile |
| PUT | `/api/v1/{guard}/profile` | Update profile |
| GET | `/api/v1/{guard}/notifications` | List notifications |
| PUT | `/api/v1/{guard}/notifications/{id}/read` | Mark notification as read |
| GET | `/api/v1/{guard}/notifications/unread-count` | Unread notification count |
| POST | `/api/v1/{guard}/firebase-token` | Register FCM device token |
| DELETE | `/api/v1/{guard}/firebase-token` | Remove FCM token |
| GET | `/api/v1/{guard}/dashboard` | Dashboard stats (guard-aware) |
| GET | `/api/v1/{guard}/statistics` | Statistics by period |

### Police Station App (guard-specific)
| Method | Endpoint | Description |
|---|---|---|
| GET/POST | `/api/v1/police-station/officers` | List / Create officer |
| PUT/DELETE | `/api/v1/police-station/officers/{id}` | Update / Delete officer |
| POST | `/api/v1/police-station/officers/import` | Excel import officers |
| GET | `/api/v1/police-station/officers/locations` | All officers GPS for map |
| GET/POST | `/api/v1/police-station/cameras` | List / Create camera |
| PUT/DELETE | `/api/v1/police-station/cameras/{id}` | Update / Delete camera |
| POST | `/api/v1/police-station/cameras/import` | Excel import cameras |
| GET | `/api/v1/police-station/crimes` | List crimes (with filters) |
| GET | `/api/v1/police-station/crimes/{id}` | Crime detail + scene |
| POST | `/api/v1/police-station/crimes/{id}/assign` | Manually assign officer |
| GET | `/api/v1/police-station/chat/conversations` | List chat conversations |
| GET | `/api/v1/police-station/chat/{officer}` | Messages with officer |
| POST | `/api/v1/police-station/chat/{officer}` | Send message to officer |

### Officer App (guard-specific)
| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/v1/officer/crimes` | Crimes within 5km |
| GET | `/api/v1/officer/crimes/{id}` | Crime detail + scene |
| PUT | `/api/v1/officer/crimes/{id}/accept` | Accept assignment |
| PUT | `/api/v1/officer/crimes/{id}/arrive` | Mark arrived |
| PUT | `/api/v1/officer/crimes/{id}/no-visit` | Decline with reason |
| PUT | `/api/v1/officer/crimes/{id}/resolve` | Mark crime resolved |
| POST | `/api/v1/officer/location` | Update GPS (every 100m) |
| PUT | `/api/v1/officer/status` | Update availability status |
| GET | `/api/v1/officer/chat` | Messages with police station |
| POST | `/api/v1/officer/chat` | Send message to station |

### Admin App / Dashboard (guard-specific)
| Method | Endpoint | Description |
|---|---|---|
| GET/POST | `/api/v1/admin/police-stations` | List / Create station |
| PUT/DELETE | `/api/v1/admin/police-stations/{id}` | Update / Delete station |
| POST | `/api/v1/admin/police-stations/import` | Excel import stations |
| GET/PUT | `/api/v1/admin/settings` | System settings |
| GET | `/api/v1/admin/chat/conversations` | List chat conversations |
| GET | `/api/v1/admin/chat/{policeStation}` | Messages with station |
| POST | `/api/v1/admin/chat/{policeStation}` | Send message to station |

### AI Model
| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/model/cameras` | List unclaimed active cameras (max 10) |
| POST | `/api/model/cameras/{id}/claim` | Claim camera for processing |
| DELETE | `/api/model/cameras/{id}/release` | Release camera |
| POST | `/api/model/heartbeat` | Model heartbeat |
| POST | `/api/model/alert` | Suspicious activity alert |
| POST | `/api/model/alert/stop` | Alert ended |
| POST | `/api/model/crime` | Confirmed crime report |
