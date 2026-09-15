---
name: go-review
description: Review existing Go code against idiomatic Go, SOLID design, concurrency and race safety, test quality, and appropriate design patterns. Use this when the user asks to review, audit, or critique Go code — "review this", "does this look right", "is this idiomatic", "find the race" — or when a race detector, go vet, or staticcheck finding needs interpreting. For writing new Go, use go-coding instead; the Stop hook applies this skill automatically after edits.
---

# Go review

Review Go code the way an experienced Go engineer does: mechanical checks first, then correctness, then design, then style. The ordering matters because most of what goes wrong in Go is concurrency and error handling, not formatting — and formatting is already handled by a tool.

## Run the tools before reading

These take seconds and catch things reading misses:

```bash
gofmt -l .                 # anything listed is unformatted
go vet ./...               # printf mistakes, lock copies, loop closure bugs
go build ./...             # type errors, unused imports
go test -race ./...        # observed data races
staticcheck ./...          # if installed; skip silently if not
```

`go vet` in particular catches copied `sync.Mutex` values and `printf` arg mismatches that are easy to miss by eye. If `go vet` reports something, that is a finding — do not explain it away without reading the code it points at.

## Then read for these four dimensions

Load the relevant reference file rather than working from memory. Each is a checklist plus the reasoning behind it:

| Dimension | Reference | Read it when |
|---|---|---|
| Idiomatic Go | `references/idiomatic.md` | Always — naming, errors, context, zero values, control flow |
| Concurrency & races | `references/concurrency.md` | The code contains `go`, `chan`, `sync`, `atomic`, `select`, or `context` cancellation |
| SOLID in Go | `references/solid.md` | Reviewing package boundaries, interfaces, constructors, or dependency wiring |
| Design patterns | `references/patterns.md` | Evaluating whether a pattern fits, or whether an abstraction is premature |
| Tests | `references/testing.md` | The diff adds or changes tests, or adds code with no test |

Concurrency findings outrank everything else. A race is a correctness bug that ships silently and corrupts data under load; an unidiomatic name is a readability cost. Do not report them at the same severity.

## Severity

Three levels, and the boundaries are deliberate:

- **BLOCKING** — races, goroutine leaks, deadlock paths, dropped errors on a path that matters, resource leaks, security issues, incorrect logic. These stop the work.
- **WARNING** — design problems that will cost later: interfaces on the wrong side of the boundary, missing context propagation, a package doing two unrelated jobs, tests that do not exercise the failure mode.
- **NIT** — naming, comment wording, ordering, anything a formatter or linter could have said.

Severity inflation destroys the signal. If BLOCKING sometimes means "I would have named this differently," the author learns to skim the BLOCKING section, and then a real race gets skimmed too. When genuinely unsure, use WARNING and say what you are unsure about.

## Output format

Use this structure:

```
## Mechanical checks
gofmt: <clean | files>
go vet: <clean | summary>
build: <ok | error>
race: <not run | clean | findings>

## BLOCKING
1. `path/file.go:42` — <one-line problem>
   Why: <the concrete failure mode>
   Fix: <the specific change>

## WARNING
## NIT

VERDICT: CLEAN | CHANGES_REQUIRED
```

`VERDICT: CLEAN` requires zero BLOCKING findings. Warnings and nits do not block.

## Two habits worth keeping

**Read whole files, not diff hunks.** A diff hides the contract. An added `return err` is correct or catastrophic depending on the `defer` above it and the mutex below it, and the diff shows neither.

**Say when the code is fine.** One line — "reviewed, no blocking findings, two nits below" — and stop. Manufacturing findings to look thorough trains the author to discount the review.
