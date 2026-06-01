# Errors & Concurrency

## Error handling

Errors are values; design them like any other part of your API.

### Wrapping & inspection

- Wrap to add context as an error travels up: `fmt.Errorf("load user %d: %w", id, err)`. The `%w`
  verb preserves the chain for `errors.Is`/`errors.As`.
- Add context that the caller *doesn't already have* — the operation and key inputs, not "error:".
- Don't wrap *and* log the same error; pick one. Log once, at the boundary where you decide to
  handle it (usually the handler/`run`), then stop propagating.

### Sentinel vs typed errors

- **Sentinel** (`var ErrNotFound = errors.New("not found")`) when callers only need to branch on
  *which* error: `if errors.Is(err, store.ErrNotFound) { ... }`.
- **Typed** when callers need *data* from the error:

  ```go
  type ValidationError struct{ Field, Reason string }
  func (e *ValidationError) Error() string { return fmt.Sprintf("%s: %s", e.Field, e.Reason) }

  var ve *ValidationError
  if errors.As(err, &ve) { /* use ve.Field */ }
  ```

- Map domain errors → transport codes in **one** place (e.g. a `httpStatus(err) int` helper), so
  status mapping isn't scattered across handlers.

### Don'ts

- No `panic` for ordinary failures; panics are for programmer bugs / unrecoverable invariants. The
  only routine `recover` is the HTTP recoverer middleware.
- No `log.Fatal`/`os.Exit` outside `main`/`run` — it skips defers and is untestable.
- Don't return `nil, nil`. Don't compare error strings (`err.Error() == "..."`) — use `Is`/`As`.
- Don't ignore errors with `_ =` unless you've genuinely reasoned it's safe (and say why in a
  comment). `errcheck` (in golangci-lint) catches these.

## Concurrency

Go makes concurrency easy and goroutine leaks easier. Every goroutine needs a **clear lifecycle**:
who starts it, how it's cancelled, where its error goes.

### `errgroup` for coordinated work

`golang.org/x/sync/errgroup` is the workhorse: run N tasks, cancel the rest on first error,
propagate it, and `Wait` for completion.

```go
func fetchAll(ctx context.Context, ids []int) ([]Item, error) {
	g, ctx := errgroup.WithContext(ctx)
	g.SetLimit(8) // bound concurrency — don't spawn one goroutine per id unbounded
	out := make([]Item, len(ids))
	for i, id := range ids {
		g.Go(func() error {
			item, err := fetchOne(ctx, id) // ctx cancels when any sibling fails
			if err != nil {
				return fmt.Errorf("fetch %d: %w", id, err)
			}
			out[i] = item // distinct index per goroutine → no lock needed
			return nil
		})
	}
	if err := g.Wait(); err != nil {
		return nil, err
	}
	return out, nil
}
```

Writing to distinct slice indices is safe without a mutex; sharing a map or appending to a slice is
**not** — guard those or collect via a channel.

### Long-lived background workers

Coordinate lifecycles in `run()` with an `errgroup` whose context is the signal-cancelled one. Each
worker (HTTP server, Kafka consumer, scheduler) returns when ctx is done; the first to error tears
down the rest:

```go
g, ctx := errgroup.WithContext(ctx)
g.Go(func() error { return runHTTP(ctx, httpSrv, shutdownTimeout) })
g.Go(func() error { return runConsumer(ctx, consumer) })
if err := g.Wait(); err != nil && !errors.Is(err, context.Canceled) {
	return err
}
```

Each worker owns its own drain. For the HTTP server, the shutdown context must NOT inherit `ctx`'s
cancellation (it's already cancelled — that's what triggered the drain). Use
`context.WithoutCancel` so it keeps values (trace IDs) but gets a fresh deadline:

```go
func runHTTP(ctx context.Context, srv *http.Server, drain time.Duration) error {
	errc := make(chan error, 1)
	go func() {
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			errc <- err
		}
	}()
	select {
	case err := <-errc:
		return fmt.Errorf("listen: %w", err)
	case <-ctx.Done():
		shutdownCtx, cancel := context.WithTimeout(context.WithoutCancel(ctx), drain)
		defer cancel()
		return srv.Shutdown(shutdownCtx) // bounded graceful drain
	}
}
```

### Rules that prevent the common bugs

- **A goroutine you start, you must be able to stop.** Pass `ctx`; select on `ctx.Done()`. No
  fire-and-forget `go func(){...}()` without a lifecycle.
- **Run tests with `-race`.** Always.
- **Don't store `context.Context` in structs**; pass it per call.
- **Channels** for ownership transfer/signaling; **mutexes** for protecting shared state. Don't mix
  metaphors. Prefer `sync.RWMutex` only when reads vastly dominate and you've measured it.
- **`sync.Once`** for lazy singletons; **`sync.WaitGroup`** to wait for a known set; **`atomic`**
  (or `sync/atomic` typed `atomic.Int64` etc.) for simple counters.
- Bound everything: worker pools, in-flight requests, channel buffers. Unbounded = OOM under load.
- Beware blocking sends on unbuffered channels after the receiver has gone — a classic leak. Always
  have a `ctx.Done()` arm in selects that send/receive on long-lived channels.
- Detect leaks in tests with `go.uber.org/goleak` in `TestMain`.

### Worker pool sketch

```go
func process(ctx context.Context, jobs <-chan Job, workers int) error {
	g, ctx := errgroup.WithContext(ctx)
	for i := 0; i < workers; i++ {
		g.Go(func() error {
			for {
				select {
				case <-ctx.Done():
					return ctx.Err()
				case job, ok := <-jobs:
					if !ok {
						return nil // channel drained
					}
					if err := handle(ctx, job); err != nil {
						return err
					}
				}
			}
		})
	}
	return g.Wait()
}
```
