# Structured Logging with `slog`

`log/slog` is the standard. Don't pull in zap/zerolog for a new service unless you have a measured
hot-path allocation problem; `slog` with the JSON handler is fast enough for almost everything and
avoids a dependency.

## Setup

```go
package main

import (
	"io"
	"log/slog"
)

func newLogger(cfg config.Log, w io.Writer) *slog.Logger {
	level := parseLevel(cfg.Level) // map "info" → slog.LevelInfo, etc.

	opts := &slog.HandlerOptions{
		Level:     level,
		AddSource: level <= slog.LevelDebug, // file:line only when debugging; it costs
		ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
			// Normalize/redact here (e.g. rename "time"→"ts", drop empty).
			return a
		},
	}

	var h slog.Handler
	switch cfg.Format {
	case "text":
		h = slog.NewTextHandler(w, opts) // human-readable for local dev
	default:
		h = slog.NewJSONHandler(w, opts) // structured for prod log pipelines
	}
	return slog.New(h)
}
```

## Inject, don't globalize

- Pass `*slog.Logger` into constructors; store it on the struct that needs it. Treat it like any
  other dependency.
- Avoid `slog.SetDefault` + package-level `slog.Info(...)` as the *primary* path — it's a global
  that hurts testability and makes per-request context attachment awkward. Setting the default once
  in `run()` is fine as a fallback for code you don't control, but route your own code through the
  injected logger.

## Context-aware logging

Always use the `*Context` methods (`InfoContext`, `ErrorContext`, …) on request paths. This lets a
custom handler enrich records from context (trace IDs, request IDs) without threading attributes
manually:

```go
logger.InfoContext(ctx, "order created", "order_id", id, "amount_cents", amt)
```

Attach a request-scoped logger in middleware so handlers inherit common fields. The snippet
below lives in the same `internal/middleware` package as the `ctxKey`/`loggerKey` definitions and
the `RequestID` accessor in `references/context.md` — that's why it can reference the unexported
key directly:

```go
package middleware

// ctxKey, loggerKey, RequestID are defined alongside the other request-scoped keys
// in this package — see references/context.md.

func WithLogger(base *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			reqID := RequestID(r.Context())
			l := base.With("request_id", reqID, "method", r.Method, "path", r.URL.Path)
			ctx := context.WithValue(r.Context(), loggerKey, l)
			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}
}

// Logger returns the request-scoped logger attached by WithLogger, or fallback if none.
// Handlers call middleware.Logger(r.Context(), s.log) so tests and unwrapped calls still work.
func Logger(ctx context.Context, fallback *slog.Logger) *slog.Logger {
	if l, ok := ctx.Value(loggerKey).(*slog.Logger); ok {
		return l
	}
	return fallback
}
```

For trace correlation, prefer a small custom `slog.Handler` that reads the OTel span context out
of `ctx` in `Handle` and adds `trace_id`/`span_id` — that way *every* log line within a span is
correlated without per-call boilerplate. See `references/observability.md`.

## Attributes: do's and don'ts

- Use typed attrs on hot paths to skip the `any` boxing: `slog.Int`, `slog.String`,
  `slog.Duration`, `slog.Time`.
- Group related fields: `slog.Group("http", slog.Int("status", code), slog.Duration("took", d))`.
- **Redaction**: implement `LogValue() slog.Value` on secret types so they self-redact wherever
  logged (see `references/configuration.md`).
- Log **errors** with a consistent key: `logger.ErrorContext(ctx, "msg", "err", err)`. Log an error
  **once**, at the boundary where you handle it — don't log-and-return (double reporting).
- Don't log full request/response bodies, tokens, PII, or DSNs. Don't log inside tight loops.
- Pick levels deliberately: `Debug` for dev diagnostics, `Info` for lifecycle/business events,
  `Warn` for recoverable anomalies, `Error` for failures needing attention. Reserve high cardinality
  fields (IDs) for attrs, not the message string.

## Testing logs

The `io.Writer` parameter in `newLogger` is what makes logs testable: pass a `*bytes.Buffer`,
capture JSON lines, decode and assert on fields. Don't grep for substrings — that breaks the moment
someone reorders attributes.

```go
func TestService_LogsOrderCreated(t *testing.T) {
	var buf bytes.Buffer
	logger := slog.New(slog.NewJSONHandler(&buf, &slog.HandlerOptions{Level: slog.LevelInfo}))

	svc := NewService(logger /*, deps… */)
	_, _ = svc.CreateOrder(t.Context(), Order{ID: 42, Amount: 1000})

	var rec struct {
		Level   string `json:"level"`
		Msg     string `json:"msg"`
		OrderID int64  `json:"order_id"`
	}
	if err := json.Unmarshal(bytes.TrimSpace(buf.Bytes()), &rec); err != nil {
		t.Fatalf("decode log line: %v", err)
	}
	if rec.Msg != "order created" || rec.OrderID != 42 {
		t.Errorf("unexpected log: %+v", rec)
	}
}
```

For multi-line output, scan with `bufio.Scanner` and assert on the line you care about. For tests
that should *not* produce logs at a given level, set the handler level above what the code emits
and assert `buf.Len() == 0`.

## Levels at runtime

Use a `slog.LevelVar` if you want to change verbosity without redeploying (wire it to a signal or
admin endpoint):

```go
var lvl slog.LevelVar
lvl.Set(slog.LevelInfo)
h := slog.NewJSONHandler(w, &slog.HandlerOptions{Level: &lvl})
// later: lvl.Set(slog.LevelDebug)
```
