## Module: buzz-acp — work queue, filtering & usage accounting (`crates/buzz-acp/src`)
### Aspect: Features

#### Reachability summary

| Feature | Entry point | Reachable by default? |
|---|---|---|
| Per-channel FIFO queueing with cross-channel fairness | `EventQueue::push` / `flush_next` (`queue.rs:230`, `:260`) | yes — always on |
| Batching up to 50 events per prompt | `queue.rs:336-345` | yes; only observable when >1 event queues for one channel |
| Drop-mode dedup | `queue.rs:231-241` | **no** — CLI default is `--dedup queue` (`config.rs:344`) |
| Queue-mode accumulation | `queue.rs:231` (guard is Drop-only) | yes (default) |
| Exponential-backoff retry with jitter | `queue.rs:454-464` | yes, on any transient prompt failure |
| Dead-lettering after 10 attempts | `queue.rs:437-451` | yes |
| Immediate dead-letter on hard timeout with no recent activity | `lib.rs:3091-3109` | yes |
| Immediate dead-letter on auth error | `lib.rs:3130-3144` | yes |
| Cancel + merged re-prompt (steer / interrupt framing) | `requeue_as_cancelled` (`queue.rs:542`) + `MergeFraming` (`queue.rs:1584`) | yes — `multiple_event_handling` defaults to `steer` (`config.rs:352-359`) |
| Goose-native non-cancelling steer (withhold side table) | `mark_native_steer_pending` / `release_native_steer` / `remove_event` (`queue.rs:673`, `:703`, `:738`) | attempted for every agent; falls back to cancel+merge on `-32601` (`lib.rs:2476-2483`) |
| In-flight deadline backstop with monotonic extension | `queue.rs:263-287`, `:210-228` | yes |
| Periodic map compaction | `compact_expired_state` (`queue.rs:807`) | yes — 30 s tick (`lib.rs:1608`, `:1746`) |
| Channel teardown on agent removal | `drain_channel` (`queue.rs:625`) | yes (`lib.rs:1990`) |
| Structured prompt framing (9 section types) | `format_prompt` (`queue.rs:1406`) | yes |
| Legacy-agent prompt injection (`[Base]`/`[System]`/`[Team Instructions]`/core/canvas in the user message) | `queue.rs:1432-1462` | only when `has_system_prompt_support == false` (protocol < 2) |
| Human-aware reply anchoring | `resolve_reply_anchor` (`queue.rs:1209`) | yes, but degrades to "everyone is human" without a `profile_lookup` |
| Slash-command pass-through | `slash_command_for_batch` (`queue.rs:961`) | yes (`pool.rs:1761`) |
| Profile-aware display labels | `resolve_prompt_label` (`queue.rs:1042`) | only when `profile_lookup` is supplied — **not** on the native-steer path (`lib.rs:2831` passes `None`) |
| Rule-based subscription matching | `filter::match_event` (`filter.rs:368`) | yes |
| evalexpr content filters | `evaluate_filter` (`filter.rs:197`) | only in `--subscribe config` mode with a `filter =` key; `Mentions`/`All` rules set `filter: None` (`lib.rs:1446`, `:1464`) |
| Filter timeout circuit breaker | `filter.rs:405-415` | yes, but only exercisable via config-mode filters |
| Per-turn token/cost accounting | `UsageTracker` (`usage.rs:163`) | only when the harness emits `_goose/unstable/session/update` with `sessionUpdate: "usage_update"` |
| NIP-AM kind 44200 metric publishing | `pool.rs:3322-3420` | only when usage is present **and** `agent_owner_pubkey` is configured (`pool.rs:3333-3336`) |

#### Batching

`flush_next` drains the whole channel queue up to `MAX_BATCH_EVENTS = 50` into one `FlushBatch` (`queue.rs:27`, `:336-345`), then re-sorts ascending by `created_at` because relay replay arrives newest-first (`queue.rs:346-350`). `format_prompt` renders 1 event as `[Buzz event: {tag}]` and N events as `[Buzz events — N events]` with `--- Event i (tag) ---` separators (`queue.rs:1528-1552`). Tests: `test_batch_drain_all_events` (`queue.rs:1759`), `test_format_prompt_batch` (`:2173`), `test_flush_orders_replayed_events_chronologically` (`:1782`).

Cross-channel interleaving is by oldest head event, so a busy channel cannot starve a quiet one — `test_multi_channel_interleave` (`queue.rs:1826`), `test_fifo_fairness_picks_oldest_channel` (`:1810`).

#### Dedup

Two modes only (`config.rs:57-61`). `Drop` discards in-flight-channel events at admission; `Queue` accumulates. Beyond that there is **no content-level dedup** — the same event delivered twice by the relay queues twice. The only id-based suppression in this area is `remove_event` on a successful native steer (`queue.rs:738-751`) and `should_nudge_for_event`'s id set in setup mode (`setup_mode.rs:389`, `:495`). Tests: `test_drop_mode_discards_in_flight_events` (`queue.rs:2504`), `test_drop_mode_queues_other_channels` (`:2522`), `test_drop_mode_drops_for_any_in_flight_channel` (`:2593`).

#### Retry and dead-lettering

Per-channel attempt counter, 5 s base doubling to a 300 s cap, ±20 % jitter, 10 attempts (≈25 min of wall clock) before the batch is handed back to the caller for a user-visible failure notice (`queue.rs:429-498`; notice construction `lib.rs:3122-3127`, `:3150-3153`). `requeue_preserve_timestamps` is a deliberately budget-free variant used when no agent is free (`queue.rs:508-529`) — it can loop forever without dead-lettering.

Three distinct dead-letter triggers exist, only one of which consults the retry budget:

1. budget exhausted inside `requeue` (`queue.rs:437`),
2. hard timeout with `recently_active: false` (`lib.rs:3091-3109`) — bypasses the queue entirely,
3. auth error (`lib.rs:3130-3144`) — bypasses the queue entirely.

`hard_timeout_fate_suffix` (`lib.rs:3059`, set at `:3108`, `:3125`, `:3128`, `:3162`) exists solely to make the death message describe the batch's actual fate.

#### Prompt framing

`format_prompt` returns a `Vec<String>` of independent sections rather than one string so the observer frame trimmer can elide one section's body while keeping every `[Header]` line countable in the desktop "Prompt context" panel (`queue.rs:1394-1400`; trimmer `lib.rs:659`). Section inventory: `[Base]`, `[System]`, `[Team Instructions]`, agent-core memory block (rendered upstream by `engram_fetch::build_core_section` (`engram_fetch.rs:39`)), `[Channel Canvas]`, `[Context]`, `[Thread Context]`/`[Conversation Context]`, the cancelled/new event blocks, and a closing note.

`[Context]` has three shapes — `Scope: dm`, `Scope: thread`, `Scope: channel` — each with a different `buzz` CLI hint and, for human-facing turns, a `--reply-to` instruction (`queue.rs:1233-1315`). DM is checked first so a DM reply reports `dm`, not `thread` (`queue.rs:1246-1248`).

Merge framing is a two-variant table selected by `CancelReason` (`queue.rs:1584-1609`). Native steer reuses `new_header_single` + `closing_note` from the same table via `native_steer_framing()` (`queue.rs:1623-1626`) and the same `format_event_block`, so the two transports are structurally prevented from drifting (`lib.rs:2812-2823` states this as the requirement).

#### Slash-command extraction

Mention-stripping loop handles NIP-27 `nostr:npub1…`/`nostr:nprofile1…` tokens, `@`-prefixed multi-word known display names (longest-first, case-insensitive), and `@`-prefixed single tokens of `[A-Za-z0-9._-]` (`queue.rs:906-947`). Pass-through is gated to single-event, no-carryover batches (`queue.rs:961-967`). Known names come from the caller at `pool.rs:1761`.

#### Usage / cost accounting

Cumulative-to-delta normalization with three explicit unreliability cases (first turn, token decrease, cost decrease), multi-notification-per-turn tolerance, session-restart handling, and cross-session isolation (`usage.rs:211-302`). `turn_seq` counts **published** metrics, not notifications (`usage.rs:99-102`, `:231`). Downstream, `publish_agent_turn_metric` builds an `AgentTurnMetricPayload`, encrypts it to the owner pubkey, and publishes kind `KIND_AGENT_TURN_METRIC` (44200) with `p` and `agent` tags (`pool.rs:3363-3400`). Cache-token and total-token fields exist in the payload type but are always `None` from this path (`pool.rs:3341-3342`, `:3358-3360`) — `used` / `context_limit` from the wire payload are parsed and never read (`usage.rs:78-85`), so context-window utilization is not reported anywhere.
