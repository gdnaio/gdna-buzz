## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Conventions

---

#### 1. Crate-level lint policy

| Attribute | Line | Effect |
|-----------|------|--------|
| `#![deny(unsafe_code)]` | `lib.rs:1` | zero `unsafe` blocks in the crate; verified — the only textual `unsafe` in this group is a comment (`main.rs:1013`) |
| `#![warn(missing_docs)]` | `lib.rs:2` | warn, not deny — an undocumented public item compiles |
| crate `//!` doc | `lib.rs:3` | placed after the inner attributes (valid) |

Local `#[allow]` count in this group: **6**, all narrowly scoped, no blanket allows.

| Allow | Line | Justification present? |
|-------|------|------------------------|
| `clippy::too_many_arguments` on `ConnectionManager::register` | `state.rs:204` | yes — `state.rs:202-203` explains a params struct would only relocate the fields |
| `clippy::type_complexity` on `membership_cache` | `state.rs:544` | no |
| `clippy::type_complexity` on `accessible_channels_cache` | `state.rs:548` | no |
| `clippy::type_complexity` on `observer_owner_cache` | `state.rs:606` | no |
| `clippy::too_many_arguments` on `AppState::new` (10 params) | `state.rs:636` | no |
| `clippy::type_complexity` on the NIP-11 fence const | `nip11.rs:328` | implicit (the whole const is a deliberate signature pin) |

#### 2. Module layout convention

`lib.rs` declares 21 modules. Every `pub mod` carries a one-line `///` doc **except** `storage_sweep` (`lib.rs:43`) — the only inconsistency. `storage_sweep.rs` does have `//!` inner docs (`storage_sweep.rs:1-5`) so `missing_docs` is satisfied, but the `lib.rs` listing reads as an oversight.

Exactly **one** private module: `mod admission;` (`lib.rs:5`). It is also the only file in the group with **no module-level doc comment at all** — `admission.rs:1` is a `use` statement. Every other file starts with `//!`:

| File | Module doc | Line |
|------|-----------|------|
| `state.rs` | "Shared application state — Arc-wrapped, shared across all connections." | `:1` |
| `config.rs` | "Relay configuration from environment variables." | `:1` |
| `router.rs` | "axum routers — app (WebSocket + REST), health (K8s probes), metrics (Prometheus)." | `:1` |
| `nip11.rs` | "NIP-11 relay information document." | `:1` |
| `protocol.rs` | "NIP-01 client/relay message parsing and formatting." | `:1` |
| `tenant.rs` | 15-line conformance narrative ("row zero") | `:1-15` |
| `telemetry.rs` | ASCII architecture diagram + honoured env vars | `:1-32` |
| `metrics.rs` | ASCII architecture diagram | `:1-19` |
| `error.rs` | "Error types for the relay crate." | `:1` |
| `admission.rs` | **none** | — |

**ASCII-diagram convention**: three files open with a boxed ASCII diagram of their data flow — `metrics.rs:3-12`, `telemetry.rs:3-15`, and the `serve` fn doc (`main.rs:1097-1112`). This is a distinctive house style in the relay core.

#### 3. Error-handling convention — three coexisting styles

| Layer | Style | Cite |
|-------|-------|------|
| `main()` | `anyhow::Result` with `.map_err(|e| anyhow!("…: {e}"))` and an `error!` log immediately before returning | `main.rs:83`, `main.rs:122-127`, `main.rs:151-155` |
| `config.rs` | `thiserror` enum `ConfigError` with 2 variants | `config.rs:19-27` |
| `protocol.rs` | `thiserror` enum `RelayError` + `type Result<T>` alias | `error.rs:8-49`, used only at `protocol.rs:6` |
| handlers (outside group) | `axum::response::Response` / `IntoResponse` tuples, no shared error type | `router.rs:295-301`, `router.rs:304-375` |

`RelayError` is the crate's nominal error type but is effectively single-purpose: only `InvalidMessage` is constructed, only `protocol.rs` imports it (see debt). There is no `impl IntoResponse for RelayError`, so HTTP handlers cannot use it — that is why the third style exists.

Fallibility conventions actually followed:
- Fatal-at-boot: `.map_err(anyhow!)?` (11 sites in `main.rs`).
- Non-fatal-at-boot: `match … { Err(e) => error!(…) }` with an explicit "(non-fatal…)" phrase in the message — a consistent, greppable convention (`main.rs:174`, `:190-195`, `:253`, `:286`, `:305`, `:319`).
- Conditional fatality is always written as `if config.require_relay_membership { … return Err(…) } else { error!(…) }` (`main.rs:250-262`, `:275-288`, `:295-309`).

#### 4. `unwrap`/`expect`/panic conventions

**26 production `unwrap()`/`expect()` calls** in this file group (outside `#[cfg(test)]`), plus 1 `panic!`, 1 `unreachable!`, 1 `debug_assert!`, 1 `std::process::exit`.

| File | Count | Lines |
|------|-------|-------|
| `metrics.rs` | 17 | `:79, 84, 89, 94, 99, 104, 109, 114, 119, 124, 129, 134, 136, 141, 143, 145` (`expect`) + `:180` (`unwrap`) |
| `main.rs` | 4 | `:90, 401, 1230, 1253` |
| `state.rs` | 3 | `:446, 701, 708` |
| `config.rs` | 1 | `:507` |
| `protocol.rs` | 1 | `:189` |
| `router.rs`, `nip11.rs`, `tenant.rs`, `telemetry.rs`, `admission.rs`, `lib.rs`, `error.rs` | 0 | — |

Convention: `expect` messages are written as **assertions about an invariant**, not as error text — `"hardcoded dev key is valid"` (`main.rs:401`), `"relay fan-out frames are serialized UTF-8 JSON"` (`state.rs:446`), `"media storage was already constructed with this S3 config"` (`state.rs:701`), `"metrics exporter must build exactly once"` (`metrics.rs:143`), `"SAFETY: nostr::Event serialization is infallible for well-formed events"` (`protocol.rs:189`). Two carry an inline `// safe:` comment (`metrics.rs:180`, `config.rs:507`).

This contradicts AGENTS.md's "Do not introduce new `unwrap()` or `expect()` in production paths". The 17 in `metrics.rs` are all boot-time bucket registration where the argument is a compile-time literal array, so they are provably-infallible; still, they are production `expect`s.

Other panic-shaped exits:
- `panic!` at `main.rs:409` when `BUZZ_REQUIRE_AUTH_TOKEN=true` and no relay key — a deliberate hard stop, but inconsistent with the surrounding `return Err(anyhow!(…))` style used for the two neighbouring preconditions (`main.rs:206-211`, `:216-219`).
- `unreachable!("mesh handle is set exactly once, right here")` at `main.rs:460`.
- `debug_assert!` at `nip11.rs:143` — release builds skip the NIP-43/`self` consistency check.
- `std::process::exit(1)` at `main.rs:1153` — the only non-`main`-return process exit.

#### 5. Logging / tracing conventions

| Convention | Cite |
|-----------|------|
| JSON-only structured logs, `flatten_event(true)` | `main.rs:109` |
| Filter is `RUST_LOG` **plus** a forced `buzz_relay=info` directive | `main.rs:111` |
| OTLP layer attached only when the endpoint env var is set | `main.rs:101-107` |
| Structured fields preferred over interpolation: `info!(bind_addr = %…, relay_url = %…, …, "Config loaded")` | `main.rs:128-136` |
| `%` for `Display`, `?` for `Debug` — used consistently (`error = %e`, `error = ?e`) | `main.rs:119`, `main.rs:558`, `state.rs:886` |
| Sentence-case, no trailing period, human-readable message last | `main.rs:157`, `main.rs:197`, `state.rs:1185` |
| Startup progress logged as a linear narrative ("Postgres connected", "Redis pub/sub connected", "Media storage connected", "…listener started") | `main.rs:157-159`, `:348`, `:421`, `:1117`, `:1155` |
| Two logging import styles coexist: `use tracing::{error, info, warn}` at `main.rs:5` **and** fully-qualified `tracing::warn!`/`tracing::info!`/`tracing::error!` at `main.rs:404`, `:483`, `:513`, `:837`, `:1152` and throughout `state.rs` | mixed |

`state.rs` never imports the tracing macros; it always fully-qualifies (`state.rs:463`, `:472`, `:886`, `:1185`, …). `main.rs` does both. No lint enforces either.

#### 6. Test conventions

**Test counts in this file group** (`#[test]` + `#[tokio::test]`):

| File | Tests | `#[ignore]` | Notes |
|------|-------|-------------|-------|
| `config.rs` | 22 | 0 | serialized by a module-level `ENV_MUTEX` (`config.rs:931-934`) with an explicit rationale about flakiness |
| `state.rs` | 18 | 0 | 5 need a live Postgres/Redis via `test_state()` (`state.rs:1257-1290`) |
| `nip11.rs` | 15 | 0 | deliberately test the `SUPPORTED_NIPS` **constant** rather than `Config::from_env()` to avoid the env race (`nip11.rs:358-360`) |
| `tenant.rs` | 10 | 0 | includes a `redteam_attack2` sub-module (`tenant.rs:249-332`) |
| `protocol.rs` | 7 | 0 | table-driven with `Box<dyn Fn>` case tables and `type ParseCase`/`FormatCase` aliases to dodge `clippy::type_complexity` (`protocol.rs:230-232`, `:378-380`) |
| `main.rs` | 7 | 0 | includes `#[tokio::test(start_paused = true)]` for the cancellation loop (`main.rs:1827`) |
| `telemetry.rs` | 5 | 0 | `ENV_LOCK` mutex (`telemetry.rs:119`) |
| `router.rs` | 4 | 0 | one spins a real TCP + tungstenite client (`router.rs:463-501`) |
| `admission.rs` | 4 | 0 | stub `RateLimiter` impl (`admission.rs:64-95`) |
| `metrics.rs` | 0 | 0 | **no tests at all** |
| `lib.rs`, `error.rs` | 0 | 0 | no logic |
| **Total** | **92** | **0** | |

Conventions observed:
- **Env-mutation serialization**: two independent static mutexes, `config.rs:934` (`ENV_MUTEX`) and `telemetry.rs:119` (`ENV_LOCK`), both with a comment explaining process-global env races. `nip11.rs` avoids the problem structurally instead.
- **Save/restore around env mutation**: `let previous = std::env::var_os(…); … if let Some(v) = previous { set_var } else { remove_var }` — used at `config.rs:990-1000`, `:1015-1030`, `:1043-1055`, `:1058-1070`, `:1076-1086`, `:1192-1210`. Not used in `config.rs:1092-1105` or `:1109-1113` (rate-limit tests just `remove_var` afterwards) — inconsistent.
- **Red-team gate tests**: `tenant.rs:249-332` names its module `redteam_attack2`, documents the RED-then-GREEN cycle, and includes a *negative control* (`tenant.rs:326-331`). Notable convention.
- **Named-case tables**: `protocol.rs:234-273` and `:382-448` use `&[(name, Box<dyn Fn>)]` slices.
- **Behaviour-named test fns**: e.g. `disconnect_pubkey_is_fenced_to_the_banning_community` (`state.rs:1737`), `register_after_drain_self_signals_restart_close_and_cancel` (`state.rs:1881`), `relay_websocket_parser_rejects_oversized_messages_before_handler_reads_them` (`router.rs:504`). Long, assertion-shaped names are the norm.
- **Assertion messages**: nearly every `assert!`/`assert_eq!` carries an explanatory third argument (`state.rs:1454`, `:1770`, `:1795`, `nip11.rs:432`, `tenant.rs:320`).
- **`#[should_panic(expected = …)]`** used once, to pin the `debug_assert` (`nip11.rs:513-517`).
- **Zero `#[ignore]`d tests** in the group. However `tenant.rs:225-236` and `:238-243` still instruct the reader to "Delete this `#[ignore]` when the fix lands" — the attributes were already removed and the fix landed (`tenant.rs:81-88`). Stale instructions (see debt).
- Integration-style tests that need infrastructure are inlined into `#[cfg(test)]` modules rather than `tests/`: `crates/buzz-relay/tests/` **does not exist**.

#### 7. Naming conventions

| Pattern | Examples |
|---------|----------|
| Env vars: `BUZZ_` prefix, except deliberate exceptions | `RELAY_URL`, `RELAY_OWNER_PUBKEY`, `RELAY_OPERATOR_PUBKEYS`, `RELAY_OPERATOR_API_ORIGIN` — rationale given at `config.rs:524-525` and `:548-551` ("relay-identity config that may be shared across multiple services"); plus `DATABASE_URL`/`READ_DATABASE_URL`/`REDIS_URL`/`AWS_REGION`/`RUST_LOG`/`OTEL_*` |
| Env vars: **`SPROUT_` legacy prefix survives in 3 vars** | `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS` (`main.rs:701`), `SPROUT_REMINDER_SCHEDULER_BATCH_LIMIT` (`main.rs:705`), `SPROUT_MAX_NOT_BEFORE_DELTA` (`nip11.rs:97`) — the repo has otherwise renamed to `buzz`/`BUZZ_` |
| Metrics: `buzz_` prefix, `_total` counters, `_seconds` histograms | `state.rs:466`, `:1201`, `:1205`, `main.rs:836` |
| Framework metrics keep the CAKE names without the `buzz_` prefix | `http_requests_total`, `http_request_latency_ms` (`metrics.rs:204-205`) |
| Cross-pod method pairs: public `X` publishes, `pub(crate) X_local` applies a received drop (so a received drop is never re-published) | `state.rs:850/862`, `:876/881`, `:899/906`, `:921/928` |
| Cache accessors: `*_cached` | `is_member_cached` (`state.rs:827`), `get_accessible_channel_ids_cached` (`state.rs:1089`), `channel_visibility_cached` (`state.rs:1124`) |
| Cluster-wide operations: `*_clusterwide` | `disconnect_pubkey_clusterwide` (`state.rs:1018`), `disconnect_community_clusterwide` (`state.rs:1056`) |
| Health/internal routes prefixed `_` | `/_liveness`, `/_readiness`, `/_status`, `/_mesh`, `/_mesh/demo/echo` — and `metrics.rs:170` skips `/_*` on that basis |

#### 8. Doc-comment conventions

- Long "why" narratives on invariant-bearing items are the norm, not the exception: `state.rs:296-308` (tenant fence on ban), `state.rs:336-350` (drain rationale), `state.rs:530-540` (community-keyed dedup), `state.rs:1106-1122` (fail-safe visibility caching), `config.rs:114-128` (huddle single-pod constraint), `nip11.rs:307-327` (static-input fence), `tenant.rs:1-15` (row zero), `main.rs:36-48` (metric-cardinality cost lever).
- Spec/plan cross-references are embedded in doc comments: `COMMUNITY_MODERATION_PLAN.md §0 decision 4` (`state.rs:301`), `docs/spec/MultiTenantRelay.tla` (`state.rs:616-619`, `lib.rs:14-18`), `docs/git-on-object-storage.md` (`state.rs:563`), `PLANS/S3_STORAGE_METRICS_PLAN.md` (`storage_sweep.rs:4`), `E1 §4.8` (`state.rs:1116`, `state.rs:1155`), `spec §Push step 7, Inv_NoFork` (`state.rs:519-520`), `plan §4 fork B` / `§5b` (`config.rs:118-122`).
- Invariant names are used as identifiers in prose: `Inv_RowZero` (`tenant.rs:70`, `:251`), `Inv_LabelPropagation` (`state.rs:1159`), `Inv_NoFork` (`state.rs:520`), `A3` (`main.rs:467`), `C4`/`K1` (`main.rs:1486`, `:1506`).
- **Compile-time doc enforcement**: `nip11.rs:329-335` turns a documented conformance obligation into a type-level fence. This is the only instance of the pattern in the group and is explicitly described as "the same way a deny-lint would" (`nip11.rs:322-324`).

#### 9. Config-parsing conventions (inconsistent — see business-rules BR-RC-109)

Three helper functions exist (`positive_u64_from_env` `config.rs:270`, `parse_bool` `config.rs:363`, `parse_optional_bool` `config.rs:379`) but most fields bypass them with an inline `std::env::var(...).ok().and_then(|v| v.parse().ok()).unwrap_or(default)` chain. Result: **eight distinct boolean grammars** and two distinct numeric-invalid policies (silent default vs hard error) inside one file. `parse_optional_bool` (`config.rs:379-381`) is a one-line wrapper that just calls `parse_bool(name, false)` and has a single caller (`config.rs:792`).

Numeric-invalid policy split:
- Hard error: `BUZZ_RATE_LIMIT_*` (`config.rs:270-282`), `BUZZ_PUSH_GATEWAY_TIMEOUT_MS` (`config.rs:759-773`).
- Silent default: `BUZZ_REDIS_POOL_SIZE`, `BUZZ_MAX_CONNECTIONS`, `BUZZ_MAX_CONCURRENT_HANDLERS`, `BUZZ_SEND_BUFFER`, `BUZZ_MAX_FRAME_BYTES`, `BUZZ_SLOW_CLIENT_GRACE_LIMIT`, `BUZZ_HEALTH_PORT`, `BUZZ_METRICS_PORT`, all `BUZZ_GIT_*` numerics, all `BUZZ_MAX_*_BYTES`, all `BUZZ_MEDIA_*` numerics, and every interval var read directly in `main.rs`. `redis_pool_size` has an explicit test asserting the silent-fallback behaviour (`config.rs:988-1011`).
