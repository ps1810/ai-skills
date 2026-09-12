# Idiomatic Go

Contents: [Errors](#errors) · [Context](#context) · [Interfaces and API shape](#interfaces-and-api-shape) · [Naming](#naming) · [Zero values and construction](#zero-values-and-construction) · [Slices and maps](#slices-and-maps) · [Control flow](#control-flow) · [Resource lifecycle](#resource-lifecycle) · [Tests](#tests) · [Generics](#generics)

## Errors

**Wrap with `%w` when the caller might want the cause; use `%v` when you are deliberately sealing it off.** `%w` makes the error part of your API surface — callers can now `errors.Is` against what you wrapped, and you have promised not to change it. That is sometimes exactly right and sometimes an accidental commitment.

```go
// caller can inspect the cause
return fmt.Errorf("load config %q: %w", path, err)

// deliberately opaque: the sql driver error is not part of this API
return fmt.Errorf("query user: %v", err)
```

**Compare with `errors.Is` and `errors.As`, never `==` or a type assertion.** A single wrap breaks `err == ErrNotFound` and breaks `err.(*MyError)`. This is one of the most common real bugs in Go code that looks correct.

**Add information when wrapping, or do not wrap.** `fmt.Errorf("failed: %w", err)` is strictly worse than returning `err` — same information, longer string, extra allocation. Wrap with the operation and the identifying argument: what you were doing and to what.

**Do not log and return.** The caller logs, or you log — one of you. Both produces the same failure reported at four stack levels, which makes on-call triage harder, not easier.

**`errors.Join` for genuinely independent failures.** Validating ten fields, closing five resources. Not for a chain of causes.

**Sentinel or type?** Sentinel (`var ErrNotFound = errors.New(...)`) when the caller only needs to know *which* failure. Custom type when the caller needs data out of it (a field name, a retry-after duration, an HTTP status).

**Every returned error is checked or explicitly discarded.** `_ = f.Close()` states a decision. A bare `f.Close()` on a write path is a bug — that is where the flush error surfaces, and the data silently did not land.

## Context

**First parameter, named `ctx`, type `context.Context`.** No exceptions in exported functions.

**Never store a `Context` in a struct field.** A context carries the deadline of one operation. A struct outlives the operation, so the stored context is either stale or cancelled or both. The exception people reach for — "but my worker struct needs a context" — is solved by passing it to `Run(ctx)`.

**Honour cancellation, do not just accept it.** Accepting a `ctx` and never selecting on `ctx.Done()` in a loop, or never passing it to the call that blocks, is worse than not accepting one: it advertises a cancellation contract it does not keep.

**`context.WithCancel` and friends: `defer cancel()` immediately, unconditionally.** Not doing so leaks the parent's child-list entry until the parent is cancelled, and for `WithTimeout`/`WithDeadline` also the underlying timer. `go vet` flags the missing `cancel`; treat it as a finding.

**Values are for request-scoped metadata that crosses API boundaries** — trace IDs, auth principals. Not for optional arguments and not for dependency injection. A function that reads its database handle out of a context has an untyped, uncheckable signature.

## Interfaces and API shape

**Accept interfaces, return structs.** Returning a concrete type lets callers use new methods you add later without you breaking their code; returning an interface freezes the surface at what you guessed today. Accepting an interface lets callers substitute.

**Interfaces belong to the consumer.** The package that *calls* `Get(ctx, id)` declares the one-method interface it needs. The package that implements storage just returns its concrete `*Postgres`. This is the inverse of the Java instinct and it is what makes Go packages composable without a shared abstraction layer.

**Small interfaces.** One to three methods. An eight-method interface is a struct wearing a costume: nobody can implement it for a test without a mock generator, and no second implementation ever appears.

**One implementation plus a mock is a smell, not a crime.** It is often the right call for testability. But check whether the "mock" is really a stub for a call you could have passed as a function value.

**Do not export what you do not have to.** Exported identifiers are a support obligation. This includes struct fields — an exported field is a mutable, unvalidated entry point.

## Naming

**No stutter.** `http.Server`, not `http.HTTPServer`. `user.Store`, not `user.UserStore`. The package qualifies the name at every use site.

**Short receivers, short scopes.** `func (s *Server)`, not `func (server *Server)`. Loop variables `i`, `k`, `v`. Longer names as scope widens — a package-level variable earns a descriptive name.

**MixedCaps, not underscores**, with the test-function exceptions the toolchain itself defines: `TestType_Method` and `TestFunc_case` are conventional, and `Example_suffix`, `ExampleType_Method` *require* the underscore for `go doc` to associate them. Underscores anywhere else are a finding.

**Getters drop the `Get`.** `u.Name()`, not `u.GetName()`. Setters keep `Set`.

**Single-method interfaces take the `-er` suffix.** `Reader`, `Notifier`, `Validator`.

**Initialisms keep their case.** `ID`, `URL`, `HTTP`, `API` — `userID`, `parseURL`, not `userId`, `parseUrl`.

## Zero values and construction

**Make the zero value useful where you can.** `sync.Mutex`, `bytes.Buffer`, and `sync.WaitGroup` all work at zero. It removes a whole class of "forgot to call the constructor" bugs.

**Use a constructor when the zero value would be broken.** Then return the concrete type, validate inputs, and make the invalid state unrepresentable.

**Functional options for optional configuration**, not a config struct with fifteen fields where four combinations are illegal. See `patterns.md`.

**`init()` is nearly always avoidable.** It runs at import time in an order you do not fully control, it cannot return an error, and it makes tests order-dependent. Prefer explicit setup called from `main`.

**Avoid package-level mutable state.** It is shared memory with no synchronization contract and it makes parallel tests flaky.

## Slices and maps

**`nil` slices are usable — append, range, and `len` all work.** Do not write `s := []T{}` to "initialize" one. Do preallocate with `make([]T, 0, n)` when `n` is known and the slice is large.

**`nil` maps read fine and panic on write.** A struct field of map type that nothing initialized is a panic waiting for the first write. Check every map field's construction path.

**Returning a slice returns a view of the backing array.** The caller can mutate your internal state through it. If the slice is internal, copy it out or document the aliasing. `append` on a shared slice can either mutate the shared array or reallocate depending on capacity, which makes the bug intermittent.

**`s[i:j]` retains the whole backing array.** Slicing three bytes out of a 10 MB buffer and holding it keeps 10 MB alive. Copy when you keep a small piece of something large.

**Iteration order over a map is randomized deliberately.** Any code that depends on it is broken; sort the keys.

## Control flow

**Early return over nested `else`.** Handle the error, return, and let the happy path run unindented down the left margin.

**No naked returns outside a few short lines.** Named results are useful for documentation and for `defer`-based error modification; naked `return` at the bottom of a forty-line function forces the reader to scan for assignments.

**`switch` over an `if`/`else if` chain of three or more.** A bare `switch {` with boolean cases reads better than the chain.

**Do not shadow `err` in a way that drops it.** `if err := f(); err != nil` is fine and scoped. `x, err := f()` inside a block where the outer `err` was going to be returned is where errors vanish.

## Resource lifecycle

**`defer` runs at function exit, not block exit.** `defer rows.Close()` inside a loop accumulates every open cursor until the function returns. Extract the loop body into a function, or close explicitly.

**Deferred `Close` on a writer swallows the error.** Use a named return and `defer func() { err = errors.Join(err, f.Close()) }()`, or close explicitly and check.

**`http.Response.Body` must be closed on every non-error return, including the paths where you do not read it.** Not closing it leaks the connection out of the pool.

**Arguments to `defer` evaluate immediately; the call runs later.** `defer log.Printf("took %v", time.Since(start))` captures nothing wrong, but `defer f(x)` captures `x` as of the `defer` statement.

## Tests

**Table-driven, with `name` on every case**, so failures identify themselves.

**`t.Run` subtests, `t.Parallel()` where safe.** On Go 1.22+ the loop variable is per-iteration, so the old `tc := tc` copy is no longer needed — but check `go.mod` before removing it, because on an older `go` directive the old semantics still apply and removing the copy introduces a real bug.

**`t.Cleanup` over `defer` in helpers**, and `t.Helper()` in helpers so failures point at the caller.

**Assert on behaviour, not on internal state.** A test that reaches into unexported fields breaks on every refactor and does not catch the bugs users hit.

**Ask whether the test would catch the bug class this code is prone to.** Concurrent code with only single-goroutine tests is untested where it matters. This is a review finding, not a nit.

## Generics

**Use them when you would otherwise write the same function for three types, or return `any` and make the caller assert.** `Map`, `Filter`, `Keys`, constraint-bounded numeric helpers.

**Do not use them where an interface expresses it.** If the function only ever calls methods on the value, an interface parameter is simpler, compiles faster, and reads better.

**`any` in a signature is a review finding.** Ask what it hides. Often it is a generic parameter or a small interface that the author did not want to name.
