# Continuum — implementation plan

> **Status: plan, not implemented.** This document is the source of truth for the first implementation. It is written before the code, deliberately.

## 1. Goal

Build the smallest complete durable engine that a reviewer can understand in **60 seconds** and that proves two capabilities:

1. **Durability:** a run interrupted by a worker crash resumes from the journal.
2. **Exactly-once booking:** the same effect applied twice produces exactly one ledger row.

The MVP is a checkout workflow, one process and one database. It is not a workflow platform.

## 2. Definition of done

- [ ] `docker compose up` starts PostgreSQL 16 and the app with seeded history; the page is never empty.
- [ ] `checkout` runs `reserve → charge → notify` with typed journal events.
- [ ] A worker killed during `charge` recovers from the journal and completes the run.
- [ ] The ledger for that run contains exactly **one** charge.
- [ ] `cargo fmt --check`, `cargo clippy -- -D warnings` and `cargo test` pass locally and in CI against a real PostgreSQL service.
- [ ] The README has the 60-second demo, a real benchmark profile (only when measured), architecture, review path and limitations.
- [ ] Four ADRs and three learning docs in Spanish exist.
- [ ] No secrets; `.env.example` only.

## 3. Phases and tasks

### P0 — Skeleton

| ID | Task | Acceptance |
|---|---|---|
| N0.1 | Create the Rust crate with Tokio, Axum, SQLx, serde and tracing | `cargo check` passes |
| N0.2 | Create `config.rs`, `db.rs`, `main.rs` and `/health` | `GET /health` returns 200 with database status |
| N0.3 | Create `docker-compose.yml` with `db` (postgres:16) and `app` | `docker compose config` valid; both services reach healthy |
| N0.4 | Create `.github/workflows/ci.yml` with a PostgreSQL service | CI runs fmt, clippy and an empty test suite |
| N0.5 | Create `.env.example`, `.gitignore`, `LICENSE` | No secret is committed; `.env.example` is visible |

### P1 — Journal

| ID | Task | Acceptance |
|---|---|---|
| N1.1 | Create `journal_events` with a per-run sequence | One row per event; `UNIQUE(run_id, seq)` |
| N1.2 | Implement append in a transaction with a per-run advisory lock | Concurrent appends keep a contiguous sequence |
| N1.3 | Implement the fold from events to `RunState` | Fold tests cover each event and terminal states |
| N1.4 | Implement the incomplete-run query | Returns only runs without `run_completed` |
| N1.5 | Create the first migration | `sqlx migrate` or the startup migrator applies it on an empty database |

### P2 — Workflow and engine

| ID | Task | Acceptance |
|---|---|---|
| N2.1 | Define the declarative `checkout` workflow with `reserve`, `charge`, `notify` | Unknown workflows return 400 |
| N2.2 | Implement the worker pool with a bounded queue | A full queue does not lose work; the run stays durable |
| N2.3 | Implement the interpreter: gate, effect, completion, retry | A transient failure retries with backoff |
| N2.4 | Implement durable timers from journal timestamps | A retry or gate deadline survives a restart |
| N2.5 | Implement signals (`ship_ack`) stored in the journal | A signal releases the notify gate before its timer |

### P3 — Exactly-once effects

| ID | Task | Acceptance |
|---|---|---|
| N3.1 | Create `ledger_entries` with `UNIQUE(idempotency_key)` | The database is the final arbiter |
| N3.2 | Implement the atomic transaction: ledger row + `effect_applied` | Both commit or neither does |
| N3.3 | Implement the duplicate path: existing row → `duplicate: true` + missing event | Applying twice books one row |
| N3.4 | Add a 12-way concurrent test | One ledger row, one `effect_applied` |
| N3.5 | Add a ledger-row-without-event test | The missing event is appended; no duplicate booking |

### P4 — Recovery and chaos

| ID | Task | Acceptance |
|---|---|---|
| N4.1 | Replay incomplete runs on startup | A fresh engine instance finishes a run it has never seen |
| N4.2 | Add a recovery sweep for runs left behind by a dead worker | The run is enqueued without duplicating effects |
| N4.3 | Implement `POST /chaos` to abort the task that owns a run | The run moves to `recovering`; a new worker picks it up |
| N4.4 | Add the crash matrix with a fixed seed | Killing at reserve, charge or notify never duplicates an effect |
| N4.5 | Verify a real `docker compose restart app` manually | The run resumes and books one charge |
| N4.6 | Release the ownership advisory lock when a task is aborted | A fresh session can acquire the run; test covers it |

### P5 — API and demo

| ID | Task | Acceptance |
|---|---|---|
| N5.1 | `POST /runs`, `GET /runs`, `GET /runs/{id}` | 201/200; unknown run → 404 |
| N5.2 | `POST /runs/{id}/signal` and `POST /chaos` | 202; invalid state → 409 |
| N5.3 | `GET /ledger` and `GET /stats` | Ledger totals and queue/worker counters |
| N5.4 | One-page UI: start, run list, journal, ledger, CHAOS | Vanilla JS, polling every second, no build |
| N5.5 | Seeded history marked `"seed": true` | The page is never empty on first load |
| N5.6 | Verify the full beat: start → charge → CHAOS → recovered → one charge | The observable result matches the demo script |

### P6 — Evidence and closure

| ID | Task | Acceptance |
|---|---|---|
| N6.1 | Run the benchmark with the app stopped and the bench database | Publish throughput, p50/p99, scheduling, recovery, queue latency |
| N6.2 | Write the README first screen, architecture, review path and limitations | A reviewer can follow it in five minutes |
| N6.3 | Write four ADRs: journal as source of truth; exactly-once effects; worker model and recovery; timers, signals and polling UI | Each has context, decision, alternatives, consequences |
| N6.4 | Write three learning docs in Spanish | They explain the engine, how to verify it and the demo |
| N6.5 | Verify CI locally with the same commands as the workflow | fmt, clippy, tests clean; CI pending the first push |
| N6.6 | Add a screenshot or a short recording of the CHAOS beat | The artifact matches the current UI |

## 4. Test strategy

- **Unit:** state fold, retry backoff, gate/timer names, input validation.
- **Integration:** journal append ordering, exactly-once (sequential + concurrent), recovery from a crash after the effect, a crash while waiting for a signal, a pre-applied ledger row, ownership-lock release.
- **Manual:** the 60-second demo, a real Docker restart, the error cases (`400/404/409`).
- **CI:** the same commands against `postgres:16`; no external service.

The tests must not use SQLite. The guarantees depend on PostgreSQL transactions and unique constraints.

## 5. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Session advisory locks survive a task abort and leak through the pool | Release hook on pool return + a test that acquires the lock from a fresh session |
| `/chaos` is not a real process kill | Verify a real container restart manually; document the difference |
| Exactly-once is confused with exactly-once delivery | Define it as exactly-one booking; document the external-provider gap |
| Benchmark numbers are unstable | Report a range, the machine and the app-stopped condition; never a single cherry-picked number |
| Scope creep into a workflow platform | One workflow, one process, one database; non-goals explicit |
| The UI becomes a frontend project | One HTML file and vanilla JS; polling is honest |

## 6. Non-goals

- General-purpose workflow platform, Temporal/Cadence parity.
- Distributed workers, leader election, replication.
- Real payment provider integration.
- Workflow versioning for in-flight runs.
- Auth, TLS, multi-tenancy.
- Hosted demo in the first milestone.

## 7. Deliverables

```text
continuum/
  Cargo.toml
  README.md
  docker-compose.yml
  Dockerfile
  .env.example
  .github/workflows/ci.yml
  migrations/0001_init.sql
  src/
    main.rs  config.rs  db.rs
    models.rs  journal.rs  effects.rs
    engine.rs  workflow.rs  service.rs  api.rs  seed.rs
    bin/bench.rs
  static/index.html  static/app.js
  tests/
    journal.rs  exactly_once.rs  recovery.rs
  docs/
    decisions/0001..0004
    learning/01..03
```

## 8. First implementation order

1. P0.1 → P0.5 (skeleton and CI).
2. P1.1 → P1.5 (journal; verify the concurrent sequence).
3. P2.1 → P2.5 (workflow and engine; verify retries and signals).
4. P3.1 → P3.5 (exactly-once; verify the duplicate path).
5. P4.1 → P4.6 (recovery and chaos; verify the crash matrix).
6. P5.1 → P5.6 (API and UI; verify the 60-second beat).
7. P6.1 → P6.6 (benchmark and evidence; publish only measured numbers).

Do not start the UI before P4 is green: a demo that cannot recover is a lie with a button.
