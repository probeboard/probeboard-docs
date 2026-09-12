# 6. Product requirements (PRD)

Defines the product from the user's side: who uses it, what they do, and what
"done" means for each capability. Engineering detail lives in §7 architecture;
the reasoning behind the concepts lives in §3–§5.

## 6.1 Summary

probeboard lets a developer register the HTTP APIs they depend on, have every
endpoint probed continuously from outside, see the health of each one, and get
an email when something breaks.

**Primary user:** a backend developer or small team running a handful of
services, with no observability platform. They cannot instrument the
third-party APIs they depend on, and nobody is watching at 03:00.

**Core loop:** register → probe → observe → alert → diagnose.

## 6.2 Domain model — services contain endpoints

The user does not think in flat monitors. They think: *"my payments API, and
these five endpoints on it."* The model follows that:

```
User
 └── Service            "Payments API"          base URL, shared auth headers, tags
      └── Endpoint      "POST /orders"          path, method, interval, assertions
           ├── Probe result   one observation
           ├── Aggregate      rolled-up buckets
           └── Incident       sustained failure
```

**This supersedes the flat monitor model in §2.** Every FR that says "monitor"
now means **endpoint**; `Service` is new. Three things make the extra level
worth its cost:

1. **Shared configuration.** Base URL, auth header and default interval are
   declared once. Rotating an API key touches one row, not fifteen.
2. **Service-level health.** "Is the Payments API healthy?" is a real question
   with a real answer: the aggregate of its endpoints (§6.6).
3. **Alert grouping.** A backend falling over takes all five endpoints with it.
   That is one email about one service, not five emails (§6.8).

A service with exactly one endpoint is the degenerate case and must stay
frictionless — "add a URL" has to remain a single-form action, with the service
created implicitly.

## 6.3 Scope

| | In v1 | Later |
|---|---|---|
| Auth | email + password | OAuth, teams |
| Targets | HTTP/HTTPS endpoints | TCP, DNS, ping, gRPC |
| Probing | single location | multi-region |
| Assertions | status, latency, body contains, JSON path | full DSL, chained requests |
| Alerting | **email**, webhook | Telegram, Slack, escalation |
| Views | service list, service detail, endpoint detail, incidents | user-composed dashboards |
| Extras | tags, maintenance windows, SLOs | status page, anomaly baselines |

## 6.4 Epic A — account and access

| ID | Story | Acceptance |
|---|---|---|
| A-1 | As a visitor I register with email and password | Password ≥ 10 chars, stored Argon2id. Duplicate email returns the same generic message as success — no account enumeration. |
| A-2 | As a user I log in | Returns a session credential. Wrong password and unknown email are indistinguishable in response and timing. |
| A-3 | As a user I stay logged in across reloads | Token survives refresh; expiry forces re-login. |
| A-4 | As a user I only ever see my own data | Every read and write is scoped by owner. A request for another user's resource returns 404, not 403 — no existence leak. |
| A-5 | As a user I change my password | All other sessions invalidated. |
| A-6 | As a user I am rate limited on auth | Repeated failures from one IP or for one account are throttled. |

## 6.5 Epic B — registering an API

| ID | Story | Acceptance |
|---|---|---|
| B-1 | I create a service with a name and base URL | Base URL must be absolute http/https and pass the SSRF policy at save time. |
| B-2 | I add endpoints under it: method + path | Effective URL = base + path, re-validated against the SSRF policy. |
| B-3 | I paste one full URL and get monitoring in one step | Service created implicitly from the origin; endpoint is the path. Never more than one form. |
| B-4 | I set headers on the service, inherited by every endpoint | Endpoint-level headers override by name. Secret values are write-only — never returned by any read API, shown masked. |
| B-5 | I tag services and endpoints | `key:value`. Filterable everywhere. |
| B-6 | I edit, pause, resume, delete | Paused = not probed, history retained. Delete removes probe history after confirmation. |
| B-7 | I am stopped from registering something unmonitorable | Private/loopback/link-local/metadata addresses rejected at save with a plain explanation, not a stack trace. |
| B-8 | I hit a quota | Endpoint count per user is capped and configurable; the message says the limit and the current count. |

### Per-endpoint configuration

| Field | Default | Bounds |
|---|---|---|
| interval | 60 s | 30 s / 1 m / 5 m / 15 m / 1 h |
| timeout | 10 s | ≤ 30 s, < interval |
| method | GET | GET, HEAD, POST, PUT, PATCH, DELETE |
| expected status | 200–299 | list or ranges |
| latency warning threshold | none | ms — crossing it means `DEGRADED`, not `DOWN` |
| failures to open incident | 3 | 1–10 |
| successes to close incident | 2 | 1–10 |
| follow redirects | yes, max 5 | 0–10 |
| assertions | none | body contains / not contains, JSON path compare |

## 6.6 Epic C — seeing health

### C-1 Service list — the landing screen

Answers "is anything broken right now" in one glance. Per service: name, rolled
up state, endpoint count, worst endpoint, 24 h uptime, sparkline. Sorted
unhealthy first. Filterable by tag.

**Service state rollup:** `DOWN` if any endpoint is `DOWN`; else `DEGRADED` if
any is `DEGRADED`; else `UNKNOWN` if any is `UNKNOWN`; else `UP`. Paused and
in-maintenance endpoints are excluded. Worst-wins, so a green service means
every endpoint is genuinely green.

### C-2 Service detail

Every endpoint with its current state, last response time, 24 h uptime and a
status timeline. Open incidents pinned at the top.

### C-3 Endpoint detail — the diagnostic screen

| Element | Requirement |
|---|---|
| Current state | with the reason: failure class, or which assertion failed |
| Response time chart | p50 / p95 / max over the window |
| Phase breakdown | stacked DNS / connect / TLS / TTFB / transfer (§3.3) |
| Status timeline | up / degraded / down / unknown / maintenance bands |
| Uptime | 24 h, 7 d, 30 d — with the window's definition visible |
| Incidents | list with duration and cause |
| Recent probes | last 50 raw results, expandable to status, timings, assertion outcomes |
| TLS | certificate expiry, days remaining |

Windows: 24 h, 7 d, 30 d, plus custom range. Long windows are served from
aggregates — the 30 d view must not read raw rows (NFR-9).

### C-4 Check now

A user can trigger an immediate probe and see the result within seconds without
waiting for the next scheduled run. Rate limited per endpoint.

## 6.7 Epic D — incidents

| ID | Story | Acceptance |
|---|---|---|
| D-1 | An incident opens only on sustained failure | N consecutive failures (default 3). One bad probe never opens one. |
| D-2 | It is timed honestly | `opened_at` = first failed probe, not the Nth (§3.8). |
| D-3 | It closes on sustained recovery | M consecutive successes (default 2). |
| D-4 | I see why | Failure class and the failing probe's detail are attached. |
| D-5 | I see history | Per endpoint and per service, with duration and cause. |
| D-6 | Maintenance suppresses incidents | Inside a declared window probes still run and are recorded, but no incident opens and no email is sent. |
| D-7 | Monitoring failure is not endpoint failure | A probe blocked by policy, or never executed, is `UNKNOWN` — it never opens an incident or counts against uptime. |

## 6.8 Epic E — email alerting

The answer to "will I get an email when something goes wrong" is yes, with
three rules that decide whether the feature is useful or ignored.

| ID | Story | Acceptance |
|---|---|---|
| E-1 | I get an email when an incident opens | Sent once, on open — not per failed probe. |
| E-2 | I get an email when it resolves | Includes total duration. |
| E-3 | I get one email per service, not per endpoint | Endpoints of one service failing within a short grouping window (default 60 s) produce **one** email listing them. |
| E-4 | A long outage does not spam me | Re-notification at a decaying interval (30 m, 2 h, 6 h, then daily), not per probe. |
| E-5 | I get warned before a certificate expires | At 30 / 14 / 7 / 1 days. Once per threshold, not daily. |
| E-6 | I control what I receive | Per service: open / resolve / cert. Off is allowed. |
| E-7 | I verify my address | Unverified addresses receive nothing but the verification email. |
| E-8 | A failing mail provider does not lose the alert | Send is retried with backoff; permanent failure is recorded and surfaced in the UI. Never silently dropped. |
| E-9 | A webhook is an alternative channel | Same events, signed payload, retried. |

**Open email content:** service and endpoint name, what failed (class + message),
when it started, how many consecutive failures, last successful probe time, last
response time, direct link. Enough to triage from a phone without opening the
dashboard.

## 6.9 Epic F — SLOs

| ID | Story | Acceptance |
|---|---|---|
| F-1 | I declare a target on an endpoint or service | e.g. 99.5% over 30 days. |
| F-2 | I see the error budget | Total, consumed, remaining, as time and percent. |
| F-3 | I see burn rate | Current rate, and projected exhaustion date. |
| F-4 | I am alerted on fast burn | Tier-1 multiwindow rule: 14.4× over 1 h confirmed by 5 min (§3.9). |

## 6.10 Screens

| # | Screen | Purpose |
|---|---|---|
| 1 | Register / log in | A-1, A-2 |
| 2 | Service list | C-1 — the landing screen |
| 3 | Service detail | C-2 |
| 4 | Endpoint detail | C-3 — the diagnostic screen |
| 5 | Service / endpoint form | B-1…B-6 |
| 6 | Incidents | D-5 |
| 7 | Notification settings | E-6, E-7 |
| 8 | Maintenance windows | D-6 |
| 9 | SLOs | F-1…F-3 |
| 10 | Account settings | A-5, quota |

Screens 1–6 are the v1 requirement; 7–10 follow.

## 6.11 Non-goals for v1

Teams and shared ownership · multi-region probing · public status page ·
user-composed dashboards · log/metric/trace ingest · on-call scheduling ·
browser transactions · protocols other than HTTP.

Each is listed with its reason in §5.2 Tier C or §1.5.

## 6.12 Done

v1 is complete when a user can, without operator help:

1. register, create a service, add three endpoints, and see them probed on schedule;
2. open the service list and correctly identify which service is unhealthy;
3. receive an email within one probe interval + one minute of an endpoint going down, and another when it recovers;
4. open the endpoint detail and see 30 days of p95 latency served from aggregates;
5. read the incident history and see why each incident started.

Plus the thesis criteria in §2.3: the load, fault-injection, SSRF and retention
tests.

## 6.13 Decisions taken here

| Decision | Choice | Why |
|---|---|---|
| Hierarchy | Service → Endpoint | Matches how users describe their systems; enables shared config and alert grouping (§6.2) |
| Percentile storage | Fixed histogram buckets | Mergeable by integer addition, no extension, heatmap free (§3.6) |
| Probe worker | Separate deployable, same `probeboard-api` codebase | Independent scaling for NFR-7 without a fourth repo |
| Primary alert channel | Email | Explicitly requested; webhook second |
| Missing-data semantics | `UNKNOWN`, excluded from uptime | Never let monitoring failure read as health (§3.5.2) |
