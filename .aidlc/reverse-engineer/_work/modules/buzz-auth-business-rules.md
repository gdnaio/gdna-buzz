## Module: buzz-auth (`crates/buzz-auth`)

### Business Rules

---

### 1. NIP-42 verification sequence

Entry point `verify_nip42_event(event, expected_challenge, relay_url)`
(`crates/buzz-auth/src/nip42.rs:47-86`). Guard-clause style: each step returns
early on failure; there is no accumulation of errors. Order matters — the first
failing check determines the error variant.

| Step | Rule enforced | Failure | file:line |
|------|---------------|---------|-----------|
| 1 | `event.kind` must equal `Kind::Authentication` (kind 22242) | `AuthError::InvalidSignature` | `crates/buzz-auth/src/nip42.rs:52-54` |
| 2 | Event id hash + Schnorr signature must verify via `buzz_core::verify_event` | `AuthError::InvalidSignature` (original error discarded) | `crates/buzz-auth/src/nip42.rs:56` |
| 3 | A `challenge` tag must be present with content | `AuthError::ChallengeMismatch` | `crates/buzz-auth/src/nip42.rs:58-62` |
| 4 | `challenge` tag content must equal `expected_challenge` byte-for-byte (`!=` on `&str`) | `AuthError::ChallengeMismatch` | `crates/buzz-auth/src/nip42.rs:64-66` |
| 5 | A `relay` tag must be present with content | `AuthError::RelayUrlMismatch` | `crates/buzz-auth/src/nip42.rs:68-72` |
| 6 | `normalize_relay_url(relay_tag)` must equal `normalize_relay_url(relay_url)` | `AuthError::RelayUrlMismatch` | `crates/buzz-auth/src/nip42.rs:74-76` |
| 7 | `abs_diff(now, event.created_at)` must be `<= 60` | `AuthError::EventExpired` | `crates/buzz-auth/src/nip42.rs:78-83` |

Trigger: called from `AuthService::verify_auth_event`, which wraps it in
`tokio::task::spawn_blocking` because Schnorr verification is CPU-bound
(`crates/buzz-auth/src/lib.rs:125-132`). A `spawn_blocking` panic maps to
`AuthError::Internal("spawn_blocking panicked")`
(`crates/buzz-auth/src/lib.rs:132`). In the relay, the caller is
`handlers::auth::handle_auth` (`crates/buzz-relay/src/handlers/auth.rs:87-89`),
which supplies the connection's pending challenge
(`crates/buzz-relay/src/handlers/auth.rs:48`) and a tenant-derived relay URL
(`crates/buzz-relay/src/handlers/auth.rs:80-81`).

Notable ordering consequence: signature verification (step 2) happens **before**
the challenge and relay-URL checks, so an unsigned/forged event can never reach
the tag comparisons. Conversely a validly-signed event for a different relay
still consumes a Schnorr verification.

---

### 2. NIP-42 timestamp tolerance

| Rule | Value | file:line |
|------|-------|-----------|
| `TIMESTAMP_TOLERANCE_SECS` | `60` (private `const u64`) | `crates/buzz-auth/src/nip42.rs:35` |
| Comparison | `delta = now.abs_diff(event_ts); if delta > TIMESTAMP_TOLERANCE_SECS { EventExpired }` | `crates/buzz-auth/src/nip42.rs:78-83` |

`abs_diff` makes the window symmetric: future-dated events are rejected the same
way as stale ones, so the effective window is ±60s inclusive (delta of exactly 60
passes; 61 fails). Test pins rejection at 120s in the past
(`crates/buzz-auth/src/nip42.rs:143-157`). `AuthError::EventExpired`'s Display
text also states "±60 seconds" (`crates/buzz-auth/src/error.rs:22-24`).

---

### 3. Relay-URL comparison rules for AUTH

`normalize_relay_url` (`crates/buzz-auth/src/nip42.rs:19-33`), applied to **both**
sides before comparison (`crates/buzz-auth/src/nip42.rs:74`):

| Rule | Behaviour | file:line |
|------|-----------|-----------|
| Unparseable input | returned verbatim (`raw.to_string()`) — falls back to exact string comparison | `crates/buzz-auth/src/nip42.rs:20-23` |
| Loopback aliasing | host `localhost` **or** `::1` is rewritten to `127.0.0.1`; the three are treated as equivalent | `crates/buzz-auth/src/nip42.rs:25-29` |
| Trailing slash | path is `trim_end_matches('/')` then re-set | `crates/buzz-auth/src/nip42.rs:30-31` |
| Scheme/host case, default ports, percent-encoding | delegated to `url::Url` parsing + `to_string()` | `crates/buzz-auth/src/nip42.rs:20`, `:32` |

Scheme is **not** normalised across `ws`/`wss` — `ws://h` and `wss://h` produce
different strings and would mismatch. Port is part of the comparison (the
localhost test uses matching `:3030` on both sides,
`crates/buzz-auth/src/nip42.rs:171-175`).

This is deliberately **different** from NIP-98's URL rule (rule 5 below):
NIP-42 collapses loopback aliases, NIP-98 does not.

---

### 4. NIP-98 verification sequence

Entry point `verify_nip98_event(event_json, expected_url, expected_method, body)`
(`crates/buzz-auth/src/nip98.rs:55-131`). Steps are numbered in the code itself.

| Step | Rule enforced | Failure message (all wrapped in `AuthError::Nip98Invalid`) | file:line |
|------|---------------|-----------------------------------------------------------|-----------|
| 1 | `event_json` must deserialise into `nostr::Event` | `event JSON parse error: {e}` | `crates/buzz-auth/src/nip98.rs:62-63` |
| 2 | `event.kind` must equal `Kind::HttpAuth` (27235) | `expected kind 27235, got {n}` | `crates/buzz-auth/src/nip98.rs:66-71` |
| 3 | Event id hash + Schnorr signature via `buzz_core::verify_event` | `invalid Schnorr signature` | `crates/buzz-auth/src/nip98.rs:74-75` |
| 4 | `abs_diff(now, created_at) <= 60` | `event timestamp outside ±60s window (delta: {d}s)` | `crates/buzz-auth/src/nip98.rs:78-85` |
| 5a | A single-letter lowercase `u` tag must be present with content | `missing \`u\` tag` | `crates/buzz-auth/src/nip98.rs:89-95` |
| 5b | `normalize_url(u_tag)` must equal `normalize_url(expected_url)` | `URL mismatch: event has \`..\`, expected \`..\`` | `crates/buzz-auth/src/nip98.rs:97-101` |
| 6a | A `method` tag must be present with content | `missing \`method\` tag` | `crates/buzz-auth/src/nip98.rs:104-108` |
| 6b | `method_tag.eq_ignore_ascii_case(expected_method)` | `method mismatch: ...` | `crates/buzz-auth/src/nip98.rs:110-114` |
| 7 | **Only if** a `payload` tag exists **and** `body` is `Some`: `hex(SHA-256(body))` must equal the tag value | `payload tag SHA-256 mismatch: request body does not match signed hash` | `crates/buzz-auth/src/nip98.rs:117-127` |
| 8 | Return `event.pubkey` | — | `crates/buzz-auth/src/nip98.rs:130` |

Step 7 is conditional in both directions: a missing `payload` tag with a present
body passes (test: `crates/buzz-auth/src/nip98.rs:269-276`), and a present
`payload` tag with `body == None` is silently ignored. The tuple destructure
`(Some(payload_hex), Some(body_bytes))` is what makes both cases skip
(`crates/buzz-auth/src/nip98.rs:119`). Enforcing the tag's presence is the
caller's job — `buzz-relay` does that with a `require_payload` flag before
calling in (`crates/buzz-relay/src/api/bridge.rs:99-112`).

Trigger: HTTP requests carrying `Authorization: Nostr <base64(event JSON)>`; the
caller base64-decodes and passes the JSON string
(`crates/buzz-auth/src/nip98.rs:38`, caller at
`crates/buzz-relay/src/api/bridge.rs:81-112`). This function is **synchronous**
and is not wrapped in `spawn_blocking` by this crate, unlike the NIP-42 path.

---

### 5. NIP-98 URL comparison rules

`normalize_url` (`crates/buzz-auth/src/nip98.rs:145-153`), applied to both sides:

| Rule | Behaviour | file:line |
|------|-----------|-----------|
| Unparseable input | `raw.to_lowercase()` (note: differs from NIP-42's verbatim fallback) | `crates/buzz-auth/src/nip98.rs:147-148` |
| Scheme/host case | lowercased by `url::Url` | `crates/buzz-auth/src/nip98.rs:135` |
| Trailing slash | stripped from path | `crates/buzz-auth/src/nip98.rs:150-151` |
| Loopback aliasing | **explicitly not performed** — `localhost`, `::1`, `127.0.0.1` are three distinct hosts | `crates/buzz-auth/src/nip98.rs:138-144` |

Stated rationale: under multi-tenant the `u`-tag host is the row-zero community
binding, so collapsing loopback aliases would be a host-binding side door
(`crates/buzz-auth/src/nip98.rs:138-144`). Pinned by a three-way test asserting
both directions reject and same-host still passes
(`crates/buzz-auth/src/nip98.rs:288-316`).

Query string and fragment are part of `Url::to_string()` and therefore part of
the comparison — no explicit stripping.

---

### 6. Scope-granting rules — which auth path grants what

| Auth path | Scopes granted | file:line |
|-----------|----------------|-----------|
| NIP-42 (`AuthService::verify_auth_event`) | `Scope::all_known()` — all **16** known variants, unconditionally, for every successfully-authenticated pubkey | `crates/buzz-auth/src/lib.rs:136-142`, list at `crates/buzz-auth/src/scope.rs:68-87` |
| NIP-98 (`verify_nip98_event`) | **none** — returns only a `PublicKey`; no `AuthContext`, no scope set | `crates/buzz-auth/src/nip98.rs:60`, `:130` |

Other `AuthContext` fields on the NIP-42 path are fixed:
`channel_ids: None` (unrestricted), `auth_method: AuthMethod::Nip42`,
`agent_owner_pubkey: None` with the comment "Set later by relay membership gate
if NIP-OA" (`crates/buzz-auth/src/lib.rs:139-141`).

Stated justification: "In pure Nostr mode, all authenticated connections get full
scopes. Per-channel access is enforced by the relay's membership checks (NIP-29)"
(`crates/buzz-auth/src/lib.rs:134-135`; same claim in the module doc at
`crates/buzz-auth/src/scope.rs:3-5`).

`Scope::all_non_admin()` (14 variants, excludes `AdminChannels` and `AdminUsers`)
exists as an alternative grant set (`crates/buzz-auth/src/scope.rs:94-111`) but is
never called anywhere in the repo — no auth path in this crate or outside it uses
it. Its doc comment describes a dev-mode `X-Pubkey` / `require_auth_token=false`
scenario (`crates/buzz-auth/src/scope.rs:91-93`); the same doc text is
copy-pasted onto `all_known()` (`crates/buzz-auth/src/scope.rs:66-67`) even though
`all_known()` is what the production NIP-42 path uses.

Repo-wide, the only external caller of either is
`crates/buzz-relay/src/api/bridge.rs:829`, which builds its own auth struct with
`buzz_auth::Scope::all_known()`.

---

### 7. Scope-checking logic

| Rule | Implementation | file:line |
|------|----------------|-----------|
| `require_scope` | `scopes.contains(&required)` → `Ok(())`, else `AuthError::InsufficientScope { required, have }` where `have` is the caller's full scope list stringified | `crates/buzz-auth/src/access.rs:60-69` |
| `AuthContext::has_scope` | `self.scopes.contains(scope)` — bool, no error | `crates/buzz-auth/src/lib.rs:84-86` |
| Matching semantics | exact `PartialEq` on the enum. No hierarchy, no wildcards, no implication (`admin:channels` does **not** imply `channels:write`; `messages:write` does not imply `messages:read`) | derive at `crates/buzz-auth/src/scope.rs:15`; enumerated grant list at `crates/buzz-auth/src/scope.rs:69-86` |
| `Unknown(s)` scopes | compare by inner string only; never satisfy a known-variant requirement | `crates/buzz-auth/src/scope.rs:60`, `:164` |

Because NIP-42 grants all 16 known scopes, `require_scope` can only fail for a
caller whose scope list was built some other way (e.g. an empty list, as in the
test at `crates/buzz-auth/src/access.rs:189`).

---

### 8. Combined access rules (scope + membership)

| Function | Sequence | file:line |
|----------|----------|-----------|
| `check_read_access` | 1) `require_scope(scopes, Scope::MessagesRead)?` 2) `checker.can_access(ctx, pubkey, channel_id)` → `Ok(())` or `AuthError::ChannelAccessDenied` | `crates/buzz-auth/src/access.rs:72-85` |
| `check_write_access` | identical but requires `Scope::MessagesWrite` | `crates/buzz-auth/src/access.rs:88-101` |

Ordering rule: the scope check runs **before** the membership lookup, so an
under-scoped caller never triggers a DB round-trip
(`crates/buzz-auth/src/access.rs:79-80`, `:95-96`). A `Err` from the checker
propagates as-is via `?` (fail-closed, no swallow).

`can_access` default implementation: fetch the whole accessible set, then
`contains` (`crates/buzz-auth/src/access.rs:52-55`).

Tenant rule for implementors: every query MUST be scoped by `ctx.community()`
because the `channels` PK is `(community_id, id)` and a bare `WHERE id = $1`
would be a cross-community existence oracle
(`crates/buzz-auth/src/access.rs:22-30`). Pinned by
`access_does_not_cross_communities` (`crates/buzz-auth/src/access.rs:225-251`).

Neither `check_read_access` nor `check_write_access` is called anywhere outside
this crate's own tests.

---

### 9. Rate-limit contract rules (interface-level, defined here)

| Rule | Statement | file:line |
|------|-----------|-----------|
| Pubkey-keyed quotas are per-community | key is `buzz:{community}:ratelimit:{pubkey_hex}:{suffix}`; same pubkey in two communities = two independent quotas | `crates/buzz-auth/src/rate_limit.rs:201-208`, doc `:151-156` |
| IP-keyed quotas are operator-global | key `buzz:ratelimit:ip:{ip}:conn`, deliberately no `TenantContext` (runs before host→community resolution) | `crates/buzz-auth/src/rate_limit.rs:213-215`, doc `:158-163` |
| Key stability | keys must be all-lowercase ASCII, else the same principal would get two counters (effective 2× quota) — pinned by test | `crates/buzz-auth/src/rate_limit.rs:290-306` |
| Fixed-window burst | documented as allowing up to 2× at window boundaries | `crates/buzz-auth/src/rate_limit.rs:6-7`, `:165-167` |
| Allow/deny boundary | decided by the implementor, not here. `RedisRateLimiter` uses `count <= limit` → allowed | `crates/buzz-pubsub/src/rate_limiter.rs:74-78` |

---

### 10. NIP-98 replay rules (contract defined here, enforced elsewhere)

| Rule | Statement | file:line |
|------|-----------|-----------|
| Verify-then-mark ordering | Verify the event first, then claim the seen-set slot. Marking before verifying would let an attacker who knows a future event id DoS the legitimate event | `crates/buzz-auth/src/nip98_replay.rs:19-27` |
| Return semantics | `Ok(true)` = newly inserted, proceed. `Ok(false)` = already seen, caller MUST reject as replay | `crates/buzz-auth/src/nip98_replay.rs:78-81` |
| Fail-closed on error | On `Err` (e.g. Redis unreachable) callers MUST reject, not admit | `crates/buzz-auth/src/nip98_replay.rs:83-86` |
| Atomicity | implementations MUST use atomic set-if-absent; read-then-write forfeits the freshness proof | `crates/buzz-auth/src/nip98_replay.rs:88-90` |
| TTL floor | `ttl_secs` ≥ `DEFAULT_REPLAY_TTL_SECS` (120); implementations MAY clamp up, MUST NOT honour smaller | `crates/buzz-auth/src/nip98_replay.rs:92-94`, const `:46` |
| TTL ceiling | clamp down to `MAX_REPLAY_TTL_SECS` (3600); larger values risk Redis `EX` (i64) parse failure | `crates/buzz-auth/src/nip98_replay.rs:96-100`, const `:59` |
| Why 120s | 2× the ±60s verifier tolerance — the full span over which a duplicate id is physically plausible | `crates/buzz-auth/src/nip98_replay.rs:29-31`, `:42-45` |
| No in-process caching | with multiple relay pods an in-process cache (moka, DashMap) does not carry the freshness proof across pods | `crates/buzz-auth/src/nip98_replay.rs:6-10` |

Three const-drift tripwire tests pin these bounds: TTL ≥ 120
(`crates/buzz-auth/src/nip98_replay.rs:210-218`), floor < ceiling
(`:220-230`), ceiling fits `i64::MAX` (`:232-237`).

---

### 11. Dev-only username→key derivation rule

| Rule | Statement | file:line |
|------|-----------|-----------|
| Derivation | secret key material = `SHA-256("buzz-test-key:{username}")`, then `nostr::SecretKey::from_slice` → `Keys::new(...).public_key()` | `crates/buzz-auth/src/lib.rs:162-166` |
| Gate | `#[cfg(any(test, feature = "dev"))]` | `crates/buzz-auth/src/lib.rs:159` |
| Stated invariant | "must never be compiled into a production release build"; keys are predictable from the username alone | `crates/buzz-auth/src/lib.rs:152-158` |
| Failure mode | `from_slice` error → `AuthError::Internal("key derivation failed: {e}")` | `crates/buzz-auth/src/lib.rs:164-165` |

Stated purpose: match the desktop's `set_test_identity` derivation so the relay
can resolve usernames to pubkeys in dev mode
(`crates/buzz-auth/src/lib.rs:148-151`). No caller exists anywhere in the repo.

---

### 12. Security invariants asserted by the crate docs

| Invariant | file:line | Verified in this crate? |
|-----------|-----------|-------------------------|
| "AUTH events (kind:22242) are NEVER stored or logged" | `crates/buzz-auth/src/lib.rs:14`, `crates/buzz-auth/src/nip42.rs:7` | Negatively — this crate has no logging or persistence on the NIP-42 path. Nothing enforces it for callers. |
| "All paths produce an `AuthContext` bound to the connection" | `crates/buzz-auth/src/lib.rs:15` | **Not accurate for NIP-98** — `verify_nip98_event` returns a `PublicKey` (`crates/buzz-auth/src/nip98.rs:60`). |
| "No JWT validation, no token management, no IdP runtime dependency" | `crates/buzz-auth/src/lib.rs:16`, `:98` | Accurate — no such types or deps exist in the crate. |
