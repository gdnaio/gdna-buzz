<!-- Analyzed: 2026-07-25T01:12:08Z | Scope: full project -->
# Infrastructure & Deployment

## Summary

Buzz ships a **containerized relay** (multi-stage Rust + web build → debian-slim
runtime, published as `ghcr.io/block/buzz`) and a **Docker Compose** local dev stack
(Postgres, Redis, MinIO, Adminer, Prometheus, Keycloak). Production deployment is via
**Helm charts** under `deploy/charts/` (a `buzz` chart and a `buzz-push-gateway`
chart), oriented toward Kubernetes/ArgoCD per the ecosystem docs. CI/CD is a mature,
path-filtered **GitHub Actions** matrix (12 workflows) covering Rust lint/test,
desktop core + sharded Playwright E2E, web, mobile, security (cargo-deny), cross-compile
(musl x86_64/aarch64), Windows, and macOS Tauri builds. Desktop and mobile apps are
built and signed in separate internal pipelines (per `AGENTS.md`: `sprout-releases`,
`sprout-oss`, `block-coder-tf-stacks`). Database migrations are applied at relay boot
(`BUZZ_AUTO_MIGRATE`) and via `buzz-admin migrate`; a `pgschema`-based desired-state
apply path also exists for CI. Observability is Prometheus metrics (`:9102`) + optional
OpenTelemetry OTLP tracing.

## Containerization

### Service Images

| Component | Dockerfile | Base image | Multi-stage | Health check |
|---|---|---|---|---|
| buzz-relay (+ buzz-admin, buzz-pair-relay, web, admin-web bundles) | `Dockerfile` | `rust:1.95-bookworm` (build) → `debian:bookworm-slim` (runtime) | Yes — cargo-chef plan/cook/build, separate node web-builder, slim runtime | `/_liveness`, `/_readiness` (:8080) |
| buzz-push-gateway | `Dockerfile.push-gateway` | (separate) | Yes | HTTP probes |

Dockerfile highlights:
- **cargo-chef** dependency caching (plan → cook → build) for fast incremental image builds.
- **Multi-arch** via native amd64/arm64 runners (no `--platform` pins); see `.github/workflows/docker.yml`.
- Builds `buzz-relay`, `buzz-admin`, `buzz-pair-relay`; bundles `web/dist` + `admin-web/dist`.
- Runtime installs `git` (relay shells out for git upload-pack/receive-pack), runs as non-root `buzz:buzz` (uid/gid 1000).
- Optional corporate-proxy CA (`EXTRA_CA_CERTS`) and npm mirror (`NPM_REGISTRY`) build args.
- Exposes **3000** (app WS+REST), **8080** (liveness/readiness), **9102** (metrics).
- OCI labels set `org.opencontainers.image.source` so GHCR links + inherits repo visibility.

### Docker Compose (local dev — `docker-compose.yml`, project `buzz`)

| Service | Image | Ports | Notes |
|---|---|---|---|
| postgres | `postgres:17-alpine` | 5432 | `pg_isready` healthcheck; 512m limit; `buzz-postgres-data` volume |
| redis | `redis:7-alpine` | 6379 | `redis-cli ping` healthcheck; 128m limit |
| adminer | `adminer:latest` | 8082→8080 | DB web UI (dev only) |
| keycloak | `quay.io/keycloak/keycloak:26.0` | 8180→8080 | `start-dev`, in-mem DB — OIDC dev infra (not documented in ARCHITECTURE.md) |
| minio | `minio/minio:latest` | 9000 (API), 9001 (console) | S3-compatible media store; `buzz-minio-data` volume |
| minio-init | `minio/mc:latest` | — | Creates `buzz-media` bucket, sets anonymous none |
| prometheus | `prom/prometheus:latest` | 9090 | Mounts `./prometheus.yml`; scrapes relay via `host.docker.internal` |

- Network: `buzz-net` (bridge). Volumes: postgres-data, minio-data, prometheus-data.
- Additional compose files: `docker-compose.harness.yml` (benchmark harness), `deploy/compose/` (caddy + prod-style compose with `run.sh`, `Caddyfile`).

## CI/CD Pipeline

### GitHub Actions Workflows (`.github/workflows/`)

| Workflow | Purpose |
|---|---|
| `ci.yml` | Main gate — path-filtered jobs (see below) |
| `docker.yml` | Build + push relay image to GHCR (multi-arch) |
| `helm-chart.yml` / `push-gateway-helm-chart.yml` | Package + publish Helm charts |
| `release.yml` | Desktop/relay release automation |
| `auto-tag-on-release-pr-merge.yml` | Auto-tag on release PR merge |
| `mobile-release-candidate.yml` | Mobile RC publishing |
| `signed-macos-canary.yml` / `linux-canary.yml` / `windows-canary.yml` | Signed/unsigned desktop canary builds |
| `sprig.yml` | Sprig harness build |
| `benchmark-harbor.yml` | Orchestra benchmark |

### `ci.yml` Stages

| Stage | Tool | Trigger scope | Description |
|---|---|---|---|
| changes | dorny/paths-filter | all | Routes jobs by changed paths (rust/desktop/web/mobile) |
| rust-lint | just fmt-check / clippy | rust or desktop-rust | Workspace + Tauri fmt + clippy `-D warnings` |
| unit-tests | cargo-nextest | rust | `just test-unit` (core, auth, db-lib, conformance, push-gateway) |
| desktop-core | pnpm/biome/tsc/cargo | desktop | Lint, unit, build, Tauri clippy/check/test + compiled-flag verification |
| desktop-smoke-e2e | Playwright (4 shards) | desktop | Mock-bridge browser smoke tests |
| desktop-e2e-relay + integration (2 shards) | Playwright + live relay | desktop/rust | Relay-backed E2E; seeds `localhost:3000` community |
| backend-integration | cargo-nextest + docker | rust | Invite-claim security + NIP-ER reminder E2E against live relay |
| relay-e2e | cargo test | rust | Persona, nostr-interop, invite, membership E2E |
| web | just web-check/build | web | Biome + build |
| mobile | flutter analyze/test + APK | mobile | Format, analyze, test, debug APK |
| security | cargo-deny | rust | Dependency policy (`deny.toml`) |
| dead-token-guard | grep guard | all | Blocks reintroduction of dead API-token patterns in clients |
| server-cross-compile | cross (musl) | rust | x86_64 + aarch64 musl relay/agent/git binaries |
| windows-rust | cargo (MSVC) | rust/desktop-rust | Windows workspace + Tauri clippy/check/test |
| desktop-build-macos | tauri build | desktop | macOS Tauri build incl. mesh-llm llama native libs |

- **Characteristics**: PR-triggered with `cancel-in-progress`; heavy caching (rust-cache, pnpm store, Playwright browsers, Hermit, mesh-llama). Approval gates: none in-repo (handled by release PRs). Artifacts: relay binaries, Playwright reports, APK. Toolchain via Hermit (`cashapp/activate-hermit`).

## Infrastructure as Code

| Tool | Files | Resources managed |
|---|---|---|
| Helm | `deploy/charts/buzz/` (Chart.yaml, values.yaml, values.schema.json, Chart.lock, README) | Relay deployment (Kubernetes) |
| Helm | `deploy/charts/buzz-push-gateway/` (Chart.yaml, values.yaml, values-production.yaml, values.schema.json) | Push gateway deployment |
| Compose | `deploy/compose/` (compose.yml, compose.caddy.yml, compose.dev.yml, Caddyfile, run.sh, .env.example) | Self-host via Docker Compose + Caddy TLS |
| Local k8s | `deploy/local/` (quickstart-ha-values.yaml, build-and-deploy.sh) | Local HA quickstart |

Per `AGENTS.md`, the deployed staging cluster is provisioned by the external
`squareup/block-coder-tf-stacks` repo (Terraform + ArgoCD); the relay image is built
by `squareup/sprout-oss` and pushed to internal ECR. This OSS repo owns the chart,
Dockerfile, and application source.

### Cloud Resources (inferred)

| Name | Type | Purpose | Config location |
|---|---|---|---|
| Relay pods | K8s Deployment | WS+REST relay | `deploy/charts/buzz/values.yaml` |
| Postgres | Managed/StatefulSet | Event store | chart values / external |
| Redis | Managed/StatefulSet | Pub/sub, presence | chart values / external |
| S3 bucket | Object storage | Media (Blossom) + git packs | env (`BUZZ_GIT_*`, media vars); EKS Pod Identity noted in Cargo.toml |
| Push gateway | K8s Deployment | Web Push | `deploy/charts/buzz-push-gateway/` |

## Hosting / Runtime

- **Platform**: Kubernetes (Helm + ArgoCD, external stack) for hosted; Docker Compose for self-host/dev.
- **Compute**: Rust relay binary (debian-slim container), non-root.
- **Database**: PostgreSQL 17 (event store, FTS, audit, workflows), optional read replica (`READ_DATABASE_URL`).
- **Cache/events**: Redis 7 (pub/sub fan-out, presence SET EX, typing sorted sets).
- **Storage**: S3/MinIO (Blossom media, git pack cache); AWS EKS Pod Identity credentials supported (via patched aws-creds).
- **CDN**: none detected (media served via relay `/media/*`).

## Database Management

- **Migration tool**: sqlx-embedded migrations applied at relay boot (`BUZZ_AUTO_MIGRATE`) and via `buzz-admin migrate` (`just migrate`).
- **Location**: `migrations/` — 24 forward-only SQL files (`0001_initial_schema.sql` … `0024_event_ttl_refresh_shared_lock.sql`).
- **Desired-state path**: `schema/schema.sql` applied via `bin/pgschema apply` + `scripts/attach-schema-partitions.sql` in CI.
- **Partitioning**: monthly range partitioning on `events` and `delivery_log` via a custom partition manager (DDL-injection-guarded).
- **Seed data**: `scripts/seed-local-community.sh`, `scripts/setup-desktop-test-data.sh`, `scripts/seed-admin-dashboard.sh`.
- **Backup strategy**: not defined in-repo (delegated to hosting platform).

## Monitoring & Observability

| Aspect | Tool | Config | Coverage |
|---|---|---|---|
| Logging | `tracing` + `tracing-subscriber` (env-filter, json) | `RUST_LOG` env | Structured logs across relay + crates |
| Metrics | `metrics` + `metrics-exporter-prometheus` | `:9102/metrics`, `prometheus.yml` | Relay metrics scraped by Prometheus |
| Tracing | OpenTelemetry OTLP (gRPC) | `OTEL_EXPORTER_OTLP_ENDPOINT` (optional) | Distributed traces when configured |
| Health probes | Axum handlers | `/health`, `/_liveness`, `/_readiness` (:8080) | K8s liveness/readiness |
| Alerting | none in-repo | — | Delegated to hosting platform |
| Error tracking | none detected (no Sentry/etc.) | — | — |

<!-- CI security scanning (cargo-deny) and per-crate security posture detailed in security.md. -->
