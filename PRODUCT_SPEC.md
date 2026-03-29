# Workout Tracker V1 — Implementation Pack

This document converts the original MVP spec into four build-ready artifacts:
1) Simple database schema
2) Page-by-page wireframes
3) Prioritized build order
4) Codex prompt that starts frontend-only

---

## 1) Simple Database Schema (V1)

> Goal: Keep schema minimal, single-user, and easy to evolve.

```sql
-- Single-user assumption: no users table required for V1.

CREATE TABLE workout_templates (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  day_tag TEXT,              -- e.g. Mon, Push, Lower
  goal TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE template_exercises (
  id TEXT PRIMARY KEY,
  template_id TEXT NOT NULL REFERENCES workout_templates(id) ON DELETE CASCADE,
  sort_order INTEGER NOT NULL,
  exercise_name TEXT NOT NULL,
  notes TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE template_sets (
  id TEXT PRIMARY KEY,
  template_exercise_id TEXT NOT NULL REFERENCES template_exercises(id) ON DELETE CASCADE,
  set_index INTEGER NOT NULL,
  target_reps INTEGER,
  target_load REAL,
  target_rpe REAL,
  rest_seconds INTEGER,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE workout_sessions (
  id TEXT PRIMARY KEY,
  template_id TEXT REFERENCES workout_templates(id) ON DELETE SET NULL,
  session_date TEXT NOT NULL, -- YYYY-MM-DD
  status TEXT NOT NULL DEFAULT 'in_progress', -- in_progress | completed | skipped
  started_at TEXT,
  ended_at TEXT,
  notes TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE session_exercises (
  id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL REFERENCES workout_sessions(id) ON DELETE CASCADE,
  sort_order INTEGER NOT NULL,
  exercise_name TEXT NOT NULL,
  notes TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE session_sets (
  id TEXT PRIMARY KEY,
  session_exercise_id TEXT NOT NULL REFERENCES session_exercises(id) ON DELETE CASCADE,
  set_index INTEGER NOT NULL,
  reps INTEGER,
  load REAL,
  rpe REAL,
  completed INTEGER NOT NULL DEFAULT 0, -- 0/1 boolean
  completed_at TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE pr_records (
  id TEXT PRIMARY KEY,
  exercise_name TEXT NOT NULL,
  pr_type TEXT NOT NULL, -- max_load | est_1rm | session_volume
  value REAL NOT NULL,
  achieved_on TEXT NOT NULL, -- YYYY-MM-DD
  session_id TEXT REFERENCES workout_sessions(id) ON DELETE SET NULL,
  created_at TEXT NOT NULL
);

CREATE TABLE bodyweight_entries (
  id TEXT PRIMARY KEY,
  entry_date TEXT NOT NULL, -- YYYY-MM-DD
  weight REAL NOT NULL,
  note TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

-- Helpful indexes
CREATE INDEX idx_template_exercises_template_id ON template_exercises(template_id);
CREATE INDEX idx_template_sets_template_exercise_id ON template_sets(template_exercise_id);
CREATE INDEX idx_sessions_date ON workout_sessions(session_date);
CREATE INDEX idx_session_exercises_session_id ON session_exercises(session_id);
CREATE INDEX idx_session_sets_session_exercise_id ON session_sets(session_exercise_id);
CREATE INDEX idx_pr_exercise_date ON pr_records(exercise_name, achieved_on DESC);
CREATE INDEX idx_bodyweight_date ON bodyweight_entries(entry_date DESC);
```

### Notes on behavior
- When starting a workout from template, copy template exercises/sets into `session_exercises`/`session_sets`.
- Weekly completion = completed sessions / planned sessions in a 7-day window.
- “Duplicate last week” creates new planned sessions based on prior week templates.

---

## 2) Page-by-Page Wireframes (Low-Fidelity)

## A. Dashboard (`/`)

```
+------------------------------------------------+
| Header: Week of Mar 23–29         [Settings]   |
+------------------------------------------------+
| Completion Card                                  |
| 4 / 5 workouts completed        80%             |
+------------------------------------------------+
| Next Workout                                     |
| Upper Strength (Today 6:00 PM)   [Start]        |
+------------------------------------------------+
| PR Highlights (last 7 days)                      |
| Bench Press +5 lb   Deadlift est1RM +8 lb       |
+------------------------------------------------+
| Bodyweight Trend (mini chart)                    |
+------------------------------------------------+
| [Start Workout] [Templates] [History] [Week]    |
+------------------------------------------------+
```

## B. Templates List (`/templates`)

```
+------------------------------------------------+
| Templates                          [+ New]      |
+------------------------------------------------+
| Push Day          6 exercises      [Edit]       |
| Pull Day          5 exercises      [Edit]       |
| Lower Day         7 exercises      [Edit]       |
+------------------------------------------------+
```

## C. Template Builder (`/templates/:id`) — desktop-friendly

```
+---------------------------------------------------------------+
| Template: Push Day                      [Save] [Duplicate]    |
+---------------------------------------------------------------+
| Exercise 1: Bench Press                               [Move]  |
|  Set 1  reps: 5  load: 185  rpe: 8  rest:120                 |
|  Set 2  reps: 5  load: 185  rpe: 8  rest:120                 |
|  [+ Add Set]                                                 |
+---------------------------------------------------------------+
| Exercise 2: Incline DB Press                          [Move]  |
| ...                                                         |
+---------------------------------------------------------------+
| [+ Add Exercise]                                             |
+---------------------------------------------------------------+
```

## D. Start Workout (`/log/start`)

```
+------------------------------------------------+
| Start Workout                                   |
+------------------------------------------------+
| Select Template                                 |
| ( ) Push Day                                    |
| ( ) Pull Day                                    |
| ( ) Blank Workout                               |
+------------------------------------------------+
| Bodyweight (optional): [ 181.2 ]                |
+------------------------------------------------+
|                 [Start Session]                 |
+------------------------------------------------+
```

## E. Active Logger (`/log/:sessionId`) — mobile-first

```
+----------------------------------------------+
| Push Day                           42:10     |
+----------------------------------------------+
| Bench Press                                     
| Last: 185 x 5                                   
| [✓] Set1  reps [-]5[+]  load [-]185[+]         
| [ ] Set2  reps [-]5[+]  load [-]185[+]         
| [ ] Set3  reps [-]5[+]  load [-]185[+]         
| Notes: [..............................]         
+----------------------------------------------+
| Incline DB Press ...                           |
+----------------------------------------------+
| Workout Notes: [..........................]    |
+----------------------------------------------+
| [Save Draft]                    [Finish]      |  <- sticky
+----------------------------------------------+
```

## F. Workout Summary (`/log/:sessionId/summary`)

```
+------------------------------------------------+
| Workout Complete ✅                             |
+------------------------------------------------+
| Duration: 58m    Volume: 12,430 lb             |
| New PRs: 2                                     |
| - Bench Press max load +5 lb                   |
| - Row volume PR +420 lb                        |
+------------------------------------------------+
| [View History] [Back to Dashboard]             |
+------------------------------------------------+
```

## G. History (`/history`)

```
+------------------------------------------------+
| History                                         |
+------------------------------------------------+
| Exercise: [Bench Press v]  Range: [12 weeks v] |
+------------------------------------------------+
| (Line Chart: load / est1RM / volume)           |
+------------------------------------------------+
| Mar 27   185 x 5 x 3   est1RM 216              |
| Mar 20   180 x 5 x 3   est1RM 210              |
+------------------------------------------------+
```

## H. Weekly Completion (`/week`)

```
+------------------------------------------------+
| Week View                      [Dup Last Week] |
+------------------------------------------------+
| Mon  Push     ✅                                |
| Tue  Pull     ✅                                |
| Wed  Rest     -                                 |
| Thu  Lower    ⬜ planned                         |
| Fri  Upper    ⬜ planned                         |
+------------------------------------------------+
| Completion: 2 / 4 (50%)                        |
+------------------------------------------------+
```

## I. PR Center (`/prs`)

```
+------------------------------------------------+
| PR Center                                       |
+------------------------------------------------+
| Bench Press | max_load    | 190 lb | Mar 27    |
| Deadlift    | est_1rm     | 405 lb | Mar 22    |
| Pull-up     | session_vol | 5200   | Mar 18    |
+------------------------------------------------+
```

## J. Bodyweight (`/bodyweight`)

```
+------------------------------------------------+
| Bodyweight                                      |
+------------------------------------------------+
| Today: [181.2] [Add]                           |
+------------------------------------------------+
| (Trend chart + 7-day moving average)           |
+------------------------------------------------+
| Mar 29  181.2                                  |
| Mar 28  181.8                                  |
+------------------------------------------------+
```

---

## 3) Prioritized Build Order

### Phase 0 — Project setup (Day 1)
1. Initialize app shell, design tokens, responsive layout primitives.
2. Define schema + migrations + seed data.
3. Create base components (cards, buttons, inputs, bottom action bar).

### Phase 1 — Frontend-only prototype (Days 1–3)
1. Build all routes/screens with mocked local JSON data.
2. Implement mobile logging interactions (tap targets, +/- steppers, sticky footer).
3. Build template builder UI interactions (add/reorder sets/exercises).
4. Add low-fi charts with mock series.
5. Add navigation + empty/loading/error visual states.

**Exit criteria:** Clickable end-to-end demo with no backend required.

### Phase 2 — Data layer + CRUD (Days 4–6)
1. Connect templates CRUD to DB.
2. Start session from template snapshot.
3. Persist logging edits with autosave.
4. Persist notes + bodyweight entries.

### Phase 3 — Intelligence features (Days 7–8)
1. PR detection logic (max load, est1RM, session volume).
2. Weekly completion calculations.
3. Duplicate last week action.

### Phase 4 — Hardening (Days 9–10)
1. Unit tests for PR + weekly completion logic.
2. Basic E2E: create template → log workout → observe PR.
3. Performance pass for iPhone (bundle + render optimizations).
4. PWA installability and offline cache essentials.

---

## 4) Codex Prompt (Frontend-Only First)

```text
Build Version 1 of a mobile-first browser-based workout tracker/dashboard for a single user.

IMPORTANT DELIVERY ORDER:
1) First deliver a frontend-only prototype using mock data and local state.
2) Then (in a second pass) wire the same UI to a real database and APIs.
Do not start with backend-first architecture.

Product goals:
- Clean, modern, athletic dashboard feel (simpler than TRAQ)
- Better UX than spreadsheet logging
- Desktop-friendly template building, iPhone-friendly workout logging
- Very low-friction in-workout data entry

Tech stack:
- Next.js + TypeScript + App Router
- Tailwind CSS + shadcn/ui
- Recharts
- (Second pass) Prisma + SQLite

Routes to build in prototype:
- /
- /templates
- /templates/:id
- /log/start
- /log/:sessionId
- /log/:sessionId/summary
- /history
- /week
- /prs
- /bodyweight

Frontend-only prototype requirements:
- Use realistic mock data in a single source file
- Full clickable navigation between all pages
- Mobile-first layout and >=44px tap targets on logger controls
- Sticky bottom action bar on active workout page
- One-tap set completion and +/- steppers for reps/load
- Notes fields for workout + exercise
- Mock charts for progress and bodyweight
- Weekly completion card and “Duplicate last week” button (UI + mocked behavior)
- Empty/loading/error visual states

Second pass requirements (after prototype):
- Implement DB schema with tables:
  workout_templates, template_exercises, template_sets,
  workout_sessions, session_exercises, session_sets,
  pr_records, bodyweight_entries
- Snapshot template -> session on workout start
- Autosave session updates
- PR detection: max load, est1RM, session volume
- Weekly completion calculation

Quality bar:
- Strict TypeScript typing
- Reusable UI components
- Clean folder structure by feature
- README with setup and run steps
- Seeded demo data
- Unit tests for PR + weekly completion logic
- Basic E2E happy path test

Output format:
- Provide complete runnable code
- Explain architecture briefly
- List what is prototype-only vs fully wired
```
