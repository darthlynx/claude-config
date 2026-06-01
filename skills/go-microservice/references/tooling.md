# Tooling, Quality Gates & Build Info

## Formatting (non-negotiable)

- `gofmt` / `goimports` (or `gci` for grouped imports). No debate, no PR comments about formatting —
  the tool decides. Wire it into pre-commit and CI (CI *checks*, fails on diff).

## Linting — `golangci-lint` (v2)

A single aggregator that runs many linters fast. Check in a `.golangci.yml` (template in
`assets/.golangci.yml`) using the **v2 config schema** (`version: "2"`, `linters.settings`,
`exclusions`, `formatters`). The v1 schema (`linters-settings`, `issues.exclude-rules`,
`gosimple`/`stylecheck` as separate linters) is no longer accepted by `golangci-lint` v2.

A solid baseline set: `govet`, `staticcheck` (which now absorbs `gosimple` and `stylecheck`),
`errcheck`, `ineffassign`, `unused`, `gosec`, `bodyclose`, `noctx`, `errorlint`, `containedctx`,
`contextcheck`, `revive`, `gocritic`. Add `sloglint` to enforce consistent `slog` key style. Run:

```bash
golangci-lint run ./...
```

Pin `golangci-lint` in CI (`golangci/golangci-lint-action` accepts a `version:` input) so an
upstream release can't break the build mid-PR.

Tune, don't disable wholesale. If a linter is noisy on a legitimate pattern, exclude that specific
case with a `//nolint:linter // reason` comment (always with a reason) rather than turning the
linter off globally.

## Vulnerability scanning — `govulncheck`

Official tool that reports vulnerabilities reachable from *your* code paths (not just "a vulnerable
version is in go.sum"), which keeps noise low. **Pin the version** — `@latest` is
reproducibility-hostile and lets a govulncheck release break your build mid-PR:

```bash
go run golang.org/x/vuln/cmd/govulncheck@v1.1.4 ./...
```

Bump the pin deliberately (Renovate/Dependabot can manage it). Run in CI and fail on findings.
Complement with image scanning (`trivy`/`grype`) for OS-level CVEs in the container (see
`references/docker.md`).

## `go vet` and the race detector

```bash
go vet ./...
go test -race -shuffle=on ./...   # -shuffle surfaces inter-test coupling
```

`-race` and `-shuffle=on` belong in CI permanently.

## Build info via `-ldflags`

Inject version metadata at build time and log it at startup (you already saw `buildVersion` in the
`run()` skeleton):

```go
// cmd/myservice/main.go (package main)
var (
	version = "dev"
	commit  = "none"
	date    = "unknown"
)
```

```bash
go build -trimpath \
  -ldflags="-s -w -X main.version=$(git describe --tags --always) \
            -X main.commit=$(git rev-parse --short HEAD) \
            -X main.date=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  ./cmd/myservice
```

Expose it on `/version` and include it in every startup log line and trace resource (see
`references/observability.md`). `runtime/debug.ReadBuildInfo()` also gives you VCS info when built
with module mode and `-buildvcs` — useful as a fallback.

## Makefile (template in `assets/Makefile`)

Codify the common commands so everyone (and CI) runs the same thing:

```makefile
.PHONY: build test lint vuln race tidy run docker
build:  ; go build -o bin/myservice ./cmd/myservice
test:   ; go test -race -shuffle=on -cover ./...
lint:   ; golangci-lint run ./...
vuln:   ; go run golang.org/x/vuln/cmd/govulncheck@$(GOVULNCHECK_VERSION) ./...
tidy:   ; go mod tidy && go mod verify
run:    ; go run ./cmd/myservice
docker: ; docker build -t myservice:$(shell git describe --tags --always) .
```

(Some teams prefer `Taskfile.yml` / `just` — fine; the point is a single source of truth for
commands.)

## CI pipeline (the gates, in order)

1. `go mod verify` + `go mod tidy` diff check (tidy must produce no changes).
2. `gofmt`/`goimports` diff check.
3. `go vet ./...`
4. `golangci-lint run ./...`
5. `go test -race -shuffle=on -coverprofile=cover.out ./...`
6. `govulncheck ./...`
7. Integration tests: `go test -tags=integration ./...` (separate, slower stage).
8. Build the image, then for `main`/tags:
   - **Scan** with `trivy image --severity HIGH,CRITICAL --exit-code 1 <image>` (fails build).
   - **SBOM** with `syft <image> -o spdx-json=sbom.spdx.json` (attach to the release).
   - **Sign** with `cosign sign --yes <image>@<digest>` (keyless via OIDC in GitHub Actions).
   - **Attest** the SBOM with `cosign attest --yes --predicate sbom.spdx.json --type spdxjson <image>@<digest>`.
   - Push on success.

Cache the Go build + module cache between runs. Pin the Go toolchain version in CI from `go.mod`:

```yaml
# GitHub Actions
- uses: actions/setup-go@v5
  with:
    go-version-file: go.mod   # keeps CI and go.mod in lockstep
    check-latest: false
```

### Why SBOM + signing matter (briefly)

- **SBOM** (Software Bill of Materials) is the "what's inside" manifest. When the next Log4j-class
  CVE drops, a checked-in SBOM is the difference between "we know in five minutes" and
  "we're paging engineers for two days." SPDX or CycloneDX — pick one and be consistent.
- **Cosign signatures + attestations** let downstream consumers (your deploy pipeline, your
  customers) verify an image was built by *your* CI from *your* repo, not swapped in transit or by
  a compromised registry account. Pair with an admission controller (`policy-controller`, Kyverno)
  that rejects unsigned images.
- **SLSA provenance** (`cosign attest --type slsaprovenance`) goes one step further: it records
  *how* the image was built (commit, builder, args), so a tampered build is detectable even when
  the signature is valid. Worth the few extra CI lines once you're already running cosign.

## Dependency hygiene

- `go mod tidy` after every dependency change; commit `go.sum`.
- Keep the dependency tree shallow — each transitive dep is attack surface and a future CVE.
  Audit `go mod graph` occasionally.
- Automate bumps (Dependabot/Renovate) but gate them through the same CI.
- Prefer the standard library and `golang.org/x/...` over third-party where they're equivalent.
