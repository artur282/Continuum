# Continuum — the 60-second demo

> **Status: plan.** This is the script the finished MVP must make true. It is also the acceptance test for the UI.

## Before the demo

```bash
docker compose up --build
# wait for the app health check
open http://localhost:3000
```

The database is seeded with three history runs, so the page is never empty. If the page is empty, the demo fails.

## The script

### 0:00 — The screen

The reviewer sees:

- a run list with three seeded runs: a plain completed checkout, a retry + signal, and a declined card;
- a journal viewer for the selected run;
- a ledger with the seeded charges;
- a **Start checkout** button and a **CHAOS** button.

**What it proves:** the engine has history and a visible source of truth before the reviewer touches anything.

### 0:10 — Start a checkout

Press **Start checkout**.

A new run appears at the top. `reserve`, `charge` and `notify` advance in the stepper. The journal grows with `run_started`, `step_started` and `effect_applied` events. The charge step has a simulated gateway failure on attempt 1, so the reviewer also sees a retry with a durable `TimerFired`.

**What it proves:** a workflow is a durable sequence of events, not a function call.

### 0:25 — Press CHAOS

While `charge` is in progress, press **CHAOS**.

The supervisor aborts the Tokio task that owns the run. The run moves to **recovering** and the journal grows a `worker_crashed` event.

**What it proves:** the worker can die mid-effect; the run is not lost.

### 0:30 — Watch the recovery

A new worker replays the journal, appends the missing step event and finishes the run. The banner says **RECOVERED**, and the ledger shows **exactly one charge** for that run. The whole beat takes about 2–4 seconds.

**What it proves:** durability and exactly-one booking, not just a retry.

### 0:45 — Read the journal

Open the journal for the crashed run. The reviewer sees the sequence:

```text
run_started
step_started (reserve)
effect_applied (reserve)
step_completed (reserve)
step_started (charge)
worker_crashed
step_failed (worker lost)
timer_fired
step_started (charge)
effect_applied (charge)
step_completed (charge)
step_started (notify)
timer_fired
effect_applied (notify)
step_completed (notify)
run_completed
```

The exact order may vary, but the required events are present and the ledger has one charge.

**What it proves:** the state is reconstructible from the journal, not from a hidden mutable row.

### 0:55 — The review path

Open `README.md` and `docs/decisions/`. The reviewer sees the architecture, the four decisions, the limitations and the exact commands to reproduce the demo.

**What it proves:** the project is documented like an engineering artifact, not a magic trick.

## Failure cases the demo must not have

- An empty page on first load.
- A CHAOS button that only changes the UI without killing a worker.
- Two charges for the crashed run.
- A run stuck in `recovering`.
- A browser console error.

## What the demo deliberately does not show

- A real payment provider.
- A distributed cluster.
- Workflow versioning.
- Authentication.
- A production deployment.

Those are non-goals, not omissions. The README says so.
