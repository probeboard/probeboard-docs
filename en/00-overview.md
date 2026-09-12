# 0. probeboard 101 — the whole project in plain language

Orientation document. No jargon without explaining it first. Read this before
the other chapters; it maps to all of them at the end.

---

## 0.1 One paragraph

You give probeboard a list of web addresses you care about. probeboard calls
each one every minute, forever, and writes down what happened: did it answer,
how fast, and was the answer correct. When something breaks it emails you. When
you open the dashboard it shows you charts of how things have been behaving.

That is the whole product. Everything else in these documents is about doing
that *correctly* when there are 500 addresses instead of 3, and when the
program doing the checking might itself crash.

---

## 0.2 A concrete example

Say you run an online shop. It depends on three things:

| Thing | Address | Why you care |
|---|---|---|
| Your own backend | `api.myshop.com/health` | if it is down, nothing works |
| A payment provider | `api.stripe.com/v1/charges` | if it is down, nobody can pay |
| Your search service | `search.myshop.com/query` | if it is slow, people leave |

Today, you find out any of these broke when a customer complains. That might be
six hours after it happened, and at 03:00 nobody complains at all — they just
leave.

With probeboard:

1. You register a **service** called "MyShop API", with a base address.
2. Under it you add **endpoints** — the specific addresses to check.
3. probeboard checks each one every 60 seconds, from now on, without being told
   again.
4. At 03:14 the payment provider starts returning errors. After three failed
   checks in a row — 03:17 — you get an email.
5. At 03:41 it recovers. You get a second email saying it lasted 27 minutes.
6. Next morning you open the dashboard and see exactly when it broke, how long
   it lasted, and that this is the third time this month.

That is the loop: **register → check → observe → alert → diagnose**.

---

## 0.3 How it works, end to end

Follow one single check through the system. Every step here has a whole section
in chapter 7; this is the story without the detail.

**1. Something decides it is time.** A part of the program called the
**scheduler** keeps a list: "endpoint #42 is due at 03:14:00". Every second it
asks the database "what is due now?" and takes what it finds.

**2. It claims the work.** If we run three copies of the program for speed, all
three would try to check endpoint #42 at the same time — three emails, three
rows of data, all duplicated. So a copy must first *claim* it, like taking a
ticket. The database guarantees only one copy gets each ticket.

**3. It makes the call.** The **prober** does what your browser does: looks up
the address, opens a connection, sends the request, waits, reads the answer. It
gives up after a set time so one dead address cannot freeze everything else.

**4. It measures the pieces, not just the total.** Not "it took 900 ms" but
where those 900 ms went — looking up the name, connecting, securing the
connection, waiting for the server to think, downloading the answer. That is the
difference between "it's slow" and "*your DNS provider* is slow".

**5. It checks the answer is actually right.** An address can answer `200 OK` —
the code meaning "fine" — while the body says `{"error": "database down"}`.
Checking only the status code would call this healthy. So we also check the
content.

**6. It writes down what happened.** One row: when, succeeded or not, how fast,
which piece was slow, and if it failed, *what kind* of failure.

**7. It updates the summaries.** Storing every row forever does not work (see
§0.5). So alongside the raw row we update running totals: this minute, this
hour, this day.

**8. It decides whether this is an emergency.** One failed check is not an
outage — networks hiccup. Three in a row is. The program tracks the streak, and
only when it crosses the line does it open an **incident**.

**9. It sends the email.** Not immediately per failure — that is how people
learn to ignore alerts. One email per incident, per service, with a summary.

**10. You open the dashboard** and the charts are already there, because step 7
has been running all along.

---

## 0.4 The pieces

```
     your browser
          │
   ┌──────▼──────┐   the screens: list of services, charts, forms
   │     web     │
   └──────┬──────┘
          │
   ┌──────▼──────┐   answers the screens' questions.
   │     api     │   never checks anything itself.
   └──────┬──────┘
          │
   ┌──────▼──────────────────────────────┐
   │            PostgreSQL               │   everything is remembered here
   └──────▲──────────────────────────────┘
          │
   ┌──────┴──────┐   does the actual checking, forever, in the background.
   │   worker    │   run 1 of these, or 5. they share the work automatically.
   └─────────────┘
```

Why **api** and **worker** are separate: the checking must keep running at
exactly the right times even while a hundred people are loading dashboards. If
they were the same program, a heavy page load could delay a check. Separating
them also means you can run five workers and one api — you scale the part that
is actually busy.

Why only **PostgreSQL** and nothing else: many systems like this also need
Redis, a message queue, a time-series database. Each is another thing to install,
configure, back up and explain. We get the same guarantees from Postgres alone,
which means the whole system starts with one command.

---

## 0.5 Why this is harder than it looks

If a professor asks "isn't this just a `for` loop that calls `curl`?", these five
answers are why it is not. Each one is a chapter of the thesis.

### Problem 1 — the loop falls behind

Naive version: loop over all addresses, check each, sleep 60 seconds, repeat.

With 500 addresses where each takes 200 ms, one pass takes 100 seconds. So you
check every 160 seconds, not every 60 — and it gets worse each pass. This is
called **drift**, and it accumulates.

Worse: one address that accepts your connection and then never answers blocks
every address behind it. One broken thing freezes everything.

*Fix:* checks run in parallel, each with its own time limit, and the next run
time is calculated from the *scheduled* time, not from when the last one
finished.

### Problem 2 — two copies do the same work

You want to run several copies for speed and safety. But all copies read the
same list, so every check happens twice — duplicated data, duplicated emails.

And if a copy crashes mid-check, its work is silently lost. Nobody notices,
because there is nothing to notice *with*: the address just stops being checked.

*Fix:* a copy must claim an address before checking it, and the claim expires.
If a copy dies, its claims become available again after 30 seconds. One database
query does all of this.

### Problem 3 — the data becomes enormous

One address, checked every 60 seconds:

```
  1 per minute × 60 × 24        =     1,440 rows per day
                  × 30          =    43,200 rows per month
  500 addresses × 43,200        = 21,600,000 rows per month
```

Twenty-one million rows a month, growing forever. And drawing a 30-day chart
would mean reading 43,200 rows *for one address*.

*Fix:* keep the detailed rows for a short while, and alongside them keep
summaries — one row per minute, per hour, per day. The 30-day chart reads 30
rows instead of 43,200. Old detail is deleted in one cheap operation.

**The catch that makes this interesting:** you cannot average percentiles.
"The slowest 5% of requests today" cannot be recovered from 24 hourly values of
"the slowest 5% of requests that hour". This is not an implementation
difficulty, it is arithmetic. Our solution is to keep, per summary row, a count
of how many checks fell into each speed range — those counts *can* be added
together. The closest comparable open-source project, Uptime Kuma, keeps only
averages, and therefore genuinely cannot answer this question at all.

### Problem 4 — alerts that get ignored

If you email on every failed check, a 30-minute outage sends 30 emails. After
one week of that, the user filters your emails to a folder they never read. The
feature is now worse than useless — it is a false sense of safety.

*Fix:* alert on *sustained* failure, not on a failure. Three in a row opens an
incident; two successes in a row closes it. One email when it opens, one when it
closes, with re-reminders slowing down over time. And if six endpoints of one
service fail together, that is one email, not six.

### Problem 5 — the user hands us a URL and we fetch it

This is the security problem, and it is the most serious one.

The user types an address and *our server* calls it. Nothing stops them typing
`http://localhost:5432` — our own database. Or `http://169.254.169.254/`, which
on a cloud server returns the server's own access credentials. They would see the
response in the dashboard. This attack has a name: **SSRF**, server-side request
forgery.

Blocking those addresses in the text field is not enough. An attacker registers
`evil.com`, which points to a harmless address when we check it, and to
`127.0.0.1` two seconds later when we actually connect. The name stayed the same;
the address changed underneath us. This is **DNS rebinding**.

*Fix:* look up the address ourselves, check every result, and then force the
connection to use *that exact address* — so it cannot be swapped between the
check and the connection. This is a live problem: real security advisories were
published against Node.js libraries for exactly this bypass.

---

## 0.6 Glossary

Terms you will hear in every meeting about this project.

> ⚠️ The Armenian column is a **draft** — Armenian technical terminology varies
> by department. Confirm each with your supervisor before the thesis text uses
> it; some may be better left as transliterations.

### The domain

| Term | What it actually means | Armenian (draft) |
|---|---|---|
| **Endpoint** | One specific web address we check, e.g. `POST /orders` | վերջնակետ |
| **Service** | A group of endpoints belonging to one API | ծառայություն |
| **Probe / check** | One single attempt to call an endpoint | ստուգում |
| **Probe result** | What that one attempt produced | ստուգման արդյունք |
| **Interval** | How often we check — every 60 s, every 5 min | ստուգման պարբերություն |
| **Timeout** | How long we wait before giving up | սպասման սահմանաժամ |
| **Uptime** | Percentage of the time it was working | հասանելիության տոկոս |
| **Downtime** | Time it was not working | անհասանելիության ժամանակ |
| **Incident** | One continuous period of being broken | միջադեպ |
| **Assertion** | A rule the answer must satisfy, e.g. body contains `"ok"` | պնդում / ստուգման պայման |
| **Alert / notification** | The email we send you | ծանուցում |
| **Maintenance window** | "We are deploying, do not alert me" | սպասարկման պատուհան |

### Measurement

| Term | What it actually means | Armenian (draft) |
|---|---|---|
| **Latency / response time** | How long the answer took | պատասխանի ժամանակ |
| **TTFB** (time to first byte) | How long the *server itself* took to start answering. The one number that measures the API rather than the network | առաջին բայթի ժամանակ |
| **Percentile / p95** | "95% of checks were faster than this." Far more useful than an average — see below | պրոցենտիլ |
| **MTTR** | Mean time to recovery — on average, how long outages last | վերականգնման միջին ժամանակ |
| **MTBF** | Mean time between failures — on average, how often it breaks | խափանումների միջև միջին ժամանակ |
| **Flapping** | Rapidly alternating up and down | անկայուն վիճակ |
| **Degraded** | Working, but too slow to be called healthy | դեգրադացված վիճակ |

**Why p95 and not the average.** Suppose 95 checks take 40 ms and 5 take
30 seconds. The average is about 1.5 seconds — a number that describes *none* of
the 100 checks. It is slower than every fast one and 20× faster than every slow
one. p95 says "95% were under 40 ms", p99 says "but the worst ones are 30 s".
Both are true and useful; the average is neither.

### Reliability engineering

| Term | What it actually means | Armenian (draft) |
|---|---|---|
| **SLI** | A thing you measure, e.g. "% of checks that succeeded" | ծառայության ցուցանիշ |
| **SLO** | A target for it, e.g. "at least 99.5% over 30 days" | ծառայության նպատակ |
| **SLA** | An SLO with money attached to failing it. Not our scope | ծառայության համաձայնագիր |
| **Error budget** | The failure the SLO permits. 99.5% over 30 days = **3h 36m** of allowed downtime | սխալի բյուջե |
| **Burn rate** | How fast you are spending that budget. 14.4× means a 30-day budget gone in 50 minutes | բյուջեի ծախսման տեմպ |

Error budget is the idea worth internalising: it converts "be reliable" from a
feeling into arithmetic. 99.5% sounds nearly perfect until you compute that it
permits three and a half hours of downtime a month — and then you can say "we
have used 40 minutes, we have 2h 56m left".

### Engineering mechanics

| Term | What it actually means | Armenian (draft) |
|---|---|---|
| **Scheduler** | The part that decides what to check and when | ժամանակացույց |
| **Worker** | A background process doing the checking. Run several | ֆոնային գործընթաց |
| **Drift** | Checks gradually happening later than they should | ժամանակային շեղում |
| **Lease / claim** | "I am handling this one" — expires, so a crash releases it | ժամանակավոր զբաղեցում |
| **`SKIP LOCKED`** | A Postgres feature: "give me rows nobody else took, do not wait" | — |
| **Rollup / aggregate** | A precomputed summary of many rows | ամփոփ տվյալներ |
| **Downsampling** | Keeping summaries instead of full detail as data ages | տվյալների նոսրացում |
| **Retention** | How long we keep data before deleting it | պահպանման ժամկետ |
| **Partition** | A slice of a table by date, so deleting old data is instant | բաժանում |
| **Histogram bucket** | A speed range with a counter. Ranges can be added together; percentiles cannot | հիստոգրամի միջակայք |
| **Hysteresis** | Needing several failures to declare "down" and several successes to declare "up" — stops flapping | հիստերեզիս |
| **Idempotent** | Doing it twice has the same effect as once. Essential when retrying | իդեմպոտենտ |
| **Outbox** | Saving "send this email" in the database in the same transaction as the event, so neither can exist without the other | ելքային հերթ |

### Monitoring concepts

| Term | What it actually means | Armenian (draft) |
|---|---|---|
| **Black-box monitoring** | Checking from outside, like a normal user. **This is us** | արտաքին մոնիթորինգ |
| **White-box monitoring** | The program reports on itself from inside. Datadog, Prometheus | ներքին մոնիթորինգ |
| **Synthetic monitoring** | Generating fake traffic to test, rather than watching real traffic | սինթետիկ մոնիթորինգ |
| **SSRF** | Tricking a server into calling addresses on its own private network | սերվերային հարցման կեղծում |
| **DNS rebinding** | Changing what an address points to between the safety check and the connection | DNS վերակապում |

---

## 0.7 Questions you will be asked, and the answers

**"Isn't this just UptimeRobot?"**
Feature-wise, it is a smaller version of one. The thesis is not the feature
list — it is the two problems those products solve behind closed doors and never
show you: spreading the checking across several machines without duplicating or
losing work, and keeping the data bounded without losing the ability to answer
questions about the past. We show both.

**"Why not just use Uptime Kuma, it is free?"**
Because we read its source and found the limits. It runs in exactly one process
— its scheduling state lives in memory, so a second copy would double every
check. And it stores only averages, so it cannot tell you your slowest 5%. Both
are consequences of design decisions we make differently, and we can point at
the specific file.

**"Why not Prometheus and Grafana?"**
Different tool for a different question. They watch systems that report on
themselves from the inside; they cannot tell you anything about Stripe's API,
because you cannot install anything inside Stripe. We check from outside, which
is the only way to measure something you do not own. We also export our data in
Prometheus format, so anyone who *does* run Grafana can point it at us.

**"How is checking a URL a thesis?"**
Checking one URL is not. The thesis is: 500 of them, on time, across several
machines, with no duplicates, no lost work when a machine dies, bounded storage,
statistics that survive the data being summarised, and a security guard against
an attack that had CVEs published against real libraries this year. Each of those
is measurable, and we measure them.

**"What happens if your own program crashes?"**
This is the question we are proudest of. Most monitoring systems answer it
badly: if the checker dies, nothing is checked, and the dashboard stays green —
so the failure is invisible in exactly the direction that hurts. We treat a
missing check as a distinct state called `UNKNOWN` — not up, not down — which is
visible on screen and excluded from uptime calculations. The system can report
its own failure.

---

## 0.8 What we are deliberately not building

Saying no, and being able to explain why, is worth as much as the features.

| Not building | Why |
|---|---|
| Drag-and-drop dashboards like Grafana | That editor *is* Grafana's product — months of UI work proving nothing |
| Log collection, tracing (like Datadog's other half) | Different problem entirely: swallowing millions of messages a second |
| Checking from several countries | Needs servers in several countries |
| Who-is-on-call rotas, escalation | PagerDuty's product, unrelated to measuring |
| Multi-step scripted browser tests | The hard part is driving a browser, not monitoring |
| Teams sharing monitors | Reasonable, just not needed to prove the thesis |

Every one of these goes in the final chapter as future work, with this reasoning.

---

## 0.9 Where everything is written down

| Chapter | Contains | Read it when |
|---|---|---|
| [01 Problem statement](01-problem-statement.md) | Why this matters, what is in and out of scope | writing the introduction |
| [02 Requirements](02-requirements.md) | 38 numbered features + 20 quality requirements | you need to cite a requirement |
| [03 API health](03-api-health.md) | The deep version of §0.6 — the formulas, the failure list, the maths | writing the theory chapter |
| [04 Prior art](04-prior-art.md) | What we found reading other projects' source code | defending "why not just use X" |
| [05 Capabilities](05-capabilities.md) | Where we sit next to Datadog and Grafana, feature tiers | scoping arguments |
| [06 PRD](06-prd.md) | Every screen and user story with acceptance criteria | building the frontend |
| [07 Architecture](07-architecture.md) | The tables, the queries, the module layout | writing the backend |
| [08 Plan](08-plan.md) | Milestones M0–M10 in order | deciding what to do next |

Armenian thesis text lives in [`../hy/`](../hy/).
