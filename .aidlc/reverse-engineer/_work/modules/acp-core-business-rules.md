## Module: buzz-acp — harness core & orchestration (`crates/buzz-acp/src`)
### Aspect: Business Rules

#### Inbound event pipeline order (fixed, per event)

The main loop's relay branch (`lib.rs:1855-2300`) applies gates in a strict order. Order is load-bearing and commented as such at several sites.

| # | Gate | Line | Behaviour on reject |
|---|---|---|---|
| 1 | membership notification intercept (44100/44101) | `lib.rs:1861-1949` | `continue` — never reaches the agent |
| 2 | `ignore_self` (own pubkey) | `lib.rs:2027-2030` | drop |
| 3 | `!shutdown` from owner | `lib.rs:2033-2059` | consume |
| 4 | `!cancel` from owner | `lib.rs:2064-2092` | consume |
| 5 | `!rotate` from owner | `lib.rs:2102-2133` | consume |
| 6 | inbound author gate | `lib.rs:2135-2172` | drop (debug log only) |
| 7 | subscription rule match | `lib.rs:2172-2181` | drop |
| 8 | `queue.push` + dedup | `lib.rs:2196-2201` | `DedupMode::Drop` returns `false`, no 👀 |
| 9 | mid-turn mode gate | `lib.rs:2215-2258` | may steer/interrupt the running turn |

Owner control commands are deliberately checked **before** the author gate (comment `lib.rs:2133-2136`) so `--respond-to nobody` cannot lock the owner out.

#### Author gate (`author_allowed`, `lib.rs:235-256`)

| Mode | Non-DM channel | DM channel |
|---|---|---|
| `Anyone` | accept all | **owner ∪ verified siblings only** |
| `Nobody` | reject all | reject all |
| `OwnerOnly` | owner ∪ siblings | owner ∪ siblings |
| `Allowlist` | allowlist ∪ owner ∪ siblings | **owner ∪ siblings only — allowlist ignored** |

The DM branch short-circuits first (`lib.rs:242-247`). Rationale documented at `lib.rs:220-233`: clients auto-`p`-tag every DM participant, so in a DM any participant's message reads as a mention, and since the agent can be asked to DM a third party, `anyone`/`allowlist` would become a transitive access grant.

`is_dm_channel` (`lib.rs:273-286`) **fails closed** — an unresolvable channel type is treated as a DM and the misclassification is deliberately *not* cached so a later event retries (documented `lib.rs:261-272`, enforced by `ChannelInfoResolver::resolve`).

`is_owner_or_sibling` (`lib.rs:192-215`) also fails closed: no configured owner ⇒ `false` (`lib.rs:200`). Sibling proof requires a locally verified NIP-OA Schnorr signature over the `["auth", owner, conditions, sig]` tag from the author's kind:0 profile — `check_sibling_via_profile` (`lib.rs:291-362`) re-verifies rather than trusting the relay (comment `lib.rs:322-323`), with a 2,000 ms timeout that fails closed (`lib.rs:310-315`).

Sibling verdicts cache for the process lifetime; at 256 entries the whole map is cleared rather than evicted LRU (`lib.rs:183-186`), so a hot sibling can be forced back through a REST round-trip by cache churn.

#### Owner resolution priority

`resolve_agent_owner` (`lib.rs:123-148`): `BUZZ_AUTH_TAG` NIP-OA attestation verified against the agent's own pubkey (`lib.rs:125-137`), else `--agent-owner` / `BUZZ_ACP_AGENT_OWNER` (`lib.rs:147`). A failed attestation verification logs a warning and silently falls through to the unverified flag (`lib.rs:138-141`).

Owner is resolved exactly once at startup. `OwnerCache.pubkey` is immutable (`lib.rs:161-163` returns `&str`, no setter). Consequences: with `respond-to=owner-only` and no owner, every event is dropped for the process lifetime — warned at `lib.rs:1379-1384` but never retried. `--relay-observer` is silently downgraded to disabled when no owner resolved (`lib.rs:1421-1425`).

#### Turn lifecycle

Dispatch (`dispatch_pending`, `lib.rs:2889-3000`) loops until the queue or the pool is exhausted:

1. `queue.flush_next()` drains one channel's whole backlog into one `FlushBatch` (`lib.rs:2892`).
2. Typing scope is derived from the **last** event in the batch (`lib.rs:2897-2900`).
3. `pool.try_claim(Some(channel_id))` prefers session affinity (`lib.rs:2902`).
4. On pool exhaustion: `requeue_preserve_timestamps` then `mark_complete`, then `break` (`lib.rs:2906-2910`) — timestamps preserved so retry ordering is not reset.
5. A recovery copy of the batch is retained **only** under `DedupMode::Queue` (`lib.rs:2915-2918`); `DedupMode::Drop` stores `None`, so a panic loses the batch.
6. A capacity-1 steer channel is installed on **every** prompt task regardless of agent capability (`lib.rs:2933-2934`, comment `lib.rs:2929-2932`) — "try-and-tolerate".
7. `turn_id = Uuid::new_v4()` (`lib.rs:2963`), task spawned into `pool.join_set`, `TaskMeta` recorded keyed by `abort_handle.id()` (`lib.rs:2977-2987`).

Exactly one prompt per channel is in flight; concurrency across channels is bounded by `config.agents` (1–32, enforced in `config.rs:292-296`).

#### Mid-turn arrival: `mode_gate_signal` (`lib.rs:2741-2757`)

| `--multiple-event-handling` | Signal | Author re-check |
|---|---|---|
| `queue` | `None` — event waits | — |
| `steer` | `ControlSignal::Steer` | none (gate already ran) |
| `interrupt` | `ControlSignal::Interrupt` | none |
| `owner-interrupt` | `ControlSignal::Interrupt` | owner only (`lib.rs:2751-2754`) |

`Steer` takes a two-stage path. `try_native_steer` (`lib.rs:2803-2887`) first attempts the non-cancelling ACP steer:

- Builds the body from `queue::native_steer_framing()` + `queue::format_event_block` so native and fallback framing cannot drift (comment `lib.rs:2812-2825`).
- Deliberately passes `None` for `channel_info` and `profile_lookup` — a steer is a *delta*, the agent already saw that context (`lib.rs:2820-2825`).
- On `Ok(())` it withholds the queued event **synchronously before** spawning the ack watcher (`lib.rs:2836-2839`) to close the `mark_complete` → stray-`flush_next` re-delivery race.
- On `Err` it returns `false` and the caller falls through to cancel+merge (`lib.rs:2249-2256`).

The ack arm (`lib.rs:2417-2500`) resolves a three-way decision `(release_withheld, drop_withheld, signal_fallback)`:

| Ack | release | drop | fallback |
|---|---|---|---|
| `Success` | no | yes | no |
| `AgentError{code == -32601}` | yes | no | **yes** |
| `AgentError{other}` | yes | no | no |
| other `SteerError` (transport / `ExpectedRunIdMissing`) | yes | no | **yes** |
| `PromptCompletedNeutral` | yes | no | no |
| watcher `Err` (oneshot dropped) | yes | no | no |

On `Success` the in-flight deadline is extended by `max_turn_duration_secs` (`lib.rs:2487-2489`) — a steered turn gets a fresh full budget, so repeated steering can extend a single turn indefinitely past the configured cap.

#### Outcome → retry policy (`handle_prompt_result`, `lib.rs:3034-3399`)

Requeue happens **before** `mark_complete` because `requeue()` sets `retry_after` that `mark_complete()` reads to decide whether to preserve `retry_counts` (comment `lib.rs:3048-3051`). Inverting the order silently resets every retry to attempt 1.

| Outcome | Batch fate | Agent fate |
|---|---|---|
| `Ok` | consumed | returned to pool (`lib.rs:3241`) |
| `Cancelled` / `CancelDrainTimeout` | `requeue_as_cancelled(reason)`, default `Steer` if unset (`lib.rs:3071-3072`) | `Cancelled` → pool (`lib.rs:3335`); `CancelDrainTimeout` → **respawn** (`lib.rs:3290-3305`) |
| `Timeout(Hard{recently_active:false})` | dead-lettered immediately + user notice (`lib.rs:3082-3096`) | respawn |
| `Timeout(Hard{recently_active:true})` | `requeue()`; notice only if budget exhausted (`lib.rs:3097-3117`) | respawn |
| `Timeout(Idle)` | `requeue()` (`lib.rs:3135`) | respawn |
| `AgentExited` | `requeue()` | respawn |
| `Error(e)` where `is_auth_error(e)` | dead-lettered immediately, no retry (`lib.rs:3118-3133`) | pool (pipe intact) |
| `Error(transport-class)` — `Io`/`WriteTimeout`/`Timeout`/`Protocol` | `requeue()` | respawn (`lib.rs:3341-3346`) |
| `Error(application-class)` | `requeue()` | pool (`lib.rs:3373-3383`) |
| any outcome, channel in `removed_channels` | dropped (`lib.rs:3149-3155`) | unchanged |

`is_auth_error` (`lib.rs:3003-3011`) matches on **substrings of vendor error text**: `"Re-authenticate"` or `"API Error: 401"`. Rationale for high-precision matching is documented at `lib.rs:2989-3002`. This is a string-matching contract against upstream CLI messages with no version pin.

Sessions for channels the agent lost membership to are stripped when the agent returns (`lib.rs:3166-3169`), covering the gap that `invalidate_channel_sessions` only reaches *idle* agents.

#### Circuit breaker (`SlotCircuit`, `lib.rs:1048-1134`)

- 3 crashes inside a 60 s window opens the circuit for 300 s (`lib.rs:1082-1085`).
- Backoff before threshold: `1s × 2^(recent-1)` capped at 30 s, ±20 % jitter derived from `SystemTime` subsec nanos (`lib.rs:1088-1097`) — not a PRNG, so co-started slots crashing in the same nanosecond window get correlated jitter.
- Half-open probe pre-seeds `crash_times` to `THRESHOLD-1` so a single probe failure re-opens immediately (`lib.rs:1064-1074`, mirrored in `can_refill` `lib.rs:1113-1131`) — "prove stability for one full window".
- `mark_spawn_failed` (`lib.rs:1103-1105`) uses a fresh `Instant::now()` so spawn latency cannot shorten the cooldown.
- `record_crash` is documented as the single canonical crash→respawn path (`lib.rs:1050-1053`), called from `spawn_respawn_task` (`lib.rs:3646`), `recover_panicked_agent` (`lib.rs:3465`), and slot refill via `can_refill` (`lib.rs:1762`).

Exit condition: `pool.live_count() == 0 && !any_respawn_in_flight(&crash_history)` (`lib.rs:2373`, `3277`, `3306`, `3355`, `3527`). The `any_respawn_in_flight` conjunct (`lib.rs:1136-1138`) prevents a premature "all agents dead" exit while a respawn is pending.

#### Heartbeat rules (`lib.rs:2265-2298`, `dispatch_heartbeat` `lib.rs:3537-3586`)

Priority order, all enforced in the tick arm:

1. skipped if `!pool_ready` (`lib.rs:2270`);
2. queued work wins — the tick becomes a `dispatch_pending` call instead (`lib.rs:2272-2278`);
3. requires an idle agent (`lib.rs:2279`), else dropped with `heartbeat_skipped_busy` — no queuing;
4. at most one globally, guarded by `heartbeat_in_flight` (`lib.rs:3541-3543`).

Interval must be `0` or ≥ 10 s (README; validated in `config.rs`). Default prompt (`default_heartbeat_prompt`, `lib.rs:3609-3633`) instructs `buzz feed get --types needs_action` then `--types mentions`, and explicitly tells the agent not to run `buzz channels list` or `buzz messages search`. The README (`README.md § Heartbeat Semantics`) still documents the older `get_feed_actions()` / `get_feed_mentions()` call names.

#### Membership notification dedup (two layers, `lib.rs:1865-1889`)

1. Exact event-ID rejection across two generations (`seen_membership_current` ∪ `seen_membership_previous`), rotated at 1,000 entries so there is no amnesia window (`lib.rs:1877-1881`).
2. Timestamp watermark per channel using strict `<` (`lib.rs:1882-1888`). Strict is deliberate (`lib.rs:1653-1658`, `1867-1876`): `<=` would suppress a legitimate live add→remove pair in the same second, leaving the harness with wrong membership state.

Known accepted race, documented at `lib.rs:1670-1680`: a batch in flight when a channel is removed **and re-added** before it returns may be requeued. The comment states the fix would need per-channel epoch tracking on `TaskMeta` and `PromptResult` and judges it not worth the complexity.

#### Observer frame pacing and trimming

- Pacer (`lib.rs:372-409`) enforces 167 ms spacing plus a 90-frames-per-rolling-minute ceiling, with **no initial burst** — even the first snapshot frame waits a slot (`lib.rs:375-376`).
- Snapshot/live overlap is deduped by the snapshot's high-water `seq` (`lib.rs:443`, `lib.rs:470-473`); subscribe happens before snapshot so no event can fall between them (`lib.rs:420-423`).
- Oversized frames are trimmed rather than dropped. `fit_observer_event_to_budget` (`lib.rs:659-687`) repeatedly elides the largest *strictly shrinkable* string leaf and reserializes, falling back to a stub payload when no leaf can shrink. Termination is argued in the doc comment (`lib.rs:645-651`): monotone decrease bounded below by the stub. `leaf_shrinks` (`lib.rs:739-747`) includes a marker-pushback guard so a leaf near its retained floor is never touched.
- Elision boundaries snap to UTF-8 char boundaries (`lib.rs:770-789`).

#### Owner control-frame acceptance (`lib.rs:837-870`)

Three defence-in-depth checks before decryption, each rejecting outright:

1. `buzz_core::verify_event` signature re-verification (`lib.rs:845-848`);
2. sender pubkey must equal the resolved owner hex (`lib.rs:851-858`);
3. `created_at` within ±300 s (`lib.rs:861-869`).
