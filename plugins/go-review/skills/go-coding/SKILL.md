---
name: go-coding
description: Conventions for writing Go — errors, context, interfaces, concurrency, package layout, tests — applied while writing or modifying any .go file. Use this whenever you are about to create or edit Go code, add a package, write a test, or wire a dependency in a Go project. For reviewing Go that already exists, use go-review instead.
---

# Writing Go

This skill loads when Go is being written. It carries the same conventions the reviewer will hold you to, so that the code is right the first time and the Stop hook has nothing to say. The detailed reasoning lives in the shared reference files; this file is the short form.

## Before writing

1. **Read `go.mod`.** The `go` directive decides real semantics: loop-variable capture (1.22), range-over-func (1.23), `time.After` collection (1.23), `testing/synctest` (1.25). Do not write code for a version the module does not declare.
2. **Read the neighbouring package.** Match its constructor shape, error style, logger, and test layout. A package with two conventions is worse than a package with one imperfect convention.
3. **Load the reference for the ground you are on.** Paths are relative to the `go-review` skill directory next to this one:

| Writing | Read first |
|---|---|
| Anything | `../go-review/references/idiomatic.md` |
| A goroutine, channel, `sync`, `atomic`, or `select` | `../go-review/references/concurrency.md` |
| A new package, interface, or constructor | `../go-review/references/solid.md` |
| Something that feels like it wants a pattern | `../go-review/references/patterns.md` |
| Tests | `../go-review/references/testing.md` |

## The conventions in ten lines

1. `ctx context.Context` first, propagated, honoured with `ctx.Done()` in anything that loops or blocks. Never stored in a struct.
2. Wrap errors with the operation and the identifying argument: `fmt.Errorf("load config %q: %w", path, err)`. Compare with `errors.Is`/`errors.As`. Check or explicitly discard every error. Log or return, not both.
3. The consumer declares the interface it needs, one to three methods. The producer returns a concrete type. `main` wires.
4. Zero values useful where possible; a constructor when the zero value would be broken; functional options only for genuinely optional configuration.
5. Every `go` statement has an answer to: how does it exit, who waits for it, what does it share and what synchronises that. Bound fan-out with `errgroup.SetLimit` or a semaphore.
6. `defer cancel()`, `defer mu.Unlock()`, `defer resp.Body.Close()` immediately after acquisition. Never `defer` inside a loop body that iterates many times.
7. Guard maps written from more than one goroutine. An unsynchronised concurrent map write is a fatal runtime error, not a race report.
8. No package-level mutable state, no `init()` unless a driver registration forces it, no `panic` for expected failures.
9. Package names are short, lower-case, and describe what the package is about. No `utils`, `common`, `helpers`, `models`.
10. Tests are table-driven with named cases, exercise the failure mode the code is prone to, and run concurrently when the code is concurrent.

## Do not add

- An interface with one implementation and no test that needs it.
- A generic function where an interface parameter says the same thing.
- A `Repository` that is a thin rename of `*sql.DB`.
- A config struct with fields whose combinations are illegal.
- Abstraction against a change that is not actually coming.

## Before declaring done

```bash
gofmt -l .
go vet ./...
go build ./...
go test -race ./...
```

All four clean, or say which is not and why. Tests are written alongside the code, not promised for later. A change that adds a goroutine adds a test that runs it concurrently.
