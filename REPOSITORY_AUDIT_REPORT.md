# Greeni — Repository Audit Report

| | |
|---|---|
| **Repository** | `saba787/Greeni` |
| **Commit audited** | `3adfdc9` — "Initial Greeni backend with Docker, FastAPI, Celery, Redis" (the only commit; `main` and the working branch are identical) |
| **Audit date** | 2026-10-07 |
| **Scope** | Every tracked file (170 files, ~12.4k lines; 150 Python files / 11,562 lines under `app/`) |
| **Method** | Full static read of all files, plus dynamic verification in an isolated scratch environment (see [Appendix B](#appendix-b--verification-log)) |
| **Code changes made** | None. No commits, no pushes. This report is the only file added. |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Methodology and Evidence Levels](#2-methodology-and-evidence-levels)
3. [Repository Map](#3-repository-map)
4. [Product Understanding](#4-product-understanding)
5. [Architecture Analysis](#5-architecture-analysis)
6. [Feature Inventory](#6-feature-inventory)
7. [Code Quality Review](#7-code-quality-review)
8. [Frontend Review](#8-frontend-review)
9. [Backend Review](#9-backend-review)
10. [Database Review](#10-database-review)
11. [API Audit](#11-api-audit)
12. [Security Audit](#12-security-audit)
13. [DevOps Audit](#13-devops-audit)
14. [Testing Audit](#14-testing-audit)
15. [Documentation Audit](#15-documentation-audit)
16. [Dependency Audit](#16-dependency-audit)
17. [Business Logic Review](#17-business-logic-review)
18. [Technical Debt](#18-technical-debt)
19. [Missing Features](#19-missing-features)
20. [Over-Engineering Review](#20-over-engineering-review)
21. [Recommendations](#21-recommendations)
22. [Final Scorecard](#22-final-scorecard)
23. [Appendix A — Endpoint Inventory](#appendix-a--endpoint-inventory)
24. [Appendix B — Verification Log](#appendix-b--verification-log)
25. [Appendix C — Dead, Empty, and Duplicate Files](#appendix-c--dead-empty-and-duplicate-files)

---

## 1. Executive Summary

**What it is.** Greeni is the **backend API for a smart-greenhouse / IoT management platform** (`app/main.py:26-29`: *"Greeni Management API — Backend API for Smart Greenhouse Management"*). It is a Python/FastAPI service with PostgreSQL, Redis, Celery and an MQTT broker (Mosquitto). It manages edge devices (the code comments name Raspberry Pi), sensor catalogs, actuator commands over MQTT, IP/USB cameras with PTZ, audio/media distribution to devices, and multi-channel notifications (SMS via Iranian providers, email, push, in-app). The frontend is **not in this repository**; `docker-compose.yml` builds it from a sibling directory `../nginx`.

**Current state.** This is an **early-stage prototype**: a single "initial" commit, no tests, no README, no committed database migrations, and a lot of debug output. The service starts under `uvicorn`, and the basic CRUD paths (users, devices, sensors, cameras, in-app notifications) work. But several advertised subsystems are **non-functional**: camera streaming, PTZ, device telemetry ingestion, command-status feedback, email and push notifications, hardware protocol drivers, and the device-registration notifications.

**Bottom line for decision-makers.**

- **It is not production-ready and must not be exposed to any network you don't fully trust.** In a live run I confirmed that an unauthenticated caller can dispatch actuator commands, read SMS/email provider credentials, read and delete any user's notifications, and queue SMS messages to arbitrary numbers (which costs money). Any logged-in user can **take over the superuser account** by changing its password. The JWT signing key falls back to the literal string `"secret"`, so if `SECRET_KEY` is unset, anyone can forge admin tokens (also verified).
- **It can't be deployed from the repository alone.** `alembic/versions/` is git-ignored (`.gitignore:17`), so `alembic upgrade head` on a fresh database creates **no application tables** (verified). Compose also builds the frontend from `../nginx`, which is outside the repository.
- **The domain model and module layout are a reasonable foundation.** A team could keep the data model (devices → sensors → typed attributes; cameras; commands; providers) and the router/CRUD/schema split. But the auth layer, the migrations, the IoT messaging contract, and the camera subsystem need to be rebuilt rather than patched.

### Top 10 findings

| # | Finding | Severity | Evidence |
|---|---|---|---|
| 1 | 81 of 160 HTTP/WS routes have **no authentication**, including actuator command dispatch, provider credential CRUD, sensor CRUD, notification send and in-app notifications | Critical | Runtime introspection plus live calls ([§12 SEC-01](#sec-01)) |
| 2 | **No authorization model is enforced.** Any authenticated user can change any user's password (verified superuser takeover), delete devices, and bypass device approval | Critical | [SEC-04](#sec-04) |
| 3 | JWT secret defaults to `"secret"`; forged admin token accepted (verified) | Critical | `app/core/config.py:12` |
| 4 | Seeded superuser with password `1234` and hardcoded real-looking personal data, re-created on every startup if missing | Critical | `app/db/init_data.py:39-52` |
| 5 | MQTT broker allows anonymous clients and is published on all interfaces, so anyone on the network can command or spoof devices | Critical | `mosquitto/config/mosquitto.conf:3`, `docker-compose.yml:125-127` |
| 6 | **No migrations in version control.** A fresh deploy creates zero tables | Critical (operational) | `.gitignore:17`; verified |
| 7 | Secrets returned in API responses: SMS `api_key`, SMTP `password`, push `api_key`, camera `password`, device `local_key` (the last one unauthenticated) | High | Verified live |
| 8 | Camera streaming and PTZ are completely broken (missing method, wrong response type, placeholder WSDL path) | High | `app/camera/utils/camera_utils.py:43`, `app/camera/manager/ptz_controller.py:49` |
| 9 | Notification pipeline is broken in several places: missing templates, duplicate sends, stub email/push, reversed SMS argument order, seeded permission name never matches, so **nobody is notified** | High | [§17](#17-business-logic-review); verified "no recipients" |
| 10 | 41 known advisories in 13 pinned dependencies (Starlette, python-multipart, cryptography, urllib3, …) | High | `pip-audit` ([§16](#16-dependency-audit)) |

**Overall score: 2.1 / 10** (see [§22](#22-final-scorecard)). **Confidence in the codebase: very low.** There are zero automated tests, and roughly a third of the functional surface is broken or stubbed.

---

## 2. Methodology and Evidence Levels

1. **Static review.** I read every tracked file in full: source, configuration, Docker, Alembic, requirements, Mosquitto config, and the empty files.
2. **Dynamic verification** in an isolated scratch copy of the repo, so no runtime artifacts touched the working tree:
   - Python 3.11 virtualenv with `requirements.txt` (the Dockerfile uses 3.10); `pip check` and `pip-audit`.
   - Throwaway PostgreSQL 16 and plain Redis 7. Compose uses Postgres 15 and Redis Stack.
   - `alembic upgrade head` on an empty database.
   - Schema created with `Base.metadata.create_all` to approximate the developer's untracked migrations.
   - The API run with `uvicorn app.main:app`, the same command compose uses.
   - Scripted HTTP probes for authentication, authorization, and functional bugs.
   - Route-level dependency introspection to classify every endpoint's auth.
3. **Limits.** There was no MQTT broker, no camera hardware, and no Redis Stack (RedisJSON), and Docker was unavailable in the audit sandbox. MQTT paths were therefore exercised only in the disconnected state, and the cache-backed `/roles` and `/tags` endpoints were not exercised end to end. Where a conclusion rests on reading code alone, it is marked as such.

**Evidence tags used below:**

- **[Verified]**: reproduced at runtime (details in Appendix B).
- **[Static]**: certain from reading the code (for example a missing method or a wrong signature).
- **[Inferred]**: a reasoned conclusion with stated uncertainty.

---

## 3. Repository Map

```
Greeni/
├── Dockerfile                  # python:3.10-slim; ~36 of 60 lines are commented-out earlier versions
├── docker-compose.yml          # db, redis(stack), redisinsight, backend, celery_worker, frontend(../nginx), mosquitto
├── alembic.ini / alembic/      # env.py + template only — NO versions/ (git-ignored)
├── mosquitto/config/mosquitto.conf   # anonymous MQTT on 1883 + websockets on 9001
├── requirements.txt            # used by Dockerfile (66 pins)
├── requirements.lock.txt       # stale subset (missing gmqtt, loguru, psutil, opencv, zeep…) — misleading
├── requirementsback.txt        # older backup with conflicting pins
├── wait-for-it.sh              # standard vendored script
├── To                          # empty stray file
├── .gitignore                  # ignores .env, logs/, media/, alembic/versions/
└── app/                        # 150 .py files, 11,562 lines
    ├── main.py                 # FastAPI app, CORS, /media static mount, startup (Redis listener, MQTT, seed)
    ├── celery.py / worker.py   # TWO separate Celery app definitions
    ├── api/v1/                 # 26 files, 3,949 lines — REST routers + ws/ (WebSockets)
    ├── crud/                   # 15 files, 1,856 lines — data access (incl. "device copy.py")
    ├── bll/                    # 4 files — "business logic" (device, notifications, role/tag caching)
    ├── models/                 # 16 files — SQLAlchemy models
    ├── schemas/                # 16 files — Pydantic models
    ├── camera/                 # 14 files, 1,262 lines — camera models/CRUD/schemas/managers (OpenCV, ffmpeg, ONVIF)
    ├── iot/                    # 26 files, 1,657 lines — MQTT client/handler/publisher, protocol drivers, command executors
    ├── notifications/ sms/ email/   # notification manager + channel services
    ├── core/                   # config, security (JWT/bcrypt), logging, healthcheck, permission constants
    ├── db/                     # Base, session, seed data
    ├── cache/                  # Redis (RedisJSON) cache wrapper
    ├── tasks/                  # Celery tasks (notifications, example)
    ├── dependencies/           # get_db, get_current_user
    └── utils/
```

**Not present:** tests, README or any docs, CI/CD config, `.env.example`, `.dockerignore`, linters or formatters config, `pyproject.toml`, frontend code, nginx config, infrastructure-as-code.

**Size by area:** `api` 34%, `crud` 16%, `iot` 14%, `camera` 11%, `schemas` 8%, `models` 6%, other 11%. Around 1,300 lines of `iot/protocols` are hardware-driver simulations that nothing calls ([§20](#20-over-engineering-review)).

---

## 4. Product Understanding

### 4.1 What product this is

| Conclusion | How I reached it |
|---|---|
| Backend for a **smart-greenhouse management system** | `app/main.py:27-29` title and description; product name "Greeni"; the path comment `backend_greeniHouse/...` at `mosquitto/config/mosquitto.conf:1` |
| An **IoT control plane** for edge devices (likely Raspberry Pi) | `app/crud/device.py:96` comment *"device registration by system (e.g. Raspberry Pi)"*; GPIO/I2C/SPI/UART/Modbus/LoRa/Zigbee driver scaffolding in `app/iot/protocols/`; `SystemCommandExecutor` issues `sudo reboot` (`app/iot/commands/executors/system.py:12-13`), which is device-side code |
| Devices also act as **audio/media players** | `app/api/v1/ws/media_ws.py` sends `{"type":"AUDIO","action":"play"}`; `/media/upload` broadcasts download commands to "multimedia" devices (`app/api/v1/media.py:373-397`). **[Inferred]** The purpose is unclear (possibly announcements or deterrents); nothing in the repo states it |
| **Camera surveillance** of greenhouse sites | `app/camera/**`: IP/USB/RTSP cameras, MJPEG/HLS streaming, ONVIF PTZ, recording/retention fields |
| Target market is **Iran / Persian-speaking operators** | Persian comments and user-facing error messages across 59 files; Iranian SMS providers (sms.ir, Melipayamak; "Kavenegar" mentioned in comments); login by phone number in `09xxxxxxxxx` format (`app/api/v1/routes/auth.py:16`) |
| Frontend is a separate SPA served by nginx | `docker-compose.yml:104-107` builds `../nginx`. **[Inferred, low confidence]** Vue/Vuetify, based on `Tag.icon` examples like `mdi-lightbulb` and colors like `'success'` (`app/models/tag.py:23-25`), which are Material Design Icons / Vuetify conventions |
| API version label "04.05.05" | `app/main.py:28`, which suggests prior iterations elsewhere. The git history holds only a single squashed commit |

### 4.2 Target users

- **Superusers / administrators**: approve or reject self-registered devices and cameras, manage users, roles, permissions and notification providers (the `is_superuser` checks in `device.py` and `camera.py`).
- **Operators / regular users**: view devices, sensors and cameras, upload media, control PTZ, send commands.
- **Edge devices (machine clients)**: self-register (`POST /devices/register`), poll approval status (`GET /devices/status/{uuid}`), toggle online/offline with `X-Device-Key`, and talk MQTT.

### 4.3 Business goals (inferred)

Remote monitoring and control of greenhouse equipment: sensors, actuators such as pumps (`Command` example `"turn_on_pump"`, `app/models/command.py:16`), cameras, and audio. Operators get alerts through SMS, email, push and in-app channels.

### 4.4 Core workflows

1. **Login**: phone and password → JWT access token (24 h) and refresh token (7 d).
2. **Device onboarding**: device calls `/devices/register` → `is_pending=True` → admins notified → admin `PATCH /devices/{id}/confirm` or rejects → device polls `/devices/status/{uuid}`.
3. **Sensor catalog**: define sensor types (protocol, default pins, config schema, tags) → attributes (name/type/unit) → attach sensors to devices → record attribute values.
4. **Command dispatch**: `POST /commands/` → saved as `pending` → published to MQTT `greeni/device/{id}/commands` → (intended) status reply.
5. **Media distribution**: upload files → MQTT "download" to multimedia devices → WebSocket control: list, play, delete, download.
6. **Camera lifecycle**: create (pending) → admin approval → frame, MJPEG or HLS streaming → PTZ.
7. **Notifications**: Celery task renders a template → SMS, email, push or in-app → in-app delivered in real time via Redis pub/sub and WebSocket.
8. **Monitoring**: `/health` and WebSockets for health and host system info.

---

## 5. Architecture Analysis

### 5.1 Technology stack

| Layer | Technology | Notes |
|---|---|---|
| Language/runtime | Python 3.10 (Docker) | Python 3.10 reaches end-of-life this month (Oct 2026) |
| Web framework | FastAPI 0.116 / Starlette 0.47 / Uvicorn 0.35 | Deprecated `@app.on_event` lifecycle hooks |
| Validation | Pydantic 2.11 | Mixed v1 idioms (`.dict()`, `from_orm`, `class Config`) |
| ORM / DB | SQLAlchemy 2.0 (legacy `Query` API, sync engine) / PostgreSQL 15 | No async engine, although many endpoints are `async def` |
| Migrations | Alembic 1.16 | Scaffolding only; revisions git-ignored |
| Cache / pub-sub | Redis Stack (RedisJSON required by `BaseCache`) | Hardcoded host `greeni_redis` |
| Background jobs | Celery 5.5 (Redis broker/backend) | Two app instances |
| IoT messaging | MQTT (Mosquitto 2.0; `gmqtt` client) | Anonymous broker |
| Video | OpenCV 4.10, ffmpeg (not installed in image), ONVIF (`onvif-zeep`) | All in-process |
| Auth | `python-jose` JWT HS256, `passlib[bcrypt]` | |
| Logging | loguru plus stdlib intercept; 259 `print()` calls | |
| Ops tooling | Docker Compose, RedisInsight 1.14 | No CI/CD |

### 5.2 Component diagram

```mermaid
flowchart LR
  subgraph Clients
    SPA[Frontend SPA<br/>(../nginx, not in repo)]
    DEV[Edge devices<br/>(Raspberry Pi?)]
  end
  subgraph Compose network
    API[FastAPI backend<br/>uvicorn --reload]
    CW[Celery worker]
    PG[(PostgreSQL)]
    RS[(Redis Stack<br/>cache + broker + pub/sub)]
    RI[RedisInsight :8001]
    MQ[[Mosquitto :1883/:9001<br/>allow_anonymous]]
  end
  EXT[(SMS.ir / Melipayamak APIs)]
  CAM[IP/USB cameras<br/>RTSP / ONVIF]

  SPA -- REST/WS (JWT) --> API
  DEV -- REST register/status/online --> API
  DEV <-- MQTT --> MQ
  API <-- MQTT (gmqtt) --> MQ
  API --> PG
  API -- cache / publish tasks --> RS
  CW -- consume tasks --> RS
  CW --> PG
  CW -- publish 'notifications' --> RS
  RS -- pub/sub 'notifications' --> API
  CW --> EXT
  API -- OpenCV/ffmpeg/ONVIF --> CAM
  RI --> RS
```

### 5.3 Layering

The intended layering is `api → bll → crud → models`. In practice:

- Routers call CRUD directly, BLL, or run raw `db.query(...)` inline (for example `app/api/v1/sensor.py:90-92, 324-336, 420-470`; `app/api/v1/device.py:29-41`).
- **The dependency direction is violated.** The data layer imports the API layer: `app/crud/device.py:12` imports `app.api.v1.notifications.send_notification`. The business layer imports API modules: `app/bll/notification_bll.py:5,11`. `app/notifications/inapp.py:5` imports `app.api.v1.ws.notifications`. This creates import-order fragility and circular-import risk.
- Shared infrastructure is duplicated: two Celery apps, three Redis pub/sub listeners, two `get_db` implementations (`app/db/session.py:10`, `app/dependencies/db.py:3`, plus a third in `app/dependencies/auth.py:11`), and two `BaseSMSService` classes (`app/sms/base.py`, `app/email/base.py`).

### 5.4 Data flows

**Authentication flow** (`app/api/v1/routes/auth.py`, `app/dependencies/auth.py`, `app/core/security.py`):

1. `POST /v1/api/auth/token` uses an OAuth2 password form; `username` is the phone number. The handler verifies the bcrypt hash and rejects inactive users.
2. It appends the login timestamp to the `users.last_logins` JSON column (capped at 50), then issues an access JWT (`sub`, `phone_number`, `scope=access_token`, 24 h) and a refresh JWT (`scope=refresh_token`, 7 d).
3. `get_current_user` decodes **any** valid token **without checking `scope`** and **without checking `is_active`** [Verified].
4. `POST /auth/refresh` requires a valid **access** token in addition to the refresh token. Refresh is therefore impossible once the access token expires, which defeats its purpose [Verified 401].
5. Logout is a no-op (`auth.py:85-89`); there is no revocation or blacklist.

**Authorization model:**

- Roles: `superuser` (id 1) and `user` (id 2), seeded in `app/db/init_data.py:20-28`.
- `Permission` and `RolePermission` tables plus CRUD exist, but **no endpoint checks permissions**. Permissions are used only to choose notification recipients (`app/crud/user.py:136-150`).
- The only enforcement is an ad-hoc `is_superuser` check in 15 endpoints (8 device approval/status endpoints, 5 camera approval/activation endpoints, role delete, permission delete). There are no ownership checks anywhere.

**Multi-tenancy:** none. There is no organization, farm, site or greenhouse entity. Devices have no owner. Every authenticated user sees and can modify every device, sensor, camera and user.

**Device registration flow:**

1. `POST /devices/register` (unauthenticated) → `register_device_bll` → `crud.register_device`. This looks up by `mac_address`, not `unique_code`, and returns the existing record with only IP and online status updated.
2. `notify_users` runs, then the router calls `notify_users` **again** (`app/api/v1/device.py:106-109` and `app/bll/device_bll.py:11-13`).
3. Each call builds per-user, per-channel payloads and enqueues the Celery task `notifications.send`.

**Command flow:**

1. `POST /commands/` (unauthenticated) inserts a `Command(status=pending)` row.
2. `publish_command` → `mqtt_publish("greeni/device/{device_id}/commands", …)` using the **integer DB id**. If MQTT is disconnected, the message is **silently dropped** and the API still returns 200 [Verified].
3. The MQTT handler has a branch for `…commands…/status` (`app/iot/mqtt/handler.py:37-39`), but the client never subscribes to that topic (`app/iot/mqtt/client.py:46`), and the update calls a nonexistent `crud_command.get_by_uuid` (`handler.py:67`). **The status loop is never closed.**

**Notification flow:**

1. A producer (device BLL or `POST /notifications/`) enqueues `notifications.send` (`app/tasks/notifications_tasks.py`).
2. The worker runs `NotificationManager.send` → `render_template` → the channel service.
3. For in-app, `InAppService` stores the row and publishes JSON to the Redis channel `notifications` (`app/notifications/inapp.py:28`).
4. **Two** API-side listeners consume that channel: `app/main.py:66-98`, which forwards to the `/ws/notifications` registry, and `app/api/v1/ws/notifications_ws.py:56-86`, which forwards to the `/ws/database-notifications` registry. A third, unused listener sits in `app/api/v1/ws/notifications.py:59-75`.
5. `notify_device_status_change` **also** pushes directly via `asyncio.run(send_to_user(...))` from a threadpool thread (`app/bll/notification_bll.py:113-124`). **[Inferred]** This uses WebSocket objects bound to the main event loop from a different loop. Behaviour is undefined: it is likely to raise, which makes `send_to_user` treat the socket as dead and drop the user's live connection.

**Media flow:** upload is saved to `./media` with a SHA-256 de-dup side file (`*.hash`). Files are served **publicly** via `StaticFiles` at `/media` (`app/main.py:51-53`), and an MQTT download command goes to every active, online multimedia device.

**Camera flow:** `CameraStreamingService.get_connected_camera` → `CameraManager` (in-process OpenCV `VideoCapture`) → MJPEG generator or an ffmpeg HLS subprocess. All state lives in process memory singletons.

### 5.5 Event, background and messaging systems

| System | Implementation | Observations |
|---|---|---|
| Celery | `app/celery.py` (broker/backend hardcoded `localhost`) and `app/worker.py` (env-driven); worker starts `-A app.worker` | Tasks are declared on `app.celery.celery_app` but the worker starts `app.worker`. **[Inferred]** This works only because Celery shares `@app.task` tasks across app instances by default, and because Celery's settings let the `CELERY_BROKER_URL` env var override the hardcoded broker (I confirmed this override in Celery's source). The result backend stays `localhost:6379/1` unless `CELERY_RESULT_BACKEND` is set; no `.env` template exists to confirm what is set. There is no Celery Beat, no retries, and no dead-lettering |
| Redis pub/sub | Channel `notifications` | Duplicate consumers (see above) |
| MQTT | `gmqtt` client created at import time, fixed client ID `greeni-device-client` | A fixed client ID means a second API replica or worker would kick the first off the broker. The only subscription is `greeni/device/+/media/media_list`. QoS 0, no retain, no LWT, no reconnect strategy for the initial connect; a failed connect is silent [Verified] |
| WebSockets | 6 endpoints | Connection registries are in-process dicts. Real-time delivery across multiple API processes works only through the Redis fan-out |
| In-process background tasks | `asyncio.create_task` at import time (`notifications_ws.py:86`) and in startup | The import-time task breaks any non-uvicorn import ([Verified] `RuntimeError: no running event loop`) |

### 5.6 Storage architecture

- **PostgreSQL** is the system of record (21 tables in metadata plus 1 orphan; see [§10](#10-database-review)).
- **Redis** serves four roles at once: Celery broker and backend (db 0/1), RedisJSON cache (roles, tags), and pub/sub. There is no Redis password.
- **Local filesystem:** `./media` for uploaded media (inside the bind-mounted source tree in compose); `./logs/app.json`; `./mqtt_logs.json`; `/tmp/streams` for HLS segments (not served, not persisted).
- **Mosquitto** persistence volume.

### 5.7 Architecture observations

1. **Positive:** a clear domain decomposition (devices, sensors, cameras, notifications, IoT), a uniform response envelope (`BaseAPIResponse`) on most routers, sensible infrastructure choices (Postgres, Redis, Celery, MQTT), and a Redis pub/sub fan-out for real-time in-app notifications, which is the right pattern for multi-process delivery.
2. **Edge and server concerns are mixed.** Hardware protocol drivers, GPIO, and executors that run `sudo reboot` belong to an edge agent, not a central API.
3. **The process-local state blocks horizontal scaling:** camera connections, streams, PTZ sessions, protocol instances, WebSocket registries, and the fixed MQTT client ID.
4. **Sync I/O on the event loop.** 40 `async def` REST handlers call synchronous SQLAlchemy, OpenCV and ONVIF (SOAP) code directly, blocking the loop for every request in flight.
5. **No contract for device↔server messaging.** Topic names and device identifiers are inconsistent ([§17](#17-business-logic-review)).

---

## 6. Feature Inventory

Status legend: **Complete** · **Partial** (works with significant gaps) · **Broken** (doesn't work as written) · **Stub** (placeholder) · **Unused** (unreachable or dead) · **Planned** (only config or comments)

| # | Feature | Purpose | Status | Main files | Dependencies | Key risks |
|---|---|---|---|---|---|---|
| F1 | Login / JWT | Authenticate users | **Partial** | `api/v1/routes/auth.py`, `core/security.py`, `dependencies/auth.py` | python-jose, passlib | Default secret, refresh token usable as access, refresh endpoint unusable, no revocation, inactive users keep access |
| F2 | User management | CRUD users, login history | **Partial** | `api/v1/user.py`, `crud/user.py`, `schemas/user.py` | — | No authz: any user can edit or delete any user; permission errors become 500s |
| F3 | Roles & permissions (RBAC) | Role/permission admin | **Partial / Unused for enforcement** | `api/v1/role.py`, `bll/role_service.py`, `crud/role.py`, `crud/permission.py`, `models/role*.py` | Redis Stack (RedisJSON) | Permissions never enforced; role endpoints fail without RedisJSON; PATCH schemas broken |
| F4 | Seed data | Bootstrap roles, permission, admin | **Complete but insecure** | `db/init_data.py` | — | Hardcoded admin with password `1234` and real-looking personal data |
| F5 | Device registry & approval | Self-registration, admin approve/reject, activate/deactivate | **Partial** | `api/v1/device.py`, `crud/device.py`, `bll/device_bll.py` | Celery | Approval bypassable via PATCH; `GET /devices/{id}` shadowed; identity by MAC vs code is inconsistent; delete fails once commands exist |
| F6 | Device online/offline via device key | Device presence | **Partial** | `api/v1/device.py:242-264` | — | `local_key` leaked by unauthenticated GET; no heartbeat/timeout |
| F7 | Device event notifications | Alert admins on register/status change | **Broken** | `bll/notification_bll.py`, `notifications/*` | Celery, Redis | No recipients with default seed [Verified]; missing template raises; duplicate sends; cross-loop WS push |
| F8 | Sensor types / attributes / tags | Sensor catalog | **Partial** | `api/v1/sensor.py`, `crud/sensor.py`, `api/v1/tag.py`, `bll/tag_service.py` | Redis Stack (tags) | 38 sensor ops, 33 unauthenticated; assign/unassign tags → 500 [Verified]; PATCH needs all fields [Verified]; `GET /sensors/values` unreachable [Verified] |
| F9 | Sensor attribute values | Current value per sensor attribute | **Partial** | `crud/sensor.py:389-486` | — | History table `sensor_attribute_value_history` never written |
| F10 | Sensor data (time series) | Historical readings | **Partial** | `api/v1/sensor_data.py`, `crud/sensor_data.py` | — | Manual REST only, unauthenticated, no ingestion pipeline, no index for time queries |
| F11 | Device telemetry over MQTT | Ingest device data | **Broken / not wired** | `iot/device_data/receiver.py`, `crud/device_data.py`, `models/device_data.py`, `iot/mqtt/topics.py` | MQTT | Never subscribed; imports nonexistent `async_session` (`crud/device_data.py:7`) |
| F12 | Hardware protocol layer | GPIO/I2C/Modbus/LoRa/Zigbee drivers | **Stub / Unused / Broken import** | `iot/protocols/*` | — | All reads return hardcoded values; `protocol_manager.py` fails to import (empty `spi.py`/`uart.py`/`pwm.py`) [Verified]; nothing calls it |
| F13 | Command dispatch | Send actuator commands via MQTT | **Partial** | `api/v1/command.py`, `crud/command.py`, `iot/mqtt/publisher.py` | MQTT | Unauthenticated [Verified]; silent drop when disconnected [Verified]; no status feedback |
| F14 | Command executors / dispatcher / scheduler | Execute commands | **Stub / Broken** | `iot/commands/**` | — | Empty files; `actuator.py` imports nonexistent `client` [Verified]; `system.py` references undefined attributes and runs `sudo reboot` |
| F15 | Media library & distribution | Upload files, push to devices | **Partial** | `api/v1/media.py`, `main.py:51-53` | MQTT | Broadcast reaches no devices (`is_multimedia` not settable via API); stored XSS [Verified]; no size/type limits |
| F16 | Media control over WebSocket | Play/list/delete/download on devices | **Partial** | `api/v1/ws/media_ws.py`, `iot/mqtt/handler.py` | MQTT | Unauthenticated; topic injection; fire-and-forget |
| F17 | Camera CRUD & approval | Register/approve cameras | **Partial** | `api/v1/camera.py`, `camera/crud/camera.py`, `camera/schemas/camera.py` | — | Camera password returned in responses [Verified] |
| F18 | Camera streaming (frame/MJPEG/HLS) | Live video | **Broken** | `camera/utils/camera_utils.py`, `camera/manager/*.py` | OpenCV, ffmpeg | `is_connected` missing [Verified]; `Response(content=async_gen)`; HLS path never served; ffmpeg not installed |
| F19 | PTZ control | Pan/tilt/zoom, presets | **Broken** | `api/v1/ptz.py`, `camera/manager/ptz_controller.py` | onvif-zeep | WSDL placeholder path; presets never persisted; sync SOAP calls in async handlers |
| F20 | Recording / retention | Store camera footage | **Planned** | `camera/models/camera.py:50-51` | — | Fields only |
| F21 | SMS notifications | sms.ir, Melipayamak | **Partial** | `sms/*` | requests | Hardcoded sms.ir template ID; Melipayamak argument order reversed; no timeouts on `requests.post` |
| F22 | Email notifications | SMTP | **Stub** | `email/smtp_service.py` | — | Only `print()`s; SMTP password stored in plaintext |
| F23 | Push notifications | FCM/OneSignal | **Stub** | `notifications/push.py` | — | `# TODO`; wrong constructor args |
| F24 | In-app notifications | DB + real-time WS | **Partial (works)** | `crud/inapp_notification.py`, `api/v1/inapp.py`, `api/v1/notifications.py`, `api/v1/ws/notifications*.py` | Redis pub/sub | REST unauthenticated with IDOR [Verified]; duplicated listeners |
| F25 | Provider management | Configure SMS/email/push providers | **Partial** | `api/v1/{sms,email,push}_provider.py`, `crud/*_provider.py` | — | Unauthenticated, leaks secrets [Verified]; non-atomic activation; partial-update schemas broken |
| F26 | Caching | Roles/tags in RedisJSON | **Partial** | `cache/*`, `bll/role_service.py`, `bll/tag_service.py` | Redis Stack | Hardcoded host; requires the RedisJSON module; `KEYS` scans; staleness |
| F27 | Health & system info | Liveness, host metrics | **Partial** | `core/healthcheck.py`, `api/v1/ws/system_*.py` | psutil | 200 when degraded [Verified]; unauthenticated host metrics WS; `cpu_freq()` can be `None` in containers |
| F28 | Celery demo task | `add(x, y)` example | **Unused / should be removed** | `tasks/example.py`, `api/v1/task.py` | Celery | Unauthenticated demo endpoint in production router |
| F29 | Command WebSocket | Push command status to UI | **Unused** | `api/v1/ws/command_ws.py` | — | Router never included |
| F30 | Telegram integration | Notify superuser via Telegram | **Planned** | `core/config.py:9-10` | — | Config only |
| F31 | OTP SMS login | Phone OTP | **Planned** | `notifications/templates.py:15-18` | — | Template with `{code}` commented out |

---

## 7. Code Quality Review

### 7.1 Metrics

| Metric | Value | Source |
|---|---|---|
| Python files / lines (`app/`) | 150 / 11,562 | `find`/`wc` |
| `print()` calls | **259** | grep |
| Commented-out code lines | **~290** (heuristic regex) | grep |
| Empty `.py` files (excluding `__init__`) | **14** | [Appendix C](#appendix-c--dead-empty-and-duplicate-files) |
| Duplicate/backup files | `app/crud/device copy.py`, `requirementsback.txt`, `requirements.lock.txt` (stale), empty `To` | — |
| `async def` REST handlers doing sync DB/IO | 40 | grep |
| `datetime.utcnow()` uses (deprecated since 3.12) | 26 | grep |
| Automated tests | **0** | — |

### 7.2 Maintainability and readability

- **Debug output instead of logging.** `app/api/v1/media.py` (upload/delete) and `app/api/v1/ws/media_ws.py` are dominated by `print()` banners, for example `media.py:246-282` with 20+ prints for one MQTT publish. This makes logs unparseable and bypasses the structured loguru setup.
- **Large blocks of commented-out code** sit at the top of files, often a full previous version: `media.py:1-207`, `media_ws.py:257-335`, `inapp.py:162-196`, `Dockerfile:1-36`, `models/role.py:7-32`, `models/permission.py:6-32`, `bll/device_bll.py:17-47`, `crud/device.py:141-192`.
- **Mixed languages.** Code comments and many user-facing error messages are in Persian (`auth.py:20,29,59`, `dependencies/auth.py:21`), others are English, and some OpenAPI summaries are Persian (`sensor.py:83+`). There is no i18n strategy.
- **Inconsistent idioms:** Pydantic v1 vs v2 APIs (`.dict()` vs `.model_dump()`, `from_orm` vs `model_validate`, `class Config` vs `model_config`); SQLAlchemy legacy `Query.get()` (`crud/role.py:11`, `crud/permission.py:34,57`, `crud/sensor.py:152,179`); deprecated `sqlalchemy.ext.declarative.declarative_base` (`db/base_class.py:1`).

### 7.3 Complexity

Individual functions are mostly simple. The complexity sits in **cross-module coupling**: import-time side effects, singletons, and global loggers (`logger.remove()` in `iot/mqtt/client.py:26`, then again in `core/logging_config.py:48`, so `mqtt_logs.json` is always empty [Verified]). The largest modules, `api/v1/sensor.py` (737 lines, 38 endpoints) and `crud/sensor.py` (638 lines), would benefit from splitting by resource.

### 7.4 Duplication

| Duplicate | Locations |
|---|---|
| `get_db` | `db/session.py:10`, `dependencies/db.py:3`, `dependencies/auth.py:11` |
| `get_password_hash` / `verify_password` / `CryptContext` | `core/security.py:7-19`, `crud/user.py:12-20` |
| WebSocket token auth + connection registry + Redis listener | `api/v1/ws/notifications.py`, `api/v1/ws/notifications_ws.py`, `main.py:66-98` |
| Celery app | `app/celery.py`, `app/worker.py` |
| `PTZControl` model | `camera/models/camera.py:64-87` and `camera/models/ptz.py` (import of the latter raises [Verified]) |
| PTZ schemas | `camera/schemas/camera.py:104-131` and `camera/schemas/ptz.py` |
| `InAppNotificationCreate` / `InAppNotificationResponse` / `InAppNotificationListResponse` | Each defined twice in `schemas/inapp_notification.py` (13/69, 38/91, 57/108); the later definitions win silently |
| `SensorAttributeShort` / `SensorAttributeValueWithAttr` | `schemas/sensor.py:217-231` and `schemas/device_full.py:7-21` |
| Model→dict helpers | `utils/sqlalchemy_to_dict.py`, `bll/role_service.py:19-41`, `Device.to_dict`/`to_dict_simple`, `InAppNotification.to_dict` |
| Device CRUD | `crud/device.py` vs `crud/device copy.py` |
| Route function names redefined | `api/v1/device.py`: `get_devices` (122, 127), `get_device_by_unique_code` (132, 139), `update_device` (267, 275) |
| Reject-device endpoints | `DELETE /devices/{id}/reject` (`device.py:165`) and `DELETE /devices/pending/{id}/reject` (`device.py:302`), with different pending checks |
| `/devices/` vs `/devices/half` | Same implementation (`device.py:121-129`); `get_all_devices_half` in CRUD is unused |

### 7.5 Dead code and unused components

See [Appendix C](#appendix-c--dead-empty-and-duplicate-files). Highlights:

- the entire `iot/protocols/` package (nothing references `protocol_manager`);
- `iot/commands/**`;
- `iot/device_data/receiver.py`;
- `iot/mqtt/topics.py`;
- `api/v1/ws/command_ws.py` (never mounted);
- `crud/notification.py` and `models/notification.py` (table never created);
- `models/sensor.py:117-133` (`SensorAttributeValueHistory`);
- `cache` methods `_convert_dict_numeric_keys_to_list` hacks;
- `crud/inapp_notification.py:122` `delete_read_notifications`;
- camera CRUD `get_with_ptz`, `get_all_active`, `update_status`;
- `TELEGRAM_*` settings.

### 7.6 Unused dependencies

`smsir-python` (the code uses raw `requests`), `PyYAML`, and `python-dateutil` are not imported anywhere in `app/` (the last two arrive transitively). `onvif-zeep`/`zeep` are only used by the broken PTZ feature.

### 7.7 Circular dependencies and architecture violations

- `crud → api` (`crud/device.py:12`), `bll → api` (`bll/notification_bll.py:5,11`), `notifications → api.ws` (`notifications/inapp.py:5`).
- These currently import successfully only because of import ordering. Any refactor that imports `app.crud.device` earlier, for example from a Celery task, risks a cycle through `app.api.v1.__init__`, which imports every router.

### 7.8 Anti-patterns

- **Import-time side effects:** `asyncio.create_task` (`ws/notifications_ws.py:86`), `atexit` hooks, `os.makedirs(..., mode=0o777)` (`main.py:52`), `LOG_DIR.mkdir()` (`logging_config.py:20`), MQTT client creation and `load_dotenv()` in three modules.
- **`HTTPException` raised from the data layer** (`crud/permission.py`, `crud/sms_provider.py`, `crud/sensor.py`) while other CRUD raises `ValueError`/`PermissionError`. The latter surface as 500s ([Verified] on role escalation attempt).
- **Swallowed exceptions:** `except Exception: logger.error(...); return None/False` throughout the camera and PTZ managers hides root causes. That is how the missing `is_connected` method surfaces only as "Failed to connect camera".
- **Business logic in routers:** inline queries and formatting in `sensor.py:427-517`.
- **Wrong response handling:** `BaseAPIResponse(..., message=...)` is used in about a dozen places (tags, sensors, cameras), but `BaseAPIResponse` has no `message` field (`schemas/common.py:21-25`), so the message is silently dropped.
- **Mutable default literal** on an ORM column: `User.last_logins = Column(JSON, default=[])` (`models/user.py:29`). This is a code smell rather than a live bug, because `create_user` passes `[]` explicitly.

### 7.9 Naming inconsistencies

- URL prefixes: `/sms-providers`, `/email-providers` vs `/push_provider`, `/inapp_provider` (which isn't a provider).
- Status enums: `CameraStatus` values `"onLine"`/`"offLine"`.
- Similar endpoints: `on_line`/`off_line` vs `online`/`offline` (`device.py:209-264`).
- Comments vs file names: `core/level_permissions.py` has header comment `# app/core/device_permissions.py`; `tasks/notifications_tasks.py` says `# app/tasks/notifications.py`.
- An executable-named file with a trailing space: `app/iot/protocols/rtsp.py ` (filename ends in a space).

---

## 8. Frontend Review

**There is no frontend code in this repository.** `docker-compose.yml:104-120` defines a `frontend` service that builds from `../nginx`, a sibling directory outside this repo. Consequences:

- `docker compose up` fails on a fresh clone unless that sibling directory exists. **[Inferred]** I could not run Docker in the sandbox, but a missing build context is a hard failure in Compose.
- Routing, layouts, components, state management, form validation, accessibility and performance **cannot be assessed**.
- From the backend's point of view, these API traits will burden any frontend:
  - inconsistent response envelopes (some routes return `BaseAPIResponse`, others raw lists or dicts; see [§11](#11-api-audit));
  - mixed-language error messages;
  - no pagination metadata on most list endpoints;
  - `/devices/full` returns every device with nested sensors, types, attributes and values in one call (`device.py:24-42`), which will degrade as data grows;
  - WebSocket auth via query-string token.
- **Missing pieces that a frontend team will need:** an OpenAPI contract that matches reality (the custom OpenAPI marks *every* operation as OAuth2-secured, `main.py:178-180`, including the public ones), a stable error schema, and CORS configured for real origins.

---

## 9. Backend Review

### 9.1 APIs and services

There are 154 HTTP operations and 6 WebSocket endpoints. Full list in [Appendix A](#appendix-a--endpoint-inventory); analysis in [§11](#11-api-audit).

### 9.2 Models and business logic

See [§10](#10-database-review) and [§17](#17-business-logic-review).

### 9.3 Validation

- **Too permissive:**
  - `UserCreate.password: str` has no length or complexity rules (`schemas/user.py:16`).
  - `phone_number` has no format validation.
  - Device `lat`/`lon` are free strings.
  - `Command.payload` is a free string though it is documented as JSON (`models/command.py:17`).
  - `streaming_quality`, `status` and `source` fields are free strings rather than enums.
  - `PTZMove.duration` is unbounded, so an operator can make a camera move for hours.
- **Too strict / broken (Pydantic v2 semantics):** `Optional[X]` without a default is **required**. This breaks partial updates for `PermissionUpdate` (`schemas/permission.py:15-19`), `SMSProviderUpdate` (`schemas/sms_provider.py:14-17`), `PushProviderUpdate` (`schemas/push_provider.py:14-17`), `SensorTypeUpdate` (`schemas/sensor.py:71-78`) and `RoleNameUpdate` (`schemas/role.py:21-22`) [Verified 422].
- **Optional request bodies by accident:** `payload: SensorTypeCreate = None` and similar in PUT/PATCH/POST handlers (`sensor.py:122, 137, 241, 256, 290, 302, 365, 381, 560`). Omitting the body results in `None.model_dump()`, which is a 500.
- **Inconsistent range validation:** the PTZ API validates pan/tilt in 0–1 (`api/v1/ptz.py:90`) while the controller documents −1…1 (`ptz_controller.py:94-95`).

### 9.4 Error handling

- There is no global exception handler or error schema.
- `ValueError`/`PermissionError` from CRUD become 500s [Verified].
- Internal exception text is returned to clients (`media.py:442, 484, 541`; `healthcheck.py` returns raw DB/Redis errors publicly).
- Camera and PTZ managers swallow every exception.
- The Celery task doesn't close the DB session on exception (`tasks/notifications_tasks.py:9-12`) and has no retry policy.

### 9.5 Authentication, authorization and permissions

See [§5.4](#54-data-flows) and [§12](#12-security-audit).

### 9.6 Background processing

- Celery is used only for notifications.
- There are no scheduled jobs (Beat). Device offline detection, notification cleanup (`delete_read_notifications` exists but is never called), media retention and camera retention would all need them.

### 9.7 Caching

`BaseCache` (`cache/base_cache.py`) stores JSON via `redis_client.json()`, which **requires the RedisJSON module** (hence the `redis-stack-server` image). Problems:

- the host is hardcoded as `greeni_redis` (`cache/redis.py:5`), ignoring `REDIS_URL`;
- invalidation uses `KEYS` (O(N), blocking) (`base_cache.py:41`);
- a 10-hour TTL on roles (`role_service.py:16`);
- cached tags are rebuilt as **transient ORM objects** (`Tag(**item)`, `tag_service.py:17,31`);
- role caches are not invalidated when permissions change (`crud/permission.py`), so stale data is served.

The tables involved are tiny, so the cache adds risk without measurable benefit.

### 9.8 Scalability

- **Event-loop blocking:** sync DB calls in 40 `async def` handlers, OpenCV `cap.read()` in MJPEG generators, blocking ONVIF SOAP calls, and synchronous file hashing and IO in `upload_files`.
- **Process-local state** (camera/PTZ/streams/WS registries) and a fixed MQTT client ID prevent running more than one replica.
- **Unbounded queries:** `GET /users/`, `/devices/`, `/devices/full`, `/sms-providers/`, `/sensors/{id}/attribute-values`; `by-protocol` endpoints load everything then slice in Python (`sensor.py:605, 680, 695`).
- **Write amplification:** every login rewrites a JSON array on the user row (`crud/user.py:113-123`).
- Compose runs a **single uvicorn process with `--reload`** (`docker-compose.yml:74`).

### 9.9 Weaknesses and missing functionality (summary)

The main gaps are:

1. no permission-based authorization or ownership;
2. no migrations;
3. no telemetry ingestion;
4. no command acknowledgement;
5. no device heartbeat or offline detection;
6. no real email or push;
7. no working video;
8. no global error handling;
9. no tests;
10. no rate limiting;
11. no audit trail.

---

## 10. Database Review

### 10.1 Schema overview

| Table | PK | Key columns / FKs | Notes |
|---|---|---|---|
| `users` | int | `role_id → roles.id` | `last_logins` JSON array; naive `DateTime` |
| `roles` | int | — | `role_permissions` eager-loaded `lazy="joined"` |
| `permissions` | int | `name` unique | Naive `DateTime` with `default=func.now()` (client-side SQL expression) |
| `role_permissions` | int | `role_id`, `permission_id` (CASCADE) | **No unique (role_id, permission_id)** |
| `devices` | int | `unique_code` unique; `uuid` unique+index | `uuid` default evaluated once at import (`models/device.py:26`) [Verified]; `lat`/`lon` strings; `mac_address` not unique but used as identity |
| `sensors` | int | `sensor_type_id`, `device_id`, `created_by`, `updated_by` | Soft delete; **no FK indexes** |
| `sensor_types` | int | `name` unique | `created_at` naive vs `updated_at` tz-aware in the same table (`models/sensor.py:53-54`) |
| `sensor_attributes` | int | `sensor_type_id` | Soft delete; no index |
| `sensor_attribute_values` | int | `sensor_id`, `attribute_id` (CASCADE) | No unique (sensor_id, attribute_id); no indexes |
| `sensor_attribute_value_history` | int | `sensor_id`, `attribute_id` | **Never written** |
| `sensor_data` | int | `sensor_id`, `attribute_id` | Time series with **no (sensor_id, created_at) index**; `value` as string |
| `device_data` | **UUID** | `device_id → devices.uuid` | Different FK target than every other table (`devices.id`); naive timestamp; never written |
| `commands` | int | `device_id → devices.id` NOT NULL | No index on `device_id`; free-text status |
| `tags` | int | `name` unique | |
| `sensor_type_tags` | (composite) | M:N | Fine |
| `cameras` | int | `device_id` (indexed) | Plaintext `password`; enum stored as PG enum |
| `ptz_controls` | int | `camera_id` (indexed) | Mixes "current position" and "preset" rows in one table |
| `sms_providers` | **varchar(36)** | `name` unique | Plaintext `api_key` |
| `email_providers` | int | `name` unique | Plaintext `password` |
| `push_providers` | int | `name` unique | Plaintext `api_key`; no timestamps |
| `inapp_notifications` | int | `user_id` (indexed), `is_read` (indexed) | Good; consider composite (user_id, created_at) |
| `notifications` | int | — | **Declared on a separate `declarative_base()`** (`models/notification.py:6`). Never part of `Base.metadata`, so never migrated or created [Verified] |

### 10.2 Migrations

- **No migration files are tracked** (`.gitignore:17`). On a fresh DB, `alembic upgrade head` succeeds but creates only `alembic_version` [Verified]. `startup_event` → `init_data` would then fail on the missing tables. **[Inferred, high confidence]** I did not run that sequence end to end.
- The comment on that `.gitignore` line, in Persian, reads *"optional — if you don't want migrations to go [to git]"*, so this was a deliberate choice. Every developer's database is therefore **unreproducible**, and schema drift between environments is guaranteed.
- `alembic.ini:57` contains a hardcoded URL with a password, which `alembic/env.py:19` overrides from `DATABASE_URL`. If `DATABASE_URL` is unset, `set_main_option(..., None)` raises.

### 10.3 Missing indexes

`sensors.device_id`, `sensors.sensor_type_id`, `sensor_attributes.sensor_type_id`, `sensor_attribute_values.sensor_id`/`attribute_id`, `sensor_data(sensor_id, created_at DESC)`, `commands.device_id`, `commands(status)`, `device_data(device_id, timestamp)`, `users.role_id`, `role_permissions(role_id, permission_id)`.

### 10.4 Missing constraints

- Unique `(role_id, permission_id)`; unique `(sensor_id, attribute_id)` if "current value" semantics are intended.
- `CHECK`/enum constraints for `status`, `source`, `type`, `streaming_quality`, `commands.status`.
- `NOT NULL` on `devices.uuid`, `sensors.device_id` (if sensors must belong to a device), and `sensor_attributes.sensor_type_id`.
- Only one active provider per type should be enforced. Today this happens in application code (`crud/sms_provider.py:44`) and could be done with a partial unique index.

### 10.5 Relationships and data integrity risks

- **Device delete fails once commands exist.** `commands.device_id` is `NOT NULL` with no `ON DELETE` and no ORM relationship, so Postgres raises an FK violation and the API returns 500. **[Static]**
- **Device delete orphans sensors.** `Device.sensors` has no delete cascade, so SQLAlchemy nulls `sensors.device_id` instead of deleting or blocking. **[Static]**
- **User delete fails once the user has in-app notifications** (`inapp_notifications.user_id` NOT NULL FK) or has created sensors or tags. **[Static]**
- **Soft vs hard delete is inconsistent.** Sensors, types and attributes are soft-deleted; devices, users, tags, cameras and notifications are hard-deleted. `api/v1/device.py:39,69` has a commented-out filter on a nonexistent `Device.is_deleted`.
- **Timezones are mixed:** naive `DateTime` (users, roles, permissions, cameras, ptz, device_data, sensor_types.created_at) alongside `DateTime(timezone=True)`.
- **Inconsistent PK strategies:** int, UUID and `String(36)` are all in use.
- **Three identifiers per device** (`id`, `uuid`, `unique_code`), plus MAC as a de facto identity in `register_device`.

### 10.6 Redundant / overlapping tables

Four telemetry-ish stores overlap:

- `sensor_attribute_values` (current value, with status),
- `sensor_attribute_value_history` (unused),
- `sensor_data` (time series per sensor/attribute),
- `device_data` (JSON blobs per device, unused).

The team should pick one time-series design, for example `sensor_data` on TimescaleDB with a "latest value" materialized view, and drop the rest. `notifications` (outbox/log of sent notifications) is designed but orphaned.

---

## 11. API Audit

### 11.1 Summary by router

Auth column: **JWT** = `get_current_user`; **NONE** = no authentication; **Admin** = additional `is_superuser` check in the handler.

| Router (prefix) | Ops | Auth | Envelope | Notable issues |
|---|---|---|---|---|
| `/v1/api/auth` | 4 | token: NONE; others JWT | Raw dict | Refresh requires access token; logout no-op |
| `/v1/api/users` | 8 | JWT (no admin checks) | `BaseAPIResponse` | Any user can update or delete any user; `/{user_id}/login` lets anyone append login timestamps for others |
| `/v1/api/roles` | 5 | JWT (delete: Admin) | `BaseAPIResponse` | Requires RedisJSON; any user can rewrite role permissions |
| `/v1/api/permissions` | 4 | JWT (delete: Admin) | `BaseAPIResponse` | PUT needs all fields |
| `/v1/api/devices` | 24 | 18 JWT (8 Admin), 6 NONE | `BaseAPIResponse` | `/{device_id}` shadowed; unauthenticated GET leaks `local_key`; PATCH/PUT/DELETE without admin check; `/multimedia` response model mismatch (returns `List[str]` into `List[DeviceMultiMediaPublic]`, a 500 when non-empty) **[Static]** |
| `/v1/api/sensors` | 38 | 33 NONE, 5 JWT | `BaseAPIResponse` | Writes unauthenticated; assign-tags 500; `/values` unreachable; Persian summaries |
| `/v1/api/sensors/data` | 3 | NONE | `BaseAPIResponse` | Unauthenticated ingest and read |
| `/v1/api/tags` | 5 | Reads NONE, writes JWT | `BaseAPIResponse` | Delete `message` dropped |
| `/v1/api/commands` | 3 | **NONE** | `BaseAPIResponse` | Unauthenticated actuator control |
| `/v1/api/cameras` | 16 | JWT (5 Admin) | Mixed | Password in responses; streaming broken; `/device/{id}` returns a different envelope (`CameraListResponse`) |
| `/v1/api/ptz` | 9 | JWT | Raw dict | Broken; presets list returns raw ORM objects |
| `/v1/api/media` | 4 | JWT | Raw dict | XSS; any user can delete all media |
| `/v1/api/notifications` | 3 | send: **NONE**; reads JWT | Raw | Unauthenticated SMS/email send |
| `/v1/api/inapp_provider` | 8 | **NONE** | Raw dict | IDOR on every operation |
| `/v1/api/sms-providers` | 9 | **NONE** | Raw ORM | Leaks `api_key`; DELETE returns 204 |
| `/v1/api/email-providers` | 4 | **NONE** | `EmailProviderOut` (includes password) | Non-atomic activation |
| `/v1/api/push_provider` | 5 | **NONE** | `PushProviderOut` (includes api_key) | — |
| `/v1/api/tasks` | 1 | **NONE** | Raw | Demo endpoint |
| `/health` | 1 | NONE | Raw | 200 when degraded |
| WS `/v1/api/ws/media` | 1 | **NONE** | — | Publishes MQTT commands |
| WS `/v1/api/ws/system-info` | 1 | **NONE** | — | Host telemetry |
| WS `/v1/api/ws/health`, `/v1/ws/health` | 2 | NONE | — | Mounted twice (`main.py:56`, `api/v1/__init__.py:42`) |
| WS `/v1/api/ws/notifications`, `/database-notifications` | 2 | JWT in query string | — | Two parallel implementations |

**Totals:** 160 routes (154 HTTP + 6 WS); **81 with no authentication** [Verified by dependency introspection]. Of those, only about 6 are plausibly meant to be public: `/health`, `/auth/token`, `/devices/register`, `/devices/status/{uuid}`, and device-key online/offline.

### 11.2 Versioning

The prefix is `/v1/api/...`, an unusual ordering; `/api/v1` is conventional. There is no strategy for v2 or deprecation, and WebSockets live under both `/v1/ws` and `/v1/api/ws`.

### 11.3 Consistency issues

- **Response shapes:** mixed `BaseAPIResponse`, raw ORM, raw dicts and Pydantic lists, with no consistent error envelope.
- **HTTP semantics:**
  - `DELETE` used for "reject" (`/devices/{id}/reject`);
  - `POST` used for state transitions in some places (`/cameras/{id}/activate`, `/push_provider/{id}/deactivate`) and `PATCH` in others (`/devices/{id}/activate`, `/email-providers/{id}/deactivate`);
  - `PUT` endpoints that are actually partial;
  - verb paths like `/media/delete/{filename}` and `/media/delete-all`.
- **Identifiers:** SMS providers addressed by `name`, others by `id`; devices by `id`, `unique_code` and `uuid` on overlapping paths (which causes the shadowing).
- **Status codes:** create endpoints return 200 except SMS and email providers (201). Duplicate names return 400 (sensor types, SMS) or 409 (devices, permissions) or 500 (email providers [Verified]).

### 11.4 Documentation

- Auto-generated Swagger exists (`/docs`). The custom OpenAPI marks **every** operation as OAuth2-secured (`main.py:178-180`), so the docs misrepresent the 77 unauthenticated HTTP operations.
- No examples, no error documentation for most routes, no WebSocket documentation (FastAPI doesn't document WS), and no MQTT topic contract.
- **Undocumented APIs:** all 6 WebSocket endpoints; the MQTT topic interface (`greeni/device/{id}/commands`, `greeni/device/{code}/command`, `greeni/device/+/media/media_list`); the `/media/*` static file mount.

---

## 12. Security Audit

Severity scale: **Critical** (remote, unauthenticated or trivially reachable compromise of control, data or credentials) · **High** · **Medium** · **Low**.

<a id="sec-01"></a>
### SEC-01 — Unauthenticated control and data plane — **Critical** [Verified]

81 routes lack authentication ([Appendix A](#appendix-a--endpoint-inventory)). I confirmed these at runtime:

- `POST /v1/api/commands/` creates and publishes actuator commands (`api/v1/command.py:16-26`);
- `GET /v1/api/sms-providers/` returns `api_key` in plaintext;
- `GET /v1/api/email-providers/` returns the SMTP `password`;
- `POST`/`GET /v1/api/inapp_provider/...` creates and reads **any** user's notifications (IDOR);
- `POST /v1/api/notifications/` queues SMS or email to arbitrary recipients. This is a **toll-fraud and spam vector** that bills the organization's SMS account;
- full sensor catalog and data CRUD;
- WS `/ws/media` publishes MQTT commands to any device topic.

**Fix:** apply `dependencies=[Depends(get_current_user)]` at router level by default, add explicit role dependencies, and allow-list the genuinely public routes.

<a id="sec-02"></a>
### SEC-02 — Default JWT signing key — **Critical** [Verified]

`SECRET_KEY = os.getenv("SECRET_KEY", "secret")` (`app/core/config.py:12`). With the variable unset, a token forged with `"secret"` for `sub=1` was accepted as the superuser.

**Fix:** fail startup if the key is missing or weak, and rotate keys.

<a id="sec-03"></a>
### SEC-03 — Hardcoded superuser with weak password and personal data — **Critical** [Verified]

`app/db/init_data.py:39-52` creates a superuser with password **`1234`** and a hardcoded real-looking name, email and phone number. It runs on **every startup**: if that email isn't found, the account is re-created. Login with these credentials succeeded.

Publishing personal data in source is also a privacy and compliance issue.

**Fix:** remove it; bootstrap the admin through a one-time CLI or env with a forced password change.

<a id="sec-04"></a>
### SEC-04 — Missing authorization (privilege escalation and account takeover) — **Critical** [Verified]

Any authenticated user can:

- `PUT /users/{id}` with a new `password` for **any** user, including the superuser. I changed the superuser's password as a regular user and then logged in as the superuser (`api/v1/user.py:71-90`, `crud/user.py:82-109`);
- change other users' email, phone and `is_active` (lock out admins);
- `DELETE /users/{id}` for non-superusers;
- `PATCH /devices/{id}` with `is_pending=false,is_active=true` to **bypass admin approval** (`api/v1/device.py:266-280`; `DeviceUpdate` exposes these fields, `schemas/device.py:32-36`);
- `DELETE /devices/{id}` (no admin check, `device.py:282-288`);
- rewrite role-permission mappings (`PUT /roles/{id}`), create or edit permissions;
- delete all media (`DELETE /media/delete-all`);
- control any camera's PTZ, and update or delete any camera.

<a id="sec-05"></a>
### SEC-05 — Anonymous MQTT broker exposed — **Critical** [Static]

`mosquitto.conf:3` has `allow_anonymous true`. Listeners are on 1883 and 9001 (`0.0.0.0`), published on all host interfaces (`docker-compose.yml:125-127`). There are no ACLs. Anyone on the network can publish to `greeni/device/<id>/commands` (actuators) or spoof device messages, bypassing the API entirely.

**Fix:** per-device credentials or mTLS, ACLs restricting each device to its own topics, and don't publish the ports.

### SEC-06 — Token handling weaknesses — **High** [Verified]

- **Refresh tokens are accepted as access tokens.** `get_current_user` ignores `scope` (`dependencies/auth.py:24-30`), so a 7-day refresh token is a long-lived access token.
- **Deactivated users stay logged in.** `get_current_user` doesn't check `is_active` (`dependencies/auth.py:31-34`).
- **Long access lifetime:** 24 hours (`config.py:14`).
- **No revocation:** logout does nothing.
- **Tokens in URLs.** The WebSocket token is passed in the query string (`ws/notifications.py:81`, `ws/notifications_ws.py:91`), where proxy and access logs can capture it.

### SEC-07 — Secrets disclosed in API responses — **High** [Verified]

| Secret | Exposed by |
|---|---|
| SMS `api_key` | `GET /sms-providers/*` (unauthenticated, raw ORM) |
| SMTP `password` | `EmailProviderOut` (`schemas/email_provider.py:16`) |
| Push `api_key` | `PushProviderOut` (`schemas/push_provider.py:19`) |
| Camera `password` | `CameraResponse` inherits `CameraBase.password` (`camera/schemas/camera.py:22,57`) |
| Device `local_key` | `DevicePublic` via `DeviceBase.local_key` (`schemas/device.py:20`), served by **unauthenticated** `GET /devices/{unique_code}` and `/devices/by-unique-code/{code}` (`device.py:131-143`) |

The `local_key` leak lets anyone who knows a device code impersonate that device's online/offline state (`device.py:242-264`).

### SEC-08 — Stored XSS via media uploads — **High** [Verified]

`/media/upload` accepts any file type and size (`api/v1/media.py:285-412`) and serves it from the API's own origin through `StaticFiles` (`main.py:51-53`) with its natural content type. An uploaded `.html` was served as `text/html` to unauthenticated clients, so script runs in the API origin.

**Fix:**

- allow-list media types and enforce a size cap;
- serve files with `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff` from a separate domain;
- require auth if the media is not meant to be public.

### SEC-09 — Permissive CORS — **High** [Verified]

`allow_origins=["*"]` with `allow_credentials=True` (`main.py:43-49`). Starlette reflects any `Origin` (preflight returned `access-control-allow-origin: https://evil.example` and `allow-credentials: true`). Auth uses bearer tokens rather than cookies, so token theft via CORS is limited. However, combined with SEC-01, **any web page visited by someone on the greenhouse LAN can drive the unauthenticated API** (send commands, read provider credentials) from that person's browser.

### SEC-10 — Unauthenticated WebSockets — **High** [Static]

- `/ws/media` publishes `play`/`delete`/`download` to `greeni/device/{device_id}/command` using client-supplied `device_id` (`ws/media_ws.py:41-231`). A `device_id` containing `/` can address other topic levels (topic injection).
- `/ws/system-info` streams host CPU, RAM, disk and process counts.
- The `download` actions build URLs from client-supplied filenames.

### SEC-11 — Exposed RedisInsight and password-less Redis — **High** [Static]

RedisInsight is published on `8001` on all interfaces with no auth (`docker-compose.yml:41-55`). It has full access to Redis, which is the Celery broker (task injection, for example enqueueing `notifications.send` to send SMS), the cache (cache poisoning of roles and tags), and the pub/sub channel (forged in-app notifications to any user). Redis itself has no password.

### SEC-12 — Plaintext credentials at rest and in source — **High** [Static]

- Provider secrets, camera passwords and device keys are stored unencrypted in the database.
- `alembic.ini:57` and `app/db/session.py:5` contain a default DB URL with a password.
- RTSP URLs with embedded credentials are passed on the `ffmpeg` command line (`camera/manager/streaming.py:78-81`), visible in the process list.

### SEC-13 — No brute-force protection or password policy — **Medium** [Static]

There is no rate limiting or lockout on `/auth/token`, no password length or complexity rules, and no MFA. The planned OTP flow is not implemented.

### SEC-14 — Vulnerable dependencies — **Medium→High** [Verified]

`pip-audit` reports 41 advisories across 13 pinned packages, including Starlette (7), python-multipart (6), cryptography (7) and urllib3 (6). See [§16](#16-dependency-audit). Severity depends on the specific advisories; the web-facing Starlette and python-multipart advisories matter most.

### SEC-15 — Insecure MQTT TLS option — **Medium** [Static]

When `MQTT_USE_TLS=true`, certificate verification is disabled (`check_hostname=False`, `CERT_NONE`, `iot/mqtt/client.py:18-23`), allowing man-in-the-middle attacks.

### SEC-16 — Sensitive data in logs — **Medium** [Static]

- loguru console sink uses `diagnose=True` (`core/logging_config.py:58-59`), which prints **local variable values** (including passwords and tokens) in exception tracebacks.
- 259 `print()` calls dump request payloads, user objects and URLs.
- `logger.info(f"Updating user id={user_id} with data={updates.dict()}")` logs **plaintext new passwords** (`api/v1/user.py:82`).

### SEC-17 — Container hardening — **Medium** [Static]

- No `.dockerignore`, so `COPY . .` (`Dockerfile:58`) bakes `.env`, `.git`, `logs/` and `media/` into the image.
- The container runs as root, and compose bind-mounts the source with `--reload` in what is otherwise configured as an always-restart service.

### SEC-18 — Information disclosure via errors and health — **Low** [Verified]

- `/health` publicly returns raw backend error strings (hostnames, driver errors).
- Media endpoints return exception text in `detail`.
- There is no global exception handler or error schema, so failure modes are inconsistent: generic 500s, raw `ValueError`/`PermissionError` 500s, or detailed messages, depending on the route.

### SEC-19 — Media filename handling — **Low** [Static]

Upload filenames are only space-replaced (`media.py:338`). Traversal is currently blunted by the timestamp/UUID prefix, and delete by Starlette's `{filename}` not matching `/`. Neither is a deliberate control. Use `secure_filename`-style sanitization and an allow-list.

### SEC-20 — CSRF / session management — **Low** [Static]

Bearer tokens (no cookies) make classic CSRF largely inapplicable. There is no server-side session store, so sessions can't be invalidated.

### Security summary

| Severity | Count | IDs |
|---|---|---|
| Critical | 5 | SEC-01, 02, 03, 04, 05 |
| High | 7 | SEC-06, 07, 08, 09, 10, 11, 12 |
| Medium | 5 | SEC-13, 14, 15, 16, 17 |
| Low | 3 | SEC-18, 19, 20 |

---

## 13. DevOps Audit

### 13.1 Docker

- **Dockerfile:** `python:3.10-slim` (Python 3.10 EOL Oct 2026). 36 of 60 lines are commented-out earlier versions. No multi-stage build, no non-root user, no `HEALTHCHECK`, no `.dockerignore`. `ffmpeg` is missing although HLS needs it.
- **Compose** (`docker-compose.yml`):
  - `version: '3.9'` is obsolete.
  - `redis/redis-stack-server:latest` is unpinned; `redisinsight:1.14.0` is a legacy, unmaintained line.
  - `db` healthcheck hardcodes `-U greeni` while the user comes from `${POSTGRES_USER}` (line 13).
  - `depends_on` without `condition: service_healthy`; the backend doesn't wait for Redis or Mosquitto, and Mosquitto isn't in its `depends_on`.
  - The backend runs `alembic upgrade head` (a no-op, see §10.2) then `uvicorn --reload` with a source bind mount (`.:/app`), a **development** configuration.
  - `frontend` builds from `../nginx` (outside the repo).
  - Ports published publicly: 8000 (API), 8001 (RedisInsight), 1883/9001 (MQTT), 80 (nginx). Postgres and Redis are correctly bound to 127.0.0.1.
  - `mosquitto_config` volume is declared but unused.

### 13.2 Deployment process

There is none documented. There's no environment separation (dev/staging/prod), no image registry, no tagging or versioning, and no infrastructure-as-code. **[Inferred]** Deployment is `docker compose up` on a single host.

### 13.3 CI/CD

**None.** There is no `.github/workflows`, no lint, test or build pipeline, and no dependency scanning.

### 13.4 Environment management

- There is no `.env.example`. The required variables have to be reverse-engineered from code: `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`, `DATABASE_URL`, `SECRET_KEY`, `CELERY_BROKER_URL`, `CELERY_RESULT_BACKEND`, `MQTT_BROKER_HOST`, `MQTT_BROKER_PORT`, `MQTT_CLIENT_ID`, `MQTT_USERNAME`, `MQTT_PASSWORD`, `MQTT_USE_TLS`, `SERVER_HOST`, `TELEGRAM_ENABLED`, `TELEGRAM_SUPERUSER_ID`.
- Configuration is a hand-rolled `Settings` class with `load_dotenv()` called in three modules. There is no validation or typing, and silent defaults hide misconfiguration.
- `REDIS_URL` is derived from `CELERY_BROKER_URL` (`config.py:19`), while the cache ignores both (hardcoded host).

### 13.5 Logging

- loguru to stdout plus a rotating JSON file `logs/app.json` (10 MB, 5 days).
- A second loguru configuration in the MQTT client is wiped by the first, so `mqtt_logs.json` stays empty [Verified].
- 259 unstructured `print()` calls.
- The request ID is bound via `contextvars`, but it is not returned in responses and not propagated to Celery.
- No centralized log shipping.

### 13.6 Monitoring

- `/health` checks DB, Redis, MQTT and Celery, but **always returns HTTP 200**, even when `"status":"degraded"` [Verified]. The Docker healthcheck (`curl --fail`) therefore never fails.
- There are no metrics, no tracing, and no alerting.

### 13.7 Backups and recovery

- **No backup strategy:** no `pg_dump` job, no volume snapshots.
- Uploaded media lives in `./media` inside the bind-mounted source directory, mixed with code.
- Without tracked migrations, **schema recovery is impossible**.
- No disaster-recovery documentation.

### 13.8 Operational risks (ranked)

1. Not reproducible: no migrations, external build context, no `.env` template.
2. Public exposure of unauthenticated services: MQTT, RedisInsight, the API.
3. Single process with no horizontal scaling path.
4. Healthchecks that can't fail.
5. No backups.
6. Dev-mode server in deployment.

---

## 14. Testing Audit

| Area | Status |
|---|---|
| Unit tests | **None** |
| Integration tests | **None** |
| End-to-end tests | **None** |
| Test infrastructure (pytest config, fixtures, factories, test DB) | **None** |
| Coverage | **0%** |

**Testability problems that must be fixed first:**

- `import app.main` raises `RuntimeError: no running event loop` outside uvicorn [Verified]. A standard `from app.main import app` in a pytest module fails at collection.
- Hardcoded hosts (`greeni_redis`, `localhost` Celery broker).
- Import-time network clients (MQTT, Redis).
- Seed data runs on startup.

**Highest-value missing tests:**

1. **Authorization matrix:** every route × {anonymous, user, superuser}. This is the regression net for SEC-01/SEC-04.
2. **Token scope and expiry:** refresh vs access tokens, inactive users.
3. **Device lifecycle:** register → pending → confirm/reject → status; delete with dependants.
4. **Notification rendering per channel and template.**
5. **MQTT publish/handle contract**, using a broker test container.
6. **Pydantic schema round-trips** for update DTOs (would have caught the required-`Optional` bugs).
7. **Route-ordering smoke tests** (would have caught shadowing).
8. **Migration upgrade/downgrade on an empty DB.**

**Estimated confidence in the codebase: very low (≈10–15%).** Basic CRUD works by inspection and in my probes, but there is no regression safety, and I found more than 25 functional defects by reading alone.

---

## 15. Documentation Audit

| Document | Status | Gap |
|---|---|---|
| README | **Missing** | Purpose, quick start, architecture, ownership |
| Setup guide | **Missing** | Prerequisites, `.env` variables, how to create the DB (impossible without migrations), how to obtain `../nginx` |
| API docs | Auto-generated Swagger only | Misleading security annotations; no WS or MQTT docs; mixed languages; no error catalogue |
| Architecture docs | **Missing** | Component diagram, data flows, device protocol, edge vs server responsibilities |
| Deployment docs | **Missing** | Environments, secrets, ports, TLS, backups, scaling |
| Device integration guide | **Missing** | Registration handshake, device key, topic scheme, payload schemas, command lifecycle |
| Security model | **Missing** | Roles, permissions, intended public endpoints |
| Code comments | Present, mostly Persian | Inaccessible to non-Persian-speaking teams; often describe obsolete code |
| `alembic/README` | Empty | — |
| `iot/protocols/compare.txt` | A protocol comparison table | Informational only |

---

## 16. Dependency Audit

### 16.1 Major dependencies

| Package | Version | Purpose | Risk | Notes |
|---|---|---|---|---|
| fastapi / starlette | 0.116.1 / 0.47.1 | Web framework | **High** | 7 Starlette advisories; fixes in ≥ 0.47.2 … 1.3.1 (may require a FastAPI upgrade) |
| python-multipart | 0.0.20 | Form/file uploads | **High** | 6 advisories; fix ≥ 0.0.31; used by `/media/upload` and login |
| uvicorn | 0.35.0 | ASGI server | Low | — |
| pydantic | 2.11.7 | Validation | Low | v1-style usage |
| SQLAlchemy | 2.0.41 | ORM | Low | Legacy Query API |
| alembic | 1.16.4 | Migrations | Low | Unused in practice |
| psycopg2-binary | 2.9.10 | PG driver | Medium | `-binary` is not recommended for production |
| celery / kombu | 5.5.3 / 5.5.4 | Task queue | Low | — |
| redis | 6.2.0 | Redis client | Low | Requires the RedisJSON server module for cache |
| python-jose | 3.5.0 | JWT | **High** | 1 advisory (CVE-2026-85394, no fix version reported); low maintenance activity, consider PyJWT |
| ecdsa | 0.19.1 | (jose dependency) | Medium | 2 advisories |
| passlib / bcrypt | 1.7.4 / 3.2.2 | Password hashing | Medium | passlib is unmaintained; `requirements.lock.txt` pins bcrypt 4.3.0, which conflicts |
| cryptography | 45.0.5 | Crypto | **High** | 7 advisories; fix ≥ 46.0.7 (some ≥ 50.0.0) |
| gmqtt | 0.7.0 | MQTT client | Medium | Small project; consider `aiomqtt` |
| opencv-python | 4.10.0.84 | Camera frames | Medium | Heavy (~60 MB + libGL); the headless variant fits servers better |
| onvif-zeep / zeep | 0.2.12 / 4.3.1 | ONVIF PTZ | Medium | onvif-zeep is old and sync; 1 zeep advisory |
| requests / urllib3 | 2.32.5 / 2.5.0 | HTTP to SMS APIs | **High** | 1 + 6 advisories; no timeouts in code |
| psutil | 7.0.0 | System info | Low | — |
| loguru | 0.7.3 | Logging | Low | — |
| python-dotenv | 1.1.1 | Env loading | Low | 1 advisory |
| anyio, idna, mako, pyasn1 | various | Transitive | Medium | 2, 1, 1, 5 advisories |

### 16.2 Vulnerabilities (pip-audit, 2026-10-07)

13 packages with 41 unique advisories:

| Package | Advisories |
|---|---|
| cryptography | 7 |
| starlette | 7 |
| python-multipart | 6 |
| urllib3 | 6 |
| pyasn1 | 5 |
| anyio | 2 |
| ecdsa | 2 |
| idna | 1 |
| mako | 1 |
| python-dotenv | 1 |
| python-jose | 1 |
| requests | 1 |
| zeep | 1 |

I did not individually assess each advisory's exploitability against this codebase. Treat this as a scan result, not a triage.

### 16.3 Outdated and conflicting manifests

- **Three requirement files disagree.** `requirements.txt` has bcrypt 3.2.2; `requirements.lock.txt` and `requirementsback.txt` have bcrypt 4.3.0. The "lock" file is **missing** gmqtt, loguru, psutil, opencv, zeep and others, so it doesn't describe the real environment.
- No hash pinning and no separation of dev and prod dependencies.

### 16.4 Unused or suspicious packages

- `smsir-python` is unused (raw HTTP is used instead).
- `PyYAML` and `python-dateutil` are not used directly.
- Nothing appears malicious. All packages are well-known PyPI projects.

---

## 17. Business Logic Review

### 17.1 Critical workflows and their defects

**Device onboarding:**

1. **Identity is inconsistent.**
   - The router rejects a duplicate `unique_code` (409), but `register_device` deduplicates by `mac_address` (`crud/device.py:103`), which is neither unique nor required.
   - A device that re-registers with the same code always gets 409, so it can never refresh its IP through `/register`.
   - A device with a new code but a known MAC silently receives the *old* device record.
2. **Notifications reach nobody by default.**
   - Recipients come from users holding the permission names `device.pending_notification` or `full.permission` (`core/device_permissions.py:4`, `core/level_permissions.py:4`).
   - The seed creates a permission with `name="تمام دسترسی‌ها"` (Persian, "all access") and `title="full_access"` (`db/init_data.py:15`), which matches neither name.
   - Registration ran and queued **zero** notifications despite a superuser existing [Verified].
3. **Duplicate notification fan-out.** The router and the BLL both call `notify_users` (`api/v1/device.py:109`, `bll/device_bll.py:13`), so each registration would send every SMS twice.
4. **Missing templates.**
   - Email and in-app payloads carry no `template` (`bll/notification_bll.py:39-52`), so `render_template(None, …)` raises `ValueError` in the worker (`notifications/templates.py:35-37`).
   - SMS has a template only when the active provider is `smsir`; Melipayamak SMS is never sent.
5. **Approval is bypassable.** Any user can PATCH `is_pending`/`is_active` (SEC-04).
6. **Two reject endpoints disagree.** `/pending/{id}/reject` treats "pending" as `not is_active` (`device.py:307`); the other uses `is_pending`.

**Command execution:**

7. Commands are addressed by **DB integer id** on topic `/commands` (`iot/mqtt/publisher.py:12`), but media commands use **`unique_code`** on topic `/command` (`api/v1/media.py:268`, `ws/media_ws.py:66`), and `iot/mqtt/topics.py` defines yet another helper. A device can't know its DB id unless the server tells it, and nothing does.
8. Status is never updated: no subscription, nonexistent CRUD method, and the `executed_at` field is never set.
9. A publish while MQTT is disconnected is dropped, yet the API returns 200 with `status: pending` [Verified]. There is no outbox or retry.

**Media distribution:**

10. `is_multimedia` cannot be set through any API: it isn't in `DeviceCreate` or `DeviceUpdate` (`schemas/device.py:7-36`). `get_all_active_multimedia_device_codes` therefore always returns empty unless the DB is edited by hand, and uploads reach **no devices**.
11. `GET /devices/multimedia` declares `List[DeviceMultiMediaPublic]` but returns `List[str]`, a 500 as soon as any multimedia device exists. **[Static]**
12. `DELETE /media/delete/{filename}` and `/media/delete-all` remove media but keep the `.hash` side files (`delete_file` computes `hash_path` but never removes it, `media.py:425-437`; `delete-all` explicitly skips `.hash` files, `media.py:471`). Later uploads of the same content are reported as duplicates pointing to **deleted** files, and no download command is sent (`media.py:315-330`). **[Static]**

**Cameras and PTZ:**

13. Streaming never connects (`is_connected` missing) [Verified].
14. Even after a fix:
    - MJPEG would fail because `Response` can't take an async generator (`api/v1/camera.py:291-294`);
    - HLS returns a server filesystem path (`/tmp/streams/.../playlist.m3u8`) that no route serves (`streaming.py:62-112`).
15. PTZ:
    - `save_preset` returns a token but never stores it, while `GET /presets` reads stored rows (`api/v1/ptz.py:176-194, 216-227`);
    - the IP is parsed from the URL naively (`ptz.py:29`);
    - the default ONVIF port `camera.port or 80` reuses the RTSP port field (554 default);
    - the `home` direction passes schema validation but `continuous_move` rejects it.
16. Approving a camera sets `status=ONLINE` without any connectivity check (`camera/crud/camera.py:128`).

**Notifications:**

17. Push: `PushService(self.db)` passes a Session as `api_key`, and `send_push` only prints (`notifications/manager.py:31`, `notifications/push.py`).
18. Email: `SMTPEmailService.send` only prints (`email/smtp_service.py:9-11`).
19. Melipayamak: `send(self, message, mobile)` is called as `send(to, message)`, so the phone number and the message are swapped (`sms/melipayamak.py:5`, `sms/manager.py:31`). It also maps `username=api_key, password=sender`.
20. sms.ir: a hardcoded `templateId: 793797` with the rendered body passed as parameter `Code` (`sms/smsir.py:16-17`). Every SMS uses the same OTP-style template.
21. Status-change notifications also push directly from a thread with `asyncio.run` (cross-loop; see §5.4).
22. Creating an active email provider first deactivates the current one and commits, then inserts. A duplicate name returns 500 **and leaves no active provider** [Verified] (`api/v1/email_provider.py:17-23`).

**Users:**

23. Non-superusers assigning `role_id=1` get a 500 instead of 403 [Verified].
24. `UserBasicUpdate` ("basic info") also accepts `role_id`.
25. Users can't change their own password safely: there is no "current password" check, and the endpoint doesn't distinguish self from others.

### 17.2 Edge cases not handled

- Concurrent registration of the same device: a race between the check and the insert produces a 500 on the unique violation.
- Sensor values out of range or of the wrong type (`value` is a free string; `SensorAttribute.type` is never enforced).
- Device offline detection: there is no heartbeat timeout, so `is_online` is only what the device last reported.
- Large uploads, since files are read fully into memory for hashing and writing.
- Deleting entities with dependants (devices with commands, users with notifications; see §10.5).
- An expired access token makes refresh impossible.
- `psutil.cpu_freq()` returning `None` in containers crashes `/ws/system-info`.

---

## 18. Technical Debt

Impact scale: **H** blocks production or causes incidents · **M** slows the team or causes bugs · **L** hygiene.

### Architecture debt

| Item | Impact |
|---|---|
| Edge/server concerns mixed (protocol drivers, executors with `sudo reboot`) | M |
| Process-local state (camera, PTZ, streams, WS registries) and fixed MQTT client ID prevent scaling | H |
| Sync DB and IO in async handlers | H |
| Layering violations (`crud`/`bll` → `api`) | M |
| Duplicate infrastructure (2 Celery apps, 3 Redis listeners, 2 WS notification endpoints, 3 `get_db`) | M |
| No device messaging contract (topics, identifiers, payload schemas, acknowledgements) | H |
| Overlapping telemetry tables (4) | M |
| No tenancy or ownership model | M (H if SaaS is intended) |

### Code debt

| Item | Impact |
|---|---|
| 14 empty modules, `device copy.py`, ~290 lines of commented-out code | L |
| 259 `print()` calls | M |
| Broken imports and dead modules (`protocol_manager`, `actuator`, `ptz.py`, `device_data`) | M |
| Duplicate schema and model definitions | M |
| Pydantic v1/v2 and SQLAlchemy 1.x/2.x idiom mix; deprecated APIs (`on_event`, `utcnow`, `Query.get`, `declarative_base`) | L |
| Mixed-language messages without i18n | M |
| Route shadowing, signature mismatches, wrong response models | H |

### Documentation debt

| Item | Impact |
|---|---|
| No README, setup, architecture, deployment, device-integration or security docs | H |
| Misleading OpenAPI security annotations | M |
| Comments mostly in Persian; non-Persian-speaking teams can't use them | M |

### Testing debt

| Item | Impact |
|---|---|
| Zero tests, no harness, untestable import side effects | H |

### DevOps debt

| Item | Impact |
|---|---|
| Migrations not versioned | H |
| External build context (`../nginx`) | H |
| No CI/CD, no image versioning, no environment separation | H |
| Dev server configuration (`--reload`, bind mount) in compose | M |
| Healthchecks that can't fail; no metrics or alerting | M |
| No backups or DR | H |
| Conflicting requirement manifests | L |

### Security debt

| Item | Impact |
|---|---|
| SEC-01 to SEC-05 (see §12) | H |
| Secrets in DB, responses and source | H |
| Vulnerable dependencies | M |
| No rate limiting, audit log or password policy | M |

**Rough remediation estimate** (one experienced backend engineer, excluding new features):

- critical security and reproducibility: about 2–3 weeks;
- functional bug fixes plus a test harness and authorization tests: about 3–5 weeks;
- architecture cleanup (async strategy, messaging contract, telemetry, camera service): about 6–10 weeks.

**[Inferred]** These are order-of-magnitude figures only.

---

## 19. Missing Features

| Missing feature | Evidence that it is planned or expected | Why it matters |
|---|---|---|
| **Database migrations in VCS** | Alembic configured, `alembic upgrade head` in compose; versions ignored | Without it, nothing can be deployed |
| **Permission-based authorization** | `Permission`/`RolePermission` models and CRUD; permission constants in `core/*_permissions.py` | Core security requirement |
| **Resource ownership / multi-site tenancy** | Implied by "management" of multiple greenhouses; absent | Users see and modify everything |
| **Telemetry ingestion over MQTT** | `iot/device_data/receiver.py`, `models/device_data.py`, `topics.get_data_topic` | The core value of a monitoring product |
| **Command acknowledgement / status loop** | `handler.py:37-51`, `Command.status`/`executed_at`, unmounted `ws/command_ws.py` | Operators need to know whether a pump actually turned on |
| **Device heartbeat / offline detection** | Commented `/devices/ping` and `last_ping_at` (`device.py:312-326`); `Camera.last_heartbeat` | Accurate online status |
| **Alerting / threshold rules** | `SensorAttributeValue.status` (normal/warning/critical), `device_alert` templates | Proactive notifications |
| **Scheduling / automation** | Empty `iot/commands/executors/scheduler.py` | Irrigation and lighting schedules are standard greenhouse features |
| **Email sending** | `EmailManager`, provider model and CRUD; `SMTPEmailService` prints | Channel advertised but fake |
| **Push notifications** | `PushProvider` model and CRUD; `# TODO: FCM or OneSignal` (`push.py:10`) | Channel advertised but fake |
| **Telegram notifications** | `TELEGRAM_ENABLED`, `TELEGRAM_SUPERUSER_ID` (`config.py:9-10`) | — |
| **OTP / passwordless login** | `otp_sms` template; sms.ir verify template | Typical for Iranian phone-based auth |
| **Token revocation / logout** | Comment in `auth.py:87-88` | Session security |
| **Working camera streaming** | Endpoints exist; implementation broken; empty `ip_camera.py`, `usb_camera.py`, `ffmpeg_handler.py`, `camera_tasks.py` | — |
| **Camera recording and retention** | `recording_enabled`, `retention_days` fields | — |
| **PTZ presets persistence** | `PTZControl.preset_name`; `get_presets` | — |
| **Sensor value history** | `SensorAttributeValueHistory` model (never written) | Trend charts |
| **SPI/UART/PWM protocols** | Empty files imported by `protocol_manager` | Edge-side, if kept |
| **Device startup registration** | Empty `iot/startup/device_registration.py` | Edge-side |
| **Config-schema validation** | `SensorType.config_schema`; `validate_sensor_protocol_config` always `True` (`crud/sensor.py:624-639`) | — |
| **Setting `is_multimedia` via API** | Used in filters, absent from schemas | Media distribution can't work |
| **Notification log / outbox** | `models/notification.py`, `crud/notification.py` (orphaned table) | Delivery tracking and retries |
| **Pagination metadata, filtering and sorting** conventions | Ad hoc `skip`/`limit` on some lists | Frontend usability at scale |
| **Audit trail** | `created_by`/`updated_by` on some tables only | Accountability for actuator actions |
| **Rate limiting** | None | Abuse protection (SMS cost) |

---

## 20. Over-Engineering Review

| Item | Why it is premature or over-built |
|---|---|
| **Hardware protocol framework** (`iot/protocols/*`, ~1,300 lines: abstract base, factory, manager, 8 protocols incl. LoRa and Zigbee cluster enums) | Every driver is a **simulation that returns hardcoded values** (for example I2C `value = 3.14`, `i2c.py:71`; Modbus `[0x1234]`, `modbus.py:64`). Three protocols are empty files that break the manager's imports [Verified]. **Nothing in the server calls it.** It belongs in an edge agent, if anywhere. |
| **RedisJSON cache layer** (`cache/base_cache.py`, role/tag services) | It caches two tiny tables, adds a hard dependency on Redis Stack, needs workarounds (numeric-key dict→list conversion for RedisInsight display), rebuilds ORM objects from JSON, and introduces staleness. A Postgres primary-key lookup is cheaper than the complexity. |
| **Pluggable provider registry** for SMS, email and push with DB-stored "active provider" and full CRUD APIs | Two of three channels are print stubs. One real SMS integration plus env-configured credentials would have been enough for this stage. |
| **Three parallel real-time notification paths** (two WS endpoints, three Redis listeners, plus direct cross-loop pushes) | The Redis pub/sub fan-out is the right idea, but it is implemented three times. |
| **Thin BLL wrappers** (`bll/device_bll.py`) | They mostly add `print("start notify users")` around CRUD calls. They aren't harmful, but they add a layer without logic, and routers bypass them anyway. |
| **In-process video stack** (OpenCV capture, ffmpeg HLS, ONVIF PTZ in the API process) | Ambitious scope with nothing working. A dedicated media server (MediaMTX, go2rtc) would be simpler and more robust than building one inside a CRUD API. |
| **`SensorType.config_schema` / JSON-schema per protocol** | The validation always returns `True`. |
| **Device model `to_dict` / `to_dict_simple` methods** | Duplicate what Pydantic response models already do. |

By contrast, the project is **under-engineered** in the areas that matter most: authentication and authorization, migrations, tests, configuration, and observability.

---

## 21. Recommendations

Each recommendation is rated for **Impact** (H/M/L), **Effort** (S ≤ 2 days, M ≤ 2 weeks, L > 2 weeks) and **Risk** of the change itself (H/M/L).

### 21.1 Immediate fixes (this week)

| # | Recommendation | Impact | Effort | Risk |
|---|---|---|---|---|
| I-1 | Restrict network exposure of any running deployment: firewall ports 8000, 8001, 1883 and 9001 to trusted networks; remove RedisInsight from compose | H | S | L |
| I-2 | **Rotate all secrets** that may have been exposed: SMS API keys, SMTP password, push keys, camera passwords, device `local_key`s, DB password, JWT secret. **Change the seeded admin password** | H | S | L |
| I-3 | Make `SECRET_KEY` mandatory (fail fast); remove the `"secret"` default | H | S | L |
| I-4 | Remove hardcoded admin and personal data from `init_data.py`; bootstrap the first admin via CLI or env with a forced password change | H | S | L |
| I-5 | Apply authentication to **all** routers by default; add `require_superuser` to admin operations (users, roles, permissions, providers, device approval and update, camera management, media delete); add ownership checks to in-app notifications (`user_id` must equal the current user) | H | M | M |
| I-6 | Fix tokens: enforce `scope == "access_token"` and `is_active` in `get_current_user`; make `/auth/refresh` not require an access token; reduce access TTL to 15–30 minutes | H | S | L |
| I-7 | Remove secrets from response schemas (`api_key`, `password`, `local_key`, camera `password`); make them write-only | H | S | L |
| I-8 | Lock down Mosquitto: `allow_anonymous false`, password file or mTLS, per-device ACLs; don't publish ports publicly | H | M | M |
| I-9 | **Commit Alembic migrations**: remove `alembic/versions/` from `.gitignore`, autogenerate a baseline from the models, and fix `Notification` to use the shared `Base` | H | S | M |
| I-10 | Restrict CORS to known frontend origins | M | S | L |
| I-11 | Media: allow-list extensions and content types, cap size, serve with `Content-Disposition: attachment` and `nosniff` | H | S | L |
| I-12 | Remove `/tasks/add-task`; stop logging request bodies with passwords (`user.py:82`); set loguru `diagnose=False` | M | S | L |
| I-13 | Upgrade vulnerable dependencies (Starlette/FastAPI, python-multipart, cryptography, urllib3, requests, pyasn1, …) and re-run `pip-audit` | H | M | M |

### 21.2 Short-term improvements (2–6 weeks)

| # | Recommendation | Impact | Effort | Risk |
|---|---|---|---|---|
| S-1 | Add a test harness (pytest, `TestClient`/`httpx`, Postgres test container) and an **authorization matrix test** covering every route | H | M | L |
| S-2 | CI pipeline: ruff (lint and format), mypy (gradual), pytest, pip-audit, Docker build | H | S | L |
| S-3 | Remove import-time side effects: move the Redis listener and MQTT start into a FastAPI `lifespan`; delete the duplicate listeners and WS endpoint; consolidate to one Celery app; use `pydantic-settings` for config (no hardcoded hosts) | H | M | M |
| S-4 | Fix the verified functional bugs: route ordering (`/devices/{id}`, `/sensors/values`); assign/unassign tags signature; partial-update schemas (`= None` defaults); `BaseAPIResponse.message`; `/devices/multimedia` response model; device and user delete cascades; non-atomic provider activation | H | M | L |
| S-5 | Repair the notification pipeline: one `notify_users` call; templates for every channel; seed permission names that match constants (or match by `title`); fix the Melipayamak argument order; configurable sms.ir template ID; real SMTP (`smtplib`/`aiosmtplib`) or remove email; remove push until implemented; drop the `asyncio.run` cross-loop push | H | M | M |
| S-6 | Health endpoint: return 503 when degraded; split liveness and readiness | M | S | L |
| S-7 | Replace `print()` with structured logging; propagate the request ID to responses and Celery | M | M | L |
| S-8 | Delete dead code and files (Appendix C) or move them to an `edge/` prototype folder outside the server package | M | S | L |
| S-9 | Write a README, `.env.example`, architecture overview, device integration guide (registration, keys, topics, payloads) and security model. Translate key comments and error messages to English, or introduce i18n | H | M | L |
| S-10 | Production compose profile: no bind mount, no `--reload`, gunicorn with uvicorn workers or multiple uvicorn workers, non-root image, `.dockerignore`, pinned image tags, `depends_on: service_healthy`, ffmpeg if HLS is kept | H | M | M |
| S-11 | Add rate limiting on `/auth/token` and notification sending; a password policy | M | S | L |
| S-12 | Postgres backups (scheduled `pg_dump` or WAL archiving) and media volume backups; a documented restore drill | H | S | L |

### 21.3 Medium-term improvements (1–3 months)

| # | Recommendation | Impact | Effort | Risk |
|---|---|---|---|---|
| M-1 | Choose one async strategy. Either make DB-bound handlers plain `def` (threadpool), or adopt SQLAlchemy async (`asyncpg`) consistently. Offload OpenCV and ONVIF calls to threads or workers | H | M | M |
| M-2 | **Define the device↔server contract**: one device identifier (`unique_code` or UUID), a topic hierarchy (`greeni/{device}/telemetry`, `/cmd`, `/cmd/ack`, `/status`), versioned JSON schemas, QoS 1 for commands, LWT for presence, and server-side subscription handlers | H | M | M |
| M-3 | **Telemetry pipeline**: MQTT → validation → a single time-series table (consider TimescaleDB) with proper indexes and retention; consolidate the 4 overlapping tables; a "latest value" view | H | L | M |
| M-4 | **Command lifecycle**: outbox table and retry, acknowledgement handling, timeouts, `executed_at`; real-time status to the UI via the existing WS fan-out | H | M | M |
| M-5 | **Real RBAC**: a permission-check dependency (`require_permission("device.update")`) driven by the existing `Permission` tables; audit log of privileged actions | H | M | M |
| M-6 | **Camera subsystem redesign**: delegate ingest and transcode to a media server (MediaMTX or go2rtc); the API only stores configuration and issues signed playback URLs; PTZ via async ONVIF with presets persisted | M | L | M |
| M-7 | Encrypt provider and camera credentials at rest (KMS or Fernet key from env), or move them to a secrets manager | M | M | M |
| M-8 | Data-model hygiene: indexes and constraints from §10.3–10.4; enums; timezone-aware timestamps everywhere; consistent soft-delete policy; explicit `ON DELETE` | M | M | M |
| M-9 | Observability: Prometheus metrics (HTTP, Celery, MQTT), OpenTelemetry tracing, centralized logs, alerting on health | M | M | L |
| M-10 | Remove the RedisJSON dependency (drop the cache, or use plain `GET`/`SET` with JSON strings); stop using `KEYS` | L | S | L |

### 21.4 Long-term improvements (3–12 months)

| # | Recommendation | Impact | Effort | Risk |
|---|---|---|---|---|
| L-1 | Split the **edge agent** (protocol drivers, executors, local scheduling, media playback) into its own repo with a proper provisioning flow (claim tokens or device certificates, OTA updates) | H | L | M |
| L-2 | Introduce **tenancy** (organization → site/greenhouse → devices) and resource ownership if the product serves more than one customer | H | L | H |
| L-3 | Rules and automation engine (threshold alerts, schedules, scenes) built on the telemetry and command contracts | H | L | M |
| L-4 | Horizontal scalability: stateless API replicas, unique MQTT client IDs or shared subscriptions, externalized WS fan-out (already Redis-based) | M | M | M |
| L-5 | Formal i18n for user-facing messages (Persian and English) | M | M | L |
| L-6 | Security program: threat model for the OT/IoT side, periodic pentest, SBOM and dependency update automation (Renovate/Dependabot) | H | M | L |

---

## 22. Final Scorecard

| Dimension | Score (1–10) | Explanation |
|---|---|---|
| **Architecture** | **4** | A sensible high-level decomposition (API/CRUD/schemas/models; domain packages for devices, sensors, cameras, notifications, IoT) and appropriate infrastructure (Postgres, Redis, Celery, MQTT), with a correct Redis fan-out idea for real-time events. Undermined by layering violations, triplicated infrastructure, process-local state, sync I/O in async handlers, edge code in the server, and no device messaging contract. |
| **Code Quality** | **3** | Readable small functions and consistent CRUD patterns in places. But 259 `print()`s, ~290 lines of commented-out code, 14 empty modules, a duplicate CRUD file, duplicate classes, broken imports, route shadowing, signature mismatches, mixed Pydantic and SQLAlchemy idioms, and mixed-language messages. |
| **Security** | **1** | Unauthenticated actuator control and credential disclosure, account takeover by any user, a default JWT key, a seeded `1234` superuser, an anonymous MQTT broker, exposed RedisInsight, stored XSS, permissive CORS, and 41 dependency advisories. Most of these I reproduced. |
| **Testing** | **1** | No tests of any kind and no test infrastructure. The app can't even be imported by a standard test runner. |
| **Documentation** | **2** | No README or any written docs. The only documentation is auto-generated Swagger (with misleading security markers) and inline comments, mostly in Persian. |
| **Maintainability** | **3** | The module layout makes things findable, but hidden import-time side effects, duplicate implementations, dead code and the lack of tests make changes risky. Untracked migrations mean the schema lives only in developers' databases. |
| **Scalability** | **2** | A single dev-mode process. Blocking calls on the event loop, in-memory camera, PTZ and stream state, a fixed MQTT client ID, unbounded list queries, and JSON-array write amplification on login. Only the Redis-based notification fan-out would survive multiple replicas. |
| **Production Readiness** | **1** | Can't be bootstrapped from the repo (no migrations, external build context, no env template). Critical security exposure, broken core features (video, telemetry, command feedback, notifications), healthchecks that can't fail, and no backups, CI or monitoring. |
| **Overall (unweighted mean)** | **2.1** | **Early prototype.** Keep the domain model and module structure; rebuild auth, migrations, the device messaging contract, and the camera subsystem before any production use. |

---

## Appendix A — Endpoint Inventory

Generated by introspecting the running app's FastAPI dependency graph. **JWT** = depends on `get_current_user`; **NONE** = no authentication; **query-token** = JWT read from the WebSocket query string. Admin checks inside handlers are noted in §11.

### A.1 Routes without authentication (81)

| Method | Path | Module |
|---|---|---|
| GET | `/health` | main |
| GET, POST | `/v1/api/commands/` | command |
| GET | `/v1/api/commands/{command_id}` | command |
| GET | `/v1/api/devices/by-unique-code/{code}` | device |
| POST | `/v1/api/devices/register` | device |
| GET | `/v1/api/devices/status/{uuid}` | device |
| GET | `/v1/api/devices/{unique_code}` | device |
| PATCH | `/v1/api/devices/{unique_code}/online`, `/offline` (X-Device-Key) | device |
| GET, POST | `/v1/api/email-providers/` | email_provider |
| GET | `/v1/api/email-providers/active` | email_provider |
| PATCH | `/v1/api/email-providers/{provider_id}/deactivate` | email_provider |
| POST | `/v1/api/inapp_provider/` | inapp |
| GET | `/v1/api/inapp_provider/{user_id}`, `/recent/{user_id}`, `/unread/count/{user_id}` | inapp |
| PATCH | `/v1/api/inapp_provider/mark-read/{notification_id}`, `/user/{user_id}/mark-all-read` | inapp |
| DELETE | `/v1/api/inapp_provider/{notification_id}`, `/user/{user_id}/all` | inapp |
| POST | `/v1/api/notifications/` | notifications |
| GET, POST | `/v1/api/push_provider/` | push_provider |
| GET | `/v1/api/push_provider/active` | push_provider |
| PUT | `/v1/api/push_provider/{provider_id}` | push_provider |
| POST | `/v1/api/push_provider/{provider_id}/deactivate` | push_provider |
| POST | `/v1/api/auth/token` | auth |
| GET, POST | `/v1/api/sensors/` | sensor |
| GET | `/v1/api/sensors/{sensor_id}` | sensor |
| PUT, DELETE | `/v1/api/sensors/{sensor_id}` | sensor |
| GET, POST | `/v1/api/sensors/attributes` | sensor |
| GET, PUT, DELETE | `/v1/api/sensors/attributes/{attribute_id}` | sensor |
| GET, POST | `/v1/api/sensors/types` | sensor |
| GET, PUT, DELETE | `/v1/api/sensors/types/{type_id}` | sensor |
| GET | `/v1/api/sensors/types/tags`, `/types/{type_id}/tags` | sensor |
| POST | `/v1/api/sensors/types/assign-tags`, `/types/unassign-tags` | sensor |
| POST | `/v1/api/sensors/attribute-values` | sensor |
| GET | `/v1/api/sensors/attribute-values/full`, `/attribute-values/{value_id}` | sensor |
| GET | `/v1/api/sensors/values` (unreachable — shadowed) | sensor |
| GET | `/v1/api/sensors/{sensor_id}/attribute-values`, `/{sensor_id}/attribute-values/full` | sensor |
| GET | `/v1/api/sensors/{sensor_id}/protocol-config` | sensor |
| POST | `/v1/api/sensors/{sensor_id}/initialize-protocol`, `/{sensor_id}/validate-protocol-config` | sensor |
| GET | `/v1/api/sensors/by-protocol/{protocol}`, `/protocol/{protocol}/active`, `/protocol-types/{protocol}`, `/protocols/available`, `/device/{device_id}/sensors` | sensor |
| POST | `/v1/api/sensors/data/` | sensor_data |
| GET | `/v1/api/sensors/data/{sensor_id}/latest`, `/{sensor_id}/history` | sensor_data |
| GET, POST | `/v1/api/sms-providers/` | sms_provider |
| GET | `/v1/api/sms-providers/active`, `/countinfo`, `/{provider_name}`, `/{provider_name}/status` | sms_provider |
| PATCH | `/v1/api/sms-providers/{provider_name}`, `/{provider_name}/set-active` | sms_provider |
| DELETE | `/v1/api/sms-providers/{provider_name}` | sms_provider |
| GET | `/v1/api/tags/`, `/v1/api/tags/{tag_id}` | tag |
| GET | `/v1/api/tasks/add-task` | task |
| WS | `/v1/api/ws/media` | ws.media_ws |
| WS | `/v1/api/ws/health`, `/v1/ws/health` | ws.system_health |
| WS | `/v1/api/ws/system-info` | ws.system_info |

Plus the public static mount `/media/*` and the docs routes (`/docs`, `/redoc`, `/openapi.json`).

### A.2 Routes requiring JWT (77 HTTP + 2 WS)

| Module | Endpoints |
|---|---|
| auth (3) | `POST /auth/refresh`, `GET /auth/me`, `POST /auth/logout` |
| users (8) | `POST /users/`, `GET /users/`, `GET/PUT/DELETE /users/{user_id}`, `PATCH /users/{user_id}/basic`, `POST /users/{user_id}/login`, `GET /users/by-permission/{permission_name}` |
| roles (5) | `POST/GET /roles/`, `PUT /roles/{role_id}`, `PATCH /roles/{role_id}/name`, `DELETE /roles/{role_id}` (Admin) |
| permissions (4) | `POST/GET /permissions/`, `PUT/DELETE /permissions/{permission_id}` (delete: Admin) |
| devices (18) | `GET /devices/full`, `/multimedia`, `/{device_id}/full`, `/pending` (Admin), `POST /devices/`, `GET /devices/`, `/half`, `GET /devices/{device_id}` (shadowed), `PATCH /{device_id}/confirm` (Admin), `DELETE /{device_id}/reject` (Admin), `PATCH /{device_id}/activate`, `/deactivate`, `/on_line`, `/off_line` (Admin), `PATCH/PUT/DELETE /devices/{device_id}`, `DELETE /devices/pending/{device_id}/reject` (Admin) |
| sensors (5) | `PATCH /sensors/{sensor_id}`, `/attributes/{attribute_id}`, `/types/{type_id}`, `/attribute-values/{value_id}`, `/{sensor_id}/protocol-config` |
| tags (3) | `POST /tags/`, `PATCH/DELETE /tags/{tag_id}` |
| notifications (2) | `GET /notifications/recent`, `/all` |
| media (4) | `POST /media/upload`, `DELETE /media/delete/{filename}`, `/delete-all`, `/delete-all-hash` |
| cameras (16) | `GET /cameras/pending` (Admin), `POST /cameras/pending/{id}/approve` (Admin), `GET /cameras/pending/count` (Admin), `POST /cameras/{id}/activate`, `/deactivate` (Admin), `GET/POST /cameras/`, `GET /cameras/device/{device_id}`, `GET/PUT/DELETE /cameras/{camera_id}`, `GET /cameras/{id}/frame`, `/stream/mjpeg`, `POST /cameras/{id}/stream/hls/start`, `/stream/hls/stop`, `/stream/stop` |
| ptz (9) | `POST /ptz/{camera_id}/initialize`, `/move`, `/absolute-move`, `/zoom`, `/home`, `/stop`, `/preset/save`, `/preset/load`, `GET /ptz/{camera_id}/presets` |
| WS (2, query-token) | `/v1/api/ws/notifications`, `/v1/api/ws/database-notifications` |

---

## Appendix B — Verification Log

All runs used a **scratch copy** of the repository. Nothing was written to the working tree, and no commits or pushes were made. Environment: Python 3.11 venv with `requirements.txt`, PostgreSQL 16, Redis 7 (without the RedisJSON module), no MQTT broker, no camera hardware.

| # | Check | Result |
|---|---|---|
| 1 | `pip install -r requirements.txt`; `pip check` | Installed; "No broken requirements found" |
| 2 | `pip-audit -r requirements.txt` | 41 unique advisories in 13 packages (76 rows including duplicates) |
| 3 | `import app.main` outside an event loop | `RuntimeError: no running event loop` at `app/api/v1/__init__.py:19` → `ws/notifications_ws.py:86` |
| 4 | `alembic upgrade head` on an empty DB | Exit 0; only the `alembic_version` table exists |
| 5 | `Base.metadata` contents | 21 tables; `notifications` absent (`Notification.metadata is Base.metadata` → `False`) |
| 6 | `Device.uuid` column default | Not callable; a constant UUID evaluated at import |
| 7 | `import app.camera.models.ptz` | `InvalidRequestError: Table 'ptz_controls' is already defined…` |
| 8 | `import app.iot.protocols.protocol_manager` | `ImportError: cannot import name 'SPIProtocol'` |
| 9 | `ActuatorCommandExecutor.execute()` | `ImportError: cannot import name 'client' from 'app.iot.mqtt.client'` |
| 10 | `uvicorn app.main:app` startup | Starts; two Redis listeners log startup; MQTT connect failure produces no error log |
| 11 | `GET /health` (Redis host unresolvable, no broker) | HTTP **200** with `"status":"degraded"` |
| 12 | Login with the seeded superuser (`1234`) | 200, tokens issued |
| 13 | Token forged with key `"secret"` → `GET /auth/me` | 200, superuser profile |
| 14 | Refresh token as bearer → `GET /auth/me` | 200 |
| 15 | `POST /auth/refresh` with only a refresh token | 401 "Not authenticated" |
| 16 | Unauthenticated `GET /sms-providers/` | 200, includes `api_key` |
| 17 | Unauthenticated `GET /email-providers/` | 200, includes `password` |
| 18 | Unauthenticated in-app create and read for user 1 | 200 / 200 |
| 19 | Unauthenticated `GET /devices/{code}` | 200, includes `local_key` |
| 20 | Unauthenticated `POST /commands/`, `GET /commands/` | 200 / 200; server logs "⚠ MQTT not connected yet" (dropped) |
| 21 | Unauthenticated `POST /notifications/` (SMS to an arbitrary number) | 200 `{"status":"queued"}` |
| 22 | `GET /devices/1` (int-id route) | 404 — handled by the `/{unique_code}` route |
| 23 | `GET /sensors/values` | 422 `int_parsing` for `sensor_id` |
| 24 | Regular user `PUT /users/1 {"password":…}` then login as superuser | 200 / 200 (**takeover**) |
| 25 | Regular user `PUT /users/1 {"role_id":1}` | 500 (`PermissionError` not mapped) |
| 26 | Regular user `PATCH /devices/{id}` `is_pending=false` | 200 (approval bypass) |
| 27 | Regular user `DELETE /devices/{id}` | 200 |
| 28 | `POST /sensors/types/assign-tags` | 500 `TypeError: … missing 1 required positional argument: 'tag_ids'` |
| 29 | Partial PATCH/PUT on sensor types, permissions, SMS provider | 422 "Field required" for omitted fields |
| 30 | `GET /cameras/{id}` | 200, includes `password` |
| 31 | `GET /cameras/{id}/frame`, `/stream/mjpeg` (approved USB camera) | 400; log: `'CameraManager' object has no attribute 'is_connected'` |
| 32 | Upload `.html` via `/media/upload`, then unauthenticated `GET /media/<file>` | 200 `content-type: text/html` with script body |
| 33 | CORS preflight from `https://evil.example` | `access-control-allow-origin: https://evil.example`, `allow-credentials: true` |
| 34 | `POST /email-providers/` with a duplicate name while one is active | 500; the existing provider is left `is_active=false` |
| 35 | Device registration with a superuser present | Log shows `start/done notify users` with **no** "Queued" lines (no recipients) |
| 36 | `mqtt_logs.json` after a run | Created, 0 bytes |
| 37 | Route introspection | 160 API/WS routes; 81 without authentication |

Throwaway services were stopped and their data removed after testing.

---

## Appendix C — Dead, Empty, and Duplicate Files

**Empty Python modules (14):**

- `app/tasks/camera_tasks.py`
- `app/iot/startup/device_registration.py`
- `app/iot/protocols/spi.py`, `uart.py`, `pwm.py`, and `rtsp.py ` (filename ends with a space)
- `app/iot/commands/dispatcher.py`, `schemas.py`, `utils.py`
- `app/iot/commands/executors/gpio.py`, `scheduler.py`
- `app/camera/manager/ip_camera.py`, `usb_camera.py`
- `app/camera/crud/ptz.py`
- `app/camera/utils/ffmpeg_handler.py`

**Unreferenced or broken modules:**

- `app/iot/protocols/protocol_manager.py` (import fails) and all protocol drivers
- `app/iot/commands/executors/actuator.py` (bad import), `system.py`, `base.py`
- `app/iot/device_data/receiver.py`, `app/crud/device_data.py` (bad import)
- `app/iot/mqtt/topics.py`
- `app/api/v1/ws/command_ws.py` (router never mounted)
- `app/camera/models/ptz.py` (conflicting duplicate), `app/camera/schemas/ptz.py` (duplicate)
- `app/crud/notification.py`, `app/models/notification.py` (orphaned table)
- `app/tasks/example.py` and `app/api/v1/task.py` (demo)

**Duplicate or backup files:**

- `app/crud/device copy.py` (an older variant that references nonexistent `User.push_token` and `SMSProvider.get_active`)
- `requirementsback.txt`, `requirements.lock.txt` (stale)
- `To` (empty file in the repo root)
- Commented-out prior versions inside `Dockerfile`, `app/api/v1/media.py`, `app/api/v1/ws/media_ws.py`, `app/api/v1/inapp.py`, `app/models/role.py`, `app/models/permission.py`, `app/models/role_permission.py`, `app/bll/device_bll.py`, `app/bll/notification_bll.py`, `app/tasks/notifications_tasks.py`, `app/schemas/common.py`, `app/iot/mqtt/client.py`

---

*End of report.*
