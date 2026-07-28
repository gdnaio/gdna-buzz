## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Integrations

---

### 1. External systems

| System | Protocol | Client | Usage |
|---|---|---|---|
| Redis | RESP over TCP | `deadpool-redis` pool (`Cargo.toml:14`) | All imperative commands: `PUBLISH`, `SET`, `GET`, `MGET`, `DEL`, `TTL`, `EXPIRE`, `EVALSHA` |
| Redis | RESP pub/sub mode | `redis::Client` → `get_async_pubsub()` (`Cargo.toml:13`) | Three independent stateful connections: events (`subscriber.rs:85-87`), cache-invalidate (`cache_invalidation.rs:135-139`), conn-control (`conn_control.rs:126-130`) |

Redis is the **only** external system. No Postgres, no HTTP client, no object
store, no message broker. The crate declares no `buzz-db`, `reqwest`, `axum`, or
`sqlx` dependency (`Cargo.toml:11-24`).

Redis command inventory (exhaustive, by call site):

| Command | Site |
|---|---|
| `PUBLISH` | `publisher.rs:32-36`; `lib.rs:276-280` (cache), `:296-300` (conn-control) |
| `SET … EX` | `presence.rs:37-43` |
| `SET … NX EX` | `nip98_replay.rs:68-78` |
| `DEL` | `presence.rs:53-56` |
| `GET` | `presence.rs:69` |
| `MGET` | `presence.rs:88` |
| `EXPIRE` (repair) | `rate_limiter.rs:60-65` |
| `INCR` + `EXPIRE` + `TTL` (in Lua) | `rate_limiter.rs:24-31`, invoked `:38-42` |
| `SUBSCRIBE` / `UNSUBSCRIBE` | `subscriber.rs:98`, `:119`, `:127` |
| `PSUBSCRIBE` | `cache_invalidation.rs:139`, `conn_control.rs:130` |
| `TTL` (assertions only) | `lib.rs:491`, `presence.rs:196` — test code |

Four connections per relay pod minimum: one pool (shared, size set by the caller)
plus three dedicated pub/sub sockets. The pool is **injected**, never created here —
every constructor takes `deadpool_redis::Pool` as a parameter (`lib.rs:117`, `:122`,
`rate_limiter.rs:94`, `nip98_replay.rs:29`), so pool sizing and TLS configuration
are the relay's concern. Only the pub/sub connections are opened from the crate's
own `redis_url` string (`lib.rs:105`).

### 2. Internal crate dependencies

| Dependency | Declared | What is actually used |
|---|---|---|
| `buzz-core` | `Cargo.toml:11` | `TenantContext` (`topic.rs:6`, `presence.rs:7`, `lib.rs:50`), `CommunityId` (`topic.rs:6`, `cache_invalidation.rs:17`, `conn_control.rs:20`) |
| `buzz-auth` | `Cargo.toml:12` | `RateLimiter` / `LimitType` / `RateLimitResult` / `rate_limit_key` / `ip_rate_limit_key` (`rate_limiter.rs:12-15`, `:110`, `:118`); `Nip98ReplayGuard` / `nip98_replay_key_for_scope` / `DEFAULT_REPLAY_TTL_SECS` / `MAX_REPLAY_TTL_SECS` (`nip98_replay.rs:8-14`, `:81`); `AuthError` (`rate_limiter.rs:13`, `nip98_replay.rs:9`) |
| `nostr` | `Cargo.toml:22` | `Event` + `JsonUtil` for wire (de)serialization (`publisher.rs:4`, `subscriber.rs:8`), `PublicKey` (`presence.rs:9`), `EventId` (`nip98_replay.rs:16`) |
| `uuid` | `Cargo.toml:17` | Channel and community ids (`topic.rs:7`) |
| `futures-util` | `Cargo.toml:23` | `StreamExt` for pub/sub stream consumption + sink/stream split (`subscriber.rs:7`, `:86`) |
| `serde` / `serde_json` | `Cargo.toml:15-16` | Control-message payloads (`cache_invalidation.rs:18`, `conn_control.rs:21`) |
| `tokio` | `Cargo.toml:14` | `broadcast`, `mpsc`, `Mutex`, `select!`, `sleep`, `spawn` |
| `tracing` | `Cargo.toml:20` | Structured logging throughout |
| `thiserror` | `Cargo.toml:21` | `PubSubError` derive (`error.rs:1`) |
| `chrono` | `Cargo.toml:18` | **Declared but no `chrono::` path appears in any source file** — unused |

`buzz-auth` is a genuine inbound coupling: this crate exists partly to provide the
Redis-backed implementations of two `buzz-auth` traits, which inverts the usual
layering (a lower-level transport crate depending on the auth crate). Noted as a
structural observation, not a defect — the alternative would be a `buzz-auth`
dependency on Redis.

### 3. Consumers

| Consumer | Manifest | Verified code usage |
|---|---|---|
| `buzz-relay` | `crates/buzz-relay/Cargo.toml` | Extensive — see table below |
| `buzz-admin` | `crates/buzz-admin/Cargo.toml` | Declared; no `buzz_pubsub::` path found in the relay-side grep sweep of `crates/**/*.rs` outside `buzz-relay` and `buzz-test-client` comments |
| `buzz-conformance` | `crates/buzz-conformance/Cargo.toml` | Declared; same — no verified call site |

Relay integration points (all verified by grep against `crates/**/*.rs`):

| Seam | Relay site |
|---|---|
| Manager construction | `buzz-relay/src/state.rs` (imports `state.rs:27`) |
| `run_conn_control_subscriber` spawn | `main.rs:366` |
| `subscribe_local` | `main.rs:822`, `handlers/event.rs:1644` |
| `subscribe_conn_control` | `main.rs:903`; dispatch `main.rs:908`, `:913` |
| `retain_topic` | `handlers/req.rs:256`, `handlers/event.rs:1683`, `:1687` |
| `release_topic` | `connection.rs:268`, `handlers/close.rs:21`, `handlers/req.rs:251`, `handlers/side_effects.rs:81` |
| `publish_cache_invalidation` | `state.rs:970` |
| `publish_conn_control` | `state.rs:1044`, `:1066` |
| `set_presence` / `clear_presence` | `handlers/event.rs:798` / `connection.rs:280`, `handlers/event.rs:793` |
| `get_presence_bulk` | `api/bridge.rs:1954` |
| `RedisRateLimiter` | import `state.rs:26`, field `state.rs:584`, construction `state.rs:712`, enforcement `connection.rs:593-648` via `admission.rs:14-34` |
| `RedisNip98ReplayGuard` | import `state.rs:27`, construction `state.rs:711`, two-pod tests `api/bridge.rs:2275-2276`, `:2304` |

Cross-node behaviour is additionally pinned from outside the crate by
`crates/buzz-test-client/tests/conformance_multitenant.rs:2371` and `:2484`, which
document that presence answers come from Redis via `get_presence_bulk` with **no DB
fallback**, and that community A's query must return only A's data.

### 4. Contract boundaries and coupling risks

- **Redis is a hard availability dependency for authenticated traffic.** A Redis
  outage makes `check_and_increment` return `AuthError::Internal`
  (`rate_limiter.rs:44-46`), which the relay maps to `AdmissionError::Unavailable`
  (`admission.rs:29-33`) and treats as denial — so every authenticated WS
  `EVENT`/`REQ`/`COUNT` is rejected (`connection.rs:612-621`). Fail-closed is the
  correct security posture, but it means Redis is on the critical path for reads,
  not just for fan-out.
- **Shared Redis across all tenants.** Isolation is by key prefix only
  (`topic.rs:13`), and two subscribers use cross-tenant `buzz:*` patterns
  (`cache_invalidation.rs:27`, `conn_control.rs:30`). Any process with access to the
  Redis instance sees every community's traffic.
- **No schema versioning on control payloads.** `CacheInvalidation`
  (`cache_invalidation.rs:57-80`) and `ConnControl` (`conn_control.rs:55-73`) are
  internally tagged with no version field and are not `#[non_exhaustive]`, so a
  rolling deploy that adds a variant produces `warn`-level deserialization failures
  on old pods (`cache_invalidation.rs:161-165`, `conn_control.rs:152-156`) —
  silently skipped, which for `ConnControl` means a ban may not be enforced on
  older pods until they restart. Mitigated by the DB backstop
  (`conn_control.rs:18-21`) but not by the transport.
- **Event payloads are whole Nostr events**, so Redis bandwidth scales with event
  size, and the pub/sub message size limit becomes an implicit cap on event size
  (`publisher.rs:31`).
