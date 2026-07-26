# FireGuard — Database

## 1. Overview

FireGuard uses **SQLite3** as its server-side database, managed entirely through the Django ORM. The database file (`db.sqlite3`) resides in the `backend/` directory and is created/updated via Django migrations.

Additionally, the Flutter mobile application uses a **separate local SQLite database** (`fire_guard.db`) on the device for caching notification history offline.

---

## 2. SQLite3 Configuration

### 2.1 Server-Side (Django Backend)

```python
# fireguard/settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}
```

| Parameter | Value | Description |
|-----------|-------|-------------|
| Engine | `django.db.backends.sqlite3` | Django's built-in SQLite backend |
| Database file | `backend/db.sqlite3` | File-based storage, no separate server process |
| Migration tool | `python manage.py migrate` | Django ORM generates and applies schema changes |
| Default PK | `BigAutoField` | 64-bit auto-incrementing integer primary keys |

### 2.2 Client-Side (Flutter App)

```dart
// core/services/database_service.dart
Future<Database> _initDB(String filePath) async {
    final dbPath = await getDatabasesPath();
    final path = join(dbPath, filePath);
    return await openDatabase(path, version: 2, onCreate: _createDB);
}
```

| Parameter | Value | Description |
|-----------|-------|-------------|
| Database name | `fire_guard.db` | Stored in the app's sandboxed directory |
| Package | `sqflite` | Flutter SQLite plugin |
| Schema version | `2` | Current schema version |

---

## 3. Entity Relationship Diagram

```mermaid
erDiagram
    User ||--o| UserProfile : "has one"
    User ||--o| NotificationPreference : "has one"
    User ||--o{ FCMDevice : "owns many"
    User ||--o{ Token : "has one"
    Zone ||--o{ Camera : "contains many"
    Camera ||--o{ FireEvent : "generates many"

    User {
        BigAutoField id PK
        CharField username UK
        CharField email
        CharField first_name
        CharField last_name
        CharField password
        BooleanField is_staff
        BooleanField is_active
        DateTimeField date_joined
    }

    UserProfile {
        BigAutoField id PK
        OneToOneField user_id FK "User"
        CharField phone "max_length=20, nullable"
        ImageField avatar "upload_to=avatars/, nullable"
        DateTimeField created_at "auto_now_add"
        DateTimeField updated_at "auto_now"
    }

    Zone {
        BigAutoField id PK
        CharField name "max_length=100"
        TextField description "nullable"
        DateTimeField created_at "auto_now_add"
    }

    Camera {
        BigAutoField id PK
        ForeignKey zone_id FK "Zone, CASCADE"
        CharField name "max_length=100"
        CharField location "max_length=200"
        URLField stream_url "nullable"
        ImageField thumbnail "upload_to=cameras/, nullable"
        CharField status "online | offline, default=online"
        DateTimeField created_at "auto_now_add"
        DateTimeField updated_at "auto_now"
    }

    FireEvent {
        BigAutoField id PK
        ForeignKey camera_id FK "Camera, CASCADE"
        CharField event_type "fire | smoke"
        CharField status "active | resolved | false_alarm"
        FloatField ai_confidence "0-100"
        ImageField snapshot "upload_to=snapshots/, nullable"
        TextField notes "nullable"
        DateTimeField detected_at "auto_now_add"
        DateTimeField resolved_at "nullable"
    }

    FCMDevice {
        BigAutoField id PK
        ForeignKey user_id FK "User, CASCADE"
        TextField token "unique"
        CharField platform "android | ios"
        BooleanField is_active "default=True"
        DateTimeField created_at "auto_now_add"
        DateTimeField updated_at "auto_now"
    }

    NotificationPreference {
        BigAutoField id PK
        OneToOneField user_id FK "User, CASCADE"
        BooleanField fire_alerts "default=True"
        BooleanField resolved_alerts "default=True"
        BooleanField false_alarm_alerts "default=False"
        BooleanField sound_enabled "default=True"
        DateTimeField created_at "auto_now_add"
        DateTimeField updated_at "auto_now"
    }

    Token {
        BigAutoField id PK
        ForeignKey user_id FK "User"
        CharField key "unique"
        DateTimeField created "auto_now_add"
    }
```

---

## 4. Database Entities

### 4.1 User (Django built-in `auth_user`)

The standard Django `User` model from `django.contrib.auth.models`.

| Field | Type | Constraints | Description |
|-------|------|------------|-------------|
| `id` | BigAutoField | PK | Auto-incrementing primary key |
| `username` | CharField(150) | Unique, required | Login username |
| `email` | CharField(254) | Optional | User email address |
| `first_name` | CharField(150) | Optional | First name |
| `last_name` | CharField(150) | Optional | Last name |
| `password` | CharField(128) | Required | Hashed password |
| `is_staff` | BooleanField | Default: False | Admin access flag |
| `is_active` | BooleanField | Default: True | Account active flag |
| `date_joined` | DateTimeField | Auto | Registration timestamp |

### 4.2 UserProfile (`accounts_userprofile`)

| Field | Type | Constraints | Description |
|-------|------|------------|-------------|
| `id` | BigAutoField | PK | Auto-incrementing primary key |
| `user_id` | OneToOneField | FK → User, CASCADE | One-to-one link to User |
| `phone` | CharField(20) | Nullable | Phone number |
| `avatar` | ImageField | Nullable, upload_to=`avatars/` | Profile picture |
| `created_at` | DateTimeField | auto_now_add | Creation timestamp |
| `updated_at` | DateTimeField | auto_now | Last update timestamp |

**Auto-creation:** Created automatically via `post_save` signal when a new `User` is created.

### 4.3 Zone (`cameras_zone`)

| Field | Type | Constraints | Description |
|-------|------|------------|-------------|
| `id` | BigAutoField | PK | Auto-incrementing primary key |
| `name` | CharField(100) | Required | Zone name (e.g., "Storage Area") |
| `description` | TextField | Nullable | Zone description |
| `created_at` | DateTimeField | auto_now_add | Creation timestamp |

### 4.4 Camera (`cameras_camera`)

| Field | Type | Constraints | Description |
|-------|------|------------|-------------|
| `id` | BigAutoField | PK | Auto-incrementing primary key |
| `zone_id` | ForeignKey | FK → Zone, CASCADE | Parent zone |
| `name` | CharField(100) | Required | Camera display name |
| `location` | CharField(200) | Required | Physical location description |
| `stream_url` | URLField | Nullable | Camera stream URL |
| `thumbnail` | ImageField | Nullable, upload_to=`cameras/` | Camera thumbnail image |
| `status` | CharField(10) | Choices: `online`, `offline` | Current camera status |
| `created_at` | DateTimeField | auto_now_add | Registration timestamp |
| `updated_at` | DateTimeField | auto_now | Last update timestamp |

### 4.5 FireEvent (`events_fireevent`)

| Field | Type | Constraints | Description |
|-------|------|------------|-------------|
| `id` | BigAutoField | PK | Auto-incrementing primary key |
| `camera_id` | ForeignKey | FK → Camera, CASCADE | Source camera |
| `event_type` | CharField(10) | Choices: `fire`, `smoke` | Detection class |
| `status` | CharField(15) | Choices: `active`, `resolved`, `false_alarm` | Event lifecycle status |
| `ai_confidence` | FloatField | 0.0 – 100.0 | YOLOv8 confidence percentage |
| `snapshot` | ImageField | Nullable, upload_to=`snapshots/` | Annotated detection image |
| `notes` | TextField | Nullable | Event notes |
| `detected_at` | DateTimeField | auto_now_add | Detection timestamp |
| `resolved_at` | DateTimeField | Nullable | Resolution timestamp |

**Default ordering:** `-detected_at` (most recent first)

**Status lifecycle:**

```mermaid
stateDiagram-v2
    [*] --> active: Event created by AI
    active --> resolved: PATCH /resolve/
    active --> false_alarm: PATCH /false-alarm/
    resolved --> [*]
    false_alarm --> [*]
```

### 4.6 FCMDevice (`notifications_fcmdevice`)

| Field | Type | Constraints | Description |
|-------|------|------------|-------------|
| `id` | BigAutoField | PK | Auto-incrementing primary key |
| `user_id` | ForeignKey | FK → User, CASCADE | Device owner |
| `token` | TextField | Unique | FCM device token |
| `platform` | CharField(10) | Choices: `android`, `ios` | Device platform |
| `is_active` | BooleanField | Default: True | Whether token is valid |
| `created_at` | DateTimeField | auto_now_add | Registration timestamp |
| `updated_at` | DateTimeField | auto_now | Last update timestamp |

**Token lifecycle:** Tokens are set to `is_active=False` when FCM delivery fails. The Flutter app re-registers the token on each login.

### 4.7 NotificationPreference (`notifications_notificationpreference`)

| Field | Type | Constraints | Description |
|-------|------|------------|-------------|
| `id` | BigAutoField | PK | Auto-incrementing primary key |
| `user_id` | OneToOneField | FK → User, CASCADE | Preference owner |
| `fire_alerts` | BooleanField | Default: True | Receive fire/smoke alerts |
| `resolved_alerts` | BooleanField | Default: True | Receive resolution notifications |
| `false_alarm_alerts` | BooleanField | Default: False | Receive false alarm notifications |
| `sound_enabled` | BooleanField | Default: True | Enable notification sounds |
| `created_at` | DateTimeField | auto_now_add | Creation timestamp |
| `updated_at` | DateTimeField | auto_now | Last update timestamp |

**Auto-creation:** Created automatically via `post_save` signal when a new `User` is created.

### 4.8 Token (`authtoken_token` — DRF built-in)

| Field | Type | Constraints | Description |
|-------|------|------------|-------------|
| `key` | CharField(40) | PK, unique | Authentication token string |
| `user_id` | OneToOneField | FK → User | Token owner |
| `created` | DateTimeField | auto_now_add | Token creation timestamp |

---

## 5. Relationships

| Relationship | Type | On Delete | Description |
|-------------|------|-----------|-------------|
| User → UserProfile | One-to-One | CASCADE | Each user has exactly one profile |
| User → NotificationPreference | One-to-One | CASCADE | Each user has exactly one preference set |
| User → FCMDevice | One-to-Many | CASCADE | A user can have multiple registered devices |
| User → Token | One-to-One | CASCADE | Each user has one active auth token |
| Zone → Camera | One-to-Many | CASCADE | A zone contains multiple cameras |
| Camera → FireEvent | One-to-Many | CASCADE | A camera generates multiple events |

### Cascade Deletion Behavior

- Deleting a **User** cascades to: UserProfile, NotificationPreference, all FCMDevices, Token
- Deleting a **Zone** cascades to: all Cameras in that zone → all FireEvents from those cameras
- Deleting a **Camera** cascades to: all FireEvents from that camera

---

## 6. Django ORM Integration

### 6.1 Query Patterns Used

| Pattern | Location | Example |
|---------|----------|---------|
| `select_related` | events/views.py | `FireEvent.objects.select_related('camera', 'camera__zone')` |
| `annotate` | cameras/views.py | `Zone.objects.annotate(camera_count=Count('cameras'))` |
| `filter` with query params | events/views.py | `.filter(status=event_status)`, `.filter(camera_id=camera_id)` |
| `update_or_create` | notifications/views.py | `FCMDevice.objects.update_or_create(token=token, defaults={...})` |
| `get_or_create` | accounts/views.py | `Token.objects.get_or_create(user=user)` |
| `values_list` | notifications/fcm.py | `.values_list('user_id', flat=True)` |
| Post-save signals | accounts/models.py | Auto-create UserProfile and NotificationPreference |

### 6.2 Migration Management

```bash
# Generate migration files from model changes
python manage.py makemigrations

# Apply migrations to the database
python manage.py migrate

# View migration status
python manage.py showmigrations
```

---

## 7. Event Storage Workflow

```mermaid
sequenceDiagram
    participant AI as camera.py
    participant DRF as Django REST Framework
    participant SERIAL as FireEventCreateSerializer
    participant ORM as Django ORM
    participant DB as SQLite3

    AI->>DRF: POST /api/events/<br/>{camera, event_type, ai_confidence, snapshot}
    DRF->>SERIAL: Validate request data
    SERIAL->>SERIAL: validate_ai_confidence (0-100)
    SERIAL->>SERIAL: validate_camera (not offline)
    SERIAL->>ORM: serializer.save()
    ORM->>DB: INSERT INTO events_fireevent<br/>(camera_id, event_type, status='active',<br/>ai_confidence, snapshot, notes, detected_at)
    DB-->>ORM: Return new FireEvent instance
    ORM-->>DRF: event object
    DRF->>DRF: Trigger FCM notification
    DRF-->>AI: 201 Created
```

---

## 8. Client-Side Database (Flutter)

The Flutter app uses `sqflite` to maintain a local notification cache:

```sql
CREATE TABLE notifications (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    type TEXT,
    event_id TEXT UNIQUE,
    camera_id TEXT,
    camera_name TEXT,
    location TEXT,
    zone TEXT,
    event_type TEXT,
    ai_confidence REAL,
    detected_at TEXT
);
```

| Operation | Method | Description |
|-----------|--------|-------------|
| Insert | `insertNotification(AlertPayload)` | Store incoming FCM notification data |
| Read all | `getAllNotifications()` | Retrieve all cached notifications (ordered by `detected_at DESC`) |
| Clear | `clearAll()` | Delete all cached notifications |

The `event_id` field has a `UNIQUE` constraint with `ConflictAlgorithm.replace` to prevent duplicate entries.

---

## 9. Limitations

- **SQLite3 concurrency** — SQLite3 supports limited concurrent writes. This is acceptable for the graduation project but would need to be replaced with PostgreSQL for production multi-user workloads.
- **No database backup strategy** — The SQLite3 file on PythonAnywhere is not automatically backed up.
- **File-based storage** — SQLite3 is a single file (`db.sqlite3`), making it simple to deploy but not suitable for distributed or high-availability setups.
- **No database indexes** — Beyond Django's default primary key and foreign key indexes, no custom indexes are defined. Query performance is adequate for the current data volume.
