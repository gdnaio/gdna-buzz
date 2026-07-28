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
