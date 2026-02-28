# Execution Timeline & Sprint Plan

> **مدة المشروع المقدرة**: 8-10 أسابيع (بافتراض فريق من 3: Backend + Flutter + AI Model)
>
> **Estimated Duration**: 8–10 weeks (assuming a team of 3: Backend + Flutter + AI Model)

---

## Overview

The project is divided into **4 Sprints** (2 weeks each) + 1 buffer/polish week. Each sprint has deliverables for all 3 team members. Tasks are ordered by **dependency** — later tasks depend on earlier ones.

```
Sprint 1 (Weeks 1-2)  → Foundation: DB, Auth, Core APIs
Sprint 2 (Weeks 3-4)  → Features: Dashboard, Mobile APIs, Model API
Sprint 3 (Weeks 5-6)  → Integration: Real-time, Notifications, Recording, Chat
Sprint 4 (Weeks 7-8)  → Polish: Reports, Testing, Optimization
Buffer   (Week 9-10)  → Bug fixes, Final testing, Deployment prep
```

---

## Sprint 1: Foundation (Weeks 1–2)

### Week 1

| Day | Backend (أنت) | Flutter Developer | AI Model Developer |
|-----|--------------|-------------------|-------------------|
| 1-2 | Phase 1: Database migrations + Models + Enums + Factories + Seeders | Setup Flutter project structure, shared packages (dio, provider/riverpod, pusher, firebase) | Review MODEL_GUIDE.md, understand API contract |
| 3 | Phase 2: Architecture setup — base controller, ApiResponse, base service classes, middleware skeleton | Auth screens UI (login for officer + police station) | Setup Python project, prepare RTSP stream reader |
| 4-5 | Phase 3: Multi-guard auth (config/auth.php, 4 guards, Sanctum, AuthController, ResolveGuardMiddleware, VerifyModelIpMiddleware) | Connect login to API (once auth API is ready), token storage | Implement login flow + AES-256-CBC decryption for camera data |

### Week 2

| Day | Backend (أنت) | Flutter Developer | AI Model Developer |
|-----|--------------|-------------------|-------------------|
| 1-2 | Phase 5: Officer API — CrimeController, LocationController, StatusController + Services | Officer app: Crime list screen, crime detail screen, map view | Test login + camera fetch + decryption against backend |
| 3-4 | Phase 4: Police Station API — OfficerController, CameraController, CrimeController + Services + password reset | Police Station app: Officers list (with new fields), cameras list, crimes list | Implement alert sending (POST /api/model/alert) |
| 5 | Shared routes: Profile, Notifications, Firebase Token, Dashboard endpoints | Profile screen, notifications list (shared for both apps) | Implement crime reporting (POST /api/model/crime) |

**Sprint 1 Deliverables:**
- ✅ Database fully migrated with seed data
- ✅ Auth working for all 4 guards (admin, police_station, officer, ai_model)
- ✅ Core CRUD APIs for officer + police station apps
- ✅ Flutter apps can login and fetch data
- ✅ Model can login, decrypt camera data, and send alerts/crimes

---

## Sprint 2: Features (Weeks 3–4)

### Week 3

| Day | Backend (أنت) | Flutter Developer | AI Model Developer |
|-----|--------------|-------------------|-------------------|
| 1-2 | Phase 6: AI Model Integration API — AiModelController, VerifyModelIpMiddleware, EncryptionService, CrimeService (full crime creation flow) | Officer app: Accept/arrive/decline/resolve crime actions | Integrate with real RTSP streams from Tapo cameras |
| 3-4 | Phase 3 (Dashboard): Admin login + Dashboard page (Livewire) — metrics cards, charts | Police Station app: Create/edit officer (with national_id, rank, badge_number), create/edit camera forms | Implement direct camera alarm triggering |
| 5 | Admin Dashboard: Police Stations CRUD + AI Models CRUD (with camera assignment checkboxes) | Officer app: GPS location tracking (background service) | Implement confidence scoring + crime type detection |

### Week 4

| Day | Backend (أنت) | Flutter Developer | AI Model Developer |
|-----|--------------|-------------------|-------------------|
| 1-2 | Phase 8: Excel Import (officers, cameras, police stations) | Police Station app: Excel upload UI + import functionality | Test full flow: detect → alert → crime → alarm |
| 3-4 | Phase 9: Geolocation — NearestOfficerService, auto-assignment logic, Haversine queries | Officer app: Map showing crime locations with distance | Fine-tune model accuracy, optimize processing speed |
| 5 | Phase 12: Settings — Settings model, service, admin settings page | Police Station app: Officer locations map (real-time) | Load testing: 10 cameras simultaneously |

**Sprint 2 Deliverables:**
- ✅ Full crime detection → assignment flow working end-to-end
- ✅ Admin dashboard functional with CRUD (police stations + AI models) + charts
- ✅ Excel import working for all entities
- ✅ Officer auto-assignment by proximity
- ✅ Flutter apps have all CRUD screens
- ✅ Model processing real camera streams

---

## Sprint 3: Integration (Weeks 5–6)

### Week 5

| Day | Backend (أنت) | Flutter Developer | AI Model Developer |
|-----|--------------|-------------------|-------------------|
| 1-2 | Phase 7: Real-time — Pusher events (CrimeDetected, CrimeStatusUpdated, OfficerLocationUpdated), channel auth | Integrate Pusher in Flutter — listen to crime events, status updates | Implement heartbeat mechanism (every 60s) |
| 3-4 | Phase 7: Firebase — FirebaseService, SendCrimeNotification job, FCM push notifications | Integrate Firebase push notifications (foreground + background handlers) | Handle graceful shutdown + reconnection on crash |
| 5 | Phase 7: Escalation — EscalationService, scheduled task, auto-reassignment | Real-time UI updates: crime status changes, new notifications | Test multi-instance deployment (multiple model servers) |

### Week 6

| Day | Backend (أنت) | Flutter Developer | AI Model Developer |
|-----|--------------|-------------------|-------------------|
| 1-2 | Phase 14: Chat System — ChatMessage model, ChatService, chat endpoints, Pusher broadcasting | Chat UI for both apps — conversation list, message thread, send message | Implement alert/stop endpoint (scene ended) |
| 3-4 | Phase 10: Camera Health Monitoring — health check job, CameraRecordingService (ffmpeg), SceneExtractorService | Chat: real-time message receiving via Pusher, unread badges | Optimize: reduce false positives, improve detection speed |
| 5 | Phase 10: Recording cleanup, camera:recording artisan command | Polish: loading states, error handling, offline mode | Documentation: write model API usage examples |

**Sprint 3 Deliverables:**
- ✅ Real-time updates working (Pusher + Firebase)
- ✅ Push notifications delivered to mobile
- ✅ Chat fully functional (both apps + admin dashboard)
- ✅ Camera health monitoring active
- ✅ Scene recording + extraction working
- ✅ Escalation auto-runs on schedule

---

## Sprint 4: Polish (Weeks 7–8)

### Week 7

| Day | Backend (أنت) | Flutter Developer | AI Model Developer |
|-----|--------------|-------------------|-------------------|
| 1-2 | Phase 11: Reports — ReportService, PDF/Excel generation, analytics endpoints | UI polish: animations, transitions, dark mode | Edge case handling: network drops, camera offline |
| 3-4 | Phase 13: Testing — Pest tests for auth, CRUD, crime flow, escalation, geolocation | Testing: widget tests, integration tests | Testing: accuracy metrics, performance benchmarks |
| 5 | Admin Dashboard polish: charts, crime heatmap, officer availability timeline | Admin chat Livewire integration testing | Final model optimization |

### Week 8

| Day | Backend (أنت) | Flutter Developer | AI Model Developer |
|-----|--------------|-------------------|-------------------|
| 1-2 | More tests: model API, settings, recording, chat, Excel import | End-to-end testing with real backend | End-to-end testing with real cameras |
| 3-4 | Bug fixes from testing, performance optimization (query optimization, caching) | Bug fixes, UI refinements | Bug fixes, detection refinements |
| 5 | Phase 15 (optional): pick highest-value features | App store preparation (icons, splash, screenshots) | Deploy model service, document deployment |

**Sprint 4 Deliverables:**
- ✅ Reports and analytics working
- ✅ 80%+ test coverage on critical paths
- ✅ All known bugs fixed
- ✅ Performance optimized
- ✅ System ready for demo

---

## Buffer / Final Week (Week 9–10)

| Task | Owner |
|------|-------|
| Full end-to-end system demo rehearsal | All |
| Fix any remaining issues discovered during demo | All |
| Deployment documentation | Backend |
| APK/IPA build for demo devices | Flutter |
| Model deployment + monitoring setup | AI Model |
| Final presentation preparation | All |
| Graduation project documentation/report | All |

---

## Dependency Graph

```
Phase 1 (DB + Models) ─────────────────────────────────┐
    │                                                    │
    ├─► Phase 2 (Architecture) ──┐                      │
    │                            │                      │
    ├─► Phase 3 (Auth) ─────────┤                      │
    │                            │                      │
    │                            ▼                      │
    │              Phase 4 (Police Station API) ──┐     │
    │              Phase 5 (Officer API) ─────────┤     │
    │              Phase 6 (Model API) ───────────┤     │
    │                                              │     │
    │                                              ▼     │
    │                           Phase 7 (Real-time + Notifications)
    │                           Phase 8 (Excel Import)
    │                           Phase 9 (Geolocation)
    │                                              │
    │                                              ▼
    │                           Phase 10 (Camera Health + Recording)
    │                           Phase 14 (Chat)
    │                                              │
    │                                              ▼
    │                           Phase 11 (Reports)
    │                           Phase 12 (Settings)
    │                           Phase 13 (Testing)
    │                                              │
    │                                              ▼
    └──────────────────────── Phase 15 (Optional Enhancements)
```

---

## Critical Path (Must-Have for Demo)

These are the absolute minimum features needed for a working graduation demo:

| Priority | Feature | Phases | Est. Days |
|----------|---------|--------|-----------|
| 🔴 P0 | Database + Models | Phase 1 | 2 |
| 🔴 P0 | Multi-guard Auth (login/logout) | Phase 3 | 2 |
| 🔴 P0 | Officer API (crimes, location, status) | Phase 5 | 3 |
| 🔴 P0 | Police Station API (officers, cameras, crimes) | Phase 4 | 3 |
| 🔴 P0 | AI Model API (login, cameras, alert, crime) | Phase 6 | 3 |
| 🔴 P0 | Crime detection → assignment flow | Phase 6+9 | 2 |
| 🟡 P1 | Push notifications (Firebase) | Phase 7 | 2 |
| 🟡 P1 | Real-time updates (Pusher) | Phase 7 | 2 |
| 🟡 P1 | Admin Dashboard (basic) | Phase 3 | 4 |
| 🟡 P1 | Scene recording + extraction | Phase 10 | 3 |
| 🟢 P2 | Chat system | Phase 14 | 3 |
| 🟢 P2 | Excel import | Phase 8 | 2 |
| 🟢 P2 | Reports | Phase 11 | 2 |
| 🟢 P2 | Camera health monitoring | Phase 10 | 2 |
| ⚪ P3 | Escalation system | Phase 7 | 2 |
| ⚪ P3 | Tests | Phase 13 | 4 |
| ⚪ P3 | Optional features | Phase 15 | varies |

**Minimum viable demo = P0 features (~15 days of backend work)**

---

## Team Sync Points

| When | What | Who |
|------|------|-----|
| End of Week 1 | Auth API ready → Flutter can start integrating | Backend → Flutter |
| End of Week 1 | Model API contract finalized (login + encryption) → Model can start implementing | Backend → Model |
| End of Week 2 | CRUD APIs ready → Flutter builds full screens | Backend → Flutter |
| Mid Week 3 | Crime flow API ready → full end-to-end test | All |
| End of Week 4 | Excel import ready → Flutter adds upload UI | Backend → Flutter |
| Mid Week 5 | Pusher events ready → Flutter integrates real-time | Backend → Flutter |
| Mid Week 5 | Firebase ready → Flutter handles push notifications | Backend → Flutter |
| End of Week 6 | Chat API ready → Flutter builds chat UI | Backend → Flutter |
| Week 7-8 | Integration testing sessions | All together |

---

## Risk Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| AI model accuracy too low | Crime flow useless without good detection | Use mock/simulated crimes for demo, show model separately |
| Camera unavailability | Can't demo live detection | Pre-record crimes, use mock model that sends fake alerts |
| Team member delays | Sprint plan slips | P0 features first, everything else is bonus |
| Real-time (Pusher) issues | No live updates in demo | Fallback to polling (refresh button) |
| Firebase setup complexity | No push notifications | Demo with in-app notifications only |
| PostgreSQL spatial queries complex | Officer assignment fails | Use simple Haversine PHP calculation instead of PostGIS |
