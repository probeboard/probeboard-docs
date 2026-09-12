# 4. Prior art and reusable patterns

Findings from reading the source of comparable systems, not just their feature
pages. Local clones live in `probeboard/references/` (outside every repo — see
its README for the license boundary).

## 4.1 The field

| System | Lang | License | Stars | Shape |
|---|---|---|---|---|
| [Uptime Kuma](https://github.com/louislam/uptime-kuma) | JS | MIT | ~91k | Single-process self-hosted monitor. The reference point. |
| [openstatus](https://github.com/openstatusHQ/openstatus) | TS + Go | **AGPL-3.0** | ~9.1k | Modern "monitoring as code" + status page. Closest to our stack. |
| [Gatus](https://github.com/TwiN/gatus) | Go | Apache-2.0 | ~12k | Config-in-Git health dashboard, single binary. |
| [blackbox_exporter](https://github.com/prometheus/blackbox_exporter) | Go | Apache-2.0 | ~5.9k | Probe executor only; Prometheus scrapes it. |
| [OneUptime](https://github.com/OneUptime/oneuptime) | TS | Apache-2.0 | ~7.6k | Full Datadog/PagerDuty alternative. Postgres + ClickHouse + Redis. |
| [Checkmate](https://github.com/bluewave-labs/Checkmate) | TS | AGPL-3.0 | ~10.8k | Self-hosted monitor, MongoDB. |
| [Statping-ng](https://github.com/statping-ng/statping-ng) | Go | GPL-3.0 | ~2k | Unmaintained since mid-2025. |

The licenses matter for a public thesis repo. MIT and Apache-2.0 code can be
borrowed with attribution; AGPL code (openstatus, Checkmate) can be *read and
cited* but never copied, since AGPL triggers on network use and would relicense
probeboard itself.

## 4.2 Uptime Kuma — the closest comparison, and where it breaks

### How it schedules

Each monitor runs its own recursive `setTimeout` inside the single Node process
(`server/model/monitor.js`). Drift is compensated per cycle:

```js
let intervalRemainingMs = Math.max(1, beatInterval * 1000 - dayjs().diff(bean.time));
this.heartbeatInterval = setTimeout(safeBeat, intervalRemainingMs);
```

Subtracting elapsed time before re-arming is the correct anti-drift technique
and we should use it. Everything around it is the problem:

- **One live timer per monitor, in one process.** 500 monitors is 500 timers in
  one event loop. There is no notion of a second worker.
- **Scheduling state is in-process.** Restarting loses every timer; they are
  rebuilt at boot.
- **No claim or lease.** Running two instances against one database would
  double-probe everything. NFR-3 and NFR-7 exist precisely because of this.

### How it aggregates — and the flaw to exploit

`server/uptime-calculator.js` keeps three rollup tables, `stat_minutely`
(24 h), `stat_hourly` (30 d), `stat_daily` (365 d), updated on every heartbeat.
The tiering is right and we should adopt it. Three defects are worth naming in
the thesis, because each is a concrete improvement probeboard can make:

1. **No percentiles.** Each bucket stores only `up`, `down`, `ping` (mean),
   `pingMin`, `pingMax`. So Uptime Kuma *cannot* report p95 over 30 days at
   all — the data is gone. Our histogram-bucket design (§3.6) fixes this, and
   the gap is a clean, citable justification for the extra column.

2. **Read-modify-write in application code.** `getDailyStatBean()` loads the
   row, mutates it in JS, and stores it back. Safe only because there is
   exactly one process. Under concurrent workers this loses updates. probeboard
   must aggregate with an atomic SQL upsert (`INSERT … ON CONFLICT DO UPDATE
   SET up = stat.up + 1, …`) — the increment happens in the database, not in a
   worker's memory.

3. **Retention by `DELETE` on the hot path.** Every heartbeat issues two
   `DELETE … WHERE timestamp < ?` statements. That is a full retention sweep per
   probe. A scheduled job — or partition drop — belongs there instead.

There is also an in-memory `UptimeCalculator` singleton per monitor
(`static list = {}`) holding `LimitQueue` ring buffers. Fast, and the single
hardest thing to distribute: it is the architectural reason Uptime Kuma is
one-process-only.

### What to copy outright

- The three-tier rollup shape (minute/hour/day, 24 h/30 d/365 d).
- The four-state heartbeat encoding `0=DOWN, 1=UP, 2=PENDING, 3=MAINTENANCE`.
  `PENDING` makes the retry window visible instead of hidden.
- The `important` flag marking heartbeats where the status *changed* — it makes
  "show me only transitions" a cheap indexed query and drives notifications.
- Drift-compensated re-arming, as above.

## 4.3 openstatus — nearest in stack, read-only in license

A TypeScript monorepo with a **Go probe executor** (`apps/checker`) deployed to
multiple fly.io regions, results streamed to Tinybird (ClickHouse). Splitting
the prober out of the application in a different language is a deliberate choice
and an option we should evaluate explicitly in the architecture ADR.

Its schema (`packages/db/src/schema/`) is an independent confirmation of the
entity set derived in §3: `monitors`, `monitor_tags`, `monitor_groups`,
`monitor_run`, `monitor_status`, `monitor_transition`, `incidents`,
`maintenances`, `notifications`, `pages`, `page_components`,
`page_subscribers`, `status_reports`, `private_locations`, `frozen_uptime`.
Two are instructive: `monitor_transition` stores state changes as first-class
rows rather than as a flag, and `frozen_uptime` precomputes historical uptime so
old windows never recompute.

**Phase timing as absolute boundaries** (`apps/checker/checker/http.go`):

```go
type Timing struct {
    DnsStart, DnsDone                  int64
    ConnectStart, ConnectDone          int64
    TlsHandshakeStart, TlsHandshakeDone int64
    FirstByteStart, FirstByteDone      int64
    TransferStart, TransferDone        int64
}
```

This is the pattern §3.3.1 recommends, in production. Durations are derived at
read time; nothing is lost to redirects or connection reuse.

**Assertions as structured JSON, not a string DSL**
(`packages/assertions/src/v1.ts`): a discriminated union on
`type ∈ {status, header, textBody, jsonBody, dnsRecord}`, with comparator
enums — `stringCompare`, `numberCompare ∈ {eq, not_eq, gt, gte, lt, lte}`,
`recordCompare` — and an explicit `version: "v1"` field for forward migration.
Versioning a user-authored rule format from day one is the detail worth
stealing.

## 4.4 Gatus — assertions as a string DSL

The opposite design, and a clean one. Conditions are plain strings evaluated
against a probe result:

```
[STATUS] == 200
[RESPONSE_TIME] < 500
[BODY].user.name == john
len([BODY].data) < 5
has([BODY].errors) == false
[BODY].name == pat(john*)
[STATUS] == any(200, 429)
[CERTIFICATE_EXPIRATION] > 48h
```

Placeholders: `[STATUS]`, `[RESPONSE_TIME]`, `[IP]`, `[BODY]` (JSONPath),
`[CONNECTED]`, `[CERTIFICATE_EXPIRATION]`, `[DOMAIN_EXPIRATION]`, `[DNS_RCODE]`.
Functions: `len`, `has`, `pat`, `any`.

`Result` (`config/endpoint/result.go`) carries `HTTPStatus`, `DNSRCode`,
`Hostname`, `IP`, `Connected`, `Duration`, `Errors[]`, `ConditionResults[]`,
`Success`, `Timestamp`, `CertificateExpiration`, `DomainExpiration`, `Body`.
Storing **per-condition results**, not just an overall pass/fail, is what lets
the UI say *which* assertion failed — worth adopting either way.

### The choice this forces on us

| | String DSL (Gatus) | Structured JSON (openstatus) |
|---|---|---|
| Authoring | terse, one text field | needs a form builder |
| Validation | parse errors at save time | schema-validated by construction |
| UI | free-text box + docs | rich editor, discoverable |
| Evolution | parser versioning is painful | explicit `version` field |
| Thesis value | a real parser is defensible work | mostly plumbing |

**Recommendation:** structured JSON as the stored form, with a Gatus-style
string as optional sugar over it if time allows. Storage should be the
machine-friendly form; the parser is a nice-to-have, not the foundation. This
belongs in an ADR.

## 4.5 blackbox_exporter — the probe executor as a pure function

The exporter does one thing: given a target and a module, probe once and return
metrics. It holds no schedule, no database, no state — Prometheus decides when
to scrape, which is what decides when to probe.

Phase timing uses `net/http/httptrace` hooks (`DNSStart`, `DNSDone`,
`ConnectDone`, `TLSHandshakeStart/Done`, `GotFirstResponseByte`), summed across
redirects and exposed as `probe_http_duration_seconds{phase=...}`.

Two lessons:

1. **Separating "decide when" from "execute one probe" is the right seam.**
   probeboard's probe executor should be a pure function
   `(monitor config) → probe result`, trivially unit-testable against a local
   test server, with the scheduler as a separate concern. This is the single
   most important structural idea in this document.
2. **Their known bug is our warning.** `probe_http_duration_seconds{phase="tls"}`
   reports nonsense with more than one redirect, because durations are summed
   per phase with no way to disambiguate. Recording absolute boundaries (§4.3)
   avoids the entire class.

## 4.6 Scheduling: what the ecosystem actually does

NFR-2/3/4/7 (no drift, no duplicates, no lost work, scales by adding workers)
rule out the obvious options:

| Approach | Verdict |
|---|---|
| `@nestjs/schedule` / `node-cron` | **No.** Every instance fires the same cron — the standard NestJS multi-instance duplicate-job problem. Violates NFR-3 by construction. |
| Uptime Kuma's per-monitor `setTimeout` | **No.** In-process, single-instance (§4.2). |
| BullMQ repeatable jobs (Redis) | **Viable.** Deduplicates by job id; one instance processes each. Cost: Redis as a dependency, and delayed jobs pile up in memory if workers fall behind. |
| pg-boss (Postgres) | **Viable.** Cron with singleton enforcement across replicas, via `SKIP LOCKED`. No new infrastructure if we are already on Postgres. |
| Hand-rolled `SELECT … FOR UPDATE SKIP LOCKED` lease loop | **Viable, and the most defensible.** |

The lease pattern, which is what pg-boss implements internally:

```sql
UPDATE monitors SET
    leased_until = now() + interval '30 seconds',
    leased_by    = $worker_id
WHERE id IN (
    SELECT id FROM monitors
    WHERE enabled AND next_run_at <= now()
      AND (leased_until IS NULL OR leased_until < now())
    ORDER BY next_run_at
    FOR UPDATE SKIP LOCKED
    LIMIT $batch
)
RETURNING *;
```

`SKIP LOCKED` (Postgres ≥ 9.5) gives each worker a disjoint batch with no
blocking — NFR-3. `leased_until` in the past makes a dead worker's claim
reclaimable — NFR-4. `next_run_at = scheduled_at + interval` (not
`now() + interval`) keeps drift from accumulating — NFR-2. Adding a worker
requires no configuration change anywhere — NFR-7.

**Recommendation:** hand-roll it. Every NFR maps to one clause of one query,
it is directly measurable in the evaluation chapter, and a library would hide
exactly the mechanism the thesis is about. pg-boss and BullMQ go in the ADR as
the alternatives considered.

## 4.7 Percentiles under downsampling

The problem in §3.6 is not unique to us. TimescaleDB's toolkit solves it with
**mergeable sketches** — `uddsketch` by default, `tdigest` optionally — stored
in the aggregate and queried later with `approx_percentile(0.95, digest)`.
t-digest is explicitly "partializable", which is the property that makes it
work in continuous aggregates.

That confirms the shape of our answer while leaving the specific mechanism open:
sketches and fixed histogram buckets are both mergeable, and buckets need no
extension and render as a heatmap for free. Prior art therefore supports the
recommendation in §3.6 rather than overriding it.

## 4.8 SSRF defence — the current state of the art

NFR-11 is not satisfied by validating the URL. The attack is DNS rebinding: the
hostname resolves to a safe address at validation time and to `127.0.0.1` when
the connection is actually made. Recent CVEs in Node libraries are exactly this
bypass.

The defence has three parts, and all three are necessary:

1. Resolve the hostname yourself, first.
2. Classify every resolved address with `node:net`'s `BlockList` — not a
   regex — including IPv4-mapped IPv6 forms (`::ffff:127.0.0.1`) and the cloud
   metadata address `169.254.169.254`.
3. **Pin the connection to the validated IP** with a custom `undici` dispatcher
   or `lookup` callback, so the socket cannot reach a different address than
   the one that was checked.

Re-validate on every redirect hop, since each is a fresh resolution.
`request-filtering-agent` (MIT) implements parts 1–2 for the legacy `http`
agent; it does not cover `fetch`/undici, which is what a modern Node service
uses. Expect to implement the dispatcher ourselves — and note that this makes a
genuinely strong thesis section, since it is an active CVE class rather than a
textbook exercise.

## 4.9 Summary — what probeboard reuses

| Pattern | From | License | Use |
|---|---|---|---|
| Minute/hour/day rollup tiers | Uptime Kuma | MIT | adopt |
| `UP/DOWN/PENDING/MAINTENANCE` encoding | Uptime Kuma | MIT | adopt |
| `important` transition flag | Uptime Kuma | MIT | adopt |
| Drift-compensated re-arming | Uptime Kuma | MIT | adopt, but server-side |
| Phase timing as absolute boundaries | openstatus | AGPL — idea only | reimplement |
| Versioned structured assertions | openstatus | AGPL — idea only | reimplement |
| Per-condition results in the probe row | Gatus | Apache-2.0 | adopt |
| String condition DSL | Gatus | Apache-2.0 | optional sugar, later |
| Probe executor as a pure function | blackbox_exporter | Apache-2.0 | adopt — key structural idea |
| `SKIP LOCKED` lease scheduling | pg-boss / Postgres | — | hand-roll |
| Mergeable percentile structures | TimescaleDB toolkit | — | histogram buckets |
| DNS-rebinding-safe SSRF guard | OWASP / undici | — | implement |

### What probeboard does that none of the self-hosted alternatives do

1. Percentiles that survive downsampling (Uptime Kuma keeps only mean/min/max).
2. Horizontally scalable probing with lease-based claims (all single-process).
3. Phase-level latency attribution in a self-hosted monitor with a UI
   (blackbox_exporter has the data but no UI; Uptime Kuma has neither).
4. Error budgets computed from the same aggregates that serve the charts.

Those four are the thesis contribution, and each one is now backed by a specific
gap in a specific named system rather than by assertion.
