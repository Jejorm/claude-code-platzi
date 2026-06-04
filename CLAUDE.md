# PlatziFlixProject

Online video course platform (Netflix-style for Platzi courses). One FastAPI backend serves three independent clients: a Next.js web app, an iOS app, and an Android app.

## Repository Structure

```
ClaudeCode/
├── Backend/      # Python/FastAPI REST API
├── Frontend/     # Next.js 15 web app
└── Mobile/
    ├── PlatziFlixiOS/      # Swift/SwiftUI
    └── PlatziFlixAndroid/  # Kotlin/Jetpack Compose
```

---

## Backend

**Stack:** Python 3.11, FastAPI 0.104+, SQLAlchemy 2.0, PostgreSQL 15, Alembic, Docker + docker-compose, `uv` package manager

**Architecture:** 3-layer (Routes → Service → ORM). No repository layer. Service methods return raw Python dicts.

**Run:**
```bash
cd Backend
make start       # docker-compose up
make migrate     # alembic upgrade head
make seed        # load dev seed data
make seed-fresh  # drop + re-seed
```

**Domain entities (PostgreSQL tables):**

| Table | Key fields |
|---|---|
| `teachers` | id, name, email (unique) |
| `courses` | id, name, description, thumbnail (URL), slug (unique, indexed) |
| `lessons` | id, course_id (FK), name, description, slug, video_url |
| `course_teachers` | course_id + teacher_id (composite PK, M:M) |

All tables have soft-delete via `deleted_at`. All queries filter `deleted_at.is_(None)`.

> Note: `app/models/class.py` is an orphaned draft — the active model and migration use `Lesson`/`lessons`. The API response field is named `classes` (not `lessons`) to match the domain language.

**Implemented API routes (port 8000):**

| Method | Path | Description |
|---|---|---|
| GET | `/` | Welcome message |
| GET | `/health` | Status + DB connectivity check |
| GET | `/courses` | List all courses |
| GET | `/courses/{slug}` | Course detail with teacher IDs and class list |

**Not yet implemented:** `GET /courses/{slug}/classes/{id}` — defined in `specs/00_contracts.md` but missing from `app/main.py`.

---

## Frontend

**Stack:** Next.js 15.3.3 (App Router), React 19, TypeScript 5, SASS + CSS Modules, Vitest 3 + @testing-library/react, Yarn

**Architecture:** Server Components with inline `fetch()`. No API client abstraction. Base URL hardcoded to `http://localhost:8000`.

**Pages:**

| Route | Fetches |
|---|---|
| `/` | `GET /courses` |
| `/course/[slug]` | `GET /courses/{slug}` |
| `/classes/[class_id]` | `GET /classes/{class_id}` (not yet implemented in Backend) |

**Known integration bug — field name mismatch with Backend:**

| Backend returns | Frontend type expects | Impact |
|---|---|---|
| `name` | `title` | Course title renders as `undefined` |
| `teacher_id: int[]` | `teacher: string` | Teacher renders as `undefined` |
| — | `duration: number` | Not returned by API |

The mobile apps are correctly aligned with the Backend contract. The Frontend types need to be fixed to match the actual API response shape.

---

## Mobile — iOS (`PlatziFlixiOS`)

**Stack:** Swift 5.9+, SwiftUI, Combine, URLSession, Xcode

**Architecture:** CLEAR (Clean + MVVM). Layers: Domain / Data / Presentation.
- `CourseRepositoryProtocol` is the domain boundary
- `RemoteCourseRepository` is the concrete implementation
- ViewModel uses `@Published` + Combine debounce for search

**Implements:** `GET /courses` + `GET /courses/{slug}`. Has full Teacher and Class domain models.

**Base URL:** `http://localhost:8000` (hardcoded in `Services/CourseAPIEndpoints.swift`)

---

## Mobile — Android (`PlatziFlixAndroid`)

**Stack:** Kotlin 2.0.21, Jetpack Compose, Material 3, Retrofit 2 + OkHttp + Gson, Coroutines + StateFlow, Coil

**Architecture:** CLEAR + MVI hybrid. ViewModel exposes `StateFlow<UiState>` and accepts `UiEvent` sealed class.
- Manual DI via `AppModule` object — no Hilt/Dagger
- `USE_MOCK_DATA` flag in `AppModule` to swap implementations

**Implements:** `GET /courses` only (course detail not yet implemented).

**Base URL:** `http://10.0.2.2:8000/` (Android emulator localhost mapping). `network_security_config.xml` allows cleartext HTTP for local addresses.

---

## API Contract (source of truth)

See `Backend/specs/00_contracts.md`. Summary:

```json
// GET /courses
[{ "id", "name", "description", "thumbnail", "slug" }]

// GET /courses/{slug}
{ "id", "name", "description", "thumbnail", "slug", "teacher_id": [int], "classes": [{ "id", "name", "description", "slug" }] }

// GET /courses/{slug}/classes/{id}  — NOT YET IMPLEMENTED
{ "id", "name", "description", "slug", "video_url", "created_at", "updated_at", "deleted_at" }
```

---

## Known Issues & Gaps

1. **No authentication** — the API is fully open. No JWT, no sessions, no middleware in any project.
2. **Frontend ↔ Backend field mismatch** — `name` vs `title`, `teacher_id` vs `teacher`. Frontend renders undefined values.
3. **Missing endpoint** — `GET /courses/{slug}/classes/{id}` not implemented in Backend.
4. **iOS more complete than Android** — Android only has course list; iOS has course detail + Teacher + Class models.
5. **No shared contract tooling** — no OpenAPI spec, no code generation. Each client manually replicates types from the spec markdown.
6. **Lesson vs Class naming** — DB and ORM use `Lesson`/`lessons`; API, Frontend, and spec use `Class`/`classes`.

---

## Development Notes

- The `slug` field is the universal resource identifier across all projects and URLs.
- Soft-delete is implemented at the DB level. Never hard-delete records — set `deleted_at`.
- Frontend `Progress`, `Quiz`, and `FavoriteToggle` types exist in `src/types/index.ts` but have no UI or API implementation yet — forward-planned types.
- Android uses `10.0.2.2` to reach host `localhost` from the emulator; iOS uses `localhost` directly (physical device or simulator).
