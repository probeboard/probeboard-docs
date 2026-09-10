# 2. Requirements

Requirements are numbered so that code, tests and thesis chapters can cite them.
Each carries a priority: **M** must-have (thesis is incomplete without it),
**S** should-have, **C** could-have (implement only if time allows).

## 2.1 Functional requirements

### Accounts and access

| ID | Pri | Requirement |
|---|---|---|
| FR-1 | M | A visitor can register with an email address and a password. |
| FR-2 | M | A registered user can log in and receives a session credential. |
| FR-3 | M | Every monitor belongs to exactly one user. A user can read and modify only their own monitors and only their own probe data. |
| FR-4 | S | A user can change their password. |
| FR-5 | C | A user can delete their account, which removes all their monitors and probe history. |

### Monitor management

| ID | Pri | Requirement |
|---|---|---|
| FR-6 | M | A user can create a monitor by supplying a name and an HTTP(S) URL. |
| FR-7 | M | A monitor has a configurable probe interval, chosen from a bounded set (e.g. 30s, 1m, 5m, 15m, 1h). |
| FR-8 | M | A monitor has a configurable request timeout, bounded by a system maximum. |
| FR-9 | M | A user can edit, delete, and pause/resume a monitor. A paused monitor is not probed. |
| FR-10 | M | A user can list their monitors with each one's current status and latest response time. |
| FR-11 | S | A monitor can specify the HTTP method and custom request headers. |
| FR-12 | S | A monitor defines which HTTP status codes count as success (default: 2xx). |
| FR-13 | S | A monitor can assert that the response body contains — or does not contain — a given string. |
| FR-14 | C | A monitor can send a request body (for POST/PUT endpoints). |
| FR-15 | C | A monitor can assert on a value extracted from a JSON response via a path expression. |
| FR-16 | M | The number of monitors per user is capped by a configurable quota. |

### Probing

| ID | Pri | Requirement |
|---|---|---|
| FR-17 | M | The system probes every active monitor at its configured interval without operator intervention. |
| FR-18 | M | Each probe records: monitor, timestamp, success flag, HTTP status code, response time in milliseconds, and on failure a failure classification. |
| FR-19 | M | Each probe is bounded by the monitor's timeout. Exceeding it is recorded as a timeout failure. |
| FR-20 | M | Failures are classified into distinguishable kinds: DNS resolution failure, connection refused, connection timeout, read timeout, TLS error, unexpected status code, failed body assertion. |
| FR-21 | S | Redirect-following is configurable per monitor, with a bounded redirect count. |
| FR-22 | S | For HTTPS monitors the system records the TLS certificate's expiry date. |
| FR-23 | C | A user can trigger an immediate out-of-schedule probe of one of their monitors ("check now"). |

### Incidents and notification

| ID | Pri | Requirement |
|---|---|---|
| FR-24 | M | An incident opens when a monitor produces N consecutive failed probes, where N is configurable per monitor (default 3). A single failure does not open an incident. |
| FR-25 | M | An open incident closes when the monitor produces M consecutive successful probes (default 2). |
| FR-26 | M | An incident records its start time, end time (if closed), and the failure classification that triggered it. |
| FR-27 | M | A user can view the incident history of a monitor. |
| FR-28 | S | The system notifies the user when an incident opens and when it closes. |
| FR-29 | S | Notification channels are configurable per user: email and outgoing webhook at minimum. |
| FR-30 | C | Telegram as an additional notification channel. |
| FR-31 | S | The system notifies the user when a monitored TLS certificate is within X days of expiry. |

### Statistics and visualization

| ID | Pri | Requirement |
|---|---|---|
| FR-32 | M | For a chosen time window (24h, 7d, 30d) the system reports a monitor's uptime percentage. |
| FR-33 | M | The system renders response time over time as a line chart. |
| FR-34 | M | The chart shows aggregate response time statistics, at minimum average and 95th percentile. |
| FR-35 | M | The system renders a status timeline showing when the monitor was up and down across the window. |
| FR-36 | S | A dashboard view summarises all of a user's monitors at once. |
| FR-37 | S | The user can select an arbitrary custom time range. |
| FR-38 | C | Probe history is exportable as CSV. |

## 2.2 Non-functional requirements

### Correctness and timing

| ID | Pri | Requirement |
|---|---|---|
| NFR-1 | M | A slow or unresponsive endpoint must not delay probes of other monitors. Probes execute concurrently. |
| NFR-2 | M | A monitor due at interval T is probed with scheduling drift under 10% of T. Drift must not accumulate across cycles. |
| NFR-3 | M | No monitor is probed twice for the same scheduled slot, including when several worker instances run concurrently. |
| NFR-4 | M | If a worker dies mid-probe, its claimed work becomes available to another worker within a bounded time rather than being lost. |
| NFR-5 | S | The response time recorded is the endpoint's, not the system's: queueing delay inside probeboard must not be counted in the measurement. |

### Scale and storage

| ID | Pri | Requirement |
|---|---|---|
| NFR-6 | M | The system sustains at least 500 active monitors at a 60-second interval on a single modest host. |
| NFR-7 | M | Probe throughput scales by adding worker instances, with no change to configuration of existing components. |
| NFR-8 | M | Storage growth is bounded. Raw probe rows are retained for a configured window; older data is retained only as time-bucketed aggregates. |
| NFR-9 | M | Statistics over long windows are served from aggregates, not by scanning raw rows. A 30-day chart must not require reading 43,200 rows per monitor. |

### Security

| ID | Pri | Requirement |
|---|---|---|
| NFR-10 | M | Passwords are stored only as salted hashes from a memory-hard algorithm (Argon2id or bcrypt). |
| NFR-11 | M | The probe executor refuses URLs that resolve to loopback, link-local, or private address ranges, and to cloud metadata addresses. Validation occurs after DNS resolution, immediately before connecting, so that DNS rebinding cannot bypass it. |
| NFR-12 | M | Probe requests carry no credentials belonging to probeboard or to other users. |
| NFR-13 | M | The response body is read only up to a bounded size; it is used for assertions and is never persisted in full. |
| NFR-14 | S | Authentication endpoints are rate limited to resist credential stuffing. |
| NFR-15 | S | User-supplied request headers cannot override headers the system controls, and cannot be used to smuggle a second request. |

### Operability and usability

| ID | Pri | Requirement |
|---|---|---|
| NFR-16 | M | The whole system starts from a single command in a clean environment (`docker compose up`). |
| NFR-17 | S | Every service exposes a health endpoint and emits structured logs. |
| NFR-18 | S | The web interface is usable at both desktop and mobile widths. |
| NFR-19 | S | Charts remain legible for a monitor with 30 days of history without the browser stalling. |
| NFR-20 | C | The system exposes its own operational metrics in Prometheus format. |

## 2.3 Acceptance criteria for the thesis

The implementation is considered complete when:

1. All **M** requirements are implemented and demonstrable.
2. A load test demonstrates NFR-6 and NFR-7: 500 monitors probed on schedule,
   then throughput increased by adding a second worker.
3. A fault-injection test demonstrates NFR-3 and NFR-4: two workers running
   concurrently produce no duplicate probes, and killing one mid-probe results
   in the work being retried by the other.
4. A security test demonstrates NFR-11 against each blocked address class,
   including a DNS-rebinding attempt.
5. A storage test demonstrates NFR-8 and NFR-9: after retention runs, long-window
   statistics are still correct and are served without scanning raw rows.

Items 2–5 are what turn this from an application into a thesis; each produces a
measurement that belongs in the evaluation chapter.
