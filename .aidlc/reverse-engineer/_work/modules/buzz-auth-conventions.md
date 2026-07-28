## Module: buzz-auth (`crates/buzz-auth`)

### Conventions

### Module organisation

8 source files, 1,877 LOC total. One file per concern; `lib.rs` holds the
aggregate types plus re-exports.

| File | LOC | Responsibility |
|------|-----|----------------|
| `crates/buzz-auth/src/lib.rs` | 243 | crate docs, module decls, re-exports, `AuthMethod`/`AuthContext`/`AuthConfig`/`AuthService`, dev key derivation |
| `crates/buzz-auth/src/access.rs` | 251 | `ChannelAccessChecker`, `require_scope`, read/write helpers, `MockAccessChecker` |
| `crates/buzz-auth/src/error.rs` | 59 | `AuthError` only |
| `crates/buzz-auth/src/nip42.rs` | 183 | challenge gen + AUTH verification + relay-URL normalisation |
| `crates/buzz-auth/src/nip98.rs` | 317 | HTTP auth verification + URL normalisation |
| `crates/buzz-auth/src/nip98_replay.rs` | 249 | `Nip98ReplayGuard`, key formats, TTL constants |
| `crates/buzz-auth/src/rate_limit.rs` | 326 | `RateLimiter`, `RateLimitConfig`, `LimitType`, `RateLimitResult`, key formats |
| `crates/buzz-auth/src/scope.rs` | 249 | `Scope`, `parse_scopes` |

Structural conventions observed:

- Every module has a `//!` module-level doc block; several encode the algorithm
  as a numbered list before the code (`crates/buzz-auth/src/nip42.rs:1-7`,
  `crates/buzz-auth/src/nip98.rs:13-24`).
- Every module ends with `#[cfg(test)] mod tests { use super::*; ... }` except
  `error.rs`, which has no tests (`crates/buzz-auth/src/error.rs` — 59 lines, no
  test module).
- Test doubles live in the same file as the trait they implement, gated on
  `#[cfg(any(test, feature = "test-utils"))]`, and are re-exported from `lib.rs`
  under the same gate (`crates/buzz-auth/src/lib.rs:46-51`).
- Private helpers are placed immediately adjacent to their only caller
  (`normalize_relay_url` above `verify_nip42_event`,
  `crates/buzz-auth/src/nip42.rs:19-33`; `normalize_url` below
  `verify_nip98_event`, `crates/buzz-auth/src/nip98.rs:145-153`).
- Crate lints are declared at the top of `lib.rs`: `#![deny(unsafe_code)]`,
  `#![warn(missing_docs)]` (`crates/buzz-auth/src/lib.rs:1-2`). Every public item
  carries a doc comment, including individual enum variants and struct fields.

---

### Naming conventions

| Category | Convention | Examples |
|----------|-----------|----------|
| Verification fns | `verify_<spec>_event` | `verify_nip42_event` (`nip42.rs:47`), `verify_nip98_event` (`nip98.rs:55`) |
| Redis key builders | `<subject>_key` / `<subject>_key_for_scope` | `rate_limit_key` (`rate_limit.rs:201`), `ip_rate_limit_key` (`rate_limit.rs:213`), `nip98_replay_key` (`nip98_replay.rs:114`), `nip98_replay_key_for_scope` (`nip98_replay.rs:119`) |
| Check fns | `check_<subject>` (async, returns `Result<(), AuthError>`) or `require_<subject>` (sync) | `check_read_access`/`check_write_access` (`access.rs:72`, `:88`), `require_scope` (`access.rs:60`) |
| Trait methods that mutate shared state | verb-first, `try_`-prefixed when they can legitimately decline | `try_mark`, `try_mark_in_scope` (`nip98_replay.rs:97`, `:66`); `check_and_increment` (`rate_limit.rs:174`) |
| Traits | noun-agent (`-er` / `-Checker` / `-Guard`) | `RateLimiter`, `ChannelAccessChecker`, `Nip98ReplayGuard` |
| Test doubles | `Always<Behaviour><Trait>` or `Mock<Subject>` | `AlwaysAllowRateLimiter` (`rate_limit.rs:219`), `AlwaysFreshReplayGuard` (`nip98_replay.rs:127`), `MockAccessChecker` (`access.rs:108`) |
| Constants | SCREAMING_SNAKE with unit suffix | `TIMESTAMP_TOLERANCE_SECS` (`nip42.rs:35`, `nip98.rs:32`), `DEFAULT_REPLAY_TTL_SECS` / `MAX_REPLAY_TTL_SECS` (`nip98_replay.rs:46`, `:59`) |
| serde default fns | `default_<tier>_<metric>`, abbreviated | `default_human_msg`, `default_agent_std_api`, `default_agent_plat_msg` (`rate_limit.rs:110-130`) |
| Enum-to-wire mapping | `as_str()` returning `&str` (or `&'static str`) | `Scope::as_str` (`scope.rs:114`), `LimitType::key_suffix` (`rate_limit.rs:71`) |
| Scope wire strings | `resource:action` lowercase, `admin:resource` for admin | `messages:read`, `admin:channels` (`scope.rs:116-131`) |
| Redis key namespace | `buzz:` prefix, community segment second, subject last | `buzz:{community}:ratelimit:{pubkey}:{suffix}` (`rate_limit.rs:203`), `buzz:{scope}:nip98:{id}` (`nip98_replay.rs:120`) |

---

### Error handling

Single crate-wide error enum, `thiserror`-derived, no `anyhow`, no boxed errors,
no custom `Display` impls (`crates/buzz-auth/src/error.rs:8-9`).

| Variant | `#[error(...)]` message | Constructed at |
|---------|-------------------------|----------------|
| `InvalidSignature` | `invalid signature or malformed auth event` | `nip42.rs:53` (wrong kind), `nip42.rs:56` (sig/id failure) |
| `ChallengeMismatch` | `challenge mismatch` | `nip42.rs:62` (tag missing), `nip42.rs:65` (mismatch) |
| `RelayUrlMismatch` | `relay url mismatch` | `nip42.rs:72` (tag missing), `nip42.rs:75` (mismatch) |
| `EventExpired` | `auth event timestamp outside ±60s window` | `nip42.rs:82` |
| `Nip98Invalid(String)` | `NIP-98 HTTP Auth verification failed: {0}` | `nip98.rs:63`, `:67`, `:75`, `:82`, `:95`, `:98`, `:108`, `:111`, `:123` (9 sites) |
| `Nip98Replay` | `NIP-98 replay: event id already seen within window` | **never constructed in this crate** — intended for callers of `Nip98ReplayGuard` (usage example at `nip98_replay.rs:24`) |
| `PubkeyMismatch` | `pubkey mismatch: event pubkey does not match authenticated identity` | **never constructed in this crate** |
| `InsufficientScope { required: String, have: Vec<String> }` | `insufficient scope: required {required}, have {have:?}` | `access.rs:64-67` |
| `ChannelAccessDenied` | `channel access denied` | `access.rs:83`, `access.rs:99` |
| `Internal(String)` | `internal auth error: {0}` | `lib.rs:132` (spawn_blocking panic), `lib.rs:165` (dev key derivation) |

Handling patterns:

- **Guard clauses with early return**, not nested conditionals — both verifiers
  are flat sequences of `if ... { return Err(...) }`
  (`crates/buzz-auth/src/nip42.rs:52-84`, `crates/buzz-auth/src/nip98.rs:66-127`).
- **Coarsening on the NIP-42 path**: the underlying `VerificationError` is
  discarded via `.map_err(|_| AuthError::InvalidSignature)`
  (`crates/buzz-auth/src/nip42.rs:56`) and wrong-kind reuses the same variant
  (`:53`) — a caller cannot distinguish "wrong kind" from "bad signature".
- **Descriptive on the NIP-98 path**: each failure carries a formatted string,
  including the offending values for URL/method mismatches
  (`crates/buzz-auth/src/nip98.rs:98-100`, `:111-113`). The doc explicitly says
  the message is "safe for server logs but should not be forwarded verbatim to
  clients" (`crates/buzz-auth/src/nip98.rs:53-54`).
- **Error-hygiene rule stated on the enum**: "Do **not** include raw token
  values, database contents, or stack traces in error messages"
  (`crates/buzz-auth/src/error.rs:5-7`).
- **Double-`?` for nested Results**: `spawn_blocking` join error and the inner
  verification error are both propagated on one line with `??`
  (`crates/buzz-auth/src/lib.rs:132`).
- **`Infallible` instead of a fallible parse**: `FromStr for Scope` cannot fail;
  unknown input becomes `Scope::Unknown(_)`
  (`crates/buzz-auth/src/scope.rs:143-166`).
- **One production-path `expect`**: `parse_scopes` calls
  `.expect("infallible: Scope::from_str cannot fail")`
  (`crates/buzz-auth/src/scope.rs:175`) — statically unreachable given the
  `Infallible` error type, but it is the only non-test `expect`/`unwrap` in the
  crate. Every other `unwrap`/`expect` occurrence is inside a `#[cfg(test)]`
  module.
- **No `unsafe`**: `#![deny(unsafe_code)]` (`crates/buzz-auth/src/lib.rs:1`); the
  only occurrence of the token `unsafe` in the whole crate is that attribute.
- **No logging on error paths**: despite `tracing` being a declared dependency
  (`crates/buzz-auth/Cargo.toml:20`), nothing in `src/` emits a log or span.

---

### Async conventions

- Traits use **RPITIT** (`-> impl Future<Output = ...> + Send`) for
  `ChannelAccessChecker` (`crates/buzz-auth/src/access.rs:35-39`, `:46-51`) and
  `RateLimiter` (`crates/buzz-auth/src/rate_limit.rs:174-181`, `:188-193`) —
  these are not dyn-compatible.
- `Nip98ReplayGuard` deliberately uses `Pin<Box<dyn Future + Send + 'a>>`
  instead (`crates/buzz-auth/src/nip98_replay.rs:66-71`, `:97-103`) so it can be
  stored as `Arc<dyn Nip98ReplayGuard>` by the relay
  (`crates/buzz-relay/src/state.rs:582`). Two different async-trait styles
  coexist in one crate, chosen per dyn-dispatch need.
- Default trait method bodies use `async move { ... }` blocks
  (`crates/buzz-auth/src/access.rs:52-55`) or `Box::pin(async move { ... })`
  (`crates/buzz-auth/src/nip98_replay.rs:101`).
- Free async fns are plain `pub async fn` (`crates/buzz-auth/src/access.rs:72`,
  `:88`).
- CPU-bound crypto is offloaded with `spawn_blocking` and the values are cloned
  into the closure first (`crates/buzz-auth/src/lib.rs:125-132`); the sync
  verifier documents the requirement in its own doc comment
  (`crates/buzz-auth/src/nip42.rs:46`).

---

### Configuration conventions

- Config structs derive `Serialize + Deserialize + Default`, and **every**
  `RateLimitConfig` field carries `#[serde(default = "fn")]` so partial input
  deserialises (`crates/buzz-auth/src/rate_limit.rs:85-108`).
- The manual `Default` impl duplicates the same seven `default_*` fns rather
  than using `#[derive(Default)]` with literals, keeping one source of truth for
  each number (`crates/buzz-auth/src/rate_limit.rs:132-144`).
- Nested config uses `#[serde(default)]` on the field
  (`crates/buzz-auth/src/lib.rs:93`).
- Enum wire forms use `#[serde(rename_all = "snake_case")]`
  (`crates/buzz-auth/src/rate_limit.rs:57`).

---

### Testing patterns

Totals by attribute (grep of `crates/buzz-auth/src/*.rs`):

| File | `#[test]` | `#[tokio::test]` | Total |
|------|-----------|------------------|-------|
| `crates/buzz-auth/src/nip98.rs` | 11 | 0 | 11 |
| `crates/buzz-auth/src/nip42.rs` | 8 | 0 | 8 |
| `crates/buzz-auth/src/nip98_replay.rs` | 6 | 1 | 7 |
| `crates/buzz-auth/src/rate_limit.rs` | 4 | 1 | 5 |
| `crates/buzz-auth/src/scope.rs` | 5 | 0 | 5 |
| `crates/buzz-auth/src/access.rs` | 0 | 5 | 5 |
| `crates/buzz-auth/src/lib.rs` | 1 | 3 | 4 |
| `crates/buzz-auth/src/error.rs` | 0 | 0 | 0 |
| **Total** | **35** | **10** | **45** |

There is no `crates/buzz-auth/tests/` directory — all 45 tests are in-file unit
tests.

Patterns:

- **Builder helpers per file**: `make_auth_event(keys, challenge, relay_url)`
  (`crates/buzz-auth/src/nip42.rs:95-100`, duplicated at
  `crates/buzz-auth/src/lib.rs:174-179`), `make_nip98_event(...)` with optional
  payload/timestamp params (`crates/buzz-auth/src/nip98.rs:163-186`).
- **`fixture_ctx(host)` deterministic tenant fixtures**: SHA-256 of the host name
  → first 16 bytes → `Uuid` → `CommunityId`, so key-prefix assertions are stable
  (`crates/buzz-auth/src/rate_limit.rs:253-260`,
  `crates/buzz-auth/src/nip98_replay.rs:149-155`). `access.rs` uses a simpler
  random-UUID variant (`crates/buzz-auth/src/access.rs:159-161`).
- **`assert!(matches!(result, Err(AuthError::X)))`** as the standard rejection
  assertion (`crates/buzz-auth/src/nip42.rs:124-127`,
  `crates/buzz-auth/src/access.rs:188-191`, `crates/buzz-auth/src/lib.rs:227`).
- **Real keypairs, never fixtures**: every test calls `Keys::generate()` and
  signs with `EventBuilder`, so signature verification is genuinely exercised
  (e.g. `crates/buzz-auth/src/nip98.rs:190-191`).
- **Const-drift tripwires**: assertions deliberately made over constants, with
  `#[allow(clippy::assertions_on_constants)]` and a comment explaining why
  (`crates/buzz-auth/src/nip98_replay.rs:210-230`,
  `crates/buzz-auth/src/nip98_replay.rs:232-237`).
- **Property-style invariant tests over key formats**: all-lowercase character
  scan (`crates/buzz-auth/src/rate_limit.rs:290-306`,
  `crates/buzz-auth/src/nip98_replay.rs:194-208`), cross-community
  distinctness (`crates/buzz-auth/src/rate_limit.rs:275-288`,
  `crates/buzz-auth/src/nip98_replay.rs:178-192`,
  `crates/buzz-auth/src/access.rs:225-251`).
- **Set-cardinality + no-duplicate + no-`Unknown` assertions** on the scope
  constructors (`crates/buzz-auth/src/scope.rs:205-248`).
- **Security-rationale comments inside tests** explaining what the assertion
  protects (`crates/buzz-auth/src/nip98.rs:290-294`,
  `crates/buzz-auth/src/rate_limit.rs:277-278`,
  `crates/buzz-auth/src/access.rs:227-230`).
- Test-only `unwrap`/`expect` is used freely; no test-helper crate, no
  `proptest`/`quickcheck`, no `mockall` — hand-written doubles only.
