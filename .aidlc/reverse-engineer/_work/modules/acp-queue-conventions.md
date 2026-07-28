## Module: buzz-acp — work queue, filtering & usage accounting (`crates/buzz-acp/src`)
### Aspect: Conventions

#### Naming

| Pattern | Examples |
|---|---|
| Verb-first mutators on `EventQueue` | `push`, `flush_next`, `mark_complete`, `requeue`, `drain_channel`, `compact_expired_state` (`queue.rs:230`, `:260`, `:392`, `:429`, `:625`, `:807`) |
| `_for_test` suffix on `#[cfg(test)]` accessors used by *other* modules' tests | `set_retry_count_for_test` (`queue.rs:610`), `queued_event_count` (`queue.rs:600`) |
| `MAX_*` for hard caps, `BASE_*`/`DEFAULT_*` for tunable-by-code values | `MAX_PENDING_PER_CHANNEL`, `MAX_BATCH_EVENTS`, `MAX_RETRIES`, `MAX_RETRY_DELAY_SECS`, `MAX_EXPR_LEN`, `MAX_CONSECUTIVE_TIMEOUTS`, `MAX_CONCURRENT_FILTER_EVALS`, `MAX_PROMPT_LABEL_LEN` vs `BASE_RETRY_DELAY_SECS`, `DEFAULT_IN_FLIGHT_DEADLINE_SECS` (`queue.rs:24-42`, `:1023`, `filter.rs:162-341`) |
| `format_*` for pure string builders, `resolve_*` for lookups that can fail, `parse_*`/`extract_*` for input → structure | `format_prompt`, `format_event_block`, `format_context_hints`, `format_conversation_context`, `format_prompt_actor` / `resolve_prompt_label`, `resolve_reply_anchor` / `parse_thread_tags`, `extract_slash_command` |
| Builder-style consuming setter | `with_in_flight_deadline(self) -> Self` (`queue.rs:197`) — the only builder method; everything else is `&mut self` |
| `Args` struct instead of long parameter lists | `FormatPromptArgs<'a>` with `#[derive(Default)]` so tests set only what they need (`queue.rs:1352-1375`) |

#### Error handling

- `filter.rs` is the only module with an error type: `FilterError` via `thiserror` with three variants (`filter.rs:14-24`). `evaluate_filter` returns `Result`; `match_event` swallows all errors into `Option::None` after logging (`filter.rs:426-448`).
- `queue.rs` and `usage.rs` define **no** error types. Every fallible situation is expressed as `Option`, `bool`, or a `tracing` log:
  - `push` → `bool` (`queue.rs:230`)
  - `flush_next` / `requeue` / `slash_command_for_batch` / `extract_slash_command` / `UsageTracker::take` → `Option`
  - `mark_native_steer_pending` → `bool`; `release_native_steer` / `remove_event` → silent idempotent no-ops (`queue.rs:704-711`, `:739-750`)
  - `format_prompt` on an empty batch → `tracing::error!` + empty `Vec` (`queue.rs:1411-1417`)
- Log-level convention is consistent: `ERROR` for invariant violations (`"BUG: in-flight channel expired without mark_complete"` at `queue.rs:272-278` / `:568-574`, dead-letter at `queue.rs:439-445`, disabled filter rule at `filter.rs:406-412`); `WARN` for lossy-but-expected events (cap overflow at `queue.rs:245-249`, `:490-494`, `:723-728`, `:776-781`; requeue at `queue.rs:466-472`; withheld-steer recovery at `queue.rs:783-788`); `INFO` for deadline extension (`queue.rs:216-219`); `DEBUG` for drop-mode discards (`queue.rs:235-238`).
- `no unsafe` is enforced crate-wide by `#![deny(unsafe_code)]` (`lib.rs:1`); none of the three files contains `unsafe`.
- Production `unwrap`/`expect` in these files: `q.front().unwrap()` inside the `min_by_key` on a pre-filtered non-empty queue (`queue.rs:299`) and `q.remove(pos).expect("position came from iter so remove must succeed")` (`queue.rs:681-682`). Both are guarded by a preceding check. Everything else is `unwrap_or*`: `queue.rs:271`, `:317`, `:366`, `:459`, `:567`, `:630`, `:915`, `:939`, `:1083`, `:1087`, `:1192`, `:1222`, `:1422`. `filter.rs` and `usage.rs` have **zero** `unwrap`/`expect` outside tests.
- `filter.rs` timeouts fail closed by policy, documented three times in-code (`filter.rs:357-366`, `:413`, `:434`, `:444`).

#### Documentation style

- Every public item in all three files has a doc comment; several carry the invariant *and* its rationale (e.g. `mark_complete`'s retry-preservation contract at `queue.rs:386-391`, `requeue`'s "does NOT remove from `in_flight_channels`" at `queue.rs:426-427`).
- `EventQueue`'s state machine is documented as an ASCII pseudocode block in the type-level doc comment (`queue.rs:94-136`) — the only place the full transition table exists.
- `usage.rs` documents its three delta cases as a numbered list in the module header (`usage.rs:16-30`) and repeats the three `record` branches as a numbered list on the method (`usage.rs:198-210`).
- `filter.rs` documents the evalexpr variable surface as a markdown table inside the `build_eval_context` doc comment (`filter.rs:253-259`).
- Comments name individual reviewers as the source of a requirement — "per Dawn's framing review" (`queue.rs:1592`), "Eva's drift-proof requirement" (`queue.rs:1620`, `lib.rs:2813`), "Eva+Wren and Thufir both flagged" (`usage.rs:453`), "Regression for Thufir fix 2" (`usage.rs:507-508`). These carry design intent that exists nowhere else.

#### How state machines are expressed

There is no state-machine type or enum-based state. `EventQueue` encodes channel state implicitly across five parallel maps plus one set, and the "state" of a channel is the conjunction of its memberships:

| Channel state | Encoding |
|---|---|
| idle | absent from every map |
| pending | `queues[ch]` non-empty, not in `in_flight_channels` |
| in flight | `in_flight_channels` ∋ ch, `in_flight_deadlines[ch]` set |
| throttled | `retry_after[ch] > now` |
| retrying | `retry_counts[ch] > 0` |
| cancelled-pending | `cancelled_batches[ch]` non-empty |
| steer-withheld | `withheld_native_steer[ch]` non-empty |

Consequences: transitions are enforced by call-order discipline, not types (see the requeue-before-`mark_complete` contract at `lib.rs:3061-3065`), and the same expiry block is physically duplicated between `flush_next` (`queue.rs:263-287`) and `has_flushable_work` (`queue.rs:558-581`) rather than extracted.

`CancelReason` (`queue.rs:65-74`) and `MergeFraming` (`queue.rs:1571-1610`) are the one place a state-to-behavior mapping is expressed as an exhaustive `match` returning a data struct — `MergeFraming::for_reason` folds `None` into the `Steer` arm deliberately (`queue.rs:1586-1588`).

`UsageTracker` is a three-state machine expressed as `Option<String> in_flight_session` + `Option<TurnUsage> pending`, with the branch decided by an `is_in_flight` bool computed once (`usage.rs:219`, `:259-296`).

#### Test organization

| File | Tests | Location | Runtime |
|---|---|---|---|
| `queue.rs` | 112 | inline `#[cfg(test)] mod tests` at `queue.rs:1628-4759` (3,132 lines — 66 % of the file) | all sync `#[test]` |
| `filter.rs` | 15 | inline at `filter.rs:462-787` | 11 `#[tokio::test]`, 4 `#[test]` |
| `usage.rs` | 20 | inline at `usage.rs:322-891` | all sync `#[test]` |

Conventions inside the test modules:

- Shared fixture helpers at the top of each module: `make_event` / `make_queued` / `make_queued_at` / `make_queued_created_at` / `make_event_with_tags` / `make_merged_batch` / `make_single_batch` / `pending_count` / `any_in_flight` (`queue.rs:1635-1690`, `:1897`, `:2885`, `:4160`); `make_event` / `make_event_with_p_tag` / `any_channel` / `make_rule` (`filter.rs:469-510`); `payload` / `payload_no_context` / `payload_with_model` (`usage.rs:326-349`, `:828`).
- `queue.rs` and `filter.rs` prefix every test `test_`; `usage.rs` does **not** (e.g. `first_turn_no_prior_delta_unreliable`, `usage.rs:461`) — an inconsistency between sibling files in the same crate.
- Test helpers reach **private fields directly** (`q.queues`, `q.in_flight_channels` at `queue.rs:1684`, `:1688`; `tracker.pending` at `usage.rs:360-363`) rather than through the public API, which is why some public accessors have no test caller.
- `usage.rs` groups tests with box-drawing section banners (`usage.rs:348`, `:458`, `:586`, `:662`, `:826`).
- Regression tests carry the original defect in the comment body rather than the name — `cross_session_notification_does_not_corrupt_other_sessions_delta` explains the exact undercount it prevents (`usage.rs:404-460`).
- Float comparisons use explicit epsilons (`(dc - 0.007).abs() < 1e-9`, `usage.rs:385`), never `assert_eq!` on `f64`.
- Zero `TODO` / `FIXME` / `HACK` / `XXX` markers across all three files.
