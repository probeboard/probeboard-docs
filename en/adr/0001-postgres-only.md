# ADR-0001: PostgreSQL as the only infrastructure dependency

- **Status:** accepted
- **Date:** 2026-09-12
- **Relates to:** NFR-3, NFR-4, NFR-16

## Context

probeboard needs four things a plain application database is not always asked to
provide: a work queue with leases (scheduling), high-frequency counter updates
(aggregates), a durable outbox (notifications), and time-series storage (probe
results). The conventional answer is to add Redis for the queue and counters,
and possibly a dedicated time-series store.

Against that stands NFR-16: the whole system must start from one command in a
clean environment. Every added service is another container, another
configuration surface, another failure mode, and another thing to explain in the
thesis.

## Options considered

### Option A — Postgres only
Leases via `SELECT … FOR UPDATE SKIP LOCKED`, aggregates via `INSERT … ON
CONFLICT DO UPDATE`, outbox as a table, probe results in a partitioned table.

### Option B — Postgres + Redis
Redis for the scheduling queue (or BullMQ on top of it) and for counters;
Postgres for durable state.

### Option C — Postgres + Redis + ClickHouse/Timescale
What openstatus (Tinybird) and OneUptime (ClickHouse) do at production scale.

## Decision

Option A. PostgreSQL is the only infrastructure dependency.

## Rationale

The decisive factor is **transactional coupling**, not operational simplicity.

An incident and the notification that announces it must commit together or not
at all; a probe result and the aggregate it updates must not diverge. With Redis
holding the queue and Postgres holding the data, those pairs span two systems
and there is no transaction that covers both. Recovering from a partial failure
then requires reconciliation logic that exists purely because the state was
split.

`SKIP LOCKED` has been in Postgres since 9.5 (2016) and is precisely the
primitive a claim-based scheduler needs, so Option B buys performance we do not
need at 500 monitors while costing the one property we do need.

Option C solves a data-volume problem an order of magnitude beyond this thesis's
target, and would make the storage chapter about operating ClickHouse rather
than about designing retention.

## Consequences

- Easy: one backup, one connection string, `docker compose up` with three
  containers, one transaction boundary for every invariant.
- Harder: aggregate writes contend on the same rows a chart reads. Mitigated by
  narrow rollup rows and atomic in-database increments (ADR-0003).
- Foreclaimed for now: sub-second scheduling granularity, and probe volumes
  where Postgres write throughput becomes the limit.
- Revisit if: sustained monitor count exceeds roughly 10,000, or probe interval
  granularity below 10 s is required.
