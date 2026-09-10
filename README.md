# probeboard — documentation

Thesis text and design documentation for the probeboard API monitoring dashboard.

## Layout

```
en/    Working design docs (English) — the source of truth while building
hy/    Thesis text (Armenian) — what the committee reads
```

The English docs are written first and evolve with the code. The Armenian
chapters are derived from them; `hy/00-outline.md` maps each chapter to its
English source so the two stay reconcilable.

## English documents

| File | Purpose |
|---|---|
| `en/01-problem-statement.md` | What problem exists, why it is non-trivial, scope |
| `en/02-requirements.md` | Numbered FR/NFR, acceptance criteria |
| `en/03-architecture.md` | Components, data flow, technology choices |
| `en/04-data-model.md` | Schema, retention and aggregation design |
| `en/05-implementation-plan.md` | Milestones in dependency order |
| `en/adr/` | Architecture Decision Records — one per significant decision |

## Why ADRs

A thesis defence asks "why this and not that". An ADR records the alternatives
considered and the reason for the choice **at the time the choice was made**,
while the reasoning is still fresh. Reconstructing it months later, from code
alone, is unreliable and shows in the defence.

One ADR per decision, numbered, never edited after acceptance — a decision that
changes gets a new ADR that supersedes the old one. The old one stays: the
history of a reversed decision is itself worth writing about.
