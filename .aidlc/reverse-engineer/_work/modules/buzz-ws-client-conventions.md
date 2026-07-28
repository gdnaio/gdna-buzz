## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: Conventions

---

### 1. Module organization

| Module | Role | file:line |
|---|---|---|
| `lib.rs` | Crate lint gate + module declarations + flat re-exports; contains no logic | `lib.rs:1`–`9` |
| `connection.rs` | Transport, handshake, timeouts, read loops, one-shot helper, unit tests | `connection.rs:1`–`314` |
| `message.rs` | Wire types + parser + AUTH-event builder (pure functions, no I/O) | `message.rs:1`–`167` |
| `error.rs` | Single error enum + one manual `From` impl | `error.rs:1`–`51` |

Conventions observed: one concern per file; pure logic (`message.rs`) separated from
I/O (`connection.rs`); errors centralized in one enum rather than per-module error
types; re-exports flattened at the root so callers can use
`buzz_ws_client::{NostrWsConnection, RelayMessage, …}` (`lib.rs:7`–`9`) while module
paths remain available.

---

### 2. Naming

| Pattern | Examples | file:line |
|---|---|---|
| Types in `UpperCamelCase`, protocol-flavoured names | `NostrWsConnection`, `RelayMessage`, `OkResponse`, `WsClientError` | `connection.rs:26`, `message.rs:8`, `message.rs:44`, `error.rs:5` |
| Enum variants mirror wire message names (`OK`→`Ok`, `EOSE`→`Eose`) | `Event`, `Ok`, `Eose`, `Closed`, `Notice`, `Auth` | `message.rs:10`–`39` |
| Constants `SCREAMING_SNAKE_CASE` with unit suffix `_SECS` | `AUTH_CHALLENGE_TIMEOUT_SECS`, `AUTH_OK_TIMEOUT_SECS`, `PUBLISH_OK_TIMEOUT_SECS` | `connection.rs:17`, `:20`, `:23` |
| Verb-first fn names; `wait_for_*` for blocking waits, `recv_*`/`send_*` for I/O | `connect`, `connect_authenticated`, `authenticate`, `send_event`, `send_raw`, `next_event`, `disconnect`, `recv_one`, `wait_for_auth_challenge`, `wait_for_ok` | `connection.rs:37`–`269` |
| Parameter naming avoids shadowing the imported `timeout` fn | `timeout_dur` (not `timeout`) | `connection.rs:106`, `:159`, `:220` |
| Builder/parse free functions named after the artifact | `build_auth_event`, `parse_relay_message` | `message.rs:151`, `:55` |
| Test names assert an invariant | `auth_challenge_timeout_meets_floor` | `connection.rs:301` |

---

### 3. Error handling

- One crate-wide `thiserror` enum, `WsClientError`, `#[derive(Debug, Error)]`
  (`error.rs:4`–`5`), with a `#[error("…")]` display string on every variant
  (`error.rs:7`, `:11`, `:15`, `:19`, `:23`, `:27`, `:31`, `:35`, `:39`, `:43`).
- Automatic conversions where the source type is owned by a dependency:
  `#[from] tokio_tungstenite::tungstenite::Error` (`error.rs:8`) and
  `#[from] serde_json::Error` (`error.rs:12`). A hand-written
  `From<nostr::event::builder::Error>` stringifies instead of wrapping
  (`error.rs:47`–`51`), and the same stringify is duplicated inline at
  `message.rs:166`.
- Foreign errors that are not `#[from]` are stringified into `Url(String)` /
  `EventBuilder(String)` via `map_err(|e| … e.to_string())`
  (`connection.rs:51`, `message.rs:157`, `message.rs:166`) — source chains are not
  preserved for those.
- `?` propagation throughout; every fallible fn returns
  `Result<_, WsClientError>` (`connection.rs:41`, `:48`, `:74`, `:96`, `:107`, `:115`,
  `:121`, `:283`; `message.rs:55`, `:156`).
- Guard-clause style: early `return Err(...)` for rule violations
  (`connection.rs:88`, `:184`, `:199`, `:241`) and early `return Ok(...)` for cache
  hits (`connection.rs:109`, `:130`, `:162`, `:171`, `:203`, `:230`, `:254`).
- Repo rule "no new `unwrap()`/`expect()` in production paths"
  (`AGENTS.md`, Quality Gates) is **not fully honoured**: two `unwrap()` calls exist
  on `VecDeque::remove` immediately after a successful `position` lookup
  (`connection.rs:170`, `:229`), paired with `_ => unreachable!()` arms
  (`connection.rs:172`, `:231`). No `expect()` anywhere.
- `#[allow(clippy::result_large_err)]` is applied to `parse_relay_message`
  (`message.rs:54`) — the only lint suppression in the crate.

---

### 4. Async patterns

- All I/O methods are `async fn` on an owned `WebSocketStream`; there is no
  `Arc`/`Mutex`, no spawned task, no channel — the connection is single-owner and
  `&mut self`-driven (`connection.rs:70`, `:96`, `:104`, `:121`).
- Timeouts are applied with `tokio::time::timeout` around a single `.next()` await
  (`connection.rs:134`, `:187`, `:244`) or around a whole async block
  (`connection.rs:284`).
- Absolute deadlines with per-iteration recomputation for multi-frame waits
  (`connection.rs:176`–`185`, `:222`, `:236`–`242`) — avoids the timeout being reset
  by unrelated frames.
- Message buffering via `VecDeque` (`connection.rs:28`) with `push_back` for
  deferral and `pop_front` for delivery (`connection.rs:108`, `:129`, `:205`, `:257`).
- `disconnect(mut self)` consumes the connection so use-after-close is a compile
  error (`connection.rs:115`).
- Sink/stream traits are brought in explicitly rather than via a prelude:
  `use futures_util::{SinkExt, StreamExt};` (`connection.rs:4`).
- The three read loops share an identical `match raw { Text | Ping | Close | _ }`
  shape (`connection.rs:140`–`153`, `:193`–`:213`, `:250`–`:267`) — a deliberate
  copy of the same frame-dispatch skeleton in each waiter.

---

### 5. Documentation conventions

- Every public item has a `///` doc comment: constants (`connection.rs:16`, `:19`,
  `:22`), the struct (`:25`), each public method (`:34`–`36`, `:47`, `:67`–`69`,
  `:95`, `:103`, `:114`, `:120`), the free functions (`:272`–`276`; `message.rs:53`,
  `:146`–`150`), both public message types (`message.rs:6`, `:42`) and every enum
  variant *and* variant field (`message.rs:9`–`38`), and every error variant
  (`error.rs:3`, `:6`, `:10`, `:14`, `:18`, `:22`, `:26`, `:30`, `:34`, `:38`, `:42`).
  This satisfies the repo rule that new public API carries doc comments.
- Intra-doc links are used for cross-references: `[`RelayMessage`]`
  (`message.rs:53`), `[`crate::NostrWsConnection`]` (`error.rs:3`).
- Multi-paragraph docs explain the *why* on the two most subtle items: the
  `auth_tag` parameter (`message.rs:146`–`150`) and the bounded one-shot helper
  (`connection.rs:272`–`276`).
- No `//!` module-level docs on any file, and no `#![warn(missing_docs)]` — the
  doc-comment discipline is convention, not enforced here.

---

### 6. Lints and safety posture

- `#![deny(unsafe_code)]` at the crate root (`lib.rs:1`) — matches the repo's
  "no `unsafe` code" rule; no `unsafe` block exists in the crate (verified).
- No `#![forbid(...)]`, no `clippy.toml`, no crate-level `#![allow]`.
- Dependencies are all `{ workspace = true }` (`Cargo.toml:10`–`17`), so versions and
  features are centralized in the root manifest rather than pinned per crate;
  package metadata is likewise inherited (`Cargo.toml:3`–`7`).

---

### 7. Testing conventions

- Inline `#[cfg(test)] mod tests` with `use super::*;` at the bottom of the file
  under test (`connection.rs:296`–`298`) — no `tests/` directory, no dev-dependencies
  (`Cargo.toml` has no `[dev-dependencies]` section).
- Tests are compile-time invariant assertions rather than behavioural tests:
  `const { assert!(CONST >= floor) }` inside `#[test] fn` bodies
  (`connection.rs:302`, `:307`, `:312`) — encoding timeout *floors* so a future edit
  that lowers a timeout fails to build.
- No `#[tokio::test]`, no mock relay, no parser unit tests, no property tests.
