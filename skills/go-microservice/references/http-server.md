# HTTP Server, Middleware & Resilience

## Routing — stdlib first

`net/http.ServeMux` supports method matching and path wildcards, which covers most routing needs
without a framework:

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /healthz", healthHandler)
mux.HandleFunc("GET /users/{id}", getUserHandler) // r.PathValue("id")
mux.HandleFunc("POST /users", createUserHandler)
```

Reach for `chi` if you want grouped routes + a mature middleware ecosystem while staying close to
stdlib (`chi` handlers are plain `http.Handler`). Use `echo`/`gin` only if the team already
standardizes on them — they pull in more surface area and diverge from stdlib idioms.

## Server timeouts (always set them)

The zero-value `http.Server` has **no timeouts** — a slow client can hold a connection forever.
Minimum:

```go
&http.Server{
	Addr:              cfg.HTTP.Addr,
	Handler:           handler,
	ReadHeaderTimeout: 5 * time.Second,  // gosec G112 flags a missing one
	ReadTimeout:       10 * time.Second,
	WriteTimeout:      15 * time.Second,
	IdleTimeout:       60 * time.Second,
}
```

For long-running handlers, set per-route deadlines with `http.TimeoutHandler` or
`context.WithTimeout` inside the handler rather than a giant global `WriteTimeout`.

The **default `http.Client` also has no timeout** — never use `http.DefaultClient` for outbound
calls. Construct one with a `Timeout` and a tuned `Transport`, and pass a per-call `ctx` with its
own deadline.

### HTTP/2 and h2c

`net/http` speaks HTTP/2 automatically when TLS is enabled — `http.Server` upgrades after the TLS
handshake with no extra wiring. If TLS terminates upstream (ingress, mesh) and your service
receives plaintext HTTP/2 ("h2c"), wrap the handler with `golang.org/x/net/http2/h2c`:

```go
import (
	"golang.org/x/net/http2"
	"golang.org/x/net/http2/h2c"
)

httpSrv := &http.Server{
	Addr:              cfg.HTTP.Addr,
	Handler:           h2c.NewHandler(handler, &http2.Server{}),
	ReadHeaderTimeout: 5 * time.Second,
	// ... rest of timeouts
}
```

Without `h2c`, a plaintext HTTP/2 upgrade attempt downgrades to HTTP/1.1, which silently breaks
gRPC-over-h2c and some service-mesh paths.

## Middleware

Standard `func(http.Handler) http.Handler` chain. Order matters — outermost runs first:

```
recoverer → requestID → logger → tracing(otelhttp) → metrics → auth → rateLimit → handler
```

Essential middleware:
- **Recoverer**: catch panics, log with stack, return 500 — so one bad handler can't kill the
  process. (This is the *one* place panics are acceptable to recover from.)
- **Request ID**: generate/propagate `X-Request-ID`; attach to ctx + logger.
- **Logging**: one structured line per request (method, route pattern, status, duration, request_id).
- **Tracing/metrics**: `otelhttp` + a Prometheus middleware (see `references/observability.md`).
- **Recovery writes the response only if nothing has been written yet** — wrap the
  `ResponseWriter` to track status/bytes.

```go
// Recoverer must be the outermost middleware so it catches panics from everything inside,
// including other middleware. Use runtime/debug.Stack() for the trace.
func Recoverer(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			defer func() {
				rec := recover()
				if rec == nil {
					return
				}
				if rec == http.ErrAbortHandler {
					panic(rec) // re-raise: the server's own signal for client-cancelled writes
				}
				logger.ErrorContext(r.Context(), "panic in handler",
					"panic", rec,
					"stack", string(debug.Stack()),
					"method", r.Method, "path", r.URL.Path,
				)
				// Only write a response if the handler hasn't already. Wrap ResponseWriter
				// upstream to track status; here we best-effort with a status check.
				w.WriteHeader(http.StatusInternalServerError)
			}()
			next.ServeHTTP(w, r)
		})
	}
}
```

`http.ErrAbortHandler` is the stdlib's signal that the handler aborted intentionally (e.g. the
client disconnected mid-write). The server has its own logic for that; swallowing it would mask
real disconnects in your logs.

## Health & readiness endpoints

```go
// ready is flipped to false at shutdown so /readyz starts failing before Shutdown begins.
var ready atomic.Bool
ready.Store(true)

mux.HandleFunc("GET /livez", func(w http.ResponseWriter, _ *http.Request) {
	w.WriteHeader(http.StatusOK) // alive = process up; no dependency checks
})

mux.HandleFunc("GET /readyz", func(w http.ResponseWriter, r *http.Request) {
	if !ready.Load() {
		http.Error(w, "draining", http.StatusServiceUnavailable)
		return
	}
	ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
	defer cancel()
	if err := deps.Ping(ctx); err != nil { // DB, Kafka, etc.
		http.Error(w, "not ready", http.StatusServiceUnavailable)
		return
	}
	w.WriteHeader(http.StatusOK)
})
```

During graceful shutdown, flip `/readyz` to unhealthy **before** calling `srv.Shutdown`, then sleep
for the readiness probe period so k8s removes the pod from the Service before in-flight requests are
cut. The wiring sits in `run()` (or the `serve` half of `newApp/serve`):

```go
case <-ctx.Done():
	ready.Store(false)                  // /readyz now returns 503
	time.Sleep(cfg.HTTP.DrainDelay)     // ~ 2 × readiness probe period; lets k8s notice
	shutdownCtx, cancel := context.WithTimeout(context.WithoutCancel(ctx), cfg.HTTP.ShutdownTimeout)
	defer cancel()
	return httpSrv.Shutdown(shutdownCtx) // bounded graceful drain
```

`DrainDelay` belongs in `config.HTTP` (typical value 10–15s) so it can be tuned per environment.

## Resilience patterns for downstream calls

A microservice is only as reliable as how it handles *other* services failing.

- **Timeouts on every call** — already covered; this is the single highest-leverage practice.
- **Retries with backoff + jitter**, only for **idempotent** operations and **retryable** errors
  (timeouts, 503, connection resets — *not* 400/409). Cap attempts and total time. Use
  `cenkalti/backoff` or a small hand-rolled exponential-with-jitter loop driven by `ctx`.
- **Circuit breaker** (`sony/gobreaker`) around a flaky dependency: stop hammering a downed service,
  fail fast, and probe for recovery. Pair with a sensible fallback (cached value, degraded
  response) where the product allows.
- **Rate limiting**: protect yourself and downstreams. `golang.org/x/time/rate` (`rate.Limiter`)
  for in-process limits; a middleware keyed by client/IP for ingress; a distributed limiter (Redis)
  if you need a global budget across replicas.
- **Bulkheads**: bound concurrency to a dependency (a buffered semaphore / worker pool) so a slow
  backend can't exhaust all your goroutines/connections.
- **Load shedding**: when overloaded, return 503 early rather than queueing unboundedly. A
  concurrency limiter at the edge beats unbounded goroutine growth.

Compose these deliberately — retries *behind* a circuit breaker, all *under* an overall deadline:
`ctx (deadline) → breaker → retry(backoff) → timeout per attempt → call`.

## Request/response hygiene

- Validate and bound input: `http.MaxBytesReader` on bodies; reject oversized payloads.
- Decode JSON with `json.Decoder` + `DisallowUnknownFields()` when strictness matters.
- Return consistent error bodies (a small `{ "error": { "code", "message" } }` shape); map domain
  errors to status codes in one place (see `references/errors-and-concurrency.md`).
- Set `Content-Type` explicitly; don't rely on sniffing.
