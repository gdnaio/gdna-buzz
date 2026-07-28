## Module: buzz-acp — work queue, filtering & usage accounting (`crates/buzz-acp/src`)
### Aspect: Configuration

None of the three files reads an environment variable or a file. Every knob is parsed in `config.rs` (clap + `env`) or `config::load_rules` (TOML) and passed in as a value.

#### Knobs that reach `EventQueue`

| Knob | Env / flag | Default | Parse site | Effect on the queue |
|---|---|---|---|---|
| `dedup_mode` | `--dedup` / `BUZZ_ACP_DEDUP` | `queue` | `config.rs:344`, enum `config.rs:57-61` | `EventQueue::new(dedup_mode)` (`lib.rs:1505`). `Drop` discards events for in-flight channels at `queue.rs:231-241`; also gates `recoverable_batch` (`lib.rs:2196-2199`) and `requeue_batch_if_queue` (`pool.rs:2971-2977`) |
| `max_turn_duration_secs` | `--max-turn-duration` / `BUZZ_ACP_MAX_TURN_DURATION` | 7200 (`config.rs:30`) | `config.rs:270`, validated `config.rs:876-899`, ceiling `MAX_TURN_DURATION_CEILING_SECS = 604_800` (`config.rs:36`) | `with_in_flight_deadline(max_turn + 100)` (`lib.rs:1506`, `queue.rs:197-201`); also the argument to `extend_in_flight_deadline` on a successful steer (`lib.rs:2509`) |
| `multiple_event_handling` | `--multiple-event-handling` / `BUZZ_ACP_MULTIPLE_EVENT_HANDLING` | `steer` (`config.rs:356`) | `config.rs:352-359`, enum `config.rs:65-84` | selects the mid-turn signal (`mode_gate_signal`, `lib.rs:2224` → `mode_gate_signal` `lib.rs:2741`) which becomes the `CancelReason` stamped by `requeue_cancelled_batch` (`pool.rs:2985-2995`) and consumed by `requeue_as_cancelled` (`queue.rs:542-548`) |

Note the coupling between the last two: `MultipleEventHandling::Steer` is documented as requiring `DedupMode::Queue` (`config.rs:74`, `:79`, `:83`). With `--dedup drop`, a steer still cancels the turn but `requeue_batch_if_queue` returns `None` and the batch is lost (`pool.rs:2973-2976`).

#### Knobs that reach `filter.rs`

| Knob | Env / flag | Default | Parse site | Effect |
|---|---|---|---|---|
| `subscribe_mode` | `--subscribe` / `BUZZ_ACP_SUBSCRIBE` | `mentions` | `config.rs:326-331` | picks which `SubscriptionRule` set is built (`lib.rs:1439-1474`) |
| `kinds_override` | `--kinds` / `BUZZ_ACP_KINDS` (comma-delimited) | `None` | `config.rs:332-333` | `Mentions` → replaces `[9, 46010, 40007]`; `All` → `unwrap_or_default()`, i.e. **empty vec** when unset (`lib.rs:1447-1451`, `:1462`) |
| `no_mention_filter` | `--no-mention-filter` / `BUZZ_ACP_NO_MENTION_FILTER` | `false` | `config.rs:338-339` | `require_mention = !no_mention_filter` (`lib.rs:1452`) |
| `channels_override` | `--channels` / `BUZZ_ACP_CHANNELS` | `None` | `config.rs:335-336` | not read by `filter.rs`; consumed by `resolve_channel_filters` / `resolve_dynamic_channel_filter` (`config.rs:1134`, `:1249-1258`) and ignored entirely in `Config` mode |
| `config_path` | `--config` / `BUZZ_ACP_CONFIG` | `./buzz-acp.toml` | `config.rs:341-342` | `load_rules(&config.config_path)` in `Config` mode (`lib.rs:1472`) |

`buzz-acp.toml` rule keys (deserialized into `SubscriptionRule`, `filter.rs:83-114`):

| Key | Type | Default | Validation |
|---|---|---|---|
| `name` | string | required | non-empty, unique across rules (`config.rs:1084-1093`) |
| `channels` | `"all"` or `[uuid, …]` | required | `"all"` must be exact lowercase (`config.rs:1118-1124`) |
| `kinds` | `[u32]` | `[]` = wildcard | none |
| `require_mention` | bool | `false` | none |
| `filter` | evalexpr string | `None` | ≤4096 bytes, compiled at load (`config.rs:1097-1116`) |
| `prompt_tag` | string | falls back to `name` | none (`filter.rs:451`) |

Global rule-file limit: 100 rules (`config.rs:1067-1072`); zero rules emits a `WARN` and the agent receives nothing (`config.rs:1075-1080`).

#### Knobs that reach the prompt formatter

| Knob | Env / flag | Default | Effect on `format_prompt` |
|---|---|---|---|
| `context_message_limit` | `--context-message-limit` / `BUZZ_ACP_CONTEXT_MESSAGE_LIMIT` | 12, range 0..=100 (`config.rs:364-368`) | `0` disables context fetching entirely, so `conversation_context` is `None` and no `[Thread Context]`/`[Conversation Context]` section is emitted (`pool.rs:1746`, `:2493-2502`) |
| `base_prompt_content` | `--base-prompt` family (`config.rs`) | built-in `base_prompt.md` | `FormatPromptArgs::base_prompt`; rendered as `[Base]` only for legacy agents (`queue.rs:1435-1437`) |
| `system_prompt` / file | `--system-prompt` / `BUZZ_ACP_SYSTEM_PROMPT`, `BUZZ_ACP_SYSTEM_PROMPT_FILE` | none | `[System]` section, legacy agents only (`queue.rs:1438-1440`) |
| protocol version (not a knob — negotiated) | — | — | `has_system_prompt_support` gates `[Base]`/`[System]`/`[Team Instructions]`/core/canvas (`queue.rs:1432-1462`) |

#### Hard-coded values with no configuration path

These are compile-time constants that operators cannot change:

| Constant | Value | Line |
|---|---|---|
| `MAX_PENDING_PER_CHANNEL` | 500 | `queue.rs:24` |
| `MAX_BATCH_EVENTS` | 50 | `queue.rs:27` |
| `MAX_RETRIES` | 10 | `queue.rs:30` |
| `BASE_RETRY_DELAY_SECS` | 5 | `queue.rs:33` |
| `MAX_RETRY_DELAY_SECS` | 300 | `queue.rs:36` |
| `IN_FLIGHT_DEADLINE_BUFFER_SECS` | 100 | `queue.rs:39` |
| `DEFAULT_IN_FLIGHT_DEADLINE_SECS` | 7300 | `queue.rs:42` |
| `MAX_PROMPT_LABEL_LEN` | 64 | `queue.rs:1023` |
| `MAX_EXPR_LEN` | 4096 | `filter.rs:162` |
| `EVAL_TIMEOUT` | 100 ms | `filter.rs:165` |
| `MAX_CONCURRENT_FILTER_EVALS` | 4 | `filter.rs:173` |
| `MAX_CONSECUTIVE_TIMEOUTS` | 5 | `filter.rs:341` |

`DEFAULT_IN_FLIGHT_DEADLINE_SECS = 7300` duplicates `DEFAULT_MAX_TURN_DURATION_SECS (7200) + IN_FLIGHT_DEADLINE_BUFFER_SECS (100)` as a literal rather than computing it; `config.rs:30` and `queue.rs:42` must be edited together. The test `default_in_flight_deadline_exceeds_default_max_turn_duration` (`queue.rs:4530`) is the only guard.

`MAX_EXPR_LEN = 4096` (`filter.rs:162`) is duplicated as a bare `4096` literal in `load_rules` (`config.rs:1098`) — the two can drift.

#### Parsed and never read

| Item | Evidence |
|---|---|
| `UsageUpdatePayload::used` | `#[serde(default)] #[allow(dead_code)]`, `usage.rs:78-81`; no reader anywhere — `pool.rs` never populates a context-utilization field |
| `UsageUpdatePayload::context_limit` | same, `usage.rs:82-85` |
| `MatchedRule::rule_index` | `#[cfg_attr(not(test), allow(dead_code))]`, `filter.rs:152-153`; `lib.rs:2175` and `setup_mode.rs:449` both discard it |

Both usage fields are tested for wire-compatibility (`notification_deserializes_without_used_and_context_limit`, `usage.rs:691`; `buzz_agent_payload_no_context_fields_processes_correctly`, `usage.rs:797`) but never surface in a published metric — `TokenCounts.total_tokens`, `cache_read_tokens`, `cache_write_tokens` are hard-`None` at `pool.rs:3339-3342` and `:3358-3360`.

#### Missing from `.env.example`

`.env.example` documents 5 of the queue/filter-relevant vars — `BUZZ_ACP_SUBSCRIBE` (`.env.example:186`), `BUZZ_ACP_KINDS` (`:189`), `BUZZ_ACP_CHANNELS` (`:192`), `BUZZ_ACP_NO_MENTION_FILTER` (`:195`), `BUZZ_ACP_CONFIG` (`:198`), `BUZZ_ACP_DEDUP` (`:202`), `BUZZ_ACP_CONTEXT_MESSAGE_LIMIT` (`:209`) — and omits:

| Missing var | Reaches | Why it matters |
|---|---|---|
| `BUZZ_ACP_MAX_TURN_DURATION` | `with_in_flight_deadline` (`lib.rs:1506`) | directly sets the in-flight backstop; the only knob that can wedge or prematurely release a channel |
| `BUZZ_ACP_MULTIPLE_EVENT_HANDLING` | cancel/steer path (`lib.rs:2224` → `mode_gate_signal` `lib.rs:2741`) | selects whether mid-turn mentions cancel the turn at all, and which framing the merged prompt uses |
| `BUZZ_ACP_IDLE_TIMEOUT` | validated against `max_turn_duration` (`config.rs:894-899`) | the pair is cross-validated at startup; documenting one without the other is misleading |

Worse, the one timeout variable `.env.example` **does** document — `BUZZ_ACP_TURN_TIMEOUT=320` (`.env.example:152`) — is marked `hide = true` in clap and warns "deprecated and ignored" at runtime (`config.rs:274`, `:853-862`). So the only timeout knob visible to an operator reading `.env.example` is the deprecated one, while the two live knobs that actually set the queue's in-flight deadline are undocumented.
