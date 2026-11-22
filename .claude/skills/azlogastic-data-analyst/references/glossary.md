# Terminology (Arabic ↔ English)

The product, its UI and its error messages are Arabic; the schema and code are English.
This maps the words the owner will use in a question to what is actually stored.

## Business terms

| Arabic | English | Where it lives | Notes |
|---|---|---|---|
| طالب | student | `users` where `owner = 0` | Not the same as "approved student" |
| المالك | owner / operator | `users.owner = 1` | One account; excluded from student metrics |
| اعتماد | approval | `student_access.approval` | `pending` / `approved` / `rejected` |
| معتمد | approved | `approval = 'approved'` | Owner decision, independent of phone |
| مرفوض | rejected | `approval = 'rejected'` | Reversible; history in the audit table |
| قيد الانتظار | pending | missing row **or** `approval = 'pending'` | Default when no row exists |
| هاتف موثّق | verified phone | `account_details.verified_phone` non-empty | Twilio Verify |
| رمز التحقق | OTP / verification code | `signup_verifications` | 10-minute window, 5 attempts |
| إثبات القيد | enrolment evidence | `student_evidence` | Uploaded file, BLOB |
| مجموعة مشروع | project group | `project_memberships.group_number` | Catalogue is non-contiguous |
| مقرر | course | `$.courses[]` in `users.data` | Per-student JSON, not a table |
| محاضرة | lecture | `$.lectures[]` | `day` is 0–6 where **0 = الأحد (Sunday)**; `start`/`end` are `HH:MM` |
| مهمة / واجب | task / assignment | `$.tasks[]` with `kind = 'assignment'` | |
| مشروع | project (task kind) | `$.tasks[]` with `kind = 'project'` | Distinct from project **group** |
| اختبار | exam | `$.tasks[]` `kind = 'exam'`, or `$.practiceAttempts[].mode = 'exam'` | Two different meanings — ask which |
| تدريب / تمرين | practice | `$.practiceAttempts[].mode = 'practice'` | Ungraded rehearsal |
| ملاحظات | notes | `$.notes[]` | `summary` is AI-generated |
| خطة الدراسة | study plan | computed, **not stored** | `buildStudyPlan()` derives it at read time |
| سجل الدراسة | study log | `$.studyLog[]` | Completed sessions with minutes |
| المالية | finance | `$.transactions[]`, `$.incomeRules[]` | Per-student currency |
| مصروف | expense | `$.transactions[].kind = 'expense'` | Feeds category breakdown |
| دخل | income | `$.transactions[].kind = 'income'` | Excluded from categories |
| أخرى | "Other" | category fallback | Used when a category string is blank |
| ميزانية شهرية | monthly budget | `$.settings.monthlyBudget` | Target, not actual |
| الفرص | opportunities | `$.opportunityLinks[]`, `$.applications[]` | Jobs and courses |
| توظيف | employment / jobs | `kind = 'job'` | |
| دورات | courses (training) | `kind = 'course'` | Not academic `$.courses[]` — different concept |
| شهادات مهنية | professional certificates | opportunity category | Category label only |
| تجسير وإكمال دراسة | bridging / degree completion | opportunity category | Category label only |
| الملف الشخصي | student profile | `$.studentProfile` | **PII**: name, email, phone, CV, photo |
| السيرة الذاتية | CV | `$.studentProfile.cv` | base64 `dataUrl` attachment — large |
| الجامعة | university | `$.settings.universityName`, `$.studentProfile.university` | Free text |
| سلاسل الإمداد | supply chain | subject domain | Default `course_id` in the Firebase backend |
| إدارة المخزون والمخازن | inventory & warehouse management | course `'inventory-management'` | Name has hamza variants — match on id |

## Stored value vocabularies

Use these exact strings; they are enforced by `CHECK` constraints or by the TypeScript
types, so a typo returns zero rows rather than an error.

| Field | Allowed values |
|---|---|
| `student_access.approval`, `student_access_audit.approval` | `pending`, `approved`, `rejected` |
| `project_reviews.status` | `pending`, `approved`, `rejected` |
| `signup_verifications.state` | `pending` (default), `checking`, `verified`, `used` |
| `$.tasks[].kind` | `assignment`, `project`, `exam` |
| `$.tasks[].priority` | `low`, `medium`, `high` |
| `$.transactions[].kind` | `income`, `expense` |
| `$.applications[].status` | `saved`, `preparing`, `submitted`, `accepted`, `rejected` |
| `$.applications[].kind`, `$.opportunityLinks[].kind` | `job`, `course` |
| `$.practiceAttempts[].mode`, `$.activePractice.mode` | `practice`, `exam` |
| `$.universityEvents[].kind` | `lecture`, `assignment`, `exam`, `event` |

## Ambiguous asks — clarify before querying

- **"اختبار / exam"** — a scheduled exam task (`$.tasks[].kind = 'exam'`) or a completed
  exam-mode attempt (`$.practiceAttempts[].mode = 'exam'`)?
- **"دورة / course"** — an academic course (`$.courses[]`) or a training opportunity
  (`kind = 'course'`)?
- **"مجموعة / group"** — a numbered project group (`project_memberships`) or Private
  Group 3 (`group3_members`)?
- **"طالب نشط / active student"** — has a live session, or has logged study recently?
  There is no stored "active" flag.
- **"عدد الطلاب / student count"** — all student accounts, phone-verified ones, or
  approved ones? These three numbers differ.
