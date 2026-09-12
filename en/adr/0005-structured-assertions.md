# ADR-0005: Assertions stored as versioned structured JSON

- **Status:** accepted
- **Date:** 2026-09-12
- **Relates to:** FR-12, FR-13, FR-15

## Context

Users must express what counts as a correct response: status codes, body
content, JSON path comparisons. This is user-authored configuration that the
system will evaluate on every probe, forever, and whose format will need to
evolve.

## Options considered

### Option A — string DSL
Gatus's approach: conditions are plain strings evaluated against a result.

```
[STATUS] == 200
[BODY].user.name == john
len([BODY].data) < 5
has([BODY].errors) == false
[CERTIFICATE_EXPIRATION] > 48h
```

Placeholders `[STATUS]`, `[RESPONSE_TIME]`, `[IP]`, `[BODY]`, `[CONNECTED]`,
`[CERTIFICATE_EXPIRATION]`, `[DNS_RCODE]`; functions `len`, `has`, `pat`, `any`.

### Option B — structured JSON
openstatus's approach: a discriminated union on
`type ∈ {status, header, textBody, jsonBody, dnsRecord}` with comparator enums
and an explicit `version: "v1"` field.

## Decision

Option B for storage. Option A may later be added as optional sugar that
compiles into Option B.

## Rationale

- **Validation happens at save time, by construction.** A schema rejects an
  invalid assertion when the user submits the form, not on the first probe at
  03:00.
- **The API needs the same schema the worker does.** The API validates on save
  (PRD B-2), the worker evaluates on probe. A shared schema keeps those two in
  step; a parser would have to be shared identically anyway.
- **Versioning from day one.** A user-authored format that will outlive its
  first design needs an explicit version field. Retrofitting versioning onto a
  string DSL means writing a parser that can recognise which dialect it is
  reading.
- **The UI can be discoverable**: a form with typed fields rather than a free-text
  box plus documentation.

The string DSL is more pleasant to author and more interesting to implement, and
a parser would be defensible thesis work. It is not the foundation to build on,
so it is deferred rather than rejected.

## Consequences

- Easy: validation, evolution, form-based editing, sharing the definition
  between api and worker.
- Harder: authoring by hand is verbose; a builder UI is required rather than
  optional.
- Per-assertion results are stored on each probe row, so the UI can report
  *which* assertion failed rather than only that one did — a pattern taken from
  Gatus's `ConditionResults`.
