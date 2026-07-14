# Naveed Public School — Technical Specification
### Database Schema · API Contract · MVP Delivery Timeline

Prepared as the concrete build-out of the original product prompt. This document is written for a development team to implement directly.

---

## 1. Database Schema (PostgreSQL)

Design notes:
- `UUID` primary keys throughout for safe merging/sync and to avoid exposing sequential IDs (e.g. guessing student record counts).
- Soft deletes (`is_active` / `deleted_at`) on people-facing tables so historical attendance and grades stay intact after a withdrawal or staff change.
- All timestamps are `timestamptz`, stored in UTC.
- `attendance_sessions.qr_token` is single-use per session and short-lived, per the tokenized-attendance requirement in the original spec.

```sql
-- ========== Core identity & access ==========

CREATE TABLE schools (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name              TEXT NOT NULL,
  address           TEXT,
  academic_year     TEXT NOT NULL,          -- e.g. '2025-2026'
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE roles (
  id                SMALLSERIAL PRIMARY KEY,
  name              TEXT UNIQUE NOT NULL     -- 'admin' | 'principal' | 'teacher' | 'student' | 'parent'
);

CREATE TABLE permissions (
  id                SMALLSERIAL PRIMARY KEY,
  code              TEXT UNIQUE NOT NULL     -- e.g. 'attendance.edit', 'grades.publish'
);

CREATE TABLE role_permissions (
  role_id           SMALLINT REFERENCES roles(id) ON DELETE CASCADE,
  permission_id     SMALLINT REFERENCES permissions(id) ON DELETE CASCADE,
  PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE users (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id         UUID NOT NULL REFERENCES schools(id),
  role_id           SMALLINT NOT NULL REFERENCES roles(id),
  full_name         TEXT NOT NULL,
  email             CITEXT UNIQUE,
  phone             TEXT,
  password_hash     TEXT NOT NULL,
  is_active         BOOLEAN NOT NULL DEFAULT true,
  last_login_at     TIMESTAMPTZ,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_role ON users(role_id);

-- ========== People ==========

CREATE TABLE teachers (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id           UUID UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  employee_code     TEXT UNIQUE NOT NULL,
  qualification     TEXT,
  joined_on         DATE,
  is_active         BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE students (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id           UUID UNIQUE REFERENCES users(id) ON DELETE SET NULL,  -- null until portal access is issued
  admission_no      TEXT UNIQUE NOT NULL,        -- e.g. 'NPS-2026-0142'
  full_name         TEXT NOT NULL,
  date_of_birth     DATE,
  guardian_name     TEXT,
  guardian_phone    TEXT,
  emergency_contact TEXT,
  qr_code_value     TEXT UNIQUE NOT NULL,        -- opaque token encoded in the ID-card QR
  status             TEXT NOT NULL DEFAULT 'enrolled', -- 'enrolled' | 'withdrawn' | 'transferred' | 'graduated'
  enrolled_on       DATE NOT NULL DEFAULT CURRENT_DATE,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_students_status ON students(status);

CREATE TABLE parent_links (               -- optional parent-portal module
  parent_user_id    UUID REFERENCES users(id) ON DELETE CASCADE,
  student_id        UUID REFERENCES students(id) ON DELETE CASCADE,
  relationship      TEXT,                 -- 'father' | 'mother' | 'guardian'
  PRIMARY KEY (parent_user_id, student_id)
);

-- ========== Academic structure ==========

CREATE TABLE subjects (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name              TEXT NOT NULL,
  code              TEXT UNIQUE NOT NULL
);

CREATE TABLE classes (                    -- a grade + section, e.g. "Grade 9 - A"
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id         UUID NOT NULL REFERENCES schools(id),
  grade             TEXT NOT NULL,        -- 'Grade 9'
  section           TEXT NOT NULL,        -- 'A'
  room              TEXT,
  homeroom_teacher_id UUID REFERENCES teachers(id),
  academic_year     TEXT NOT NULL,
  UNIQUE (grade, section, academic_year)
);

CREATE TABLE class_enrollments (          -- student ↔ class, per academic year
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  class_id          UUID NOT NULL REFERENCES classes(id) ON DELETE CASCADE,
  student_id        UUID NOT NULL REFERENCES students(id) ON DELETE CASCADE,
  roll_no           INTEGER NOT NULL,
  enrolled_on       DATE NOT NULL DEFAULT CURRENT_DATE,
  UNIQUE (class_id, student_id),
  UNIQUE (class_id, roll_no)
);

CREATE TABLE class_subject_teachers (     -- which teacher teaches which subject in which class
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  class_id          UUID NOT NULL REFERENCES classes(id) ON DELETE CASCADE,
  subject_id        UUID NOT NULL REFERENCES subjects(id) ON DELETE CASCADE,
  teacher_id        UUID NOT NULL REFERENCES teachers(id) ON DELETE CASCADE,
  UNIQUE (class_id, subject_id)
);

CREATE TABLE schedule_periods (           -- weekly timetable
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  class_id          UUID NOT NULL REFERENCES classes(id) ON DELETE CASCADE,
  subject_id        UUID NOT NULL REFERENCES subjects(id),
  teacher_id        UUID NOT NULL REFERENCES teachers(id),
  day_of_week       SMALLINT NOT NULL,    -- 1=Mon .. 7=Sun
  start_time        TIME NOT NULL,
  end_time          TIME NOT NULL,
  room              TEXT
);

-- ========== Attendance ==========

CREATE TABLE attendance_sessions (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  class_id          UUID NOT NULL REFERENCES classes(id),
  subject_id        UUID REFERENCES subjects(id),
  taken_by          UUID NOT NULL REFERENCES users(id),
  session_date      DATE NOT NULL,
  qr_token          TEXT UNIQUE NOT NULL,   -- single-use, rotated per session
  qr_expires_at     TIMESTAMPTZ NOT NULL,
  status            TEXT NOT NULL DEFAULT 'open', -- 'open' | 'closed'
  opened_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  closed_at         TIMESTAMPTZ
);
CREATE INDEX idx_sessions_class_date ON attendance_sessions(class_id, session_date);

CREATE TABLE attendance_records (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id        UUID NOT NULL REFERENCES attendance_sessions(id) ON DELETE CASCADE,
  student_id        UUID NOT NULL REFERENCES students(id),
  status            TEXT NOT NULL,          -- 'present' | 'absent' | 'late' | 'excused'
  method            TEXT NOT NULL,          -- 'qr_scan' | 'manual' | 'auto_absent'
  marked_by         UUID REFERENCES users(id),   -- null for QR self-scan
  marked_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  device_info       TEXT,                    -- optional: scanning device fingerprint for audit
  UNIQUE (session_id, student_id)             -- prevents duplicate check-in for the same session
);
CREATE INDEX idx_records_student ON attendance_records(student_id);

-- ========== Exams & grading ==========

CREATE TABLE exams (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  class_id          UUID NOT NULL REFERENCES classes(id),
  name              TEXT NOT NULL,           -- 'Mid Term Examination'
  exam_date         DATE NOT NULL,
  status            TEXT NOT NULL DEFAULT 'draft', -- 'draft' | 'published'
  created_by        UUID NOT NULL REFERENCES users(id),
  published_at      TIMESTAMPTZ
);

CREATE TABLE exam_subjects (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  exam_id           UUID NOT NULL REFERENCES exams(id) ON DELETE CASCADE,
  subject_id        UUID NOT NULL REFERENCES subjects(id),
  max_marks         NUMERIC(6,2) NOT NULL DEFAULT 100,
  weight            NUMERIC(4,2) NOT NULL DEFAULT 1.0,   -- for weighted grade calculation
  UNIQUE (exam_id, subject_id)
);

CREATE TABLE grades (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  exam_subject_id   UUID NOT NULL REFERENCES exam_subjects(id) ON DELETE CASCADE,
  student_id        UUID NOT NULL REFERENCES students(id),
  marks_obtained    NUMERIC(6,2) NOT NULL,
  entered_by        UUID NOT NULL REFERENCES users(id),
  entered_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (exam_subject_id, student_id)
);

-- ========== Notifications & audit ==========

CREATE TABLE notifications (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id         UUID NOT NULL REFERENCES schools(id),
  author_id         UUID NOT NULL REFERENCES users(id),
  category          TEXT NOT NULL,           -- 'General' | 'Exams' | 'Attendance' | 'Event'
  title             TEXT NOT NULL,
  body              TEXT,
  audience          TEXT NOT NULL DEFAULT 'all', -- 'all' | class_id | role name
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
  id                BIGSERIAL PRIMARY KEY,
  actor_id          UUID REFERENCES users(id),
  action            TEXT NOT NULL,
  entity_type       TEXT,
  entity_id         UUID,
  metadata          JSONB,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_created ON audit_log(created_at DESC);
```

**Entity relationship summary**
- `users` is the auth root; `teachers`, `students`, and parent accounts all extend it 1-to-1 (or, for students, 0-to-1 until portal access is issued).
- `classes` connects to `students` via `class_enrollments` (many-to-many across years) and to `teachers` via `class_subject_teachers`.
- `attendance_sessions` → `attendance_records` is 1-to-many; the `UNIQUE (session_id, student_id)` constraint is what actually enforces "no duplicate check-in," rather than relying on application logic alone.
- `exams` → `exam_subjects` → `grades` lets each subject carry its own max marks/weight, so weighted grade calculation isn't hardcoded.

---

## 2. API Contract (REST, JSON)

Base URL: `https://api.naveedpublicschool.edu.pk/v1`
Auth: `Authorization: Bearer <JWT>` on every request except `/auth/login`. Tokens are short-lived (15 min access / 7-day refresh).

### 2.1 Auth
| Method | Endpoint | Description |
|---|---|---|
| POST | `/auth/login` | Exchange credentials for an access + refresh token |
| POST | `/auth/refresh` | Exchange a refresh token for a new access token |
| POST | `/auth/logout` | Revoke the current refresh token |

```json
// POST /auth/login  →  request
{ "email": "ayesha.rahman@nps.edu.pk", "password": "••••••••" }

// 200 response
{
  "access_token": "eyJhbGciOi...",
  "refresh_token": "eyJhbGciOi...",
  "user": { "id": "5b2e...", "full_name": "Ayesha Rahman", "role": "teacher" }
}
```

### 2.2 Students
| Method | Endpoint | Description |
|---|---|---|
| GET | `/students?class_id=&q=&page=` | List / search students (paginated) |
| POST | `/students` | Enroll a new student *(admin)* |
| GET | `/students/{id}` | Student profile |
| PATCH | `/students/{id}` | Update profile / status *(admin)* |
| GET | `/students/{id}/attendance-summary` | Attendance rate + recent history |
| GET | `/students/{id}/id-card` | Returns QR payload + printable card data |

```json
// POST /students → request
{
  "full_name": "Zara Ahmed",
  "class_id": "c19e...",
  "date_of_birth": "2011-03-14",
  "guardian_name": "Ahmed family",
  "guardian_phone": "0301-1234567"
}

// 201 response
{
  "id": "a71f...",
  "admission_no": "NPS-2026-0143",
  "qr_code_value": "QR-NPS-2026-0143",
  "class_id": "c19e...",
  "status": "enrolled"
}
```

### 2.3 Teachers & Classes
| Method | Endpoint | Description |
|---|---|---|
| GET | `/teachers` | List teaching staff |
| POST | `/teachers` | Add a teacher *(admin)* |
| GET | `/classes` | List classes with roster counts |
| GET | `/classes/{id}/roster` | Full student roster for a class |
| GET | `/classes/{id}/schedule` | Weekly timetable |

### 2.4 Attendance
| Method | Endpoint | Description |
|---|---|---|
| POST | `/attendance/sessions` | Open a new session for a class (generates `qr_token`) |
| GET | `/attendance/sessions/{id}` | Session status + live roster |
| POST | `/attendance/sessions/{id}/close` | Close session; unmarked students → `absent` |
| POST | `/attendance/scan` | Submit a scanned QR payload to mark attendance |
| POST | `/attendance/manual` | Manually set a student's status *(teacher/admin override, audit-logged)* |
| GET | `/attendance/reports?class_id=&from=&to=` | Aggregated attendance report |
| GET | `/attendance/reports/export.csv` | CSV export of the same report |

```json
// POST /attendance/sessions → request
{ "class_id": "c19e...", "subject_id": "s02...", "session_date": "2026-07-14" }

// 201 response
{
  "id": "sess_88a1...",
  "qr_token": "SES-C19E-20260714-9F3K",
  "qr_expires_at": "2026-07-14T09:15:00Z",
  "status": "open"
}
```

```json
// POST /attendance/scan → request  (called by the scanning device after jsQR decodes a card)
{ "session_id": "sess_88a1...", "qr_code_value": "QR-NPS-2026-0143" }

// 200 response — success
{ "student_id": "a71f...", "full_name": "Zara Ahmed", "status": "present", "marked_at": "2026-07-14T08:04:12Z" }

// 409 response — duplicate scan
{ "error": "already_marked", "message": "Zara Ahmed is already marked present for this session." }

// 422 response — wrong class
{ "error": "class_mismatch", "message": "This student is not enrolled in the scanning class." }
```

### 2.5 Exams & Results
| Method | Endpoint | Description |
|---|---|---|
| POST | `/exams` | Create an exam for a class *(admin/teacher)* |
| GET | `/exams/{id}` | Exam detail incl. subjects and status |
| PUT | `/exams/{id}/grades` | Bulk upsert marks (array of `{student_id, subject_id, marks}`) |
| POST | `/exams/{id}/publish` | Publish results to students/parents |
| GET | `/exams/{id}/results` | Full class result sheet |
| GET | `/students/{id}/results` | A single student's published results across exams |

```json
// PUT /exams/{id}/grades → request
{
  "entries": [
    { "student_id": "a71f...", "subject_id": "s02...", "marks_obtained": 87 },
    { "student_id": "b02c...", "subject_id": "s02...", "marks_obtained": 74 }
  ]
}
```

### 2.6 Notifications & Reports
| Method | Endpoint | Description |
|---|---|---|
| GET | `/notifications?audience=` | List announcements visible to the caller |
| POST | `/notifications` | Post an announcement *(admin/teacher)* |
| GET | `/reports/attendance-trend?class_id=&days=7` | Time-series for dashboard charts |
| GET | `/reports/grade-distribution?exam_id=` | Grade-band histogram |
| GET | `/audit-log?entity_type=&from=&to=` | Audit trail *(admin only)* |

### 2.7 Common conventions
- Pagination: `?page=1&page_size=25`, response wraps data in `{ "data": [...], "meta": { "total": 120, "page": 1 } }`.
- Errors: consistent shape `{ "error": "snake_case_code", "message": "human-readable" }` with standard HTTP status codes (400/401/403/404/409/422/500).
- Every write to `attendance_records` and `grades` writes a matching row to `audit_log` server-side — not optional, not client-controlled.

---

## 3. MVP Delivery Timeline

Scoped as roughly a **12-week build** for a small team (1 tech lead, 2 backend/frontend engineers, 1 QA, part-time UX/DevOps). Adjust pacing to your team's actual velocity — this is a sequencing plan, not a fixed-price estimate.

| Phase | Weeks | Milestone | Scope |
|---|---|---|---|
| **0 — Discovery & Setup** | 1–2 | Signed-off spec | Finalize this schema/API contract with stakeholders, environment setup (repo, CI, staging), design system starter kit, confirm hosting choice |
| **1 — Foundations** | 3–4 | Login works end-to-end | Auth (JWT + roles/permissions), user/student/teacher CRUD, class & roster management, base UI shell with role-based navigation |
| **2 — Attendance Core** | 5–6 | A teacher can run a real class session | Attendance sessions, manual marking, QR code generation per student (ID cards), camera-based QR scan-in with duplicate/mismatch handling, audit logging |
| **3 — Exams & Results** | 7–8 | A class's results can be entered and published | Exam creation, bulk mark entry, grade calculation, publish workflow, student/parent results view |
| **4 — Reporting & Notifications** | 9 | Admin has visibility across the school | Attendance trend & grade distribution dashboards, CSV export, announcements module |
| **5 — Hardening & Pilot** | 10–11 | Ready for a real classroom pilot | Accessibility pass, load testing for exam-week concurrency, data encryption/retention review, staff training materials, bug-fix buffer |
| **6 — Launch** | 12 | Production release | Deploy, monitor, staff onboarding, feedback loop for phase-2 backlog (parent portal, SMS alerts, offline caching) |

**Suggested MVP cut line** (if the timeline needs to compress): ship Phases 0–3 first — auth, attendance (manual + QR), and exams/results are the features that directly serve the "streamline management + reliable contactless attendance for exams and audits" goal stated in the original brief. Reporting dashboards and the parent portal can follow in a fast-second release without blocking the first rollout.

---

*Companion document to the Naveed Public School Campus Portal prototype. Developer and Designed by: Jamal Abdul Nasir.*
