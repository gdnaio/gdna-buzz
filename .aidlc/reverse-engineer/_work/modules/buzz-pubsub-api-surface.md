## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: API Surface

`buzz-pubsub` exposes **no HTTP routes, no WebSocket handlers, and no CLI**. Its
entire API surface is a Rust library facade consumed by `buzz-relay`. Public items
were enumerated from all 10 `.rs` files.

Consumers declaring the dependency (`grep buzz-pubsub` across manifests):
`crates/buzz-relay/Cargo.toml`, `crates/buzz-admin/Cargo.toml`,
`crates/buzz-conformance/Cargo.toml`. Only `buzz-relay` exercises it in the code
paths verified below.

---

### 1. Module tree (`lib.rs:24-58`)

| Item | Line | Visibility |
|---|---|---|
| `pub mod cache_invalidation` | `lib.rs:25` | public |
| `pub mod conn_control` | `lib.rs:27` | public |
| `pub mod error` | `lib.rs:29` | public |
| `pub mod nip98_replay` | `lib.rs:31` | public |
| `pub use nip98_replay::RedisNip98ReplayGuard` | `lib.rs:32` | re-export |
| `pub mod presence` | `lib.rs:34` | public |
| `pub mod publisher` | `lib.rs:36` | public |
| `pub mod rate_limiter` | `lib.rs:38` | public |
| `pub mod subscriber` | `lib.rs:40` | public — but every item inside is `pub(crate)` |
| `pub mod topic` | `lib.rs:42` | public |
| `pub use error::PubSubError` | `lib.rs:44` | re-export |
| `pub use crate::topic::{channel_key, global_key, EventTopic, EventTopicKey}` | `lib.rs:58` | re-export |

`pub mod subscriber` (`lib.rs:40`) is declared public yet contains **zero public
items** — `DesiredTopics` (`subscriber.rs:21`), `SubscriptionCommand`
(`subscriber.rs:26`), and `run_subscriber` (`subscriber.rs:41`) are all
`pub(crate)`. The module is therefore an empty public namespace.

`channel_key` / `global_key` are reachable by **three distinct paths**:
`buzz_pubsub::channel_key` (re-export `lib.rs:58`), `buzz_pubsub::topic::channel_key`
(`topic.rs:103`), and `buzz_pubsub::publisher::channel_key` (`publisher.rs:12`) —
the last being a one-line delegate to the first (`publisher.rs:12-14`, `:17-19`).

### 2. `PubSubManager` — the primary facade (`lib.rs:100-368`)

| Method | Line | Signature summary | Verified production caller |
|---|---|---|---|
| `new` | `lib.rs:117` | `(&str, Pool) -> Result<Self, PubSubError>` | `buzz-relay/src/state.rs` construction path |
| `with_config` | `lib.rs:122` | `(PubSubConfig, Pool) -> Result<Self, PubSubError>` | tests only (`lib.rs:513`, `:594`) |
| `run_subscriber` | `lib.rs:148` | `(self: Arc<Self>)` — never returns | `buzz-relay/src/main.rs` (spawned) |
| `run_cache_invalidation_subscriber` | `lib.rs:165` | `(self: Arc<Self>)` — never returns | `buzz-relay/src/main.rs` |
| `run_conn_control_subscriber` | `lib.rs:175` | `(self: Arc<Self>)` — never returns | `buzz-relay/src/main.rs:366` |
| `subscribe_local` | `lib.rs:184` | `-> broadcast::Receiver<ChannelEvent>` | `buzz-relay/src/main.rs:822`, `handlers/event.rs:1644` |
| `retain_topic` | `lib.rs:192` | `(&TenantContext, EventTopic)` | `handlers/req.rs:256`, `handlers/event.rs:1683`, `:1687` |
| `release_topic` | `lib.rs:215` | `(&TenantContext, EventTopic)` | `connection.rs:268`, `handlers/close.rs:21`, `handlers/req.rs:251`, `handlers/side_effects.rs:81` |
| `topic_refcount` | `lib.rs:248` | `-> usize` | **none** — in-crate tests only |
| `subscribe_cache_invalidations` | `lib.rs:259` | `-> broadcast::Receiver<ScopedCacheInvalidation>` | `buzz-relay/src/main.rs` |
| `subscribe_conn_control` | `lib.rs:264` | `-> broadcast::Receiver<ScopedConnControl>` | `buzz-relay/src/main.rs:903` |
| `publish_cache_invalidation` | `lib.rs:272` | `-> Result<i64, PubSubError>` | `buzz-relay/src/state.rs:970` |
| `publish_conn_control` | `lib.rs:292` | `-> Result<i64, PubSubError>` | `buzz-relay/src/state.rs:1044`, `:1066` |
| `publish_event` | `lib.rs:322` | `(&TenantContext, EventTopic, &nostr::Event) -> Result<i64, PubSubError>` | relay event ingest |
| `set_presence` | `lib.rs:332` | `(&TenantContext, &PublicKey, &str)` | `handlers/event.rs:798` |
| `clear_presence` | `lib.rs:342` | — | `connection.rs:280`, `handlers/event.rs:793` |
| `get_presence` | `lib.rs:351` | `-> Result<Option<String>, _>` | — (bulk variant is used instead) |
| `get_presence_bulk` | `lib.rs:360` | `-> Result<HashMap<String,String>, _>` | `api/bridge.rs:1954` |

All 18 methods return either `()`, a `broadcast::Receiver`, `usize`, or
`Result<_, PubSubError>`; the three `publish_*` methods return the Redis
subscriber count as `i64` (`lib.rs:279`, `:299`, `publisher.rs:27`).

`topic_refcount` (`lib.rs:248`) is documented "for tests and metrics" and has
**zero non-test callers** — no metric is exported from it.

### 3. `PubSubConfig` (`lib.rs:73-97`)

`DEFAULT_UNSUBSCRIBE_DEBOUNCE = 500ms` (`lib.rs:82`); `new` (`lib.rs:85`);
builder `with_unsubscribe_debounce` (`lib.rs:93`). The builder has **zero
production callers** — only `lib.rs:514`, `:596`, `:621` (tests). Production
therefore always runs the 500 ms default via `PubSubManager::new` → `with_config`
(`lib.rs:117-119`).

### 4. `topic` module (`topic.rs`)

`BUZZ_PREFIX` (`:13`), `EventTopic` (`:17`), `EventTopicKey` (`:26`),
`EventTopicKey::from_context` (`:35`), `::redis_channel` (`:43`),
`::parse_redis_channel` (`:53`), free fns `channel_key` (`:103`), `global_key`
(`:108`). 8 public items.

### 5. `presence` module (`presence.rs`)

`PRESENCE_TTL_SECS: u64 = 90` (`:16`), `presence_key` (`:19`), `set_presence`
(`:28`), `clear_presence` (`:47`), `get_presence` (`:62`), `get_presence_bulk`
(`:74`). All four operations take `&Pool` directly, so callers may bypass
`PubSubManager` entirely — the relay's own presence integration test does exactly
that (`lib.rs:477-508`).

### 6. `cache_invalidation` module (`cache_invalidation.rs`)

`CACHE_INVALIDATION_SUFFIX` (`:23`), `CACHE_INVALIDATION_PATTERN` (`:27`),
`cache_invalidation_channel` (`:30`), `parse_cache_invalidation_channel` (`:38`),
`CacheInvalidation` (`:58`), `ScopedCacheInvalidation` (`:83`),
`run_cache_invalidation_subscriber` (`:100`). 7 public items.

### 7. `conn_control` module (`conn_control.rs`)

`CONN_CONTROL_SUFFIX` (`:26`), `CONN_CONTROL_PATTERN` (`:30`),
`conn_control_channel` (`:33`), `parse_conn_control_channel` (`:38`), `ConnControl`
(`:56`), `ScopedConnControl` (`:75`), `run_conn_control_subscriber` (`:90`).
7 public items. Both `ConnControl` variants have live producers and a consumer:
`DisconnectPubkey` built at `buzz-relay/src/state.rs:1034`, `DisconnectCommunity`
at `:1066`, both dispatched in `buzz-relay/src/main.rs:908` and `:913`.

### 8. Trait implementations exported to `buzz-auth` seams

| Impl | Line | Trait | Methods |
|---|---|---|---|
| `RedisRateLimiter` | `rate_limiter.rs:99` | `buzz_auth::rate_limit::RateLimiter` | `check_and_increment` (`:100`), `check_ip_connection` (`:112`) |
| `RedisNip98ReplayGuard` | `nip98_replay.rs:34` | `buzz_auth::nip98_replay::Nip98ReplayGuard` | `try_mark_in_scope` (`:35`) — returns a boxed pinned future, i.e. the trait is object-safe rather than `async fn` |

`RedisRateLimiter::new` (`rate_limiter.rs:94`) is constructed once at
`buzz-relay/src/state.rs:712` and stored as
`admission_rate_limiter: Arc<RedisRateLimiter>` (`state.rs:584`, import `state.rs:26`).
`RedisNip98ReplayGuard::new` (`nip98_replay.rs:29`) is constructed at
`buzz-relay/src/state.rs:711` (import `state.rs:27`) and additionally instantiated
in relay tests as two simulated pods (`api/bridge.rs:2275-2276`, `:2304`).

`check_ip_connection` (`rate_limiter.rs:112`) has **no production caller anywhere**.
The only other implementations are the trait declaration
(`buzz-auth/src/rate_limit.rs:188`, default/blanket at `:234`) and a `#[cfg(test)]`
`StubLimiter` inside `buzz-relay/src/admission.rs:85`.

### 9. Aggregate

Roughly 55 public items across 9 public modules. No `#[non_exhaustive]` markers on
any public enum or struct, so adding a `CacheInvalidation` or `ConnControl` variant,
or a `PubSubError` variant, is a semver-breaking change for downstream matchers.
