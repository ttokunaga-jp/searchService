# Security Assurance & Vulnerability Management

This playbook defines how `searchService` prevents, detects, and responds to security issues. For functional and performance testing, see `docs/ci-cd.md`. Everything below focuses on code scanning, dependency hygiene, and operational follow-up.

## Security Scans

- **SAST** – `gosec ./...` (`.github/workflows/security.yaml`, job `GoSec SAST`).
- **SCA / Config scan** – `trivy fs` with HIGH/CRITICAL severities, ignores unfixed issues by default.
- **DAST** – ZAP baseline scan against the metrics endpoint (`http://127.0.0.1:9464/metrics`). The stack is launched with `docker compose up search-service` so we scan the actual container build. DAST runs on the nightly cron (`03:30 UTC`) or via manual trigger.
- All security jobs may be re-run locally:

```bash
go install github.com/securego/gosec/v2/cmd/gosec@v2.21.3
gosec ./...

docker run --rm -v "$PWD":/repo -w /repo aquasec/trivy:0.54.1 fs --severity CRITICAL,HIGH --ignore-unfixed .

docker compose up -d search-service
docker run --rm -v "$PWD":/zap/wrk -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t http://host.docker.internal:9464/metrics -m 2 -I
docker compose down -v
```

## Dependency Vulnerability Policy

| Severity | Allowed window | Action |
|----------|----------------|--------|
| CRITICAL / HIGH | Immediate | Patch or upgrade within **72 hours**. Accept only patch/minor updates that keep the same major version unless breaking fixes are unavoidable. |
| MEDIUM | 2 weeks | Batch into the next scheduled hardening sprint if no exploit exists. |
| LOW | 1 quarter | Address during regular dependency refresh cycles. |

Additional guidelines:

- Enforce [Semantic Versioning](https://semver.org/) – upgrades that cross a major boundary require design review and regression testing.
- Use Dependabot (or equivalent) to monitor `go.mod`, Docker base images, and GitHub Actions runners.
- All dependency bumps must pass `make ci`, integration tests, and the security workflow before merging.
- No direct commits to `main` for security fixes; use PRs so the automated gates run.

## Operational Follow-up

- **Alerting** – load-test or security job failures should page the owning team via GitHub code owners.
- **Artifact retention** – k6 JSON summaries and ZAP reports can be added by uploading artifacts in future iterations if deeper analysis is needed.
- **Runbook** – update `docs/security.md` whenever new scanners or thresholds are introduced so the workflow remains auditable.
