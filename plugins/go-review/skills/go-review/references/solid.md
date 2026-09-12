# SOLID in Go

SOLID was written for languages with class hierarchies. Go has no inheritance, no constructors in the language sense, and structural rather than nominal interfaces. The principles still apply, but the unit they apply to is usually the **package**, not the type — and two of the five have Go-specific translations that look nothing like the textbook version.

Contents: [Single responsibility](#single-responsibility-the-package-is-the-unit) · [Open/closed](#openclosed-composition-and-function-values) · [Liskov](#liskov-interface-contracts) · [Interface segregation](#interface-segregation-the-one-that-matters-most) · [Dependency inversion](#dependency-inversion-consumer-defined-interfaces) · [When not to apply this](#when-not-to-apply-this)

## Single responsibility: the package is the unit

In Go, cohesion is judged at package scope. The question is not "does this struct do one thing" but **"can you describe this package's job in one sentence without using 'and'?"**

Review findings:

- A `utils`, `common`, `helpers`, `shared`, or `base` package. These are named after their lack of responsibility. Split by what the code is *about*, not by what shape it is.
- A `models` or `types` package holding every struct in the system. It creates a dependency hub that every package imports and nothing can change, and it separates data from the behaviour that owns it.
- One file with an HTTP handler, SQL queries, and business rules in the same function. The signature is the tell: a function taking both `*http.Request` and `*sql.DB` is doing transport, persistence, and logic.
- A type with two clusters of methods that never call each other, touching disjoint field sets. That is two types.

What good looks like: `user` owns user identity and its persistence interface; `httpapi` owns transport and calls `user`; `postgres` implements what `user` declares. Each is one sentence.

## Open/closed: composition and function values

Go's extension mechanisms are struct embedding, interface satisfaction, and passing functions. There is no `virtual`, no subclassing, no template method.

**Middleware is the canonical open/closed shape:**

```go
type Middleware func(http.Handler) http.Handler

func WithAuth(next http.Handler) http.Handler { /* ... */ }
func WithLogging(next http.Handler) http.Handler { /* ... */ }
```

New behaviour arrives as a new `Middleware`. Nothing existing is edited.

**Functional options** extend a constructor without breaking callers or growing a config struct with illegal field combinations. See `patterns.md`.

**Strategy as a function type**, not an interface, when there is exactly one method:

```go
type Pricer func(Order) Money      // callers pass a closure or a method value
```

Review findings:

- A `switch` on a type field (`switch order.Kind { case "digital": ... }`) that grows a case for every new variant, in more than one place. Each new kind requires editing every switch. Move the behaviour onto the variant.
- Adding a parameter to an exported function signature to support a new caller — a breaking change where an option would not have been.
- Embedding to reuse implementation rather than to satisfy a contract. Embedding promotes the whole method set, including methods you did not intend to expose, and it is not subtyping — the embedded type's methods still see only the embedded value.

## Liskov: interface contracts

Structural typing means the compiler checks the *shape* and nothing else. Semantics are entirely on you, which makes this principle more relevant in Go than in a language with declared inheritance, not less.

An implementation violates the contract when it:

- Returns an error for an input the interface documents as valid.
- Panics where the interface promises an error.
- Blocks indefinitely where callers reasonably expect to return.
- Ignores `ctx` cancellation while satisfying a `ctx`-accepting interface.
- Silently no-ops. A `NullStore` that satisfies `Store` and drops every `Save` will pass every type check and fail every expectation.
- Is not safe for concurrent use when the interface's other implementations are, and callers were written against those.

Practical review checks: does the interface document its contract at all — error conditions, nil behaviour, concurrency safety? If not, that is the finding. And is there a shared test suite that every implementation runs? A `func TestStoreContract(t *testing.T, s Store)` exercised by each implementation is how Go codebases actually enforce Liskov.

## Interface segregation: the one that matters most

This is where Go's idioms and SOLID agree hardest, and where most review value lives.

**Interfaces should have one to three methods.** `io.Reader` has one. `io.ReadWriteCloser` composes three single-method interfaces.

**Declare the narrowest interface the function needs:**

```go
// Bad: needs one method, demands twelve.
func SendWelcome(ctx context.Context, db Database, id int64) error

// Good: the signature now documents exactly what this touches.
type userGetter interface {
    GetUser(ctx context.Context, id int64) (*User, error)
}
func SendWelcome(ctx context.Context, g userGetter, id int64) error
```

The narrow version is testable with a five-line fake instead of a generated mock, and its signature tells a reader what it can and cannot do — which is a real safety property, not just tidiness.

Review findings:

- An interface with more than about five methods. Ask which callers use which subsets.
- An interface where every implementation panics or returns `ErrNotImplemented` for some methods. That is a segregation failure the compiler cannot see.
- A test fake that must implement ten methods to exercise one. The pain is the diagnostic.
- Mock frameworks used because hand-writing the fake would be too tedious. Sometimes justified; often a symptom.

## Dependency inversion: consumer-defined interfaces

Go's version is the strongest inversion of the five, and it is the one most often gotten backwards by people arriving from Java or C#.

**The consumer declares the interface. The producer returns a concrete type.**

```go
// package order — the consumer. It declares what it needs.
type Inventory interface {
    Reserve(ctx context.Context, sku string, n int) error
}

func NewService(inv Inventory) *Service { return &Service{inv: inv} }
```

```go
// package warehouse — the producer. It knows nothing about `order`.
func New(db *sql.DB) *Client { ... }
func (c *Client) Reserve(ctx context.Context, sku string, n int) error { ... }
```

`warehouse` does not import `order`. `order` does not import `warehouse`. `main` wires them. There is no shared interface package, and the dependency arrow points from the concrete implementation toward the abstraction that the consumer owns.

Contrast with the anti-pattern: a central `interfaces` package that both sides import. Now every change to any contract touches a package everything depends on, and you have reintroduced the coupling you were trying to remove.

Review findings:

- Interfaces defined next to their single implementation in the producer package, then imported by consumers.
- A package named `interfaces`, `contracts`, or `ports` holding interfaces for the whole system.
- Dependencies reached through package-level variables or `init()` rather than passed to a constructor. Untestable and order-dependent.
- Dependencies pulled out of `context.Value`. That is dependency injection with the type system switched off.
- A DI container or reflection-based wiring framework. `main` calling constructors in order is compile-time checked, greppable, and adequate well past the point people reach for a container.

## When not to apply this

Go's culture pushes back on speculative abstraction, and that pushback is correct often enough that it belongs in the review.

- **One implementation and no test double? Do not extract an interface.** Add it when the second implementation or the test need arrives. `*sql.DB` passed directly is fine.
- **Splitting a 120-line package into six 20-line packages** costs more in navigation than it recovers in cohesion.
- **An interface per struct, mechanically,** doubles the surface area and buys nothing.
- **Wrapping every dependency in an adapter** in case you switch databases — you will not, and if you do, the adapter you wrote today will have leaked the old database's semantics anyway.

The review question is not "does this follow SOLID" but **"what change is likely next, and does this structure make that change cheap or expensive?"** If the answer is that the structure is already fine for the changes actually coming, say so and move on. Abstraction added against a hypothetical is a cost paid now for a benefit that may never arrive.
