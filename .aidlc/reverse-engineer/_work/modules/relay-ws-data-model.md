## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Data Model

Scope: `connection.rs` (893), `subscription.rs` (1562), `handlers/{mod,event,req,count,close,auth}.rs` (62 / 2461 / 1946 / 281 / 35 / 350) = 7,590 LOC.

---

#### 1. `AuthState` — per-connection NIP-42 state machine

`connection.rs:37-48`. Three states, no others; `#[derive(Debug, Clone)]` (`:36`).

| State | Payload | Set at | Exit transitions |
|---|---|---|---|
| `Pending { challenge: String }` | the random challenge string sent to the client | `connection.rs:171-173` (at construction, before any frame is read) | → `Authenticated` (`auth.rs:278`), → `Failed` (`auth.rs:172`, `:206`, `:230`, `:287`) |
| `Authenticated(AuthContext)` | `buzz_auth::AuthContext` | `auth.rs:278` | **terminal** — a second AUTH is refused (`auth.rs:49-57`) |
| `Failed` | none | `auth.rs:172` (ban / ban-check error), `:206` (allowlist), `:230` (relay membership), `:287` (NIP-42 verify failure) | **terminal** — further AUTH refused (`auth.rs:58-66`) |

`Failed` is *sticky for the life of the socket*: `handle_auth` reads the state first (`auth.rs:45-74`) and only proceeds from `Pending`. There is no retry path. The ban-seam code comments call this out explicitly at `auth.rs:98-112` ("without … pinning `Failed` for the connection's life on a false premise") — hence the deliberate `Banned` vs `DbError` split at `auth.rs:113-117`.

`AuthContext` fields the WS path consumes (`crates/buzz-auth/src/lib.rs:64-79`):

| Field | Type | Read by |
|---|---|---|
| `pubkey` | `nostr::PublicKey` | `event.rs:617`, `req.rs:63`, `count.rs:42`, `connection.rs:607`, `connection.rs:277` |
| `scopes` | `Vec<Scope>` | `event.rs:618` (→ `:658`, `:676`), `req.rs:54` |
| `channel_ids` | `Option<Vec<Uuid>>` | `event.rs:619`, `req.rs:74`, `count.rs:42` |
| `auth_method` | `AuthMethod` | `auth.rs:191` (allowlist gate applies only to `Nip42`) |
| `agent_owner_pubkey` | `Option<PublicKey>` | `event.rs:989` (observer fast path), `connection.rs:607` (agent-vs-human rate tier) |

**Delta vs doc.** `channel_ids` is documented as "reserved for future per-channel access control" (`buzz-auth/src/lib.rs:69`). It is **not** reserved — it is load-bearing today: it narrows `accessible_channels` at `req.rs:87-88`, `count.rs:94-96`, and bounds the request-local repair at `req.rs:137-139` / `count.rs:145-149`.

---

#### 2. `ConnectionState` — every field

`connection.rs:53-80`. Deliberately split by access pattern (doc `:50-52`).

| Field | Type | Lock | Purpose / notes |
|---|---|---|---|
| `conn_id` | `Uuid` | — | `Uuid::new_v4()` at `connection.rs:127`; primary key into `ConnectionManager` and `SubscriptionRegistry` |
| `tenant` | `TenantContext` | — | resolved from the HTTP `Host` **before** the upgrade (`router.rs:286-300`); doc `connection.rs:57-60` says "never overridable by client-supplied input" — verified: no write path exists after construction |
| `remote_addr` | `SocketAddr` | — | stored but **only** used for the two `info!` log lines (`connection.rs:183`, `:289`). Never used for authorization or rate limiting |
| `auth_state` | `RwLock<AuthState>` | tokio `RwLock` | read on every EVENT/REQ/COUNT/AUTH; written only by `handle_auth` |
| `subscriptions` | `Arc<Mutex<HashMap<String, Vec<Filter>>>>` | tokio `Mutex` | alias `ConnectionSubscriptions` (`connection.rs:30`). **Second copy** of the filters already in `sub_registry` |
| `send_tx` | `mpsc::Sender<WsMessage>` | — | capacity = `config.send_buffer_size` (`connection.rs:159`), default **1000** (`config.rs:459-462`) |
| `ctrl_tx` | `mpsc::Sender<WsMessage>` | — | capacity hard-coded **8** (`connection.rs:162`). Pong / Close / ban-reason / restart-close |
| `cancel` | `CancellationToken` | — | created in `handle_connection` (`connection.rs:128`), shared with `ConnEntry` |
| `backpressure_count` | `Arc<AtomicU8>` | atomic | shared with `ConnEntry.backpressure_count` (`state.rs:54`, `:212`) so direct sends and fan-out sends decrement one counter |
| `grace_limit` | `u8` | — | copied from `config.slow_client_grace_limit` (`connection.rs:179`), default **15** (`config.rs:470-473`) |

Not stored on `ConnectionState`: the authenticated pubkey (it lives separately in `ConnEntry.authenticated_pubkey`, `state.rs:56`, set at `auth.rs:279-281`). Fan-out reads the `ConnEntry` copy (`event.rs:146`, `:184`, `:460`), never `ConnectionState`.

Task-local state not on the struct:
- `missed_pongs: Arc<AtomicU8>` — `connection.rs:218`, shared between `heartbeat_loop` (`:389`) and `recv_loop` (`:462`).
- The owned semaphore permit — `connection.rs:149`, dropped at `:287`.

---

#### 3. `SubscriptionRegistry` — six concurrent maps

`subscription.rs:44-59`. `#[derive(Debug, Default)]`; `new()` (`:66`) is `Self::default()`.

##### 3.1 Authoritative store

```
subs: DashMap<ConnId, HashMap<SubId, SubEntry>>          // subscription.rs:49
SubEntry = (Vec<Filter>, CommunityId, Option<Uuid>)      // subscription.rs:16
ConnId   = Uuid                                          // subscription.rs:12
SubId    = String                                        // subscription.rs:14
```

`SubEntry` carries the **server-resolved** community and the derived channel scope alongside the filters. Doc `:47-48` claims this "enables O(1) targeted index removal and gives lifecycle code the exact Redis topic to release" — verified at `remove_from_index` (`:391-479`) and at the three `release_topic` sites (`req.rs:251`, `close.rs:21`, `connection.rs:268`).

##### 3.2 Five fan-out indexes

| Index | Key | Value | Populated when | Sites |
|---|---|---|---|---|
| `channel_kind_index` | `(CommunityId, IndexKey{channel_id, kind})` | `Vec<(ConnId, SubId)>` | `channel_id.is_some()` and every filter has a non-empty `kinds` | insert `:107-111`, read `:278`, remove `:426-433` |
| `channel_wildcard_index` | `(CommunityId, Uuid)` | `Vec<(ConnId, SubId)>` | `channel_id.is_some()` and ≥1 filter has **no** `kinds` | insert `:90-93`, read `:284`, remove `:405-411` |
| `global_kind_index` | `(CommunityId, Kind)` | `Vec<(ConnId, SubId)>` | `channel_id.is_none()`, kinds present, **and** the p-kind key extraction returned `None` | insert `:136-140`, read `:305-308`, remove `:459-467` |
| `global_p_kind_index` | `GlobalPKindIndexKey{community_id, kind, p}` (`:27-32`, private) | `Vec<(ConnId, SubId)>` | `channel_id.is_none()` and **every** filter has non-empty `kinds` **and** a non-empty `#p` (`:486-520`) | insert `:120-123`, read `:294-302`, remove `:442-450` |
| `global_wildcard_index` | `CommunityId` | `Vec<(ConnId, SubId)>` | `channel_id.is_none()`, ≥1 filter kindless, no p-kind key | insert `:128-131`, read `:314`, remove `:452-458` |

`IndexKey` is `pub` (`:19-25`) — `{channel_id: Uuid, kind: Kind}`, `Hash + Eq`. `GlobalPKindIndexKey` is private (`:28`). `RemovedSubscription` is `pub` (`:35-41`) — `{community_id, channel_id: Option<Uuid>}`, `Copy`.

##### 3.3 Index-selection rule (`extract_kinds_from_filters`, `:546-567`)

Tri-state over the whole filter *set* (NIP-01 OR semantics, doc `:539-544`):

| Return | Meaning | Index used |
|---|---|---|
| `None` | at least one filter omits `kinds` → subscription is a wildcard | `*_wildcard_index` |
| `Some(vec![])` | every filter had `kinds: []` | **none** — deliberately unindexed (`:95-100`, `:415-418`); matches nothing |
| `Some(kinds)` | union of all kinds across filters | one entry per kind in `*_kind_index` |

`extract_global_p_kind_index_keys` (`:486-520`) returns `None` (falling back to the generic global indexes) if **any** filter lacks `kinds`, or lacks a `#p` tag; it `continue`s past a filter with empty `kinds` (`:494-496`) — so a `[{kinds:[]}, {kinds:[9], #p:[x]}]` pair still lands only on the p-kind index. `p` values are stored as raw `String`, not normalized (no lowercasing), while `event_p_tag_values` (`:522-537`) also returns raw strings — so p-index matching is **case-sensitive** on hex.

##### 3.4 No refcounting in the registry

There is **no** per-topic refcount inside `SubscriptionRegistry`. Refcounting lives one layer out, in `buzz_pubsub`: `desired_topics: HashMap<EventTopicKey, usize>` with `retain_topic` (`buzz-pubsub/src/lib.rs:192-208`) and `release_topic` (`:215-232`). The registry's contribution is returning `Option<RemovedSubscription>` / `Vec<RemovedSubscription>` so callers can issue exactly one matching `release_topic`.

`Vec<(ConnId, SubId)>` index buckets are linear-scanned on removal (`retain`, `:407`, `:429`, `:446`, `:455`, `:464`) and emptied buckets are removed from the DashMap (`:409-410`, `:431-432`, `:448-449`, `:456-457`, `:465-466`) — so no empty-bucket leak, but a channel+kind with N subscribers costs O(N) per removal.

---

#### 4. Per-connection limits and counters

| Limit | Value | Definition | Enforced at | Configurable |
|---|---|---|---|---|
| Subscriptions per connection | **1024** | `req.rs:26` `MAX_SUBSCRIPTIONS` | `req.rs:66` (against `conn.subscriptions.len()`) | no |
| Filters per REQ / COUNT | **10** | `protocol.rs:14` `MAX_FILTERS_PER_REQ` | `protocol.rs:93-98`, `:135-140` | no |
| Sub-id length | **256 B** | `protocol.rs:11` `MAX_SUB_ID_LENGTH` | `protocol.rs:87-91`, `:128-133` | no |
| Inbound frame bytes | **524288** (512 KiB) | `config.rs:14` `DEFAULT_MAX_FRAME_BYTES` | parser `router.rs:340-342`; app-level re-check `connection.rs:421`, `:440` | `BUZZ_MAX_FRAME_BYTES` |
| Historical rows per filter | **2000** | `req.rs:25` `MAX_HISTORICAL_LIMIT` | `req.rs:885-886`, search `:538-539` | no |
| Concurrent per-REQ DB queries | **4** | `req.rs:35` `FILTER_QUERY_CONCURRENCY` (compile-time bounded 2..=8 at `:41`) | `req.rs:314` | no |
| Search pages per filter | **10** × 100 rows | `req.rs:421` `MAX_SEARCH_PAGES`, per-page `:589` | `req.rs:594` | no |
| COUNT fallback candidate rows | **5000** (+1 probe) | `req.rs:753` `COUNT_FALLBACK_CANDIDATE_LIMIT` | `req.rs:756-765`, `count.rs:173`/`:243` | no |
| Outbound data-frame batch | **64** | `connection.rs:33` `MAX_WS_SEND_BATCH` | `connection.rs:351` | no |
| Send-buffer depth | **1000** msgs | `config.rs:459-462` | `connection.rs:159` | `BUZZ_SEND_BUFFER` |
| Control-buffer depth | **8** msgs | hard-coded `connection.rs:162` | — | no |
| Backpressure strikes | **15** | `config.rs:470-473` | `connection.rs:100` / `state.rs:464` | `BUZZ_SLOW_CLIENT_GRACE_LIMIT` |
| Missed pongs | **3** | `connection.rs:389-393` | same | no |
| Auth grace | **5 s** | `connection.rs:27` `AUTH_TIMEOUT` | `connection.rs:232` | no |
| Observer telemetry | **100/s per (community, agent)** | `event.rs:894-916` | `event.rs:1036` | no |
| Observer freshness | **±300 s** | `event.rs:952` | same | no |

Counters: `backpressure_count` (`AtomicU8`, reset on any successful send — `connection.rs:92`, `state.rs:456`), `missed_pongs` (`AtomicU8`, reset on Pong — `connection.rs:462`), `observer_rate_limiter` (`DashMap<(CommunityId,[u8;32]), (u32, Instant)>`, `state.rs:589`, ctor `:773`).

**`AtomicU8` overflow risk:** `grace_limit` is a `u8` from an unvalidated `parse()` (`config.rs:470-473`). Setting `BUZZ_SLOW_CLIENT_GRACE_LIMIT` > 255 makes the parse fail and silently fall back to 15; setting it to `0` makes `count >= grace_limit` true on the *first* full buffer (`connection.rs:100`) — instant disconnect. There is no `>0` filter, unlike `max_frame_bytes` (`config.rs:467`).

---

#### 5. How filters are stored for matching

Filters are `nostr::Filter` values, stored **twice**:

1. `ConnectionState.subscriptions: HashMap<SubId, Vec<Filter>>` — written at `req.rs:236-239`, removed at `close.rs:16`. Its **only** read is the `len()`/`contains_key` cap check at `req.rs:66`. `ConnectionManager::subscriptions_for` (`state.rs:383`) hands the same `Arc` to `side_effects.rs:71`.
2. `SubscriptionRegistry.subs[conn_id][sub_id].0` — written at `subscription.rs:79-82`, and this is the copy actually used for matching (`push_match` → `filters_match`, `:377`).

Matching is a **full linear predicate evaluation** per candidate: the index narrows candidates by `(community, channel, kind)` or `(community, kind, #p)`, then `buzz_core::filter::filters_match` (`buzz-core/src/filter.rs:11-13` → `filter_match_one` `:35-103`) re-checks kinds, authors, since, until, `ids` (**prefix** match, `:64-68`), and every generic tag. There is no precompiled/normalized filter representation.

`filter_match_one`'s `#h` handling (`buzz-core/src/filter.rs:69-102`) has an `h`-specific fallback: if the event carries no `h` tag at all, `StoredEvent.channel_id` is used instead — so reactions/deletions that derive their channel match `#h` filters. If the event *does* carry `h` tags and none match, it is strictly rejected (`:98-100`).

Per-filter clones: `register_scoped` clones the whole `Vec<Filter>` (`subscription.rs:81`) and each `sub_id` once per index key (`:92`, `:110`, `:122`, `:130`, `:138`). `req.rs` clones the filter vec twice more (`:238`, `:245`). Memory per subscription therefore scales with `filters × (1 + copies) + kinds × sub_id_len`, with `sub_id_len` up to 256 B and `kinds` unbounded within one filter.

---

#### 6. Shared registries this group writes into (`state.rs`)

| Structure | Type | Written by this group |
|---|---|---|
| `ConnectionManager.connections` | `DashMap<Uuid, ConnEntry>` (`state.rs:183`) | `register` `connection.rs:199-212`; `deregister` `:271`; `set_authenticated_pubkey` `auth.rs:279-281` |
| `ConnEntry` | `state.rs:41-58` — `{tx, ctrl_tx, cancel, community_id, backpressure_count, subscriptions, authenticated_pubkey: Arc<std::sync::RwLock<Option<Vec<u8>>>>, grace_limit}` | as above |
| `CommunityConnectionRegistry.connections` | `DashMap<Uuid, (CommunityId, CancellationToken)>` (`state.rs:66`) | via `run_registered_community_connection` `connection.rs:132-140` |
| `local_event_ids` | `moka::sync::Cache<(CommunityId,[u8;32]), ()>`, cap 10 000, TTL 60 s (`state.rs:734-739`) | `mark_local_event` `event.rs:394`, `:824`, `:852`, `:1046`; read+invalidate `event.rs:278-280` |
| `observer_rate_limiter` | `DashMap<(CommunityId,[u8;32]), (u32, Instant)>` (`state.rs:589`, ctor `:773`) | `event.rs:894-916` |
| `observer_owner_cache` | `moka::sync::Cache<(CommunityId, Vec<u8>, Vec<u8>), bool>`, cap 1000, TTL 300 s (`state.rs:782-787`) | `event.rs:995-1013` |

**Unbounded structure:** `observer_rate_limiter` is a plain `DashMap` with **no capacity bound and no eviction** (`state.rs:589`, ctor `:773`, `event.rs:897-901` only ever calls `entry().or_insert()`). One entry per distinct `(community, agent pubkey)` accumulates for process lifetime — 40 B key + 24 B value per observed agent key, never reclaimed. Compare `local_event_ids` / `observer_owner_cache`, which are moka caches with explicit caps.
