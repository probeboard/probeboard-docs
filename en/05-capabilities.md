# 5. Capabilities — positioning and scope

Given the health model (§3) and the prior art (§4), this chapter fixes what
probeboard does and, as importantly, what it refuses to do.

## 5.1 What class of system this is

**White-box observability** — Datadog, Grafana + Prometheus, New Relic,
OneUptime. An agent or SDK runs *inside* the monitored system and emits metrics,
logs and traces outward. It knows internals: per-endpoint request counts, GC
pauses, query times, distributed traces. The central engineering problem is
**ingest** — accepting and storing millions of points per second from many
sources.

**Black-box / synthetic monitoring** — probeboard, UptimeRobot, Pingdom,
Uptime Kuma, Gatus, and specifically *Datadog Synthetics* and *Grafana Synthetic
Monitoring*, which are single products inside those larger platforms. The system
sits *outside* and exercises the endpoint as a real client would. It knows
nothing internal, but it measures what users actually experience and works on
endpoints you neither own nor can instrument. The central problem is **scheduled
distributed execution** — probing the right target at the right time, exactly
once, at scale.

```
   white-box (Datadog / Grafana)        black-box (probeboard)
   ┌──────────────┐                     ┌──────────────┐
   │   service    │──agent──▶ storage   │   service    │
   │  (you own &  │                     │ (you may not │
   │  instrument) │                     │   own it)    │
   └──────────────┘                     └──────▲───────┘
                                               │ HTTP probe
                                        ┌──────┴───────┐
                                        │  probeboard  │
                                        └──────────────┘
```

Neither is better. White-box answers *why is it slow*; black-box answers *is it
working for the people using it, right now*. Real deployments run both, and §3.10
already states the limits of the black-box half honestly.

**probeboard is black-box and must not drift white-box.** Log ingestion, APM and
arbitrary metric ingest are not scope cuts made for time — they are a different
thesis with a different central problem.

### The framing to defend

> probeboard implements the synthetic-monitoring subset of an observability
> platform — what Datadog sells as *Synthetics* and Grafana as *Synthetic
> Monitoring* — with emphasis on two properties those products do not expose for
> inspection: how probe scheduling stays correct across multiple workers, and
> how storage stays bounded under continuous collection.

## 5.2 Borrowed capabilities

Compatible with a black-box design: each operates on data probeboard already
collects and needs no agent inside anyone's system.

### Tier A — recommended

| Capability | Origin | Why it earns its place |
|---|---|---|
| **Tags** (`env:prod`, `team:payments`) with filter and group | Datadog tags, Prometheus labels | Turns a flat list into a queryable inventory. Prerequisite for dashboards, per-team views and SLOs. One join table. Confirmed by openstatus's `monitor_tags`. |
| **SLOs with error budgets** | Datadog SLOs, Google SRE | Computed from aggregates NFR-8 already forces (§3.9). Moves the statistics chapter from "a percentage" to "a policy". |
| **Maintenance windows / mute** | Datadog Downtimes | Deploys stop manufacturing incidents. openstatus has `maintenances`; Uptime Kuma has a `MAINTENANCE` heartbeat state. |
| **Latency threshold → `DEGRADED`** | Datadog multi-condition monitors | Implements H4; makes health non-boolean in the UI, not just in the data. |
| **Phase-level latency** (§3.3) | blackbox_exporter, openstatus | Distinguishes "our API got slower" from "DNS got slower". Cheap, and visually compelling. |
| **Multi-panel dashboard, shared time range** | Grafana | FR-36 already implies it. Panels purpose-built, not user-composed. |
| **`/metrics` in Prometheus format** | Prometheus | NFR-20. Strategically important — see §5.3. |

### Tier B — if time allows

| Capability | Origin | Notes |
|---|---|---|
| **Public status page** | Better Stack, Uptime Kuma, openstatus | Highly demoable. Needs a share token and strict field filtering so URLs and headers never leak. |
| **Adaptive latency baselines** — "slower than the same hour last week by k·σ" | Datadog anomaly monitors | The most research-flavoured option available. Runs on stored hourly aggregates, so data cost is zero. Would justify its own thesis subsection on threshold selection. |
| **Notification throttling and dedup** | every alerting system that survived users | One alert per incident with decaying re-notification. Directly answers the alert-fatigue argument in §1.4. |
| **Heartbeat monitors (dead-man's switch)** | Cronitor, Healthchecks.io | Inverts probe direction; covers cron jobs and workers that cannot be probed from outside. Small addition, noticeably wider applicability. |
| **Latency heatmap** (time × latency bucket) | Grafana heatmap | ⚠️ Free *if* §3.6 option B is chosen, impossible to retrofit later. The aggregate schema decision must be made before downsampling is implemented. |
| **String condition DSL** | Gatus | Sugar over the structured assertion form (§4.4). |

### Tier C — refused, with reasons

| Capability | Why not |
|---|---|
| User-composed dashboards / panel builder | The composition engine *is* Grafana's product. Months of UI work demonstrating no thesis argument. Purpose-built panels answer §1.3's four questions better anyway. |
| A query language (PromQL-style) | Enormous surface for one metric family. Parameterised fixed queries cover every real use. |
| Log ingestion, APM, distributed tracing | White-box. Different system, different central problem. |
| Multi-region probing | Already out of scope in §1.5. Needs infrastructure in several regions. The scheduling work does generalise — one paragraph of future work. |
| On-call scheduling, escalation policies | PagerDuty's product. Orthogonal to measurement. |
| Browser multi-step synthetic transactions | The hard parts are browser automation, not monitoring. |
| User-pushed arbitrary metrics | Turns storage from bounded-and-designed into unbounded-and-adversarial, destroying the property that makes the retention design tractable. |

## 5.3 Interoperate with Grafana rather than rebuild it

Expose probeboard's collected data in Prometheus/OpenMetrics format, so anyone
already running Grafana can point it at probeboard and build whatever panels
they want.

This converts the Tier C refusal from a gap into a position: *probeboard is a
correct collector with a focused UI, and composes with the visualization tool
that already exists*. One endpoint, and "why didn't you build dashboards like
Grafana" answers itself.

## 5.4 Concept model

```
  User ──owns──▶ Monitor ──tagged──▶ Tag
                    │
                    ├──produces──▶ Probe result ──rolls up──▶ Aggregate
                    │                    │
                    │                    └──triggers──▶ Incident ──▶ Notification
                    │
                    ├──evaluated against──▶ SLO ──▶ error budget
                    │
                    └──suppressed by──▶ Maintenance window
```

Independently confirmed by openstatus's schema (§4.3), which reached the same
entity set from the same problem.

| Concept | Definition |
|---|---|
| **Monitor** | A target, how to probe it, and what counts as success |
| **Probe result** | One observation: outcome, status, phase boundaries, failure class, per-assertion results |
| **Aggregate** | Time-bucketed rollup. The only thing long windows read (NFR-9) |
| **Incident** | A sustained failure with hysteresis (FR-24/25), timed from the first failed probe (§3.8) |
| **Notification** | An incident delivered to a channel, deduplicated |
| **SLO** | A target over a window, plus the error budget remaining |
| **Maintenance window** | An interval during which incidents are suppressed |

## 5.5 Cut line

Everything marked **M** in [02-requirements.md](02-requirements.md), plus:

- phase-level latency (§3.3)
- health states including `DEGRADED` and `UNKNOWN` (§3.7)
- failure taxonomy (§3.4)
- histogram-bucket aggregates so percentiles survive downsampling (§3.6)
- tags, SLOs with error budgets, maintenance windows (Tier A)

That set is a coherent product and a thesis with four defensible arguments,
each backed in §4.9 by a specific gap in a specific named system:

1. Scheduling correctness across workers — lease-based, `SKIP LOCKED`
2. Bounded storage with percentiles that survive downsampling
3. SSRF defence against DNS rebinding — an active CVE class
4. Health as a multi-dimensional measurement, not a boolean

Tier B is attempted only after §2.3's acceptance criteria are met. Tier C stays
refused and goes into the closing chapter as future work.
