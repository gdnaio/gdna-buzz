## Module: buzz-acp — relay client & observer (`crates/buzz-acp/src`)
### Aspect: Configuration

#### Environment variables read in these three files

Exactly one:

| Variable | Parse site | Type | Default | Gates | In `.env.example`? |
|---|---|---|---|---|---|
| `BUZZ_ACP_EVENT_BUFFER` | `relay.rs:35-42` | `usize`, `.max(1)` | `EVENT_CHANNEL_CAPACITY_DEFAULT` = 256 (`relay.rs:29`) | capacity of **both** the `event_tx` channel (`relay.rs:610`) and the `observer_control_tx` channel (`relay.rs:611-612`) | Yes — `.env.example:221`, commented out: `# BUZZ_ACP_EVENT_BUFFER=256` |

`event_channel_capacity()` is called once per `HarnessRelay::connect`
(`relay.rs:610-612`), so the value is read at connect time, not process start.
The `.max(1)` guard exists because `mpsc::channel` panics on capacity 0
(`relay.rs:40`). A non-numeric value silently falls back to 256 — no warning is
emitted.

The doc comment at `relay.rs:27-28` says the variable overrides "the event
channel" capacity; it is actually applied to two channels. `.env.example:219-220`
describes it as "Event channel buffer capacity (WebSocket → harness)", which
matches the doc comment and understates the observer-control effect.

`observer.rs` and `engram_fetch.rs` read no environment variables.

#### Variables consumed by this module but read elsewhere

These reach `relay.rs` as function arguments, not through `std::env` in these
files:

| Variable | Read at | Path into this module |
|---|---|---|
| `BUZZ_RELAY_URL` (via `Config.relay_url`) | `config.rs` clap `env` attribute | `HarnessRelay::connect(&config.relay_url, …)` (`lib.rs:1344`), stored at `relay.rs:545`, used by `do_connect` (`relay.rs:3830`), `send_auth_response` (`relay.rs:3440`, `:3446`), and `relay_ws_to_http` for `RestClient.base_url` (`relay.rs:722`) |
| `BUZZ_PRIVATE_KEY` (via `Config.keys`) | `config.rs` | `HarnessRelay::connect(…, &config.keys, …)`, cloned into `bg_keys` (`relay.rs:615`) and `RestClient.keys` (`relay.rs:723`) |
| `BUZZ_AUTH_TAG` | `lib.rs:1370-1373` (`std::env::var` + `buzz_sdk::nip_oa::parse_auth_tag`) | `auth_tag: Option<nostr::Tag>` argument, stored at `relay.rs:549`, serialised to `auth_tag_json` for the `x-auth-tag` header (`relay.rs:724-727`), and injected into the AUTH event (`relay.rs:3444-3456`) |

`BUZZ_ACP_NO_TYPING` (`.env.example:216`) gates `config.typing_enabled`, which
controls whether the 3-second refresh tick that calls `build_typing_event` is
created at all (`lib.rs:1593-1596`). `BUZZ_ACP_NO_MEMORY` gates
`ctx.memory_enabled`, which controls whether `engram_fetch::build_core_section`
is reached (`pool.rs:1382`). Neither is read in these files.

#### Compile-time constants — `relay.rs`

| Constant | Value | Location | Effect |
|---|---|---|---|
| `EVENT_CHANNEL_CAPACITY_DEFAULT` | 256 | `:29` | fallback for `BUZZ_ACP_EVENT_BUFFER` |
| `CMD_CHANNEL_CAPACITY` | 64 | `:31` | harness → bg command channel; not env-tunable |
| `SEEN_ID_LIMIT` | 12_000 | `:45` | dedup set, rotates at 6,000 |
| `PING_INTERVAL` | 30 s | `:48` | client-initiated ping |
| `PONG_TIMEOUT` | 10 s | `:51` | no-pong → reconnect |
| `WS_SEND_TIMEOUT_SECS` | 10 | `:54` | every `ws.send()` |
| `STABLE_CONNECTION_SECS` | 60 | `:59` | resets `backoff_step` to 0 |
| `SINCE_SKEW_SECS` | 5 | `:57` | subtracted from `since` on resubscribe |
| `AUTH_TIMEOUT` | 20 s | `:64` | both NIP-42 steps; comment records the raise from 5 s |
| `CONNECT_TIMEOUT` | 30 s | `:69` | TCP + WS handshake; comment records the raise from 10 s |
| `STARTUP_CONNECT_BACKOFFS` | `[1,2,4,8,16] s` | `:83-89` | shared by `retry_initial_connect` and `try_autonomous_reconnect` |
| `DNS_RETRY_INTERVAL` | 2 s | `:98` | flat DNS retry, no ladder rung |
| `REQ_PACING_INTERVAL` | 125 ms | `:107` | ≈8 REQ/s |
| `DRAIN_BUDGET_PER_ITER` | 1 | `:113` | frames per select tick |
| `GATED_OBSERVER_QUEUE_CAP` | 256 | `:116` | parked + in-flight observer frames |
| `REST_RETRY_BASE_DELAYS` | `[500, 1000, 2000] ms` | `:249-253` | HTTP bridge retry |
| `MEMBERSHIP_NOTIF_SUB_ID` | `"membership-notif"` | `:497` | subscription id |
| `OBSERVER_CONTROL_SUB_ID` | `"agent-observer-control"` | `:499` | subscription id |
| `CHANNEL_ACCESS_DENIED_REASONS` | two exact strings | `:3498-3501` | drop-channel-not-connection |

Local (function-scope) constants: `MAX_DNS_FLAT_RETRIES` = 10
(`relay.rs:2914`) bounds the DNS brownout retry in `try_autonomous_reconnect`;
the `wait_for_reconnect` ladder `[1,2,4,8,16,32] s` with a 60 s tail is an inline
array (`relay.rs:3055-3062`, `:3128-3132`) rather than a named constant, so it
cannot be tuned or asserted alongside `STARTUP_CONNECT_BACKOFFS`.

None of these are env-overridable. Notably `CMD_CHANNEL_CAPACITY` (64) bounds
`try_publish_event`, which is the typing-indicator path — a full command channel
silently drops the event (`relay.rs:834-840`) — and there is no way to raise it
without a rebuild.

#### Compile-time constants — `observer.rs` / `engram_fetch.rs`

| Constant | Value | Location | Effect |
|---|---|---|---|
| `OBSERVER_BUFFER_CAP` | 1_000 | `observer.rs:19` | sizes both the broadcast channel (`:46`) and the replay `VecDeque` (`:51`) |
| `SECTION_LABEL` | `"Agent Memory — core"` | `engram_fetch.rs:23` | prompt section header |
| `ONBOARDING_NUDGE` | fixed sentence | `engram_fetch.rs:29-31` | rendered on confirmed absence |

The engram query's `limit(16)` is a bare literal at `engram_fetch.rs:88`, not a
named constant.

#### Parsed-and-never-read / unused configuration

- `send_subscribe` takes `_state: &BgState` and never reads it (`relay.rs:3162`) — a vestigial parameter threaded through six call sites (`relay.rs:1390`, `:2306-2313`, `:2544`, `:2708`, `:2765`).
- `HarnessRelay::publish_event` is `#[allow(dead_code)]` (`relay.rs:820-821`); no in-repo caller uses it.
- `relay.rs:3849-3854` computes `event_id` from `wait_for_any_ok` and uses it only in a debug log (`relay.rs:3856`) — the value it was intended to match against is never derived.
- No env var in this module is parsed and then discarded; `BUZZ_ACP_EVENT_BUFFER` is the only one and it is used.

#### Documentation drift on configuration

`.env.example:219-220` and the doc comment at `relay.rs:27-28` both describe
`BUZZ_ACP_EVENT_BUFFER` as sizing the WebSocket → harness event channel only.
It also sizes the observer-control channel (`relay.rs:611-612`), so an operator
tuning it for high message throughput silently changes observer-control
backpressure behaviour too — and that channel drops on full with a warning
(`relay.rs:2072-2074`) rather than triggering the replay machinery that channel
events get.

`ARCHITECTURE.md:452-458` documents typing indicators as a Redis sorted-set
protocol; the kind:20002 ephemeral event this module builds
(`relay.rs:866-868`, published from `lib.rs:2333-2341`) is handled by the
generic ephemeral fan-out instead — there is no `KIND_TYPING_INDICATOR`
reference in `crates/buzz-relay/src` or `crates/buzz-pubsub/src`. Any operator
reading that section to size Redis is sizing something this path does not use.
