## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: API Surface

Crate lints: `#![deny(unsafe_code)]`, `#![warn(missing_docs)]` (`lib.rs:1-2`). Public modules: `action_sink`, `error`, `executor`, `schema` (`lib.rs:33-36`). Re-exports: `ActionSink`, `ActionSinkError`, `PartialProgress`, `WorkflowError`, `ExecutionResult`, `ActionDef`, `Step`, `TriggerDef`, `WorkflowDef` (`lib.rs:38-41`).

---

### `WorkflowEngine` (crate root)

| Fn | Signature | Purpose | Returns | Line |
|---|---|---|---|---|
| `new` | `fn new(db: Db, config: WorkflowConfig) -> Self` | Construct engine; semaphore permits = `max_concurrent.max(1)`; moka cache 10 000 entries / 10 s TTL | `Self` | `lib.rs:109-127` |
| `invalidate_channel_workflows` | `fn invalidate_channel_workflows(&self, community_id: CommunityId, channel_id: Uuid)` | Drop the cached enabled-workflow list for a channel (same-pod only) | `()` | `lib.rs:131-133` |
| `set_action_sink` | `fn set_action_sink(&self, sink: Arc<dyn ActionSink>)` | Late-init the side-effect sink; **panics if called twice** | `()` | `lib.rs:139-143` |
| `action_sink` | `pub(crate) fn action_sink(&self) -> Result<&dyn ActionSink, WorkflowError>` | Accessor; `InvalidDefinition` error if not wired | `Result<&dyn ActionSink, _>` | `lib.rs:148-156` |
| `parse_yaml` | `fn parse_yaml(yaml: &str) -> Result<(WorkflowDef, String), WorkflowError>` (associated, no `self`) | Parse + validate + canonical JSON | `(WorkflowDef, String)` | `lib.rs:163-165` |
| `finalize_run` | `async fn finalize_run(&self, community_id: CommunityId, run_id: Uuid, result: Result<ExecutionResult, (WorkflowError, PartialProgress)>, existing_trace: Option<Vec<serde_json::Value>>)` | Single mapping point from executor result → DB run status; approval-gated results are written as `Failed` | `()` (logs DB errors) | `lib.rs:175-263` |
| `on_event` | `async fn on_event(self: &Arc<Self>, community_id: CommunityId, event: &buzz_core::StoredEvent) -> Result<(), WorkflowError>` | Post-store hook: channel workflow lookup, trigger match, run creation, spawn execution | `Result<(), WorkflowError>` | `lib.rs:276-383` |
| `interval_prefilter_should_fire` | `fn interval_prefilter_should_fire(&self, community_id, workflow_id: Uuid, dur: &str, last: Option<DateTime<Utc>>, now: DateTime<Utc>) -> bool` (private) | Interval prefilter + cold-start anchor seed | `bool` | `lib.rs:401-403` |
| `run` | `async fn run(self: &Arc<Self>)` | Scheduler entry point — infinite 60 s tick loop for `schedule` triggers | never returns | `lib.rs:428-672` |

Free functions in `lib.rs`:

| Fn | Visibility | Purpose | Returns | Line |
|---|---|---|---|---|
| `build_trigger_context(event: &StoredEvent)` | `pub` | Map a stored event to `TriggerContext` (author prefers `actor` tag, reaction target from last 64-hex `e` tag) | `executor::TriggerContext` | `lib.rs:884-953` |
| `cron_fire_instant(expr, now, window_secs, workflow_id)` | private | Window-based cron match; returns the cron's *scheduled* instant | `Option<DateTime<Utc>>` | `lib.rs:688-706` |
| `interval_fire_instant(dur, now, workflow_id)` | private | Quantize `now` to `floor(now/interval)*interval` | `Option<DateTime<Utc>>` | `lib.rs:719-745` |
| `interval_should_fire(dur, last_fired, now, workflow_id)` | private | Elapsed-interval predicate; `None` ⇒ treated as `now` ⇒ false | `bool` | `lib.rs:753-774` |
| `interval_prefilter_should_fire(last_fired: &DashMap<…>, …)` | private | Testable prefilter + seed | `bool` | `lib.rs:784-797` |
| `should_fire_workflow(def, trigger_ctx, workflow_id)` | private, `async` | Emoji equality + `MessagePosted`/`DiffPosted` filter evaluation | `bool` | `lib.rs:806-882` |
| `trigger_matches_event(trigger, kind_u32)` | private | Kind→trigger-type match | `bool` | `lib.rs:955-964` |

---

### `schema` module

| Fn / method | Signature | Purpose | Returns | Line |
|---|---|---|---|---|
| `parse_yaml` | `pub fn parse_yaml(yaml: &str) -> Result<(WorkflowDef, String), WorkflowError>` | `serde_yaml` deserialize → `validate()` → `serde_json` canonical string | `(WorkflowDef, String)` | `schema.rs:262-268` |
| `WorkflowDef::validate` | `pub fn validate(&self) -> Result<(), WorkflowError>` | Name, steps, step-ID charset/uniqueness, schedule cron/interval invariants | `Result<(), WorkflowError>` | `schema.rs:152-229` |
| `validate_cron` | private `fn validate_cron(expr: &str) -> Result<(), WorkflowError>` | Normalize + parse via `cron::Schedule` | `Result<(), WorkflowError>` | `schema.rs:237-243` |
| `normalize_cron` | `pub(crate) fn normalize_cron(expr: &str) -> String` | 5→7 field / 6→7 field normalization | `String` | `schema.rs:250-257` |
| `default_true` | private | serde default for `enabled` | `bool` | `schema.rs:29-31` |

---

### `executor` module

| Fn | Signature | Purpose | Returns | Line |
|---|---|---|---|---|
| `TriggerContext::get_field` | `pub fn get_field(&self, name: &str) -> Option<&str>` | Named field lookup, falling back to `webhook_fields` | `Option<&str>` | `executor.rs:49-59` |
| `resolve_template` | `pub fn resolve_template(template: &str, trigger_ctx: &TriggerContext, step_outputs: &HashMap<String, JsonValue>) -> Result<String, WorkflowError>` | Single-pass `{{…}}` substitution with `| truncate(N)` / `| npub` filters | `String` | `executor.rs:70-123` |
| `resolve_variable` | private | `trigger.X` / `steps.ID.output.FIELD` lookup | `Option<String>` | `executor.rs:126-151` |
| `json_get_str` / `json_to_string` | private | JSON→string coercion for substitution | `Option<String>` / `String` | `executor.rs:154-174` |
| `apply_filter` | private | `truncate(N)`, `npub`, `truncate_pubkey`; unknown ⇒ `TemplateError` | `Result<String, WorkflowError>` | `executor.rs:176-201` |
| `build_eval_context` | `pub fn build_eval_context(trigger_ctx, step_outputs) -> Result<HashMapContext, WorkflowError>` | Register 4 custom fns + webhook vars + 6 trigger vars + step-output vars | `HashMapContext` | `executor.rs:224-316` |
| `json_value_to_eval` | private | `serde_json::Value` → `evalexpr::Value` | `evalexpr::Value` | `executor.rs:318-335` |
| `evaluate_condition` | `pub async fn evaluate_condition(expr: &str, trigger_ctx, step_outputs) -> Result<bool, WorkflowError>` | 4096-byte length gate, `spawn_blocking` + 100 ms timeout, `eval_boolean_with_context` | `bool` | `executor.rs:350-384` |
| `resolve_step_templates` | `pub fn resolve_step_templates(step: &Step, trigger_ctx, step_outputs) -> Result<ActionDef, WorkflowError>` | Returns a new `ActionDef` with templated fields substituted | `ActionDef` | `executor.rs:390-453` |
| `resolve_send_message_channel` | private | Destination resolution/validation for `send_message` | `Result<String, WorkflowError>` | `executor.rs:468-517` |
| `dispatch_action` | `pub async fn dispatch_action(step_id: &str, action: &ActionDef, engine: &WorkflowEngine, community_id: CommunityId, run_id: Uuid, trigger_ctx: &TriggerContext) -> Result<StepResult, WorkflowError>` | Per-action side effect | `StepResult` | `executor.rs:519-692` |
| `generate_approval_token` | private `fn(_run_id: Uuid, _step_id: &str) -> String` | `Uuid::new_v4().to_string()`; both args unused | `String` | `executor.rs:698-700` |
| `parse_duration_secs` | `pub(crate) fn parse_duration_secs(duration: &str) -> Result<u64, WorkflowError>` | `h`/`m`/`s`/bare-number parsing with checked multiply | `u64` | `executor.rs:705-735` |
| `check_ssrf` | private, `#[cfg(feature = "reqwest")]` `async fn(host: &str, port: u16) -> Result<IpAddr, WorkflowError>` | Resolve + reject private/reserved IPs; returns first IP for DNS pinning | `IpAddr` | `executor.rs:745-776` |
| `call_webhook_impl` | private, `#[cfg(feature = "reqwest")]` | Outbound HTTP with SSRF pinning, no redirects, 1 MiB cap | `JsonValue` | `executor.rs:781-866` |
| `shared_http_client` | private, `#[cfg(feature = "reqwest")]` | `LazyLock<reqwest::Client>`, 10 s timeout | `&'static Client` | `executor.rs:871-882` |
| `add_reaction_impl` | private, `#[cfg(feature = "reqwest")]` | `POST {BUZZ_RELAY_BASE_URL}/api/messages/{id}/reactions` | `JsonValue` | `executor.rs:885-930` |
| `execute_run` | `pub async fn execute_run(engine, community_id, run_id, def, trigger_ctx) -> Result<ExecutionResult, (WorkflowError, PartialProgress)>` | Acquire permit (`try_acquire`), set `Running`, run from step 0 | `ExecutionResult` | `executor.rs:967-1000` |
| `execute_from_step` | `pub async fn execute_from_step(engine, community_id, run_id, def, trigger_ctx, start_index: usize, initial_outputs: Option<HashMap<String, JsonValue>>) -> Result<ExecutionResult, (WorkflowError, PartialProgress)>` | Same but starts at `start_index`, preserves existing trace, seeds outputs | `ExecutionResult` | `executor.rs:1015-1072` |
| `execute_steps` | private `async fn` | Shared step loop (condition → templates → per-step `timeout` → dispatch → trace) | `ExecutionResult` | `executor.rs:1080-1214` |

---

### `action_sink` module

| Item | Signature | Purpose | Line |
|---|---|---|---|
| `ActionSink::send_message` | `fn send_message(&self, community_id: CommunityId, channel_id: &str, text: &str, author_pubkey: &str) -> Pin<Box<dyn Future<Output = Result<String, ActionSinkError>> + Send + '_>>` | Only sink operation; returns the created event id hex | `action_sink.rs:50-70` |
| `From<ActionSinkError> for WorkflowError` | maps to `WorkflowError::WebhookError(e.to_string())` | error bridging | `action_sink.rs:34-38` |

`error` module: `From<buzz_db::error::DbError> for WorkflowError` → `Database(String)` (`error.rs:62-66`).

---

### External call sites (who uses this API)

| API | Caller | Line |
|---|---|---|
| `WorkflowEngine::new` + `WorkflowConfig::default` | relay startup and many relay test fixtures | `crates/buzz-relay/src/main.rs:389-390`, `crates/buzz-relay/src/state.rs:1274-1276` |
| `set_action_sink` | relay startup | `crates/buzz-relay/src/main.rs:595` |
| `run()` (scheduler) | spawned after the sink is wired | `crates/buzz-relay/src/main.rs:597-599` |
| `on_event` | relay post-store fan-out hook | `crates/buzz-relay/src/handlers/event.rs:522` |
| `WorkflowEngine::parse_yaml` | workflow upsert command handler | `crates/buzz-relay/src/handlers/command_executor.rs:684` |
| `invalidate_channel_workflows` | workflow upsert + NIP-09 deletion | `crates/buzz-relay/src/handlers/command_executor.rs:787`, `crates/buzz-relay/src/handlers/side_effects.rs:2018`, `:2039` |
| `executor::execute_from_step` + `finalize_run` | manual trigger, inbound webhook, approval resume | `crates/buzz-relay/src/handlers/command_executor.rs:926`, `:1314`, `crates/buzz-relay/src/api/bridge.rs:1890` |
| `ActionSink` impl | `RelayActionSink` | `crates/buzz-relay/src/workflow_sink.rs:13`, `:159` |

`execute_run` has no callers outside this crate (only `lib.rs:373` and `lib.rs:651`); `build_trigger_context`, `resolve_template`, `resolve_step_templates`, `build_eval_context`, `evaluate_condition`, `dispatch_action` have no callers outside this crate.
