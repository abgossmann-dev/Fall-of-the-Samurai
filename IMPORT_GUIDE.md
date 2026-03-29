# Spreadsheet-to-App Import Guide (Excel / Google Sheets)

Yes — you can absolutely build workouts in Excel or Google Sheets, then import them into your web app so sessions are pre-structured.

This is a strong approach for V1 because:
- you keep your existing planning workflow,
- logging in-app becomes fast,
- the app can automatically show **last week’s numbers** and **current PRs** per lift.

---

## 1) Recommended import model

Use a **template import** (planned workouts) rather than importing raw historical logs first.

### What gets imported
- Workout template name/day
- Exercise order
- Planned sets per exercise
- Target reps/load/RPE/rest
- Optional notes

### What gets calculated in-app
- Last week result per set or top set
- PR badges (max load / est 1RM / volume)
- Suggested progression next session

---

## 2) CSV format (simple and robust)

Create one CSV export from Excel/Sheets with this schema:

```csv
week_tag,template_name,day_tag,exercise_name,exercise_order,set_index,target_reps,target_load,target_rpe,rest_seconds,exercise_note
2026-W13,Upper Strength,Mon,Bench Press,1,1,5,185,8,120,Pause on chest
2026-W13,Upper Strength,Mon,Bench Press,1,2,5,185,8,120,
2026-W13,Upper Strength,Mon,Barbell Row,2,1,8,155,7,90,
2026-W13,Upper Strength,Mon,Barbell Row,2,2,8,155,7,90,
```

### Validation rules
- `template_name`, `exercise_name`, `exercise_order`, `set_index` are required.
- Numeric fields must be valid numbers or blank.
- `exercise_order` and `set_index` must start at 1.
- Duplicate row key should be rejected:
  - `(week_tag, template_name, day_tag, exercise_name, exercise_order, set_index)`

---

## 3) Mapping CSV -> database

- `template_name`, `day_tag` -> `workout_templates`
- `exercise_name`, `exercise_order`, `exercise_note` -> `template_exercises`
- `set_index`, `target_*`, `rest_seconds` -> `template_sets`

When user starts workout:
1. Pick a template.
2. Snapshot template into `workout_sessions` + `session_exercises` + `session_sets`.
3. Show last-week and PR context per exercise.

---

## 4) How to show “last week’s numbers” per lift

For each exercise on active workout screen:
1. Find most recent prior completed session for same exercise (ideally 7–14 days lookback).
2. Show either:
   - last top set (`max load`), and/or
   - last performed sets (`reps x load` list).

Example display:
- **Bench Press**
  - Last week: `185 x 5 x 3`
  - PR: `190 x 3` (Max Load PR)

---

## 5) How to compute PRs (V1)

Store PR types in `pr_records`:
- `max_load`: highest single-set load
- `est_1rm`: use Epley estimate: `load * (1 + reps/30)`
- `session_volume`: `sum(load * reps)` for exercise in session

On session completion:
1. Calculate these values for each exercise.
2. Compare against existing best in `pr_records`.
3. Insert new record when value is greater.
4. Mark set/session with PR badge in UI.

---

## 6) Progressive overload suggestions (simple rules)

After loading last week + PR context:
- If all sets hit target reps at target RPE or lower:
  - increase load next session (upper body +2.5 to +5 lb, lower body +5 to +10 lb)
- If reps missed significantly:
  - keep load same next week
- If RPE too high for target reps:
  - reduce load 2.5–5% or keep constant

Keep this as recommendation text in V1 (user can override).

---

## 7) Suggested UX for import flow

1. **Import screen** (`/import`)
   - Upload CSV
   - Validate and preview row errors
   - Confirm import
2. **Template review screen**
   - Show imported templates and exercises
   - Allow quick edits/reordering
3. **Start workout**
   - Imported templates immediately available for logging

---

## 8) Practical rollout plan

### Step A (fastest)
- Implement CSV import for templates only.
- No historical backfill.

### Step B
- Add optional historical workout import (past logs) to seed PR history.

### Step C
- Add one-click “Duplicate last week” for imported week plans.

This gives you value quickly while keeping complexity under control.
