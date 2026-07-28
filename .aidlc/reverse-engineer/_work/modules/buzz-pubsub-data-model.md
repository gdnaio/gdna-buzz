## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Data Model

Source of truth read: `crates/buzz-pubsub/Cargo.toml` (26 lines) plus all 10 `.rs`
files — `src/lib.rs` (629), `src/subscriber.rs` (205), `src/cache_invalidation.rs`
(251), `src/conn_control.rs` (229), `src/presence.rs` (209), `src/nip98_replay.rs`
(202), `src/topic.rs` (197), `src/rate_limiter.rs` (121), `src/error.rs` (38),
`src/publisher.rs` (37). Total 2,144 lines (2,118 Rust).

The crate owns **no SQL, no migrations, and no persistent schema**. Its data model
is (1) a Redis keyspace, (2) in-process routing state, and (3) three JSON wire
payloads. It declares no `buzz-db` dependency (`Cargo.toml:11-24`).

---

### 1. Redis keyspace owned by this crate

All keys are prefixed by `BUZZ_PREFIX = "buzz"` (`topic.rs:13`). Community scoping
is achieved by embedding `ctx.community()` as the second path segment — key naming
is the *only* tenant separation mechanism (no separate Redis db index, no ACL).

| Key / channel | Redis type | TTL | Built at | Notes |
|---|---|---|---|---|
| `buzz:{community}:channel:{channel_uuid}` | pub/sub channel | n/a | `topic.rs:45-48` | Exact-channel event fan-out |
| `buzz:{community}:global` | pub/sub channel | n/a | `topic.rs:49` | Community-global events not routed to one channel |
| `buzz:{community}:cache-invalidate` | pub/sub channel | n/a | `cache_invalidation.rs:30-35` | Cross-pod cache-key drops |
| `buzz:{community}:conn-control` | pub/sub channel | n/a | `conn_control.rs:33-35` | Cross-pod disconnect commands |
| `buzz:{community}:presence:{pubkey_hex}` | string | `EX 90` | `presence.rs:19-25`, TTL const `presence.rs:16` | Value is a free-form status string |
| `buzz:{community}:ratelimit:{pubkey_hex}:{suffix}` | integer counter | `EX window_secs` | key from `buzz_auth::rate_limit::rate_limit_key` (`rate_limiter.rs:110`); format documented `rate_limiter.rs:82-84` | Fixed-window counter |
| `buzz:ratelimit:ip:{ip}:conn` | integer counter | `EX window_secs` | `buzz-auth/src/rate_limit.rs:213`, asserted `:312`; called from `rate_limiter.rs:118` | **Not community-scoped** — operator-global by design (`buzz-auth/src/rate_limit.rs:158`) |
| `buzz:{community}:nip98:{event_id_hex}` | string `"1"` | `EX ttl` (clamped) | `nip98_replay.rs:19`, key via `nip98_replay_key_for_scope` (`nip98_replay.rs:81`) | Replay seen-set marker |

Two subscribers use **cross-tenant wildcard patterns**, not per-community
subscriptions:
- `CACHE_INVALIDATION_PATTERN = "buzz:*:cache-invalidate"` (`cache_invalidation.rs:27`), `psubscribe` at `cache_invalidation.rs:139`
- `CONN_CONTROL_PATTERN = "buzz:*:conn-control"` (`conn_control.rs:30`), `psubscribe` at `conn_control.rs:130`

So every pod receives every community's invalidation and disconnect traffic and
demultiplexes by parsing the channel name (`parse_cache_invalidation_channel`
`cache_invalidation.rs:38-51`; `parse_conn_control_channel` `conn_control.rs:38-51`).
Event fan-out is the opposite: dynamically scoped `subscribe`/`unsubscribe` on the
exact topic keys with local interest (`subscriber.rs:96-100`, `:116-131`).

### 2. Key-parsing contract

`EventTopicKey::parse_redis_channel` (`topic.rs:53-99`) is a strict 3-or-4 segment
parser: prefix must equal `buzz` (`:58-63`), segment 2 must be a parseable UUID
(`:68-71`), segment 3 must be exactly `global` (no 4th segment) or `channel` plus a
UUID (no 5th segment) (`:77-95`). Anything else yields
`PubSubError::InvalidChannelKey`. Notably it **rejects** `presence:` keys — asserted
explicitly (`topic.rs:175`), which is what keeps the presence keyspace from ever
being interpreted as an event topic.

### 3. In-process structures

| Type | Definition | Shape / notes |
|---|---|---|
| `EventTopic` | `topic.rs:17-22` | `Channel(Uuid)` \| `Global`. `Copy`, `Hash`, `Eq` (`topic.rs:16`) |
| `EventTopicKey` | `topic.rs:26-31` | `{ community_id: CommunityId, topic: EventTopic }` — the composite routing identity |
| `ChannelEvent` | `lib.rs:62-69` | `{ community_id, topic, event: nostr::Event }` — the broadcast payload |
| `PubSubConfig` | `lib.rs:73-78` | `{ redis_url: String, unsubscribe_debounce: Duration }` |
| `PubSubManager` | `lib.rs:100-113` | 9 fields: pool, `redis_url`, `unsubscribe_debounce`, `desired_topics`, `subscription_tx`, `subscription_rx: Mutex<Option<..>>`, and 3 broadcast senders |
| `DesiredTopics` | `subscriber.rs:21` | `Arc<Mutex<HashMap<EventTopicKey, usize>>>` — refcount map, declared "source of truth across Redis reconnects" (`lib.rs:108`) |
| `SubscriptionCommand` | `subscriber.rs:26-31` | `pub(crate)`; `Subscribe(EventTopicKey)` \| `UnsubscribeIfIdle(EventTopicKey)` |
| `active_topics` | `subscriber.rs:88` | `HashSet<String>` of Redis channel names, **connection-local** — rebuilt from `desired_topics` on every reconnect (`subscriber.rs:90-101`) |
| `RedisRateLimiter` | `rate_limiter.rs:88-90` | Single field: `pool` |
| `RedisNip98ReplayGuard` | `nip98_replay.rs:23-25` | Single field: `pool` |

Channel capacities are all hardcoded to 4096 with no configuration knob: three
`broadcast::channel(4096)` (`lib.rs:126-128`) and one `mpsc::channel(4096)`
(`lib.rs:129`).

`PubSubManager` derives nothing and is **not `Clone`** (`lib.rs:100`); the three
`run_*_subscriber` methods take `self: Arc<Self>` (`lib.rs:148`, `:165`, `:175`), so
shared ownership is via `Arc` only.

### 4. Wire payloads

| Payload | Serialization | Definition |
|---|---|---|
| Event fan-out | `nostr::Event` JSON via `event.as_json()` (`publisher.rs:31`), decoded by `nostr::Event::from_json` (`subscriber.rs:151`) | Full signed Nostr event, not a delta |
| `CacheInvalidation` | `serde_json`, internally tagged `#[serde(tag = "op")]` (`cache_invalidation.rs:57`) | 4 variants: `Membership { channel_id: Uuid, pubkey: Vec<u8> }`, `AccessibleAll`, `Visibility { channel_id }`, `ChannelDeleted` (`cache_invalidation.rs:58-80`) |
| `ConnControl` | `serde_json`, internally tagged `#[serde(tag = "op")]` (`conn_control.rs:55`) | 2 variants: `DisconnectCommunity`, `DisconnectPubkey { pubkey: Vec<u8>, event_id: String, reason: String }` (`conn_control.rs:56-73`) |

Both control enums are wrapped for delivery with their parsed community:
`ScopedCacheInvalidation { community_id, invalidation }` (`cache_invalidation.rs:83-88`)
and `ScopedConnControl { community_id, command }` (`conn_control.rs:75-80`).

Neither control payload carries a version field, a timestamp, a nonce, or an
origin-pod id. Unknown `op` values fail deserialization and are skipped with a
`warn` (`cache_invalidation.rs:159-165`, `conn_control.rs:150-156`); a test pins
that an unknown command does not poison later messages (`conn_control.rs:209-217`).
`pubkey` is declared as "32 raw bytes" in the doc (`conn_control.rs:60`) but typed
`Vec<u8>` with **no length validation** on either send or receive.

### 5. Error model

`PubSubError` (`error.rs:5-28`) — 6 variants: `Redis` (from `redis::RedisError`),
`Pool` (from `deadpool_redis::PoolError`), `Serialization` (from
`serde_json::Error`), `BroadcastLagged(u64)`, `SubscriberStopped`,
`InvalidChannelKey(String)`. A `From<broadcast::error::RecvError>` impl maps
`Lagged(n) → BroadcastLagged(n)` and `Closed → SubscriberStopped`
(`error.rs:31-38`). The rate limiter and replay guard do **not** use this type —
they return `buzz_auth::error::AuthError` to satisfy the borrowed traits
(`rate_limiter.rs:35`, `nip98_replay.rs:37`).
