# Docker & Build

Goal: a small, non-root, reproducible image. Go's static linking makes this easy — *except* when
CGO is in play (e.g. `confluent-kafka-go` / librdkafka), which needs a different recipe. Both are
in `assets/`.

## Default: pure-Go, static, multi-stage → UBI micro

Use this when the binary doesn't need CGO (most services). Builder: the official `golang` image.
Runtime: Red Hat **UBI 9 micro** — minimal, no shell, no package manager (same security profile
as `gcr.io/distroless/static`, but the supported Red Hat equivalent). See `assets/Dockerfile`.

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.24 AS build
WORKDIR /src

# Cache modules separately so deps re-download only when go.mod/sum change.
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download

COPY . .
# Static, stripped, reproducible. ldflags inject build metadata (see references/tooling.md).
ARG VERSION=dev
ARG COMMIT=none
ARG DATE=unknown
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=linux go build \
      -trimpath \
      -ldflags="-s -w -X main.version=${VERSION} -X main.commit=${COMMIT} -X main.date=${DATE}" \
      -o /out/app ./cmd/myservice

# ubi9-micro ships glibc + ca-certificates; no shell, no package manager.
# UID 1001 satisfies OpenShift's "arbitrary UID" requirement and works on plain k8s.
FROM registry.access.redhat.com/ubi9-micro:9.4
COPY --from=build /out/app /app
USER 1001
EXPOSE 8080
ENTRYPOINT ["/app"]
```

Why these choices:
- **`CGO_ENABLED=0`** → fully static binary, runs on `ubi9-micro` (glibc present but unused) with
  no dynamic dependencies. It would also run on `scratch` / `gcr.io/distroless/static`.
- **`-trimpath`** strips local filesystem paths → reproducible builds, no info leak.
- **`-ldflags="-s -w"`** drops the symbol table and DWARF → smaller binary (keep symbols if you
  need readable production stack traces; the size win is modest).
- **`ubi9-micro`** ships CA certs (so TLS works) without a shell or package manager — comparable
  to `gcr.io/distroless/static` but with Red Hat's support lifecycle.
- **cache mounts** (`--mount=type=cache`) speed rebuilds dramatically; require BuildKit (default in
  modern Docker; the `# syntax` line enables it).
- **non-root UID 1001** is required by most hardened k8s `PodSecurity` policies, and works on
  OpenShift, which assigns the container a high random UID inside a defined range.

## CGO case: `confluent-kafka-go` / librdkafka

`confluent-kafka-go/v2` links librdkafka via CGO, so `CGO_ENABLED=0` won't compile it. You have two
paths:

**Option A — switch to a pure-Go Kafka client** (preferred when feasible): `twmb/franz-go` or
`segmentio/kafka-go` need no CGO, so you keep the clean micro/distroless build above. franz-go has
the most complete protocol support and good performance.

**Option B — keep CGO, build a dynamically-linked image.** See `assets/Dockerfile.cgo`. The runtime
is **`ubi9-minimal`, not `ubi9-micro` or distroless**: micro/distroless images have no shell and no
package manager, so there's no way to install librdkafka into them. You'd have to `COPY` librdkafka
and every transitive `.so` it needs from the build stage by hand — workable but brittle (and easy
to miss a dep). `ubi9-minimal` ships `microdnf`, which lets you install librdkafka cleanly:

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.24 AS build
WORKDIR /src
RUN apt-get update && apt-get install -y --no-install-recommends librdkafka-dev pkg-config \
 && rm -rf /var/lib/apt/lists/*
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=1 GOOS=linux go build -trimpath -tags dynamic \
      -ldflags="-s -w" -o /out/app ./cmd/myservice

# ubi9-minimal has microdnf; librdkafka is not in default UBI repos, so we enable EPEL.
# Replace the EPEL line with your internal mirror if you don't allow third-party repos.
FROM registry.access.redhat.com/ubi9-minimal:9.4
RUN rpm -i https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm \
 && microdnf install -y librdkafka ca-certificates \
 && microdnf clean all
COPY --from=build /out/app /usr/local/bin/app
USER 1001
EXPOSE 8080
ENTRYPOINT ["app"]
```

(`confluent-kafka-go` also bundles a static librdkafka via the default build tags on some platforms;
the `dynamic` tag above opts into the system lib. Verify against your target arch — musl/Alpine
needs the `musl` tag and extra care.)

## `.dockerignore`

Always ship one (see `assets/.dockerignore`) so the build context stays tiny and secrets/local
junk never enter an image layer:

```
.git
*.md
bin/
dist/
.env
*.local
testdata/
**/*_test.go
```

## Operational notes

- **Pin base images by tag and ideally digest** (`@sha256:...`) for reproducibility; renovate/
  dependabot to bump them.
- **One process per container**; let k8s handle restarts. No init system, no supervisor.
- **Health probes** are HTTP endpoints (see `references/http-server.md`), not shell scripts —
  ubi9-micro and distroless have no shell.
- **Scan images** (`trivy`, `grype`) and the binary (`govulncheck`, see `references/tooling.md`) in
  CI.
- Build multi-arch with `docker buildx build --platform linux/amd64,linux/arm64` if you deploy to
  mixed nodes.
- Keep `EXPOSE` documentary; the actual port comes from config/k8s.
