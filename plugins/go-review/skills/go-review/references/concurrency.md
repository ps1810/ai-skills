# Go concurrency and race safety

Contents: [The three questions](#the-three-questions) · [What the race detector proves](#what-the-race-detector-proves) · [Goroutine leaks](#goroutine-leaks) · [Data races](#data-races) · [Channels](#channels) · [sync package](#sync-package) · [atomic](#atomic) · [Context and cancellation](#context-and-cancellation) · [errgroup](#errgroup) · [Timers](#timers) · [Deadlock shapes](#deadlock-shapes) · [Testing concurrency](#testing-concurrency)

## The three questions

For every `go` statement in the diff, answer these explicitly. Most concurrency bugs are a missing answer, not a subtle one.

1. **How does this goroutine exit?** Every path — including the error paths and the cancelled path. If the answer is "when the channel is closed," find who closes it and confirm they always do.
2. **Who waits for it?** A `WaitGroup`, an `errgroup`, a result channel read to completion, or nothing. If nothing, the process can exit or the test can pass while the goroutine is mid-write.
3. **What memory does it share, and what establishes happens-before?** Name the edge: a mutex unlock→lock, a channel send→receive, `WaitGroup.Wait` after `Done`, an atomic store→load. "The write is a single word so it's atomic" is false in Go's memory model and false on some architectures.

## What the race detector proves

`go test -race` reports races it **observed on the paths the test actually executed**. It has no static analysis component. A clean run on a test that never starts two goroutines against the same field proves nothing about that field.

So: run it, and separately state what it did not cover.

```bash
go test -race ./...
go test -race -count=10 ./pkg/...       # shake out schedule-dependent races
go test -race -cpu=1,4,8 ./pkg/...      # vary parallelism
```

Distinct from the detector, these are **runtime panics that fire without `-race`**:

- `fatal error: concurrent map writes`
- `fatal error: concurrent map read and map write`

An unsynchronized map touched by two goroutines is BLOCKING regardless of test results. It is not a race that *might* corrupt; it is a crash that *will* happen.

The detector also costs ~10x CPU and ~5-10x memory, which is why it lives in CI rather than production. Code that only races under production load and has no load test is a coverage gap worth naming.

## Goroutine leaks

The dominant shape: a goroutine blocked forever on a channel operation nobody will complete.

```go
// LEAK: if the caller returns early, nothing ever receives, and this
// goroutine holds its stack and everything it closed over, forever.
func fetch(ctx context.Context, url string) (*Result, error) {
    ch := make(chan *Result)
    go func() {
        ch <- doWork(url)     // blocks forever if no receive
    }()
    select {
    case r := <-ch:
        return r, nil
    case <-ctx.Done():
        return nil, ctx.Err() // <- leaked the goroutine
    }
}
```

Two fixes, and which one you want depends on whether the work is still worth doing:

```go
ch := make(chan *Result, 1)   // buffered: send always completes, result discarded
```

```go
go func() {                    // or make the send cancellable
    select {
    case ch <- doWork(ctx, url):
    case <-ctx.Done():
    }
}()
```

Other leak shapes to look for:

- A `for range ch` where the producer never closes `ch` on its error path.
- A worker pool whose workers exit but whose `results` channel is never drained, blocking the last sends.
- A goroutine that selects only on a work channel with no `ctx.Done()` case.
- `time.Tick` — has no stop, leaks the underlying ticker for the life of the process. Use `time.NewTicker` and `defer t.Stop()`.

Per-request leaks are the serious kind: they look like a slow memory climb and a rising goroutine count, and they take down the process hours after deploy. Report them as BLOCKING.

Detection: `go.uber.org/goleak` in `TestMain`, or read `/debug/pprof/goroutine?debug=2` under load and look for a stack with an implausible count.

## Data races

A race is two goroutines accessing the same memory, at least one writing, with no synchronization between them. The consequences in Go are not limited to a stale read — a torn write to an interface value or a slice header can produce a pointer that was never valid, which crashes somewhere unrelated.

Places races hide:

**Copying a lock.** `func (s Server) Get()` on a struct containing a `sync.Mutex` copies the mutex, so each call locks its own copy. `go vet` catches this; take the finding seriously. Methods on lock-bearing structs need pointer receivers.

**Embedded mutex with exported fields.** `mu` guards nothing if callers can reach the fields directly.

**Guarded by comment only.** A `// guarded by mu` comment on a field that some path reads without the lock. Grep every use of the field, not just the ones near the lock.

**Lazy initialization.** Double-checked locking is not safe in Go's memory model. Use `sync.Once`.

**Slice and map aliasing.** Returning an internal slice hands the caller a writable view of your state.

**Loop variable capture, on Go 1.21 and earlier.** `for _, v := range items { go func() { use(v) }() }` — all goroutines see the same variable. Go 1.22 changed this to per-iteration scoping, but only when the module's `go` directive is 1.22 or later. Check `go.mod` before deciding whether this is a bug or fine.

**`WaitGroup.Add` inside the goroutine.** `Add` must happen before the `go` statement, or `Wait` can return before the goroutine registers.

## Channels

**Only the sender closes, and only one sender closes.** Closing from the receive side, or from two senders, panics. With multiple senders, close a separate `done` channel or use a `WaitGroup` and have one owner close after `Wait`.

**Closing twice panics. Sending on a closed channel panics. Receiving from a closed channel returns the zero value immediately** — which is why `v, ok := <-ch` exists, and why a `for` loop reading without `ok` on a closed channel spins.

**A `nil` channel blocks forever.** This is occasionally useful (set a case's channel to `nil` to disable it in a `select`) and more often an uninitialized field.

**Unbuffered means the send blocks until a receive begins** — it is a synchronization point, not a queue. Buffered capacity 1 is a handoff slot; large buffers are a queue that hides backpressure, and hiding backpressure is usually the bug rather than the fix.

**`select` with a `default` in a bare loop is a busy-wait** burning a core. Almost always wants a blocking `select` or a ticker.

**Prefer a mutex when you are protecting state; prefer a channel when you are transferring ownership.** "Share memory by communicating" is about ownership transfer. A `chan` used as a mutex around a counter is slower and harder to read than `sync.Mutex`.

## sync package

**`sync.Mutex`** — zero value ready. Guard the smallest region that keeps the invariant. `defer mu.Unlock()` immediately after `Lock()` unless there is a measured reason not to; the reason is usually that you are holding the lock across an I/O call, which is itself the finding.

**`sync.RWMutex`** — only pays off with genuinely read-heavy contention. It is slower than `Mutex` under low contention. Its scheduling surprises people in the other direction from what they expect: a pending `Lock` blocks all *new* readers, so writers do not starve, but a goroutine that holds `RLock` and calls something that takes `RLock` again deadlocks the moment a writer is waiting. Recursive read locking is a finding. Not a default.

**Never hold a lock while calling into code you do not control** — a callback, an interface method, an RPC. That is how lock-ordering deadlocks and multi-second stalls appear.

**`sync.Once`** — the correct answer for lazy init. `once.Do` establishes happens-before for everything the function wrote.

**`sync.WaitGroup`** — `Add` before `go`, `defer wg.Done()` as the goroutine's first line, `Wait` exactly once. Do not copy a `WaitGroup`; pass `*sync.WaitGroup`. Reusing one across rounds without full drainage is a race.

**`sync.Map`** — narrow: keys written once and read many times, or disjoint key sets per goroutine. For anything else a `map` plus `RWMutex` is faster and type-safe. Using `sync.Map` as a general concurrent map is usually a performance regression plus a loss of types.

**`sync.Pool`** — for reducing allocation of large short-lived buffers. Entries can vanish at any GC. Always reset an object on `Get`; a pool that hands back dirty buffers leaks data between requests, which is a security finding.

## atomic

On Go 1.19+ prefer the typed forms — `atomic.Int64`, `atomic.Bool`, `atomic.Pointer[T]` — over the free functions. They cannot be accidentally read non-atomically, and they are correctly aligned by construction.

`atomic.Value` requires every stored value to be the same concrete type; storing two types panics.

Atomics do establish happens-before: since Go 1.19 the memory model specifies that atomic operations are sequentially consistent, so a load that observes `atomic.Bool.Store(true)` also sees every write that preceded the store. The problem is not visibility, it is that the correctness now depends on every reader checking the flag before touching the fields and every writer finishing the fields before setting it — an invariant the compiler cannot check and the next edit will not know about. If you are reasoning about ordering across several fields, a mutex is the maintainable choice. Atomics are for counters, flags, and `atomic.Pointer` swaps of immutable values.

64-bit atomic ops on 32-bit platforms need 8-byte alignment; the typed structs handle it, raw `int64` struct fields do not.

## Context and cancellation

`ctx` first, always. `defer cancel()` immediately after `WithCancel`/`WithTimeout`/`WithDeadline`, unconditionally — skipping it leaks the timer and the parent's reference.

Passing `context.Background()` down inside a request handler severs cancellation for everything below. It is occasionally deliberate (fire-and-forget work that must outlive the request) and when it is, it needs its own timeout and its own lifecycle, not `Background()` alone.

`ctx.Err()` distinguishes `context.Canceled` from `context.DeadlineExceeded`. Mapping both to a 500 loses the difference between "client hung up" and "we were too slow," which matters a lot on a dashboard.

Cancellation is cooperative. A tight CPU loop with no `ctx.Done()` check does not stop.

## errgroup

`golang.org/x/sync/errgroup` is the right default for "run N things, stop on first error, collect it":

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(8)                       // bounded concurrency
for _, item := range items {
    item := item                    // needed on go < 1.22
    g.Go(func() error {
        return process(ctx, item)
    })
}
if err := g.Wait(); err != nil {
    return err
}
```

Use the `ctx` that `WithContext` returned, not the outer one — that is the whole mechanism. Unbounded `g.Go` in a loop over a large slice is a fan-out that will exhaust file descriptors or hammer a downstream; `SetLimit` is not optional at scale.

## Timers

`time.After` in a `select` inside a loop allocates a timer per iteration. On Go 1.22 and earlier that timer is not collected until it fires, so a long-lived loop with a long duration is a real leak. On Go 1.23+ (per the module's `go` directive) unreferenced timers are collectable immediately and this is only an allocation cost, not a leak. Check `go.mod` before filing it. Either way, one `time.NewTimer` with `Reset`, or a `Ticker` with `defer Stop()`, is the cleaner shape.

`time.Tick` has no stop at all. Never in library or long-lived code.

## Deadlock shapes

- **Lock ordering.** Two paths take A then B and B then A. Fix by imposing a global order and documenting it.
- **Re-entrant locking.** Go's `Mutex` is not reentrant; a method that locks and calls another method that locks self-deadlocks. Split into an exported locking wrapper and an unexported `mustHoldLock` implementation.
- **Full channel, no reader.** Producer blocks; consumer is waiting on the producer's result.
- **`WaitGroup.Wait` inside the work.** Waiting for a group from a goroutine that is itself in the group.
- **All goroutines asleep.** Runtime prints `all goroutines are asleep - deadlock!` only when *every* goroutine is blocked. A partial deadlock on one request path is silent — it looks like a hung request, which is why timeouts on every blocking operation matter.

## Testing concurrency

- `-race` in CI on the default test target, not a separate optional job.
- `-count=N` and `-cpu=1,4,8` for schedule-dependent bugs.
- `go.uber.org/goleak` in `TestMain` to catch leaks as test failures.
- Write at least one test that actually runs the concurrent path concurrently — N goroutines through the same object with a `WaitGroup`. Single-goroutine tests over concurrent code are the most common false sense of safety in Go.
- `t.Parallel()` combined with shared package-level fixtures is itself a race. Check that parallel tests share nothing mutable.
