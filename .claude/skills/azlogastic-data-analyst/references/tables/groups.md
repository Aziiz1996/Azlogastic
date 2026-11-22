# Groups, Projects & Owner Tables

Numbered project groups, the older Private Group 3, and owner-only configuration.

## Business Context

Coursework is done in numbered groups, each with a weekly topic and one shared uploaded
deliverable that the owner reviews. Private Group 3 is a separate, older arrangement with
its own membership list and single shared document, kept for backward compatibility.
Authorisation is checked on every read and write for both.

## Entity Clarification

**"Group" means one of two unrelated things:**
- **Project group** — `project_memberships.group_number`, the numbered coursework groups.
- **Private Group 3** — `group3_members`, a standalone private group. The name contains a
  `3` but it is **not** project group number 3.

**"Project" also means two things:** a project *group*, and a task of
`kind = 'project'` inside `users.data` (see `academics.md`).

---

## Key Tables

### project_memberships
**Location**: `project_memberships`
**Description**: which group a student belongs to.
**Primary Key**: `user_id`

| Column | Type | Description | Notes |
|---|---|---|---|
| **user_id** | TEXT PK | → `users.id` | PK ⇒ **at most one group per student** |
| **group_number** | INTEGER NOT NULL | → the app's `PROJECT_GROUP_CATALOG` | Validated against the catalogue on write |

The catalogue is **not contiguous** — it runs 1, 2, 3, 5, 6, 7, 8, 9, 10, 11, 12, with
**no group 4**. Do not generate a 1..N series to fill gaps; a missing number is not a
group with zero members.

Group size in the catalogue is 7–9 students. A group_number with far fewer rows here means
students have not been enrolled in the table, not that the group shrank.

```sql
SELECT pm.group_number, COUNT(*) AS members
FROM project_memberships pm
JOIN users u ON u.id = pm.user_id AND u.owner = 0
GROUP BY pm.group_number
ORDER BY pm.group_number;
```

**Group topics and the member roster live in the app source, not the database** — in
`src/projectGroupCatalog.ts`, which lists members by **full name**. Join on `user_id` for
anything analytical, and do not copy names from the catalogue into reports.

---

### project_documents
**Location**: `project_documents`
**Description**: the single current deliverable per group.
**Primary Key**: `group_number`

| Column | Type | Description | Notes |
|---|---|---|---|
| **group_number** | INTEGER PK | → `project_memberships.group_number` | One document per group |
| **filename** | TEXT NOT NULL | Uploaded filename | May contain a student's name |
| bytes | BLOB NOT NULL | File contents | **Never select**; use `length(bytes)` |
| **updated_by** | TEXT NOT NULL | Uploader's `users.id` | Not a name |
| **updated_at** | TEXT NOT NULL | **ISO 8601 string** | Not epoch ms |

A re-upload **replaces** the row, so there is no version history — `updated_at` is the
latest upload only, and an earlier submission cannot be recovered or counted.

---

### project_reviews
**Location**: `project_reviews`
**Description**: the owner's verdict on a group's deliverable.
**Primary Key**: `group_number`

| Column | Type | Description | Notes |
|---|---|---|---|
| **group_number** | INTEGER PK | The group reviewed | |
| **status** | TEXT NOT NULL | `CHECK(status IN ('pending','approved','rejected'))` | Same values as student approval, different meaning |
| reviewed_by | TEXT | Reviewer's `users.id` | **Nullable** |
| reviewed_at | TEXT | Timestamp | **Nullable**, ISO string |

A group with no row is **unreviewed** — treat it as pending, the same pattern as
`student_access`. Unlike student approvals, there is **no audit table here**, so group
review history does not exist.

```sql
-- Group status board
SELECT pm.group_number,
       COUNT(DISTINCT pm.user_id)                          AS members,
       COALESCE(pr.status,'pending')                       AS review_status,
       CASE WHEN pd.group_number IS NULL THEN 'missing' ELSE 'uploaded' END AS document,
       pd.updated_at                                       AS uploaded_at,
       CASE WHEN pd.bytes IS NULL THEN NULL
            ELSE ROUND(length(pd.bytes)/1024.0, 1) END     AS size_kb
FROM project_memberships pm
LEFT JOIN project_reviews   pr ON pr.group_number = pm.group_number
LEFT JOIN project_documents pd ON pd.group_number = pm.group_number
GROUP BY pm.group_number, pr.status, pd.group_number, pd.updated_at, pd.bytes
ORDER BY pm.group_number;
```

A group can be `approved` with no document row if the owner reviewed it outside the app —
the two tables do not constrain each other.

---

### group3_members
**Location**: `group3_members`
**Description**: membership of Private Group 3.
**Primary Key**: `user_id`

| Column | Type | Description | Notes |
|---|---|---|---|
| **user_id** | TEXT PK | → `users.id` | |
| **display_name** | TEXT NOT NULL | Stored personal name | **PII** |
| **leader** | INTEGER NOT NULL DEFAULT 0 | `1` = group leader | |

Access is granted to owners **or** anyone with a row here (`group3Allowed()`), so this
table is an authorisation list as much as a roster.

### group3_document
`id INTEGER PK CHECK(id=1)`, `filename TEXT NOT NULL`, `bytes BLOB NOT NULL`.
A **single-row** table — the one shared Group 3 document. No uploader, no timestamp, so
nothing about when or by whom it changed is recoverable.

---

## Owner-only tables

### app_design
`id INTEGER PK CHECK(id=1)`, `data TEXT NOT NULL`. A single row holding app-wide theme
configuration as JSON. Not student data; ignore it in student metrics.

### owner_backups
`user_id TEXT PK`, `data TEXT NOT NULL`, `updated_at TEXT NOT NULL` (ISO string).
A snapshot of an account's document. **One row per user, overwritten on save**, so it is
a backup, not a history — it cannot be used to see how a student's data changed over time.

A stale `owner_backups.updated_at` beside a much larger `users.data` is a sign the backup
has not been refreshed, which is worth flagging operationally.

### owner_bootstrap
`user_id TEXT PK`, `credentials_path TEXT NOT NULL`. Records the owner-initialisation
path. In production the first registrant does **not** become owner automatically; an
explicit bootstrap or an existing owner account is required. Contains a filesystem path —
not analysis data, and not something to print in a report.

---

## Cross-domain query: groups vs student standing

```sql
SELECT pm.group_number,
       COUNT(*)                                                  AS members,
       SUM(COALESCE(sa.approval,'pending') = 'approved')          AS approved_members,
       SUM(COALESCE(ad.verified_phone,'') <> '')                  AS phone_verified_members,
       COALESCE(pr.status,'pending')                              AS review_status
FROM project_memberships pm
JOIN users u            ON u.id = pm.user_id AND u.owner = 0
LEFT JOIN student_access  sa ON sa.user_id = pm.user_id
LEFT JOIN account_details ad ON ad.user_id = pm.user_id
LEFT JOIN project_reviews pr ON pr.group_number = pm.group_number
GROUP BY pm.group_number, pr.status
ORDER BY pm.group_number;
```

This is the most useful owner-facing view: which groups are staffed by students who can
actually reach the material.

---

## Common Gotchas

1. **Assuming group numbers are contiguous.** There is no group 4.
2. **Confusing Private Group 3 with project group 3.** Different tables, different
   purposes.
3. **Confusing `project_reviews.status` with `student_access.approval`.** Same three
   values, one keyed by `group_number` and one by `user_id`.
4. **`INNER JOIN project_reviews`** hides unreviewed groups. `LEFT JOIN` +
   `COALESCE(status,'pending')`.
5. **`SELECT *` on `project_documents`, `group3_document` or `student_evidence`** dumps
   BLOBs. Use `length(bytes)`.
6. **Expecting submission history.** Documents are replaced in place; `owner_backups` is
   overwritten; only `student_access_audit` keeps history.
7. **`reviewed_by` / `reviewed_at` are nullable** even when `status` is set. Do not assume
   an approved group has a recorded reviewer or time.
8. **Treating group rosters as database facts.** Topics and names come from the app source
   catalogue; only `user_id` → `group_number` is in the database.
