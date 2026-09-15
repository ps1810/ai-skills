# Idiomatic TypeScript

Contents: [Strictness](#strictness) · [Types are not runtime](#types-are-not-runtime) · [Type honesty](#type-honesty) · [Modelling with types](#modelling-with-types) · [Errors](#errors) · [Absence](#absence-null-and-undefined) · [Modules](#modules) · [Immutability and mutation](#immutability-and-mutation) · [Functions and parameters](#functions-and-parameters) · [Naming](#naming) · [Runtime pitfalls](#runtime-pitfalls) · [Node specifics](#node-specifics)

## Strictness

TypeScript checks what `tsconfig.json` tells it to. A review that assumes `strict` is on when it is off will miss an entire class of bugs.

- **`strict: true`** is the floor. Without it, `null` and `undefined` are assignable to everything and the type system is decorative.
- **`noUncheckedIndexedAccess`** makes `arr[i]` and `obj[key]` return `T | undefined`. Without it, every index is a trusted-but-unchecked read. Turning it on in an existing codebase is a project; recommend it, do not demand it in a review of a one-line change.
- **`exactOptionalPropertyTypes`** distinguishes "absent" from "present and `undefined`". Worth it in new code.
- **`noImplicitOverride`**, **`noFallthroughCasesInSwitch`**, **`noImplicitReturns`**: cheap, on.
- **`verbatimModuleSyntax`** (or `isolatedModules`) forces `import type`, which keeps bundlers and `tsc` agreeing about what is erased.
- **`skipLibCheck: true`** is fine and normal; it does not skip checking your code.

State the strictness in the mechanical-checks section of every review.

## Types are not runtime

The single most important idea in TypeScript review: **every type annotation is erased.** `const user: User = await res.json()` does not check anything. It tells the compiler to stop asking questions.

Data crossing a boundary is `unknown` until something at runtime proves otherwise:

```ts
// Boundary: parse once, get a real type
const Env = z.object({ PORT: z.coerce.number().int().positive(), DATABASE_URL: z.string().url() });
export const env = Env.parse(process.env);

// Boundary: request body
const body = CreateOrder.parse(await req.json());   // throws a structured error on mismatch
```

Boundaries: `JSON.parse`, `res.json()`, `process.env`, request bodies and params, database rows (unless the query builder is typed end-to-end), message payloads, `localStorage`, anything from a third-party SDK typed as `any`.

**Parse, don't validate.** A validator that returns `boolean` leaves the caller with the same `unknown`. A parser returns the typed value or throws. `zod`, `valibot`, `arktype`, and `typebox` all do this; pick the one the codebase already uses.

Inside the boundary, trust the type. Re-validating at every layer is noise. The finding is a *missing* parse at the edge, or a `as User` where a parse should be.

## Type honesty

Each of these tells the compiler to stop checking. Each is a review finding unless the line above it justifies it.

- **`any`** — disables checking for everything it touches, transitively. `unknown` is the honest version: it forces a narrow before use. `any` in a signature is BLOCKING in new code.
- **`as T`** — a downcast the compiler cannot verify. Legitimate when narrowing from `unknown` after a runtime check the compiler cannot see, or for `as const`. Illegitimate when it silences an error about a real shape mismatch. `as unknown as T` is a confession.
- **`!`** (non-null assertion) — "trust me." Fine immediately after a check the compiler lost track of (a `Map.get` right after `has`). A finding when it is the only thing between the code and `undefined`.
- **`@ts-ignore`** — hides the next line's error forever, including new errors introduced later. Use `@ts-expect-error` with a reason; it fails when the error goes away, so it cannot go stale.
- **`Function`, `object`, `{}`** as types — nearly meaningless. Name the shape.
- **Return type annotations on exported functions.** Inferred return types leak implementation details and change silently when the body changes. Annotate exports; let locals infer.

## Modelling with types

**Discriminated unions for anything that has states.** Not a bag of optional fields where four combinations are illegal:

```ts
// Bad: which fields are set when?
type Job = { status: string; result?: Result; error?: string; startedAt?: Date };

// Good: the compiler knows
type Job =
  | { status: "queued" }
  | { status: "running"; startedAt: Date }
  | { status: "done"; result: Result }
  | { status: "failed"; error: string };
```

**Exhaustive `switch` with a `never` check**, so adding a variant is a compile error everywhere it is not handled:

```ts
default: {
  const _exhaustive: never = job;
  throw new Error(`unhandled status: ${(job as Job).status}`);
}
```

**No `enum`.** Numeric enums are bidirectional maps with surprising runtime shape; string enums are nominal in a structural language. Use `as const` and derive the union:

```ts
export const Status = { Queued: "queued", Running: "running" } as const;
export type Status = (typeof Status)[keyof typeof Status];
```

**Branded types for identifiers that must not be mixed up.** `UserId` and `OrderId` are both `string` structurally; a brand makes passing one where the other is expected a compile error:

```ts
type UserId = string & { readonly __brand: "UserId" };
```

**`readonly` and `ReadonlyArray` on inputs you do not mutate.** It documents the contract and the compiler enforces it. `Readonly<T>` is shallow; say so if depth matters.

**`type` vs `interface`.** `interface` for object shapes that may be extended or implemented; `type` for unions, intersections, and mapped types. Either is fine for a plain object shape. Match the codebase. This is a NIT, never higher.

**Utility types over hand-copied shapes.** `Pick`, `Omit`, `Partial`, `Required`, `Record`, `ReturnType`, `Parameters`. A second type that is 80% of the first, maintained by hand, drifts.

**Generics when the type relationship matters, not for show.** A generic parameter used once, or only ever instantiated with one type, is noise. A generic that connects an input type to an output type (`function first<T>(xs: readonly T[]): T | undefined`) is the point.

## Errors

**Throw `Error` subclasses only.** Throwing a string or an object loses the stack trace and makes `catch` untyped. Give custom errors a `name` and use `cause`:

```ts
export class NotFoundError extends Error {
  constructor(what: string, options?: { cause?: unknown }) {
    super(`${what} not found`, options);
    this.name = "NotFoundError";
  }
}

throw new ConfigError("load config", { cause: err });
```

**`catch (e: unknown)` and narrow.** With `useUnknownInCatchVariables` (part of `strict`) that is the default. `e.message` without an `instanceof Error` check is a finding: anything can be thrown.

**Never swallow.** An empty `catch {}` or a `catch (e) { console.log(e) }` on a path that matters hides the failure and continues with a broken invariant. Either handle it (and say what "handled" means), rethrow with context, or let it propagate to the top-level handler.

**Expected failures are values; unexpected failures throw.** A user-not-found, a validation failure, a declined card — these are outcomes the caller must handle, and an exception makes them easy to forget. A `Result<T, E>` type (hand-rolled or `neverthrow`) makes the failure part of the signature. A database connection dropping, a bug — those throw and are caught once at the top. Do not wrap every function in `Result`; that is Go cosplay with worse ergonomics.

**Add context when rethrowing, or do not catch.** `catch (e) { throw e }` is noise. `catch (e) { throw new ServiceError("sync users", { cause: e }) }` adds the operation.

**Top-level handlers exist and terminate.** `process.on("unhandledRejection")` and `process.on("uncaughtException")` should log and exit, not swallow and continue; the process is in an unknown state.

**`finally` for cleanup; `using` (TS 5.2+) for resources with a `[Symbol.dispose]`.** `await using` for async disposal. Where the runtime supports it, this replaces the try/finally dance for connections, locks, and temp files.

## Absence: `null` and `undefined`

Two values for "nothing" is a language wart; the codebase needs one convention. `undefined` is the runtime's own choice (missing properties, missing arguments, `Map.get` misses) and the usual pick. Convert `null` from libraries and JSON at the boundary.

- **`??` for defaults, not `||`.** `||` treats `0`, `""`, and `false` as missing. `count || 10` when `count` is `0` is a bug that ships.
- **`?.` for optional access**, and stop the chain where the absence should be handled rather than propagated three calls further.
- **Optional parameter vs `| undefined`**: `f(x?: string)` allows omission; `f(x: string | undefined)` requires the caller to say so explicitly. The second is right when forgetting is the likely bug.
- **Do not return `null` from your own functions** to mean "not found" when a discriminated union or `undefined` would do. Mixing both forces every caller to check both.

## Modules

- **ESM.** `"type": "module"` in `package.json`, `import`/`export`, `.js` extensions in relative imports when `moduleResolution` requires them. CommonJS in new code needs a stated reason (an old framework, a tool that cannot load ESM).
- **Named exports.** They are greppable, refactorable, and auto-importable. A default export has a different name in every file that imports it.
- **No import-time side effects.** A module that connects to a database, reads env, or starts a timer when imported is untestable and makes import order matter. Export a function that does it; call it from `main`.
- **Barrel files (`index.ts` re-exporting a directory) are a cost, not a convenience.** They create circular imports, defeat tree-shaking, and make every consumer pay to load everything. One for the package's public API is fine; one per directory is a finding.
- **Circular imports are silent.** Module A imports B imports A: one of them sees `undefined` at import time, and the failure surfaces somewhere else. `madge --circular` or the `import/no-cycle` lint rule. Fix by extracting the shared piece.
- **`import type`** for types. It keeps the import erased and makes intent visible.
- **Dependency direction points inward.** `http` imports `service` imports `domain`. `domain` imports nothing of yours. A `domain` module importing from `http` is the finding.

## Immutability and mutation

- **Do not mutate arguments.** A function that sorts the array it was given, or sets a field on the object it received, has a side effect its signature does not declare. `readonly` in the parameter type makes the compiler enforce it.
- **Return new values.** Spread (`{ ...obj, x }`, `[...arr, x]`) is shallow. `structuredClone` for deep copies of plain data; it does not copy functions, class instances keep their prototype loss, and it is not free.
- **`Object.freeze` is shallow and runtime-only.** `as const` is the compile-time version and costs nothing.
- **`Array.prototype.sort` and `reverse` mutate in place.** `toSorted` and `toReversed` (ES2023) do not. Sorting a shared array you did not own is a classic.

## Functions and parameters

- **Options object over positional parameters** once there are three, or once any is a boolean. `createUser(name, true, false)` is unreadable at the call site; `createUser({ name, sendWelcome: true, admin: false })` is not.
- **Boolean parameters are a smell.** They usually mean the function does two things. Two functions, or a discriminated option.
- **Arrow functions for callbacks and closures; `function` declarations for top-level named functions** — they hoist and they have a name in stack traces. A NIT either way.
- **Prefer functions to classes for stateless behaviour.** A class with no state and one method is a namespace. A class with state and lifecycle is fine; classes are not a smell, unnecessary classes are.
- **`this` is a trap.** Methods passed as callbacks lose it. Arrow-function class fields or explicit binding, or do not pass methods around.

## Naming

- `camelCase` for values and functions, `PascalCase` for types, classes, and enums-as-const objects, `SCREAMING_CASE` only for true module-level constants.
- **No `I` prefix on interfaces**, no `T` prefix on types beyond single-letter generics, no Hungarian.
- **File names**: match the codebase (kebab-case is the common Node choice). One convention; a directory with `userService.ts` and `order-service.ts` is a finding.
- **Booleans read as predicates**: `isReady`, `hasPermission`, `canRetry`.
- **Do not repeat the type in the name**: `users`, not `userArray`; `config`, not `configObject`.

## Runtime pitfalls

- **`===` always.** `==` coerces. The lint rule is `eqeqeq`.
- **`Array.prototype.sort()` with no comparator sorts as strings.** `[10, 9, 1].sort()` is `[1, 10, 9]`. Always pass a comparator for numbers.
- **`typeof null === "object"`**, `typeof [] === "object"`. `Array.isArray` for arrays; a discriminated union for everything else.
- **Numbers are doubles.** `0.1 + 0.2 !== 0.3`; integers are exact only up to `Number.MAX_SAFE_INTEGER`. Money is integer minor units or a decimal library. IDs over 2^53 are strings or `BigInt`.
- **`Date` is mutable, timezone-confused, and parses strings inconsistently.** `Temporal` where available; `date-fns` or `luxon` otherwise. Store and transmit ISO 8601 UTC strings.
- **`for...in` iterates inherited enumerable keys**; use `for...of` with `Object.keys`/`entries`.
- **`Map` and `Set` for keyed collections**, not objects with dynamic keys: no prototype pollution, any key type, correct `.size`.
- **`JSON.stringify` drops `undefined`, functions, and symbols, and throws on `BigInt` and cycles.** `Date` becomes a string and does not come back.
- **`parseInt` without a radix**, and `parseInt("12px") === 12`. `Number()` or a parser with a schema.
- **Regex with the `g` flag is stateful** (`lastIndex`). A module-level `/x/g` reused across calls returns alternating results.

## Node specifics

- **Do not block the event loop on a request path.** `fs.readFileSync`, `child_process.execSync`, `crypto.pbkdf2Sync`, `JSON.parse` of an unbounded body, a tight loop over a large array — each stalls every other request for the duration. Async equivalents, streaming, or `worker_threads`.
- **`process.env` values are `string | undefined`.** Parse them once at startup into a typed config object; do not read `process.env` inside functions.
- **Use `node:` prefixed imports** (`node:fs`, `node:path`) so the builtin cannot be shadowed by a package.
- **`fetch` is built in (Node 18+) and has no timeout.** Pass `AbortSignal.timeout(ms)`.
- **Streams for anything larger than memory**, with `stream/promises.pipeline` so errors propagate and backpressure works. Reading a whole upload into a buffer is a DoS.
- **Graceful shutdown.** Handle `SIGTERM`: stop accepting, `server.close()`, drain in-flight work with a deadline, close pools, exit. Without it every deploy drops requests.
- **Timers keep the process alive.** `setInterval` without `.unref()` or a clear prevents exit; a test suite that hangs at the end usually has one.
