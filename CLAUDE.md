# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Test Commands

```bash
make build-so              # Build libgolang.so in Docker (CI default)
make build-so-local        # Build libgolang.so locally (requires CGO + Envoy dev headers)

make unit-test             # Run unit tests in Docker (matches CI)
make unit-test-local       # Run unit tests locally with race detection + coverage

make lint-go               # Run golangci-lint v1.62.2 on ./plugins/... ./pkg/...
make lint-license           # Check Apache-2.0 license headers
make fix-license            # Auto-fix missing license headers

make gen-proto              # Regenerate *.pb.go and *.pb.validate.go from .proto files

make start-aigw-local      # Run Envoy with static config (ports 10000/10001/15000)
make stop-aigw              # Stop the dev Envoy container
```

Run a single test:
```bash
go test -tags envoydev -v -race ./pkg/prediction/... -run TestRLS
```

## Architecture

AIGW is an **Envoy Golang HTTP filter** (built via `mosn.io/htnn/api`) that compiles to `libgolang.so` — a C shared library loaded by Envoy at runtime. It acts as an intelligent inference scheduler for LLM services.

### Entry Point & Plugin Registration

`cmd/libgolang/main.go` registers two Envoy filter factories:
- `"fm"` — FilterManager (HTNN plugin orchestrator)
- `"cm"` — ConsumerManager

The `llmproxy` plugin is auto-registered via `import _ "github.com/aigw-project/aigw/plugins/llmproxy"` in `plugins/plugins.go`.

### Request Lifecycle (plugins/llmproxy/filter.go)

1. `DecodeHeaders` → waits for full body (`WaitAllData`)
2. `DecodeRequest` → parses request via Transcoder, resolves model, runs LB, overrides upstream host, records prompt to Metadata Center
3. `EncodeHeaders` / `EncodeData` / `EncodeResponse` → transcodes backend response to OpenAI format, handles SSE streaming
4. `OnLog` → records TTFT/token timestamps, cleans up Metadata Center counters

### Key Subsystems

- **Load Balancing** (`pkg/aigateway/loadbalancer/`): Two-tier — global LB selects cluster, then `inference_lb` scores hosts by queue depth + prompt length + KV-cache hit rate. Register new algorithms via `manager.RegisterLbType`.
- **Prediction** (`pkg/prediction/`): Online RLS models — TTFT uses 6-parameter polynomial RLS; TPOT uses piecewise-linear RLS segmented by batch-size thresholds. Fallback to EMA bucketed by prompt length when `AIGW_USE_MOVING_AVERAGE` is set.
- **Metadata Center** (`pkg/metadata_center/`): HTTP client to external Metadata Center service for load stats, prompt tracking, and KV-cache queries. Uses async worker pool with retry/failover and circuit breaking.
- **Transcoder** (`plugins/llmproxy/transcoder/`): Protocol abstraction — implement `Transcoder` interface and register via `RegisterTranscoderFactory`. OpenAI implementation is in `transcoder/openai/`.
- **Cluster Manager** (`pkg/aigateway/clustermanager/`): Provides cluster info to LB. Implement `types.ClusterInfoProvider` to add new providers.

### Build Tags

- `so` — required for shared object build
- `envoydev` — default Envoy API version tag (used in builds, tests, and linting)
- `integrationtest` — for integration test SO builds (no integration tests exist yet)

## Code Conventions

- **Go 1.22**, Apache-2.0 license headers on all `.go` files (`make fix-license`)
- Protobuf-generated files (`*.pb.go`, `*.pb.validate.go`) must not be hand-edited; use `make gen-proto`
- JSON serialization uses `github.com/bytedance/sonic` (not `encoding/json`)
- Logging uses `mosn.io/htnn/api/pkg/filtermanager/api` (`api.LogInfof`, `api.LogDebugf`, etc.)
- Configuration is injected via environment variables (see `AIGW_*` constants throughout the codebase)
- The `openai-go` dependency is replaced with a fork at `github.com/aigw-project/openai-go`

## CI

- **test.yml**: `make unit-test` + `make build-so` on push/PR to `main` or `release/**`
- **lint.yml**: `make lint-go` + `make lint-license` on push/PR to `main` or `release/**`