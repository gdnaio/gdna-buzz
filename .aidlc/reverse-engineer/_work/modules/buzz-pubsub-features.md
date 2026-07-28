## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Features

`buzz-pubsub` is a horizontal-scaling substrate, not a user-facing feature area.
It supplies six capabilities that make a multi-pod relay behave like one relay.

---

### F-PS-1 — Community-scoped realtime event fan-out

Cross-pod delivery of signed Nostr events. A pod publishes to
`buzz:{community}:channel:{id}` or `buzz:{community}:global` (`topic.rs:45-50`),
and every pod holding local interest re-broadcasts to its WebSocket sessions
through a `broadcast::channel(4096)` (`lib.rs:126`, forward at `subscriber.rs:165`).

Architecture as documented in the crate header (`lib.rs:8-16`): a
`deadpool-redis` pool serves all imperative commands, while a **dedicated**
`redis::aio::PubSub` connection — explicitly not from the pool because it is
stateful (`lib.rs:19-20`) — handles subscriptions. Obtained at
`subscriber.rs:85-87` and split into sink/stream so commands and messages can be
serviced from one `tokio::select!` (`subscriber.rs:109-171`).

Relay integration: `subscribe_local` consumers at `buzz-relay/src/main.rs:822` and
`handlers/event.rs:1644`; publishes flow through `PubSubManager::publish_event`
(`lib.rs:322-330`).

### F-PS-2 — Dynamic, refcounted subscription scoping

Rather than subscribing to a firehose, each pod subscribes only to topics with
live local interest. `retain_topic` / `release_topic` (`lib.rs:192`, `:215`)
maintain a refcount map (`subscriber.rs:21`) and emit `Subscribe` /
`UnsubscribeIfIdle` commands (`subscriber.rs:26-31`) to the pub/sub task.

Two properties make this safe:
- **Debounced teardown** (default 500 ms, `lib.rs:82`) with a re-check of the
  refcount at execution time (`subscriber.rs:123-130`), so subscription churn from
  a client re-issuing `REQ` doesn't thrash Redis.
- **Reconnect reconciliation** — the refcount map, not the connection, is the
  source of truth; a fresh connection re-subscribes from a snapshot before
  processing messages (`subscriber.rs:90-101`).

Wired into the relay's subscription lifecycle at `handlers/req.rs:251`, `:256`,
`handlers/close.rs:21`, `handlers/side_effects.rs:81`, `connection.rs:268`, and the
global-topic retains at `handlers/event.rs:1683`, `:1687`.

### F-PS-3 — Presence (online/away) with TTL

`SET buzz:{community}:presence:{pubkey_hex} <status> EX 90` (`presence.rs:36-43`),
TTL const `presence.rs:16`. Designed around a 30 s client heartbeat with 3× TTL
headroom so a single missed beat doesn't flap (`presence.rs:4-6`). Clean disconnect
deletes eagerly (`presence.rs:52-56`).

Reads: single (`presence.rs:62-72`) and bulk `MGET` returning `pubkey_hex → status`
for present keys only (`presence.rs:74-95`). The bulk path is what the relay
actually uses, at `buzz-relay/src/api/bridge.rs:1954`; multi-tenant isolation of
this exact call is pinned by conformance tests
(`crates/buzz-test-client/tests/conformance_multitenant.rs:2371`, `:2484`).
Writes come from the relay event handler (`handlers/event.rs:798`) and clears from
`connection.rs:280` / `handlers/event.rs:793`.

Capability limit: there is **no per-community presence index** (no `SET`/`ZSET` of
online members). `get_presence_bulk` requires the caller to already know the
candidate pubkey list, so "list everyone online in this community" is not
answerable from this crate.

### F-PS-4 — Cross-pod cache invalidation

Each pod keeps in-memory (moka) membership / accessible-channels / visibility
caches with a 10 s TTL; without this feature a membership change would stay stale on
every pod except the writer for up to that TTL (`cache_invalidation.rs:4-9`).

Four drop operations, each mirroring one relay-local `invalidate_*` call
(`cache_invalidation.rs:58-80`): `Membership { channel_id, pubkey }`,
`AccessibleAll`, `Visibility { channel_id }`, `ChannelDeleted`.

Delivery is a wildcard `psubscribe "buzz:*:cache-invalidate"`
(`cache_invalidation.rs:27`, `:139`) with per-message community demux
(`cache_invalidation.rs:38-51`, `:144-148`). Publisher: `buzz-relay/src/state.rs:970`.

The design invariant is that the message is a *pure key drop*, never an
authorization decision — the per-event access gate remains the enforcement point
and the next read re-fetches from the DB (`cache_invalidation.rs:11-14`).

### F-PS-5 — Cross-pod connection control (live ban enforcement)

Solves the "banned member's socket is on another pod" problem
(`conn_control.rs:3-6`). Two commands (`conn_control.rs:56-73`):
- `DisconnectPubkey { pubkey, event_id, reason }` — carries enough context to
  reproduce the same NIP-01 `OK` frame the origin pod sent, so the member learns
  why regardless of which pod held the socket (`conn_control.rs:62-65`).
- `DisconnectCommunity` — drops every socket bound to the carrying community.

Producers: `buzz-relay/src/state.rs:1034` (`DisconnectPubkey`), `:1066`
(`DisconnectCommunity`), both published via `publish_conn_control`
(`state.rs:1044`, `:1066`). Consumer: the subscriber spawned at
`buzz-relay/src/main.rs:366`, receiving on `subscribe_conn_control`
(`main.rs:903`) and dispatching both variants at `main.rs:908` and `:913`.

Deliberately a separate channel from F-PS-4 so that the cache channel's
idempotent-hint invariant is preserved (`conn_control.rs:12-18`). The DB ban row is
the durable backstop: a dropped message still results in refusal at the next auth
attempt (`conn_control.rs:18-21`).

### F-PS-6 — HA primitives borrowed by `buzz-auth` seams

Two trait implementations that let single-pod auth logic work across pods:

- **`RedisRateLimiter`** (`rate_limiter.rs:88-120`) — fixed-window `INCR`+`EXPIRE`
  via one atomic Lua script (`rate_limiter.rs:24-31`), with self-repair for
  TTL-less keys left by an older non-atomic implementation (`rate_limiter.rs:57-70`).
  Constructed at `buzz-relay/src/state.rs:712`, held as
  `admission_rate_limiter` (`state.rs:584`), and enforced for WS `EVENT`/`REQ`/`COUNT`
  at `buzz-relay/src/connection.rs:593-648`.
- **`RedisNip98ReplayGuard`** (`nip98_replay.rs:23-101`) — atomic set-if-absent
  seen-set for NIP-98 HTTP auth, described as the "§5 pre-build gate for
  multi-tenant HA replay protection" (`nip98_replay.rs:4-6`). Constructed at
  `buzz-relay/src/state.rs:711`; two-pod behaviour simulated in relay tests at
  `api/bridge.rs:2275-2276`.

### Documented-but-absent capability: typing indicators

The crate description advertises typing indicators —
`description = "Redis pub/sub fan-out, presence, and typing indicators for Buzz"`
(`Cargo.toml:8`) — and a doc comment reading "Typing indicator tracking in Redis."
sits at `lib.rs:43`. **No typing module, key, or function exists.** The module list
ends at `pub mod topic` (`lib.rs:42`), and the doc comment at `:43` is
mis-attached to `pub use error::PubSubError;` (`lib.rs:44`). A repo-wide grep for
typing-related Redis operations (`ZADD`, `ZREMRANGE`, `typing_key`, `mod typing`)
across `crates/**/*.rs` returns nothing. Feature F-PS-7 does not exist; see the
debt file.
