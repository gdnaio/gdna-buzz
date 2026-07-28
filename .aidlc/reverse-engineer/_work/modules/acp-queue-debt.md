## Module: buzz-acp — work queue, filtering & usage accounting (`crates/buzz-acp/src`)
### Aspect: Technical Debt

Zero `TODO` / `FIXME` / `HACK` / `XXX` markers across all three files — the debt below is structural, not annotated.

#### File and function size

| File | Lines | Non-test lines | Test lines |
|---|---|---|---|
| `queue.rs` | 4,759 | 1,627 (`:1-1627`) | 3,132 (`:1628-4759`, 66 %) |
| `filter.rs` | 787 | 461 (`:1-461`) | 326 (`:462-787`) |
| `usage.rs` | 892 | 321 (`:1-321`) | 571 (`:322-892`, 64 %) |

`AGENTS.md` enforces a 1,000-line-per-file ceiling for desktop, web, and mobile (`mobile/scripts/check-file-sizes.mjs` via `just mobile-check`) with **no Rust equivalent**. `queue.rs` is 4.8× that ceiling; even its non-test half exceeds it by 60 %.

Longest functions:

| Function | Lines | Span |
|---|---|---|
| `format_prompt` | 159 | `queue.rs:1406-1564` |
| `flush_next` | 121 | `queue.rs:260-380` |
| `record` (UsageTracker) | 92 | `usage.rs:211-302` |
| `match_event` | 92 | `filter.rs:368-459` |
| `format_context_hints` | 83 | `queue.rs:1233-1315` |
| `build_eval_context` | 75 | `filter.rs:264-338` |
| `format_event_block` | 72 | `queue.rs:1076-1147` |
| `requeue` | 70 | `queue.rs:429-498` |

`flush_next` does four distinct jobs in one body — expire stuck in-flight channels, pick a fair candidate, handle the cancelled-only fallback with an early `return`, then drain/sort/merge — with a shadowed `channel_id` binding across the fallback `match` (`queue.rs:291`, `:305`).

#### Duplicated logic

- The in-flight expiry block is copy-pasted between `flush_next` (`queue.rs:263-287`) and `has_flushable_work` (`queue.rs:558-581`): same filter, same `ERROR` message text, same three removals, same `recover_withheld_for_expired_channel` call. A change to one must be mirrored by hand; the comment at `queue.rs:559` ("same logic as flush_next") acknowledges this.
- The candidate predicate is also duplicated: `queue.rs:294-298` vs `:583-586`.
- The per-channel cap-enforcement `while` loop appears four times with four different log messages (`queue.rs:242-249`, `:486-495`, `:722-729`, `:775-782`).
- `MAX_EXPR_LEN = 4096` (`filter.rs:162`) is re-encoded as a bare literal in `load_rules` (`config.rs:1098`).
- `DEFAULT_IN_FLIGHT_DEADLINE_SECS = 7300` (`queue.rs:42`) hard-codes `DEFAULT_MAX_TURN_DURATION_SECS (7200) + IN_FLIGHT_DEADLINE_BUFFER_SECS (100)` rather than computing it.

#### Untested surface

| Item | Line | Status |
|---|---|---|
| `EventQueue::remove_event` | `queue.rs:738-751` | **zero** calls in the `queue.rs` test module — referenced only in a section comment at `queue.rs:4268`. The `SteerAck::Success` path (`lib.rs:2512`) has no unit test for its queue effect |
| `EventQueue::is_channel_in_flight` | `queue.rs:645-647` | never called in tests; the test helper `any_in_flight` reads the private field instead (`queue.rs:1687-1689`) |
| `native_steer_framing()` | `queue.rs:1623-1626` | no test; the "native and fallback must not drift" requirement (`queue.rs:1618-1621`, `lib.rs:2812-2814`) is unasserted |
| `EventQueue::pending_channels` | `queue.rs:594-596` | never asserted; tests use the private-field helper `pending_count` (`queue.rs:1683-1685`) |
| `format_context_hints` (direct) | `queue.rs:1233-1315` | only exercised transitively through `format_prompt` |
| `FilterError::Timeout` real path | `filter.rs:426-436` | no test drives an actual timeout; `test_consecutive_timeouts_disables_rule` pre-seeds the counter (`filter.rs:766-786`) |
| `MAX_CONCURRENT_FILTER_EVALS` saturation | `filter.rs:173-183` | no test; the permit-held-until-thread-finishes property documented at `filter.rs:167-172` is unverified |
| `ChannelScope` / `SubscriptionRule` deserialization | `filter.rs:55-61`, `:82-114` | no `serde` round-trip test in `filter.rs`; `#[serde(untagged)]` behaviour is only covered indirectly by `config.rs` tests |
| Jitter distribution in `requeue` | `queue.rs:457-464` | no test; `SystemTime`-nanos entropy is untestable as written |
| `queues` map growth in channel count | `queue.rs:138` | no test asserts a bound because there isn't one |

Test **count** is high (112 / 15 / 20) but skewed: 46 of the 112 `queue.rs` tests are `format_prompt`/framing assertions, while the concurrency-sensitive native-steer and expiry paths have 7 between them (`queue.rs:4278-4451`, `:4633-4759`).

#### Unbounded state (no eviction path)

| Structure | Line | Note |
|---|---|---|
| `cancelled_batches[ch]` | `queue.rs:151` | `extend`ed on every cancel (`queue.rs:543-546`) with no `MAX_BATCH_EVENTS`-style cap; flows straight into prompt size |
| `withheld_native_steer[ch]` | `queue.rs:166` | vec has no cap of its own |
| `in_flight_batch_sizes` | `queue.rs:143` | not touched by `compact_expired_state` (`queue.rs:807-818`); leaks for any channel that never completes or expires |
| `queues` (channel count) | `queue.rs:138` | drain paths prune emptied deques (`queue.rs:352-355`, `:683-685`, `:747-749`) but `compact_expired_state` never touches `queues`, so a zero-length entry created by `entry().or_default()` (`queue.rs:475`, `:510`, `:720`, `:771`) persists |
| `UsageTracker::sessions` | `usage.rs:166` | "one entry per goose `sessionId` ever seen in this process" (`usage.rs:165`) — stated as a fact, not flagged |

`compact_expired_state`'s doc comment claims it exists "to prevent unbounded map growth" (`queue.rs:791`) but only touches `retry_after` and `retry_counts` (`queue.rs:809-817`), leaving four other maps uncompacted.

#### Invariants carried only in comments

| Invariant | Where it lives | Blast radius if violated |
|---|---|---|
| `requeue()` must run **before** `mark_complete()` | comment `queue.rs:426-427`, `:386-389` and `lib.rs:3061-3065` | every retry silently resets to attempt 1; exponential backoff and dead-lettering both stop working, with no error and no log |
| `mark_native_steer_pending` must be called synchronously after `send_steer` returns `Ok` and **before** the watcher spawns | comment `queue.rs:667-671`, `pool.rs:636-640` | reopens the `mark_complete` → `flush_next` double-delivery race the side table exists to close |
| `in_flight_deadline` must be strictly greater than `max_turn_duration` | comment `queue.rs:167-169` | a turn running to the hard cap gets its channel released early and re-dispatched while still in flight. Guarded only by `default_in_flight_deadline_exceeds_default_max_turn_duration` (`queue.rs:4530`), which checks the **defaults**, not a configured value |
| Native-steer and cancel+merge framing must not diverge | comment `queue.rs:1618-1621`, `lib.rs:2812-2814` | shared via `native_steer_framing()` + `format_event_block`, but nothing asserts it |
| `drain_channel` must preserve **both** `in_flight_channels` and `in_flight_deadlines` | comment `queue.rs:636-641` | removing deadlines alone disables auto-expiry and permanently wedges the channel |

None of these is enforced by a type, a `debug_assert!`, or a builder that makes the wrong order unrepresentable.

#### Dead / vestigial code

| Item | Line | Note |
|---|---|---|
| `UsageUpdatePayload::used` | `usage.rs:78-81` | `#[allow(dead_code)]`; parsed, tested for wire compat, never read |
| `UsageUpdatePayload::context_limit` | `usage.rs:82-85` | same |
| `MatchedRule::rule_index` | `filter.rs:152-153` | `#[cfg_attr(not(test), allow(dead_code))]`; both call sites discard it (`lib.rs:2175`, `setup_mode.rs:444-451`) |
| `UsageTracker::take` | `usage.rs:303-304` | carries `#[cfg_attr(not(test), allow(dead_code))]` **despite** a live production caller at `acp.rs:784` — a stale attribute that suppresses genuine dead-code signal |
| `pub use usage::TurnUsage` | `lib.rs:15` | the crate's only public re-export, with no external consumer: `sprig` calls `buzz_acp::run()` only (`sprig/src/main.rs:17`) |
| `EventQueue::pending_channels` | `queue.rs:594-596` | production callers are log-field-only (`lib.rs:2910`, `:2975`) |

#### `unwrap` / `expect` in production paths

`AGENTS.md` forbids introducing new `unwrap()`/`expect()` in production paths. Current count in these files outside `#[cfg(test)]`:

| Site | Line | Justification present? |
|---|---|---|
| `q.front().unwrap()` | `queue.rs:299` | no comment; safe only because the enclosing `filter` excludes empty queues (`queue.rs:295`) |
| `.expect("position came from iter so remove must succeed")` | `queue.rs:681-682` | yes, in the expect message |

`filter.rs` and `usage.rs` have zero. The remaining 13 occurrences in `queue.rs`'s non-test half are `unwrap_or*` combinators, which are not in scope of the rule.

#### Documentation drift

| Claim | Reality | Evidence |
|---|---|---|
| `ARCHITECTURE.md`: `queue.rs` = 2,565 LOC | 4,759 | `ARCHITECTURE.md:661` |
| `ARCHITECTURE.md`: `filter.rs` = 814 LOC | 787 | `ARCHITECTURE.md:666` |
| `ARCHITECTURE.md`: `main.rs` = 2,457 LOC, "Event loop, pool orchestration, heartbeat" | `main.rs` is **3 lines** (`fn main() { buzz_acp::run() }`); the event loop lives in `lib.rs` (6,570) | `ARCHITECTURE.md:662`, `crates/buzz-acp/src/main.rs` |
| `ARCHITECTURE.md`: `relay.rs` = 3,143, `pool.rs` = 2,253, `config.rs` = 1,903, `acp.rs` = 1,785 | 6,233 / 5,620 / 2,709 / 3,717 | `ARCHITECTURE.md:660`, `:663-665` |
| `ARCHITECTURE.md` module table | omits `usage.rs`, `observer.rs`, `setup_mode.rs`, `engram_fetch.rs`, `pool_lifecycle.rs` entirely | `ARCHITECTURE.md:656-666` |
| `queue.rs` module header: "**Drop** (default)" | CLI default is `queue` (`config.rs:344`); only `impl Default for EventQueue` uses `Drop`, and that impl is test-only | `queue.rs:11-14` vs `config.rs:344`, `queue.rs:822-824` |
| `queue.rs:163`, `:702`, `:765`: "`requeue_preserve_timestamps` at line 453" | it is at line 508 | `queue.rs:508` |
| `queue.rs:654`: "the contiguous drain at line 285" | the drain is at 336-345 | `queue.rs:336` |
| `lib.rs:2846`: "`EventQueue::mark_native_steer_pending` docs at queue.rs:606" | it is at 673 (line 606 is inside `set_retry_count_for_test`'s doc comment) | `queue.rs:673`, `:604-612` |
| `queue.rs:848`: "see relay messages.rs:762-783" | cross-crate line pointer into `buzz-relay` with no verification mechanism | `queue.rs:846-848` |
| `EventQueue` state-machine doc block | describes `push`/`flush_next`/`mark_complete`/`requeue` but omits `cancelled_batches`, `cancel_reasons`, `withheld_native_steer`, `in_flight_batch_sizes` and the cancelled-only fallback path — the doc is a snapshot of an earlier design | `queue.rs:94-136` vs fields `:151-166` |
| `crates/buzz-acp/README.md` | documents subscription modes, `--kinds`, and the mention filter but never mentions dedup modes, retry/dead-letter behaviour, `MAX_RETRIES`, or the cancel/steer merge framing | `README.md:118-249` |

Five stale in-code line-number references (`queue.rs:163`, `:654`, `:702`, `:765`, `lib.rs:2846`) is a systemic pattern: the codebase uses `file:line` in comments as a cross-reference idiom with no tooling to keep them accurate.

#### Design-level debt

- **Five parallel maps as implicit state.** A channel's state is the conjunction of memberships across `queues` / `in_flight_channels` / `in_flight_deadlines` / `retry_after` / `retry_counts` / `cancelled_batches` / `withheld_native_steer`, so illegal combinations (throttled *and* in flight *and* withheld) are representable and only prevented by call discipline.
- **Cancelled-batch fallback bypasses the retry throttle.** `flush_next`'s fallback (`queue.rs:308-312`) and `has_flushable_work`'s final clause (`queue.rs:586-590`) check only `in_flight_channels`, not `retry_after` — a backed-off channel's cancelled batch can flush during its own backoff window. Nothing tests this interaction.
- **Non-deterministic tie-breaking.** Both the fair-pick `min_by_key` (`queue.rs:299`) and the cancelled fallback `keys().find` (`queue.rs:308-312`) iterate a `HashMap` with no secondary sort key, so behaviour under equal `received_at` (or multiple cancelled channels) is unspecified.
- **`requeue_preserve_timestamps` has no retry budget.** It never increments `retry_counts` and never sets `retry_after` (`queue.rs:508-529`), so a persistently exhausted pool re-queues the same batch indefinitely with no dead-letter and no user-visible failure.
- **No prompt size budget.** `format_prompt` can emit an arbitrarily large prompt (unbounded `content`, unbounded tags, unbounded `cancelled_events`, unbounded `mentioned_pubkeys`); the only trimmer in the crate operates on the observer telemetry copy, not the outbound prompt (`lib.rs:659`).
