# Workout Tracker Dashboard — V1 Product Spec (Single-User MVP)

## 1) MVP Feature Set

### In scope (V1)
- **Single-user accountless/local-first experience** (one athlete: you).
- **Dashboard home screen** with:
  - This week completion snapshot.
  - Next planned workout.
  - Last session summary.
  - Quick links (Log Workout, Templates, History, PRs).
- **Workout template builder** (desktop-first editing UX):
  - Create/edit template name, training day, goal.
  - Add/reorder exercises.
  - Define set schema per exercise (sets/reps/target load/RPE/rest).
  - Save as reusable template.
- **Workout logging screen** (mobile-first):
  - Large tap targets, sticky save bar, thumb-friendly controls.
  - One-tap set completion.
  - Quick +/− adjustments for reps/load.
  - Notes at workout and exercise level.
  - Timer/rest helper (basic).
- **Exercise history + progress charts**:
  - Per-exercise trend (load, reps, estimated 1RM, volume).
  - Session list with filters.
- **Weekly completion view**:
  - Calendar/week grid of planned vs completed workouts.
  - Completion % for current and past weeks.
- **PR tracking**:
  - Auto-detect new bests (top set load, estimated 1RM, volume PR).
  - Dedicated PR list with date + linked workout.
- **Bodyweight tracking**:
  - Manual daily/weekly entries.
  - Trend line with 7-day moving average.
- **Duplicate last week’s workouts**:
  - One-click action from weekly view to copy prior week plan.

### Explicitly out of scope (for now)
- Team/coach/athlete management.
- Social features, comments, messaging.
- Wearable integrations.
- Complex periodization engine.
- Nutrition tracking beyond bodyweight notes.
- Offline conflict sync across many devices (basic local persistence only).

---

## 2) Product Spec

### Product goal
Build a fast, mobile-first web app that makes logging workouts feel like an app (not a spreadsheet), while keeping template planning efficient on desktop.

### Primary user
- **You**, a single self-coached athlete.
- Uses desktop for planning and phone (iPhone) during sessions.

### Success criteria (first 4–6 weeks)
- Log a full workout in < 90 seconds setup + in-session taps.
- Complete > 95% of workout entries without switching to spreadsheet.
- Open logging page on iPhone in < 2 seconds on repeat visits.
- Duplicate next week plan in < 10 seconds.

### Core user flows
1. **Build template on desktop**
   - Create template → add exercises/sets → save.
2. **Start workout on phone**
   - Tap "Log Workout" → choose today’s template → start session.
3. **Log sets quickly**
   - Tap set complete, adjust load/reps, add notes as needed.
4. **Finish + review**
   - End workout → summary + PR detection.
5. **Review progress weekly**
   - View completion, PRs, bodyweight trend.
6. **Duplicate last week**
   - Weekly view action copies templates/sessions into next week.

### Data model (V1)
- **Template**
  - id, name, dayTag, goal, createdAt, updatedAt
- **TemplateExercise**
  - id, templateId, order, exerciseName, defaultNotes
- **TemplateSet**
  - id, templateExerciseId, setIndex, targetReps, targetLoad, targetRPE, restSec
- **WorkoutSession**
  - id, templateId (nullable), date, startedAt, endedAt, workoutNotes, status
- **SessionExercise**
  - id, sessionId, exerciseName, order, exerciseNotes
- **SessionSet**
  - id, sessionExerciseId, setIndex, reps, load, rpe, completedAt, isPRFlag
- **BodyweightEntry**
  - id, date, weight, note
- **PRRecord**
  - id, exerciseName, prType, value, date, sessionId

### Functional requirements
- Create/edit/delete templates.
- Reorder exercises and sets.
- Start session from template or blank session.
- Autosave logging entries after every meaningful change.
- Mark sets complete/incomplete quickly.
- Add notes at workout and exercise levels.
- Detect PRs when workout ends (and optionally live after each set).
- Show exercise history chart and session table.
- Show weekly completion metrics + duplicate last week action.
- Add/edit bodyweight entries and show trend.

### Non-functional requirements
- **Performance**: first interactive paint on mobile target < 2.5s; route transitions < 200ms perceived.
- **Responsiveness**: mobile-first layouts, but usable desktop builder.
- **Reliability**: no data loss during logging (autosave + local persistence).
- **Accessibility**: large hit areas (>=44px), sufficient contrast, keyboard support on desktop.
- **Privacy**: personal-only data, no external sharing by default.

### UX principles
- One primary action per screen.
- Logging should require minimal typing.
- Big numeric inputs and steppers over freeform text.
- Show just enough context (last set, target set, rest countdown).
- Clean "athletic dashboard" visual language: high contrast cards, concise metrics, strong typography.

### Risks and mitigations
- **Risk**: Logging friction from too many fields.
  - **Mitigation**: progressive disclosure; hide advanced fields by default.
- **Risk**: Data inconsistency between template and session copies.
  - **Mitigation**: snapshot template into session at start.
- **Risk**: iPhone performance degradation with heavy charts.
  - **Mitigation**: lightweight chart library + virtualized history lists.

### Delivery plan
- **Phase 1 (week 1):** data schema + template CRUD + dashboard shell.
- **Phase 2 (week 2):** mobile logging flow + autosave + notes.
- **Phase 3 (week 3):** charts, PR engine, weekly completion, bodyweight.
- **Phase 4 (week 4):** polish, QA, PWA installability, performance pass.

---

## 3) Proposed Pages / Screens

1. **Dashboard (`/`)**
   - Weekly completion card
   - Next workout card
   - Recent PRs card
   - Bodyweight mini trend
   - CTA buttons: Start Workout, Templates, History

2. **Templates List (`/templates`)**
   - Template cards by day or goal
   - Create new template
   - Duplicate template

3. **Template Builder (`/templates/:id`)**
   - Desktop-optimized editor
   - Exercise reorder (drag/handle)
   - Set rows with quick presets (3x5, 4x8, etc.)

4. **Start Workout (`/log/start`)**
   - Pick today template or blank workout
   - Optional quick bodyweight entry

5. **Active Workout Logger (`/log/:sessionId`)**
   - Exercise accordion or cards
   - Large set rows and complete toggles
   - +/− steppers for reps/load
   - Workout notes + exercise notes
   - Sticky bottom: Save/Finish

6. **Workout Summary (`/log/:sessionId/summary`)**
   - Session totals (volume, duration)
   - New PR highlights
   - Quick "repeat this template" shortcut

7. **Exercise History (`/history`)**
   - Filter by exercise
   - Trend chart + session table
   - PR markers on chart

8. **Weekly View (`/week`)**
   - Week calendar/list
   - Planned vs completed
   - Duplicate last week button

9. **PR Center (`/prs`)**
   - PR feed by exercise and type
   - Drill into related session

10. **Bodyweight (`/bodyweight`)**
    - Entry list + add entry
    - Weight trend + moving average

---

## 4) Best Tech Stack (for your use case)

### Recommended stack
- **Frontend/App**: Next.js (App Router) + TypeScript.
- **UI**: Tailwind CSS + shadcn/ui primitives.
- **State/data**: TanStack Query + Zustand (UI/session state).
- **Database**: SQLite with Prisma (or Drizzle) for simple single-user persistence.
- **Auth**: None for V1 (single local user assumption).
- **Charts**: Recharts (or lightweight alternative like uPlot if needed for performance).
- **Forms**: React Hook Form + Zod validation.
- **PWA**: next-pwa (or native service worker setup) for installable app-like behavior.
- **Deployment**: Vercel (easy), or Fly.io/Render if you want persistent file volume control.

### Why this stack
- Fast iteration and strong DX.
- Excellent mobile web performance potential.
- Easy responsive component model for desktop builder + mobile logger.
- Prisma + SQLite keeps setup simple and low-maintenance for personal use.

### Alternative if you want super-minimal backend ops
- Next.js + Supabase (Postgres + auth/storage, though auth can remain trivial).
- Good if you later expand to multi-device robust sync.

---

## 5) Single Codex Build Prompt (copy/paste)

```text
Build a production-quality Version 1 of a mobile-first browser workout tracker/dashboard for a single user.

Product intent:
- Feels like a clean athletic dashboard (inspired by TRAQ style), much simpler.
- Better UX than Excel mobile.
- Desktop-friendly template building, iPhone-friendly workout logging.

Tech requirements:
- Next.js (latest stable) + TypeScript + App Router
- Tailwind CSS + shadcn/ui
- Prisma + SQLite
- TanStack Query
- React Hook Form + Zod
- Recharts for charts
- PWA install support

Core V1 features to implement:
1) Dashboard home
2) Workout template builder
3) Mobile workout logging screen with large tap targets
4) Exercise history/progress charts
5) Weekly completion view
6) PR tracking
7) Bodyweight tracking
8) Workout and exercise notes
9) Duplicate last week’s workout

Data model:
- Template, TemplateExercise, TemplateSet
- WorkoutSession, SessionExercise, SessionSet
- PRRecord
- BodyweightEntry
Include createdAt/updatedAt and relational integrity.

UX requirements:
- Mobile-first UI with >=44px tap targets on logging interactions
- Sticky bottom action bar on active workout screen
- One-tap set completion
- Quick +/- controls for reps and load
- Minimal typing while logging
- Clear contrast and concise metric cards

Pages/routes required:
- / dashboard
- /templates
- /templates/:id
- /log/start
- /log/:sessionId
- /log/:sessionId/summary
- /history
- /week
- /prs
- /bodyweight

Behavior requirements:
- Autosave workout changes immediately (debounced acceptable)
- Snapshot template into session when starting workout
- PR detection (best load, estimated 1RM, and volume PR)
- Weekly completion metrics from planned vs completed sessions
- “Duplicate last week” action that copies prior week templates/sessions into current week plan

Implementation quality bar:
- Strict TypeScript types
- Server actions or API routes with validation
- Error/empty/loading states on all data screens
- Seed script with realistic sample data
- Unit tests for PR calculation and weekly completion logic
- Basic E2E test for creating template -> logging workout -> seeing PR

Output expectations:
- Provide complete runnable code
- Include README with setup/run instructions
- Include npm scripts for dev, build, test, db:migrate, db:seed
- Explain architecture decisions briefly
```
