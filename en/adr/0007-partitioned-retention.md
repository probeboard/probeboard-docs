# ADR-0007: Daily partitions with DROP for retention

- **Status:** accepted
- **Date:** 2026-09-12
- **Relates to:** NFR-8

## Context

Raw probe results accumulate at roughly 21.6M rows per month at the target scale
of 500 endpoints on a 60-second interval. They must be retained for a configured
window and then removed, without the removal itself becoming a load problem.

## Options considered

### Option A — `DELETE FROM probe_results WHERE started_at < ...`
What Uptime Kuma does — and it issues two such statements **on every heartbeat**,
verified in `server/uptime-calculator.js`.

### Option B — declarative range partitioning by day, retention by `DROP TABLE`

## Decision

Option B.

## Rationale

`DELETE` at this volume writes as much WAL as the original inserts, leaves dead
tuples for autovacuum to reclaim, and competes with the probe write path for I/O.
Running it per probe, as Uptime Kuma does, means a full retention sweep for every
observation recorded.

Dropping a partition is a catalogue operation: constant time, no row scanning, no
dead tuples, no vacuum pressure. Partition pruning additionally narrows every
time-ranged query to the relevant days.

## Consequences

- Easy: O(1) retention, faster range queries, no vacuum debt.
- Harder: partitions must be created ahead of time by a maintenance job; a
  missing future partition makes inserts fail. This must be monitored — with a
  visible alert, not a silent fallback to a default partition, which would
  quietly reintroduce the unbounded table.
- The partition key is fixed at `started_at`, so retention is time-based only;
  per-endpoint retention policies are foreclosed.
- Revisit if: per-user retention tiers become a product requirement.
