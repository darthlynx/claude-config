---
name: go-microservice
description: >-
  Best practices and idiomatic patterns for building production-grade Go microservices,
  following the Google Go Style Guide. Use this skill WHENEVER the user is starting a new Go
  service, scaffolding a Go project, or asks about Go service project layout, the cmd/ + internal/
  convention, graceful shutdown, structured logging with slog, configuration via environment
  variables, multi-stage Docker builds (UBI / distroless), testing (table-driven, testcontainers,
  synctest), observability (OpenTelemetry / Prometheus), error handling, concurrency (errgroup),
  HTTP/gRPC server setup, CockroachDB / pgx persistence, Kafka producers/consumers/admin, or
  application security (TLS, OIDC/JWT, input validation, secret handling) in Go. Also trigger for
  code review of Go services, "how should I structure my Go service", "make this Go service
  production-ready", or any request touching configuration, logging, testing, or deployment of a
  Go backend — even when the user never says the word "microservice".
---

# Go Microservice Development

A reference for building production-grade Go microservices: idiomatic, testable, observable,
and deployable. Default to Go's standard library; reach for dependencies only when they earn
their place.

**Target Go version: 1.24 or above.** Examples in this skill use `log/slog` (1.21+),
`context.WithoutCancel` (1.21+), `net/http` pattern routing (1.22+), and `t.Context()` (1.24+).
Pin the project's `go` directive in `go.mod` to `1.24` or later, and set CI's Go version from
the file (`setup-go` with `go-version-file: go.mod`) so they can never drift.

## How to use this skill

1. Read this file end to end — it carries the principles, the `main`/`run` wiring that ties
   everything together, the checklist, and the anti-patterns.
2. Pull in the relevant `references/*.md` file(s) for whatever the task touches (layout, config,
   logging, testing, Docker, observability, HTTP, errors/concurrency, tooling). Don't load all of
   them — load what the task needs.
3. Copy and adapt templates from `assets/` (Dockerfile, `.golangci.yml`, `Makefile`,
   `.dockerignore`) rather than writing them from scratch.

When scaffolding a brand-new service, work in this order: **layout → config → logging → the
`run()` skeleton → HTTP/server → tests → Dockerfile → CI/tooling**. Each later step assumes the
earlier ones exist.

## Core principles (non-negotiable)

- **Stdlib first.** `net/http` (method matching + path patterns are sufficient for most routing),
  `log/slog`, `errors`, `context`, `testing`. A framework or library must justify its cost.
  Vendor lock-in and transitive CVEs are real liabilities.
- **`context.Context` flows through everything.** First parameter, named `ctx`, never stored in a
  struct. Every blocking call (HTTP, DB, Kafka, gRPC) takes a context and respects cancellation.
  Derive from the `ctx` you're given; only mint a root (`context.Background()`) at an entry point.
  See `references/context.md` for the full root-vs-derived rules, `WithoutCancel`, and middleware.
  Derive from the context you're given by default; reach for a fresh root or `WithoutCancel` only at
  lifecycle boundaries — see `references/context.md`.
- **Accept interfaces, return structs.** Define interfaces at the *consumer*, keep them small.
  Inject dependencies through constructors; no global mutable state, no `init()` side effects.
- **Errors are values.** Wrap with `%w`, inspect with `errors.Is`/`errors.As`. Libraries never
  `panic` for ordinary failures and never call `os.Exit`. Only `main`/`run` decides the exit code.
- **Fail fast at startup, degrade gracefully at runtime.** Validate all config before serving
  traffic; once serving, prefer timeouts, retries, and shed-load over crashing.
- **Configuration comes from the environment** (12-factor); secrets are injected, never committed.
- **Everything is observable.** Structured logs (`slog`), metrics, and traces are part of "done,"
  not an afterthought.
- **The race detector is mandatory in CI** (`go test -race`). A passing test suite without `-race`
  proves little for concurrent code.

## Preferred dependencies (and why each earned its place)

"Stdlib first" doesn't mean "stdlib only." These are the dependencies this skill recommends, with
the *specific* problem each solves that the stdlib doesn't. Adding any other library should require
the same level of justification.

| Package | Used for | Why this one (not stdlib / not an alternative) |
|---|---|---|
| `golang.org/x/sync/errgroup` | Coordinated goroutines | Quasi-stdlib (`x/...`); standard pattern for cancel-on-first-error. No real competition. |
| `github.com/caarlos0/env/v11` | Env → typed config | Stdlib has no struct-tag env parser. Actively maintained; small surface; no reflection magic beyond what's necessary. |
| `github.com/jackc/pgx/v5` | Postgres / CockroachDB driver | Native protocol implementation, first-class types, real connection pool (`pgxpool`). `database/sql` + `lib/pq` is slower and `lib/pq` is unmaintained. |
| `github.com/cockroachdb/cockroach-go/v2/crdb/crdbpgx` | CRDB transaction retries | Cockroach requires application-level retry on 40001 serialization failures; this is the official, correct loop. |
| `github.com/google/uuid` | Request IDs | `crypto/rand` + manual formatting works, but `uuid.NewString()` is one line and universally recognized. |
| `github.com/coreos/go-oidc/v3` + `github.com/golang-jwt/jwt/v5` | OIDC / JWT verification | Writing your own JWT verifier is the canonical "looks easy, has subtle bugs" task. `go-oidc` handles JWKS caching and discovery. |
| `github.com/go-playground/validator/v10` | Struct-tag validation | Hand-rolled validation accumulates inconsistencies; declarative tags keep request DTOs honest. |
| `go.opentelemetry.io/otel` (+ `otelhttp`, `otelgrpc`, `otelpgx`) | Tracing | Vendor-neutral standard. Stdlib has no tracing. |
| `github.com/prometheus/client_golang` | Metrics | De facto standard for Prometheus exposition. OTel metrics SDK is fine if you want one pipeline; either is acceptable. |
| `github.com/pressly/goose` *or* `github.com/golang-migrate/migrate` | DB migrations | Pick one. Both are mature. Goose embeds nicely as a Go library; golang-migrate has more sources. |
| `github.com/testcontainers/testcontainers-go` | Integration tests against real deps | Real Postgres/Kafka beats mocks for adapter tests. Test-time only. |
| `go.uber.org/mock` | Generated mocks (when needed) | Maintained successor to `golang/mock`. Use only for large or call-order-sensitive interfaces; hand-written fakes win for small ones. |
| `go.uber.org/goleak` | Goroutine-leak detection in tests | Catches lifecycle bugs `-race` misses. Test-time only. |
| `github.com/google/go-cmp` | Diff in tests | `reflect.DeepEqual` gives no useful failure message. Test-time only. |
| `github.com/sony/gobreaker` | Circuit breaker | Tiny, focused, no surprises. Add only when a flaky dependency justifies it. |
| `github.com/cenkalti/backoff/v5` | Retry with backoff | Hand-rolled exp+jitter is fine for one call site; this earns its place when you have several. |
| `github.com/confluentinc/confluent-kafka-go/v2` | Kafka **admin** API (topics, ACLs, configs) | Wraps librdkafka's `AdminClient`, the most battle-tested admin surface. Pull in for management/ops binaries even when producers and consumers run on a pure-Go client. Drags in CGO — see `references/docker.md` for the CGO image recipe. |
| `github.com/twmb/franz-go` *or* `github.com/segmentio/kafka-go` | Kafka **producer / consumer** | Pure-Go, no CGO; keeps your runtime image on UBI micro / distroless. franz-go has the most complete protocol support; segmentio's API is simpler. Pick one per service. |

Things this skill **does not** recommend reaching for by default:

- HTTP frameworks (`gin`, `echo`, `fiber`): stdlib `net/http` covers most needs; `chi` is the right
  step up when you actually need grouped routes + middleware ecosystem while staying close to stdlib.
- Logging libraries (`zap`, `zerolog`): `log/slog` is fast enough for almost everything.
- DI containers (`wire`, `fx`): hand-wire in `newApp(ctx, cfg, logger) (*app, error)`. A 30-line
  composition root is clearer than a generated graph.
- ORMs (`gorm`, `ent`): pgx + small repository functions beats an ORM for service-shaped workloads.
  Reach for `sqlc` if you want generated type-safe queries without an ORM's runtime overhead.

## Project layout

For *layout* there's no official Go standard beyond `cmd/` and `internal/` — use the
compiler-enforced minimum, not the unofficial `golang-standards/project-layout`:

```
myservice/
├── cmd/
│   └── myservice/
│       └── main.go          # thin: just calls run()
├── internal/                # private to this module; compiler-enforced
│   ├── config/              # env loading + validation
│   ├── server/              # HTTP/gRPC wiring, middleware, routes
│   ├── handler/             # transport-layer handlers
│   ├── service/             # business logic (no transport, no SQL drivers)
│   └── store/               # persistence (pgx, repositories)
├── api/                     # OpenAPI/proto contracts (optional)
├── migrations/              # goose SQL migrations
├── helm-chart/              # Helm chart (team convention: charts live here)
├── Dockerfile               # at repo root so `docker build .` works with no -f
├── Makefile
├── go.mod
└── go.sum
```

Rules of thumb: keep `cmd/<name>/main.go` thin; put everything importable only by this service
under `internal/`; **avoid `pkg/`** unless you publish a library for external consumers. Keep the
`Dockerfile` at the repo root (build context = module root) and Helm charts in `helm-chart/`. See
`references/project-layout.md` for *why* this layout (vs `golang-standards/project-layout`), the
package-naming rules, the deployment-artifact conventions, and the clean-boundary dependency
direction.

## The `main` / `run` skeleton

`main` is a thin shell; all real logic lives in a testable `run` function that takes its
dependencies as parameters (so tests can inject fakes and a cancelable context). This is the
single most important pattern in the skill — everything else hangs off it.

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/acme/myservice/internal/config"
	"github.com/acme/myservice/internal/server"
)

func main() {
	if err := run(context.Background(), os.Stdout); err != nil {
		fmt.Fprintf(os.Stderr, "fatal: %v\n", err)
		os.Exit(1)
	}
}

// run takes its writer as io.Writer so tests can inject *bytes.Buffer.
func run(ctx context.Context, stdout io.Writer) error {
	// Cancel ctx on SIGINT/SIGTERM so every context-aware call unwinds cleanly.
	ctx, stop := signal.NotifyContext(ctx, syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	cfg, err := config.Load() // parse env, validate, fail fast
	if err != nil {
		return fmt.Errorf("load config: %w", err)
	}

	logger := newLogger(cfg.Log, stdout) // slog; see references/logging.md
	logger.InfoContext(ctx, "starting", "version", buildVersion, "env", cfg.Env)

	// Construct dependencies (db pool, clients, services) here and inject them.
	srv, err := server.New(cfg, logger /*, deps... */)
	if err != nil {
		return fmt.Errorf("build server: %w", err)
	}

	httpSrv := &http.Server{
		Addr:              cfg.HTTP.Addr,
		Handler:           srv,
		ReadHeaderTimeout: 5 * time.Second, // gosec G112; never leave unset
		ReadTimeout:       cfg.HTTP.ReadTimeout,
		WriteTimeout:      cfg.HTTP.WriteTimeout,
		IdleTimeout:       cfg.HTTP.IdleTimeout,
	}

	// Serve in the background; surface listen errors through a channel.
	serveErr := make(chan error, 1)
	go func() {
		logger.InfoContext(ctx, "listening", "addr", httpSrv.Addr)
		if err := httpSrv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			serveErr <- err
		}
	}()

	// Block until a signal cancels ctx or the server fails to listen.
	select {
	case err := <-serveErr:
		return fmt.Errorf("listen: %w", err)
	case <-ctx.Done():
		stop() // restore default handling so a second SIGINT/SIGTERM force-quits a stuck drain
		logger.InfoContext(ctx, "shutdown signal received")
	}

	// Graceful shutdown with a bounded deadline.
	// NOTE: ctx is ALREADY cancelled here (that's what released <-ctx.Done()), so the shutdown
	// context must NOT derive its cancellation from it — otherwise Shutdown returns instantly and
	// nothing drains. context.WithoutCancel keeps ctx's values (trace IDs) but drops its
	// cancellation; it's equivalent to context.Background() when ctx carries no values.
	shutdownCtx, cancel := context.WithTimeout(context.WithoutCancel(ctx), cfg.HTTP.ShutdownTimeout)
	defer cancel()
	if err := httpSrv.Shutdown(shutdownCtx); err != nil {
		return fmt.Errorf("graceful shutdown: %w", err)
	}
	logger.InfoContext(ctx, "stopped cleanly")
	return nil
}
```

`buildVersion` and `newLogger` are sketched here; wire them per `references/tooling.md` (ldflags)
and `references/logging.md`. For multiple long-lived goroutines (consumers, workers) coordinate
their lifecycles with `errgroup.WithContext` — see `references/errors-and-concurrency.md`.

### Where `run` lives, and keeping it testable

- **Placement:** keep `run` in `package main`. It stays testable via a `main_test.go` in the same
  package — the "you can't test `main`" belief is false; what's true is that `package main` can
  never be *imported*. Hoist the logic into `internal/app` (`func Run(ctx, …) error`) only when you
  need to share bootstrap across multiple `cmd/` binaries or want external (`package app_test`)
  tests. For a single service, `package main` is correct.
- **Don't fragment the wiring.** `run` is the composition root; splitting it into `setupDB()`/
  `setupServer()` helpers adds indirection without test value — assembly code isn't where bugs
  hide. Testability comes from the logic living in `config`/`service`/`store`/`server` (each
  unit-tested with fakes), not from chopping up the coordinator.
- **The one valuable split** is *build* vs *serve*: a `newApp(ctx, cfg, logger) (*app, error)` that
  wires dependencies (testable with fakes) and an `(*app).serve(ctx) error` that runs the lifecycle
  via `errgroup` and drains on shutdown. `run` then collapses to: load config → logger → `newApp`
  → `serve`. Validate `run`/`serve` end to end with an **integration smoke test** (boot → probe
  `/readyz` → send SIGTERM → assert clean drain), not isolated unit tests. The `serve` pattern is
  in `references/errors-and-concurrency.md`.
- **Shutdown context (common gotcha):** never derive the shutdown context from the incoming `ctx`
  — by shutdown time it's already cancelled, so `Shutdown` would return instantly and skip the
  drain. Use `context.WithTimeout(context.WithoutCancel(ctx), d)` (keeps values, drops the fired
  cancellation), and call `stop()` once before shutting down so a second signal can force-quit a
  hung drain.

## Quick checklist (use for new services and reviews)

- [ ] `cmd/<name>/main.go` is thin; logic in a testable `run(ctx, ...)`.
- [ ] All config from env, parsed into a struct, validated, fails fast on bad/missing values.
- [ ] `slog` with a JSON handler in prod; logger injected, not global; trace/request IDs attached.
- [ ] `context.Context` is the first arg everywhere; every external call has a timeout.
- [ ] HTTP server sets `ReadHeaderTimeout` + read/write/idle timeouts.
- [ ] Graceful shutdown on SIGTERM with a bounded deadline; `/readyz` flips to 503 *before* `Shutdown`.
- [ ] Liveness endpoint checks **only** "process up" — no dep checks (or you'll cause cascading restarts).
  Readiness endpoint checks real dependencies.
- [ ] Request-ID middleware generates/propagates `X-Request-ID` and attaches it to the logger.
- [ ] Recoverer middleware is the outermost wrap (catches panics from everything inside).
- [ ] TLS handled: in-app via `crypto/tls` with `MinVersion: TLS12`, **or** terminated upstream
  with trusted `X-Forwarded-*` headers stripped at the edge.
- [ ] AuthN/AuthZ wired where applicable (OIDC/JWT or mTLS); authorization enforced in the service
  layer, not just handlers.
- [ ] Input validation: `MaxBytesReader` on bodies, struct-tag validation (`validator/v10`) on DTOs.
- [ ] Errors wrapped with `%w`; sentinel/typed errors where callers must branch; no panics in libs.
- [ ] Table-driven tests with `t.Parallel()`; integration tests via `testcontainers-go`.
- [ ] `go test -race ./...` and `go vet ./...` pass in CI.
- [ ] `golangci-lint` and `govulncheck` run in CI.
- [ ] CI image pipeline: scan (`trivy`), SBOM (`syft`), sign + attest (`cosign`) on `main`/tags.
- [ ] Multi-stage Dockerfile → UBI 9 micro (or distroless), non-root UID, static where possible, `.dockerignore`.
- [ ] Metrics + traces exported (Prometheus / OpenTelemetry); OTel tracer `Shutdown` wired into the drain path.
- [ ] Build info (version, commit, date) injected via `-ldflags`, logged at startup, exposed at `/version`.

## Anti-patterns to reject on sight

- Global `*sql.DB`, global logger, config read ad hoc via `os.Getenv` scattered across packages.
- `panic`/`log.Fatal` deep in library or handler code (only `main`/`run` exits).
- `interface{}`/`any` as a substitute for designing types; "stuttering" interfaces.
- Naked `go func()` with no lifecycle, no error handling, no context — goroutine leaks.
- Swallowing errors (`_ = doThing()`) or logging-and-returning the same error (double reporting).
- `util`/`common`/`models` god-packages; deep import cycles "solved" by dumping into one package.
- Returning `nil, nil`; mutating shared maps/slices without synchronization.
- `time.Sleep` in tests; sleeping instead of synchronizing.
- HTTP servers/clients with no timeouts (the default client waits forever).

## Reference files

Load only what the task needs:

| File | When to read |
|---|---|
| `references/project-layout.md` | Structuring a service; package boundaries; clean dependency rules |
| `references/configuration.md` | Env config structs, validation, secrets, feature flags |
| `references/context.md` | Root vs derived contexts, `WithoutCancel`, request lifecycle, middleware values, timeouts |
| `references/logging.md` | `slog` setup, handlers, context propagation, redaction |
| `references/testing.md` | Table-driven tests, fakes/mocks, `httptest`, testcontainers, golden files |
| `references/docker.md` | Multi-stage builds, UBI 9 micro/minimal, the CGO/librdkafka case |
| `references/observability.md` | OpenTelemetry tracing, Prometheus metrics, RED/USE method |
| `references/http-server.md` | Routing, timeouts, middleware, health probes, retries, rate limiting |
| `references/grpc.md` | gRPC server/client, interceptors, graceful shutdown, error mapping, bufconn tests |
| `references/database.md` | CockroachDB + pgx pool sizing, retries, transactions, migrations |
| `references/security.md` | TLS, OIDC/JWT, authorization, input validation, CORS/CSRF, secret hygiene |
| `references/errors-and-concurrency.md` | Error wrapping, `errgroup`, worker pools, cancellation |
| `references/tooling.md` | `golangci-lint`, `govulncheck`, Makefile, CI, build-info ldflags |

## Style guide alignment

This skill follows the Google Go Style Guide hierarchy: **clarity > simplicity > concision >
consistency**, in that order. When two patterns conflict, prefer the one that makes the code
clearest to a reader who is *not* the author. Run `gofmt`/`goimports` (non-negotiable) and let
`golangci-lint` enforce the rest.
