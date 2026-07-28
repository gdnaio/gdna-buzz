## Module: buzz-acp — relay client & observer (`crates/buzz-acp/src`)
### Aspect: Conventions

#### Actor-over-channels concurrency

One background tokio task owns the `WsStream` exclusively
(`relay.rs:615-634`); `HarnessRelay` never touches the socket. All
communication is through three `mpsc` channels:

| Channel | Direction | Capacity | Location |
|---|---|---|---|
| `event_tx`/`event_rx: Option<BuzzEvent>` | bg → harness | `event_channel_capacity()`, default 256 | `relay.rs:610` |
| `observer_control_tx`/`_rx: Event` | bg → harness | same as above | `relay.rs:611-612` |
| `cmd_tx`/`cmd_rx: RelayCommand` | harness → bg | `CMD_CHANNEL_CAPACITY` = 64 (`relay.rs:31`) | `relay.rs:613` |

`Option<BuzzEvent>` is used as an in-band sentinel: `None` = connection lost
(`relay.rs:811-813`). All recovery-path sends use `try_send` so a full event
channel cannot stall reconnection (`relay.rs:1697`, `:1770`, `:1826`, `:1885`,
`:1919`, `:2131`, `:2170`), with the reason stated inline at `relay.rs:1819-1821`.

`observer.rs` uses a different shape: `tokio::sync::broadcast` for live fan-out
plus a `std::sync::Mutex<VecDeque>` replay buffer, both sized by
`OBSERVER_BUFFER_CAP` (`observer.rs:45-53`). `AtomicU64` with
`Ordering::Relaxed` for the sequence counter (`observer.rs:107`).

#### Single-dispatch discipline

`handle_ws_message` (`relay.rs:2043-2390`) is the only frame handler.
`process_handshake_buffer` re-serialises buffered `RelayMessage`s back to JSON
text specifically to route them through it, with the tradeoff acknowledged in a
comment: "slightly wasteful but keeps the handler as the single source of truth"
(`relay.rs:2405-2407`).

Similarly `ws_send_timeout` (`relay.rs:3312-3323`) is the single send path —
"All `ws.send()` calls go through here" (`relay.rs:3307`). The three read loops
each answer `Ping` through it.

#### State-intent separation

Commands are applied to `BgState` independently of whether the wire send
succeeded. Three named helpers encode the policy:

| Helper | Location | Contract |
|---|---|---|
| `apply_command_to_state` | `relay.rs:1244-1290` | disconnected path; `Shutdown` arm is a `debug_assert!(false)` (`:1285-1288`) |
| `retain_failed_command_intent` | `relay.rs:1307-1319` | live-send-failure path; observer frames parked, other publishes discarded |
| `retain_deferred_command_intent` | `relay.rs:1324-1338` | replay-lost-socket path; drains a `VecDeque` in arrival order |

`execute_connected_command` (`relay.rs:1346-1531`) returns `bool` — `false`
means "dead socket, reconnect" — and its `Shutdown`/`Reconnect` arm is also a
`debug_assert!(false)` (`:1526-1529`). The invariants are documented at the
function level rather than encoded in the type: `RelayCommand` still carries the
control variants the function refuses to handle.

#### Typed outcome enums instead of booleans-plus-comments

`ReconnectOutcome { Ok, Failed, Shutdown }` (`relay.rs:2778-2787`) and
`ResubscribeResult { Ok, RetryConnection, Shutdown }` (`relay.rs:2457-2466`).
Callers `matches!` on `Shutdown` and return immediately; the doc comment on
`try_autonomous_reconnect` spells out why falling through would loop forever
(`relay.rs:2884-2888`). `send_subscribe`,
`send_membership_subscribe`, `send_observer_control_subscribe`, and
`send_publish_event_frame` all return bare `bool`.

#### `select!` conventions

The main loop is a four-arm `tokio::select!` (`relay.rs:1784-2021`): socket read,
command receive, ping tick, and a pacing timer. The pacing arm parks on
`std::future::pending()` when no drain is scheduled so it "never fires
spuriously and never blocks the other select! arms"
(`relay.rs:2011-2013`, implementation `:1991-1998`).

Every sleep that could swallow a `Shutdown` is a `select!` over
`sleep_until(deadline)` and `cmd_rx.recv()`. Three variants:
`pacing_sleep` defers commands for later live execution
(`relay.rs:3369-3392`); `dns_flat_sleep` applies them to state immediately
(`relay.rs:3395-3411`); the two inline backoff sleeps also apply to state
(`relay.rs:2996-3008`, `:3133-3145`). Deadlines rather than `sleep(d)` are used
so command traffic cannot reset the timer — stated at `relay.rs:2990-2992` and
`:3121-3126`.

`ping_interval` sets `MissedTickBehavior::Delay` (`relay.rs:1613`).

#### Error handling

`RelayError` is a `thiserror` enum (`relay.rs:435-459`). `WebSocket` boxes the
inner `tungstenite::Error` (`relay.rs:437`) to keep the enum small. Errors are
mapped with `map_err` closures at every boundary — `RelayError::Http` is reused
as a catch-all for URL parse (`relay.rs:3831`, `:3440`), tag parse
(`relay.rs:3446`, `:3448`), reqwest client build (`relay.rs:594`), and NIP-98
signing (`relay.rs:272`, `:294`, `:296`), which flattens genuinely different
failures into one variant.

**Zero `unwrap()` / `expect()` / `panic!()` in the production half of all three
files.** `relay.rs` lines 1–3,994 contain none; the first occurrence is
`relay.rs:4040`, inside `mod tests`. `observer.rs` has none at all.
`engram_fetch.rs` has 9, all inside `#[cfg(test)] mod tests` (lines 178-234).
`unwrap_or_default()` is used freely on `SystemTime` arithmetic
(`relay.rs:257`, `:3190`, `:3245`, `:3282`, `:3341`), and
`nip98_header(...).unwrap_or_default()` at `relay.rs:379` silently yields an
empty `Authorization` header on signing failure — the comment claims it is
"infallible in practice" (`relay.rs:377-378`).

`#![deny(unsafe_code)]` is set crate-wide (`lib.rs:1`); no `unsafe` block exists
in any of the three files. The one place that reaches for a raw pointer
alternative avoids it explicitly: `largest_shrinkable_leaf`'s two-pass borrow
dance in `lib.rs` is annotated "keep the borrow checker happy without unsafe".

#### Logging discipline

`tracing` only, imported as `use tracing::{debug, info, warn};`
(`relay.rs:126`). No `println!`/`eprintln!` in any of the three files.
No `error!` level is used anywhere in `relay.rs` — the most severe events
(reconnect exhaustion, dropped telemetry) are `warn!`.

Structured fields are used inconsistently: some sites use key-value form
(`relay.rs:1218-1221`, `:2103-2106`, `:2160-2168`, `:2656-2659`) while most use
interpolated messages (`relay.rs:1408`, `:2246-2254`, `:2688`, `:3186`).
`observer.rs` uses `target: "observer"` on its two warnings
(`observer.rs:98`, `:130`); `engram_fetch.rs` uses `target: "engram::core"`
(`engram_fetch.rs:51`). `relay.rs` sets no `target`.

Sensitive-value discipline is good on the AUTH path: `relay.rs:3461` logs a
fixed string, never the frame. It is weaker on the failure path:
`relay.rs:2059` logs the entire unparsed relay frame
(`warn!("failed to parse relay message: {e} — raw: {text}")`).

#### Documentation style

Long rationale comments are the dominant convention — most constants carry a
paragraph explaining the number (`relay.rs:47-113`), and several functions have
20-40 line doc comments covering invariants and known tradeoffs
(`relay.rs:2468-2487` on `resubscribe_after_reconnect`, `:3625-3656` on
terminal-error classification, `:925-933` on the dedup amnesia window,
`:3489-3496` on the CLOSED string matching). Cross-file coupling is called out
explicitly where it exists (`relay.rs:3494-3496` names `req.rs:153` and
`side_effects.rs:71`; `relay.rs:3762-3763` names the rustls version).

Every public item in the three files has a doc comment. `#[allow(...)]` is used
sparingly and always locally: `clippy::too_many_arguments` on the five
9-argument background functions (`relay.rs:1533`, `:2042`, `:2392`, `:2892`,
`:3021`), `dead_code` on `HarnessRelay::publish_event` (`relay.rs:820`),
`private_interfaces` on `parse_relay_message` (`relay.rs:3532`).

#### Naming

`send_*` for wire writes, `drain_*` for paced queue processing, `is_terminal_*`
/ `is_dns_error` / `is_retriable_status` for classifiers, `*_sleep` for
shutdown-aware waits, `apply_*`/`retain_*` for state mutation. Subscription ids
are built and parsed by a matched pair, `channel_sub_id` (`relay.rs:3478-3480`)
and `channel_id_from_sub_id` (`relay.rs:3484-3488`), with a round-trip test
(`relay.rs:4048-4053`).

#### Test conventions

All tests are inline `#[cfg(test)] mod tests` — `relay.rs:3995-6233` (2,239
lines, 36 % of the file), `engram_fetch.rs:167-247`. `observer.rs` has no tests
of its own. Async tests use `#[tokio::test(start_paused = true)]` (10
occurrences) so backoff and pacing are asserted deterministically. Real
WebSocket pairs are built over `127.0.0.1:0` by the `test_ws_pair` helper
(`relay.rs:4340-4355`) with `next_test_frame` for assertions
(`relay.rs:4357-4367`). `RelayEventPublisher::test_pair` (`relay.rs:575-588`) is
a `#[cfg(test)]` seam that forwards published events to a receiver instead of a
socket. Test names read as behaviour statements
(`dropped_channel_is_not_resubscribed_so_loop_cannot_re_form`,
`:5044`; `membership_dedup_does_not_touch_last_seen`, `:4872`).
