## Module: buzz-acp — agent pool & lifecycle (`crates/buzz-acp/src`)
### Aspect: Data Model

The group is two files: `pool.rs` (5,620 lines — pool container, per-agent record, turn task) and `pool_lifecycle.rs` (312 lines — the deferred-start state machine). No type in either file derives `Serialize`/`Deserialize`; nothing here is persisted.

#### Pool container

| Type | Site | Fields |
|---|---|---|
| `AgentPool` | `pool.rs:212-219` | `agents: Vec<Option<OwnedAgent>>`, `result_tx`/`result_rx: mpsc::Unbounded*<PromptResult>`, `pub join_set: JoinSet<()>`, `task_map: HashMap<tokio::task::Id, TaskMeta>` |
| `TaskMeta` | `pool.rs:51-72` | `agent_index: usize`, `channel_id: Option<Uuid>`, `turn_id: String`, `recoverable_batch: Option<FlushBatch>`, `control_tx: Option<oneshot::Sender<ControlSignal>>`, `steer_tx: Option<mpsc::Sender<SteerRequest>>` |
| `PromptResult` | `pool.rs:221-231` | `agent: OwnedAgent`, `source: PromptSource`, `turn_id: String`, `outcome: PromptOutcome`, `batch: Option<FlushBatch>` |

Slot invariant: an agent is either idle in `agents[i]` or checked out inside a spawned task with a `task_map` entry (`pool.rs:206-210`). `agent.index` must equal its `agents` index — `from_slots` (`pool.rs:541`) preserves positions specifically so a failed startup slot (`None`) does not shift later agents; the doc calls out that a densely-packing `new()` "would break the index invariant" (`pool.rs:536-540`). No `new()` exists in the file; `from_slots` is the only constructor.

`agents` is a fixed-length `Vec` sized once at construction — `(0..config.agents).map(|_| None)` for the lazy path (`lib.rs:1318`) or one entry per startup attempt (`lib.rs:3747`). There is no grow/shrink API; `agents_mut()` (`pool.rs:695`) hands out `&mut Vec` but the only callers `take()` slots during shutdown (`lib.rs:2649`, `lib.rs:3710`).

#### Per-agent record

`OwnedAgent` (`pool.rs:150-171`): `index: usize`, `acp: AcpClient`, `state: SessionState`, `model_capabilities: Option<AgentModelCapabilities>`, `desired_model: Option<String>`, `model_overridden: bool`, `agent_name: String`, `goose_system_prompt_supported: Option<bool>`, `protocol_version: u32`.

- `AcpClient` is not `Clone`; ownership moves out on claim and back on return (`pool.rs:22`).
- `agent_name` is lower-cased from `agentInfo.name`/`serverInfo.name` (`lib.rs:3689-3699`); the string `"goose"` is a behavioural switch at `pool.rs:174-179`, `pool.rs:826`.
- `goose_system_prompt_supported: Option<bool>` is a three-state cache: `None` = unprobed, `Some(false)` = agent answered `-32601`, latched for the process lifetime (`pool.rs:846-856`).
- `model_overridden` distinguishes a live `SwitchModel` from a config/persona-derived model; reset on spawn/respawn because every construction site passes `false` (`lib.rs:1794`, `lib.rs:3800`).
- `AgentModelCapabilities` (`pool.rs:72-81`) holds `config_options_raw: Vec<serde_json::Value>` and `available_models_raw: Option<serde_json::Value>`, populated once on first `session/new` (`pool.rs:862-867`).

`SessionState` (`pool.rs:82-105`, `#[derive(Default)]`) is the per-agent session map, deliberately split from `OwnedAgent` "so the state machine is testable without spawning a real agent subprocess" (`pool.rs:83-84`):

| Field | Type | Keyed by |
|---|---|---|
| `sessions` | `HashMap<Uuid, String>` | channel → ACP session id |
| `heartbeat_session` | `Option<String>` | — |
| `turn_counts` | `HashMap<Uuid, u32>` | channel → turns since session creation |
| `heartbeat_turn_count` | `u32` | — |
| `core_sections` | `HashMap<Uuid, String>` | channel → rendered NIP-AE core block |
| `canvas_sections` | `HashMap<Uuid, String>` | channel → rendered `[Channel Canvas]` block |

Three mutators, all in `pool.rs:107-139`: `invalidate(&PromptSource)` dispatches to channel or heartbeat; `invalidate_channel(&Uuid) -> bool` removes turn count, core, canvas, and session and returns whether a session existed (`pool.rs:123-129`); `invalidate_all()` clears all six fields (`pool.rs:131-138`). All six maps are unbounded and only shrink through these calls — a long-lived agent that is mentioned in many channels accumulates one entry per channel per map.

#### Control- and outcome-carrying enums

| Enum | Variants | Site |
|---|---|---|
| `PromptSource` | `Channel(Uuid)`, `Heartbeat` | `pool.rs:233-236` |
| `ControlSignal` | `Cancel`, `Interrupt`, `Steer`, `Rotate`, `SwitchModel(String)` | `pool.rs:263-289` |
| `PromptOutcome` | `Ok(StopReason)`, `Error(AcpError)`, `AgentExited`, `Timeout(TimeoutKind)`, `Cancelled`, `CancelDrainTimeout(Duration)` | `pool.rs:405-431` |
| `TimeoutKind` | `Idle`, `Hard { recently_active: bool }` | `pool.rs:394-403` |
| `SteerError` | `AgentError { code: i64, message: String }`, `Transport(String)`, `ExpectedRunIdMissing`, `PromptCompleted` | `pool.rs:335-373` |
| `SteerAck` | `Success`, `Err(SteerError)`, `PromptCompletedNeutral` | `pool.rs:375-392` |
| `IdleSwitchResult` | `Switched`, `UnsupportedModel`, `NoIdleAgent` | `pool.rs:765-774` |

`ControlSignal` derives `Clone, Debug, Eq, PartialEq` and is explicitly not `Copy` because `SwitchModel` owns a `String` (`pool.rs:259-262`). `SteerRequest` (`pool.rs:319-333`) carries `prompt_blocks: Vec<String>` plus a `oneshot::Sender<SteerAck>`; the read loop, not the sender, fills `sessionId` and `expectedRunId` at write time (`pool.rs:296-318`).

#### Shared immutable turn context

`PromptContext` (`pool.rs:482-532`) is the `Arc`-shared, 24-field config snapshot every prompt task reads: `mcp_servers`, `initial_message`, `idle_timeout`, `max_turn_duration`, `turn_liveness_interval`, `dedup_mode`, `system_prompt`, `team_instructions`, `heartbeat_prompt`, `base_prompt: Option<&'static str>`, `cwd: String`, `rest_client`, `channel_info`, `context_message_limit`, `max_turns_per_session`, `permission_mode`, `agent_keys: nostr::Keys`, `agent_owner_pubkey`, `memory_enabled`, `harness_name`, `relay_url`. `base_prompt` is `'static` via `Box::leak` of the file-supplied content (`lib.rs:1539-1544`) — an intentional one-shot leak.

`ChannelInfoResolver` (`pool.rs:436-440`) wraps `Arc<RwLock<HashMap<Uuid, PromptChannelInfo>>>` plus a `RestClient`; `new` filters out startup entries whose `channel_type == "unknown"` (`pool.rs:446-452`), and `resolve` caches only successful lazy lookups (`pool.rs:464-479`). It is the one shared-mutable structure in the module and uses a std (not tokio) `RwLock`, read across `.await` boundaries only after the guard is dropped.

#### Turn-scoped drop guards

Three RAII types own per-turn side effects, all private:

| Guard | Site | Holds | Drop effect |
|---|---|---|---|
| `ReactionGuard` | `pool.rs:3111-3141` | `Option<RestClient>`, `Vec<String>` | spawns `clear_reactions` if a runtime handle exists |
| `LivenessGuard` | `pool.rs:3228-3264` | `JoinHandle<()>`, `Arc<Mutex<LivenessState>>` | sets `closed = true` under the lock, then `abort()` |
| `TurnCompletionGuard` | `pool.rs:3267-3302` | observer handle + `turn_id` | emits `turn_completed` |

`LivenessState` (`pool.rs:3206-3209`) is `{ closed: bool, session_id: Option<String> }` behind a single `std::sync::Mutex` shared with `run_turn_liveness`, so a tick's check-then-emit cannot interleave with the guard's set-closed-then-abort (`pool.rs:3145-3160`, `pool.rs:3211-3227`). Declaration order in `run_prompt_task` is load-bearing: `_turn_guard` (`pool.rs:1309`) before `liveness_guard` (`pool.rs:1342`) so reverse-order drop aborts liveness before `turn_completed` is emitted (`pool.rs:1305-1308`); `_reaction_guard` follows at `pool.rs:1351`.

#### Lifecycle state machine (`pool_lifecycle.rs`)

`PoolLifecycle<P>` is `pub(crate)`, generic over the pooled value, `#[derive(Debug)]`, four states (`pool_lifecycle.rs:13-25`):

| State | Payload | Site |
|---|---|---|
| `Listening` | — | `pool_lifecycle.rs:15` |
| `Waking` | `attempt: u32` | `pool_lifecycle.rs:16-18` |
| `Ready` | `P` | `pool_lifecycle.rs:19` |
| `Failed` | `attempt: u32`, `retry_at: tokio::time::Instant`, `error: String` | `pool_lifecycle.rs:20-24` |

Instantiated once as `PoolLifecycle<AgentPool>` (`lib.rs:1323`). Two module constants gate its timing and are not configurable: `INITIAL_RETRY_DELAY = 5s` (`pool_lifecycle.rs:11`) and `MAX_RETRY_DELAY = 300s` (`pool_lifecycle.rs:12`). `Ready` is transient — `take_ready` replaces the state with `Listening` while handing out the pool (`pool_lifecycle.rs:60-68`), so the machine holds no record that a wake ever succeeded; the caller's separate `pool_ready: bool` (`lib.rs:1322`) is what suppresses further wakes.

#### Constants that shape turn state

`RECENT_ACTIVITY_WINDOW = 60s` (`pool.rs:45`) decides `TimeoutKind::Hard { recently_active }`. `CONTEXT_FETCH_TIMEOUT = 3000ms` / `CONTEXT_FETCH_RETRY_DELAY = 500ms` (`pool.rs:780`, `pool.rs:783`), `MODEL_SWITCH_TIMEOUT = 5s` (`pool.rs:786`), `CONTROL_CANCEL_GRACE = 5s` (`pool.rs:793`), `PERMISSION_MODE_TIMEOUT = 5s` (`pool.rs:796`), `REACTION_TIMEOUT = 500ms` (`pool.rs:3437`), `REACTION_CONCURRENCY = 10` (`pool.rs:3618`), plus three function-local `const`s: `CORE_FETCH_TIMEOUT = 3s` (`pool.rs:1386`), `CANVAS_FETCH_TIMEOUT = 3s` (`pool.rs:2316`), and `METRIC_TIMEOUT = 3s` (`pool.rs:3415`).
