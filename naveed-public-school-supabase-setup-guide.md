# Naveed Public School — Supabase Setup Guide

Step-by-step path from the prototype app to a real Supabase-backed database,
using VS Code for local editing/preview. Companion to
`naveed-public-school-technical-specification.md` (full schema + API contract)
and `naveed-public-school-app.html` (the working prototype).

---

## Step 1 — Create the Supabase project

1. Go to supabase.com → **New project**.
2. Pick a name (e.g. `naveed-public-school`), set a database password (save it
   somewhere), pick a region close to Peshawar (Singapore or Mumbai will have
   the lowest latency).
3. Wait ~2 minutes for it to provision.

## Step 2 — Adapt the schema for Supabase (one important change)

Supabase already has its own `auth.users` table that handles login. The
original schema in the technical specification creates a separate `users`
table — before running it, swap that block out so the app's user data links
to Supabase's real auth system instead of duplicating it.

In the schema doc, **replace the `users` table block** with this:

```sql
-- Enable needed extensions
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS citext;

-- Profile row = 1 per Supabase Auth user, holds role + school info
CREATE TABLE profiles (
  id                UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  school_id         UUID NOT NULL REFERENCES schools(id),
  role_id           SMALLINT NOT NULL REFERENCES roles(id),
  full_name         TEXT NOT NULL,
  phone             TEXT,
  is_active         BOOLEAN NOT NULL DEFAULT true,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Then, everywhere the original schema said `REFERENCES users(id)` — in
`teachers`, `students.user_id`, `attendance_sessions.taken_by`,
`grades.entered_by`, `notifications.author_id`, `audit_log.actor_id` — change
it to `REFERENCES profiles(id)`. Everything else in the schema stays the same.

## Step 3 — Run the schema

1. In Supabase, open **SQL Editor** (left sidebar) → **New query**.
2. Paste in the full schema (with the `profiles` swap from Step 2) and click
   **Run**.
3. Go to **Table Editor** and confirm you see `students`, `classes`,
   `attendance_sessions`, etc.

## Step 4 — Turn on Row Level Security

Supabase blocks all access by default once RLS is enabled, which is what you
want. Run this in the SQL Editor for a sensible starting point —
authenticated users can read, everyone authenticated can insert attendance
for now:

```sql
ALTER TABLE students ENABLE ROW LEVEL SECURITY;
ALTER TABLE attendance_records ENABLE ROW LEVEL SECURITY;
ALTER TABLE grades ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Authenticated users can read students"
  ON students FOR SELECT TO authenticated USING (true);

CREATE POLICY "Authenticated users can read attendance"
  ON attendance_records FOR SELECT TO authenticated USING (true);

CREATE POLICY "Authenticated users can insert attendance"
  ON attendance_records FOR INSERT TO authenticated WITH CHECK (true);
```

This is intentionally loose to get you moving. Tighten it later so students
can only read their *own* row, e.g. `USING (student_id = auth.uid())`, once
student logins are wired to real accounts.

## Step 5 — Get your API keys

**Settings → API** in Supabase. Copy:
- **Project URL** (`https://xxxx.supabase.co`)
- **anon public key**

## Step 6 — Open the project folder in VS Code

Put `naveed-public-school-app.html` in a folder and open that folder in VS
Code. Install the **Live Server** extension (by Ritwick Dey) from the
Extensions panel — this lets you preview the page at `http://localhost`
instead of `file://`, which matters because the camera QR scanner needs a
secure context (`localhost` counts; a plain opened file does not, in some
browsers).

## Step 7 — Connect the app to Supabase

At the top of the HTML, right after the other `<script>` tags, add:

```html
<script type="module">
  import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'
  window.supabase = createClient(
    'https://xxxx.supabase.co',   // your Project URL
    'YOUR-ANON-PUBLIC-KEY'
  )
</script>
```

## Step 8 — Replace the mock login with real Supabase auth

Swap `doLogin()` to actually authenticate:

```js
async function doLogin(email, password){
  const { data, error } = await supabase.auth.signInWithPassword({ email, password });
  if (error) { toast(error.message); return; }
  // fetch their profile row to get role/name
  const { data: profile } = await supabase.from('profiles').select('*').eq('id', data.user.id).single();
  state.user = profile;
  // continue into renderShell() as before
}
```

## Step 9 — Replace mock data reads with real queries

Wherever the app currently reads from the `STUDENTS` / `CLASSES` arrays, swap
for a Supabase call, e.g.:

```js
async function loadStudents(classId){
  const { data, error } = await supabase
    .from('students')
    .select('*, class_enrollments!inner(class_id, roll_no)')
    .eq('class_enrollments.class_id', classId);
  return data;
}
```

## Step 10 — Wire the QR scan to write real attendance

Replace the in-memory `RECORDS.push(...)` in `handleScannedCode()` with:

```js
async function markPresent(sessionId, studentId){
  const { error } = await supabase.from('attendance_records').insert({
    session_id: sessionId, student_id: studentId, status: 'present', method: 'qr_scan'
  });
  if (error?.code === '23505') { /* unique constraint = already marked */ }
}
```

That `23505` error code is the duplicate-scan protection — enforced by the
database constraint from the schema, not just app logic.

## Step 11 — Run it

Right-click the HTML file in VS Code → **Open with Live Server**. Test
signup/login with a real Supabase Auth user (create one under
**Authentication → Users** in Supabase to start).

---

*Companion to the Naveed Public School Campus Portal prototype. Developer and
Designed by: Jamal Abdul Nasir.*
