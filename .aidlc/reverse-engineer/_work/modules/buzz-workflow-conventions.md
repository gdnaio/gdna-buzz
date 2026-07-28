## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Conventions

---

### 1. Module organization

| File | LOC | Role |
|---|---|---|
| `src/lib.rs` | 1564 | crate root: `WorkflowConfig`, `WorkflowEngine`, event hook, scheduler loop, trigger matching, `build_trigger_context` |
| `src/schema.rs` | 878 | definition types + validation + YAML parsing |
| `src/executor.rs` | 1834 | templates, conditions, action dispatch, HTTP impls, step loop |
| `src/error.rs` | 66 | `WorkflowError`, `PartialProgress` |
| `src/action_sink.rs` | 69 | `ActionSink` trait + `ActionSinkError` |

Flat module layout — four sibling modules, all declared `pub` at `lib.rs:33-36`, with the commonly used types re-exported at the crate root (`lib.rs:38-41`) so callers write `buzz_workflow::WorkflowDef` rather than `buzz_workflow::schema::WorkflowDef`.

Crate-level lints: `#![deny(unsafe_code)]` and `#![warn(missing_docs)]` (`lib.rs:1-2`); every public item carries a doc comment, and `///` docs are also used on private helpers and enum fields.

---

### 2. Naming

| Convention | Examples |
|---|---|
| Types `UpperCamelCase`, suffix conveys role | `WorkflowDef`, `TriggerDef`, `ActionDef`, `TriggerContext`, `ExecutionResult`, `StepResult`, `PartialProgress`, `WorkflowError`, `ActionSinkError` |
| Serde variant naming | `rename_all = "snake_case"` on both tagged enums so Rust `SendChannelTopic`-style variants map to YAML `set_channel_topic` (`schema.rs:37`, `schema.rs:91`) |
| Keyword collisions | Rust field `if_expr` renamed to YAML `if` (`schema.rs:79-80`) |
| Verb-first functions | `parse_yaml`, `validate_cron`, `normalize_cron`, `resolve_template`, `resolve_variable`, `resolve_step_templates`, `build_eval_context`, `build_trigger_context`, `evaluate_condition`, `dispatch_action`, `execute_run`, `execute_steps`, `execute_from_step`, `finalize_run` |
| Predicate functions prefixed `is_`/`should_`/`_matches_` | `interval_should_fire`, `interval_prefilter_should_fire`, `should_fire_workflow`, `trigger_matches_event` |
| `_impl` suffix for feature-gated internals | `call_webhook_impl`, `add_reaction_impl` (`executor.rs:781`, `executor.rs:885`) |
| Constants `SCREAMING_SNAKE_CASE`, unit in the name | `EVAL_TIMEOUT`, `MAX_EXPR_LEN`, `MAX_DELAY_SECS`, `WEBHOOK_MAX_RESPONSE_BYTES` |
| Deliberately unused params prefixed `_` | `generate_approval_token(_run_id, _step_id)` (`executor.rs:698`) |
| evalexpr variable mangling | dots → underscores: `trigger_text`, `steps_{id}_output_{field}` (documented as a table in the fn doc, `executor.rs:207-217`) |

---

### 3. Error handling

Single crate error enum `WorkflowError` (`error.rs:18-60`), `thiserror`-derived, all variants documented:

| Variant | Payload | `#[error]` message | Line |
|---|---|---|---|
| `InvalidYaml` | `#[from] serde_yaml::Error` | `invalid YAML: {0}` | `error.rs:20-22` |
| `InvalidDefinition` | `String` | `invalid definition: {0}` | `error.rs:24-26` |
| `ConditionError` | `String` | `condition evaluation error: {0}` | `error.rs:28-30` |
| `TemplateError` | `String` | `template error: {0}` | `error.rs:32-34` |
| `StepTimeout` | `{ step_id: String, timeout_secs: u64 }` | `step '{step_id}' timed out after {timeout_secs}s` | `error.rs:36-43` |
| `WebhookError` | `String` | `webhook error: {0}` | `error.rs:45-47` |
| `CapacityExceeded` | — | `capacity exceeded` | `error.rs:49-51` |
| `Database` | `String` | `database error: {0}` | `error.rs:53-55` |
| `NotImplemented` | `String` | `action not implemented: {0}` | `error.rs:57-59` |

Companion `ActionSinkError` (`action_sink.rs:12-32`) has 6 variants (`InvalidInput`, `ChannelNotFound`, `ChannelArchived`, `EventBuild`, `Database`, `EmptyContent`) and collapses into `WorkflowError::WebhookError` (`action_sink.rs:34-38`).

Patterns:
- `?` with `map_err` closures everywhere; no `unwrap()`/`expect()` in production paths except `LazyLock` client construction `expect("HTTP client build must succeed")` (`executor.rs:879`) and `parts.next().unwrap_or("")` style safe fallbacks (`executor.rs:99`).
- Fallible operations that must not abort a batch use "log-and-continue": `tracing::warn!`/`error!` then `continue` (`lib.rs:333-336`, `lib.rs:466-469`, `lib.rs:604-610`).
- Partial results are first-class: `Result<ExecutionResult, (WorkflowError, PartialProgress)>` is the executor's return type so trace data survives failure (`executor.rs:972`, `executor.rs:1085`).
- One deliberate panic: double `set_action_sink` (`lib.rs:139-143`), documented with `# Panics`.
- `WorkflowError::WebhookError` is overloaded — it also carries `send_message` DB-lookup failures (`executor.rs:539-541`, `executor.rs:548-551`) and SSRF/DNS failures (`executor.rs:757-763`).

---

### 4. Async patterns

| Pattern | Usage |
|---|---|
| Detached background execution | `tokio::spawn` for each triggered run, both event and cron paths (`lib.rs:371-381`, `lib.rs:649-661`) |
| Non-blocking admission | `Semaphore::try_acquire()` (no `acquire().await`), permit held in a `_permit` binding for the run's lifetime (`executor.rs:975`, `executor.rs:1025`) |
| CPU/blocking isolation | `spawn_blocking` for evalexpr evaluation (`executor.rs:372`) and for the blocking DNS resolver (`executor.rs:747-755`) |
| Timeouts | `tokio::time::timeout` around expression evaluation (`executor.rs:370`) and around each `dispatch_action` (`executor.rs:1136-1148`) |
| Sleep-based loop | `loop { sleep(60s).await; … }` scheduler with sleep-first ordering (`lib.rs:430-432`) |
| `Arc<Self>` receivers | `on_event(self: &Arc<Self>)` and `run(self: &Arc<Self>)` so spawned tasks can clone the engine without `'static` on `&self` (`lib.rs:276-279`, `lib.rs:428`) |
| Late init without `Mutex` | `OnceLock<Arc<dyn ActionSink>>` (`lib.rs:90`) |
| dyn-compatible async trait | manual `Pin<Box<dyn Future … + Send + '_>>` return instead of `async_trait` (`action_sink.rs:60-70`) |
| Lock-free shared state | `DashMap` for interval anchors, `moka::sync::Cache` for workflow lookups (`lib.rs:87`, `lib.rs:104`) |
| Sync-in-async caution | `evaluate_condition` is `async` purely to host the timeout; `resolve_template`/`build_eval_context` stay synchronous |

---

### 5. Testing patterns

All tests are inline `#[cfg(test)] mod tests` blocks: `schema.rs:270`, `lib.rs:966`, `executor.rs:1216`. No `tests/` directory, no fixtures directory, no mocking crate.

| File | `#[test]` | `#[tokio::test]` | Total |
|---|---|---|---|
| `schema.rs` | 50 | 0 | 50 |
| `lib.rs` | 38 | 0 | 38 |
| `executor.rs` | 39 | 22 | 61 |
| `error.rs` | 0 | 0 | 0 |
| `action_sink.rs` | 0 | 0 | 0 |
| **Total** | **127** | **22** | **149** |

Conventions observed:
- YAML fixtures are inline `&str` literals built with `concat!` or `\n`-escaped strings, with a comment explaining why raw strings are avoided (`schema.rs:275-276`, `schema.rs:326-328`).
- Error assertions use `matches!(err, WorkflowError::Variant(_))` plus substring checks on the message (`schema.rs:404-407`, `schema.rs:428-436`).
- Deterministic time: fixed RFC-3339 instants parsed for cron/interval tests (`lib.rs:969-985`, `lib.rs:1140-1166`) alongside `Utc::now()`-relative tests for elapsed-interval logic (`lib.rs:1168-1235`).
- Pure-function extraction for testability: `interval_prefilter_should_fire` is a free function over `&DashMap` explicitly "so it is unit-testable without a `Db`/Postgres" (`lib.rs:777-782`).
- Shared builders instead of a framework: `make_trigger()` (`executor.rs:1220-1230`), `make_message_event()` (`lib.rs:1338-1347`), `make_reaction_event()` (`lib.rs:1350-1371`).
- Regression tests carry intent comments naming the bug they lock down (`lib.rs:1211-1216`, `executor.rs:1809-1811`, `schema.rs:756-758`).
- Nothing that requires Postgres or an `ActionSink` is unit-tested — no test constructs a `WorkflowEngine`, so `on_event`, `run`, `execute_run`, `execute_from_step`, `execute_steps`, `dispatch_action`, and `finalize_run` have zero unit coverage in this crate.

---

### 6. Documentation conventions

- Module-level `//!` headers with a Responsibilities list (`executor.rs:1-11`) and an Architecture/Usage section including a `rust,ignore` example (`lib.rs:3-31`).
- Markdown tables inside doc comments to specify mappings (`executor.rs:207-217`).
- Long rationale comments attached to consistency-critical fields — the workflow cache's cross-pod invalidation trade-off (`lib.rs:92-103`) and the interval cold-start liveness argument (`lib.rs:385-399`).
- Ticket-tagged deferrals: `WF-07`, `WF-08`, `WF-09` (`executor.rs:582`, `:588`, `:663`, `:675`; `lib.rs:192`).
- Numbered "Fix N" comments preserving review history (`schema.rs:218`, `lib.rs:465`, `lib.rs:572`, `lib.rs:638`, `lib.rs:664`).
