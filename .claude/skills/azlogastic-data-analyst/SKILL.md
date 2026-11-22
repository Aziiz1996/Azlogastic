---
name: azlogastic-data-analyst
description: "Azlogastic (Leader Study / عز اللوجستيات) data analysis skill. Provides context for querying the production leader.sqlite database, including entity definitions, the users.data JSON document model, metric calculations, Arabic terminology, and common query patterns. Use when analyzing Azlogastic or Leader Study data for: (1) student signup, phone-verification and owner-approval funnels, (2) study, practice and quiz performance, (3) per-student finance and career-application tracking, or any data question requiring Azlogastic-specific context."
---

# Azlogastic Data Analysis

Azlogastic (product name **Leader Study** / **عز اللوجستيات**) is an Arabic-language
study platform for logistics and supply-chain students. One Node server
(`server/index.mjs`) serves a React app and a JSON API, and stores everything in a
single SQLite file.

## SQL Dialect: SQLite (`node:sqlite`, WAL mode)

The database is `leader.sqlite` under `LEADER_DATA_DIR` (production: `/var/data/leader`).
There is no warehouse, no `INFORMATION_SCHEMA`, and no separate analytics replica.

```sql
-- Inspect the schema
SELECT name, sql FROM sqlite_master WHERE type='table' ORDER BY name;

-- Column details for one table
SELECT name, type, "notnull", pk FROM pragma_table_info('users');
```

Dialect notes that matter here:

| Need | SQLite form |
|------|-------------|
| Read a field from a JSON column | `json_extract(data, '$.settings.currency')` |
| Expand a JSON array into rows | `FROM users, json_each(users.data, '$.tasks')` |
| Epoch **milliseconds** → timestamp | `datetime(expires/1000, 'unixepoch')` |
| Now, in epoch ms (to match stored values) | `unixepoch()*1000` |
| Month bucket from an ISO date string | `substr(date, 1, 7)` |
| Safe division | `1.0*num/NULLIF(den, 0)` |
| Boolean columns | Integers `0`/`1` — there is no BOOLEAN type |

No `DATE`/`TIMESTAMP`/`BOOLEAN` types, no window-function-free fallbacks needed
(SQLite ≥ 3.25 has window functions), and string comparison is case-sensitive for
non-ASCII — which matters for Arabic (see Gotcha 4 below).

### Query the database read-only

`leader.sqlite` is live production data in WAL mode. Always open it read-only, and
never analyse a raw `cp` of an open WAL database — use `sqlite3 .backup` or
`VACUUM INTO`.

```bash
sqlite3 "file:/var/data/leader/leader.sqlite?mode=ro" -readonly ".backup /tmp/leader-snapshot.sqlite"
```

---

## Entity Disambiguation

**"User" is never just one thing here:**

- **Account** — a login row in `users`. `users.id` (TEXT) is the one identifier every
  other table joins on. `users.username` is the **email address** used to log in and is
  `UNIQUE`.
- **Owner** — `users.owner = 1`. The platform operator (the instructor), not a student.
  There is normally exactly one. Owners have no student record and are **rejected** by
  the approval service, so they must be excluded from every student metric.
- **Student** — an account with `users.owner = 0`. A student's *approval* lives in
  `student_access`, and a row there may not exist yet (see Gotcha 2).
- **Handle** — `account_details.handle`, a separate `UNIQUE` public identifier chosen at
  signup. The same human therefore has **two unique keys**: `username` (email) and
  `handle`. Do not assume one implies the other; `account_details` only exists for
  accounts that completed the phone-verified signup flow.
- **Group member** — `project_memberships.user_id` maps an account to a numbered project
  group (`group_number`). `group3_members` is a separate, older private-group table kept
  for Private Group 3 only; it is not a general membership table.

**Relationships:**

- `users` → `sessions`: 1:many (`sessions.user_id`), one row per live login
- `users` → `account_details`: 1:0-or-1 (`user_id`)
- `users` → `student_access`: 1:0-or-1 (`user_id`), current approval state
- `users` → `student_access_audit`: 1:many (`user_id`), append-only decision history
- `users` → `course_summaries`: 1:many (`user_id`, `course_id`)
- `users` → `project_memberships`: 1:0-or-1 → `project_documents` / `project_reviews` on
  `group_number`
- Courses, lectures, tasks, notes, transactions, study logs and practice attempts are
  **not tables** — they live inside the `users.data` JSON document (see
  `references/tables/academics.md`).

---

## The central fact: `users.data` is the whole app

`users.data` is a `TEXT NOT NULL` column holding one JSON document per account
(the `AppData` shape). Most questions people ask — "how many hours did students study",
"what are students spending on", "how many tasks are overdue" — are answered by
unnesting that column, not by joining tables.

```sql
-- Study minutes per student, from the JSON study log
SELECT u.id,
       COUNT(*)                        AS sessions,
       SUM(json_extract(log.value, '$.minutes')) AS minutes
FROM users u, json_each(u.data, '$.studyLog') AS log
WHERE u.owner = 0
  AND COALESCE(json_extract(u.data, '$.demo'), 0) = 0
GROUP BY u.id;
```

Arrays that may be **absent** rather than empty: `studyLog`, `practiceAttempts`,
`quizzes`, `applications`, `opportunityLinks`, `projectGroups`, `incomeRules`,
`universityEvents`, `studentProfile`. `json_each` over a missing path yields no rows,
which is usually what you want — but it means `COUNT(*) = 0` and "key missing" are
indistinguishable. Always report students with no rows via a `LEFT JOIN` from `users`.

Always present: `version`, `courses`, `lectures`, `tasks`, `notes`, `transactions`,
`settings`.

---

## Standard Filters

Apply these to every student-facing query unless told otherwise:

```sql
WHERE u.owner = 0                                              -- exclude the owner/operator
  AND COALESCE(json_extract(u.data, '$.demo'), 0) = 0          -- exclude demo/seeded accounts
```

- `users.owner = 1` is the operator account. It holds app-wide config, not student data.
- `data.demo = true` is set by `demoData()` for seeded demo accounts. The key is **absent**
  on real accounts, so `json_extract(...) = 0` alone drops every real student —
  `COALESCE` is required.
- For "active students", also require a live session:
  `EXISTS (SELECT 1 FROM sessions s WHERE s.user_id = u.id AND s.expires > unixepoch()*1000)`.
  Expired session rows are **not** deleted, so an unfiltered `sessions` count is a count of
  all logins ever, not current ones.
- For approval questions, `LEFT JOIN student_access` and
  `COALESCE(sa.approval, 'pending')` — never `INNER JOIN` (Gotcha 2).

**When to override:**

- Auditing the signup/OTP pipeline: keep unverified and rejected accounts, that is the
  population under study.
- Counting licence or capacity usage: include demo accounts if they occupy real rows.

---

## Key Metrics

### Approved students
- **Definition**: student accounts the owner has approved for restricted content.
- **Formula**: `COUNT(DISTINCT u.id)` where `COALESCE(sa.approval,'pending') = 'approved'`
- **Source**: `users` LEFT JOIN `student_access`
- **Caveats**: approval is independent of phone verification. A student can be approved
  with an unverified phone and vice versa; `/api/student/restricted` requires **both**.

### Phone verification rate
- **Definition**: share of student accounts with a Twilio-verified phone on file.
- **Formula**: `1.0 * COUNT(ad.verified_phone) / NULLIF(COUNT(*), 0)`
- **Source**: `users` LEFT JOIN `account_details`
- **Caveats**: the service treats a **non-empty string** as verified, so test for
  `COALESCE(ad.verified_phone,'') <> ''`, not `IS NOT NULL`. Accounts created before the
  phone-verification release have no `account_details` row at all.

### Study minutes
- **Definition**: minutes of completed, logged study.
- **Formula**: `SUM(json_extract(log.value, '$.minutes'))` over `$.studyLog`
- **Time grain**: `json_extract(log.value, '$.date')` is a `YYYY-MM-DD` string; month =
  `substr(..., 1, 7)`
- **Caveats**: `$.activeStudy` is an **in-progress** run and is not in `studyLog`; do not
  add the two. `studyLog[].score` is a self-reported 0–100 reflection score, not a quiz grade.

### Practice / exam score
- **Definition**: correct answers out of questions served in one attempt.
- **Formula**: `score / total`, surfaced in the app as
  `ROUND(100.0 * score / NULLIF(total,0))`
- **Source**: `$.practiceAttempts[]` in `users.data`
- **Caveats**: `mode` is `'practice'` or `'exam'` — never mix them in one average. Skipped
  questions count toward `total` but not `score`, so a low percent can mean "abandoned",
  not "wrong". `$.activePractice` is an unfinished run with no score yet.

### Monthly balance
- **Definition**: income minus expense for one calendar month, per student.
- **Formula**: `SUM(CASE kind WHEN 'income' THEN amount ELSE -amount END)`
  over `$.transactions` where `substr(date,1,7) = :month`
- **Caveats**: the app ignores transactions with `amount <= 0` or a malformed date, and
  computes in integer cents — mirror that with `ROUND(SUM(...)*100)/100.0` to avoid float
  drift. Expense **categories** are built from expenses only, with `'أخرى'` ("Other") as
  the fallback for a blank category. Currency is per-student (`$.settings.currency`), so
  **never sum amounts across students** without grouping by currency.

### Application funnel
- **Definition**: career/course applications by status.
- **Formula**: `COUNT(*)` over `$.applications[]` grouped by `$.status`
- **Statuses**: `saved` → `preparing` → `submitted` → `accepted` / `rejected`
- **Caveats**: status is a current state with no history; a rejected application was
  previously submitted but nothing records when.

---

## Data Freshness

Everything is written synchronously by the app, so there is no ETL lag — but "last
updated" differs per table and **two different time encodings are in use**.

| Table / field | Encoding | Notes |
|---|---|---|
| `sessions.expires` | epoch **ms** | login + 7 days |
| `signup_verifications.expires` | epoch **ms** | 10-minute OTP window |
| `otp_limits.expires` | epoch **ms** | rate-limit window |
| `student_access_audit.at` | epoch **ms** | decision time |
| `course_summaries.updated_at` | **ISO 8601 string** | AI summary generation |
| `project_documents.updated_at` | **ISO 8601 string** | group upload |
| `student_evidence.submitted_at` | **ISO 8601 string** | enrolment proof upload |
| dates inside `users.data` | `YYYY-MM-DD` / `HH:MM` strings | local, no timezone |

```sql
-- Latest activity, both encodings normalised to UTC text
SELECT MAX(datetime(at/1000,'unixepoch')) AS last_approval FROM student_access_audit;
SELECT MAX(updated_at)                    AS last_summary  FROM course_summaries;
```

There is **no `created_at` on `users`**. Account age is not recoverable from the row; the
earliest `student_access_audit.at` or the signup verification record is the closest proxy.

---

## Knowledge Base Navigation

| Domain | Reference File | Use For |
|--------|----------------|---------|
| Accounts & auth | `references/tables/accounts.md` | `users`, `sessions`, `account_details`, `signup_verifications`, `otp_limits`, `recovery_codes` |
| Student lifecycle | `references/tables/student-lifecycle.md` | `student_access`, `student_access_audit`, `student_evidence`, approval funnel |
| Academics | `references/tables/academics.md` | `users.data` JSON: courses, lectures, tasks, notes, study log, practice attempts; `course_summaries` |
| Finance | `references/tables/finance.md` | `users.data` transactions and income rules, budgets, currency |
| Groups & projects | `references/tables/groups.md` | `project_memberships`, `project_documents`, `project_reviews`, `group3_*`, `app_design`, `owner_backups` |
| Entities | `references/entities.md` | Entity definitions, ID fields, join keys |
| Metrics | `references/metrics.md` | Full KPI formulas with runnable SQL |
| Terminology | `references/glossary.md` | Arabic ↔ English business terms and stored values |

Not covered by this skill: the separate FastAPI/Firebase backend in `python_backend/`
(`courses/{id}/documents`, `courses/{id}/quizzes`, `users/{uid}/quiz_attempts`,
`logistics_terms`). It is a Firestore document store with its own IDs and does not join to
`leader.sqlite`.

---

## Common Query Patterns

### Signup → verification → approval funnel
```sql
SELECT COUNT(*)                                                        AS students,
       SUM(COALESCE(ad.verified_phone,'') <> '')                       AS phone_verified,
       SUM(COALESCE(sa.approval,'pending') = 'pending')                AS pending,
       SUM(COALESCE(sa.approval,'pending') = 'approved')               AS approved,
       SUM(COALESCE(sa.approval,'pending') = 'rejected')               AS rejected,
       SUM(COALESCE(ad.verified_phone,'') <> ''
           AND COALESCE(sa.approval,'pending') = 'approved')           AS fully_eligible
FROM users u
LEFT JOIN account_details ad ON ad.user_id = u.id
LEFT JOIN student_access  sa ON sa.user_id = u.id
WHERE u.owner = 0
  AND COALESCE(json_extract(u.data,'$.demo'),0) = 0;
```

### Study minutes by month
```sql
SELECT substr(json_extract(log.value,'$.date'),1,7)      AS month,
       COUNT(DISTINCT u.id)                              AS students,
       SUM(json_extract(log.value,'$.minutes'))          AS minutes
FROM users u, json_each(u.data,'$.studyLog') AS log
WHERE u.owner = 0
  AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
  AND json_extract(log.value,'$.date') GLOB '[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]'
GROUP BY month
ORDER BY month;
```

### Exam performance distribution
```sql
SELECT CASE WHEN pct >= 90 THEN '90-100'
            WHEN pct >= 75 THEN '75-89'
            WHEN pct >= 50 THEN '50-74'
            ELSE 'under 50' END                          AS band,
       COUNT(*)                                          AS attempts
FROM (
  SELECT ROUND(100.0 * json_extract(a.value,'$.score')
               / NULLIF(json_extract(a.value,'$.total'),0)) AS pct
  FROM users u, json_each(u.data,'$.practiceAttempts') AS a
  WHERE u.owner = 0
    AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
    AND json_extract(a.value,'$.mode') = 'exam'
    AND json_extract(a.value,'$.total') > 0
)
GROUP BY band ORDER BY MIN(pct) DESC;
```

### Approval decision history (who decided what, when)
```sql
SELECT a.user_id,
       a.approval,
       a.actor,                                    -- deciding account id, not a name
       datetime(a.at/1000,'unixepoch')             AS decided_at_utc
FROM student_access_audit a
JOIN users u ON u.id = a.user_id AND u.owner = 0
ORDER BY a.at DESC
LIMIT 50;
```

### Group project review status
```sql
SELECT pm.group_number,
       COUNT(DISTINCT pm.user_id)                       AS members,
       COALESCE(pr.status,'pending')                    AS review_status,
       CASE WHEN pd.group_number IS NULL THEN 0 ELSE 1 END AS has_document,
       pd.updated_at
FROM project_memberships pm
LEFT JOIN project_reviews   pr ON pr.group_number = pm.group_number
LEFT JOIN project_documents pd ON pd.group_number = pm.group_number
GROUP BY pm.group_number, pr.status, pd.group_number, pd.updated_at
ORDER BY pm.group_number;
```

---

## Privacy rules

This database is personal data about identifiable students, and the reports it feeds are
not anonymous by default.

- **Direct identifiers**: `users.username` (email), `account_details.verified_phone`,
  `account_details.birthdate`, `account_details.handle`, and
  `users.data -> $.studentProfile` (full name, email, phone, city, university, CV and
  photo as base64 `dataUrl` attachments). Student full names also appear in the project
  group catalogue shipped in the app source.
- **Credentials — never select**: `users.salt`, `users.password_hash`, `sessions.hash`,
  `recovery_codes.hash`, `signup_verifications.hash`, `signup_verifications.sid`. These
  are secrets, not analysis columns, and a `SELECT *` on `users` exposes the password hash.
- **Report aggregates, not rosters.** Default to counts and distributions grouped by
  cohort, group number or status. Produce a per-student list only when the owner asks for
  one, and identify students by `users.id`, not by email or phone.
- Reports derived from this database are local artifacts. Do not upload them, and keep
  them out of git — the same rule the repos apply to `reports/`.

---

## Troubleshooting

### Common Mistakes
- **`SELECT *` on `users`** — returns `password_hash`, `salt`, and a very large `data`
  blob. → Name the columns you need and pull JSON fields with `json_extract`.
- **`SELECT *` on a document table** — `project_documents.bytes`,
  `student_evidence.bytes` and `group3_document.bytes` are BLOBs that will flood the
  output. → Use `length(bytes) AS size_bytes`.
- **Treating epoch-ms columns as seconds** — `datetime(expires,'unixepoch')` returns a
  year in the distant future. → `datetime(expires/1000,'unixepoch')`.
- **`INNER JOIN student_access`** — silently drops every student who has never been
  reviewed, which is the entire pending population. → `LEFT JOIN` +
  `COALESCE(approval,'pending')`.
- **Counting `sessions` as active users** — expired rows persist. → filter
  `expires > unixepoch()*1000`.
- **Averaging practice and exam attempts together** — different `mode`, different
  meaning. → always filter or group by `$.mode`.
- **Summing money across students** — each student sets their own
  `$.settings.currency`. → group by currency, or scope to one student.
- **Reading `student_access` for history** — it holds only current state. → use
  `student_access_audit` for the timeline.
- **`json_array_length` on a JSON object returns 0, not the key count** — `$.practiceAttempts[].answers` is an object. → `(SELECT COUNT(*) FROM json_each(a.value,'$.answers'))`.

### Arabic text matching (Gotcha 4)
The app normalises Arabic before comparing course names: NFKC, then `أ إ آ → ا`, then
strips tatweel (`ـ`) and diacritics (`ً-ٟ`). SQLite does none of this, so an
exact `=` or `LIKE` on a course name will miss rows that differ only by hamza or a
diacritic. Prefer matching on a stable id (`$.courses[].id`, e.g.
`'inventory-management'`), and when you must match text, replace the variants yourself:

```sql
WHERE replace(replace(replace(replace(json_extract(c.value,'$.name'),
        'أ','ا'),'إ','ا'),'آ','ا'),'ـ','') LIKE '%ادارة المخزون%'
```

### Access Issues
- The database is on the Render persistent disk at `/var/data/leader`, reachable only from
  that host. Analyse a `.backup` snapshot, not the live file over a network mount.
- `node:sqlite` requires Node ≥ 24 (`engines` in `package.json`); the `sqlite3` CLI reads
  the same file and is fine for analysis.
- WAL mode means a plain file copy can miss committed data. Use
  `.backup` / `VACUUM INTO`.
- Run exactly one writer. The deploy notes warn that a second server instance on a
  separate disk gets separate accounts and separate rate limits.
