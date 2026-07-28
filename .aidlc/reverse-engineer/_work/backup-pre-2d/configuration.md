<!-- Analyzed: 2026-07-25T01:12:08Z | Scope: full project -->
# Configuration & Environment

## Summary

Configuration is **environment-variable-first**, loaded from a `.env` file (copied from
`.env.example` by `just bootstrap`) with `set dotenv-load := true` in the `Justfile`.
**140 distinct `BUZZ_*`/infra env vars are read by the Rust workspace**, plus 27 read by
the Tauri desktop shell (including 7 compile-time `BUZZ_BUILD_*` flags baked in by
`build.rs`) and 6 Vite `import.meta.env` flags in the frontends. All local-dev defaults
work out of the box against Docker Compose. Secrets management is **env-var based** with
no vault integration in-repo; the desktop app stores identity keys in the **OS keychain**
(`secret_store.rs`, `app_state_keyring.rs`) rather than env. A separate declarative
feature-flag file (`preview-features.json`) gates preview UI features per platform.
Notable drift: `.env.example` still documents **Typesense** (`TYPESENSE_API_KEY`,
`TYPESENSE_URL`) although search moved to Postgres FTS, and Keycloak runs in
docker-compose without any documented env surface.

### Batch 2b findings (service crates)

The service crates are almost entirely **configuration-free by design** — they receive
configuration as constructor arguments from the relay rather than reading the environment.
`buzz-pubsub` reads **zero** env vars in any production path (its single `env::var` is a
`REDIS_URL` lookup inside a `#[cfg(test)]` module at
`crates/buzz-pubsub/src/nip98_replay.rs:110`).

Three configuration defects surfaced:

**Three of seven `BUZZ_RATE_LIMIT_*` variables are dead.**
`AGENT_STANDARD_API_CALLS`, `AGENT_ELEVATED_MESSAGES`, and `AGENT_PLATFORM_MESSAGES` are
parsed at `crates/buzz-relay/src/config.rs:303-314` and read nowhere. Only four reach
enforcement. Operators can set the dead three and observe no behaviour change. This sits
alongside the larger documentation error that rate limiting isn't enforced at all — it is
(see `security.md` SEC-13).

**Media size limits are documented at the wrong layer.** `ARCHITECTURE.md`'s "50 MB limit"
is a *relay default for images only*; `buzz-media` itself enforces video 500 MB and file
100 MB. An operator reading the architecture doc would size infrastructure for the wrong
ceiling.

**`BUZZ_REQUIRE_MEDIA_GET_AUTH` defaults to false**, so Blossom media reads are
unauthenticated out of the box even though `verify_blossom_get_auth` is implemented.

Operationally significant items that are **not** configurable and arguably should be, all
in `buzz-pubsub`: presence TTL (90 s, `src/presence.rs:16`), all four channel capacities
(4096 each, `src/lib.rs:126-129`), and the reconnect backoff envelope (1 s → 30 s,
triplicated). One knob that *was* built is unreachable —
`PubSubConfig::with_unsubscribe_debounce` (`src/lib.rs:93`) has no production caller
because every construction goes through `PubSubManager::new`, which hardcodes the 500 ms
default (`src/lib.rs:117-119`).

Two smaller inconsistencies: the relay's Redis default is spelled `localhost:6379`
(`crates/buzz-relay/src/config.rs:419`) while `buzz-pubsub`'s test helper hardcodes
`127.0.0.1:6379` (`src/lib.rs:371-376`) and ignores `REDIS_URL` — different resolution on
IPv6-preferring hosts. And nothing anywhere asserts a Redis `noeviction` policy, which the
NIP-98 replay seen-set silently depends on.

### Batch 2c configuration (relay, mesh, conformance)

- **The most security-relevant default is undocumented.** `BUZZ_REQUIRE_AUTH_TOKEN` defaults
  to **false** (`crates/buzz-relay/src/config.rs:475-477`) and is **absent from the root
  `.env.example`**, so the header-impersonation path (SEC-29 in `security.md`) is open on a
  default deployment with no configuration signal. The one place it *is* surfaced is the
  compose bundle template, which sets it to `true`
  (`deploy/compose/.env.example:16`) — so a compose-based deployment is safe by default while
  a `.env.example`-derived one is not.
- **The health listener bind address is hard-coded `0.0.0.0`**
  (`crates/buzz-relay/src/main.rs:1116`, `:1119`) — not configurable — and it is what exposes
  the unauthenticated `GET /_mesh`.
- **`ARCHITECTURE.md:624`'s "50 MB" is actually 500 MB** in the router's configured limit
  (`crates/buzz-relay/src/router.rs:33-37`, applied at `:45`). Still true at `07d0265c`;
  neither file changed in the 2c sync — the earlier `:623` citation was off by one line and
  is corrected here to match §6 of this document.
- **`ARCHITECTURE.md:824` documents `/api/presence`, which does not exist.**
- **A subsystem whose entire purpose is switchable observation has no switch.**
  `buzz-conformance` has no `[features]` table (`crates/buzz-conformance/Cargo.toml`, 35
  lines), no `#[cfg(feature = …)]` anywhere in `src/`, and no env var or config field selecting
  a tracer. `buzz-relay`'s dependency is unconditional
  (`crates/buzz-relay/Cargo.toml:20`), so the crate compiles into every relay build. Two
  comments promise a feature gate that does not exist
  (`crates/buzz-conformance/src/lib.rs:321-322`,
  `crates/buzz-relay/src/conformance/tracers.rs:9-13`).
- **Switching to a recording tracer requires a code edit.** The only binding site is
  `crates/buzz-relay/src/state.rs:798`; `JsonlTracer::create` takes its output path as a plain
  argument with no default, env fallback, or directory convention
  (`conformance/tracers.rs:37-45`).
- **The one conformance env var is test-only and presence-checked, not value-parsed** —
  `BUZZ_CONFORMANCE_UPDATE` regenerates golden fixtures whether set to `1` or `0`
  (`crates/buzz-conformance/tests/replay_fixtures.rs:210-214`).
- **An observability path costs a Postgres round-trip per read, gated on the wrong
  condition.** `db.communities_of_channels` runs per REQ filter and per search page
  (`handlers/req.rs:345`, `:661`) gated on `trace_state.is_some()` (`:334`, `:649`), and
  `trace_state` depends only on whether the authenticated pubkey bytes parse (`:116-118`) —
  not on which tracer is bound. The `Tracer` trait has no `is_recording()` to short-circuit it
  (`crates/buzz-conformance/src/lib.rs:314-318`).

## Environment Variables

Grouped by concern. "Required" means the system will not function correctly without it in
a non-default deployment. Full canonical reference: [`.env.example`](../../.env.example).

### Core infrastructure

| Variable | Purpose | Module | Required | Default | Sensitive |
|---|---|---|---|---|---|
| `DATABASE_URL` | Postgres connection string | buzz-db, relay, admin | yes | `postgres://buzz:buzz_dev@localhost:5432/buzz` | yes |
| `READ_DATABASE_URL` | Optional read-replica DSN | buzz-db | no | unset (reads go to writer) | yes |
| `PGHOST`/`PGPORT`/`PGUSER`/`PGPASSWORD`/`PGDATABASE` | libpq-style overrides (scripts, CI) | scripts/CI | no | localhost/5432/buzz/buzz_dev/buzz | yes |
| `REDIS_URL` | Redis connection string | buzz-pubsub | yes | `redis://localhost:6379` | no |
| `BUZZ_REDIS_POOL_SIZE` | Shared Redis pool size | relay | no | 16 | no |
| `BUZZ_AUTO_MIGRATE` | Apply embedded migrations at boot | relay | no | (boot-time gate) | no |
| `BUZZ_TEST_DATABASE_URL` | Test-only DB DSN | tests | no | unset | yes |

### Relay server

| Variable | Purpose | Module | Required | Default | Sensitive |
|---|---|---|---|---|---|
| `BUZZ_BIND_ADDR` | App bind address (WS + REST) | relay | no | `0.0.0.0:3000` | no |
| `RELAY_URL` | Public WS URL used in NIP-42 challenges | relay | yes (prod) | `ws://localhost:3000` | no |
| `BUZZ_RELAY_BASE_URL` | Public base URL for links | relay | no | derived | no |
| `BUZZ_HEALTH_PORT` | Liveness/readiness port | relay | no | 8080 | no |
| `BUZZ_METRICS_PORT` | Prometheus metrics port | relay | no | 9102 | no |
| `BUZZ_RELAY_PRIVATE_KEY` | Stable relay signing key | relay | no (dev) / yes (prod) | ephemeral | **yes** |
| `BUZZ_RELAY_PUBKEY` | Relay identity pubkey | relay/clients | no | derived | no |
| `RELAY_OWNER_PUBKEY` | Relay owner identity | relay | no | unset | no |
| `BUZZ_MAX_CONNECTIONS` | Connection semaphore capacity | relay | no | (code default) | no |
| `BUZZ_MAX_CONCURRENT_HANDLERS` | EVENT/REQ handler semaphore | relay | no | 1024 | no |
| `BUZZ_MAX_FRAME_BYTES` | Max WS frame size | relay | no | 65536 | no |
| `BUZZ_SEND_BUFFER` | Per-connection send buffer | relay | no | (code default) | no |
| `BUZZ_SLOW_CLIENT_GRACE_LIMIT` | Full-buffer strikes before disconnect | relay | no | 3 | no |
| `BUZZ_CORS_ORIGINS` | Allowed CORS origins | relay | no | (code default) | no |
| `BUZZ_WEB_DIR` | Serve web SPA from directory | relay | no | unset (`/srv/buzz/web` in image) | no |
| `BUZZ_ADMIN_HOST` / `BUZZ_ADMIN_WEB_DIR` | Admin dashboard host + bundle | relay | no | unset (inert) | no |
| `BUZZ_SERVE_GIT_WEB_GUI` | Enable repo-browser web routes | relay | no | false | no |
| `BUZZ_REAPER_INTERVAL_SECS` | Ephemeral-channel reaper cadence | relay | no | 60 | no |
| `BUZZ_EPHEMERAL_TTL_OVERRIDE` | Force TTL for ephemeral channels | relay | no | unset | no |
| `BUZZ_RECONCILE_CHANNELS` | Emit missing channel discovery events at boot | relay | no | false | no |
| `BUZZ_NIP43_RECONCILE_INTERVAL_SECS` | Membership roster reconcile cadence | relay | no | (code default) | no |
| `BUZZ_COMMUNITY_REVALIDATE_INTERVAL_SECS` | Community/host map refresh | relay | no | (code default) | no |
| `BUZZ_HUDDLE_AUDIO_AVAILABLE` | Advertise huddle audio capability | relay | no | (code default) | no |

### Authentication & authorization

| Variable | Purpose | Module | Required | Default | Sensitive |
|---|---|---|---|---|---|
| `BUZZ_PRIVATE_KEY` | Client/agent Nostr identity key | acp, cli, agent, desktop | yes (agents) | unset | **yes** |
| `NOSTR_PRIVATE_KEY` | Alternate key var (git tooling) | git-sign-nostr | no | unset | **yes** |
| `BUZZ_API_TOKEN` / `BUZZ_AUTH_TAG` | API token + auth tag (harness-injected) | cli, acp | no | unset | **yes** |
| `BUZZ_REQUIRE_AUTH_TOKEN` | Require token auth | relay | no | (code default) | no |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | Gate access on relay roster membership | relay | no | false | no |
| `BUZZ_ALLOW_NIP_OA_AUTH` | Allow NIP-OA auth path | relay | no | false | no |
| `BUZZ_PUBKEY_ALLOWLIST` | Restrict to listed pubkeys | relay | no | unset | no |
| `BUZZ_TERMS_OF_SERVICE_MARKDOWN` / `BUZZ_PRIVACY_POLICY_MARKDOWN` / `BUZZ_AGE_ATTESTATION_REQUIRED` | Join-policy documents + age gate | relay | no | unset (policy off) | no |
| `BUZZ_TEST_OWNER_PRIVATE_KEY` / `BENCH_PRIVATE_KEY` | Test/benchmark identities | tests, benchmarks | no | unset | **yes** |

### Rate limiting (Redis-backed admission)

| Variable | Purpose | Module | Required | Default | Sensitive |
|---|---|---|---|---|---|
| `BUZZ_RATE_LIMIT_HUMAN_MESSAGES_PER_MIN` | Human message cap | relay | no | 60 | no |
| `BUZZ_RATE_LIMIT_HUMAN_API_CALLS_PER_MIN` | Human API cap | relay | no | 300 | no |
| `BUZZ_RATE_LIMIT_HUMAN_WS_EVENTS_PER_SEC` | Human WS event cap | relay | no | 10 | no |
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_MESSAGES_PER_MIN` | Agent (standard) message cap | relay | no | 120 | no |
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_API_CALLS_PER_MIN` | Agent (standard) API cap | relay | no | 600 | no |
| `BUZZ_RATE_LIMIT_AGENT_ELEVATED_MESSAGES_PER_MIN` | Agent (elevated) cap | relay | no | 300 | no |
| `BUZZ_RATE_LIMIT_AGENT_PLATFORM_MESSAGES_PER_MIN` | Agent (platform) cap | relay | no | 600 | no |

> ⚠️ `ARCHITECTURE.md` §9 states rate limiting is **not enforced** (only a test stub
> `AlwaysAllowRateLimiter` exists), while `.env.example` documents these as
> "shared Redis-backed admission limits" and CI sets them to 100000. To be verified
> against `buzz-auth`/`buzz-relay` code in Phase 2 — flagged in `debt.md`.

### Media (Blossom / S3)

| Variable | Purpose | Module | Required | Default | Sensitive |
|---|---|---|---|---|---|
| `BUZZ_S3_BUCKET` / `BUZZ_S3_ENDPOINT` / `BUZZ_S3_REGION` | Media object store target | buzz-media | yes (prod) | MinIO local | no |
| `BUZZ_S3_ACCESS_KEY` / `BUZZ_S3_SECRET_KEY` | S3 credentials | buzz-media | yes (prod) | dev creds | **yes** |
| `AWS_REGION` | AWS region fallback | buzz-media | no | unset | no |
| `BUZZ_MEDIA_BASE_URL` | Public media URL base | relay | no | derived | no |
| `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS` | Global upload concurrency | relay | no | 8 | no |
| `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS_PER_PUBKEY` | Per-identity upload concurrency | relay | no | 2 | no |
| `BUZZ_MEDIA_UPLOADS_PER_MINUTE` | Upload rate cap | relay | no | 30 | no |
| `BUZZ_REQUIRE_MEDIA_GET_AUTH` (alias `BUZZ_REQUIRE_MEDIA_READ_AUTH`) | Require Blossom auth on media GET/HEAD | relay | no | false | no |
| `BUZZ_MAX_FILE_BYTES` / `BUZZ_MAX_IMAGE_BYTES` / `BUZZ_MAX_VIDEO_BYTES` / `BUZZ_MAX_GIF_BYTES` | Per-type size caps | relay | no | (code defaults) | no |
| `BUZZ_MEDIA_UPLOAD_IP_HEADER` / `BUZZ_MEDIA_UPLOAD_PORT_HEADER` | Trusted client-IP headers | relay | no | unset | no |
| `BUZZ_MEDIA_UPLOAD_RECORDS` | Upload record retention/telemetry | relay | no | (code default) | no |
| `BUZZ_STORAGE_SWEEP_INTERVAL_SECS` / `_MAX_OBJECTS` / `_TIMEOUT_SECS` / `BUZZ_STORAGE_METRICS` | Orphan-object sweeper | relay | no | (code defaults) | no |

### Git hosting (NIP-34)

| Variable | Purpose | Module | Required | Default | Sensitive |
|---|---|---|---|---|---|
| `BUZZ_GIT_REPO_PATH` | Ephemeral workspace root | relay | no | `./repos` | no |
| `BUZZ_GIT_MAX_PACK_BYTES` / `BUZZ_GIT_MAX_REPO_BYTES` | Size limits | relay | no | 500MB / 1GB | no |
| `BUZZ_GIT_MAX_REPOS_PER_PUBKEY` / `BUZZ_GIT_MAX_CONCURRENT_OPS` | Per-identity + global caps | relay | no | (code defaults) | no |
| `BUZZ_GIT_PACK_CACHE_PATH` / `_MAX_BYTES` / `_MAX_CONCURRENT_POPULATIONS` | Immutable pack cache | relay | no | `./repos/.pack-cache`, 5GB, 2 | no |
| `BUZZ_GIT_S3_BUCKET` / `_ENDPOINT` / `_REGION` / `_ACCESS_KEY` / `_SECRET_KEY` | Git-on-object-storage backend | relay | no | unset | **yes** (keys) |
| `BUZZ_GIT_HOOK_HMAC_SECRET` | HMAC for internal git policy hook | relay | yes (git enabled) | unset | **yes** |
| `BUZZ_GIT_PROBE_WRITERS` / `BUZZ_GIT_PROBE_ROUNDS` / `BUZZ_GIT_S3_PROBE` / `BUZZ_GIT_CONFORMANCE_PROBE` | Boot deployment probes | relay | no | (code defaults) | no |
| `GIT_CREDENTIAL_NOSTR_BIN` / `GIT_BASH` / `GIT_CONFIG_COUNT` | Git tooling resolution | dev-mcp, tests | no | resolved | no |

### Agent harness (buzz-acp)

Full list in `.env.example`. Key vars: `BUZZ_RELAY_URL`, `BUZZ_ACP_AGENT_COMMAND`
(default `goose`), `BUZZ_ACP_AGENT_ARGS`, `BUZZ_ACP_MCP_COMMAND`, `BUZZ_ACP_AGENTS`
(1–32, default 1), `BUZZ_ACP_MODEL`, `BUZZ_ACP_TURN_TIMEOUT` (320s),
`BUZZ_ACP_MAX_TURNS_PER_SESSION` (0=off), `BUZZ_ACP_SYSTEM_PROMPT[_FILE]`,
`BUZZ_ACP_INITIAL_MESSAGE`, `BUZZ_ACP_HEARTBEAT_INTERVAL` (0 or ≥10) +
`BUZZ_ACP_HEARTBEAT_PROMPT[_FILE]`, `BUZZ_ACP_SUBSCRIBE`
(`mentions`|`all`|`config`), `BUZZ_ACP_KINDS`, `BUZZ_ACP_CHANNELS`,
`BUZZ_ACP_NO_MENTION_FILTER`, `BUZZ_ACP_CONFIG`, `BUZZ_ACP_DEDUP` (`queue`|`drop`),
`BUZZ_ACP_NO_IGNORE_SELF`, `BUZZ_ACP_CONTEXT_MESSAGE_LIMIT` (0–100, default 12),
`BUZZ_ACP_NO_PRESENCE`, `BUZZ_ACP_NO_TYPING`, `BUZZ_ACP_EVENT_BUFFER` (256),
`BUZZ_ACP_ALLOWED_CHANNEL_ADD_POLICIES`. Legacy alias: `BUZZ_ACP_PRIVATE_KEY` →
`BUZZ_PRIVATE_KEY`. Agent shell: `BUZZ_SHELL` (Windows requires Git Bash).

### Push gateway

| Variable | Purpose | Module | Required | Default | Sensitive |
|---|---|---|---|---|---|
| `BUZZ_PUSH_GATEWAY_DELIVERY_URL` | Gateway delivery endpoint | relay | no | unset | no |
| `BUZZ_PUSH_GATEWAY_TIMEOUT_MS` | Delivery timeout | relay | no | (code default) | no |
| `BUZZ_PUSH_EXECUTOR_KEY_ID` | Signing key id for push executor | push-gateway | no | unset | **yes** |
| `BUZZ_PUSH_RUNTIME_DATABASE_ROLE` | Least-privilege DB role | push-gateway | no | unset | no |

### Mesh (opt-in `mesh-llm` / relay mesh)

`BUZZ_MESH`, `BUZZ_MESH_BIND_ADDR`, `BUZZ_MESH_ADVERTISE_ADDR`, `BUZZ_MESH_DEMO_ECHO`,
`MESH_LLM_NATIVE_RUNTIME_CACHE_DIR`, `MESH_ROLE`, `MESH_STACK_ROLE`, `MESH_STACK_SIZE`,
`MESH_OPENAI_BASE`, `MESH_SMOKE_MODEL`, `MESH_E2E_MODEL`.

### Pairing (NIP-AB)

`BUZZ_PAIRING_RELAY_URL`, `BUZZ_PAIR_RELAY_BIND_ADDR`, `BUZZ_UDS_PATH`.

### Observability & misc

`RUST_LOG` (default `buzz_relay=debug,buzz_db=debug,buzz_auth=debug,buzz_pubsub=debug,tower_http=debug`),
`OTEL_EXPORTER_OTLP_ENDPOINT` (optional), `BUZZ_AUDIT_ENABLED`,
`BUZZ_POOL_METRICS_INTERVAL_SECS`, `BUZZ_USAGE_METRICS_INTERVAL_SECS`,
`BUZZ_USAGE_METRICS_IDLE_TIMEOUT_SECS`, `BUZZ_USAGE_METRICS_PER_COMMUNITY`,
`BUZZ_CONFORMANCE_UPDATE`, `BUZZ_MANAGED_AGENT_START_NONCE`,
`BUZZ_E2E_GIT_COMMUNITY_ID`, `DATABRICKS_HOST`, `CODEX_CONFIG`/`CODEX_HOME`,
`GOOSE_PATH_ROOT`, `FAKE_MCP_*` (test doubles).

### Stale / drifting variables

| Variable | Status |
|---|---|
| `TYPESENSE_API_KEY`, `TYPESENSE_URL` | ⚠️ Documented in `.env.example` but search moved to Postgres FTS (`buzz-search`); no Typesense service in `docker-compose.yml` |
| Keycloak (`KEYCLOAK_ADMIN`, `KC_DB`) | ⚠️ Set only in `docker-compose.yml`; no documented relay integration |
| `BUZZ_REQUIRE_MEDIA_READ_AUTH` | Legacy alias retained for `BUZZ_REQUIRE_MEDIA_GET_AUTH` |
| `BUZZ_ACP_PRIVATE_KEY` | Legacy alias retained for `BUZZ_PRIVATE_KEY` |
| `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS` | `sprout`-prefixed legacy naming (set in CI) |

## Configuration Files

| File | Purpose | Format | Environment-Specific |
|---|---|---|---|
| `.env.example` → `.env` | Canonical env template (dev defaults) | dotenv | yes — copied and edited per developer |
| `Cargo.toml` (workspace) | Rust deps, profiles (`ci`, `sprig`), dep patch | TOML | no |
| `rust-toolchain.toml` | Pinned Rust 1.95.0 | TOML | no |
| `Justfile` | Task runner; `set dotenv-load := true` | just | flags via `mesh=1`, `fresh=1` |
| `docker-compose.yml` | Local dev stack | YAML | dev only |
| `docker-compose.harness.yml` | Benchmark harness stack | YAML | benchmark |
| `deploy/compose/{compose,compose.dev,compose.caddy}.yml` + `.env.example` | Self-host compose + TLS | YAML/dotenv | yes |
| `deploy/charts/buzz/values.yaml` (+ `values.schema.json`) | Relay Helm values | YAML/JSON-Schema | yes — per cluster |
| `deploy/charts/buzz-push-gateway/values{,-production}.yaml` | Gateway Helm values | YAML | yes — prod overlay |
| `deploy/local/quickstart-ha-values.yaml` | Local HA quickstart | YAML | local |
| `migrations/*.sql` (24) | Forward-only schema migrations | SQL | no |
| `schema/schema.sql` | Desired-state schema (pgschema) | SQL | no |
| `prometheus.yml` | Scrape config | YAML | dev |
| `biome.json` | JS/TS lint + format | JSON | no |
| `deny.toml` | cargo-deny dependency policy | TOML | no |
| `lefthook.yml` | Pre-commit / pre-push hooks | YAML | no |
| `renovate.json` | Dependency update policy | JSON | no |
| `preview-features.json` | Preview feature registry | JSON | per-platform |
| `pnpm-workspace.yaml` | JS workspace, overrides, patches | YAML | no |
| `desktop/src-tauri/tauri.conf.json` | Tauri app config, sidecars, updater | JSON | via `BUZZ_TAURI_CONFIG` per instance |
| `mobile/pubspec.yaml` | Flutter deps | YAML | no |
| `buzz-acp.toml` (optional) | Rule-based ACP subscriptions | TOML | per agent |
| `ct.yaml` | Helm chart-testing config | YAML | CI |

## Feature Flags

| Flag | Purpose | Default | Location | Mechanism |
|---|---|---|---|---|
| `workflows` | YAML automations with approval gates | preview | `preview-features.json` | Declarative registry (desktop) |
| `projects` | Git repository browser | preview | `preview-features.json` | Declarative registry |
| `pulse` | Activity feed | preview | `preview-features.json` | Declarative registry |
| `forum` | Forum-style channels | preview | `preview-features.json` | Declarative registry |
| `agentManagedProfiles` | Agents manage own name/avatar | preview | `preview-features.json` | Declarative registry |
| `mesh-llm` | On-device LLM mesh compute | off | Cargo feature; `just mesh=1` | Compile-time cargo feature |
| `dev` (buzz-auth) | Dev-only key derivation | off | `buzz-auth` `#[cfg(feature="dev")]` | Compile-time — **must stay off in prod** |
| `test-utils` | Test stubs (e.g. AlwaysAllowRateLimiter) | off | `buzz-auth` | Compile-time |
| `BUZZ_BUILD_OBSERVER_ARCHIVE_DEFAULT` | Observer archive default on | false | `desktop/src-tauri/build.rs` | Compile-time env flag (CI-verified both states) |
| `BUZZ_BUILD_AUTO_CONNECT_DEFAULT_RELAY` | Auto-connect to default relay | false | `desktop/src-tauri/build.rs` | Compile-time env flag |
| `BUZZ_BUILD_AGENT_METRIC_ARCHIVE_DEFAULT` | Agent metric archive default | false | `build.rs` | Compile-time |
| `BUZZ_BUILD_AGENT_ENV` / `BUZZ_BUILD_BUZZ_AGENT_MODEL` / `_PROVIDER` / `BUZZ_BUILD_RELAY_RECONNECT_CMD` | Internal-build agent wiring | unset | `build.rs` | Compile-time |
| `BUZZ_SERVE_GIT_WEB_GUI` | Repo-browser web routes | false | relay env | Runtime env var |
| `BUZZ_REQUIRE_MEDIA_GET_AUTH` | Media read auth rollout gate | false | relay env | Runtime env var |
| `VITE_BUZZ_FORCE_FRESH_ONBOARDING` | Replay onboarding each launch (dev) | off | Vite env | Build-time frontend flag |
| `VITE_NOSTR_BIND_PREVIEW` / `VITE_SHOW_TRANSCRIPT_ACP_SOURCE` | Frontend preview toggles | off | Vite env | Build-time frontend flag |

## Environments Detected

| Environment | Config Source | Key Differences |
|---|---|---|
| local dev | `.env` (from `.env.example`), `docker-compose.yml`, `just dev` | Local Postgres/Redis/MinIO, debug `RUST_LOG`, `ws://localhost:3000`, dev keyring service, ephemeral relay key |
| desktop standalone | `just desktop-standalone` (no relay/DB) | No infra; app prompts for a community; isolated keyring per instance |
| CI | `.github/workflows/ci.yml` env blocks | `BUZZ_REQUIRE_AUTH_TOKEN=false`, rate limits at 100000, `BUZZ_RECONCILE_CHANNELS=true`, pgschema-applied schema, pre-seeded `localhost:3000` community |
| staging | `just staging` → `BUZZ_RELAY_URL=wss://sprout-oss.stage.blox.sqprod.co` | Release-mode sidecars, internal staging relay |
| production | `just production` → `BUZZ_RELAY_URL=wss://buzz.block.builderlab.xyz` | Release-mode sidecars, production relay |
| container/prod relay | `Dockerfile` ENV + Helm values | `BUZZ_WEB_DIR=/srv/buzz/web`, `BUZZ_ADMIN_WEB_DIR=/srv/buzz/admin-web`, ports 3000/8080/9102, non-root user |
| benchmark | `docker-compose.harness.yml`, `BENCH_PRIVATE_KEY` | Isolated compose project `buzz-benchmark` |

## Secrets Management

- **Approach**: environment variables (`.env` locally, Helm values/platform secrets in
  deployment). No Vault/AWS Secrets Manager integration in this repo. The optional
  GitHub MCP token path is documented as external/org-specific.
- **Desktop identity keys**: stored in the **OS keychain** via
  `desktop/src-tauri/src/secret_store.rs` and `app_state_keyring.rs`; dev instances use
  a per-instance service name (`BUZZ_DEV_KEYRING_SERVICE`).
- **Cloud credentials**: S3 access via `BUZZ_S3_*`/`BUZZ_GIT_S3_*`, or AWS EKS **Pod
  Identity** (`AWS_CONTAINER_CREDENTIALS_FULL_URI` + token file) through the patched
  `aws-creds` fork pinned in `Cargo.toml`.
- **In-repo test keys**: `Justfile` `mesh-dev-fresh` hardcodes a public test
  `BUZZ_PRIVATE_KEY` and a `0x…01` relay key, explicitly labelled local-only — must
  never be pointed at staging/prod (noted in `security.md`).
- **Rotation strategy**: none automated in-repo.
- **Access pattern**: loaded at process startup (dotenv + `env::var`); desktop keys read
  on demand from the keychain.

## Configuration Patterns

- **Loading**: `just` loads `.env` (`set dotenv-load := true`); Rust reads via
  `std::env::var` in per-crate config modules (e.g. `buzz-acp/src/config.rs`, 1,903 LOC
  covering CLI flags + env + TOML precedence). Tauri reads env plus compile-time
  `BUZZ_BUILD_*` values injected by `build.rs`.
- **Validation**: partial — documented constraints (positive integers for rate limits,
  `BUZZ_ACP_AGENTS` 1–32, heartbeat 0 or ≥10, context limit 0–100) suggest parse-time
  validation in `buzz-acp`; relay-wide startup schema validation not confirmed in this
  pass (to verify in Phase 2). Helm values are validated by `values.schema.json`.
- **Overrides**: precedence is CLI flag > env var > TOML config file > code default
  (documented for `buzz-acp`); env overrides `.env` defaults; per-worktree dev ports and
  identities are derived by `scripts/instance-env.sh`.
- **Location**: `.env.example` (canonical documentation), `crates/buzz-acp/src/config.rs`
  (richest config surface), `desktop/src-tauri/build.rs` (compile-time flags).

<!-- Per-module configuration findings are appended during Phase 2. -->

---

# Phase 2 — Module Findings

## Module: buzz-core (`crates/buzz-core`)

### Aspect: Configuration

---

### 1. Environment variables

**None.** A recursive search of `crates/buzz-core/src` for `env::var` and `std::env` returns zero matches. The crate performs no environment reads, no file reads, and no config parsing — consistent with the zero-I/O charter stated at `crates/buzz-core/Cargo.toml:29`.

Runtime configuration for anything built on this crate (relay URL, keys, DB, Redis) is read by the consuming crates, not here.

---

### 2. Cargo features

| Feature | Declared | Default? | What it gates | Enabled by |
|---------|----------|----------|---------------|-----------|
| `test-utils` | `crates/buzz-core/Cargo.toml:10-11` (`test-utils = []`) | no (no `default` key exists) | `pub mod test_helpers` — `make_event` (`src/lib.rs:55`), `make_event_with_keys` (`:64`), `make_stored_event` (`:72`); the gate is `#[cfg(any(test, feature = "test-utils"))]` at `src/lib.rs:47` | `crates/buzz-relay/Cargo.toml:89`, inside that crate's `[dev-dependencies]` (`:86`) |

No other `#[cfg(feature = …)]` appears anywhere in `src/` — the only conditional compilation in the crate is `#[cfg(test)]` and the single `#[cfg(any(test, feature = "test-utils"))]`.

Notably, this crate has **no `dev` feature** (unlike `buzz-auth`, whose `dev` feature is switched on from `crates/buzz-relay/Cargo.toml:84`/`:90`).

---

### 3. Package metadata (all inherited from the workspace)

| Key | Value / source | file:line |
|-----|----------------|-----------|
| `name` | `buzz-core` | `crates/buzz-core/Cargo.toml:2` |
| `version` | `version.workspace = true` → `0.1.0` | `Cargo.toml:3`; root `Cargo.toml:35` |
| `edition` | `edition.workspace = true` → `2021` | `Cargo.toml:4`; root `Cargo.toml:36` |
| `rust-version` | `rust-version.workspace = true` → `1.88.0` | `Cargo.toml:5`; root `Cargo.toml:37` |
| `license` | `license.workspace = true` → `Apache-2.0` | `Cargo.toml:6`; root `Cargo.toml:38` |
| `repository` | `repository.workspace = true` → `https://github.com/block/sprout` | `Cargo.toml:7`; root `Cargo.toml:39` |
| `description` | "Core types, event verification, and filter matching for Buzz" | `Cargo.toml:8` |

Dependency versions are all workspace-inherited except one local pin: `percent-encoding = "2.3"` (`crates/buzz-core/Cargo.toml:26`).

---

### 4. Compile-time configuration (lints and assertions)

| Setting | Effect | file:line |
|---------|--------|-----------|
| `#![deny(unsafe_code)]` | any `unsafe` in the crate is a hard compile error | `src/lib.rs:1` |
| `#![warn(missing_docs)]` | undocumented public items warn | `src/lib.rs:2` |
| `const _: () = assert!(...)` × 25 | kind-range and u16-fit invariants are enforced at compile time; violating one fails the build | `src/kind.rs:783-820` |

No `[lints]` table exists in this crate's manifest and no `[workspace.lints]` exists in the root manifest, so lint configuration is entirely via these crate-level attributes.

---

### 5. Tunable constants (the crate's de-facto configuration surface)

These are compile-time constants, not runtime config. Changing one requires a rebuild of every dependent crate.

| Constant | Value | Governs | file:line |
|---|---|---|---|
| `EPHEMERAL_KIND_MIN` / `MAX` | 20000 / 29999 | ephemeral classification | `src/kind.rs:397`, `:399` |
| `PARAM_REPLACEABLE_KIND_MIN` / `MAX` | 30000 / 39999 | NIP-33 addressable classification | `src/kind.rs:392`, `:394` |
| `NIP44_MIN_CONTENT_LEN` | 132 | minimum accepted NIP-44 ciphertext length | `src/observer.rs:21` |
| `NIP44_MAX_CONTENT_LEN` | 87_472 | maximum accepted NIP-44 ciphertext length | `src/observer.rs:23` |
| `OBSERVER_MAX_PLAINTEXT_LEN` | 65_535 | observer plaintext cap | `src/observer.rs:25` |
| `OBSERVER_AGENT_TAG` / `OBSERVER_FRAME_TAG` | `"agent"` / `"frame"` | observer frame tag names | `src/observer.rs:13`, `:15` |
| `OBSERVER_FRAME_TELEMETRY` / `OBSERVER_FRAME_CONTROL` | `"telemetry"` / `"control"` | observer frame direction values | `src/observer.rs:17`, `:19` |
| `CORE_SLUG` | `"core"` | reserved engram slug | `src/engram.rs:20` |
| `D_TAG_DOMAIN` | `b"agent-memory/v1/d-tag"` | HMAC domain separator; doc says future revisions MUST change it | `src/engram.rs:22-24` |
| `NIP44_PLAINTEXT_MAX` | 65_535 | engram body cap | `src/engram.rs:28` |
| `SLUG_MAX_LEN` | 255 | engram slug byte cap (per-segment cap of 64 is an inline literal at `src/engram.rs:98`) | `src/engram.rs:31` |
| `MEM_PREFIX` (private) | `"mem/"` | engram slug namespace | `src/engram.rs:34` |
| `MAX_PROTECTION_RULES` | 50 | `buzz-protect` rules per repo | `src/git_perms.rs:19` |
| `MAX_PATTERN_LENGTH` | 256 | ref-pattern character cap | `src/git_perms.rs:21` |
| `MAX_WILDCARDS_PER_PATTERN` | 3 | wildcard segments per pattern | `src/git_perms.rs:23` |
| `ZERO_OID` (private, fn-local) | 40 `0` chars | create/delete detection | `src/git_perms.rs:213` |
| `DEFAULT_TIMEOUT` (private) | `Duration::from_secs(120)` | pairing session lifetime | `src/pairing/session.rs:43` |
| `PAIRING_KIND` (private) | `KIND_PAIRING as u16` = 24134 | pairing event kind | `src/pairing/session.rs:46` |
| `INFO_SESSION_ID` / `INFO_SAS` / `INFO_TRANSCRIPT` (private) | `"nostr-pair-session-id"`, `"nostr-pair-sas-v1"`, `"nostr-pair-transcript-v1"` | HKDF domain separation | `src/pairing/crypto.rs:23-25` |
| SAS modulus (inline literal) | `1_000_000` | 6-digit SAS space | `src/pairing/crypto.rs:73` |
| QR URI length cap (inline literal) | 2048 | max scanned URI length | `src/pairing/qr.rs:106` |
| pairing content-length window (inline literals) | `132..=87472` | inbound ciphertext gate — duplicates `observer::NIP44_MIN/MAX_CONTENT_LEN` rather than referencing them | `src/pairing/session.rs:595` |
| pairing plaintext cap (inline literal) | `65_535` | decrypted plaintext gate — duplicates `OBSERVER_MAX_PLAINTEXT_LEN` / `NIP44_PLAINTEXT_MAX` | `src/pairing/session.rs:609` |
| jitter window (inline literal) | `% 31` → 0–30 s | `created_at` metadata jitter | `src/pairing/session.rs:578` |

Non-configurable by design: nothing in the crate reads these from the environment, and there is no builder or options struct that would let a consumer override them at runtime.


## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: Configuration

This crate has **no runtime configuration surface**. Findings are recorded so the
absence is explicit rather than assumed.

---

### 1. Environment variables

| Variable | Read where | Notes |
|---|---|---|
| — | — | A search for `env::var`, `env!`, `option_env!`, and `std::env` across `crates/buzz-sdk/src/` returns **no matches**. No SDK behavior is environment-dependent. |

The only `std::env` use in the crate is `std::env::args()` in the example binary,
which reads positional CLI arguments, not environment variables
(`crates/buzz-sdk/examples/compute_auth_tag.rs:12`).

Note for contrast: the agent-facing environment variables documented for Buzz
(`BUZZ_RELAY_URL`, `BUZZ_PRIVATE_KEY`, `BUZZ_AUTH_TAG`) are consumed by
`buzz-cli` / the ACP harness, **not** by this crate — `buzz-sdk` has no relay
URL, no key source, and no transport.

---

### 2. Cargo features

| Feature | Gates | Declared at |
|---|---|---|
| — | — | `crates/buzz-sdk/Cargo.toml` has **no `[features]` section** (`Cargo.toml:1-17`) |

No `#[cfg(feature = ...)]` attribute exists anywhere in `src/` or `examples/`.
The only conditional compilation is `#[cfg(test)]` on the three test modules
(`builders.rs:1825`, `mentions.rs:389`, `nip_oa.rs:301`).

Feature selection the crate *inherits* (workspace-pinned, not switchable here):

| Dependency | Features enabled by the workspace | Declared at |
|---|---|---|
| `nostr` `0.44` | `nip44`, `nip98` | `Cargo.toml:61` (root) |
| `uuid` `1` | `v4`, `serde` | `Cargo.toml:89` (root) |
| `serde` `1` | `derive` | `Cargo.toml:64` (root) |

All six dependencies are declared as `{ workspace = true }`, so versions and
feature sets are centrally controlled (`crates/buzz-sdk/Cargo.toml:10-16`).

---

### 3. Package metadata

Everything except `name` and `description` is inherited from the workspace
(`crates/buzz-sdk/Cargo.toml:1-8`):

| Key | Value |
|---|---|
| `name` | `buzz-sdk` |
| `version` / `edition` / `rust-version` / `license` / `repository` | `.workspace = true` |
| `description` | `"Typed Nostr event builders for Buzz operations"` |

There is no `[[bin]]`, no `[lib]` override, no `build.rs`, and no
`[dev-dependencies]` section.

---

### 4. Compile-time behavior switches

| Switch | Effect | File:line |
|---|---|---|
| `#![deny(unsafe_code)]` | build fails on any `unsafe` | `crates/buzz-sdk/src/lib.rs:1` |
| `#![warn(missing_docs)]` | undocumented public items warn | `crates/buzz-sdk/src/lib.rs:2` |

---

### 5. Effective "configuration" — hardcoded limits

Because nothing is configurable at runtime, the crate's tunable behavior is
fixed in constants and inline literals. Consumers cannot override them.

| Limit | Value | File:line |
|---|---|---|
| `MENTION_CAP` | 50 mention p-tags | `mentions.rs:38` |
| `MAX_CONTACTS` | 10 000 NIP-02 contacts | `builders.rs:751` |
| `MAX_REASON_BYTES` | 64 UTF-8 bytes | `builders.rs:1704` |
| `CUSTOM_EMOJI_SET_D_TAG` | `"buzz:custom-emoji"` | `builders.rs:503` |
| Default content cap | 64 KiB (13 call sites) | `builders.rs:227`, `284`, `299`, `383`, `742`, `1087`, `1227`, `1335`, `1421`, `1468`, `1486`, `1795`, `1816` |
| Diff / patch content cap | 60 KiB | `builders.rs:314`, `1023` |
| Reaction emoji cap | 64 chars | `builders.rs:467` |
| Emoji shortcode cap | 64 bytes | `builders.rs:136` |
| Emoji / contact URL cap | 2048 bytes | `builders.rs:157`, `787` |
| Repo name / description / clone URL / relay caps | 128 / 1024 / 512 / 256 chars | `builders.rs:845`, `856`, `875`, `907` |
| Repo clone-URL and relay counts | 5 / 10 | `builders.rs:868`, `900` |
| Subject cap (issue, PR) | 256 chars | `builders.rs:1092`, `1340` |
| Petname cap | 256 bytes | `builders.rs:794` |
| DM participants | 1–8 | `builders.rs:1546` |
| NIP-OA `kind` clause range | 0–65535 | `nip_oa.rs:63` |
| NIP-OA `created_at` clause range | 0–4294967295 | `nip_oa.rs:65`, `67` |


## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: Configuration

### 1. Environment variables read by this crate

**None.** There is no `std::env::var`, `env::var_os`, `envy`, or `dotenv` usage anywhere in
`crates/buzz-persona/src/` or `crates/buzz-persona/tests/` (verified by grep — the only
`std::env` occurrences in the crate are prose inside
`crates/buzz-persona/PERSONA_PACK_SPEC.md`).

This is a stated design property, not an accident:

- `crates/buzz-persona/src/resolve.rs:11` — "**Pure**: no env access, no network, no side effects."
- `crates/buzz-persona/src/resolve.rs:359-364` — "Pure function — does NOT read the current
  process env. ACP is responsible for filtering based on operator precedence (level 1): if
  the operator already set an env var, ACP skips injection so the operator's value wins."

Consequence: this crate cannot be configured through the environment. All behavior is a
function of the pack directory contents plus the arguments passed in.

---

### 2. Environment variables *emitted* by this crate

The crate produces env vars as **data** for a consumer to inject into an agent subprocess.
`runtime_env_vars` — `crates/buzz-persona/src/resolve.rs:365-397`.

| Emitted key | Source field | Condition | Line |
|---|---|---|---|
| `BUZZ_AGENT_MODEL` | model id (post colon-split) | `runtime == "buzz-agent"` | `resolve.rs:375` |
| `BUZZ_AGENT_PROVIDER` | model provider prefix | `runtime == "buzz-agent"` **and** model contains a colon | `resolve.rs:376-378` |
| `GOOSE_PROVIDER` | model provider prefix | any other runtime (incl. `None`, `"goose"`, `"claude"`) **and** model contains a colon | `resolve.rs:381-383` |
| `GOOSE_MODEL` | model id (post colon-split) | any other runtime | `resolve.rs:384` |
| `GOOSE_TEMPERATURE` | `temperature.to_string()` | `temperature` is `Some` — **regardless of runtime** | `resolve.rs:391-393` |
| `GOOSE_CONTEXT_LIMIT` | `max_context_tokens.to_string()` | `max_context_tokens` is `Some` — **regardless of runtime** | `resolve.rs:395-397` |

The runtime-independence of the last two is deliberate and commented: "temperature and
context_limit stay as GOOSE_* (only goose reads them)"
(`crates/buzz-persona/src/resolve.rs:389`).

Consumer-side handling of these keys (outside this crate): the ACP `Config` carries them as
`persona_env_vars: Vec<(String, String)>` (`crates/buzz-acp/src/config.rs:533-535`) and
passes them to spawn as `extra_env` (`crates/buzz-acp/src/lib.rs:3733`).

A related consumer convention is mirrored in this crate's tests: the desktop import path
strips the four derived provider/model keys while preserving the "knob" keys —
`DERIVED_PROVIDER_MODEL_ENV_KEYS` at `crates/buzz-persona/tests/e2e_env_flow.rs:15-20`,
asserted at `:206-228`.

---

### 3. Cargo features

**None declared.** `crates/buzz-persona/Cargo.toml:1-18` has no `[features]` section, and a
grep for `cfg(feature` across the crate returns zero matches. There is no `default`
feature, no optional dependency, and therefore nothing gated by features.

The only conditional compilation in the crate is platform and test based:

| Directive | Location | Gates |
|---|---|---|
| `#[cfg(windows)]` | `crates/buzz-persona/src/pack.rs:334` | Windows drive-letter rejection in `safe_resolve` |
| `#[cfg(unix)]` | `crates/buzz-persona/src/pack.rs:589` | the symlink-escape test |
| `#[cfg(test)]` | `manifest.rs:195`, `merge.rs:200`, `pack.rs:447`, `persona.rs:331`, `resolve.rs:400`, `validate.rs:436` | per-module test modules |

Dependency feature selection is minimal: only `serde` enables a feature (`derive`) —
`crates/buzz-persona/Cargo.toml:10`. `serde_json`, `serde_yaml`, `thiserror`, and
`tempfile` use defaults.

---

### 4. Package-level configuration

`crates/buzz-persona/Cargo.toml:1-8`:

| Key | Value | Note |
|---|---|---|
| `name` | `buzz-persona` | |
| `version` | `0.1.0` | hard-coded, **not** `version.workspace = true` — unlike e.g. `crates/buzz-acp/Cargo.toml:3` |
| `edition` | `2021` | hard-coded; no `rust-version` key |
| `description` | `Parser and loader for Buzz persona pack files (.persona.md)` | |
| `license` | `Apache-2.0` | hard-coded |
| `repository` | `https://github.com/block/sprout` | stale — the OSS repo is `block/buzz` per `AGENTS.md` |

Workspace membership: `Cargo.toml:23` (`"crates/buzz-persona"`).

---

### 5. Compile-time constants (the crate's tunables)

Because there are no env vars or features, all tunable behavior is expressed as constants.
Changing any of them requires a recompile.

| Constant | Value | Visibility | Location | Governs |
|---|---|---|---|---|
| `MAX_FRONTMATTER_BYTES` | `1_048_576` (1 MiB) | `pub` | `crates/buzz-persona/src/persona.rs:21` | YAML frontmatter cap; also a term in the pack-level file cap |
| `MAX_BODY_BYTES` | `262_144` (256 KiB) | `pub` | `crates/buzz-persona/src/persona.rs:24` | persona prompt cap |
| `DEFAULT_THREAD_REPLIES` | `true` | private | `crates/buzz-persona/src/merge.rs:38` | built-in default (precedence level 5) |
| `DEFAULT_BROADCAST_REPLIES` | `false` | private | `crates/buzz-persona/src/merge.rs:39` | built-in default (level 5) |
| `MAX_NAME_LEN` | `64` | fn-local | `crates/buzz-persona/src/validate.rs:168` | persona name length (validator path) |
| inline `64` | `64` | literal | `crates/buzz-persona/src/resolve.rs:146` | persona name length (resolve path) — duplicated, not shared with `MAX_NAME_LEN` |
| inline trigger built-ins | `mentions=true`, `keywords=[]`, `all_messages=false` | literals | `crates/buzz-persona/src/merge.rs:181`, `:191`, `:196` and again `crates/buzz-persona/src/resolve.rs:264-266` | built-in trigger defaults — declared twice |
| `+ 100` / `+ 200` slack in size guards | — | literals | `crates/buzz-persona/src/persona.rs:266` (`+ 100`), `crates/buzz-persona/src/pack.rs:154` and `:169` (`+ 200`) | delimiter/whitespace headroom above the two caps; the two paths use different slack values |
| `KNOWN_MANIFEST_KEYS` | 16 entries | private | `crates/buzz-persona/src/validate.rs:99-118` | which `plugin.json` top-level keys avoid an "unknown key" warning |
| `KNOWN_BEHAVIORAL_KEYS` | 8 entries (incl. legacy `respond_to`) | private | `crates/buzz-persona/src/validate.rs:121-130` | which `defaults` keys avoid a warning |
| `KNOWN_RESPOND_TO_KEYS` | `mentions`, `keywords`, `all_messages` | private | `crates/buzz-persona/src/validate.rs:133` | which `defaults.triggers` sub-keys avoid a warning |

`KNOWN_MANIFEST_KEYS` full list (`crates/buzz-persona/src/validate.rs:99-118`):
`$schema`, `id`, `name`, `version`, `description`, `author`, `license`, `homepage`,
`repository`, `keywords`, `engines`, `personas`, `defaults`, `pack_instructions`,
`hooks_config`, `mcp_config`.

Drift note: `KNOWN_MANIFEST_KEYS` includes `repository`
(`crates/buzz-persona/src/validate.rs:108`), but `PackManifest` has no `repository` field
(`crates/buzz-persona/src/manifest.rs:79-121`) — the key is warning-exempt yet discarded at
parse time.

---

### 6. Pack-file configuration surface (what a pack author can configure)

This is the crate's real "configuration" input. Paths are pack-relative and resolved
against the canonicalized pack root.

| Config location | Purpose | Discovery rule |
|---|---|---|
| `.plugin/plugin.json` | pack identity + persona list + pack-wide `defaults` | fixed path, required (`crates/buzz-persona/src/pack.rs:132-135`) |
| `<manifest>.personas[]` | which `.persona.md` files to load | explicit list only — no directory globbing; each must exist (`crates/buzz-persona/src/pack.rs:155-164`) |
| `<manifest>.pack_instructions` → else `instructions.md` | pack-level instruction text | explicit path preferred; implicit `instructions.md` at pack root as fallback (`crates/buzz-persona/src/pack.rs:167-188`) |
| `<manifest>.mcp_config` → else `.mcp.json` | shared MCP server map under key `mcpServers` | explicit path preferred; implicit `.mcp.json` at pack root as fallback (`crates/buzz-persona/src/pack.rs:191-220`); shape read at `crates/buzz-persona/src/resolve.rs:285` |
| `skills/<name>/SKILL.md` | skill definitions | presence of `skills/` recorded (`crates/buzz-persona/src/pack.rs:223-230`); directory entries enumerated by `resolve_skills` (`:275-293`) and by the validator (`crates/buzz-persona/src/validate.rs:375-386`) |
| `<manifest>.hooks_config` | pack-level lifecycle hooks | **parsed into `PackManifest` but never loaded** — dropped with an explicit comment at `crates/buzz-persona/src/pack.rs:111-112` |
| `<manifest>.defaults` | behavioral defaults (precedence level 4) | `crates/buzz-persona/src/manifest.rs:49-70`; re-encoded to JSON at `crates/buzz-persona/src/pack.rs:143-147` |
| `.persona.md` frontmatter | per-persona identity + behavioral config (level 3) | `crates/buzz-persona/src/persona.rs:176-196` |

Not honored / not searched:

- No user-level or global persona path. The crate never reads `~/.buzz/packs/`,
  `$XDG_CONFIG_HOME`, or any well-known location — the pack directory is always an
  explicit `&Path` argument (`load_pack`, `resolve_pack`, `resolve_persona_by_name`,
  `validate_pack`). The `~/.buzz/packs/<pack-id>/` convention appears only as prose in
  `crates/buzz-persona/PERSONA_PACK_SPEC.md:993-996` and is the consumer's responsibility.
- No config-file layering, no `.env`, no TOML/INI config for the crate itself.
- No persona directory auto-discovery: a `.persona.md` not listed in `personas[]` is
  invisible to `load_pack`.
- `engines.buzz` is parsed (`crates/buzz-persona/src/manifest.rs:39-40`) but never
  evaluated — no minimum-version gate exists in this crate.

---

### 7. The five-level precedence model, and which levels this crate implements

Documented at `crates/buzz-persona/src/merge.rs:1-9` and
`crates/buzz-persona/PERSONA_PACK_SPEC.md:625-637`.

| Level | Source | Implemented here? | Evidence |
|---|---|---|---|
| 1 | Operator env vars already set in the parent process | **No** — explicitly the consumer's job | `crates/buzz-persona/src/resolve.rs:359-364` |
| 2 | Desktop UI per-agent overrides | **No** — runtime concern | `crates/buzz-persona/src/merge.rs:9` |
| 3 | Per-persona frontmatter | **Yes** | persona arm of `merge_behavioral_config` (`crates/buzz-persona/src/merge.rs:68-73`) |
| 4 | Pack `defaults` in `plugin.json` | **Yes** | defaults iteration (`crates/buzz-persona/src/merge.rs:66-74`) |
| 5 | Built-in hardcoded defaults | **Yes** | constants `crates/buzz-persona/src/merge.rs:38-39`; trigger built-ins `:180-198`; fallback `:159-168` |

Fields with **no** level-5 default (they stay `None` and the consumer decides):
`model`, `temperature`, `max_context_tokens` (`crates/buzz-persona/src/merge.rs:95-97`),
and `subscribe` when neither side sets it (`:114-121`) — though `LoadedPersona.subscribe`
collapses `None` to an empty `Vec` at `crates/buzz-persona/src/pack.rs:432`, so the
distinction is lost past that point.


## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: Configuration

Summary: this crate has **no runtime configuration mechanism of its own**. Everything
configurable is a function parameter or a compile-time constant.

---

### 1. Environment variables

**None.** Verified by search across `crates/buzz-ws-client/`: no `env::var`, no
`std::env`, no `dotenv`, no `option_env!`. The crate never reads process
environment. Environment-driven values such as `BUZZ_RELAY_URL`,
`BUZZ_PRIVATE_KEY`, and `BUZZ_AUTH_TAG` are resolved by the consuming crates and
passed in as arguments — e.g. `buzz-cli` supplies `ws_url`, `self.keys`, and
`self.auth_tag` at `crates/buzz-cli/src/client.rs:1080`.

---

### 2. Cargo features

| Feature | Declared? | Notes |
|---|---|---|
| (none) | No `[features]` section exists in `crates/buzz-ws-client/Cargo.toml` (file is 17 lines: `[package]` `:1`–`7`, `[dependencies]` `:9`–`17`) | No `#[cfg(feature = …)]` anywhere in the source (verified) |

All dependency features are inherited from the workspace root via
`{ workspace = true }` (`crates/buzz-ws-client/Cargo.toml:10`–`17`). The two that
change this crate's behaviour:

| Inherited feature set | Declared at | Effect on this crate |
|---|---|---|
| `tokio-tungstenite = { version = "0.29", features = ["rustls-tls-webpki-roots"] }` | root `Cargo.toml:113` | Determines the TLS backend and trust anchors used by `connect_async` (`connection.rs:53`) |
| `tokio = { version = "1", features = ["rt-multi-thread", "macros", "net", "time", "sync", "io-util", "signal", "process"] }` | root `Cargo.toml:43` | `time` powers `timeout`/`Instant` (`connection.rs:7`, `:134`, `:176`); `net` provides `TcpStream` in the stream alias (`connection.rs:14`) |
| `nostr = { version = "0.44", features = ["nip44", "nip98"] }` | root `Cargo.toml:61` | Event/keys/builder types (`message.rs:1`); neither listed feature is referenced by this crate's code |

The only `#[cfg]` in the crate is the test gate `#[cfg(test)]`
(`connection.rs:296`).

---

### 3. Compile-time constants (the crate's tunables)

| Constant | Value | Visibility | Where applied | file:line |
|---|---|---|---|---|
| `AUTH_CHALLENGE_TIMEOUT_SECS` | `20` | `pub` | `authenticate` challenge wait | decl `connection.rs:17`; use `:76` |
| `AUTH_OK_TIMEOUT_SECS` | `20` | `pub` | `authenticate` OK wait | decl `connection.rs:20`; use `:85` |
| `PUBLISH_OK_TIMEOUT_SECS` | `30` | `pub` | `send_event` OK wait | decl `connection.rs:23`; use `:99` |
| Challenge byte cap | `1024` (inline literal, not a named constant) | n/a | `wait_for_auth_challenge` socket-read path | `connection.rs:198` |

These are `const`, so they are not overridable at runtime; changing a timeout requires
a code edit. Lower bounds are guarded by compile-time assertions in the test module
(`connection.rs:300`–`313`), so an edit that reduces them below 20/20/30 fails to
build.

---

### 4. Caller-supplied configuration (the actual configuration surface)

| Parameter | Type | Function | file:line | Notes |
|---|---|---|---|---|
| `url` / `relay_url` | `&str` | `connect`, `connect_authenticated`, `publish_event` | `connection.rs:38`, `:48`, `:278` | No default; no scheme restriction |
| `keys` | `&Keys` | `connect_authenticated`, `authenticate`, `publish_event`, `build_auth_event` | `connection.rs:39`, `:72`, `:280`; `message.rs:177` | Signing identity |
| `auth_tag` | `Option<&Tag>` | same four | `connection.rs:40`, `:73`, `:281`; `message.rs:178` | Optional single authorization tag added to the AUTH event (`message.rs:182`–`186`) |
| `timeout_dur` | `Duration` | `next_event` | `connection.rs:106` | Per-call read budget; no default provided by the crate |
| `timeout_secs` | `u64` | `publish_event` | `connection.rs:282` | Whole-operation budget; `buzz-cli` passes `75` (`crates/buzz-cli/src/client.rs:1080`) |

Note the composition consequence: `publish_event`'s outer budget must exceed the sum
of the inner fixed waits (20 s challenge + 20 s auth OK + 30 s publish OK = 70 s) for
the inner errors to surface rather than the outer `Timeout`; `buzz-cli`'s `75`
(`crates/buzz-cli/src/client.rs:1080`) sits just above that sum, with a comment
referencing the three constants at `crates/buzz-cli/src/client.rs:1077`.

---

### 5. Logging configuration

`tracing::debug!` is used at three sites (`connection.rs:57`, `:91`, `:123`). The
crate installs no subscriber and reads no log-level configuration — verbosity is
controlled entirely by the host binary's `tracing` setup.

---

### 6. Package metadata

All inherited from the workspace: `version.workspace`, `edition.workspace`,
`rust-version.workspace`, `license.workspace`, `repository.workspace`
(`crates/buzz-ws-client/Cargo.toml:3`–`7`). No `build.rs`, no `[lib]` section
(default lib name `buzz_ws_client`), no `[[bin]]`, no `[dev-dependencies]`.
Registered in the workspace at root `Cargo.toml:16` and exposed as a workspace
dependency at root `Cargo.toml:134`.


## Module: buzz-db (`crates/buzz-db`)

### Configuration

#### 1. Environment variables

**Corrected by `2a051a40` (post-analysis sync).** The crate now reads exactly
**one** production environment variable, so the previous "reads no environment
variables in production code" statement no longer holds:

| Variable | Read at | Type / default | Semantics |
|----------|---------|----------------|-----------|
| `BUZZ_MAX_COMMUNITIES_PER_OWNER` | `crates/buzz-db/src/relay_members.rs:391` | positive `i64`; falls back to `MAX_COMMUNITIES_PER_OWNER` = `3` (`:379`) | per-owner community cap. Read **once per process** and cached in a `OnceLock` (`:387-396`), so a mid-process change has no effect. Parse/fallback is isolated in the pure `effective_owner_limit` (`:401-405`): missing, non-numeric, zero, and negative all resolve to `3`. Enforcement sites are unchanged — provisioning (`create_community_with_owner`) and ownership transfer inside the advisory-lock transaction. |

Two gaps come with it, both verified at `07d0265c`:

- **Undocumented in the canonical template.** The variable appears nowhere in
  the root `.env.example` (grep count 0), nor in `deploy/compose/.env.example`,
  so the only description of the knob is the Rust doc comment at
  `crates/buzz-db/src/relay_members.rs:381-386`. Same class of drift as the
  Typesense entries noted at `configuration.md:251`, but inverted: here the code
  has a variable the template does not.
- **No runtime reconfiguration path.** Because the `OnceLock` resolves on first
  use and is never invalidated (`relay_members.rs:388-395`), changing the cap
  requires a process restart. There is no admin event, HTTP endpoint, or SIGHUP
  path that re-reads it — consistent with the rest of the crate, but worth
  knowing before an operator raises the cap on a running relay.

Everything else still holds. Connection
details arrive as a `DbConfig` struct from the caller
(`crates/buzz-db/src/lib.rs:222-234`); the doc comments name the variables the
*relay* is expected to source them from, but the lookup happens outside this
crate:

| Variable | Referenced as | Where documented |
|----------|---------------|------------------|
| `DATABASE_URL` | source of `DbConfig::database_url` | `crates/buzz-db/src/lib.rs:225` |
| `READ_DATABASE_URL` | source of `DbConfig::read_database_url` (e.g. an Aurora `cluster-ro-` endpoint) | `crates/buzz-db/src/lib.rs:227-230` |
| `BUZZ_AUTO_MIGRATE` | relay-side switch that decides whether `Db::migrate` runs; affects fence-probe gating | `crates/buzz-db/src/lib.rs:437-441`; test at `:5946-5951` |

Test-only environment reads (all inside `#[cfg(test)] mod tests`):

| Variable | Precedence | Files |
|----------|-----------|-------|
| `BUZZ_TEST_DATABASE_URL` → `DATABASE_URL` → const default | first match wins | `crates/buzz-db/src/event.rs:1478-1480`, `crates/buzz-db/src/feed.rs:258-260`, `crates/buzz-db/src/git_repo.rs:190-192`, `crates/buzz-db/src/moderation.rs:639-641`, `crates/buzz-db/src/thread.rs:822-824`, `crates/buzz-db/src/push.rs:1270-1272`, `crates/buzz-db/src/workflow.rs:1702-1704`, `crates/buzz-db/src/migration.rs:1028-1030`, `crates/buzz-db/src/relay_members.rs:616-618`, `crates/buzz-db/src/product_feedback.rs:127-129` |
| `BUZZ_TEST_DATABASE_URL` → const default | no `DATABASE_URL` fallback | `crates/buzz-db/src/channel.rs:1794` |
| `TEST_DATABASE_URL` → const default | different name from the above | `crates/buzz-db/src/lib.rs:3936`, `:4655`, `:5228`; `crates/buzz-db/src/replica_fence.rs:515` |
| (none) | hard-coded const only | `crates/buzz-db/src/archived_identities.rs:130-135`, `crates/buzz-db/src/user.rs:406-408`, `crates/buzz-db/src/usage.rs:365-369`, `crates/buzz-db/src/api_token.rs` test setup |

The default test URL literal is
`postgres://buzz:buzz_dev@localhost:5432/buzz`, repeated as a
`const TEST_DB_URL` in 16 test modules.

#### 2. Cargo features

None. `crates/buzz-db/Cargo.toml` has no `[features]` section
(`crates/buzz-db/Cargo.toml:1-25`). All dependency feature selection is inherited
from the workspace table — most relevantly
`sqlx = { version = "0.9", features = ["runtime-tokio", "tls-rustls", "postgres", "uuid", "chrono", "json"] }`
(`Cargo.toml:52-54`). Package metadata (`version`, `edition`, `rust-version`,
`license`, `repository`) is all `.workspace = true`
(`crates/buzz-db/Cargo.toml:3-8`); the workspace pins `edition = "2021"` and
`rust-version = "1.88.0"` (`Cargo.toml:36-37`).

#### 3. Pool tunables (`DbConfig`)

Defined at `crates/buzz-db/src/lib.rs:222-234`, defaults at `:236-249`, applied
at `:387-393`.

| Field | Type | Default | Applied as |
|-------|------|---------|-----------|
| `database_url` | `String` | `postgres://buzz:buzz_dev@localhost:5432/buzz` | writer pool target |
| `read_database_url` | `Option<String>` | `None` (replica routing disabled) | replica pool target |
| `max_connections` | `u32` | `20` | `PgPoolOptions::max_connections` |
| `min_connections` | `u32` | `2` | `PgPoolOptions::min_connections` |
| `acquire_timeout_secs` | `u64` | `3` | `PgPoolOptions::acquire_timeout` |
| `max_lifetime_secs` | `u64` | `1800` | `PgPoolOptions::max_lifetime` |
| `idle_timeout_secs` | `u64` | `600` | `PgPoolOptions::idle_timeout` |

Sizing rationale recorded in the `Default` doc comment: staging measured 51 idle
+ 1 active out of 50, so 20 main + 5 audit = 25 connections per pod lets four
relay pods fit a Postgres `max_connections = 100`
(`crates/buzz-db/src/lib.rs:236-239`). Both pools use the **same** sizing
(`crates/buzz-db/src/lib.rs:361-364`), and `DbPoolStats::max` reports the
configured ceiling for either pool (`:494-511`).

`Db::from_pool` / `Db::from_pools` derive `max_connections` from
`pool.options().get_max_connections()` instead of a config value
(`crates/buzz-db/src/lib.rs:404-427`) and do **not** arm the floor guard, so
callers that build pools themselves are unguarded unless they set the GUC.

#### 4. Session GUCs (runtime configuration in Postgres)

| GUC | Set by | Scope | Effect |
|-----|--------|-------|--------|
| `buzz.created_at_floor` | writer-pool `after_connect` hook, value = `CREATED_AT_FLOOR_SECS` (`crates/buzz-db/src/lib.rs:394-405`) | session (`is_local = false`) | arms the deferred commit-time floor guard from `migrations/0021_created_at_fence_floor.sql:44-68`; unset/blank ⇒ guard is a no-op |
| `buzz.nip_rs_hard_delete` | `set_config(..., true)` inside the NIP-RS replacement transaction (`crates/buzz-db/src/lib.rs:3742-3747`) and inside the purge trigger (`migrations/0011_nip_rs_exact_tag_cardinality.sql:143`) | **transaction-local** | authorises the BEFORE DELETE fence at `migrations/0011_…:45-60`; verified not to leak across commit or rollback by test `crates/buzz-db/src/lib.rs:4194-4218` |

Tests arm the floor guard transaction-locally with
`set_config('buzz.created_at_floor', $1, true)`
(`crates/buzz-db/src/lib.rs:5811-5815`, `:5991-5995`).

#### 5. Compile-time constants

| Constant | Value | File:line | Meaning |
|----------|-------|-----------|---------|
| `replica_fence::CREATED_AT_FLOOR_SECS` | `960` | `crates/buzz-db/src/replica_fence.rs:51` | seconds of `created_at` history the commit-time guard tolerates; must exceed the relay's ±900 s ingest envelope. The GUC value and the fence subtrahend must never diverge. |
| `replica_fence::FENCE_CLOCK_MARGIN_SECS` | `5` | `crates/buzz-db/src/replica_fence.rs:59` | extra margin subtracted from the fence |
| `replica_fence::PROBE_INTERVAL` | `5 s` | `crates/buzz-db/src/replica_fence.rs:62` | probe cadence |
| `replica_fence::FENCE_STALENESS` | `30 s` | `crates/buzz-db/src/replica_fence.rs:66` | fence age after which it is treated as closed |
| `replica_fence::CLOSED` (private) | `i64::MIN` | `crates/buzz-db/src/replica_fence.rs:69` | closed sentinel |
| `feed::FEED_MAX_LIMIT` | `100` | `crates/buzz-db/src/feed.rs:29` | hard cap on every feed query |
| `workflow::LIST_DEFAULT_LIMIT` | `100` | `crates/buzz-db/src/workflow.rs:25` | default workflow list page |
| `workflow::LIST_MAX_LIMIT` | `1000` | `crates/buzz-db/src/workflow.rs:27` | workflow list ceiling |
| `admin_moderation::MAX_PAGE_SIZE` | `200` | `crates/buzz-db/src/admin_moderation.rs:15` | admin page clamp |
| `relay_members::MAX_COMMUNITIES_PER_OWNER` | `3` | `crates/buzz-db/src/relay_members.rs:379` | per-pubkey community ownership cap — **no longer hard-coded as of `2a051a40`**: it is now the *fallback* default, overridable per process by `BUZZ_MAX_COMMUNITIES_PER_OWNER` (read once and cached in `max_communities_per_owner`, `relay_members.rs:387-396`; parse/fallback rules in `effective_owner_limit`, `:401-405` — missing, unparseable, or non-positive values fall back to `3`) |
| `push::MAX_MATCH_ATTEMPTS` | `8` | `crates/buzz-db/src/push.rs:70` | matcher-job attempt ceiling |
| `push::PUSH_GATE_LOCK_NAMESPACE` (private) | `"buzz_push_gate:"` | `crates/buzz-db/src/push.rs:21` | advisory-lock key domain; must match `migrations/0023_push_match_gate.sql:34` |
| `push::PUSH_GATE_BACKFILL_SECS` (private) | `120` | `crates/buzz-db/src/push.rs:41` | activation backfill window over `received_at` |
| `event::D_TAG_MAX_LEN` | `1024` | `crates/buzz-db/src/event.rs:140` | documented ceiling, **not enforced in this crate** |
| `event::HUDDLE_LINK_CONTENT_MAX_BYTES` (private) | `512` | `crates/buzz-db/src/event.rs:147` | max candidate content size for huddle-link resolution |
| `event::HUDDLE_LINK_CANDIDATE_LIMIT` (private) | `32` | `crates/buzz-db/src/event.rs:150` | max candidate rows inspected |
| `partition::PARTITIONED_TABLES` (private) | `["events", "delivery_log"]` | `crates/buzz-db/src/partition.rs:12` | DDL allowlist |
| `moderation::MODERATION_ACTION_CHECK_VOCAB` | 12 strings | `crates/buzz-db/src/moderation.rs:104` | mirror of the SQL CHECK |

Hard-coded limits embedded directly in SQL rather than named constants:

| Limit | Value | Where |
|-------|-------|-------|
| Default query limit / clamp | 100 / 1000 | `crates/buzz-db/src/event.rs:350-352` |
| `list_channels`, `get_members`, `get_bot_members`, `get_accessible_channels` | `LIMIT 1000` | `crates/buzz-db/src/channel.rs:696`, `:713`, `:588`, `:903`, `:840` |
| `list_active_tokens` | `LIMIT 1000` | `crates/buzz-db/src/lib.rs:2396` |
| Active-token quota per (community, owner) | `< 10` | `crates/buzz-db/src/api_token.rs:119` |
| Thread/window participant cap | 10 | `crates/buzz-db/src/thread.rs:520`, `:698` |
| `list_dms_for_user` limit cap | 200 | `crates/buzz-db/src/dm.rs:233` |
| `search_users` limit clamp | `[1, 500]` | `crates/buzz-db/src/user.rs:233` |
| `list_workflow_runs` limit cap | 1000 | `crates/buzz-db/src/workflow.rs:824` |
| DM participant bounds | 2–9 | `crates/buzz-db/src/dm.rs:107-117` |
| `get_events_by_ids` debug bound | 500 | `crates/buzz-db/src/event.rs:997` |
| `UsageMetricsLeader::is_live` ping timeout | 5 s | `crates/buzz-db/src/lib.rs:216` |
| Push-eligible kinds (trigger allowlist) | `7, 9, 1059, 40007, 46010` | `migrations/0018_push_match_queue.sql:25`, `migrations/0023_push_match_gate.sql:26`, mirrored in Rust at `crates/buzz-db/src/push.rs:58` |
| FTS excluded / allowlisted kinds | see data-model §6 | `migrations/0001:223`, `0005:31`, `0008:15`, `0014:29` |
| TTL-refresh exempt kind | `9007` | `migrations/0024_event_ttl_refresh_shared_lock.sql:30` |
| NIP-29 discovery kinds soft-deleted together | `39000, 39001, 39002` | `crates/buzz-db/src/lib.rs:3287` |
| Declared partition ranges | 2026-01 … 2026-07 plus `_p_past`/`_p_future` | `migrations/0001_initial_schema.sql:237-252`, `:343-354` |

#### 6. Operational configuration expectations

- `ensure_future_partitions(months_ahead)` is meant to run on startup and monthly
  via cron; the caller chooses `months_ahead`
  (`crates/buzz-db/src/partition.rs:1-4`, `:15`).
- `spawn_fence_probe` must be called **after** the migration decision, otherwise
  an armed GUC with an unapplied migration 0021 could open a fence over an
  unenforced floor (`crates/buzz-db/src/lib.rs:434-448`).
- The probe role needs `pg_monitor`; without it the fence stays closed
  (`crates/buzz-db/src/replica_fence.rs:369-374`).
- Backfills of channel-bearing `events` rows must run on a connection **without**
  `buzz.created_at_floor` and with the replica breaker held closed
  (`migrations/0021_created_at_fence_floor.sql:26-42`).
- `prune_scheduled_workflow_fires_before`'s cutoff must be older than the largest
  supported interval schedule (`crates/buzz-db/src/workflow.rs:589-596`).
- Migration `0004` builds a GIN index on a partitioned parent without
  `CONCURRENTLY` (unsupported there), so brownfield application takes a share
  lock per partition and should run in a deploy window
  (`migrations/0004_events_tags_gin.sql:13-17`).


## Module: buzz-auth (`crates/buzz-auth`)

### Configuration

### Environment variables read by this crate

**None.** A grep of `crates/buzz-auth/` for `std::env`, `env::var`, `env!`, and
`option_env!` returns zero matches. The crate is configured purely by the
`AuthConfig` value its caller passes to `AuthService::new`
(`crates/buzz-auth/src/lib.rs:106-108`).

The seven `BUZZ_RATE_LIMIT_*` variables that populate this crate's
`RateLimitConfig` are read in `buzz-relay`
(`crates/buzz-relay/src/config.rs:284-316`) — see the summary section below.

---

### Cargo features

Declared in `crates/buzz-auth/Cargo.toml:10-12`. There is **no** `default`
feature set, so both are opt-in.

| Feature | Declared | Deps enabled | What it gates |
|---------|----------|--------------|---------------|
| `test-utils` | `crates/buzz-auth/Cargo.toml:11` | none (empty list) | The three test doubles and their root re-exports, all via `#[cfg(any(test, feature = "test-utils"))]`: `MockAccessChecker` struct + inherent impl + `Default` + `ChannelAccessChecker` impl (`crates/buzz-auth/src/access.rs:107`, `:112`, `:128`, `:135`); `AlwaysAllowRateLimiter` struct + `RateLimiter` impl (`crates/buzz-auth/src/rate_limit.rs:218`, `:221`); `AlwaysFreshReplayGuard` struct + `Nip98ReplayGuard` impl (`crates/buzz-auth/src/nip98_replay.rs:126`, `:129`); re-exports (`crates/buzz-auth/src/lib.rs:46-51`) |
| `dev` | `crates/buzz-auth/Cargo.toml:12` | none (empty list) | Exactly one item: `derive_pubkey_from_username` (`crates/buzz-auth/src/lib.rs:159-167`) — the `SHA-256("buzz-test-key:{username}")` deterministic key derivation |

Both features use the `any(test, feature = "...")` form, so the gated code is
also compiled during this crate's own `cargo test` run without the flag.

#### Who enables them

| Feature | Enabled by | Where | Effect on production build |
|---------|-----------|-------|----------------------------|
| `dev` | `buzz-relay`'s own `dev` feature (`dev = ["buzz-auth/dev"]`) | `crates/buzz-relay/Cargo.toml:84` | none unless `--features dev` is passed; no invocation in `justfile`, CI, or Dockerfiles |
| `dev` | `buzz-relay` `[dev-dependencies]` entry `buzz-auth = { workspace = true, features = ["dev"] }` | `crates/buzz-relay/Cargo.toml:90` | none for a non-test build: workspace uses `resolver = "2"` (`Cargo.toml:32`), which does not unify dev-dependency features into normal builds. `Dockerfile:67` passes no feature flags |
| `test-utils` | nobody | — | The feature is never requested by any crate in the repo (grep of all `Cargo.toml` for `buzz-auth` shows plain `{ workspace = true }` at `crates/buzz-pubsub/Cargo.toml:12`, `crates/buzz-admin/Cargo.toml:17`, `crates/buzz-relay/Cargo.toml:22`, and `features = ["dev"]` only at `crates/buzz-relay/Cargo.toml:90`) |

`buzz-relay`'s `[features]` table contains only `dev`
(`crates/buzz-relay/Cargo.toml:83-84`) — no `default`.

---

### Compile-time constants

All five constants in the crate, with values:

| Constant | Value | Visibility | Purpose | file:line |
|----------|-------|-----------|---------|-----------|
| `TIMESTAMP_TOLERANCE_SECS` | `60` (`u64`) | private to `nip42` | NIP-42 `created_at` skew window (symmetric via `abs_diff`) | `crates/buzz-auth/src/nip42.rs:35` |
| `TIMESTAMP_TOLERANCE_SECS` | `60` (`u64`) | private to `nip98` | NIP-98 `created_at` skew window; separate definition with the same name/value | `crates/buzz-auth/src/nip98.rs:32` |
| `DEFAULT_REPLAY_TTL_SECS` | `120` (`u64`) | `pub`, re-exported at root | Floor for the NIP-98 replay window = 2× the ±60s tolerance; implementations MAY raise, MUST NOT lower | `crates/buzz-auth/src/nip98_replay.rs:46`, re-export `crates/buzz-auth/src/lib.rs:38` |
| `MAX_REPLAY_TTL_SECS` | `3600` (`u64`) | `pub`, re-exported at root | Ceiling for the replay window; implementations MUST clamp down. Rationale: 30× the natural maximum and safely inside Redis `EX`'s i64 range | `crates/buzz-auth/src/nip98_replay.rs:59`, re-export `crates/buzz-auth/src/lib.rs:39` |

Three of these are pinned by const-drift tripwire tests:
`DEFAULT_REPLAY_TTL_SECS >= 120` (`crates/buzz-auth/src/nip98_replay.rs:210-218`),
`DEFAULT_REPLAY_TTL_SECS < MAX_REPLAY_TTL_SECS` (`:220-230`),
`MAX_REPLAY_TTL_SECS <= i64::MAX as u64` (`:232-237`).

Related constant defined outside the crate but consumed against this crate's
config: `WS_BURST_WINDOW_SECS = 5` (`crates/buzz-relay/src/admission.rs:9`), which
multiplies `human_ws_events_per_sec` into a 5-second budget
(`crates/buzz-relay/src/admission.rs:40-45`).

---

### Configuration structs (deserialisation contract)

`AuthConfig` — `Serialize + Deserialize + Default`
(`crates/buzz-auth/src/lib.rs:90-95`). One field, `rate_limits: RateLimitConfig`
with `#[serde(default)]` (`:93-94`), so an empty `[auth]` section is valid. The
doc says it is "typically loaded from the relay's TOML config file"
(`crates/buzz-auth/src/lib.rs:89`); in practice the relay builds it from env vars.

`RateLimitConfig` — every field has a `#[serde(default = "fn")]` fallback, so any
subset of keys deserialises successfully
(`crates/buzz-auth/src/rate_limit.rs:85-108`):

| Field | Type | Default | Default fn |
|-------|------|---------|-----------|
| `human_messages_per_min` | `u64` | 60 | `crates/buzz-auth/src/rate_limit.rs:110-112` |
| `human_api_calls_per_min` | `u64` | 300 | `crates/buzz-auth/src/rate_limit.rs:113-115` |
| `human_ws_events_per_sec` | `u64` | 10 | `crates/buzz-auth/src/rate_limit.rs:116-118` |
| `agent_standard_messages_per_min` | `u64` | 120 | `crates/buzz-auth/src/rate_limit.rs:119-121` |
| `agent_standard_api_calls_per_min` | `u64` | 600 | `crates/buzz-auth/src/rate_limit.rs:122-124` |
| `agent_elevated_messages_per_min` | `u64` | 300 | `crates/buzz-auth/src/rate_limit.rs:125-127` |
| `agent_platform_messages_per_min` | `u64` | 600 | `crates/buzz-auth/src/rate_limit.rs:128-130` |

The manual `Default` impl reuses the same seven fns rather than repeating the
literals (`crates/buzz-auth/src/rate_limit.rs:132-144`), so each number has one
source of truth.

`LimitType` deserialises from snake_case strings (`messages`, `api_calls`,
`ws_events`, `ip_connections`) via
`#[serde(rename_all = "snake_case")]` (`crates/buzz-auth/src/rate_limit.rs:56-67`).

No other type in the crate is serde-annotated — `AuthContext`, `AuthMethod`,
`Scope`, and `RateLimitResult` are all plain in-memory types.

---

### Rate-limit configuration — summary of the verified state

Full analysis with per-element scoring lives in the security aspect under
`### Rate limiting — verified state`. Configuration-relevant summary:

`ARCHITECTURE.md:823` (§9 item 2) claims rate limiting is unimplemented and that
`AlwaysAllowRateLimiter` is the only `RateLimiter` implementation. **That is
stale.** The `.env.example:58` heading "Shared Redis-backed admission limits" is
the accurate description.

- A production `RateLimiter` exists: `RedisRateLimiter`
  (`crates/buzz-pubsub/src/rate_limiter.rs:88-121`), ungated, using an atomic
  `INCR`+`EXPIRE` Lua script (`crates/buzz-pubsub/src/rate_limiter.rs:24-31`).
- It is constructed unconditionally at relay startup
  (`crates/buzz-relay/src/state.rs:712`) and held on `AppState`
  (`crates/buzz-relay/src/state.rs:584`).
- Checks run **before** work is admitted: WebSocket EVENT/REQ/COUNT
  (`crates/buzz-relay/src/connection.rs:498-500` → `:594-653`) and HTTP
  `POST /events` / `/query` / `/count`
  (`crates/buzz-relay/src/api/bridge.rs:760`, `:955`, `:1386` → `:24-56`).
  Limiter errors fail closed (`crates/buzz-relay/src/admission.rs:33-36`).
- Media upload uses a **separate, per-pod, in-process** `DashMap` limiter keyed on
  `media_uploads_per_minute` (`crates/buzz-relay/src/api/media.rs:88-111`, window
  const `:66`) — not the Redis limiter, not any `BUZZ_RATE_LIMIT_*` var.
- `check_ip_connection` / `LimitType::IpConnections` is implemented
  (`crates/buzz-pubsub/src/rate_limiter.rs:112-120`) but has **no production
  caller** — there is no per-IP connection cap.

Env-var wiring (`crates/buzz-relay/src/config.rs`):

| Env var | Parsed at | Default | Reaches an enforcement point? |
|---------|-----------|---------|-------------------------------|
| `BUZZ_RATE_LIMIT_HUMAN_MESSAGES_PER_MIN` | `config.rs:287-290` | 60 | yes — `crates/buzz-relay/src/connection.rs:636` (WS EVENT) |
| `BUZZ_RATE_LIMIT_HUMAN_API_CALLS_PER_MIN` | `config.rs:291-294` | 300 | yes — `crates/buzz-relay/src/api/bridge.rs:29` (HTTP bridge) |
| `BUZZ_RATE_LIMIT_HUMAN_WS_EVENTS_PER_SEC` | `config.rs:295-298` | 10 | yes — `crates/buzz-relay/src/connection.rs:614` (×5s burst window) |
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_MESSAGES_PER_MIN` | `config.rs:299-302` | 120 | yes — `crates/buzz-relay/src/connection.rs:634` (NIP-OA agents) |
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_API_CALLS_PER_MIN` | `config.rs:303-306` | 600 | **no consumer** |
| `BUZZ_RATE_LIMIT_AGENT_ELEVATED_MESSAGES_PER_MIN` | `config.rs:307-310` | 300 | **no consumer** |
| `BUZZ_RATE_LIMIT_AGENT_PLATFORM_MESSAGES_PER_MIN` | `config.rs:311-314` | 600 | **no consumer** |

Parsing rules (`positive_u64_from_env`,
`crates/buzz-relay/src/config.rs:270-282`): unset → the `RateLimitConfig::default()`
value; zero or non-integer → `ConfigError::InvalidValue("{name} must be a positive
integer")`; non-Unicode → `ConfigError::InvalidValue`. So a value of `0` is a
startup failure, not "disabled" — pinned by test at
`crates/buzz-relay/src/config.rs:1109-1120`. Override plumbing is pinned by
`rate_limits_can_be_overridden` (`crates/buzz-relay/src/config.rs:1092-1106`).

CI raises three of the limits (`.github/workflows/ci.yml:492-494`:
`HUMAN_MESSAGES_PER_MIN=100000`, `HUMAN_API_CALLS_PER_MIN=100000`,
`HUMAN_WS_EVENTS_PER_SEC=10000`) so the integration suite does not trip them —
corroborating that enforcement is live rather than a no-op.

Documentation drift to fix: `ARCHITECTURE.md:390`, `ARCHITECTURE.md:460`, and
`ARCHITECTURE.md:823`.


## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Configuration

`buzz-pubsub` reads **no environment variables in any production path**. All
configuration is injected by the caller as constructor arguments. A repo-wide grep
for `env::var` inside `crates/buzz-pubsub/src/` returns exactly one hit, and it is
inside a `#[cfg(test)]` module.

---

### 1. Injected configuration

| Input | Type | Entry point | Supplied by |
|---|---|---|---|
| Redis URL (for pub/sub connections) | `String` | `PubSubManager::new` (`lib.rs:117`) → stored `lib.rs:105` | `buzz-relay/src/main.rs:344` passes `&config.redis_url` |
| Redis connection pool (for all commands) | `deadpool_redis::Pool` | `PubSubManager::new` (`lib.rs:117`), `RedisRateLimiter::new` (`rate_limiter.rs:94`), `RedisNip98ReplayGuard::new` (`nip98_replay.rs:29`) | `buzz-relay/src/state.rs:711-712`, `:717`; pool field `state.rs:494` |
| Unsubscribe debounce | `Duration` | `PubSubConfig::with_unsubscribe_debounce` (`lib.rs:93`) | **nobody** — see §3 |
| Rate-limit window + limit | `u64, u64` per call | `check_and_increment` args (`rate_limiter.rs:100-107`) | `buzz-relay/src/connection.rs:610-611`, `:628-646` from `state.auth.config().rate_limits` (`connection.rs:610`) |
| Replay TTL | `u64` per call, then clamped | `try_mark_in_scope` (`nip98_replay.rs:38`), clamp `:47` | `buzz-auth` constants `DEFAULT_REPLAY_TTL_SECS` / `MAX_REPLAY_TTL_SECS` (`nip98_replay.rs:11-12`) |

The Redis URL reaches the crate from `REDIS_URL`, resolved in the relay with a
default of `redis://localhost:6379` (`buzz-relay/src/config.rs:418-419`, field
declared `config.rs:60`). The crate never reads that variable itself.

Note the pool is **injected, not constructed** — so pool size, timeouts, and any TLS
settings are entirely the relay's concern; `buzz-pubsub` cannot influence them. Only
the three dedicated pub/sub sockets are opened from the crate's own stored URL
(`subscriber.rs:85-86`, `cache_invalidation.rs:135-136`, `conn_control.rs:126-127`),
each via `redis::Client::open` on every reconnect attempt.

### 2. Compile-time constants (the crate's real tunables)

| Constant | Value | Site | Configurable at runtime? |
|---|---|---|---|
| `PubSubConfig::DEFAULT_UNSUBSCRIBE_DEBOUNCE` | 500 ms | `lib.rs:82` | Only via the builder — unused in production |
| Event broadcast capacity | 4096 | `lib.rs:126` | No |
| Cache-invalidation broadcast capacity | 4096 | `lib.rs:127` | No |
| Conn-control broadcast capacity | 4096 | `lib.rs:128` | No |
| Subscription-command mpsc capacity | 4096 | `lib.rs:129` | No |
| `PRESENCE_TTL_SECS` | 90 | `presence.rs:16` | No |
| `BACKOFF_INITIAL_SECS` | 1 | `subscriber.rs:16`, `cache_invalidation.rs:91`, `conn_control.rs:81` | No — three separate copies |
| `BACKOFF_MAX_SECS` | 30 | `subscriber.rs:18`, `cache_invalidation.rs:93`, `conn_control.rs:83` | No — three separate copies |
| `BUZZ_PREFIX` | `"buzz"` | `topic.rs:13` | No |
| `CACHE_INVALIDATION_SUFFIX` / `_PATTERN` | `"cache-invalidate"` / `"buzz:*:cache-invalidate"` | `cache_invalidation.rs:23`, `:27` | No |
| `CONN_CONTROL_SUFFIX` / `_PATTERN` | `"conn-control"` / `"buzz:*:conn-control"` | `conn_control.rs:26`, `:30` | No |
| `RATE_LIMIT_SCRIPT` | Lua source | `rate_limiter.rs:24-31` | No |

Operationally significant: **presence TTL, all four channel capacities, and the
reconnect backoff envelope cannot be tuned without a recompile.** For a horizontally
scaled deployment these are exactly the knobs an operator would want — e.g. raising
broadcast capacity for pods with many concurrent sockets, or lengthening presence TTL
for clients on flaky networks.

### 3. The unsubscribe-debounce knob is unreachable

`PubSubConfig` exists (`lib.rs:73-97`) and `with_unsubscribe_debounce`
(`lib.rs:93`) works, but every production construction goes through
`PubSubManager::new` (`lib.rs:117-119`), which hardcodes
`PubSubConfig::new(redis_url)` and therefore always takes the 500 ms default.

All 11 `PubSubManager::new` call sites in the relay confirm this — the production one
is `buzz-relay/src/main.rs:344`; the other ten are test fixtures
(`state.rs:1267`, `api/operator.rs:591`, `api/bridge.rs:3309`, `api/invites.rs:536`,
`api/admin/mod.rs:348`, `api/media.rs:970`, `workflow_sink.rs:585`,
`handlers/identity_archive.rs:451`, `handlers/event.rs:1990`, `:2043`). Zero call
sites for `with_config` or `with_unsubscribe_debounce` exist outside this crate's own
tests (`lib.rs:514`, `:596`, `:621`).

### 4. Rate-limit configuration reaching this crate

The crate holds no limits of its own; window and limit arrive per call. The relay
derives them from `state.auth.config().rate_limits` (`buzz-relay/src/connection.rs:610`):

| Limit | Derivation | Site |
|---|---|---|
| WS admission | `human_ws_events_per_sec` converted to a 5 s window via `ws_admission_budget` → `(5, limit × 5)` | `buzz-relay/src/admission.rs:8`, `:35-40`; used `connection.rs:611-621` |
| Per-message (human) | `human_messages_per_min`, fixed 60 s window | `connection.rs:629-633`, `:641` |
| Per-message (agent) | `agent_standard_messages_per_min`, fixed 60 s window | `connection.rs:629-631`, `:641` |

The IP-connection limit path (`rate_limiter.rs:112-120`) receives no configuration
because it has no production caller.

### 5. Test-only configuration

| Item | Site | Behaviour |
|---|---|---|
| `REDIS_URL` | `nip98_replay.rs:110` | `std::env::var("REDIS_URL")` with fallback `redis://127.0.0.1:6379` — the only env read in the crate, and it is inside `#[cfg(test)] mod tests` |
| `test_util::make_test_pool` | `lib.rs:371-376` | Hardcodes `redis://127.0.0.1:6379`, ignoring `REDIS_URL` |
| `#[ignore = "requires Redis"]` | 11 tests (`lib.rs:400`, `:438`, `:476`, `:511`; `presence.rs:137`, `:160`, `:187`; `nip98_replay.rs:128`, `:145`, `:165`, `:180`) | Integration tests opt out of the default `cargo test` run |

Inconsistency: `nip98_replay.rs` honours `REDIS_URL` while `test_util::make_test_pool`
(`lib.rs:371-376`) and every test using it hardcode `127.0.0.1:6379`. Pointing CI at
a non-default Redis host would run some of this crate's tests against the wrong
endpoint. Also note the relay's default (`localhost`, `config.rs:419`) and the
crate's test default (`127.0.0.1`) differ in spelling, which matters on hosts where
`localhost` resolves to IPv6 first.

### 6. Not configured anywhere

No configuration exists for: Redis TLS or auth (must be embedded in the URL string),
Redis logical db index, `maxmemory`/eviction expectations for the replay seen-set,
pub/sub message size limits, or a cap on topics retained per connection.


## Module: buzz-search (`crates/buzz-search`)

### Configuration

#### Environment variables

| Variable | Read in production code? | Where | Default | Purpose |
|---|---|---|---|---|
| `BUZZ_TEST_DATABASE_URL` | No — test harness only | `tests/fts_integration.rs:33`, `tests/fts_integration.rs:92` | `TEST_DB_URL` = `postgres://buzz:buzz_dev@localhost:5432/buzz` (`tests/fts_integration.rs:21`) | Postgres connection for the ignored integration tests |

**No environment variable is read anywhere in `src/`.** All runtime configuration
reaches the crate through the `PgPool` the caller passes to
`SearchService::new(pool)` (`crates/buzz-search/src/lib.rs:46-48`) — connection
string, pool sizing, TLS, and timeouts are entirely the caller's concern
(constructed by the relay, e.g. `crates/buzz-relay/src/state.rs:1273`).

#### Cargo features

| Feature | Status |
|---|---|
| Crate-defined features | None — `Cargo.toml` has no `[features]` section (`crates/buzz-search/Cargo.toml:1-18`) |
| `#[cfg(feature = ...)]` in code | None in any `.rs` file |
| `#[cfg(test)]` | One, gating the unit-test module (`src/query.rs:325`) |
| Inherited sqlx features (from workspace) | `runtime-tokio`, `tls-rustls`, `postgres`, `uuid`, `chrono`, `json` (root `Cargo.toml:52-54`) |
| Inherited uuid features | `v4`, `serde` (root `Cargo.toml:89`) |

Package metadata is fully workspace-inherited (`version`, `edition`,
`rust-version`, `license`, `repository`) — `crates/buzz-search/Cargo.toml:3-7`.

#### Compile-time constants (complete list)

| Constant | Value | Type | Line | Effect |
|---|---|---|---|---|
| `PER_PAGE_MAX` | `500` | `u32` | `src/query.rs:129` | Upper clamp on `SearchQuery.per_page` → `LIMIT` (`query.rs:224`) |
| `PER_PAGE_DEFAULT` | `100` | `u32` | `src/query.rs:130` | Substituted when `per_page == 0` (`query.rs:225-229`) |
| `SEARCH_TEXT_MAX_CHARS` | `4096` | `usize` | `src/query.rs:134` | Hard char cap on search text before it reaches Postgres' text-search parsers (`query.rs:185`) |
| `PAGE_MAX` | `1000` | `u32` | `src/query.rs:138` | Upper clamp on `page`, bounding `OFFSET = (page-1) * per_page` (`query.rs:220`, `230-231`) |

Effective caps composed with those constants: maximum reachable `OFFSET` is
`999 * 500 = 499_500` rows, and maximum rows returned per call is 500.

Test-file constants (not production configuration):

| Constant | Value | Line |
|---|---|---|
| `TEST_DB_URL` | `postgres://buzz:buzz_dev@localhost:5432/buzz` | `tests/fts_integration.rs:21` |
| `MIGRATION_0001_SQL` … `MIGRATION_0008_SQL`, `MIGRATION_0014_SQL` | `include_str!` of the corresponding `migrations/*.sql` | `tests/fts_integration.rs:22-32` |

#### Hard-coded behavioral settings (not configurable)

| Setting | Value | Line |
|---|---|---|
| Text-search configuration | `'simple'` (no stemming, no stopwords) — used for both `websearch_to_tsquery` and the prefix-mode `to_tsvector` | `query.rs:143`, `query.rs:173` |
| Ranking function | `ts_rank_cd` (no weights, no normalization flag) | `query.rs:236` |
| Ordering | `rank DESC, created_at DESC, id` | `query.rs:295` |
| Prefix-mode token split | `regexp_split_to_table(<term>, '\s+')` | `query.rs:167-170` |
| Prefix-mode conjunction | `' & '` | `query.rs:157` |
| Table queried | `events` (unqualified; resolved via the connection's `search_path`) | `query.rs:237` |

#### Caller-side settings that shape this crate's inputs (for reference)

| Setting | Value | Site |
|---|---|---|
| `MAX_SEARCH_PAGES` (WS pages iterated) | `10` | `crates/buzz-relay/src/handlers/req.rs:421` |
| WS `per_page` | `100` fixed | `crates/buzz-relay/src/handlers/req.rs:591` |
| Bridge `per_page` | `filter.limit.unwrap_or(100).min(500)` | `crates/buzz-relay/src/api/bridge.rs:1660` |
| Bridge `page` | raw `page` / `search_page` / `searchPage`, `> 0`, default `1` | `crates/buzz-relay/src/api/bridge.rs:345-353` |
| Bridge `mode` | `"prefix"` → `SearchMode::Prefix`, else `FullText` | `crates/buzz-relay/src/api/bridge.rs:337-345` |


## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Configuration

### Environment variables read by this crate

| Variable | Read at | Scope | Default |
|---|---|---|---|
| `DATABASE_URL` | `crates/buzz-audit/src/service.rs:276` | **Test-only** helper `test_pool()` inside `#[cfg(test)] mod tests` | `postgres://buzz:buzz_dev@localhost:5432/buzz` (`service.rs:277`) |

That is the **only** `std::env` access in the crate (grep for `env::var`/`std::env`
across `crates/buzz-audit/src` returns exactly `service.rs:276`).

`BUZZ_AUDIT_ENABLED` is **not** consumed here. It is read by the relay:

| Aspect | Location |
|---|---|
| Field declaration + doc | `crates/buzz-relay/src/config.rs:199-202` |
| Parse, default `true` | `crates/buzz-relay/src/config.rs:793` (`parse_bool("BUZZ_AUDIT_ENABLED", true)`) |
| Gate on service construction | `crates/buzz-relay/src/main.rs:321-334` — `Some(AuditService::new(pool))` or `None` + info log |
| Exposed as a gauge | `crates/buzz-relay/src/main.rs:139` (`buzz_audit_enabled` = 1.0/0.0) |
| Test | `crates/buzz-relay/src/config.rs:1041-1047` |

The relay's doc note states it does not control the separate `moderation_actions`
audit trail (`crates/buzz-relay/src/config.rs:200`).

### Runtime configuration surface of the crate

All configuration is by constructor argument, not environment:

| Input | Where | Notes |
|---|---|---|
| `PgPool` | `AuditService::new(pool)` (`service.rs:43-45`) | the crate never builds a pool; the relay supplies one sized `max_connections(5)`, `min_connections(1)` from `config.database_url` (`crates/buzz-relay/src/main.rs:322-328`) |
| `community: CommunityId` | `verify_chain` (`service.rs:162`), `get_entries` (`service.rs:214`) | per-call tenant scope |
| `from_seq` / `to_seq` / `limit` | `service.rs:163-164`, `:215-216` | caller-supplied; no defaults or caps in the crate |

### Cargo features

`crates/buzz-audit/Cargo.toml` declares **no `[features]` section**, no
`default-features` toggles on its own dependencies, and no `[dev-dependencies]`.
Every dependency is inherited via `workspace = true`
(`crates/buzz-audit/Cargo.toml:11-22`), so feature selection is entirely the
workspace's (e.g. sqlx features `runtime-tokio, tls-rustls, postgres, uuid, chrono,
json` at `Cargo.toml:52-54`).

Package metadata is all workspace-inherited except `description`
(`crates/buzz-audit/Cargo.toml:1-9`): `description = "Hash-chain audit log for Buzz"`
(`:8`).

### Compile-time constants

| Constant | Value | Visibility | Line |
|---|---|---|---|
| `GENESIS_HASH` | `[0u8; 32]` — 32 zero bytes | `pub` (re-exported at `lib.rs:34`) | `crates/buzz-audit/src/hash.rs:9` |
| `AUDIT_LOCK_NAMESPACE` | `"buzz_audit:"` | private to `service` module | `crates/buzz-audit/src/service.rs:29` |
| `AuditAction::ALL` | slice of all 11 variants | private (`const ALL: &'static [Self]`) | `crates/buzz-audit/src/action.rs:51-63` |

Other hard-coded values that behave as configuration:

| Value | Meaning | Line |
|---|---|---|
| `6` in `trunc_subsecs(6)` | timestamp precision = microseconds, matched to `TIMESTAMPTZ` | `hash.rs:23` |
| `0` in `hashtextextended($1, 0)` | Postgres hash seed for the advisory-lock key | `service.rs:59`, `:71` |
| `1` starting `seq` | derived from `prev_seq = 0` when a community has no rows | `service.rs:108-110` |
| presence tags `1u8` / `0u8` | optional-field framing in the pre-image | `hash.rs:55`, `:58`, `:62`, `:65` |

### Crate-level lint configuration

`#![deny(unsafe_code)]`, `#![warn(missing_docs)]` (`crates/buzz-audit/src/lib.rs:1-2`).
No `#[allow(...)]` attributes appear anywhere in the crate.

### Schema/DDL configuration

The crate ships no migrations — the `audit_log` table is owned by
`migrations/0001_initial_schema.sql:606-619`, stated explicitly at
`crates/buzz-audit/src/lib.rs:17-18`. The advisory-lock namespace `'buzz_audit:'` is
registered in migration commentary alongside other lock families
(`migrations/0023_push_match_gate.sql:21`).

### Test-environment configuration

- `test_pool()` returns `Option<PgPool>` so ignored tests skip silently when Postgres
  is unreachable (`service.rs:275-280`).
- `#[ignore = "requires Postgres"]` on all 6 async tests (`service.rs:319`, `:339`,
  `:377`, `:438`, `:476`, `:513`), so `cargo test` without `--ignored` needs no infra.
- DB tests require a `communities` row to satisfy the FK; `make_community()` inserts one
  with a unique host (`service.rs:296-305`).
- A `static OnceLock<Mutex<()>>` serializes DB tests that share the table
  (`service.rs:263-267`).


## Module: buzz-media (`crates/buzz-media`)

### Aspect: Configuration

### 1. Environment variables read *inside* this crate

**None in `src/`.** A scan of all 11 source files finds zero `std::env`/`env::var` uses. `MediaConfig` is a plain `serde::Deserialize` struct (`crates/buzz-media/src/config.rs:16-17`); the caller populates it. Env reading happens only in the ignored integration test:

| Var | Default in test | file:line |
|---|---|---|
| `BUZZ_S3_ENDPOINT` | `http://localhost:9000` | `crates/buzz-media/tests/static_creds_minio.rs:23-24` |
| `BUZZ_S3_ACCESS_KEY` | `buzz_dev` | `crates/buzz-media/tests/static_creds_minio.rs:25-26` |
| `BUZZ_S3_SECRET_KEY` | `buzz_dev_secret` | `crates/buzz-media/tests/static_creds_minio.rs:27-28` |
| `BUZZ_S3_BUCKET` | `buzz-media` | `crates/buzz-media/tests/static_creds_minio.rs:29` |

Indirect env consumption: `Credentials::default()` triggers the AWS credential chain, which reads `AWS_*` variables (env, profile, web-identity/IRSA, container — including the Pod Identity `AWS_CONTAINER_CREDENTIALS_FULL_URI` + `AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE` pair the patched `aws-creds` fork adds) inside the dependency, not in this crate (`crates/buzz-media/src/storage.rs:25-33`, `:51-56`; fork rationale `Cargo.toml:163-171`).

---

### 2. `MediaConfig` fields and the env vars that populate them

Env-var names and default values come from the relay (`crates/buzz-relay/src/config.rs:614-657`); the type and any serde default come from this crate.

| Field | Type | Serde default (this crate) | Env var (relay) | Relay default |
|---|---|---|---|---|
| `s3_endpoint` | `String` | required | `BUZZ_S3_ENDPOINT` | `http://localhost:9000` |
| `s3_access_key` | `String` | required | `BUZZ_S3_ACCESS_KEY` | `buzz_dev` |
| `s3_secret_key` | `String` | required | `BUZZ_S3_SECRET_KEY` | `buzz_dev_secret` |
| `s3_bucket` | `String` | required | `BUZZ_S3_BUCKET` | `buzz-media` |
| `s3_region` | `String` | `default_s3_region()` = `"us-east-1"` (`crates/buzz-media/src/config.rs:11-13`, `:24-27`) | `BUZZ_S3_REGION`, falling back to `AWS_REGION` | `us-east-1` |
| `max_image_bytes` | `u64` | **none — required** (`crates/buzz-media/src/config.rs:39-41`) | `BUZZ_MAX_IMAGE_BYTES` | `50 * 1024 * 1024` = 52_428_800 |
| `max_gif_bytes` | `u64` | **none — required** (`crates/buzz-media/src/config.rs:42-44`) | `BUZZ_MAX_GIF_BYTES` | `10 * 1024 * 1024` = 10_485_760 |
| `max_video_bytes` | `u64` | `default_max_video_bytes()` = `524_288_000` (500 MB) (`crates/buzz-media/src/config.rs:3-5`, `:45-47`) | `BUZZ_MAX_VIDEO_BYTES` | `500 * 1024 * 1024` = 524_288_000 |
| `max_file_bytes` | `u64` | `default_max_file_bytes()` = `104_857_600` (100 MB) (`crates/buzz-media/src/config.rs:7-9`, `:48-50`) | `BUZZ_MAX_FILE_BYTES` | `100 * 1024 * 1024` = 104_857_600 |
| `public_base_url` | `String` | required | `BUZZ_MEDIA_BASE_URL` | `http://localhost:3000/media` |
| `upload_records_enabled` | `bool` | `#[serde(default)]` → `false` (`crates/buzz-media/src/config.rs:49-50`) | `BUZZ_MEDIA_UPLOAD_RECORDS` (`"true"`/`"1"`) | `false` |
| `upload_ip_header` | `Option<String>` | `#[serde(default)]` → `None` (`crates/buzz-media/src/config.rs:55-56`) | `BUZZ_MEDIA_UPLOAD_IP_HEADER` (trimmed, lowercased) | unset |
| `upload_port_header` | `Option<String>` | `#[serde(default)]` → `None` (`crates/buzz-media/src/config.rs:60-61`) | `BUZZ_MEDIA_UPLOAD_PORT_HEADER` (trimmed, lowercased) | unset |

Related relay-side knobs that shape this crate's behaviour but are not `MediaConfig` fields: `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS` (default 8), `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS_PER_PUBKEY` (default 2, clamped to the global), `BUZZ_MEDIA_UPLOADS_PER_MINUTE` (default 30) (`crates/buzz-relay/src/config.rs:659-676`), and `BUZZ_REQUIRE_MEDIA_GET_AUTH` (default `false`, `crates/buzz-relay/src/config.rs:677-685`). `.env.example` documents the first three at `.env.example:85-87` and the read-auth flags at `.env.example:90-92`.

---

### 3. Startup validation rules (`MediaConfig::validate`, `crates/buzz-media/src/config.rs:66-122`)

| Check | Failure message | file:line |
|---|---|---|
| `public_base_url` must end with `/media` | `public_base_url must end with /media: got '…'` | `crates/buzz-media/src/config.rs:67-72` |
| `public_base_url` must not end with `/` | `public_base_url must not end with /: got '…'` | `crates/buzz-media/src/config.rs:73-78` |
| `max_image_bytes > 0` | `max_image_bytes must be > 0` | `crates/buzz-media/src/config.rs:79-81` |
| `max_gif_bytes > 0` and `<= max_image_bytes` | `max_gif_bytes must be > 0 and <= max_image_bytes` | `crates/buzz-media/src/config.rs:82-84` |
| `max_video_bytes > 0` | `max_video_bytes must be > 0` | `crates/buzz-media/src/config.rs:85-87` |
| `max_file_bytes > 0` | `max_file_bytes must be > 0` | `crates/buzz-media/src/config.rs:88-90` |
| IP header set without records enabled | long message ending "Enable upload records or unset the header." | `crates/buzz-media/src/config.rs:91-102` |
| Port header set without IP header | long message ending "Set the IP header or unset the port header." | `crates/buzz-media/src/config.rs:103-110` |
| Both header names must be valid HTTP header names (`axum::http::HeaderName::from_bytes`) | `{VAR} is not a valid header name: {h:?}` | `crates/buzz-media/src/config.rs:111-124` |

The two coherence checks are justified in-code as fail-loud choices: "an operator who set an IP header believes they are meeting a reporting obligation" (`crates/buzz-media/src/config.rs:91-95`).

---

### 4. Cargo features

The crate declares **no `[features]` section** (`crates/buzz-media/Cargo.toml`), and no `#[cfg(feature = …)]` appears in the source. Feature selection is only on dependencies:

| Dependency | Features |
|---|---|
| `s3` (`rust-s3` 0.37) | `default-features = false`, `tokio-rustls-tls`, `fail-on-err`, `tags` (`crates/buzz-media/Cargo.toml:24`) |
| `image` 0.25 | `default-features = false`, `jpeg`, `png`, `gif`, `webp` (`crates/buzz-media/Cargo.toml:26`) |
| `tokio-util` 0.7 | `io` (`crates/buzz-media/Cargo.toml:31`) |
| `tokio` (dev) | `test-util` (`crates/buzz-media/Cargo.toml:33-34`) |

Version/edition/license/rust-version are all inherited via `workspace = true` (`crates/buzz-media/Cargo.toml:3-8`).

---

### 5. Compile-time constants (all values)

| Constant | Value | Applies to | file:line |
|---|---|---|---|
| `ALLOWED_MIME_TYPES` | `["image/jpeg", "image/png", "image/gif", "image/webp"]` | image upload allowlist | `crates/buzz-media/src/validation.rs:15` |
| `MP4_BRANDS` | `isom iso2 iso3 iso4 iso5 iso6 iso7 iso8 iso9 mp41 mp42 avc1 dash "M4V "` (14 brands) | ISO-BMFF brand acceptance | `crates/buzz-media/src/validation.rs:17-20` |
| `BLOCKED_FILE_MIME_TYPES` | 14 entries: `text/html`, `application/xhtml+xml`, `image/svg+xml`, `application/javascript`, `text/javascript`, `application/x-msdownload`, `application/x-executable`, `application/vnd.microsoft.portable-executable`, `application/x-mach-binary`, `application/x-sharedlib`, `application/x-elf`, `application/x-msi`, `application/vnd.android.package-archive`, `application/x-apple-diskimage` | generic-file deny-list | `crates/buzz-media/src/validation.rs:75-95` |
| `MAX_PIXELS` | `25_000_000` (25 MP; comment: "100MB max RGBA decode") | image bomb guard | `crates/buzz-media/src/validation.rs:269` |
| duration limit | `600.0` seconds (literal, not a named const) | video length | `crates/buzz-media/src/validation.rs:357` |
| resolution limit | `3840` × `2160` (literals) | video resolution | `crates/buzz-media/src/validation.rs:364` |
| `MAX_ATOMS` | `1024` | top-level MP4 atoms scanned in the moov-order check | `crates/buzz-media/src/validation.rs:413` |
| `MAX_BOXES` | `100_000` | total boxes walked in the MP4 metadata check | `crates/buzz-media/src/validation.rs:832` |
| `MAX_BOX_DEPTH` | `32` | MP4 box nesting depth | `crates/buzz-media/src/validation.rs:833` |
| `EMPTY_FFMPEG_UDTA` | 53-byte literal sequence (`meta`/`hdlr`/`mdirappl`/`ilst`) | the only permitted `udta` payload | `crates/buzz-media/src/validation.rs:834-839` |
| `FORBIDDEN` (MP4 boxes) | `meta ilst keys data uuid "xml " bxml loci \xa9xyz name chap` (11) | metadata box denial | `crates/buzz-media/src/validation.rs:840-852` |
| `CONTAINERS` (MP4) | `moov trak mdia minf stbl edts dinf sinf schi` (9) | boxes walked recursively | `crates/buzz-media/src/validation.rs:853-855` |
| `ALLOWED` (MP4 boxes) | 35 box types (`ftyp moov mdat free skip wide trak mdia minf stbl edts dinf sinf schi udta mvhd tkhd mdhd hdlr vmhd smhd dref "url " "urn " stsd stts stss ctts stsc stsz stco co64 sgpd sbgp elst`) | box allowlist | `crates/buzz-media/src/validation.rs:856-864` |
| `PNG_SNAPSHOT_KEYWORDS` | `["buzz_agent_snapshot", "buzz_team_snapshot"]` | permitted `tEXt` keywords | `crates/buzz-media/src/validation.rs:579` |
| thumbnail max dimension | `320` × `320` (literals, aspect preserved) | thumbnail size | `crates/buzz-media/src/thumbnail.rs:30` |
| blurhash components | `4`, `3` (x, y) | blurhash resolution | `crates/buzz-media/src/thumbnail.rs:37` |
| thumbnail format | `ImageFormat::Jpeg` | thumbnail encoding | `crates/buzz-media/src/thumbnail.rs:31` |
| `MIN_SNIFF_BYTES` | `4096` | leading bytes retained for MP4 magic detection during streaming | `crates/buzz-media/src/upload.rs:351` |
| video read buffer | `64 * 1024` (64 KiB) | streaming chunk size | `crates/buzz-media/src/upload.rs:353` |
| buffered auth window | `600` seconds | image/file Blossom `created_at` age | `crates/buzz-media/src/upload.rs:85` |
| video auth window | `3600` seconds | video Blossom `created_at` age | `crates/buzz-media/src/upload.rs:412` |
| clock-skew tolerance | `5` seconds (future `created_at`) | Blossom auth | `crates/buzz-media/src/auth.rs:117` |
| Blossom auth kind | `24242` | auth event kind check | `crates/buzz-media/src/auth.rs:42` |
| `BUF` | `8 * 1024 * 1024` (8 MiB) | `put_file` read buffer | `crates/buzz-media/src/storage.rs:91` |
| `UPLOAD_RECORD_VERSION` | `1` | upload-record schema version | `crates/buzz-media/src/upload_record.rs:52` |
| sidecar key prefix | `_meta/` | tenant metadata namespace | `crates/buzz-media/src/storage.rs:184` |
| record key prefix | `_uploads/` | moderation namespace | `crates/buzz-media/src/upload_record.rs:182` |
| thumb key suffix | `.thumb.jpg` | derived artifact naming | `crates/buzz-media/src/upload.rs:531` |
| sha256 hex length | `64`, charset `[0-9a-f]` | key classification | `crates/buzz-media/src/bucket_index.rs:75-79` |
| blob ext bounds | 1–8 ASCII alphanumeric (uppercase allowed) | key classification | `crates/buzz-media/src/bucket_index.rs:84-87` |
| ULID length | `26`, uppercase Crockford base32 | record key classification | `crates/buzz-media/src/bucket_index.rs:93-105` |
| UUID form | exactly 36 chars, lowercase hex + hyphens at 8/13/18/23 | community segment parsing | `crates/buzz-media/src/bucket_index.rs:112-127` |

No concurrency-limit constants exist in this crate (semaphore sizes live in the relay); the only caller-supplied bounds are the sweep `cap` and `max_keys` arguments (`crates/buzz-media/src/bucket_index.rs:377-379`, `crates/buzz-media/src/storage.rs:242-246`).

---

### 6. Verification: where does "50 MB" actually live?

ARCHITECTURE.md states "PUT `/media/upload` — Upload media blob (Blossom, 50 MB limit)" (`ARCHITECTURE.md:624`). In this crate, 50 MB is **not** a production constant:

| Occurrence | Context |
|---|---|
| `crates/buzz-media/src/config.rs:34` | doc comment only: "Maximum upload size for images (bytes). Default: 50 MB." — the field itself has no serde default (`crates/buzz-media/src/config.rs:41`) |
| `crates/buzz-media/src/upload.rs:573` | test fixture |
| `crates/buzz-media/src/validation.rs:952`, `:1590` | test fixtures |
| `crates/buzz-media/src/storage.rs:288` | test fixture |
| `crates/buzz-media/tests/static_creds_minio.rs:33` | ignored integration test fixture |

The only place 50 MB is a real runtime default is the relay: `BUZZ_MAX_IMAGE_BYTES` → `unwrap_or(50 * 1024 * 1024)` (`crates/buzz-relay/src/config.rs:626-629`). It also applies to **images only** — video (500 MB) and generic files (100 MB) have larger caps, and the relay's request-body limit layer uses `max(max_image_bytes, max_video_bytes)` (`crates/buzz-relay/src/router.rs:33-45`).


## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Configuration

---

### 1. Environment variables

All three are read inside `add_reaction_impl` and only exist under `#[cfg(feature = "reqwest")]`. There is no config struct wiring, no `.env` loading, and no other `std::env::var` call in the crate (verified by grep).

| Variable | Read at | Default | Purpose |
|---|---|---|---|
| `BUZZ_RELAY_BASE_URL` | `executor.rs:889-890` | `"http://localhost:3000"` | Base URL for the reaction POST |
| `BUZZ_API_TOKEN` | `executor.rs:901` | none | If present, sent as `Authorization: Bearer {token}` |
| `BUZZ_RELAY_PUBKEY` | `executor.rs:903` | none | Fallback when `BUZZ_API_TOKEN` is absent, sent as `X-Pubkey` |

If neither auth variable is set, the request goes out unauthenticated (`executor.rs:901-905`).

---

### 2. Cargo features

| Feature | Definition | Effect | Enabled by |
|---|---|---|---|
| `reqwest` | `reqwest = ["dep:reqwest"]` (`Cargo.toml:28-29`), dependency declared `optional = true` (`Cargo.toml:27`) | Compiles the real HTTP paths: `check_ssrf` (`executor.rs:744-776`), `WEBHOOK_MAX_RESPONSE_BYTES` (`executor.rs:777-778`), `call_webhook_impl` (`executor.rs:780-869`), `shared_http_client` (`executor.rs:873-885`), `add_reaction_impl` (`executor.rs:887-933`). Disabled ⇒ `call_webhook` returns `{status:0, body:null, skipped:true}` (`executor.rs:636-647`) and `add_reaction` returns `{added:false, skipped:true}` (`executor.rs:606-616`) | `buzz-relay` (`crates/buzz-relay/Cargo.toml:63`: `features = ["reqwest"]`). `buzz-admin` depends without it (`crates/buzz-admin/Cargo.toml:21`) |

No default features are declared, so a bare `cargo build -p buzz-workflow` compiles the skip-stub variants.

---

### 3. Runtime configuration struct — `WorkflowConfig`

`lib.rs:57-71`. Every construction site in the repo uses `WorkflowConfig::default()`; no environment variable or CLI flag overrides these values.

| Field | Type | Default | Consumed at |
|---|---|---|---|
| `max_concurrent` | `usize` | `100` (`lib.rs:68`) | semaphore permits, `max(1)`-clamped (`lib.rs:110-111`) |
| `default_timeout_secs` | `u64` | `300` (`lib.rs:69`) | per-step timeout fallback when `step.timeout_secs` is absent (`executor.rs:1136-1138`) |

Construction sites: `crates/buzz-relay/src/main.rs:389-390` (production), `crates/buzz-relay/src/state.rs:1274-1276`, plus ~8 relay test fixtures — all `::default()`.

---

### 4. Compile-time constants (complete list, with values)

| Constant | Value | Where | Meaning |
|---|---|---|---|
| `EVAL_TIMEOUT` | `Duration::from_millis(100)` | `executor.rs:342` | Wall-clock cap on one evalexpr evaluation (does not cancel the blocking thread — `executor.rs:358-360`) |
| `MAX_EXPR_LEN` | `4096` (bytes) | `executor.rs:362` (fn-local const in `evaluate_condition`) | Rejects long condition expressions before evaluation |
| `MAX_DELAY_SECS` | `270` (seconds) | `executor.rs:676` (fn-local const in `dispatch_action`) | Upper bound on the `delay` action; chosen to stay under the 300 s default step timeout |
| `WEBHOOK_MAX_RESPONSE_BYTES` | `1024 * 1024` = 1 048 576 (1 MiB) | `executor.rs:778` (`#[cfg(feature = "reqwest")]`) | Outbound webhook response body cap |

---

### 5. Hard-coded values that behave like configuration (not named constants)

| Value | Where | Meaning |
|---|---|---|
| Semaphore permits = `config.max_concurrent.max(1)` → 100 by default | `lib.rs:110-111` | Concurrency admission; `try_acquire()` so overflow is rejected, not queued (`executor.rs:978`, `executor.rs:1028`) |
| Workflow cache `max_capacity(10_000)` | `lib.rs:119` | Max cached `(community_id, channel_id)` entries |
| Workflow cache `time_to_live(10s)` | `lib.rs:120` | TTL, documented as matching the relay's other moka caches and bounding cross-pod staleness (`lib.rs:92-103`) |
| Scheduler tick `sleep(Duration::from_secs(60))` | `lib.rs:432` | Cron/interval loop period; sleep happens before the first scan |
| Cron match window `60` (passed as `window_secs`) | `lib.rs:484` → `lib.rs:688-706` | Drift-tolerant lookback window for cron matching |
| Minimum allowed interval = `60` seconds | `schema.rs:220-224` | Definition-time rejection of sub-minute intervals, justified by the 60 s tick |
| HTTP client timeout `Duration::from_secs(10)` | `executor.rs:807` (per-request webhook client), `executor.rs:879` (shared reaction client) | Total request timeout |
| Redirect policy `Policy::none()` | `executor.rs:812` | Webhook redirects disabled |
| Default HTTP method `"POST"` | `executor.rs:621` | `call_webhook` when `method` is omitted |
| Default port fallback `80` | `executor.rs:798` | Used when `port_or_known_default()` yields `None` |
| Approval timeout default string `"24h"` | `executor.rs:653` | **Log-only.** The string is never parsed into a duration and no expiry is persisted, because no approval record is created (`executor.rs:663`). The documented "Defaults to 24h" in the schema (`schema.rs:138-140`) has no runtime effect today |
| Step-ID limits: non-empty, ≤ `64` chars, `[A-Za-z0-9_]` | `schema.rs:168-172` | Definition-time validation protecting evalexpr variable names |
| Cron field normalization: 5→7, 6→7 fields | `schema.rs:250-257` | `0 {expr} *` / `{expr} *` |
| Reaction `e`-tag length check `64` hex chars | `lib.rs:924-928` | Distinguishes hex event IDs from UUID channel refs when resolving `trigger.message_id` |

---

### 6. Not configurable

- The four evalexpr helper functions are registered unconditionally with no allow/deny list (`executor.rs:236-283`).
- The SSRF blocklist is compiled into `buzz-core` (`crates/buzz-core/src/network.rs:46-95`); there is no allowlist/bypass hook for internal webhook targets. Still true after `c26bf594` widened the blocklist — the added prefixes are also compile-time constants (`network.rs:7`, `:10`), not configuration.
- Scheduler tick period, cache TTL/capacity, delay cap, expression cap, evaluation timeout, HTTP timeouts, and the response cap are all literals — none are surfaced through `WorkflowConfig` or environment variables.


## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Configuration

**93 distinct environment variables** are read by the files in this group: 74 in `config.rs`, 15 in `main.rs` (14 explicit + `RUST_LOG` via `EnvFilter`), 1 in `nip11.rs`, 3 in `telemetry.rs` (2 explicit + `OTEL_RESOURCE_ATTRIBUTES` via the SDK detector). A further 4 are read by `storage_sweep.rs` and consumed only from `main.rs` (§4).

There is no config file, no CLI flag parsing, and no config reload. Everything is env-only, read once at boot — except two vars read **per request/per tick** (§5).

---

#### 1. `config.rs` — every env var (74)

##### 1a. Listeners and identity

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_BIND_ADDR` | `SocketAddr` | `0.0.0.0:3000` | `config.rs:407` | `main.rs:1157` (TCP bind). Invalid ⇒ **hard error** `InvalidBindAddr` (`config.rs:265-268`) |
| `RELAY_URL` | `String` | `ws://localhost:3000` | `config.rs:428` | `main.rs:239` (deployment community), `main.rs:568`, `nip11.rs:241`, `tenant.rs:130`. **Never validated as a URL** |
| `BUZZ_PAIRING_RELAY_URL` | `Option<String>` | `None` | `config.rs:430` | `nip11.rs:243` only. Must be `ws`/`wss` with a host ⇒ else hard error (`config.rs:432-446`) |
| `BUZZ_UDS_PATH` | `Option<String>` | `None` | `config.rs:604` | `main.rs:1162-1187`; a `warn` on non-unix (`main.rs:1207-1211`) |
| `BUZZ_HEALTH_PORT` | `u16` | `8080` | `config.rs:609` | `main.rs:1116` — bound to hard-coded `0.0.0.0`, ignores `BUZZ_BIND_ADDR`'s IP |
| `BUZZ_METRICS_PORT` | `u16` | `9102` | `config.rs:614` | `main.rs:138` → `metrics.rs:74` — also hard-coded `0.0.0.0` |
| `BUZZ_RELAY_PRIVATE_KEY` | `Option<String>` | `None` | `config.rs:602` | `main.rs:393-394` (parse or abort), `nip11.rs:302` (gates `self` + NIP-43). **Not validated at config load** — only when `main` parses it |
| `RELAY_OWNER_PUBKEY` | `Option<String>` (64 hex) | `None` | `config.rs:526` | `main.rs:201`, `main.rs:292-294`. Invalid ⇒ **warn-and-ignore, value echoed in the log** (`config.rs:538-542`) |

##### 1b. Datastores

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `DATABASE_URL` | `String` | `postgres://buzz:buzz_dev@localhost:5432/buzz` | `config.rs:410` | `main.rs:146` (main pool), `main.rs:328` (audit pool), `main.rs:375` (search fallback) |
| `READ_DATABASE_URL` | `Option<String>` | `None`; blank/whitespace normalised to `None` (`config.rs:413-416`) | `config.rs:413` | `main.rs:147` (replica pool), `main.rs:373` (search pool preference), `main.rs:384` (log) |
| `REDIS_URL` | `String` | `redis://localhost:6379` | `config.rs:419` | `main.rs:337`, `main.rs:344` |
| `BUZZ_REDIS_POOL_SIZE` | `usize` >0 | `16` | `config.rs:421` | `main.rs:338`. Zero/junk ⇒ silent fallback to 16 (tested `config.rs:988-1011`) |

##### 1c. Connection limits

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_MAX_CONNECTIONS` | `usize` | `10_000` | `config.rs:449` | `state.rs:649` → `conn_semaphore` (`state.rs:729`) |
| `BUZZ_MAX_CONCURRENT_HANDLERS` | `usize` | `1_024` | `config.rs:454` | `state.rs:650` → `handler_semaphore` (`state.rs:730`) |
| `BUZZ_SEND_BUFFER` | `usize` | `1_000` | `config.rs:459` | `connection.rs:159` |
| `BUZZ_MAX_FRAME_BYTES` | `usize` >0 | `524_288` (`config.rs:15`) | `config.rs:464` | `router.rs:302` → `router.rs:330-331`, `nip11.rs:242` (advertised `max_message_length`) |
| `BUZZ_SLOW_CLIENT_GRACE_LIMIT` | `u8` | `15` | `config.rs:470` | `connection.rs:179`, `connection.rs:212` |

##### 1d. Auth / membership

| Var | Grammar | Default | Read | Consumed |
|-----|---------|---------|------|----------|
| `BUZZ_REQUIRE_AUTH_TOKEN` | `=="true"\|\|=="1"` | `false` | `config.rs:475` | `api/bridge.rs`, `main.rs:395` (dev-key gate), `main.rs:409` (panic gate). Emits a startup `warn` when false (`config.rs:590-594`) |
| `BUZZ_PUBKEY_ALLOWLIST` | `=="true"\|\|=="1"` | `false` | `config.rs:479` | `handlers/auth.rs:187` |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | `=="true"\|\|=="1"` | `false` | `config.rs:483` | 22 sites; `main.rs:201/214/242/250/276/296/506`, `nip11.rs:304` |
| `BUZZ_ALLOW_NIP_OA_AUTH` | `=="true"\|\|=="1"` | `false` | `config.rs:520` | `api/mod.rs:81` |
| `RELAY_OPERATOR_PUBKEYS` | CSV of 64-hex | empty (⇒ provisioning disabled) | `config.rs:555` | `api/operator.rs:91`, `handlers/community_provisioning.rs:261`. Invalid entry ⇒ **hard error, entry echoed** (`config.rs:566-571`); deduped |
| `RELAY_OPERATOR_API_ORIGIN` | http(s) origin, no creds/path/query/fragment | `None` | `config.rs:549` | `api/operator.rs:70`. **Required** when `RELAY_OPERATOR_PUBKEYS` is non-empty (`config.rs:577-582`) |

##### 1e. Web / CORS / admin / policy

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_CORS_ORIGINS` | CSV | empty ⇒ **permissive CORS** | `config.rs:595` | `router.rs:190` → `router.rs:409-432` |
| `BUZZ_WEB_DIR` | `Option<PathBuf>` | `None` | `config.rs:843` | `router.rs:145-183`, `router.rs:320-330`. Must contain `index.html` ⇒ else hard error (`config.rs:850-856`) |
| `BUZZ_SERVE_GIT_WEB_GUI` | `=="true"\|\|=="1"` | `false` | `config.rs:848` | `router.rs:143`, `router.rs:320` |
| `BUZZ_ADMIN_HOST` | exact authority | `None` ⇒ admin surface absent | `config.rs:814` | `router.rs:47`, `api/admin::is_admin_host`. Rejects `/ \ @` (`config.rs:819-823`) |
| `BUZZ_ADMIN_WEB_DIR` | `Option<PathBuf>` | `None` | `config.rs:826` | `router.rs:49-52`, `router.rs:265-271`. Must contain `index.html` (`config.rs:830-836`) |
| `BUZZ_TERMS_OF_SERVICE_MARKDOWN` | `Option<String>` ≤256 KiB | `None` | `config.rs:790` (helper `:776`) | `api/invites::join_policy*` via `config.join_policy` |
| `BUZZ_PRIVACY_POLICY_MARKDOWN` | `Option<String>` ≤256 KiB | `None` | `config.rs:791` | same |
| `BUZZ_AGE_ATTESTATION_REQUIRED` | `parse_bool` strict | `false` | `config.rs:792` (helper `:379`) | same; invalid ⇒ hard error (tested `config.rs:1074-1090`) |

##### 1f. Media

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_S3_ENDPOINT` | `String` | `http://localhost:9000` | `config.rs:620` | `state.rs:695` (git store), `buzz_media::MediaStorage` |
| `BUZZ_S3_ACCESS_KEY` | `String` | `buzz_dev` | `config.rs:622` | same |
| `BUZZ_S3_SECRET_KEY` | `String` | `buzz_dev_secret` | `config.rs:624` | same |
| `BUZZ_S3_BUCKET` | `String` | `buzz-media` | `config.rs:626` | same |
| `BUZZ_S3_REGION` | `String` | falls back to `AWS_REGION`, then `us-east-1` | `config.rs:627` | same |
| `AWS_REGION` | `String` | — (fallback only) | `config.rs:628` | region resolution |
| `BUZZ_MAX_IMAGE_BYTES` | `u64` | `52_428_800` (50 MB) | `config.rs:630` | `router.rs:34` (body limit), media validation |
| `BUZZ_MAX_GIF_BYTES` | `u64` | `10_485_760` (10 MB) | `config.rs:634` | media validation |
| `BUZZ_MAX_VIDEO_BYTES` | `u64` | `524_288_000` (500 MB) | `config.rs:638` | `router.rs:35`, media validation |
| `BUZZ_MAX_FILE_BYTES` | `u64` | `104_857_600` (100 MB) | `config.rs:642` | media validation |
| `BUZZ_MEDIA_BASE_URL` | `String` | `http://localhost:3000/media` | `config.rs:646` | media URL generation |
| `BUZZ_MEDIA_UPLOAD_RECORDS` | `=="true"\|\|=="1"` | `false` | `config.rs:651` | `MediaConfig::validate` coherence check (`main.rs:417`) |
| `BUZZ_MEDIA_UPLOAD_IP_HEADER` | `Option<String>` lowercased | `None` | `config.rs:654` | same |
| `BUZZ_MEDIA_UPLOAD_PORT_HEADER` | `Option<String>` lowercased | `None` | `config.rs:658` | same |
| `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS` | `usize` >0 | `8` | `config.rs:664` | `state.rs:693` → `media_upload_semaphore` |
| `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS_PER_PUBKEY` | `u32` >0 | `2`, clamped to the above | `config.rs:670` | `api/media.rs` |
| `BUZZ_MEDIA_UPLOADS_PER_MINUTE` | `u32` >0 | `30` | `config.rs:676` | `api/media.rs` |
| `BUZZ_REQUIRE_MEDIA_GET_AUTH` | `true\|1\|yes\|on` (ci) | `false` | `config.rs:682` | `api/media.rs` GET/HEAD gate |

##### 1g. Git

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_GIT_REPO_PATH` | `PathBuf` (created) | `./repos` | `config.rs:705` | `api/git/*`; `create_dir_all` failure ⇒ hard error (`config.rs:384-397`) |
| `BUZZ_GIT_PACK_CACHE_PATH` | `PathBuf` (created) | `<git_repo_path>/.pack-cache` | `config.rs:709` | `state.rs:704` |
| `BUZZ_GIT_MAX_PACK_BYTES` | `u64` | `524_288_000` (500 MB) | `config.rs:713` | `api/git/transport.rs:1757` (body limit) |
| `BUZZ_GIT_MAX_REPO_BYTES` | `u64` | `git_max_pack_bytes * 2` (1 GB) | `config.rs:717` | `api/git/hydrate.rs` |
| `BUZZ_GIT_PACK_CACHE_MAX_BYTES` | `u64` | `git_max_repo_bytes * 5` (5 GB) | `config.rs:721` | `state.rs:705` |
| `BUZZ_GIT_PACK_CACHE_MAX_CONCURRENT_POPULATIONS` | `usize` >0 | `2` | `config.rs:726` | `state.rs:706` |
| `BUZZ_GIT_MAX_REPOS_PER_PUBKEY` | `u32` | `100` | `config.rs:731` | `handlers/side_effects.rs:2487` |
| `BUZZ_GIT_MAX_CONCURRENT_OPS` | `usize` | `20` | `config.rs:735` | `state.rs:692` → `git_semaphore` |
| `BUZZ_GIT_HOOK_HMAC_SECRET` | `String` ≥32 when set | random 32-byte hex **per process** | `config.rs:739`, re-read `config.rs:865` | `api/git/policy.rs`, `api/git/hook.rs` |

##### 1h. Push (NIP-PL)

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_PUSH_EXECUTOR_KEY_ID` | `String` 1..=64 | `relay-v1` | `config.rs:746` | `nip11.rs:241`. Empty or >64 ⇒ hard error (`config.rs:747-751`) |
| `BUZZ_PUSH_GATEWAY_DELIVERY_URL` | exact `https://…/v1/deliveries/apns` | **`https://push.buzz.xyz/v1/deliveries/apns`** (`config.rs:332`) — explicit empty string disables | `config.rs:752` | `main.rs:686` (worker gate), `nip11.rs:244/251`, `push_runtime.rs` |
| `BUZZ_PUSH_GATEWAY_TIMEOUT_MS` | `u64` in `100..=10000` | `2000` | `config.rs:759` | `push_runtime.rs:314`. Out of range ⇒ hard error (`config.rs:763-771`) |

##### 1i. Mesh

| Var | Grammar | Default | Read | Consumed |
|-----|---------|---------|------|----------|
| `BUZZ_MESH` | `on` (ci) \| `true` \| `1` | off | `config.rs:498` | `mesh_boot::boot_mesh` (`main.rs:442`) |
| `BUZZ_MESH_BIND_ADDR` | `SocketAddr` | `0.0.0.0:3478` | `config.rs:501` | `boot_mesh`. Invalid ⇒ hard error (`config.rs:502-506`) |
| `BUZZ_MESH_DEMO_ECHO` | `on` (ci) \| `true` \| `1` | off | `config.rs:516` | `main.rs:456` (`wire_consumers`), `api/mesh_demo.rs` |

##### 1j. Misc

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_HUDDLE_AUDIO_AVAILABLE` | `!(=="false"\|\|=="0")` (**inverted**) | `true` | `config.rs:489` | `audio/handler.rs:357` |
| `BUZZ_AUDIT_ENABLED` | `parse_bool` strict | `true` | `config.rs:793` | `main.rs:130/139/323`, `state.rs:763`. Invalid ⇒ hard error (tested `config.rs:1056-1072`) |
| `BUZZ_EPHEMERAL_TTL_OVERRIDE` | `i32` >0 | `None` | `config.rs:691` | `handlers/ingest.rs:2125`, `handlers/side_effects.rs:1681`. Emits a startup `warn` when set (`config.rs:696-702`) |

##### 1k. Rate limits (7 vars → `buzz_auth::RateLimitConfig`)

Read by `rate_limit_config_from_env` (`config.rs:284-317`) via `positive_u64_from_env` (`config.rs:270-282`). **All 7 hard-error on zero, junk, or non-Unicode** — no silent fallback.

| Var | Read | `RateLimitConfig` field | Production consumer |
|-----|------|-------------------------|---------------------|
| `BUZZ_RATE_LIMIT_HUMAN_MESSAGES_PER_MIN` | `config.rs:288` | `human_messages_per_min` | `connection.rs:636` |
| `BUZZ_RATE_LIMIT_HUMAN_API_CALLS_PER_MIN` | `config.rs:292` | `human_api_calls_per_min` | `api/bridge.rs:29` |
| `BUZZ_RATE_LIMIT_HUMAN_WS_EVENTS_PER_SEC` | `config.rs:296` | `human_ws_events_per_sec` | `connection.rs:614` |
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_MESSAGES_PER_MIN` | `config.rs:300` | `agent_standard_messages_per_min` | `connection.rs:634` |
| **`BUZZ_RATE_LIMIT_AGENT_STANDARD_API_CALLS_PER_MIN`** | `config.rs:304` | `agent_standard_api_calls_per_min` | **NONE — parsed, validated, stored, never read** |
| **`BUZZ_RATE_LIMIT_AGENT_ELEVATED_MESSAGES_PER_MIN`** | `config.rs:308` | `agent_elevated_messages_per_min` | **NONE** |
| **`BUZZ_RATE_LIMIT_AGENT_PLATFORM_MESSAGES_PER_MIN`** | `config.rs:312` | `agent_platform_messages_per_min` | **NONE** |

Confirmed by workspace grep across `crates/**` and `desktop/src-tauri/**`: the only occurrences of the three field names outside `config.rs` are their declarations (`crates/buzz-auth/src/rate_limit.rs:101,104,107`) and their default assignments (`:139,140,141`). No reader. **3 of 7 rate-limit vars are dead** — an operator setting them gets no effect and no warning. Two of the three name a tier ("elevated", "platform") for which no enforcement path exists at all.

---

#### 2. `main.rs` — 15 env vars read outside `Config`

These bypass `Config` entirely, so they are invisible to `Config`'s tests, invisible to the startup "Config loaded" log (`main.rs:128-136`), and not covered by any of `config.rs`'s validation helpers.

| Var | Type | Default | Read | Consumed | Invalid-value policy |
|-----|------|---------|------|----------|----------------------|
| `RUST_LOG` | filter directives | none (plus forced `buzz_relay=info`) | `main.rs:111` (`EnvFilter::from_default_env`) | log filtering | tracing-subscriber default |
| `BUZZ_AUTO_MIGRATE` | `true\|1\|yes\|on` trimmed/ci | off | `main.rs:162` (helper `main.rs:25-33`) | `main.rs:163-172` | anything else ⇒ off |
| `BUZZ_RECONCILE_CHANNELS` | **presence only** (`is_ok()`) | absent | `main.rs:549` | `main.rs:549-590` | any value incl. empty ⇒ on |
| `BUZZ_GIT_CONFORMANCE_PROBE` | `!= "false"` | on | `main.rs:469` | `main.rs:466-503` | anything but `false` ⇒ on |
| `BUZZ_GIT_PROBE_WRITERS` | `usize` | `32` | `main.rs:473` | `main.rs:485` | silent default |
| `BUZZ_GIT_PROBE_ROUNDS` | `usize` | `3` | `main.rs:477` | `main.rs:486` | silent default |
| `BUZZ_NIP43_RECONCILE_INTERVAL_SECS` | `u64` `.max(1)` | `60` | `main.rs:517` | `main.rs:523` | silent default |
| `BUZZ_REAPER_INTERVAL_SECS` | `u64` | `60` | `main.rs:609` | `main.rs:619` | silent default; **no floor — `0` yields a busy loop** |
| `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS` | `u64` | `10` | `main.rs:701` | `main.rs:721` | silent default; **no floor** |
| `SPROUT_REMINDER_SCHEDULER_BATCH_LIMIT` | `i64` | `100` | `main.rs:705` | `main.rs:727` | silent default |
| `BUZZ_COMMUNITY_REVALIDATE_INTERVAL_SECS` | `u64` `.clamp(1,300)` | `30` | `main.rs:882` | `main.rs:889` | silent default + clamp |
| `BUZZ_POOL_METRICS_INTERVAL_SECS` | `u64` `.max(1)` | `10` | `main.rs:945` | `main.rs:951` | silent default; floored because `interval` panics on zero (`main.rs:949`) |
| `BUZZ_USAGE_METRICS_INTERVAL_SECS` | `u64` `.max(5)` | `300` | `main.rs:1258` | `main.rs:133`, `main.rs:1013/1021` | silent default + floor |
| `BUZZ_USAGE_METRICS_IDLE_TIMEOUT_SECS` | `u64` | `900`, raised to `interval*3` | `main.rs:1267` | `main.rs:138` → `metrics.rs:77` | silent default |
| `BUZZ_USAGE_METRICS_PER_COMMUNITY` | `""\|all\|off` | `all` | `main.rs:58` | `main.rs:1002`, `main.rs:1335/1509/1467` | unknown ⇒ `warn` + `all` (`main.rs:66-72`) |

Two of these have **no lower bound** (`BUZZ_REAPER_INTERVAL_SECS`, `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS`), unlike the three neighbours that do — a `0` turns the corresponding task into a `sleep(0)` hot loop hammering Postgres. The two adjacent pollers were explicitly floored for exactly this reason (`main.rs:949`, `main.rs:1261`).

---

#### 3. `nip11.rs` and `telemetry.rs`

| Var | Type | Default | Read | Consumed | Note |
|-----|------|---------|------|----------|------|
| `SPROUT_MAX_NOT_BEFORE_DELTA` | `u64` | `31_536_000` (1 y) | `nip11.rs:97` | `nip11.rs:113` advertised `max_not_before_delta` | **read on every NIP-11 request**, not cached in `Config`; legacy `SPROUT_` prefix |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | presence | unset ⇒ tracing disabled | `telemetry.rs:81` | `telemetry.rs:79-90` | value itself is consumed by the OTLP SDK, not by this code |
| `OTEL_SERVICE_NAME` | non-empty `String` | `buzz-relay` | `telemetry.rs:50` | `telemetry.rs:56-59` | empty string falls back (tested `telemetry.rs:157-171`) |
| `OTEL_RESOURCE_ATTRIBUTES` | OTEL spec | — | `telemetry.rs:53` (`EnvResourceDetector::new()`) | overlaid **last**, so a `service.name` there wins over `OTEL_SERVICE_NAME` (`telemetry.rs:32`, `:48-52`) | |

**Documented-but-unimplemented (verified delta):** `telemetry.rs:23-24` lists `OTEL_TRACES_SAMPLER` (documented default `parentbased_always_on`) and `OTEL_TRACES_SAMPLER_ARG` as "Standard OTEL env vars honoured". No code in this crate reads either — `try_init_tracer` (`telemetry.rs:79-90`) and `classify_exporter_result` (`telemetry.rs:96-113`) build the provider with only `.with_resource(...)` and `.with_batch_exporter(...)`, never `.with_sampler(...)`. Whether they take effect therefore depends entirely on `opentelemetry_sdk` 0.32 internals (`Cargo.toml:78` → workspace `Cargo.toml:78`). I could not verify the SDK's behaviour offline; treat the doc claim as unproven and the sampler as effectively unconfigurable from this crate.

---

#### 4. Consumed by `main.rs` but declared in `storage_sweep.rs`

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_STORAGE_SWEEP_INTERVAL_SECS` | `u64` `.max(60)` | `3600` | `storage_sweep.rs:56` | `main.rs:1454` → `main.rs:1457` |
| `BUZZ_STORAGE_SWEEP_TIMEOUT_SECS` | `u64` | `120` | `storage_sweep.rs:61` | `main.rs:1458` |
| `BUZZ_STORAGE_SWEEP_MAX_OBJECTS` | `u64` | `1_000_000` | `storage_sweep.rs:65` | `main.rs:1454`, `main.rs:1462` |
| `BUZZ_STORAGE_METRICS` | `off` (trimmed, ci) disables; anything else incl. unset enables | enabled | `storage_sweep.rs:69` | `main.rs:1452` |

These live in a function-local `OnceLock` (`main.rs:1444-1451`) rather than on `Config`/`AppState`, with an explicit rationale: localized feature config, read once on the first leader tick, stable for the process lifetime.

---

#### 5. Config read outside boot (2 vars)

Everything else is read once. Two are not:

| Var | Frequency | Cite | Consequence |
|-----|-----------|------|-------------|
| `SPROUT_MAX_NOT_BEFORE_DELTA` | **every NIP-11 request** (`GET /`, `GET /info`) | `nip11.rs:96-100` | a `std::env::var` + parse per unauthenticated request; inconsistent with every other advertised limit, which comes from `Config` |
| `BUZZ_STORAGE_*` / `BUZZ_STORAGE_METRICS` | first leader tick only, then cached | `main.rs:1444-1451` | documented and intentional |

---

#### 6. Parsed-but-never-consumed summary (the known defect class)

| Item | Where parsed | Reader |
|------|--------------|--------|
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_API_CALLS_PER_MIN` | `config.rs:303-306` | **none** |
| `BUZZ_RATE_LIMIT_AGENT_ELEVATED_MESSAGES_PER_MIN` | `config.rs:307-310` | **none** |
| `BUZZ_RATE_LIMIT_AGENT_PLATFORM_MESSAGES_PER_MIN` | `config.rs:311-314` | **none** |

All three are fields of `buzz_auth::RateLimitConfig` nested inside `Config.auth` (`config.rs:82`), so the dead surface is *inside* a live field — which is why a top-level field audit misses it.

At the top level, every one of `Config`'s **51** fields (`config.rs:53-262`) has at least one production consumer — verified individually by grepping each field name across `crates/**` and `desktop/src-tauri/**` and discarding `#[cfg(test)]` hits. Two are consumed only through a continuation-line access that a naive `config.X` grep misses: `relay_operator_api_origin` (`api/operator.rs:70`) and `relay_operator_pubkeys` (`api/operator.rs:91`, `handlers/community_provisioning.rs:261`); both are live.

The near-misses worth recording:

| Field | Only production consumer | Note |
|-------|--------------------------|------|
| `pairing_relay_url` | `nip11.rs:243` | advertisement only — the relay never dials it |
| `pubkey_allowlist_enabled` | `handlers/auth.rs:187` | single site |
| `huddle_audio_available` | `audio/handler.rs:357` | single site |
| `allow_nip_oa_auth` | `api/mod.rs:81` | single site |
| `git_max_repos_per_pubkey` | `handlers/side_effects.rs:2487` | single site |
| `push_gateway_timeout` | `push_runtime.rs:314` | single site |
| `started_at` (AppState) | `router.rs:388` | single site (`/_status`) |

---

#### 7. `.env.example` coverage — 5 of 93

`.env.example` (233 lines) declares only 12 variable names, of which just **5 are read by this module**: `BUZZ_BIND_ADDR`, `DATABASE_URL`, `REDIS_URL`, `RELAY_URL`, `RUST_LOG`. The rest are `PG*` helpers for local tooling plus:

**Two dead entries**: `TYPESENSE_API_KEY` (`.env.example:40`) and `TYPESENSE_URL` (`.env.example:41`). No Rust code in the workspace reads either — search moved to Postgres FTS (`main.rs:369-372`, `handlers/event.rs:505-506` confirm "The old Typesense `index_event` worker and its `search_index_tx` mpsc are gone"). The only remaining `Typesense` mentions are historical comments (`crates/buzz-search/src/query.rs:20,46`).

**88 of 93 vars are undocumented in `.env.example`**, including every security-relevant switch: `BUZZ_REQUIRE_AUTH_TOKEN`, `BUZZ_REQUIRE_RELAY_MEMBERSHIP`, `BUZZ_PUBKEY_ALLOWLIST`, `BUZZ_REQUIRE_MEDIA_GET_AUTH`, `BUZZ_CORS_ORIGINS`, `BUZZ_RELAY_PRIVATE_KEY`, `RELAY_OWNER_PUBKEY`, `RELAY_OPERATOR_PUBKEYS`, `RELAY_OPERATOR_API_ORIGIN`, `BUZZ_AUDIT_ENABLED`, `BUZZ_AUTO_MIGRATE`, `BUZZ_ADMIN_HOST`. An operator following `AGENTS.md`'s `cp .env.example .env` gets a relay with every permissive default active and no indication that the switches exist.


## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Configuration

Every value this group reads comes from `AppState.config: Arc<Config>` (`state.rs:490`). Struct definition: `config.rs:51` onward; loader: `Config::from_env` at `config.rs:405`, called exactly once at `main.rs:122`.

> The WS path reads config through `state.config.*` on every frame (e.g. `connection.rs:420`, `:439`) rather than snapshotting it per connection — but there is no reload path, so the values are effectively immutable for the process lifetime.

---

#### 1. Env vars consumed directly by this group

| Env var | Config field | Default | Read at | Effect |
|---|---|---|---|---|
| `BUZZ_MAX_CONNECTIONS` | `max_connections` | **10 000** (`config.rs:449-452`) | `state.rs:727` (`Semaphore::new`), acquired `connection.rs:149` | hard cap on concurrent WS connections; exhaustion rejects with **no frame sent** (`connection.rs:151-154`) |
| `BUZZ_MAX_CONCURRENT_HANDLERS` | `max_concurrent_handlers` | **1024** (`config.rs:454-457`) | `state.rs:728`, acquired `connection.rs:513`/`:541`/`:563` | cap on in-flight EVENT/REQ/COUNT handler tasks; exhaustion → `rate-limited: too many concurrent requests` |
| `BUZZ_SEND_BUFFER` | `send_buffer_size` | **1 000** (`config.rs:459-462`) | `connection.rs:159` | per-connection outbound **message** queue depth (not bytes). Depth is what the backpressure strike counter measures |
| `BUZZ_MAX_FRAME_BYTES` | `max_frame_bytes` | **524 288** (512 KiB) — `DEFAULT_MAX_FRAME_BYTES` `config.rs:14`, parsed `:464-468` with a `>0` filter | parser `router.rs:334-342`; app re-check `connection.rs:420`, `:439` | max inbound frame; oversize → one `NOTICE` then disconnect. Doc'd as needing headroom over the 256 KiB content cap after JSON+NIP-44 overhead (`config.rs:11-13`) |
| `BUZZ_SLOW_CLIENT_GRACE_LIMIT` | `slow_client_grace_limit` (`u8`) | **15** (`config.rs:470-473`) | copied to `ConnectionState.grace_limit` `connection.rs:179` and `ConnEntry.grace_limit` `:212`; compared `connection.rs:100`, `state.rs:464` | consecutive buffer-full events tolerated before cancelling. **No `>0` filter** — `0` disconnects on the first full buffer; a value >255 fails to parse and silently falls back to 15 |
| `RELAY_URL` | `relay_url` | `ws://localhost:3000` (`config.rs:428`) | `auth.rs:80-81` via `nip42_expected_relay_url` | the URL a NIP-42 AUTH event's `relay` tag must match; tenant-adjusted per connection |
| `BUZZ_PUBKEY_ALLOWLIST` | `pubkey_allowlist_enabled` | **false** (`config.rs:479-481`) | `auth.rs:189` | when true, NIP-42 pubkey-only auth additionally requires a `pubkey_allowlist` row; DB error → deny |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | `require_relay_membership` | **false** (`config.rs:483-485`) | `auth.rs:245` | closed vs open relay. On an open relay the NIP-OA owner is still extracted for agent→owner backfill |
| `BUZZ_ALLOW_NIP_OA_AUTH` | `allow_nip_oa_auth` | **false** (`config.rs:520`) | indirectly, inside `enforce_relay_membership` (`auth.rs:217-223`) | permits NIP-OA owner attestation to *grant* membership on a closed relay |
| `BUZZ_RATE_LIMIT_HUMAN_WS_EVENTS_PER_SEC` | `auth.rate_limits.human_ws_events_per_sec` | **10** (`buzz-auth/src/rate_limit.rs:116-118`) | `connection.rs:614` → `admission.rs:44-49` | multiplied by the 5 s burst window → **50 frames per 5 s** for EVENT+REQ+COUNT combined |
| `BUZZ_RATE_LIMIT_HUMAN_MESSAGES_PER_MIN` | `auth.rate_limits.human_messages_per_min` | **60** (`rate_limit.rs:110-112`) | `connection.rs:636` | EVENT-only, 60 s window, human tier |
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_MESSAGES_PER_MIN` | `auth.rate_limits.agent_standard_messages_per_min` | **120** (`rate_limit.rs:119-121`) | `connection.rs:634` | EVENT-only, 60 s window, agent tier (selected by `agent_owner_pubkey.is_some()`) |

All `BUZZ_RATE_LIMIT_*` values go through `positive_u64_from_env` (`config.rs:270-283`), so `0` or a non-integer is a **hard startup error**, not a silent fallback — verified by `rate_limit_overrides_reject_zero`, `config.rs:1109`.

---

#### 2. Env vars consumed indirectly (services this group calls)

| Env var | Default | Reaches this group via |
|---|---|---|
| `REDIS_URL` | `redis://localhost:6379` (`config.rs:419`) | `admission_rate_limiter` (`state.rs:712`), `pubsub` publish/retain/release, presence |
| `BUZZ_REDIS_POOL_SIZE` | **16** (`config.rs:421-425`) — deadpool's own default (`CPU×2`) is called out as too small for a 2-vCPU pod (`config.rs:61-64`) | shared pool behind admission, presence, pub/sub |
| `DATABASE_URL` | `postgres://buzz:buzz_dev@localhost:5432/buzz` (`config.rs:410-411`) | every `state.db.*` call listed in the integrations aspect |
| `READ_DATABASE_URL` | unset → all reads on the writer (`config.rs:413-417`) | handled inside `buzz-db`; not visible to this group |
| `BUZZ_AUDIT_ENABLED` | **true** (`config.rs:793`) | when false, `audit_tx` is `None` and `enqueue_event_created_audit` short-circuits (`event.rs:571-573`) — removing the only awaited step in `dispatch_persistent_event` |
| `BUZZ_EPHEMERAL_TTL_OVERRIDE` | unset (`config.rs:691-695`) | `handlers::resolve_ttl` (`mod.rs:42-62`) — consumed by `ingest.rs:2125` / `side_effects.rs:1681`, not by the WS handlers themselves |
| `BUZZ_RELAY_PRIVATE_KEY` | unset → fresh keypair each start (`config.rs:602`) | `state.relay_keypair`, compared at `event.rs:517` to skip workflow triggering for relay-signed events |
| `BUZZ_CORS_ORIGINS` | empty → permissive (`config.rs:595-600`) | applied to the merged router (`router.rs:191`); does not gate the WS upgrade |
| `BUZZ_ADMIN_HOST` | unset (`config.rs:814-836`) | short-circuits `/` before NIP-11 or the WS upgrade (`router.rs:259-277`) — an admin-host request can never open a relay socket |

---

#### 3. Hard-coded values that are *not* configurable

These are the tuning knobs an operator cannot reach:

| Value | Constant | Site | Consequence of being fixed |
|---|---|---|---|
| **5 s** NIP-42 auth deadline | `AUTH_TIMEOUT` | `connection.rs:27` | a client on a high-latency link that cannot sign+round-trip in 5 s cannot connect |
| **30 s** heartbeat interval | — | `connection.rs:383` | not tunable for mobile radio-sleep profiles |
| **3** missed pongs | — | `connection.rs:389-393` | fixed 90 s dead-peer detection |
| **8** control-channel slots | — | `connection.rs:162` | a stalled writer becomes terminal after 8 queued control frames |
| **64** frames per flush | `MAX_WS_SEND_BATCH` | `connection.rs:33` | fan-out write batching |
| **1024** subscriptions/connection | `MAX_SUBSCRIPTIONS` | `req.rs:26` | matches the NIP-11 advertisement |
| **10** filters/REQ | `MAX_FILTERS_PER_REQ` | `protocol.rs:14` | NIP-11 advertised |
| **256 B** sub_id | `MAX_SUB_ID_LENGTH` | `protocol.rs:11` | NIP-11 advertised |
| **2 000** historical rows/filter | `MAX_HISTORICAL_LIMIT` | `req.rs:25` | also clamps the search per-filter limit (`req.rs:538`) |
| **4** concurrent filter queries | `FILTER_QUERY_CONCURRENCY` | `req.rs:35`; compile-time asserted 2..=8 at `:41` | intentionally locked; raising it requires re-running the relay bench (doc `req.rs:37-40`) |
| **10** search pages, **100** hits/page | `MAX_SEARCH_PAGES`, `per_page` | `req.rs:421`, `:589` | search result ceiling is 1000 hits/filter before post-filtering |
| **5 000** COUNT candidate rows | `COUNT_FALLBACK_CANDIDATE_LIMIT` | `req.rs:753` | a non-pushable COUNT over a large set is refused, not approximated |
| **100/s** observer telemetry per agent | — | `event.rs:912` | shared by every deployment size |
| **±300 s** observer freshness | — | `event.rs:952` | tighter than the ingest ±900 s drift window (`ingest.rs:1506`) |
| **±900 s** ingest timestamp drift | `MAX_TIMESTAMP_DRIFT_SECS` | `ingest.rs:1506` | — |
| **256 KiB** event content | `MAX_EVENT_CONTENT_BYTES` | `ingest.rs:1515` | the reason `max_frame_bytes` defaults to 512 KiB |
| **128 B** presence status truncation | — | `event.rs:780-785` | UTF-8-boundary-safe |
| **60 s / 10 000** local-echo cache | — | `state.rs:734-739` | cross-node dedup window |
| **10 s** membership / accessible-channels / visibility TTL | — | `state.rs:740-760` | upper bound on stale-access exposure when a cross-pod invalidation is dropped |
| **300 s / 1 000** observer owner cache | — | `state.rs:782-787` | safe because `agent_owner_pubkey` is immutable within a community (`state.rs:601-606`) |
| **1 000** audit channel capacity | — | `state.rs:654` | the back-pressure point for `dispatch_persistent_event` |
| **5 s** window / **×5** burst multiplier for `WsEvents` | `WS_BURST_WINDOW_SECS` | `admission.rs:10`, `:44-49` | fixed-window, not a token bucket — the code notes a Redis token bucket would be a better long-term fit (`admission.rs:5-8`) |
| **60 s** window for `Messages` | — | `connection.rs:643` | — |

---

#### 4. Startup-time validation relevant to this group

| Check | Behaviour | Site |
|---|---|---|
| `BUZZ_MAX_FRAME_BYTES` must be > 0 | invalid/zero → silent fallback to 512 KiB | `config.rs:464-468` |
| `BUZZ_RATE_LIMIT_*` must be a positive integer | invalid/zero → **`ConfigError::InvalidValue`, startup fails** | `config.rs:270-283` |
| `BUZZ_SLOW_CLIENT_GRACE_LIMIT` | **no validation** — unparsable → 15; `0` accepted verbatim | `config.rs:470-473` |
| `BUZZ_MAX_CONNECTIONS` / `BUZZ_MAX_CONCURRENT_HANDLERS` / `BUZZ_SEND_BUFFER` | **no `>0` filter** — `0` is accepted and would make `Semaphore::new(0)` / `mpsc::channel(0)`; `mpsc::channel(0)` **panics** at `connection.rs:159` | `config.rs:449-462` |
| `BUZZ_REQUIRE_AUTH_TOKEN=false` warns that REST bypasses token auth and states WS is unaffected | `warn!` only | `config.rs:588-593` |
| Defaults are asserted valid by a test | `config.rs:938-1000` — asserts `max_connections > 0`, `send_buffer_size > 0`, `slow_client_grace_limit > 0`, `max_frame_bytes == DEFAULT_MAX_FRAME_BYTES` | — |

The `defaults_are_valid` test (`config.rs:938`, 17 assertions through `:1000`) asserts the invariants that the **loader does not enforce**. It only covers the default path, so an operator setting `BUZZ_SEND_BUFFER=0` gets a runtime panic on the first connection rather than a config error.

---

#### 5. Configuration deltas against docs

| Claim | Source | Actual | Verdict |
|---|---|---|---|
| "Max frame size: 65,536 bytes" | `ARCHITECTURE.md:161` | `512 * 1024` (`config.rs:14`) | **wrong by 8×** |
| "Max historical results per filter: 500" | `ARCHITECTURE.md:161` | `2_000` (`req.rs:25`) | **wrong by 4×** |
| "After `SLOW_CLIENT_GRACE_LIMIT` (3) consecutive full-buffer events" | `ARCHITECTURE.md:208` | default **15** (`config.rs:473`); the value is a config field, not a constant | **wrong by 5×**, and misnames the mechanism |
| "`BUZZ_MAX_FRAME_BYTES` … default 65536" | `.aidlc/reverse-engineer/configuration.md:89` | 524 288 | **wrong** (inherits the ARCHITECTURE error) |
| "`BUZZ_SLOW_CLIENT_GRACE_LIMIT` … default 3" | `.aidlc/reverse-engineer/configuration.md:91` | 15 | **wrong** |
| "`BUZZ_SEND_BUFFER` … default (code default)" | `.aidlc/reverse-engineer/configuration.md:90` | 1 000 (`config.rs:462`) | resolvable — should be stated |
| "No rate limiting implementation … none are enforced" | `ARCHITECTURE.md:823` | `human_ws_events_per_sec`, `human_messages_per_min`, `agent_standard_messages_per_min` all enforced (`connection.rs:612-650`) | **wrong**; the accurate residual gap is `IpConnections` + the elevated/platform tiers |
| §3 Step 3 describes auth with no deadline | `ARCHITECTURE.md:181-187` | `AUTH_TIMEOUT` 5 s (`connection.rs:27`) | **omission** |
| §3 Step 5 cleanup lists 5 steps | `ARCHITECTURE.md:211-217` | 7 — adds per-subscription `release_topic` (`connection.rs:265-270`) and last-connection presence clear (`:274-285`) | **incomplete** |
| §4 pipeline step 10 "SEARCH INDEX — search_index_tx.send (bounded worker queue)" | `ARCHITECTURE.md:235` | removed; Postgres FTS makes the persisted row the searchable row, and both the worker and the mpsc are gone | **stale** — the code itself documents the removal at `event.rs:502-508` |
| `BUZZ_MAX_CONNECTIONS` / `BUZZ_MAX_CONCURRENT_HANDLERS` / rate-limit env vars | not in `.env.example` per `grep` of the documented table | present in the loader (`config.rs:449`, `:454`, `:288-320`) | **undocumented knobs** |

---

#### 6. Operational tuning notes derived from the code

- **Buffer sizing is coupled to the strike limit.** A 1000-message buffer with a 15-strike grace means a client must fail 15 *consecutive* sends; since any success resets to 0 (`connection.rs:92`), a client that drains slowly-but-steadily is never disconnected. Lowering `BUZZ_SLOW_CLIENT_GRACE_LIMIT` is the only lever, and there is no timeout-based eviction to complement it.
- **`max_frame_bytes` must exceed 256 KiB** or large legitimate events become unsendable: the ingest content cap is 256 KiB (`ingest.rs:1489`) and Nostr JSON + NIP-44 base64 roughly doubles it. The 512 KiB default is the documented reason (`config.rs:11-13`); a 65 536 setting (as `ARCHITECTURE.md` claims is the default) would silently break large messages.
- **The WS burst budget is not the advertised rate.** `human_ws_events_per_sec=10` yields a 5 s fixed window of 50, so a client can legitimately emit 50 frames in the first 10 ms of a window and then be starved for 4.99 s. Desktop startup is the stated reason (`admission.rs:5-7`).
- **Horizontal scaling requires `BUZZ_HUDDLE_AUDIO_AVAILABLE=false`** (`config.rs:114-129`, loader `config.rs:489-492`) — not a WS-protocol knob, but it shares the socket lifecycle registry this group registers into (`connection.rs:132`).
- **`BUZZ_MESH` defaults off** (loader `config.rs:498-500`) and is not consulted anywhere in this group; mesh-on and mesh-off WS behaviour is identical.


## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: Configuration

---

### 1. Config fields consumed (via `state.config`)

Exactly **four** fields of `crate::config::Config` are read across all 8 911 lines.

| Field | Env var | Default | Read at | Effect in this module |
|---|---|---|---|---|
| `require_relay_membership: bool` | `BUZZ_REQUIRE_RELAY_MEMBERSHIP` (`"true"` or `"1"` → true) | **`false`** (`config.rs:482-484`; asserted `config.rs:954-956`) | `ingest.rs:1847` | Gates kind 28936 (NIP-43 leave request). When `false` — the default — every 28936 is rejected with `invalid: relay membership is not enabled`. So **self-service relay leave is off out of the box.** |
| `ephemeral_ttl_override: Option<i32>` | `BUZZ_EPHEMERAL_TTL_OVERRIDE` (parsed `i32`, filtered `> 0`) | `None` (`config.rs:691-695`) | `ingest.rs:2125`, `side_effects.rs:1681` | Passed to `resolve_ttl` (`handlers/mod.rs:42-62`). When set, it **clobbers** any client-supplied `ttl` tag on kind 9007 rather than clamping it. Config emits a startup `warn!` when set (`config.rs:696-700`). Note: the 9002 `ttl` update path (`side_effects.rs:1449-1481`) does **not** apply the override — a client can change a channel's TTL post-creation to escape it. |
| `relay_url: String` | `RELAY_URL` (no `BUZZ_` prefix) | `"ws://localhost:3000"` (`config.rs:427-428`) | `ingest.rs:2238` | Input to `media_base_url_for_tenant(&state.config.relay_url, tenant.host())`, which yields the per-tenant `media_base_url` that `validate_imeta_tags` accepts absolute `imeta` URLs against (`imeta.rs:375-386`). A misconfigured `RELAY_URL` therefore rejects every absolute-form `imeta` URL while relative `/media/...` still works. |
| `git_max_repos_per_pubkey: u32` | `BUZZ_GIT_MAX_REPOS_PER_PUBKEY` | **`100`** (`config.rs:731-734`) | `side_effects.rs:2487` | Per-pubkey repo quota checked **before** claiming a name on kind 30617. Exceeded → `repo limit exceeded: {owned} >= {limit}`. Checked only on the fresh-claim path; a same-owner re-announce never grows the count (`side_effects.rs:2481-2490`). |

---

### 2. Env vars read directly (bypassing `Config`)

| Env var | Default | Read at | Effect |
|---|---|---|---|
| `SPROUT_MAX_NOT_BEFORE_DELTA` | **`31_536_000`** (1 year) | `ingest.rs:1325-1328` | Upper bound on how far in the future a kind:30300 `not_before` may be. Exceeded → `not_before too far in future`. |

⚠ Three problems with this one:
1. It is the **only** env var in the group read via `std::env::var` at request time instead
   of through `Config`. It is re-parsed on **every** kind:30300 ingest — no caching.
2. It uses the legacy `SPROUT_` prefix while every other var in this group uses `BUZZ_`
   (or bare `RELAY_URL`).
3. It is read in two places that must agree: here and `nip11.rs:97`, which advertises the
   value in the NIP-11 document. Both default to `31_536_000` independently; there is no
   shared constant. A change to one silently desynchronises the advertised limit from the
   enforced limit. The code comment at `ingest.rs:1298` asserts the coupling ("the same
   SPROUT_MAX_NOT_BEFORE_DELTA env var is advertised in NIP-11") without enforcing it.

---

### 3. Compile-time constants (effectively configuration, not tunable)

| Constant | Value | Declared | Governs |
|---|---|---|---|
| `MAX_TIMESTAMP_DRIFT_SECS` | `900` (±15 min) | `ingest.rs:1506` | Global created_at fence, all kinds |
| `MAX_EVENT_CONTENT_BYTES` | `256 * 1024` | `ingest.rs:1515` | Global content size cap |
| `MAX_REACTION_EMOJI_CHARS` | `64` | `ingest.rs:2309` | kind:7 emoji length (Unicode chars, not bytes) |
| `MAX_NOT_BEFORE` | `9_007_199_254_740_991` (`Number.MAX_SAFE_INTEGER`) | `ingest.rs:1250` | kind:30300 `not_before` ceiling |
| diff content cap | `61_440` (60 KiB) | `ingest.rs:898` | kind:40008 |
| thread depth cap | `100` | `ingest.rs:647` | NIP-10 nesting |
| 28936 freshness window | `120` s | `ingest.rs:1834` | NIP-43 leave request |
| DM participant cap | `8` others / `9` total | `command_executor.rs:322`, `:544` | kinds 41010, 41011 |
| NIP-44 v2 minimum decoded length | `99` bytes | `ingest.rs:1130` | kinds 30174, 44200 |
| `D_TAG_MAX_LEN` | `1024` | `buzz-db/src/event.rs:140`, used `ingest.rs:2404`, `command_executor.rs:141` | all NIP-33 kinds |
| persona slug max | `64` chars | `ingest.rs:1055` | kind 30175 |
| repo id max | `64` chars | `side_effects.rs:2393` | kind 30617 |
| imeta `filename` max | `255` chars | `imeta.rs:143` | all imeta |
| imeta MIME max | `255` chars | `imeta.rs:345` | all imeta |
| imeta duration tolerance | `0.1` s | `imeta.rs:272` | video imeta |
| `DEFAULT_HEAD` | `"refs/heads/main"` | `side_effects.rs:2606` | seeded git manifest + initial kind:30618 |
| audit channel capacity | `1000` | `state.rs:654`, referenced `handlers/event.rs:540` | audit backpressure on the ingest path |

None of these 17 are configurable. The 256 KiB content cap and the ±15 min drift window are
the two most likely to need per-deployment tuning and are hard-coded.

---

### 4. Metrics emitted (observability configuration surface)

| Metric | Type | Labels | Emitted at |
|---|---|---|---|
| `buzz_events_stored_total` | counter | `kind` (bounded), `author_type` (`agent`/`human`) | `ingest.rs:1422-1427` — at the shared seam so WS and HTTP count identically |
| `buzz_events_rejected_total` | counter | `transport` (`ws`/`http`), `reason` (`auth`/`invalid`/`scope`/`error`) | `reject_with_transport` `ingest.rs:157-163`; called from the transports, not from ingest |
| `buzz_channels_created_total` | counter | `community` (host), `type` | `ingest.rs:2150-2155`; `side_effects.rs:1704`, `:1731`; `command_executor.rs:373`, `:527` |
| `buzz_users_created_total` | counter | `community` (host) | `side_effects.rs:1097-1101`, `:1141-1145`; `command_executor.rs:51-56` |
| `buzz_nip43_membership_publications_total` | counter | `result` (`attempted`/`succeeded`/`failed`) | `side_effects.rs:2828`, `:2834-2838` |
| `buzz_nip43_membership_publication_seconds` | histogram | — | `side_effects.rs:2832` |
| `buzz_nip43_membership_reconciliations_total` | counter | — | `side_effects.rs:2818` |
| `buzz_nip43_membership_reconciliation_failures_total` | counter | — | `side_effects.rs:2811` |

Reached transitively via `dispatch_persistent_event`:
`buzz_post_commit_dispatch_scheduled_total` (`handlers/event.rs:371`),
`buzz_audit_send_errors_total` (`handlers/event.rs:599`),
`buzz_workflow_runs_total` with `trigger` + `community` (`handlers/event.rs:547-552`).

⚠ Gaps: there is **no** counter for side-effect failures, despite 32 `warn!` sites in
`side_effects.rs` that each represent a silently-degraded write (BR-IN-69). There is also no
histogram for ingest latency in this module, and `buzz_channels_created_total` is emitted
from five sites whose non-double-counting relies on a prose argument
(`side_effects.rs:1670-1679`) rather than a single emit point.

`author_type_label` (`ingest.rs:1328-1361`) performs a DB lookup
(`get_agent_channel_policy`) on cache miss purely to compute a metric label, cached in
`state.author_type_cache` (`state.rs:613`, `:788`). It is explicitly fail-open — unknown
pubkeys and lookup errors count as `"human"` so "the label must not add a failure path to
ingest" (`ingest.rs:1322-1324`). That cache is never invalidated, so an agent registered
after its first event is mislabelled `human` for the cache's lifetime.

---

### 5. Runtime-state dependencies (not env config, but deployment-shaping)

| `AppState` field | Used at | Note |
|---|---|---|
| `db` | everywhere | Postgres, required |
| `pubsub` | `side_effects.rs:81`, `:701`, `:794`, `:873` | Redis; publish failures are `warn!`-only |
| `media_storage` | `ingest.rs:2241` | S3/Blossom; required for any event with `imeta` |
| `git_store` | `side_effects.rs:2642-2726` | object store; required for kind 30617 |
| `workflow_engine` | `command_executor.rs:783`, `:900`, `:1076`; `side_effects.rs:2017`, `:2038` | in-process |
| `relay_keypair` | 20+ sites in `side_effects.rs` | signs every relay-minted event and is the sentinel in `effective_message_author` (`ingest.rs:731`) |
| `conn_manager`, `sub_registry` | `side_effects.rs:44-140` | subscription eviction |
| `tracer` | `ingest.rs:1409` | `NoopTracer` in production (`state.rs:798`) |
| `author_type_cache` | `ingest.rs:1363-1379` | metric labelling only |
| `audit_tx` | via `handlers/event.rs:539` | `None` disables auditing entirely (`main.rs:330` decides) |

---

### 6. `.env.example` coverage check

| Var this module reads | In `.env.example`? | In `Config`? |
|---|---|---|
| `RELAY_URL` | ✅ `.env.example:49` | ✅ `config.rs:68`, `:427-428` |
| `BUZZ_EPHEMERAL_TTL_OVERRIDE` | ✅ commented out, `.env.example:100` | ✅ `config.rs:209`, `:691-695` |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | ❌ **absent** | ✅ `config.rs:112`, `:482-484` |
| `BUZZ_GIT_MAX_REPOS_PER_PUBKEY` | ❌ **absent** | ✅ `config.rs:234`, `:731-734` |
| `SPROUT_MAX_NOT_BEFORE_DELTA` | ❌ **absent** | ❌ **absent** — only two independent `std::env::var` reads |

Two actionable rows:
- `BUZZ_REQUIRE_RELAY_MEMBERSHIP` gates a whole feature (self-service relay leave, kind
  28936) and defaults off, yet is not discoverable from `.env.example`.
- `SPROUT_MAX_NOT_BEFORE_DELTA` changes protocol-visible behaviour (the NIP-11-advertised
  reminder horizon) and exists only as two independent `std::env::var` reads
  (`ingest.rs:1299`, `nip11.rs:97`). It never appears in the config struct, so it is absent
  from any config dump, validation, or startup log — and the enforced value can silently
  diverge from the advertised one.


## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Configuration

All values are read from `crate::config::Config` (constructed once in
`Config::from_env`, `crates/buzz-relay/src/config.rs`) and reached through
`state.config`. No file in this module group calls `std::env::var` outside
`#[cfg(test)]`.

---

#### 1. Env vars consumed by this module group

##### Authentication / admission

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `BUZZ_REQUIRE_AUTH_TOKEN` | `require_auth_token` | **`false`** | `bridge.rs:640`, `:908`, `:1341`, `:2033` | ⚠️ **Permissive.** `false` enables the unsigned `X-Pubkey` impersonation path (`bridge.rs:118-127`). Parsed at `config.rs:475-477`; startup warning at `config.rs:588-593`. **Not present in `.env.example`.** |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | `require_relay_membership` | **`false`** | `api/mod.rs:67`, `bridge.rs:808`, `operator.rs:435` | ⚠️ Open relay by default — any valid key may publish, read, and upload. `config.rs:483-485`; default asserted `config.rs:953-956`. |
| `BUZZ_ALLOW_NIP_OA_AUTH` | `allow_nip_oa_auth` | **`false`** | `api/mod.rs:81` | Gates NIP-OA owner delegation *for admission*. The owner-**backfill** path is intentionally unflagged (`api/mod.rs:151-156`). `config.rs:520-522`. |
| `BUZZ_RATE_LIMIT_HUMAN_API_CALLS_PER_MIN` | `auth.rate_limits.human_api_calls_per_min` | **`300`** | `bridge.rs:29` | Window is a fixed 60 s (`bridge.rs:35`). Default `buzz-auth/src/rate_limit.rs:113-115`; env parse `config.rs:291-294`; documented `.env.example:61`. |

##### Relay identity (used for scheme derivation and HMAC keys)

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `BUZZ_RELAY_URL` (see `config.rs`) | `relay_url` | — | `bridge.rs:635`, `:903`, `:1336`, `:2031`; `media.rs:404`, `:449`; `invites.rs:212`, `:267`; `nip05.rs:55`, `:107`; `operator.rs:219` | **Only its scheme prefix is used** for URL construction (`wss://`/`https://` ⇒ TLS, else plaintext). Its *host* is deliberately never used as an identity — see `bridge.rs:184-193`. `operator.rs:219` is the one exception: `relay_url_authority` extracts the deployment host to protect it from archival. |
| `BUZZ_RELAY_PRIVATE_KEY` | → `state.relay_keypair` | generated if absent (`config.rs:594`) | `bridge.rs:526`, `:1972`; `invites.rs:185`, `:259`, `:300` | Signs 39005/39006/20001 overlays **and** derives the invite/policy HMAC key (`invite_token.rs:112-117`). Rotating it invalidates every outstanding invite — the documented blast-radius control (`invite_token.rs:24-30`). |

##### Media

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `BUZZ_REQUIRE_MEDIA_GET_AUTH` | `require_media_get_auth` | **`false`** | `media.rs:491`, `:517`, `:626`, `:805` | ⚠️ **Permissive.** Off ⇒ `GET`/`HEAD /media/*` unauthenticated and `Cache-Control: public`. Accepts `true`/`1`/`yes`/`on` case-insensitively (`config.rs:682-689`). Default explicitly justified "for staged client rollout" (`config.rs:973-976`). Documented `.env.example:90`. |
| `BUZZ_MEDIA_UPLOADS_PER_MINUTE` | `media_uploads_per_minute` | `30` | `media.rs:96` | Per `(community, pubkey)`, per pod. Must be > 0 (`config.rs:676-680`). `.env.example:87`. |
| `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS` | `media_max_concurrent_uploads` | `8` | `state.rs:730` (`Semaphore`), read at `media.rs:117` | Global per pod. `config.rs:663-668`. `.env.example:85`. |
| `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS_PER_PUBKEY` | `media_max_concurrent_uploads_per_pubkey` | `2`, clamped to the global | `media.rs:126` | `config.rs:669-675`. `.env.example:86`. |
| `BUZZ_MAX_IMAGE_BYTES` | `media.max_image_bytes` | `50 MB` | `media.rs:363`; `router.rs:34` | Contributes to the buffered non-video ceiling **and** to the route body limit. |
| `BUZZ_MAX_VIDEO_BYTES` | `media.max_video_bytes` | `500 MB` | `router.rs:35` | Sets the media-router `RequestBodyLimitLayer` to `max(image, video)` = **500 MB** by default. |
| `BUZZ_MAX_FILE_BYTES` | `media.max_file_bytes` | `100 MB` | `media.rs:364` | Buffered non-video ceiling is `max(image, file)` = **100 MB** in RAM per upload. |
| `BUZZ_MEDIA_UPLOAD_RECORDS` | `media.upload_records_enabled` | **`false`** | `media.rs:247` | Off ⇒ `upload_attribution` returns `None` and no `_uploads/` record is written. `config.rs:651-653`. |
| `BUZZ_MEDIA_UPLOAD_IP_HEADER` | `media.upload_ip_header` | `None` | `media.rs:279` | Trusted-edge header name. **Fail-empty**: missing/malformed/non-public ⇒ nothing recorded; the socket address is never used (`media.rs:243-251`). `config.rs:654-657`. |
| `BUZZ_MEDIA_UPLOAD_PORT_HEADER` | `media.upload_port_header` | `None` | `media.rs:280` | Only kept alongside a valid IP. `config.rs:658-661`. |

##### Join policy (deployment-global, not per-community)

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `BUZZ_TERMS_OF_SERVICE_MARKDOWN` | `join_policy.terms_markdown` | `None` | `invites.rs:82`, `:98` | Size-capped by `MAX_POLICY_MARKDOWN_BYTES` (`config.rs:780-787`). |
| `BUZZ_PRIVACY_POLICY_MARKDOWN` | `join_policy.privacy_markdown` | `None` | `invites.rs:83`, `:107` | same |
| `BUZZ_AGE_ATTESTATION_REQUIRED` | `join_policy.age_attestation_required` | `false` | `invites.rs:84`, `:177` | `config.rs:792` |
| *(derived)* | `join_policy.version` | `sha256(terms ‖ 0x00 ‖ privacy ‖ 0x00‖age)` | `invites.rs:85`, `:178`, `:186`, `:319`, `:333` | Any edit invalidates every outstanding receipt (`config.rs:799-810`). |
| *(derived)* | `join_policy: Option<…>` | `None` when all three inputs are unset | `invites.rs:76`, `:169`, `:314` | `None` ⇒ the claim gate is skipped entirely and all three policy endpoints 404. Default asserted `config.rs:977-980`. **None of these three vars appear in `.env.example`.** |

##### Operator control plane

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `RELAY_OPERATOR_API_ORIGIN` | `relay_operator_api_origin` | `None` | `operator.rs:68` | Deliberately **un-prefixed** (shared relay-identity family, `config.rs:544-548`). Unset ⇒ every operator request **500s** (`operator.rs:69-72`). Parsed by `parse_operator_api_origin` (`config.rs:549-553`). |
| `RELAY_OPERATOR_PUBKEYS` | `relay_operator_pubkeys` | empty | `operator.rs:89-95` | Comma-separated 64-hex, lowercased, deduped. An invalid entry is a **hard config error** (unlike `RELAY_OWNER_PUBKEY`, which warns and ignores) because silently dropping an operator would silently disable provisioning (`config.rs:555-576`). Non-empty without the origin is also a hard error (`config.rs:577-581`). Default asserted `config.rs:961-964`. **Neither var appears in `.env.example`.** |

##### Deployment-admin surface

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `BUZZ_ADMIN_HOST` | `admin.host` | `None` | `admin/auth.rs:8-14`, `:18-21`; `router.rs:56-59`, `:264` | ⚠️ **This single value is the entire authorization boundary** for 5 cross-tenant read endpoints (see security SEC-02). Must be a bare authority — `/`, `\`, `@` are rejected (`config.rs:819-825`). Unset ⇒ the sub-router (and the only security-header middleware in the app) is never mounted. **Not in `.env.example`.** |
| `BUZZ_ADMIN_WEB_DIR` | `admin.web_dir` | `None` | `router.rs:52-55`, `:159-176`, `:266-275` | Validated at startup: must contain `index.html` (`config.rs:826-837`). |

##### Mesh demo

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `BUZZ_MESH` | `mesh.enabled` → `state.mesh()` | **`false`** | `mesh_demo.rs:64` | Strict parse: `on`/`true`/`1` only (`config.rs:509-512`). |
| `BUZZ_MESH_DEMO_ECHO` | `mesh_demo_echo` | **`false`** | `mesh_demo.rs:67` | Same strict parse (`config.rs:514-518`). Both must be on or the route 404s. **Not in `.env.example`.** |

##### Infrastructure reached indirectly

| Env var | Used by this module via | `file:line` |
|---|---|---|
| `REDIS_URL` / `redis_url` | `state.nip98_replay` (replay seen-set), `state.admission_rate_limiter`, `state.pubsub` (presence), mesh `SessionDirectory` | `state.rs:710-712`; `bridge.rs:141`, `:31`, `:1951` |
| `DATABASE_URL` / `database_url` | `state.db` — every handler | throughout |
| `BUZZ_S3_*` | `state.media_storage` | `media.rs:637-679` |
| `BUZZ_CORS_ORIGINS` | `build_cors_layer` — applies to **every** route including this module's | `router.rs:188`, `:373-397` |
| `BUZZ_WEB_DIR`, `BUZZ_SERVE_GIT_WEB_GUI` | SPA fallback that serves `/invite/{code}` | `router.rs:155-186`, `:191-198` |

---

#### 2. Permissive security-relevant defaults (flagged)

| # | Var | Default | Consequence | Cited at |
|---|---|---|---|---|
| 1 | `BUZZ_REQUIRE_AUTH_TOKEN` | `false` | Unsigned `X-Pubkey` header impersonates any pubkey on `/events`, `/query`, `/count`, and all three `/moderation/*` reads; NIP-98 replay protection is bypassed on that path. **Highest-impact default in the module.** | `config.rs:475-477`; `bridge.rs:118-127`, `:150-153` |
| 2 | `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | `false` | Open relay: any valid key may publish, read, and upload media. Also means the unbounded media limiter map (`state.rs:774`) is keyed by an attacker-mintable identity. | `config.rs:483-485`; `api/mod.rs:67-69`; `media.rs:196-206` |
| 3 | `BUZZ_REQUIRE_MEDIA_GET_AUTH` | `false` | `GET`/`HEAD /media/*` unauthenticated; blob responses cached as `public`. `verify_blossom_get_auth` is dead code in this configuration. | `config.rs:682-689`; `media.rs:491-494`, `:517-523` |
| 4 | `BUZZ_CORS_ORIGINS` | empty ⇒ `CorsLayer::permissive()` | Any origin may read cross-origin responses from every route in this module, including `/query` results and `/moderation/*`. Mitigated for the *authenticated* routes by the NIP-98 requirement (no cookies/ambient credentials), but a real concern for `/media/*` when read auth is off. Note the failure path is correct: an unparseable explicit list falls back to **deny**, not permissive (`router.rs:383-392`). | `router.rs:373-397`; `config.rs:595-600` |
| 5 | `BUZZ_ADMIN_HOST` | `None` (safe) — but when **set**, it is the whole boundary | Setting it exposes 5 cross-tenant read endpoints with no credential; a spoofable `Host` header is sufficient. Safe-by-default, unsafe-once-enabled. | `admin/auth.rs:16-33`; `docs/admin/README.md:64-70` |
| 6 | `BUZZ_MAX_VIDEO_BYTES` = 500 MB | route body limit becomes `max(image, video)` | The media router's `RequestBodyLimitLayer` is **500 MB** — 10× the "50 MB limit" claimed in `ARCHITECTURE.md:623`. Bounded in practice by the pre-body auth extractor (`media.rs:29-38`) and the concurrency semaphore. | `router.rs:33-36` |
| 7 | `BUZZ_MAX_FILE_BYTES` = 100 MB | non-video buffered ceiling | Worst case per pod ≈ `max(image, file) × media_max_concurrent_uploads` = 100 MB × 8 = **800 MB** resident. | `media.rs:362-367`; `state.rs:730` |
| 8 | `join_policy` | `None` | The consent gate does not exist unless explicitly configured; `claim` accepts without any receipt. | `config.rs:794-799`; `invites.rs:314-316` |

**Non-permissive defaults worth noting as correct:** `mesh`/`mesh_demo_echo` off with a strict
`on`/`true`/`1` parse so typos fail closed (`config.rs:509-518`); `upload_records_enabled` off with
fail-empty IP capture (`media.rs:243-251`); `allow_nip_oa_auth` off; `relay_operator_pubkeys` empty
so provisioning is disabled until explicitly configured, with a startup hard-error if the pubkeys
are set without the origin (`config.rs:577-581`).

---

#### 3. Hard-coded constants (not configurable)

| Constant | Value | Purpose | `file:line` |
|---|---|---|---|
| `BRIDGE_FEED_MAX_LIMIT` | 100 | feed page cap | `bridge.rs:270` |
| `BRIDGE_THREAD_MAX_LIMIT` | 500 | thread page cap | `bridge.rs:271` |
| `BRIDGE_WINDOW_DEFAULT_LIMIT` / `_MAX_LIMIT` | 50 / 200 | channel-window rows | `bridge.rs:374-375` |
| aux-closure per-hop limit | 1000 | reaction/deletion closure | `bridge.rs:492` |
| FTS `per_page` ceiling | 500 | `limit.min(500)` | `bridge.rs:1665` |
| `REJECT_REASON_MAX_BYTES` | 256 | log truncation of attacker-controlled text | `bridge.rs:595` |
| `MODERATION_READ_LIMIT` | 500 | moderation row cap | `bridge.rs:2077` |
| `MAX_RANGE_CHUNK` | 16 MiB | 206 response cap | `media.rs:587` |
| `MEDIA_UPLOAD_RATE_WINDOW` | 60 s | upload limiter window | `media.rs:66` |
| upload-extractor Blossom window | 3600 s | pre-type-detection permissive window | `media.rs:177` |
| media-read Blossom window | 3600 s | `verify_blossom_get_auth` max age | `media.rs:502` |
| `SNIFF_BYTES` | 4096 | content-sniff prefix | `media.rs:321` |
| `is_safe_ext` bounds | 1–8 chars, `[a-z0-9]` | path-segment gate | `media.rs:533-535` |
| `CLAIM_RATE_WINDOW` / `CLAIM_RATE_LIMIT` / `CLAIM_RATE_CACHE_CAPACITY` | 60 s / 10 / 10 000 | invite-claim limiter | `invites.rs:34`, `:37`, `:41` |
| `DEFAULT_INVITE_TTL_SECS` / `MAX_INVITE_TTL_SECS` / mint TTL floor | 72 h / 30 d / 60 s | invite lifetime | `invite_token.rs:52`, `:55`, `:129` |
| `MAX_CODE_LEN` | 1024 | pre-parse input bound | `invite_token.rs:57` |
| policy-receipt max length / TTL | 2048 bytes / 10 min | receipt bounds | `invite_token.rs:369`, `:350` |
| `KEY_DERIVATION_LABEL` | `b"buzz-invite-v1"` | HMAC domain separation | `invite_token.rs:58` |
| `OPERATOR_REPLAY_SCOPE` | `"operator-management"` | replay namespace | `operator.rs:55` |
| admin `limit` range | 1–200, default 50 | report page cap | `admin/mod.rs:75-83` |
| admin feedback limit | **100, hard-coded** | no client control | `admin/mod.rs:155` |
| admin `summarize_body` cap | 240 chars | summary truncation | `admin/mod.rs:296` |
| admin body limit | 1024 bytes | `RequestBodyLimitLayer` | `admin/mod.rs:39` |
| `ECHO_TIMEOUT` | 10 s | mesh demo wait | `mesh_demo.rs:44` |
| `FILTER_QUERY_CONCURRENCY` | 4 (compile-time asserted 2..=8) | bounded-concurrent DB reads | `handlers/req.rs:35`, `:41` |
| bridge/api body limit | 1 MiB | `RequestBodyLimitLayer` | `router.rs:130` |

Candidates for promotion to config: `MODERATION_READ_LIMIT`, `MAX_RANGE_CHUNK`, the invite TTLs, and
the admin feedback limit — all are operational tuning knobs currently requiring a rebuild.

---

#### 4. `.env.example` coverage gap

Documented (`.env.example`): `BUZZ_RATE_LIMIT_HUMAN_API_CALLS_PER_MIN` (:61),
`BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS` (:85), `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS_PER_PUBKEY` (:86),
`BUZZ_MEDIA_UPLOADS_PER_MINUTE` (:87), `BUZZ_REQUIRE_MEDIA_GET_AUTH` (:90).

**Undocumented but consumed by this module — 11 vars, including the four most
security-significant ones:**

| Var | Why it matters |
|---|---|
| `BUZZ_REQUIRE_AUTH_TOKEN` | the unsigned-`X-Pubkey` switch (SEC-01) |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | open-vs-closed relay |
| `BUZZ_ADMIN_HOST` | the sole authorization boundary for 5 cross-tenant read routes (SEC-02) |
| `BUZZ_ADMIN_WEB_DIR` | admin SPA |
| `RELAY_OPERATOR_API_ORIGIN` | operator NIP-98 audience |
| `RELAY_OPERATOR_PUBKEYS` | operator allowlist |
| `BUZZ_MESH_DEMO_ECHO` | unauthenticated, body-tenanted demo route (SEC-03) |
| `BUZZ_ALLOW_NIP_OA_AUTH` | agent→owner delegation |
| `BUZZ_TERMS_OF_SERVICE_MARKDOWN` | join-policy gate |
| `BUZZ_PRIVACY_POLICY_MARKDOWN` | join-policy gate |
| `BUZZ_AGE_ATTESTATION_REQUIRED` | join-policy gate |

An operator following `.env.example` alone ships a relay where `/events`, `/query`, `/count`, and the
moderation reads accept unsigned `X-Pubkey` impersonation, with no hint in the template that the
knob exists.

---

#### 5. Startup validation performed for this surface

| Check | Behaviour | `file:line` |
|---|---|---|
| `RELAY_OPERATOR_PUBKEYS` entry not 64-hex | **hard error**, refuses to start | `config.rs:565-570` |
| `RELAY_OPERATOR_PUBKEYS` set without `RELAY_OPERATOR_API_ORIGIN` | **hard error** | `config.rs:577-581` |
| `BUZZ_ADMIN_HOST` contains `/`, `\`, or `@` | **hard error** | `config.rs:819-825` |
| `BUZZ_ADMIN_WEB_DIR` without `index.html` | **hard error** | `config.rs:830-836` |
| policy Markdown over `MAX_POLICY_MARKDOWN_BYTES` | **hard error** | `config.rs:780-787` |
| `BUZZ_REQUIRE_AUTH_TOKEN=false` | warning only | `config.rs:588-593` |
| `BUZZ_CORS_ORIGINS` set but nothing parseable | error log + deny-all CORS (not permissive) | `router.rs:383-392` |
| media knobs ≤ 0 | silently fall back to the default via `.filter(\|&v\| v > 0)` | `config.rs:663-680` |
| `BUZZ_MESH*` typos (e.g. `BUZZ_MESH=yes`) | silently **off** | `config.rs:509-518` |
| `BUZZ_REQUIRE_MEDIA_GET_AUTH` accepts `yes`/`on` | more lenient than the other booleans, which take only `true`/`1` | `config.rs:682-689` vs `:475-477` |

The boolean-parsing inconsistency is worth noting: `require_media_get_auth` accepts four spellings,
`require_auth_token` / `require_relay_membership` / `allow_nip_oa_auth` accept two, and the mesh vars
accept three (case-insensitive `on` plus `true`/`1`). An operator writing `BUZZ_REQUIRE_AUTH_TOKEN=yes`
silently gets `false`.


## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: Configuration

#### 1. `BUZZ_GIT_*` environment variables — complete inventory

Ten `BUZZ_GIT_*` names are read anywhere in the shipped binary. Three of them (`_S3_PROBE`, plus the `_S3_*` family) are **test-only**.

| Var | Type | Default | Read at | Stored as | Consumed at |
|---|---|---|---|---|---|
| `BUZZ_GIT_REPO_PATH` | `PathBuf`, `create_dir_all`'d | `./repos` | `config.rs:704-706` | `Config::git_repo_path` (`config.rs:218`) | scratch root for every tempdir/tempfile: `transport.rs:604`, `:622`, `:626`, `:797`, `:884`, `:940`; via `repo_path.parent()` in `cas_publish.rs:1028-1030`; `hydrate.rs:294`, `:229` |
| `BUZZ_GIT_PACK_CACHE_PATH` | `PathBuf`, `create_dir_all`'d | `<git_repo_path>/.pack-cache` | `config.rs:707-712` | `Config::git_pack_cache_path` (`:220`) | `state.rs:704` → `GitPackCache::new` (`pack_cache.rs:107-160`) |
| `BUZZ_GIT_MAX_PACK_BYTES` | `u64` | `500 * 1024 * 1024` = 524 288 000 | `config.rs:713-716` | `Config::git_max_pack_bytes` (`:222`) | router body limit `transport.rs:1757`; receive-pack decoded cap `:861`; `HydrationOptions.max_pack_bytes` `:607`, `:801`, `:887`; `PublishLimits.max_pack_bytes` `:1590`; per-pack + per-idx caps `hydrate.rs:305-317`, `:431-446`; `--max-pack-size` `cas_publish.rs:703` |
| `BUZZ_GIT_MAX_REPO_BYTES` | `u64` | **derived**: `git_max_pack_bytes * 2` (saturating) = 1 048 576 000 | `config.rs:717-720` | `Config::git_max_repo_bytes` (`:227`) | `HydrationOptions.max_repo_bytes` `transport.rs:608`, `:802`, `:888`; `hydrate.rs:318-329`; `PublishLimits.max_repo_bytes` `:1591` → `cas_publish.rs:1136-1146`, `:774-782` |
| `BUZZ_GIT_PACK_CACHE_MAX_BYTES` | `u64` | **derived**: `git_max_repo_bytes * 5` (saturating) = 5 242 880 000 | `config.rs:721-724` | `Config::git_pack_cache_max_bytes` (`:230`) | `state.rs:705` → `GitPackCache.max_bytes` (`pack_cache.rs:71`), used by `prune` (`:365-390`) and the bypass check (`:299-313`) |
| `BUZZ_GIT_PACK_CACHE_MAX_CONCURRENT_POPULATIONS` | `usize`, filtered `> 0` | `2` | `config.rs:725-730` | `Config::git_pack_cache_max_concurrent_populations` (`:232`) | `state.rs:706` → `population_semaphore` (`pack_cache.rs:110-113`, acquired `:253-269`) |
| `BUZZ_GIT_MAX_REPOS_PER_PUBKEY` | `u32` | `100` | `config.rs:731-734` | `Config::git_max_repos_per_pubkey` (`:234`) | `handlers/side_effects.rs:2486-2492` (announce-time quota) — **not** consumed inside this module |
| `BUZZ_GIT_MAX_CONCURRENT_OPS` | `usize` | `20` | `config.rs:735-738` | `Config::git_max_concurrent_ops` (`:236`) | `state.rs:693`, `:729` → `git_semaphore` (`state.rs:517-521`), acquired `transport.rs:322` |
| `BUZZ_GIT_HOOK_HMAC_SECRET` | `String` | **random 32 bytes hex per process** | `config.rs:739-744`; length validated `:863-871` | `Config::git_hook_hmac_secret` (`:239`) | `transport.rs:919-921` (`BUZZ_HOOK_SECRET` env for the hook); `policy.rs:236-241` (verification key) |
| `BUZZ_GIT_S3_PROBE` | `"1"` gate | unset | `store.rs:997`, `hydrate.rs:580`, `cas_publish.rs:1582` | — | **test-only**: enables the live-MinIO test bodies |
| `BUZZ_GIT_S3_{ENDPOINT,ACCESS_KEY,SECRET_KEY,BUCKET,REGION}` | `String` | falls back to `BUZZ_S3_*`, then hardcoded MinIO dev values | `cas_publish.rs:1585-1601` (test helper); `crates/buzz-test-client/tests/e2e_git.rs:118-131` | — | **test-only.** `config.rs` never reads them. |

##### Two `BUZZ_GIT_*` names read outside `config.rs`

| Var | Type | Default | Read at | Effect |
|---|---|---|---|---|
| `BUZZ_GIT_CONFORMANCE_PROBE` | `!= "false"` | enabled | `main.rs:470-472` | when `false`, skips the fatal A3 admission gate entirely |
| `BUZZ_GIT_PROBE_WRITERS` | `usize` | `32` | `main.rs:474-477` | `ProbeConfig.race_width`; must be ≥ 2 (`store.rs:572-586`) |
| `BUZZ_GIT_PROBE_ROUNDS` | `usize` | `3` | `main.rs:478-481` | `ProbeConfig.race_rounds`; must be ≥ 1 |

`ProbeConfig::default()` is `{ race_width: 32, race_rounds: 3 }` (`store.rs:114-121`) — matching the `main.rs` defaults, though `main.rs` builds the struct literally rather than using `Default`.

#### 2. Non-`BUZZ_GIT_*` configuration this module depends on

| Config | Source | Used for |
|---|---|---|
| `config.media.s3_{endpoint,access_key,secret_key,bucket,region}` | `BUZZ_S3_*` | **the git object store.** `state.rs:694-701` builds `GitStore` from the *media* config with `.expect("media storage was already constructed with this S3 config")`. There is no dedicated git bucket at runtime. |
| `config.bind_addr` | `BUZZ_BIND_ADDR`, default `0.0.0.0:3000` (`config.rs:365-368`) | only its **port** — the hook callback URL is `http://127.0.0.1:<port>/internal/git/policy` (`transport.rs:911-915`) |
| `config.relay_url` | `RELAY_URL`, default `ws://localhost:3000` | **scheme only** for the NIP-98 `u` derivation (`transport.rs:236-241`); the host always comes from the resolved tenant |
| `config.require_relay_membership` | `BUZZ_REQUIRE_RELAY_MEMBERSHIP`, **default false** (`config.rs:483-485`) | when false, the membership gate in `GitAuth` is a no-op (`api/mod.rs:130-131`) |
| `config.serve_git_web_gui` | `BUZZ_SERVE_GIT_WEB_GUI`, default false (`config.rs:848-850`) | serves the web bundle at `/`, `/repos`, `/repos/*` (`router.rs:206-213`) |
| `state.relay_keypair` | `BUZZ_RELAY_PRIVATE_KEY` | signs kind:30618 (`transport.rs:1690`) |
| `config.uds_path` | `BUZZ_UDS_PATH` | when set, a second listener without `ConnectInfo` — makes `/internal/git/policy` return 403 over UDS (`main.rs:1179-1188`, `mod.rs:41-49`) |
| `PATH` | process env | inherited into every git subprocess (`transport.rs:304`, `hydrate.rs:458-460`) |

#### 3. Derived-default chain

```
BUZZ_GIT_MAX_PACK_BYTES                 = 500 MiB                     (config.rs:713-716)
  └─ BUZZ_GIT_MAX_REPO_BYTES            = pack  × 2  =   1 GiB        (config.rs:717-720)
       └─ BUZZ_GIT_PACK_CACHE_MAX_BYTES = repo  × 5  =   5 GiB        (config.rs:721-724)

BUZZ_GIT_REPO_PATH                      = ./repos                     (config.rs:704-706)
  └─ BUZZ_GIT_PACK_CACHE_PATH           = <repo_path>/.pack-cache     (config.rs:707-712)
```

Both multiplications use `saturating_mul`, so an extreme `max_pack_bytes` saturates instead of wrapping. Consequences of the chain that are easy to miss:

- Raising `BUZZ_GIT_MAX_PACK_BYTES` alone **silently multiplies the pack-cache disk budget by 10×** unless the two downstream vars are also pinned. Setting `pack=2 GiB` yields a 20 GiB cache budget.
- The pack cache lives **inside** the scratch root by default, so a single volume holds ephemeral workspaces, subprocess tempfiles, compaction tempdirs, and up to 5 GiB of cache — and `git_max_repo_bytes` (1 GiB) is the per-request workspace bound on the *same* volume.
- The Helm chart breaks the co-location by pinning `packCachePath` to its own `emptyDir` (`deploy/charts/buzz/templates/deployment.yaml:148`, `:226-241`).

#### 4. Constants that are **not** configurable

These behave like configuration but are compile-time literals.

| Constant | Value | Site |
|---|---|---|
| `INFO_REFS_TIMEOUT` | 120 s | `transport.rs:42` |
| `PACK_OPS_TIMEOUT` | 300 s | `transport.rs:44` |
| `RECEIVE_PACK_MAX_OUTPUT_BYTES` | 1 MiB | `transport.rs:50` |
| `INFO_REFS_MAX_OUTPUT_BYTES` | 4 MiB | `transport.rs:52` |
| `UPLOAD_PACK_MAX_DECODED_BYTES` | 64 MiB | `transport.rs:59` |
| `MAX_FAST_PATH_REFNAME_LEN` | 4096 | `transport.rs:378` |
| `PKT_LINE_MAX_PAYLOAD` | `0xffff - 4` | `transport.rs:422` |
| `PACK_CAPTURE_TIMEOUT` | 300 s | `cas_publish.rs:81` |
| `PACK_COMPACTION_OPERATION_TIMEOUT` | 600 s | `cas_publish.rs:82` |
| `PACK_OBJECTS_WINDOW_MEMORY_BYTES` | 64 MiB | `cas_publish.rs:83` |
| `PACK_OBJECTS_WINDOW` | `"10"` | `cas_publish.rs:84` |
| `MAX_COMPACTION_OBJECTS` | 1 000 000 | `cas_publish.rs:85` |
| `PACK_COMPACTION_SEMAPHORE` | 1 permit | `cas_publish.rs:86` |
| `MAX_REF_SNAPSHOT_BYTES` | 4 MiB | `cas_publish.rs:270` |
| `MAX_COUNT_OBJECTS_OUTPUT_BYTES` | 64 KiB | `cas_publish.rs:595` |
| `MANIFEST_VERSION` | 1 | `manifest.rs:34` |
| `MAX_MANIFEST_PACKS` | 128 | `manifest.rs:39` |
| `PACK_COMPACTION_THRESHOLD` | 96 (`= 128 * 3 / 4`) | `manifest.rs:45` |
| `MAX_MANIFEST_REFS` | 10 000 | `manifest.rs:47` |
| `MAX_MANIFEST_BYTES` | 4 MiB | `hydrate.rs:45` |
| `HEARTBEAT_INTERVAL` | 60 s | `pack_cache.rs:20` |
| `STALE_SESSION_AGE` | 10 min | `pack_cache.rs:21` |
| `MAX_CALLBACK_AGE_SECS` | 30 s | `policy.rs:51` |
| future-skew tolerance | 5 s | `policy.rs:253` |
| policy body limit | 1 MiB | `mod.rs:63` |
| policy ref-update count | 1..=500 | `policy.rs:213` |
| policy `ref_name` max | 256 | `policy.rs:227` |
| hook `curl --max-time` | 10 s | `hook.rs:129` |
| stderr capture prefix | 64 KiB | `transport.rs:678`, `:1122`; `cas_publish.rs:288`, `:551`, `:761` |
| `MAX_PROTECTION_RULES` / `MAX_PATTERN_LENGTH` / `MAX_WILDCARDS_PER_PATTERN` | 50 / 256 / 3 | `buzz-core/src/git_perms.rs:19-23` |
| `DEFAULT_HEAD` | `refs/heads/main` | `handlers/side_effects.rs:2611` |

The hook's 10 s `curl` timeout sits well inside the 300 s `receive-pack` timeout, so a slow policy endpoint fails the push rather than hanging it.

#### 5. Validation and failure behaviour at load

| Check | Behaviour |
|---|---|
| `BUZZ_GIT_REPO_PATH` / `BUZZ_GIT_PACK_CACHE_PATH` | `create_dir_all`; failure is a hard `ConfigError::InvalidValue` naming the setting (`config.rs:388-400`); pinned by `config.rs:1304-1341` |
| Numeric vars | `.ok().and_then(parse)` — **an unparseable value silently falls back to the default**, no warning. Applies to `_MAX_PACK_BYTES`, `_MAX_REPO_BYTES`, `_PACK_CACHE_MAX_BYTES`, `_MAX_REPOS_PER_PUBKEY`, `_MAX_CONCURRENT_OPS`, and both probe vars (`config.rs:713-738`, `main.rs:474-481`) |
| `_PACK_CACHE_MAX_CONCURRENT_POPULATIONS` | additionally `.filter(|v| *v > 0)`, so `0` falls back to 2 (`config.rs:729`); `GitPackCache::new` rejects 0 defensively (`pack_cache.rs:110-113`) |
| `BUZZ_GIT_HOOK_HMAC_SECRET` | if **set**, must be ≥ 32 chars, else hard error (`config.rs:863-871`); if unset, a 64-hex random value is generated with no warning |
| Pack-cache parent is a symlink | hard error at `GitPackCache::new` → `.expect()` in `state.rs:702-709` ⇒ process abort |
| S3 config half-set (one key empty) | `StoreError::Config` → `.expect()` in `state.rs:694-701` ⇒ process abort |
| A3 conformance probe | any phase failure ⇒ `anyhow` error ⇒ startup aborts (`main.rs:490-495`); `race_width < 2` or `race_rounds == 0` fails the `config` phase (`store.rs:572-586`) |
| `BUZZ_GIT_MAX_PACK_BYTES` on 32-bit | `as usize` cast for the body limit (`transport.rs:1757`) would truncate; irrelevant on 64-bit targets |

Zero of the `BUZZ_GIT_*` numeric vars log a warning on a parse failure, and none is range-checked. A typo like `BUZZ_GIT_MAX_CONCURRENT_OPS=20x` silently yields 20.

#### 6. Deployment surfaces

##### `.env.example` (`.env.example:71-79`)

Documents 5 of the 8 real config vars: `_REPO_PATH`, `_MAX_PACK_BYTES`, `_MAX_REPO_BYTES`, `_PACK_CACHE_PATH`, `_PACK_CACHE_MAX_BYTES`, `_PACK_CACHE_MAX_CONCURRENT_POPULATIONS`. **Missing:** `BUZZ_GIT_MAX_REPOS_PER_PUBKEY`, `BUZZ_GIT_MAX_CONCURRENT_OPS`, `BUZZ_GIT_HOOK_HMAC_SECRET`, and all three probe vars.

##### Helm chart (`deploy/charts/buzz/templates/deployment.yaml`)

| Env | Source |
|---|---|
| `BUZZ_GIT_REPO_PATH` | `.Values.persistence.git.mountPath` (`:146`) |
| `BUZZ_GIT_MAX_PACK_BYTES` | `.Values.git.maxPackBytes` (`:147`) |
| `BUZZ_GIT_PACK_CACHE_PATH` | `.Values.git.packCachePath` (`:148`) |
| `BUZZ_GIT_PACK_CACHE_MAX_BYTES` | `.Values.git.packCacheMaxBytes` (`:149`) |
| `BUZZ_GIT_PACK_CACHE_MAX_CONCURRENT_POPULATIONS` | `.Values.git.packCacheMaxConcurrentPopulations` (`:150`) |
| `BUZZ_GIT_MAX_REPOS_PER_PUBKEY` | `.Values.git.maxReposPerPubkey` (`:151`) |
| `BUZZ_GIT_MAX_CONCURRENT_OPS` | `.Values.git.maxConcurrentOps` (`:152`) |
| `BUZZ_GIT_HOOK_HMAC_SECRET` | Kubernetes Secret key (`:168-172`) |
| `BUZZ_S3_*` | ConfigMap + Secret (`:157-159`, `:191-201`) |

Volumes: `git-repos` (PVC or `emptyDir` with `sizeLimit`) at the repo path, and `git-pack-cache` as a dedicated `emptyDir` with `packCacheVolumeSize` (`:226-241`). **`BUZZ_GIT_MAX_REPO_BYTES` is not set by the chart**, so it always takes the `pack × 2` derived value — and if an operator raises `maxPackBytes`, both the repo bound and the cache bound move with it while `packCacheVolumeSize` does not. A `packCacheMaxBytes` larger than `packCacheVolumeSize` results in ENOSPC rather than eviction.

#### 7. Deltas found

| Claim | Source | Reality |
|---|---|---|
| `BUZZ_GIT_S3_BUCKET/_ENDPOINT/_REGION/_ACCESS_KEY/_SECRET_KEY` listed as relay configuration | `.aidlc/reverse-engineer/configuration.md:159` | **Not configuration.** No `config.rs` read exists; the names appear only in `#[cfg(test)]` helpers (`cas_publish.rs:1585-1601`) and the e2e test (`crates/buzz-test-client/tests/e2e_git.rs:118-131`). At runtime the git store reuses `config.media.s3_*` (`state.rs:694-701`). |
| `BUZZ_GIT_HOOK_HMAC_SECRET` "required (git enabled)" | `.aidlc/reverse-engineer/configuration.md:160` | Optional — a random per-process secret is generated when unset (`config.rs:739-744`), and that is functionally safe because the hook always dials `127.0.0.1` into the same process (`transport.rs:911-915`). |
| `BUZZ_GIT_MAX_REPO_BYTES` "consumed at `api/git/hydrate.rs`" | `.aidlc/reverse-engineer/_work/modules/relay-core-configuration.md:98` | Also consumed on the publish side via `PublishLimits` (`transport.rs:1591` → `cas_publish.rs:1136-1146`, `:774-782`). |
| `BUZZ_GIT_MAX_PACK_BYTES` "consumed at `transport.rs:1757` (body limit)" | `relay-core-configuration.md:97` | Six additional consumers: decoded-gzip cap (`transport.rs:861`), three `HydrationOptions` sites, `PublishLimits`, per-pack/per-idx caps (`hydrate.rs:305-317`, `:431-446`), and `--max-pack-size` (`cas_publish.rs:703`). |
| `BUZZ_GIT_PACK_CACHE_MAX_BYTES` default "5 GB" | `.env.example:78` shows `5368709120` | Consistent: `1 048 576 000 × 5 = 5 242 880 000`. The `.env.example` literal `5368709120` (= 5 × 2³⁰) is **not** the derived default — the derived value is 5 242 880 000. Setting the documented value changes behaviour by ~125 MB. |
| Doc: "Each process may retain a byte-bounded, process-lifetime cache … Deployment mounts that cache on a per-pod ephemeral volume" | `docs/git-on-object-storage.md` §v1 | Matches the chart (`deployment.yaml:238-241`) but **not** the code default, which nests the cache inside the scratch root (`config.rs:707-712`). |


## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: Configuration

---

#### 1. Environment variables read *directly* by the 12 assigned files

Exactly **4** env vars are read inside this module's own code, all in `storage_sweep.rs`. Every other configuration input arrives pre-parsed on `state.config`.

| Variable | Parse | Default | Floor/validation | Read at | Consumed at |
|---|---|---|---|---|---|
| `BUZZ_STORAGE_SWEEP_INTERVAL_SECS` | `u64`, unparseable ⇒ default | `3600` | `.max(60)` — a misconfigured value cannot become a listing busy-loop (`:36-38`) | `storage_sweep.rs:56-60` | `main.rs:1462` → `maybe_spawn_sweep(interval)` → `should_spawn` (`storage_sweep.rs:105-127`) |
| `BUZZ_STORAGE_SWEEP_TIMEOUT_SECS` | `u64`, unparseable ⇒ default | `120` | none | `storage_sweep.rs:61-64` | `main.rs:1463` → `tokio::time::timeout` (`storage_sweep.rs:249-252`) |
| `BUZZ_STORAGE_SWEEP_MAX_OBJECTS` | `u64`, unparseable ⇒ default | `1_000_000` | none | `storage_sweep.rs:65-68` | `main.rs:1458`, `:1464-1467` → `buzz_media::fold_bucket_listing` cap check (`bucket_index.rs:392-395`) |
| `BUZZ_STORAGE_METRICS` | trimmed + lowercased; `"off"` ⇒ disabled, anything else **including unset** ⇒ enabled | enabled | none | `storage_sweep.rs:69-73` | `main.rs:1454-1456` — early return suppressing the sweep *and every* storage gauge including health ones |

##### 1.1 Read timing and caching

All four are read **once**, on the first leader tick, behind a function-local `OnceLock` in `main.rs`:

```
static SWEEP_CONFIG: OnceLock<StorageSweepConfig> = OnceLock::new();   // main.rs:1447-1448
let config = *SWEEP_CONFIG.get_or_init(StorageSweepConfig::from_env);   // main.rs:1453
```

The `OnceLock` is deliberately function-local rather than on `Config`/`AppState`, with the in-code rationale that this is localized feature config with a single consumer, stable for the process lifetime because env is immutable (`main.rs:1449-1452`). Consequence: these four vars are **not** validated at boot — a typo is silently absorbed into the default, and nothing is discoverable until the first leader tick.

`StorageSweepConfig` is `Copy` (`storage_sweep.rs:33`), so the deref-copy at `main.rs:1453` is cheap.

##### 1.2 Silent-default behaviour

All four use the `.ok().and_then(parse).unwrap_or(default)` idiom, so `BUZZ_STORAGE_SWEEP_TIMEOUT_SECS=abc` yields 120 with **no warning**. This is the opposite of the crate's `Config::from_env` convention, where an invalid value is a hard `ConfigError` (`config.rs:764-771` for `BUZZ_PUSH_GATEWAY_TIMEOUT_MS`, `:747-751` for `BUZZ_PUSH_EXECUTOR_KEY_ID`, `:563-568` for `RELAY_OPERATOR_PUBKEYS`). Two config subsystems, two error philosophies.

##### 1.3 Kill-switch semantics

Only the exact (trimmed, case-insensitive) string `off` disables (`storage_sweep.rs:69-73`). `BUZZ_STORAGE_METRICS=false`, `=0`, `=disabled` all **enable** the feature. This differs from `config.rs`'s `parse_bool` helper, which accepts `true|1|on` / `false|0|off|""` (`config.rs:364-378`) — and `storage_sweep` does not use it. A unit test pins the asymmetry (`storage_sweep.rs:379-397`, helper `:395-397`).

Documented for operators at `deploy/charts/buzz/values.yaml:308-313`.

---

#### 2. `Config` fields consumed by this module

Read from `state.config`, parsed at boot in `crates/buzz-relay/src/config.rs`.

| Field | Env var | Default | Boot validation | Consumed in this module |
|---|---|---|---|---|
| `relay_operator_pubkeys: Vec<String>` | `RELAY_OPERATOR_PUBKEYS` (comma-separated 64-hex, deduped, lowercased) | **empty ⇒ provisioning disabled** (`config.rs:962-963`) | each entry must be valid 64-hex (`config.rs:563-568`); **must be paired with `RELAY_OPERATOR_API_ORIGIN` or boot fails** (`config.rs:577-580`) | `community_provisioning.rs:258-266` — the operator gate |
| `require_relay_membership: bool` | `REQUIRE_RELAY_MEMBERSHIP` | — | — | `community_provisioning.rs:215` — gates NIP-43 snapshot publication; also overridden to `false` in two test helpers (`workflow_sink.rs:577`) |
| `push_gateway_delivery_url: Option<url::Url>` | `BUZZ_PUSH_GATEWAY_DELIVERY_URL` | **`https://push.buzz.xyz/v1/deliveries/apns`** (`config.rs:339`, `:755-758`) | exact HTTPS + host + no userinfo + path `/v1/deliveries/apns` + no query + no fragment (`config.rs:341-361`) — else boot error | `push_lease.rs:480-482` (gates lease acceptance), `push_runtime.rs:422-424` (delivery target), `main.rs:686` (gates both worker spawns) |
| `push_gateway_timeout: Duration` | `BUZZ_PUSH_GATEWAY_TIMEOUT_MS` | `2000` ms (`config.rs:771`) | integer in `100..=10_000`, else boot error (`config.rs:759-770`) | `push_runtime.rs:314` — whole-request `reqwest` timeout |
| `push_executor_key_id: String` | `BUZZ_PUSH_EXECUTOR_KEY_ID` | `"relay-v1"` (`config.rs:745-746`) | 1..=64 bytes, else boot error (`config.rs:747-751`) | `push_lease.rs:484-486` — must equal the lease's `exec` tag |
| `relay_url: String` | `RELAY_URL` | — | — | `push_lease.rs:494` → `canonical_origin` scheme derivation (`:585-596`); `product_feedback.rs:29` → tenant media base |
| `media: MediaConfig` | `BUZZ_S3_*`, `BUZZ_MEDIA_*` | see §3 | — | via `state.media_storage` in `report.rs:69`, `product_feedback.rs:35`, and the sweep's `list_page` (`main.rs:1462-1470`) |
| `database_url`, `redis_url`, `auth` | — | — | — | **test-only** construction in `identity_archive.rs:441-448` and `workflow_sink.rs:576-596` |

##### 2.1 The push-gateway default is the most consequential setting in this module

Because unset falls back to a concrete third-party URL rather than `None`:

| Consequence | Site |
|---|---|
| Lease acceptance is **on** by default | `push_lease.rs:480-482` returns `push not supported` only when the URL is `None` |
| The matcher **and** delivery worker are spawned by default | `main.rs:686-692` — "enabled as one unit" |
| NIP-11 advertises push support by default | `nip11.rs:208-209` |
| Outbound HTTPS to `push.buzz.xyz` is attempted by default | `push_runtime.rs:422-428` |

Disabling requires setting the variable to an **empty** string (`config.rs:753` matches `Ok(raw) if raw.trim().is_empty() => None`). Unsetting it does the opposite of what an operator would expect. A unit test pins both behaviours (`config.rs:1189-1212`).

---

#### 3. Indirect configuration (consumed via `state`, not read here)

| Setting | Env var | Default | Why this module cares |
|---|---|---|---|
| Usage-tick cadence | `BUZZ_USAGE_METRICS_INTERVAL_SECS` | `300`, `.max(5)` (`main.rs:1257-1262`) | **This is the storage sweep's real retry cadence after a failure** — `should_spawn` returns `true` unconditionally on `!ok` (`storage_sweep.rs:110`), so a permanently failing sweep retries every tick, not every `interval` (documented `:89-103`) |
| Gauge idle timeout | `BUZZ_USAGE_METRICS_IDLE_TIMEOUT_SECS` | `900` (`main.rs:1264-1269`) | the reason `emit_storage_metrics` explicitly zeroes disappearing per-community series instead of relying on idle eviction (`storage_sweep.rs:284-291`) |
| S3 endpoint / credentials / bucket / region | `BUZZ_S3_ENDPOINT`, `BUZZ_S3_ACCESS_KEY`, `BUZZ_S3_SECRET_KEY`, `BUZZ_S3_BUCKET` (default `buzz-media`), `BUZZ_S3_REGION` (falls back to `AWS_REGION`) | `config.rs:620-629` | the sweep lists the whole bucket with these credentials — needs `s3:ListBucket` added to the read-write media key |
| Relay keypair | `RELAY_PRIVATE_KEY` (random if unset, `config.rs:740-744`) | — | signs notice DMs (`moderation_notices.rs:165`, `:198`), workflow messages (`workflow_sink.rs:304`), the outbound NIP-98 header (`push_runtime.rs:562`), and decrypts lease plaintext (`push_lease.rs:488`) |
| Relay owner bootstrap | `RELAY_OWNER_PUBKEY` | — | the **only** supported path to owner promotion, since 9030 and 9032 both refuse `role=owner` (`relay_admin.rs:184-186`, `:300-302`); referenced in-code at `relay_admin.rs:298` and `community_provisioning.rs:38`, `:245` |
| Operator API origin | `RELAY_OPERATOR_API_ORIGIN` | — | NIP-98 URL binding for `POST /operator/communities` (`api/operator.rs:155-163`); required whenever the allowlist is non-empty (`config.rs:577-580`) |
| `BUZZ_RELAY_BASE_URL`, `BUZZ_API_TOKEN`, `BUZZ_RELAY_PUBKEY` | — | `http://localhost:3000` | read by `buzz-workflow`'s `add_reaction_impl` (`buzz-workflow/src/executor.rs:886`, `:895`, `:897`), **not** by `workflow_sink`. That action targets an unregistered route, so these three vars configure a permanently broken path |

---

#### 4. Hard-coded constants that behave like configuration

None of these are env-tunable. They are policy that requires a rebuild to change.

##### 4.1 Freshness / skew

| Constant | Value | Site |
|---|---|---|
| `MAX_COMMAND_SKEW_SECS` (9040–9044) | `120` | `moderation_commands.rs:81` |
| relay-admin skew (bare literal) | `120` | `relay_admin.rs:125` |
| identity-archive skew (bare literal) | `120` | `identity_archive.rs:147` |
| `ALLOWED_SKEW` (lease expiration) | `120` | `push_lease.rs:476` |

Four independent declarations of the same policy value; only one is named.

##### 4.2 Push-lease policy

| Constant | Value | Site |
|---|---|---|
| `MAX_LEASE_TTL` | 30 days (`30*24*60*60`) | `push_lease.rs:475` |
| `MAX_CONTENT` (ciphertext) | 65,536 bytes | `:477` |
| `MAX_PLAINTEXT` | 32,768 bytes | `:478` |
| `MAX_ACTIVE_LEASES` per author | 16 | `:479` |
| `PUSH_KINDS` | `[7, 9, 1059, 40007, 46010]` | `:15` — pinned to `migrations/0018_push_match_queue.sql` by a test (`:696-710`) |
| `URGENT_KINDS` | `[]` (empty) | `:16` — advertised in NIP-11 (`nip11.rs:209`) |
| `MAX_SAFE_JSON_INTEGER` | `2^53 - 1` | `:21` |
| app profiles | `buzz-ios-production`, `buzz-ios-sandbox` (both `apns`) | `:504-512` |
| supported classes | `silent`, `default`, `time_sensitive` | `:509` |
| `max_subscriptions` / `max_kinds` | 16 / 16 | `:513-514` |
| `max_authors` / `max_h` / `max_tag_values` / `max_ignore` | 20 / 50 / 20 / 8 | `:515-518` |
| `max_endpoint_len` / `max_string_len` | 4096 / 512 | `:519-520` |
| `d`-tag length bound | 1..=64 | `:139-141` |

##### 4.3 Push runtime

| Constant | Value | Site |
|---|---|---|
| `CLAIM_SECS` | 30 | `push_runtime.rs:15` |
| `EVENT_USEFUL_SECS` | 3600 | `:16` |
| `MAX_ATTEMPTS` (delivery) | 8 | `:17` |
| `MATCH_BATCH_LIMIT` | 64 | `:20` |
| `IDLE_POLL_FLOOR` / `IDLE_POLL_CEILING` (matcher) | 250 ms / 2 s | `:24-25` |
| delivery-worker idle floor | 500 ms (inline) | `:317`, `:341` |
| `REAP_INTERVAL` | 30 s | `:29` |
| wake claim batch size | 16 (inline) | `:325` |
| default retry delay | 2 s (inline, 4 sites) | `:392`, `:481`, `:486`, `:498` |
| backoff shift clamp | `2^6` | `:544` |
| `MAX_MATCH_ATTEMPTS` | 8 (`buzz-db/src/push.rs:70`) | referenced `:172`, `:190` |
| batch-retry delay | 2 s | `:141`, `:210` |
| claim-error sleep | 2 s | `:84` |

##### 4.4 Moderation / admin / feedback limits

| Constant | Value | Site |
|---|---|---|
| `MAX_WORKSPACE_ICON_URL_LEN` | 2048 | `relay_admin.rs:60` |
| `MAX_WORKSPACE_ICON_DATA_URL_LEN` | 98,304 (~72 KB image) | `relay_admin.rs:64` |
| `MAX_HOST_LEN` | 255 (matches `communities.host VARCHAR(255)`) | `community_provisioning.rs:39` |
| DNS total / label length | 253 / 63 | `community_provisioning.rs:151`, `:158` |
| notice idempotency scan limit | 1000 (the query clamp) | `moderation_notices.rs:240`, rationale `:222-226` |
| `MAX_BODY_BYTES` (feedback) | 32,768 | `product_feedback.rs:12` |
| `MAX_TAGS_BYTES` (feedback) | 65,536 | `product_feedback.rs:13` |
| feedback categories | `bug`, `praise`, `needs-work` | `product_feedback.rs:11` |
| `REPORT_TYPES` | 7 NIP-56 values | `report.rs:29-37` |
| report note size | **uncapped** | `report.rs:222-228` |
| `MODERATION_READ_LIMIT` | 500 | `api/bridge.rs:2089` |
| `MODERATION_SOURCE_TAG` | `"moderation_source"` | `moderation_notices.rs:38` |
| `MODERATION_ACTION_CHECK_VOCAB` | 12 values, must match `migrations/0006_moderation.sql` | `buzz-db/src/moderation.rs:104-117`, pinned by `moderation_commands.rs:659-667` |

**Notable gaps:** ban/timeout durations have no configurable maximum (any `expiration` in `i64` range is accepted, `moderation_commands.rs:592-606`); the report note has no size cap while feedback does; the push quotas are the only limits documented as "advertised descriptor policy" yet none of them are actually advertised via NIP-11 except `push_kinds`/`urgent_kinds`.

---

#### 5. Legacy `SPROUT_*` names

**Zero `SPROUT_*` variables are read by any of the 12 assigned files.** Verified by grep.

For completeness, the legacy names surviving elsewhere in `buzz-relay`:

| Variable | Read at | Purpose |
|---|---|---|
| `SPROUT_MAX_NOT_BEFORE_DELTA` | `nip11.rs:97`, `ingest.rs:1325` | advertised in NIP-11 and enforced at ingest |
| `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS` | `main.rs:701` | NIP-ER reminder scheduler cadence |
| `SPROUT_REMINDER_SCHEDULER_BATCH_LIMIT` | `main.rs:705` | NIP-ER batch size |

All three belong to other module groups. This module group is fully on the `BUZZ_*` prefix.

---

#### 6. Configuration-related findings

| # | Severity | Finding |
|---|---|---|
| C-1 | **HIGH** | `BUZZ_PUSH_GATEWAY_DELIVERY_URL` unset ⇒ `Some("https://push.buzz.xyz/v1/deliveries/apns")` (`config.rs:755-758`), which enables lease acceptance (`push_lease.rs:480`), spawns both background workers (`main.rs:686-692`), advertises push in NIP-11 (`nip11.rs:208`), and attempts outbound HTTPS carrying the device token to a host the operator never chose. Disabling requires an explicitly **empty** value (`config.rs:753`) — unsetting does the opposite of the intuitive thing. |
| C-2 | MEDIUM | The four `BUZZ_STORAGE_*` vars silently absorb invalid values into defaults (`storage_sweep.rs:56-73`), unlike every `Config::from_env` field which hard-fails at boot (`config.rs:747-751`, `:764-771`, `:563-568`). A typo is undiscoverable. |
| C-3 | MEDIUM | Those same four vars are read only on the **first leader tick** behind a function-local `OnceLock` (`main.rs:1447-1453`), so a non-leader pod never validates them and a misconfiguration surfaces late and only on one pod. |
| C-4 | MEDIUM | The storage sweep's effective retry cadence after a failure is `BUZZ_USAGE_METRICS_INTERVAL_SECS` (default 300 s), not `BUZZ_STORAGE_SWEEP_INTERVAL_SECS` (`storage_sweep.rs:105-127`). The coupling is documented in-code (`:89-103`) but the variable name implies otherwise. |
| C-5 | LOW | `BUZZ_STORAGE_METRICS` uses bespoke `off`-only parsing rather than the crate's `parse_bool` (`config.rs:364-378`), so `false`/`0`/`disabled` all **enable** the feature. Pinned by a test that documents the asymmetry (`storage_sweep.rs:379-397`). |
| C-6 | LOW | `BUZZ_STORAGE_SWEEP_TIMEOUT_SECS` and `_MAX_OBJECTS` have no floor or ceiling. `TIMEOUT_SECS=0` yields an immediately-timing-out sweep that respawns on every usage tick forever (`should_spawn` returns `true` on `!ok`, `storage_sweep.rs:110`). Only `INTERVAL_SECS` is floored. |
| C-7 | LOW | Owner promotion has no event-based path — 9030 and 9032 both refuse `owner` (`relay_admin.rs:184-186`, `:300-302`) — so it is config-only via `RELAY_OWNER_PUBKEY` or the operator endpoint. 9030's error message "use kind:9032 to promote to owner" (`:185`) points at a path that also refuses. |
| C-8 | LOW | 12 push-lease quotas plus 13 push-runtime timing constants are compile-time only (`push_lease.rs:475-521`, `push_runtime.rs:15-29`). Tuning batch size, claim lease, retry ceiling, or per-author lease quota for a differently-sized deployment requires a rebuild. |
| C-9 | LOW | `moderation_notices.rs:240` hard-codes the idempotency scan limit to 1000 with the reasoning that "1000 moderation notices to one user in one community is a practical ceiling" (`:225-226`). Beyond that, crash-retry can re-send a duplicate notice. |
| C-10 | INFO | Only `BUZZ_STORAGE_*` appears in `deploy/charts/buzz/values.yaml` (`:308-313`) among this module's direct env surface. The push-gateway vars are not documented in the chart. |


## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Configuration

---

#### 1. Environment variables consumed (directly or via `Config`)

| Env var | Parse rule | Default | Read at | Consumed in this group at |
|---|---|---|---|---|
| `BUZZ_HUDDLE_AUDIO_AVAILABLE` | `!(v == "false" \|\| v == "0")` — i.e. **anything except `false`/`0` is true**, including typos | **`true`** | `config.rs:487-491` | `handler.rs:357` |
| `BUZZ_MESH` | `eq_ignore_ascii_case("on") \|\| v == "true" \|\| v == "1"` | **`false`** (absent, `off`, or a typo → off) | `config.rs:494-500` | `mesh_boot.rs:418-421` |
| `BUZZ_MESH_BIND_ADDR` | `SocketAddr::parse`; a parse failure is a **hard `ConfigError::InvalidValue`**; an *absent* var falls back to the default | `0.0.0.0:3478` | `config.rs:501-508` | `mesh_boot.rs:423-431`, logged `:436` |
| `BUZZ_MESH_DEMO_ECHO` | same strict `on`/`true`/`1` as `BUZZ_MESH` | **`false`** | `config.rs:514-518` | `mesh_boot.rs:293-297`; gate `api/mesh_demo.rs:70-72`; passed at `main.rs:456` |
| `BUZZ_MESH_ADVERTISE_ADDR` | trimmed; empty string ignored | unset | **read directly from the environment**, not via `Config` | `mesh_boot.rs:384-389` |
| `POD_IP` | trimmed; used only when a bound port is known | unset | **read directly** | `mesh_boot.rs:394-399` |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | `v == "true" \|\| v == "1"` | **`false`** | `config.rs:481-483` | reached via `enforce_relay_membership` at `handler.rs:244`; the gate itself short-circuits at `api/mod.rs:67` |
| `REDIS_URL` | used only by test helpers | `redis://127.0.0.1:6379` | `directory.rs:593`, `reliable.rs:708`, `api/mesh_demo.rs:170` | test-only |

`BUZZ_MESH_ADVERTISE_ADDR` and `POD_IP` are the only two env vars in this group read
with a bare `std::env::var` instead of going through `Config`
(`mesh_boot.rs:384`, `:394`). They are therefore invisible to `Config::from_env`'s
validation and to any config-dump/diagnostic surface.

##### 1.1 Advertise-address preference chain (`mesh_boot.rs:379-402`)

1. `BUZZ_MESH_ADVERTISE_ADDR` if set and non-empty → used verbatim, **no
   `SocketAddr` validation** (`mesh_boot.rs:384-389`). A malformed value is
   published to the ready registry and silently breaks peer dialing.
2. `POD_IP` + the actual bound port, when both are usable
   (`mesh_boot.rs:392-399`). Comment: "k8s Downward API, zero RBAC".
3. Every IP transport address the endpoint reports (`mesh_boot.rs:401`).

The bound port is taken from `ip_addrs().first()` (`mesh_boot.rs:392-393`); if that
is empty the port is `0` and branch 2 is skipped, falling through to an empty
vector from branch 3 — publishing a ready record with **no addresses** and no error.

---

#### 2. `Config` fields consumed

| Field | Type | Default | Definition | Read at |
|---|---|---|---|---|
| `huddle_audio_available` | `bool` | `true` | `config.rs:114-129` | `handler.rs:357` |
| `mesh: buzz_relay_mesh::MeshConfig` | struct | see §2.1 | `config.rs:131-136` | `mesh_boot.rs:418`, `:423`, `:445` |
| `mesh_demo_echo` | `bool` | `false` | `config.rs:138-144` | `main.rs:456` → `mesh_boot.rs:293` |
| `relay_url` | `String` | — | (elsewhere) | `handler.rs:219` via `nip42_expected_relay_url` |
| `require_relay_membership` | `bool` | `false` | `config.rs:110-112` | indirectly, `api/mod.rs:67` |

##### 2.1 `MeshConfig` (`buzz-relay-mesh/src/lib.rs:54-63`)

| Field | Source | Default | Tunable? |
|---|---|---|---|
| `enabled` | `BUZZ_MESH` | `false` | yes |
| `bind_addr` | `BUZZ_MESH_BIND_ADDR` | `0.0.0.0:3478` | yes |
| `registry_refresh` | **hard-coded** `Duration::from_secs(15)` | 15 s | **no** — `config.rs:511` writes a literal |

`registry_refresh` is a `MeshConfig` field with no env var behind it. It controls the
ready-registry heartbeat cadence (`mesh_boot.rs:445`), so ready-record staleness on
a mesh deployment is not operator-tunable.

---

#### 3. Other `AppState` inputs this group depends on

| Input | Where set | Consumed |
|---|---|---|
| `conn_semaphore: Arc<Semaphore>` sized by `max_connections` | `state.rs:727` | `handler.rs:90` — **shared with ordinary relay WebSockets**, so `max_connections` is the only operator-tunable audio limit |
| `community_connections: Arc<CommunityConnectionRegistry>` | `state.rs:724` | `handler.rs:153-164` |
| `audio_rooms: Arc<AudioRoomManager>` | `state.rs:768` | throughout |
| `relay_keypair: nostr::Keys` | `state.rs` | `handler.rs:1268` (lifecycle-event signing); `mesh_boot.rs:441-443`, `:452` (mesh attestation) |
| `redis_pool: deadpool_redis::Pool` | `state.rs` | `main.rs:444` → `mesh_boot.rs:442`, `:512` |
| `shutting_down: Arc<AtomicBool>` | `state.rs:769` | `main.rs:445`, `:457` → `mesh_boot.rs:466-472`, `:481-496`, `:317` |
| `tracer: Arc<dyn Tracer>` | `state.rs:794-798` | **unconditionally `NoopTracer`** |
| `mesh: Arc<OnceLock<MeshHandle>>` | `state.rs:627`, set `main.rs:458` | `state.rs:812-814` |

---

#### 4. `BUZZ_HUDDLE_AUDIO_AVAILABLE` — verifying the documented single-pod constraint

##### 4.1 What the doc says

`config.rs:114-129`:

> Huddle audio frames are relayed peer-to-peer *within a single pod*
> (`AudioRoomManager` is an in-process map; only huddle lifecycle events cross pods
> via Redis). Under horizontal scaling … two peers in the same huddle can land on
> different pods and never hear each other. … Operators running multiple relay pods
> MUST set `BUZZ_HUDDLE_AUDIO_AVAILABLE=false` until the out-of-relay media/SFU
> service lands.

##### 4.2 What actually breaks with more than one pod, mesh off

Verified accurate. `AudioRoomManager` is a per-process `DashMap`
(`room.rs:490-492`), constructed once per `AppState` (`state.rs:768`). Frame
fan-out (`room.rs:398-411`) iterates only that map. Nothing crosses pods except the
three lifecycle events, which go through `pubsub.publish_event`
(`handler.rs:1322-1325`). So two peers on different pods:

- each create their own `Room` for the same `(community, channel)` key;
- each independently pin a protocol version;
- each independently allocate `peer_index` from 0, so indices collide across pods;
- see a `joined` roster containing only their own pod's peers
  (`handler.rs:614-619` from `room.peer_pubkeys()`);
- hear nothing from the other pod;
- **each become "the last peer"** on leave and each run
  `remove_peer_and_check_ended` → `archive_channel` (`handler.rs:833-866`), so the
  channel is archived while the other pod's peer is still connected, and two 48103
  events are emitted (the second de-duplicated only if the event ids collide, which
  they will not — different `created_at`/content ordering).

The failure is worse than "never hear each other": the auto-end/archive path is
actively wrong under multi-pod. So the `MUST` in the doc is well-founded.

##### 4.3 Does the mesh path change that? Yes — and it also bypasses the gate

`handler.rs:306-378` is a two-arm match on `state.mesh()`:

```rust
match state.mesh() {
    Some(mesh) => { /* drain check, resolve_join_owner_ready */ }
    None => { if !state.config.huddle_audio_available { /* reject */ } }
}
```

**`huddle_audio_available` is only consulted on the `None` arm.** With `BUZZ_MESH=on`,
setting `BUZZ_HUDDLE_AUDIO_AVAILABLE=false` has **no effect at all** — joins are
accepted and routed through the mesh. This is intentional (the comment at
`handler.rs:289-296` says "When the mesh is off, we keep today's behavior exactly —
including the `huddle_audio_available=false` rejection under a non-mesh horizontal
deployment") but the *field documentation* at `config.rs:120-128` was written before
the mesh landed and still says operators "MUST set
`BUZZ_HUDDLE_AUDIO_AVAILABLE=false`" with no mention of the mesh. **Documented
delta.** An operator following `config.rs` on a mesh-enabled multi-pod deployment
would set a flag that does nothing.

With the mesh on, multi-pod huddles work correctly: one pod owns the room (Redis
CAS, `join.rs:317-379`), that pod is the sole `peer_index` allocator
(`mesh.rs:18-24`), and ingress pods never archive (`handler.rs:803-810`). So the
mesh does remove the single-pod constraint — the config doc has simply not caught up.

##### 4.4 The `false` value is nearly unreachable

The parse rule is `!(v == "false" || v == "0")` (`config.rs:489-491`). Only the exact
lowercase `false` or `0` disables it. `False`, `FALSE`, `no`, `off`, `disabled`, and
an empty string all evaluate to **true**. Contrast `BUZZ_MESH`, which uses
`eq_ignore_ascii_case("on")` (`config.rs:498-500`) — three flags in the same file
with three different parse conventions:

| Flag | Convention | Case-insensitive? |
|---|---|---|
| `BUZZ_HUDDLE_AUDIO_AVAILABLE` | deny-list (`false`/`0` → off) | no |
| `BUZZ_MESH` | allow-list (`on`/`true`/`1` → on) | `on` only |
| `BUZZ_MESH_DEMO_ECHO` | allow-list (`on`/`true`/`1` → on) | `on` only |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | allow-list (`true`/`1` → on) | no |

##### 4.5 Test coverage of the defaults

| Assertion | Line |
|---|---|
| `huddle_audio_available` defaults true | `config.rs:980-984` |
| `BUZZ_HUDDLE_AUDIO_AVAILABLE=false` disables it | `config.rs:1256-1265` |
| `BUZZ_MESH` absent → mesh off | `mesh_boot.rs:546-556` (skips if externally forced) |
| mesh off boots nothing (unroutable Redis proves it) | `mesh_boot.rs:527-541` |

---

#### 5. `BUZZ_MESH` / `BUZZ_MESH_BIND_ADDR` gating — verified behaviour

| Condition | Behaviour |
|---|---|
| `BUZZ_MESH` absent/off | `boot_mesh` returns `Ok(None)` at `mesh_boot.rs:418-421` before touching anything. `state.mesh()` stays `None`, so every consumer takes its single-pod branch (`handler.rs:306`, `:449`, `:577`, `:875`). `/_mesh` returns `{"enabled": false}` (`router.rs:404`). `/_mesh/demo/echo` 404s (`api/mesh_demo.rs:66-68`). No UDP port bound, no Redis key written, no task spawned — pinned by `mesh_boot.rs:527-541` using an unroutable Redis pool |
| `BUZZ_MESH=on`, bind succeeds, publish succeeds | endpoint bound on `bind_addr`, boot-unique `RuntimeId` (`mesh_boot.rs:423-432`), ready record published (`:459-464`), heartbeat spawned (`:466-472`), `MeshRuntime` started (`:475`), immediate `reconcile_now()` (`:478`), drain watcher spawned (`:481-496`), dispatcher installed as the single inbound slot (`:505`) |
| `BUZZ_MESH=on`, bind fails | **fatal** — `anyhow!("mesh endpoint bind on {} failed: {e}")` propagates to `main.rs:442` and aborts startup (`mesh_boot.rs:423-431`) |
| `BUZZ_MESH=on`, first ready-registry publish fails | **fatal** — `anyhow!("mesh ready-registry publish failed: {e}")` (`mesh_boot.rs:459-463`). Rationale at `:457-458`: "if Redis can't take the attested record, peers can never find us — fail loudly now, not quietly forever" |
| `BUZZ_MESH_BIND_ADDR` malformed | **fatal at config load** — `ConfigError::InvalidValue` (`config.rs:501-508`) |
| `BUZZ_MESH_BIND_ADDR` absent | default `0.0.0.0:3478` (`config.rs:508`) — binds on **all interfaces**. §7 |

`main.rs:454-459` wires consumers, then publishes the handle, then logs. The
`OnceLock::set` failure is `unreachable!("mesh handle is set exactly once, right
here")` (`main.rs:459`) — a genuine single-writer invariant, not a swallowed error.

Ordering note: consumers are registered **before** `state.mesh.set(handle)`, so
inbound mesh traffic can be served before any local join on this pod can resolve
ownership. That is the correct order (a peer must not hit an unregistered slot), and
`mesh_boot.rs:47-55` documents the residual boot-window drop as bounded and safe
because of fencing.

---

#### 6. Hard-coded values that should be tunable

Grouped by operational consequence.

##### 6.1 Capacity / QoS — no env var exists

| Constant | Value | Line | Why it should be tunable |
|---|---|---|---|
| `MAX_PEERS_PER_ROOM` | 25 | `room.rs:50` | The doc itself calls it "the soft one" and shows the `N×(N−1)` math (`room.rs:47-50`). A deployment with fatter pods or smaller huddles has no way to move it. This is the single most likely value an operator would want to change |
| `AUDIO_CHANNEL_CAPACITY` | 8 (≈160 ms) | `room.rs:40` | Jitter tolerance vs. latency tradeoff; hardware- and network-dependent |
| `CTRL_CHANNEL_CAPACITY` | 32 | `room.rs:45` | Sized by a comment ("even 30 simultaneous join/leave events fit") that is tied to `MAX_PEERS_PER_ROOM`; if the peer cap moved, this would need to move with it — and neither can |
| `MAX_AUDIO_FRAME_BYTES` | 4096 | `handler.rs:44` | Caps Opus bitrate implicitly. A higher-fidelity or multi-channel configuration would need this raised |
| WS data / control queue depths | 16 / 8 | `handler.rs:659-660` | Same tradeoff, second layer |
| `roster_tx` broadcast capacity | 64 | `room.rs:179` | Determines how much roster churn a slow control stream can absorb before a full resync |

##### 6.2 Timing — no env var exists

| Constant | Value | Line | Consequence |
|---|---|---|---|
| `HEARTBEAT_INTERVAL` | 30 s | `handler.rs:55` | With `MAX_MISSED_PONGS=3` this fixes dead-peer detection at 60–90 s. Behind an aggressive LB idle timeout this is too slow; on a lossy mobile link, too aggressive |
| `MAX_MISSED_PONGS` | 3 | `handler.rs:58` | as above |
| `AUTH_TIMEOUT` | 5 s | `handler.rs:61` | Also the window during which an unauthenticated client holds a shared `conn_semaphore` permit (see security §6.1) — the one timing constant with a direct DoS consequence |
| `DEFAULT_LEASE_TTL` | 30 s | `directory.rs:17` | `with_lease_ttl` exists (`directory.rs:187-189`) and tests use it (`directory.rs:601`), but production calls `SessionDirectory::new` (`mesh_boot.rs:512`), so the TTL is fixed. TTL is the failover-latency dial for cross-pod huddles |
| `DEFAULT_HUDDLE_RENEW_INTERVAL` | 10 s | `join.rs:452` | Must stay ≪ TTL; coupled to a value that is also fixed |
| `DEFAULT_RENEW_INTERVAL` (reliable) | 10 s | `reliable.rs:34` | duplicate of the above, in a second file |
| `OWNER_READY_RETRY_INTERVAL` / `_MAX_ATTEMPTS` | 20 ms / 25 | `join.rs:387-388` | Bounds a join's worst-case latency at ~500 ms and its Redis round trips at 25 |
| `registry_refresh` | 15 s | `config.rs:511` | A `MeshConfig` field with no env var — see §2.1 |
| drain-watcher poll | 500 ms | `mesh_boot.rs:495` | SIGTERM→drain latency |
| demo-echo drain poll | 100 ms | `mesh_boot.rs:315` | demo only |
| `ECHO_TIMEOUT` | 10 s | `api/mesh_demo.rs:45` | demo only |

##### 6.3 Protocol / wire — arguably correct to hard-code

| Constant | Value | Line | Note |
|---|---|---|---|
| `CURRENT_PROTOCOL_VERSION` | 2 | `handler.rs:123-124` | Correct as a compile-time constant, but there is **no way to pin a deployment to v1** during a staged rollout even though the doc says "older versions stay supported indefinitely for staged rollouts" (`handler.rs:120-122`). A `BUZZ_HUDDLE_MAX_PROTOCOL_VERSION` would make that claim actionable |
| `default_protocol_version()` | 1 | `handler.rs:140-142` | Correct — backwards compatibility |
| `V2_HEADER_LEN` / `FLAG_DTX` | 8 / `0x01` | `wire.rs:29`, `:33` | Wire format; must not be tunable. Duplicated in `desktop/src-tauri/src/huddle/wire.rs:48` — two copies of one protocol constant |
| `MAX_RELIABLE_PAYLOAD_BYTES` | 1 MiB | `reliable.rs:31` | Justified against goose's 50 MiB bodies (`reliable.rs:26-30`) and asserted against the 16 MiB wire cap (`reliable.rs:945`) |
| `ReliableWireFrame::VERSION` | 1 | `reliable.rs:423` | wire format |
| `PROTO_VERSION` | `WIRE_VERSION as u16` | `mesh_boot.rs:367` | derived, correct |
| `capabilities()` | all three profiles, static | `mesh_boot.rs:369-377` | Correct — "All three tunnel profiles ship in the same binary". But it means a pod cannot advertise *not* serving huddles, so there is no capability-based way to keep huddle ownership off a given pod |

##### 6.4 Conformance — no configuration at all

| Item | Status |
|---|---|
| Tracer selection | **unconditional** `NoopTracer` at `state.rs:794-798`. No env var, no config field, no cargo feature. `crates/buzz-relay/Cargo.toml` `[features]` contains only `dev = ["buzz-auth/dev"]` |
| `JsonlTracer` output path | `create<P: AsRef<Path>>` (`tracers.rs:36`) takes a path — but nothing calls it, so no path is ever configured |
| Feature-gated elision | `tracers.rs:11-13` says "the build can have the compiler eliminate them entirely behind a feature flag if the cost ever shows up in benches". No such flag exists, so the emit arguments are constructed and discarded on every ingest and REQ |
| Seam names | `"ingest_event_exited_without_trace"` (`ingest.rs:1411`), `"row_community_lookup_missing"` (`conformance/mod.rs:251`) — string literals, correctly not configurable |

Making the tracer selectable (`BUZZ_CONFORMANCE_TRACE=<path>`) is the single change
that would turn this module from inert scaffolding into an operable diagnostic.

---

#### 7. Deployment-facing configuration notes

- **`0.0.0.0:3478` binds all interfaces by default.** `config.rs:508`. On a
  mesh-enabled deployment the mesh QUIC endpoint is reachable from anywhere the pod
  is. Admission is gated on the Redis ready registry
  (`buzz-relay-mesh/src/runtime.rs:275-283`), so an unattested dialer is rejected,
  but the port is open and the default is not a loopback or pod-network address.
  `3478` is the IANA STUN port — coincidental, and potentially confusing to a
  network operator reading a port scan.
- **`/_mesh/demo/echo` is on the public router** (`router.rs:123`), not the admin
  host, and is unauthenticated when both mesh flags are on. See security §5.
- **`/_mesh` is on the health router** (`router.rs:230`) which
  `router.rs:222-224` documents as having "No metrics middleware, no auth, no CORS,
  no body limit". It returns peer runtime ids and addresses.
- **`max_connections` is the only operator dial that affects huddle capacity**, and
  it is shared with every other relay WebSocket (`state.rs:727`, `handler.rs:90`).
  There is no way to reserve or cap the audio share of that budget.
- **Nothing in `.env.example` or `deploy/compose/.env.example`** was found to
  document `BUZZ_HUDDLE_AUDIO_AVAILABLE`, `BUZZ_MESH`, `BUZZ_MESH_BIND_ADDR`,
  `BUZZ_MESH_DEMO_ECHO`, `BUZZ_MESH_ADVERTISE_ADDR`, or `POD_IP` — operators
  discover them only from `config.rs` doc comments and `mesh_boot.rs`.
- **Redis is a hard dependency for mesh-enabled huddles**, not a cache: joins fail
  (`join_rejected`) when the directory is unreachable, and a renewal failure is
  treated as owner loss, closing every local owner client on that pod
  (`join.rs:521-529` → `handler.rs:756-765`).


## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Configuration

The crate reads **no environment variables itself** — verified: `std::env` appears
nowhere in `crates/buzz-relay-mesh/src`. All configuration is resolved by
`buzz-relay` and passed in as `MeshConfig` (`lib.rs:53-64`) plus two direct env
reads in `mesh_boot.rs`.

---

#### 1. Environment variables

Complete set (grep for `BUZZ_MESH` + `POD_IP` across the repo, excluding the
unrelated MeshLLM `BUZZ_MESH_API_PORT`/`BUZZ_MESH_CONSOLE_PORT`/
`BUZZ_MESH_IROH_RELAYS` desktop vars, `desktop/src-tauri/src/mesh_llm/mod.rs:37-47`):

| Variable | Default | Accepted values | Resolved at | Consumed at |
|---|---|---|---|---|
| `BUZZ_MESH` | **off** | `on` (case-insensitive), `true`, `1` → enabled; **absent, `off`, or any other value → disabled** | `config.rs:498-500` | `mesh_boot.rs:417` |
| `BUZZ_MESH_BIND_ADDR` | `0.0.0.0:3478` | any parseable `SocketAddr`; parse failure is a **fatal `ConfigError::InvalidValue`** | `config.rs:501-507` | `MeshEndpoint::bind`, `mesh_boot.rs:383`; logged `mesh_boot.rs:394` |
| `BUZZ_MESH_ADVERTISE_ADDR` | unset | free-form string, trimmed; empty → ignored. **Not validated as a `SocketAddr` at boot** — a typo is only discovered when a *peer* fails to parse it (`runtime.rs:328-334`) | read inline, `mesh_boot.rs:384-389` | published in `ReadyRecord.endpoint_addrs` / `GossipRecord.endpoint_addrs` |
| `POD_IP` | unset | IP string, trimmed; used only when non-empty **and** a bound port was resolved | read inline, `mesh_boot.rs:398-403` | same |
| `BUZZ_MESH_DEMO_ECHO` | **off** | `on`/`true`/`1` only (same strict parser) | `config.rs:516-518` | `main.rs:456` → `wire_consumers(demo_echo)`; route gate `api/mesh_demo.rs`; owner-side echo `mesh_boot.rs:290-292` |

Two more relay-level vars materially affect the mesh's exposure:

| Variable | Default | Relevance |
|---|---|---|
| `BUZZ_HEALTH_PORT` (`config.health_port`) | 8080 | the listener carrying `GET /_mesh` (`router.rs:230`), bound to **hard-coded `0.0.0.0`** at `main.rs:1119`, ignoring `BUZZ_BIND_ADDR` |
| `BUZZ_RELAY_PRIVATE_KEY` (→ `state.relay_keypair`) | — | the secp256k1 key that signs and anchors every ready-record attestation (`mesh_boot.rs:445`, `:450`) |

##### Parser semantics (verified)

```rust
// config.rs:498-500
let mesh_enabled = std::env::var("BUZZ_MESH")
    .map(|v| v.eq_ignore_ascii_case("on") || v == "true" || v == "1")
    .unwrap_or(false);
```

Note the asymmetry: `on` is case-insensitive, `true`/`1` are exact. `TRUE`, `On `
(trailing space), `yes`, `enabled` all evaluate to **off**. Deliberate — rationale at
`config.rs:494-497`: "an image upgrade with untouched env must not bind a new UDP
port or write a new Redis key… anything else (absent, `off`, other values) keeps
exact single-instance behavior." `BUZZ_MESH_DEMO_ECHO` uses the identical parser
(`config.rs:514-518`, comment "same strict pattern as BUZZ_MESH — explicit
`on`/`true`/`1` only, anything else (absent, `off`, typos) is off"). The cost of the
strictness is silent misconfiguration: `BUZZ_MESH=ON` yields a meshless relay with
no warning.

`BUZZ_MESH_BIND_ADDR` uses `unwrap_or_else(|_| Ok(default))?` (`config.rs:502-507`),
so a **missing** var falls back to the default while a **malformed** var is fatal —
the correct split.

---

#### 2. `MeshConfig` fields

`lib.rs:53-64`, `#[derive(Clone, Debug)]`, no `Default`, no `Deserialize`.
Constructed only at `config.rs:508-512`; stored as `Config.mesh` (`config.rs:136`).

| Field | Type | Value source | Default | Read at |
|---|---|---|---|---|
| `enabled` | `bool` | `BUZZ_MESH` | `false` | `mesh_boot.rs:417` |
| `bind_addr` | `std::net::SocketAddr` | `BUZZ_MESH_BIND_ADDR` | `0.0.0.0:3478` | `mesh_boot.rs:383`, `:396` |
| `registry_refresh` | `std::time::Duration` | **hardcoded** | `15s` (`config.rs:511`) | `mesh_boot.rs:447` → `ReadyRegistry::new` |

**`registry_refresh` has no environment variable.** `config.rs:511` is a literal
`std::time::Duration::from_secs(15)`. The crate exposes
`DEFAULT_REGISTRY_REFRESH = 15s` (`registry.rs:20`) for exactly this purpose and the
relay does not use it (grep: zero external callers). Operators cannot tune heartbeat
cadence, and therefore cannot tune the derived **45 s registry TTL**
(`REGISTRY_EXPIRY_MULTIPLIER = 3`, `registry.rs:21`; `expiry_for`, `:154-156`) — the
window in which a crashed pod remains dialable.

##### Documented-default delta

`lib.rs:55-56` documents `BUZZ_MESH` as "`on` (default when replicas can exist) |
`off` kill switch." The implementation defaults it **off**, unconditionally, with no
replica detection. `config.rs:131-135` and `mesh_boot.rs:4-6` state the correct
behaviour, and `mesh_boot.rs:544-555` tests it
(`mesh_defaults_off_when_env_absent`). The `lib.rs` doc comment is stale — it
describes the design as originally proposed, before the review blocker noted at
`mesh_boot.rs:544-546` ("Blocker fix (Wren review of 8b077fdb): absent `BUZZ_MESH`,
the mesh is OFF"). Also, "kill switch" implies default-on semantics that no longer
apply.

---

#### 3. Compile-time constants (the real tuning surface)

None of these are configurable at runtime; all require a rebuild.

| Constant | Value | Location | Overridable? |
|---|---|---|---|
| `DEFAULT_GOSSIP_INTERVAL` | 2 s | `runtime.rs:44` | only via `MeshRuntime::start_with_intervals` (`:102`) — **zero external callers**, tests only |
| `DEFAULT_RECONCILE_INTERVAL` | 5 s | `runtime.rs:42` | same |
| `CONTROL_QUEUE_DEPTH` | 64 | `runtime.rs:46` | no — private const |
| `DEFAULT_PHI_SUSPECT_THRESHOLD` | 8.0 | `membership.rs:17` | only via `with_phi_suspect_threshold` (`:66`) — **zero callers**; the relay never overrides it (`mesh_boot.rs:444-445`) |
| `PhiAccrual` sample window | 100 | `gossip.rs:176` (`Default::new(100)`) | only via `PhiAccrual::new` — zero external callers |
| `DEFAULT_REGISTRY_REFRESH` | 15 s | `registry.rs:20` | unused by the relay (hardcoded duplicate at `config.rs:511`) |
| `REGISTRY_EXPIRY_MULTIPLIER` | 3 | `registry.rs:21` | no |
| `READY_KEY_PREFIX` | `"mesh:ready:"` | `registry.rs:19` | no |
| `ATTESTATION_CONTEXT` | `"buzz-relay-mesh-ready-v1"` | `registry.rs:22` | no |
| `GOSSIP_PAYLOAD_VERSION` | 1 | `gossip.rs:13` | no |
| `ALPN` | `b"buzz/mesh/1"` | `wire.rs:37` | no |
| `WIRE_VERSION` | 1 | `wire.rs:42` | no |
| `MAX_STREAM_FRAME` | 16 MiB | `wire.rs:46` | no |
| `PROTO_VERSION` (relay) | `WIRE_VERSION as u16` = 1 | `mesh_boot.rs:367` | no |
| capabilities list | `["reliable-stream","realtime-media","huddle-control"]` | `mesh_boot.rs:371-377` | no — static, "all three tunnel profiles ship in the same binary" |
| drain-watcher poll | 500 ms | `mesh_boot.rs:495` | no |
| demo-echo drain tick | 100 ms | `mesh_boot.rs:319` | no |
| demo-echo `ECHO_TIMEOUT` | 10 s | `api/mesh_demo.rs:~41` | no |

Derived timings worth recording: with the shipped values a peer is marked
`Suspect` after `8 × ln(10) × 2s ≈ 36.8 s` of gossip silence (BR-MESH-34), its
registry record expires after 45 s, and a dead peer is re-dialed every 5 s
indefinitely with no backoff.

---

#### 4. Address advertisement resolution

`advertise_addrs(&MeshEndpoint)` (`mesh_boot.rs:382-403`), documented at
`mesh_boot.rs:379-381`. Strict precedence, first match wins:

1. **`BUZZ_MESH_ADVERTISE_ADDR`** (`:384-389`) — trimmed; empty string falls through.
   Returns a single-element list. Intended for "explicit, classic-LB shapes."
2. **`POD_IP` + actual bound port** (`:391-403`) — the port comes from
   `endpoint.ip_addrs().first().port()`, i.e. the *real* bound port, so
   `BUZZ_MESH_BIND_ADDR=0.0.0.0:0` works. Requires `bound_port != 0`. Intended for
   k8s Downward API with "zero RBAC."
3. **All endpoint IP transport addrs** (`:403`) — `endpoint.ip_addrs()`
   (`endpoint.rs:61-71`, filters `TransportAddr::Ip`, drops relay paths). Dev/local
   fallback.

Consumed identically by both the Redis record (`ReadyRecord::new(…, addrs, …)`,
`mesh_boot.rs:448-453`) and the gossip record (`GossipRecord::new(runtime_id, addrs,
PROTO_VERSION)`, `mesh_boot.rs:439`).

Gaps:

- `BUZZ_MESH_ADVERTISE_ADDR` is **never parsed at boot**. It is stored as a `String`
  (`registry.rs:105-107` explains the string choice as layer independence) and only
  parsed by the *dialing peer* (`runtime.rs:329-335`), where a bad value produces
  `warn!("mesh: bad peer addr")` on every remote pod every 5 s. A typo is silently
  self-isolating.
- Path 3 can advertise loopback or a container-internal address in environments
  where neither env var is set, producing a mesh that forms only within one host.
- The advertised list is captured **once at boot** (`mesh_boot.rs:385`) and republished
  verbatim by the heartbeat (`ReadyRecord` is moved into `spawn_registry_heartbeat`,
  `mesh_boot.rs:467-471`). If the pod's IP changes, the record never updates.

---

#### 5. Deployment configuration state

- **No deployment in this repo enables the mesh.** grep for `BUZZ_MESH` across
  `.env.example`, `deploy/`, and the helm charts returns nothing. There is likewise
  no `3478` port declaration and no `POD_IP` Downward-API stanza anywhere in the repo.
  The k8s wiring the code anticipates (`lib.rs:59-61` "Excluded from istio sidecar
  capture in k8s"; `mesh_boot.rs:380` Downward API) lives in the external
  `squareup/block-coder-tf-stacks` repo, if at all.
- **`BUZZ_HUDDLE_AUDIO_AVAILABLE`** is the current answer to horizontal scaling:
  `config.rs:120-129` instructs operators running multiple relay pods to set it
  `false` "until the out-of-relay media/SFU service lands." That is the shipped
  multi-pod story; the mesh is the not-yet-enabled replacement.
- `BUZZ_MESH_BIND_ADDR` defaults to **`0.0.0.0`** — all interfaces. In any
  environment where the mesh is enabled without network policy, the QUIC endpoint is
  publicly reachable (see `-security.md` F-04, F-06).

---

#### 6. Configuration-related test coverage

| Test | Asserts | Location |
|---|---|---|
| `mesh_off_boots_nothing` | `boot_mesh` returns `None` and never touches Redis (pool points at unroutable `redis://127.0.0.1:1`) | `mesh_boot.rs:526-541` |
| `mesh_defaults_off_when_env_absent` | absent `BUZZ_MESH` ⇒ `config.mesh.enabled == false`; self-skips if the var is externally forced | `mesh_boot.rs:544-555` |
| `expiry_is_three_refreshes` | `expiry_for(15s) == 45s` | `registry.rs:324-327` |
| `ready_key_is_stable_and_namespaced` | `ready_key(rid(0xAB)) == "mesh:ready:" + "ab"*32` | `registry.rs:317-322` |

Untested: `BUZZ_MESH_BIND_ADDR` parse-failure fatality, `advertise_addrs`'
three-way precedence (all three branches), `BUZZ_MESH_DEMO_ECHO` gating, and the
`registry_refresh` value flowing through to the Redis TTL.

---

#### 7. Summary of configuration findings

1. `registry_refresh` (and hence the 45 s TTL) is unreachable from the environment
   despite `MeshConfig` exposing the field and `registry.rs:20` providing the default
   constant.
2. Gossip interval, reconcile interval, phi threshold, and phi sample window are all
   API-overridable but **have zero callers** — effectively compile-time constants.
3. `lib.rs:55-56` documents the wrong default for `BUZZ_MESH` (says on, is off).
4. `BUZZ_MESH_ADVERTISE_ADDR` accepts an unvalidated string; failures surface on
   remote pods, not at boot.
5. The strict-opt-in parsers silently ignore near-miss values (`ON`, `yes`) with no
   warning.
6. `/_mesh` rides a listener whose bind address is hard-coded `0.0.0.0`
   (`main.rs:1119`) and cannot be restricted by `BUZZ_BIND_ADDR`.


## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: Configuration

This aspect is genuinely thin, and the thinness is the finding: **a subsystem whose entire
purpose is switchable observation has no switch.** There is no environment variable, no config
struct field, no CLI flag, and no Cargo feature that selects a tracer. The only runtime
selector is a hard-coded constructor line.

---

### Cargo features

**None.** `crates/buzz-conformance/Cargo.toml` (35 lines) has `[package]`
(`:1-8`), `[dependencies]` (`:25-29`), and `[dev-dependencies]` (`:33-34`). There is no
`[features]` table — verified by grep. No `#[cfg(feature = ...)]` appears anywhere in `src/`.

Two comments promise a feature that does not exist:

| Claim | Location |
|---|---|
| "Zero cost: the build can omit emission entirely behind a feature" | `src/lib.rs:321-322` |
| "the build can have the compiler eliminate them entirely behind a feature flag if the cost ever shows up in benches" | `crates/buzz-relay/src/conformance/tracers.rs:9-13` |

`buzz-relay`'s dependency is unconditional: `buzz-conformance = { workspace = true }`
(`crates/buzz-relay/Cargo.toml:20`) with no `optional = true` and no feature gate, so the crate
compiles into every relay build.

---

### Environment variables

| Variable | Scope | Read at | Effect |
|---|---|---|---|
| `BUZZ_CONFORMANCE_UPDATE` | test-only | `tests/replay_fixtures.rs:210` (`std::env::var(...).is_ok()`) | when set to anything, `assert_fixture_matches` writes the golden JSONL instead of comparing (`:211-214`) |

That is the complete list. Grep for `CONFORMANCE`, `TRACER`, and `BUZZ_TRACE` across
`.env.example` and `crates/buzz-relay/src/config.rs` returns nothing. `.env.example` has no
tracing/conformance section.

Note the variable is presence-checked, not value-parsed — `BUZZ_CONFORMANCE_UPDATE=0`
regenerates the fixtures just as `=1` does.

---

### How a non-`Noop` tracer would be selected

There is no selection mechanism. The single binding site is:

```
tracer: Arc::new(crate::conformance::NoopTracer),
```
`crates/buzz-relay/src/state.rs:798`, inside `AppState`'s constructor. The field is
`pub tracer: Arc<dyn buzz_conformance::Tracer>` (`:620`), so it is publicly readable and
writable — but grep across `crates/buzz-relay/src/` finds exactly one assignment (`:798`) and
four reads (`handlers/ingest.rs:1383`, `handlers/req.rs:145`, `:356`, `:672`).

Switching to `JsonlTracer` therefore requires one of:

1. Mutating `AppState.tracer` after construction — the path the constructor comment envisions:
   "Conformance tests overwrite this with a JsonlTracer after construction (see test helpers in
   `crates/buzz-test-client` once those land)" (`state.rs:794-797`). Those helpers do not exist;
   `crates/buzz-test-client/tests/conformance_multitenant.rs` never references
   `buzz_conformance` (verified by grep).
2. Editing `state.rs:798` and rebuilding.

`JsonlTracer::create` (`crates/buzz-relay/src/conformance/tracers.rs:37-45`) takes the output
path as a plain `P: AsRef<Path>` argument — no default, no env fallback, no directory
convention. It is never called anywhere in the repo.

---

### Constants that behave like configuration

| Constant | Value | Line | Notes |
|---|---|---|---|
| `SCHEMA_VERSION` | `1` | `src/lib.rs:86` | compared at `src/checker.rs:85`, `:97`; stamped by `TraceStep::new` (`:305`) |
| `ProptestConfig::with_cases` | `128` | `tests/proptest_checker.rs:193` | per-property case count; not overridable via env in this file |
| `POOL` | `3` | `tests/proptest_checker.rs:51` | community/channel pool width, chosen so foreign-vs-resolved collisions are frequent (`:44-49`) |
| `EmitGuard` seam name | `"ingest_event_exited_without_trace"` | `crates/buzz-relay/src/handlers/ingest.rs:1411` | `&'static str`, passed as `kind` |
| projection breach tag | `"row_community_lookup_missing"` | `crates/buzz-relay/src/conformance/mod.rs:250` | `&'static str` |

`PROPTEST_CASES` and the other standard proptest env overrides are honoured by the proptest
library itself, but the crate ships no `proptest-regressions/` directory and no
`proptest.toml` — verified by `ls crates/buzz-conformance` (only `Cargo.toml`, `LIMITS.md`,
`TRACE_SCHEMA.md`, `src/`, `tests/`).

---

### Test-invocation configuration

| Entry point | Command | Line |
|---|---|---|
| `just test-unit` (nextest present) | `cargo nextest run -p buzz-conformance` | `justfile:290` |
| `just test-unit` (fallback) → `scripts/run-tests.sh unit` | `cargo test -p buzz-conformance -- --nocapture` | `scripts/run-tests.sh:98-99` |
| `just ci` | `check test-unit desktop-test …` | `justfile:266` |

Both run all targets (lib + `proptest_checker` + `replay_fixtures`); the justfile comment at
`:286-289` states the intent ("Run all targets (lib + the tests/replay_fixtures.rs integration
test), not just --lib"). No `--ignored` tests exist in this crate, so nothing is gated behind a
second invocation.

`crates/buzz-conformance/LIMITS.md:88-118` documents a three-command CI contract
(`cargo test -p buzz-conformance --lib`, `--test replay_fixtures`, and
`cargo test -p buzz-relay --lib conformance::`) and totals it as "9 + 5 + 2 = 16 tests". The
actual counts are 9 lib + 6 `replay_fixtures` + 7 `proptest_checker` = 22 in this crate, plus 9
in `crates/buzz-relay/src/conformance/mod.rs` and 1 in `handlers/ingest.rs`. The proptest lane
is absent from that doc's command list entirely, and no justfile recipe or GitHub workflow runs
the `-p buzz-relay --lib conformance::` command it calls mandatory (grep `conformance` in
`.github/workflows/` returns nothing).

