## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: Features

Library crate, no binary targets (`crates/buzz-ws-client/Cargo.toml:1`–`7`, no
`[[bin]]`/`[features]` sections). Purpose per the module docs: connect, NIP-42
authenticate, publish, and read relay messages.

---

### 1. Capability matrix

| Capability | State | Evidence |
|---|---|---|
| Connect to a relay over WebSocket (ws/wss via `MaybeTlsStream`) | Full | `connection.rs:48`–`65`, `:8`, `:14` |
| NIP-42 authentication (await challenge → sign → send AUTH → await OK) | Full | `connection.rs:70`–`93`; `message.rs:151`–`167` |
| Connect + authenticate in one call | Full | `connection.rs:37`–`45` |
| Attach an extra authorization tag (documented as "NIP-OA") to the AUTH event | Full (single tag only) | `connection.rs:36`, `message.rs:146`–`163` |
| Publish a signed event and await its `OK` | Full | `connection.rs:96`–`101` |
| One-shot publish helper with overall timeout | Full | `connection.rs:277`–`294` |
| Read the next relay message with a caller-set timeout | Full | `connection.rs:104`–`112`, `:128`–`155` |
| Out-of-order message buffering while awaiting a specific frame | Full | `connection.rs:28`, `:205`, `:257`, `:259` |
| Parse the six NIP-01/NIP-42 relay message types | Full for `EVENT`, `OK`, `EOSE`, `CLOSED`, `NOTICE`, `AUTH` | `message.rs:63`–`139` |
| WebSocket keepalive (Ping → Pong) | Full | `connection.rs:148`–`150`, `:208`–`:210`, `:262`–`:264` |
| Graceful close | Full | `connection.rs:115`–`118` |
| Send arbitrary JSON frames (escape hatch used for `REQ`/`CLOSE`/`COUNT`) | Full, but untyped | `connection.rs:121`–`126` |
| Subscription management (typed `REQ`/`CLOSE`, filter types, EOSE-driven collection) | **Absent** — no such API; callers hand-roll it (e.g. `crates/buzz-test-client/src/lib.rs:154`, `:160`) | no `REQ`/`CLOSE` literal anywhere in the crate |
| `COUNT` message support | **Absent** — no `RelayMessage` variant; a `COUNT` reply would fail as `UnexpectedMessage` | `message.rs:63`–`142` |
| Reconnect / retry / backoff | **Absent** | no retry or backoff code in `connection.rs` (verified over the whole file) |
| Automatic re-auth on mid-session challenge | **Partial** — challenge is captured and buffered, but re-auth is not triggered; caller must call `authenticate` again | `connection.rs:255`–`258` vs `:161` |
| Rejection surfacing for published events | **Partial** — `accepted: false` is returned to the caller rather than raised as an error, and the `EventRejected` variant exists unused | `connection.rs:96`–`101`; `error.rs:40` |
| Binary frame handling | **Ignored by design** (`_ => {}`) | `connection.rs:152`, `:212`, `:266` |
| Metrics / instrumentation | **Minimal** — three `debug!` lines, no spans, no counters | `connection.rs:57`, `:91`, `:123` |
| Configuration surface (env vars, cargo features) | **Absent** — see the configuration aspect | no `env::var`, no `cfg(feature = …)` in the crate |

---

### 2. Completeness notes

- The public API is closed over a narrow flow: connect → auth → publish → read →
  close. Everything a NIP-01 subscriber needs beyond that is delegated to
  `send_raw` + `next_event` (`connection.rs:121`, `:104`).
- `next_event` re-checks the buffer immediately before delegating to `recv_one`,
  which checks it again (`connection.rs:108`–`111` then `:129`–`131`) — behaviourally
  redundant but harmless.
- The `Auth` challenge cap (1024 bytes) exists on only one of the three challenge
  intake paths (`connection.rs:198`; the paths at `:161` and `:170` are uncapped).

---

### 3. TODO / FIXME / HACK / XXX markers

**Zero.** A case-insensitive search for `TODO`, `FIXME`, `HACK`, `XXX`,
`unimplemented`, and `todo` across `crates/buzz-ws-client/` returned no matches
(exit code 1, no output). There are also no `#[deprecated]`, `#[allow(dead_code)]`,
or `unimplemented!()`/`todo!()` markers. The only lint attributes present are
`#![deny(unsafe_code)]` (`lib.rs:1`) and `#[allow(clippy::result_large_err)]`
(`message.rs:54`).

---

### 4. Tests shipped with the crate

Three unit tests, all compile-time assertions on the timeout floors — no I/O, no
mock relay, no parser tests:

| Test | Asserts | file:line |
|---|---|---|
| `auth_challenge_timeout_meets_floor` | `AUTH_CHALLENGE_TIMEOUT_SECS >= 20` | `connection.rs:300`–`303` |
| `auth_ok_timeout_meets_floor` | `AUTH_OK_TIMEOUT_SECS >= 20` | `connection.rs:305`–`308` |
| `publish_ok_timeout_meets_floor` | `PUBLISH_OK_TIMEOUT_SECS >= 30` | `connection.rs:310`–`313` |

Module gate at `connection.rs:296`–`298`. Each body uses an inline `const { assert!(…) }`
block, so the check is evaluated at compile time rather than at run time.
