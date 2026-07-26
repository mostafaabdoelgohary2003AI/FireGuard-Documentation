# FireGuard — Architecture

## 1. High-Level Architecture

FireGuard follows a **three-tier architecture** composed of an edge AI service, a centralized backend API, and a mobile client. The tiers communicate asynchronously through HTTP and Firebase Cloud Messaging.

```mermaid
graph TB
    subgraph EDGE["Edge Tier (AI Service)"]
        direction LR
        CAM[🎥 Camera Hardware]
        CV[OpenCV<br/>Frame Capture]
        YOLO[YOLOv8<br/>Inference Engine]
        SNAP[Snapshot<br/>Generator]
    end

    subgraph BACKEND_TIER["Backend Tier (Django)"]
        direction TB
        DRF[Django REST Framework<br/>API Gateway]
        AUTH_APP[accounts app<br/>Authentication]
        CAM_APP[cameras app<br/>Camera Management]
        EVT_APP[events app<br/>Event Lifecycle]
        NOTIF_APP[notifications app<br/>FCM Integration]
        SQLITE[(SQLite3<br/>Database)]
    end

    subgraph CLIENT_TIER["Client Tier (Flutter)"]
        direction TB
        API_SVC[ApiService<br/>Dio HTTP Client]
        PROVIDERS[Provider<br/>State Management]
        SCREENS[Feature Screens<br/>UI Layer]
        FCM_CLIENT[FirebaseNotifications<br/>Push Handler]
        LOCAL_DB[(Local SQLite<br/>Notification Cache)]
    end

    subgraph EXTERNAL["External Services"]
        FCM[Firebase Cloud<br/>Messaging]
    end

    CAM --> CV --> YOLO --> SNAP
    SNAP -->|HTTP POST /api/events/| DRF
    DRF --> AUTH_APP
    DRF --> CAM_APP
    DRF --> EVT_APP
    DRF --> NOTIF_APP
    EVT_APP --> SQLITE
    CAM_APP --> SQLITE
    AUTH_APP --> SQLITE
    NOTIF_APP --> SQLITE
    NOTIF_APP -->|Firebase Admin SDK| FCM
    FCM -->|Push Notification| FCM_CLIENT
    FCM_CLIENT --> PROVIDERS
    FCM_CLIENT --> LOCAL_DB
    PROVIDERS --> SCREENS
    API_SVC -->|HTTP GET/POST/PATCH/DELETE| DRF
    SCREENS --> API_SVC
```

---

## 2. Component Responsibilities

### 2.1 AI Service — Edge Tier

| Component | File | Responsibility |
|-----------|------|---------------|
| Camera Capture | `camera.py` | Opens webcam via `cv2.VideoCapture(0)`, reads frames in a loop |
| YOLOv8 Inference | `camera.py` | Loads `best.pt`, runs `model(frame, conf=0.5, stream=True)` |
| Cooldown Controller | `camera.py` | Enforces 10-second minimum interval between alerts |
| Snapshot Generator | `camera.py` | Saves annotated frames as `alert_YYYYMMDD_HHMMSS.jpg` |
| Event Reporter | `camera.py` | Sends `POST /api/events/` with multipart form data (snapshot + metadata) |
| Display Window | `camera.py` | Shows live detection feed via `cv2.imshow()` |

### 2.2 Django Backend — Backend Tier

| App | Responsibility | Key Models |
|-----|---------------|------------|
| `fireguard` | Project configuration, URL routing, Firebase initialization | — |
| `accounts` | User registration, authentication (Token), profile management | `UserProfile` |
| `cameras` | Camera and zone CRUD, status management, statistics | `Zone`, `Camera` |
| `events` | Fire event lifecycle (create, list, resolve, false alarm, stats) | `FireEvent` |
| `notifications` | FCM device management, notification preferences, push dispatch | `FCMDevice`, `NotificationPreference` |

### 2.3 Flutter Mobile App — Client Tier

| Layer | Component | Responsibility |
|-------|-----------|---------------|
| **Services** | `ApiService` | Dio-based HTTP client with token interceptor |
| **Services** | `CacheHelper` | SharedPreferences wrapper for token storage |
| **Services** | `FirebaseNotifications` | FCM initialization, foreground/background handling, local notification display |
| **Services** | `DatabaseService` | Local SQLite for offline notification history |
| **State** | `AuthProvider` | Login/logout state, token management |
| **State** | `CameraProvider` | Camera list, CRUD operations state |
| **State** | `ActivityProvider` | Event history list, resolve actions |
| **State** | `HomeProvider` | Dashboard statistics |
| **Navigation** | `GoRouter` | Declarative route definitions, deep linking from notifications |
| **UI** | Feature screens | Splash, Login, Signup, Home, Cameras, Alert, Activity, Settings, Profile |

---

## 3. Integration Workflow

### 3.1 Event Creation Flow

```mermaid
sequenceDiagram
    participant AI as camera.py
    participant API as Django API
    participant DB as SQLite3
    participant FCM_SRV as fcm.py
    participant FCM as Firebase Cloud
    participant APP as Flutter App

    AI->>API: POST /api/events/<br/>{camera, event_type, ai_confidence, snapshot}
    API->>API: Validate via FireEventCreateSerializer
    API->>DB: FireEvent.objects.create()
    API->>FCM_SRV: send_fire_alert_to_all(event)
    FCM_SRV->>FCM_SRV: Query NotificationPreference (fire_alerts=True)
    FCM_SRV->>FCM_SRV: Fetch active FCMDevice tokens
    FCM_SRV->>FCM: MulticastMessage with notification + data payload
    FCM->>APP: Push notification delivered
    API-->>AI: 201 Created {event, fcm_result}
```

### 3.2 Event Resolution Flow

```mermaid
sequenceDiagram
    participant APP as Flutter App
    participant API as Django API
    participant DB as SQLite3
    participant FCM_SRV as fcm.py
    participant FCM as Firebase Cloud

    APP->>API: PATCH /api/events/{id}/resolve/
    API->>DB: event.status = 'resolved'<br/>event.resolved_at = now()
    API->>FCM_SRV: send_resolved_alert(event)
    FCM_SRV->>FCM_SRV: Query opted-in users (resolved_alerts=True)
    FCM_SRV->>FCM: MulticastMessage (event_resolved)
    FCM->>APP: Push notification (resolved)
    API-->>APP: 200 OK {event data}
```

### 3.3 User Authentication Flow

```mermaid
sequenceDiagram
    participant APP as Flutter App
    participant CACHE as SharedPreferences
    participant API as Django API
    participant DB as SQLite3

    APP->>API: POST /api/auth/register/ {username, email, password, password2}
    API->>DB: User.objects.create_user()
    API->>DB: Token.objects.get_or_create(user)
    API-->>APP: 201 {token, user}
    APP->>CACHE: Save token

    Note over APP: Subsequent requests
    APP->>API: GET /api/events/<br/>Header: Authorization: Token {token}
    API->>API: TokenAuthentication validates token
    API->>DB: Query events
    API-->>APP: 200 {events}
```

---

## 4. Runtime Workflow

```mermaid
stateDiagram-v2
    [*] --> CameraCapture: Start camera.py
    CameraCapture --> FrameRead: Read frame
    FrameRead --> YOLOInference: Pass to YOLO model
    YOLOInference --> CheckDetection: Check results

    CheckDetection --> FrameRead: No detection
    CheckDetection --> CheckCooldown: Detection found

    CheckCooldown --> FrameRead: Cooldown active (< 10s)
    CheckCooldown --> SaveSnapshot: Cooldown expired

    SaveSnapshot --> PostToAPI: HTTP POST event
    PostToAPI --> FrameRead: Continue loop

    PostToAPI --> [*]: User presses 'q'
```

---

## 5. Detection Workflow

```mermaid
flowchart TD
    A[Camera frame captured] --> B[YOLOv8 inference<br/>conf=0.5, stream=True]
    B --> C{Boxes detected?}
    C -->|No| A
    C -->|Yes| D[Extract class, confidence]
    D --> E{event_type = fire or smoke?}
    E -->|Yes| F{Current time - last_sent > 10s?}
    F -->|No| A
    F -->|Yes| G[Save annotated frame as JPEG]
    G --> H[POST to /api/events/]
    H --> I[Backend creates FireEvent]
    I --> J[FCM notification sent]
    J --> A
```

---

## 6. Notification Workflow

```mermaid
flowchart TD
    A[FireEvent created] --> B[send_fire_alert_to_all]
    B --> C[Get event config<br/>fire → 🔥 / smoke → 💨]
    C --> D[Query NotificationPreference<br/>fire_alerts=True]
    D --> E[Fetch FCMDevice tokens<br/>is_active=True, user in opted-in]
    E --> F{Tokens found?}
    F -->|No| G[Return skipped]
    F -->|Yes| H[Build MulticastMessage]
    H --> I[Set Android config<br/>priority=high, channel=alerts]
    H --> J[Set APNS config<br/>priority=10, sound=default]
    I --> K[send_each_for_multicast]
    J --> K
    K --> L{Check responses}
    L --> M[Deactivate failed tokens]
    L --> N[Return success/failure counts]
```

### Notification Payload Structure

The FCM message includes both a `notification` object (for system tray display) and a `data` object (for client-side routing):

```json
{
  "notification": {
    "title": "🔥 Fire Detected!",
    "body": "Building 3, Floor 2 — Storage Area — 87.5% confidence"
  },
  "data": {
    "type": "fire_alert",
    "event_id": "42",
    "camera_id": "1",
    "camera_name": "Warehouse Cam A",
    "location": "Building 3, Floor 2",
    "zone": "Storage Area",
    "event_type": "fire",
    "ai_confidence": "87.5",
    "detected_at": "2026-05-28T14:30:00+00:00"
  }
}
```

---

## 7. Deployment Architecture

The current deployment uses the following infrastructure:

```mermaid
graph LR
    subgraph LOCAL["Developer/Demo Machine"]
        AI_SVC["camera.py<br/>(Python 3.10+)"]
        WEBCAM["USB/WiFi Camera"]
    end

    subgraph PA["PythonAnywhere (Free Tier)"]
        DJANGO_SRV["Django 5.2<br/>WSGI"]
        SQLITE_FILE["db.sqlite3"]
        MEDIA["media/<br/>snapshots/"]
    end

    subgraph GCP["Google Cloud (Managed)"]
        FCM_SVC["Firebase Cloud<br/>Messaging"]
    end

    subgraph USER_DEVICE["Android / iOS"]
        FLUTTER_APP["Flutter App<br/>(APK / IPA)"]
    end

    WEBCAM --> AI_SVC
    AI_SVC -->|HTTPS| DJANGO_SRV
    DJANGO_SRV --> SQLITE_FILE
    DJANGO_SRV --> MEDIA
    DJANGO_SRV -->|Firebase Admin SDK| FCM_SVC
    FCM_SVC --> FLUTTER_APP
    FLUTTER_APP -->|HTTPS| DJANGO_SRV
```

### Deployment Notes

- The Django backend runs as a WSGI application on PythonAnywhere. The `Procfile` contains `web: gunicorn fireguard.wsgi`.
- Static files are served from the `staticfiles/` directory via PythonAnywhere's static file configuration.
- Media files (snapshots, avatars, camera thumbnails) are stored in the `media/` directory on the PythonAnywhere filesystem.
- The AI service (`camera.py`) runs on a local machine and requires a physical camera connection. It communicates with the deployed backend over HTTPS.
- Firebase credentials are stored as a JSON file on the server (not in version control).

---

## 8. Activity Diagram — Complete System Flow

```mermaid
flowchart TB
    START([System Start]) --> INIT_CAM[Initialize Camera<br/>cv2.VideoCapture 0]
    INIT_CAM --> LOAD_MODEL[Load YOLOv8 model<br/>best.pt]
    LOAD_MODEL --> READ_FRAME[Read frame from camera]
    READ_FRAME --> INFER[Run YOLO inference<br/>conf=0.5]
    INFER --> DET{Fire/Smoke<br/>detected?}
    DET -->|No| DISPLAY[Display frame<br/>cv2.imshow]
    DET -->|Yes| COOL{Cooldown<br/>expired?}
    COOL -->|No| DISPLAY
    COOL -->|Yes| SAVE[Save annotated<br/>snapshot JPEG]
    SAVE --> POST[POST /api/events/]
    POST --> VALIDATE{Serializer<br/>valid?}
    VALIDATE -->|No| ERR[Return 400 error]
    VALIDATE -->|Yes| STORE[Create FireEvent<br/>in SQLite3]
    STORE --> NOTIFY[send_fire_alert_to_all]
    NOTIFY --> FCM_SEND[FCM multicast<br/>to all devices]
    FCM_SEND --> MOBILE_RECV[Flutter receives<br/>push notification]
    MOBILE_RECV --> SHOW_ALERT[Display alert screen<br/>+ play alarm sound]
    SHOW_ALERT --> USER_ACTION{User action}
    USER_ACTION -->|Resolve| RESOLVE[PATCH /resolve/<br/>status=resolved]
    USER_ACTION -->|False Alarm| FALSE[PATCH /false-alarm/<br/>status=false_alarm]
    RESOLVE --> NOTIFY_RESOLVED[send_resolved_alert]
    ERR --> DISPLAY
    DISPLAY --> QUIT{User pressed<br/>q?}
    QUIT -->|No| READ_FRAME
    QUIT -->|Yes| CLEANUP[Release camera<br/>Destroy windows]
    CLEANUP --> END([End])
```

---

## 9. DFD — Level 0 (Context Diagram)

```mermaid
flowchart LR
    CAMERA([External Camera]) -->|Video feed| FG["FireGuard<br/>System"]
    FG -->|Push notifications| USER([Mobile User])
    USER -->|Event resolution commands| FG
    ADMIN([Admin User]) -->|Camera/Zone management| FG
```

## 10. DFD — Level 1

```mermaid
flowchart TB
    CAM([Camera Source]) -->|Video frames| P1["P1: AI Detection<br/>(camera.py + YOLOv8)"]
    P1 -->|Event data + snapshot| P2["P2: Event Management<br/>(events app)"]
    P2 -->|INSERT/UPDATE| D1[(D1: SQLite3<br/>FireEvent table)]
    P2 -->|Trigger notification| P3["P3: Notification<br/>Dispatch (fcm.py)"]
    P3 -->|Query preferences| D2[(D2: SQLite3<br/>NotificationPreference)]
    P3 -->|Query tokens| D3[(D3: SQLite3<br/>FCMDevice)]
    P3 -->|FCM message| EXT_FCM([Firebase Cloud])
    EXT_FCM -->|Push| P4["P4: Mobile Alert<br/>(Flutter)"]
    P4 -->|Display to| USER([User])
    USER -->|Resolve / False Alarm| P2
    USER -->|View events| P2
    ADMIN([Admin]) -->|Manage cameras| P5["P5: Camera Management<br/>(cameras app)"]
    P5 -->|CRUD| D4[(D4: SQLite3<br/>Camera, Zone tables)]
    USER -->|Register / Login| P6["P6: Authentication<br/>(accounts app)"]
    P6 -->|CRUD| D5[(D5: SQLite3<br/>User, UserProfile, Token)]
```
