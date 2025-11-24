# Development Quality & CI Guide

This document is the entry point for everything that keeps `searchService` shippable: local verification, automated tests, CI workflow, and performance checks. Paired with `docs/security.md`, it guarantees we catch regressions before code reaches production.

## Local Verification Workflow

| Goal | Command | Notes |
| --- | --- | --- |
| Format enforcement | `make fmt-check` | Fails when any Go file (except `gen/`) diverges from `goimports`. |
| Proto hygiene | `make proto-lint` | Executes `buf lint` for all definitions under `api/`. |
| Static analysis | `make lint` | Runs `golangci-lint` with the repository default config. |
| Vet | `make vet` | `go vet ./...` for compiler-assisted diagnostics. |
| Unit tests | `make test-unit` | Race detector + coverage; runs fast enough for local loops. |
| Integration tests | `make test-integration` | Uses `testcontainers-go` to spin up Elasticsearch/Qdrant/Kafka (Docker required). |
| Aggregate check | `make ci` | Sequentially runs `fmt-check → proto-lint → lint → vet → test-unit`. |
| Container parity | `docker build -t search-service:local .` | Validates Docker build context when iterating locally. |

**Tooling prerequisites**

```bash
go install golang.org/x/tools/cmd/goimports@latest
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
brew install bufbuild/buf/buf   # or download from buf.build
```

## Test Matrix

| Layer | Command / Target | Scope | Dependencies | Notes |
| --- | --- | --- | --- | --- |
| Unit | `make test-unit` | Package-level logic with mocks | Go toolchain | Runs with `-race -cover`. Failing here blocks CI immediately. |
| Integration | `make test-integration` | Elasticsearch/Qdrant/Kafka adapters & repositories | Docker + testcontainers | Set `TESTCONTAINERS_RYUK_DISABLED=true` in CI to avoid privileged containers. |
| End-to-end | `make test-e2e` | Public gRPC surface exercised via in-memory endpoints | Go toolchain | Optional but recommended before releasing new RPCs. |
| Load / soak | `make test-load` (k6) | Sustained traffic against perf harness | Go toolchain + k6 | Default: 10 VUs / 30s. Override with `K6_VUS`, `K6_DURATION`. |

Developers can reproduce the full CI signal locally with:

```bash
make ci
TESTCONTAINERS_RYUK_DISABLED=true make test-integration
```

## GitHub Actions Pipeline

Workflow: `.github/workflows/ci.yaml`

1. Checkout repository and configure Go `1.24.7` with module caching.
2. Install Buf, `goimports`, and `golangci-lint`.
3. Run `make ci` to cover formatting, protobuf lint, static analysis, vet, and unit tests (`go test -race -cover ./...`).
4. Run `make test-integration` to validate Elasticsearch/Qdrant/Kafka flows inside containers.
5. On pushes to `main`, build the Docker image (`docker build -t search-service:${GITHUB_SHA} .`) to ensure the delivery artifact stays green.

Failures in any step block merges. Additional quality/supply-chain scans live in `docs/security.md`.

## Load & Performance Validation

- **Script**: `tests/perf/k6/search.js` (k6 gRPC scenario).
- **Harness**: `go run ./tests/perf/harness` spins up an in-memory backend so performance tests do not depend on external stores.
- **CI automation**: `.github/workflows/load-test.yaml` executes weekly (Sunday 18:00 UTC) and via `workflow_dispatch`. The job runs a 1 minute soak (`25 VUs`, `p95 < 500 ms` threshold).
- **Local workflow**:
  ```bash
  go run ./tests/perf/harness &
  make test-load
  kill %1
  ```
- **Tuning**: Override concurrency/duration with `K6_VUS`, `K6_DURATION`, and target via `GRPC_TARGET`.

## Release Checklist

1. `make ci` is green locally and in CI.
2. `make test-integration` passes with Docker.
3. Performance harness meets `p95 < 500 ms` and `error_rate < 0.1%`.
4. Any new proto changes have `buf lint` + generated code committed.
5. If security scans (GoSec, Trivy, ZAP) introduce new findings, follow the remediation SLA described in `docs/security.md`.

Maintaining this guide ensures engineers know exactly which levers to pull before opening or merging a PR.
