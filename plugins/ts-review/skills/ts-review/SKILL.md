---
name: ts-review
description: Review existing TypeScript code against type safety, boundary validation, error handling, async correctness, SOLID module design, test quality, and appropriate design patterns. Use this when the user asks to review, audit, or critique TypeScript or Node code — "review this", "does this look right", "why is this promise not resolving", "is this type-safe" — or when a tsc, eslint, or unhandled-rejection finding needs interpreting. For writing new TypeScript, use ts-coding instead; the Stop hook applies this skill automatically after edits.
---

# TypeScript review

Review TypeScript the way an experienced engineer does: mechanical checks first, then boundary safety, then async correctness, then design, then style. The ordering matters because what goes wrong in TypeScript in production is almost never a type error — the compiler caught those — it is data that did not match its annotation, a promise nobody awaited, or an error nobody caught.

## Run the tools before reading

Find the toolchain first: the lockfile names the package manager, `package.json` scripts name what the team runs, `tsconfig.json` names the strictness. Use the project's scripts when they exist.

```bash
<pm> exec tsc --noEmit -p tsconfig.json     # type errors; respects the project's strictness
<pm> run lint                               # or: <pm> exec eslint <changed files>
<pm> exec prettier --check <changed files>  # if a prettier config exists
<pm> test -- <changed test files>           # vitest/jest accept paths; do not run everything
```

Two things the tools tell you that reading does not:

- **Strictness.** If `strict` is off, or `noUncheckedIndexedAccess` is off, the compiler did not check the things you would assume it checked. Say so in the mechanical-checks section; it changes how carefully you must read.
- **Lint coverage of async bugs.** `@typescript-eslint/no-floating-promises` and `no-misused-promises` are the only mechanical defence against the most common production bug in Node. If they are not on, you must read for floating promises by hand, and the absence is itself a WARNING.

## Then read for these five dimensions

Load the relevant reference file rather than working from memory. Each is a checklist plus the reasoning behind it:

| Dimension | Reference | Read it when |
|---|---|---|
| Idiomatic TypeScript | `references/idiomatic.md` | Always — types, boundaries, errors, null handling, modules, naming |
| Async & concurrency | `references/async.md` | The code contains `async`, `Promise`, streams, timers, event emitters, or outbound calls |
| SOLID in TypeScript | `references/solid.md` | Reviewing module boundaries, interfaces, classes, or dependency wiring |
| Design patterns | `references/patterns.md` | Evaluating whether a pattern fits, or whether an abstraction is premature |
| Tests | `references/testing.md` | The diff adds or changes tests, or adds code with no test |

Boundary and async findings outrank everything else. Unvalidated input with a confident type annotation is a runtime crash or a security hole that the compiler blessed; a floating promise is a process crash on the first rejection. An unidiomatic name is a readability cost. Do not report them at the same severity.

## Severity

Three levels, and the boundaries are deliberate:

- **BLOCKING** — unvalidated data crossing a trust boundary, floating promises and missing awaits on paths that matter, swallowed errors, check-then-act races across an `await`, unbounded fan-out, event-loop-blocking calls on a server request path, `any` or an assertion that hides a real type error, security issues, incorrect logic, resource leaks (listeners, timers, handles). These stop the work.
- **WARNING** — design problems that will cost later: missing cancellation or timeouts, interfaces on the wrong side of the boundary, a module doing two unrelated jobs, circular imports, import-time side effects, tests that do not exercise the failure mode, lint rules that would have caught a BLOCKING class but are off.
- **NIT** — naming, comment wording, ordering, `type` vs `interface`, anything a formatter or linter could have said.

Severity inflation destroys the signal. If BLOCKING sometimes means "I would have named this differently," the author learns to skim the BLOCKING section, and then a real floating promise gets skimmed too. When genuinely unsure, use WARNING and say what you are unsure about.

## Output format

Use this structure:

```
## Mechanical checks
tsc: <clean | errors> (strict: <on | off>, noUncheckedIndexedAccess: <on | off>)
lint: <clean | summary | not configured> (no-floating-promises: <on | off>)
format: <clean | files | not configured>
tests: <not run | pass | failures>

## BLOCKING
1. `path/file.ts:42` — <one-line problem>
   Why: <the concrete failure mode>
   Fix: <the specific change>

## WARNING
## NIT

VERDICT: CLEAN | CHANGES_REQUIRED
```

`VERDICT: CLEAN` requires zero BLOCKING findings. Warnings and nits do not block.

## Two habits worth keeping

**Read whole files, not diff hunks.** A diff hides the contract. An added `await` is correct or a deadlock depending on the lock above it, and a removed `await` is a bug or an optimisation depending on whether anything downstream needs the result.

**Say when the code is fine.** One line — "reviewed, no blocking findings, two nits below" — and stop. Manufacturing findings to look thorough trains the author to discount the review.
