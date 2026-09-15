---
name: ts-coding
description: Conventions for writing TypeScript — strict types, runtime validation at boundaries, error handling, async and cancellation, module layout, tests — applied while writing or modifying any .ts or .tsx file. Use this whenever you are about to create or edit TypeScript code, add a module, write a test, or wire a dependency in a Node or browser project. For reviewing TypeScript that already exists, use ts-review instead.
---

# Writing TypeScript

This skill loads when TypeScript is being written. It carries the same conventions the reviewer will hold you to, so that the code is right the first time and the Stop hook has nothing to say. The detailed reasoning lives in the shared reference files; this file is the short form.

## Before writing

1. **Read `tsconfig.json`.** `strict` on or off changes what the compiler will catch for you. If `noUncheckedIndexedAccess` is off, every `arr[i]` is a possible `undefined` the compiler will not mention. Write to the strictness the project has, and say so if it is too loose to be safe.
2. **Read `package.json` and the lockfile.** The lockfile names the package manager; use it, never a different one. Scripts name what the team runs for lint, typecheck, and test; use those, not your own flags. The `type` field (`module` or absent) decides ESM vs CommonJS, which decides import syntax.
3. **Read the neighbouring module.** Match its export style, error style, validation library, and test layout. A codebase with two conventions is worse than one with one imperfect convention.
4. **Load the reference for the ground you are on.** Paths are relative to the `ts-review` skill directory next to this one:

| Writing | Read first |
|---|---|
| Anything | `../ts-review/references/idiomatic.md` |
| Anything with `async`, `Promise`, streams, timers, or outbound calls | `../ts-review/references/async.md` |
| A new module, interface, class, or dependency wiring | `../ts-review/references/solid.md` |
| Something that feels like it wants a pattern | `../ts-review/references/patterns.md` |
| Tests | `../ts-review/references/testing.md` |

## The conventions in twelve lines

1. Types are compile-time only. Anything that crosses a boundary (JSON, env, request body, database row, message payload) is `unknown` until a schema parses it. Parse once at the edge; trust the type inside.
2. No `any`. `unknown` and narrow. No `as` except to narrow after a check the compiler cannot see. No `!` unless the line above proves it. No `@ts-ignore`; `@ts-expect-error` with a reason if you must.
3. Discriminated unions for state and variants, exhaustive `switch` with a `never` check in `default`. Not optional-field bags.
4. Prefer `type` for unions and `interface` for object shapes; either is fine, but match the codebase. No `enum`; use `as const` objects and derive the union.
5. Throw only `Error` subclasses, with `cause`. `catch (e: unknown)` and narrow. Never swallow; either handle, rethrow with context, or let it propagate. Expected failures are return values (a `Result` type), not exceptions.
6. Every promise is awaited, returned, or deliberately `void`-ed with a comment and a rejection handler. `return await` inside `try`. `Promise.all` only over bounded input; otherwise a limiter.
7. Every outbound call takes an `AbortSignal` and has a timeout. Every long-running process handles `SIGTERM` and drains.
8. Every `await` is a yield point. Read-then-write on shared state across an `await` is a race; hold the invariant in one synchronous step or use a lock.
9. Do not block the event loop in a server: no `*Sync` I/O on the request path, no `JSON.parse` of unbounded input, no CPU-heavy loops without a worker.
10. Named exports. No barrel files that re-export whole directories. No import-time side effects. `import type` for types.
11. Options object with named fields, not positional booleans. `readonly` on inputs you do not mutate. Return new values; do not mutate arguments.
12. Tests are colocated, test the exported behaviour, and cover the failure mode the code is prone to. Async tests `await` everything.

## Coming from Go

The places where Go instincts produce wrong TypeScript:

- **Errors are exceptions, not values, by default.** Any call can throw. The `Result` pattern gives you Go-shaped error handling for *expected* failures; use it at module boundaries, but do not wrap every function in it. Unexpected failures should throw and be caught at the top.
- **There is one thread, and it still has races.** No data races in the Go sense, but every `await` interleaves with every other request. Check-then-act across an `await` is the same bug as an unguarded map. There is no mutex in the standard library; use a single synchronous step, a queue, or a small lock utility.
- **No goroutines, no `errgroup`.** Concurrency is `Promise.all` and friends, and it is unbounded by default. `p-limit` or a hand-written semaphore is your `SetLimit`. CPU work needs `worker_threads`; there is no runtime scheduler to spread it.
- **No zero values.** A missing field is `undefined`, not `""` or `0`. `?? default` at the read site, or a schema with defaults at the boundary. `||` treats `0` and `""` as missing; use `??`.
- **Structural typing is everywhere, including for object literals.** An extra property in a literal is a compile error (excess property check), but the same object passed through a variable is not. Do not rely on the shape check as validation.
- **`null` and `undefined` are both present.** Pick one for "absent" in your own code (usually `undefined`) and convert at the boundary where a library uses the other.
- **Packages are modules, not directories.** Circular imports are possible and silently produce `undefined` at import time. Dependencies point one way; if two modules import each other, extract the shared piece.
- **`interface` does not mean what it means in Go.** Consumer-defined small interfaces are still the right idea (see `solid.md`), but any object with the right shape satisfies it, including ones that were never meant to. Prefer branded types for identifiers that must not be mixed up.
- **Unhandled rejections crash the process.** A floating promise is not a leaked goroutine that sits quietly; when it rejects, Node exits.

## Do not add

- A class where a function and a closure say the same thing.
- An interface with one implementation and no test that needs it.
- A generic parameter that is only ever instantiated with one type.
- A `utils.ts` or `helpers.ts`.
- A dependency for something the standard library or the runtime already does (`fetch`, `AbortSignal.timeout`, `structuredClone`, `node:test`).
- Abstraction against a change that is not actually coming.

## Before declaring done

Run the project's own scripts where they exist; otherwise:

```bash
<pm> exec tsc --noEmit
<pm> exec eslint .
<pm> exec prettier --check .
<pm> test
```

All clean, or say which is not and why. Tests are written alongside the code, not promised for later. A change that adds concurrency adds a test that exercises it concurrently.
