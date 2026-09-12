# ADR-0002: Hand-rolled lease scheduling with SKIP LOCKED

- **Status:** accepted
- **Date:** 2026-09-12
- **Relates to:** NFR-2, NFR-3, NFR-4, NFR-7, FR-17

## Context

Every active endpoint must be probed at its interval, exactly once per slot,
across an arbitrary number of worker processes, with a crashed worker's claimed
work becoming available again rather than being lost silently.

## Options considered

### Option A — `@nestjs/schedule` / `node-cron`
In-process cron. Every instance fires the same job, so N instances probe
everything N times. Broken by construction for multi-instance deployment; this
is a widely reported NestJS problem, not a subtlety.

### Option B — Uptime Kuma's model
One recursive `setTimeout` per monitor inside one process, with in-memory state
per monitor. Verified by reading `server/model/monitor.js`: correct drift
compensation, but scheduling state lives in process memory, so a second instance
would double every probe. This is the architectural reason Uptime Kuma is
single-process.

### Option C — BullMQ repeatable jobs
Redis-backed, deduplicates by job id. Requires Redis (see ADR-0001) and
accumulates delayed jobs in memory when workers fall behind.

### Option D — pg-boss
Postgres-backed cron with singleton enforcement, implemented internally with
`SKIP LOCKED`.

### Option E — Hand-rolled claim query
One `UPDATE … WHERE id IN (SELECT … FOR UPDATE SKIP LOCKED)` statement issued on
a tick.

## Decision

Option E.

## Rationale

Each requirement maps to exactly one clause of one statement, which is both the
implementation and the argument:

| Clause | Requirement |
|---|---|
| `FOR UPDATE SKIP LOCKED` | NFR-3 — workers take disjoint batches without blocking |
| `leased_until < now()` | NFR-4 — a dead worker's claim is reclaimable |
| `next_run_at + interval` computed from the schedule | NFR-2 — drift cannot accumulate |
| no per-worker configuration | NFR-7 — scale by starting a process |

Option D would satisfy the same requirements, but it would hide the mechanism
the thesis exists to examine and measure. Writing it out makes the evaluation
chapter possible: the fault-injection test in §2.3 measures the behaviour of a
query we can point at.

## Consequences

- Easy: the correctness argument, and the evaluation. Adding workers needs no
  configuration anywhere.
- Harder: we own the edge cases a library would have handled — notably the
  catch-up guard, so a worker that was offline for an hour does not fire sixty
  probes in a burst.
- The lease duration must exceed probe timeout plus slack, or a slow probe's own
  lease expires mid-flight and a second worker duplicates it.
- Revisit if: scheduling needs grow into general-purpose job processing with
  retries, priorities and dead-letter handling.
