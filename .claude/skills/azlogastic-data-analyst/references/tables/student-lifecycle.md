# Student Lifecycle Tables

Owner approval, its audit trail, and enrolment evidence.

## Business Context

A student who registers is **not** automatically allowed into restricted content. The
owner reviews each student and sets an approval state; the decision and its author are
recorded. Access to `/api/student/restricted` needs approval **and** a verified phone —
two independent conditions, deliberately kept apart on the student status screen.

## Entity Clarification

**"Approved" can mean two unrelated things:**
- **Student approval**: `student_access.approval = 'approved'` — the owner let a person in.
- **Project review approval**: `project_reviews.status = 'approved'` — a group's
  deliverable passed. Keyed by `group_number`, not `user_id`. See `groups.md`.

Always confirm which before querying; both use the same three values.

---

## Key Tables

### student_access
**Location**: `student_access`
**Description**: **current** approval state, one row per reviewed student.
**Primary Key**: `user_id`

| Column | Type | Description | Notes |
|---|---|---|---|
| **user_id** | TEXT PK | → `users.id` | |
| **approval** | TEXT NOT NULL | `CHECK(approval IN ('pending','approved','rejected'))` | Current state only |

**The most important fact about this table: a student with no row is `pending`.** The
service defaults a missing row to `'pending'`, and new students have no row at all. An
`INNER JOIN` therefore hides the entire never-reviewed population — usually the exact
group the owner is asking about.

```sql
-- Right: every student appears, unreviewed ones as pending
SELECT COALESCE(sa.approval,'pending') AS approval, COUNT(*) AS students
FROM users u
LEFT JOIN student_access sa ON sa.user_id = u.id
WHERE u.owner = 0 AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
GROUP BY approval;
```

Owner accounts are structurally excluded: `decide()` refuses a non-existent or owner
account, so no owner row can exist here.

---

### student_access_audit
**Location**: `student_access_audit`
**Description**: append-only history of approval decisions. The only real event log in
the database.
**Primary Key**: `id` (INTEGER, autoincrementing rowid)

| Column | Type | Description | Notes |
|---|---|---|---|
| **id** | INTEGER PK | Insertion order — usable as a sequence | |
| **user_id** | TEXT NOT NULL | Student decided about | |
| **approval** | TEXT NOT NULL | Value set by this decision | No `CHECK` here, unlike `student_access` |
| **actor** | TEXT NOT NULL | Who decided — an **account id**, not a name | Resolve via `users.id` |
| **at** | INTEGER NOT NULL | Decision time, **epoch ms** | |

Written in the same `BEGIN IMMEDIATE` transaction as the `student_access` upsert, so
current state and history never disagree.

```sql
-- Current state reconstructed from history (should match student_access)
SELECT a.user_id, a.approval AS latest, datetime(a.at/1000,'unixepoch') AS at_utc
FROM student_access_audit a
JOIN (SELECT user_id, MAX(at) AS at FROM student_access_audit GROUP BY user_id) m
  ON m.user_id = a.user_id AND m.at = a.at;
```

```sql
-- Owner review throughput by day
SELECT date(at/1000,'unixepoch') AS day,
       COUNT(*)                  AS decisions,
       SUM(approval='approved')  AS approved,
       SUM(approval='rejected')  AS rejected,
       COUNT(DISTINCT actor)     AS deciders
FROM student_access_audit
GROUP BY day ORDER BY day DESC;
```

This is the **only** table that supports a genuine time series. Use it whenever the
question is "when" or "how fast".

---

### student_evidence
**Location**: `student_evidence`
**Description**: an uploaded file proving enrolment, one per student.
**Primary Key**: `user_id`

| Column | Type | Description | Notes |
|---|---|---|---|
| **user_id** | TEXT PK | → `users.id` | One upload per student; a new one replaces it |
| **filename** | TEXT NOT NULL | Original filename | May contain the student's name — **PII** |
| bytes | BLOB NOT NULL | File contents | **Never select**; use `length(bytes)` |
| **submitted_at** | TEXT NOT NULL | **ISO 8601 string**, not epoch ms | Different encoding from the audit table |

```sql
-- Evidence coverage against approval state
SELECT COALESCE(sa.approval,'pending')                  AS approval,
       COUNT(*)                                         AS students,
       SUM(se.user_id IS NOT NULL)                      AS with_evidence,
       ROUND(AVG(length(se.bytes))/1024.0, 1)           AS avg_kb
FROM users u
LEFT JOIN student_access   sa ON sa.user_id = u.id
LEFT JOIN student_evidence se ON se.user_id = u.id
WHERE u.owner = 0 AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
GROUP BY approval;
```

Approved-without-evidence and evidence-without-review are both legitimate states; the
tables are independent and neither enforces the other.

---

## The full eligibility picture

```sql
SELECT u.id,
       COALESCE(ad.verified_phone,'') <> ''              AS phone_verified,
       COALESCE(sa.approval,'pending')                   AS approval,
       se.user_id IS NOT NULL                            AS has_evidence,
       (COALESCE(sa.approval,'pending') = 'approved'
        AND COALESCE(ad.verified_phone,'') <> '')        AS can_access_restricted
FROM users u
LEFT JOIN account_details  ad ON ad.user_id = u.id
LEFT JOIN student_access   sa ON sa.user_id = u.id
LEFT JOIN student_evidence se ON se.user_id = u.id
WHERE u.owner = 0 AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
ORDER BY can_access_restricted, approval;
```

---

## Common Gotchas

1. **`INNER JOIN student_access` drops every unreviewed student.**
   - Wrong: `FROM users u JOIN student_access sa ON sa.user_id = u.id`
   - Right: `LEFT JOIN` + `COALESCE(sa.approval,'pending')`
2. **`actor` is an account id, not a person's name.** Join to `users` to describe it, and
   prefer reporting "the owner" over an identifier.
3. **Mixed time encodings in one domain**: `student_access_audit.at` is epoch **ms**,
   `student_evidence.submitted_at` is an **ISO string**. Normalise before comparing them.
4. **`student_access` has no history.** "How long were they pending?" needs
   `student_access_audit`, and even then the clock starts at the first *decision* — signup
   time is not stored anywhere.
5. **Approval does not imply phone verification**, or evidence, or group membership. Four
   independent facts; do not collapse them into one "status" column without saying so.
