## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Business Rules

---

### 1. Definition validation (`WorkflowDef::validate`, `schema.rs:152-229`)

| Rule | Enforcement | Line | Trigger |
|---|---|---|---|
| `name` must be non-empty after `trim()` | `InvalidDefinition("name is required and must not be empty")` | `schema.rs:153-157` | every parse via `parse_yaml` |
| At least one step | `InvalidDefinition("at least one step is required")` | `schema.rs:159-163` | every parse |
| Step id non-empty after trim | `InvalidDefinition("step id must not be empty")` | `schema.rs:176-180` | per step |
| Step id ≤ 64 chars, only `[A-Za-z0-9_]` | closure `valid_step_id` (`schema.rs:168-172`); error names the offending id | `schema.rs:181-186` | per step — protects evalexpr variable names (`steps_{id}_output_{field}`) |
| Step ids unique | `HashSet` insert; `InvalidDefinition("duplicate step id: …")` | `schema.rs:187-192` | per step |
| `schedule` requires `cron` **or** `interval` | `InvalidDefinition("schedule trigger requires either 'cron' or 'interval'")` | `schema.rs:196-200` | schedule triggers only |
| `schedule` forbids both `cron` and `interval` | `InvalidDefinition("… cannot specify both …")` | `schema.rs:202-206` | schedule triggers only |
| `cron` must parse after normalization | `validate_cron` → `cron::Schedule` parse | `schema.rs:208-210`, `schema.rs:237-243` | schedule with cron |
| `interval` must parse and be ≥ 60 s | `parse_duration_secs`; `InvalidDefinition("interval must be at least 60s (cron loop ticks every 60 seconds)")` at `schema.rs:220-224` | `schema.rs:212-225` | schedule with interval |

Cron normalization: 5 fields → `0 {expr} *`; 6 fields → `{expr} *`; otherwise unchanged (`schema.rs:250-257`). Nothing else is validated at definition time — notably `call_webhook.url` scheme/host, `send_dm.to`, `add_reaction.emoji`, `delay.duration`, and `request_approval.timeout` are **not** checked by `validate()`.

---

### 2. Trigger matching semantics

Kind gate (`trigger_matches_event`, `lib.rs:955-964`):

| Trigger | Fires only on kind | Constant value |
|---|---|---|
| `message_posted` | `KIND_STREAM_MESSAGE` | 9 (`crates/buzz-core/src/kind.rs:343`) |
| `reaction_added` | `KIND_REACTION` | 7 (`crates/buzz-core/src/kind.rs:58`) |
| `diff_posted` | `KIND_STREAM_MESSAGE_DIFF` | 40008 (`crates/buzz-core/src/kind.rs:357`) |
| `schedule`, `webhook` | never from events (`false`) | `lib.rs:962` |

Pre-conditions inside `on_event` (`lib.rs:276-383`), evaluated in order:
1. Event must carry `channel_id`, else skip (`lib.rs:281-289`).
2. `is_workflow_execution_kind(kind)` (46001–46012) ⇒ skip (`lib.rs:293-295`).
3. Enabled-workflow list for `(community_id, channel_id)` from the 10 s moka cache, else DB (`lib.rs:297-311`); empty ⇒ return (`lib.rs:313-315`).
4. Definition JSON must deserialize; parse failure logs a warning and skips that workflow (`lib.rs:331-337`).
5. `def.enabled` must be true **and** the kind must match (`lib.rs:335`).
6. `should_fire_workflow` gate (below).
7. Run row created; execution spawned as a detached tokio task (`lib.rs:344-381`).

`should_fire_workflow` (`lib.rs:806-882`):
- `reaction_added` with `emoji: Some(e)`: exact string equality against `trigger_ctx.emoji` — no normalization, no shortcode↔unicode mapping (`lib.rs:807-822`). `emoji: None` matches any reaction.
- `message_posted` with `filter`: `evaluate_condition(expr, ctx, &HashMap::new())`; `false` ⇒ skip, `Err` ⇒ skip with a warning (fail-closed) (`lib.rs:824-846`).
- `diff_posted` with `filter`: identical logic, duplicated block (`lib.rs:848-880`).
- Note: step outputs are always empty at trigger-filter time, so `steps_*` variables are unavailable in filters.

`TriggerContext` construction (`build_trigger_context`, `lib.rs:884-953`):
- `text` = event content (`lib.rs:886`, `lib.rs:945`).
- `author` = first tag whose kind stringifies to `actor`, else `event.pubkey` hex (`lib.rs:888-897`).
- `emoji` = content when kind == `KIND_REACTION`, else empty (`lib.rs:901-905`).
- `message_id` = for reactions, the **last** `e` tag whose value is 64 ASCII-hex chars, falling back to the reaction's own id; for other kinds, the event's own id (`lib.rs:910-938`).
- `channel_id` = event channel UUID string or empty (`lib.rs:947-950`); `timestamp` = `created_at` seconds as string; `webhook_fields` always empty on this path (`lib.rs:952`).

---

### 3. Condition evaluation

| Rule | Detail | Line |
|---|---|---|
| Engine | `evalexpr` v11, `HashMapContext`, `eval_boolean_with_context` | `executor.rs:224-231`, `executor.rs:373` |
| Name mangling | dots → underscores: `trigger_text`, `trigger_author`, `trigger_channel_id`, `trigger_timestamp`, `trigger_emoji`, `trigger_message_id`; step outputs `steps_{step_id}_output_{field}` | `executor.rs:288-296`, `executor.rs:306-315` |
| Custom functions | exactly four: `str_contains(h, n)`→bool, `str_starts_with(s, p)`→bool, `str_ends_with(s, s2)`→bool, `str_len(s)`→int (byte length via `s.len()`) | `executor.rs:236-283` |
| Webhook field vars | registered as `trigger_{key}` **before** the standard fields so standard fields always win; keys already starting `trigger_`/`steps_` are skipped outright | `executor.rs:285-296` |
| Typing of step outputs | JSON string→String, bool→Boolean, i64→Int, other numbers→Float, null→Empty, arrays/objects→their JSON text as String | `executor.rs:318-335` |
| Expression length cap | `MAX_EXPR_LEN = 4096` bytes; longer ⇒ `ConditionError` before evaluation | `executor.rs:362-368` |
| Timeout | `EVAL_TIMEOUT = 100 ms`, applied via `tokio::time::timeout` around `spawn_blocking` | `executor.rs:342`, `executor.rs:370-380` |
| Timeout caveat (in-code) | the comment states the `spawn_blocking` thread is **not cancelled** by the timeout and runs to completion; the length cap is the actual mitigation | `executor.rs:358-361` |
| Failure semantics | eval error, task panic, or timeout all map to `ConditionError`; in `execute_steps` this **fails the run** (not "skip") | `executor.rs:375-383`, `executor.rs:1110-1118` |

---

### 4. Template variable resolution

| Rule | Detail | Line |
|---|---|---|
| Fast path | no `{{` in the string ⇒ returned unchanged | `executor.rs:78-80` |
| Single pass | one left-to-right scan; substituted values are **not** re-scanned, so `{{` inside a resolved value is never re-expanded | `executor.rs:82-121` |
| Supported paths | `trigger.<field>` (incl. any `webhook_fields` key via `get_field`'s fallback arm) and `steps.<ID>.output.<FIELD>` — exactly 3 dot segments after `steps.`, middle segment must literally be `output` | `executor.rs:126-151`, `executor.rs:49-59` |
| Depth | one level only: `json_get_str` reads a single key off a JSON **object**; nested paths and non-object outputs resolve to `None` | `executor.rs:154-163` |
| Unknown variable | the original `{{expr}}` is re-emitted verbatim, no error | `executor.rs:110-117` |
| Unclosed `{{` | emitted literally, remainder appended, `Ok` returned | `executor.rs:96-104` |
| Filters | `| truncate(N)` (char-wise `chars().take(n)`); `| npub` / `| truncate_pubkey` (hex pubkey → full bech32; non-pubkey passes through). Any other filter ⇒ `TemplateError` | `executor.rs:178-201` |
| `truncate` arg | must parse as `usize`, else `TemplateError` | `executor.rs:183-185` |
| JSON coercion | string as-is, bool/number via `to_string`, null → empty string, arrays/objects → JSON text | `executor.rs:165-174` |
| Which fields are templated | `send_message.text` + `.channel`; `send_dm.to` + `.text`; `set_channel_topic.topic`; `add_reaction.emoji`; `call_webhook.url` + header **values** + `.body`. **Not** templated: `call_webhook.method`, header **names**, `request_approval.timeout`, `delay.duration` | `executor.rs:406-452` |

---

### 5. Per-action semantics

| Action | Behaviour today | Validation performed | Line |
|---|---|---|---|
| `send_message` | Loads the run (`get_workflow_run`) then the workflow (`get_workflow`), both community-scoped; resolves the destination; hex-encodes `workflow.owner_pubkey` for attribution; calls `ActionSink::send_message`. Output `{sent, event_id}` | destination rules below; DB/lookup failure ⇒ `WebhookError`; missing sink ⇒ `InvalidDefinition` | `executor.rs:530-578` |
| `send_dm` | logs a warning, returns `WorkflowError::NotImplemented("SendDm")` — always fails the run | none | `executor.rs:580-584` |
| `set_channel_topic` | logs a warning, returns `WorkflowError::NotImplemented("SetChannelTopic")` | none | `executor.rs:586-590` |
| `add_reaction` | requires non-empty `trigger_ctx.message_id`; with feature `reqwest` performs `POST {BUZZ_RELAY_BASE_URL}/api/messages/{message_id}/reactions`; without the feature returns `{added:false, skipped:true}` | empty `message_id` ⇒ `InvalidDefinition` | `executor.rs:592-617`, `executor.rs:885-930` |
| `call_webhook` | with `reqwest`: SSRF check, DNS-pinned per-request client, redirects disabled, 10 s timeout, 1 MiB response cap, output `{status, body}`; without the feature returns `{status:0, body:null, skipped:true}` | URL must parse and have a host; method must be a valid HTTP method token | `executor.rs:619-648`, `executor.rs:781-866` |
| `request_approval` | logs, generates a v4 UUID token, returns `StepResult::Suspended` — **no DB record is created, no event is emitted** (`TODO (WF-08)` at `executor.rs:663`) | none; `timeout` string is neither parsed nor validated (default literal `"24h"` used only for the log line) | `executor.rs:650-668` |
| `delay` | parses the duration, rejects > `MAX_DELAY_SECS = 270`, then `tokio::time::sleep`; output `{slept_secs}` | `> 270 s` ⇒ `InvalidDefinition`; unparseable ⇒ `InvalidDefinition` | `executor.rs:671-690` |

`send_message` destination rules (`resolve_send_message_channel`, `executor.rs:468-517`):
1. Blank/whitespace `channel` overrides are treated as absent (`executor.rs:473-475`).
2. If the workflow row has `channel_id`: any override must parse as a UUID **and equal** that channel, else `InvalidDefinition("channel override must match the workflow channel …")`; the bound channel is always used (`executor.rs:477-493`).
3. Workflow with no bound channel: a valid UUID override is used (`executor.rs:495-503`).
4. Otherwise the trigger's channel is used; if that is blank ⇒ `InvalidDefinition("no channel_id available …")` (`executor.rs:505-513`).

---

### 6. Step sequencing and failure handling (`execute_steps`, `executor.rs:1080-1214`)

1. Steps run strictly in definition order; indices `< start_index` are skipped with a debug log (`executor.rs:1088-1091`).
2. `if:` false ⇒ trace `{"status":"skipped"}` and continue; `if:` error ⇒ return `(ConditionError, PartialProgress{step_index: i, trace})` (`executor.rs:1093-1121`).
3. Template resolution error ⇒ return with partial progress (`executor.rs:1123-1132`).
4. Per-step timeout = `step.timeout_secs` or `engine.config.default_timeout_secs` (300 s default), wrapped around `dispatch_action`; expiry ⇒ `StepTimeout { step_id, timeout_secs }` (`executor.rs:1133-1174`).
5. Any action error aborts the whole run — there is no per-step retry, no continue-on-error, and no compensation (`executor.rs:1157-1164`).
6. `Completed` ⇒ trace entry with output and `step_outputs.insert(step.id, output)` (`executor.rs:1176-1184`).
7. `Suspended` ⇒ returns immediately with `approval_token: Some(_)`, `step_index: i`, accumulated outputs and trace (`executor.rs:1185-1196`).
8. `Skipped` ⇒ trace entry (unreachable: `dispatch_action` never returns `StepResult::Skipped`) (`executor.rs:1197-1203`).
9. Normal end ⇒ `ExecutionResult { approval_token: None, step_index: def.steps.len(), … }` (`executor.rs:1205-1213`).

---

### 7. Run status state machine

Only three statuses are ever written by this crate.

```
                     create_workflow_run (relay / on_event / cron)
                                 │  status = Pending (buzz-db default)
                                 ▼
            ┌──────────── try_acquire() permit ────────────┐
            │ fail                                          │ ok
            ▼                                               ▼
   CapacityExceeded error                            update_workflow_run(Running)
   (status stays Pending;                       executor.rs:982-991 / :1044-1053
    finalize_run then writes Failed)                        │
                                                            ▼
                                                    execute_steps loop
                                       ┌────────────┬───────┴────────────┐
                                       │ Ok, token  │ Ok, no token       │ Err(e, progress)
                                       ▼            ▼                    ▼
                                    Failed      Completed             Failed
                        "approval gates not   lib.rs:218-238     error = e.to_string()
                         yet implemented —                       lib.rs:242-261
                         see WF-08"
                         lib.rs:192-215
```

`WaitingApproval`, `Pending` (explicitly) and `Cancelled` are never written by this crate — repo-wide grep finds `WaitingApproval` only in the comment at `lib.rs:193` and in two relay guards (`crates/buzz-relay/src/handlers/command_executor.rs:1197`, `:1253`).

`finalize_run` is the single mapping point and prepends `existing_trace` when supplied (`lib.rs:180`, `lib.rs:218-221`, `lib.rs:242-245`). DB update failures are logged, not propagated (`lib.rs:207-213`, `lib.rs:231-236`, `lib.rs:256-260`).

---

### 8. Approval gate lifecycle (as implemented)

| Step | Reality | Line |
|---|---|---|
| Token generation | `Uuid::new_v4().to_string()`; `run_id`/`step_id` args deliberately unused | `executor.rs:698-700` |
| Persistence | none — `TODO (WF-08): create approval record in DB, emit kind:46010` | `executor.rs:663` |
| Executor return | `StepResult::Suspended { approval_token }`, bubbled up as `ExecutionResult.approval_token` | `executor.rs:665-667`, `executor.rs:1185-1196` |
| Finalization | `finalize_run` logs `"Workflow hit approval gate — not yet implemented, marking as failed"` and writes `RunStatus::Failed` with error text `"approval gates not yet implemented — see WF-08"` | `lib.rs:192-215` |
| Resume | `execute_from_step` supports resumption (start index + `initial_outputs` + trace preservation) and the relay calls it, but the relay's `resume_workflow` refuses unless `run.status == WaitingApproval` — a status this codebase never writes | `executor.rs:1015-1072`, `crates/buzz-relay/src/handlers/command_executor.rs:1244-1320` |

---

### 9. Concurrency admission

| Rule | Detail | Line |
|---|---|---|
| Mechanism | `Arc<Semaphore>` created with `config.max_concurrent.max(1)` permits (default 100) | `lib.rs:68`, `lib.rs:110-111` |
| Admission | `try_acquire()` — non-blocking, no queueing; failure ⇒ `WorkflowError::CapacityExceeded` with an empty `PartialProgress` | `executor.rs:975-980`, `executor.rs:1025-1030` |
| Scope | permit held for the whole run including `delay` sleeps and step timeouts (`_permit` binding lives to end of fn) | `executor.rs:975`, `executor.rs:1025` |
| Rejected runs | `finalize_run` records them as `Failed` with message `"capacity exceeded"` | `lib.rs:242-261`, `error.rs:52-54` |

---

### 10. Delay bounds and duration parsing

- `MAX_DELAY_SECS = 270` (`executor.rs:676`), justified in-code as "must be less than default_timeout_secs (300s)" (`executor.rs:673-675`). Exceeding it ⇒ `InvalidDefinition`, i.e. run failure at execution time, not at parse time.
- `parse_duration_secs` (`executor.rs:705-735`): suffix `h` → `×3600` checked, `m` → `×60` checked, `s` → as-is, bare number → seconds; any other form ⇒ `InvalidDefinition`. Overflow ⇒ `InvalidDefinition("duration overflow: …")`. Fractional values (`"1.5h"`) are rejected.

---

### 11. Cron / interval matching (`run`, `lib.rs:428-672`)

| Rule | Detail | Line |
|---|---|---|
| Tick | `tokio::time::sleep(60 s)` **first**, then work — nothing fires in the first minute after startup | `lib.rs:430-432` |
| Scan | `list_all_enabled_workflows()` each tick (no community filter; each row carries its own `community_id`) | `lib.rs:436-442` |
| Definition parse failure | warn + skip that workflow | `lib.rs:449-459` |
| Disabled defs | `def.enabled == false` ⇒ skip | `lib.rs:461-463` |
| Missing channel | schedule workflow with `channel_id == None` ⇒ warn + skip ("Fix 2") | `lib.rs:465-472` |
| Trigger dispatch | `match &def.trigger` computing `(scheduled_for, trigger_type)`; cron arm `lib.rs:481-486`, interval arm `lib.rs:487-536` | `lib.rs:480-538` |
| Cron match | `cron_fire_instant(expr, now, 60, id)`: `schedule.after(now - 60s).next()` filtered to `<= now`; returns the **scheduled** instant so all pods agree; invalid expression ⇒ warn + `None` | `lib.rs:484`, `lib.rs:688-706` |
| Interval prefilter | in-memory `last_fired`, falling back to `latest_scheduled_workflow_fire`; DB read failure ⇒ `None` ⇒ suppress this tick (fail-closed) with a warning | `lib.rs:498-517` |
| Interval elapse test | `elapsed = (now - last).num_seconds().unsigned_abs() >= interval_secs`; `last = None` treated as `now` ⇒ no immediate fire | `lib.rs:753-774` |
| Cold-start seed | on the `None` suppress path the anchor is seeded to `now` so the next tick has a real anchor; an existing `Some` anchor is never advanced on suppress | `lib.rs:784-797` |
| Interval instant | `floor(now.timestamp() / interval) * interval` bucket boundary (`div_euclid`) so pods collide on one claim; zero/unparseable ⇒ warn + skip | `lib.rs:532`, `lib.rs:719-745` |
| Durable claim | `claim_scheduled_workflow_fire(community_id, workflow_id, scheduled_for)`; `Ok(None)` ⇒ another pod won, skip (interval anchors still advanced at `lib.rs:559`); `Err` ⇒ log + skip | `lib.rs:547-570` |
| Trigger-context serialization | built at `lib.rs:574-578`, serialization failure ⇒ error log + skip ("Fix 5") | `lib.rs:572-589` |
| Claim/run failure policy | if run creation fails after a won claim, the claim row is intentionally kept — at-most-once beats exactly-once | `lib.rs:590-611` |
| Anchor write ordering | `last_fired` updated only after claim **and** run insert succeed | `lib.rs:630-636` |
| Map hygiene | `last_fired.retain(...)` prunes entries whose `(community_id, id)` is no longer in the enabled scan ("Fix 1") | `lib.rs:664-670` |
| Non-schedule triggers | `_ => continue` — event triggers are never handled here | `lib.rs:537` |

---

### 12. Tenant scoping rules

Every DB call and side effect carries the server-resolved `CommunityId`: workflow lookup (`lib.rs:301-306`), run creation (`lib.rs:346`, `lib.rs:592`), status updates (`executor.rs:984`, `lib.rs:201`), `send_message` metadata lookups (`executor.rs:535-556`), and the sink call itself (`executor.rs:569-571`). The cron loop uses the workflow row's own `community_id` (`lib.rs:455`) rather than a deployment default. `last_fired` and `workflow_cache` are both keyed `(CommunityId, Uuid)` (`lib.rs:87`, `lib.rs:104`).
