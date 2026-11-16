# Kidconomy Database Schema

This is the authoritative design for Kidconomy's SQLite database. It covers every table, column, constraint, and index, and why each exists. When migrations under `migrations/` are written, they should match this document, and any schema change should update both.

> Status: design. Nothing here has been migrated yet.

---

## Conventions

| Topic | Convention | Why |
|---|---|---|
| Engine | SQLite 3.37+ with `STRICT` tables | `STRICT` enforces declared column types instead of SQLite's loose type affinity. |
| Pragmas | `foreign_keys = ON`, `journal_mode = WAL`, `busy_timeout = 5000` | FKs are off by default in SQLite. WAL allows concurrent readers during writes and is required by Litestream. |
| Primary keys | `id TEXT` holding a **UUIDv7** | Globally unique across households (safe for merges and exports), time-ordered so inserts stay index-friendly, and not guessable in URLs. |
| Timestamps | `TEXT` in RFC 3339 **UTC** with a `Z` suffix (e.g. `2026-11-13T18:04:05.123Z`) | Sorts correctly as text and is human-readable in the SQLite CLI. Converted to the household time zone only for display and for "today" logic. |
| Local dates | `TEXT` `YYYY-MM-DD` | Used where a value is a calendar day in the household's time zone (e.g. "the chore for Tuesday"), not an instant. |
| Booleans | `INTEGER` `0`/`1` with `CHECK (x IN (0, 1))` | SQLite has no boolean type. |
| Money | `INTEGER` **cents** | Never floats, so there are no rounding errors. |
| Points | `INTEGER` | Points are whole numbers. |
| Rates | `INTEGER` **basis points** (1 bp = 0.01%) | Exact arithmetic: 5% = `500` bp. |
| Enums | `TEXT` with a `CHECK (col IN (...))` | Readable values, enforced by the database, and mapped to Rust enums. |
| Soft delete | `archived_at TEXT NULL` | Rows referenced by ledgers (kids, tasks, store items) must never be hard-deleted. Archiving hides them from the UI. |
| Household scoping | Every domain table has a `household_id` | Tenancy filter on every query, and supports multiple households in one database. The API layer guarantees referenced rows belong to the same household. |
| Ledgers | Append-only. Rows are never `UPDATE`d or `DELETE`d | Balances are `SUM(amount)`. Corrections are new `adjust` rows, which preserves a full audit trail. |
| Snapshots | Values like task points and item cost are copied onto instance or purchase rows | Editing a task or store item later must not rewrite history. |

### Table overview

| Area | Tables |
|---|---|
| Tenancy & identity | `households`, `users`, `sessions` |
| Tasks & calendar | `tasks`, `task_assignees`, `task_occurrence_overrides`, `task_templates`, `task_template_kids`, `task_instances` |
| Points | `point_transactions` |
| Store | `store_items`, `store_item_kids`, `purchases` |
| Cash | `conversion_rates`, `conversions`, `cash_transactions`, `withdrawals` |
| Investing | `investment_accounts`, `investment_rates`, `investment_transactions` |
| Devices | `agents`, `devices`, `device_kids`, `device_overrides` |
| Operations | `audit_log`, `job_runs` |

### Entity relationships (simplified)

```
households ─┬─< users ─┬─< sessions
            │          ├─< point_transactions
            │          ├─< cash_transactions
            │          ├─< investment_accounts ─┬─< investment_rates
            │          │                        └─< investment_transactions
            │          └─< withdrawals
            ├─< tasks ─┬─< task_assignees >─ users
            │          ├─< task_occurrence_overrides
            │          └─< task_instances >─ users (assignee)
            ├─< task_templates ─┬─< task_template_kids >─ users
            │                   └─< task_instances (claims)
            ├─< store_items ─┬─< store_item_kids >─ users
            │                └─< purchases >─ users
            ├─< conversion_rates ─< conversions
            └─< agents ─< devices ─┬─< device_kids >─ users
                                   └─< device_overrides
```

---

## Tenancy & Identity

### `households`

A family workspace. Every other domain row belongs to one household.

```sql
CREATE TABLE households (
    id                        TEXT PRIMARY KEY,                 -- UUIDv7
    name                      TEXT NOT NULL,                    -- Display name, e.g. "The Warrens"
    timezone                  TEXT NOT NULL,                    -- IANA zone, e.g. "America/Chicago"; defines "today", daily device locks, and instance dates
    currency                  TEXT NOT NULL DEFAULT 'USD',      -- ISO 4217 code used to format cash amounts
    default_requires_approval INTEGER NOT NULL DEFAULT 1
        CHECK (default_requires_approval IN (0, 1)),            -- Applied when a task/template leaves requires_approval NULL
    week_starts_on            INTEGER NOT NULL DEFAULT 0
        CHECK (week_starts_on BETWEEN 0 AND 6),                 -- 0 = Sunday; used by calendar views and weekly accruals
    created_at                TEXT NOT NULL,
    updated_at                TEXT NOT NULL,
    archived_at               TEXT                              -- Set when a household is closed
) STRICT;
```

### `users`

Parents and kids. Parents log in with email and password, and kids pick their profile and enter a PIN.

```sql
CREATE TABLE users (
    id                   TEXT PRIMARY KEY,
    household_id         TEXT NOT NULL REFERENCES households(id),
    role                 TEXT NOT NULL CHECK (role IN ('parent', 'kid')),
    display_name         TEXT NOT NULL,                 -- Shown on profile picker, calendar rows, approvals
    avatar_key           TEXT,                          -- Storage key/URL of an avatar image or emoji code
    color                TEXT,                          -- Hex color used for this person's calendar resource row
    birthdate            TEXT,                          -- YYYY-MM-DD; kids only; drives store age gating
    email                TEXT COLLATE NOCASE,           -- Parents only; login identifier
    password_hash        TEXT,                          -- Parents only; argon2id PHC string
    pin_hash             TEXT,                          -- Kids only; argon2id hash of the PIN
    failed_login_count   INTEGER NOT NULL DEFAULT 0,    -- Consecutive failed password/PIN attempts
    locked_until         TEXT,                          -- Lockout after too many failures (PINs are short, so this matters)
    sort_order           INTEGER NOT NULL DEFAULT 0,    -- Order on the profile picker and calendar
    created_at           TEXT NOT NULL,
    updated_at           TEXT NOT NULL,
    archived_at          TEXT,                          -- Archived users keep their ledger history
    CHECK (
        (role = 'parent' AND email IS NOT NULL AND password_hash IS NOT NULL AND pin_hash IS NULL)
     OR (role = 'kid'    AND email IS NULL     AND password_hash IS NULL     AND pin_hash IS NOT NULL)
    )
) STRICT;

CREATE UNIQUE INDEX users_email_uq ON users(email) WHERE email IS NOT NULL;
    -- Parents log in by email, which must be globally unique. Partial, because kids have no email.
CREATE INDEX users_household_idx ON users(household_id, role, sort_order);
    -- Profile picker and "list kids in household".
```

### `sessions`

Server-side login sessions, referenced by an HTTP-only cookie.

```sql
CREATE TABLE sessions (
    id            TEXT PRIMARY KEY,                        -- SHA-256 hash of the cookie token (the raw token is never stored)
    user_id       TEXT NOT NULL REFERENCES users(id),
    household_id  TEXT NOT NULL REFERENCES households(id), -- Denormalized for fast tenancy checks on every request
    created_at    TEXT NOT NULL,
    last_seen_at  TEXT NOT NULL,                           -- Updated (throttled) on use; supports idle timeout
    expires_at    TEXT NOT NULL,                           -- Absolute expiry (kids on a shared tablet may get shorter sessions)
    user_agent    TEXT,                                    -- Shown in "active sessions" so parents can revoke devices
    revoked_at    TEXT                                     -- Logout / remote revoke
) STRICT;

CREATE INDEX sessions_user_idx ON sessions(user_id);
    -- "Log out everywhere" and active-session list.
CREATE INDEX sessions_expires_idx ON sessions(expires_at);
    -- Cleanup job deletes expired sessions.
```

---

## Tasks & Calendar

### `tasks`

The definition of a chore: one-off, scheduled, or recurring. Kids never complete a `task` directly. They complete its `task_instances`.

```sql
CREATE TABLE tasks (
    id                    TEXT PRIMARY KEY,
    household_id          TEXT NOT NULL REFERENCES households(id),
    kind                  TEXT NOT NULL CHECK (kind IN ('one_off', 'scheduled', 'recurring')),
    title                 TEXT NOT NULL,
    description           TEXT,
    icon                  TEXT,                          -- Icon name or emoji for UI
    points                INTEGER NOT NULL CHECK (points >= 0),
    requires_approval     INTEGER CHECK (requires_approval IN (0, 1)),
        -- NULL = inherit households.default_requires_approval
    required_for_unlock   INTEGER NOT NULL DEFAULT 0 CHECK (required_for_unlock IN (0, 1)),
        -- If 1, the assignee's devices stay locked until today's instance is approved
    all_day               INTEGER NOT NULL DEFAULT 1 CHECK (all_day IN (0, 1)),
        -- 1 = shows as an all-day item; 0 = has a time slot on the calendar
    due_date              TEXT,     -- one_off: optional YYYY-MM-DD it should be done by
    dtstart               TEXT,     -- scheduled/recurring: first start, as household-local datetime "YYYY-MM-DDTHH:MM" (local so DST is handled correctly)
    duration_minutes      INTEGER CHECK (duration_minutes > 0),  -- Calendar block length for timed tasks
    rrule                 TEXT,     -- recurring only: RFC 5545 RRULE body, e.g. "FREQ=WEEKLY;BYDAY=MO,WE,FR"
    generated_through     TEXT,     -- recurring only: local date up to which instances have been materialized
    created_by            TEXT NOT NULL REFERENCES users(id),
    created_at            TEXT NOT NULL,
    updated_at            TEXT NOT NULL,
    archived_at           TEXT,     -- Stops future instance generation; past instances keep their history
    CHECK (kind <> 'recurring' OR (rrule IS NOT NULL AND dtstart IS NOT NULL)),
    CHECK (kind <> 'scheduled' OR (dtstart IS NOT NULL AND rrule IS NULL)),
    CHECK (kind <> 'one_off'   OR (rrule IS NULL AND dtstart IS NULL))
) STRICT;

CREATE INDEX tasks_household_idx ON tasks(household_id, archived_at);
    -- Task management list.
CREATE INDEX tasks_generation_idx ON tasks(kind, generated_through) WHERE archived_at IS NULL;
    -- Instance-generation job finds recurring tasks whose horizon needs extending.
```

**Instance generation:** a background job expands each recurring task's RRULE (in the household time zone) up to a rolling horizon (e.g. 60 days ahead) and inserts `task_instances`, applying `task_occurrence_overrides`. Editing a recurring task deletes **future, untouched** (`pending`) instances and regenerates them. Past and submitted instances are never changed.

### `task_assignees`

Which kids a task is assigned to. Each assignee gets their own instance per occurrence, so siblings don't share a checkbox.

```sql
CREATE TABLE task_assignees (
    task_id   TEXT NOT NULL REFERENCES tasks(id),
    user_id   TEXT NOT NULL REFERENCES users(id),   -- Must be a kid in the same household
    PRIMARY KEY (task_id, user_id)
) STRICT, WITHOUT ROWID;

CREATE INDEX task_assignees_user_idx ON task_assignees(user_id);
    -- "All tasks assigned to this kid".
```

### `task_occurrence_overrides`

Single-occurrence changes to a recurring task, such as when a parent drags just this Tuesday's chore to Wednesday or skips it for a holiday. This plays the role of RRULE `EXDATE`/`RECURRENCE-ID`.

```sql
CREATE TABLE task_occurrence_overrides (
    id                    TEXT PRIMARY KEY,
    task_id               TEXT NOT NULL REFERENCES tasks(id),
    original_occurrence   TEXT NOT NULL,   -- Local datetime the RRULE originally produced
    action                TEXT NOT NULL CHECK (action IN ('skip', 'move')),
    new_start             TEXT,            -- move only: new local start datetime
    new_duration_minutes  INTEGER CHECK (new_duration_minutes > 0),
    created_by            TEXT NOT NULL REFERENCES users(id),
    created_at            TEXT NOT NULL,
    CHECK (action <> 'move' OR new_start IS NOT NULL)
) STRICT;

CREATE UNIQUE INDEX task_overrides_uq ON task_occurrence_overrides(task_id, original_occurrence);
    -- At most one override per occurrence; also the lookup key during generation.
```

### `task_templates`

Pre-filled chores kids can claim anytime for impromptu points, such as "Take out the trash" or "Unload the dishwasher".

```sql
CREATE TABLE task_templates (
    id                     TEXT PRIMARY KEY,
    household_id           TEXT NOT NULL REFERENCES households(id),
    title                  TEXT NOT NULL,
    description            TEXT,
    icon                   TEXT,
    points                 INTEGER NOT NULL CHECK (points >= 0),
    requires_approval      INTEGER CHECK (requires_approval IN (0, 1)),  -- NULL = household default
    max_claims_per_day     INTEGER CHECK (max_claims_per_day > 0),       -- Per kid, per local day; NULL = unlimited
    cooldown_minutes       INTEGER CHECK (cooldown_minutes > 0),         -- Minimum gap between a kid's claims; NULL = none
    sort_order             INTEGER NOT NULL DEFAULT 0,                   -- Order in the "earn points" picker
    created_by             TEXT NOT NULL REFERENCES users(id),
    created_at             TEXT NOT NULL,
    updated_at             TEXT NOT NULL,
    archived_at            TEXT
) STRICT;

CREATE INDEX task_templates_household_idx ON task_templates(household_id, archived_at, sort_order);
    -- Template picker list.
```

### `task_template_kids`

Optional restriction of a template to specific kids (e.g. "Mow the lawn" for the older kids only). **No rows means every kid is eligible.**

```sql
CREATE TABLE task_template_kids (
    template_id  TEXT NOT NULL REFERENCES task_templates(id),
    user_id      TEXT NOT NULL REFERENCES users(id),
    PRIMARY KEY (template_id, user_id)
) STRICT, WITHOUT ROWID;
```

### `task_instances`

One concrete thing one kid needs to do, or did. It is the unit of completion and approval and the source of point earnings. It is created by the generator (from `tasks`), when a one-off or scheduled task is created, or when a kid claims a template.

```sql
CREATE TABLE task_instances (
    id                   TEXT PRIMARY KEY,
    household_id         TEXT NOT NULL REFERENCES households(id),
    task_id              TEXT REFERENCES tasks(id),           -- Set for task-derived instances
    template_id          TEXT REFERENCES task_templates(id),  -- Set for template claims
    assignee_id          TEXT NOT NULL REFERENCES users(id),  -- The kid responsible
    -- Snapshots: copied at creation so later edits to the task/template don't rewrite history
    title                TEXT NOT NULL,
    points               INTEGER NOT NULL CHECK (points >= 0),
    requires_approval    INTEGER NOT NULL CHECK (requires_approval IN (0, 1)),  -- Resolved (no NULL)
    required_for_unlock  INTEGER NOT NULL CHECK (required_for_unlock IN (0, 1)),
    -- Scheduling
    occurrence_key       TEXT,          -- Original RRULE occurrence (local datetime) for recurring; idempotency key
    local_date           TEXT NOT NULL, -- Household-local day this instance belongs to ("today's chores")
    starts_at            TEXT,          -- UTC start for timed instances
    ends_at              TEXT,          -- UTC end for timed instances
    all_day              INTEGER NOT NULL CHECK (all_day IN (0, 1)),
    -- Lifecycle
    status               TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
                             'pending',    -- Not done yet
                             'submitted',  -- Kid marked done; awaiting parent review
                             'approved',   -- Points awarded
                             'rejected',   -- Parent sent it back; kid may resubmit (-> submitted)
                             'skipped',    -- Parent excused it (sick day); counts as satisfied for unlock
                             'missed'      -- Day passed without completion (set by a job)
                         )),
    submitted_at         TEXT,
    submission_note      TEXT,          -- Kid's note ("did both bathrooms!")
    photo_key            TEXT,          -- Optional proof photo in object storage
    reviewed_by          TEXT REFERENCES users(id),   -- Parent who approved/rejected/skipped; NULL for auto-approval
    reviewed_at          TEXT,
    review_note          TEXT,          -- Parent feedback ("still socks on the floor")
    created_at           TEXT NOT NULL,
    updated_at           TEXT NOT NULL,
    CHECK ((task_id IS NOT NULL) + (template_id IS NOT NULL) = 1)  -- Exactly one source
) STRICT;

CREATE UNIQUE INDEX task_instances_occurrence_uq
    ON task_instances(task_id, assignee_id, occurrence_key) WHERE task_id IS NOT NULL;
    -- Makes generation idempotent: re-running the job can't create duplicate instances.
CREATE INDEX task_instances_calendar_idx ON task_instances(household_id, local_date, assignee_id);
    -- Calendar range queries ("week of Nov 9, all kids") and "today's chores".
CREATE INDEX task_instances_review_idx ON task_instances(household_id, submitted_at) WHERE status = 'submitted';
    -- Parent approval queue, oldest first.
CREATE INDEX task_instances_template_claims_idx ON task_instances(template_id, assignee_id, local_date)
    WHERE template_id IS NOT NULL;
    -- Enforces max_claims_per_day and cooldown_minutes for template claims.
CREATE INDEX task_instances_unlock_idx ON task_instances(assignee_id, local_date, status)
    WHERE required_for_unlock = 1;
    -- Device-lock evaluation: "does kid K have any unfinished required chores today?"
```

---

## Points

### `point_transactions`

The append-only points ledger. A kid's balance is `SUM(amount)`.

```sql
CREATE TABLE point_transactions (
    id                 TEXT PRIMARY KEY,
    household_id       TEXT NOT NULL REFERENCES households(id),
    kid_id             TEXT NOT NULL REFERENCES users(id),
    amount             INTEGER NOT NULL CHECK (amount <> 0),   -- Signed: +earn, -spend
    type               TEXT NOT NULL CHECK (type IN (
                           'earn',          -- Approved task instance (+)
                           'spend',         -- Store purchase (-)
                           'refund',        -- Cancelled purchase (+)
                           'convert_out',   -- Points converted to cash (-)
                           'adjust'         -- Manual parent correction or bonus (+/-)
                       )),
    task_instance_id   TEXT REFERENCES task_instances(id),  -- earn
    purchase_id        TEXT REFERENCES purchases(id),       -- spend / refund
    conversion_id      TEXT REFERENCES conversions(id),     -- convert_out
    note               TEXT,                                -- Required in practice for 'adjust' ("birthday bonus")
    created_by         TEXT REFERENCES users(id),           -- Acting user; NULL for system jobs
    created_at         TEXT NOT NULL,
    CHECK (type <> 'earn'        OR task_instance_id IS NOT NULL),
    CHECK (type NOT IN ('spend', 'refund') OR purchase_id IS NOT NULL),
    CHECK (type <> 'convert_out' OR conversion_id IS NOT NULL)
) STRICT;

CREATE INDEX point_tx_kid_idx ON point_transactions(kid_id, created_at);
    -- Balance (SUM) and the kid's history feed, newest first.
CREATE UNIQUE INDEX point_tx_earn_uq ON point_transactions(task_instance_id) WHERE type = 'earn';
    -- A task instance can award points at most once, even if approval is double-clicked.
CREATE UNIQUE INDEX point_tx_purchase_uq ON point_transactions(purchase_id, type) WHERE purchase_id IS NOT NULL;
    -- One spend and at most one refund per purchase.
```

Balance checks before a spend happen in the same write transaction as the insert (SQLite serializes writers), so a balance can't go negative from racing requests.

---

## Store

### `store_items`

What kids can buy with points: prizes, privileges, events.

```sql
CREATE TABLE store_items (
    id                     TEXT PRIMARY KEY,
    household_id           TEXT NOT NULL REFERENCES households(id),
    name                   TEXT NOT NULL,
    description            TEXT,
    image_key              TEXT,
    cost_points            INTEGER NOT NULL CHECK (cost_points > 0),
    stock                  INTEGER CHECK (stock >= 0),   -- Remaining quantity; NULL = unlimited
    requires_fulfillment   INTEGER NOT NULL DEFAULT 0 CHECK (requires_fulfillment IN (0, 1)),
        -- 1 = a parent must deliver it (zoo trip, toy); purchases enter the fulfillment queue
    min_age                INTEGER CHECK (min_age >= 0),  -- Age gating via users.birthdate; NULL = any age
    max_per_kid_per_week   INTEGER CHECK (max_per_kid_per_week > 0), -- e.g. "30 min extra screen time" x3/week; NULL = unlimited
    is_active              INTEGER NOT NULL DEFAULT 1 CHECK (is_active IN (0, 1)), -- Temporarily hide without archiving
    sort_order             INTEGER NOT NULL DEFAULT 0,
    created_by             TEXT NOT NULL REFERENCES users(id),
    created_at             TEXT NOT NULL,
    updated_at             TEXT NOT NULL,
    archived_at            TEXT
) STRICT;

CREATE INDEX store_items_household_idx ON store_items(household_id, is_active, archived_at, sort_order);
    -- Storefront listing.
```

### `store_item_kids`

Optional restriction of an item to specific kids. **No rows means available to all kids** (subject to `min_age`).

```sql
CREATE TABLE store_item_kids (
    item_id  TEXT NOT NULL REFERENCES store_items(id),
    user_id  TEXT NOT NULL REFERENCES users(id),
    PRIMARY KEY (item_id, user_id)
) STRICT, WITHOUT ROWID;
```

### `purchases`

A kid buying an item. Writes a `spend` point transaction, decrements stock, and optionally enters the fulfillment queue.

```sql
CREATE TABLE purchases (
    id               TEXT PRIMARY KEY,
    household_id     TEXT NOT NULL REFERENCES households(id),
    item_id          TEXT NOT NULL REFERENCES store_items(id),
    kid_id           TEXT NOT NULL REFERENCES users(id),
    quantity         INTEGER NOT NULL DEFAULT 1 CHECK (quantity > 0),
    item_name        TEXT NOT NULL,                       -- Snapshot of store_items.name
    unit_cost_points INTEGER NOT NULL CHECK (unit_cost_points > 0),  -- Snapshot of price at purchase time
    total_points     INTEGER NOT NULL CHECK (total_points > 0),      -- quantity * unit_cost_points
    status           TEXT NOT NULL CHECK (status IN (
                         'completed',             -- No fulfillment needed; done
                         'awaiting_fulfillment',  -- Parent must deliver
                         'fulfilled',             -- Parent delivered
                         'cancelled'              -- Refunded (refund point transaction posted, stock restored)
                     )),
    resolved_by      TEXT REFERENCES users(id),           -- Parent who fulfilled/cancelled
    resolved_at      TEXT,
    parent_note      TEXT,                                -- e.g. "Zoo trip scheduled for Saturday"
    created_at       TEXT NOT NULL,
    updated_at       TEXT NOT NULL
) STRICT;

CREATE INDEX purchases_kid_idx ON purchases(kid_id, created_at);
    -- Kid's purchase history; per-week limit checks.
CREATE INDEX purchases_fulfillment_idx ON purchases(household_id, created_at) WHERE status = 'awaiting_fulfillment';
    -- Parent fulfillment queue.
```

---

## Cash

### `conversion_rates`

The household's points-to-cash rate over time. The rate in effect is the newest row with `effective_from <= now`, so changing the rate never alters past conversions.

```sql
CREATE TABLE conversion_rates (
    id               TEXT PRIMARY KEY,
    household_id     TEXT NOT NULL REFERENCES households(id),
    points           INTEGER NOT NULL CHECK (points > 0),  -- e.g. 100 points ...
    cents            INTEGER NOT NULL CHECK (cents > 0),   -- ... = 100 cents ($1.00)
    min_points       INTEGER NOT NULL DEFAULT 0 CHECK (min_points >= 0),  -- Smallest allowed conversion
    cooldown_hours   INTEGER CHECK (cooldown_hours > 0),   -- Minimum time between a kid's conversions; NULL = none
    effective_from   TEXT NOT NULL,
    created_by       TEXT NOT NULL REFERENCES users(id),
    created_at       TEXT NOT NULL
) STRICT;

CREATE INDEX conversion_rates_current_idx ON conversion_rates(household_id, effective_from);
    -- "Current rate" lookup and rate history.
```

### `conversions`

One points-to-cash conversion. It links the `convert_out` point transaction and the `convert_in` cash transaction, both posted in the same database transaction.

```sql
CREATE TABLE conversions (
    id             TEXT PRIMARY KEY,
    household_id   TEXT NOT NULL REFERENCES households(id),
    kid_id         TEXT NOT NULL REFERENCES users(id),
    rate_id        TEXT NOT NULL REFERENCES conversion_rates(id),  -- Rate used (for audit)
    points         INTEGER NOT NULL CHECK (points > 0),
    cents          INTEGER NOT NULL CHECK (cents > 0),  -- floor(points * rate.cents / rate.points); only whole multiples of rate.points are allowed, so there's no remainder
    created_at     TEXT NOT NULL
) STRICT;

CREATE INDEX conversions_kid_idx ON conversions(kid_id, created_at);
    -- History and cooldown enforcement.
```

### `cash_transactions`

The append-only cash wallet ledger, in cents. Balance is `SUM(amount_cents)`.

```sql
CREATE TABLE cash_transactions (
    id                          TEXT PRIMARY KEY,
    household_id                TEXT NOT NULL REFERENCES households(id),
    kid_id                      TEXT NOT NULL REFERENCES users(id),
    amount_cents                INTEGER NOT NULL CHECK (amount_cents <> 0),  -- Signed
    type                        TEXT NOT NULL CHECK (type IN (
                                    'convert_in',   -- From points (+)
                                    'withdrawal',   -- Parent confirmed real cash handed over (-)
                                    'invest',       -- Moved into an investment account (-)
                                    'divest',       -- Moved back from an investment account (+)
                                    'adjust'        -- Manual correction, allowance, gift (+/-)
                                )),
    conversion_id               TEXT REFERENCES conversions(id),
    withdrawal_id               TEXT REFERENCES withdrawals(id),
    investment_transaction_id   TEXT REFERENCES investment_transactions(id),
    note                        TEXT,
    created_by                  TEXT REFERENCES users(id),
    created_at                  TEXT NOT NULL,
    CHECK (type <> 'convert_in' OR conversion_id IS NOT NULL),
    CHECK (type <> 'withdrawal' OR withdrawal_id IS NOT NULL),
    CHECK (type NOT IN ('invest', 'divest') OR investment_transaction_id IS NOT NULL)
) STRICT;

CREATE INDEX cash_tx_kid_idx ON cash_transactions(kid_id, created_at);
    -- Balance and wallet history.
CREATE UNIQUE INDEX cash_tx_withdrawal_uq ON cash_transactions(withdrawal_id) WHERE withdrawal_id IS NOT NULL;
    -- A withdrawal can be posted only once.
```

### `withdrawals`

A kid's request to receive real money, confirmed by a parent once handed over. Nothing hits the ledger until confirmation. While a request is pending, its amount is subtracted from the kid's **available** balance (`balance - pending withdrawals`) so it can't be double-spent.

```sql
CREATE TABLE withdrawals (
    id             TEXT PRIMARY KEY,
    household_id   TEXT NOT NULL REFERENCES households(id),
    kid_id         TEXT NOT NULL REFERENCES users(id),
    amount_cents   INTEGER NOT NULL CHECK (amount_cents > 0),
    status         TEXT NOT NULL DEFAULT 'requested' CHECK (status IN (
                       'requested',   -- Awaiting parent
                       'confirmed',   -- Parent paid out; 'withdrawal' cash transaction posted
                       'declined',    -- Parent said no
                       'cancelled'    -- Kid withdrew the request
                   )),
    kid_note       TEXT,              -- "For the book fair"
    method         TEXT,              -- How it was paid: "cash", "Venmo"; set on confirm
    parent_note    TEXT,
    requested_at   TEXT NOT NULL,
    resolved_by    TEXT REFERENCES users(id),
    resolved_at    TEXT
) STRICT;

CREATE INDEX withdrawals_pending_idx ON withdrawals(household_id, requested_at) WHERE status = 'requested';
    -- Parent's pending-withdrawal queue.
CREATE INDEX withdrawals_kid_idx ON withdrawals(kid_id, status);
    -- Available-balance calculation and kid history.
```

---

## Investing

### `investment_accounts`

A pretend investment account owned by a kid. Its balance is `SUM(investment_transactions.amount_cents)`.

```sql
CREATE TABLE investment_accounts (
    id                           TEXT PRIMARY KEY,
    household_id                 TEXT NOT NULL REFERENCES households(id),
    kid_id                       TEXT NOT NULL REFERENCES users(id),
    name                         TEXT NOT NULL,        -- "College Fund", "Bike Savings"
    compounding                  TEXT NOT NULL CHECK (compounding IN ('simple', 'compound')),
        -- simple: interest on net deposited principal; compound: interest on full balance including past interest
    accrual_period               TEXT NOT NULL CHECK (accrual_period IN ('daily', 'weekly', 'monthly')),
    lockup_days                  INTEGER NOT NULL DEFAULT 0 CHECK (lockup_days >= 0),
        -- Each deposit must stay this long before it can be withdrawn penalty-free (FIFO)
    early_withdrawal_penalty_bps INTEGER NOT NULL DEFAULT 0 CHECK (early_withdrawal_penalty_bps BETWEEN 0 AND 10000),
        -- Penalty on locked funds withdrawn early; 0 = early withdrawal not penalized
    allow_early_withdrawal       INTEGER NOT NULL DEFAULT 1 CHECK (allow_early_withdrawal IN (0, 1)),
        -- 0 = locked funds cannot be withdrawn at all
    accrued_through              TEXT,                 -- Local date of the last completed accrual period
    created_by                   TEXT NOT NULL REFERENCES users(id),
    created_at                   TEXT NOT NULL,
    updated_at                   TEXT NOT NULL,
    closed_at                    TEXT                  -- Closed accounts stop accruing; balance must be 0
) STRICT;

CREATE INDEX investment_accounts_kid_idx ON investment_accounts(kid_id);
    -- Kid's accounts list.
CREATE INDEX investment_accounts_accrual_idx ON investment_accounts(accrued_through) WHERE closed_at IS NULL;
    -- Interest job finds accounts with periods due.
```

### `investment_rates`

The return rate per account over time. Accrual for a period uses the rate effective at the **start** of that period, so changing the rate never rewrites past returns.

```sql
CREATE TABLE investment_rates (
    id               TEXT PRIMARY KEY,
    account_id       TEXT NOT NULL REFERENCES investment_accounts(id),
    rate_bps         INTEGER NOT NULL CHECK (rate_bps >= -10000),
        -- Return per accrual_period in basis points (500 = 5% per period). Negative allowed to teach that markets go down
    effective_from   TEXT NOT NULL,     -- Local date
    created_by       TEXT NOT NULL REFERENCES users(id),
    created_at       TEXT NOT NULL
) STRICT;

CREATE UNIQUE INDEX investment_rates_uq ON investment_rates(account_id, effective_from);
    -- One rate per account per effective date; also the "rate at date" lookup.
```

### `investment_transactions`

The append-only investment ledger. Deposits and withdrawals pair with `invest`/`divest` cash transactions in the same database transaction.

```sql
CREATE TABLE investment_transactions (
    id              TEXT PRIMARY KEY,
    household_id    TEXT NOT NULL REFERENCES households(id),
    account_id      TEXT NOT NULL REFERENCES investment_accounts(id),
    kid_id          TEXT NOT NULL REFERENCES users(id),     -- Denormalized for kid-wide net-worth queries
    amount_cents    INTEGER NOT NULL CHECK (amount_cents <> 0),  -- Signed
    type            TEXT NOT NULL CHECK (type IN (
                        'deposit',    -- From wallet (+)
                        'interest',   -- Accrued return (+, or - if rate is negative)
                        'withdraw',   -- To wallet (-)
                        'penalty',    -- Early-withdrawal penalty (-)
                        'adjust'      -- Manual correction (+/-)
                    )),
    rate_id         TEXT REFERENCES investment_rates(id),  -- interest: rate applied
    period_start    TEXT,                                  -- interest: local date period began
    period_end      TEXT,                                  -- interest: local date period ended
    note            TEXT,
    created_by      TEXT REFERENCES users(id),             -- NULL for the interest job
    created_at      TEXT NOT NULL,
    CHECK (type <> 'interest' OR (rate_id IS NOT NULL AND period_start IS NOT NULL AND period_end IS NOT NULL))
) STRICT;

CREATE INDEX investment_tx_account_idx ON investment_transactions(account_id, created_at);
    -- Balance, history, growth chart, and FIFO lockup calculation.
CREATE UNIQUE INDEX investment_tx_interest_uq ON investment_transactions(account_id, period_start) WHERE type = 'interest';
    -- Interest is posted at most once per period, so re-running the job after a crash is safe.
```

Interest amounts are computed in `kc-core` and rounded **down** to whole cents (half-cent remainders are dropped, never invented).

---

## Devices

### `agents`

Local `kc-agent` processes running at home. Each authenticates with its own token and polls for desired device states.

```sql
CREATE TABLE agents (
    id             TEXT PRIMARY KEY,
    household_id   TEXT NOT NULL REFERENCES households(id),
    name           TEXT NOT NULL,          -- "Living room mini PC"
    token_hash     TEXT NOT NULL UNIQUE,   -- SHA-256 of the agent's bearer token
    last_seen_at   TEXT,                   -- Updated on each poll; UI shows agent health
    version        TEXT,                   -- Reported agent version
    created_by     TEXT NOT NULL REFERENCES users(id),
    created_at     TEXT NOT NULL,
    revoked_at     TEXT
) STRICT;
```

### `devices`

A controllable device (smart plug, TV) and the adapter used to control it.

```sql
CREATE TABLE devices (
    id               TEXT PRIMARY KEY,
    household_id     TEXT NOT NULL REFERENCES households(id),
    name             TEXT NOT NULL,          -- "Switch (plug)", "Basement TV"
    kind             TEXT NOT NULL CHECK (kind IN ('smart_plug', 'tv', 'console', 'other')),
    adapter          TEXT NOT NULL CHECK (adapter IN ('shelly', 'kasa', 'smartthings', 'lg_thinq')),
    agent_id         TEXT REFERENCES agents(id),
        -- Local adapters (shelly, kasa) must have an agent; cloud adapters are driven by the server and leave it NULL
    adapter_config   TEXT NOT NULL DEFAULT '{}' CHECK (json_valid(adapter_config)),
        -- Adapter-specific JSON: local IP, device ID. Cloud credentials are stored as secrets, never here
    lock_enabled     INTEGER NOT NULL DEFAULT 1 CHECK (lock_enabled IN (0, 1)),  -- Master switch for chore locking
    desired_state    TEXT NOT NULL DEFAULT 'unlocked' CHECK (desired_state IN ('locked', 'unlocked')),
        -- Computed by the lock-evaluation job; what agents/adapters should enforce
    reported_state   TEXT CHECK (reported_state IN ('locked', 'unlocked', 'unknown')),  -- Last state reported back
    reported_at      TEXT,
    last_error       TEXT,                   -- Last adapter error, shown to parents
    created_at       TEXT NOT NULL,
    updated_at       TEXT NOT NULL,
    archived_at      TEXT
) STRICT;

CREATE INDEX devices_agent_idx ON devices(agent_id) WHERE archived_at IS NULL;
    -- An agent's poll: "which devices do I manage and what state should they be in?"
CREATE INDEX devices_household_idx ON devices(household_id);
```

**Lock rule:** a device is `locked` if any linked kid has an instance for today (household-local `local_date`) with `required_for_unlock = 1` and status not in (`approved`, `skipped`), unless an active `device_overrides` row says otherwise. If a device is shared by several kids, **all** of them must be done.

### `device_kids`

Which kids' chores gate a device.

```sql
CREATE TABLE device_kids (
    device_id  TEXT NOT NULL REFERENCES devices(id),
    user_id    TEXT NOT NULL REFERENCES users(id),
    PRIMARY KEY (device_id, user_id)
) STRICT, WITHOUT ROWID;

CREATE INDEX device_kids_user_idx ON device_kids(user_id);
    -- When a kid's instance is approved, find their devices to re-evaluate immediately.
```

### `device_overrides`

A parent's manual override, such as "unlock the TV for the next 2 hours" or "lock it until tomorrow".

```sql
CREATE TABLE device_overrides (
    id           TEXT PRIMARY KEY,
    device_id    TEXT NOT NULL REFERENCES devices(id),
    state        TEXT NOT NULL CHECK (state IN ('locked', 'unlocked')),
    starts_at    TEXT NOT NULL,
    ends_at      TEXT,               -- NULL = until manually cleared
    reason       TEXT,
    created_by   TEXT NOT NULL REFERENCES users(id),
    created_at   TEXT NOT NULL,
    cleared_at   TEXT                -- Ended early by a parent
) STRICT;

CREATE INDEX device_overrides_active_idx ON device_overrides(device_id, starts_at) WHERE cleared_at IS NULL;
    -- Lock evaluation: newest active override for a device wins.
```

---

## Operations

### `audit_log`

A record of administrative changes that aren't already captured by a ledger, such as rate changes, task edits, approvals, role and PIN resets, and overrides. Ledgers cover money and points, and this table covers "who changed what".

```sql
CREATE TABLE audit_log (
    id             TEXT PRIMARY KEY,
    household_id   TEXT NOT NULL REFERENCES households(id),
    actor_id       TEXT REFERENCES users(id),   -- NULL for system jobs
    action         TEXT NOT NULL,               -- Dotted verb, e.g. "task.update", "instance.approve", "pin.reset"
    entity_type    TEXT NOT NULL,               -- Table name of the affected row
    entity_id      TEXT NOT NULL,
    details        TEXT CHECK (details IS NULL OR json_valid(details)),  -- JSON diff/context; never secrets
    created_at     TEXT NOT NULL
) STRICT;

CREATE INDEX audit_log_household_idx ON audit_log(household_id, created_at);
    -- Parent activity feed.
CREATE INDEX audit_log_entity_idx ON audit_log(entity_type, entity_id);
    -- History of a single row.
```

### `job_runs`

Bookkeeping for background jobs, so jobs can tell when they last ran and parents can see job health.

```sql
CREATE TABLE job_runs (
    job_name        TEXT PRIMARY KEY,   -- "generate_instances", "accrue_interest", "evaluate_locks", "mark_missed", "cleanup_sessions"
    last_started_at TEXT,
    last_success_at TEXT,
    last_error      TEXT,
    last_error_at   TEXT
) STRICT, WITHOUT ROWID;
```

All jobs are written to be **idempotent**, relying on the unique indexes above (`task_instances_occurrence_uq`, `investment_tx_interest_uq`, `point_tx_earn_uq`). A crash or an overlapping run can't double-post anything.

---

## Background Jobs ↔ Tables

| Job | Cadence | Reads | Writes |
|---|---|---|---|
| `generate_instances` | Hourly | `tasks`, `task_assignees`, `task_occurrence_overrides` | `task_instances`, `tasks.generated_through` |
| `mark_missed` | Hourly (after each household's local midnight) | `task_instances` | `task_instances.status = 'missed'` |
| `accrue_interest` | Hourly (posts only completed periods) | `investment_accounts`, `investment_rates`, `investment_transactions` | `investment_transactions` (interest), `accrued_through` |
| `evaluate_locks` | Every minute, plus immediately on approval | `task_instances`, `device_kids`, `device_overrides` | `devices.desired_state` |
| `cleanup_sessions` | Daily | `sessions` | Deletes expired rows |

## Derived Values (not stored)

| Value | Computed as |
|---|---|
| Points balance | `SUM(point_transactions.amount) WHERE kid_id = ?` |
| Cash balance | `SUM(cash_transactions.amount_cents) WHERE kid_id = ?` |
| Available cash | Cash balance minus `SUM(withdrawals.amount_cents) WHERE status = 'requested'` |
| Investment balance | `SUM(investment_transactions.amount_cents) WHERE account_id = ?` |
| Net worth (cents) | Cash balance + all investment balances (points shown separately) |
| Today's unlock status | See the lock rule under `devices` |

Balances are cheap to compute at family scale thanks to the `(kid_id, created_at)` indexes. A cached balance table can be added later if needed, but the ledger stays the source of truth.
