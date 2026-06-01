# Project Layout & Package Design

## Why not `golang-standards/project-layout`

The widely-cited `golang-standards/project-layout` repo is **not** an official standard and has
been publicly criticized by Go team members (notably for promoting `pkg/`). Don't cargo-cult it.
The only layout conventions with real backing are:

- **`cmd/<binary>/main.go`** — one directory per produced binary. Keep `main.go` thin.
- **`internal/`** — the Go *compiler* forbids imports of `internal/...` from outside the parent
  module subtree. This is the real mechanism for "private to this service." Use it liberally.
- Everything else is a judgment call. Favor flat over deep; add structure when a package gets big,
  not preemptively.

`pkg/` only earns its place when you deliberately publish importable libraries to *other* modules.
For a single deployable service, it's noise — put code in `internal/`.

## The layout itself

The canonical directory tree lives in `SKILL.md` (it's the artifact you reach for on every
scaffold, so it stays in the always-loaded layer). This file covers the *reasoning* behind it:
package boundaries, naming, deployment-artifact placement, and module versioning. In short the tree
is `cmd/<binary>/main.go` (thin) + `internal/` (everything private) + `api/`, `migrations/`, and the
deployment artifacts below — and deliberately **no `pkg/`**.

## Deployment artifacts

There is no Go-blessed location for a `Dockerfile` or Helm charts; these are conventions, not
standards. Sensible defaults (and the ones this skill assumes):

- **`Dockerfile` at the repo root.** Docker's build context defaults to the Dockerfile's directory,
  and `docker build .` at the root "just works" — the build can see `go.mod` and the full source
  tree with no `-f`/context gymnastics. Putting it under a subdir forces `docker build -f sub/Dockerfile .`
  with the context still pinned to root, which trips people up. (Per-image exception: a repo that
  builds several images can use `cmd/<svc>/Dockerfile` so each binary owns its build.)
- **Helm charts in `helm-chart/` at the repo root** (team convention). If you ever switch to a
  top-level `charts/` instead, note that Helm reserves a `charts/` *subdirectory inside a chart* for
  bundled dependencies, so place the chart at `charts/<name>/`, not loose in `charts/`.
- **GitOps caveat:** many teams running Kubernetes seriously keep manifests/charts in a *separate*
  GitOps repo (Argo CD / Flux) so deploys decouple from app builds. If that's the model, the service
  repo holds only the root `Dockerfile` and the chart lives elsewhere — `helm-chart/` then applies
  to the GitOps repo, not the service repo.

## Dependency direction (clean boundaries)

Dependencies point **inward**, toward business logic:

```
handler  ──>  service  ──>  store (interface)
                 ▲                  │ implemented by
                 └────── store (pgx concrete) ───┘
```

- `service` depends on a **store interface it defines**, not on `pgx` or SQL. This keeps business
  logic testable with fakes and free of transport/driver concerns.
- `handler` translates transport (HTTP request → domain call → HTTP response). No business rules.
- `store` (concrete, e.g. `pgxstore`) implements the interface `service` declares. Per Go idiom,
  **the interface lives with the consumer (`service`), not the producer (`store`)**.

This is "ports and adapters" applied lightly. Don't over-engineer with a `domain/entities/` +
`usecases/` + `infrastructure/` ceremony for a small service — the three-layer split above is
enough for most.

## Package naming (Google style)

- Short, lowercase, single word, no `under_scores` or `mixedCaps`: `package config`, not
  `package configManager`.
- Name for what the package *provides*. The caller writes `config.Load()`, so the package is
  `config` (not `configs`, not `configutil`).
- Avoid `util`, `common`, `helpers`, `base`, `shared`, `models` — they attract unrelated code and
  create import cycles. If `models` would hold domain types, those types usually belong *with the
  package that owns the behavior*.
- Don't stutter: in package `user`, the type is `user.Service`, not `user.UserService`.
- One package, one responsibility. If you can't describe a package's purpose in a sentence without
  "and," split it.

## Module path & versioning

- Module path matches the repo: `module github.com/acme/myservice`.
- For libraries that hit v2+, the major version goes in the path (`.../myservice/v2`) per semantic
  import versioning. Services rarely need this.
- Keep `go.mod`'s `go` directive at the minimum version you actually require; bump deliberately.

## Internal sub-modules

For a monorepo with several services, a single module with multiple `cmd/` entrypoints is simpler
than many modules. Reach for a multi-module workspace (`go.work`) only when services must version
their shared libraries independently.
