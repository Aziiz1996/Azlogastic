# Finance & Career Tables

Per-student money tracking and job/course applications. Both live inside `users.data`;
there are no finance tables.

## Business Context

Each student records their own income and expenses, sets a monthly budget, and may define
recurring income rules. This is **personal student budgeting**, not company revenue —
there is no platform billing, subscription or payment data anywhere in this database.
Say so plainly when someone asks about "revenue".

Each student also sets their **own currency**, so amounts are not comparable across
accounts without grouping by it.

## Standard Filters

```sql
WHERE u.owner = 0
  AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
  AND json_extract(x.value,'$.amount') > 0                  -- the app ignores non-positive
  AND json_extract(x.value,'$.date') GLOB '[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]'
```

---

## `$.transactions[]` — Transaction

| Field | Type | Description | Notes |
|---|---|---|---|
| **id** | string | Identifier, unique within the student | |
| title | string | Description | Free text, may be personal |
| **kind** | string | `income` / `expense` | |
| **amount** | number | Positive magnitude | Sign comes from `kind`, never from `amount` |
| **category** | string | Expense category | Blank → `'أخرى'` ("Other") |
| **date** | `YYYY-MM-DD` | Transaction date | Local, no timezone |
| ruleId | string? | → `$.incomeRules[].id` if auto-generated | Optional |

`financeSummary()` in `src/logic.ts` is the source of truth for the arithmetic, and it:

- **skips** transactions with a malformed date, a non-finite `amount`, or `amount <= 0`
- works in **integer cents** (`Math.round(amount * 100)`) to avoid float drift
- builds the category breakdown from **expenses only** — income has no category
- falls back to `'أخرى'` for a blank/whitespace category
- sorts categories by amount descending, then by name using Arabic collation

Mirror it with `ROUND(SUM(...)*100)/100.0`, or your totals will disagree with the app by
fractions of a unit.

```sql
-- One student, by month, matching the app
SELECT substr(json_extract(x.value,'$.date'),1,7)                                   AS month,
       json_extract(u.data,'$.settings.currency')                                   AS currency,
       ROUND(SUM(CASE WHEN json_extract(x.value,'$.kind')='income'
                 THEN json_extract(x.value,'$.amount') ELSE 0 END)*100)/100.0       AS income,
       ROUND(SUM(CASE WHEN json_extract(x.value,'$.kind')='expense'
                 THEN json_extract(x.value,'$.amount') ELSE 0 END)*100)/100.0       AS expense,
       ROUND(SUM(CASE WHEN json_extract(x.value,'$.kind')='income'
                 THEN json_extract(x.value,'$.amount')
                 ELSE -json_extract(x.value,'$.amount') END)*100)/100.0             AS balance
FROM users u, json_each(u.data,'$.transactions') AS x
WHERE u.id = :user_id
  AND json_extract(x.value,'$.amount') > 0
  AND json_extract(x.value,'$.date') GLOB '[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]'
GROUP BY month, currency
ORDER BY month;
```

```sql
-- Expense categories (expenses only, blank → أخرى)
SELECT COALESCE(NULLIF(TRIM(json_extract(x.value,'$.category')),''),'أخرى') AS category,
       ROUND(SUM(json_extract(x.value,'$.amount'))*100)/100.0               AS amount,
       COUNT(*)                                                             AS entries
FROM users u, json_each(u.data,'$.transactions') AS x
WHERE u.id = :user_id
  AND json_extract(x.value,'$.kind') = 'expense'
  AND json_extract(x.value,'$.amount') > 0
  AND substr(json_extract(x.value,'$.date'),1,7) = :month
GROUP BY category
ORDER BY amount DESC, category;
```

### Cross-student finance — group by currency

```sql
SELECT json_extract(u.data,'$.settings.currency')                      AS currency,
       COUNT(DISTINCT u.id)                                            AS students,
       ROUND(SUM(json_extract(x.value,'$.amount'))*100)/100.0          AS total_expense
FROM users u, json_each(u.data,'$.transactions') AS x
WHERE u.owner = 0 AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
  AND json_extract(x.value,'$.kind') = 'expense'
  AND json_extract(x.value,'$.amount') > 0
GROUP BY currency;
```

A single bare total across currencies is always wrong. Prefer medians per student over
sums when comparing cohorts, since one student's scale can dominate.

---

## `$.incomeRules[]` — recurring income rule

| Field | Type | Description | Notes |
|---|---|---|---|
| **id** | string | Unique within the student | Referenced by `$.transactions[].ruleId` |
| title | string | Label | |
| **amount** | number | Validated `0.01`–`1e9` | |
| **day** | integer or null | Day of month, **1–31**, or `null` | Not 0–6 — unlike lectures |
| category | string | Category | |

Max 1000 rules per student, ids unique (enforced by `validateData()`).

A rule is a **template**, not money. Actual income only counts once a transaction exists
with that `ruleId`. Never sum `incomeRules.amount` as income.

```sql
-- Rules that never produced a transaction
SELECT u.id, json_extract(r.value,'$.title') AS rule
FROM users u, json_each(u.data,'$.incomeRules') AS r
WHERE u.owner = 0
  AND NOT EXISTS (
    SELECT 1 FROM json_each(u.data,'$.transactions') AS x
    WHERE json_extract(x.value,'$.ruleId') = json_extract(r.value,'$.id'));
```

---

## `$.settings` — budget context

`monthlyBudget` is a **target**, not spend. `currency` scopes every amount.
`dailyMinutes`, `studyStart` and `studyDays[]` belong to the study planner, not finance.

```sql
-- Budget variance for a month
SELECT u.id,
       json_extract(u.data,'$.settings.currency')     AS currency,
       json_extract(u.data,'$.settings.monthlyBudget') AS budget,
       ROUND(SUM(json_extract(x.value,'$.amount'))*100)/100.0 AS spent,
       ROUND((SUM(json_extract(x.value,'$.amount'))
              - json_extract(u.data,'$.settings.monthlyBudget'))*100)/100.0 AS over_under
FROM users u, json_each(u.data,'$.transactions') AS x
WHERE u.owner = 0 AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
  AND json_extract(x.value,'$.kind') = 'expense'
  AND json_extract(x.value,'$.amount') > 0
  AND substr(json_extract(x.value,'$.date'),1,7) = :month
GROUP BY u.id, currency, budget;
```

A `monthlyBudget` of `0` means unset, not "zero budget" — exclude it from variance
reporting rather than showing everyone as over budget.

---

## `$.applications[]` — career application

| Field | Type | Description | Notes |
|---|---|---|---|
| **id** | string | Unique within the student | |
| title / organization | string | What and where | |
| **kind** | string | `job` / `course` | `course` = training, **not** `$.courses[]` |
| url | string | Posting link | |
| **status** | string | `saved` → `preparing` → `submitted` → `accepted` / `rejected` | Current state, no history |
| createdAt | string | When saved | Only timestamp available |
| notes | string | Free text | Personal content |

```sql
SELECT json_extract(a.value,'$.kind')   AS kind,
       json_extract(a.value,'$.status') AS status,
       COUNT(*)                         AS applications,
       COUNT(DISTINCT u.id)             AS students
FROM users u, json_each(u.data,'$.applications') AS a
WHERE u.owner = 0 AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
GROUP BY kind, status ORDER BY kind, status;
```

Status is a current state with no transition log, so "time to submission" and "how many
were rejected after an interview" are not answerable. Acceptance rates on a handful of
decided applications are noise — always print the denominator next to the percentage.

`$.opportunityLinks[]` (`{id, name, url, kind}`) is the catalogue a student saved from.
The app's categories are `توظيف` (jobs), `دورات` (courses), `شهادات مهنية`
(professional certificates) and `تجسير وإكمال دراسة` (bridging / degree completion);
these are derived labels, **not stored** on the row.

---

## `$.studentProfile` — PII, handle with care

`{fullName, email, phone, city, headline, summary, university, major, graduationYear,
education, experience, skills[], linkedin, website, photo?, cv?}`.

`photo` and `cv` are `Attachment` objects holding the file as a **base64 `dataUrl`**, so a
single profile can be megabytes and `SELECT data FROM users` will return all of it.

Report **completeness**, never contents:

```sql
SELECT u.id,
       (json_extract(u.data,'$.studentProfile.fullName')   IS NOT NULL)
     + (json_extract(u.data,'$.studentProfile.university') IS NOT NULL)
     + (json_extract(u.data,'$.studentProfile.major')      IS NOT NULL)
     + (json_extract(u.data,'$.studentProfile.cv')         IS NOT NULL) AS fields_filled,
       json_array_length(COALESCE(json_extract(u.data,'$.studentProfile.skills'),'[]')) AS skills
FROM users u
WHERE u.owner = 0 AND COALESCE(json_extract(u.data,'$.demo'),0) = 0;
```

---

## Common Gotchas

1. **"Revenue" does not exist here.** `$.transactions` is student personal budgeting.
   There is no platform billing or subscription data in `leader.sqlite`.
2. **Summing across students without grouping by currency.**
3. **Negating by `amount`** instead of by `kind`. `amount` is always positive; the app
   skips `amount <= 0` entirely.
4. **Float drift.** Compute in cents (`ROUND(x*100)/100.0`) to match the app.
5. **Income categories.** Only expenses are categorised; a category breakdown that
   includes income does not match the app's.
6. **Counting `incomeRules` as income.** They are templates until a transaction exists.
7. **`incomeRules.day` is 1–31**, while lecture and `studyDays` values are 0–6. Easy to
   cross up.
8. **`monthlyBudget = 0` means unset.**
9. **`SELECT data FROM users`** can return megabytes per row because of base64 CV and
   photo attachments. Project specific JSON paths instead.
