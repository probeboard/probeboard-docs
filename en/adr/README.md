# Architecture decision records

One file per decision that was not obvious. Each states the forces, the options
actually considered, the choice, and what it costs — so a reader (including the
author, later) can tell whether a decision still holds when an assumption
changes.

| # | Decision | Status | Key reason |
|---|---|---|---|
| [0001](0001-postgres-only.md) | PostgreSQL as the only infrastructure dependency | accepted | Transactional coupling: an incident and its notification must commit together |
| [0002](0002-scheduling-lease.md) | Hand-rolled lease scheduling with `SKIP LOCKED` | accepted | Each NFR maps to one clause of one query — implementation and argument in the same place |
| [0003](0003-histogram-percentiles.md) | Fixed histogram buckets for percentiles | accepted | Percentiles are not averageable; integer addition merges provably |
| [0004](0004-phase-boundaries.md) | Absolute phase boundaries, not durations | accepted | Durations are lossy and break on redirects |
| [0005](0005-structured-assertions.md) | Versioned structured JSON assertions | accepted | Validated at save time; api and worker share one schema |
| [0006](0006-one-repo-split-ready.md) | One repository, structured to split | accepted | A published SDK admits version skew — silently wrong charts, no error |
| [0007](0007-partitioned-retention.md) | Daily partitions, retention by `DROP` | accepted | `DELETE` at 21.6M rows/month is its own load problem |
| [0008](0008-notification-outbox.md) | Transactional outbox for notifications | accepted | No correct ordering exists for "write incident" and "send email" otherwise |

Template: [0000-template.md](0000-template.md).

## When to write one

Write an ADR when the decision has a real alternative someone would reasonably
choose, and when reversing it later would be expensive. Do not write one for
choices that follow from an existing ADR, or that can be changed in an
afternoon.
