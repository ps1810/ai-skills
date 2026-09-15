# SOLID in TypeScript

SOLID was written for class hierarchies. TypeScript has classes, but it also has first-class functions, structural typing, and modules as the real unit of encapsulation. The principles apply; the unit they apply to is usually the **module**, and several have translations that look nothing like the textbook version.

Contents: [Single responsibility](#single-responsibility-the-module-is-the-unit) · [Open/closed](#openclosed-composition-and-unions) · [Liskov](#liskov-structural-typing-checks-shape-not-contract) · [Interface segregation](#interface-segregation-narrow-types-at-the-parameter) · [Dependency inversion](#dependency-inversion-inject-functions-not-containers) · [When not to apply this](#when-not-to-apply-this)

## Single responsibility: the module is the unit

The question is not "does this class do one thing" but **"can you describe this module's job in one sentence without 'and'?"**

Review findings:

- A `utils.ts`, `helpers.ts`, `common.ts`, or `lib.ts`. Named after their lack of responsibility. Split by what the code is *about*.
- A `types.ts` holding every type in the system. It becomes a dependency hub every module imports and nothing can change, and it separates data from the behaviour that owns it. Types live next to the code that produces them.
- One function with a route handler, a SQL query, and business rules. The signature is the tell: a function taking `req: Request` and `db: Pool` is doing transport, persistence, and logic.
- A class with two clusters of methods that never call each other, over disjoint fields. Two classes, or two modules of functions.
- A React component (or any UI unit) that fetches, transforms, and renders. Separate the data hook from the presentational piece.

What good looks like: `user/` owns user identity and declares its persistence interface; `http/` owns transport and calls `user/`; `postgres/` implements what `user/` declares. Each is one sentence.

## Open/closed: composition and unions

Extension mechanisms in TypeScript are function composition, higher-order functions, discriminated unions, and (rarely) class inheritance. Prefer them in that order.

**Middleware is the canonical open/closed shape:**

```ts
type Middleware = (next: Handler) => Handler;
const withAuth: Middleware = (next) => async (req) => { /* ... */ return next(req); };
const withLogging: Middleware = (next) => async (req) => { /* ... */ };
```

New behaviour arrives as a new middleware. Nothing existing is edited. The same shape works for `fetch` wrappers, queue consumers, and store decorators.

**Discriminated unions plus an exhaustive `switch`** are how you add a variant safely: the compiler lists every place that must change. This is closed for modification of the existing cases and open for adding one, with a compile error as the checklist.

**Strategy as a function**, not an interface, when there is one method:

```ts
type Pricer = (order: Order) => Money;   // pass a closure or a method
```

Review findings:

- A `switch` on a string kind field, grown a case per variant, in more than one place. Either make it a discriminated union with exhaustiveness (so the compiler finds every switch), or move the behaviour onto the variant.
- Adding a positional parameter to an exported function to support a new caller. A breaking change where an options-object field would not have been.
- Class inheritance to reuse implementation rather than to express an is-a contract. Three levels deep and nobody knows which `save()` runs. Composition: hold the collaborator, call it.
- Monkey-patching a module or prototype to extend it. It is global, invisible, and order-dependent.

## Liskov: structural typing checks shape, not contract

TypeScript checks that an implementation has the right members. It does not check that they behave. Structural typing makes this sharper than in a nominal language: an object was never declared to implement the interface, it just happens to match.

An implementation violates the contract when it:

- Rejects for an input the interface documents as valid.
- Throws where the interface promises a `Result` or a resolved value.
- Never resolves (hangs) where callers reasonably expect a return.
- Ignores the `signal` while accepting an `AbortSignal`-taking signature.
- Silently no-ops. A `NoopMailer` that satisfies `Mailer` and drops every `send()` passes every type check and fails every expectation.
- Returns a mutable object where others return frozen, or `null` where others return `undefined`, and callers were written against the majority.

Structural-typing traps specific to TypeScript:

- **Optional methods in an interface** (`flush?(): Promise<void>`) mean every caller must check; most will not.
- **Excess property checks apply to literals only.** `const x: Config = { ...obj, extra: 1 }` is an error; `const y = { ...obj, extra: 1 }; const x: Config = y` is not. Do not rely on the shape check as validation.
- **Method parameter bivariance.** Methods declared with method syntax (`save(u: User): void`) are checked bivariantly, so a narrower implementation type-checks. Declare callbacks as property signatures (`save: (u: User) => void`) under `strictFunctionTypes` to get the sound check.

Practical review checks: does the interface document its contract — error conditions, `undefined` behaviour, whether calls may be concurrent? If not, that is the finding. Is there a shared test suite run against every implementation? `describe.each([postgresStore, memoryStore])` is how TypeScript codebases actually enforce Liskov.

## Interface segregation: narrow types at the parameter

This is where TypeScript's type system and SOLID agree hardest, and where most review value lives.

**Declare the narrowest type the function needs, at the parameter:**

```ts
// Bad: needs one method, demands the whole client.
function sendWelcome(db: PrismaClient, id: string): Promise<void>

// Good: the signature documents exactly what this touches.
type UserGetter = { getUser(id: string): Promise<User | undefined> };
function sendWelcome(users: UserGetter, id: string): Promise<void>
```

`Pick<Big, "getUser">` is the one-liner when the big type already exists. The narrow version is testable with a three-line object literal instead of a mock framework, and its signature tells a reader what it can and cannot do.

Review findings:

- A function parameter typed as the whole client, ORM, or SDK object when it calls one method.
- An interface with more than about five methods. Which callers use which subsets?
- An implementation where several methods `throw new Error("not implemented")`. A segregation failure the compiler cannot see.
- A test double that must stub twenty methods to exercise one. The pain is the diagnostic.
- `jest.mock`/`vi.mock` of an entire module to control one export. Usually means the dependency should have been a parameter.

## Dependency inversion: inject functions, not containers

**The consumer declares the type it needs. The producer exports a concrete implementation. `main` wires.**

```ts
// order/service.ts — the consumer. It declares what it needs.
export type Inventory = { reserve(sku: string, n: number, opts: { signal: AbortSignal }): Promise<void> };
export function createOrderService(deps: { inventory: Inventory; clock: () => Date }) { /* ... */ }

// warehouse/client.ts — the producer. Knows nothing about `order`.
export function createWarehouseClient(baseUrl: string) { return { reserve: async (...) => { /* ... */ } }; }

// main.ts — wires
const orders = createOrderService({ inventory: createWarehouseClient(env.WAREHOUSE_URL), clock: () => new Date() });
```

`warehouse/` does not import `order/`. `order/` does not import `warehouse/`. Structural typing means no shared interface package is needed: the client satisfies `Inventory` by shape.

Contrast with the anti-patterns:

- **Importing the concrete dependency at module scope** (`import { db } from "../db"`). Now the module cannot be tested without the real database, and cannot be reused with a different one. This is the most common DIP violation in TypeScript, and it is usually fixed by making the dependency a parameter of a factory function.
- **A central `interfaces/` or `contracts/` directory** that both sides import. Every contract change touches a module everything depends on.
- **Reflection or decorator-based DI containers** (`tsyringe`, `inversify`, NestJS's) outside the framework that mandates them. They move wiring errors from compile time to startup, need `emitDecoratorMetadata`, and make the dependency graph invisible to the type checker and to grep. A `main.ts` calling factory functions is compile-time checked and adequate well past the point people reach for a container. If the framework is NestJS, use its container consistently; do not fight it.
- **Service locators** (`Container.get(Foo)` inside a function). Dependency injection with the type system switched off.
- **Reading `process.env` inside a function.** Config is a dependency; parse it once and pass it.

Review findings:

- A module whose top-level `import`s include a database client, an HTTP client, or a config reader, and whose functions use them directly.
- A test that `vi.mock`s three modules to test one function. The function should have taken those as parameters.
- A class whose constructor does work (connects, reads files). Constructors assign; a static `create()` or the caller does the work.

## When not to apply this

TypeScript's ecosystem pushes toward abstraction — frameworks, decorators, generics — and the pushback belongs in the review.

- **One implementation and no test double? Do not extract the type.** Pass the concrete thing. Add the narrow type when the second implementation or the test need arrives.
- **Do not wrap a library in an adapter "in case we switch."** You will not, and the adapter will have leaked the library's semantics anyway.
- **Do not make it a class** because the pattern was drawn with a class. A factory function returning an object of closures is a class without `this` problems.
- **Do not split a 150-line module into six 25-line modules.** Navigation cost exceeds cohesion gain.
- **Do not add a generic parameter** that is only ever instantiated once.

The review question is not "does this follow SOLID" but **"what change is likely next, and does this structure make that change cheap or expensive?"** If the structure is already fine for the changes actually coming, say so and move on.
