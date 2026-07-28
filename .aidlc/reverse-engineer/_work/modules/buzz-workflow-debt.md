## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Technical Debt

---

### 1. Incomplete features carrying explicit ticket tags

| Item | State | Ticket | Line |
|---|---|---|---|
| `send_dm` | returns `NotImplemented`, fails the run | WF-07 | `executor.rs:580-584` |
| `set_channel_topic` | returns `NotImplemented`, fails the run | WF-07 | `executor.rs:586-590` |
| `request_approval` | token generated, nothing persisted, no kind-46010 emission; `finalize_run` turns the suspension into `Failed` | WF-08 | `executor.rs:650-668`, `lib.rs:192-215` |
| Long delays | capped at 270 s; "scheduled resume pattern" deferred | WF-09 | `executor.rs:673-684` |

---

### 2. Complexity hotspots

| Location | Size / shape | Notes |
|---|---|---|
| `WorkflowEngine::run` | `lib.rs:428-672` — ~245 lines, single `loop` with a nested per-workflow body, 5 sequential DB calls, two `continue`-heavy match arms, and inline `tokio::spawn` | The largest function in the crate. Cron vs interval branching, claim handling, trigger-context building, run creation, claim attachment, anchor updates, and map pruning are all inlined; only the two instant-computation helpers and the interval prefilter are extracted |
| `dispatch_action` | `executor.rs:519-692` — ~175 lines, 7 arms, two of which contain `#[cfg]`-split bodies | The `send_message` arm performs two DB round-trips inline (`executor.rs:535-556`); `#[cfg(feature = "reqwest")]` / `#[cfg(not(...))]` duplication in `add_reaction` and `call_webhook` doubles the surface to review |
| `on_event` | `lib.rs:276-383` — ~108 lines mixing cache management, per-workflow deserialization, gating, run creation and task spawning | |
| `should_fire_workflow` | `lib.rs:806-882` — the `MessagePosted` filter block (`:824-846`) and `DiffPosted` filter block (`:848-880`) are near-identical copies differing only in the matched variant | Straight duplication; a shared `filter_of(&TriggerDef) -> Option<&str>` would collapse both |
| `execute_steps` | `executor.rs:1080-1214` — five distinct early-return error paths, each rebuilding `PartialProgress { step_index: i, trace }` by moving `trace` | Repeated construct at `:1113-1117`, `:1126-1130`, `:1158-1162`, `:1166-1173` |
| `resolve_send_message_channel` | `executor.rs:468-517` — four-branch precedence ladder with two separate UUID parses producing the same error text | |
| `parse_duration_secs` | `executor.rs:705-735` — three near-identical suffix blocks | |
| `finalize_run` | `lib.rs:175-263` — three DB-write branches each with its own error log; approval branch is dead-weight until WF-08 lands | |
| File sizes | `executor.rs` 1834 LOC (≈1215 non-test, test module starts `executor.rs:1216`), `lib.rs` 1564 LOC (≈965 non-test, `lib.rs:966`), `schema.rs` 878 LOC (≈269 non-test, `schema.rs:270`) | Test code is roughly 40% of `lib.rs` and 70% of `schema.rs` |

---

### 3. Dead / unreachable code

| Item | Why it is dead | Line |
|---|---|---|
| `StepResult::Skipped` | `dispatch_action` never constructs it (it returns only `Completed` or `Suspended`), so the handling arm in `execute_steps` is unreachable. Condition-based skipping is handled earlier by `continue`, not by this variant | `executor.rs:463-464`, `executor.rs:1197-1203`, `executor.rs:1093-1109` |
| `finalize_run` approval branch | Reachable only if a run suspends, and its sole purpose is to convert that into `Failed` — i.e. the approval feature's "success" path is a failure path | `lib.rs:192-215` |
| `ExecutionResult.step_outputs` | Populated on every return but never read by any caller in the repo (relay call sites pass the whole `Result` to `finalize_run`, which reads only `trace`, `step_index`, `approval_token`) | `executor.rs:946`, `lib.rs:186-238` |
| `execute_run` | No caller outside this crate; both internal callers are in `lib.rs`. Meanwhile all three relay entry points use `execute_from_step(..., 0, None)` instead, duplicating `execute_run`'s behaviour | `executor.rs:967`, `lib.rs:373`, `lib.rs:651`; relay: `crates/buzz-relay/src/handlers/command_executor.rs:926`, `crates/buzz-relay/src/api/bridge.rs:1890` |
| `build_trigger_context` (`pub`) | No caller outside this crate | `lib.rs:884` |
| `resolve_template`, `resolve_step_templates`, `build_eval_context`, `evaluate_condition`, `dispatch_action` (`pub`) | All public, none called outside this crate | `executor.rs:70`, `:224`, `:350`, `:390`, `:519` |
| `generate_approval_token` parameters | `_run_id` and `_step_id` are accepted and deliberately unused | `executor.rs:698-700` |
| Approval-resume machinery end-to-end | `execute_from_step`'s `start_index`/`initial_outputs` path is exercised only with `(0, None)`. The relay's `resume_workflow` gates on `run.status == WaitingApproval` (`crates/buzz-relay/src/handlers/command_executor.rs:1253`), and **no code in the repository ever writes `WaitingApproval`** — grep finds it only in that guard, a sibling guard at `:1197`, and the comment at `lib.rs:193`. `Db::create_approval` likewise has no caller outside `buzz-db` | `executor.rs:1015-1072` |
| `add_reaction_impl` target route | Posts to `/api/messages/{id}/reactions`, which the relay router does not register (`crates/buzz-relay/src/router.rs:39-125`) — the code is live but cannot succeed | `executor.rs:885-891` |

---

### 4. Stale documentation inside the crate

| Claim | Reality | Line |
|---|---|---|
| "Action dispatch uses placeholder implementations that log intent. Real event emission is wired in WF-07/08 (relay integration)." | `send_message` emits real events through `ActionSink`; `call_webhook` performs real HTTP | `executor.rs:9-10` |
| "For MVP, most actions log their intent and return a success output." | Only `send_dm`/`set_channel_topic` are log-and-fail stubs | `executor.rs:521-522` |
| `CallWebhook.url` — "must be a public HTTPS endpoint" | No scheme validation; `http://` accepted | `schema.rs:120` vs `executor.rs:786-798` |
| `RequestApproval.timeout` — "Defaults to 24h" | Never parsed; used only in a log line | `schema.rs:138-140` vs `executor.rs:653-658` |
| Crate-root usage example shows `engine.on_event(...)` then `engine.run()` | Accurate, but omits the mandatory `set_action_sink` call without which `send_message` fails | `lib.rs:19-31` vs `lib.rs:148-156` |
| `Db::create_workflow` doc "(No current callers.)" | Confirms the workflow creation path is `upsert_workflow` only | `crates/buzz-db/src/workflow.rs:272-275` |

---

### 5. Test coverage gaps

149 tests total (127 `#[test]`, 22 `#[tokio::test]`) split `schema.rs` 50, `lib.rs` 38, `executor.rs` 61. All are pure-function tests. Untested:

| Untested surface | Reason | Line |
|---|---|---|
| `WorkflowEngine::on_event` | needs a `Db` (Postgres) | `lib.rs:276` |
| `WorkflowEngine::run` (scheduler loop) | needs `Db` + a 60 s tick; only the extracted helpers `cron_fire_instant`, `interval_fire_instant`, `interval_should_fire`, `interval_prefilter_should_fire` are covered | `lib.rs:428` |
| `WorkflowEngine::finalize_run` | needs `Db`; the approval→`Failed` mapping — the single most surprising behaviour in the crate — has no test | `lib.rs:175` |
| `execute_run` / `execute_from_step` / `execute_steps` | need `Db`; therefore step sequencing, per-step timeout, `StepTimeout`, partial-progress trace accumulation, resume-from-index, and semaphore admission (`CapacityExceeded`) are all untested | `executor.rs:967`, `:1015`, `:1080` |
| `dispatch_action` (all 7 arms) | needs `WorkflowEngine`; only the extracted `resolve_send_message_channel` has 3 tests (`executor.rs:1801-1833`) | `executor.rs:519` |
| `check_ssrf`, `call_webhook_impl`, `add_reaction_impl` | no HTTP test server, no `is_private_ip` integration test in this crate | `executor.rs:745`, `:781`, `:885` |
| `WorkflowEngine::new` / `set_action_sink` / `invalidate_channel_workflows` | no test constructs an engine, so the double-init panic and cache invalidation are unverified here | `lib.rs:109`, `:139`, `:131` |
| `MAX_DELAY_SECS` boundary | no test asserts that a 271 s delay is rejected or that 270 s is accepted | `executor.rs:676` |
| `WorkflowError` display strings / `From<DbError>` | `error.rs` has zero tests | `error.rs` |
| `ActionSink` / `ActionSinkError` | `action_sink.rs` has zero tests | `action_sink.rs` |
| Trigger-context `webhook_fields` shadowing defence | the "webhook keys cannot spoof `trigger_*`" skip logic (`executor.rs:287-291`) has no negative test; only the positive case is tested (`executor.rs:1700-1708`) | `executor.rs:285-296` |
| `evaluate_condition` timeout path | `MAX_EXPR_LEN` rejection is tested (`executor.rs:1392-1409`), but the 100 ms timeout branch is not | `executor.rs:370-380` |

Relay-side integration coverage exists but explicitly documents the approval gap — see `crates/buzz-test-client/tests/conformance_multitenant.rs:1863-1946`, which states the approval-request half is unreachable ("see WF-08") and that `create_approval` is reached only from unit tests.

---

### 6. Design debt / operational risk

| Item | Detail | Line |
|---|---|---|
| Non-cancellable blocking evaluation | The code documents that `spawn_blocking` work survives its own timeout; the mitigation is a length cap, not cancellation. Enough concurrent pathological expressions could saturate the blocking pool | `executor.rs:355-380` |
| Permit held across sleeps | A `delay` step holds a concurrency permit for up to 270 s, so ~100 delaying runs exhaust admission and new runs are rejected with `CapacityExceeded` (which finalizes as `Failed`, not retried) | `executor.rs:975`, `executor.rs:685`, `lib.rs:242-261` |
| No retry / backoff anywhere | Any step error terminates the run permanently; there is no dead-letter, no re-drive, and no idempotency key for side effects | `executor.rs:1157-1173` |
| Fire-and-forget spawns | Runs are spawned detached with no `JoinHandle` tracking, so a relay shutdown mid-run leaves the row in `Running` forever — there is no reaper for stuck `Running` rows in this crate | `lib.rs:371-381`, `lib.rs:649-661` |
| Interval anchors are in-memory | `last_fired` is a `DashMap`, documented as "lost on restart. Missed fires during downtime are not replayed (acceptable for MVP)"; the durable claim row is the only cross-restart guard | `lib.rs:84-86`, `lib.rs:494-520` |
| Cron scan is unbounded | `list_all_enabled_workflows()` loads every enabled workflow in every community each tick, then filters to schedule triggers in-process | `lib.rs:436` |
| Cache staleness accepted by design | No cross-pod invalidation; the documented worst case is a just-deleted workflow firing (or a new one missing events) for up to 10 s | `lib.rs:92-103` |
| Trace can grow to megabytes | Each `call_webhook` step can append up to 1 MiB of response body into `execution_trace`, and the full array is rewritten on every status update | `executor.rs:862-865`, `executor.rs:1176-1180` |
| Two HTTP client strategies | Per-request client for webhooks (needed for DNS pinning) vs a `LazyLock` shared client for reactions — inconsistent, and the shared one lacks the SSRF/redirect/cap hardening | `executor.rs:800-812` vs `executor.rs:871-882` |
| Duplicated entry points | `execute_run` and `execute_from_step(…, 0, None)` are behaviourally equivalent for fresh runs but differ in trace handling (the latter reads the existing trace first, `executor.rs:1034-1042`), and the relay uses only the latter | `executor.rs:967`, `:1015` |
| Direct (non-workspace) dependency versions | `evalexpr = "11"` and `cron = "0.16"` are pinned in the crate rather than the workspace table, unlike every other dependency | `Cargo.toml:20-21` |
| No deprecated API usage found | Grep found no `#[deprecated]`, no `#[allow(deprecated)]`, and no deprecation warnings referenced in the crate | — |
