# Context Handling

`context.Context` carries three things down a call tree: **cancellation**, **deadlines**, and
**request-scoped values**. Getting context flow right is what makes a service cancel cleanly,
respect timeouts end to end, and keep logs/traces correlated. Most bugs here come from severing the
chain (a stray `context.Background()` mid-request) or from letting a cancelled context kill work
that should have survived.

## The default rule

**Derive from the context you were given.** A function that receives `ctx` passes that same `ctx`
(or a child of it) to everything it calls. This propagates cancellation, deadlines, and values
without any extra plumbing. Breaking the chain — calling `context.Background()` halfway down a
request — silently drops the caller's deadline and loses trace correlation.

```go
func (s *Service) GetUser(ctx context.Context, id int64) (User, error) {
	// derive a per-call timeout from the inherited ctx; never start a new root here
	ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
	defer cancel() // ALWAYS — or you leak the timer/goroutine until the parent is done
	return s.store.UserByID(ctx, id)
}
```

## When to use a fresh root (`context.Background()`)

Only at a genuine lifecycle boundary, where there is no meaningful parent to inherit from:

- **`main` / `run`** — the top of the process (see SKILL.md skeleton).
- **An independent long-lived loop** that owns its own lifecycle (a scheduler, a consumer group)
  and is *not* serving a single inbound request.
- **Tests** — `t.Context()`, which auto-cancels at test end.

Use **`context.TODO()`** (not `Background()`) as a visible placeholder when context *should* be
threaded but isn't yet — it documents the gap and is greppable. Never pass a `nil` context.

## When you have a parent but must escape its cancellation

This is the subtle case. Sometimes you hold a request context but the work must **outlive** the
request or survive a cancellation that has already fired. Don't reach for `Background()` (you'd lose
the values too) — use **`context.WithoutCancel(ctx)`**, which keeps the parent's values (trace IDs,
request ID) but drops its deadline and cancellation. Then bound it with a fresh timeout.

Two canonical situations:

**1. Graceful shutdown.** By the time you drain, the run context is already cancelled — that's what
triggered shutdown. Deriving the shutdown context from it would make `Shutdown` return instantly:

```go
// keeps values, drops the already-fired cancellation, adds a fresh deadline
shutdownCtx, cancel := context.WithTimeout(context.WithoutCancel(ctx), drain)
defer cancel()
_ = srv.Shutdown(shutdownCtx)
```

**2. Fire-and-forget work after responding.** A request handler's `ctx` is cancelled the moment the
response is written or the client disconnects (see lifecycle below), so async work started with the
raw request ctx dies prematurely:

```go
func (h *Handler) create(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	order, err := h.svc.Create(ctx, /* … */)
	// … write the response to the client …

	// Audit must complete even after the client is gone, but keep its trace IDs.
	bg, cancel := context.WithTimeout(context.WithoutCancel(ctx), 5*time.Second)
	defer cancel()
	if err := h.audit.Record(bg, order.ID); err != nil {
		h.logger.ErrorContext(bg, "audit failed", "err", err)
	}
}
```

Caveat: a bare `go func(){…}()` for this is itself a lifecycle risk — it isn't tracked at shutdown
and can be killed mid-flight on process exit. For correctness-critical async work, hand it to a
**durable queue or a tracked worker pool** (an `errgroup` you `Wait` on), not a detached goroutine.

## The `net/http` request context lifecycle

`r.Context()` is **cancelled when** the client connection closes, the request is canceled, **or
`ServeHTTP` returns**. Consequences:

- Use it for everything that should stop when the client goes away (DB queries, downstream calls).
- Do **not** use it for work that must outlive the response — detach with `WithoutCancel` first.
- To tie request contexts to *server* lifecycle, set `http.Server.BaseContext`:

  ```go
  httpSrv := &http.Server{
      BaseContext: func(net.Listener) context.Context { return baseCtx },
  }
  ```

  `BaseContext` is the parent of every request context. Use it to inject base values. **Don't** make
  it the SIGTERM-cancelled context if you want a graceful drain — that would cancel all in-flight
  requests the instant a signal arrives, fighting `Shutdown`, which is designed to *let them
  finish* within the drain window. Tie `BaseContext` to a "hard stop" context, not the first signal.

## Context in middleware (request-scoped values)

Middleware is where request-scoped values get attached. The mechanics: build a child context with
the value, then swap it onto the request with `r.WithContext`.

**Use an unexported key type** so keys can't collide across packages and can't be read by code that
doesn't own them (string/built-in keys are a `staticcheck SA1029` smell):

```go
type ctxKey int

const (
	requestIDKey ctxKey = iota
	loggerKey
	principalKey
)

// expose typed accessors, never the key itself
func RequestID(ctx context.Context) string {
	id, _ := ctx.Value(requestIDKey).(string)
	return id
}

func RequestIDMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		id := r.Header.Get("X-Request-ID")
		if id == "" {
			id = uuid.NewString()
		}
		w.Header().Set("X-Request-ID", id)
		ctx := context.WithValue(r.Context(), requestIDKey, id) // derive from r.Context()
		next.ServeHTTP(w, r.WithContext(ctx))                   // swap it in for downstream
	})
}
```

Context accretes as the chain runs outer→inner, so order matters: put request-ID/logger/tracing
middleware *outside* anything that reads those values. A logger middleware typically reads the
request ID added upstream and attaches an enriched `*slog.Logger` to the context (see
`references/logging.md`).

**What belongs in context values:** request-scoped data that crosses API boundaries — request ID,
authenticated principal/tenant, trace span, the request logger. **What does not:** optional function
parameters, configuration, or dependencies (inject those through constructors). The stdlib guidance
is explicit: context values are for request-scoped data that transits APIs, not for passing
optional args.

### Per-request timeouts in middleware — a real gotcha

`http.TimeoutHandler` bounds the *response* (writes 503 after the limit) but **does not cancel the
handler's context** — the handler and its downstream calls keep running in the background. For a
deadline that actually propagates (so DB/HTTP calls observe `context.DeadlineExceeded` and stop),
derive a timeout context in middleware instead:

```go
func Timeout(d time.Duration) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			ctx, cancel := context.WithTimeout(r.Context(), d)
			defer cancel()
			next.ServeHTTP(w, r.WithContext(ctx)) // downstream calls now honor the deadline
		})
	}
}
```

Then map `context.DeadlineExceeded` → `504 Gateway Timeout` in your error handling (see
`references/errors-and-concurrency.md`). Use `http.TimeoutHandler` only when you also want a
guaranteed bounded response and accept that the abandoned work runs to completion.

## Modern helpers worth knowing

- **`context.WithoutCancel(parent)`** — copy that ignores parent cancellation; keeps values.
- **`context.WithCancelCause` / `context.Cause(ctx)`** — cancel with a specific error you can later
  retrieve, instead of a bare `context.Canceled`. Great for explaining *why* a request was aborted.
- **`context.AfterFunc(ctx, f)`** — run `f` in its own goroutine once `ctx` is done; returns a stop
  func. Clean way to wire cleanup without a manual `<-ctx.Done()` goroutine.
- **`context.WithTimeoutCause` / `WithDeadlineCause`** — timeouts that report a custom cause.

## Decision table

| Situation | Use |
|---|---|
| Top of process (`run`), independent worker root | `context.Background()` |
| Tests | `t.Context()` |
| Any call inside a request/operation | derive: thread `ctx`; add `WithTimeout`/`WithValue` as needed |
| Per-call timeout | `context.WithTimeout(ctx, d)` + `defer cancel()` |
| Attach request-scoped value (middleware) | `context.WithValue(r.Context(), key, v)` + `r.WithContext` |
| Work must outlive the request / shutdown, keep values | `context.WithTimeout(context.WithoutCancel(ctx), d)` |
| Context should be threaded but isn't yet | `context.TODO()` |

## Anti-patterns

- A stray `context.Background()`/`TODO()` mid-request — severs deadline + trace propagation.
- Missing `defer cancel()` after `WithTimeout`/`WithCancel` — leaks the timer/goroutine (`govet`
  `lostcancel` and `golangci-lint` catch many of these).
- Storing `ctx` in a struct field instead of passing it per call (`containedctx` lint).
- `string`/built-in context-value keys (collisions; `SA1029`).
- Smuggling dependencies or config through context values.
- Using `r.Context()` for fire-and-forget work that must survive the response.
