# Entities

Every table in `leader.sqlite` keys off `users.id`. There is no surrogate analytics key
and no slowly-changing-dimension history except `student_access_audit`.

---

## Account

- **Definition**: one login to Leader Study. The root entity.
- **Primary table**: `users`
- **ID field**: `users.id` (TEXT, application-generated) — the join key everywhere
- **Business key**: `users.username`, which is the **email address**, `UNIQUE NOT NULL`
- **Relationships**: parent of `sessions`, `account_details`, `student_access`,
  `student_access_audit`, `course_summaries`, `student_evidence`, `project_memberships`,
  `group3_members`, `recovery_codes`, `owner_backups`
- **Common filters**: `owner = 0` for students;
  `COALESCE(json_extract(data,'$.demo'),0) = 0` to drop demo accounts
- **Note**: no `created_at`. Account age is not stored.

## Owner (operator)

- **Definition**: the platform operator / instructor. Holds app-wide configuration and
  approves students.
- **Primary table**: `users` where `owner = 1`
- **Relationships**: `owner_backups` (own data snapshot), `owner_bootstrap`
  (credentials path), `app_design` (app-wide theme, single row `id = 1`)
- **Note**: the approval service **refuses** to create a `student_access` row for an
  owner account (`decide()` throws `'حساب الطالب غير موجود.'`). Owners are structurally
  not students — exclude them rather than filtering them out downstream.
- In production the first registrant does **not** become owner automatically; an explicit
  owner bootstrap or an existing owner account is required.

## Student

- **Definition**: a non-owner account. Eligibility for restricted content is two
  independent flags, not one status.
- **Primary table**: `users` where `owner = 0`
- **Approval**: `student_access.approval` ∈ `pending` / `approved` / `rejected`.
  **A row may not exist** — the service defaults a missing row to `'pending'`.
- **Phone verification**: `account_details.verified_phone` non-empty.
- **Eligibility**: `/api/student/restricted` requires approved **and** phone-verified.
  Neither implies the other.
- **Note**: phone verification grants no ownership, no group membership and no approval.

## Handle

- **Definition**: a public identifier chosen during the phone-verified signup flow.
- **Primary table**: `account_details`
- **ID field**: `account_details.handle` (`UNIQUE NOT NULL`)
- **Relationship**: 1:1 with an account, but **only for accounts that completed that
  flow**. Older accounts have no `account_details` row, so `handle` is not a universal
  key — always resolve people through `users.id`.

## Signup verification (challenge)

- **Definition**: one in-flight phone-verification attempt during registration.
- **Primary table**: `signup_verifications`
- **ID field**: `hash` (PK, a hashed challenge token); `sid` is the provider's
  Verification SID
- **Lifecycle**: `state` defaults `'pending'`; `expires` is epoch ms (10-minute window);
  `attempts` caps at 5; a consumed `sid` moves to `signup_used_sids` so it cannot be
  reused.
- **Note**: proof is bound to the email **and** phone **and** `sid` together. Challenges
  issued before that binding was added are unusable and require a fresh code.

## Project group

- **Definition**: a numbered coursework group with one shared deliverable.
- **Primary tables**: `project_memberships` (`user_id` → `group_number`),
  `project_documents` (one row per `group_number`), `project_reviews` (one row per
  `group_number`)
- **ID field**: `group_number` (INTEGER). Valid values come from the app's
  `PROJECT_GROUP_CATALOG`; the catalogue is **not contiguous** (there is no group 4).
- **Relationships**: a student belongs to at most one group
  (`project_memberships.user_id` is the PK). A group has at most one document and at most
  one review.
- **Caution**: the catalogue in the app source lists members by **full name**. Join on
  `user_id`, and do not copy names into reports.

## Private Group 3

- **Definition**: a separate, older private group with its own membership and single
  shared document. Not part of the numbered project-group system.
- **Primary tables**: `group3_members` (`user_id`, `display_name`, `leader`),
  `group3_document` (single row, `id = 1`)
- **Note**: access is authorised on every read and write; `group3_members.display_name` is
  a stored personal name.

## Course

- **Definition**: a subject a student tracks. **Not a table** — an element of
  `users.data -> $.courses[]`.
- **ID field**: `$.courses[].id` (string, e.g. `'inventory-management'`)
- **Relationships**: referenced by `$.lectures[].courseId`, `$.tasks[].courseId`,
  `$.notes[].courseId`, `$.universityEvents[].courseId`, and by
  `course_summaries.course_id`
- **Caution**: course ids are **per-student**, created in that student's own document.
  Two students' `'inventory-management'` are the same shared course by convention, but a
  hand-added course id is local to one student. Shared academic content is merged into a
  student's document at read time, so the same course can appear with different `color`
  or `resources` per student.
