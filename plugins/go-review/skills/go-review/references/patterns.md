# Design patterns in Go

Go has no inheritance, first-class functions, cheap goroutines, and structural interfaces. That combination makes some GoF patterns disappear into the language, some become one-liners, and a few become actively harmful. Judge a pattern by whether it removes a real repetition or a real coupling — not by whether it has a name.

Contents: [Functional options](#functional-options) · [Constructor and consumer interface](#constructor-and-consumer-interface) · [Middleware / decorator](#middleware-decorator) · [Concurrency patterns](#concurrency-patterns) · [Strategy as a function](#strategy-as-a-function) · [Repository](#repository) · [Patterns that dissolve](#patterns-that-dissolve-into-the-language) · [Patterns to avoid](#patterns-to-avoid-in-go) · [Review questions](#review-questions)

## Functional options

For constructors with optional configuration, especially in a library where you cannot break the signature later.

```go
type Server struct {
    addr    string
    timeout time.Duration
    logger  *slog.Logger
}

type Option func(*Server)

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}

func New(addr string, opts ...Option) *Server {
    s := &Server{addr: addr, timeout: 30 * time.Second, logger: slog.Default()}
    for _, o := range opts {
        o(s)
    }
    return s
}
```

Required arguments stay positional so the compiler enforces them; optional ones become options with real defaults. Adding an option later breaks nobody.

Use `Option func(*Server) error` when an option can be invalid, so `New` can reject it rather than storing a bad value.

Do not use options for two or three fields where a config struct is clearer, and do not use them for required arguments — an option that must be passed is a positional parameter in disguise, discoverable only by reading the source.

## Constructor and consumer interface

The default shape for a dependency in Go:

```go
// consumer package declares the narrow interface it needs
type Store interface {
    Get(ctx context.Context, id string) (*Item, error)
}

type Service struct{ store Store }

func NewService(s Store) *Service { return &Service{store: s} }
```

The producer returns its concrete type. `main` wires. Accept an interface, return a struct. See `solid.md` for why the interface belongs on this side.

## Middleware / decorator

Go's strongest composition pattern, and the reason `net/http` scaled without a framework.

```go
type Middleware func(http.Handler) http.Handler

func Chain(h http.Handler, ms ...Middleware) http.Handler {
    for i := len(ms) - 1; i >= 0; i-- {
        h = ms[i](h)
    }
    return h
}
```

The same shape works for any single-method interface — a `RoundTripper` for HTTP client retries and tracing, a `Store` wrapper for caching or metrics, a gRPC interceptor.

Watch the ordering: recovery outermost, then tracing, then logging, then auth, then rate limiting, then the handler. A recovery middleware inside the logger cannot log the panic it caught; a rate limiter outside auth lets unauthenticated traffic consume the budget.

## Concurrency patterns

**Worker pool** — bounded parallelism over a queue. Reach for `errgroup.SetLimit` first; hand-rolling a pool is worth it only when you need per-worker state.

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(runtime.GOMAXPROCS(0))
for _, job := range jobs {
    job := job                 // needed on go < 1.22
    g.Go(func() error { return process(ctx, job) })
}
err := g.Wait()
```

**Pipeline** — stages connected by channels, each stage a function taking an input channel and returning an output channel. Every stage must select on `ctx.Done()` or the whole pipeline leaks when a downstream stage exits early.

**Fan-out / fan-in** — one producer, N workers, one collector. The collector must be drained or the workers block on their final sends. The classic bug is returning early from the collector on the first error and leaking every worker.

**Singleflight** (`golang.org/x/sync/singleflight`) — collapses concurrent identical requests into one. The correct answer to a cache stampede, and the thing most hand-rolled caches are missing. See the caching architect skill.

**Semaphore** — a buffered channel as a token bucket, or `golang.org/x/sync/semaphore` for weighted acquisition.

Review finding: unbounded goroutine creation per input item. `for _, x := range items { go f(x) }` over a slice of unknown size is a resource exhaustion bug, not a pattern.

## Strategy as a function

When the strategy has one method, use a function type. An interface adds a named type and an implementing struct to express what `func(Order) Money` already says.

```go
type Pricer func(Order) Money

func (s *Service) Total(o Order, p Pricer) Money { return p(o) }
```

Method values satisfy this — `svc.PremiumPricing` passes directly. Use an interface once the strategy needs two methods or its own state with a lifecycle.

## Repository

Useful when it earns its keep, which is when the domain type differs from the storage schema, or when the consumer genuinely needs to be swappable for tests.

```go
type UserRepo interface {
    Find(ctx context.Context, id UserID) (*User, error)
    Save(ctx context.Context, u *User) error
}
```

Two failure modes to look for in review:

**A leaky repository.** Methods returning `*sql.Rows`, taking a `squirrel.SelectBuilder`, or accepting a raw SQL fragment. The abstraction has published its implementation and now constrains you to it.

**A repository that is a thin rename of the driver.** If every method is one query with no mapping and no domain type, and the only implementation is Postgres, you have added a layer of indirection and no capability. Passing `*sql.DB` and writing queries in the service is a legitimate choice.

Transactions are where this pattern usually breaks. Spanning two repository methods in one transaction needs a deliberate answer — a `WithTx(ctx, func(Repo) error)` method, or a unit-of-work type — decided up front rather than discovered when the second write appears.

## Patterns that dissolve into the language

- **Iterator** — `range`, and `range`-over-func iterators on Go 1.23+.
- **Adapter** — an interface satisfied structurally; often just a function type conversion.
- **Template method** — a function taking a callback.
- **Observer** — a slice of channels or callbacks; no framework needed.
- **Command** — a closure.
- **Builder** — usually functional options, or a struct literal with field names.
- **Prototype** — struct assignment copies (watch reference-typed fields: maps, slices, pointers are shared, so a "copy" that then mutates a map field mutates the original).

Naming these in Go code adds vocabulary without adding structure. `NewUserIteratorFactory` is a review finding.

## Patterns to avoid in Go

**Singleton via package-level mutable state.** Shared memory with no synchronization contract, order-dependent initialization, and flaky parallel tests. Construct one instance in `main` and pass it.

**Abstract factory / factory of factories.** Two levels of indirection to choose a constructor. A `switch` in `main` is clearer and greppable.

**Deep embedding chains to simulate inheritance.** Embedding promotes the full method set, and the embedded type's methods only ever see the embedded value — so an "override" is not polymorphic. Three levels deep and nobody can tell which method runs.

**Service locators and reflection-based DI containers.** Compile-time wiring in `main` scales further than people expect and survives refactoring; a container fails at startup instead of at build.

**`interface{}` / `any` as a generic escape hatch.** Generics or a small interface. `any` in a signature is a review finding — ask what it is hiding.

**Exceptions via `panic`.** `panic` is for programmer error and unrecoverable state. Using `panic`/`recover` for control flow across package boundaries breaks every caller's expectations. `recover` in an HTTP middleware to keep the server up is the legitimate case.

## Review questions

For any pattern in a diff, ask in this order:

1. **What repetition or coupling does this remove?** If the answer is hypothetical, the abstraction is speculative.
2. **What is the simplest thing that works?** Function value, struct literal, direct call.
3. **How many implementations exist today?** One, with no test double, means do not extract the interface yet.
4. **Does it make the likely next change cheaper?** Not any change — the one actually coming.
5. **Can a new reader follow the call path without a diagram?** Indirection has a real, permanent readability cost that abstraction discussions tend to price at zero.
