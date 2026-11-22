# Academics: the `users.data` JSON document

Courses, lectures, tasks, notes, study logs and practice attempts are **not tables**.
They are arrays inside `users.data`, one document per account, validated on write by
`validateData()` in `src/validation.ts`. `course_summaries` is the one real table here.

## Business Context

Each student keeps their own timetable, task list, notes and study history in the app.
Shared academic content (a common course, the college calendar, a provided question bank)
is **merged into the document at read time** by the server, so the same course can appear
with different colours or resources for different students.

## Standard Filters

```sql
WHERE u.owner = 0
  AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
```

---

## Document shape

`users.data` is one JSON object (`AppData`, `version: 1`).

**Always present**: `version`, `courses`, `lectures`, `tasks`, `notes`, `transactions`,
`settings`.

**Optional — may be absent rather than empty**: `studentProfile`, `opportunityLinks`,
`applications`, `activeStudy`, `studyLog`, `activePractice`, `practiceAttempts`,
`quizzes`, `demo`, `profile`, `projectGroups`, `incomeRules`, `universityEvents`.

`json_each` over a missing path returns **no rows**, so a student with no study history
and a student with an empty array are indistinguishable. Always `LEFT JOIN` from `users`
so students with nothing still appear with a zero.

```sql
-- Students including those with no study history
SELECT u.id, COALESCE(SUM(json_extract(log.value,'$.minutes')), 0) AS minutes
FROM users u
LEFT JOIN json_each(u.data,'$.studyLog') AS log
WHERE u.owner = 0 AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
GROUP BY u.id;
```

---

## `$.courses[]` — Course

| Field | Type | Description | Notes |
|---|---|---|---|
| **id** | string | Course identifier, e.g. `'inventory-management'` | **Per-student**; shared courses share an id by convention |
| **name** | string | Arabic course name | Hamza/diacritic variants — match on `id`, not `name` |
| color | string | UI colour | |
| code | string? | Course code | Optional |
| resources | array? | `{id, title, url, kind, section}` | Merged from shared content |

Referenced by `$.lectures[].courseId`, `$.tasks[].courseId`, `$.notes[].courseId`,
`$.universityEvents[].courseId`, and `course_summaries.course_id`. `validateData()`
rejects a document whose lectures reference an unknown `courseId`, so these references
are internally consistent — but only **within one student's document**.

```sql
-- Course adoption across students
SELECT json_extract(c.value,'$.id')   AS course_id,
       json_extract(c.value,'$.name') AS course_name,
       COUNT(DISTINCT u.id)           AS students
FROM users u, json_each(u.data,'$.courses') AS c
WHERE u.owner = 0 AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
GROUP BY course_id, course_name
ORDER BY students DESC;
```

The same course may appear under slightly different `name` spellings with the same `id` —
group by `id` and take any one name.

---

## `$.lectures[]` — Lecture (timetable slot)

| Field | Type | Description | Notes |
|---|---|---|---|
| **courseId** | string | → `$.courses[].id` | Validated |
| **day** | integer 0–6 | **0 = الأحد (Sunday)** … 6 = السبت | Saudi week; not ISO |
| start / end | `HH:MM` | Slot times, `end > start` | Strings; compare lexically |
| location | string | Room or campus | |
| attendanceUrl / fileUrl | string? | Must pass the university-URL allowlist | |
| timePending | string? | Marks a slot whose time is unconfirmed | Present ⇒ time not final |

**Blank weekday placeholders are filtered out at read time** (`isLessonTemplate()`), so
the timetable the student sees is smaller than the raw array. If you count lectures
straight from JSON you will include template rows the app hides. Exclude slots with no
real time:

```sql
SELECT json_extract(l.value,'$.day') AS weekday, COUNT(*) AS slots
FROM users u, json_each(u.data,'$.lectures') AS l
WHERE u.owner = 0 AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
  AND COALESCE(json_extract(l.value,'$.start'),'') <> ''
  AND json_extract(l.value,'$.timePending') IS NULL
GROUP BY weekday ORDER BY weekday;
```

---

## `$.tasks[]` — Task

| Field | Type | Description | Notes |
|---|---|---|---|
| **kind** | string | `assignment` / `project` / `exam` | `project` here ≠ project **group** |
| **due** | `YYYY-MM-DD` | Due date | Local, **no timezone** |
| **priority** | string | `low` / `medium` / `high` | |
| **minutes** | number | Student's own workload estimate | Estimate, not actual time spent |
| **completed** | boolean | Done flag | JSON boolean → `0`/`1` in SQLite |
| time / endTime / timePending | string? | Optional scheduling | |

Dates are local strings with no timezone, so `date('now')` (UTC) can be a day off from
what the student sees. For an owner-facing overdue count that is close enough, compare to
`date('now')`; flag the caveat when the number is near a boundary.

---

## `$.notes[]` — Note

`{id, title, courseId, content, summary, updatedAt}`. `summary` is **AI-generated** by
the server (`OPENAI_SUMMARY_MODEL`), so an empty `summary` with a long `content` means
summarisation never ran or failed. `content` is free-text student writing — treat as
personal content, report lengths and counts rather than text.

---

## `$.studyLog[]` — completed study session

| Field | Type | Description | Notes |
|---|---|---|---|
| **taskId** | string | Task studied | May reference a deleted task |
| **date** | `YYYY-MM-DD` | Session date | Month = `substr(date,1,7)` |
| **minutes** | number | Minutes studied | The study-time metric |
| **score** | number | **Self-reported** reflection score 0–100 | Not a quiz grade |
| reflection | string | Free-text note | Personal content |
| **reviewDates** | string[] | Scheduled spaced-repetition dates | |
| **reviewedDates** | string[] | Reviews actually done | Backlog = length difference |

`$.activeStudy` is a **single in-progress run** (`StudyRun`), not part of `studyLog`.
It has `remainingSeconds`, `runningSince` and `elapsedSeconds` instead of `minutes`.
Never add it to logged minutes — it is unfinished and may be abandoned.

`$.studyLog[].taskId` can point at a task the student has since deleted, so an inner join
from log to task loses history. Left-join it.

**`StudyPlan` / `StudySession` are computed, never stored.** `buildStudyPlan()` derives
them at read time from tasks and settings, so "what was the plan last week" is not
answerable from the database.

---

## `$.practiceAttempts[]` — finished practice or exam attempt

| Field | Type | Description | Notes |
|---|---|---|---|
| **bankId** | string | Question bank used | |
| **questionIds** | string[] | Questions served | `length` ≈ `total` |
| **answers** | object | `{questionId: answer}` | Only **answered** questions appear |
| **mode** | string | `practice` / `exam` | Never average across modes |
| **score** | number | Correct answers | |
| **total** | number | Questions served | Includes skipped |
| startedAt / finishedAt | string | Timestamps | |

Percent as the app computes it: `ROUND(100.0 * score / NULLIF(total,0))`.

Skipped questions count toward `total` but not `score`, so compare
the **number of keys** in `answers` against `total` to tell a weak attempt from an abandoned
one. `answers` is a JSON **object**, so count it with `json_each`, not
`json_array_length` (which returns 0 for an object):

```sql
SELECT json_extract(a.value,'$.score')                              AS score,
       json_extract(a.value,'$.total')                              AS served,
       (SELECT COUNT(*) FROM json_each(a.value,'$.answers'))        AS answered
FROM users u, json_each(u.data,'$.practiceAttempts') AS a
WHERE u.owner = 0 AND COALESCE(json_extract(u.data,'$.demo'),0) = 0;
```

`$.activePractice` is an **unfinished** run (`PracticeRun`) with `currentIndex` and no
score. Exclude it from performance metrics.

`$.quizzes[]` holds teacher-imported question sets (`ImportedQuiz`) — content, not
results. Each question carries `correct` as an **index** into `options`.

---

## `course_summaries` (real table)

**Location**: `course_summaries`
**Description**: AI-generated per-course summaries, cached server-side.
**Primary Key**: (`user_id`, `course_id`)

| Column | Type | Description | Notes |
|---|---|---|---|
| **user_id** | TEXT NOT NULL | → `users.id` | |
| **course_id** | TEXT NOT NULL | → `$.courses[].id` **in that student's document** | Not a global course table |
| summary_json | TEXT NOT NULL | Generated summary payload | JSON; use `json_extract` |
| **updated_at** | TEXT NOT NULL | **ISO 8601 string** | Not epoch ms |

Upserted on conflict, so there is **one row per student-course and no history** — a
regenerated summary overwrites the previous one and its timestamp.

```sql
-- Summary coverage: courses in the document vs summaries generated
SELECT u.id,
       COUNT(DISTINCT json_extract(c.value,'$.id')) AS courses,
       COUNT(DISTINCT cs.course_id)                 AS summarised,
       MAX(cs.updated_at)                           AS last_summary
FROM users u
LEFT JOIN json_each(u.data,'$.courses') AS c
LEFT JOIN course_summaries cs ON cs.user_id = u.id
WHERE u.owner = 0 AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
GROUP BY u.id;
```

A `course_summaries` row can outlive the course it describes — if the student deletes the
course from their document, the summary row stays. Join from the document, not the table,
when counting live courses.

---

## `$.universityEvents[]` and `$.settings`

`$.universityEvents[]`: `{id, title, date, time, endDate?, endTime?, kind, description,
url?, courseId?}` with `kind` ∈ `lecture` / `assignment` / `exam` / `event`. Partly
merged from the shared college calendar, so the same event appears in many students'
documents — **deduplicate by `id` before counting institution-level events.**

`$.settings`: `{name, currency, monthlyBudget, dailyMinutes, studyStart, studyDays[],
universityName?, universityUrl?}`. `studyDays` uses the same 0–6 Sunday-first encoding as
lectures. `name` and `universityName` are free text and may identify the student.

---

## Common Gotchas

1. **Querying academics as tables.** There is no `courses` or `tasks` table.
   - Wrong: `SELECT * FROM tasks`
   - Right: `FROM users u, json_each(u.data,'$.tasks') AS t`
2. **Comma-join hides students with no rows.** `FROM users, json_each(...)` is an inner
   join. Use `LEFT JOIN json_each(...)` when the zero matters.
3. **Adding `$.activeStudy` to `$.studyLog` minutes**, or `$.activePractice` to attempts.
   Both are unfinished.
4. **Averaging `practice` and `exam` attempts together.** Different `mode`, different
   meaning.
5. **Reading `$.studyLog[].score` as a grade.** It is a self-reported reflection score.
6. **Matching course names as Arabic text.** The app normalises hamza and diacritics
   before comparing; SQLite does not. Match on `$.courses[].id`.
7. **Counting shared university events per document.** They are duplicated across
   students by design — dedupe by event `id`.
8. **Weekday 0 is Sunday**, not Monday. A Mon-first assumption shifts the whole timetable.
9. **`json_array_length` on a JSON object silently returns 0.** `$.practiceAttempts[].answers`
   is an object keyed by question id, not an array.
   - Wrong: `json_array_length(json_extract(a.value,'$.answers'))` → always `0`
   - Right: `(SELECT COUNT(*) FROM json_each(a.value,'$.answers'))`
   `$.reviewDates`, `$.reviewedDates`, `$.quizzes`, `$.questionIds` and
   `$.studentProfile.skills` **are** arrays, so `json_array_length` is correct for those.
