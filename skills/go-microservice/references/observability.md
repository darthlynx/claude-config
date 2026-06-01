# Observability

Three pillars: **logs** (see `references/logging.md`), **metrics**, **traces**. For a service to be
"done," all three must be wired and correlated.

## Traces — OpenTelemetry

OpenTelemetry (OTel) is the vendor-neutral standard; export to whatever backend you run (Tempo,
Jaeger, Honeycomb, Datadog) via OTLP.

```go
// env is the deployment environment (dev|staging|prod) from the top-level Config.
// We take it as a separate argument because cfg.OTel intentionally doesn't duplicate it.
func initTracer(ctx context.Context, cfg config.OTel, env string) (func(context.Context) error, error) {
	exp, err := otlptracegrpc.New(ctx) // reads OTEL_EXPORTER_OTLP_ENDPOINT etc. from env
	if err != nil {
		return nil, fmt.Errorf("otlp exporter: %w", err)
	}
	res, err := resource.New(ctx,
		resource.WithAttributes(
			semconv.ServiceName(cfg.ServiceName),
			semconv.ServiceVersion(buildVersion),
			semconv.DeploymentEnvironment(env),
		),
	)
	if err != nil {
		return nil, fmt.Errorf("otel resource: %w", err)
	}
	tp := trace.NewTracerProvider(
		trace.WithBatcher(exp),
		trace.WithResource(res),
		trace.WithSampler(trace.ParentBased(trace.TraceIDRatioBased(cfg.SampleRatio))),
	)
	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
		propagation.TraceContext{}, propagation.Baggage{}))
	return tp.Shutdown, nil // call during graceful shutdown
}
```

- **Instrument inbound HTTP** with `otelhttp.NewHandler(mux, "server")` and **outbound** with
  `otelhttp.NewTransport`. gRPC has `otelgrpc`. DB: `pgx` has OTel tracers; or wrap calls in spans.
- **Propagate context** end to end — that's what stitches spans across services. This is *why*
  `ctx` must be the first arg everywhere.
- **Correlate logs with traces**: a custom `slog.Handler` that reads `trace.SpanContextFromContext(ctx)`
  and adds `trace_id`/`span_id` to every record makes logs jump-to-trace in your backend.
- Sample (don't trace 100% in prod); use `ParentBased` so a sampled request stays sampled across
  hops.

## Metrics — Prometheus (or OTel metrics)

Expose `/metrics` with `promhttp.Handler()`, or use the OTel metrics SDK with a Prometheus
exporter if you want one pipeline. The four signals worth instrumenting first:

- **RED** for request-driven services: **R**ate, **E**rrors, **D**uration — per endpoint.
- **USE** for resources (DB pools, queues): **U**tilization, **S**aturation, **E**rrors.

```go
var (
	httpReqs = prometheus.NewCounterVec(
		prometheus.CounterOpts{Name: "http_requests_total", Help: "HTTP requests."},
		[]string{"method", "route", "status"},
	)
	httpDur = prometheus.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "http_request_duration_seconds",
			Buckets: prometheus.DefBuckets,
		},
		[]string{"method", "route"},
	)
)
```

- Label by **bounded** dimensions only (`route` = the *pattern* `/users/{id}`, never the raw path
  with the ID). High-cardinality labels (user IDs, full URLs) will blow up your TSDB.
- Histograms for latency (so you get p50/p95/p99), counters for events, gauges for current values.
- Track Go runtime + process metrics via the default collectors (`prometheus.NewGoCollector`,
  enabled by default in client_golang).
- Instrument the **DB pool** (`pgxpool.Stat()`), Kafka consumer lag, queue depth, retry counts.

## Health & readiness (k8s)

These belong to the HTTP layer; details in `references/http-server.md`. In short:

- **`/livez`** — process is alive (cheap, no dependency checks). Failing → kubelet restarts the pod.
- **`/readyz`** — ready to serve: checks DB ping, Kafka connectivity, migrations applied. Failing →
  removed from Service endpoints but not restarted. During graceful shutdown, flip `/readyz` to
  failing *first* so traffic drains before the server stops accepting.

## Don'ts

- Don't trace or log inside tight inner loops; sample.
- Don't put PII/secrets in span attributes or metric labels.
- Don't forget to **flush/shutdown** the tracer provider on exit, or you lose the last batch of
  spans — wire `tp.Shutdown(ctx)` into the graceful-shutdown path.
- Don't reinvent dashboards: RED/USE + Go runtime metrics cover the 90% case.
