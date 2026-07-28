## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: API Surface

Crate root: `crates/buzz-ws-client/src/lib.rs`. Three public modules
(`connection`, `error`, `message` — `lib.rs:3`–`5`) plus a flat re-export set
(`lib.rs:7`–`9`). No `#[doc(hidden)]`, no `pub(crate)` gating, no traits, no
macros. Library only — no `main.rs`, no binaries in `Cargo.toml`.

---

### 1. Re-exports (the intended entry surface)

| Re-export | Source | file:line |
|---|---|---|
| `publish_event`, `NostrWsConnection` | `connection` | `crates/buzz-ws-client/src/lib.rs:7` |
| `WsClientError` | `error` | `crates/buzz-ws-client/src/lib.rs:8` |
| `build_auth_event`, `parse_relay_message`, `OkResponse`, `RelayMessage` | `message` | `crates/buzz-ws-client/src/lib.rs:9` |

Because the modules themselves are `pub`, everything below is also reachable via
its module path (e.g. `buzz_ws_client::connection::AUTH_OK_TIMEOUT_SECS`, which is
**not** in the flat re-export list).

---

### 2. Public constants

| Constant | Type | Value | file:line |
|---|---|---|---|
| `AUTH_CHALLENGE_TIMEOUT_SECS` | `u64` | `20` | `crates/buzz-ws-client/src/connection.rs:17` |
| `AUTH_OK_TIMEOUT_SECS` | `u64` | `20` | `crates/buzz-ws-client/src/connection.rs:20` |
| `PUBLISH_OK_TIMEOUT_SECS` | `u64` | `30` | `crates/buzz-ws-client/src/connection.rs:23` |

---

### 3. Public functions and methods (full signatures)

```rust
// crates/buzz-ws-client/src/connection.rs:37
pub async fn NostrWsConnection::connect_authenticated(
    url: &str, keys: &Keys, auth_tag: Option<&Tag>,
) -> Result<Self, WsClientError>

// crates/buzz-ws-client/src/connection.rs:48
pub async fn NostrWsConnection::connect(url: &str) -> Result<Self, WsClientError>

// crates/buzz-ws-client/src/connection.rs:70
pub async fn NostrWsConnection::authenticate(
    &mut self, keys: &Keys, auth_tag: Option<&Tag>,
) -> Result<(), WsClientError>

// crates/buzz-ws-client/src/connection.rs:96
pub async fn NostrWsConnection::send_event(&mut self, event: Event)
    -> Result<OkResponse, WsClientError>

// crates/buzz-ws-client/src/connection.rs:104
pub async fn NostrWsConnection::next_event(&mut self, timeout_dur: Duration)
    -> Result<RelayMessage, WsClientError>

// crates/buzz-ws-client/src/connection.rs:115
pub async fn NostrWsConnection::disconnect(mut self) -> Result<(), WsClientError>

// crates/buzz-ws-client/src/connection.rs:121
pub async fn NostrWsConnection::send_raw(&mut self, value: &Value)
    -> Result<(), WsClientError>

// crates/buzz-ws-client/src/connection.rs:277
pub async fn publish_event(
    relay_url: &str, event: Event, keys: &Keys,
    auth_tag: Option<&Tag>, timeout_secs: u64,
) -> Result<OkResponse, WsClientError>

// crates/buzz-ws-client/src/message.rs:55
#[allow(clippy::result_large_err)]
pub fn parse_relay_message(text: &str) -> Result<RelayMessage, WsClientError>

// crates/buzz-ws-client/src/message.rs:151
pub fn build_auth_event(
    challenge: &str, relay_url: &str, keys: &Keys, auth_tag: Option<&Tag>,
) -> Result<Event, WsClientError>
```

Private helpers (not part of the surface): `recv_one`
(`crates/buzz-ws-client/src/connection.rs:128`), `wait_for_auth_challenge`
(`:157`), `wait_for_ok` (`:217`).

---

### 4. Public types

| Type | Kind | Public members | file:line |
|---|---|---|---|
| `NostrWsConnection` | struct | none (all 4 fields private) | `connection.rs:26` |
| `RelayMessage` | enum, `Debug + Clone` | 6 variants: `Event`, `Ok`, `Eose`, `Closed`, `Notice`, `Auth` | `message.rs:8` |
| `OkResponse` | struct, `Debug + Clone` | `event_id: String`, `accepted: bool`, `message: String` | `message.rs:44`–`50` |
| `WsClientError` | enum, `Debug + thiserror::Error` | 10 variants | `error.rs:5`–`45` |

Trait impls exposed: `Error`/`Display` on `WsClientError` via `#[derive(Error)]`
(`error.rs:4`), `From<tokio_tungstenite::tungstenite::Error>` (`error.rs:8`),
`From<serde_json::Error>` (`error.rs:12`),
`From<nostr::event::builder::Error>` (`error.rs:47`).

---

### 5. Function → wire message mapping

| Public fn | Sends on the wire | Consumes/handles from the wire | Return type |
|---|---|---|---|
| `connect` | WebSocket HTTP upgrade only (`connect_async`, `connection.rs:53`) | nothing | `Result<Self, WsClientError>` |
| `authenticate` | `["AUTH", <signed event>]` (`connection.rs:82`) | waits for `AUTH` challenge (`connection.rs:76`→`:197`), then `OK` matching the AUTH event id (`connection.rs:85`→`:254`); replies `Pong` to `Ping` (`:209`) | `Result<(), WsClientError>` |
| `connect_authenticated` | upgrade + `["AUTH", …]` (`connection.rs:42`–`43`) | as `authenticate` | `Result<Self, WsClientError>` |
| `send_event` | `["EVENT", <event>]` (`connection.rs:98`) | `OK` with matching event id (`connection.rs:99`, `:254`); buffers `EVENT`/`EOSE`/`CLOSED`/`NOTICE`, records `AUTH` (`:255`–`:259`) | `Result<OkResponse, WsClientError>` |
| `next_event` | nothing (may emit `Pong`, `connection.rs:149`) | any one `RelayMessage` (buffer first, `connection.rs:108`) | `Result<RelayMessage, WsClientError>` |
| `send_raw` | any caller-provided JSON array — this is how `REQ`/`CLOSE`/`COUNT` are sent, since the crate has no typed subscribe API (`connection.rs:121`–`124`) | nothing | `Result<(), WsClientError>` |
| `disconnect` | WebSocket Close frame (`close(None)`, `connection.rs:116`) | nothing | `Result<(), WsClientError>` |
| `publish_event` | upgrade + `["AUTH", …]` + `["EVENT", …]` + Close (`connection.rs:285`–`288`) | `AUTH` challenge, both `OK`s | `Result<OkResponse, WsClientError>` |
| `parse_relay_message` | — (pure parser) | `EVENT`, `OK`, `EOSE`, `CLOSED`, `NOTICE`, `AUTH` (`message.rs:64`–`139`) | `Result<RelayMessage, WsClientError>` |
| `build_auth_event` | — (pure builder; produces the event the caller/`authenticate` sends) | — | `Result<Event, WsClientError>` |

WebSocket control frames: inbound `Ping` is answered with `Pong` in all three read
loops (`connection.rs:148`–`150`, `:208`–`:210`, `:262`–`:264`); inbound `Close`
maps to `WsClientError::ConnectionClosed` (`:151`, `:211`, `:265`); all other frame
kinds (Binary, Pong, Frame) are silently ignored via `_ => {}` (`:152`, `:212`,
`:266`).

**Not implemented in this crate:** typed `REQ`/`CLOSE`/`COUNT` builders, a
subscription abstraction, `EOSE`-driven collect helpers, and any reconnect API.
Callers assemble those from `send_raw` + `next_event` (as `buzz-test-client` does at
`crates/buzz-test-client/src/lib.rs:154` and `:160`).

---

### 6. Known consumers (for API-contract impact)

| Consumer | Uses | file:line |
|---|---|---|
| `buzz-cli` | `buzz_ws_client::publish_event(&ws_url, event, &self.keys, self.auth_tag.as_ref(), 75)` | `crates/buzz-cli/src/client.rs:1080`; dep at `crates/buzz-cli/Cargo.toml:77` |
| `buzz-test-client` | wraps `NostrWsConnection` (`connect`, `send_event`, `send_raw`, `next_event`) and re-exports `parse_relay_message`, `OkResponse`, `RelayMessage`, `WsClientError` | `crates/buzz-test-client/src/lib.rs:13`–`14`, `:85`, `:98`, `:123`, `:154`, `:175`; dep at `crates/buzz-test-client/Cargo.toml:13` |
| workspace | path dependency declared once | `Cargo.toml:16`, `Cargo.toml:134` |
