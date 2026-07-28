## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: Data Model

Scope of files read: `Cargo.toml` (17 lines), `src/lib.rs` (9), `src/error.rs` (51),
`src/message.rs` (167), `src/connection.rs` (314). There is no `tests/` directory,
no `build.rs`, no `benches/`. No persistence layer, no serde-derived DTOs: all
types are in-memory only and constructed from/into JSON arrays by hand.

---

### 1. Connection type

`NostrWsConnection` — the only stateful type in the crate.

| Field | Type | Visibility | Purpose (as coded) | file:line |
|---|---|---|---|---|
| `ws` | `WsStream` | private | The live WebSocket stream | `crates/buzz-ws-client/src/connection.rs:27` |
| `buffer` | `VecDeque<RelayMessage>` | private | Out-of-order relay messages parked while waiting for a specific message | `crates/buzz-ws-client/src/connection.rs:28` |
| `pending_challenge` | `Option<String>` | private | Last NIP-42 `AUTH` challenge string observed on the socket | `crates/buzz-ws-client/src/connection.rs:29` |
| `relay_url` | `String` | private | The URL string passed to `connect`, reused verbatim as the `relay` tag source when building the AUTH event | `crates/buzz-ws-client/src/connection.rs:30` |

Declaration: `crates/buzz-ws-client/src/connection.rs:26`–`31`. The struct is `pub`
but **every field is private** — external mutation is only possible through the
methods in `impl NostrWsConnection` (`crates/buzz-ws-client/src/connection.rs:33`).
No `Debug`, `Clone`, `Default`, or `Drop` impl is derived or written for it.

Private type alias for the transport:

```rust
type WsStream = WebSocketStream<MaybeTlsStream<tokio::net::TcpStream>>;
```
`crates/buzz-ws-client/src/connection.rs:14` (not exported — it does not appear in
`crates/buzz-ws-client/src/lib.rs:7`).

Ownership note: `disconnect` takes `mut self` by value
(`crates/buzz-ws-client/src/connection.rs:115`), so closing consumes the
connection; all other methods take `&mut self`.

---

### 2. Connection / auth state representation

There is **no explicit state enum and no state-machine type** in this crate
(verified: no `enum` other than `RelayMessage` in `message.rs:8` and
`WsClientError` in `error.rs:5`). Auth/connection state is represented implicitly:

| Implicit state | How represented | file:line |
|---|---|---|
| Connected, challenge not yet seen | `pending_challenge: None` after construction | `crates/buzz-ws-client/src/connection.rs:62` |
| Challenge seen but not yet consumed | `pending_challenge = Some(challenge)` set opportunistically while reading | `crates/buzz-ws-client/src/connection.rs:144`, `crates/buzz-ws-client/src/connection.rs:256` |
| Challenge consumed | `pending_challenge.take()` at the top of `wait_for_auth_challenge` | `crates/buzz-ws-client/src/connection.rs:161` |
| Authenticated | Not recorded anywhere; `authenticate` returns `Ok(())` and stores no flag | `crates/buzz-ws-client/src/connection.rs:70`–`93` |

Consequence visible in code: nothing prevents `authenticate` being called twice, and
no method checks an "is authenticated" predicate before sending
(`send_event` at `crates/buzz-ws-client/src/connection.rs:96` sends unconditionally).

---

### 3. Relay message model (inbound wire types)

`RelayMessage` — `#[derive(Debug, Clone)]` (`crates/buzz-ws-client/src/message.rs:7`),
declared `crates/buzz-ws-client/src/message.rs:8`–`40`. Variant fields are named
(struct-style) except `Ok`.

| Variant | Fields (name: type) | Wire form parsed from | file:line |
|---|---|---|---|
| `Event` | `subscription_id: String`, `event: Box<Event>` | `["EVENT", <sub_id>, <event object>]` | decl `message.rs:10`–`15`; parse `message.rs:64`–`79` |
| `Ok` | `OkResponse` (tuple variant) | `["OK", <event_id>, <bool>, <message>]` | decl `message.rs:17`; parse `message.rs:80`–`97` |
| `Eose` | `subscription_id: String` | `["EOSE", <sub_id>]` | decl `message.rs:19`–`22`; parse `message.rs:98`–`107` |
| `Closed` | `subscription_id: String`, `message: String` | `["CLOSED", <sub_id>, <message>]` | decl `message.rs:24`–`29`; parse `message.rs:108`–`123` |
| `Notice` | `message: String` | `["NOTICE", <message>]` | decl `message.rs:31`–`34`; parse `message.rs:124`–`131` |
| `Auth` | `challenge: String` | `["AUTH", <challenge>]` | decl `message.rs:36`–`39`; parse `message.rs:132`–`139` |

`event` is boxed (`Box<Event>`, `crates/buzz-ws-client/src/message.rs:14`;
`Box::new(event)` at `crates/buzz-ws-client/src/message.rs:77`) — the only
indirection in the enum.

Not modelled (no variant exists): `COUNT`, `NEG-*`, or any other NIP message type.
Anything else is rejected as `WsClientError::UnexpectedMessage("unknown message
type: {other}")` (`crates/buzz-ws-client/src/message.rs:140`–`142`).

---

### 4. `OkResponse`

`#[derive(Debug, Clone)]` (`crates/buzz-ws-client/src/message.rs:43`), declared
`crates/buzz-ws-client/src/message.rs:44`–`51`. This is the only type with public
fields.

| Field | Type | Semantics per doc comment / parse code | file:line |
|---|---|---|---|
| `event_id` | `String` | Hex-encoded ID of the acknowledged event; taken from `arr[1]` as a string, required | decl `message.rs:46`; parse `message.rs:81`–`85` |
| `accepted` | `bool` | Relay acceptance flag; taken from `arr[2]`, **defaults to `false`** when absent or non-boolean (`unwrap_or(false)`) | decl `message.rs:48`; parse `message.rs:86` |
| `message` | `String` | Human-readable reason; taken from `arr[3]`, **defaults to `""`** when absent | decl `message.rs:50`; parse `message.rs:87`–`91` |

Matching of an `OK` to a request is by exact string comparison of `event_id`
against the hex id of the locally built/sent event
(`crates/buzz-ws-client/src/connection.rs:227`, `:254`).

---

### 5. Outbound wire values

No dedicated request/DTO types exist. Outbound frames are built ad hoc with
`serde_json::json!` and sent as text:

| Outbound value | Built at | Sent via |
|---|---|---|
| `["AUTH", <signed event>]` | `crates/buzz-ws-client/src/connection.rs:82` | `send_raw` → `Message::Text` (`connection.rs:121`–`125`) |
| `["EVENT", <signed event>]` | `crates/buzz-ws-client/src/connection.rs:98` | same |
| `Message::Pong(data)` | echo of an inbound `Ping` | `connection.rs:149`, `:209`, `:263` |
| Arbitrary `serde_json::Value` (e.g. `REQ` / `CLOSE`, composed by callers) | caller-supplied | `send_raw` (`connection.rs:121`) |

Serialization of the `nostr::Event` inside those arrays is delegated to
`serde_json` via the `nostr` crate's own `Serialize` impl
(`crates/buzz-ws-client/src/connection.rs:82`, `:98`).

---

### 6. Error model (summary; detail in the conventions/API aspects)

`WsClientError` — `#[derive(Debug, Error)]` (`crates/buzz-ws-client/src/error.rs:4`),
10 variants at `crates/buzz-ws-client/src/error.rs:5`–`45`:
`WebSocket` (`:8`, `#[from]` tungstenite), `Json` (`:12`, `#[from]` serde_json),
`EventBuilder(String)` (`:16`), `Url(String)` (`:20`), `Timeout` (`:24`),
`ConnectionClosed` (`:28`), `UnexpectedMessage(String)` (`:32`),
`AuthFailed(String)` (`:36`), `EventRejected(String)` (`:40`),
`NoAuthChallenge` (`:44`). Plus a manual `From<nostr::event::builder::Error>`
(`crates/buzz-ws-client/src/error.rs:47`–`51`).

---

### 7. Domain types borrowed from `nostr` (not defined here)

| Type | Used as | file:line |
|---|---|---|
| `nostr::Event` | payload in `RelayMessage::Event`, argument to `send_event` / `publish_event`, return of `build_auth_event` | `message.rs:1`, `message.rs:14`, `connection.rs:96`, `connection.rs:279` |
| `nostr::Keys` | signing material for the AUTH event | `connection.rs:39`, `message.rs:154` |
| `nostr::Tag` | optional extra authorization tag on the AUTH event | `connection.rs:40`, `message.rs:155` |
| `nostr::EventBuilder` | AUTH event construction | `message.rs:158` |
| `nostr::RelayUrl` | parsed relay URL embedded in the AUTH event | `message.rs:157` |

The AUTH event's kind and its standard tag set are produced by
`nostr::EventBuilder::auth` (`crates/buzz-ws-client/src/message.rs:158`) — **not
visible in this crate**, so the concrete kind integer cannot be confirmed from
these files. (For cross-reference only, outside this module: the relay side
validates `Kind::Authentication` at `crates/buzz-auth/src/nip42.rs:52` and
`crates/buzz-core/src/kind.rs:77` defines `KIND_AUTH: u32 = 22242`.)
