## Module: buzz-push-gateway (`crates/buzz-push-gateway`)
### Aspect: Conventions

#### Error handling style

Every fallible public operation returns a `Result` with a crate-local `thiserror`-derived enum: `ConfigError` (config.rs:38-43), `GrantError` (grant.rs:23-29), `TokenError` (token.rs:20-25), `AppAttestError` (app_attest.rs:26-28), `ApnsError` (apns.rs:184-190), `AuthorityError` (authority.rs:101-106). None of these error types carry the underlying cause as a field beyond `AuthorityError`'s two bare variants (`Rejected`/`Unavailable`) — errors are intentionally coarse-grained and non-leaking rather than wrapping `sqlx::Error`, `serde_json::Error`, etc. directly in the public type; `postgres.rs`'s `db(sqlx::Error) -> AuthorityError` (postgres.rs:71-73) is the single funnel point that collapses every SQL failure to `AuthorityError::Unavailable`, discarding the original `sqlx::Error` entirely rather than boxing/wrapping it. This is a deliberate security property (avoiding SQL error text in an HTTP response — see `push-gateway-security.md`), but it also means a genuine bug (e.g. a malformed query) is indistinguishable from a transient DB outage from the caller's perspective.

No production code path uses `.unwrap()` or `.expect()`. I grepped `crates/buzz-push-gateway/src/*.rs` for `unwrap()`/`expect(`/`panic!`/`unreachable!` outside `#[cfg(test)]` modules and found none — every occurrence of `.unwrap()` (apns.rs:284-297, grant.rs:171-180, authority.rs:528-570, postgres.rs:703, config.rs:239) and `.expect(` (postgres.rs:424, 892; metrics.rs:160) is inside a `mod tests` block, and `main.rs` propagates every fallible call with `?` all the way up to `fn main() -> Result<(), Box<dyn std::error::Error>>` (main.rs:21). This matches AGENTS.md's rule "Do not introduce new `unwrap()` or `expect()` in production paths — use `?` and proper error types" — this crate fully complies. There is also zero `unsafe` code anywhere in the crate (grep for `unsafe` returns no matches).

#### Logging

The crate is minimal on logging: only one `tracing::warn!` call exists in the entire crate, in the reaper's failure branch (main.rs:76, `"push gateway retention reaper failed"`), and it logs a static string with no interpolated values — no risk of a secret leaking into that specific log line. `main.rs:22-25` configures `tracing_subscriber::fmt().json().with_env_filter(EnvFilter::from_default_env())` at startup, consistent with the relay's own JSON-structured logging convention (per `.env.example`'s `RUST_LOG` guidance, though this crate reads `RUST_LOG` only implicitly via `EnvFilter::from_default_env()`, not as a `Config` field). No other module logs anything — `http.rs`'s handlers return typed error responses but never log rejected requests, meaning there is no audit trail in logs for repeated `invalid_auth`/`invalid_grant` rejections (see `push-gateway-security.md` for the abuse-visibility implication).

#### Naming

Consistent `snake_case` module and function naming throughout. Wire-facing struct fields use `snake_case` matching the JSON wire format directly (e.g. `endpoint_grant`, `challenge_id`) rather than a separate internal naming convention with `#[serde(rename)]` translation — the one exception is `AppProfile`'s `#[serde(rename_all = "kebab-case")]` (model.rs:12-13) because Rust enum variants are `PascalCase` but the wire values are kebab-case strings (`buzz-ios-production`). Transcript-building structs are named `<Route>Transcript` consistently (`EnrollTranscript`, `DelegateTranscript`, `RotateTranscript`, `RevokeDelegationTranscript`, `RevokeInstallationTranscript`) — a clear, repeatable local convention for one-off serialization shapes local to a single handler.

#### `unsafe` / lint attributes

Zero `unsafe` blocks (see above). Exactly one `#[allow(...)]` attribute in the whole crate: `#[allow(clippy::too_many_arguments)]` on the `AuthorityStore::authorize_delivery` trait method (authority.rs:151), justified by an adjacent doc comment explaining the method's role as a single atomic linearization point (authority.rs:148-150) — this is a defensible, narrowly-scoped, documented lint suppression, not a blanket `#[allow(dead_code)]` or similar. No other `#[allow]`, `#[warn]`, or `#![...]` crate-level attribute exists anywhere in the crate (confirmed by grep for `#!\[` and `#\[allow` returning only this one hit).

#### File-size discipline

| File | Lines |
|---|---|
| `postgres.rs` | 952 |
| `http.rs` | 776 |
| `authority.rs` | 576 |
| `apns.rs` | 367 |
| `config.rs` | 297 |
| `metrics.rs` | 207 |
| `grant.rs` | 240 |
| `model.rs` | 170 |
| `app_attest.rs` | 140 |
| `main.rs` | 143 |
| `token.rs` | 122 |
| `strict_json.rs` | 89 |
| `lib.rs` | 13 |

AGENTS.md documents a hard 1000-line/file ceiling only for the **mobile** Flutter codebase ("Hard ceiling: 1000 lines/file, enforced by `mobile/scripts/check-file-sizes.mjs`"); I found no equivalent stated line-count policy for Rust crates in AGENTS.md or CONTRIBUTING.md. Under the mobile-specific number as an informal yardstick, `postgres.rs` at 952 lines is close to (but technically under) that ceiling; note over half of `postgres.rs` (postgres.rs:407-952, its `#[cfg(test)]` module) is test code, not production logic — the production portion is roughly 405 lines. `http.rs` at 776 lines mixes seven full request handlers, their private transcript structs, and router construction in one file with no submodule split by route group.

#### Doc-comment discipline

Module-level (`//!`) doc comments are present and substantive on most files (`lib.rs:1`, `postgres.rs:1-2`, `model.rs:1`, `authority.rs:1-5`, `apns.rs:1`, `app_attest.rs:1-2`, `grant.rs:1-2`, `token.rs:1-2`, `metrics.rs:1-13`, `strict_json.rs:1`) — every source file except `config.rs`, `main.rs`, and `http.rs` has one. As detailed in `push-gateway-api-surface.md`, item-level doc comments are inconsistent: 28 of 92 public items (~30%) have a preceding `///`. The pattern skews toward documenting *why* (module-level design rationale, e.g. `authority.rs`'s note on the bootstrap trust assumption) over documenting *what* (individual field/parameter semantics on public structs), which partially explains the low item-level percentage — several undocumented items are self-descriptive one-line structs where a doc comment would be low-value, but others (e.g. every `AppState` field, `router` itself) would clearly benefit.

#### Test organization

Every test module follows the same shape: a trailing `#[cfg(test)] mod tests { use super::*; ... }` block at the end of the file (apns.rs:273, authority.rs:489, config.rs:187, grant.rs:152, metrics.rs:122, postgres.rs:407, strict_json.rs:80) — no file places tests in a separate `tests/` directory or a sibling `_test.rs` file, and there is no top-level `tests/` integration-test directory for this crate at all (confirmed: `crates/buzz-push-gateway/` contains only `Cargo.toml`, `migrations/`, `src/`). `http.rs`, `app_attest.rs`, `model.rs`, `main.rs`, and `token.rs` have **no** `#[cfg(test)] mod tests` block at all — of these, `token.rs` is the most notable gap since it implements the AEAD sealing of the raw APNs token (see `push-gateway-data-model.md`'s test-coverage section). Tests needing Postgres are consistently marked `#[ignore = "requires PostgreSQL"]` with a human-readable reason string rather than a bare `#[ignore]`, which is a positive, self-documenting convention. Several Postgres tests include unusually long inline comments narrating an adversarial "red-team" scenario before the test body (e.g. postgres.rs:790-795, 838-853) — a documentation-in-test style not seen elsewhere in the crate's non-Postgres test modules, suggesting these specific tests were written to pin down a previously-discussed race condition rather than as routine coverage.
