# Testing TypeScript

Contents: [What to test](#what-to-test) · [Shape of a test](#shape-of-a-test) · [Doubles](#doubles-fakes-before-mocks) · [Module mocking](#module-mocking) · [Integration and the database](#integration-and-the-database) · [Async and time](#async-and-time) · [Type-level tests](#type-level-tests) · [Snapshot tests](#snapshot-tests) · [Property-based tests](#property-based-tests) · [Flakiness](#flakiness) · [Coverage](#coverage) · [Review questions](#review-questions)

## What to test

**Test the behaviour the caller depends on, through the exported API.** A test that reaches into private fields, spies on internal calls, or asserts on the order of helper invocations breaks on every refactor and misses the bugs users hit.

**Test the failure modes the code is prone to.** For each function ask: what input, timing, or dependency failure would make this wrong? That is the test. A parser has a malformed-input case. A function that retries has a second-attempt-succeeds case and an all-attempts-fail case. A function that takes an `AbortSignal` has an already-aborted case. A function with an `await` between a check and an act has a concurrent-call case.

**The type checker is a test.** Anything the compiler already enforces does not need a runtime test. Do not write `expect(typeof result).toBe("string")` for a function typed to return `string`. Do write tests for what the types cannot say: values, ordering, side effects, and boundaries where `unknown` comes in.

**One behaviour per test.** A test that asserts eight things tells you nothing useful when the third one fails.

## Shape of a test

vitest and jest share the API. Use whichever the project has; `node:test` is fine for dependency-free libraries.

```ts
describe("parseDuration", () => {
  it.each([
    ["30s", 30_000],
    ["2m", 120_000],
  ])("parses %s", (input, expected) => {
    expect(parseDuration(input)).toEqual({ ok: true, value: expected });
  });

  it.each([
    ["", "empty"],
    ["-1s", "negative"],
  ])("rejects %s", (input, error) => {
    expect(parseDuration(input)).toEqual({ ok: false, error });
  });
});
```

- **`it.each` / `test.each`** for table-driven cases, with the input in the name so the failure identifies itself.
- **`toEqual` for structural comparison, `toBe` for identity.** `toStrictEqual` when `undefined` properties matter.
- **Arrange, act, assert, in that order, visibly.** A test that interleaves setup and assertions is hard to read when it fails.
- **Colocate**: `thing.test.ts` next to `thing.ts`. A parallel `__tests__` tree drifts.
- **`beforeEach` for fresh state; never share mutable fixtures across tests.** A module-level `const db = createFakeDb()` shared by every test is order-dependence waiting to happen.
- **No logic in tests.** A test with an `if` or a loop that computes the expected value is testing itself.

## Doubles: fakes before mocks

**A fake is a working implementation with a shortcut** — an in-memory `UserRepo`, a `Map`-backed cache, a `fetch` that returns canned responses. **A mock records calls and replays scripted answers.** Prefer fakes: they test behaviour, survive refactors, and read like code.

Because interfaces are structural and narrow (see `solid.md`), a fake is usually an object literal:

```ts
const users: UserGetter = { getUser: async (id) => (id === "u1" ? alice : undefined) };
await sendWelcome(users, "u1");
```

Mocks (`vi.fn()`) are the right tool when the interaction itself is the contract — "must call `rollback` when `commit` throws" — or when the collaborator is a side effect with no observable result (an email sender). Then assert on the call, and keep it to the one call that matters. A test with twelve `toHaveBeenCalledWith` assertions is a transcript of the implementation, not a test.

Do not fake what you do not own at the boundary. Fake your `PaymentGateway` type; do not mock `fetch` internals or the ORM's query builder. Use `msw` (mock service worker) to fake HTTP at the network layer when the code under test really must call `fetch`.

## Module mocking

`vi.mock("../db")` / `jest.mock` replaces a module for every importer in the test file. It works, and it is a smell:

- It means the dependency was imported at module scope rather than injected. The fix is usually in the code, not the test: make the dependency a parameter.
- It is hoisted above imports, which surprises people: variables declared in the test file are not available inside the factory unless prefixed `vi.hoisted` / referenced lazily.
- It couples the test to the module path. Move the file and the mock silently mocks nothing.
- Partial mocks (`vi.mock(path, async (orig) => ({ ...(await orig()), one: vi.fn() }))`) are where the confusing failures live.

Acceptable for: third-party modules with global side effects you cannot inject (some SDKs), and legacy code you are not refactoring today. Say which in the test.

## Integration and the database

Unit tests with an in-memory fake do not prove the SQL is right. For code that talks to a database:

- **Run the real engine.** `testcontainers` for Postgres, or a local instance addressed by env var. SQLite as a stand-in tests a different planner and different semantics; acceptable only when production is SQLite.
- **One database per test worker, one transaction or schema per test.** vitest runs files in parallel workers by default; two workers sharing one database interleave. Use a per-worker database name (`process.env.VITEST_POOL_ID`) or serialise the integration project.
- **Apply the real migrations** in setup. A hand-written test schema tests a schema that does not exist.
- **Separate the integration project** (`vitest.workspace.ts` or a second config) so the unit run stays fast, and run integration on every push, not nightly.
- **Typed query builders are not a substitute.** Prisma and Drizzle catch column typos at compile time. They do not catch a wrong join, a missing index, or a constraint violation.

## Async and time

- **Every async test awaits.** `it("x", async () => { await expect(p).rejects.toThrow() })` — the `await` on `expect(...).rejects` is mandatory and commonly forgotten; without it the test passes before the assertion runs.
- **Fake timers for anything with `setTimeout`/`setInterval`/`Date.now`.** `vi.useFakeTimers()` in `beforeEach`, `vi.useRealTimers()` in `afterEach`. Advance with `await vi.advanceTimersByTimeAsync(ms)`; the non-`Async` variant does not flush microtasks and is why fake-timer tests hang.
- **`vi.setSystemTime`** for date-dependent logic. Never `new Date()` in a test assertion.
- **Test the race** for any check-then-act across an `await`: start two calls without awaiting the first, await both, assert the invariant. If the race is hard to trigger, inject a deferred promise between the check and the act.
- **Test cancellation**: pass `AbortSignal.abort()` and assert rejection with `AbortError` and no side effect.
- **Test the bound**: a fake downstream that counts in-flight calls; assert the max never exceeds the limit.
- **Real network in a unit test is a finding.** `msw` or a fake at the boundary.

## Type-level tests

For libraries and shared types, the types are the API and deserve tests:

```ts
import { expectTypeOf } from "vitest";
expectTypeOf(parse("x")).toEqualTypeOf<Result<Value, ParseError>>();
expectTypeOf<Config>().toHaveProperty("port").toBeNumber();
```

`vitest --typecheck` runs them. `tsd` is the older equivalent. Use them for exported generics, inferred return types that matter, and anything where a refactor could silently widen a type to `any`.

## Snapshot tests

`toMatchSnapshot()` is a golden file. It is right for large, stable, human-reviewable output: a rendered template, generated code, a serialised API response. It is wrong as a default assertion, because an `-u` run regenerates every snapshot without anyone reading them, and a snapshot of a large object asserts on everything and therefore on nothing.

`toMatchInlineSnapshot()` keeps the expected value in the test file where the reviewer sees it. Prefer it for anything under about twenty lines.

## Property-based tests

`fast-check` generates inputs and shrinks failures:

```ts
fc.assert(fc.property(fc.string(), (s) => decode(encode(s)) === s));
```

Cheap to add for anything that parses, encodes, or validates untrusted input. Round-trip properties and "does not throw" properties find bugs that example tables do not. Seed with the table-test inputs.

## Flakiness

A flaky test is a bug in the test or a bug in the code, and it is never acceptable to retry it into passing. The usual causes, in order: shared mutable state between tests or workers, real time, real network, unawaited promises, unordered `Object.keys` on a `Map` converted to an object, and test-order dependence. `vitest --sequence.shuffle` exposes the last one.

Quarantine with `it.skip` and a linked issue if you cannot fix it now; do not leave it red and do not configure `retry: 3`.

## Coverage

`vitest --coverage` (v8 or istanbul). Coverage is a map of what is untested, not a score. 100% with tests that assert nothing is worse than 70% with tests that exercise the failure modes. Read the uncovered lines; if they are `catch` blocks and `else` branches, that is where the bugs are. A coverage threshold in CI is fine as a ratchet (never goes down), not as a target.

## Review questions

1. Does a test exist for the failure mode this code is prone to? (Boundary input → malformed input test. `await` between check and act → concurrent test. Retry → exhausted-retries test. `signal` → aborted test.)
2. Would this test fail if the bug it targets were introduced? A test whose assertions would pass against a stubbed-out function is decoration.
3. Does it test through the exported API, or through internals that will change?
4. Is there a real `setTimeout`, a real network call, a real `Date`, or a `vi.mock` of a module that should have been a parameter?
5. If it talks to a database, is it the real engine with the real migrations, isolated per worker?
6. Is every promise in the test awaited, including `expect(...).rejects`?
7. When it fails, will the message say what was expected and what was got, with a name that identifies the case?
