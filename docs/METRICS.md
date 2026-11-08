# Continuum — benchmark plan

> **Status: plan.** No benchmark has been published. The numbers below are the measurements the finished MVP must produce, with the conditions and the honest caveats.

## What to measure

| Metric | Definition | Why it matters |
|---|---|---|
| Throughput | Completed checkouts per second with 4 workers | Shows the cost of the journal + ledger transactions |
| Execution latency | First event → `run_completed`, p50 and p99 | Shows the end-to-end cost per run |
| In-run scheduling | `step_completed` → next `step_started`, p50 and p99 | Shows engine overhead, not queue wait |
| Burst queue latency | 200 runs submitted at once, p50 and p99 | Shows the honest consequence of a bounded pool |
| Recovery time | 20 interrupted runs → all completed, total and per run | Shows the recovery path |
| Docker restart | Start a run, restart the app, time to completion | Shows recovery after a real process restart |

## Conditions

- PostgreSQL 16 in Docker, localhost.
- The application container stopped during the benchmark.
- A dedicated `continuum_bench` database; the benchmark refuses to run otherwise and truncates only that database.
- Release build (`cargo run --release --bin bench`).
- Report the machine, the date and the worker count.
- Run the benchmark at least three times; publish a range, not a single cherry-picked number.

## What will be published

The README will include:

- the exact command;
- the environment (CPU, OS, PostgreSQL version, Docker);
- a table with the measured range;
- one raw JSON line from a run;
- a short interpretation: what is engine cost, what is queue wait, what is recovery.

If the numbers move with desktop load, the README will say so and report the spread.

## What will never be published

- A number without a reproducible command.
- A comparison with Temporal, Cadence or a production deployment that was not measured under the same conditions.
- A throughput claim without the queue latency next to it.
- A p99 with a sample too small to support it.
- A benchmark run with the app competing for the same CPU.

## Test plan for the guarantees

The benchmark is not a correctness test. Correctness is covered by:

- journal fold and concurrent append tests;
- sequential and 12-way concurrent exactly-once tests;
- crash-after-effect, crash-during-signal and fixed-seed crash-matrix tests;
- a ledger row without a journal event;
- ownership-lock release after an abort.

The benchmark proves cost, not correctness. The README will keep the two separate.
