# ADR-0006: One repository for api and worker, structured to split

- **Status:** accepted
- **Date:** 2026-09-12
- **Relates to:** NFR-7, NFR-16

## Context

probeboard runs two processes: **api**, which serves the REST API and never
probes, and **worker**, which schedules and executes probes, rolls up
statistics, opens incidents and sends notifications. They are deployed as
separate containers and scaled independently — one api, N workers — because
NFR-7 requires throughput to grow by adding workers.

Separate deployment does not require separate repositories, and the question is
where the source lives.

What the two actually share is not a thin types package. It is most of the
domain:

| Shared | api needs it to | worker needs it to |
|---|---|---|
| Config schema | boot | boot |
| DB types + migrations | read every table | write every table |
| Failure taxonomy, states | render them | produce them |
| Assertion schema **and evaluator** | validate on save (B-2) | evaluate on probe |
| SSRF validation | reject URLs on save (B-7) | enforce at connect (NFR-11) |
| Histogram edges + percentile math | interpolate p95 for charts | write the buckets |
| Incident state machine | display state | drive transitions |

## Options considered

### Option A — one repository, layered directories
`src/core` shared, `src/api` and `src/worker` consuming it. Two entrypoints, one
image, two container commands.

### Option B — one repository, npm workspaces
`packages/core` as an internal package with `apps/api` and `apps/worker`
depending on it. What openstatus does (`apps/checker`, `apps/server`,
`packages/*`), and what OneUptime does with its `Common` module.

### Option C — separate repositories plus a published SDK
`probeboard-core` published to a registry; `probeboard-api` and
`probeboard-worker` depending on a version range.

## Decision

Option A, with the layering enforced automatically so that moving to Option B or
C later is a directory move rather than an untangling.

```
src/
  core/      depends on nothing else in src/
  api/       may depend on core, never on worker
  worker/    may depend on core, never on api
```

## Rationale

The decisive argument against Option C is **version skew**. With a published
SDK, api can run `core@1.1` while worker runs `core@1.2`. The worker then writes
histograms with 20 bucket edges while the api interpolates p95 using 18 —
silently wrong charts, no error raised anywhere, in a system whose entire
contribution is measurement correctness. That class of bug **cannot exist** when
both processes are built from one tree.

Option C also costs a publish-and-bump cycle on every shared change, three CI
pipelines, an arbitrary owner for the migrations, and three clones for anyone
reading the system.

The cases where splitting is genuinely right do not apply here: the worker is
not written in a different language (openstatus splits a *Go* checker from a
TypeScript app), and it is not a distributable artifact deployed by third
parties into their own networks — private and multi-region probing are out of
scope per §1.5.

Option B was rejected for now only on cost: project references, build wiring and
a more complex Docker context, in exchange for a boundary that Option A already
provides by enforcement rather than by packaging.

## Enforcement

The rule is checked by a test (`src/architecture.test.ts`), not merely written
down, because an unenforced convention decays. It parses every relative import —
including bare side-effect imports and `require()`, not just `from` clauses —
and asserts the direction of every edge.

The check was validated by deliberately introducing a violation and confirming
it fails. That first attempt **passed incorrectly**, because the initial regex
matched only `from '…'` and missed `import '../api/x'`. The lesson is recorded
here: a guard that has never been seen to fail is not known to work.

## Consequences

- Easy: shared domain logic with no packaging, no version skew, one CI pipeline,
  one image, independent scaling already available via compose.
- Harder: nothing prevents a future contributor from putting worker-only code in
  `core` — only the test's dependency-direction rule, not a package boundary.
- Splitting later means `git mv src/core packages/core`, adding workspace
  manifests, and following the import errors. The boundary test guarantees there
  are no cycles to unpick first.
- Revisit if: a probe executor in another language is introduced, or private
  locations (customer-deployed workers) enter scope.
