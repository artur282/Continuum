# Continuum — roadmap

> **Status: plan, not implemented.** Dates are intentionally relative: the project moves when the previous milestone is green, not when a calendar says so.

## M0 — Plan (current)

- [x] Product scope, non-goals and market fit.
- [x] Architecture and data model.
- [x] Demo script and acceptance criteria.
- [x] Exactly-once definition and failure model.
- [x] Planned decisions.
- [ ] Review by the owner.
- [ ] First code commit.

## M1 — Skeleton

- [ ] Rust crate, Tokio, Axum, SQLx.
- [ ] `/health`, settings, tracing.
- [ ] PostgreSQL via Docker Compose.
- [ ] CI with fmt, clippy and tests.
- [ ] `.env.example`, `.gitignore`, LICENSE.

**Exit:** `docker compose up` reaches healthy; CI is green.

## M2 — Journal

- [ ] `journal_events` with per-run sequence.
- [ ] Append under an advisory lock.
- [ ] Fold to `RunState`.
- [ ] Incomplete-run query.
- [ ] First migration.

**Exit:** concurrent appends keep a contiguous sequence; fold tests pass.

## M3 — Workflow and engine

- [ ] `checkout` definition.
- [ ] Bounded queue and worker pool.
- [ ] Interpreter: gate, effect, completion, retry.
- [ ] Durable timers from journal timestamps.
- [ ] Signals stored in the journal.

**Exit:** a run advances through the three steps with durable events.

## M4 — Exactly-once

- [ ] `ledger_entries` with unique idempotency key.
- [ ] Atomic ledger + `effect_applied`.
- [ ] Duplicate path with `duplicate: true`.
- [ ] Sequential and 12-way concurrent tests.
- [ ] Ledger row without event test.

**Exit:** applying the same effect twice books one row.

## M5 — Recovery and chaos

- [ ] Startup replay.
- [ ] Recovery sweep.
- [ ] `POST /chaos`.
- [ ] Fixed-seed crash matrix.
- [ ] Ownership-lock release after abort.
- [ ] Manual Docker restart verification.

**Exit:** a worker killed mid-run recovers with one charge.

## M6 — Demo and evidence

- [ ] API endpoints.
- [ ] One-page UI with start, journal, ledger and CHAOS.
- [ ] Seeded history.
- [ ] Real benchmark with the app stopped.
- [ ] README, four ADRs, three Spanish learning docs, screenshot/recording.

**Exit:** the 60-second beat works and the reviewer can explain the guarantees.

## M7 — Follow-ups (not MVP)

- [ ] Projection/index for incomplete runs.
- [ ] Workflow versioning in the journal.
- [ ] Real payment-provider adapter with provider-side idempotency.
- [ ] Distributed workers and leader election (only if the project needs it).
- [ ] Hosted demo with a read-only token.
