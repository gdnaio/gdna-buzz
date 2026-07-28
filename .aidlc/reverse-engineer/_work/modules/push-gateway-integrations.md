## Module: buzz-push-gateway (`crates/buzz-push-gateway`)
### Aspect: Integrations

#### Relationship to the main relay: independent process, no shared runtime, one HTTPS contract

`buzz-push-gateway` is compiled as its own binary (`Cargo.toml:14-16`) and ships in its own container image via `Dockerfile.push-gateway`, distinct from the relay's `Dockerfile`. I grepped `crates/buzz-relay/**` for `buzz_push_gateway`/`buzz-push-gateway`/`PostgresAuthorityStore` and got zero matches — **the relay binary never links this crate**. The relay's `Cargo.toml` does not list `buzz-push-gateway` as a dependency (confirmed by reading `crates/buzz-relay/Cargo.toml`'s dependency list). The two processes communicate over exactly one HTTPS contract: the relay's `crates/buzz-relay/src/push_runtime.rs` calls `POST /v1/deliveries/apns` on the gateway using a plain `reqwest::Client` (`push_runtime.rs:517-528`, `send_gateway_request`), signing each request with a NIP-98 event built fresh per call (`push_runtime.rs:551-565`, `nip98_header`) and passing `endpoint_grant`/`request_id`/`expires_at` as the body (`push_runtime.rs:32-39`, `push_runtime.rs:507-515`). The relay's own copy of the request/response shapes (`push_runtime.rs:32-49`) is a hand-duplicated mirror of this crate's `model::DeliveryRequest`/`DeliveryResponse` — there is no shared type crate between them (see `push-gateway-debt.md` for the duplication risk this creates).

There is **no shared database access, no event subscription, and no other IPC** between the relay and this gateway. The gateway's `DATABASE_URL` must point at a dedicated database per `docs/push-gateway-deployment.md:39` ("The URL MUST name a dedicated gateway database, not the relay database"), and confirmed independently by the fact that the relay creates an inert, never-queried copy of the same six tables in its own database via migration `0015_push_gateway_authority.sql` (see `push-gateway-data-model.md`). The two services are related only by the NIP-PL protocol contract documented in `docs/nips/NIP-PL.md`, not by any code-level dependency in either direction.

#### Confirmed: no dependency on `buzz-core`, `buzz-db`, `buzz-auth`, or any other workspace crate

`crates/buzz-push-gateway/Cargo.toml:19-50`'s `[dependencies]` list contains **zero** `{ workspace = true, path = ... }`-style intra-repo crate references and zero `path = "../..."` entries — every dependency is an external crates.io package (`aes-gcm`, `appattest`, `axum`, `async-trait`, `base64`, `byteorder`, `minicbor`, `chrono`, `hex`, `getrandom`, `metrics`, `metrics-exporter-prometheus`, `nostr`, `p256`, `rand`, `reqwest`, `serde`, `serde_json`, `sqlx`, `sha2`, `thiserror`, `tokio`, `tower`, `tower-http`, `tracing`, `tracing-subscriber`, `url`, `uuid`). This directly confirms the module inventory's claim that this crate has no internal dependency on `buzz-core` or any other workspace crate — it is genuinely standalone at the `cargo` dependency-graph level, unlike almost every other crate in `crates/` (e.g. `buzz-media`, `buzz-pubsub`, `buzz-sdk` all depend on `buzz-core`). The `nostr` dependency (Cargo.toml:31) is the external `rust-nostr` SDK crate (`nostr = { version = "0.44", ... }` in the workspace root `Cargo.toml:61`, features `nip44`+`nip98`), used only for NIP-98 auth-header verification (`http.rs:21-24`) and Nostr `Event`/`Kind` types — not this repo's own `buzz-core`/`buzz-sdk` Nostr helpers.

#### Outbound: Apple Push Notification service (APNs)

| Aspect | Detail | Citation |
|---|---|---|
| Endpoint | `https://api.push.apple.com/3/device/{endpoint}` (production) or `https://api.sandbox.push.apple.com/3/device/{endpoint}` (sandbox), selected by `AppProfile` | apns.rs:189-190, 233-237 |
| Protocol | `reqwest::Client` over HTTPS; the workspace `reqwest` dependency is configured `default-features = false, features = ["json", "rustls"]` (root `Cargo.toml:93`), so outbound TLS is rustls-backed, not OpenSSL/native-tls | Cargo.toml:93 (root) |
| Auth mechanism | Token-based (ES256 JWT provider token per Apple's HTTP/2 API), **not** certificate-based. `SigningKey::from_pkcs8_pem` loads the `.p8` key once at startup (apns.rs:176-178, `token_with_client`) | apns.rs:141-192 |
| JWT construction | Header `{"alg":"ES256","kid":<key_id>}`, claims `{"iss":<team_id>,"iat":<now>}`, ES256-signed, base64url-no-pad encoded, cached for 50 minutes (`now - jwt.issued_at < 50*60`) before regenerating | apns.rs:194-217 |
| Client timeout | 15 seconds (`Duration::from_secs(15)`) | apns.rs:159 |
| Credential refresh | On `RefreshCredential` outcome (APNs `403 ExpiredProviderToken`), the cached JWT is dropped (`refresh_credential`, apns.rs:264-269) and exactly one retry is issued from the calling handler (http.rs:667-671) | apns.rs:52-54, 264-269; http.rs:667-671 |
| Request headers | `Authorization: bearer <jwt>`, `Content-Type: application/json`, `apns-id`, `apns-topic`, `apns-push-type: alert`, `apns-priority: 10`, `apns-expiration` | apns.rs:230-236 |
| Body | Always the fixed constant `APNS_RECONNECT_PAYLOAD` (model.rs:8-9) — verified never to vary by request in `real_outbound_http_body_is_the_exact_constant_for_every_attempt` (apns.rs:287-341) | apns.rs:150, 227 |

This is a direct HTTP/2 integration with Apple's provider API (no third-party push-delivery SDK). The `p256` crate (Cargo.toml:32) supplies the ECDSA signing primitive for the JWT; no APNs client library is used.

#### Outbound: PostgreSQL

`PostgresAuthorityStore` (postgres.rs:11-58) is a thin wrapper around a `sqlx::PgPool`. Access patterns:
- Production runtime pool: `PgPoolOptions::new().max_connections(20)` (main.rs:62-65).
- Migration-only pool: `max_connections(1)` (main.rs:29-31), used exclusively by the `--migrate-only` path, which also runs the DDL-adjacent `REVOKE`/`GRANT` statements in `apply_migrations_and_grants` (postgres.rs:20-53) to lock the runtime role down to `CONNECT` + `USAGE` + `SELECT,INSERT,UPDATE,DELETE` on exactly the six gateway tables, and revoke `CREATE` from both the database and the `public` schema for that role and `PUBLIC`.
- Every mutating multi-step operation (`upsert_delegation`, `authorize_delivery`, `reap_expired`) uses an explicit `pool.begin()` transaction with `FOR UPDATE` row locks rather than relying on statement-level atomicity alone (postgres.rs:184, 267, 341).
- `readiness` (`ready()`, postgres.rs:81-110) queries `has_table_privilege`/`has_database_privilege`/`has_schema_privilege` system functions directly against `current_user` — it is a live privilege probe, not a cached config check, so a privilege change made *after* startup (e.g. an operator accidentally re-granting `CREATE`) would flip readiness to failing on the next probe.

#### `metrics.rs`'s metrics backend

Uses the `metrics` facade crate + `metrics-exporter-prometheus` (Cargo.toml:29-30), matching the pattern used elsewhere in the repo (e.g. the relay). `install()` (metrics.rs:29-36) calls `PrometheusBuilder::new().set_buckets_for_metric(...).install_recorder()`, installing a **process-global** recorder — the doc comment explicitly notes this "Must be called at most once per process" (metrics.rs:26). Unlike the relay's exporter (per metrics.rs's own doc comment, "Unlike the relay's exporter, this installs **no** HTTP listener"), this crate does not use `PrometheusBuilder`'s built-in HTTP listener; instead `main.rs:38` calls `buzz_push_gateway::metrics::install()` once and threads the returned `PrometheusHandle` into `router_with_metrics` (main.rs:91), which serves `/metrics` on the **private health router** only (http.rs:756-772) — metrics are never reachable on the public port. See `push-gateway-configuration.md` for the Prometheus scrape opt-in wiring and `push-gateway-security.md` for the label-cardinality argument.

#### App Attest dependency (`appattest` crate)

The `appattest` crate (Cargo.toml:20, version `0.1.1`, `default-features = false`) is an external crates.io package (confirmed via `Cargo.lock:126-129`: `source = "registry+https://github.com/rust-lang/crates.io-index"`) that supplies `Attestation`/`Assertion` CBOR parsing and Apple chain verification (`app_attest.rs:1`). This crate does not implement App Attest verification itself; it delegates the cryptographic chain-of-trust verification to this third-party library and layers its own transcript/challenge/replay logic on top (see `push-gateway-business-rules.md` and `push-gateway-security.md`). `minicbor` (Cargo.toml:24) is used directly by `app_attest.rs::assertion_counter` (app_attest.rs:118-136) for a narrow, hand-rolled parse of the two-field assertion CBOR map (extracting only `signCount`), separate from whatever CBOR parsing `appattest` does internally for full verification.

#### No integration found: FCM, UnifiedPush, or any other push transport

`AppProfile` (model.rs:12-16) and the whole `apns.rs` module are APNs-only. Grepping the crate for `fcm`, `firebase`, or `unifiedpush` (case-insensitive) returns no matches — there is no partial or stubbed integration with any other push transport in this codebase, consistent with `docs/nips/NIP-PL.md` stating FCM and UnifiedPush are "not a conforming v1 public-gateway profile."

#### Test coverage of integrations

- The Postgres integration's transactional/concurrency behavior is tested only by the six `#[ignore]`d tests in `postgres.rs` (not run in CI — see `push-gateway-data-model.md`).
- The APNs integration is tested against a **local mock HTTP server** standing in for Apple (`apns.rs:287-341`), not against Apple's sandbox environment; there is no live-APNs-sandbox test anywhere in the repo for this crate (confirmed by grep for `sandbox.push.apple.com` outside `apns.rs` itself returning no other matches).
- There is no test exercising the relay-to-gateway HTTP contract from either side jointly — `push_runtime.rs`'s own tests (`gateway_retries_send_the_same_request_id_over_http`, referenced in that file) run against a mock server on the relay side, and this crate has no corresponding test acting as the gateway counterpart.
