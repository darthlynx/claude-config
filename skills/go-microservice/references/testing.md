# Testing

Go's `testing` package plus a few well-chosen helpers cover almost everything. The goal is fast,
deterministic, parallel-safe unit tests and a thin layer of integration tests against real
dependencies via containers.

## Table-driven tests (the default shape)

```go
func TestParseAmount(t *testing.T) {
	t.Parallel()

	tests := map[string]struct {
		in      string
		want    int64
		wantErr bool
	}{
		"plain":     {in: "12.34", want: 1234},
		"no cents":  {in: "12", want: 1200},
		"negative":  {in: "-1.00", wantErr: true},
		"garbage":   {in: "abc", wantErr: true},
	}

	for name, tc := range tests {
		t.Run(name, func(t *testing.T) {
			t.Parallel()
			got, err := ParseAmount(tc.in)
			if tc.wantErr {
				if err == nil {
					t.Fatalf("expected error, got nil")
				}
				return
			}
			if err != nil {
				t.Fatalf("unexpected error: %v", err)
			}
			if got != tc.want {
				t.Errorf("ParseAmount(%q) = %d, want %d", tc.in, got, tc.want)
			}
		})
	}
}
```

- A `map` of cases reads cleanly and randomizes order (surfacing hidden ordering deps). A `[]struct`
  with a `name` field is equally idiomatic — both are fine.
- `t.Parallel()` at both levels for CPU-bound, side-effect-free tests.
- Prefer plain `if got != want` assertions for clarity; `go-cmp` (`cmp.Diff`) for structs/slices.
  `testify/require` is acceptable if the team already uses it, but stdlib keeps deps down.

## Mocks & fakes

- Test against **interfaces defined by the consumer** (see `references/project-layout.md`). Inject
  a fake implementation in tests.
- Prefer **hand-written fakes** for small interfaces — they're clearer than generated mocks and
  don't drift. For large or call-order-sensitive interfaces, generate with `uber-go/mock`
  (`go.uber.org/mock/gomock`, the maintained successor to `golang/mock`) or `mockery`.
- Don't over-mock: mocking the standard library or `pgx` directly is a smell. Mock *your* ports;
  test the adapters with real (containerized) backends.

## HTTP tests with `httptest`

```go
func TestHealthHandler(t *testing.T) {
	t.Parallel()
	req := httptest.NewRequest(http.MethodGet, "/healthz", nil)
	rec := httptest.NewRecorder()

	NewHealthHandler().ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Fatalf("status = %d, want 200", rec.Code)
	}
}
```

Use `httptest.NewServer` when you need a real socket (testing a client). For outbound calls in unit
tests, inject an `*http.Client` whose `Transport` is a `RoundTripper` fake.

## Integration tests with `testcontainers-go`

Spin up real Postgres/Kafka/etc. so adapters are tested against the genuine article. Guard with a
build tag or `testing.Short()` so unit runs stay fast.

```go
//go:build integration

func TestStore_Integration(t *testing.T) {
	ctx := t.Context() // auto-cancelled at test end

	pg, err := postgres.Run(ctx, "postgres:16-alpine",
		postgres.WithDatabase("app"),
		postgres.WithUsername("app"),
		postgres.WithPassword("secret"),
		testcontainers.WithWaitStrategy(
			wait.ForLog("database system is ready to accept connections").
				WithOccurrence(2).WithStartupTimeout(30*time.Second)),
	)
	if err != nil {
		t.Fatalf("start postgres: %v", err)
	}
	t.Cleanup(func() { _ = pg.Terminate(ctx) })

	dsn, err := pg.ConnectionString(ctx, "sslmode=disable")
	if err != nil {
		t.Fatalf("dsn: %v", err)
	}
	// migrate (goose), construct store, exercise real queries...
	_ = dsn
}
```

- Use `t.Context()` for a context that cancels when the test ends.
- Reuse one container across a package's tests with `TestMain` when startup cost dominates.
- Run integration tests in CI with `go test -tags=integration ./...` as a separate (slower) stage.

## Golden files

For serializers, renderers, or anything with a stable textual output, compare against a checked-in
golden file and regenerate with a flag:

```go
var update = flag.Bool("update", false, "update golden files")
// ...
if *update {
	os.WriteFile(golden, got, 0o644)
}
want, _ := os.ReadFile(golden)
if !bytes.Equal(got, want) {
	t.Errorf("output mismatch (run with -update to refresh)\n%s", cmp.Diff(string(want), string(got)))
}
```

## Discipline

- **`go test -race ./...` always** — locally and in CI. Concurrency bugs hide without it.
- No `time.Sleep` to "wait" for async work — synchronize on channels, `sync.WaitGroup`, or poll with
  a deadline. For time-dependent code, prefer **`testing/synctest`** (Go 1.24+, GA in 1.25) over
  injecting a clock: it runs the test in a "bubble" with a fake clock that the runtime advances
  whenever all goroutines block, so timeouts, tickers, and retries execute instantly and
  deterministically. Inject a clock (`func() time.Time`) only for code paths `synctest` can't reach
  (e.g. interactions with external systems mid-test).
- Keep tests in `package foo` for white-box, `package foo_test` for black-box API tests; use the
  latter to verify the public surface.
- `t.Helper()` in assertion helpers so failures point at the caller.
- Measure with `testing.B` benchmarks and `-benchmem` before optimizing; don't guess.
- Fuzz parsers and decoders with `testing.F` (`func FuzzX(f *testing.F)`).
