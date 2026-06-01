# Configuration (Environment-Driven, 12-Factor)

## Principles

- **One typed config struct**, populated from the environment at startup, validated once, then
  passed (read-only) into constructors. Never call `os.Getenv` scattered through the codebase.
- **Fail fast.** Missing required values, unparseable durations, or out-of-range numbers must
  return an error from `Load()` so the process exits before serving traffic.
- **Defaults are explicit** and safe for local dev; production overrides via env.
- **Secrets are injected** (env from a secret manager / k8s Secret), never committed, never logged.
- Config is **immutable after load**. Live reconfiguration (feature flags) is a separate concern —
  use a flag provider, not env mutation.

## Pattern: struct + tags + validation

Use a small, well-maintained parser. `kelseyhightower/envconfig` and `caarlos0/env` are both fine;
prefer `caarlos0/env` for active maintenance, or `kelseyhightower/envconfig` if already in use.
Reach for Viper only when you genuinely need layered file+env+flag config (most services don't).

```go
package config

import (
	"fmt"
	"time"

	"github.com/caarlos0/env/v11"
)

type Config struct {
	Env  string `env:"ENV" envDefault:"dev"` // dev|staging|prod
	HTTP HTTP
	DB   DB
	Log  Log
	OTel OTel
}

type HTTP struct {
	Addr            string        `env:"HTTP_ADDR" envDefault:":8080"`
	ReadTimeout     time.Duration `env:"HTTP_READ_TIMEOUT" envDefault:"10s"`
	WriteTimeout    time.Duration `env:"HTTP_WRITE_TIMEOUT" envDefault:"15s"`
	IdleTimeout     time.Duration `env:"HTTP_IDLE_TIMEOUT" envDefault:"60s"`
	ShutdownTimeout time.Duration `env:"HTTP_SHUTDOWN_TIMEOUT" envDefault:"20s"`
	DrainDelay      time.Duration `env:"HTTP_DRAIN_DELAY" envDefault:"10s"` // /readyz=503 → wait → Shutdown
}

type DB struct {
	DSN             string        `env:"DB_DSN,required"` // contains secret — never log
	MaxConns        int32         `env:"DB_MAX_CONNS" envDefault:"10"`
	MinConns        int32         `env:"DB_MIN_CONNS" envDefault:"1"`
	MaxConnIdle     time.Duration `env:"DB_MAX_CONN_IDLE" envDefault:"5m"`
	MaxConnLifetime time.Duration `env:"DB_MAX_CONN_LIFETIME" envDefault:"30m"` // recycle to spread load
}

type Log struct {
	Level  string `env:"LOG_LEVEL" envDefault:"info"`  // debug|info|warn|error
	Format string `env:"LOG_FORMAT" envDefault:"json"` // json|text
}

// OTel configures the OpenTelemetry exporter. Endpoint is optional — when empty, the SDK
// also honors OTEL_EXPORTER_OTLP_ENDPOINT directly. SampleRatio is the head sampler ratio
// applied via trace.ParentBased(TraceIDRatioBased(ratio)); see references/observability.md.
type OTel struct {
	ServiceName string  `env:"OTEL_SERVICE_NAME" envDefault:"myservice"`
	Endpoint    string  `env:"OTEL_EXPORTER_OTLP_ENDPOINT"`
	SampleRatio float64 `env:"OTEL_TRACES_SAMPLE_RATIO" envDefault:"0.1"`
}

func Load() (Config, error) {
	var cfg Config
	if err := env.Parse(&cfg); err != nil {
		return Config{}, fmt.Errorf("parse env: %w", err)
	}
	if err := cfg.Validate(); err != nil {
		return Config{}, fmt.Errorf("invalid config: %w", err)
	}
	return cfg, nil
}

func (c Config) Validate() error {
	switch c.Env {
	case "dev", "staging", "prod":
	default:
		return fmt.Errorf("ENV %q must be one of dev|staging|prod", c.Env)
	}
	if c.HTTP.Addr == "" {
		return fmt.Errorf("HTTP_ADDR must not be empty")
	}
	if c.HTTP.ReadTimeout <= 0 || c.HTTP.WriteTimeout <= 0 ||
		c.HTTP.IdleTimeout <= 0 || c.HTTP.ShutdownTimeout <= 0 || c.HTTP.DrainDelay < 0 {
		return fmt.Errorf("HTTP timeouts/drain must be > 0 (DrainDelay may be 0)")
	}
	switch c.Log.Level {
	case "debug", "info", "warn", "error":
	default:
		return fmt.Errorf("LOG_LEVEL %q must be one of debug|info|warn|error", c.Log.Level)
	}
	switch c.Log.Format {
	case "json", "text":
	default:
		return fmt.Errorf("LOG_FORMAT %q must be one of json|text", c.Log.Format)
	}
	if c.DB.MaxConns < 1 {
		return fmt.Errorf("DB_MAX_CONNS must be >= 1, got %d", c.DB.MaxConns)
	}
	if c.OTel.SampleRatio < 0 || c.OTel.SampleRatio > 1 {
		return fmt.Errorf("OTEL_TRACES_SAMPLE_RATIO must be in [0,1], got %v", c.OTel.SampleRatio)
	}
	return nil
}
```

## Secrets

- Pull from a secret manager (Vault, AWS/GCP Secrets Manager) or a k8s Secret mounted as env.
- Never include secret-bearing fields in logs. Give them a `String()` that redacts, or exclude
  them from any logged struct. A simple guard:

  ```go
  type Secret string
  func (Secret) String() string  { return "[REDACTED]" }
  func (Secret) LogValue() slog.Value { return slog.StringValue("[REDACTED]") }
  ```

  `LogValue` is honored by `slog`, so `slog.Any("dsn", cfg.DB.DSN)` stays safe if `DSN` is a
  `Secret`. See `references/logging.md`.
- For DSNs that must stay strings, log only the redacted/host portion, never the full value.

## Config validation as documentation

A thorough `Validate()` doubles as a spec: it tells the next engineer exactly which env vars exist
and what values are legal. Pair it with a checked-in `.env.example` (no real secrets) listing every
variable with a placeholder and a one-line comment.

## Local dev: `.env` files

For local development, devs typically want a `.env` file rather than `export FOO=bar` lines in
their shell rc. Two options:

- **Run via `make run` that sources the file**: e.g. `set -a; . ./.env; set +a; go run ./cmd/...`
  — no code change, no extra dependency. Cleanest.
- **`github.com/joho/godotenv`** loaded conditionally in `run()`: load `.env` *only when not in
  prod* and *only as a fallback* (don't let it override real env). Production must still get all
  config from the real environment — `.env` is a dev convenience, not a deployment mechanism.

Always git-ignore `.env`; commit `.env.example` instead. If you reach for godotenv, gate it:

```go
if cfg.Env == "dev" {
    _ = godotenv.Load() // best-effort, ignored if file is missing
}
```

(Or load *before* `config.Load()` so the env parser sees the values.)

## What NOT to do

- Don't read env vars lazily on first use — startup should fully determine viability.
- Don't store config in package-level vars; pass it in.
- Don't overload one var with structured data (JSON-in-env); prefer discrete vars, or a mounted
  file path if the data is genuinely structured.
- Don't branch business logic on `cfg.Env == "prod"` everywhere; expose specific feature toggles.
