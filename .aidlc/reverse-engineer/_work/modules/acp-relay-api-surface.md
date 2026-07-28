## Module: buzz-acp — relay client & observer (`crates/buzz-acp/src`)
### Aspect: API Surface

#### `HarnessRelay` — public methods

| Method | Location | Signature notes |
|---|---|---|
| `connect` | `relay.rs:596-601` | `async (relay_url: &str, keys: &Keys, agent_pubkey_hex: &str, auth_tag: Option<nostr::Tag>) -> Result<Self, RelayError>`; spawns the background task |
| `discover_channels` | `relay.rs:656` | `async (&self) -> Result<HashMap<Uuid, ChannelInfo>, RelayError>` |
| `rest_client` | `relay.rs:719` | `(&self) -> RestClient` |
| `subscribe_channel` | `relay.rs:735-739` | `async (&mut self, Uuid, ChannelFilter)` → delegates to `subscribe_channel_from(.., None)` |
| `subscribe_channel_from` | `relay.rs:749-754` | adds `replay_since: Option<u64>` |
| `subscribe_membership_notifications` | `relay.rs:768` | `async (&mut self)` |
| `subscribe_observer_controls` | `relay.rs:777` | `async (&mut self)` |
| `take_observer_control_rx` | `relay.rs:786` | `(&mut self) -> Option<mpsc::Receiver<Event>>` — one-shot take |
| `event_publisher` | `relay.rs:791` | `(&self) -> RelayEventPublisher` |
| `unsubscribe_channel` | `relay.rs:798` | `async (&mut self, Uuid)` |
| `next_event` | `relay.rs:806` | `async (&mut self) -> Option<BuzzEvent>` |
| `publish_event` | `relay.rs:821` | `async (&self, Event)`; marked `#[allow(dead_code)]` at `relay.rs:820` |
| `try_publish_event` | `relay.rs:834` | `(&self, Event) -> Result<(), RelayError>` — `try_send`, never blocks |
| `build_typing_event` | `relay.rs:843-848` | `(&self, Uuid, root: Option<&str>, parent: Option<&str>) -> Result<Event, RelayError>` |
| `set_startup_watermark` | `relay.rs:877` | `async (&self, ts: u64)` |
| `reconnect` | `relay.rs:886` | `async (&mut self)` |
| `shutdown` | `relay.rs:900` | `async (self)` — consumes, waits 5 s, then aborts |

`Drop for HarnessRelay` (`relay.rs:915-923`) sends `Shutdown` via `try_send`
then `abort()`s the task immediately — no close frame guarantee.

All the mutating subscribe/unsubscribe methods take `&mut self` but only push a
`RelayCommand`; every one returns `Ok(())` on a successful enqueue, not on a
successful REQ. `RelayError::ConnectionClosed` here means "background task
gone", not "socket down".

#### `RelayEventPublisher`

| Method | Location |
|---|---|
| `publish_event` | `relay.rs:563-573` — `async (&self, Event) -> Result<(), RelayError>` |
| `test_pair` | `relay.rs:575-588`, `#[cfg(test)]` only |

#### `RestClient` — the HTTP bridge surface

This module **does** use the relay's HTTP bridge for every read, in addition to
the WebSocket:

| Method | Location | HTTP |
|---|---|---|
| `query` | `relay.rs:399-406` | `POST {base_url}/query`, body = JSON array of `nostr::Filter` |
| `submit_event` | `relay.rs:411-423` | `POST {base_url}/events`, body = signed event JSON |
| `bridge_post` (private) | `relay.rs:368-393` | adds `Authorization: Nostr <b64>` and optional `x-auth-tag` |
| `request_with_retry` (private) | `relay.rs:314-361` | 1 attempt + 3 retries on 429/502/503/504/timeout/connect |
| `sign_nip98` / `nip98_header` (private) | `relay.rs:266-305` | kind:27235 NIP-98 with `u`, `method`, `nonce`, optional `payload` sha256 tags |

`base_url` is derived from the WebSocket URL by string replacement
(`relay_ws_to_http`, `relay.rs:3470-3475`). Auth is NIP-98 (signed kind:27235),
**not** the unauthenticated `X-Pubkey` header path — so the
`BUZZ_REQUIRE_AUTH_TOKEN=false` impersonation weakness on
`POST /events|/query|/count` is not exercised by this module. The optional
`x-auth-tag` header carries the NIP-OA attestation JSON (`relay.rs:389-391`,
built at `relay.rs:724-727`).

#### Event kinds published

| Kind | Constant | Where built | Transport |
|---|---|---|---|
| 22242 (`Kind::Authentication`) | `nostr::Kind::Authentication` | `relay.rs:3452` / `EventBuilder::auth` at `relay.rs:3457` | WS `["AUTH", ev]` |
| 27235 (`Kind::HttpAuth`) | `nostr::Kind::HttpAuth` | `relay.rs:292` | HTTP `Authorization` header |
| 20002 | `KIND_TYPING_INDICATOR` (`buzz-core/src/kind.rs:331`) | `relay.rs:866` | WS `["EVENT", ev]` |
| 24200 | `KIND_AGENT_OBSERVER_FRAME` (`buzz-core/src/kind.rs:333`) | built in `lib.rs` via `buzz_sdk::build_agent_observer_frame`; routed through `RelayCommand::PublishEvent` and detected by kind at `relay.rs:1234`, `:1310`, `:1470`, `:1497` | WS `["EVENT", ev]` |

Anything else reaching `PublishEvent` (kind:20001 presence from `lib.rs:85`,
kind:44200 metrics via `RestClient::submit_event`) is treated as droppable
ephemera by the rate-limit gate — see `relay.rs:1483-1493` for the stated
invariant.

#### REQ filter shapes

| Subscription id | Builder | Filter |
|---|---|---|
| `ch-<uuid>` (`channel_sub_id`, `relay.rs:3478`) | `send_subscribe`, `relay.rs:3160-3222` | `kinds` **only if** `filter.kinds.is_some()` (`relay.rs:3172-3175`); `#h: [channel_uuid]` always (`:3178`); `#p: [agent_pubkey]` only when `require_mention` (`:3181-3183`); `since` always (`:3187-3194`) |
| `membership-notif` (`relay.rs:497`) | `send_membership_subscribe`, `relay.rs:3227-3270` | `kinds: [44100, 44101]` via `KIND_MEMBER_ADDED_NOTIFICATION` / `KIND_MEMBER_REMOVED_NOTIFICATION` (`:3233-3239`); `#p: [agent_pubkey]` (`:3240`); `since` (`:3242-3250`) — no `#h` (global) |
| `agent-observer-control` (`relay.rs:499`) | `send_observer_control_subscribe`, `relay.rs:3273-3305` | `kinds: [24200]` (`:3278`); `#p: [agent_pubkey]` (`:3279`); `since: now` (`:3280-3283`) — no `#h`, no `authors` |

`AGENTS.md` requires every REQ to carry `kinds`. The channel REQ can omit it:
`ChannelFilter.kinds` is `Option<Vec<u32>>` with `None` documented as wildcard
(`config.rs:479-480`), and `--subscribe-mode all` without `--kinds` produces
exactly that (`config.rs:1180`, `config.rs:1272`), as does a `Config`-mode rule
with an empty `kinds` list (`config.rs:1195-1196`, `config.rs:1286-1287`).
`send_subscribe` then emits a REQ with only `#h` + `since`, which trips the
relay p-gate. No guard or warning exists at the send site.

#### HTTP query filters

| Call site | Filter |
|---|---|
| `discover_channels` step 1 | `kind: 39002` (`KIND_NIP29_GROUP_MEMBERS`) + `#p: [agent_pubkey]` — `relay.rs:662-668` |
| `discover_channels` step 2 | `kind: 39000` (`KIND_NIP29_GROUP_METADATA`) + `#d: [uuid…]` — `relay.rs:700-706` |
| `fetch_core_body` | `kind: 30174` (`KIND_AGENT_ENGRAM`) + `author: agent_pubkey` + `#d: [d_tag]` + `#p: [owner_hex]` + `limit(16)` — `engram_fetch.rs:79-89` |

All three specify `kinds`, so none trips the p-gate.

#### `engram_fetch` surface

| Item | Location | Signature |
|---|---|---|
| `ONBOARDING_NUDGE` (`pub const`) | `engram_fetch.rs:29-31` | `&'static str` |
| `build_core_section` (`pub`) | `engram_fetch.rs:39-44` | `async (&RestClient, &Keys, &PublicKey) -> Option<String>` |
| `fetch_core_body` (private) | `engram_fetch.rs:71-76` | `async (...) -> Result<Option<String>, String>` |
| `decode_core_body` (private, pure) | `engram_fetch.rs:110-115` | `(&[Value], &Keys, &PublicKey) -> Result<Option<String>, String>` |

Only `build_core_section` is called outside the module — from
`pool.rs:1387-1391`, wrapped in a 3 s timeout (`pool.rs:1386`, `:1394`).

#### `observer` surface

| Item | Location | Signature |
|---|---|---|
| `ObserverHandle::in_process` | `observer.rs:83-85` | `() -> Self` |
| `ObserverHandle::subscribe` | `observer.rs:88-90` | `(&self) -> broadcast::Receiver<ObserverEvent>` |
| `ObserverHandle::snapshot` | `observer.rs:93-102` | `(&self) -> Vec<ObserverEvent>` |
| `ObserverHandle::emit` | `observer.rs:105-136` | `(&self, kind: impl Into<String>, agent_index: Option<usize>, &ObserverContext, payload: Value)` — infallible, no return value |
| `context_for` | `observer.rs:138-149` | `(Option<Uuid>, Option<String>, Option<String>) -> ObserverContext` |
| `context_for_turn` | `observer.rs:152-164` | `(Option<Uuid>, Option<String>, String, String) -> ObserverContext` |

The module doc explicitly states this is process-local and deliberately exposes
no local HTTP port (`observer.rs:1-6`). All relay publication of these events
happens in `lib.rs` (`spawn_relay_observer_publisher`, `run_relay_observer_publisher`,
`publish_relay_observer_event`), not here.

#### `pub(crate)` items in `relay.rs`

`channel_type_from_tags` (`:138`), `merge_discovered_channels` (`:171`),
`parse_relay_message` (`:3533`, annotated `#[allow(private_interfaces)]` at
`:3532` because it returns the private `RelayMessage`),
`parse_rate_limit_retry_secs` (`:3328`), `is_dns_error` (`:3354`),
`relay_ws_to_http` (`:3470`), `channel_sub_id` (`:3478`).
