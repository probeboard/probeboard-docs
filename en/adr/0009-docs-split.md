# ADR-0009: Where each document lives

- **Status:** accepted
- **Date:** 2026-09-12
- **Relates to:** all documentation

## Context

probeboard has a documentation repository and, now, code repositories that
also need documentation. Without a rule, the same material gets written twice
and the two copies diverge — at which point a reader cannot tell which is
current, and both become untrustworthy.

The failure is not hypothetical. The api repository's README describes the
module layout; chapter 7 describes the architecture. Both are about structure,
and left unmanaged they will drift the first time a directory is renamed.

## Options considered

### Option A — everything in probeboard-docs
One place to look. But a README is what a developer reads first on arriving in
a repository, and a setup guide that lives elsewhere is a setup guide nobody
reads. It also cannot be reviewed in the pull request that changes the code it
describes.

### Option B — everything beside the code
Keeps documentation and code in step. But the thesis is not about a repository;
it is about a problem and a design, and chapters that live inside an
implementation repository implicitly claim the implementation is the subject.
The Armenian thesis text has no home there at all.

### Option C — split by a rule

## Decision

Option C, with one test applied to each document:

> **Would this go stale if the code changed and nobody updated it?**
> Yes → it lives with the code.
>
> **Would it still be true if the backend were rewritten in Go?**
> Yes → it lives in probeboard-docs.

| Lives in `probeboard-docs` | Lives with the code |
|---|---|
| Problem statement, requirements | README: how to run, build, extend |
| The health model and its formulas | Module layout and the dependency rule |
| Prior art and comparisons | Configuration reference |
| Capabilities, scope, positioning | Migration workflow |
| PRD: user stories, acceptance criteria | API endpoint reference (generated) |
| Architecture: components, data flow | Schema reference (generated from migrations) |
| ADRs | Verification records from running the system |
| Implementation plan | Runbooks and troubleshooting |
| Armenian thesis text | Test strategy as implemented |

The two are different kinds of writing. probeboard-docs answers *what and why*
for a reader who may never open the source. The code repositories answer *how*
for someone who has it checked out.

## Rationale

The test is really about **review and lifecycle**. A document that must change
when the code changes should be reviewed in the same pull request, by someone
looking at the diff; otherwise it is updated late, or never. A document that is
true independently of the implementation should not be churned by refactors,
and should stay reviewable as prose.

It also matches how each is read. Nobody clones a repository to read a problem
statement, and nobody consults a thesis chapter to find out which port the
service listens on.

## Avoiding drift at the boundary

Three rules, since the seam is where duplication appears:

1. **The code repository links rather than restates.** Its README says *why*
   only far enough to make the *how* make sense, then links to the chapter.
2. **The docs repository avoids file paths.** Chapter 7 describes components
   and responsibilities; it does not name directories, because those are
   exactly what a refactor changes.
3. **Generated reference is generated.** API and schema references are produced
   from code, never written by hand, so they cannot be stale.

## Consequences

- Easy: a pull request that renames a directory updates the README in the same
  diff, and the thesis chapters are untouched by refactoring.
- Harder: a reader needs both repositories for the full picture. Mitigated by
  the index in each README.
- A decision that spans both — an ADR — lives in probeboard-docs, because it
  survives the implementation. The code links to it.
- Revisit if: the repositories are ever merged into a monorepo, where the
  distinction becomes directories rather than repositories, though the rule
  itself would still apply.
