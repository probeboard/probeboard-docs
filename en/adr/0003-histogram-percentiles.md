# ADR-0003: Fixed histogram buckets for percentiles under downsampling

- **Status:** accepted
- **Date:** 2026-09-12
- **Relates to:** FR-34, NFR-8, NFR-9

## Context

FR-34 requires p95. NFR-8 requires bounded storage, so raw probe rows cannot be
kept indefinitely. NFR-9 requires long windows to be served from aggregates.

These collide on an arithmetic fact: **percentiles are not averageable**. Given
p95 for each of 24 hourly buckets, no arithmetic recovers the true p95 of the
day. Whatever is stored per bucket must be a structure that *merges*.

## Options considered

### Option A — retain raw rows
Exact, and violates NFR-8 outright: 500 endpoints at 60 s is ~21.6M rows/month.

### Option B — fixed histogram buckets
~20 counters per bucket over documented millisecond edges. Merging is integer
addition. p95 is linear interpolation inside the containing bucket, with error
bounded by that bucket's width.

### Option C — mergeable sketch (t-digest / DDSketch / UDDSketch)
A serialized sketch per bucket, ~1–3 KB, with a relative-error guarantee. This is
what TimescaleDB's toolkit does (`percentile_agg`, `approx_percentile`) and what
Datadog uses internally; t-digest is explicitly partializable, which is the
property that makes it work in continuous aggregates.

## Decision

Option B — fixed histogram buckets, 20 edges, stored as an `integer[]`.

## Rationale

- **Merging is provably associative**, because it is integer addition. The
  minute → hour → day rollup needs no library and no special care.
- **No extension and no dependency.** Option C would mean either the
  `timescaledb_toolkit` extension, which changes the deployment story of
  ADR-0001, or a JavaScript sketch implementation whose merge semantics we would
  have to validate ourselves.
- **The heatmap comes free.** The same array is exactly the data a latency
  heatmap needs, so the Tier-B feature costs nothing extra later — and it is
  impossible to retrofit, since it depends on what was stored at write time.
- **The error is measurable**, which turns a weakness into an evaluation result:
  compare exact p95 computed from retained raw rows against bucket-interpolated
  p95, and report the divergence.

The closest comparable system, Uptime Kuma, stores only mean, min and max per
bucket and therefore cannot answer p95 over long windows at all. This decision is
the direct response to that gap.

## Consequences

- Easy: rollups, heatmaps, and a defensible accuracy claim.
- Harder: the bucket edges are fixed at write time. Changing them invalidates
  historical comparisons, so they must be chosen once and documented.
- Accuracy is worst where buckets are widest — above 10 s. Acceptable, since
  precision at 12 s versus 14 s is not operationally meaningful.
- Revisit if: percentiles beyond p99 are needed, where tail resolution matters
  and a sketch genuinely outperforms fixed buckets.
