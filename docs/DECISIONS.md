# Continuum — decisions to make before the code

> **Status: plan.** These are the decision records the implementation will turn into ADRs. They are written now so the code does not decide them by accident.

## 0001 — The journal is the only source of truth

**Context.** A workflow needs to survive crashes. A mutable `runs` table can be updated before or after the effect, leaving a window where the state and the side effect disagree.

**Decision.** Store every state change as an append-only `journal_events` row with a per-run sequence. Fold the journal into `RunState` on every read. There is no `runs` table.

**Alternatives.** A mutable run row plus an audit log; an in-memory state with periodic snapshots; an external workflow service.

**Consequences.** The state is reconstructible and auditable. Reads fold the journal; a projection/index is a later optimization. The journal grows forever unless a retention policy is added.

## 0002 — Exactly-once effects with an idempotency key

**Context.** A worker can die after the effect commits but before `step_completed`. Retrying the step must not book the effect twice.

**Decision.** Each effect has a deterministic key: `checkout:<run-id>:<step>`. The ledger has `UNIQUE(idempotency_key)`. The ledger row and `effect_applied` commit in the same transaction. The duplicate path appends the missing event and returns the existing row.

**Alternatives.** A best-effort retry with no key; a provider-side idempotency key only; a distributed transaction.

**Consequences.** Exactly-one booking is guaranteed when the effect and the journal share a database. A real payment provider needs provider-side idempotency and an outbox; that is a documented follow-up. The README will not claim exactly-once delivery of an arbitrary external call.

## 0003 — Bounded workers, ownership and crash recovery

**Context.** The engine must process runs concurrently without two workers advancing the same run, and it must recover runs left behind by a dead worker.

**Decision.** A bounded queue of run IDs, a fixed worker pool, a per-run PostgreSQL advisory lock for ownership and a recovery sweep that enqueues incomplete runs. A pool release hook clears session locks before a connection is reused.

**Alternatives.** A single worker (simpler, slower); a separate scheduler service; a `locked_until` column with optimistic locking.

**Consequences.** The queue is not durable; the journal is. A full queue returns `scheduled: false` and the sweeper picks the run up. Session locks plus a pool are a trap; the release hook and a test are mandatory.

## 0004 — Durable timers, signals and a polling UI

**Context.** Retries, gate timeouts and signals must survive a restart. The demo needs to show them without a frontend build.

**Decision.** Derive timers from journal timestamps instead of a separate timer table. Store signals as journal events. Poll the API every second from one HTML page.

**Alternatives.** A timer wheel in memory; a separate timers table; SSE/WebSocket for the UI.

**Consequences.** Timers survive because the journal survives; a timer fires when the fold says its deadline passed. Polling is simple and honest; it adds up to one second of UI latency.

## Open questions

- Whether to add a projection/index for incomplete runs after the MVP.
- Whether to expose a `workflow version` in the journal before the first public release.
- Whether a real payment-provider adapter belongs in the MVP or in a follow-up.
