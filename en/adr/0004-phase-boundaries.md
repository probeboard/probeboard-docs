# ADR-0004: Record absolute phase boundaries, not phase durations

- **Status:** accepted
- **Date:** 2026-09-12
- **Relates to:** FR-18, NFR-5

## Context

A probe passes through DNS resolution, TCP connect, TLS handshake, server
processing and body transfer. Attributing time to each phase is what separates
"the endpoint is slow" from "our DNS resolver is slow". The measurement can be
stored as per-phase durations, or as the timestamp of each boundary.

## Options considered

### Option A — per-phase durations
`dns_ms`, `connect_ms`, `tls_ms`, … computed at probe time. What
`blackbox_exporter` exposes as `probe_http_duration_seconds{phase=...}`.

### Option B — absolute boundary timestamps
`dns_start`, `dns_done`, `connect_start`, `connect_done`, … stored as recorded;
durations derived at read time. What openstatus's Go checker does.

## Decision

Option B.

## Rationale

- **Durations are lossy and the loss is not recoverable.** Any duration is
  derivable from boundaries; boundaries cannot be reconstructed from durations.
- **Redirects break Option A.** With more than one redirect there are several
  DNS lookups and handshakes, and summing them per phase produces a number that
  describes nothing. `blackbox_exporter` has an open issue of exactly this shape,
  where `phase="tls"` reports nonsense after multiple redirects.
- **Gaps stay visible.** Time between phases — socket-pool wait, scheduler
  jitter — remains observable instead of being silently folded into a
  neighbouring phase. This matters for NFR-5, which requires that probeboard's
  own queueing delay not be counted as the endpoint's response time.

A consequence to record in the thesis: phases do not sum to the total, with
discrepancies around 10% being normal. `total_ms` is therefore measured
directly and never computed as a sum.

## Consequences

- Easy: per-phase analysis, redirect-aware attribution, honest measurement of
  our own overhead.
- Harder: more columns per probe row, and derivation logic at read time.
- Revisit if: storage per probe row becomes a constraint, which the retention
  design (ADR-0001, ADR-0003) makes unlikely.
