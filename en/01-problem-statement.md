# 1. Problem statement

## 1.1 Context

Modern software is assembled from HTTP services: a product depends on its own
backend, on third-party payment providers, on mapping and authentication
services, on internal microservices. When one of these endpoints stops
responding, or starts responding slowly, the failure propagates to end users.

The party that suffers the outage is usually not the first to notice it.
Detection is commonly reactive: a user complains, and only then does anyone look
at the service. The interval between "the endpoint broke" and "somebody noticed"
is unmeasured and frequently long.

## 1.2 The problem

There is no *feedback loop* between an HTTP endpoint's actual behaviour and the
people responsible for it. Specifically, three things are missing:

1. **Continuous observation.** Nobody is checking the endpoint when no user
   happens to be exercising it. Failures during quiet hours go unseen.
2. **Historical record.** Even when a failure is noticed, there is no record of
   when it began, how long it lasted, or whether it has happened before.
   Questions like "is this endpoint getting slower over the last month?" cannot
   be answered because the data was never collected.
3. **Notification.** Discovery depends on a human happening to look.

## 1.3 Proposed solution

**probeboard** is a web application that closes this loop. A user registers the
endpoints they care about. The system probes each one on a configured interval,
records the outcome of every probe, detects sustained failures, notifies the
user, and presents the accumulated history as statistics and charts.

The user-facing value is answering four questions at any moment:

- Is this endpoint up **right now**?
- How **fast** is it responding, and is that changing over time?
- What **percentage** of the last day / week / month was it available?
- **When** did it break, and for how long?

## 1.4 Why this is a non-trivial engineering problem

The naive implementation — a loop that walks a list of URLs and calls each one —
fails on contact with reality. The substance of this thesis is in the problems
that appear immediately afterwards:

**Scheduling.** Monitors have different intervals. A monitor due every 60s must
actually be probed every 60s, not every 60s *plus however long the previous pass
took*. Drift accumulates. Work must be distributed across multiple workers
without two workers probing the same monitor simultaneously, and without a
crashed worker silently dropping its monitors.

**Isolation.** A single endpoint that accepts a TCP connection and then never
responds will, in a sequential implementation, block every monitor behind it.
Probes must be concurrent and independently bounded by timeout.

**Data volume.** One monitor at a 60-second interval produces ~43,200 rows per
month. Five hundred monitors produce ~21.6 million. Retaining raw probe results
indefinitely is not viable, but the statistics must still cover long windows.
This forces a retention and downsampling design.

**Distinguishing failure from flapping.** A single failed probe is not an
outage — it may be a transient packet loss or a brief deploy. Alerting on every
failed probe produces noise that trains the user to ignore alerts. The system
needs a notion of *sustained* failure and hysteresis between states.

**Security.** The user supplies the URL, and the server fetches it. This is a
textbook Server-Side Request Forgery primitive: an unprotected implementation
lets any registered user aim the server at `http://169.254.169.254/`, at
`http://localhost:5432`, or at any host inside the deployment's private network,
and read the response. Any system of this shape must defend against it
deliberately; the defence is discussed in the architecture document.

## 1.5 Scope

**In scope:** HTTP/HTTPS endpoint monitoring, user accounts, configurable probe
intervals and assertions, incident detection, notification, statistics and
charts, TLS certificate expiry tracking.

**Out of scope:** monitoring protocols other than HTTP (ICMP ping, TCP port,
DNS), synthetic multi-step browser transactions, distributed probing from
multiple geographic regions, on-call scheduling and escalation policies,
team/organization accounts with shared monitors.

These exclusions are deliberate. Each is a reasonable extension and is revisited
in the closing chapter as future work.

## 1.6 Comparable systems

| System | Nature | Relevant difference |
|---|---|---|
| UptimeRobot | Commercial SaaS | Closed source; probing architecture not inspectable |
| Pingdom | Commercial SaaS | Same; oriented to page-load rather than API semantics |
| Better Stack | Commercial SaaS | Same |
| Uptime Kuma | Open source, self-hosted | Closest comparison. Single-process Node application; does not scale probing horizontally and stores every check row indefinitely |

probeboard's distinguishing goal is not feature parity with commercial products
but a defensible design for the two problems above: horizontally scalable
probe scheduling, and bounded storage growth under continuous data collection.
