# Continuum — planned architecture

> **Status: plan.** This is the architecture the MVP will implement. It is deliberately small: one workflow, one process, one database.

## Components

```text
┌──────────────────────────────────────────────┐
│ Browser — one HTML page + vanilla JS          │
│ start · run list · journal · ledger · CHAOS   │
└──────────────────────┬───────────────────────┘
                       │ fetch (JSON, 1 s poll)
┌──────────────────────▼───────────────────────┐
│ Axum HTTP API                                │
│ /runs · /runs/{id} · /signal · /chaos · /ledger│
└───────┬──────────────────────────┬───────────┘
        │ append                    │ enqueue
        v                           v
┌───────────────────────┐   ┌──────────────────────┐
│ journal_events        │   │ bounded queue (64)   │
│ append-only, per-run  │   │ 4 workers            │
│ sequence, no run table│   └──────────┬───────────┘
└───────────┬───────────┘              │
            │ fold on read             v
            │                ┌──────────────────────┐
            │                │ workflow interpreter  │
            │                │ reserve → charge →    │
            │                │ notify                │
            │                └──────────┬───────────┘
            │                           │ effect
            v                           v
┌──────────────────────────────────────────────────┐
│ ledger_entries                                   │
│ UNIQUE(idempotency_key)                          │
│ ledger row + EffectApplied in one transaction    │
└──────────────────────────────────────────────────┘
```

## Data model (planned)

- `journal_events(id, run_id, seq, kind, payload jsonb, created_at)` with `UNIQUE(run_id, seq)`.
- `ledger_entries(id, kind, idempotency_key UNIQUE, run_id, amount_cents, created_at)`.
- `workflow_definitions(name, version, definition jsonb)` — one row for `checkout`.
- No `runs` table. State is folded from the journal.

## Event types

| Event | Meaning |
|---|---|
| `run_started` | Workflow name and validated input; the run becomes `running` |
| `step_started` | A worker owns one attempt of a step |
| `step_completed` | The step effect is durable and the step is closed |
| `step_failed` | Attempt failed; `will_retry` says whether another attempt is allowed |
| `timer_fired` | A durable timer elapsed: retry backoff or a gate timeout |
| `signal_received` | External signal stored in the journal |
| `effect_applied` | The ledger row exists; `duplicate` records whether it was already booked |
| `worker_crashed` | Supervisor observation when `/chaos` aborts a worker task (extension) |
| `run_completed` | Terminal status, effects and step summary |

## Worker model

- A bounded in-memory queue of run IDs; the journal is the durable queue.
- A fixed worker pool (default 4). Each worker takes a run, acquires a per-run advisory lock, replays the journal and advances the next step.
- A recovery sweep every 2.5 s queries incomplete runs and enqueues the ones left behind.
- On startup, the same query enqueues incomplete runs before serving traffic.
- A task abort during an effect must release the advisory lock before the connection returns to the pool.

## Exactly-once

1. Each effect has a deterministic key: `checkout:<run-id>:<reserve|charge|notify>`.
2. The ledger inserts with `ON CONFLICT (idempotency_key) DO NOTHING`.
3. The ledger row and `effect_applied` commit in the same transaction.
4. On replay, the engine checks the folded state; if the effect is already there, it closes the step without calling it again.
5. If the ledger row exists but the event is missing, the conflict path appends the missing event with `duplicate: true`.

The guarantee is **exactly-one booking per effect**, not exactly-once delivery of an arbitrary external call.

## Failure modes

| Failure | Behavior |
|---|---|
| Worker task aborted | `worker_crashed`; a new worker replays and resumes |
| Process restart | Startup replay; incomplete runs resume |
| Database unavailable | `/health` fails; the app is unavailable |
| Duplicate effect | Unique key; one row; `duplicate: true` |
| Queue full | `201` with `scheduled: false`; the sweeper picks the durable run up |
| Signal after completion | `409` |
| Unknown run | `404` |

## What is intentionally missing

- Distributed workers, leader election, replication.
- A real payment provider; the ledger is a mock.
- Workflow versioning for in-flight runs.
- Auth, TLS, rate limiting.
- A hosted demo.

## Open decisions

- Whether to expose `crashes` and `phase` in the run summary or derive them only in the UI.
- The exact benchmark shape: a dedicated binary is the default candidate.
- Whether the recovery sweep should use a projection/index after the MVP; the first version can scan incomplete runs.
