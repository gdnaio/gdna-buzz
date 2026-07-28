## Module: buzz-acp — relay client & observer (`crates/buzz-acp/src`)
### Aspect: Business Rules

#### NIP-42 handshake sequence

`do_connect` (`relay.rs:3825-3862`) is the whole handshake:

1. Parse the URL with `url::Url` (`relay.rs:3830-3832`) — parse only, no scheme assertion.
2. `connect_async` wrapped in `CONNECT_TIMEOUT` = 30 s (`relay.rs:3834-3837`); a timeout maps to `RelayError::ConnectionClosed` (`relay.rs:3836`).
3. `wait_for_auth_challenge(ws, &mut buffer, AUTH_TIMEOUT)` — `AUTH_TIMEOUT` = 20 s (`relay.rs:64`, called at `relay.rs:3843`). Non-AUTH frames are pushed into `buffer` (`relay.rs:3902`); `Ping` is answered with `Pong` (`relay.rs:3903-3907`); `Close` → `ConnectionClosed` (`relay.rs:3909`).
4. `send_auth_response` (`relay.rs:3845`, defined `:3433-3461`). With no `auth_tag` it uses `EventBuilder::auth(challenge, relay_nostr_url)` (`relay.rs:3457`); with one it hand-builds `["relay", url]`, `["challenge", challenge]`, plus the NIP-OA tag and signs `Kind::Authentication` (`relay.rs:3444-3456`), because `EventBuilder::auth` accepts no extra tags (`relay.rs:3443`).
5. `wait_for_any_ok` (`relay.rs:3849`, defined `:3924-3982`) accepts the **first** OK of any event id. The inline comment concedes the event id is not tracked (`relay.rs:3847-3854`). `accepted == false` → `RelayError::AuthFailed(ok.message)` (`relay.rs:3850-3852`).
6. Returns `(ws, buffer)`; the buffer is replayed by `process_handshake_buffer` (`relay.rs:2393-2450`).

`retry_initial_connect` (`relay.rs:3786-3822`) wraps step 1-6: 1 immediate
attempt + 5 delayed retries over `STARTUP_CONNECT_BACKOFFS` = 1/2/4/8/16 s
(`relay.rs:83-89`), each jittered ±20 %. A terminal error short-circuits
(`relay.rs:3809-3812`).

Terminal-vs-transient classification is exhaustive and compile-checked:
`is_terminal_connect_error` (`relay.rs:3657-3664`) → `is_terminal_ws_error`
(`relay.rs:3669-3717`, no wildcard arm) → `is_terminal_rustls_io_error`
(`relay.rs:3724-3766`, source-chain downcast to `rustls::Error` with a
5-variant terminal allowlist at `:3757-3764`). Auth denials split on the NIP-01
prefix: only `error:` is retried (`is_terminal_auth_failure`,
`relay.rs:3773-3775`).

#### Mid-session AUTH: this module re-authenticates

On a mid-session `AUTH` frame `handle_ws_message` calls `send_auth_response`
immediately and forces a reconnect if the send fails
(`relay.rs:2344-2353`). A rejected re-auth (`OK false` with a message starting
`auth`) also forces reconnect (`relay.rs:2359-2363`).

This is a functional divergence from the shared crate:
`buzz-ws-client` only records the challenge — `recv_one` stores it in
`pending_challenge` and returns the message
(`buzz-ws-client/src/connection.rs:144`); `wait_for_ok` does the same and
re-buffers the frame (`buzz-ws-client/src/connection.rs:256-258`). No
re-authentication is ever issued.

Buffered `AUTH` frames from the handshake window are **not** answered — they are
skipped on replay (`relay.rs:2432-2433`).

#### Subscription lifecycle

Subscribe intent is recorded before, and independently of, the wire send.
`execute_connected_command` (`relay.rs:1346-1531`):

- Rate-gated → `apply_command_to_state` + park in `rate_limited_pending`, return `true` ("connection is fine") (`relay.rs:1360-1375`).
- Otherwise seed `subscribe_since` (`relay.rs:1380-1384`), compute `since = last_seen.or(subscribe_since)` (`relay.rs:1385-1389`), send.
- On success: insert into `active_subscriptions` + `active_filters`, evict stale drain entries (`relay.rs:1392-1404`).
- On failure: `apply_command_to_state` records intent, return `false` → caller reconnects (`relay.rs:1405-1417`).

`since` policy in `send_subscribe` (`relay.rs:3185-3194`): `Some(ts)` →
`ts - SINCE_SKEW_SECS` (5 s, `relay.rs:57`); `None` → `now`, deliberately
skipping history.

`Unsubscribe` is best-effort at the socket level: a failed CLOSE frame does not
fail the command because state has already been mutated
(`relay.rs:1419-1432`).

CLOSED handling is a four-way branch (`relay.rs:2210-2346`):

1. Exact per-channel denial → drop just that channel, keep the socket. Only the two literals in `CHANNEL_ACCESS_DENIED_REASONS` match — `"restricted: not a channel member"`, `"restricted: channel access revoked"` (`relay.rs:3498-3501`) — compared with `contains(&message)`, never `starts_with`, so connection-level `restricted: insufficient scope` still reconnects (`relay.rs:3515-3517`, rationale at `:3489-3496`).
2. `rate-limited:` prefix → arm the gate, park, keep the socket (`relay.rs:2226-2246`).
3. `auth-required` / `starts_with("restricted")` / `contains("auth")` → return `false`, full reconnect (`relay.rs:2250-2264`).
4. Anything else → targeted resubscribe of that one subscription, with a fail-closed guard: a missing `active_filters` entry triggers reconnect rather than a wildcard REQ (`relay.rs:2308-2318`, and `:2513-2521` on the reconnect path). A CLOSED for an already-unsubscribed channel is ignored so it cannot resurrect a subscription (`relay.rs:2300-2306`).

#### EOSE handling

`RelayMessage::Eose` is logged at debug and otherwise discarded
(`relay.rs:2190-2192`). Nothing in `BgState` records EOSE, so the module cannot
distinguish stored-history replay from live events, and no caller is told when
initial replay finished.

#### Dedup

Channel events go through `record_event` (`relay.rs:1091-1107`): insert into
`TwoGenDedup`, return `false` on duplicate, otherwise advance `last_seen` with
`max` (`relay.rs:1100-1105`).

Membership notifications deliberately bypass `record_event` and touch
`seen_ids` directly (`relay.rs:2093-2100`) so membership timestamps cannot
contaminate per-channel replay watermarks — the reason is spelled out at
`relay.rs:2086-2092`.

On backpressure the id is **removed** from the dedup set so reconnect replay can
re-deliver it (`relay.rs:2124` for membership, `:2162` for channel events).

#### Backpressure → proactive resubscribe

`event_tx.try_send` full (`relay.rs:2121-2143` membership, `:2155-2181`
channel):
1. `seen_ids.remove(id)`.
2. Record the oldest dropped ts (`membership_dropped_since` via `min`, or `channel_dropped_since` entry via `min`).
3. Set `proactive_resubscribe_needed = true`.

The main loop checks the flag at the top of every iteration and resubscribes on
the **existing** socket with `is_fresh_connection = false`, explicitly
preserving the rate-limit gate (`relay.rs:1628-1640`). An 80 %-capacity warning
fires before the drop (`relay.rs:2109-2113`, `:2144-2151`).

#### Rate-limit gate

`set_rate_limit_gate` (`relay.rs:1156-1165`): a hint below 2 s (including the
no-hint 0) floors to 5 s; the deadline is jittered; the gate takes the **max**
of any existing deadline so overlapping CLOSED/NOTICE cannot shorten it.
`check_rate_gate` lazily clears an expired gate (`relay.rs:1173-1181`).
`parse_rate_limit_retry_secs` splits on `"retry in "` and takes ASCII digits —
no regex (`relay.rs:3328-3332`).

The gate is armed from both `NOTICE` (`relay.rs:2193-2208`) and `CLOSED`
(`relay.rs:2226-2246`). A NOTICE additionally requeues all unacked observer
frames because NOTICE carries no event id (`relay.rs:2200`, implementation
`relay.rs:1202-1210`).

Drain pacing: `REQ_PACING_INTERVAL` = 125 ms (`relay.rs:107`) and
`DRAIN_BUDGET_PER_ITER` = 1 (`relay.rs:113`) — one frame per select-loop tick,
stated to stay under the relay's 50-frames/5 s window
(`relay.rs:103-106`). Drain order per tick (`relay.rs:1706-1779`): membership
control sub → observer control sub → `rate_limited_pending` →
`resubscribe_retry` → `gated_observer_pending`. When nothing is sent because the
gate is still armed, the pacing timer is set to the gate deadline so parked
observer frames drain promptly (`relay.rs:1774-1779`).

A failed drain REQ re-queues the channel with a flat +5 s penalty
(`relay.rs:2712-2716`).

#### Observer frame durability rule

Kind 24200 is treated as durable telemetry, every other WS publish as droppable
ephemera:

- While gated **or** while a parked backlog exists, observer frames are parked to preserve order (`relay.rs:1470-1481`).
- Every other publish is silently dropped while gated (`relay.rs:1483-1495`); the invariant that only ephemeral kinds reach this path is documented rather than enforced (`relay.rs:1487-1493`).
- Successful writes move to `observer_in_flight` (`relay.rs:1500-1502`), retired on the matching `OK` (`relay.rs:2364`, implementation `relay.rs:1224-1232`).
- Both queues are bounded at `GATED_OBSERVER_QUEUE_CAP` = 256 with drop-oldest and a visible counter (`relay.rs:1187-1200`, `:1212-1222`, summarised at `:2655-2662`).
- While disconnected, `apply_command_to_state` parks observer frames and drops everything else (`relay.rs:1229-1237`); `retain_failed_command_intent` does the same after a live send failure (`relay.rs:1307-1316`).

#### Reconnect / backoff

Three distinct loops:

| Loop | Location | Ladder | DNS handling |
|---|---|---|---|
| `retry_initial_connect` | `relay.rs:3786-3822` | 1 immediate + 5 delayed (all of `STARTUP_CONNECT_BACKOFFS`) | none — DNS is transient `Io`, consumes a rung |
| `try_autonomous_reconnect` | `relay.rs:2893-3018` | 5 attempts, first 4 delays used (sleep skipped on the last, `relay.rs:3000`) | flat 2 s (`DNS_RETRY_INTERVAL`, `relay.rs:98`), capped at 10 retries (`relay.rs:2914`, `:3006-3008`) |
| `wait_for_reconnect` | `relay.rs:3022-3151` | 1/2/4/8/16/32 s then flat 60 s (`relay.rs:3055-3062`, `:3128-3132`), resumes from `state.backoff_step` (`relay.rs:3063`) | flat 2 s, **unbounded** (`relay.rs:3111-3117`) |

Backoff sleeps are deadline-based (`sleep_until`) and processed inside a
`select!` so `Shutdown` is honoured mid-sleep without resetting the timer — the
stated reason is that 3-second typing refreshes would otherwise collapse the
backoff into a reconnect storm (`relay.rs:3122-3127`, `:2993-2999`).
`backoff_step` is persisted before sleeping (`relay.rs:3125`) and reset to 0 by
the stability block after `STABLE_CONNECTION_SECS` = 60 s
(`relay.rs:2026-2035`).

A handshake-buffer drop signal after reconnect falls through to the backoff
sleep rather than tight-looping (`relay.rs:2932-2940`, `:3084-3090`).

#### Resubscribe after reconnect

`resubscribe_after_reconnect` (`relay.rs:2489-2607`):
- On a fresh connection, `rate_limited_pending` and `resubscribe_retry` are cleared but the gate is deliberately preserved, because the relay's quota is keyed by community+pubkey and survives socket replacement (`relay.rs:2496-2503`, rationale `:2481-2485`).
- Channels are replayed with 125 ms pacing via `pacing_sleep` (`relay.rs:2543-2547`), which defers non-`Shutdown` commands in arrival order (`relay.rs:3369-3392`).
- A gate re-armed mid-burst parks the remaining channels instead of sending (`relay.rs:2518-2526`).
- A failed **channel** REQ is parked in `resubscribe_retry` and does not abort the reconnect (`relay.rs:2548-2557`); a failed **membership** or **observer-control** REQ, or a failed deferred command, returns `RetryConnection` (`relay.rs:2576-2578`, `:2597-2601`).
- Deferred commands are then executed via `drain_commands` (`relay.rs:2600`), which preserves FIFO order across deferred and newly-arrived commands and paces only actual live sends (`relay.rs:2793-2865`).

#### Membership-notification rules

Both membership kinds require an `h` tag; without one the event is dropped
(`relay.rs:2078-2085`, extractor at `relay.rs:3418-3427`). Successful enqueue
advances `membership_last_seen` with `max` (`relay.rs:2117-2119`).
`SetStartupWatermark` seeds `membership_last_seen` if unset
(`relay.rs:1263-1266`, `:1523-1525`).

#### Archived-channel exclusion

`merge_discovered_channels` (`relay.rs:171-232`) drops any channel whose
kind:39000 metadata carries `["archived","true"]` — even when the agent is still
a listed member. The stated purpose is to stop a `CLOSED restricted` reconnect
loop for channels reaped while the agent was offline (`relay.rs:163-170`).
`archived=false` is kept (test at `relay.rs:4140-4149`). A channel with no
metadata becomes `("unknown","unknown")` (`relay.rs:219-221`) so security
consumers can fail closed.

#### Engram fetch ordering and caching

`decode_core_body` (`engram_fetch.rs:110-165`) implements a three-way
fail-closed rule:

| Relay result | Outcome | Rendered section |
|---|---|---|
| empty array | `Ok(None)` — confirmed absence | `[Agent Memory — core]\n<ONBOARDING_NUDGE>` (`engram_fetch.rs:47`) |
| ≥1 event decrypts, head is `Body::Core` | `Ok(Some(profile))` | `[Agent Memory — core]\n<profile>` (`engram_fetch.rs:46`) |
| ≥1 event decrypts, head is tombstone/other | `Ok(None)` (`engram_fetch.rs:159-161`) | nudge |
| non-empty but nothing decrypts | `Err` (`engram_fetch.rs:139-148`) | **no section** (`engram_fetch.rs:49-56`) |
| transport / non-array / parse failure | `Err` (`engram_fetch.rs:92-99`) | no section |

The `Err` → no-section rule exists so a relay outage is never mistaken for "no
core", which would invite the agent to overwrite real memory
(`engram_fetch.rs:8-11`, `:63-70`). Every candidate is signature-verified
locally before decryption (`engram_fetch.rs:126-128`) — the relay is not
trusted. `select_head` resolves the LWW winner (`engram_fetch.rs:149`) and the
body is matched back by event id (`engram_fetch.rs:152-156`).

Caching lives in the caller, not here: `pool.rs:1382-1414` fetches once per new
channel session under a 3 s timeout and stores the rendered string in
`SessionState::core_sections`; a timeout yields `None` (`pool.rs:1394-1404`).
Invalidation is via `SessionState::invalidate_channel` / `invalidate_all`. No
mid-session refresh.

#### Observer gating

`observer.rs` applies almost no gating: `emit` always assigns a `seq` and always
attempts the broadcast send (`observer.rs:106-135`). The only bounds are the
1,000-entry drop-front replay buffer (`observer.rs:126-129`) and the
1,000-slot broadcast channel (`observer.rs:46`). A poisoned buffer mutex is
logged and skipped, not propagated (`observer.rs:96-100`, `:129-131`).
The handle is only constructed when `config.relay_observer` is true
(`lib.rs:1244-1246`), which is the actual on/off gate.

Relay-side gating of observer control frames is in `lib.rs`, not this module:
signature re-verification, owner-pubkey match, and a ±300 s freshness window
(`OBSERVER_CONTROL_FRESHNESS_SECS`). `relay.rs:2069-2076` forwards the raw
signed event with no checks beyond subscription-id routing.
