## Module: buzz-acp — agent pool & lifecycle (`crates/buzz-acp/src`)
### Aspect: Business Rules

#### Admission: claiming an agent

`try_claim` (`pool.rs:558-575`) runs two passes and never blocks:

1. Affinity pass — if a `channel_id` is supplied, take the first idle agent whose `state.sessions` already contains that channel (`pool.rs:560-568`). Session reuse is the sole affinity criterion; no load, age, or model is considered.
2. Fallback pass — take the first idle slot in index order (`pool.rs:571-574`).

`None` means every agent is checked out. The caller then requeues the batch preserving timestamps, marks the channel complete, and **breaks out of the dispatch loop entirely** rather than trying the next channel (`lib.rs:2909-2916`), logging `pool_exhausted` at `debug`. One exhausted claim therefore stalls dispatch for all other pending channels until the next loop iteration.

Concurrency limits:
- Pool width is `config.agents`, clamped to `1..=32` by clap (`config.rs:292-295`), default `1`. The `agents` vector is allocated once and never resized.
- At most one prompt in flight per channel — enforced by `EventQueue`, not the pool; `dispatch_pending` calls `queue.mark_complete(channel_id)` and `flush_next` skips in-flight channels (`lib.rs:2910-2914`).
- Heartbeats claim with `try_claim(None)` and are simply dropped when nothing is idle (`lib.rs:3545-3552`); `heartbeat_in_flight` is a single bool, so at most one heartbeat turn exists at a time.
- No per-agent queue depth, no fairness or aging, no priority. There is no idle-eviction path anywhere in the module: an idle agent is never reaped for inactivity, only on crash, poisoning, or process shutdown.

#### Return path and the double-return rule

`return_agent` (`pool.rs:577-591`) writes `agents[agent.index] = Some(agent)`. If the slot is already occupied it logs `tracing::error!("BUG: return_agent called for slot {idx} which is already occupied — overwriting")` and overwrites, deliberately choosing to leak the previously-resident `AcpClient` (dropping its child via best-effort `Drop`) over permanently leaking the slot (`pool.rs:579-588`). Nothing prevents the caller from returning an agent whose index is out of range — `self.agents[idx]` would panic.

Every `PromptResult`-based return funnels through `send_prompt_result` (`pool.rs:1235-1252`), which clears the per-turn steer receiver first so the next dispatch's `install_steer_rx` assertion cannot trip (`pool.rs:1223-1233`).

#### Lifecycle state machine — deferred pool start

States: `Listening`, `Waking { attempt }`, `Ready(P)`, `Failed { attempt, retry_at, error }` (`pool_lifecycle.rs:14-25`). Complete transition table:

| From | Call | Guard | To | Return | Site |
|---|---|---|---|---|---|
| `Listening` | `start_wake_if_due(false, _)` | no pending work | `Listening` | `None` | `pool_lifecycle.rs:41-43` |
| `Listening` | `start_wake_if_due(true, now)` | — | `Waking{1}` | `Some(1)` | `:47`, `:55-58` |
| `Failed{a, retry_at}` | `start_wake_if_due(true, now)` | `now >= retry_at` | `Waking{a+1}` | `Some(a+1)` | `:48-51`, `:55-58` |
| `Failed{a, retry_at}` | `start_wake_if_due(true, now)` | `now < retry_at` | unchanged | `None` | `:52` |
| `Failed` | `start_wake_if_due(false, _)` | no pending work | unchanged | `None` | `:41-43` |
| `Waking{a}` | `start_wake_if_due(true, _)` | — | unchanged | `None` | `:52` |
| `Ready(p)` | `start_wake_if_due(true, _)` | — | unchanged | `None` | `:52` |
| `Waking{a}` | `complete_wake(a, Ok(p), _)` | attempt matches | `Ready(p)` | `Ok(())` | `:101`, `:110-111` |
| `Waking{a}` | `complete_wake(a, Err(e), now)` | attempt matches | `Failed{a, now+retry_delay(a), e}` | `Ok(())` | `:112-117` |
| `Waking{a}` | `complete_wake(b, _, _)` | `b != a` | unchanged | `Err("wake result attempt did not match Waking attempt")` | `:105-106` |
| `Listening`/`Ready`/`Failed` | `complete_wake(..)` | — | unchanged | `Err("wake completed while lifecycle was not Waking")` | `:107` |
| `Waking{a}` | `cancel_wake(a, e, now)` | attempt matches | `Failed{a, now+retry_delay(a), e}` | `true` | `:91-93` |
| `Waking{a}` | `cancel_wake(b, ..)` | `b != a` | unchanged | `false` | `:91-93` via `:105` |
| `Ready(p)` | `take_ready()` | — | `Listening` | `Some(p)` | `:60-68` |
| any other | `take_ready()` | — | unchanged | `None` | `:63-66` |

Invariants the machine enforces:
- Exactly one wake task per `Waking` entry: the attempt token is handed out only on the transition, and a second `start_wake_if_due` in the same state returns `None` (`pool_lifecycle.rs:31-35`, test `first_pending_event_starts_exactly_one_wake` at `:139`).
- A stale attempt's success cannot replace a newer pool — the doc states accepting it "could replace a newer pool" (`pool_lifecycle.rs:95-98`), test `stale_attempt_result_cannot_replace_current_wake` (`:244`).
- `Ready` is transient: `take_ready` mem-replaces with `Listening` (`:61`), so after a successful wake the machine no longer records success. Suppression of further wakes depends on the caller's separate `pool_ready` flag (`lib.rs:1322`, gate at `lib.rs:1714`).
- A wake starts only when work is buffered, so an idle harness never spawns subprocesses (`pool_lifecycle.rs:41-43`; caller feeds `queue.has_flushable_work()` at `lib.rs:1715`).

Backoff policy (`pool_lifecycle.rs:122-131`): `retry_delay(attempt) = min(5s * 2^(attempt-1), 300s)`, with the shift exponent clamped to 63 and `checked_shl` falling back to `u64::MAX`, then `saturating_mul`. Sequence: 5, 10, 20, 40, 80, 160, then 300 s forever. Retries are unbounded — there is no attempt ceiling and no circuit breaker on the wake path, unlike the per-slot respawn circuit. Tests pin `retry_delay(7) == 300s` and `retry_delay(u32::MAX) == 300s` (`pool_lifecycle.rs:196-197`).

Caller-side coupling: the retry sleep arm is gated on `lazy_wake_work_pending` because a past `retry_at` would otherwise resolve instantly every iteration — an explicit busy-spin guard (`lib.rs:1709-1713`, `lib.rs:1852-1861`). A panicked wake task is converted into `cancel_wake` on the current attempt (`lib.rs:1862-1878`). A wake result that arrives after the channel send fails is shut down rather than leaked (`lib.rs:1736-1741`).

#### Turn lifecycle inside `run_prompt_task`

Ordered phases (`pool.rs:1265-2212`):

1. Classify source from `batch` presence (`pool.rs:1276-1279`), set observer context, emit `turn_started` (`pool.rs:1295-1304`).
2. Install `TurnCompletionGuard` (`:1309`), start liveness task + `LivenessGuard` (`:1325-1342`), install `ReactionGuard` (`:1351`).
3. NIP-AE core fetch — only when `memory_enabled`, source is a channel, an owner pubkey exists, the channel has no session, and no cached section; bounded by 3 s, all failures inject nothing (`pool.rs:1379-1416`).
4. Canvas fetch — same once-per-new-session lifecycle; DM status resolved from the resolver and **unknown is treated as DM (fail closed, no canvas)** (`pool.rs:1428-1449`); the fetched section is held in `pending_canvas` and committed only after session creation succeeds (`pool.rs:1429`, commit at `:1489-1491`).
5. Session resolve or create (`pool.rs:1466-1560`). Creation applies, in order: combined system prompt → Goose custom system-prompt probe → capability capture → `desired_model` switch → `session_config_captured` observer emit → permission mode (`pool.rs:804-936`).
6. Optional `initial_message` on brand-new channel sessions only (`pool.rs:1580-1712`).
7. Prompt assembly: slash-command extraction (`pool.rs:1761-1769`), context fetch when `context_message_limit > 0`, profile lookup, `format_prompt`.
8. 💬 reaction fired without awaiting (`pool.rs:1802-1809`).
9. The prompt itself, wrapped in `biased` `select!` against the control channel when `control_rx` is `Some` (`pool.rs:1827-1990`).
10. Outcome classification (`pool.rs:1992-2209`), then guards drop.

Rotation rules on success (`pool.rs:1994-2022`): rotate when `stop_reason` is `MaxTokens` or `MaxTurnRequests`, **or** when `max_turns_per_session > 0` and the per-source counter reaches it. Rotation means `state.invalidate(&source)` — the next turn creates a fresh session, re-fetching core and canvas.

Error → session-state policy:

| Condition | Session action | Outcome | Site |
|---|---|---|---|
| `session/new` returns `AgentExited` | `invalidate_all` | `AgentExited` | `pool.rs:1494-1505` |
| `session/new` other error | none | `Error(e)` | `pool.rs:1507-1518` |
| `initial_message` idle timeout | `cancel_with_cleanup`, then `invalidate(&source)` | `Timeout(Idle)` | `pool.rs:1640-1681` |
| `initial_message` hard timeout | `invalidate_all` | `Timeout(Hard{recently_active})` | `pool.rs:1683-1700` |
| prompt `AgentExited` | `invalidate_all` | `AgentExited` | `pool.rs:2047-2069` |
| prompt idle timeout | cancel, then `invalidate(&source)` on cancel error | `Timeout(Idle)` | `pool.rs:2070-2152` |
| prompt hard timeout | `invalidate_all` | `Timeout(Hard{..})` | `pool.rs:2155-2180` |
| prompt `AgentError{..}` | **session preserved** — agent caught the problem before mutating state | `Error(e)` | `pool.rs:2182-2188` |
| any other prompt error | `invalidate(&source)` | `Error(e)` | `pool.rs:2186-2188` |

`recently_active` is `silence < RECENT_ACTIVITY_WINDOW` (60 s) at hard-cap firing (`pool.rs:1683`, `pool.rs:2154`), and is the flag the queue uses to decide requeue-vs-dead-letter for a hard-capped batch.

#### Cancellation semantics

Five control signals with distinct batch fates (`pool.rs:263-289`, mapping at `pool.rs:2986-3004`):

| Signal | Cancels turn | Batch fate | Session |
|---|---|---|---|
| `Cancel` | yes | dropped | invalidated |
| `Interrupt` | yes | requeued, `CancelReason::Interrupt` | invalidated |
| `Steer` | yes | requeued, `CancelReason::Steer` | invalidated |
| `Rotate` | yes | dropped | invalidated |
| `SwitchModel(id)` | yes | requeued, `CancelReason::Interrupt` | invalidated after `desired_model` is set |

`requeue_cancelled_batch` returns `None` for `Cancel`/`Rotate` before consulting dedup mode (`pool.rs:2992-2995`); for the others it defers to `requeue_batch_if_queue`, so in `DedupMode::Drop` even a steer loses the batch (`pool.rs:2973-2979`).

Race 1 — the signal arrives after the turn already completed. `has_in_flight_prompt()` is false (`pool.rs:1861`), so no cancel is sent; instead the harness synthesizes `PromptOutcome::Ok(StopReason::EndTurn)` with **no batch requeue** (`pool.rs:1931-1988`). `apply_completed_before_control_signal` (`pool.rs:242-257`) still applies the load-bearing part: `Rotate` and `SwitchModel` invalidate the session; `Cancel`/`Interrupt`/`Steer` leave it intact. The code notes this branch is unreachable during the pre-prompt phase because `biased` polls the prompt arm first and it sets `last_prompt_id` before its first yield (`pool.rs:1935-1943`). The comment "MUST send a PromptResult or the main loop deadlocks" (`pool.rs:1945`) states the invariant: the main loop's `task_map`/slot accounting only clears on a `PromptResult` or a `JoinError`.

Cancel drain deadline: control-signal cancels use `cancel_with_cleanup_grace(session_id, CONTROL_CANCEL_GRACE)` — a fixed 5 s (`pool.rs:788-793`, call at `pool.rs:1863-1866`). Failure is classified once by `classify_control_cancel_failure` (`pool.rs:3029-3056`):

| Error | Outcome | `invalidate_all` |
|---|---|---|
| `AgentExited` | `AgentExited` | yes |
| `IdleTimeout(_)` | `Timeout(Idle)` | no |
| `CancelDrainTimeout(g)` | `CancelDrainTimeout(g)` | no |
| `HardTimeout{..}` | `CancelDrainTimeout(CONTROL_CANCEL_GRACE)` — deliberately **not** `Timeout(Hard)` so it can never claim the configured cap or trigger hard-cap dead-lettering | no |
| anything else | `Error(other)` | no |

Non-cancelling steer (Goose-native): `send_steer` finds the `TaskMeta` whose `channel_id` matches, requiring an installed `steer_tx`, and `try_send`s into a capacity-1 channel (`pool.rs:646-662`). `Full`/`Closed` become `SteerError::Transport`; a missing task becomes `SteerError::PromptCompleted`. Every failure path is documented to fall back to the universal `ControlSignal::Steer` cancel+merge route. Ack semantics are locked (`pool.rs:375-392`): `Success` → drop the withheld event; `Err(_)` → release it and fire the fallback; `PromptCompletedNeutral` → release it and do **not** fire the fallback.

#### What happens to in-flight turns when a child dies

The pool never observes the child directly — death surfaces as `AcpError::AgentExited` from a read, which becomes `PromptOutcome::AgentExited` and returns the (now-useless) `OwnedAgent` through `result_tx`. `handle_prompt_result` then removes every `task_map` entry for that agent index (`lib.rs:3047-3051`), requeues the batch before `mark_complete` so retry backoff survives (`lib.rs:3061-3069`), and routes `AgentExited | Timeout(_)` and `CancelDrainTimeout(_)` to `spawn_respawn_task` (`lib.rs:3239-3283`, `:3292-3323`). `Cancelled` and non-transport `Error` return the agent to the pool; transport-class errors (`Io`, `WriteTimeout`, `Timeout`, `Protocol`) respawn (`lib.rs:3347-3396`).

If the task panics instead of returning, `recover_panicked_agent` (`lib.rs:3402-3495`) reconstructs state purely from `TaskMeta`: requeue `recoverable_batch` (only populated in `DedupMode::Queue`, `lib.rs:2919-2922`), `mark_complete` the channel or clear `heartbeat_in_flight`, emit `agent_panic`, count the panic against the circuit breaker, and respawn in the background. A panic in `DedupMode::Drop` therefore silently loses the message.

Respawn/backoff (all in `lib.rs`, outside this module's files): `SlotCircuit` (`lib.rs:1027-1035`) with `CIRCUIT_BREAKER_THRESHOLD = 3` crashes per `CIRCUIT_BREAKER_WINDOW = 60s`, `CIRCUIT_BREAKER_COOLDOWN = 300s`, respawn delay `RESPAWN_BASE_DELAY = 1s` doubling to `RESPAWN_MAX_DELAY = 30s` (`lib.rs:1008-1016`). Circuit-open slots stay empty until the maintenance loop's refill pass (`lib.rs:1748-1768`). When `live_count() == 0` and no respawn is in flight, the harness exits (`lib.rs:3275-3278`, `:3315-3318`, `:3378-3382`, `:3529-3531`).

#### Health checks

There is no active health probe. The only liveness mechanism is outbound and observational: `run_turn_liveness` (`pool.rs:3162-3204`) emits a `turn_liveness` observer frame every `turn_liveness_interval` while a turn runs, skipping the immediate first tick so the first ping lands one interval after `turn_started` (`pool.rs:3184-3187`). A zero interval parks forever without emitting (`pool.rs:3172-3174`), as does a missing observer handle (`pool.rs:3169-3171`). Failure detection is entirely reactive: idle timeout, hard cap, EOF-on-read, or a task panic.
