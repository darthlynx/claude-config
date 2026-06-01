BINARY      := myservice
PKG         := ./cmd/$(BINARY)
VERSION     := $(shell git describe --tags --always --dirty 2>/dev/null || echo dev)
COMMIT      := $(shell git rev-parse --short HEAD 2>/dev/null || echo none)
DATE        := $(shell date -u +%Y-%m-%dT%H:%M:%SZ)
LDFLAGS     := -s -w -X main.version=$(VERSION) -X main.commit=$(COMMIT) -X main.date=$(DATE)

.PHONY: help build run test test-integration race lint vuln fmt tidy clean docker

help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | \
		awk 'BEGIN{FS=":.*?## "}{printf "  \033[36m%-16s\033[0m %s\n", $$1, $$2}'

build: ## Build the binary into bin/
	CGO_ENABLED=0 go build -trimpath -ldflags="$(LDFLAGS)" -o bin/$(BINARY) $(PKG)

run: ## Run locally
	go run $(PKG)

test: ## Unit tests with race detector and coverage
	go test -race -shuffle=on -coverprofile=cover.out ./...

test-integration: ## Integration tests (testcontainers); needs Docker
	go test -tags=integration -race ./...

race: ## Run the full suite under the race detector
	go test -race ./...

lint: ## Run golangci-lint
	golangci-lint run ./...

GOVULNCHECK_VERSION := v1.1.4

vuln: ## Scan for known vulnerabilities reachable from the code
	go run golang.org/x/vuln/cmd/govulncheck@$(GOVULNCHECK_VERSION) ./...

fmt: ## Format and group imports (goimports is a superset of gofmt)
	goimports -w .

tidy: ## Tidy and verify modules
	go mod tidy
	go mod verify

clean: ## Remove build artifacts
	rm -rf bin cover.out

docker: ## Build the container image
	docker build \
		--build-arg VERSION=$(VERSION) \
		--build-arg COMMIT=$(COMMIT) \
		--build-arg DATE=$(DATE) \
		-t $(BINARY):$(VERSION) .
