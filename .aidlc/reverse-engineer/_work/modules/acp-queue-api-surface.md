## Module: buzz-acp — work queue, filtering & usage accounting (`crates/buzz-acp/src`)
### Aspect: API Surface

Nothing in these three modules is reachable from outside the crate except `TurnUsage`, which is re-exported at `lib.rs:15` (`pub use usage::TurnUsage;`) — the crate's only public re-export. `lib.rs` never names `TurnUsage` itself; the only consumers are inside the crate (`acp.rs:783`, `pool.rs:3324`). The sole downstream crate, `sprig`, calls `buzz_acp::run()` only (`sprig/src/main.rs:17`), so the re-export currently has no external reader.

#### `EventQueue` — constructors and lifecycle

| Item | Line | Callers |
|---|---|---|
| `EventQueue::new(DedupMode)` | `queue.rs:179-195` | `lib.rs:1505-1506` (only production site) |
| `.with_in_flight_deadline(max_turn_duration_secs)` | `queue.rs:197-201` | `lib.rs:1506` |
| `impl Default` → `new(DedupMode::Drop)` | `queue.rs:822-824` | tests only |
| `extend_in_flight_deadline(channel_id, max_turn_secs)` | `queue.rs:210-228` | `lib.rs:2509` (`SteerAck::Success` arm) |

#### `EventQueue` — hot path

| Item | Line | Callers |
|---|---|---|
| `push(QueuedEvent) -> bool` | `queue.rs:230-252` | `lib.rs:2198` |
| `flush_next() -> Option<FlushBatch>` | `queue.rs:260-380` | `lib.rs:2896` (`dispatch_pending` loop) |
| `mark_complete(channel_id)` | `queue.rs:392-410` | `lib.rs:2913` (pool exhausted), `lib.rs:3171` (`handle_prompt_result`), `lib.rs:3440` |
| `requeue(FlushBatch) -> Option<FlushBatch>` | `queue.rs:429-498` | `lib.rs:3120` (hard-timeout recently-active), `lib.rs:3145` (generic failure), `lib.rs:3428` (panic recovery) |
| `requeue_preserve_timestamps(FlushBatch)` | `queue.rs:508-529` | `lib.rs:2912` (no agent available) |
| `requeue_as_cancelled(FlushBatch, CancelReason)` | `queue.rs:542-548` | `lib.rs:3090` |
| `has_flushable_work() -> bool` (`&mut self`) | `queue.rs:556-591` | `lib.rs:1715`, `:1777`, `:2288` |
| `is_channel_in_flight(channel_id) -> bool` | `queue.rs:645-647` | `lib.rs:2219` |
| `drain_channel(channel_id) -> Vec<String>` | `queue.rs:625-642` | `lib.rs:1990` (agent removed from channel); returns hex event ids for 👀 reaction cleanup |
| `pending_channels() -> usize` | `queue.rs:594-596` | `lib.rs:2910`, `:2975` (log fields) |
| `compact_expired_state()` | `queue.rs:807-818` | `lib.rs:1746` (30 s maintenance tick, interval at `lib.rs:1608`) |

#### `EventQueue` — goose-native steer side table

| Item | Line | Callers |
|---|---|---|
| `mark_native_steer_pending(channel_id, event_id) -> bool` | `queue.rs:673-691` | `lib.rs:2847` |
| `release_native_steer(channel_id, event_id)` | `queue.rs:703-730` | `lib.rs:2515` (`SteerAck::Err` / `PromptCompletedNeutral`) |
| `remove_event(channel_id, event_id)` | `queue.rs:738-751` | `lib.rs:2512` (`SteerAck::Success`) |
| `recover_withheld_for_expired_channel` (private) | `queue.rs:766-789` | `flush_next` (`queue.rs:286`) and `has_flushable_work` (`queue.rs:580`) expiry blocks |

#### `EventQueue` — test-only accessors

| Item | Line | Callers |
|---|---|---|
| `queued_event_count(&Uuid) -> usize` (`#[cfg(test)]`) | `queue.rs:600-602` | `lib.rs:5507`, `:5612`, `:5808`, `:6231`, `:6316` |
| `set_retry_count_for_test(Uuid, u32)` (`#[cfg(test)]`) | `queue.rs:610-612` | `lib.rs:5733` |
| `MAX_RETRIES` (`pub(crate)`) | `queue.rs:30` | `lib.rs:5733` and doc reference `lib.rs:5722` |

All other queue consts (`MAX_PENDING_PER_CHANNEL`, `MAX_BATCH_EVENTS`, `BASE_RETRY_DELAY_SECS`, `MAX_RETRY_DELAY_SECS`, `IN_FLIGHT_DEADLINE_BUFFER_SECS`, `DEFAULT_IN_FLIGHT_DEADLINE_SECS`) are private to `queue.rs` (`queue.rs:24-42`) and cannot be observed or overridden by callers.

#### Prompt-formatting surface (all in `queue.rs`, all consumed by `pool.rs` / `lib.rs`)

| Item | Visibility | Line | Callers |
|---|---|---|---|
| `format_prompt(&FlushBatch, &FormatPromptArgs) -> Vec<String>` | `pub` | `queue.rs:1406-1564` | `pool.rs:1771` |
| `FormatPromptArgs<'a>` | `pub` | `queue.rs:1353-1375` | `pool.rs:1773` |
| `format_event_block(channel_id, channel_info, &BatchEvent, profile_lookup) -> String` | `pub(crate)` | `queue.rs:1076-1147` | `lib.rs:2831` (native steer); internally by `format_prompt` |
| `base_section(&str) -> String` | `pub(crate)` | `queue.rs:1382-1384` | `pool.rs:1097`, `:1145`, `:1147`; internally by `format_prompt` (`queue.rs:1436`) |
| `native_steer_framing() -> (&'static str, &'static str)` | `pub(crate)` | `queue.rs:1623-1626` | `lib.rs:2824` |
| `parse_thread_tags(&Event) -> ThreadTags` | `pub` | `queue.rs:849-900` | `lib.rs:2904`, `:3023`; `pool.rs:2512`, `:2546`; `setup_mode.rs:605` |
| `extract_slash_command(&str, &[&str]) -> Option<String>` | `pub` | `queue.rs:902-959` | only `slash_command_for_batch` + tests |
| `slash_command_for_batch(&FlushBatch, &[&str]) -> Option<String>` | `pub` | `queue.rs:961-967` | `pool.rs:1761` |
| `ThreadTags`, `ConversationContext`, `ContextMessage`, `PromptChannelInfo`, `PromptProfile`, `PromptProfileLookup`, `CancelReason`, `FlushBatch`, `BatchEvent`, `QueuedEvent` | `pub` | `queue.rs:45-1012` | imported by `pool.rs:37-41`, `lib.rs:44` |

Private helpers with no external surface: `normalize_lookup_key` (`:1017`), `sanitize_prompt_label` (`:1028`), `resolve_prompt_label` (`:1042`), `format_prompt_actor` (`:1060`), `append_reply_instruction` (`:1149`), `append_new_thread_reply_instruction` (`:1164`), `turn_is_human_facing` (`:1182`), `resolve_reply_anchor` (`:1209`), `format_context_hints` (`:1233`), `format_conversation_context` (`:1317`), `MergeFraming::for_reason` (`:1584`).

#### `filter.rs` surface

| Item | Visibility | Line | Callers |
|---|---|---|---|
| `match_event(&Event, Uuid, &[SubscriptionRule], &str) -> Option<MatchedRule>` (async) | `pub` | `filter.rs:368-459` | `lib.rs:2174`, `setup_mode.rs:444` |
| `evaluate_filter(&str, &FilterContext, Option<Arc<Node>>) -> Result<bool, FilterError>` (async) | `pub` | `filter.rs:197-262` | `match_event` (`filter.rs:428`) + tests only |
| `FilterContext::from_event(&Event, Uuid)` | `pub` | `filter.rs:42-51` | `match_event` (`filter.rs:374`) + tests |
| `SubscriptionRule` (+ `Default`, `Clone`, `Deserialize`) | `pub` | `filter.rs:83-148` | `config.rs:16`, `config.rs:1060` (`load_rules`), `lib.rs:1439-1474`, `setup_mode.rs:545-565` |
| `ChannelScope::matches(&Uuid) -> bool` | `pub` | `filter.rs:67-72` | `match_event` (`filter.rs:378`), `config.rs:1118` |
| `MatchedRule`, `FilterError` | `pub` | `filter.rs:150-156`, `:16-24` | `lib.rs:2175`, internal |
| `build_eval_context` | private | `filter.rs:264-338` | `evaluate_filter` |
| `MAX_EXPR_LEN`, `EVAL_TIMEOUT`, `MAX_CONCURRENT_FILTER_EVALS`, `MAX_CONSECUTIVE_TIMEOUTS`, `FILTER_EVAL_SEMAPHORE` | private | `filter.rs:162`, `:165`, `:173`, `:341`, `:182-183` | internal; `MAX_EXPR_LEN`'s 4096 value is duplicated as a literal at `config.rs:1098` |

#### `usage.rs` surface

| Item | Visibility | Line | Callers |
|---|---|---|---|
| `TurnUsage` | `pub` (re-exported `lib.rs:15`) | `usage.rs:117-140` | `acp.rs:783` return type, `pool.rs:3324` parameter, constructed in `pool.rs` tests `:5136`, `:5168`, `:5201`, `:5234` |
| `UsageTracker` | `pub(crate)` | `usage.rs:163-173` | field `AcpClient.goose_usage` (`acp.rs:202`), initialized `acp.rs:498` |
| `UsageTracker::begin_turn(&str)` | `pub(crate)` | `usage.rs:182-185` | `acp.rs:690` (immediately before `session/prompt`) |
| `UsageTracker::record(&str, &UsageUpdatePayload)` | `pub(crate)` | `usage.rs:211-302` | `acp.rs:1659` (via `handle_goose_usage_update`, called at `acp.rs:1141` and `:1488`) |
| `UsageTracker::take() -> Option<TurnUsage>` | `pub(crate)`, `#[cfg_attr(not(test), allow(dead_code))]` | `usage.rs:304-320` | `acp.rs:784` (`AcpClient::take_turn_usage`) |
| `GooseSessionUpdateNotification`, `GooseSessionUpdateVariant`, `UsageUpdatePayload` | `pub(crate)` | `usage.rs:58-93` | `acp.rs:1638` (import inside `handle_goose_usage_update`) |

`UsageTracker::take` carries `#[cfg_attr(not(test), allow(dead_code))]` (`usage.rs:303`) even though `acp.rs:784` calls it in production — the attribute is unnecessary and masks real dead-code signal. `MatchedRule::rule_index` (`filter.rs:152`) and `UsageUpdatePayload::{used, context_limit}` (`usage.rs:80`, `:84`) carry the same allow and genuinely are unread outside tests.
