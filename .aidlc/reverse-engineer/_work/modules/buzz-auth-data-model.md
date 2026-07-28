## Module: buzz-auth (`crates/buzz-auth`)

### Data Model

All types below are defined in this crate. There is no persistence layer here —
no SQL, no ORM types. Types are either in-memory auth results, config structs
(serde-deserialised by the caller), or key-format helpers for Redis strings
produced elsewhere.

---

### `AuthMethod` — enum (`crates/buzz-auth/src/lib.rs:55-60`)

Derives: `Debug, Clone, PartialEq, Eq` (`crates/buzz-auth/src/lib.rs:54`).

| Variant | Meaning (from doc comment) | file:line |
|---------|----------------------------|-----------|
| `Nip42` | NIP-42 challenge/response — Schnorr signature over kind:22242 | `crates/buzz-auth/src/lib.rs:57` |
| `Nip98` | NIP-98 HTTP Auth — Schnorr signature over kind:27235 | `crates/buzz-auth/src/lib.rs:59` |

Note: `AuthMethod::Nip98` is never constructed inside this crate — the only
producer of `AuthContext` here (`AuthService::verify_auth_event`) hardcodes
`AuthMethod::Nip42` (`crates/buzz-auth/src/lib.rs:140`). `verify_nip98_event`
returns a bare `nostr::PublicKey`, not an `AuthContext`
(`crates/buzz-auth/src/nip98.rs:60`).

---

### `AuthContext` — struct (`crates/buzz-auth/src/lib.rs:64-80`)

Derives: `Debug, Clone` (`crates/buzz-auth/src/lib.rs:63`). Not `PartialEq`, not
`Serialize`/`Deserialize`.

| Field | Type | Semantics | file:line |
|-------|------|-----------|-----------|
| `pubkey` | `nostr::PublicKey` | The authenticated Nostr public key | `crates/buzz-auth/src/lib.rs:66` |
| `scopes` | `Vec<Scope>` | Permission scopes granted to this connection | `crates/buzz-auth/src/lib.rs:68` |
| `channel_ids` | `Option<Vec<uuid::Uuid>>` | Channel restriction; doc says "reserved for future per-channel access control", `None` means unrestricted | `crates/buzz-auth/src/lib.rs:72` |
| `auth_method` | `AuthMethod` | How the connection was authenticated | `crates/buzz-auth/src/lib.rs:74` |
| `agent_owner_pubkey` | `Option<nostr::PublicKey>` | NIP-OA verified owner pubkey; `None` for direct relay members / non-NIP-OA paths. Doc: "Set by the relay membership gate when NIP-OA fallback succeeds." | `crates/buzz-auth/src/lib.rs:79` |

Inherent method: `has_scope(&self, scope: &Scope) -> bool` — a `Vec::contains`
lookup (`crates/buzz-auth/src/lib.rs:84-86`).

`channel_ids` is set to `None` at every construction site found in the repo:
`crates/buzz-auth/src/lib.rs:139`, `crates/buzz-auth/src/lib.rs:191` (test), and
`crates/buzz-relay/src/handlers/event.rs:1370`.

---

### `Scope` — enum (`crates/buzz-auth/src/scope.rs:16-61`)

Derives: `Debug, Clone, PartialEq, Eq, Hash` (`crates/buzz-auth/src/scope.rs:15`).
Not `Serialize`/`Deserialize`; wire form goes through `as_str`/`FromStr`.

17 variants total: 16 known + 1 catch-all. Wire strings from `Scope::as_str`
(`crates/buzz-auth/src/scope.rs:114-134`) and `FromStr`
(`crates/buzz-auth/src/scope.rs:146-166`) — the two tables are exact inverses.

| # | Variant | Wire string | In `all_known()` | In `all_non_admin()` | Variant decl |
|---|---------|-------------|------------------|----------------------|--------------|
| 1 | `MessagesRead` | `messages:read` | yes | yes | `scope.rs:18` |
| 2 | `MessagesWrite` | `messages:write` | yes | yes | `scope.rs:20` |
| 3 | `ChannelsRead` | `channels:read` | yes | yes | `scope.rs:22` |
| 4 | `ChannelsWrite` | `channels:write` | yes | yes | `scope.rs:24` |
| 5 | `AdminChannels` | `admin:channels` | yes | **no** | `scope.rs:26` |
| 6 | `UsersRead` | `users:read` | yes | yes | `scope.rs:28` |
| 7 | `UsersWrite` | `users:write` | yes | yes | `scope.rs:30` |
| 8 | `AdminUsers` | `admin:users` | yes | **no** | `scope.rs:32` |
| 9 | `JobsRead` | `jobs:read` | yes | yes | `scope.rs:34` |
| 10 | `JobsWrite` | `jobs:write` | yes | yes | `scope.rs:36` |
| 11 | `SubscriptionsRead` | `subscriptions:read` | yes | yes | `scope.rs:38` |
| 12 | `SubscriptionsWrite` | `subscriptions:write` | yes | yes | `scope.rs:40` |
| 13 | `FilesRead` | `files:read` | yes | yes | `scope.rs:42` |
| 14 | `FilesWrite` | `files:write` | yes | yes | `scope.rs:44` |
| 15 | `ReposRead` | `repos:read` | yes | yes | `scope.rs:50` |
| 16 | `ReposWrite` | `repos:write` | yes | yes | `scope.rs:56` |
| 17 | `Unknown(String)` | the inner string verbatim | no | no | `scope.rs:60` |

Counts are pinned by tests: `all_known()` = **16**
(`crates/buzz-auth/src/scope.rs:237`), `all_non_admin()` = **14**
(`crates/buzz-auth/src/scope.rs:208`).

`ReposRead` / `ReposWrite` carry doc caveats about non-enforcement:
`ReposRead` — "Not currently enforced by git HTTP routes"
(`crates/buzz-auth/src/scope.rs:47-49`); `ReposWrite` — "Enforced for
kind:30617/30618 events via WebSocket ingest, but NOT enforced by git HTTP push
routes" (`crates/buzz-auth/src/scope.rs:52-55`). Both are crate-doc claims about
code in `buzz-relay`, not verified here.

Trait impls: `fmt::Display` delegates to `as_str`
(`crates/buzz-auth/src/scope.rs:137-141`); `FromStr` with
`type Err = std::convert::Infallible` (`crates/buzz-auth/src/scope.rs:143-167`)
— unknown strings become `Unknown(_)` rather than an error.

Persistence note from the doc comment: "Scopes are stored as `TEXT[]` in the
database so new variants can be added without schema migrations"
(`crates/buzz-auth/src/scope.rs:12-14`). No DB code in this crate confirms that.

---

### `AuthConfig` — struct (`crates/buzz-auth/src/lib.rs:91-95`)

Derives: `Debug, Clone, Default, serde::Serialize, serde::Deserialize`
(`crates/buzz-auth/src/lib.rs:90`).

| Field | Type | serde | file:line |
|-------|------|-------|-----------|
| `rate_limits` | `RateLimitConfig` | `#[serde(default)]` | `crates/buzz-auth/src/lib.rs:93-94` |

Doc: "typically loaded from the relay's TOML config file"
(`crates/buzz-auth/src/lib.rs:89`). In practice `buzz-relay` builds it from env
vars (`crates/buzz-relay/src/config.rs:585`).

---

### `AuthService` — struct (`crates/buzz-auth/src/lib.rs:100-102`)

Derives `Debug, Clone` (`crates/buzz-auth/src/lib.rs:99`). Single private field
`config: AuthConfig` (`crates/buzz-auth/src/lib.rs:101`). Holds no keys, no
connections, no caches.

---

### `RateLimitConfig` — struct (`crates/buzz-auth/src/rate_limit.rs:86-108`)

Derives: `Debug, Clone, Serialize, Deserialize`
(`crates/buzz-auth/src/rate_limit.rs:85`). Manual `Default` impl
(`crates/buzz-auth/src/rate_limit.rs:132-144`) — values identical to the serde
default fns.

Four tiers (human, agent-standard, agent-elevated, agent-platform) across seven
fields:

| Tier | Field | Type | Default | Default fn | Field decl |
|------|-------|------|---------|-----------|------------|
| human | `human_messages_per_min` | `u64` | 60 | `default_human_msg` (`rate_limit.rs:110-112`) | `rate_limit.rs:89` |
| human | `human_api_calls_per_min` | `u64` | 300 | `default_human_api` (`rate_limit.rs:113-115`) | `rate_limit.rs:92` |
| human | `human_ws_events_per_sec` | `u64` | 10 | `default_human_ws` (`rate_limit.rs:116-118`) | `rate_limit.rs:95` |
| agent-standard | `agent_standard_messages_per_min` | `u64` | 120 | `default_agent_std_msg` (`rate_limit.rs:119-121`) | `rate_limit.rs:98` |
| agent-standard | `agent_standard_api_calls_per_min` | `u64` | 600 | `default_agent_std_api` (`rate_limit.rs:122-124`) | `rate_limit.rs:101` |
| agent-elevated | `agent_elevated_messages_per_min` | `u64` | 300 | `default_agent_elev_msg` (`rate_limit.rs:125-127`) | `rate_limit.rs:104` |
| agent-platform | `agent_platform_messages_per_min` | `u64` | 600 | `default_agent_plat_msg` (`rate_limit.rs:128-130`) | `rate_limit.rs:107` |

Every field carries `#[serde(default = "...")]`, so a partial TOML/JSON section
deserialises successfully with per-field fallbacks.

---

### `LimitType` — enum (`crates/buzz-auth/src/rate_limit.rs:58-67`)

Derives: `Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize` with
`#[serde(rename_all = "snake_case")]`
(`crates/buzz-auth/src/rate_limit.rs:56-57`).

| Variant | serde string | `key_suffix()` | Doc meaning | file:line |
|---------|--------------|----------------|-------------|-----------|
| `Messages` | `messages` | `msg` | Nostr message events sent via WebSocket | `rate_limit.rs:60` / suffix `:72` |
| `ApiCalls` | `api_calls` | `api` | HTTP REST API calls | `rate_limit.rs:62` / suffix `:73` |
| `WsEvents` | `ws_events` | `ws` | All WebSocket events (broader than `Messages`) | `rate_limit.rs:64` / suffix `:74` |
| `IpConnections` | `ip_connections` | `conn` | Concurrent WS connections from a single IP | `rate_limit.rs:66` / suffix `:75` |

`key_suffix(&self) -> &'static str` at `crates/buzz-auth/src/rate_limit.rs:71-78`.

---

### `RateLimitResult` — struct (`crates/buzz-auth/src/rate_limit.rs:19-28`)

Derives `Debug, Clone, PartialEq, Eq` (`crates/buzz-auth/src/rate_limit.rs:18`).

| Field | Type | Meaning | file:line |
|-------|------|---------|-----------|
| `allowed` | `bool` | permitted (`true`) vs reject (`false`) | `rate_limit.rs:21` |
| `current` | `u64` | counter value after this increment | `rate_limit.rs:23` |
| `limit` | `u64` | configured limit for this window | `rate_limit.rs:25` |
| `reset_in_secs` | `u64` | seconds until window reset | `rate_limit.rs:27` |

Constructors: `RateLimitResult::allowed(current, limit, reset_in_secs)`
(`crates/buzz-auth/src/rate_limit.rs:32-39`) and
`RateLimitResult::denied(...)` (`crates/buzz-auth/src/rate_limit.rs:42-49`).

---

### `AuthError` — enum (`crates/buzz-auth/src/error.rs:9-59`)

See the conventions aspect for the full variant/message table. Derives
`Debug, thiserror::Error` (`crates/buzz-auth/src/error.rs:8`). One variant is
structured: `InsufficientScope { required: String, have: Vec<String> }`
(`crates/buzz-auth/src/error.rs:45-50`).

---

### Challenge / token types

There is **no** challenge struct and **no** token type in this crate.

- The NIP-42 challenge is a bare `String`: `generate_challenge() -> String`,
  32 CSPRNG bytes hex-encoded (`crates/buzz-auth/src/nip42.rs:38-41`). The test
  pins 64 hex chars (`crates/buzz-auth/src/nip42.rs:103-109`). Challenge state
  lives in the caller (`AuthState::Pending { challenge }` in
  `crates/buzz-relay/src/connection.rs:44` / `handlers/auth.rs:48`).
- No JWT, opaque token, refresh token, or API-token struct exists. The crate doc
  states this explicitly: "No JWT validation, no token management, no IdP runtime
  dependency" (`crates/buzz-auth/src/lib.rs:16`, repeated at
  `crates/buzz-auth/src/lib.rs:98`).
- The NIP-98 credential is a raw JSON string parsed into `nostr::Event` inside
  `verify_nip98_event` (`crates/buzz-auth/src/nip98.rs:62`); the crate never
  defines a wrapper type for it.

---

### Replay-guard data (`crates/buzz-auth/src/nip98_replay.rs`)

No struct — the seen-set is modelled entirely as the `Nip98ReplayGuard` trait
plus two key-format functions and two constants.

| Constant | Value | Purpose | file:line |
|----------|-------|---------|-----------|
| `DEFAULT_REPLAY_TTL_SECS` | `120` | Floor for the replay window (2× the ±60s NIP-98 tolerance) | `nip98_replay.rs:46` |
| `MAX_REPLAY_TTL_SECS` | `3600` | Ceiling; implementations MUST clamp down to it | `nip98_replay.rs:59` |

Key formats:

| Function | Format | file:line |
|----------|--------|-----------|
| `nip98_replay_key(ctx, event_id)` | delegates to the scope form with `ctx.community().to_string()` | `nip98_replay.rs:114-116` |
| `nip98_replay_key_for_scope(scope, event_id)` | `buzz:{scope}:nip98:{event_id_hex}` | `nip98_replay.rs:119-121` |
| `rate_limit_key(ctx, pubkey, limit_type)` | `buzz:{community}:ratelimit:{pubkey_hex}:{suffix}` | `rate_limit.rs:201-208` |
| `ip_rate_limit_key(ip)` | `buzz:ratelimit:ip:{ip}:conn` (no community prefix — operator-global) | `rate_limit.rs:213-215` |

`MockAccessChecker` holds one in-memory field:
`allowed: HashSet<(uuid::Uuid, String, Uuid)>` keyed on
`(community_id, pubkey_hex, channel_id)`
(`crates/buzz-auth/src/access.rs:108-110`, insert at `:123-125`) — cfg-gated to
`any(test, feature = "test-utils")` (`crates/buzz-auth/src/access.rs:107`).

---

### External types referenced (not defined here)

| Type | Origin | Used for | file:line |
|------|--------|----------|-----------|
| `nostr::PublicKey` | `nostr` | authenticated identity | `lib.rs:66`, `nip98.rs:60` |
| `nostr::Event` | `nostr` | AUTH / HTTP-auth events | `nip42.rs:48`, `nip98.rs:62` |
| `nostr::EventId` | `nostr` | replay seen-set key | `nip98_replay.rs:37` |
| `nostr::Timestamp` | `nostr` | clock-skew checks | `nip42.rs:78`, `nip98.rs:78` |
| `buzz_core::TenantContext` | `buzz-core` | community scoping on access + rate-limit + replay | `access.rs:9`, `rate_limit.rs:11`, `nip98_replay.rs:36` |
| `uuid::Uuid` | `uuid` | channel ids | `access.rs:11`, `lib.rs:72` |
| `std::net::IpAddr` | std | IP-keyed limits | `rate_limit.rs:9` |
