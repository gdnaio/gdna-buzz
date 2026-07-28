## Module: buzz-acp — relay client & observer (`crates/buzz-acp/src`)
### Aspect: Technical Debt

#### File size

| File | Lines | Production | Tests |
|---|---|---|---|
| `relay.rs` | 6,233 | 1–3,994 (3,994) | 3,995–6,233 (2,239, 36 %) |
| `engram_fetch.rs` | 248 | 1–166 | 167–247 (81) |
| `observer.rs` | 166 | all | none |

`relay.rs` at 6,233 lines is a single module holding: the harness handle, the
HTTP bridge client, the wire-format parser, the NIP-42 handshake, the background
task, `BgState`, four drain loops, three reconnect loops, the terminal-error
classifier, and 70 tests. The mobile guard enforces a 1,000-line ceiling
(`mobile/scripts/check-file-sizes.mjs`, per `AGENTS.md`); no equivalent guard
covers `crates/`, so this file has grown past 6× that without tripping anything.

Natural seams already exist and are unused: `RestClient` (`relay.rs:231-424`,
~194 lines) has no dependency on `WsStream` and is doc-commented as
"Extracted from `HarnessRelay` fields so it can be shared" (`relay.rs:227-229`)
— it is a self-contained HTTP client living in a WebSocket file.
`parse_relay_message` + `RelayMessage` + `OkResponse` (`relay.rs:471-495`,
`:3531-3620`, `:3917-3921`) are a self-contained codec.
`is_terminal_connect_error` and its three helpers (`relay.rs:3625-3775`,
~150 lines) are a pure classifier with 40 lines of doc comment.

#### Reimplementation of `buzz-ws-client`

`crates/buzz-acp/Cargo.toml` does not depend on `buzz-ws-client`. The full
side-by-side duplication table is in `acp-relay-integrations.md`; the summary:

| Duplicated behaviour | `relay.rs` | `buzz-ws-client/src/connection.rs` |
|---|---|---|
| `WsStream` alias | `:525` | `:14` |
| URL parse + `connect_async` | `:3830-3837` | `:46-52` |
| `debug!("connected to relay at {url}")` | `:3838` | `:57` |
| `RelayMessage` enum | `:470-495` | `message.rs:8-41` |
| `OkResponse` | `:3917-3921` | `message.rs:44-52` |
| Frame parser | `:3531-3620` | `message.rs:55-144` |
| AUTH-challenge wait | `:3865-3913` | `:156-206` |
| AUTH OK wait | `:3924-3982` | `:208-263` |
| Build + send AUTH | `:3433-3461` | `:70-93` + `message.rs:151-166` |
| Ping → Pong in read loop | `:2370-2376`, `:3903-3907`, `:3963-3967` | `:148-150`, `:208-210`, `:262-264` |
| Frame buffering `VecDeque` | `:3833`, `:3902`, `:3960` | `:28`, `:205`, `:257` |
| 20 s AUTH timeouts | `:64` | `:17`, `:20` |

Roughly 300 lines of duplicated protocol code with three behavioural
divergences: mid-session AUTH (re-auth vs. record-only), AUTH-OK matching
(any-OK vs. id-matched), and the 1024-byte challenge cap (absent vs. present).
Two of the three make this copy weaker. The one place `relay.rs` is stronger —
not logging the AUTH frame — means neither copy can simply adopt the other.

`buzz-ws-client` pins its timeout floors with compile-time tests
(`buzz-ws-client/src/connection.rs:297-313`). `relay.rs` has no equivalent for
`AUTH_TIMEOUT` or `CONNECT_TIMEOUT`, so the two can drift silently.

#### Structural duplication inside the file

The membership `since` computation `match (dropped_since, last_seen)` is written
out four times, identically:

| Site | Location |
|---|---|
| `execute_connected_command` | `relay.rs:1421-1428` |
| main-loop drain | `relay.rs:1721-1728` |
| CLOSED handler | `relay.rs:2295-2301` |
| `resubscribe_after_reconnect` | `relay.rs:2569-2574` |

`channel_since` (`relay.rs:1115-1131`) exists as the channel-side equivalent; no
`membership_since` was extracted.

The reconnect-then-drain-then-fallback block is duplicated five times in
`run_background_task`, each ~40 lines of nested `match` on `ReconnectOutcome`
followed by four state resets (`ping_sent`, `last_pong`, `connected_since`,
`stable_logged`):

| Trigger | Location |
|---|---|
| Handshake-buffer drop signal | `relay.rs:1697-1750` |
| Proactive resubscribe failure | `relay.rs:1600-1652` |
| Socket lost on read | `relay.rs:1766-1830` |
| Command send failure | `relay.rs:1804-1852` |
| Ping/pong death | `relay.rs:1861-1927` |

The `select!` body in that function shows the resulting formatting damage:
indentation is visibly inconsistent from `relay.rs:1702` onward, with argument
lists broken across columns (`relay.rs:1776-1780`, `:1836-1840`) — the block is
past what `rustfmt` can lay out cleanly.

Two shutdown-aware sleeps differ only in what they do with a non-`Shutdown`
command: `pacing_sleep` defers it (`relay.rs:3369-3392`), `dns_flat_sleep`
applies it to state (`relay.rs:3395-3411`). The two inline backoff sleeps
(`relay.rs:2996-3008`, `:3133-3145`) re-implement `dns_flat_sleep`'s body
without the jitter.

Three near-identical REQ builders: `send_subscribe` (`relay.rs:3160-3222`),
`send_membership_subscribe` (`:3227-3270`), `send_observer_control_subscribe`
(`:3273-3305`). Each hand-builds a `serde_json::Map` or inline `json!`, each
repeats the same `serde_json::to_string` → `ws_send_timeout` → log-on-error
tail. `nostr::Filter` is used for HTTP queries but not for WS REQs, so filter
construction is done two different ways in the same file.

#### Known correctness gaps left in place

| Gap | Evidence |
|---|---|
| AUTH OK is not matched to the AUTH event id | `relay.rs:3846-3854` — the comment describes re-deriving the id as "wasteful" and settles for the first OK. An unrelated `OK` arriving first is accepted as the auth result. |
| No challenge length cap on any of three intake paths | `relay.rs:3901`, `:3610-3616`, `:2344` |
| No `ws://` scheme rejection | `relay.rs:3830-3832` |
| EOSE is inert | `relay.rs:2190-2192` — no initial-replay-complete signal |
| Wildcard REQ omits `kinds`, tripping the relay p-gate | `relay.rs:3172-3175`; reachable via `config.rs:1180`, `:1272` (`--subscribe-mode all` with no `--kinds`) and `config.rs:1195-1196`, `:1286-1287` (empty-`kinds` rule). `AGENTS.md` states omitting `kinds` returns 403. No guard, no warning. |
| Dedup amnesia window | `relay.rs:925-933` documents that an id seen 6,001+ inserts ago may be replayed as new |
| `send_subscribe` takes an unused `&BgState` | `relay.rs:3162`, threaded through six call sites |
| `#[allow(dead_code)] pub async fn publish_event` | `relay.rs:820-821` — no in-repo caller |
| Full unparsed frame logged on parse failure | `relay.rs:2059` |
| In-flight-batch / re-add race | documented at `relay.rs`'s caller (`lib.rs`), acknowledged as accepted rather than fixed |
| `RelayError::Http` used as a catch-all | URL parse (`relay.rs:3831`, `:3440`), tag parse (`:3446`, `:3448`), reqwest build (`:594`), NIP-98 signing (`:272`, `:294`, `:296`) all collapse into one variant |
| `nip98_header(...).unwrap_or_default()` sends an empty `Authorization` on signing failure | `relay.rs:379`, justified as "infallible in practice" (`:377-378`) |
| `wait_for_reconnect` ladder is an inline array | `relay.rs:3055-3062` — cannot be asserted against `STARTUP_CONNECT_BACKOFFS` |

#### Repo-rule compliance

| Rule | Status |
|---|---|
| No `unsafe` | Clean. `#![deny(unsafe_code)]` at `lib.rs:1`; no `unsafe` in any of the three files. |
| No new `unwrap()`/`expect()` in production paths | Clean. `relay.rs` production half (1–3,994): **0** occurrences — first is `relay.rs:4040`, inside `mod tests`. `observer.rs`: **0** anywhere. `engram_fetch.rs`: **9**, all in `mod tests` (`:178`, `:190`, `:191`, `:192`, `:212`, `:213`, `:232`, `:233`, `:234`). |
| New public API must have doc comments | Clean across all three files. |

`unwrap_or_default()` on `SystemTime` arithmetic appears five times
(`relay.rs:257`, `:3190`, `:3245`, `:3282`, `:3341`) — not a rule violation, but
it silently maps a pre-epoch clock to `since = 0`, which would request the
relay's full history.

#### TODO / FIXME / HACK / XXX

**Zero** across all three files. The known gaps are instead written as prose in
doc comments — e.g. `relay.rs:3846-3854` (AUTH-OK matching), `:925-933` (dedup
amnesia), `:1487-1493` (the unenforced "only ephemeral kinds on this path"
invariant), `:2481-2485` (gate survives socket replacement). This makes them
invisible to any tooling that greps for markers, and several read as
justifications rather than open items.

#### Test coverage

`relay.rs` `mod tests` (`:3995-6233`) contains **80 test functions**: 62
`#[test]` and 18 `#[tokio::test]` (10 of those with `start_paused = true`), plus
7 helpers (`meta_event` `:4070`, `make_test_event` `:4331`, `test_ws_pair`
`:4340`, `next_test_frame` `:4357`, `test_channel_filter` `:4369`,
`seed_test_subscription` `:4376`, and the nested `ws` shim at `:5079`).

Covered well:

| Area | Representative tests |
|---|---|
| URL / sub-id helpers | `:3999-4068` (10 tests) |
| `merge_discovered_channels` incl. archived | `:4087-4149` (5 tests) |
| `parse_relay_message` all variants + malformed | `:4151-4311` (14 tests) |
| Replay pacing with live socket | `:4388-4536` (4 async tests) |
| `TwoGenDedup` + `last_seen` | `:4566-4753` (9 tests) |
| Backpressure `dropped_since` bookkeeping | `:4755-4925` (5 tests) |
| CLOSED access-denial exact matching | `:4941-5071` (6 tests) |
| Terminal-error classification (exhaustive, `:5073`) | `:5073-5561` |
| `retry_initial_connect` incl. sleep assertions | `:5563-5702` (5 async tests) |
| Rate-limit gate + hint parsing | `:5704-5786` (6 tests) |
| Gated observer queue, ordering, overflow | `:5788-5996` (5 tests) |
| DNS classification | `:5998-6034` |
| Drain / resub retry paths | `:6036-6232` (7 tests) |

Not covered:

| Gap |
|---|
| `do_connect`'s happy path — no test drives a full AUTH challenge → AUTH event → OK handshake against the `test_ws_pair` fixture. The only `do_connect` test is the wrong-scheme terminal case (`:5549`). |
| Mid-session AUTH re-authentication (`relay.rs:2344-2353`) — the divergent behaviour has no test. |
| `wait_for_any_ok`'s accept-any-OK semantics — untested, so the known gap is unpinned. |
| `wait_for_reconnect` (`relay.rs:3022-3151`) — the unbounded loop with the 60 s tail and persisted `backoff_step`; `:2029-2038`'s stability reset is also untested. |
| `process_handshake_buffer` (`relay.rs:2393-2450`) — including the deliberate AUTH-skip at `:2433`. |
| EOSE handling. |
| `send_subscribe` with `kinds: None` — the p-gate-tripping wildcard REQ has no test asserting the emitted frame shape. |
| `RestClient` — `sign_nip98`, `nip98_header`, `request_with_retry`, `bridge_post`, `query`, `submit_event` (`relay.rs:261-424`, ~164 lines) have **no tests in this module**. `discover_channels` (`:657-714`) is likewise untested; only its pure tail `merge_discovered_channels` is. |
| `observer.rs` — **zero tests in the file**. `emit`/`snapshot`/`subscribe` are exercised indirectly by `lib.rs`'s `observer_snapshot_race_tests` and `observer_publish_pacer_tests`, but the 1,000-entry drop-front eviction and the poisoned-mutex degradation paths (`observer.rs:96-100`, `:129-131`) are not asserted anywhere. |

`engram_fetch.rs` has 5 tests (`:167-247`) and they cover the decision table
well — confirmed absence (`:177`), happy path (`:186`), the undecryptable →
`Err` regression (`:206`), non-`Core` body → absent (`:229`), unparseable →
`Err` (`:241`). `fetch_core_body`'s filter construction and `build_core_section`'s
rendering are untested (no HTTP fixture), so the `[Agent Memory — core]` header
format and the `limit(16)` are unpinned.

#### Documentation drift

| Claim | Reality |
|---|---|
| `relay.rs:5` "discovers channels via REST API" | Uses the Nostr HTTP bridge `POST /query` (`relay.rs:399-406`, `:670`, `:705`), not a REST resource API. Same wording at `relay.rs:230` ("Lightweight HTTP client … via the Nostr HTTP bridge") is accurate — the header is stale. |
| `relay.rs:27-28` / `.env.example:219-220` — `BUZZ_ACP_EVENT_BUFFER` sizes "the event channel" | Sizes two channels: `event_tx` and `observer_control_tx` (`relay.rs:610-612`) |
| `ARCHITECTURE.md:452-458` — typing indicators use a Redis sorted set with a 5 s window and 60 s TTL | The kind:20002 event built at `relay.rs:866-868` and published from `lib.rs:2333-2341` is handled by the generic ephemeral (20000–29999) fan-out; `grep KIND_TYPING_INDICATOR crates/buzz-relay/src crates/buzz-pubsub/src` returns nothing |
| `relay.rs:1341-1343` — `execute_connected_command` "handles the five data commands" | It handles six: `Subscribe`, `Unsubscribe`, `SubscribeMembership`, `SubscribeObserverControls`, `PublishEvent`, `SetStartupWatermark` (`relay.rs:1352-1527`) |
| Kind numbers repeated in prose | `relay.rs:162` (39000), `:262` (27235), `:653-654` (39002/39000), `:842` (20002), `:1036`/`:1469` (24200), `:3225` (44100+44101) — each is a drift surface independent of `buzz-core/src/kind.rs` |
| `relay.rs:3489-3496` names `req.rs:153` and `side_effects.rs:71` as the only CLOSED senders | A cross-crate coupling asserted in a comment with no test or compile-time link |
| `relay.rs:3762-3763` — the rustls downcast "relies on a single rustls version in the dep tree (0.23.40)" | `crates/buzz-acp/Cargo.toml` pins `rustls = { version = "0.23", … }` — a caret range, so a patch bump satisfying `0.23` could break the downcast without a version-conflict error |
