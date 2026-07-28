## Module: buzz-auth (`crates/buzz-auth`)

### Integrations

### Infrastructure reach — explicit answer

| Resource | Does this crate touch it? | Evidence |
|----------|---------------------------|----------|
| **Redis** | **No.** No `redis`, `deadpool-redis`, or any Redis client in the manifest (`crates/buzz-auth/Cargo.toml:14-26`). The crate only *formats Redis key strings* (`crates/buzz-auth/src/rate_limit.rs:201-215`, `crates/buzz-auth/src/nip98_replay.rs:114-121`) and *documents* Redis semantics that implementors must satisfy (`crates/buzz-auth/src/nip98_replay.rs:60-62`, `crates/buzz-auth/src/rate_limit.rs:3-4`). No command is ever issued from here. |
| **Postgres** | **No.** No `sqlx` and no `buzz-db` dependency (`crates/buzz-auth/Cargo.toml:14-26`). The `ChannelAccessChecker` trait exists precisely so the crate can enforce access "without depending on `buzz-db` directly" (`crates/buzz-auth/src/access.rs:3-4`, `:18-20`). |
| **Network (HTTP/WS/DNS)** | **No.** No `reqwest`, `axum`, `hyper`, `tokio-tungstenite`, or socket usage. NIP-42 verification is documented as "Pure cryptographic verification — no network calls, no JWT, no tokens" (`crates/buzz-auth/src/lib.rs:117`). `url` is used only for string parsing/normalisation (`crates/buzz-auth/src/nip42.rs:10`, `crates/buzz-auth/src/nip98.rs:28`) — it performs no resolution or I/O. `std::net::IpAddr` is used only as a key-formatting input (`crates/buzz-auth/src/rate_limit.rs:9`, `:213-215`). |
| **Filesystem** | **No.** No `std::fs`, no path handling, no config file reading. `AuthConfig` is a plain serde struct; the caller loads it (`crates/buzz-auth/src/lib.rs:89-95`, loaded at `crates/buzz-relay/src/config.rs:585`). |
| **Clock** | Yes — `nostr::Timestamp::now()` for skew checks (`crates/buzz-auth/src/nip42.rs:78`, `crates/buzz-auth/src/nip98.rs:78`). |
| **OS CSPRNG** | Yes — `rand::random::<[u8; 32]>()` for challenges (`crates/buzz-auth/src/nip42.rs:39`). |
| **Async runtime** | Yes — `tokio::task::spawn_blocking` for the CPU-bound Schnorr verify (`crates/buzz-auth/src/lib.rs:128-132`). |

Net: `buzz-auth` is a pure-compute library. Its only side effects are reading the
wall clock and drawing OS randomness.

---

### Internal dependencies

| Crate | Declared | What is used | Where |
|-------|----------|--------------|-------|
| `buzz-core` | `crates/buzz-auth/Cargo.toml:15` | `verify_event(&Event) -> Result<(), VerificationError>` — verifies event id hash then Schnorr signature (`crates/buzz-core/src/verification.rs:11-32`) | `crates/buzz-auth/src/nip42.rs:56`, `crates/buzz-auth/src/nip98.rs:74` |
| `buzz-core` | same | `TenantContext` (and `.community()`) for community scoping | `crates/buzz-auth/src/access.rs:9`, `crates/buzz-auth/src/rate_limit.rs:11`, `crates/buzz-auth/src/nip98_replay.rs:36` |
| `buzz-core` | same | `CommunityId` — test fixtures only | `crates/buzz-auth/src/rate_limit.rs:247`, `crates/buzz-auth/src/access.rs:156`, `crates/buzz-auth/src/nip98_replay.rs:144` |

`buzz-core` is the crate's only internal dependency. It does **not** depend on
`buzz-db`, `buzz-pubsub`, `buzz-relay`, or any other workspace crate.

---

### External crates and why

| Crate | Declared | Used for | Where |
|-------|----------|----------|-------|
| `nostr` | `crates/buzz-auth/Cargo.toml:16` | `Event`, `PublicKey`, `EventId`, `Kind` (`Authentication`, `HttpAuth`), `TagKind` (`Challenge`, `Relay`, `Method`, `Payload`), `SingleLetterTag`/`Alphabet` for the `u` tag, `Timestamp`, `SecretKey`, `Keys` | `crates/buzz-auth/src/nip42.rs:9`, `crates/buzz-auth/src/nip98.rs:26`, `crates/buzz-auth/src/lib.rs:66`, `crates/buzz-auth/src/nip98_replay.rs:37` |
| `serde` | `crates/buzz-auth/Cargo.toml:17` | `Serialize`/`Deserialize` on `AuthConfig`, `RateLimitConfig`, `LimitType`; `#[serde(default = ...)]` field fallbacks and `rename_all = "snake_case"` | `crates/buzz-auth/src/lib.rs:90`, `crates/buzz-auth/src/rate_limit.rs:13`, `:56-57`, `:85` |
| `serde_json` | `crates/buzz-auth/Cargo.toml:18` | parse the NIP-98 `Authorization` event JSON (`from_str`) | `crates/buzz-auth/src/nip98.rs:62`; also test serialisation `:185` |
| `tokio` | `crates/buzz-auth/Cargo.toml:19` | `task::spawn_blocking` to keep Schnorr verification off async worker threads; `#[tokio::test]` in tests | `crates/buzz-auth/src/lib.rs:128`, tests `:199`, `crates/buzz-auth/src/access.rs:163` |
| `sha2` | `crates/buzz-auth/Cargo.toml:22` | `Sha256` for the NIP-98 `payload` body hash and for the dev key derivation | `crates/buzz-auth/src/nip98.rs:27`, `:120`; `crates/buzz-auth/src/lib.rs:161-163` |
| `hex` | `crates/buzz-auth/Cargo.toml:23` | encode the 32-byte challenge and the computed body hash for comparison | `crates/buzz-auth/src/nip42.rs:40`, `crates/buzz-auth/src/nip98.rs:121` |
| `rand` | `crates/buzz-auth/Cargo.toml:24` | `rand::random::<[u8; 32]>()` challenge entropy | `crates/buzz-auth/src/nip42.rs:39` |
| `url` | `crates/buzz-auth/Cargo.toml:26` | `Url::parse` + path/host manipulation for both URL normalisers | `crates/buzz-auth/src/nip42.rs:10`, `:20-32`; `crates/buzz-auth/src/nip98.rs:28`, `:146-152` |
| `uuid` | `crates/buzz-auth/Cargo.toml:25` | `Uuid` channel ids in `AuthContext` and the access trait | `crates/buzz-auth/src/lib.rs:72`, `crates/buzz-auth/src/access.rs:11` |
| `thiserror` | `crates/buzz-auth/Cargo.toml:21` | derive `Error` + `Display` on `AuthError` | `crates/buzz-auth/src/error.rs:8` |
| `tracing` | `crates/buzz-auth/Cargo.toml:20` | **declared but unused** — no `tracing::` call, no `use tracing`, and no `#[instrument]` anywhere in `crates/buzz-auth/src/` | manifest `crates/buzz-auth/Cargo.toml:20` |

All dependencies use `{ workspace = true }` — no crate-local version pins
(`crates/buzz-auth/Cargo.toml:15-26`).

---

### Consumers of this crate

| Consumer | Declared | What it uses |
|----------|----------|--------------|
| `buzz-relay` | `crates/buzz-relay/Cargo.toml:22` (plus `features = ["dev"]` in `[dev-dependencies]`, `:90`) | `AuthService`/`AuthConfig` (`crates/buzz-relay/src/state.rs:19`, `:500`; `crates/buzz-relay/src/main.rs:368`), `AuthContext` (`crates/buzz-relay/src/connection.rs:17`, `:44`), `generate_challenge` + `verify_auth_event` (`crates/buzz-relay/src/handlers/auth.rs:87-89`, `crates/buzz-relay/src/audio/handler.rs:222`), `verify_nip98_event` (`crates/buzz-relay/src/api/bridge.rs:111`), `LimitType`/`RateLimiter` (`crates/buzz-relay/src/admission.rs:1`), `Nip98ReplayGuard` + `DEFAULT_REPLAY_TTL_SECS` (`crates/buzz-relay/src/api/bridge.rs:16`, `crates/buzz-relay/src/state.rs:582`), `Scope::all_known()` (`crates/buzz-relay/src/api/bridge.rs:829`), `RateLimitConfig` (`crates/buzz-relay/src/config.rs:284-316`) |
| `buzz-pubsub` | `crates/buzz-pubsub/Cargo.toml:12` | implements `RateLimiter` as `RedisRateLimiter` (`crates/buzz-pubsub/src/rate_limiter.rs:99-121`) and `Nip98ReplayGuard` as `RedisNip98ReplayGuard` (`crates/buzz-pubsub/src/nip98_replay.rs:34`); calls `rate_limit_key` / `ip_rate_limit_key` (`crates/buzz-pubsub/src/rate_limiter.rs:108`, `:118`) |
| `buzz-admin` | `crates/buzz-admin/Cargo.toml:17` | **nothing** — a grep of `crates/buzz-admin/` for `buzz_auth` returns zero matches; the dependency appears unused |

`crates/buzz-conformance/Cargo.toml:18` carries an explicit prohibition comment
naming `buzz-auth` among crates it must never depend on.

---

### Protocol / spec conformance surface

| Spec | Kind | Where implemented |
|------|------|-------------------|
| NIP-42 (client authentication of clients to relays) | `Kind::Authentication` = 22242 | `crates/buzz-auth/src/nip42.rs:52`; module doc describes the 3-step handshake at `:1-7` |
| NIP-98 (HTTP Auth) | `Kind::HttpAuth` = 27235 | `crates/buzz-auth/src/nip98.rs:66`; `Authorization: Nostr <base64(event)>` header shape documented at `:9-11` (base64 decoding is the caller's job, `:38`) |
| NIP-OA (owner attestation) | — | not implemented here; only the `AuthContext.agent_owner_pubkey` field is reserved for the relay to fill (`crates/buzz-auth/src/lib.rs:75-79`). The tag extraction/verification lives in `crates/buzz-relay/src/handlers/auth.rs:26-36`. |
| NIP-29 (relay-based groups) | — | referenced only as the justification for granting full scopes (`crates/buzz-auth/src/lib.rs:135`) |

The `nostr` workspace dependency enables the `nip98` feature explicitly
(`Cargo.toml:61`), which is what provides `TagKind::Method` / `TagKind::Payload`.
