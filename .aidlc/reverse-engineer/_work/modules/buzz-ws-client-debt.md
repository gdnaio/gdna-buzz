## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: Technical Debt

Recorded factually, without severity judgement. Total surface: 541 LOC across 4
`.rs` files (`connection.rs` 314, `message.rs` 167, `error.rs` 51, `lib.rs` 9) plus a
17-line manifest.

---

### 1. Complexity hotspots

| Hotspot | Detail | file:line |
|---|---|---|
| Three near-identical frame-dispatch loops | `recv_one`, `wait_for_auth_challenge`, and `wait_for_ok` each contain the same `timeout(... self.ws.next())` + `match raw { Text / Ping / Close / _ }` skeleton, differing only in the `Text` arm and the timeout error | `connection.rs:133`–`154`, `:178`–`214`, `:235`–`268` |
| Duplicated Ping/Pong and Close handling | Same two arms written three times | `connection.rs:148`–`151`, `:208`–`:211`, `:262`–`:265` |
| Duplicated buffer-scan-then-remove pattern | `position(...)` followed by `remove(idx).unwrap()` and an `unreachable!()` arm, written twice with different predicates | `connection.rs:165`–`174`, `:224`–`:233` |
| Three challenge-intake paths with divergent validation | `pending_challenge.take()`, buffered `Auth`, and socket-read `Auth`; only the third applies the 1024-byte cap | `connection.rs:161`, `:170`, `:198` |
| Two timeout semantics in one file | `recv_one` applies `timeout_dur` **per read** inside its loop, while the two waiters use an absolute deadline; both are described to callers as "waiting up to" a duration | `connection.rs:133`–`134` vs `:176`–`185`, `:222`–`:242` |
| Redundant buffer drain | `next_event` pops the buffer, then `recv_one` pops it again as its first act | `connection.rs:108`–`111` and `:129`–`131` |
| Dual bookkeeping of a mid-flight `AUTH` | `wait_for_ok` stores the challenge in `pending_challenge` **and** pushes the same message onto `buffer`, so the same challenge is observable through two channels and can be consumed twice (once by `authenticate`, once by `next_event`) | `connection.rs:255`–`258` |
| Raw-vs-normalized URL split | The dialed URL is normalized (`parsed.as_str()`), the AUTH-event relay tag uses the original string | `connection.rs:53` vs `:63`, `:79` |

---

### 2. Dead / unused code

| Item | Status | file:line |
|---|---|---|
| `WsClientError::EventRejected(String)` | Never constructed anywhere in the repo. `send_event` returns `Ok(OkResponse { accepted: false, .. })` instead. Only `buzz-test-client` maps the variant across to its own error type (`crates/buzz-test-client/src/lib.rs:71`) | decl `error.rs:38`–`40`; non-use `connection.rs:96`–`101` |
| `From<nostr::event::builder::Error> for WsClientError` | No call site inside the crate — `build_auth_event` performs the same conversion inline with `map_err` rather than `?`, so the impl is only reachable by external callers | impl `error.rs:47`–`51`; inline duplicate `message.rs:166` |
| `WsClientError::UnexpectedMessage` for a *state* violation | The doc comment says "a message that was not expected at this point" (`error.rs:30`), but every construction site is a *parse* failure in `message.rs`; no state-machine violation ever produces it | `error.rs:30`–`32` vs `message.rs:61`, `:68`, `:73`, `:84`, `:102`, `:112`, `:136`, `:140` |
| `#[allow(clippy::result_large_err)]` | Suppression on `parse_relay_message` indicates the error enum is large (it wraps `tungstenite::Error`), left unaddressed rather than boxed | `message.rs:54` |
| Two `unwrap()` + `unreachable!()` pairs | Contradicts the repo rule against new `unwrap()` in production paths (`AGENTS.md`, Quality Gates); provably unreachable given the preceding `position` check, but still panic sites in a library | `connection.rs:170`, `:172`, `:229`, `:231` |

---

### 3. Test coverage gaps

Shipped tests: three compile-time constant-floor assertions
(`connection.rs:300`–`313`). No `tests/` directory, no `[dev-dependencies]`
(`Cargo.toml` has no such section), no `#[tokio::test]`, no mock relay.

| Untested behaviour | Where it lives |
|---|---|
| `parse_relay_message` — all six message types, missing/short arrays, wrong JSON types, unknown message type | `message.rs:55`–`144` |
| `OK` lenient defaults (`accepted` → `false`, `message` → `""`) | `message.rs:86`–`91` |
| `build_auth_event` — tag injection vs. no tag, invalid relay URL | `message.rs:151`–`167` |
| The whole NIP-42 handshake, including the `accepted == false` rejection path | `connection.rs:70`–`93` |
| 1024-byte challenge cap (and the two paths that bypass it) | `connection.rs:198`; bypasses at `:161`, `:170` |
| Buffering / out-of-order delivery, `OK` correlation by event id | `connection.rs:205`, `:227`, `:254`, `:259` |
| Timeout expiry → `NoAuthChallenge` / `Timeout`, deadline recomputation | `connection.rs:183`–`189`, `:240`–`:246` |
| Ping→Pong keepalive, `Close`/stream-end → `ConnectionClosed` | `connection.rs:148`–`151` (×3) |
| `publish_event` outer-budget behaviour and the discarded `disconnect` result | `connection.rs:284`–`293` |
| URL parsing/normalization and plaintext-`ws://` acceptance | `connection.rs:48`–`55` |

The crate's behaviour is exercised only indirectly, through `buzz-test-client`'s
integration suite (`crates/buzz-test-client/src/lib.rs:85`, `:98`, `:123`, `:154`,
`:175`), which requires a live relay and therefore does not run under `just test-unit`.

---

### 4. Duplication with other implementations in the repo

| Duplicate | Detail |
|---|---|
| `buzz-acp` reimplements this crate | `crates/buzz-acp/src/relay.rs` has its own `connect_async`, its own `RelayMessage` with an `Auth { challenge }` variant (`:492`, `:2344`), its own AUTH-message parse arm (`:3610`–`3616`), its own `send_auth_response` (`:3435`–`3461`), its own `wait_for_auth_challenge` (`:3864`) and handshake (`:3843`–`3845`), its own `"No auth challenge received"` error (`:447`), and its own `AUTH_TIMEOUT` — and it does **not** depend on `buzz-ws-client` |
| `buzz-acp` diverges functionally | It handles **mid-session AUTH challenges by re-authenticating** (`crates/buzz-acp/src/relay.rs:2344`–`2350`) and notes that `EventBuilder::auth()` cannot carry extra tags so it builds the event manually (`:3444`–`3456`) — whereas `buzz-ws-client` uses `EventBuilder::auth` plus `builder.tags([...])` (`message.rs:158`–`163`) and only records mid-session challenges (`connection.rs:255`–`258`) |
| `buzz-test-client` re-wraps rather than re-exports the connection | It defines its own wrapper type around `NostrWsConnection` and mirrors every `WsClientError` variant into a parallel `TestClientError` enum (`crates/buzz-test-client/src/lib.rs:65`–`72`, `:85`–`98`), so error variants must be kept in sync by hand |
| Other independent WebSocket clients | `crates/buzz-relay/src/router.rs`, `crates/buzz-relay/src/audio/handler.rs`, `crates/buzz-pairing-cli/src/main.rs`, `crates/buzz-pair-relay/tests/integration.rs`, `crates/buzz-test-client/tests/nip42_host_binding_live.rs`, `crates/buzz-test-client/tests/conformance_multitenant.rs`, `desktop/src-tauri/src/native_websocket.rs`, `desktop/src-tauri/src/huddle/relay_api.rs`, `desktop/src-tauri/src/commands/pairing.rs` all call `connect_async` directly rather than going through this crate (relay-side and test-side cases are inbound/other-protocol, so not all are candidates for consolidation) |
| Timeout knowledge duplicated in a caller | `buzz-cli` hardcodes `75` for the outer budget with a comment pointing at the three constants, so the relationship is documented in prose rather than derived in code | `crates/buzz-cli/src/client.rs:1077`, `:1080` |

---

### 5. Documentation gaps

| Gap | Evidence |
|---|---|
| **`buzz-ws-client` is absent from `ARCHITECTURE.md`'s crate reference.** The crate map lists `buzz-core`, `buzz-db`, `buzz-auth`, `buzz-pubsub`, `buzz-search`, `buzz-audit`, `buzz-workflow`, `buzz-relay`, `buzz-acp`, `buzz-sdk`, `buzz-media`, `buzz-cli`, `buzz-admin`, `buzz-test-client` — and no entry for `buzz-ws-client`; a repo-wide grep for the name in `*.md` matches only `AGENTS.md:63` | `ARCHITECTURE.md:78`–`94`; `AGENTS.md:63` |
| No module-level (`//!`) docs on any file, so `cargo doc` for the crate root shows only the re-export list | `lib.rs:1`–`9` |
| No `README.md` in the crate directory | `crates/buzz-ws-client/` contains only `Cargo.toml` and `src/` |
| No `CHANGELOG` entry or migration note for the mid-session-AUTH limitation, which callers must know about to stay authenticated | `connection.rs:255`–`258` |
| Doc/behaviour mismatch: `next_event` is documented as "waiting up to `timeout_dur`" but the underlying `recv_one` restarts the timer per frame | doc `connection.rs:103`; impl `connection.rs:133`–`134` |
| Doc/behaviour mismatch: `UnexpectedMessage`'s doc describes a protocol-state error; all uses are parse errors | `error.rs:30` vs `message.rs` (8 sites) |

---

### 6. Missing capabilities that callers must re-implement

| Missing | Consequence | Evidence |
|---|---|---|
| No reconnect/retry/backoff | Every `ConnectionClosed` is terminal for the caller; `publish_event` does a full fresh connect per call | `connection.rs:277`–`294`; no retry code in the file |
| No typed `REQ`/`CLOSE`/`COUNT` | Subscription framing is hand-built by each caller through `send_raw` | `connection.rs:121`; example `crates/buzz-test-client/src/lib.rs:154`, `:160` |
| No `COUNT` response variant | A `COUNT` reply surfaces as `UnexpectedMessage` | `message.rs:63`–`142` |
| No authenticated-state tracking | Nothing distinguishes an authenticated from an unauthenticated connection | `connection.rs:70`–`93` (no flag written) |
| No buffer capacity bound | `buffer` can grow for the duration of a wait | `connection.rs:28`, `:205`, `:257`, `:259` |
| No `Debug` impl on `NostrWsConnection` | Callers embedding it cannot derive `Debug` | `connection.rs:26` |

---

### 7. Deprecated API usage

None found. No `#[deprecated]` items are defined or referenced, and no compiler
deprecation shims appear in the source. Dependency versions in use are recent
workspace pins: `nostr` 0.44 (root `Cargo.toml:61`), `tokio-tungstenite` 0.29 (root
`Cargo.toml:113`), `thiserror` 2 (root `Cargo.toml:85`). The
`Message::Text(text.into())` conversion at `connection.rs:124` is the current
tungstenite 0.29 API shape (it requires an owned UTF-8 payload type rather than a
`String`), not a deprecated form.
