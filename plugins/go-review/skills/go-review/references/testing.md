# Testing Go

Contents: [What to test](#what-to-test) · [Shape of a test](#shape-of-a-test) · [Doubles](#doubles-fakes-before-mocks) · [Integration and the database](#integration-and-the-database) · [Concurrency](#concurrency) · [Time](#time) · [Fuzzing and property tests](#fuzzing-and-property-tests) · [Benchmarks](#benchmarks) · [Flakiness](#flakiness) · [Coverage](#coverage) · [Review questions](#review-questions)

## What to test

**Test the behaviour the caller depends on, through the exported API.** A test that reaches into unexported fields or asserts on the sequence of internal calls breaks on every refactor and misses the bugs users hit.

**Test the failure modes the code is prone to, not just the happy path.** For each function ask: what input, timing, or dependency failure would make this wrong? That is the test. A function that parses has a malformed-input test. A function that retries has a test where the second attempt succeeds and one where every attempt fails. A function that takes a `ctx` has a test that cancels it.

**One behaviour per test case.** A case that asserts eight things tells you nothing useful when the third one fails.

**The unit is the package, not the function.** Testing every unexported function individually locks the implementation in place. Test the package's contract; let internals change.

## Shape of a test

Table-driven, named cases, subtests:

```go
func TestParseDuration(t *testing.T) {
    tests := []struct {
        name    string
        in      string
        want    time.Duration
        wantErr error
    }{
        {name: "seconds", in: "30s", want: 30 * time.Second},
        {name: "empty", in: "", wantErr: ErrEmpty},
        {name: "negative", in: "-1s", wantErr: ErrNegative},
    }
    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            got, err := ParseDuration(tc.in)
            if !errors.Is(err, tc.wantErr) {
                t.Fatalf("err = %v, want %v", err, tc.wantErr)
            }
            if got != tc.want {
                t.Errorf("got %v, want %v", got, tc.want)
            }
        })
    }
}
```

- `t.Fatalf` when continuing makes no sense; `t.Errorf` to collect several failures from one case.
- `t.Helper()` in every helper so the failure line points at the test, not the helper.
- `t.Cleanup` over `defer` in helpers; it runs even when the helper returns early.
- `t.TempDir()`, `t.Setenv()` over hand-rolled equivalents; they clean up and they refuse to run under `t.Parallel()` where that would be unsafe.
- Compare structs with `cmp.Diff` (`github.com/google/go-cmp`) and print the diff; `reflect.DeepEqual` plus `%v` produces a wall of text nobody reads.
- Assertion libraries are a team choice. `testify` is fine; a bespoke assertion package with forty helpers is a finding.

## Doubles: fakes before mocks

**A fake is a working implementation with a shortcut** — an in-memory `Store`, an `httptest.Server`, a `bytes.Buffer` for an `io.Writer`. **A mock records calls and replays scripted answers.** Prefer fakes: they test behaviour, survive refactors, and read like code.

Mocks are the right tool when the interaction itself is the contract — "this must call `Rollback` when `Commit` fails." Otherwise a mock test tends to assert that the code does what the code does.

Because Go interfaces are consumer-defined and small (see `solid.md`), a hand-written fake is usually five lines. A mock generator is a signal the interface is too big.

Do not fake what you do not own at the boundary. Fake your `PaymentGateway` interface; do not mock `*http.Client` internals. Use `httptest.Server` or a custom `http.RoundTripper` to test the HTTP layer against a real request/response.

## Integration and the database

Unit tests with an in-memory fake do not prove the SQL is right. For code that talks to a database:

- **Run the real engine.** `testcontainers-go` for Postgres, or a local instance addressed by env var. SQLite as a stand-in for Postgres tests a different query planner and different semantics; it is only acceptable when the production engine is SQLite.
- **One database per test binary, one schema or transaction per test.** Wrap each test in a transaction that rolls back, or use a fresh schema per test. Tests that share rows are order-dependent.
- **Apply the real migrations** in test setup. Testing against a hand-written schema tests a schema that does not exist.
- **Gate with a build tag or env var** (`//go:build integration`, or skip when `TEST_DATABASE_URL` is unset) so the unit run stays fast, and run the integration set in CI on every push, not nightly.
- **Golden files** (`testdata/*.golden`, regenerated with `-update`) for large stable outputs: rendered templates, generated code, serialised responses. Review the diff of the golden file in the PR; a golden test whose file is regenerated without anyone reading it tests nothing.

## Concurrency

Single-goroutine tests over concurrent code are the most common false sense of safety in Go.

- Every type documented as safe for concurrent use has a test that hits it from N goroutines through a `WaitGroup`, run under `-race`.
- `go test -race` is the default test target in CI, not an optional job.
- `-count=10` and `-cpu=1,4,8` on packages with scheduling-sensitive code.
- `go.uber.org/goleak` in `TestMain` turns a leaked goroutine into a test failure.
- `t.Parallel()` on tests that share nothing. A parallel test that touches a package-level variable is itself a race; the detector will find it, eventually.

## Time

`time.Sleep` in a test is a finding. It is either too short (flaky) or too long (slow), and it is usually both on different machines.

- Inject a clock. A `func() time.Time` field, or an interface with `Now()` and `After()`, defaulting to the real one.
- Go 1.25+: `testing/synctest` runs a goroutine bubble on a fake clock, so `time.Sleep` inside the code under test advances instantly and deterministically. Prefer it to a hand-rolled clock when the module's `go` directive allows.
- For "eventually" assertions, poll with a deadline (`require.Eventually`, or a loop with `time.Tick` and a `context.WithTimeout`), never a fixed sleep.

## Fuzzing and property tests

`go test -fuzz=FuzzParse` (Go 1.18+) is built in and cheap to add for anything that parses, decodes, or validates untrusted input. Seed the corpus with the table-test inputs. Run it for a bounded time in CI and indefinitely somewhere occasionally. Fuzz targets that only check "does not panic" already find real bugs; round-trip properties (`decode(encode(x)) == x`) find more.

## Benchmarks

`func BenchmarkX(b *testing.B)` with `b.ReportAllocs()`. Compare before and after with `benchstat`; a single run's number is noise. Benchmarks belong on the paths where performance is a requirement, and the requirement should be written down next to the benchmark.

## Flakiness

A flaky test is a bug in the test or a bug in the code, and it is never acceptable to retry it into passing. The usual causes, in order: shared mutable state between parallel tests, real time, real network, map iteration order, unsynchronised goroutines, and test-order dependence. `go test -shuffle=on` exposes the last one.

Quarantine with `t.Skip` and a linked issue if you cannot fix it now; do not leave it red and do not leave it in a retry loop.

## Coverage

`go test -coverprofile=cover.out ./... && go tool cover -html=cover.out`. Coverage is a map of what is untested, not a score. 100% coverage with tests that assert nothing is worse than 70% with tests that exercise the failure modes. Read the uncovered lines; if they are error paths, that is where the bugs are.

## Review questions

1. Does a test exist for the failure mode this code is prone to? (Concurrent code → concurrent test. Parser → malformed input. Retry → exhausted retries. `ctx` → cancelled `ctx`.)
2. Would this test fail if the bug it targets were introduced? A test that passes against a stubbed-out function is decoration.
3. Does it test through the exported API, or through internals that will change?
4. Is there a `time.Sleep`, a real network call, or shared state between parallel tests?
5. If it talks to a database, is it the real engine with the real migrations?
6. When it fails, will the message say what was expected and what was got, with a name that identifies the case?
