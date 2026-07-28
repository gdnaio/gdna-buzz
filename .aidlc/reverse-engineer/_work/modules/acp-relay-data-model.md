## Module: buzz-acp — relay client & observer (`crates/buzz-acp/src`)
### Aspect: Data Model

#### Wire types (hand-rolled, module-private)

| Type | Location | Shape |
|---|---|---|
| `RelayMessage` (private `enum`) | `relay.rs:471-495` | `Event { subscription_id: String, event: Box<Event> }`, `Ok { event_id: String, accepted: bool, message: String }`, `Eose { subscription_id }`, `Closed { subscription_id, message }`, `Notice { message }`, `Auth { challenge: String }` |
| `OkResponse` (private `struct`) | `relay.rs:3917-3921` | `event_id: String`, `accepted: bool`, `message: String` — used only by `wait_for_any_ok` |
| `RelayError` (`pub enum`, `thiserror`) | `relay.rs:436-459` | `WebSocket(Box<tungstenite::Error>)`, `Json(#[from] serde_json::Error)`, `AuthFailed(String)`, `NoAuthChallenge`, `ConnectionClosed`, `Timeout`, `Http(String)`, `UnexpectedMessage(String)` |
| `WsStream` (type alias) | `relay.rs:525` | `WebSocketStream<MaybeTlsStream<tokio::net::TcpStream>>` |

`RelayMessage` carries no `serde` derives; it is produced by the hand-written
`parse_relay_message` (`relay.rs:3531-3620`) from `Vec<serde_json::Value>` and
re-serialised back to text by `process_handshake_buffer` (`relay.rs:2412-2440`)
so buffered frames can be replayed through the single dispatch function.
`RelayMessage::Auth` is deliberately dropped on replay (`relay.rs:2433`).

`impl From<nostr::event::builder::Error> for RelayError` maps every signing
failure onto `AuthFailed` (`relay.rs:461-465`), so a typing-indicator signing
error and a NIP-42 rejection are indistinguishable at the type level.

#### Harness-facing types

| Type | Location | Fields |
|---|---|---|
| `BuzzEvent` (`pub`) | `relay.rs:427-433` | `channel_id: Uuid`, `event: nostr::Event` |
| `ChannelInfo` (`pub`) | `relay.rs:132-136` | `name: String`, `channel_type: String` — `channel_type` ∈ `"dm"` / `"private"` / `"stream"` / `"unknown"` |
| `RestClient` (`pub`, `Clone`) | `relay.rs:232-240` | `http: reqwest::Client`, `base_url: String`, `keys: Keys`, `auth_tag_json: Option<String>` |
| `HarnessRelay` (`pub`) | `relay.rs:533-555` | `event_rx: mpsc::Receiver<Option<BuzzEvent>>`, `observer_control_rx: Option<mpsc::Receiver<Event>>`, `cmd_tx: mpsc::Sender<RelayCommand>`, `http`, `relay_url`, `keys`, `auth_tag: Option<nostr::Tag>`, `bg_handle: Option<JoinHandle<()>>` |
| `RelayEventPublisher` (`pub`, `Clone`) | `relay.rs:556-560` | single field `cmd_tx` |

The `Option<BuzzEvent>` on `event_rx` is a sentinel channel: `None` means
"connection lost" (`relay.rs:811-813`, sent via `try_send` at `relay.rs:1563`, `:1646`, `:1822`, `:1903`, `:1942`, `:1975`). `next_event()` flattens it
(`relay.rs:813`), so callers cannot distinguish a lost socket from a closed
channel through the return value alone.

`channel_type_from_tags` (`relay.rs:138-159`) derives the type from `hidden`,
`private`, and `t` tags; a channel with no kind:39000 metadata is preserved as
`"unknown"` (`relay.rs:219-221`) rather than defaulting to `"stream"`.

#### Command channel

`RelayCommand` (private `enum`, `relay.rs:502-524`) is the only path from
`HarnessRelay` into the background task:

| Variant | Payload |
|---|---|
| `Subscribe` | `channel_id: Uuid`, `filter: ChannelFilter`, `replay_since: Option<u64>` |
| `Unsubscribe` | `channel_id: Uuid` |
| `Reconnect` | — |
| `Shutdown` | — |
| `SubscribeMembership` | — |
| `SubscribeObserverControls` | — |
| `PublishEvent` | `event: Box<Event>` |
| `SetStartupWatermark` | `ts: u64` |

`ChannelFilter` itself lives outside this module (`config.rs:477-483`):
`kinds: Option<Vec<u32>>` (`None` = wildcard) and `require_mention: bool`.

#### Subscription bookkeeping

`BgState` (`relay.rs:976-1059`) is the single mutable state object owned by the
background task. 19 fields:

| Field | Type | Purpose |
|---|---|---|
| `active_subscriptions` | `HashMap<Uuid, String>` | channel → `ch-<uuid>` sub id |
| `last_seen` | `HashMap<Uuid, u64>` | newest `created_at` per channel |
| `seen_ids` | `TwoGenDedup` | event-id dedup |
| `active_filters` | `HashMap<Uuid, ChannelFilter>` | replay on resubscribe |
| `membership_dropped_since` | `Option<u64>` | oldest backpressure-dropped membership ts |
| `membership_last_seen` | `Option<u64>` | newest enqueued membership ts |
| `membership_sub_active` | `bool` | intent flag |
| `observer_control_sub_active` | `bool` | intent flag |
| `channel_dropped_since` | `HashMap<Uuid, u64>` | per-channel drop floor |
| `proactive_resubscribe_needed` | `bool` | backpressure trigger |
| `startup_watermark` | `Option<u64>` | replay floor |
| `subscribe_since` | `HashMap<Uuid, u64>` | per-channel first-subscribe floor |
| `rate_limit_gate` | `Option<tokio::time::Instant>` | admission gate deadline |
| `rate_limited_pending` | `HashMap<Uuid, Instant>` | parked channel REQs |
| `membership_resub_needed` | `bool` | control-sub recovery flag |
| `observer_resub_needed` | `bool` | control-sub recovery flag |
| `gated_observer_pending` | `VecDeque<Box<Event>>` | parked kind:24200 frames |
| `observer_in_flight` | `VecDeque<Box<Event>>` | written-but-unacked frames |
| `gated_observer_dropped` | `u64` | overflow counter |
| `resubscribe_retry` | `HashSet<Uuid>` | failed-REQ retry set |
| `backoff_step` | `usize` | persisted backoff ladder position |

`clear_channel_state` (`relay.rs:1133-1140`) drops six of these per channel on
unsubscribe; `active_subscriptions` is removed separately by the caller
(`relay.rs:1420`, `:3524`).

`TwoGenDedup` (`relay.rs:935-972`) holds `current: HashSet<String>`,
`previous: HashSet<String>`, `limit: usize`. `insert` rotates `current` into
`previous` at `limit / 2` (`relay.rs:961-962`), so memory is bounded between
`SEEN_ID_LIMIT / 2` = 6,000 and `SEEN_ID_LIMIT` = 12,000 ids
(`relay.rs:45`). `remove` (`relay.rs:969-972`) un-dedups an id so a
backpressure-dropped event can be replayed.

Two derived-state helpers compute `since` without storing it:
`channel_since` picks `min(last_seen, channel_dropped_since)` and falls back to
`subscribe_since` then `startup_watermark` (`relay.rs:1115-1131`); the
membership equivalent is inlined four times (`relay.rs:1421-1428` in
`execute_connected_command`, `:1965-1972` in the drain loop, `:2301-2307` in
the CLOSED handler, `:2570-2575` in `resubscribe_after_reconnect`).

#### Engram payload types

`engram_fetch.rs` owns no types of its own — it consumes `buzz_core::engram`:

| Item | Source | Use in this module |
|---|---|---|
| `conversation_key(secret, pubkey) -> ConversationKey` | `buzz-core/src/engram.rs:136` | `engram_fetch.rs:76` |
| `d_tag(&ConversationKey, slug) -> String` | `buzz-core/src/engram.rs:144` | `engram_fetch.rs:77` |
| `Body` (`Core { profile }`, `Memory { slug, value }`, tombstone) | `buzz-core/src/engram.rs:158` | matched at `engram_fetch.rs:157-161` |
| `validate_and_decrypt(...) -> Result<Body, _>` | `buzz-core/src/engram.rs:488` | `engram_fetch.rs:131-137` |
| `select_head(I) -> Option<Event>` | `buzz-core/src/engram.rs:564` | `engram_fetch.rs:149` |
| `CORE_SLUG = "core"` | `buzz-core/src/engram.rs:20` | `engram_fetch.rs:77` |

The intermediate accumulator is `Vec<(Event, Body)>` (`engram_fetch.rs:118`),
with `candidates_seen: usize` and `last_decrypt_err: Option<String>`
(`engram_fetch.rs:119-120`) tracking the fail-closed distinction. The rendered
output is a bare `Option<String>` (`engram_fetch.rs:39-44`); there is no typed
section struct. `SECTION_LABEL = "Agent Memory — core"` (`engram_fetch.rs:23`)
and `ONBOARDING_NUDGE` (`engram_fetch.rs:29-31`) are `&'static str` constants.

#### Observer state

`observer.rs` holds an in-process bus, not relay state:

| Type | Location | Shape |
|---|---|---|
| `ObserverContext` (`pub`, `Clone`, `Default`) | `observer.rs:22-32` | `channel_id: Option<String>`, `session_id: Option<String>`, `turn_id: Option<String>`, `started_at: Option<String>` |
| `ObserverHandle` (`pub`, `Clone`) | `observer.rs:35-37` | `inner: Arc<ObserverInner>` |
| `ObserverInner` (private) | `observer.rs:39-43` | `tx: broadcast::Sender<ObserverEvent>`, `buffer: Mutex<VecDeque<ObserverEvent>>`, `seq: AtomicU64` |
| `ObserverEvent` (`pub`, `Clone`, `Serialize` camelCase) | `observer.rs:56-79` | `seq: u64`, `timestamp: String` (RFC3339), `kind: String`, `agent_index: Option<usize>`, `channel_id`, `session_id`, `turn_id`, `started_at` (skipped when `None`), `payload: serde_json::Value` |

`OBSERVER_BUFFER_CAP = 1_000` (`observer.rs:19`) sizes both the broadcast
channel (`observer.rs:46`) and the replay `VecDeque` (`observer.rs:51`).
`seq` starts at 1 (`observer.rs:52`) and is the monotonic key the publisher
uses to dedup the subscribe/snapshot overlap. The `Mutex` is a
`std::sync::Mutex`; poisoning is handled by logging and degrading to an empty
snapshot (`observer.rs:96-100`) or skipping the buffer append
(`observer.rs:129-131`) — the broadcast send still happens
(`observer.rs:134`).

Two constructors project into `ObserverContext`: `context_for` leaves
`started_at: None` (`observer.rs:138-149`), `context_for_turn` requires
`turn_id: String` and `started_at: String` (`observer.rs:152-164`).
