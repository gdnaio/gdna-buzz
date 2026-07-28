## Module: buzz-acp — relay client & observer (`crates/buzz-acp/src`)
### Aspect: Integrations

#### Headline finding: `buzz-ws-client` is not a dependency

`crates/buzz-acp/Cargo.toml` lists `buzz-core`, `buzz-sdk`, and `buzz-persona`
as its only internal dependencies. `buzz-ws-client` is absent. The only crates
that depend on the shared NIP-42 client are `buzz-cli`
(`crates/buzz-cli/Cargo.toml:77`) and `buzz-test-client`
(`crates/buzz-test-client/Cargo.toml:13`).

`relay.rs` instead imports `tokio_tungstenite::{connect_async, tungstenite::Message,
MaybeTlsStream, WebSocketStream}` directly (`relay.rs:125`) — the same imports
`buzz-ws-client/src/connection.rs:9` uses — and reimplements the connect, the
`RelayMessage` enum, the frame parser, and the whole NIP-42 handshake.

#### Duplication table

| Behaviour | `crates/buzz-acp/src/relay.rs` | `crates/buzz-ws-client/src/connection.rs` | Divergence |
|---|---|---|---|
| `WsStream` type alias | `:525` | `:14` | Identical definition |
| URL parse before connect | `:3830-3832` (`url::Url`, error → `RelayError::Http`) | `:49-51` (`url::Url`, error → `WsClientError::Url`) | Neither asserts a `ws`/`wss` scheme |
| `connect_async` | `:3834-3837` | `:53-55` | acp wraps in `CONNECT_TIMEOUT` = 30 s (`:69`); ws-client has no connect timeout |
| `debug!("connected to relay at {url}")` | `:3838` | `:57` | Byte-identical log line |
| `RelayMessage` enum | `:470-495` (private) | `message.rs:8-41` (`pub`) | ws-client wraps OK in an `OkResponse` newtype variant; acp inlines the three fields |
| `OkResponse` struct | `:3917-3921` (private, no derives) | `message.rs:44-52` (`pub`) | Same three fields, duplicated shape |
| Frame parser | `parse_relay_message` `:3531-3620` | `parse_relay_message` `message.rs:55-144` | Two independent hand-written `Vec<Value>` parsers; acp's returns `RelayError`, ws-client's `WsClientError` |
| AUTH-challenge wait | `wait_for_auth_challenge` `:3865-3913` | `wait_for_auth_challenge` `:156-206` | **ws-client caps the challenge at 1024 bytes (`:198-201`); acp has no cap on any path** |
| Wait for AUTH OK | `wait_for_any_ok` `:3924-3982` | `wait_for_ok` `:208-263` | acp accepts the **first OK of any id** (comment concedes it, `:3846-3854`); ws-client matches the AUTH event id (`:226`, `:255`) |
| Build + send AUTH event | `send_auth_response` `:3433-3461` | `authenticate` `:70-93` + `build_auth_event` `message.rs:151-166` | acp hand-builds the tag vec when `auth_tag` is present (`:3442-3456`); ws-client always goes through `EventBuilder::auth` then appends the tag |
| AUTH challenge timeout | `AUTH_TIMEOUT` `:64` = 20 s | `AUTH_CHALLENGE_TIMEOUT_SECS` `:17` = 20 s | Same value, two constants; ws-client pins it with a compile-time test (`:279-281`) |
| AUTH OK timeout | reuses `AUTH_TIMEOUT` (20 s) at `:3849` | `AUTH_OK_TIMEOUT_SECS` `:20` = 20 s | Same value, separate constant |
| Ping → Pong in read loops | `:2370-2376`, `:3903-3907`, `:3963-3967` | `:148-150`, `:208-210`, `:262-264` | acp routes every send through `ws_send_timeout` (10 s, `:3312-3323`); ws-client uses bare `self.ws.send` |
| Non-target frame buffering | `VecDeque<RelayMessage>` threaded through `do_connect` (`:3833`, `:3902`, `:3960`) | `self.buffer: VecDeque<RelayMessage>` field (`:28`, `:205`, `:257`) | acp replays the buffer through `handle_ws_message` after re-serialising to text (`:2393-2450`); ws-client leaves it for `next_event` (`:105-112`) |
| Mid-session AUTH | Re-authenticates immediately, reconnects on failure (`:2344-2353`); rejects on `OK false auth*` (`:2359-2363`) | Records `pending_challenge` only (`:144`, `:256-258`) | **Functional divergence** |
| Outbound frame logging | Only `debug!("sent AUTH response for challenge")` (`:3459`) — no frame body | `send_raw` logs the full frame: `debug!("→ relay: {text}")` (`:123`) | acp **fixes** the ws-client token-leak: the AUTH event and any `auth_tag` bearer value never reach the log |
| Reconnect / retry | Three loops (`:3786`, `:2893`, `:3022`) | None | acp adds it |
| Rate-limit handling | Gate + paced drain (`:1182-1207`, `:1621-1699`) | None | acp adds it |

Verification of the line numbers cited in earlier analysis:
`relay.rs:2344-2350` is correct for the mid-session AUTH arm.
`relay.rs:3435-3461` is off by two — `send_auth_response` begins at
`relay.rs:3433` and ends at `relay.rs:3461`.
`relay.rs:3843-3845` is also shifted: `wait_for_auth_challenge` is called at
`relay.rs:3843` and `send_auth_response` at `relay.rs:3845`, with
`wait_for_any_ok` at `relay.rs:3849`.

#### Which `buzz-ws-client` weaknesses this copy repeats, fixes, or worsens

| Weakness | Status in `relay.rs` | Evidence |
|---|---|---|
| `ws://` accepted with no scheme check | **Repeated** | `:3830-3832` parses with `url::Url` and passes `parsed.as_str()` straight to `connect_async`; no scheme assertion. The `is_terminal_ws_error` test `do_connect_wrong_scheme_is_terminal` (`:5549-5557`) only confirms tungstenite's own `Url` error is treated as terminal — it does not add a check. `relay_ws_to_http` (`:3470-3475`) likewise accepts either. |
| AUTH frame logged in full at debug, including `auth_tag` | **Fixed** | `send_auth_response` logs a fixed string with no event body (`:3459`). There is no `send_raw`-equivalent that logs frames; the two other WS send helpers log only on error (`:2620`, `:3207`). |
| 1024-byte challenge cap on only 1 of 3 intake paths | **Made worse** | acp has the cap on **0 of 3** paths: `wait_for_auth_challenge` returns the challenge unchecked (`:3901`), `parse_relay_message`'s `"AUTH"` arm has no length guard (`:3610-3616`), and the mid-session arm passes it straight into `send_auth_response` (`:2344-2348`). |
| No retry / reconnect | **Fixed** | `retry_initial_connect` (`:3786-3822`), `try_autonomous_reconnect` (`:2893-3018`), `wait_for_reconnect` (`:3022-3151`), plus ping/pong death detection (`:1855-1927`). |

#### Relay protocol touch points

WebSocket (NIP-01 verbs implemented): outbound `AUTH`, `REQ`, `CLOSE`, `EVENT`,
`Ping`, `Pong`, `Close`; inbound `AUTH`, `EVENT`, `OK`, `EOSE`, `CLOSED`,
`NOTICE`. `COUNT` is not implemented in either direction.

HTTP bridge (used for all reads, so the WS/HTTP split is real, not incidental):

| Endpoint | Call site | Auth |
|---|---|---|
| `POST /query` | `relay.rs:399-406`, used at `:670`, `:705`, `engram_fetch.rs:91` | NIP-98 kind:27235 + optional `x-auth-tag` |
| `POST /events` | `relay.rs:411-423`, used by `pool.rs` reactions/metrics/notices | NIP-98 kind:27235 + optional `x-auth-tag` |

No `POST /count` call exists. The unauthenticated `X-Pubkey` header path — which
is exploitable when `BUZZ_REQUIRE_AUTH_TOKEN` defaults to false — is **not**
used: every bridge request signs a fresh NIP-98 event per attempt
(`relay.rs:378-381`).

#### `buzz-core`

| Import | Location | Use |
|---|---|---|
| `kind::{KIND_AGENT_OBSERVER_FRAME, KIND_MEMBER_ADDED_NOTIFICATION, KIND_MEMBER_REMOVED_NOTIFICATION, KIND_TYPING_INDICATOR}` | `relay.rs:118-123` | REQ kinds, publish kind, observer-frame detection |
| `kind::KIND_NIP29_GROUP_MEMBERS` | `relay.rs:665` | discovery filter |
| `kind::KIND_NIP29_GROUP_METADATA` | `relay.rs:700` | discovery filter |
| `kind::KIND_AGENT_ENGRAM` | `engram_fetch.rs:17`, used `:81` | engram filter |
| `engram::{conversation_key, d_tag, select_head, validate_and_decrypt, Body}` + `engram::CORE_SLUG` | `engram_fetch.rs:16`, `:77` | NIP-AE key derivation, LWW head selection, decrypt |

#### `buzz-sdk`

`relay.rs`, `observer.rs`, and `engram_fetch.rs` do **not** import `buzz-sdk`.
The crate is a dependency (`crates/buzz-acp/Cargo.toml`) but is consumed
elsewhere in `buzz-acp`: `nip_oa::verify_auth_tag` / `parse_auth_tag`
(`lib.rs:1370`), `build_agent_observer_frame` for the kind:24200 wrapper, and
`build_reaction` / `build_remove_reaction` / `build_message` in `pool.rs`. So the
event *builders* for the durable kinds this module transports live in the SDK,
while the two events this module builds itself — the NIP-42 AUTH event
(`relay.rs:3442-3457`) and the typing indicator (`relay.rs:866-868`) — bypass it
and use raw `nostr::EventBuilder`.

#### External crates

`tokio-tungstenite` + `futures-util` (`relay.rs:124-125`) for the socket;
`rustls` 0.23 with the `ring` provider, referenced directly for the terminal-TLS
downcast (`relay.rs:3724-3766`) — the doc comment notes this relies on a single
rustls version in the tree (`relay.rs:3762-3763`), a real coupling to the
workspace dependency graph. `reqwest` for the bridge, `sha2` + `base64` + `hex`
for NIP-98 (`relay.rs:268-269`, `:284`, `:299`), `url` for parsing,
`uuid`/`chrono` for ids and timestamps, `thiserror` for `RelayError`,
`serde`/`serde_json` throughout. `observer.rs` uses only
`tokio::sync::broadcast`, `std::sync::{Mutex, atomic}`, `serde::Serialize`,
`chrono`, and `uuid`. No `rand` — jitter is derived from
`SystemTime::subsec_nanos()` (`relay.rs:3337-3346`).

#### In-crate consumers

`lib.rs` is the only real consumer of `HarnessRelay`: `connect`
(`lib.rs:1344`), `set_startup_watermark` (`:1352`),
`subscribe_membership_notifications` (`:1359`), `event_publisher` (`:1364`,
`:1406`, `:1413`-adjacent), `subscribe_observer_controls` (`:1413`),
`take_observer_control_rx` (`:1416`), `discover_channels` (`:1432`),
`rest_client` (`:1551`, `:1552`), `subscribe_channel` / `subscribe_channel_from`,
`unsubscribe_channel`, `next_event`, `build_typing_event` + `try_publish_event`
(`:2333-2341`), `reconnect`, `shutdown`. `setup_mode.rs` uses
`event_publisher` (`:383`) and `publish_event` (`:641`). `pool.rs` consumes
`RestClient` and `ChannelInfo` (`pool.rs:41`) and calls
`engram_fetch::build_core_section` (`pool.rs:1387`).
