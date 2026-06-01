# Database (CockroachDB + pgx)

CockroachDB is Postgres wire-compatible, so the driver of choice is **`github.com/jackc/pgx/v5`**
with its native `pgxpool`. Add **`github.com/cockroachdb/cockroach-go/v2/crdb/crdbpgx`** for the
one piece pgx doesn't handle on its own: transparent retry of serialization failures.

Cockroach behaves *differently enough* from vanilla Postgres that several common Postgres patterns
become anti-patterns. The big ones:

- Default isolation is **SERIALIZABLE** (not READ COMMITTED). Transactions can fail with retry
  errors and clients are expected to retry.
- **Online schema changes**: `ALTER TABLE`, `CREATE INDEX`, etc. don't take the heavy locks
  Postgres takes. There is no `CREATE INDEX CONCURRENTLY` because plain `CREATE INDEX` is already
  non-blocking.
- **No VACUUM**, no autovacuum to tune. MVCC GC is automatic.
- **No PgBouncer** in front. Cockroach nodes are themselves the SQL gateway; put a TCP/L4 load
  balancer in front and connect direct.

## Connection pool

```go
package store

import (
	"context"
	"fmt"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
)

func NewPool(ctx context.Context, cfg config.DB) (*pgxpool.Pool, error) {
	pcfg, err := pgxpool.ParseConfig(cfg.DSN)
	if err != nil {
		return nil, fmt.Errorf("parse dsn: %w", err)
	}
	pcfg.MaxConns = cfg.MaxConns                   // see sizing notes below
	pcfg.MinConns = cfg.MinConns                   // keep a warm floor
	pcfg.MaxConnLifetime = cfg.MaxConnLifetime     // recycle to spread load across CRDB nodes
	pcfg.MaxConnIdleTime = cfg.MaxConnIdle         // close idle conns (5m)
	pcfg.HealthCheckPeriod = 1 * time.Minute       // background liveness probe

	pool, err := pgxpool.NewWithConfig(ctx, pcfg)
	if err != nil {
		return nil, fmt.Errorf("connect: %w", err)
	}
	if err := pool.Ping(ctx); err != nil {        // fail fast at startup
		pool.Close()
		return nil, fmt.Errorf("ping: %w", err)
	}
	return pool, nil
}
```

### Pool sizing

Cockroach scales horizontally — adding nodes raises capacity in a way single-node Postgres can't —
so the conservative "(vCPUs × 2) + spindles" Postgres rule isn't a binding limit here. The actual
constraints:

- **Per-replica budget**: a Cockroach cluster's total in-flight transactions are bounded by its
  CPU/IO. Size pools so `replicas × MaxConns` stays below what your cluster handles in stress
  testing. Start with **10–25 per service replica** and tune from there.
- **MinConns**: 1–2 keeps first-request latency low without tying up gateway slots.

### `MaxConnLifetime` matters even more here

Without a lifetime, a long-lived connection sticks to whichever Cockroach node it first reached
through the LB. After a rolling cluster upgrade, scale-out, or node failure, load distribution
silently skews. Set 15–30m so connections cycle through the LB regularly.

## Transactions — you MUST retry serialization failures

Under SERIALIZABLE, Cockroach returns SQLSTATE `40001` (retryable serialization failure) when
concurrent transactions conflict. This is normal under load — it's how Cockroach guarantees
correctness without locks. **Clients must retry** with backoff; the driver does not do this for you.

Use `crdbpgx.ExecuteTx`, which handles the retry loop, savepoints, and classification:

```go
import (
	"github.com/cockroachdb/cockroach-go/v2/crdb/crdbpgx"
	"github.com/jackc/pgx/v5"
)

func (s *Store) Transfer(ctx context.Context, from, to int64, amount int64) error {
	return crdbpgx.ExecuteTx(ctx, s.pool, pgx.TxOptions{}, func(tx pgx.Tx) error {
		if _, err := tx.Exec(ctx, `UPDATE accounts SET balance = balance - $1 WHERE id = $2`, amount, from); err != nil {
			return err
		}
		if _, err := tx.Exec(ctx, `UPDATE accounts SET balance = balance + $1 WHERE id = $2`, amount, to); err != nil {
			return err
		}
		return nil
	})
}
```

The callback may be invoked multiple times. **It must be idempotent** — no incrementing in-memory
counters, no logging side effects, no enqueuing external work mid-transaction. Move those after
`ExecuteTx` returns nil.

- **Keep transactions short.** Holding a transaction across a network call to another service
  (HTTP, Kafka publish) amplifies contention and inflates retry rates.
- **Pick the right `TxOptions.AccessMode`**: `pgx.ReadOnly` for queries; Cockroach uses this hint
  to route to follower replicas where appropriate.
- **For pure read-only flows that tolerate slightly stale data**, use `AS OF SYSTEM TIME
  follower_read_timestamp()` (or `SET TRANSACTION AS OF SYSTEM TIME ...`) to read from followers
  with no contention against writers. Big latency win in multi-region setups.

## Schema migrations

The pragmatic options for Cockroach:

| Approach | When to use |
|---|---|
| **K8s pre-deploy Job** (`goose up`, `golang-migrate up`) | Default. Apply migration, then roll pods. |
| **Out-of-band** (ops runs the migration manually) | Long-running migrations where you want explicit human checkpointing. |
| **Auto-migrate on app start** | Single-replica dev only. Same anti-pattern in Cockroach as in Postgres. |

What changes vs Postgres migrations:

- **No `CONCURRENTLY` keyword needed.** `CREATE INDEX` is online by default. The migration tool's
  default "wrap each statement in a transaction" mode is fine.
- **Schema changes are eventually consistent.** `ALTER TABLE` returns when the job is *queued*;
  use `SHOW JOBS WHEN COMPLETE` (or query `crdb_internal.jobs`) if your deploy pipeline needs to
  block on completion before rolling pods.
- **Backward-compatible deploys are still required.** A rolling deploy means old and new app
  versions hit the DB simultaneously; schema must support both.
- **Forward-only in production.** Down migrations on a distributed SQL DB with transactional
  schema changes are even less recoverable than on Postgres. Roll forward.

Tooling: `pressly/goose` or `golang-migrate/migrate` work against Cockroach with the Postgres
driver. Both fine; pick one.

## Repository pattern (interface lives with the consumer)

The store interface belongs in `service/` (per `references/project-layout.md`); the pgx
implementation in `internal/store/crdbstore/`. This keeps business logic testable with a fake and
free of driver-specific types.

```go
// internal/service/order.go
type Orders interface {
	Get(ctx context.Context, id int64) (Order, error)
	Create(ctx context.Context, o Order) (int64, error)
}

// internal/store/crdbstore/orders.go
type OrdersStore struct{ db *pgxpool.Pool }

func (s *OrdersStore) Get(ctx context.Context, id int64) (service.Order, error) {
	var o service.Order
	err := s.db.QueryRow(ctx, `SELECT id, total FROM orders WHERE id = $1`, id).
		Scan(&o.ID, &o.Total)
	if errors.Is(err, pgx.ErrNoRows) {
		return service.Order{}, service.ErrNotFound // translate driver errors at the boundary
	}
	if err != nil {
		return service.Order{}, fmt.Errorf("query order %d: %w", id, err)
	}
	return o, nil
}
```

Translate `pgx.ErrNoRows`, unique-violation (`23505`), and FK-violation (`23503`) into **domain**
errors at the store boundary. The service layer should never branch on SQLSTATE.

## Multi-region notes (only if you run multi-region)

Cockroach's killer feature; also where most performance footguns hide.

- Set **table localities** explicitly (`REGIONAL BY ROW`, `REGIONAL BY TABLE`, `GLOBAL`).
- Use **follower reads** for read-heavy paths that tolerate ~5s staleness — eliminates cross-region
  round-trips for reads.
- Watch **AS OF SYSTEM TIME** in your repository helpers; passing a `time.Duration` for staleness
  belongs in the read path, not the schema.
- Cross-region writes pay a consensus latency — design for it (batch, async, queue) rather than
  hoping replication is fast.

## Observability

- `pgxpool.Stat()` — expose as Prometheus metrics (`acquired_conns`, `total_conns`,
  `acquire_count`, `acquire_duration_avg`). Pool saturation is the #1 silent failure.
- OTel pgx tracer (`github.com/exaring/otelpgx`) — every query becomes a span with the SQL.
- **Track retry rate** as a first-class metric. `crdbpgx.ExecuteTx` doesn't expose attempts
  directly — wrap it and emit a counter (`crdb_tx_attempts_total{outcome="retry|commit|abort"}`).
  A retry rate >5% is a contention warning; investigate hot rows / overly broad transactions.
- Cockroach's built-in UI (`/_status/` endpoints) gives you per-statement latency histograms — link
  to it from your runbooks.

## Anti-patterns

- Treating Cockroach like single-node Postgres: no retry loop, no `MaxConnLifetime`, optimistic
  about contention rates.
- Long-running transactions wrapping HTTP/gRPC calls to other services — guarantees retry storms.
- Side effects inside the `ExecuteTx` callback (logging, metrics increments, channel sends) —
  duplicated when the tx retries.
- Putting **PgBouncer** in front of Cockroach. It's an extra hop with no benefit; the Cockroach
  gateway already pools internally.
- Using `database/sql` + `lib/pq` for a new service. Use pgx native.
- `*pgxpool.Pool` as a package-level global; pass it in.
- Down migrations as a rollback strategy on production data.
- `WHERE id = ` + raw string concatenation. Always parameterize.
