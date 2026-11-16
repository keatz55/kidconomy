# Kidconomy

**Kid + Economy.** A family app for tracking chores, schedules, and points, with a store, a cash wallet, and a pretend investment account so kids learn how earning, spending, saving, and investing work.

> Status: design phase. This document is the working spec. Nothing is implemented yet.

---

## Goals

- Track chores and schedules for each child in a household, with a rich calendar.
- Award points for **one-off**, **scheduled**, and **recurring** tasks.
- Let kids spend points in a household **store** on prizes, events, privileges, or any inventory the parents add.
- Let kids **convert points to cash** and track their money balance.
- Let kids move cash into a pretend **investment account** that earns a return the parents set.
- Optionally **lock devices** (game consoles, TVs) until the day's required chores are done.

## Core Concepts

### Households

A household works like a workspace. Every record (users, tasks, store items, ledgers) belongs to exactly one household, and each household has a time zone so "today" is well defined. The app is multi-household from day one, though a typical install will have just one.

### Users and roles

| Role   | Login             | Capabilities                                                                 |
|--------|-------------------|------------------------------------------------------------------------------|
| Parent | Email + password  | Full admin: manage tasks, store, rates, approvals, withdrawals, devices      |
| Kid    | Profile + **PIN** | Complete tasks, claim templates, shop, convert points, invest, view balances |

Kids pick their profile on a shared device and enter a PIN. They don't need an email address.

### Tasks

All tasks share one model and differ in how they're scheduled:

- **One-off**: a single task with an optional due date.
- **Scheduled**: a task at a specific date and time.
- **Recurring**: defined with an [RFC 5545 RRULE](https://datatracker.ietf.org/doc/html/rfc5545#section-3.3.10) (e.g. `FREQ=WEEKLY;BYDAY=MO,WE,FR`). The server is the source of truth: it expands rules into concrete **task instances**, and kids complete instances rather than the rule itself.

Each task has a point value, assignee(s), an optional "required for device unlock" flag, and a configurable **`requires_approval`** setting. A household default applies to new tasks and templates, and each task can override it.

### Task templates

Templates are pre-filled tasks for impromptu earning, such as "Take out the trash" or "Unload the dishwasher". A kid can **claim** a template whenever they do the chore. Claiming creates a one-off task instance that goes through the normal completion and approval flow. Parents can set limits, such as a maximum number of claims per day.

### Completion → Approval → Points

1. A kid marks a task instance complete (optionally with a note or photo).
2. If the task has `requires_approval = true`, it waits in the parent approval queue, where a parent approves it or rejects it with an optional note (e.g. "still socks on the floor"). If `requires_approval = false`, it is approved automatically.
3. On approval, a points `earn` transaction is posted to the kid's ledger.

### Ledgers (append-only)

Balances are never stored as mutable numbers. Each account is an append-only ledger of transactions, and the balance is the sum of its transactions. This gives a full audit trail ("where did my points go?").

| Ledger      | Unit                          | Transaction types                                                                     |
|-------------|-------------------------------|---------------------------------------------------------------------------------------|
| Points      | integer points                | `earn`, `spend`, `refund`, `convert_out`, `adjust`                                    |
| Cash wallet | integer **cents** (no floats) | `convert_in`, `withdrawal`, `invest`, `divest`, `adjust`                              |
| Investment  | integer cents                 | `deposit`, `interest`, `withdraw`, `penalty`, `adjust`                                |

Transfers between ledgers (e.g. points → cash) are posted atomically as paired transactions. Withdrawal *requests* live in their own table and only hit the cash ledger once a parent confirms them. See **[docs/database-schema.md](docs/database-schema.md)** for the full schema.

### Store

Parents manage inventory items with:

- Point cost
- Optional stock count
- Optional "requires parent fulfillment" (e.g. a trip to the zoo), which creates a fulfillment queue for parents
- Optional per-kid availability or age gating

### Points → Cash

- A household-level conversion rate (e.g. 100 points = $1.00), with rate history.
- Optional minimum conversion amount and cooldown.

### Cash withdrawals

Kidconomy **tracks** money and never moves real money. When a kid wants real cash, they create a withdrawal request. A parent hands over the cash (or sends it by Venmo, etc.) and **confirms** the withdrawal in the app, which then posts it to the ledger.

### Investment account

- Parents set the rate of return (e.g. 5% per month), with **rate history**, so changing the rate never rewrites past returns.
- Configurable **simple or compound** interest and accrual period (daily, weekly, monthly).
- Optional lock-up period and early-withdrawal penalty, which teach the value of waiting.
- A background job posts `interest` transactions.

### Device lockout (exploratory)

The goal is to keep a kid's devices locked until the day's required tasks are approved. The rule: *"devices X, Y for kid K are locked until all tasks flagged `required_for_unlock` today are approved."* Parents can always override.

| Target                         | Feasibility | Notes                                                                                           |
|--------------------------------|-------------|-------------------------------------------------------------------------------------------------|
| **Nintendo Switch**            | Poor        | No public parental-controls API. Unofficial APIs are fragile and may violate ToS.               |
| **Samsung TVs**                | Possible    | SmartThings has a public cloud API for power and input control.                                 |
| **LG TVs**                     | Possible    | LG ThinQ Connect cloud API.                                                                     |
| **Smart plugs (Shelly, Kasa)** | Good        | Cut power to a TV or console (including the Switch). Cheap, simple local HTTP APIs.             |
| **AT&T gateway**               | Poor        | Device pausing is only available through AT&T's Smart Home Manager app. There is no public API. |
| **Home Assistant / UniFi**     | Good        | Not used in our home, but a possible adapter for other households.                              |

**Plan:** the server runs on Fly.io and can't reach devices on the home network. A small **local agent** (`kc-agent`) runs at home on any always-on machine. It polls the API for desired device states ("Switch plug: off") and drives local devices. Adapters implement a shared `DeviceController` trait. Smart plugs come first, and cloud adapters (SmartThings, LG ThinQ) can run directly on the server.

---

## Architecture

```
kidconomy/
├── Cargo.toml              # [workspace] root
├── crates/
│   ├── kc-core/            # Domain types & pure logic (ledgers, interest, RRULE expansion, approvals) — no I/O
│   ├── kc-db/              # SQLite persistence (sqlx), migrations
│   ├── kc-devices/         # DeviceController trait + adapters (smart plugs, SmartThings, LG ThinQ)
│   ├── kc-api/             # axum server: JSON API, auth, background jobs, serves the built frontend
│   └── kc-agent/           # Local device agent (runs at home, polls kc-api, controls smart plugs)
├── web/                    # React + Vite + Mantine frontend
├── migrations/
├── Dockerfile
├── fly.toml
└── litestream.yml
```

- **Rust owns all business logic.** The frontend only displays data and sends actions, so no rules live in the browser. For example, a kid can't award themselves points from dev tools.
- **`kc-core`** has no I/O, so all money, points, interest, and scheduling math can be unit tested in isolation.
- **Shared types:** `utoipa` generates an OpenAPI spec from the Rust API, and a typed TypeScript client is generated from it, so the frontend and backend can't drift apart.

### Stack

**Backend (Rust)**
- `axum` / `tokio`: HTTP server, async runtime, and in-process background jobs (task-instance generation, interest accrual, device-lock evaluation)
- `sqlx` + **SQLite**: compile-time checked queries and migrations
- `rrule`: RRULE parsing and expansion
- `chrono` / `chrono-tz`: time zones
- `argon2`: password and PIN hashing
- `utoipa`: OpenAPI spec
- `rust-embed`: embeds the built frontend into the binary

**Frontend (TypeScript)**
- **React + Vite**: a single-page app rendered client-side (SEO is irrelevant for a private family app). It can be installed as a PWA on tablets.
- **Mantine** UI, with **`@mantine/schedule`** for the calendar (Day/Week/Month/Year views, drag and drop, and **resource views with one row per kid**). Parts of the schedule docs are still on Mantine's alpha site, so pin versions.
- `rrule.js`: previews only (e.g. "next 5 dates" while a parent builds a rule). The server stays authoritative.

### Development

Run two processes for fast iteration:

- `cargo run -p kc-api`: API on `localhost:3000`
- `npm run dev` (in `web/`): Vite dev server on `localhost:5173` with hot reload. It **proxies `/api` to `:3000`**, so the browser has a single origin with no CORS, and auth cookies just work.

A single `just dev` command will start both.

### Production

**One binary, one SQLite file.** `vite build` outputs static files that are embedded into `kc-api`. The server routes `/api/*` to handlers and serves everything else from the embedded frontend, falling back to `index.html` for client-side routes.

### Hosting: Fly.io

- A single Docker image (multi-stage build: the Node build of `web/`, then the Rust release build) deployed with `fly deploy`.
- The smallest shared-CPU machine (256–512 MB), **always on** (no auto-stop), so background jobs keep running.
- SQLite lives on a **Fly volume** (persistent disk, ~1 GB).
- **Litestream** continuously replicates the database to S3-compatible storage (Fly Tigris or Cloudflare R2), giving point-in-time restore.
- Estimated cost: about **$2–5/month**.
- Portable: the same binary and database file can move to a VPS or home server later.

## Decisions

- [x] **Households** work as workspaces (multi-household schema)
- [x] **Kid login:** profile + PIN. **Parent login:** email + password
- [x] **Recurring tasks:** RRULE, expanded server-side
- [x] **Approvals:** configurable `requires_approval`, with a household default and per-task override
- [x] **Withdrawals:** parent-confirmed. The app tracks money only
- [x] **Database:** SQLite (sqlx), with Litestream backups
- [x] **Backend:** Rust (axum)
- [x] **Frontend:** React + Vite + Mantine + `@mantine/schedule`
- [x] **Hosting:** Fly.io
- [ ] **Device control:** first adapter (Shelly or Kasa smart plugs vs. SmartThings / LG ThinQ), and which home machine runs `kc-agent`

## Roadmap

1. Workspace skeleton, `kc-core` types, ledger model
2. `kc-api` + `kc-db` skeleton, `web/` skeleton, dev proxy, Fly deploy with Litestream
3. Households, users, auth (parent password, kid PIN)
4. Tasks: one-off → scheduled → recurring (RRULE), completions and approvals, calendar UI
5. Task templates and impromptu claims
6. Store, purchases, fulfillment queue
7. Points → cash conversion, wallet, parent-confirmed withdrawals
8. Investment accounts and the interest job
9. Device lockout (`kc-devices`, `kc-agent`)
10. Dashboards (balances, savings growth charts)
