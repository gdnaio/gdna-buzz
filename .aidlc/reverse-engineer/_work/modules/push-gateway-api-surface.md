## Module: buzz-push-gateway (`crates/buzz-push-gateway`)
### Aspect: API Surface

#### Binary entry point (`main.rs`)

`buzz-push-gateway` is a single Tokio binary (`Cargo.toml:14-16`: `[[bin]] name = "buzz-push-gateway" path = "src/main.rs"`). `main()` (main.rs:21) branches on argv:

| Invocation | Behavior | Citation |
|---|---|---|
| `buzz-push-gateway --migrate-only` | Connects to `DATABASE_URL` with a 1-connection pool, reads `BUZZ_PUSH_RUNTIME_DATABASE_ROLE`, calls `PostgresAuthorityStore::apply_migrations_and_grants(&pool, &runtime_role)`, then returns `Ok(())` — never binds a listener | main.rs:26-35 |
| `buzz-push-gateway` (no args) | Loads `Config::from_env()`, installs metrics, builds the APNs transport, both AEAD keyrings, and a 20-connection Postgres pool, runs one `reap_expired` pass, spawns a background reaper (every 300s) and the two HTTP servers, then blocks on `shutdown_signal()` | main.rs:36-131 |

There are no other subcommands or CLI flags. Any `argv[1]` other than the exact string `--migrate-only` falls through to normal startup (main.rs:26 `if std::env::args().nth(1).as_deref() == Some("--migrate-only")`) — an unrecognized flag (e.g. `--help`, a typo) is silently ignored rather than rejected, and the process proceeds to load `Config::from_env()` as if no argument had been given.

`shutdown_signal()` (main.rs:132, private `async fn`, not `pub`) awaits `SIGTERM` on Unix (`tokio::signal::unix::signal(SignalKind::terminate())`) or `Ctrl-C` otherwise. On receipt, `main` flips `accepting` to `false` (main.rs:124), signals both servers to stop accepting via `tokio::sync::watch` channels (main.rs:125-129), waits up to 30s for the public server to drain (`tokio::time::timeout(Duration::from_secs(30), p)`, main.rs:127), then stops the health server and aborts the reaper task (main.rs:129-130).

#### HTTP routers (`http.rs`)

`router_with_metrics` (http.rs:734, `pub fn`) builds and returns **two** independent `axum::Router`s as a tuple `(public, health)` — they are never merged into one listener; `main.rs:91-131` binds each to its own `TcpListener`. `router()` (http.rs:727, `pub fn`) is a thin wrapper: `router_with_metrics(state, None)`.

Public router (bound to `BUZZ_PUSH_BIND_ADDR`, default `0.0.0.0:8080`, main.rs:107) — all seven routes are `POST` and share three layers applied once at `http.rs:747-752`: `RequestBodyLimitLayer::new(MAX_REQUEST_BYTES)` (8192 bytes), `ConcurrencyLimitLayer::new(256)`, and a 20-second `TimeoutLayer` returning `408 REQUEST_TIMEOUT`:

| Method | Path | Handler | Request type | Success | Citation |
|---|---|---|---|---|---|
| POST | `/v1/installations/challenges` | `challenge` | `InstallationChallengeRequest` | `200 InstallationChallengeResponse` | http.rs:110, 739 |
| POST | `/v1/installations` | `enroll` | `InstallationEnrollRequest` | `201 InstallationEnrollResponse` | http.rs:156, 740 |
| POST | `/v1/delegations` | `delegate` | `DelegationRequest` | `201 DelegationResponse` | http.rs:295, 741 |
| POST | `/v1/delegations/revoke` | `revoke_delegation` | `RevokeDelegationRequest` | `200 MutationResponse{status:"revoked"}` | http.rs:457, 742 |
| POST | `/v1/installations/endpoint` | `rotate_endpoint` | `RotateEndpointRequest` | `200 MutationResponse{status:"rotated"}` | http.rs:382, 743 |
| POST | `/v1/installations/revoke` | `revoke_installation` | `RevokeInstallationRequest` | `200 MutationResponse{status:"revoked"}` | http.rs:506, 744 |
| POST | `/v1/deliveries/apns` | `deliver` | `DeliveryRequest` + NIP-98 `Authorization` header | `200`/`410`/`503`/`400` `DeliveryResponse` or `ErrorBody` | http.rs:553, 745 |

Health/private router (bound to `BUZZ_PUSH_HEALTH_ADDR`, default `0.0.0.0:8081`, main.rs:108) — no body-limit/timeout/concurrency layers applied to this router at all:

| Method | Path | Handler | Response | Citation |
|---|---|---|---|---|
| GET | `/_liveness` | `live` | `200 {"status":"alive"}` | http.rs:713, 754 |
| GET | `/_readiness` | `ready` | `200 {"status":"ready"}` / `503 {"error":"not_ready"}` | http.rs:716-726, 755 |
| GET | `/metrics` | inline closure | Prometheus text (`text/plain; version=0.0.4`) — only present when `metrics_handle` is `Some` | http.rs:757-772 |

`/metrics` is conditionally registered: `router_with_metrics` only adds the route when its `metrics_handle: Option<PrometheusHandle>` parameter is `Some` (http.rs:756-772). `main.rs:91` always passes `Some(metrics_handle)`, so in the shipped binary `/metrics` is always present on the health listener. `router()` (http.rs:727-729) passes `None`, so a caller building the router through the public `router()` function (rather than `router_with_metrics` directly) gets no `/metrics` route at all — a library caller who reaches for the more obviously-named `router()` function silently loses metrics with no compiler warning or runtime log.

#### Error response shapes

Every handler funnels non-2xx responses through the private helper `error(status, code)` (http.rs:52-54), which returns `Json(ErrorBody { error: code })`. The private helper `authority_error(AuthorityError)` (http.rs:83-90) maps the two `AuthorityError` variants uniformly to `404 not_authorized` / `503 temporarily_unavailable` across every handler that touches the authority store — no handler distinguishes "not found" from "malformed" at the `AuthorityStore` boundary; that distinction is made earlier, inside each handler, via `400 invalid_request` before the authority store is ever called.

Observed error codes across all handlers (grep of `error(StatusCode::` call sites in `http.rs`): `invalid_request` (400), `invalid_attestation` (401), `not_authorized` (404, via `authority_error`), `invalid_auth` (401, `deliver` only), `invalid_grant` (404, `deliver` only), `temporarily_unavailable` (503), `configuration_fault` (503, `deliver` only). `docs/nips/NIP-PL.md` documents a closed error-body list — `invalid_request`, `invalid_attestation`, `not_authorized`, `invalid_auth`, `invalid_grant`, `temporarily_unavailable`, `configuration_fault`, `not_ready` — matching this code exactly, with `not_ready` emitted only by the health-router's `ready` handler (http.rs:716-726), not by any public-router handler. I found no discrepancy between the code's error codes and the NIP-PL doc's closed list.

#### `AppState` (public struct, http.rs:36-50)

```
pub struct AppState {
    pub grant_keyring: Arc<GrantKeyring>,
    pub app_attest: Arc<AppAttestVerifier>,
    pub authority: Arc<dyn AuthorityStore>,
    pub token_keyring: Arc<TokenKeyring>,
    pub transport: Arc<dyn PushTransport>,
    pub delivery_url: url::Url,
    pub max_grant_lifetime_seconds: i64,
    pub max_installation_lifetime_seconds: i64,
    pub endpoint_quota_window_seconds: i64,
    pub endpoint_quota_max_deliveries: i64,
    pub enabled_profiles: HashSet<AppProfile>,
    pub now: fn() -> i64,
    pub accepting: Arc<AtomicBool>,
}
```
Every field is `pub` and none carry a doc comment (http.rs:36-50) — a caller embedding this crate as a library has no in-source documentation of what any field means beyond its name. The meaning of `now` (an injectable clock, always `chrono::Utc::now().timestamp` in production per its literal closure at main.rs:97) and `accepting` (a shutdown-draining flag, constructed at main.rs:99, flipped at main.rs:124) is inferable only by reading `main.rs` and the handlers that consume them, not from `http.rs` itself.

#### Public library surface (`lib.rs`)

`lib.rs` is 13 lines total. It declares ten `pub mod`s (`apns`, `app_attest`, `authority`, `config`, `grant`, `http`, `metrics`, `model`, `postgres`, `token`, lib.rs:2-9,11) plus one `pub(crate) mod strict_json` (lib.rs:10), and re-exports exactly three names at the crate root:
```
pub use http::{router, router_with_metrics, AppState};
```
(lib.rs:12). Every other module's public items are still reachable via their full path (e.g. `buzz_push_gateway::apns::ApnsTransport`), but only `router`, `router_with_metrics`, and `AppState` are surfaced at the crate root. `strict_json` is not part of the external API at all — it is `pub(crate)`, confirmed private to the crate, so no downstream consumer can call `buzz_push_gateway::strict_json::from_slice` directly.

Of the 92 `pub`-prefixed items (`fn`/`struct`/`enum`/`const`/`trait`/`mod`/`type`/`use`) across the crate's source files (counted by scanning every line matching `^\s*pub(\(crate\))?\s+(async\s+)?(fn|struct|enum|const|trait|mod|type|use)\b` in each `.rs` file), 28 are immediately preceded by a `///`/`//!` doc comment and 64 are not (~30% documented). Every public struct field in `authority.rs`'s domain types (`Challenge`, `NewInstallation`, `Installation`, `Delegation`, `DeliveryAuthority`) is undocumented at the field level even where the surrounding struct has a doc comment. `router` (http.rs:727) has no doc comment at all, while the near-identical `router_with_metrics` immediately above it does (http.rs:731-733). AGENTS.md states "New public API must have doc comments" — this crate's public surface does not meet that bar for roughly two-thirds of its items (see `push-gateway-conventions.md` and `push-gateway-debt.md` for the full breakdown).

#### Test coverage of the HTTP surface

`http.rs` itself contains zero `#[test]`/`#[tokio::test]` functions — confirmed by grepping the file for `#[test]`/`#[tokio::test]`, which returns no matches. No test anywhere in the crate constructs a `Router` via `router()`/`router_with_metrics()` and drives it with `tower::ServiceExt::oneshot` or an equivalent in-process HTTP client: grepping `crates/buzz-push-gateway/src/*.rs` for `oneshot`, `ServiceExt`, and `TestClient` returns no matches anywhere in the crate. The `Justfile:291-293` comment above the `cargo nextest run -p buzz-push-gateway` line claims "Gateway unit and black-box HTTP tests are infra-free," but no black-box HTTP test of the seven public routes exists in this crate. The only HTTP-server test in the crate is `apns.rs`'s `real_outbound_http_body_is_the_exact_constant_for_every_attempt` (apns.rs:287), which spins up a **local mock APNs server** to capture outbound bytes — it is not a test of this crate's own router. Route-level behavior (status codes, validation ordering, transcript construction, App Attest verification wiring) is therefore exercised only indirectly, through unit tests of the functions each handler calls (`config.rs`, `grant.rs`, `authority.rs`'s in-memory store) — never through the handlers themselves. This means a regression introduced purely inside a handler function body (e.g. swapping two field-order checks, using the wrong transcript domain string, forgetting a `?`) would not be caught by any test in this crate.

#### Uncertainties

- I found no external client (desktop, mobile, CLI) in this repository that calls these seven routes directly. `crates/buzz-relay/src/push_runtime.rs` (outside this crate's scope, see `push-gateway-integrations.md`) calls only `/v1/deliveries/apns`. I could not verify from source whether `/v1/installations/*` and `/v1/delegations/*` have any calling client in this repository at all — they are presumably called by the mobile app referenced in NIP-PL's "Public APNs Gateway Profile" section, but `mobile/` is out of scope for this analysis and I did not search it.
