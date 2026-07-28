## Module: buzz-auth (`crates/buzz-auth`)

### Technical Debt

### Dead / orphaned public API

Determined by repo-wide grep for each identifier, excluding `target/` and this
crate's own `#[cfg(test)]` modules.

| Item | Declared | Callers outside this crate | Notes |
|------|----------|---------------------------|-------|
| `ChannelAccessChecker` (trait) | `crates/buzz-auth/src/access.rs:31` | **zero** — the identifier does not appear anywhere else in the repo | The doc claims "Implemented by the database layer (`buzz-db`) in production" (`crates/buzz-auth/src/access.rs:18-19`). No `buzz-db` type implements it. The trait's entire tenant-scoping contract (`:22-30`) is unenforced because nothing implements it |
| `check_read_access` | `crates/buzz-auth/src/access.rs:72` | zero | tested (`:208-223`) but unreachable in production |
| `check_write_access` | `crates/buzz-auth/src/access.rs:88` | zero | untested directly; only its scope constant differs from the read variant |
| `require_scope` | `crates/buzz-auth/src/access.rs:60` | zero | the only scope-enforcement function in the crate has no production caller |
| `parse_scopes` | `crates/buzz-auth/src/scope.rs:170` | zero | tested (`:199-203`); contains the crate's only production-path `expect` |
| `Scope::all_non_admin()` | `crates/buzz-auth/src/scope.rs:94` | zero | the admin-excluding grant set is dead; only `all_known()` is used (`crates/buzz-relay/src/api/bridge.rs:829`) |
| `derive_pubkey_from_username` | `crates/buzz-auth/src/lib.rs:160` | zero — no caller anywhere, not even a test | Doc says it exists so "the relay can resolve usernames to Nostr pubkeys in dev mode" (`:149-151`); nothing does |
| `ip_rate_limit_key` | `crates/buzz-auth/src/rate_limit.rs:213` | one, in a code path with no caller: `crates/buzz-pubsub/src/rate_limiter.rs:118` inside `check_ip_connection`, which is never invoked in production |
| `LimitType::IpConnections` | `crates/buzz-auth/src/rate_limit.rs:66` | zero constructions repo-wide | the variant, its key suffix (`:75`), and the whole `check_ip_connection` method are effectively dead |
| `AuthMethod::Nip98` | `crates/buzz-auth/src/lib.rs:59` | zero constructions | the relay's HTTP path uses its own `HttpAuthMethod` enum instead (`crates/buzz-relay/src/handlers/ingest.rs:2544`, `crates/buzz-relay/src/api/bridge.rs:830`), so this variant is unreachable |
| `AuthError::Nip98Replay` | `crates/buzz-auth/src/error.rs:37` | never constructed in this crate; intended for callers per the usage example at `crates/buzz-auth/src/nip98_replay.rs:24` |
| `AuthError::PubkeyMismatch` | `crates/buzz-auth/src/error.rs:41` | never constructed anywhere in the crate |
| `AuthContext.channel_ids` | `crates/buzz-auth/src/lib.rs:72` | always `None` at every construction site (`crates/buzz-auth/src/lib.rs:139`, `crates/buzz-relay/src/handlers/event.rs:1370`); no code reads it | self-documented as "reserved for future" (`:69`) |
| `MockAccessChecker`, `AlwaysAllowRateLimiter`, `AlwaysFreshReplayGuard` | `access.rs:108`, `rate_limit.rs:219`, `nip98_replay.rs:127` | zero external users | The `test-utils` feature that exposes them is never enabled by any crate. `buzz-relay` and `buzz-pubsub` hand-roll their own doubles instead (`crates/buzz-relay/src/admission.rs:64-96`, `crates/buzz-relay/src/api/bridge.rs:3266`, `crates/buzz-relay/src/api/invites.rs:419`, `crates/buzz-relay/src/api/operator.rs:521`) — including a locally-redefined `AlwaysFreshReplayGuard` in four separate files, duplicating what this crate already exports |

Roughly a third of the crate's public surface (the whole `access` module plus
`parse_scopes`, `all_non_admin`, the dev derivation, and the IP-limit path) has no
production consumer.

### Unused dependency

`tracing` is declared (`crates/buzz-auth/Cargo.toml:20`) but never used — no
`use tracing`, no `tracing::` macro call, no `#[instrument]` anywhere in
`crates/buzz-auth/src/`. `buzz-admin` also declares `buzz-auth`
(`crates/buzz-admin/Cargo.toml:17`) while referencing zero `buzz_auth` items.

---

### Documentation drift inside the crate

| Doc claim | Reality | file:line |
|-----------|---------|-----------|
| "All paths produce an [`AuthContext`] bound to the connection" | `verify_nip98_event` returns a bare `PublicKey`; no `AuthContext` is ever produced on the NIP-98 path | claim `crates/buzz-auth/src/lib.rs:15`; code `crates/buzz-auth/src/nip98.rs:60`, `:130` |
| `ChannelAccessChecker` "Implemented by the database layer (`buzz-db`) in production" | no implementor exists outside this crate | claim `crates/buzz-auth/src/access.rs:18-19` |
| `all_known()` doc: "Used in dev mode (`require_auth_token=false`) where `X-Pubkey` header auth grants unrestricted access" | it is the production NIP-42 grant set (`crates/buzz-auth/src/lib.rs:138`); the doc text is copy-pasted from `all_non_admin()` and describes the wrong caller | claim `crates/buzz-auth/src/scope.rs:66-67` vs `:91-93` |
| `RateLimiter` doc: "The Redis-backed production implementation lives in `buzz-relay` / `buzz-pubsub`" | it is specifically `buzz-pubsub`; `buzz-relay` holds the wiring, not the impl. Minor, but the ambiguity is what `ARCHITECTURE.md` got wrong | claim `crates/buzz-auth/src/rate_limit.rs:3-4`, `:148`; impl `crates/buzz-pubsub/src/rate_limiter.rs:99` |
| `Scope` doc: "Scopes are stored as `TEXT[]` in the database" | unverifiable from this crate (no DB dependency); no code here reads or writes scopes | claim `crates/buzz-auth/src/scope.rs:12-14` |
| `Scope::ReposRead` / `ReposWrite` enforcement notes | claims about `buzz-relay` git routes and WS ingest that cannot be verified from here; they date the code to a "v2 collaborator model" that does not exist yet | `crates/buzz-auth/src/scope.rs:46-56` |

Related external drift (documented in the security aspect): `ARCHITECTURE.md:390`,
`:460`, and `:823` all state that no Redis-backed rate limiter exists and that
rate limiting is unenforced. Both statements are contradicted by
`crates/buzz-pubsub/src/rate_limiter.rs:99` and the enforcement call sites at
`crates/buzz-relay/src/connection.rs:498-500` and
`crates/buzz-relay/src/api/bridge.rs:760`, `:955`, `:1386`.

---

### Correctness / robustness hotspots

| # | Issue | Detail | file:line |
|---|-------|--------|-----------|
| 1 | Duplicated tolerance constant | `TIMESTAMP_TOLERANCE_SECS = 60` is defined twice, privately, in two modules. Nothing links them, and nothing links either to `DEFAULT_REPLAY_TTL_SECS = 120`, which is documented as 2× the NIP-98 value. Changing one silently breaks the derived relationship; the tripwire test asserts against the literal `120`, not against the tolerance | `crates/buzz-auth/src/nip42.rs:35`, `crates/buzz-auth/src/nip98.rs:32`, `crates/buzz-auth/src/nip98_replay.rs:42-46`, test `:210-218` |
| 2 | Two divergent URL normalisers | `normalize_relay_url` collapses loopback aliases and returns unparseable input verbatim; `normalize_url` does neither and lowercases unparseable input. The divergence is justified for NIP-98 (`crates/buzz-auth/src/nip98.rs:138-144`) but **not documented on the NIP-42 side**, so the weaker host binding there reads as an oversight rather than a decision | `crates/buzz-auth/src/nip42.rs:19-33` vs `crates/buzz-auth/src/nip98.rs:145-153` |
| 3 | Duplicated `make_auth_event` test helper | byte-identical helper in two test modules | `crates/buzz-auth/src/nip42.rs:95-100` and `crates/buzz-auth/src/lib.rs:174-179` |
| 4 | `expect` on a production path | `parse_scopes` calls `.expect("infallible: Scope::from_str cannot fail")`. Statically unreachable given `Err = Infallible`, but it is the crate's only non-test `expect`/`unwrap` and violates the repo rule against new `unwrap`/`expect` in production paths. `Result::unwrap_or_else(\|e\| match e {})` or `Infallible`-aware destructuring would remove it | `crates/buzz-auth/src/scope.rs:170-178` |
| 5 | NIP-98 verification is not offloaded | `verify_nip98_event` performs a Schnorr verify synchronously and, unlike the NIP-42 path, this crate never wraps it in `spawn_blocking`, despite `buzz-core` documenting that requirement. The obligation is implicit on every HTTP caller | `crates/buzz-auth/src/nip98.rs:55`; NIP-42 comparison `crates/buzz-auth/src/lib.rs:128-132`; upstream note `crates/buzz-core/src/verification.rs:1-2` |
| 6 | Error coarsening loses diagnostics | NIP-42 maps both "wrong kind" and "bad signature/id" to `AuthError::InvalidSignature`, discarding the underlying `VerificationError`. Good for oracle resistance, poor for operator debugging, and there is no logging to compensate (the `tracing` dep is unused) | `crates/buzz-auth/src/nip42.rs:52-56` |
| 7 | NIP-98 error strings echo attacker input | `URL mismatch: event has \`{u_tag}\`, expected \`{expected_url}\`` interpolates the untrusted `u` tag. The doc says not to forward to clients, but nothing enforces it, and `buzz-relay` does forward it: `api_error(UNAUTHORIZED, &format!("NIP-98: {e}"))` | message `crates/buzz-auth/src/nip98.rs:98-100`; forwarded at `crates/buzz-relay/src/api/bridge.rs:112` |
| 8 | First-tag-wins on duplicate tags | `tags.find(...)` returns the first match for `challenge`, `relay`, `u`, `method`, and `payload`. A second conflicting tag is ignored rather than rejected. Contrast `buzz-relay`'s NIP-OA handling, which explicitly treats >1 `auth` tag as none | `crates/buzz-auth/src/nip42.rs:58-70`, `crates/buzz-auth/src/nip98.rs:89-117`; contrast `crates/buzz-relay/src/handlers/auth.rs:31-34` |
| 9 | Two async-trait styles in one crate | `ChannelAccessChecker` and `RateLimiter` use RPITIT (not dyn-compatible); `Nip98ReplayGuard` uses `Pin<Box<dyn Future>>`. The reason is real (the relay needs `Arc<dyn Nip98ReplayGuard>`, `crates/buzz-relay/src/state.rs:582`) but is not documented at the trait, so the inconsistency looks accidental | `crates/buzz-auth/src/access.rs:35-51`, `crates/buzz-auth/src/rate_limit.rs:174-193`, `crates/buzz-auth/src/nip98_replay.rs:66-103` |
| 10 | `AuthContext` has no invariants | all fields `pub`, no constructor, no validation. Any consumer can widen `scopes` after the fact — and the relay does hold it as `mut` (`crates/buzz-relay/src/handlers/auth.rs:91`). A private-field + builder shape would make the grant decision auditable in one place | `crates/buzz-auth/src/lib.rs:64-80` |
| 11 | `AuthService` is a near-empty wrapper | holds only `AuthConfig` and forwards to a free function; adds `spawn_blocking` and nothing else. `verify_auth_event` also ignores `self.config` entirely — the config is only there for callers to read back via `config()` | `crates/buzz-auth/src/lib.rs:100-143` |
| 12 | Fixed-window burst | documented, not fixed: up to 2× the configured rate at window boundaries. The relay compounds it deliberately with a 5s burst window for WS | `crates/buzz-auth/src/rate_limit.rs:6-7`, `:165-167`; `crates/buzz-relay/src/admission.rs:6-9` |

---

### Complexity hotspots

The crate is small and flat; no function is deeply nested.

| Function | Lines | Shape |
|----------|-------|-------|
| `verify_nip98_event` | 77 (`crates/buzz-auth/src/nip98.rs:55-131`) | 8 sequential guard clauses, one nesting level. Longest fn in the crate; readable but each step is untestable in isolation |
| `verify_nip42_event` | 40 (`crates/buzz-auth/src/nip42.rs:47-86`) | 7 guard clauses, one nesting level |
| Everything else | < 25 lines | mostly `match` tables and struct literals |

The real complexity is not in control flow but in the **cross-crate contracts
expressed as doc comments**: `Nip98ReplayGuard` carries ~30 lines of MUST/MUST NOT
prose (`crates/buzz-auth/src/nip98_replay.rs:73-100`) and `RateLimiter` ~25
(`crates/buzz-auth/src/rate_limit.rs:151-167`). None of it is machine-checked in
this crate — correctness depends on `buzz-pubsub` and `buzz-relay` honouring prose.

---

### Test coverage gaps

45 tests total (35 `#[test]`, 10 `#[tokio::test]`) across 7 of 8 files; no
integration-test directory.

| Gap | Detail |
|-----|--------|
| `crates/buzz-auth/src/error.rs` | zero tests — no `Display` output assertions, so the user-facing error strings (including the "±60s" text at `:23`) can drift from the constants unnoticed |
| `check_write_access` | never tested. Only `check_read_access` has coverage (`crates/buzz-auth/src/access.rs:178-223`); the write path's `Scope::MessagesWrite` requirement is unverified |
| `derive_pubkey_from_username` | no test at all, despite `#[cfg(any(test, ...))]` compiling it during tests. Nothing pins the `"buzz-test-key:{username}"` prefix, so the desktop↔relay derivation compatibility the doc claims (`crates/buzz-auth/src/lib.rs:149-151`) is unverified on this side |
| `parse_scopes` with `Unknown` inputs | tested only with two known scopes (`crates/buzz-auth/src/scope.rs:199-203`) |
| `Scope` round-trip | only 3 of 16 variants are round-tripped (`crates/buzz-auth/src/scope.rs:185-190`); the `as_str`/`FromStr` tables are exact inverses today but nothing enforces that for the other 13 |
| NIP-42 boundary timestamps | rejection tested at 120s (`crates/buzz-auth/src/nip42.rs:143-157`); the exact boundary (delta 60 passes, 61 fails) is untested on both paths |
| NIP-42 missing-tag paths | no test for an AUTH event with no `challenge` tag or no `relay` tag (`crates/buzz-auth/src/nip42.rs:62`, `:72`) |
| NIP-42 tampered-signature path | wrong-kind and wrong-challenge are tested; a valid kind:22242 with a mutated signature (exercising `buzz_core::verify_event` failure at `:56`) is not |
| NIP-98 missing-tag paths | no test for a missing `u` tag or missing `method` tag (`crates/buzz-auth/src/nip98.rs:95`, `:108`) |
| NIP-98 unparseable-URL fallback | the `to_lowercase()` branch (`crates/buzz-auth/src/nip98.rs:148`) is untested; likewise NIP-42's verbatim fallback (`crates/buzz-auth/src/nip42.rs:22`) |
| `payload` tag present + `body == None` | the silent-skip case (`crates/buzz-auth/src/nip98.rs:119`) is untested — the inverse case is (`:269-276`) |
| `RateLimiter` / `Nip98ReplayGuard` contracts | only the always-allow / always-fresh happy paths are tested (`crates/buzz-auth/src/rate_limit.rs:315-325`, `crates/buzz-auth/src/nip98_replay.rs:239-248`). No test asserts a denied result, an `Err` propagating, or TTL clamping — the documented MUSTs are unverified in this crate |
| `ChannelAccessChecker::can_access` override path | only the default body is exercised, via `MockAccessChecker` which does not override it |
| `spawn_blocking` panic path | `AuthError::Internal("spawn_blocking panicked")` (`crates/buzz-auth/src/lib.rs:132`) is unreachable in tests |

---

### Unimplemented / stubbed traits

| Trait | Production implementor | Assessment |
|-------|----------------------|------------|
| `ChannelAccessChecker` | **none anywhere in the repo** | genuinely unimplemented. The crate ships the enforcement helpers (`check_read_access`/`check_write_access`) with nothing to plug into them |
| `RateLimiter` | `RedisRateLimiter` (`crates/buzz-pubsub/src/rate_limiter.rs:99`), wired at `crates/buzz-relay/src/state.rs:712` | implemented and enforced (see security aspect) |
| `Nip98ReplayGuard` | `RedisNip98ReplayGuard` (`crates/buzz-pubsub/src/nip98_replay.rs:34`), wired at `crates/buzz-relay/src/state.rs:582` | implemented and enforced |

Partially-implemented tier model: `RateLimitConfig` defines 4 tiers, but only
`human` and `agent-standard` reach an enforcement point.
`agent_standard_api_calls_per_min`, `agent_elevated_messages_per_min`, and
`agent_platform_messages_per_min` are parsed from env
(`crates/buzz-relay/src/config.rs:303-314`) and never read — three documented,
CI-settable knobs that do nothing.

---

### Deprecated API usage

None found. No `#[deprecated]` items are declared in the crate, and no
`#[allow(deprecated)]` appears in `crates/buzz-auth/src/`. Dependencies are all
`workspace = true` with no pinned-back versions
(`crates/buzz-auth/Cargo.toml:15-26`).

The only lint suppressions in the crate are two
`#[allow(clippy::assertions_on_constants)]` in the const-drift tripwire tests,
each with a comment explaining why the assertion is intentionally over a constant
(`crates/buzz-auth/src/nip98_replay.rs:214`, `:226`).
