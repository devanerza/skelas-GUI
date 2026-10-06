# Skelas — Academic Course Scheduling System

Backend scheduling engine for university course timetables. Built as a
Constraint Satisfaction Problem (CSP) solved with OR-Tools CP-SAT.

Full details in [PRD.md](PRD.md), execution plan in [TASKS.md](TASKS.md).

---

## Status

| Phase | Content | Status |
|---|---|---|
| 1 | Data modeling & input validation | Done |
| 2 | Scheduling engine (CP-SAT) | Done |
| 3 | Testing & validation | Done |
| 4 | Database & API | Not started |
| 5 | Dashboard/UI | Not started |

---

## Usage

```bash
pip install -r requirements.txt

python validate.py            # validate input data
python run_scheduler.py       # generate schedule → schedule.json
python validate_schedule.py schedule.json   # independent validation

python -m unittest tests.test_scheduler      # 15 unit tests
python tests/benchmark.py                    # benchmark 10/25/50/80 courses
```

Console output (progress, schedule, diagnosis) is in Bahasa Indonesia;
status values (`FEASIBLE` / `PARTIAL` / `INFEASIBLE`) stay in English
because code and tests depend on them.

---

## Structure

```
├── data/                  # 8 JSON files (courses, lecturers, rooms, etc.)
├── src/
│   ├── models/            # dataclass per entity
│   ├── validation/        # input data validator
│   ├── data_loader.py     # load + lookup maps
│   ├── scheduler.py       # CP-SAT engine
│   └── utils.py           # consecutive slot block generator
├── tests/
│   ├── test_scheduler.py  # 15 unit tests
│   ├── benchmark.py       # deterministic benchmark
│   └── diagnose_bench.py  # capacity check when INFEASIBLE
├── run_scheduler.py       # scheduler CLI
├── validate.py            # data validation CLI
└── validate_schedule.py   # schedule validation CLI (independent)
```

---

## Problem

Find a combination of **course × lecturer × student group × day × consecutive
slots × room** satisfying 7 hard constraints:

1. A lecturer cannot teach two classes at the same time
2. A student group cannot attend two classes at the same time
3. A room cannot host two classes at the same time
4. Classes may only run in lecturers' available slots
5. Room must match the required type (e.g. computer courses need COMPUTER_LAB)
6. N SKS = N consecutive slots (no lunch break — class/lecturer handle it)
7. One class must not be split across days

**Key principle**: student conflicts come from `student_groups`, not
`curriculum_semester` — semester 3 students may take a semester 5 course.

---

## How the Engine Works

### Partial scheduling (always returns a schedule)

Every course gets a reified `sched` boolean. All of its constraints are
guarded by it, and the objective maximizes `sum(sched)` first, so the solver
itself decides which courses must be skipped when everything cannot fit:

- **0 scheduled** → `INFEASIBLE`
- **all scheduled** → `FEASIBLE`
- **some scheduled** → `PARTIAL` — the schedule is printed, then a
  `DILEWATI` section lists each skipped course with a concrete reason
  (which lecturer/group/room is saturated), plus a per-course diagnosis.

### 4.7 Diagnosis — the per-course card method

Two separate paths (kept separate on purpose):

| Path | Trigger | Output |
|---|---|---|
| `_drop_reason(course_id, schedule)` | PARTIAL run — solver already placed everything it could | `dropped[]`: structured **per-course card** |
| `diagnose()` | full INFEASIBLE/UNKNOWN — no partial schedule exists to reason against | `findings[]`: plain strings, entity-level |

`diagnose_dropped()` is now a pass-through returning `dropped` as
`List[dict]` — the cards already carry `course_name`/`reason`/`suggestion`.

**Copy this method to another repo in 4 steps:**

1. **Aggregate every blocker, don't stop at the first.** `_drop_reason`
   runs the check ladder (no room type → no valid (day, block) from
   lecturer availability → both return early, they are structural) and then
   *collects* the rest into `blockers: [(kind, text, suggestion)]`:
   `lecturer` (co-lecturer fully booked in every valid window), `group`
   (student group fully booked), `room` (no compatible room free in any
   valid window), `combination` (each resource free somewhere, never
   together — includes how many valid `(day, block)` options remain).
   `reason = "Blokir: " + "; ".join(texts)`.

2. **Add a `suggestion` column.** Each blocker carries its own generic fix
   (name the file: `data/lecturer_availability.json`,
   `data/teaching_assignments.json`, `data/course_enrollments.json`,
   `data/rooms.json`). SQLite: add the column to the `schedule_dropped`
   DDL + a forward `ALTER TABLE ... ADD COLUMN` in `connect()` so existing
   `.db` files migrate; persist/load it in `save_run()`/`load_latest()`.

3. **Probe for a concrete counterfactual** (`_suggest`): for every valid
   `(day, block)`, collect scheduled courses that both overlap it *and*
   share a resource (lecturer id / group id / room name). If evicting
   exactly those frees the slot, report the smallest such set:
   `bebaskan X, Y, Z dari Senin slot 7–9`. Otherwise fall back to blocker
   1's generic advice. Pure Python over the already-extracted schedule —
   no extra solver call.

4. **Carry card metadata in the payload** (`credits`,
   `room_type_required`, `rooms[]`, `lecturers[]`, `student_groups[]`) so
   the UI prints `MK-407 · 3 SKS · LAB · Ruang: ...` without joining other
   tables.

**Contract:** `GET /diagnose` → `{run_id, status, items: List[dict]}`;
`items` = the `dropped[]` cards when PARTIAL, or `[{"finding": str}]`
(full-model `diagnose()`) when INFEASIBLE/UNKNOWN. CLI prints the same
cards (metadata line + `Alasan:` + `Saran:`).

The full-model `diagnose()` findings (entity-level, when no schedule
exists):

Solve is capped at 30s with a live progress callback (each improving
solution prints), so long runs never look hung. The best solution found is
always returned.

### Diagnosis cards

Every dropped course carries everything the UI needs without joins:
`course_id`, `course_code`, `course_name`, `credits`, `room_type_required`,
`rooms[]`, `lecturers[]`, `student_groups[]`, `reason`, `suggestion`.

- **`reason`** — structural checks first (no compatible room, no valid
  day/block from lecturer availability), then all remaining blockers
  collected: co-lecturer fully booked, group fully booked, no room free,
  or "each resource free somewhere, never together" (with option count).
- **`suggestion`** — counterfactual probe over the extracted schedule:
  for every valid (day, block), find scheduled courses that overlap it
  AND share a resource; if evicting them frees the slot, report the
  smallest such set (`bebaskan X, Y, Z dari Senin slot 7–9`). Falls back
  to generic advice naming the JSON file to edit. Pure Python — no extra
  solver call.
- **Full-model `diagnose()`** — separate path, only when INFEASIBLE/UNKNOWN
  and no partial schedule exists; returns entity-level findings.

API: `GET /diagnose` → `{run_id, status, items}` where `items` is the
card list (PARTIAL) or `[{finding}]` (INFEASIBLE/UNKNOWN).

### Timeline encoding

Each class becomes one interval on an absolute timeline:

```
T = day * 10 + offset(slot)

slot s → offset s-1 (slots 1-9 → 0-8, no lunch break)
```

1 day = 10 units, 1 week = 60 units. An N-SKS course = interval
`[start, start+N)`.

### Constraint encoding (AddNoOverlap)

Conflicts are not compared pairwise, but through the native CP-SAT
`AddNoOverlap` primitive:

- **Lecturers** → group intervals per lecturer → `AddNoOverlap`
- **Students** → group intervals per group → `AddNoOverlap`
- **Rooms** → optional interval per (course, compatible room), presence bool
  tied to the room variable → `AddNoOverlap` per room

The solver processes `AddNoOverlap` with a dedicated propagator — far more
efficient than hundreds of thousands of hand-rolled bool clauses.

### Objective

```
Maximize 1_000_000 * scheduled_courses
       - 100 * days_used
       - 10  * sum(day_index)
       -     * sum(slot_index)
```

Scheduled count dominates (skip as few courses as possible), then compact
schedules (few days), earliest days first (Monday before Friday), earliest
slots within a day.

---

## Benchmark

Deterministic seed `SSEED = 20261003`, solver capped at 30s, every result
re-validated by the independent validator.

| Courses | Lecturers | Rooms | Time | Status | Valid |
|---|---|---|---|---|---|
| 10 | 5 | 4 | ~0.2s | FEASIBLE | VALID |
| 25 | 8 | 6 | ~4s | FEASIBLE | VALID |
| 50 | 12 | 8 | 30s cap | FEASIBLE/PARTIAL | VALID |
| 80 | 16 | 8 | 30s cap | FEASIBLE/PARTIAL | VALID |

Impact of the pairwise → `AddNoOverlap` refactor (commit `d44e353`):

| Courses | Before | After |
|---|---|---|
| 10 | 8,248 | 743 |
| 25 | 91,749 | 1,586 |
| 50 | 488,612 | 3,513 |
| 80 | 1,397,302 → **UNKNOWN** (timeout) | 5,648 → **FEASIBLE + VALID** |

Constraints dropped ~250x at 80 courses. 50/80 sit near the time cap because
the solver keeps optimizing the objective, not because feasibility is slow.

---

## Testing

**15 unit tests** (`tests/test_scheduler.py`) — all pass:

- 3 conflicts (lecturer, room, student) + 2 negative cases (different groups
  MAY overlap)
- Team teaching (2 lecturers per course: conflict + 2nd lecturer's availability)
- Cross-semester conflict (IF-3 vs IF-5 both taken by IF-5)
- Consecutive 4-SKS slots (valid [1-4]..[6-9]; crossing 5→6 also valid)
- Lecturer availability (day-only → INFEASIBLE)
- Room type (lab course → COMPUTER_LAB)
- Infeasible scenario (10 courses × 4 SKS, 1 room) → PARTIAL/INFEASIBLE with
  a shown schedule and a reason for every skipped course

**Independent validator** (`validate_schedule.py`) — pure Python, does not
trust the solver. Re-runs every hard constraint on `schedule.json`. Exit 0
valid, exit 1 invalid.

---

## Bugs & Insights Found During Development

1. **One-way `OnlyEnforceIf`** — `AddBoolAnd([...]).OnlyEnforceIf(x)` only
   encodes `x → AND`, not `AND → x`. Without the reverse clause the
   constraint is vacuous. Every bool variable needs both True and False cases.
2. **All classes started 08:15 in the same room** — root cause was the
   vacuous room-conflict constraint (bug above). Took several iterations to
   find.
3. **Schedule spread over 5 days when 2 sufficed** — no objective. Added the
   objective, then an earliest-day tie-break (Monday > Friday).
4. **Benchmark 50/80 INFEASIBLE** — the generator created random availability
   that fit no block; one broken lecturer killed the whole schedule. Fixed
   with load balancing + ≥2 full days per lecturer.
5. **Pairwise encoding exploded** — O(n²) pairs × several clauses = 1.4M
   constraints at 80 courses, solver timed out. Replaced with `AddNoOverlap`
   → 5,648.
6. **Single-drop probe loop got UNKNOWN** — removing one course at a time
   never isolated the conflict; the fix needed multiple drops. Replaced with
   reified `sched` booleans so CP-SAT picks the skipped set in one solve.

---

## Roadmap

- **Phase 4**: SQLite data layer + FastAPI endpoints (done — see
  `uvicorn main:app`)
- **Phase 5**: dashboard CRUD + trigger scheduler + schedule visualization
- Next soft constraints: lecturer preferences, even distribution, gaps
  between classes
- Tiered constraint relaxation (PRD §17)
