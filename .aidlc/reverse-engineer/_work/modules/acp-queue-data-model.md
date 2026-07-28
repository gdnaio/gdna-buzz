## Module: buzz-acp — work queue, filtering & usage accounting (`crates/buzz-acp/src`)
### Aspect: Data Model

All state is **in-process only** — no persistence, no serialization to disk or relay. `EventQueue` and `UsageTracker` live in the main-loop stack frame (`lib.rs:1504-1506`) and die with the process.

#### `EventQueue` — 11 fields, all keyed by channel UUID (`queue.rs:137-171`)

| Field | Type | Line | Bound / eviction |
|---|---|---|---|
| `queues` | `HashMap<Uuid, VecDeque<QueuedEvent>>` | `queue.rs:138` | Per-channel deque capped at `MAX_PENDING_PER_CHANNEL = 500` (`queue.rs:24`). Overflow drops **oldest** (`pop_front`) on `push` (`queue.rs:242-249`) and **newest** (`pop_back`) on all three front-insert paths (`queue.rs:486-495`, `:521-528`, `:722-729`). Map entry removed when the deque empties (`queue.rs:352-355`, `:683-685`, `:747-749`). Otherwise unbounded in channel count. |
| `in_flight_channels` | `HashSet<Uuid>` | `queue.rs:139` | Insert on flush (`queue.rs:357`, `:319`), remove on `mark_complete` (`queue.rs:393`) or deadline expiry (`queue.rs:279`, `:575`). |
| `in_flight_deadlines` | `HashMap<Uuid, Instant>` | `queue.rs:141` | Set to `now + in_flight_deadline` on flush; removed with the in-flight entry. Monotonically extendable via `extend_in_flight_deadline` (`queue.rs:210-228`). |
| `in_flight_batch_sizes` | `HashMap<Uuid, usize>` | `queue.rs:143` | Purely for expiry logging (`lost_events`); removed at `mark_complete` (`queue.rs:395`) and expiry (`queue.rs:271`, `:567`). |
| `retry_after` | `HashMap<Uuid, Instant>` | `queue.rs:144` | Written only by `requeue` (`queue.rs:496`). Cleared on dead-letter (`queue.rs:449`), `drain_channel` (`queue.rs:631`), expired-throttle `mark_complete` (`queue.rs:403`), and by `compact_expired_state`'s `retain(|_, d| *d > now)` (`queue.rs:809`). |
| `retry_counts` | `HashMap<Uuid, u32>` | `queue.rs:146` | Incremented per `requeue` attempt (`queue.rs:431-435`); dead-letter threshold `MAX_RETRIES = 10` (`queue.rs:30`). Removed on dead-letter, `drain_channel`, healthy `mark_complete`, and orphan compaction (`queue.rs:813-817`). |
| `dedup_mode` | `DedupMode` | `queue.rs:147` | Immutable after construction. |
| `cancelled_batches` | `HashMap<Uuid, Vec<BatchEvent>>` | `queue.rs:151` | **No cap.** `requeue_as_cancelled` `extend`s the vec on every cancel (`queue.rs:543-546`); only drained by `flush_next` (`queue.rs:317`, `:362-365`) or `drain_channel` (`queue.rs:633`). Repeated cancel-without-flush grows without limit. |
| `cancel_reasons` | `HashMap<Uuid, CancelReason>` | `queue.rs:155` | One entry per channel; last writer wins (`queue.rs:547`). |
| `withheld_native_steer` | `HashMap<Uuid, Vec<QueuedEvent>>` | `queue.rs:166` | Side table for goose-native steer. **No cap on the vec.** Entries removed by `release_native_steer` (`queue.rs:714-721`), `remove_event` (`queue.rs:740-745`), `drain_channel` (`queue.rs:635`), or bulk recovery on in-flight expiry (`queue.rs:766-789`). |
| `in_flight_deadline` | `Duration` | `queue.rs:170` | `DEFAULT_IN_FLIGHT_DEADLINE_SECS = 7300` (`queue.rs:42`) unless overridden by `with_in_flight_deadline(max_turn + 100)` (`queue.rs:197-201`, buffer const `queue.rs:39`). |

`EventQueue` derives nothing — no `Debug`, no `Clone`. `Default` maps to `DedupMode::Drop` (`queue.rs:822-824`).

#### Event carriers

| Type | Fields | Line |
|---|---|---|
| `QueuedEvent` | `channel_id: Uuid`, `event: nostr::Event`, `received_at: Instant`, `prompt_tag: String` | `queue.rs:46-53` |
| `BatchEvent` | `event`, `prompt_tag`, `received_at` (no `channel_id` — carried by the batch) | `queue.rs:56-61` |
| `FlushBatch` | `channel_id`, `events: Vec<BatchEvent>` (≤ `MAX_BATCH_EVENTS = 50`, `queue.rs:27`), `cancelled_events: Vec<BatchEvent>` (unbounded), `cancel_reason: Option<CancelReason>` | `queue.rs:77-90` |
| `CancelReason` | `Interrupt` \| `Steer`; `Copy` | `queue.rs:65-74` |

`FlushBatch`/`BatchEvent` are `Clone` specifically so `dispatch_pending` can stash `recoverable_batch` in `TaskMeta` for panic recovery (`queue.rs:76`, comment at `pool.rs:45-46`, clone site `lib.rs:2199-2202`).

`received_at` is a monotonic `Instant`, not a wall clock — fairness ordering survives clock changes but cannot be persisted or logged as a timestamp.

#### Prompt-formatting model

| Type | Shape | Line |
|---|---|---|
| `ThreadTags` | `root_event_id: Option<String>`, `parent_event_id: Option<String>`, `mentioned_pubkeys: Vec<String>` — all raw hex from tags, unbounded count | `queue.rs:829-836` |
| `ConversationContext` | enum `Thread { messages, total, truncated }` \| `Dm { … }` | `queue.rs:970-984` |
| `ContextMessage` | `pubkey`, `timestamp` (pre-rendered string), `content` | `queue.rs:987-992` |
| `PromptChannelInfo` | `name`, `channel_type` (`"dm"` is the only value tested against, `queue.rs:1420-1422`) | `queue.rs:995-999` |
| `PromptProfile` | `display_name: Option<String>`, `nip05_handle: Option<String>`, `is_agent: bool` | `queue.rs:1002-1010` |
| `PromptProfileLookup` | `HashMap<String, PromptProfile>` keyed by `trim().to_ascii_lowercase()` pubkey (`normalize_lookup_key`, `queue.rs:1017-1021`) | `queue.rs:1012` |
| `FormatPromptArgs<'a>` | 9 borrowed optional fields + `has_system_prompt_support: bool`; `#[derive(Default)]` | `queue.rs:1353-1375` |
| `MergeFraming` | 4 `&'static str` framing slots; private, constructed only via `for_reason` | `queue.rs:1571-1610` |

Only labels are bounded: `sanitize_prompt_label` strips control chars and truncates at `MAX_PROMPT_LABEL_LEN = 64` chars (`queue.rs:1023`, `:1028-1040`). Event `content`, tag JSON, and `ContextMessage::content` are inlined verbatim with no cap (`queue.rs:1108`, `:1112-1115`, `:1345`).

#### `filter.rs` model

| Type | Fields | Line |
|---|---|---|
| `FilterContext` | `content: String`, `author: String` (hex), `kind: u32`, `channel_id: String`, `timestamp: u64` — a flattened snapshot built per event by `from_event` (`filter.rs:42-51`) | `filter.rs:27-38` |
| `ChannelScope` | `#[serde(untagged)]` `All(String)` \| `List(Vec<String>)`; `All` matches only the literal `"all"` (`filter.rs:67-72`) | `filter.rs:56-61` |
| `SubscriptionRule` | `name`, `channels`, `kinds: Vec<u32>` (empty = wildcard), `require_mention: bool`, `filter: Option<String>`, `prompt_tag: Option<String>`, `compiled_filter: Option<Arc<evalexpr::Node>>` (`#[serde(skip)]`), `consecutive_timeouts: Arc<AtomicU32>` (`#[serde(skip)]`) | `filter.rs:83-114` |
| `MatchedRule` | `rule_index: usize` (`#[cfg_attr(not(test), allow(dead_code))]`), `prompt_tag: String` | `filter.rs:150-156` |
| `FilterError` | `ExpressionTooLong { len, max }`, `Timeout`, `EvalError(String)` | `filter.rs:16-24` |

`SubscriptionRule`'s hand-written `Clone` deliberately **shares** the `Arc<AtomicU32>` timeout counter across clones (`filter.rs:131-145`), so a rule disabled in one clone is disabled in all. `Vec<u32>` kinds and rule count are bounded only at load time: max 100 rules, filter expression ≤ 4096 bytes (`config.rs:1067-1072`, `:1097-1104`).

#### `usage.rs` model

| Type | Fields | Line |
|---|---|---|
| `UsageUpdatePayload` | `used: u64`, `context_limit: u64` (both `#[serde(default)]` + `#[allow(dead_code)]`), `accumulated_input_tokens: u64`, `accumulated_output_tokens: u64`, `accumulated_cost: Option<f64>`, `model: Option<String>` | `usage.rs:76-93` |
| `GooseSessionUpdateNotification` | `session_id: String`, `update: GooseSessionUpdateVariant` | `usage.rs:58-61` |
| `GooseSessionUpdateVariant` | `UsageUpdate(UsageUpdatePayload)` \| `Other` (`#[serde(other)]` catch-all) | `usage.rs:67-72` |
| `SessionState` (private) | `published_seq: u64` (`:102`), `last_input: u64` (`:105`), `last_output: u64` (`:107`), `last_cost: Option<f64>` (`:109`) — snapshot at end of last **published** turn | `usage.rs:97-110` |
| `TurnUsage` | `session_id`, `turn_seq: u64`, `delta_reliable: bool`, `turn_input_tokens/turn_output_tokens: Option<u64>`, `turn_cost_usd: Option<f64>`, `cumulative_input_tokens/cumulative_output_tokens: u64`, `cumulative_cost_usd: Option<f64>`, `model: Option<String>` | `usage.rs:117-140` |
| `UsageTracker` | `sessions: HashMap<String, SessionState>` (`:166`), `in_flight_session: Option<String>` (`:170`), `pending: Option<TurnUsage>` (`:172`); `#[derive(Debug, Default)]` | `usage.rs:163-173` |

`sessions` is documented as "one entry per goose `sessionId` **ever seen** in this process" (`usage.rs:165`) — there is no eviction path, so a harness that rotates sessions frequently accumulates one `SessionState` (32 bytes + key string) per session for process lifetime. Tokens are `u64`; cost is `f64` (`turn_cost_usd` computed as raw `f64` subtraction, `usage.rs:242`).
