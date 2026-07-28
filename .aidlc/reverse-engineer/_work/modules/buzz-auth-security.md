## Module: buzz-auth (`crates/buzz-auth`)

### Security

### Signature verification

Both auth paths delegate to one primitive: `buzz_core::verify_event`
(`crates/buzz-core/src/verification.rs:11-32`), which does two things in order:

1. `event.verify_id()` — recomputes the event id from
   `(pubkey, created_at, kind, tags, content)` and compares
   (`crates/buzz-core/src/verification.rs:13-25`). This is what binds the tags
   (challenge, relay, `u`, method, payload) to the signature.
2. `event.verify_signature()` — Schnorr verification
   (`crates/buzz-core/src/verification.rs:27-29`).

Call sites: `crates/buzz-auth/src/nip42.rs:56` and
`crates/buzz-auth/src/nip98.rs:74`.

| Property | State | file:line |
|----------|-------|-----------|
| Kind checked before signature | yes, NIP-42 (`kind != Authentication` → reject) | `crates/buzz-auth/src/nip42.rs:52-54` |
| Kind checked before signature | yes, NIP-98 (`kind != HttpAuth` → reject) | `crates/buzz-auth/src/nip98.rs:66-71` |
| Signature checked before tag comparisons | yes, both paths | `crates/buzz-auth/src/nip42.rs:56` vs `:58-83`; `crates/buzz-auth/src/nip98.rs:74` vs `:89-127` |
| CPU-bound verify offloaded | NIP-42 yes (`spawn_blocking` in `AuthService::verify_auth_event`) | `crates/buzz-auth/src/lib.rs:128-132` |
| CPU-bound verify offloaded | NIP-98 **no** — `verify_nip98_event` is a sync fn and this crate never wraps it | `crates/buzz-auth/src/nip98.rs:55`; the `spawn_blocking` requirement is documented upstream at `crates/buzz-core/src/verification.rs:1-2` |
| Error detail on failure | NIP-42 collapses everything to `InvalidSignature` (no oracle) | `crates/buzz-auth/src/nip42.rs:53`, `:56` |
| Error detail on failure | NIP-98 returns descriptive strings including the offending URL/method values | `crates/buzz-auth/src/nip98.rs:98-100`, `:111-113` — doc warns not to forward verbatim to clients (`:53-54`) |

---

### Replay protection

| Path | Replay protection in this crate | file:line |
|------|--------------------------------|-----------|
| NIP-42 | Single-use is achieved indirectly: the challenge is compared against a per-connection value the relay holds, and the relay refuses AUTH once the connection is `Authenticated` or `Failed`. This crate only performs the equality check | `crates/buzz-auth/src/nip42.rs:64-66`; state machine at `crates/buzz-relay/src/handlers/auth.rs:45-68` |
| NIP-98 | **Not** performed by the verifier. `verify_nip98_event` has no seen-set; the module doc states this explicitly: "It does **not** check whether the same event id has already been used" | `crates/buzz-auth/src/nip98_replay.rs:4-8` |

The replay contract is expressed as the `Nip98ReplayGuard` trait
(`crates/buzz-auth/src/nip98_replay.rs:64-104`) with these security-relevant
obligations:

- **Verify-then-mark ordering.** Marking before verifying would let an attacker
  who can predict a victim's future event id burn the slot and DoS the legitimate
  request (`crates/buzz-auth/src/nip98_replay.rs:19-27`).
- **Fail closed on error.** "On `Err` (Redis unreachable, etc.) callers MUST fail
  closed" (`crates/buzz-auth/src/nip98_replay.rs:83-86`).
- **Atomic set-if-absent.** "a read-then-write sequence loses to concurrent
  inserts and forfeits the freshness proof"
  (`crates/buzz-auth/src/nip98_replay.rs:88-90`).
- **TTL bounds.** floor 120s = 2× the ±60s tolerance
  (`crates/buzz-auth/src/nip98_replay.rs:46`), ceiling 3600s to stay inside Redis
  `EX`'s i64 range (`crates/buzz-auth/src/nip98_replay.rs:59`).
- **No process-local caching** — with multiple relay pods an in-process cache does
  not carry freshness across pods (`crates/buzz-auth/src/nip98_replay.rs:6-10`).

The production implementation is `RedisNip98ReplayGuard`
(`crates/buzz-pubsub/src/nip98_replay.rs:34`), held by the relay as
`Arc<dyn Nip98ReplayGuard>` (`crates/buzz-relay/src/state.rs:582`) and invoked on
the HTTP bridge paths (`crates/buzz-relay/src/api/bridge.rs:766`, `:956`,
`:1387`). This crate itself ships only `AlwaysFreshReplayGuard`
(`crates/buzz-auth/src/nip98_replay.rs:126-139`), cfg-gated to tests.

---

### Timestamp tolerance

| Path | Constant | Value | Comparison | file:line |
|------|----------|-------|-----------|-----------|
| NIP-42 | `TIMESTAMP_TOLERANCE_SECS` | 60 | `now.abs_diff(created_at) > 60` → reject | const `crates/buzz-auth/src/nip42.rs:35`, check `:78-83` |
| NIP-98 | `TIMESTAMP_TOLERANCE_SECS` | 60 | same | const `crates/buzz-auth/src/nip98.rs:32`, check `:78-85` |

`abs_diff` makes both windows symmetric, so future-dated events are rejected too.
The two constants are separate private definitions with the same name and value —
drifting one would silently desynchronise the pair, and the NIP-98 replay TTL
floor (`DEFAULT_REPLAY_TTL_SECS = 120`) is derived from the NIP-98 value by hand
(`crates/buzz-auth/src/nip98_replay.rs:42-45`). The 120-vs-tolerance relationship
is asserted only against the literal `120`, not against
`nip98::TIMESTAMP_TOLERANCE_SECS` (`crates/buzz-auth/src/nip98_replay.rs:210-218`).

---

### Constant-time comparisons

None. Every security-relevant comparison in the crate is a short-circuiting
equality:

| Comparison | Operator | file:line |
|-----------|----------|-----------|
| Challenge vs expected challenge | `challenge != expected_challenge` (`&str` `PartialEq`) | `crates/buzz-auth/src/nip42.rs:64` |
| Relay URL | `normalize_relay_url(a) != normalize_relay_url(b)` (`String` `PartialEq`) | `crates/buzz-auth/src/nip42.rs:74` |
| NIP-98 URL | `normalize_url(a) != normalize_url(b)` | `crates/buzz-auth/src/nip98.rs:97` |
| NIP-98 method | `eq_ignore_ascii_case` | `crates/buzz-auth/src/nip98.rs:110` |
| NIP-98 payload hash | `computed_hex != payload_hex` (hex `String` compare) | `crates/buzz-auth/src/nip98.rs:122` |

The `subtle` crate is in the workspace (`Cargo.toml:72` of `buzz-relay` uses it)
but is **not** a dependency of `buzz-auth`
(`crates/buzz-auth/Cargo.toml:14-26`). Practical exposure is limited: the
challenge is a 256-bit CSPRNG value the attacker must match exactly, and the
payload hash is verified against an already-signature-bound tag, so a timing
oracle on either yields little. It is nonetheless a deviation from
constant-time-compare hygiene and is not documented as a considered tradeoff
anywhere in the crate.

---

### Dev-only key derivation and exactly what gates it

```rust
#[cfg(any(test, feature = "dev"))]                       // crates/buzz-auth/src/lib.rs:159
pub fn derive_pubkey_from_username(username: &str) -> Result<nostr::PublicKey, AuthError> {
    let seed = format!("buzz-test-key:{username}");      // crates/buzz-auth/src/lib.rs:162
    let hash: [u8; 32] = Sha256::digest(seed.as_bytes()).into();   // :163
    let secret_key = nostr::SecretKey::from_slice(&hash) ... ;      // :164
    Ok(nostr::Keys::new(secret_key).public_key())                   // :166
}
```

The derivation is `SHA-256("buzz-test-key:{username}")` used directly as secret
key material. The doc states the risk plainly: "The derived keys are
deterministic and predictable from the username alone. Any attacker who knows a
username can compute the corresponding private key"
(`crates/buzz-auth/src/lib.rs:157-158`).

Gate chain, verified end to end:

| Link | State | file:line |
|------|-------|-----------|
| The `dev` cargo feature exists in `buzz-auth` and is **not** in any default set (there is no `[features] default = ...`) | confirmed | `crates/buzz-auth/Cargo.toml:10-12` |
| `buzz-relay` declares a passthrough feature `dev = ["buzz-auth/dev"]`, also not default | confirmed | `crates/buzz-relay/Cargo.toml:83-84` |
| `buzz-relay`'s **normal** dependency on `buzz-auth` requests no features | confirmed | `crates/buzz-relay/Cargo.toml:22` |
| `buzz-relay`'s **`[dev-dependencies]`** entry requests `features = ["dev"]` | confirmed | `crates/buzz-relay/Cargo.toml:90` |
| Workspace uses `resolver = "2"`, under which dev-dependency features are not unified into a non-test build | confirmed | `Cargo.toml:32` |
| Production container build passes no feature flags | confirmed | `Dockerfile:67-69` (`cargo build --release --locked -p buzz-relay --bin buzz-relay ...`) |
| Any `--features dev` / `--features buzz-auth/dev` invocation in `justfile`, CI, or Dockerfiles | none found (repo-wide grep) | — |

Conclusion: `derive_pubkey_from_username` is **not** compiled into the production
relay binary. It is compiled when building `buzz-relay`'s tests (via the
dev-dependency feature at `crates/buzz-relay/Cargo.toml:90`) and when building
`buzz-auth`'s own tests. Residual risk: the `dev` feature is one CLI flag away
from a release build, and the dev-dependency declaration means the code path is
compiled and linked in every `cargo test` run. Also note the function has **no
caller anywhere in the repo** — a repo-wide grep for
`derive_pubkey_from_username` matches only its definition
(`crates/buzz-auth/src/lib.rs:160`), so the risk is currently theoretical.

---

### Scope escalation surface

| Surface | Assessment | file:line |
|---------|-----------|-----------|
| NIP-42 grants all 16 known scopes to any pubkey that completes the handshake, including `AdminChannels` and `AdminUsers` | This is the widest grant in the crate. There is no tier, allowlist, or role input to the decision — only a valid signature over the right challenge | `crates/buzz-auth/src/lib.rs:136-142`; list `crates/buzz-auth/src/scope.rs:69-86` |
| Compensating control | Stated to be NIP-29 membership checks in the relay, not scopes: "Per-channel access is enforced by the relay's membership checks (NIP-29)" | `crates/buzz-auth/src/lib.rs:134-135` |
| `Scope::all_non_admin()` (14 scopes, admin excluded) exists and is documented as the dev-mode grant | never called anywhere in the repo — the admin-excluding path is dead | `crates/buzz-auth/src/scope.rs:94-111` |
| `Scope::Unknown(s)` | cannot satisfy a known-variant requirement (exact `PartialEq`), so an attacker-supplied scope string grants nothing | `crates/buzz-auth/src/scope.rs:60`, `:164`; check `crates/buzz-auth/src/access.rs:61` |
| Scope hierarchy | none — no wildcard, no implication. `admin:channels` does not imply `channels:write` | enumerated grant lists only, `crates/buzz-auth/src/scope.rs:69-111` |
| `AuthContext` mutability | all fields are `pub` with no invariant enforcement, so any consumer holding an owned/`mut` context can widen `scopes` or set `agent_owner_pubkey` | `crates/buzz-auth/src/lib.rs:64-80`; the relay does exactly this (`let Ok(mut auth_ctx)`, `crates/buzz-relay/src/handlers/auth.rs:91`) |
| `channel_ids` restriction field | declared but never enforced by any code in this crate; always `None` | `crates/buzz-auth/src/lib.rs:69-72`, `:139` |

---

### Input-validation gaps

| Gap | Detail | file:line |
|-----|--------|-----------|
| No length bound on `event_json` | `serde_json::from_str` runs on whatever the caller passes; memory bound is the caller's HTTP body limit | `crates/buzz-auth/src/nip98.rs:62` |
| No length bound on `username` in the dev derivation | unbounded `format!` then hash; low impact | `crates/buzz-auth/src/lib.rs:162` |
| `payload` tag not required | a request whose body is not covered by a `payload` tag still authenticates, so the signed claim binds URL+method only. Presence enforcement is delegated to the caller | `crates/buzz-auth/src/nip98.rs:117-127`; caller-side `require_payload` at `crates/buzz-relay/src/api/bridge.rs:99-112` |
| `payload` hex not format-validated | compared as a raw string; a non-hex or wrong-length tag simply fails the equality | `crates/buzz-auth/src/nip98.rs:122` |
| Unparseable URL falls back to string compare | NIP-42 compares raw strings verbatim (`crates/buzz-auth/src/nip42.rs:22`); NIP-98 lowercases first (`crates/buzz-auth/src/nip98.rs:148`). Both are fail-closed (mismatch → reject) but the two fallbacks differ |
| Scheme not normalised in NIP-42 | `ws://` and `wss://` are distinct strings; a downgrade cannot pass as an upgrade, so this is fail-closed, but it also means a relay that changes scheme rejects otherwise-valid AUTH | `crates/buzz-auth/src/nip42.rs:19-32` |
| Loopback aliasing asymmetry | NIP-42 collapses `localhost`/`::1` → `127.0.0.1` (`crates/buzz-auth/src/nip42.rs:25-29`); NIP-98 deliberately does **not**, because the `u`-tag host is the community binding (`crates/buzz-auth/src/nip98.rs:138-144`). The same asymmetry means the NIP-42 relay-tag check is a weaker host binding than NIP-98's under multi-tenant. Not flagged in the NIP-42 docs |
| `expected_url` trust | the verifier trusts whatever URL the caller reconstructs; the doc pushes reverse-proxy header handling (`X-Forwarded-Proto`/`X-Forwarded-Host`) onto the caller | `crates/buzz-auth/src/nip98.rs:40-41` |
| No rejection of duplicate tags | `tags.find(...)` takes the first match; a second `challenge`/`u`/`method` tag is ignored rather than rejected | `crates/buzz-auth/src/nip42.rs:58-60`, `:68-70`; `crates/buzz-auth/src/nip98.rs:89-94`, `:104-107` |

---

### Unsafe code

None. `#![deny(unsafe_code)]` is declared at `crates/buzz-auth/src/lib.rs:1`, and a
grep of the entire `crates/buzz-auth/` directory for the token `unsafe` returns
exactly that one line — no `unsafe` blocks, fns, traits, or impls, and no
`#[allow(unsafe_code)]` escape hatch.

---

### Randomness

`generate_challenge` draws 32 bytes from `rand::random::<[u8; 32]>()`
(`crates/buzz-auth/src/nip42.rs:39`) — 256 bits, hex-encoded to 64 chars, pinned
by a uniqueness + charset test (`crates/buzz-auth/src/nip42.rs:102-109`). No
seeded or deterministic RNG path exists in the crate.

---

### Rate limiting — verified state

**Verdict: `ARCHITECTURE.md` §9 item 2 is stale and materially wrong.** A
Redis-backed `RateLimiter` implementation exists, is constructed
unconditionally at relay startup, and is called before work is admitted on both
the WebSocket and HTTP surfaces in ordinary (non-feature-gated) production
builds. The `.env.example` heading "Shared Redis-backed admission limits"
(`.env.example:58`) is the accurate description.

#### Every `RateLimiter` implementor in the repo

Search: `impl .*RateLimiter` across all `*.rs` (including
`desktop/src-tauri/`), excluding `target/`. Three results, no more:

| Implementor | Location | Gate | Backing store |
|-------------|----------|------|---------------|
| `RedisRateLimiter` | `crates/buzz-pubsub/src/rate_limiter.rs:99-121` (struct `:88-90`) | **none — always compiled** | Redis via `deadpool_redis::Pool` (`crates/buzz-pubsub/src/rate_limiter.rs:89`) |
| `AlwaysAllowRateLimiter` | `crates/buzz-auth/src/rate_limit.rs:221-242` | `#[cfg(any(test, feature = "test-utils"))]` (`:218`, `:221`) | none (always allows) |
| `StubLimiter` | `crates/buzz-relay/src/admission.rs:69-96` | inside `#[cfg(test)] mod tests` (`crates/buzz-relay/src/admission.rs:47-48`) | none (test double) |

`desktop/src-tauri/` contains no `RateLimiter` implementor.

#### The Redis-backed limiter is real

`RedisRateLimiter::check_and_increment` builds the community-scoped key via
`buzz_auth::rate_limit::rate_limit_key`
(`crates/buzz-pubsub/src/rate_limiter.rs:108`) and runs an atomic Lua script that
`INCR`s, sets `EXPIRE` on first increment, and reads back the TTL
(`crates/buzz-pubsub/src/rate_limiter.rs:24-31`, invoked at `:50-56`).
Allow/deny boundary is `count <= limit`
(`crates/buzz-pubsub/src/rate_limiter.rs:74-78`). A missing-TTL key (crash
between INCR and EXPIRE in an older non-atomic version) is repaired with a fresh
`EXPIRE` and a warning (`crates/buzz-pubsub/src/rate_limiter.rs:58-72`).
`check_ip_connection` uses `ip_rate_limit_key`
(`crates/buzz-pubsub/src/rate_limiter.rs:118`).

#### It is wired into the relay

| Wiring step | file:line |
|-------------|-----------|
| `AppState` field `admission_rate_limiter: Arc<RedisRateLimiter>`, doc "Shared Redis-backed admission limits for ordinary HTTP and WebSocket work" | `crates/buzz-relay/src/state.rs:583-584` |
| Constructed at state build: `Arc::new(RedisRateLimiter::new(redis_pool.clone()))` — no cfg, no feature flag, no `Option` | `crates/buzz-relay/src/state.rs:712`, stored `:772` |
| Shared enforcement helper `check_principal<L: RateLimiter>` — `Ok` + `allowed` → admit; `Ok` + denied → `AdmissionError::Exceeded { reset_in_secs }`; `Err` → `AdmissionError::Unavailable` (fail closed) | `crates/buzz-relay/src/admission.rs:17-38` |
| WS burst budget helper: per-second limit × 5s window | `crates/buzz-relay/src/admission.rs:9`, `:40-45` |

#### Enforcement points — a check *is* called before work is admitted

**WebSocket.** `handle_text_message` parses the frame and immediately calls
`enforce_ws_admission`; on `false` it returns without dispatching
(`crates/buzz-relay/src/connection.rs:498-500`).
`enforce_ws_admission` (`crates/buzz-relay/src/connection.rs:594-653`):

- applies to `EVENT`, `REQ`, and `COUNT` frames; all other frame types
  (including `AUTH` and `CLOSE`) bypass
  (`crates/buzz-relay/src/connection.rs:599-602`);
- bypasses unauthenticated connections (`_ => return true`,
  `crates/buzz-relay/src/connection.rs:604-610`);
- first check: `LimitType::WsEvents` with a 5s window and
  `human_ws_events_per_sec × 5` budget
  (`crates/buzz-relay/src/connection.rs:612-623`);
- second check, EVENT frames only: `LimitType::Messages`, 60s window, limit =
  `agent_standard_messages_per_min` if `agent_owner_pubkey.is_some()` else
  `human_messages_per_min` (`crates/buzz-relay/src/connection.rs:632-650`);
- rejection emits `CLOSED` for REQ or `NOTICE` otherwise plus a
  `buzz_admission_rejections_total` metric
  (`crates/buzz-relay/src/connection.rs:655-679`).

**HTTP.** `enforce_http_admission` uses `LimitType::ApiCalls`, 60s window, limit
`human_api_calls_per_min`; denial → HTTP 429, limiter error → HTTP 503
(`crates/buzz-relay/src/api/bridge.rs:24-56`). Called on all three Nostr bridge
endpoints, before body parse:

| Endpoint | Call site |
|----------|-----------|
| `POST /events` (`submit_event_authed`) | `crates/buzz-relay/src/api/bridge.rs:760` |
| `POST /query` (`query_events_authed`) | `crates/buzz-relay/src/api/bridge.rs:955` |
| `POST /count` (`count_events_authed`) | `crates/buzz-relay/src/api/bridge.rs:1386` |

**Media upload.** Not covered by the Redis limiter. It uses a separate
in-process `DashMap` fixed-window counter, `upload_rate_limited`
(`crates/buzz-relay/src/api/media.rs:88-111`), over a 60s window
(`crates/buzz-relay/src/api/media.rs:66`) against `config.media_uploads_per_minute`
(`crates/buzz-relay/src/api/media.rs:95`), plus a concurrency permit
(`crates/buzz-relay/src/api/media.rs:113-137`). State field at
`crates/buzz-relay/src/state.rs:590-592`. Per-pod, not shared.

**Not enforced:** `LimitType::IpConnections` / `check_ip_connection`. The Redis
implementation exists (`crates/buzz-pubsub/src/rate_limiter.rs:112-120`) but the
only call sites repo-wide are the trait definition, the test stubs, and doc
references — no production path invokes it, so there is no per-IP connection cap.

#### Env var → consumer trace (all seven)

Every var is read by `rate_limit_config_from_env`
(`crates/buzz-relay/src/config.rs:284-316`) via `positive_u64_from_env`
(`crates/buzz-relay/src/config.rs:270-282`), which rejects zero and non-integers
with `ConfigError::InvalidValue` and falls back to the `RateLimitConfig::default()`
value when unset. The result lands in `config.auth.rate_limits`
(`crates/buzz-relay/src/config.rs:585`).

| Env var | Read at | Field | Consumed at | Enforced? |
|---------|---------|-------|-------------|-----------|
| `BUZZ_RATE_LIMIT_HUMAN_MESSAGES_PER_MIN` | `config.rs:287-290` | `human_messages_per_min` | `crates/buzz-relay/src/connection.rs:636` | yes — WS EVENT |
| `BUZZ_RATE_LIMIT_HUMAN_API_CALLS_PER_MIN` | `config.rs:291-294` | `human_api_calls_per_min` | `crates/buzz-relay/src/api/bridge.rs:29` | yes — HTTP `/events`, `/query`, `/count` |
| `BUZZ_RATE_LIMIT_HUMAN_WS_EVENTS_PER_SEC` | `config.rs:295-298` | `human_ws_events_per_sec` | `crates/buzz-relay/src/connection.rs:614` | yes — WS EVENT/REQ/COUNT |
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_MESSAGES_PER_MIN` | `config.rs:299-302` | `agent_standard_messages_per_min` | `crates/buzz-relay/src/connection.rs:634` | yes — WS EVENT from NIP-OA agents |
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_API_CALLS_PER_MIN` | `config.rs:303-306` | `agent_standard_api_calls_per_min` | **nowhere** | no |
| `BUZZ_RATE_LIMIT_AGENT_ELEVATED_MESSAGES_PER_MIN` | `config.rs:307-310` | `agent_elevated_messages_per_min` | **nowhere** | no |
| `BUZZ_RATE_LIMIT_AGENT_PLATFORM_MESSAGES_PER_MIN` | `config.rs:311-314` | `agent_platform_messages_per_min` | **nowhere** | no |

Repo-wide greps for the last three field names return only the `buzz-auth`
declaration/default and the `buzz-relay` config parser — no read site. CI sets
three of them to large values to keep integration tests from tripping the limits
(`.github/workflows/ci.yml:492-494`), which is itself evidence that enforcement
is live: a no-op limiter would not need raising.

#### Point-by-point scoring of the `ARCHITECTURE.md` claim

Claim text is at `ARCHITECTURE.md:823` (plus the same assertion at
`ARCHITECTURE.md:390`).

| Claim element | Verdict | Evidence |
|---------------|---------|----------|
| "`RateLimiter` trait exists in `buzz-auth`" | **TRUE** | `crates/buzz-auth/src/rate_limit.rs:168-194` |
| "Only implementation is `AlwaysAllowRateLimiter` (test stub…)" | **FALSE** | `RedisRateLimiter` at `crates/buzz-pubsub/src/rate_limiter.rs:99`, ungated |
| "No Redis-backed rate limiter exists anywhere in the codebase" (`ARCHITECTURE.md:390`) | **FALSE** | `crates/buzz-pubsub/src/rate_limiter.rs:24-121` |
| "`RateLimitConfig` defines 4 tiers" | **TRUE** | `crates/buzz-auth/src/rate_limit.rs:86-108` |
| "…but none are enforced" | **FALSE for 2 of 4 tiers** — human and agent-standard are enforced (`crates/buzz-relay/src/connection.rs:614`, `:634`, `:636`; `crates/buzz-relay/src/api/bridge.rs:29`); agent-elevated and agent-platform are genuinely unenforced | see the table above |
| "rate limiting is not currently enforced" (`ARCHITECTURE.md:390`) | **FALSE** | `crates/buzz-relay/src/connection.rs:498-500` and `crates/buzz-relay/src/api/bridge.rs:760`, `:955`, `:1386` |
| "`buzz-pubsub` … Does NOT implement the rate limiter" (`ARCHITECTURE.md:460`) | **FALSE** | `crates/buzz-pubsub/src/rate_limiter.rs:99` |

Residual truth worth preserving from the doc: the two agent tiers above
`agent-standard` are still design-only, per-IP connection limits are still
unenforced, and the algorithm is a fixed window with 2× boundary burst
(`crates/buzz-auth/src/rate_limit.rs:165-167`,
`crates/buzz-pubsub/src/rate_limiter.rs:7-8`,
`crates/buzz-relay/src/admission.rs:6-8`).

---

### Secondary verification results

| # | Claim | Verdict | Evidence |
|---|-------|---------|----------|
| a | "NIP-42 grants `Scope::all_known()` (all 14 scopes)" | **PARTIAL** — the function is right, the count is wrong. `verify_auth_event` sets `scopes: Scope::all_known()` (`crates/buzz-auth/src/lib.rs:138`), and `all_known()` returns **16** variants (`crates/buzz-auth/src/scope.rs:68-87`, pinned by `assert_eq!(all.len(), 16)` at `:237`). 14 is the cardinality of `all_non_admin()` (`crates/buzz-auth/src/scope.rs:94-111`, `assert_eq!(scopes.len(), 14)` at `:208`), which no auth path calls |
| b | "NIP-42 timestamp tolerance: ±60 seconds" | **TRUE** — `TIMESTAMP_TOLERANCE_SECS = 60` (`crates/buzz-auth/src/nip42.rs:35`), symmetric via `abs_diff` (`:80-83`); test rejects a 120s-old event (`:143-157`) |
| c | "NIP-98 auth: Schnorr-signed `kind:27235` events with URL + method tags" | **TRUE** — kind gate `Kind::HttpAuth` (`crates/buzz-auth/src/nip98.rs:66`), Schnorr via `buzz_core::verify_event` (`:74`), single-letter `u` tag (`:89-95`), `method` tag (`:104-108`). Optional `payload` body-hash tag adds a third check (`:117-127`) |
| d | "Dev-only key derivation: `SHA-256(\"buzz-test-key:{username}\")` gated behind `#[cfg(any(test, feature = \"dev\"))]`" | **TRUE**, and `buzz-relay` does **not** enable the feature in non-dev builds. Derivation at `crates/buzz-auth/src/lib.rs:162-166`, gate at `:159`. `buzz-relay`'s normal dep requests no features (`crates/buzz-relay/Cargo.toml:22`); `dev = ["buzz-auth/dev"]` is an opt-in feature (`:84`); the only `features = ["dev"]` request is in `[dev-dependencies]` (`:90`), which under `resolver = "2"` (`Cargo.toml:32`) does not affect a non-test build; `Dockerfile:67` passes no feature flags |
| e | "`buzz-auth` implements `ChannelAccessChecker` itself or only defines the trait" | **Defines the trait; the only impl in the crate is test-gated.** Trait at `crates/buzz-auth/src/access.rs:31-57`; sole implementor `MockAccessChecker` under `#[cfg(any(test, feature = "test-utils"))]` (`crates/buzz-auth/src/access.rs:135-151`). No implementor exists anywhere else in the repo — the trait doc's claim that `buzz-db` implements it (`crates/buzz-auth/src/access.rs:18-19`) is unsupported by code |
