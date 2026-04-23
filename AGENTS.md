# AIGW — Agent Guide

> This file is intended for AI coding agents. It describes the project architecture, build system, conventions, and workflows so you can be productive without prior context.

---

## Project Overview

**AIGW** (AI Gateway) is an intelligent inference scheduler for large-scale LLM inference services. It is implemented as an **Envoy Golang HTTP filter** (using the `mosn.io/htnn/api` framework) that provides:

- Intelligent routing (load-aware, KVCache-aware, and LoRA-aware)
- Overload protection
- Multi-tenant QoS
- Latency prediction (TTFT / TPOT) via online Recursive Least Squares (RLS) models
- Near real-time load metric collection via a separate Metadata Center component

The project builds into a single C shared library (`libgolang.so`) that Envoy loads at runtime. It is written in **Go 1.22** and is currently in early, rapid development.

---

## Technology Stack

| Layer | Technology |
|-------|------------|
| Language | Go 1.22 |
| Runtime / Proxy | Envoy (via HTNN Golang filter API) |
| Control Plane | Istio (optional, for production xDS mode) |
| Base Images | `golang:1.22-bullseye` (build), `istio/proxyv2:1.27.3` or `envoyproxy/envoy:contrib-v1.35.6` (runtime) |
| Key Dependencies | `mosn.io/htnn/api`, `github.com/envoyproxy/envoy`, `github.com/bytedance/sonic`, `github.com/prometheus/client_golang` |
| Protocols | HTTP/1 + SSE (OpenAI), gRPC (Triton) |
| Metrics | Prometheus |
| License | Apache-2.0 |

---

## Directory Structure

```
.
├── cmd/libgolang/              # Entry point for the shared library
│   ├── main.go                 # Registers "fm" (FilterManager) and "cm" (ConsumerManager) filters
│   └── init.go                 # Starts pprof and Prometheus metrics endpoints
├── pkg/                        # Core business logic
│   ├── aigateway/              # Cluster management, service discovery, load balancing, OpenAI structs
│   ├── async_log/              # Async rotating file logger (lumberjack)
│   ├── async_request/          # Generic async worker-pool queue with retries
│   ├── circuitbreaker/         # Counting circuit breaker (closed/open/half-open)
│   ├── common/                 # Env parsing, typed context helpers
│   ├── errcode/                # Standardized gateway error codes
│   ├── metadata_center/        # Client for external Metadata Center; service discovery + types
│   ├── metrics_stats/          # TTFT matching/recording; moving-average fallback
│   ├── prediction/             # RLS-based TTFT/TPOT predictors
│   ├── prom/                   # Prometheus metric definitions
│   ├── request/                # Header/path helpers, thread-safe access-log fields
│   ├── simplejson/             # Safe JSON encoding wrapper
│   └── trace/                  # Trace ID extraction from plugin state
├── plugins/                    # Envoy filter plugins
│   ├── api/v1/                 # Shared protobuf definitions (e.g., HeaderValue)
│   ├── llmproxy/               # Main LLM proxy plugin
│   │   ├── config/             # Protobuf + Go config parsing
│   │   ├── log/                # LLM-specific log item builders
│   │   └── transcoder/         # Protocol transcoder interface + OpenAI implementation
│   └── plugins.go              # Plugin registration bootstrap
├── etc/                        # Configuration files
│   ├── clusters.json           # Static cluster definitions for local dev
│   ├── envoy-local.yaml        # Standalone Envoy bootstrap
│   ├── envoy-istio.yaml        # Istio xDS Envoy bootstrap
│   ├── istio.yaml              # Istio mesh config
│   └── config_crds/            # Istio CRDs (EnvoyFilter, Gateway, ServiceEntry, VirtualService)
├── docs/                       # Developer guides (en, zh) and architecture images
├── .github/workflows/          # GitHub Actions (test, lint)
├── Dockerfile                  # Multi-stage image build
├── Makefile                    # Primary development interface
├── go.mod / go.sum             # Go module definition
└── .golangci.yml               # Go linting configuration
```

---

## Build System

The `Makefile` is the single source of truth for builds, tests, and local development.

### Key Targets

| Command | What it does |
|---------|--------------|
| `make build-so-local` | Build `libgolang.so` locally with CGO (`buildmode=c-shared`) |
| `make build-so` | Build `libgolang.so` inside a Docker container (default for CI) |
| `make unit-test-local` | Run unit tests locally with race detection and coverage |
| `make unit-test` | Run unit tests inside Docker |
| `make lint-go` | Run `golangci-lint` on `./plugins/...` and `./pkg/...` |
| `make lint-license` | Check Apache-2.0 license headers with SkyWalking Eyes |
| `make fix-license` | Auto-fix missing license headers |
| `make gen-proto` | Generate `*.pb.go` and `*.pb.validate.go` from `.proto` files under `./plugins/` |
| `make build-image` | Build Docker image tagged `aigw` |
| `make start-aigw-local` | Start Envoy with static config (ports 10000/10001/15000) |
| `make start-aigw-xds` | Start Envoy with Istio xDS config |
| `make stop-aigw` | Stop the dev Envoy container |
| `make start-istio` | Start local Istio pilot-discovery watching `etc/config_crds` |
| `make stop-istio` | Stop the local Istio container |

### Build Tags

- `so` — required when building the shared object
- `envoydev` — default Envoy API version tag (used in builds and linting)
- `integrationtest` — used for integration-test SO builds (scaffolded, no tests yet)

---

## Testing Instructions

### Unit Tests

Test files are co-located with source code using the `*_test.go` suffix.

```bash
# Local (requires CGO + Envoy dev headers)
make unit-test-local

# Docker (recommended, matches CI)
make unit-test
```

The `unit-test-local` target runs:
```bash
go test -tags envoydev -v ./plugins/... ./pkg/... \
  -gcflags="all=-N -l" -race -covermode=atomic -coverprofile=coverage.out \
  -coverpkg=github.com/aigw-project/aigw/...
```

### Test Frameworks

- Go standard `testing`
- `github.com/stretchr/testify` for assertions
- Some tests use `mosn.io/htnn/api/plugins/tests/pkg/envoy` for mock Envoy CAPI contexts

### Integration Tests

The Makefile contains targets (`build-test-so-local`, `build-test-so`, `integration-test`) and expects an entrypoint at `tests/integration/cmd/libgolang`. However, **no integration test code currently exists** in the repository — the `integration-test` target is guarded by an `if` and is effectively a no-op until `.go` files are added under `./tests/integration/`.

---

## Development Workflow

### Prerequisites

1. Docker
2. Go 1.22+
3. A running **Metadata Center** service (separate repo: `aigw-project/metadata-center`) on port `8080`

### Quick Start (Standalone Mode)

```bash
# 1. Build the shared library
make build-so

# 2. Start AIGW with static config
make start-aigw-local
```

This exposes:
- Port `10000` — AIGW service
- Port `10001` — Mock inference backend
- Port `15000` — Envoy admin interface

### Quick Start (Istio Mode)

```bash
make start-istio      # Start local Istio pilot
make start-aigw-xds   # Start AIGW with xDS config
```

### Test a Request

```bash
curl 'localhost:10000/v1/chat/completions' \
  -H 'Content-Type: application/json' \
  --data '{
    "model": "qwen3",
    "messages": [{"role": "user", "content": "who are you"}],
    "stream": false
  }'
```

### Stop

```bash
make stop-aigw
```

---

## Code Style Guidelines

### Linting

Linting is enforced in CI. Run `make lint-go` before pushing.

- **golangci-lint version:** `1.62.2`
- **Enabled linters:** `bodyclose`, `contextcheck`, `errcheck`, `gocheckcompilerdirectives`, `gocritic`, `gosimple`, `govet`, `ineffassign`, `loggercheck`, `nilerr`, `staticcheck`, `unconvert`, `unparam`
- **Disabled:** `unused` (commented out)
- **Build tag:** `envoydev`

Exclusions:
- `_test.go` files skip `bodyclose`, `errcheck`, `gosec`, `unparam`
- `*.pb.go` files skip `staticcheck`
- `vendor/` is ignored

### License Headers

Every `.go` file must start with the Apache-2.0 copyright notice:

```go
// Copyright The AIGW Authors.
//
// Licensed under the Apache License, Version 2.0 (the "License");
// ...
```

Run `make fix-license` to auto-add missing headers. CI will fail if headers are missing (`make lint-license`).

### General Conventions

- Use Go 1.22 idioms.
- Keep packages focused by domain (see Directory Structure).
- Protobuf-generated files (`*.pb.go`, `*.pb.validate.go`) must not be hand-edited; regenerate with `make gen-proto`.

---

## Deployment & CI/CD

### Docker Image

The `Dockerfile` performs a multi-stage build:
1. **Builder** (`golang:1.22-bullseye`) → compiles `libgolang.so`
2. **Initial** (`envoyproxy/envoy:contrib-v1.35.6` or `istio/proxyv2`) → installs debug tools
3. **Final** → copies `libgolang.so` and `etc/envoy-local.yaml`

Default command: `envoy -c /etc/demo.yaml`

### GitHub Actions

| Workflow | Trigger | Jobs |
|----------|---------|------|
| `test.yml` | Push/PR to `main` or `release/**` (ignores `site/**` and `**/*.md`) | `make unit-test`, `make build-so` |
| `lint.yml` | Push/PR to `main` or `release/**` | `make lint-go`, `make lint-license` |

Both workflows set `IN_CI: true` and run on `ubuntu-latest` with Go 1.22. Timeout: 10 minutes.

### Kubernetes / Istio

No Helm charts are present. Deployment uses raw Istio CRDs in `etc/config_crds/`:
- `envoyfilter-golang-httpfilter.yaml` — registers the Golang HTTP filter
- `gateway.yaml` — Istio Gateway on port 10000
- `service-entry.yaml` — ServiceEntry for backend discovery
- `virtualhost.yaml` — VirtualService routing

---

## Security Considerations

- The project builds as a **C shared library** loaded by Envoy. Memory safety relies on Go's runtime and the HTNN/CGO boundary.
- `gosec` linter excludes `G402` (TLS `InsecureSkipVerify`), which means the codebase may skip TLS verification in certain paths; review any usage carefully.
- The Metadata Center client supports circuit breaking and retry/failover for resilience, but sensitive traffic should still be run over TLS where possible.
- No secrets or credentials are stored in this repository; runtime configuration is injected via environment variables (e.g., `AIGW_META_DATA_CENTER_HOST`, `AIGW_META_DATA_CENTER_PORT`).

---

## Architecture Notes for Agents

### Request Lifecycle (`plugins/llmproxy/`)

1. `DecodeHeaders` → waits for full body (`WaitAllData`)
2. `DecodeRequest` → parses request via **Transcoder**, resolves model name, runs load balancer, overrides upstream host
3. `EncodeHeaders` / `EncodeData` / `EncodeResponse` → transcribes backend response to OpenAI format, handles SSE streaming, splits reasoning content (`<think>` tags)
4. `OnLog` → records TTFT, token timestamps, cleans up metadata-center counters

### Load Balancing (`pkg/aigateway/loadbalancer/`)

- Two-tier: global LB selects cluster → cluster LB selects host.
- The main algorithm is **`inference_lb`**: composite scoring based on queue depth, prompt length, and KV-cache hit rate.
- Weights and awareness flags are tunable per model via `LBConfig`.

### Prediction (`pkg/prediction/`)

- **TTFT:** 6-parameter polynomial RLS trained online per model name.
- **TPOT:** Piecewise-linear RLS segmented by batch-size thresholds.
- **Fallback:** Exponential moving average (EMA) bucketed by prompt length KB (`AIGW_USE_MOVING_AVERAGE`).

### Extensibility Points

- **New LB algorithms:** implement `types.LoadBalancer`, register via `manager.RegisterLbType`.
- **New protocols:** implement `transcoder.Transcoder`, register via `transcoder.RegisterTranscoderFactory`.
- **New cluster providers:** implement `types.ClusterInfoProvider`, swap in `clustermanager/init.go`.
- **New Metadata Center backends:** implement `types.MetadataCenter`, swap in `metadata_center/init.go`.

---

## Useful Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `AIGW_META_DATA_CENTER_HOST` | local IP | Metadata Center hostname |
| `AIGW_META_DATA_CENTER_PORT` | `8080` | Metadata Center port |
| `AIGW_PROMETHEUS_ADDRESS` | `:6061` | Prometheus metrics endpoint |
| `AIGW_PPROF_ADDRESS` | — | pprof debug endpoint (if set) |
| `AIGW_USE_MOVING_AVERAGE` | — | Set to use EMA instead of RLS for TTFT |

---

## Quick Reference

```bash
# Build
make build-so

# Test
make unit-test

# Lint
make lint-go && make lint-license

# Run locally
make start-aigw-local

# Stop
make stop-aigw
```
