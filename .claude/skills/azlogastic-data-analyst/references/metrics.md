# Metrics

Runnable SQLite against `leader.sqlite`. Every student query carries the standard
filters; they are written out in full here so each block can be copied and run alone.

Shared prelude used below:

```sql
-- students := real, non-owner, non-demo accounts
WITH students AS (
  SELECT id, data
  FROM users
  WHERE owner = 0
    AND COALESCE(json_extract(data,'$.demo'),0) = 0
)
```

---

## Accounts and access

| Metric | Definition | Formula | Notes |
|---|---|---|---|
| Student accounts | Non-owner, non-demo accounts | `COUNT(*)` over `students` | No `created_at`, so this cannot be trended over time |
| Phone-verified | Twilio-verified phone on file | `COUNT(*)` where `COALESCE(ad.verified_phone,'') <> ''` | Missing `account_details` row = not verified |
| Approved | Owner approved the student | `COALESCE(sa.approval,'pending') = 'approved'` | |
| Pending | Never reviewed, or explicitly pending | `COALESCE(sa.approval,'pending') = 'pending'` | **Missing row counts as pending** |
| Rejected | Owner rejected the student | `COALESCE(sa.approval,'pending') = 'rejected'` | Reversible |
| Fully eligible | Can reach `/api/student/restricted` | approved **AND** phone-verified | The two conditions are independent |
| Approval rate | Approved ÷ reviewed | `1.0*approved/NULLIF(approved+rejected,0)` | Excludes never-reviewed students |
| Live sessions | Unexpired logins | `COUNT(*)` where `expires > unixepoch()*1000` | Expired rows are never deleted |

```sql
WITH students AS (
  SELECT id, data FROM users
  WHERE owner = 0 AND COALESCE(json_extract(data,'$.demo'),0) = 0
)
SELECT COUNT(*)                                              AS student_accounts,
       SUM(COALESCE(ad.verified_phone,'') <> '')             AS phone_verified,
       SUM(COALESCE(sa.approval,'pending') = 'pending')      AS pending,
       SUM(COALESCE(sa.approval,'pending') = 'approved')     AS approved,
       SUM(COALESCE(sa.approval,'pending') = 'rejected')     AS rejected,
       SUM(COALESCE(sa.approval,'pending') = 'approved'
           AND COALESCE(ad.verified_phone,'') <> '')         AS fully_eligible,
       ROUND(100.0 * SUM(COALESCE(sa.approval,'pending')='approved')
             / NULLIF(SUM(COALESCE(sa.approval,'pending') IN ('approved','rejected')),0), 1)
                                                             AS approval_rate_pct
FROM students s
LEFT JOIN account_details ad ON ad.user_id = s.id
LEFT JOIN student_access  sa ON sa.user_id = s.id;
```

### Time to approval

The only timestamp in the lifecycle is the decision itself, so "time to approval" can
only be measured between **decisions**, not from signup.

```sql
SELECT a.user_id,
       MIN(datetime(a.at/1000,'unixepoch'))                        AS first_decision_utc,
       MAX(datetime(a.at/1000,'unixepoch'))                        AS last_decision_utc,
       COUNT(*)                                                    AS decisions,
       ROUND((MAX(a.at)-MIN(a.at))/86400000.0, 2)                   AS days_between
FROM student_access_audit a
JOIN users u ON u.id = a.user_id AND u.owner = 0
GROUP BY a.user_id
HAVING COUNT(*) > 1
ORDER BY days_between DESC;
```

### Students who changed state (reversals)

```sql
SELECT user_id, GROUP_CONCAT(approval, ' → ') AS path
FROM (SELECT user_id, approval FROM student_access_audit ORDER BY user_id, at)
GROUP BY user_id
HAVING COUNT(*) > 1;
```

---

## Study activity

| Metric | Definition | Formula | Notes |
|---|---|---|---|
| Study minutes | Logged minutes of completed study | `SUM($.studyLog[].minutes)` | Excludes `$.activeStudy` (in progress) |
| Study sessions | Count of completed runs | `COUNT(*)` over `$.studyLog` | One row per finished run |
| Avg session length | Minutes ÷ sessions | `1.0*SUM(minutes)/NULLIF(COUNT(*),0)` | |
| Self-rated score | Student's own reflection score | `AVG($.studyLog[].score)` | 0–100 **self-reported**, not a grade |
| Reviews due | Spaced-repetition dates not yet done | `$.studyLog[].reviewDates` minus `reviewedDates` | Both are arrays of `YYYY-MM-DD` |
| Active studiers | Students with any logged study in a window | `COUNT(DISTINCT id)` | There is no stored "active" flag |

```sql
-- Monthly study activity
SELECT substr(json_extract(log.value,'$.date'),1,7)           AS month,
       COUNT(DISTINCT u.id)                                   AS students,
       COUNT(*)                                               AS sessions,
       SUM(json_extract(log.value,'$.minutes'))               AS minutes,
       ROUND(AVG(json_extract(log.value,'$.minutes')), 1)     AS avg_minutes,
       ROUND(AVG(json_extract(log.value,'$.score')), 1)       AS avg_self_score
FROM users u, json_each(u.data,'$.studyLog') AS log
WHERE u.owner = 0
  AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
  AND json_extract(log.value,'$.date') GLOB '[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]'
GROUP BY month
ORDER BY month;
```

```sql
-- Overdue reviews per student (spaced repetition backlog)
SELECT u.id,
       SUM(json_array_length(COALESCE(json_extract(log.value,'$.reviewDates'),'[]'))
         - json_array_length(COALESCE(json_extract(log.value,'$.reviewedDates'),'[]'))) AS outstanding_reviews
FROM users u, json_each(u.data,'$.studyLog') AS log
WHERE u.owner = 0 AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
GROUP BY u.id
HAVING outstanding_reviews > 0
ORDER BY outstanding_reviews DESC;
```

---

## Practice and exam performance

| Metric | Definition | Formula | Notes |
|---|---|---|---|
| Attempt percent | Correct ÷ served, as the app shows it | `ROUND(100.0*score/NULLIF(total,0))` | Matches `gradePractice()` |
| Exam average | Mean percent over `mode='exam'` | `AVG(100.0*score/total)` | Never pool with practice mode |
| Completion proxy | Answered ÷ served | `(SELECT COUNT(*) FROM json_each(attempt,'$.answers'))` vs `$.total` | `$.answers` is an **object** — `json_array_length` returns 0 on it |
| Attempts per student | Volume of rehearsal | `COUNT(*)` over `$.practiceAttempts` | |
| Imported quizzes | Teacher-imported question sets | `json_array_length($.quizzes)` | Content, not results |

```sql
-- Per-student performance split by mode
SELECT u.id,
       json_extract(a.value,'$.mode')                                   AS mode,
       COUNT(*)                                                         AS attempts,
       ROUND(AVG(100.0 * json_extract(a.value,'$.score')
                 / NULLIF(json_extract(a.value,'$.total'),0)), 1)       AS avg_pct,
       MAX(json_extract(a.value,'$.finishedAt'))                        AS last_attempt
FROM users u, json_each(u.data,'$.practiceAttempts') AS a
WHERE u.owner = 0
  AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
  AND json_extract(a.value,'$.total') > 0
GROUP BY u.id, mode
ORDER BY u.id, mode;
```

**Caveat worth repeating in any report:** a low percent can mean the student abandoned
the attempt. Skipped questions are counted in `total` but not in `score`, so separate
"answered everything and scored low" from "walked away":

```sql
SELECT ROUND(100.0*json_extract(a.value,'$.score')/NULLIF(json_extract(a.value,'$.total'),0)) AS pct,
       (SELECT COUNT(*) FROM json_each(a.value,'$.answers'))                                  AS answered,
       json_extract(a.value,'$.total')                                                        AS served
FROM users u, json_each(u.data,'$.practiceAttempts') AS a
WHERE u.owner = 0 AND COALESCE(json_extract(u.data,'$.demo'),0) = 0;
```

---

## Coursework

| Metric | Definition | Formula | Notes |
|---|---|---|---|
| Open tasks | Not completed | `$.tasks[].completed = 0` | `completed` is a JSON boolean → `0`/`1` |
| Overdue tasks | Past due and not completed | `$.due < date('now')` and not completed | `due` is `YYYY-MM-DD`, local, no timezone |
| Planned minutes | Workload the student estimated | `SUM($.tasks[].minutes)` | Estimate, not actual |
| Tasks by kind | assignment / project / exam | group by `$.kind` | |
| High-priority backlog | Open and `priority = 'high'` | | |
| Summary coverage | Courses with an AI summary | `COUNT(*)` from `course_summaries` | One row per (`user_id`,`course_id`) |

```sql
-- Task backlog by kind and priority
SELECT json_extract(t.value,'$.kind')      AS kind,
       json_extract(t.value,'$.priority')  AS priority,
       COUNT(*)                            AS open_tasks,
       SUM(json_extract(t.value,'$.due') < date('now')) AS overdue,
       SUM(json_extract(t.value,'$.minutes'))           AS planned_minutes
FROM users u, json_each(u.data,'$.tasks') AS t
WHERE u.owner = 0
  AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
  AND json_extract(t.value,'$.completed') = 0
GROUP BY kind, priority
ORDER BY kind, priority;
```

---

## Finance (per student)

Mirrors `financeSummary()` in `src/logic.ts`, which ignores non-positive amounts and
malformed dates and computes in integer cents.

| Metric | Definition | Formula | Notes |
|---|---|---|---|
| Monthly income | Positive `income` rows in the month | `SUM(amount)` where `kind='income'` | |
| Monthly expense | Positive `expense` rows in the month | `SUM(amount)` where `kind='expense'` | |
| Balance | Income − expense | `income - expense` | Can be negative |
| Category split | Expense total by category | group by `category` | **Expenses only**; blank → `'أخرى'` |
| Budget variance | Expense vs target | `expense - $.settings.monthlyBudget` | Budget is a target, not actual |

```sql
-- One student's month, matching the app's arithmetic
SELECT substr(json_extract(x.value,'$.date'),1,7)                              AS month,
       json_extract(u.data,'$.settings.currency')                              AS currency,
       ROUND(SUM(CASE WHEN json_extract(x.value,'$.kind')='income'
                      THEN json_extract(x.value,'$.amount') ELSE 0 END)*100)/100.0 AS income,
       ROUND(SUM(CASE WHEN json_extract(x.value,'$.kind')='expense'
                      THEN json_extract(x.value,'$.amount') ELSE 0 END)*100)/100.0 AS expense,
       ROUND(SUM(CASE WHEN json_extract(x.value,'$.kind')='income'
                      THEN json_extract(x.value,'$.amount')
                      ELSE -json_extract(x.value,'$.amount') END)*100)/100.0        AS balance
FROM users u, json_each(u.data,'$.transactions') AS x
WHERE u.id = :user_id
  AND json_extract(x.value,'$.amount') > 0
  AND json_extract(x.value,'$.date') GLOB '[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]'
GROUP BY month, currency
ORDER BY month;
```

```sql
-- Expense categories for one student-month (expenses only, blank → أخرى)
SELECT COALESCE(NULLIF(TRIM(json_extract(x.value,'$.category')),''),'أخرى')     AS category,
       ROUND(SUM(json_extract(x.value,'$.amount'))*100)/100.0                   AS amount
FROM users u, json_each(u.data,'$.transactions') AS x
WHERE u.id = :user_id
  AND json_extract(x.value,'$.kind') = 'expense'
  AND json_extract(x.value,'$.amount') > 0
  AND substr(json_extract(x.value,'$.date'),1,7) = :month
GROUP BY category
ORDER BY amount DESC;
```

**Never aggregate money across students** without grouping by
`$.settings.currency` — each student sets their own.

---

## Career applications

| Metric | Definition | Formula | Notes |
|---|---|---|---|
| Applications | Saved + in-progress + submitted | `COUNT(*)` over `$.applications` | |
| Submission rate | Submitted or beyond ÷ all | status in (`submitted`,`accepted`,`rejected`) | |
| Acceptance rate | Accepted ÷ decided | `accepted/(accepted+rejected)` | Small denominators — report the counts too |
| Profile completeness | Fields filled in `$.studentProfile` | count of non-empty fields | **PII** — report the count, not the values |

```sql
SELECT json_extract(a.value,'$.kind')   AS kind,
       json_extract(a.value,'$.status') AS status,
       COUNT(*)                         AS applications,
       COUNT(DISTINCT u.id)             AS students
FROM users u, json_each(u.data,'$.applications') AS a
WHERE u.owner = 0 AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
GROUP BY kind, status
ORDER BY kind, status;
```

---

## Operational: OTP and rate limits

`otp_limits` is a live counter table, not a log — rows expire and are overwritten, so it
answers "are we near a limit right now", never "how many codes did we send last month".
There is no OTP send history.

| Limit | Value | Where |
|---|---|---|
| Resend interval, same phone | 60 s | enforced in the signup service |
| Sends per phone | 3 per 15 min | `otp_limits` |
| Sends per connection address | 5 per 15 min | `otp_limits` |
| Total sends, all users | `TWILIO_DAILY_SEND_LIMIT` (default 100) per 24 h | `otp_limits` |
| Code attempts per challenge | 5 | `signup_verifications.attempts` |
| Challenge lifetime | 10 min | `signup_verifications.expires` |

```sql
-- Current rate-limit pressure
SELECT key, count, datetime(expires/1000,'unixepoch') AS window_ends_utc
FROM otp_limits
WHERE expires > unixepoch()*1000
ORDER BY count DESC;

-- In-flight and failed challenges
SELECT state, COUNT(*) AS challenges, SUM(attempts) AS total_attempts,
       SUM(expires <= unixepoch()*1000) AS expired
FROM signup_verifications
GROUP BY state;
```

Behind the Render proxy the connection-address limit may be **shared between unrelated
visitors**, so a spike in one `otp_limits` key is not evidence of one person retrying.
