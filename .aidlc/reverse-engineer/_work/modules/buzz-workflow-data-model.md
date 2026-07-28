## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Data Model

Source files read in full: `Cargo.toml`, `src/lib.rs` (1564 LOC), `src/schema.rs` (878), `src/executor.rs` (1834), `src/error.rs` (66), `src/action_sink.rs` (69).

---

### 1. Definition schema root — `WorkflowDef`

`crates/buzz-workflow/src/schema.rs:13-27`. Derives `Debug, Clone, Serialize, Deserialize`. No `deny_unknown_fields` — unknown keys in the stored JSON are silently ignored (this is how the relay's `_webhook_secret` key survives round-trips; see security doc).

| Field | Type | Serde attribute | Line |
|---|---|---|---|
| `name` | `String` | (required) | `schema.rs:16` |
| `description` | `Option<String>` | `#[serde(default)]` | `schema.rs:18-19` |
| `trigger` | `TriggerDef` | (required) | `schema.rs:21` |
| `steps` | `Vec<Step>` | (required) | `schema.rs:23` |
| `enabled` | `bool` | `#[serde(default = "default_true")]` → `true` (`schema.rs:29-31`) | `schema.rs:25-26` |

---

### 2. `TriggerDef` — internally tagged on `on`

`crates/buzz-workflow/src/schema.rs:36-68`: `#[serde(tag = "on", rename_all = "snake_case")]` (`schema.rs:37`). Five variants; fields live inline in the same YAML/JSON mapping as `on:`.

| YAML `on:` value | Rust variant | Fields (type, serde) | Line |
|---|---|---|---|
| `message_posted` | `MessagePosted` | `filter: Option<String>` — `#[serde(default)]` | `schema.rs:40-44` |
| `reaction_added` | `ReactionAdded` | `emoji: Option<String>` — `#[serde(default)]` | `schema.rs:46-50` |
| `diff_posted` | `DiffPosted` | `filter: Option<String>` — `#[serde(default)]` | `schema.rs:52-56` |
| `schedule` | `Schedule` | `cron: Option<String>` and `interval: Option<String>`, both `#[serde(default)]` | `schema.rs:58-65` |
| `webhook` | `Webhook` | unit variant, no fields | `schema.rs:67` |

Round-trip is asserted for `diff_posted` at `schema.rs:855-871`.

---

### 3. `Step`

`crates/buzz-workflow/src/schema.rs:71-87`.

| Field | Type | Serde attribute | Line |
|---|---|---|---|
| `id` | `String` | (required) | `schema.rs:74` |
| `name` | `Option<String>` | `#[serde(default)]` | `schema.rs:76-77` |
| `if_expr` | `Option<String>` | `#[serde(rename = "if", default)]` — YAML key is `if:` | `schema.rs:79-80` |
| `timeout_secs` | `Option<u64>` | `#[serde(default)]` | `schema.rs:82-83` |
| `action` | `ActionDef` | `#[serde(flatten)]` — action tag + fields inline on the step map | `schema.rs:85-86` |

---

### 4. `ActionDef` — internally tagged on `action`

`crates/buzz-workflow/src/schema.rs:90-147`: `#[serde(tag = "action", rename_all = "snake_case")]` (`schema.rs:91`). Seven variants.

| YAML `action:` value | Variant | Fields (type, serde) | Line |
|---|---|---|---|
| `send_message` | `SendMessage` | `text: String`; `channel: Option<String>` `#[serde(default)]` | `schema.rs:94-100` |
| `send_dm` | `SendDm` | `to: String`; `text: String` | `schema.rs:102-107` |
| `set_channel_topic` | `SetChannelTopic` | `topic: String` | `schema.rs:109-112` |
| `add_reaction` | `AddReaction` | `emoji: String` | `schema.rs:114-117` |
| `call_webhook` | `CallWebhook` | `url: String`; `method: Option<String>` `#[serde(default)]`; `headers: Option<HashMap<String,String>>` `#[serde(default)]`; `body: Option<String>` `#[serde(default)]` | `schema.rs:119-131` |
| `request_approval` | `RequestApproval` | `from: String`; `message: String`; `timeout: Option<String>` `#[serde(default)]` | `schema.rs:133-141` |
| `delay` | `Delay` | `duration: String` | `schema.rs:143-146` |

Because both enums are internally tagged and `ActionDef` is additionally `#[serde(flatten)]`ed into `Step`, a step is a flat mapping: `{id, name?, if?, timeout_secs?, action, <action fields>}`.

---

### 5. Runtime / execution types

| Type | Kind | Fields | Line |
|---|---|---|---|
| `WorkflowConfig` | struct, `Clone, Debug` | `max_concurrent: usize`, `default_timeout_secs: u64` | `lib.rs:57-63` |
| `WorkflowEngine` | struct (no derive) | `db: Db` (`lib.rs:76`), `config: WorkflowConfig` (`77`), `run_semaphore: Arc<Semaphore>` (`79`), `last_fired: DashMap<(CommunityId, Uuid), DateTime<Utc>>` (`87`), `action_sink: OnceLock<Arc<dyn ActionSink>>` (`90`), `workflow_cache: moka::sync::Cache<(CommunityId, Uuid), Arc<Vec<buzz_db::workflow::WorkflowRecord>>>` (`104-105`) | `lib.rs:75-106` |
| `TriggerContext` | struct, `Debug, Clone, Default, Serialize, Deserialize` | `text: String`, `author: String`, `channel_id: String`, `timestamp: String`, `emoji: String`, `message_id: String`, `webhook_fields: HashMap<String,String>` | `executor.rs:26-42` |
| `StepResult` | enum, `Debug` | `Completed(JsonValue)`, `Suspended { approval_token: String }`, `Skipped` | `executor.rs:455-465` |
| `ExecutionResult` | struct, `Debug` | `approval_token: Option<String>`, `step_index: usize`, `step_outputs: HashMap<String, JsonValue>`, `trace: Vec<JsonValue>` | `executor.rs:938-949` |
| `PartialProgress` | struct, `Debug, Default` | `step_index: usize`, `trace: Vec<serde_json::Value>` | `error.rs:9-15` |
| `WorkflowError` | enum, `Debug, Error` | 9 variants — see conventions doc | `error.rs:18-60` |
| `ActionSinkError` | enum, `Debug, thiserror::Error` | `InvalidInput(String)`, `ChannelNotFound(String)`, `ChannelArchived(String)`, `EventBuild(String)`, `Database(String)`, `EmptyContent` | `action_sink.rs:12-32` |
| `ActionSink` | trait, `Send + Sync` | one method `send_message` returning `Pin<Box<dyn Future<Output = Result<String, ActionSinkError>> + Send + '_>>` | `action_sink.rs:48-70` |

Trace entry shapes written into `execution_trace` (a JSON array):

| Shape | Written when | Line |
|---|---|---|
| `{"step_id": …, "status": "skipped"}` | `if:` evaluated false | `executor.rs:1105-1108` |
| `{"step_id": …, "status": "completed", "output": <JsonValue>}` | step dispatched successfully | `executor.rs:1176-1180` |
| `{"step_id": …, "status": "skipped"}` | `StepResult::Skipped` returned | `executor.rs:1199-1202` |

Per-action `output` payloads (become `steps.ID.output.*`):

| Action | Output JSON | Line |
|---|---|---|
| `send_message` | `{"sent": true, "event_id": <hex>}` | `executor.rs:574-577` |
| `add_reaction` (with `reqwest`) | `{"added": true, "status": <u16>, "response": <json>}` | `executor.rs:925-929` |
| `add_reaction` (no `reqwest`) | `{"added": false, "skipped": true}` | `executor.rs:613-615` |
| `call_webhook` (with `reqwest`) | `{"status": <u16>, "body": <string>}` | `executor.rs:862-865` |
| `call_webhook` (no `reqwest`) | `{"status": 0, "body": null, "skipped": true}` | `executor.rs:642-646` |
| `delay` | `{"slept_secs": <u64>}` | `executor.rs:685-687` |
| `send_dm`, `set_channel_topic` | none — return `WorkflowError::NotImplemented` | `executor.rs:583`, `executor.rs:589` |
| `request_approval` | none — returns `StepResult::Suspended { approval_token }` | `executor.rs:665-667` |

---

### 6. DB row shapes consumed via `buzz-db`

The crate does not define SQL; it uses `buzz_db::Db` methods and the record structs in `crates/buzz-db/src/workflow.rs`.

**Read: `WorkflowRecord`** (`crates/buzz-db/src/workflow.rs:165-190`) — `id: Uuid`, `community_id: CommunityId`, `name: String`, `owner_pubkey: Vec<u8>`, `channel_id: Option<Uuid>`, `definition: serde_json::Value`, `definition_hash: Vec<u8>`, `status: WorkflowStatus`, `enabled: bool`, `created_at`, `updated_at`.
Fields actually touched by this crate: `definition` (deserialized into `WorkflowDef` at `lib.rs:331`, `lib.rs:459`), `enabled` (`lib.rs:335`, `lib.rs:474`), `id` (`lib.rs:339`, `lib.rs:346`), `community_id` (`lib.rs:455`), `channel_id` (`lib.rs:478`, `executor.rs:553`), `owner_pubkey` (hex-encoded at `executor.rs:558`).

**Read: `WorkflowRunRecord`** (`crates/buzz-db/src/workflow.rs:192-229`) — `id`, `community_id`, `workflow_id`, `status: RunStatus`, `trigger_event_id: Option<Vec<u8>>`, `current_step: i32`, `execution_trace: serde_json::Value`, `trigger_context: Option<serde_json::Value>`, `started_at`, `completed_at`, `error_message`, `created_at`. This crate reads `workflow_id` (`executor.rs:546`) and `execution_trace` (`executor.rs:1035`).

**Write path:**

| Db method | Args written by this crate | Call site |
|---|---|---|
| `create_workflow_run(community_id, workflow_id, trigger_event_id: Option<&[u8]>, trigger_context: Option<&Value>)` → `Uuid` | event path passes the trigger event id bytes + serialized `TriggerContext`; cron path passes `None` event id | `lib.rs:344-355`, `lib.rs:590-600` |
| `update_workflow_run(community_id, id, status: RunStatus, current_step: i32, trace: &Value, error: Option<&str>)` | `Running` at start (`executor.rs:982-991`, `executor.rs:1044-1053`); `Completed`/`Failed` at finalize (`lib.rs:199-215`, `lib.rs:218-238`, `lib.rs:242-261`) | as listed |
| `claim_scheduled_workflow_fire(community_id, workflow_id, scheduled_for)` → `Option<ScheduledWorkflowFireClaim>` | durable at-most-once cron claim | `lib.rs:547-568` |
| `attach_scheduled_workflow_run(community_id, workflow_id, scheduled_for, run_id)` → `bool` | best-effort link claim→run | `lib.rs:617-628` |
| `latest_scheduled_workflow_fire(community_id, workflow_id)` → `Option<DateTime<Utc>>` | restart anchor for interval triggers | `lib.rs:500-517` |
| `list_enabled_channel_workflows(community_id, channel_id)` → `Vec<WorkflowRecord>` | per-event lookup (cached) | `lib.rs:301-306` |
| `list_all_enabled_workflows()` → `Vec<WorkflowRecord>` | cron scan (not community-scoped; carries `community_id` per row) | `lib.rs:436` |
| `get_workflow_run(community_id, run_id)` / `get_workflow(community_id, workflow_id)` | destination validation + attribution for `send_message`; trace restore for resume | `executor.rs:535-556`, `executor.rs:1034` |

**Status enums referenced** (defined in `crates/buzz-db/src/workflow.rs`): `RunStatus { Pending, Running, WaitingApproval, Completed, Failed, Cancelled }` (`workflow.rs:78-91`); `WorkflowStatus { Active, Disabled, Archived }` (`workflow.rs:42-50`); `ApprovalStatus { Pending, Granted, Denied, Expired }` (`workflow.rs:124-133`). This crate writes only `Running`, `Completed`, `Failed` (`executor.rs:986`, `executor.rs:1048`, `lib.rs:204`, `lib.rs:223`, `lib.rs:247`) — it never writes `WaitingApproval`, `Pending`, or `Cancelled`.

**Approval row shape it does *not* write:** `ApprovalRecord` (`crates/buzz-db/src/workflow.rs:244-268`) — `token: Vec<u8>` (SHA-256 hash of the raw token, `workflow.rs:33-35`), `workflow_id`, `run_id`, `step_id: String`, `step_index: i32`, `approver_spec: String`, `status: ApprovalStatus`, `approver_pubkey: Option<Vec<u8>>`, `note: Option<String>`, `expires_at`, `created_at`. No code path in the repository calls `Db::create_approval` (`crates/buzz-db/src/lib.rs:2729`) outside buzz-db itself — verified by repo-wide grep.
