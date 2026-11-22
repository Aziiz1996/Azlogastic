# Accounts & Authentication Tables

Login, signup, phone verification and rate limiting. All in `leader.sqlite`.

## Business Context

Registration is email + password, optionally hardened with Twilio Verify SMS. A verified
phone is recorded on the account but grants nothing on its own — it is one of two
independent conditions for restricted content, the other being owner approval. Sessions
are cookie-based (`leader_session`), hashed before storage, valid 7 days.

## Standard Filters

```sql
WHERE u.owner = 0
  AND COALESCE(json_extract(u.data,'$.demo'),0) = 0
```

---

## Key Tables

### users
**Location**: `users`
**Description**: the root entity. One row per login. Also carries the entire per-account
application document in `data`.
**Primary Key**: `id`
**Update Frequency**: on every app write (synchronous, no lag)

| Column | Type | Description | Notes |
|---|---|---|---|
| **id** | TEXT PK | Account identifier | The join key for every other table |
| **username** | TEXT UNIQUE NOT NULL | Login identifier — the **email address** | **PII** |
| salt | TEXT NOT NULL | Password salt | **Secret — never select** |
| password_hash | TEXT NOT NULL | Password hash | **Secret — never select** |
| **owner** | INTEGER NOT NULL DEFAULT 0 | `1` = platform operator | Exclude from student metrics |
| **data** | TEXT NOT NULL | The whole `AppData` JSON document | See `academics.md` / `finance.md` |

**Relationships**: parent of everything keyed on `user_id`.

**There is no `created_at`.** Account age and signup-date cohorts are not available from
this table.

```sql
-- Safe projection: no secrets, no giant blob
SELECT id, username, owner,
       json_extract(data,'$.settings.name')     AS display_name,
       json_extract(data,'$.settings.currency') AS currency,
       length(data)                             AS data_bytes
FROM users
WHERE owner = 0;
```

`length(data)` is a useful proxy for how much a student has in the app, and for spotting
accounts bloated by base64 CV/photo attachments.

---

### sessions
**Location**: `sessions`
**Description**: live login sessions. One row per successful login.
**Primary Key**: `hash`

| Column | Type | Description | Notes |
|---|---|---|---|
| hash | TEXT PK | SHA-256 of the session cookie | **Secret — never select** |
| **user_id** | TEXT NOT NULL | → `users.id` | No FK constraint declared |
| **expires** | INTEGER NOT NULL | Expiry, **epoch milliseconds** | login + 7 days |

**Expired rows are not deleted.** Count live sessions only with
`expires > unixepoch()*1000`.

```sql
-- Students with a live session
SELECT COUNT(DISTINCT s.user_id) AS signed_in_now
FROM sessions s JOIN users u ON u.id = s.user_id
WHERE u.owner = 0 AND s.expires > unixepoch()*1000;

-- Rough "last seen": newest session expiry minus the 7-day window
SELECT user_id, datetime((MAX(expires) - 604800000)/1000,'unixepoch') AS approx_last_login_utc
FROM sessions GROUP BY user_id ORDER BY MAX(expires) DESC;
```

The "last login" above is an inference from the expiry, not a stored event. Label it as
approximate in any report.

---

### account_details
**Location**: `account_details`
**Description**: profile facts captured by the phone-verified signup flow.
**Primary Key**: `user_id`

| Column | Type | Description | Notes |
|---|---|---|---|
| **user_id** | TEXT PK | → `users.id` | |
| **handle** | TEXT UNIQUE NOT NULL | Public handle chosen at signup | Second unique key per person |
| birthdate | TEXT NOT NULL | Date of birth | **PII** |
| **verified_phone** | TEXT NOT NULL | Verified phone in E.164 | **PII**; non-empty ⇒ verified |

Accounts predating the phone-verification release have **no row here**. `LEFT JOIN` and
test `COALESCE(verified_phone,'') <> ''` — the service checks truthiness of the string,
not `NULL`.

Saudi `05…` numbers are accepted at input and stored in international form.

---

### signup_verifications
**Location**: `signup_verifications`
**Description**: in-flight SMS verification challenges during registration.
**Primary Key**: `hash`

| Column | Type | Description | Notes |
|---|---|---|---|
| hash | TEXT PK | Hashed challenge token | **Secret — never select** |
| **email** | TEXT NOT NULL | Email the challenge is bound to | **PII** |
| **phone** | TEXT NOT NULL | Phone the challenge is bound to | **PII** |
| **expires** | INTEGER NOT NULL | **Epoch ms**; 10-minute window | |
| **verified** | INTEGER NOT NULL DEFAULT 0 | `1` once the provider approved the code | |
| **attempts** | INTEGER NOT NULL DEFAULT 0 | Wrong-code attempts, capped at 5 | |
| sid | TEXT | Provider Verification SID (added by migration) | **Secret — never select** |
| **state** | TEXT NOT NULL DEFAULT 'pending' | `pending` → `checking` → `verified` / `used` | Added by migration |

Proof is bound to email **and** phone **and** `sid` together, and is consumed once. The
`checking` state is a concurrency guard — a row stuck there means a provider call was
interrupted, not that a user is mid-typing.

### signup_used_sids
`sid TEXT PK`, `expires INTEGER` (epoch ms). Spent Verification SIDs, so a proof cannot
be replayed. Row count ≈ successful consumed verifications still inside the retention
window; it is not a lifetime total.

---

### otp_limits
**Location**: `otp_limits`
**Description**: durable rate-limit counters, held in SQLite so limits survive restarts.
**Primary Key**: `key`

| Column | Type | Description | Notes |
|---|---|---|---|
| **key** | TEXT PK | Limit bucket (phone, connection address, or global) | |
| **count** | INTEGER NOT NULL | Requests reserved in the window | Reserved **before** the provider call |
| **expires** | INTEGER NOT NULL | Window end, **epoch ms** | |

A **counter table, not a log.** Rows expire and are overwritten, so it cannot answer
historical "how many codes did we send". Behind the Render proxy, the connection-address
bucket can be shared by unrelated visitors — never read one key as one person.

---

### recovery_codes
`hash TEXT PK` (**secret — never select**), `user_id TEXT UNIQUE NOT NULL`.
`user_id` is `UNIQUE`, so at most **one** recovery code per account. Analysis is limited
to coverage:

```sql
SELECT COUNT(*) AS students,
       SUM(rc.user_id IS NOT NULL) AS with_recovery_code
FROM users u LEFT JOIN recovery_codes rc ON rc.user_id = u.id
WHERE u.owner = 0;
```

---

## Common Gotchas

1. **`SELECT * FROM users`** exposes `password_hash` and `salt` and dumps a multi-megabyte
   `data` document per row.
   - Wrong: `SELECT * FROM users`
   - Right: `SELECT id, username, owner FROM users`
2. **Epoch ms, not seconds** for `sessions.expires`, `signup_verifications.expires`,
   `otp_limits.expires`.
   - Wrong: `datetime(expires,'unixepoch')`
   - Right: `datetime(expires/1000,'unixepoch')`
3. **`verified_phone IS NOT NULL` is the wrong test** — the column is `NOT NULL`, so an
   unverified account either has no row or an empty string. Use
   `COALESCE(verified_phone,'') <> ''`.
4. **Session rows ≠ active users.** Expired rows persist indefinitely.
5. **No account creation date exists.** Do not build signup cohorts; say so instead.
