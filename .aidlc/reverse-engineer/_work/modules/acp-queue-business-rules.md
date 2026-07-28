## Module: buzz-acp — work queue, filtering & usage accounting (`crates/buzz-acp/src`)
### Aspect: Business Rules

#### 1. Admission (`push`, `queue.rs:230-252`)

| Rule | Evidence |
|---|---|
| In `DedupMode::Drop`, an event whose channel is in `in_flight_channels` is discarded and `push` returns `false`; only a `debug!` is logged | `queue.rs:231-241` |
| In `DedupMode::Queue`, every event is accepted regardless of in-flight state | `queue.rs:231` (guard is `matches!(Drop)`) |
| At depth ≥ 500 the **oldest** event is `pop_front`-ed to make room, with a `warn!` | `queue.rs:242-249` |
| Events for other channels always queue, even in `Drop` mode | `queue.rs:232` scopes the check to `event.channel_id`; test `test_drop_mode_queues_other_channels` (`queue.rs:2522`) |
| The 👀 "seen" reaction is only sent when `push` returns `true` | `lib.rs:2209-2217` |

#### 2. In-flight exclusion and expiry

| Rule | Evidence |
|---|---|
| At most one batch per channel is in flight; multiple channels run concurrently | `queue.rs:294-298` filter excludes `in_flight_channels`; test `test_multiple_channels_in_flight_simultaneously` (`queue.rs:2541`) |
| `flush_next` and `has_flushable_work` both auto-expire any channel where `now >= deadline`, logging `ERROR "BUG: in-flight channel expired without mark_complete"` with `lost_events` | `queue.rs:263-287` and `:558-590` (duplicated block) |
| Expiry releases `in_flight_channels` + `in_flight_deadlines` but the dispatched batch is **not** recovered — it is counted as orphaned | `queue.rs:271-281`; comment `queue.rs:275-277` |
| Deadline = `max_turn_duration + 100 s`, so a turn hitting the hard cap returns via `mark_complete` before the backstop fires | `queue.rs:39`, `:197-201`; invariant stated `queue.rs:167-169`; test `default_in_flight_deadline_exceeds_default_max_turn_duration` (`queue.rs:4530`) |
| `extend_in_flight_deadline` is monotonic (`max(current, now+budget)`) and a **no-op** when the channel is not in flight | `queue.rs:211-227`; tests `extend_in_flight_deadline_is_monotonic` (`:4574`), `extend_in_flight_deadline_noop_after_mark_complete` (`:4587`) |
| `compact_expired_state` preserves an extended deadline | test `compact_expired_state_preserves_extended_in_flight_deadline` (`queue.rs:4606`) |

#### 3. Flush selection and batching (`flush_next`, `queue.rs:260-380`)

| Rule | Evidence |
|---|---|
| Candidate = non-empty queue AND not in flight AND (`retry_after` absent OR ≤ now) | `queue.rs:291-300` |
| Winner = channel whose **head** event has the oldest `received_at` (cross-channel FIFO fairness) | `queue.rs:299`; tests `test_fifo_fairness_picks_oldest_channel` (`:1810`), `test_flush_next_picks_oldest_non_throttled` (`:2615`) |
| Ties are broken by `HashMap` iteration order — nondeterministic | `queue.rs:293-300` iterates `self.queues` with no secondary key |
| Drains `min(50, len)` events; the remainder stays queued | `queue.rs:336-345` (`MAX_BATCH_EVENTS`, `queue.rs:27`) |
| The drained batch is re-sorted ascending by `event.created_at` (stable), because relay replay arrives newest-first | `queue.rs:346-350`; test `test_flush_orders_replayed_events_chronologically` (`:1782`) |
| Empty queue entries are removed from the map after drain | `queue.rs:352-355` |
| Any stored `cancelled_batches[channel]` is merged into `FlushBatch::cancelled_events` on the same flush | `queue.rs:362-366`; test `test_requeue_as_cancelled_merges_in_flush_next` (`:3674`) |
| `cancel_reasons[channel]` is **always removed** on flush; the value is only propagated when `cancelled_events` is non-empty, otherwise `cancel_reason` is `None` | `queue.rs:367-372` |
| **Fallback:** if no channel has ready queued events but some channel has `cancelled_batches`, that channel is flushed with the cancelled events moved into the regular `events` slot and `cancelled_events` empty (plain re-dispatch, no merge framing) | `queue.rs:305-333`; test `test_requeue_as_cancelled_no_new_events_fallback` (`:3760`) |
| The fallback path checks only `in_flight_channels` — it **ignores `retry_after`**, so a throttled channel's cancelled batch can flush during its backoff window | `queue.rs:308-312` (no `retry_after` predicate); same omission in `has_flushable_work` (`queue.rs:586-590`) |
| The fallback picks with `keys().find(...)` — nondeterministic and not FIFO-ordered across cancelled channels | `queue.rs:308-312` |

#### 4. Completion vs requeue ordering (the load-bearing invariant)

`mark_complete` decides whether to keep the retry counter by **reading `retry_after`** (`queue.rs:397-409`):

- `Some(deadline)` with `deadline > now` → channel was just requeued → `retry_counts` is left intact so backoff continues.
- `Some(_)` expired → clear both `retry_after` and `retry_counts`.
- `None` → clear `retry_counts`.

Only `requeue` writes `retry_after` (`queue.rs:496`). Therefore **`requeue()` must be called before `mark_complete()`** or every retry restarts at attempt 1, silently defeating both exponential backoff and dead-lettering. `handle_prompt_result` enforces this ordering explicitly, with the rationale in a comment at `lib.rs:3061-3065`; the batch-fate block runs at `lib.rs:3067-3163` and `mark_complete` only at `lib.rs:3171`. The same ordering holds at the pool-exhausted site (`requeue_preserve_timestamps` `lib.rs:2912` then `mark_complete` `lib.rs:2913`) and in panic recovery (`requeue` `lib.rs:3428` then `mark_complete` `lib.rs:3440`). Neither `requeue` nor `requeue_preserve_timestamps` clears `in_flight_channels` themselves — documented at `queue.rs:426-427` and `:506-507`. Nothing in the type system or a debug assertion enforces the order; it is carried by comments and the tests `test_mark_complete_enables_flush` (`queue.rs:1736`) / `test_retry_throttle_blocks_requeue_channel` (`queue.rs:2854`).

#### 5. Retry budget, backoff, dead-lettering (`requeue`, `queue.rs:429-498`)

| Rule | Evidence |
|---|---|
| Attempt counter is per **channel**, not per batch | `queue.rs:431-435` |
| `attempt > MAX_RETRIES` (10) → dead-letter: `ERROR` log, `retry_counts` **and** `retry_after` cleared, batch returned to caller | `queue.rs:437-451`; test `test_requeue_dead_letters_after_max_retries` (`:2824-2846`) |
| `retry_after` is cleared on dead-letter so fresh traffic is not throttled by the poison batch's backoff | `queue.rs:447-449` |
| Backoff = `5 s × 2^min(attempt-1, 6)` capped at 300 s → 5, 10, 20, 40, 80, 160, then 300 for attempts 7-10 (≈25 min total budget) | `queue.rs:454-455` (`BASE_RETRY_DELAY_SECS` `:33`, `MAX_RETRY_DELAY_SECS` `:36`) |
| Jitter is ±20 % derived from `SystemTime::now().subsec_nanos()` — not a CSPRNG | `queue.rs:457-464` |
| Events are pushed to the **front** in reverse order, preserving original order and original `received_at` (fairness position) | `queue.rs:476-484`; doc `queue.rs:416-421` |
| Requeue overflow past 500 trims from the **back** (newest) | `queue.rs:486-495`; tests `test_requeue_preserve_timestamps_enforces_cap` (`:2715`), `test_requeue_preserve_timestamps_overflow_keeps_requeued_events` (`:2751`) |
| `requeue_preserve_timestamps` is the no-agent-available path: same front-insert, **no** `retry_after`, **no** attempt increment, so it can loop indefinitely without ever dead-lettering | `queue.rs:508-529`; test `test_requeue_preserve_timestamps_no_retry_after` (`:2699`) |

#### 6. Which failures retry, which dead-letter (decided in `lib.rs`, executed by the queue)

| Outcome | Queue action | Evidence |
|---|---|---|
| Channel in `removed_channels` | batch dropped, no queue call | `lib.rs:3157-3163` |
| `Cancelled` or `CancelDrainTimeout` | `requeue_as_cancelled(batch, reason)` — no retry accounting at all | `lib.rs:3072-3090` |
| `Timeout(Hard { recently_active: false })` | dead-lettered **immediately**, no `requeue` call, user notice posted | `lib.rs:3091-3109` |
| `Timeout(Hard { recently_active: true })` | `requeue(batch)`; only dead-letters when `requeue` returns `Some` (retry budget exhausted per §5) | `lib.rs:3110-3129` |
| `Error(e)` where `is_auth_error(e)` | dead-lettered immediately, no `requeue`, re-auth notice | `lib.rs:3130-3144` |
| everything else (`Idle` timeout, `AgentExited`, other `Error`) | `requeue(batch)` with generic notice on dead-letter | `lib.rs:3145-3156` |

"Recently active" means agent activity within `RECENT_ACTIVITY_WINDOW = 60 s` before the hard cap (`pool.rs:44-45`).

#### 7. Requeue-vs-drop per cancel reason

`requeue_as_cancelled` never drops (`queue.rs:542-548`); the drop decision happens upstream:

| `ControlSignal` | `CancelReason` | Batch fate | Evidence |
|---|---|---|---|
| `Steer` | `Steer` | requeued in `Queue` mode | `pool.rs:2988` |
| `Interrupt`, `SwitchModel(_)` | `Interrupt` | requeued in `Queue` mode | `pool.rs:2989` |
| `Cancel`, `Rotate` | — | **dropped outright** | `pool.rs:2990-2991` |

`requeue_batch_if_queue` returns `None` for `DedupMode::Drop` on **every** failure path including a steer (`pool.rs:2971-2977`), so in `Drop` mode a cancel loses the batch. The same asymmetry applies to panic recovery: `recoverable_batch` is populated only under `DedupMode::Queue` (`lib.rs:2196-2199`), so a task panic in `Drop` mode drops the batch. `DedupMode` defaults to `queue` on the CLI (`config.rs:344`), which is what keeps these paths lossless in practice.

Double-cancel accumulates: `requeue_as_cancelled` `extend`s both the prior `cancelled_events` and the new `events` into the same vec, and the **latest reason wins** (`queue.rs:543-547`). Tests `test_double_cancel_latest_reason_wins` (`:3740`), `test_double_cancel_preserves_all_events` (`:3825`).

#### 8. Goose-native steer withhold (side table)

| Rule | Evidence |
|---|---|
| `mark_native_steer_pending` moves the event **out of `queues`** into `withheld_native_steer`, making it invisible to `flush_next` / `has_flushable_work` / `drain_channel` without touching the drain path | `queue.rs:673-691`; rationale `queue.rs:157-165`, `:649-660` |
| Returns `false` (race-safe no-op) if the event id is absent — caller logs a possible-duplicate warning | `queue.rs:674-680`; caller `lib.rs:2847-2865` |
| Must be called synchronously after `send_steer` returns `Ok` and **before** the ack watcher is spawned, closing the `mark_complete` → `flush_next` race | doc `queue.rs:667-671`; enforced by ordering at `lib.rs:2841-2872`; also stated `pool.rs:636-640` |
| `SteerAck::Success` → `remove_event` (drop from both stores; the agent already got it) | `queue.rs:738-751`; caller `lib.rs:2512` |
| `SteerAck::Err` / `PromptCompletedNeutral` → `release_native_steer` pushes it back to the queue **front** with original `received_at` | `queue.rs:703-730`; caller `lib.rs:2515` |
| `SteerAck::Success` also extends the in-flight deadline (fresh turn budget) | `lib.rs:2509` |
| In-flight deadline expiry bulk-recovers withheld events in reverse order so FIFO is preserved at the queue front — recover, not log-and-drop, because they were never delivered | `queue.rs:766-789`; tests `test_native_steer_expiry_recovers_withheld` (`:4355`), `test_native_steer_bulk_release_preserves_fifo` (`:4402`) |
| A channel whose only events are withheld is **not** flushable | test `test_native_steer_withhold_only_channel_not_flushable` (`:4278`) |
| Earlier non-withheld events in the same channel still flush during the ack window | test `test_native_steer_earlier_events_flush_during_ack_window` (`:4306`) |

#### 9. Channel teardown and compaction

| Rule | Evidence |
|---|---|
| `drain_channel` removes `queues`, `retry_after`, `retry_counts`, `cancelled_batches`, `cancel_reasons`, `withheld_native_steer` and returns dropped hex event ids | `queue.rs:625-642` |
| It deliberately **preserves** `in_flight_channels` **and** `in_flight_deadlines` — removing deadlines alone would disable auto-expiry and wedge the channel forever | `queue.rs:636-641` comment; test `test_drain_channel_does_not_affect_in_flight` (`:3578`) |
| `compact_expired_state` drops past-deadline `retry_after` entries | `queue.rs:809` |
| `retry_counts` is retained only while the channel has an active throttle **or** queued events **or** an in-flight prompt; the in-flight clause prevents resetting backoff mid-retry | `queue.rs:813-817`; tests `test_compact_cleans_orphaned_retry_counts` (`:3596`), `test_compact_preserves_retry_counts_when_in_flight` (`:3630`), `test_compact_preserves_retry_counts_with_queued_events` (`:3658`) |
| `cancelled_batches`, `cancel_reasons`, `withheld_native_steer`, `in_flight_batch_sizes` are **not** compacted | `queue.rs:807-818` touches only two maps |

#### 10. Subscription rule matching (`filter::match_event`, `filter.rs:368-459`)

Rules are evaluated in order; **first match wins** (`filter.rs:376`, `:451-456`). Per rule, in this exact order:

1. **Channel scope** — `ChannelScope::All(s)` matches only when `s == "all"` literally (`filter.rs:68-70`); `List` compares `id == &channel_id.to_string()` (lowercase-hyphenated UUID string compare, `filter.rs:70`). Non-match → `continue` (`filter.rs:378-380`). Tests `test_channel_scope_all_invalid_string` (`filter.rs:692`), `test_channel_scope_list` (`:702`).
2. **Kinds** — empty vec = wildcard; otherwise `rule.kinds.contains(&(event.kind.as_u16() as u32))` (`filter.rs:383-385`).
3. **`require_mention`** — a tag whose slice is `["p", agent_pubkey_hex, …]` must exist; matched via `tag.as_slice()` to avoid depending on the tag-kind `Display` impl (`filter.rs:390-398`). Comparison is exact-case hex.
4. **`filter` expression** — evalexpr boolean, using the pre-compiled `Arc<Node>` when present (`filter.rs:402-449`).

Fail-closed semantics (`filter.rs:357-366`): **any** filter error or timeout returns `None` for the whole `match_event` call, never falling through to later rules, because falling through would silently widen the subscription. Test `test_filter_error_fails_closed_no_fallthrough` (`filter.rs:733`).

Timeout circuit breaker: `consecutive_timeouts` increments on `FilterError::Timeout` and resets to 0 on any `Ok` (`filter.rs:417-448`). At `MAX_CONSECUTIVE_TIMEOUTS = 5` (`filter.rs:341`) the rule is logged at `ERROR` and `match_event` returns `None` immediately (`filter.rs:405-415`). Because the counter is a shared `Arc<AtomicU32>` (`filter.rs:131-145`), the breaker is global across rule clones. Test `test_consecutive_timeouts_disables_rule` (`filter.rs:766`).

Expression evaluation bounds (`evaluate_filter`, `filter.rs:197-262`): length cap 4096 bytes checked **before** dispatch (`filter.rs:203-208`); a `100 ms` `EVAL_TIMEOUT` applied both to semaphore acquisition and to the blocking eval (`filter.rs:220-232`, `:234-249`); at most 4 concurrent blocking evals, with the permit moved *into* the closure so it is held until the thread actually finishes rather than until the caller times out (`filter.rs:173-183`, `:227`, `:239-241`). Variables exposed: `content`, `author`, `kind`, `channel_id`, `timestamp` plus four hand-registered helpers `str_contains` / `str_starts_with` / `str_ends_with` / `str_len` (`filter.rs:268-323`).

Rule construction rules live in `lib.rs:1439-1474`:

| `SubscribeMode` | kinds | `require_mention` | `prompt_tag` |
|---|---|---|---|
| `Mentions` | `kinds_override` else `[9, 46010, 40007]` (`KIND_STREAM_MESSAGE`/`KIND_WORKFLOW_APPROVAL_REQUESTED`/`KIND_STREAM_REMINDER`, `buzz-core/src/kind.rs:343`, `:442`, `:355`) | `!no_mention_filter` | `"@mention"` |
| `All` | `kinds_override.unwrap_or_default()` — **empty vec when `--kinds` is unset** | `false` | `"all"` |
| `Config` | from `config::load_rules` | per rule | rule `prompt_tag` else `name` |

An empty kinds vec is a wildcard *inside* `match_event` (`filter.rs:383`), but the same `kinds_override` feeds `resolve_channel_filters` (`lib.rs:1476`, `config.rs:1180`) where `kinds: None` produces a relay REQ with no `kinds` — which trips the relay p-gate and returns 403 (`AGENTS.md § Common Gotchas #2`). So `--subscribe all` without `--kinds` matches everything locally and receives nothing.

`load_rules` validation (`config.rs:1060-1129`): ≤100 rules, non-empty unique names, filter ≤4096 bytes, expression compiled eagerly (typos fail startup, not runtime), `ChannelScope::All` must be exactly `"all"`, `consecutive_timeouts` reset to a fresh `Arc`.

#### 11. Prompt framing (`format_prompt`, `queue.rs:1406-1564`)

| Rule | Evidence |
|---|---|
| Empty batch → `ERROR` log and empty `Vec` (no panic) | `queue.rs:1411-1417` |
| Scope is derived from the **last** event in the batch, preventing mixed batches from being mislabeled `thread` | `queue.rs:1407-1417` comment + `batch.events.last()` |
| Section order: `[Base]`, `[System]`, `[Team Instructions]`, `[Agent Memory — core]`, `[Channel Canvas]`, `[Context]`, `[Thread/Conversation Context]`, cancelled section, event section, closing note | `queue.rs:1430-1562`; test `test_format_prompt_ordering_with_full_context` (`:2446`) |
| `has_system_prompt_support` (protocol ≥ 2) suppresses `[Base]`/`[System]`/`[Team Instructions]`/core/canvas from the user message — they ride the system role in `session/new` | `queue.rs:1432-1462`; tests `test_format_prompt_modern_agent_suppresses_base_and_system` (`:2407`), `test_format_prompt_canvas_omitted_for_modern_agent` (`:4481`) |
| Blank/whitespace-only `team_instructions` are skipped | `queue.rs:1444-1450` |
| Each section is returned as its own `String` so the observer size trimmer can elide one section's body while keeping every `[Header]` line | doc `queue.rs:1394-1400`; trimmer `lib.rs:659` |
| Single event, no cancel → `[Buzz event: {prompt_tag}]`; multiple → `[Buzz events — N events]` with `--- Event i (tag) ---` separators | `queue.rs:1528-1552` |
| Merged (cancel) framing selected by `MergeFraming::for_reason`; `None` falls back to the **gentler `Steer`** wording | `queue.rs:1584-1609`; tests `test_format_prompt_steer_framing` (`:1916`), `test_format_prompt_interrupt_framing` (`:1940`), `test_format_prompt_no_reason_defaults_to_steer_framing` (`:1960`) |
| The `Steer` `prior_header` is deliberately `[What you were working on]` and not a transcript claim, because `session/cancel` returns nothing | comment `queue.rs:1589-1592` |
| Native-steer transport reuses the same strings via `native_steer_framing()` (returns `new_header_single` + `closing_note` only) and the same `format_event_block`, so the two transports cannot drift | `queue.rs:1612-1626`; call site `lib.rs:2812-2831` |

Reply-anchor rules (`queue.rs:1465-1497`, helpers `:1182-1231`):

- DM → anchor to the triggering event id, but only if the event has thread tags (`queue.rs:1475-1479`).
- Non-DM human-facing turn in a thread → anchor to the thread **root** (keeps replies flat at layer 1) (`queue.rs:1216-1222`).
- Non-DM human-facing top-level turn → anchor to the triggering event, which becomes the new root, with explicit "do NOT reply into any other (older) thread" wording (`queue.rs:1164-1180`).
- Agent↔agent turn → **no** anchor; deep nesting is intentional (`queue.rs:1213-1215`).
- "Human-facing" = sender is not an agent, OR some non-agent pubkey is `p`-tagged. Identity comes from `PromptProfile::is_agent` (NIP-OA `auth` tag), not raw `p`-tag presence, and an **unclassified** pubkey is treated as human (`queue.rs:1188-1195`, `:1205-1206`). Documented as a UX routing heuristic, explicitly "not a security boundary" (`queue.rs:1005-1007`).
- Tests: `test_anchor_human_in_thread_uses_root` (`:3278`) through `test_anchor_agent_only_p_tags_do_not_flatten` (`:3325`); `test_human_thread_reply_anchors_to_root_not_triggering_or_parent` (`:4008`).

Label resolution: `display_name` first, then `nip05_handle`, both passed through `sanitize_prompt_label` which drops control characters and truncates at 64 chars; empty result → fall through to the raw pubkey (`queue.rs:1028-1065`). Tests `test_sanitize_prompt_label_strips_newlines_and_control_chars` (`:3334`), `test_resolve_prompt_label_skips_whitespace_only_display_name` (`:3229`).

#### 12. Thread-tag parsing (`parse_thread_tags`, `queue.rs:849-900`)

- Only NIP-10 **marker** form is honoured: an `e` tag must have `parts.len() >= 4` and `parts[3] ∈ {"root", "reply"}` (`queue.rs:857-866`). Deprecated positional `["e", id, relay]` tags are silently ignored — documented at `queue.rs:843-848`.
- `p` tags with ≥2 parts accumulate into `mentioned_pubkeys` (`queue.rs:867-869`).
- Resolution: `(root, reply)` → both; `(root, None)` → parent = root; `(None, reply)` → root = reply; `(None, None)` → both `None` (`queue.rs:876-884`). Tests `test_parse_thread_tags_*` (`:2901`-`:2961`).

#### 13. Slash-command pass-through

`slash_command_for_batch` returns `Some` only when the batch has **exactly one** event and **no** cancelled carryover — a merged prompt needs the full context format (`queue.rs:961-967`). `extract_slash_command` (`queue.rs:902-959`) strips leading mention tokens in a loop: NIP-27 `nostr:npub1…` / `nostr:nprofile1…` whole tokens (`:911-915`), then `@` + longest matching known display name (case-insensitive, must end at whitespace or EOS) (`:922-935`), then `@` + a single token of `[A-Za-z0-9._-]` (`:936-943`). A bare `@` aborts and returns `None` (`:946`). The remainder qualifies only if it starts with `/` followed by an ASCII alphanumeric, so `"@Eva see /tmp/foo"` never matches (`queue.rs:956-958`). Tests `test_extract_slash_command_basic` (`:4174`), `..._multi_word_display_name` (`:4199`), `..._rejects_non_commands` (`:4214`), `test_slash_command_for_batch_gating` (`:4231`).

#### 14. Usage delta accounting (`usage.rs`)

Lifecycle: `begin_turn` before `session/prompt` → zero or more `record` → `take` at turn completion (`usage.rs:143-171`; call sites `acp.rs:690`, `:1659`, `:784`).

| Rule | Evidence |
|---|---|
| `begin_turn` sets `in_flight_session` and **clears** any leftover `pending` | `usage.rs:182-185` |
| No baseline for the session → `delta_reliable = false`, all `turn_*` fields `None`, `turn_seq = 1` | `usage.rs:222-226`; test `first_turn_no_prior_delta_unreliable` (`:461`) |
| A token counter **decrease** → unreliable, all `turn_*` `None`, but `turn_seq` still advances | `usage.rs:233-235`; test `counter_decrease_delta_unreliable_no_negatives` (`:483`) |
| A **cost** decrease also nulls *all* turn fields, not just cost | `usage.rs:241-254`; test `cost_decrease_sets_delta_unreliable_and_nulls_all_turn_fields` (`:506`) |
| Cost absent on either side → `turn_cost_usd = None` but token delta stays **reliable** | `usage.rs:245`; test `cost_absent_on_one_side_leaves_tokens_reliable` (`:540`) |
| Unknown `session_id` behaves as a first turn (session-restart case) | `usage.rs:222-226`; test `session_restart_new_session_id_treated_as_first_turn` (`:565`) |
| Delta is always measured from the **last published** baseline, so multiple notifications in one turn cannot compound; last notification wins on cumulative values and `turn_seq` stays constant within the turn | `usage.rs:231`, `:259-273`; test `last_update_wins_multiple_updates_same_turn` (`:620`) |
| Notification while **no** session is in flight (e.g. during `session/new`) advances the baseline but produces **no** publishable record | `usage.rs:274-291`; test `setup_notification_before_begin_turn_returns_none` (`:351`) |
| Notification for session X while Y is in flight is **ignored entirely** — advancing X's baseline would undercount X's next delta | `usage.rs:292-295`; test `cross_session_notification_does_not_corrupt_other_sessions_delta` (`:404`) |
| `take` clears `in_flight_session` **before** the `?` early-return, so a no-usage turn still ends the in-flight marker | `usage.rs:305-306`; tests `begin_turn_then_take_without_record_returns_none` (`:814`), `take_returns_none_after_drain` (`:611`) |
| `take` advances `published_seq` and the cumulative baseline to the published record only | `usage.rs:307-318` |
| Non-`usage_update` variants deserialize to `Other` and are ignored | `usage.rs:69-71`; test `other_variant_deserializes_without_error` (`:716`) |
| Downstream, `publish_agent_turn_metric` re-checks `delta_reliable` before emitting `turn` counts (defence in depth) and skips publishing entirely when `usage` is `None` or the owner pubkey is unset | `pool.rs:3332-3352` |
