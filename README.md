# Continuum — plan

**A small durable execution engine for a checkout workflow.**
Every state change is an append-only journal event in PostgreSQL. Every payment effect is booked exactly once, even when the worker dies between the effect and the step completion.

> **Status: plan, not implemented.** This repository contains the product plan, the architecture, the demo script and the acceptance criteria. There is no engine code yet. No benchmark is published until it is measured.

[![status](https://img.shields.io/badge/status-plan-6C63FF?style=flat-square)](#status)
[![stack](https://img.shields.io/badge/stack-Rust%20%C2%B7%20Tokio%20%C2%B7%20Axum%20%C2%B7%20SQLx%20%C2%B7%20PostgreSQL-000000?style=flat-square)](#planned-stack)
[![license](https://img.shields.io/badge/license-MIT-blue?style=flat-square)](LICENSE)

## The problem

A workflow that lives only in memory loses its progress when the process dies. Retrying a step can charge a customer twice. A durable engine has to answer two questions with evidence:

1. **What happened?** An append-only journal, not a mutable `runs` row.
2. **How many times did the side effect happen?** Exactly once per effect, enforced by a unique idempotency key and a transaction.

Continuum is a deliberately small engine that demonstrates both on one page: start a checkout, kill the worker mid-run, watch the run recover from the journal, and see exactly one charge in the ledger.

It is not a Temporal competitor. It is the smallest complete system that makes the hard guarantees visible.

## Why this project

The market study behind the portfolio (100 backend postings in Spain and Switzerland/Germany, plus 58 live postings reviewed in November 2026) found:

- **Rust appears in 15% of Spanish backend postings** and **70.6% of Swiss/German postings**.
- The Rust roles that pay are systems, distributed systems, infrastructure and trading — not CRUD.
- A desktop Rust project does not prove backend/systems ability. A durable engine does.

Continuum targets the Rust/systems market. Caliber, the second project, targets the Spanish Python/RAG market.

## The 60-second demo (what the finished MVP will show)

```text
docker compose up  ->  http://localhost:3000
Start checkout     ->  charge in progress
CHAOS              ->  worker dies, run recovers
Ledger             ->  exactly one charge
```

The page shows the run list, the journal, the ledger and a **CHAOS** button. The reviewer starts a checkout, kills the worker while `charge` is in progress, and watches the run move `recovering → running → completed`. The ledger for that run contains one charge.

See [docs/DEMO.md](docs/DEMO.md) for the exact script and the expected observable result.

## Scope (MVP)

- **Workflow:** `checkout` with `reserve`, `charge` (mock payment ledger) and `notify`.
- **Journal:** event-sourced in PostgreSQL (`run_started`, `step_started`, `step_completed`, `step_failed`, `timer_fired`, `signal_received`, `effect_applied`, `run_completed`). Run state is reconstructed only from the journal.
- **Worker:** bounded queue, retries with backoff, timeouts, durable timers and signals.
- **Exactly-once:** deterministic idempotency key per effect; ledger row + `effect_applied` in one transaction; duplicate application returns the existing row.
- **Recovery:** on startup, incomplete runs resume from the journal; a crash-injection test kills a worker mid-run and asserts correct progress with zero duplicate effects.
- **Demo:** one page with start, run list, journal, ledger and CHAOS; polling every second.
- **Evidence:** CI from the first commit, a real benchmark, four ADRs and learning docs in Spanish.

## Non-goals

- A general-purpose workflow platform or a Temporal/Cadence competitor.
- A distributed cluster, leader election or replication.
- A real payment provider integration.
- Workflow versioning for in-flight runs.
- Authentication, TLS or multi-tenancy.
- A production deployment in the first milestone.

## Planned stack

Rust stable · Tokio · Axum · SQLx · PostgreSQL 16 · Docker Compose · GitHub Actions · Criterion or a small benchmark binary.

The mock ledger lives in PostgreSQL so the effect and its journal event can commit atomically. A real payment provider would need provider-side idempotency keys and an outbox; that is a documented follow-up, not the MVP.

## Planned architecture

```text
browser (vanilla JS, polling every 1 s)
   |
   v
Axum HTTP API
   |-- POST /runs, POST /runs/{id}/signal, POST /chaos, GET /runs/{id}
   |
   v
journal_events (PostgreSQL)  <-- append-only, one row per event
   |                                  ^
   | fold on read                     | enqueue
   v                                  |
run state                      bounded queue -> worker pool
                                      |
                                      v
                              workflow interpreter
                              reserve -> charge -> notify
                                      |
                                      v
                              ledger_entries
                              UNIQUE(idempotency_key)
                              ledger row + EffectApplied in one transaction
```

There is no run-state table. `GET /runs/{id}` loads the journal and folds it into state. Recovery uses the same fold. A process that has never seen the run before can finish it.

The full plan is in [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Milestones

| Phase | Deliverable | Exit criterion |
|---|---|---|
| **P0 — Skeleton** | Crate, CI, Docker Compose, config, health endpoint | CI green with fmt, clippy and an empty test suite |
| **P1 — Journal** | Schema, append with per-run sequence, fold, incomplete-run query | Concurrent appends keep a contiguous sequence; fold tests pass |
| **P2 — Workflow** | Checkout definition, interpreter, retries, timers, signals | A run advances through the three steps with durable events |
| **P3 — Exactly-once** | Ledger with unique idempotency key, atomic effect + event | Applying the same effect twice books one row |
| **P4 — Recovery** | Startup replay, recovery sweep, `/chaos` | A worker killed mid-run recovers with one charge |
| **P5 — Demo** | One-page UI, seeded history, CHAOS button | `docker compose up` shows the full beat |
| **P6 — Evidence** | Real benchmark, README, four ADRs, three Spanish learning docs | A reviewer can follow the demo and the guarantees in five minutes |

The detailed task list and acceptance criteria are in [docs/PLAN.md](docs/PLAN.md). The roadmap is in [ROADMAP.md](ROADMAP.md).

## Evidence contract

This is the definition of done for the implementation:

1. **CI from the first commit** — `cargo fmt --check`, `cargo clippy -- -D warnings` and `cargo test` against a real PostgreSQL service.
2. **README first screen** — what it is, the 60-second demo, the measured benchmark (only after it exists), then architecture and trade-offs.
3. **Exactly-once is defined honestly** — exactly one ledger booking per effect, not exactly-once delivery of an arbitrary external call.
4. **One-command demo** with seeded data; never an empty screen.
5. **Four ADRs** in `docs/decisions/` and three learning docs in Spanish in `docs/learning/`.
6. **Hygiene** — no secrets, `.env.example` only, correct `.gitignore`, LICENSE, topics and a single `main` branch.
7. **Honest limitations** — the mock ledger is not a payment provider; `/chaos` aborts a task, not a process; the benchmark is a laptop measurement; recovery scans all incomplete runs.

## How to review this plan

1. Read this README (2 minutes).
2. Read [docs/PLAN.md](docs/PLAN.md) — phases, tasks and acceptance criteria (5 minutes).
3. Read [docs/DEMO.md](docs/DEMO.md) — what the reviewer will see in 60 seconds.
4. Read [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) and [docs/DECISIONS.md](docs/DECISIONS.md) — the hard choices before the code.
5. Check the [wiki](https://github.com/artur282/Continuum/wiki) for the same material in a navigable form.

## Limitations of the plan

- The MVP is a single workflow and a single PostgreSQL instance. It is not a distributed system.
- `/chaos` will abort a Tokio task, not send SIGKILL to the process. A real container restart is a manual test; the journal path is the same.
- The benchmark will be a laptop measurement, not a comparison with Temporal, Cadence or a tuned production deployment.
- Workflow definitions will be code-defined and unversioned; changing them while old runs are in flight is not supported.
- The mock ledger is not a payment provider. Atomicity is possible because the effect and the journal share a database.
- The project is not deployed. A hosted demo is a later milestone, not a promise.

## License

MIT — see [LICENSE](LICENSE).

## Links

- Portfolio hub: [github.com/artur282/Portafolio](https://github.com/artur282/Portafolio)
- Second project: [Caliber](https://github.com/artur282/Caliber) — RAG evaluation in Python
- Wiki: [github.com/artur282/Continuum/wiki](https://github.com/artur282/Continuum/wiki)
