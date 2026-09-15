# Design patterns in TypeScript

TypeScript has first-class functions, closures, structural interfaces, discriminated unions, and classes. That combination makes some GoF patterns disappear into the language, some become one-liners, and a few become actively harmful. Judge a pattern by whether it removes a real repetition or a real coupling — not by whether it has a name.

Contents: [Factory function and options object](#factory-function-and-options-object) · [Result type](#result-type) · [Discriminated-union state machine](#discriminated-union-state-machine) · [Middleware / decorator](#middleware-decorator) · [Concurrency patterns](#concurrency-patterns) · [Strategy as a function](#strategy-as-a-function) · [Repository](#repository) · [Typed events](#typed-events) · [Patterns that dissolve](#patterns-that-dissolve-into-the-language) · [Patterns to avoid](#patterns-to-avoid-in-typescript) · [Review questions](#review-questions)

## Factory function and options object

The default shape for anything with dependencies or configuration:

```ts
export type ServerOptions = {
  port: number;
  timeoutMs?: number;
  logger?: Logger;
};

export function createServer(deps: { handler: Handler }, opts: ServerOptions) {
  const timeoutMs = opts.timeoutMs ?? 30_000;
  const logger = opts.logger ?? defaultLogger;
  return {
    listen: () => { /* ... */ },
    close: () => { /* ... */ },
  };
}
```

Required arguments are non-optional fields so the compiler enforces them; optional ones get real defaults at the top of the function with `??`. Adding an option later breaks nobody. Dependencies and configuration are separate parameters because they change for different reasons.

Do not use a builder (`.withPort().withTimeout().build()`) for this; the options object is the builder, and the compiler checks it. Do not use positional parameters once there are three or any is a boolean.

## Result type

For expected failures the caller must handle:

```ts
export type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

export function parseMoney(s: string): Result<Money, "empty" | "invalid" | "negative"> { /* ... */ }

const r = parseMoney(input);
if (!r.ok) return respond400(r.error);
use(r.value);
```

A hand-rolled version is a few lines; `neverthrow` adds combinators if the codebase wants them. The error type is a union of the failures the caller can act on, not `Error`.

Use it at module boundaries where the failure is part of the contract. Do not wrap every internal function; unexpected failures (a bug, a lost connection) should throw and be caught once at the top. A codebase where everything returns `Result` has reinvented checked exceptions with worse ergonomics.

## Discriminated-union state machine

For anything with states and transitions:

```ts
type Connection =
  | { state: "idle" }
  | { state: "connecting"; attempt: number }
  | { state: "open"; socket: Socket }
  | { state: "closed"; reason: string };

function transition(c: Connection, ev: Event): Connection {
  switch (c.state) {
    case "idle": /* ... */
    case "connecting": /* ... */
    case "open": /* ... */
    case "closed": /* ... */
    default: { const _: never = c; throw new Error("unreachable"); }
  }
}
```

Illegal states are unrepresentable (no `socket` while `idle`), and adding a state is a compile error at every switch. This replaces the State pattern, most uses of boolean flags, and most uses of `null` to mean "not yet."

Review finding: a type with `status: string` and five optional fields whose validity depends on the status. That is this pattern, not yet applied.

## Middleware / decorator

The strongest composition pattern in TypeScript, and the shape of every web framework:

```ts
type Handler = (req: Request) => Promise<Response>;
type Middleware = (next: Handler) => Handler;

export const compose = (...ms: Middleware[]): Middleware =>
  (next) => ms.reduceRight((acc, m) => m(acc), next);
```

The same shape wraps `fetch` (retry, tracing, auth headers), queue consumers (dedup, metrics), and store interfaces (caching, logging). Because interfaces are structural, a wrapper that returns the same shape is a drop-in.

Watch the ordering: recovery outermost, then tracing, then logging, then auth, then rate limiting, then the handler. A recovery middleware inside the logger cannot log what it caught; a rate limiter outside auth lets unauthenticated traffic consume the budget.

## Concurrency patterns

**Bounded map** — the answer to `Promise.all(items.map(f))` over a large list:

```ts
import pLimit from "p-limit";
const limit = pLimit(8);
const results = await Promise.all(items.map((item) => limit(() => process(item, { signal }))));
```

Or a hand-written semaphore of ten lines if a dependency is unwelcome. The bound is a requirement; write it down.

**In-flight dedup (singleflight)** — collapse concurrent identical requests into one:

```ts
const inflight = new Map<string, Promise<Value>>();
function get(key: string): Promise<Value> {
  let p = inflight.get(key);
  if (!p) {
    p = load(key).finally(() => inflight.delete(key));
    inflight.set(key, p);
  }
  return p;
}
```

The check-and-set is synchronous, which is what makes it race-free. This is the correct answer to a cache stampede and the thing most hand-rolled caches are missing.

**Queue with a worker loop** — for work that must not overlap:

```ts
async function worker(queue: AsyncIterable<Job>, signal: AbortSignal) {
  for await (const job of queue) {
    if (signal.aborted) return;
    await handle(job);
  }
}
```

**Async lock** — when an invariant in process memory must hold across an `await` and there is no store to lean on. `async-mutex`, or a promise-chain lock. Prefer pushing atomicity to the store when there is one.

Review finding: unbounded promise creation per input item. `items.map(async ...)` over a list of unknown size is a resource exhaustion bug, not a pattern.

## Strategy as a function

When the strategy has one operation, use a function type. An interface adds a named type and an implementing class to express what `(order: Order) => Money` already says.

```ts
type Pricer = (order: Order) => Money;
function total(order: Order, price: Pricer): Money { return price(order); }
```

Use an object of functions once the strategy has two operations; use a class once it has state with a lifecycle.

## Repository

Useful when it earns its keep: when the domain type differs from the storage row, or when the consumer genuinely needs a test double.

```ts
export type UserRepo = {
  find(id: UserId, opts: { signal: AbortSignal }): Promise<User | undefined>;
  save(u: User, opts: { signal: AbortSignal }): Promise<void>;
};
```

Two failure modes:

**A leaky repository.** Methods returning the ORM's entity class, accepting a query-builder object, or taking a raw SQL fragment. The abstraction has published its implementation.

**A repository that is a thin rename of the ORM.** If every method is one query with no mapping and no domain type, and the only implementation is Prisma, you have added indirection and no capability. Passing the typed client directly is a legitimate choice — and with a typed query builder (Prisma, Drizzle, Kysely) it is often the right one, because the types already do the job the repository was for.

Transactions are where this pattern breaks. Two repository calls in one transaction need a deliberate answer — a `withTx(fn)` that passes a transaction-bound repo, or the ORM's interactive transaction — decided up front.

## Typed events

`EventEmitter` is untyped by default: `emit("usre-created", ...)` compiles. Type it:

```ts
type Events = { "user-created": [user: User]; "user-deleted": [id: UserId] };
const bus = new EventEmitter<Events>();          // Node 20+ supports the generic
```

Or a small typed wrapper with a `Map<keyof Events, Set<Listener>>`. For anything crossing a process boundary, this is not enough — see the backend-services architect skill on event contracts.

## Patterns that dissolve into the language

- **Iterator** — generators and `for...of`; async generators and `for await`.
- **Adapter** — an object literal with the right shape; structural typing does the rest.
- **Template method** — a function taking callbacks.
- **Observer** — a `Set` of callbacks, or `EventTarget`/`EventEmitter`.
- **Command** — a closure.
- **Builder** — an options object with defaults.
- **Singleton** — a module. Every `import` gets the same instance; do not write a class with a static `getInstance()`.
- **Null object** — `undefined` plus `??`, or a discriminated union variant.
- **Visitor** — a discriminated union and an exhaustive switch.

Naming these in TypeScript adds vocabulary without adding structure. `UserRepositoryFactoryImpl` is a review finding.

## Patterns to avoid in TypeScript

**Class hierarchies for data.** `class Animal`, `class Dog extends Animal`. Types plus functions; a discriminated union if the variants matter.

**Decorators for business logic.** They are experimental-adjacent, need `emitDecoratorMetadata`, run at class-definition time, and hide control flow. Frameworks that mandate them (NestJS, TypeORM, Angular) are a deliberate trade; do not introduce them elsewhere.

**Module-level mutable state.** `let cache = {}` at the top of a module is a singleton with no lifecycle, a test-isolation problem, and a memory leak. Create it in a factory and pass it.

**Import-time work.** A module that connects, reads env, or registers handlers when imported makes import order matter and makes the module untestable.

**Abstract factory / factory of factories.** Two levels of indirection to choose a constructor. A `switch` in `main` is clearer and greppable.

**`any`-typed event buses and message maps.** `emit(name: string, payload: any)` is the type system switched off at the seam where it matters most.

**Exceptions for control flow.** `throw` to exit a loop, or `try/catch` around a lookup to mean "not found." A `Result` or `undefined`.

**Mocking modules (`vi.mock("../db")`) as the primary test strategy.** It works, and it means the dependency should have been a parameter. See `solid.md`.

## Review questions

For any pattern in a diff, ask in this order:

1. **What repetition or coupling does this remove?** If the answer is hypothetical, the abstraction is speculative.
2. **What is the simplest thing that works?** A function, an object literal, a union.
3. **How many implementations exist today?** One, with no test double, means do not extract the type yet.
4. **Does it make the likely next change cheaper?** Not any change — the one actually coming.
5. **Can a new reader follow the call path without a diagram?** Indirection has a real, permanent readability cost that abstraction discussions tend to price at zero.
