## Module: buzz-auth (`crates/buzz-auth`)

### Features / Capabilities

Completeness legend: **full** = implemented and exercised by tests; **partial** =
implemented but with a documented or observable gap; **interface-only** = trait or
type defined here, behaviour lives elsewhere; **stubbed** = test-only
implementation.

| # | Capability | Completeness | Evidence | Notes |
|---|-----------|--------------|----------|-------|
| 1 | NIP-42 challenge generation | full | `crates/buzz-auth/src/nip42.rs:38-41` | 32 CSPRNG bytes via `rand::random`, hex-encoded; test pins 64 chars + uniqueness (`:102-109`) |
| 2 | NIP-42 AUTH event verification (kind, sig, challenge, relay, timestamp) | full | `crates/buzz-auth/src/nip42.rs:47-86` | 5 rejection tests + 2 normalisation tests |
| 3 | Relay-URL normalisation for AUTH | full | `crates/buzz-auth/src/nip42.rs:19-33` | loopback aliasing + trailing-slash stripping; `ws` vs `wss` not aliased |
| 4 | `AuthService` async wrapper with `spawn_blocking` offload | full | `crates/buzz-auth/src/lib.rs:118-143` | 3 async tests (`:199-242`) |
| 5 | NIP-98 HTTP auth verification (8-step) | full | `crates/buzz-auth/src/nip98.rs:55-131` | 11 tests including payload-hash and loopback-distinctness |
| 6 | NIP-98 body-hash binding (`payload` tag) | partial | `crates/buzz-auth/src/nip98.rs:117-127` | only enforced when the tag **and** a body are both present; presence is the caller's responsibility (`crates/buzz-relay/src/api/bridge.rs:99-112`) |
| 7 | NIP-98 replay protection | interface-only | trait `crates/buzz-auth/src/nip98_replay.rs:64-104`; keys `:114-121`; constants `:46`, `:59` | production Redis impl is `crates/buzz-pubsub/src/nip98_replay.rs:34`; this crate ships only `AlwaysFreshReplayGuard` (`:126-139`) |
| 8 | Scope model + wire-format round-trip | full | `crates/buzz-auth/src/scope.rs:16-178` | 16 known variants + `Unknown` passthrough; 5 tests |
| 9 | Scope granting on NIP-42 | full (but coarse) | `crates/buzz-auth/src/lib.rs:136-142` | grants all 16 scopes unconditionally; no per-connection differentiation |
| 10 | Scope checking (`require_scope`, `has_scope`) | full | `crates/buzz-auth/src/access.rs:60-69`, `crates/buzz-auth/src/lib.rs:84-86` | exact-match only; no hierarchy/wildcards. `require_scope` has no external caller |
| 11 | Channel access checking | interface-only | trait `crates/buzz-auth/src/access.rs:31-57` | **no production implementor anywhere in the repo** (see debt aspect); only `MockAccessChecker` (`:135-151`) |
| 12 | Combined scope+membership helpers | full but unused | `crates/buzz-auth/src/access.rs:72-101` | 4 tests; zero callers outside this crate |
| 13 | Rate limiting | interface-only | trait `crates/buzz-auth/src/rate_limit.rs:168-194` | production Redis impl is `crates/buzz-pubsub/src/rate_limiter.rs:99-121`, wired in `buzz-relay` — see security aspect verdict |
| 14 | Rate-limit tier configuration | full (parse) / partial (consumption) | `crates/buzz-auth/src/rate_limit.rs:86-144` | all 7 fields deserialise with defaults; only 4 are read by any consumer (see configuration aspect) |
| 15 | Rate-limit / replay Redis key construction | full | `crates/buzz-auth/src/rate_limit.rs:201-215`, `crates/buzz-auth/src/nip98_replay.rs:114-121` | community-prefixed for pubkey/event keys, global for IP; lowercase invariant tested |
| 16 | Test doubles (`MockAccessChecker`, `AlwaysAllowRateLimiter`, `AlwaysFreshReplayGuard`) | stubbed by design | `crates/buzz-auth/src/access.rs:108`, `crates/buzz-auth/src/rate_limit.rs:219`, `crates/buzz-auth/src/nip98_replay.rs:127` | all `#[cfg(any(test, feature = "test-utils"))]`; none referenced outside this crate |
| 17 | Dev-only username→pubkey derivation | full but orphaned | `crates/buzz-auth/src/lib.rs:159-167` | `#[cfg(any(test, feature = "dev"))]`; no caller in the repo, and no test either |
| 18 | Error taxonomy | full | `crates/buzz-auth/src/error.rs:9-59` | 10 variants; 2 (`Nip98Replay`, `PubkeyMismatch`) are never constructed in this crate |

---

### Capabilities explicitly absent

| Absent capability | Evidence |
|-------------------|----------|
| JWT / OAuth token validation, token minting, refresh | crate doc "No JWT validation, no token management, no IdP runtime dependency" (`crates/buzz-auth/src/lib.rs:16`, `:98`); no such types in any of the 8 source files |
| Session or credential persistence | no `sqlx`, no DB dependency in `crates/buzz-auth/Cargo.toml:14-26` |
| Any network I/O | no HTTP/WS client dependency; see integrations aspect |
| Per-channel scope narrowing | `AuthContext.channel_ids` is documented "reserved for future per-channel access control" and always constructed as `None` (`crates/buzz-auth/src/lib.rs:69-72`, `:139`) |
| Constant-time comparison of challenge / payload hash | plain `!=` on `&str` (`crates/buzz-auth/src/nip42.rs:64`, `crates/buzz-auth/src/nip98.rs:122`); no `subtle` dependency |
| NIP-98 → `AuthContext` construction | `verify_nip98_event` returns a bare `PublicKey` (`crates/buzz-auth/src/nip98.rs:60`), contradicting the crate-doc invariant at `crates/buzz-auth/src/lib.rs:15` |

---

### TODO / FIXME / HACK / XXX comments

A case-insensitive search of the entire `crates/buzz-auth/` directory for
`TODO`, `FIXME`, `HACK`, and `XXX` returns **zero matches**. There are no
in-code work markers in this crate.

Deferred-work statements are instead written as prose in doc comments. The ones
that carry an explicit "not yet / future / v2 / deferred" claim, verbatim:

| Marker text (verbatim) | file:line |
|------------------------|-----------|
| `/// Channel restriction (reserved for future per-channel access control).` | `crates/buzz-auth/src/lib.rs:69` |
| `/// Reserved for future use. Not currently enforced by git HTTP routes —` / `/// those use NIP-98 auth directly. Will be enforced when collaborator` / `/// access (read-only, maintainer) is added in v2.` | `crates/buzz-auth/src/scope.rs:47-49` |
| `/// Enforced for kind:30617/30618 events via WebSocket ingest, but NOT` / `/// enforced by git HTTP push routes (which use NIP-98 + owner check).` / `/// Full enforcement deferred to v2 collaborator model.` | `crates/buzz-auth/src/scope.rs:53-55` |
| `//! ⚠️ Fixed windows allow up to 2× burst at boundaries. Upgrade to sliding` / `//! window or token bucket for strict limiting.` | `crates/buzz-auth/src/rate_limit.rs:6-7` (repeated at `:165-167`) |
| `/// ⚠️ The fixed-window algorithm used by the Redis implementation allows up to 2×` / `/// burst at window boundaries. Upgrade to a sliding window or token bucket if strict` / `/// per-second limiting is required.` | `crates/buzz-auth/src/rate_limit.rs:165-167` |
| `/// If per-(community, IP) caps are ever needed as a tenant-fairness signal, that` / `/// belongs in an additive \`LimitType\` keyed on \`(community, ip)\`, not in this trait.` | `crates/buzz-auth/src/rate_limit.rs:161-163` |
| `/// # ⚠️ SECURITY — Dev/test only` … `/// and **must never be compiled into a production release build**.` | `crates/buzz-auth/src/lib.rs:152-155` |
| `// Set later by relay membership gate if NIP-OA` | `crates/buzz-auth/src/lib.rs:141` |
