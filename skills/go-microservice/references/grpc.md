# gRPC Server & Client

Reach for gRPC for service-to-service traffic where you want strong contracts, streaming, and
binary-efficient wire format. For browser-facing APIs, stick to HTTP/JSON (see
`references/http-server.md`) or add a gRPC-Gateway/Connect adapter.

## Server skeleton

```go
package server

import (
	"context"
	"crypto/tls"
	"net"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"
	"google.golang.org/grpc/health"
	healthpb "google.golang.org/grpc/health/grpc_health_v1"
	"google.golang.org/grpc/keepalive"
	"google.golang.org/grpc/reflection"

	"go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
)

func NewGRPC(cfg config.GRPC, logger *slog.Logger, tlsCfg *tls.Config) (*grpc.Server, error) {
	opts := []grpc.ServerOption{
		grpc.StatsHandler(otelgrpc.NewServerHandler()), // tracing + metrics; replaces deprecated interceptor
		grpc.ChainUnaryInterceptor(
			recoveryUnary(logger),
			loggingUnary(logger),
			authUnary(/* verifier */),
		),
		grpc.ChainStreamInterceptor(
			recoveryStream(logger),
			loggingStream(logger),
			authStream(/* verifier */),
		),
		grpc.KeepaliveParams(keepalive.ServerParameters{
			MaxConnectionIdle: 5 * time.Minute,
			Time:              30 * time.Second, // server pings idle connections
			Timeout:           10 * time.Second,
		}),
		grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
			MinTime:             10 * time.Second,
			PermitWithoutStream: true,
		}),
		grpc.MaxRecvMsgSize(8 * 1024 * 1024),
	}
	if tlsCfg != nil {
		opts = append(opts, grpc.Creds(credentials.NewTLS(tlsCfg)))
	}

	srv := grpc.NewServer(opts...)
	healthpb.RegisterHealthServer(srv, health.NewServer())
	if cfg.EnableReflection { // dev/staging only — leaks your API surface
		reflection.Register(srv)
	}
	return srv, nil
}
```

## Graceful shutdown — `GracefulStop` with a bounded deadline

`GracefulStop()` waits forever for in-flight RPCs. Always bound it; fall back to `Stop()` on timeout
so a stuck stream can't pin the process indefinitely:

```go
func runGRPC(ctx context.Context, srv *grpc.Server, lis net.Listener, drain time.Duration) error {
	errc := make(chan error, 1)
	go func() { errc <- srv.Serve(lis) }() // returns nil on GracefulStop

	select {
	case err := <-errc:
		return fmt.Errorf("grpc serve: %w", err)
	case <-ctx.Done():
		done := make(chan struct{})
		go func() { srv.GracefulStop(); close(done) }()
		select {
		case <-done:
			return nil
		case <-time.After(drain):
			srv.Stop() // force-cancel pending RPCs
			return fmt.Errorf("grpc shutdown exceeded %s, forced stop", drain)
		}
	}
}
```

## Interceptors — the pattern

Keep each interceptor small and single-purpose. Wire them in the same outer-to-inner order as HTTP
middleware: `recovery → logging → tracing → auth → handler`.

```go
func recoveryUnary(logger *slog.Logger) grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp any, err error) {
		defer func() {
			if r := recover(); r != nil {
				logger.ErrorContext(ctx, "grpc panic", "method", info.FullMethod, "panic", r)
				err = status.Error(codes.Internal, "internal error")
			}
		}()
		return handler(ctx, req)
	}
}
```

Tracing/metrics: prefer `otelgrpc.NewServerHandler()` (stats handler) over the deprecated unary/stream
interceptors — it captures both directions plus payload size in one place.

## Error mapping — `codes` + `status.WithDetails`

Map domain errors to gRPC status codes in one place, the same way `httpStatus(err) int` works for
HTTP. Use rich detail messages for structured errors (validation, retry hints):

```go
import (
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	errdetails "google.golang.org/genproto/googleapis/rpc/errdetails"
)

func toStatus(err error) error {
	var ve *ValidationError
	switch {
	case errors.Is(err, store.ErrNotFound):
		return status.Error(codes.NotFound, "not found")
	case errors.As(err, &ve):
		st := status.New(codes.InvalidArgument, ve.Error())
		st, _ = st.WithDetails(&errdetails.BadRequest_FieldViolation{Field: ve.Field, Description: ve.Reason})
		return st.Err()
	default:
		return status.Error(codes.Internal, "internal error") // don't leak err.Error() to clients
	}
}
```

Common code choices: `InvalidArgument` (bad input), `NotFound`, `AlreadyExists`, `PermissionDenied`
(authn'd but not allowed), `Unauthenticated` (no/invalid credentials), `FailedPrecondition` (state
violation), `Aborted` (concurrent update), `ResourceExhausted` (rate limit / quota), `Unavailable`
(retryable). Avoid `Unknown` — it usually means you didn't bother mapping.

## Health & reflection

- Register `grpc.health.v1.Health` and have your `/readyz` HTTP probe set it via `health.Server.SetServingStatus`.
  k8s `grpc-health-probe` (or native gRPC probes in modern kubelet) reads this.
- Reflection is great for dev/staging (`grpcurl` works without `.proto` files). **Disable in prod**
  (`cfg.EnableReflection=false`) — it lets anyone enumerate your API.

## Client side

```go
conn, err := grpc.NewClient(addr,
	grpc.WithTransportCredentials(credentials.NewTLS(tlsCfg)),
	grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
	grpc.WithKeepaliveParams(keepalive.ClientParameters{
		Time:                30 * time.Second,
		Timeout:             10 * time.Second,
		PermitWithoutStream: true,
	}),
	grpc.WithDefaultCallOptions(grpc.MaxCallRecvMsgSize(8*1024*1024)),
	grpc.WithDefaultServiceConfig(`{
		"methodConfig": [{
			"name": [{}],
			"retryPolicy": {
				"MaxAttempts": 4,
				"InitialBackoff": "0.1s", "MaxBackoff": "2s", "BackoffMultiplier": 2.0,
				"RetryableStatusCodes": ["UNAVAILABLE"]
			}
		}]
	}`),
)
```

- Use `grpc.NewClient` (lazy connect), not the deprecated `grpc.Dial` / `WithBlock`.
- **One `*grpc.ClientConn` per destination**, shared across RPCs — connections multiplex streams.
  Re-creating per call defeats keepalive and connection reuse.
- **Per-call deadlines**: `ctx, cancel := context.WithTimeout(ctx, 2*time.Second); defer cancel()`.
  Without a deadline, a hung server pins your goroutine forever.
- Retries: prefer the built-in service-config retry policy (above) over hand-rolled loops — it
  composes correctly with hedging and the underlying transport. Only retry **idempotent** methods.

## Testing — `bufconn`

In-process listener for fast, deterministic tests without a real socket:

```go
import "google.golang.org/grpc/test/bufconn"

lis := bufconn.Listen(1 << 20)
srv := grpc.NewServer()
pb.RegisterFooServer(srv, &fooServer{})
go srv.Serve(lis)
t.Cleanup(srv.Stop)

conn, _ := grpc.NewClient("passthrough://bufnet",
	grpc.WithContextDialer(func(ctx context.Context, _ string) (net.Conn, error) {
		return lis.DialContext(ctx)
	}),
	grpc.WithTransportCredentials(insecure.NewCredentials()),
)
client := pb.NewFooClient(conn)
```

For end-to-end gRPC tests against a real server (TLS, interceptors, real network), use
`net.Listen("tcp", "127.0.0.1:0")` and dial the resolved address.

## Anti-patterns

- Reflection enabled in production (API enumeration).
- `grpc.WithBlock()` / `grpc.Dial` — both deprecated; lazy `NewClient` is the modern form.
- One `ClientConn` per call (no pooling/multiplexing benefit; exhausts FDs under load).
- No deadlines on client calls.
- `status.Errorf(codes.Internal, err.Error())` — leaks internal error strings to clients.
- `GracefulStop()` with no time bound — a stuck stream blocks shutdown indefinitely.
- Mixing very large messages (>4MB default) without explicit `MaxRecvMsgSize` — fails opaquely.
- Returning `codes.Unknown` — almost always means the error wasn't mapped on purpose.
