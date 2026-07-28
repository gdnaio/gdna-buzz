## Module: buzz-push-gateway (`crates/buzz-push-gateway`)
### Aspect: Data Model

#### Overview

`buzz-push-gateway` persists all durable state in six PostgreSQL tables owned by its own migration (`crates/buzz-push-gateway/migrations/0001_push_gateway_authority.sql`), applied via `sqlx::migrate!("./migrations")` (`postgres.rs:23`). The crate also defines an equivalent in-memory model (`authority.rs`) used only by its own unit tests, and a closed set of wire types (`model.rs`) shared by the HTTP handlers. There is no ORM; `postgres.rs` issues hand-written SQL through `sqlx::query`/`sqlx::query_scalar`/`sqlx::raw_sql`.

#### Postgres Schema

All six tables are created by `crates/buzz-push-gateway/migrations/0001_push_gateway_authority.sql:4-66`:

| Table | Primary key | Columns | Notable constraints |
|---|---|---|---|
| `push_gateway_challenges` | `id` (UUID) | `id`, `challenge_hash` (BYTEA), `expires_at` (TIMESTAMPTZ), `created_at` | `challenge_hash` CHECK length = 32 (migration.sql:6) |
| `push_gateway_installations` | `id` (UUID) | `id`, `app_attest_key_id` (BYTEA, UNIQUE), `app_attest_public_key` (BYTEA), `assertion_counter` (BIGINT), `app_profile` (TEXT), `token_ciphertext` (BYTEA), `token_fingerprint` (BYTEA), `endpoint_epoch` (BIGINT), `expires_at`, `revoked_at` (nullable), `created_at`, `updated_at` | `app_attest_key_id` 1–128 bytes; `app_attest_public_key` 33–256 bytes; `assertion_counter` 0..4294967295; `app_profile IN ('buzz-ios-production','buzz-ios-sandbox')`; `token_ciphertext` 1–2048 bytes; `token_fingerprint` exactly 32 bytes; `endpoint_epoch > 0`; UNIQUE `(app_profile, token_fingerprint)` (migration.sql:12-26) |
| `push_gateway_delegations` | `id` (UUID) | `id`, `installation_id` (UUID, FK), `relay_pubkey` (BYTEA), `endpoint_epoch`, `generation`, `not_before`, `expires_at`, `revoked_at`, `updated_at` | FK → `push_gateway_installations(id)`; `relay_pubkey` exactly 32 bytes; `endpoint_epoch > 0`; `generation > 0`; UNIQUE `(installation_id, relay_pubkey)`; CHECK `not_before < expires_at` (migration.sql:29-40) |
| `push_gateway_endpoint_quotas` | `token_fingerprint` (BYTEA) | `token_fingerprint`, `window_started_at`, `admitted` (BIGINT), `updated_at` | `token_fingerprint` exactly 32 bytes; `admitted >= 0` (migration.sql:42-47) |
| `push_gateway_delivery_auth_replays` | `(relay_pubkey, auth_event_id)` | `relay_pubkey` (BYTEA), `auth_event_id` (BYTEA), `expires_at` | both BYTEA columns exactly 32 bytes (migration.sql:49-54) |
| `push_gateway_delivery_request_replays` | `(relay_pubkey, request_id)` | `relay_pubkey` (BYTEA), `request_id` (UUID), `expires_at` | `relay_pubkey` exactly 32 bytes (migration.sql:56-61) |

`push_gateway_installations.app_attest_key_id` is declared `UNIQUE` at the column level (migration.sql:14) in addition to the table's own `id` primary key — an installation's App Attest key can never be claimed by two installation rows. `push_gateway_endpoint_quotas` uses `token_fingerprint` itself as the primary key (migration.sql:43), i.e. there is exactly one quota-window row per distinct APNs-token fingerprint, independent of how many installations or delegations reference it.

#### Indexes

| Index | Table | Definition | Citation |
|---|---|---|---|
| `push_gateway_challenges_expiry` | `push_gateway_challenges` | `(expires_at)` | migration.sql:10 |
| `push_gateway_installations_expiry` | `push_gateway_installations` | partial `(expires_at) WHERE revoked_at IS NULL` | migration.sql:27 |
| `push_gateway_delegations_expiry` | `push_gateway_delegations` | partial `(expires_at) WHERE revoked_at IS NULL` | migration.sql:38 |
| `push_gateway_endpoint_quotas_updated` | `push_gateway_endpoint_quotas` | `(updated_at)` | migration.sql:47 |
| `push_gateway_delivery_auth_replays_expiry` | `push_gateway_delivery_auth_replays` | `(expires_at)` | migration.sql:54 |
| `push_gateway_delivery_request_replays_expiry` | `push_gateway_delivery_request_replays` | `(expires_at)` | migration.sql:61 |

No other indexes are declared; every lookup used by `postgres.rs` (`installation()`, `authorize_delivery()`, `upsert_delegation()`) goes through a primary key, a UNIQUE constraint, or one of the above.

#### Duplicate migration: two Postgres copies of this schema exist in the repo

The exact same six `CREATE TABLE`/`CREATE INDEX` statements also exist, byte-for-byte, at the repo-root `migrations/0015_push_gateway_authority.sql:1-66`. I diffed the two files directly: every `CREATE TABLE` and `CREATE INDEX` statement is identical between `crates/buzz-push-gateway/migrations/0001_push_gateway_authority.sql` and `migrations/0015_push_gateway_authority.sql`; the only difference is that the relay copy appends a trailing `INSERT INTO _operator_global_tables (table_name, reason) VALUES (...)` block naming all six tables (`migrations/0015_push_gateway_authority.sql:68-74`), which the crate-local copy does not have.

`buzz-db`'s migrator embeds the repo-root directory: `static MIGRATOR: sqlx::migrate::Migrator = sqlx::migrate!("../../migrations");` (`crates/buzz-db/src/migration.rs:11`), so `run_migrations` (`crates/buzz-db/src/migration.rs:14-23`) applies migration 0015 — and therefore creates all six `push_gateway_*` tables — inside the **relay's own** database whenever the relay runs its startup migration path. The only place these six table names appear in `buzz-db` beyond that migration file is a hard-coded string list inside a schema-lint allowlist helper, `operator_global_tables()` (`crates/buzz-db/src/migration.rs:332-357`, listing `push_gateway_challenges`, `push_gateway_installations`, and presumably the other four by the same pattern at `:343-344` onward) — there is no Rust struct, query, repository, or handler in `buzz-db` or `buzz-relay` that reads or writes rows in these six tables. I grepped `crates/buzz-relay/**` and `crates/buzz-db/**` for `buzz_push_gateway`/`buzz-push-gateway`/`PostgresAuthorityStore` and got zero matches — the relay binary never links this crate.

`docs/push-gateway-deployment.md:39` is explicit that the two schema copies must never point at the same database: "The URL MUST name a dedicated gateway database, not the relay database: SQLx stores its `_sqlx_migrations` history in `public`, so sharing a database would collide with another application's migration history." So by design the six tables the relay creates via migration 0015 are inert dead schema in the relay's own database — no relay code path ever touches them — while `buzz-push-gateway` maintains a live, operationally identical copy of the same six tables in a separate, dedicated database via its own `0001_push_gateway_authority.sql`. This is a concrete, unenforced duplication: identical DDL maintained in two files that must always be edited together (a new column or constraint added to one and not the other silently diverges the two schemas) but there is no test, lint, or CI check anywhere in the repo that verifies they stay in sync — see `push-gateway-debt.md`.

#### In-memory reference model (`authority.rs`)

`authority.rs` defines the same domain shapes as plain Rust structs, independent of SQL types:

| Type | Fields | Citation |
|---|---|---|
| `Challenge` | `id: Uuid`, `value: [u8; 32]`, `expires_at: i64` | authority.rs:15-19 |
| `NewInstallation` | `id`, `app_attest_key_id: Vec<u8>`, `app_attest_public_key: Vec<u8>`, `assertion_counter: u32`, `profile: AppProfile`, `token_ciphertext: Vec<u8>`, `token_fingerprint: [u8; 32]`, `endpoint_epoch: i64`, `expires_at: i64` | authority.rs:22-32 |
| `Installation` | same fields as `NewInstallation` plus `revoked: bool` | authority.rs:35-46 |
| `Delegation` | `id`, `installation_id`, `relay_pubkey: String`, `endpoint_epoch`, `generation`, `not_before`, `expires_at`, `revoked: bool` | authority.rs:49-58 |
| `DeliveryAuthority` | `delegation_id`, `installation_id`, `relay_pubkey`, `profile`, `token_ciphertext`, `endpoint_epoch`, `generation`, `expires_at` | authority.rs:61-69 |
| `DeliveryPermit` | `authority: DeliveryAuthority`, `relay_pubkey: String`, `request_id: Uuid` | authority.rs:73-78 |

The `revoked: bool` field on `Installation` and `Delegation` is populated by `PostgresAuthorityStore` as a hard-coded `false` at construction time (e.g. `postgres.rs:169` inside `installation()`), because every SQL `SELECT` that builds these structs already filters `WHERE revoked_at IS NULL` (`postgres.rs:143` in the query text passed to `installation()`), so a revoked row can never reach that construction path in the first place. On the Postgres path, `.revoked` is therefore always `false` and is never read by `http.rs` or `postgres.rs` — the real revocation check happens in SQL via `revoked_at IS NULL` predicates, not on this struct field. The field is exercised for real only inside `authority.rs`'s own `MemoryAuthorityStore`, e.g. the `i.revoked` checks in `advance_assertion_counter` (authority.rs:323), `rotate_endpoint`, `revoke_installation`, and `authorize_delivery`. See `push-gateway-debt.md` for the "unused field on the production type" finding.

`MemoryAuthorityStore` (authority.rs:192, wrapping `Mutex<MemoryState>`) is documented in its own doc comment as the store "used by conformance tests," giving every operation a single linearization point via one mutex (authority.rs:190-191 doc comment). One representational difference from the Postgres path: `Challenge.value` (the raw 32-byte challenge) is kept in the clear inside `MemoryState.challenges` (authority.rs:20, field `value: [u8; 32]`), whereas `PostgresAuthorityStore::put_challenge` stores only `Sha256::digest(c.value)` (`postgres.rs:112-120`) — the plaintext challenge is never persisted to Postgres. The external contract (`consume_challenge` still takes the plaintext value and matches it, `authority.rs:127-135` trait signature) is identical in both stores; only the at-rest representation in the test-only memory store retains the plaintext.

#### Wire types (`model.rs`)

Closed (`#[serde(deny_unknown_fields)]`) request/response types. No request type has a defaulted or optional top-level field:

| Type | Fields | `deny_unknown_fields` | Citation |
|---|---|---|---|
| `DeliveryRequest` | `v: u8`, `endpoint_grant: String`, `request_id: Uuid`, `expires_at: i64` | yes | model.rs:29-39 |
| `EndpointGrant` | `v: u8`, `delegation_id: Uuid`, `relay_pubkey: String`, `app_profile: AppProfile`, `endpoint_epoch: i64`, `generation: i64`, `expires_at: i64` | yes | model.rs:41-53 |
| `InstallationChallengeRequest` | `v: u8` | yes | model.rs:56-59 |
| `InstallationChallengeResponse` | `challenge_id: Uuid`, `challenge: String`, `expires_at: i64` | n/a (Serialize only) | model.rs:61-66 |
| `InstallationEnrollRequest` | `v`, `challenge_id`, `challenge`, `key_id`, `attestation`, `app_profile`, `endpoint`, `endpoint_epoch`, `expires_at` | yes | model.rs:69-84 |
| `InstallationEnrollResponse` | `installation_handle: Uuid`, `endpoint_epoch: i64`, `expires_at: i64` | n/a | model.rs:86-90 |
| `DelegationRequest` | `v`, `challenge_id`, `challenge`, `installation_handle`, `endpoint_epoch`, `generation`, `relay_pubkey`, `not_before`, `expires_at`, `assertion` | yes | model.rs:93-105 |
| `DelegationResponse` | `endpoint_grant: String` | n/a | model.rs:107-110 |
| `RotateEndpointRequest` | `v`, `challenge_id`, `challenge`, `installation_handle`, `endpoint_epoch`, `new_endpoint_epoch`, `endpoint`, `assertion` | yes | model.rs:113-124 |
| `RevokeDelegationRequest` | `v`, `challenge_id`, `challenge`, `installation_handle`, `relay_pubkey`, `generation`, `assertion` | yes | model.rs:126-136 |
| `RevokeInstallationRequest` | `v`, `challenge_id`, `challenge`, `installation_handle`, `endpoint_epoch`, `new_endpoint_epoch`, `assertion` | yes | model.rs:138-147 |
| `MutationResponse` | `status: &'static str` | n/a | model.rs:149-152 |
| `DeliveryResponse` (enum, `tag = "status"`, `snake_case`) | `Accepted`, `InvalidEndpoint{generation,invalid_at}`, `Retry{retry_after_seconds}` | yes | model.rs:154-165 |
| `ErrorBody` | `error: &'static str` | n/a | model.rs:167-170 |

Closed constants (model.rs:5-10):

| Constant | Value | Citation |
|---|---|---|
| `MAX_REQUEST_BYTES` | `8 * 1024` = 8192 | model.rs:5 |
| `MAX_GRANT_BYTES` | `4096` | model.rs:6 |
| `MAX_ENDPOINT_HEX_BYTES` | `512` | model.rs:7 |
| `APNS_RECONNECT_PAYLOAD` | `br#"{"aps":{"alert":{"body":"Reconnect to your relay now"},"mutable-content":1}}"#` | model.rs:8-9 |
| `WIRE_VERSION` | `1u8` | model.rs:10 |

`AppProfile` (model.rs:12-24) is a two-variant enum, `BuzzIosProduction` / `BuzzIosSandbox`, serialized `kebab-case` to `"buzz-ios-production"` / `"buzz-ios-sandbox"` (model.rs:12-13 derive attribute), with a `const fn as_str` mirror (model.rs:19-24) used everywhere the value is needed outside serde (SQL binds, AAD domain strings, config parsing).

#### Token and grant ciphertext representations

Two independent AEAD envelope formats, both `"<key-id>.<base64url-nopad(nonce‖ciphertext‖tag)>"`:

| Envelope | Producer | AAD domain string | Max size | Citation |
|---|---|---|---|---|
| Endpoint grant (`endpoint_grant`) | `grant::GrantKey::seal` | `b"buzz-stateful-delivery-capability-v1:"` + key id | `MAX_GRANT_BYTES` = 4096 (checked on both issue and open) | grant.rs:13-14, 55-56, 59-80, 137-142 |
| APNs token ciphertext (`token_ciphertext`, stored in `push_gateway_installations.token_ciphertext`) | `token::TokenKeyring::seal` | `b"buzz-apns-token-v1:"` + key id | `MAX_CIPHERTEXT_BYTES` = 2048 | token.rs:11-13, 50-51, 69-93 |

The grant's plaintext, once AES-256-GCM decrypted, is exactly the `EndpointGrant` struct above, re-parsed with the crate's own strict JSON parser rather than plain `serde_json::from_slice` (`grant.rs:92`, `crate::strict_json::from_slice(&plaintext).map_err(|_| GrantError::Invalid)?`).

#### APNs payload shape

Exactly one byte constant is ever sent as an APNs application body, `APNS_RECONNECT_PAYLOAD` (model.rs:8-9), referenced directly in `apns.rs:150` (`let body = APNS_RECONNECT_PAYLOAD;`) with an adjacent comment asserting it is "the only APNs application body in the program" and is never a serialization of any request/grant/endpoint/response data (apns.rs:147-149). This is verified by a test that captures the real outbound HTTP body across two different `DeliveryAttempt`/profile/endpoint combinations and asserts byte-equality to the constant (`apns.rs:287-341`, `real_outbound_http_body_is_the_exact_constant_for_every_attempt`).

| APNs HTTP/2 header | Value | Citation |
|---|---|---|
| `Authorization` | `bearer <cached ES256 JWT>` | apns.rs:230 |
| `Content-Type` | `application/json` | apns.rs:231 |
| `apns-id` | `attempt.request_id` (stable UUID, becomes the relay's durable job id) | apns.rs:232 |
| `apns-topic` | configured bundle id (`self.topic`) | apns.rs:233 |
| `apns-push-type` | `alert` | apns.rs:234 |
| `apns-priority` | `10` | apns.rs:235 |
| `apns-expiration` | `attempt.expires_at` | apns.rs:236 |

#### Test coverage

- `authority.rs`'s `MemoryAuthorityStore` behavior (retry release semantics, terminal fencing) is covered by `retry_releases_request_id_but_burns_auth_event` (authority.rs:547, `#[tokio::test]`) and `terminal_outcome_burns_request_id` (authority.rs:565, `#[tokio::test]`).
- `postgres.rs` has six `#[tokio::test]` functions, all marked `#[ignore = "requires PostgreSQL"]` or `#[ignore = "requires PostgreSQL with CREATEDB/CREATEROLE"]`: `readiness_requires_migrated_schema_dml_and_no_ddl` (postgres.rs:414), `reaper_deletes_active_child_of_retention_eligible_revoked_installation` (postgres.rs:502), `concurrent_same_request_id_admits_exactly_once` (postgres.rs:743), `concurrent_admissions_never_over_admit_past_quota_ceiling` (postgres.rs:796), `duplicated_retryable_release_does_not_permanently_unfence_request_id` (postgres.rs:854), `retryable_release_frees_request_id_on_real_postgres` (postgres.rs:905). None of these run under the default invocation: `Justfile:293` runs plain `cargo nextest run -p buzz-push-gateway` with no `--run-ignored`/`--include-ignored` flag, and I grepped every file under `.github/workflows/**` for `ignored` combined with `push-gateway` / `push_gateway` and got zero matches — no CI workflow ever passes `--ignored` for this package. So every Postgres-backed assertion about the actual schema (CHECK constraints, the `ON CONFLICT ... WHERE` admission predicate, concurrency behavior, the migration+grant flow) is unverified in CI; it only runs when a developer manually supplies `--ignored`/`--include-ignored` against a live Postgres.
- `model.rs` itself has no `#[test]`/`#[tokio::test]` functions — its types are exercised transitively through `grant.rs`, `token.rs`, and `config.rs` tests, never directly (e.g. no test constructs and round-trips an `InstallationEnrollRequest` through serde on its own).
- `grant.rs`'s AEAD envelope framing is exercised by its four `#[test]`s (`current_issues_and_predecessor_opens_after_rotation` grant.rs:169, `configured_route_id_is_authenticated_even_when_keys_match` grant.rs:187, `complete_envelope_length_is_bounded_on_issue_and_open` grant.rs:201, `tampering_unknown_ids_and_duplicate_configuration_fail` grant.rs:214). `token.rs` has zero `#[test]` functions of its own — confirmed by grep for `#\[test\]`/`#\[tokio::test\]` in `token.rs` returning no matches — so the APNs-token AEAD sealing/opening path (the code that actually protects the raw device token at rest in `push_gateway_installations.token_ciphertext`) is unit-tested nowhere in this crate; it is exercised only indirectly through `http.rs`'s `enroll`/`rotate_endpoint`/`deliver` handlers, none of which have handler-level tests either (see `push-gateway-api-surface.md`).

#### Uncertainties

- I did not run the Postgres-backed tests myself (no live Postgres available in this environment), so the DDL's actual runtime behavior (CHECK constraints firing, the `ON CONFLICT ... WHERE` predicate in `authorize_delivery`, FK cascade ordering in `reap_expired`) is verified only by reading the SQL text and the `#[ignore]`d tests' assertions, not by execution.
- I cannot determine from source alone whether any deployment process actually runs the ignored Postgres-backed tests outside of CI (e.g. manually, before a release); the repo gives no evidence either way — `docs/push-gateway-deployment.md` does not mention running these tests as part of any release checklist.
