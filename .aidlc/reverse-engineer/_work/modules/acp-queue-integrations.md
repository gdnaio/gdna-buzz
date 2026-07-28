## Module: buzz-acp — work queue, filtering & usage accounting (`crates/buzz-acp/src`)
### Aspect: Integrations

#### External crates

| Crate | Used for | Sites |
|---|---|---|
| `nostr` | `Event`, `Kind`, `Timestamp`, `EventId`, `PublicKey`, `ToBech32` | `queue.rs:16`; `event.kind.as_u16()` (`queue.rs:1089`, `filter.rs:47`, `:383`), `pubkey.to_hex()` (`queue.rs:1082`), `pubkey.to_bech32()` (`queue.rs:1083`), `event.id.to_hex()` (`queue.rs:1090`, `:629`, `:677`, `:709`, `:740`, `:746`), `tag.as_slice()` (`queue.rs:855`, `:1112`, `filter.rs:392`) |
| `uuid` | channel identity throughout | `queue.rs:19`, `filter.rs:44` |
| `evalexpr` | filter AST + evaluation (`Node`, `HashMapContext`, `Function`, `Value`) | `filter.rs:106`, `:197-262`, `:264-338` |
| `tokio` | `spawn_blocking`, `time::timeout`, `sync::Semaphore` | `filter.rs:182-183`, `:220-249` |
| `serde` / `serde_json` | `SubscriptionRule`/`ChannelScope` TOML deserialization; usage wire JSON; tag serialization inside prompts | `filter.rs:55`, `:82`; `usage.rs:57-93`; `queue.rs:1113` |
| `chrono` | RFC-3339 rendering of `created_at` in `[Event]` blocks | `queue.rs:1085-1087` |
| `thiserror` | `FilterError` | `filter.rs:15` |
| `tracing` | all logging | `queue.rs` (17 macro sites), `filter.rs:12` |

`buzz-core` is **not** imported by any of the three files. Kind constants (`KIND_STREAM_MESSAGE = 9`, `KIND_WORKFLOW_APPROVAL_REQUESTED = 46010`, `KIND_STREAM_REMINDER = 40007` — `buzz-core/src/kind.rs:343`, `:442`, `:355`) are pulled in by the *callers* that build `SubscriptionRule`s: `lib.rs:1447-1451`, `config.rs:1157-1162`, `config.rs:1241-1243`, `config.rs:1262-1268`, `setup_mode.rs:521-565`. `filter.rs` only ever sees raw `u32` kinds, so it has no knowledge of the Buzz kind registry.

#### Consumers inside `buzz-acp`

**`lib.rs` (orchestrator, 6,570 lines)**

| Interaction | Line |
|---|---|
| `use queue::{CancelReason, EventQueue, FlushBatch, QueuedEvent, ThreadTags}` | `lib.rs:44` |
| `use filter::SubscriptionRule` | `lib.rs:36` |
| `pub use usage::TurnUsage` (crate's only public re-export) | `lib.rs:15` |
| Builds `SubscriptionRule`s from `SubscribeMode` | `lib.rs:1439-1474` |
| Constructs the queue: `EventQueue::new(dedup_mode).with_in_flight_deadline(config.max_turn_duration_secs)` | `lib.rs:1505-1506` |
| `has_flushable_work()` — lazy-pool wake gate | `lib.rs:1715` |
| `compact_expired_state()` on the 30 s maintenance tick | `lib.rs:1746` (interval `lib.rs:1608`) |
| `has_flushable_work()` — flush retried batches on quiet channels | `lib.rs:1777` |
| `drain_channel()` when the agent loses channel access; returned ids drive 👀 cleanup | `lib.rs:1990` |
| `filter::match_event(...)` — the sole inbound gate after the author gate | `lib.rs:2174` |
| `push(QueuedEvent { channel_id, event, received_at: Instant::now(), prompt_tag })` | `lib.rs:2198` |
| `is_channel_in_flight()` — gates the mid-turn steer/interrupt fork | `lib.rs:2219` |
| `has_flushable_work()` — post-event dispatch | `lib.rs:2288` |
| `extend_in_flight_deadline` / `remove_event` / `release_native_steer` in the `SteerAck` arm | `lib.rs:2509`, `:2512`, `:2515` |
| `native_steer_framing()` + `format_event_block(channel_id, None, &be, None)` to render the steer body | `lib.rs:2824`, `:2831` |
| `mark_native_steer_pending()` before spawning the ack watcher | `lib.rs:2847` |
| `flush_next()` in `dispatch_pending` | `lib.rs:2896` |
| `parse_thread_tags` on the batch's last event → typing-indicator scope | `lib.rs:2904` |
| `pending_channels()` / `requeue_preserve_timestamps` / `mark_complete` on pool exhaustion | `lib.rs:2910-2913` |
| `recoverable_batch` clone gated on `DedupMode::Queue` | `lib.rs:2196-2199` |
| `parse_thread_tags` for the failure-notice thread anchor | `lib.rs:3023` |
| `requeue_as_cancelled` / `requeue` / `mark_complete` in `handle_prompt_result` | `lib.rs:3090`, `:3120`, `:3145`, `:3171` |
| `requeue` + `mark_complete` in panic recovery | `lib.rs:3428`, `:3440` |
| `set_retry_count_for_test` / `queued_event_count` in the `lib.rs` test module | `lib.rs:5733`; `:5507`, `:5612`, `:5808`, `:6231`, `:6316` |

**`pool.rs` (turn execution, 5,620 lines)**

| Interaction | Line |
|---|---|
| `use crate::queue::{CancelReason, ContextMessage, ConversationContext, FlushBatch, PromptChannelInfo, PromptProfile, PromptProfileLookup, ThreadTags}` | `pool.rs:37-41` |
| `base_section(bp)` for the initial/heartbeat prompt and the system-role prompt | `pool.rs:1097`, `:1145`, `:1147` |
| `slash_command_for_batch(b, &known_names)` | `pool.rs:1761` |
| `format_prompt(batch, &FormatPromptArgs { … })` | `pool.rs:1771-1773` |
| Builds `ConversationContext::{Thread, Dm}` from relay history (gated on `context_message_limit > 0`) | `pool.rs:1746`, `:2493-2502`, `:2964-2968` |
| `parse_thread_tags` for reply targets and mention fan-out | `pool.rs:2512`, `:2546` |
| `requeue_batch_if_queue` — drops the batch in `DedupMode::Drop` | `pool.rs:2971-2977` |
| `requeue_cancelled_batch` — maps `ControlSignal` → `CancelReason`, drops on `Cancel`/`Rotate` | `pool.rs:2981-2995` |
| `publish_agent_turn_metric(ctx, usage: Option<TurnUsage>, …)` | `pool.rs:3322-3420` |
| `TaskMeta.recoverable_batch: Option<FlushBatch>` for panic recovery | `pool.rs:45-46` |

**`acp.rs` (ACP client, 3,717 lines)** — the only owner of `UsageTracker`:

| Interaction | Line |
|---|---|
| `use crate::usage::{TurnUsage, UsageTracker}` | `acp.rs:17` |
| field `goose_usage: UsageTracker`, initialized `UsageTracker::default()` | `acp.rs:202`, `:498` |
| `begin_turn(session_id)` immediately before `session/prompt` | `acp.rs:687-690` |
| `handle_goose_usage_update` → `record(&notif.session_id, payload)`; malformed/other variants silently ignored | `acp.rs:1637-1666`; invoked from the read loop at `acp.rs:1141` and `:1488` |
| `take_turn_usage()` → `goose_usage.take()` | `acp.rs:783-785` |

**`config.rs`** — owns `SubscriptionRule` deserialization: `load_rules` reads TOML, enforces limits, and eagerly compiles each `filter` into `Arc<evalexpr::Node>` (`config.rs:1060-1129`). `resolve_channel_filters` (`config.rs:1134`) and `resolve_dynamic_channel_filter` (`config.rs:1236`) translate rules into relay REQ filters; `rule_applies_to_channel` reuses `ChannelScope` (`config.rs:1118`).

**`setup_mode.rs`** — second `filter::match_event` consumer, with its own rule builder (`setup_mode.rs:444-450`, `:521-565`) and its own `parse_thread_tags` call (`setup_mode.rs:605`).

#### Outbound effects

Neither `queue.rs` nor `filter.rs` performs I/O — no network, no filesystem, no database. Their only side channel is `tracing`. `usage.rs` likewise performs no I/O; the relay write happens in `pool.rs`, which encrypts the `AgentTurnMetricPayload` with `buzz_core::agent_turn_metric::encrypt_agent_turn_metric` and publishes kind 44200 tagged `["p", owner_hex]` and `["agent", agent_hex]` (`pool.rs:3376-3401`). `StopReason` is mapped from ACP to NIP-AM at `pool.rs:3305-3314`.

#### Relay-shape coupling carried in these files

- `flush_next`'s chronological re-sort exists because relay replay is `ORDER BY created_at DESC` (`queue.rs:346-350`).
- `parse_thread_tags` supports only marker-form NIP-10 `e` tags, justified by a cross-repo reference to `relay messages.rs:762-783` (`queue.rs:846-848`) — a stale pointer into `buzz-relay` that this crate cannot verify.
- `[Context]` hints hard-code `buzz` CLI invocations (`buzz messages thread --channel <UUID> --event <ID>`, `buzz messages get --channel <UUID>`) — an undeclared coupling to `buzz-cli`'s command surface with no compile-time check (`queue.rs:1253-1259`, `:1282-1284`, `:1307`).
- `[Context]` reply instructions hard-code `buzz messages send --reply-to <id>` (`queue.rs:1150-1180`).
