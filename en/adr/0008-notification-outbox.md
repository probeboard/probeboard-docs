# ADR-0008: Notifications via a transactional outbox

- **Status:** accepted
- **Date:** 2026-09-12
- **Relates to:** FR-28, FR-29, E-3, E-4, E-8

## Context

When an incident opens, the user must be emailed. The incident is a database
write; the email is a call to an external service that can be slow, can fail,
and cannot participate in a database transaction.

## Options considered

### Option A — send directly from the evaluator
Open the incident, then call the mail provider.

### Option B — transactional outbox
Write the intent to a table in the same transaction that opens the incident. A
separate dispatcher loop drains it.

## Decision

Option B.

## Rationale

Option A has no correct ordering. Send first and the transaction may roll back,
leaving an email about an incident that does not exist. Write first and the
process may die before sending, leaving an incident nobody was told about — the
silent failure mode that makes a monitoring system worse than useless. There is
no third order; the operations span two systems.

The outbox collapses the problem into one transaction: an incident and its
notification commit together or not at all. It also gives the PRD's alerting
rules a natural home:

| Rule | Mechanism |
|---|---|
| E-3, one email per service | `UNIQUE (group_key)` plus a `not_before` delay that lets siblings collect |
| E-4, no spam | a decaying re-notification schedule driven from the incident row |
| E-8, never silently dropped | `attempts` with backoff; a permanent failure records `last_error` and surfaces in the UI |

A slow or failing mail provider also cannot delay probing, since the dispatcher
is a separate loop from the scheduler.

## Consequences

- Easy: exactly-once-in-effect delivery semantics, grouping, retry, and
  visibility of delivery failures.
- Harder: an extra table and loop; delivery is delayed by up to one dispatcher
  tick plus the grouping window.
- The outbox needs its own retention, or it becomes an unbounded log of
  successful sends.
