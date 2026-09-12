# 8. Implementation plan

Milestones in dependency order. Each is a working increment with its own tests;
none depends on a later one. "Demo" is what can be shown at the end of it.

| # | Milestone | Delivers | Demo | Requirements |
|---|---|---|---|---|
| M0 | Skeleton | NestJS app, config validation at boot, structured logging, `docker compose up` with Postgres, migrations, `/healthz` | the stack starts from one command | NFR-16, NFR-17 |
| M1 | Accounts | register, login, sessions, Argon2id, rate limiting, ownership scoping | a user can log in | FR-1…4, NFR-10, NFR-14, A-1…A-6 |
| M2 | Registration | services + endpoints CRUD, quotas, SSRF validation at save, secret headers write-only | a user registers an API and its endpoints | FR-6…16, B-1…B-8 |
| M3 | Probe executor | pure `probe()`, phase boundaries, failure taxonomy, assertions, SSRF guard with IP pinning, bounded body | probe one URL from a test, all failure classes reproduced locally | FR-18…22, NFR-5, NFR-11, NFR-13 |
| M4 | Scheduler | `endpoint_runtime`, claim query, leases, catch-up guard, concurrency pool | two workers, no duplicates; kill one mid-probe, work is reclaimed | NFR-1…4, NFR-7, FR-17 |
| M5 | Storage | partitioned `probe_results`, atomic rollups, retention by `DROP`, histogram percentiles | 30-day p95 served from aggregates, raw rows dropped | NFR-8, NFR-9 |
| M6 | Incidents | state machine, hysteresis, honest `opened_at`, `UNKNOWN` sweep, maintenance windows | endpoint goes down → incident opens after 3, closes after 2 | FR-24…27, D-1…D-7 |
| M7 | Alerting | transactional outbox, grouping, decaying re-notify, email + webhook, cert expiry | an email arrives when it breaks and when it recovers | FR-28…31, E-1…E-9 |
| M8 | Statistics | uptime (both weightings), percentiles, SLOs, error budget, burn rate | 24 h / 7 d / 30 d uptime, error budget remaining | FR-32…35, F-1…F-4 |
| M9 | Web | service list, service detail, endpoint detail, forms, incidents | the whole §6.12 journey in a browser | FR-33…37, C-1…C-4, NFR-18, NFR-19 |
| M10 | Evaluation | load, fault-injection, SSRF, retention tests + `/metrics` | the measurements for the thesis evaluation chapter | §2.3, NFR-6, NFR-20 |

M0–M8 are backend (`probeboard-api`), M9 is `probeboard-web`, M10 spans both.

**Order rationale.** M3 before M4 because the executor is a pure function and
needs no scheduler to test. M4 before M5 because aggregates need a real stream
of results. M6 before M7 because there is nothing to notify about until
incidents exist. M10 last because it measures the finished system — and it is
the milestone that produces the thesis's evaluation chapter, so it is not
optional.

## Cross-cutting, from M0 onward

- Config validated at startup; the process refuses to boot on a bad value.
- Structured logs: static message, variable data in fields, never a secret.
- Every failure path explicit; nothing swallowed.
- Every bug fix ships with the test that fails without it.
- Migrations are forward-only and checked in.
