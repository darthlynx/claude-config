# syntax=docker/dockerfile:1
#
# Default multi-stage build for a pure-Go service (CGO disabled).
# Builder: official golang image. Runtime: Red Hat UBI 9 micro
# (Red Hat's minimal, distroless-style image — glibc + ca-certs, no shell, no package manager).
# For the CGO/librdkafka case, see Dockerfile.cgo.

# Pin to a specific patch tag (or @sha256 digest) in production for reproducibility.
FROM golang:1.24 AS build
WORKDIR /src

# Download modules first so this layer is cached unless go.mod/go.sum change.
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download

COPY . .

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
# Use a non-zero UID for OpenShift "arbitrary UID" compatibility.
FROM registry.access.redhat.com/ubi9-micro:9.4
COPY --from=build /out/app /app
USER 1001
EXPOSE 8080
ENTRYPOINT ["/app"]
