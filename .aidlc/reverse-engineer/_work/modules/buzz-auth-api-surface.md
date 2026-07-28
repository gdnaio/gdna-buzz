## Module: buzz-auth (`crates/buzz-auth`)

### API Surface

Crate lints: `#![deny(unsafe_code)]` and `#![warn(missing_docs)]`
(`crates/buzz-auth/src/lib.rs:1-2`).

### Modules (all `pub`)

| Module | Declared | Contents |
|--------|----------|----------|
| `access` | `crates/buzz-auth/src/lib.rs:19` | `ChannelAccessChecker`, scope/access helpers, `MockAccessChecker` |
| `error` | `crates/buzz-auth/src/lib.rs:21` | `AuthError` |
| `nip42` | `crates/buzz-auth/src/lib.rs:23` | challenge gen + AUTH verification |
| `nip98` | `crates/buzz-auth/src/lib.rs:25` | HTTP auth verification |
| `nip98_replay` | `crates/buzz-auth/src/lib.rs:27` | `Nip98ReplayGuard`, keys, TTL constants |
| `rate_limit` | `crates/buzz-auth/src/lib.rs:29` | `RateLimiter`, config, keys |
| `scope` | `crates/buzz-auth/src/lib.rs:31` | `Scope`, `parse_scopes` |

### Root re-exports

| Re-export list | file:line |
|----------------|-----------|
| `access::{check_read_access, check_write_access, require_scope, ChannelAccessChecker}` | `crates/buzz-auth/src/lib.rs:33` |
| `error::AuthError` | `crates/buzz-auth/src/lib.rs:34` |
| `nip42::{generate_challenge, verify_nip42_event}` | `crates/buzz-auth/src/lib.rs:35` |
| `nip98::verify_nip98_event` | `crates/buzz-auth/src/lib.rs:36` |
| `nip98_replay::{nip98_replay_key, nip98_replay_key_for_scope, Nip98ReplayGuard, DEFAULT_REPLAY_TTL_SECS, MAX_REPLAY_TTL_SECS}` | `crates/buzz-auth/src/lib.rs:37-40` |
| `rate_limit::{ip_rate_limit_key, rate_limit_key, LimitType, RateLimitConfig, RateLimitResult, RateLimiter}` | `crates/buzz-auth/src/lib.rs:41-43` |
| `scope::{parse_scopes, Scope}` | `crates/buzz-auth/src/lib.rs:44` |
| `access::MockAccessChecker` — `#[cfg(any(test, feature = "test-utils"))]` | `crates/buzz-auth/src/lib.rs:46-47` |
| `nip98_replay::AlwaysFreshReplayGuard` — same cfg | `crates/buzz-auth/src/lib.rs:48-49` |
| `rate_limit::AlwaysAllowRateLimiter` — same cfg | `crates/buzz-auth/src/lib.rs:50-51` |

Not re-exported at root (module-path access only): `AuthMethod`, `AuthContext`,
`AuthConfig`, `AuthService` are defined directly in `lib.rs` so they are already
root items (`crates/buzz-auth/src/lib.rs:55`, `:64`, `:91`, `:100`).

---

### Public functions

| Signature | Returns | file:line |
|-----------|---------|-----------|
| `pub fn generate_challenge() -> String` | 32 CSPRNG bytes hex-encoded | `crates/buzz-auth/src/nip42.rs:38` |
| `pub fn verify_nip42_event(event: &Event, expected_challenge: &str, relay_url: &str) -> Result<(), AuthError>` | `()` on success | `crates/buzz-auth/src/nip42.rs:47-51` |
| `pub fn verify_nip98_event(event_json: &str, expected_url: &str, expected_method: &str, body: Option<&[u8]>) -> Result<nostr::PublicKey, AuthError>` | authenticated pubkey | `crates/buzz-auth/src/nip98.rs:55-60` |
| `pub fn require_scope(scopes: &[Scope], required: Scope) -> Result<(), AuthError>` | `()` or `InsufficientScope` | `crates/buzz-auth/src/access.rs:60` |
| `pub async fn check_read_access(checker: &impl ChannelAccessChecker, ctx: &TenantContext, pubkey: &PublicKey, channel_id: Uuid, scopes: &[Scope]) -> Result<(), AuthError>` | `()` or error | `crates/buzz-auth/src/access.rs:72-78` |
| `pub async fn check_write_access(checker: &impl ChannelAccessChecker, ctx: &TenantContext, pubkey: &PublicKey, channel_id: Uuid, scopes: &[Scope]) -> Result<(), AuthError>` | `()` or error | `crates/buzz-auth/src/access.rs:88-94` |
| `pub fn parse_scopes(raw: &[impl AsRef<str>]) -> Vec<Scope>` | never fails | `crates/buzz-auth/src/scope.rs:170` |
| `pub fn rate_limit_key(ctx: &TenantContext, pubkey: &PublicKey, limit_type: &LimitType) -> String` | Redis key | `crates/buzz-auth/src/rate_limit.rs:201` |
| `pub fn ip_rate_limit_key(ip: &IpAddr) -> String` | Redis key | `crates/buzz-auth/src/rate_limit.rs:213` |
| `pub fn nip98_replay_key(ctx: &TenantContext, event_id: &EventId) -> String` | Redis key | `crates/buzz-auth/src/nip98_replay.rs:114` |
| `pub fn nip98_replay_key_for_scope(scope: &str, event_id: &EventId) -> String` | Redis key | `crates/buzz-auth/src/nip98_replay.rs:119` |
| `pub fn derive_pubkey_from_username(username: &str) -> Result<nostr::PublicKey, AuthError>` — `#[cfg(any(test, feature = "dev"))]` | derived pubkey | `crates/buzz-auth/src/lib.rs:159-160` |

Private helpers: `normalize_relay_url(raw: &str) -> String`
(`crates/buzz-auth/src/nip42.rs:19`), `normalize_url(raw: &str) -> String`
(`crates/buzz-auth/src/nip98.rs:145`), and the seven `default_*` fns
(`crates/buzz-auth/src/rate_limit.rs:110-130`).

---

### Public inherent methods

| Type | Method | file:line |
|------|--------|-----------|
| `AuthContext` | `pub fn has_scope(&self, scope: &Scope) -> bool` | `crates/buzz-auth/src/lib.rs:84` |
| `AuthService` | `pub fn new(config: AuthConfig) -> Self` | `crates/buzz-auth/src/lib.rs:106` |
| `AuthService` | `pub fn config(&self) -> &AuthConfig` | `crates/buzz-auth/src/lib.rs:111` |
| `AuthService` | `pub async fn verify_auth_event(&self, auth_event: nostr::Event, expected_challenge: &str, relay_url: &str) -> Result<AuthContext, AuthError>` | `crates/buzz-auth/src/lib.rs:118-123` |
| `Scope` | `pub fn all_known() -> Vec<Scope>` (16 items) | `crates/buzz-auth/src/scope.rs:68` |
| `Scope` | `pub fn all_non_admin() -> Vec<Scope>` (14 items) | `crates/buzz-auth/src/scope.rs:94` |
| `Scope` | `pub fn as_str(&self) -> &str` | `crates/buzz-auth/src/scope.rs:114` |
| `LimitType` | `pub fn key_suffix(&self) -> &'static str` | `crates/buzz-auth/src/rate_limit.rs:71` |
| `RateLimitResult` | `pub fn allowed(current: u64, limit: u64, reset_in_secs: u64) -> Self` | `crates/buzz-auth/src/rate_limit.rs:32` |
| `RateLimitResult` | `pub fn denied(current: u64, limit: u64, reset_in_secs: u64) -> Self` | `crates/buzz-auth/src/rate_limit.rs:42` |
| `MockAccessChecker` | `pub fn new() -> Self` (cfg-gated) | `crates/buzz-auth/src/access.rs:115` |
| `MockAccessChecker` | `pub fn allow(&mut self, ctx: &TenantContext, pubkey: &PublicKey, channel_id: Uuid)` (cfg-gated) | `crates/buzz-auth/src/access.rs:122` |

Derived/blanket trait impls: `Display for Scope`
(`crates/buzz-auth/src/scope.rs:137`), `FromStr for Scope` with
`Err = Infallible` (`crates/buzz-auth/src/scope.rs:143-144`), `Default for
RateLimitConfig` (`crates/buzz-auth/src/rate_limit.rs:132`), `Default for
MockAccessChecker` (`crates/buzz-auth/src/access.rs:129`).

---

### Trait: `ChannelAccessChecker` (`crates/buzz-auth/src/access.rs:31-57`)

Supertraits: `Send + Sync`. Uses RPITIT (return-position `impl Future`), so the
trait is **not** dyn-compatible.

```rust
pub trait ChannelAccessChecker: Send + Sync {
    fn accessible_channel_ids(
        &self,
        ctx: &TenantContext,
        pubkey: &PublicKey,
    ) -> impl Future<Output = Result<HashSet<Uuid>, AuthError>> + Send;   // access.rs:35-39

    fn can_access(
        &self,
        ctx: &TenantContext,
        pubkey: &PublicKey,
        channel_id: Uuid,
    ) -> impl Future<Output = Result<bool, AuthError>> + Send { /* default */ } // access.rs:46-56
}
```

`can_access` has a default body that calls `accessible_channel_ids` and does a
`HashSet::contains` (`crates/buzz-auth/src/access.rs:52-55`). Only
`accessible_channel_ids` is required.

Contract in the doc comment: every method takes `&TenantContext` and
"Implementations MUST scope every query by `ctx.community()`"
(`crates/buzz-auth/src/access.rs:22-30`).

**Implementors in this crate:** exactly one, and it is test-only.

| Implementor | Gate | file:line |
|-------------|------|-----------|
| `MockAccessChecker` | `#[cfg(any(test, feature = "test-utils"))]` | `crates/buzz-auth/src/access.rs:135-151` |

Repo-wide search for `impl ... ChannelAccessChecker` and for the identifier
`ChannelAccessChecker` outside this crate returns **no** matches — the trait's
doc claim that it is "Implemented by the database layer (`buzz-db`) in
production" (`crates/buzz-auth/src/access.rs:18-19`) is not backed by code.

---

### Trait: `RateLimiter` (`crates/buzz-auth/src/rate_limit.rs:168-194`)

Supertraits: `Send + Sync`. Also RPITIT — not dyn-compatible (consumers take it
as a generic bound; see `crates/buzz-relay/src/admission.rs:17`).

```rust
pub trait RateLimiter: Send + Sync {
    fn check_and_increment(
        &self,
        ctx: &TenantContext,
        pubkey: &PublicKey,
        limit_type: LimitType,
        window_secs: u64,
        limit: u64,
    ) -> impl std::future::Future<Output = Result<RateLimitResult, AuthError>> + Send; // rate_limit.rs:174-181

    fn check_ip_connection(
        &self,
        ip: &IpAddr,
        window_secs: u64,
        limit: u64,
    ) -> impl std::future::Future<Output = Result<RateLimitResult, AuthError>> + Send; // rate_limit.rs:188-193
}
```

No default bodies — both methods are required.

Documented scoping contract: pubkey-keyed limits are community-prefixed;
IP-keyed limits are deliberately operator-global and take no `TenantContext`
(`crates/buzz-auth/src/rate_limit.rs:151-163`). Documented algorithm caveat:
fixed windows permit up to 2× burst at boundaries
(`crates/buzz-auth/src/rate_limit.rs:165-167`, also `:6-7`).

**Implementors in this crate:** exactly one, test-only.

| Implementor | Gate | Behaviour | file:line |
|-------------|------|-----------|-----------|
| `AlwaysAllowRateLimiter` (unit struct) | `#[cfg(any(test, feature = "test-utils"))]` | both methods return `RateLimitResult::allowed(1, limit, window_secs)` | decl `crates/buzz-auth/src/rate_limit.rs:218-219`, impl `:221-242` |

Implementors elsewhere in the repo (see the security aspect for the full
verdict): `RedisRateLimiter` in `crates/buzz-pubsub/src/rate_limiter.rs:99` and a
test `StubLimiter` in `crates/buzz-relay/src/admission.rs:69`.

---

### Trait: `Nip98ReplayGuard` (`crates/buzz-auth/src/nip98_replay.rs:64-104`)

Supertraits: `Send + Sync`. Unlike the other two traits this one uses
`Pin<Box<dyn Future ...>>` returns, so it **is** dyn-compatible — the relay
stores it as `Arc<dyn Nip98ReplayGuard>`
(`crates/buzz-relay/src/state.rs:582`).

```rust
pub trait Nip98ReplayGuard: Send + Sync {
    fn try_mark_in_scope<'a>(
        &'a self,
        scope: &'a str,
        event_id: &'a EventId,
        ttl_secs: u64,
    ) -> Pin<Box<dyn Future<Output = Result<bool, AuthError>> + Send + 'a>>;  // nip98_replay.rs:66-71

    fn try_mark<'a>(
        &'a self,
        ctx: &'a TenantContext,
        event_id: &'a EventId,
        ttl_secs: u64,
    ) -> Pin<Box<dyn Future<Output = Result<bool, AuthError>> + Send + 'a>> { /* default */ } // nip98_replay.rs:97-103
}
```

`try_mark` default body derives the scope from `ctx.community().to_string()` and
delegates to `try_mark_in_scope` (`crates/buzz-auth/src/nip98_replay.rs:99-102`).

Documented obligations on implementors
(`crates/buzz-auth/src/nip98_replay.rs:73-96`): `Ok(true)` = newly inserted
(proceed), `Ok(false)` = replay (caller MUST reject); on `Err` callers MUST fail
closed; the operation MUST be atomic set-if-absent; `ttl_secs` MUST be clamped
up to `DEFAULT_REPLAY_TTL_SECS` and down to `MAX_REPLAY_TTL_SECS`.

**Implementors in this crate:** exactly one, test-only.

| Implementor | Gate | Behaviour | file:line |
|-------------|------|-----------|-----------|
| `AlwaysFreshReplayGuard` (unit struct) | `#[cfg(any(test, feature = "test-utils"))]` | `try_mark_in_scope` always `Ok(true)` | decl `crates/buzz-auth/src/nip98_replay.rs:126-127`, impl `:129-139` |

Production implementor lives outside the crate: `RedisNip98ReplayGuard`
(`crates/buzz-pubsub/src/nip98_replay.rs:34`).

---

### Error type

`pub enum AuthError` with 10 variants (`crates/buzz-auth/src/error.rs:9-59`) —
full table in the conventions aspect. It is the error type of every fallible
public fn in the crate.
