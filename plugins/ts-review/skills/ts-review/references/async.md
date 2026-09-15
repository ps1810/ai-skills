# Async TypeScript and concurrency

Contents: [The three questions](#the-three-questions) · [Floating promises](#floating-promises) · [Missing and misplaced await](#missing-and-misplaced-await) · [Races across await](#races-across-await) · [Fan-out and bounds](#fan-out-and-bounds) · [Cancellation and timeouts](#cancellation-and-timeouts) · [Error propagation](#error-propagation) · [Event loop](#the-event-loop) · [Event emitters and streams](#event-emitters-and-streams) · [Timers](#timers) · [Shutdown](#shutdown) · [Testing async](#testing-async)

JavaScript has one thread and no data races in the Go sense. It still has every other concurrency bug: lost work, unbounded fan-out, check-then-act races, missing cancellation, and hung processes. They are just harder to see because nothing runs "at the same time" — it runs *in between*.

## The three questions

For every promise created in the diff, answer these explicitly. Most async bugs are a missing answer, not a subtle one.

1. **Who awaits it?** Awaited, returned, passed to `Promise.all`, or nothing. If nothing, its rejection is unhandled and its completion is unobservable.
2. **What bounds it?** If it is created in a loop or a `map`, how many can be in flight at once? "As many as there are items" is the bug.
3. **What cancels it, and what shared state does the code touch on either side of the `await`?** Name the signal that stops it and the timeout that bounds it. Name the invariant that must hold across the yield point.

## Floating promises

A promise that is neither awaited, returned, nor handled:

```ts
// LEAK + CRASH: nobody awaits, nobody catches. On rejection, Node exits.
async function handle(req: Request) {
  audit.log(req);            // returns a Promise — dropped
  return ok();
}
```

Consequences: the caller returns before the work is done (a test passes before the write lands; an HTTP response is sent before the audit is written); and when it rejects, Node 15+ raises an unhandled rejection and terminates the process by default.

Fixes, in order of preference:

- `await` it.
- `return` it (the caller decides).
- Deliberately detach with `void` **and** a handler: `void audit.log(req).catch((e) => logger.error("audit failed", e))`. Fire-and-forget without a `.catch` is still a crash.

**`@typescript-eslint/no-floating-promises` and `no-misused-promises` are the mechanical defence.** If they are not enabled, say so; it means every floating promise must be found by reading.

`no-misused-promises` catches the second common shape: an `async` function passed where a `void`-returning callback is expected — `array.forEach(async ...)`, `emitter.on("x", async ...)`, an Express handler with no error middleware. The promise is dropped by the callee.

## Missing and misplaced await

- **`forEach` with an async callback runs nothing in order and awaits nothing.** `for...of` with `await`, or `Promise.all(items.map(...))` when parallel is intended.
- **`return promise` vs `return await promise` inside `try`.** Without `await`, the rejection happens after the `try` block is exited and the `catch` never runs. Inside `try`, always `return await`. Outside `try`, `return` is fine and skips a microtask.
- **Sequential awaits in a loop when the work is independent** is a latency bug: N round trips serialised. **Parallel `Promise.all` when the work is dependent or rate-limited** is a correctness or capacity bug. Decide which and say why.
- **`await` on a non-promise** is allowed and harmless but usually means the author thought a sync function was async. `@typescript-eslint/await-thenable`.
- **Async constructors do not exist.** A class that needs async setup gets a static `create()` that returns the instance after awaiting.

## Races across await

Every `await` is a yield point. Between the `await` and the next line, every other pending request, timer, and callback gets to run. Any invariant that spans the yield is unprotected:

```ts
// RACE: two concurrent requests both see "no entry", both insert.
async function getOrCreate(id: string) {
  const existing = await cache.get(id);
  if (existing) return existing;
  const created = await build(id);
  await cache.set(id, created);          // second writer overwrites first
  return created;
}
```

This is the same bug as an unguarded map write in Go; it is just deterministic enough to pass every test.

Fixes:

- **Make the check-and-act one synchronous step.** An in-memory `Map` of in-flight promises (the "singleflight" shape): check the map and insert the promise synchronously, then await it.
- **Push the atomicity to the store.** `INSERT ... ON CONFLICT`, Redis `SET NX`, a unique constraint. The database is the mutex.
- **A per-key async lock** (`async-mutex`, or a tiny promise-chain lock) when the invariant lives in process memory and there is no store to lean on.

Other shapes: a module-level counter incremented after an `await`; a "loading" flag set true, awaited, set false, where a second call runs during the await; an array pushed to across an await and read after.

## Fan-out and bounds

`Promise.all(items.map(fetchOne))` opens `items.length` connections at once. Over a list of a thousand it exhausts sockets, hits rate limits, or knocks over the downstream. This is a review finding whenever the list size is not known and small.

- **`p-limit`**, **`p-map` with `concurrency`**, or a hand-written semaphore for bounded parallelism.
- **`Promise.allSettled`** when one failure should not abort the batch and you will inspect every result. `Promise.all` rejects on the first rejection but does **not** cancel the others; they keep running and their results are dropped.
- **`Promise.race` for timeouts is a leak** unless the loser is cancelled: the slow promise keeps running and holds its resources. Use `AbortSignal.timeout` instead.
- **`Promise.any`** for "first success"; rare.

State the bound in the design, not just the code: "at most 8 concurrent calls to the payment API" is a requirement, and the limiter is where it lives.

## Cancellation and timeouts

There is no way to cancel a promise from outside. Cancellation is cooperative through `AbortSignal`:

```ts
async function fetchUser(id: string, { signal }: { signal: AbortSignal }) {
  const res = await fetch(`/users/${id}`, { signal });
  ...
}

// caller
const res = await fetchUser(id, { signal: AbortSignal.timeout(5_000) });
```

- **Every outbound call accepts a `signal`** and passes it through. A function that takes a `signal` and does not forward it to the thing that blocks advertises a cancellation contract it does not keep.
- **Every outbound call has a timeout.** `fetch` has none by default. Database clients usually have one; check it is set and shorter than the caller's.
- **`AbortSignal.any([a, b])`** (Node 20+) combines a request's signal with a timeout.
- **Long loops check `signal.aborted`** or `signal.throwIfAborted()` each iteration, or they do not stop.
- **Aborting throws an `AbortError`**; distinguish it from real failures in the catch so a cancelled request is not logged as a 500.

## Error propagation

- **An `async` function's throw is a rejection**; a rejection nobody awaits is unhandled. See floating promises.
- **`.then(onOk, onErr)` vs `.then(onOk).catch(onErr)`**: the second also catches errors thrown inside `onOk`; the first does not. Usually you want the second, or just `async`/`await` with `try`/`catch`.
- **`catch` inside a loop that continues** turns a failure into silent partial work. Decide: abort the batch, collect failures and report them, or retry. "Log and continue" needs a stated reason.
- **Retries only on idempotent operations**, with a cap, exponential backoff, and jitter. A retry on a non-idempotent write is a duplicate charge. Wrap the retry around the operation, not around the whole handler.
- **`Promise.all` rejects with the first error only.** The others are lost. If you need all failures, `allSettled`.

## The event loop

One thread runs all callbacks. Anything synchronous and slow stalls every in-flight request for its duration.

- **Synchronous I/O on a request path**: `readFileSync`, `execSync`, `pbkdf2Sync`, `zlib.gzipSync`. BLOCKING in a server.
- **CPU-heavy work**: image processing, large JSON, regex on large input, big sorts. `worker_threads` or a separate service. `setImmediate` chunking is a partial fix.
- **Catastrophic regex backtracking** on user input is a DoS with one request.
- **Microtasks starve macrotasks.** A loop that resolves promises forever (`while (true) await Promise.resolve()`) never lets I/O run. `await setImmediate` (from `timers/promises`) to yield to I/O.
- **Detection**: `perf_hooks.monitorEventLoopDelay`, or the `--cpu-prof` flag. A p99 that jumps with no corresponding downstream latency is usually the loop.

## Event emitters and streams

- **An `error` event with no listener throws.** Every `EventEmitter` and stream that can emit `error` has a listener, or the process crashes.
- **Listeners leak.** `on()` inside a request handler with no matching `off()` accumulates one listener per request. Node warns at 11 by default (`MaxListenersExceededWarning`); treat the warning as a bug report. `once()` where one-shot; `AbortSignal` in `addEventListener` options to clean up.
- **Streams need backpressure.** `readable.pipe(writable)` handles it but does not propagate errors or close on failure; `stream/promises.pipeline` does both. Manual `.on("data")` with an async handler ignores backpressure and buffers unboundedly.
- **`for await (const chunk of stream)`** is the readable form and respects backpressure; make sure the stream is destroyed on early `break` (`pipeline` and `for await` do this; manual consumption does not).
- **Async iterators from a database cursor** hold the connection until fully consumed or returned; an early `return` without closing leaks the connection.

## Timers

- **`setInterval` overlaps.** If the callback takes longer than the interval, the next one starts anyway. Use a recursive `setTimeout` scheduled at the end of the work, or an `async` loop with `await setTimeout(ms)` from `timers/promises`.
- **Timers keep the process alive.** `.unref()` for background timers that should not block exit. A test runner that never exits has an un-unref'd interval.
- **Timers are not cancelled by an abort.** Clear them in the `signal`'s `abort` handler, or use `timers/promises.setTimeout(ms, value, { signal })`.
- **`setTimeout` with a large delay overflows** at 2^31 - 1 ms (about 24.8 days) and fires immediately.

## Shutdown

A long-running Node process that does not handle `SIGTERM` is killed mid-request on every deploy.

1. Stop accepting: `server.close()` (stops new connections; existing keep-alive connections need `server.closeIdleConnections()` on Node 18+).
2. Stop consuming: pause queue consumers, clear intervals.
3. Drain: await in-flight work with a deadline (`Promise.race` against a timer is acceptable here because you are about to exit anyway).
4. Close resources: database pools, Redis clients, file handles.
5. `process.exit(0)`; on deadline, `process.exit(1)` and let the orchestrator restart.

Register once, guard against double invocation, and log each phase; shutdown bugs are only ever debugged from logs.

## Testing async

- **Every async test awaits or returns its promise.** A test that starts async work and returns synchronously passes before the assertion runs — vitest and jest catch some cases (`no-floating-promises` in test files catches more).
- **Fake timers** (`vi.useFakeTimers()`) for anything with `setTimeout`/`setInterval`; then `await vi.advanceTimersByTimeAsync(ms)` — the `Async` variant flushes microtasks between ticks, the sync one does not, and the difference is the usual reason a fake-timer test hangs.
- **Test the race.** Fire two calls without awaiting the first, then await both, then assert the invariant held. If it is hard to make the race happen in a test, inject a `Promise` that resolves on command between the check and the act.
- **Test cancellation.** Pass an already-aborted signal; assert the function rejects with `AbortError` and did not do the work.
- **Test the bound.** Instrument the fake downstream to count concurrent in-flight calls; assert it never exceeds the limit.
- **Unhandled rejections fail the run.** vitest reports them; make sure the CI config does not hide them.
