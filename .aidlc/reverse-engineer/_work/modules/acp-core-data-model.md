## Module: buzz-acp — harness core & orchestration (`crates/buzz-acp/src`)
### Aspect: Data Model

No database, no ORM, no migrations. All state is process-local and dies with the harness. `lib.rs` declares no `serde::Serialize`/`Deserialize` derives at all — only three derives exist in the whole 6,570-line file (`lib.rs:518`, `lib.rs:529`, `lib.rs:2705`). Every wire payload it constructs is an untyped `serde_json::json!` literal.

#### Types owned by `lib.rs`

| Type | Lines | Shape | Purpose |
|---|---|---|---|
| `OwnerCache` | `lib.rs:152-190` | `pubkey: Option<String>`, `siblings: std::sync::Mutex<HashMap<String,bool>>` | owner pubkey + NIP-OA sibling verdicts, process-lifetime |
| `ObserverPublishPacer` | `lib.rs:367-409` | `next_publish: tokio::time::Instant`, `published: VecDeque<Instant>` | rolling-minute rate limiter for observer frames |
| `ObserverChunkCoalescer` | `lib.rs:518-522` | `pending: Vec<PendingObserverChunk>` | merges streamed text chunks before publish |
| `PendingObserverChunk` | `lib.rs:523-528` | `key`, `event: ObserverEvent`, `text: String` | accumulator per chunk stream |
| `ObserverChunkKey` | `lib.rs:529-542` | `update_type`, `message_id`, `channel_id`, `session_id`, `turn_id`, `agent_index` (all `Option` except `update_type`) | identity of a chunk stream |
| `SlotCircuit` | `lib.rs:1027-1035` | `crash_times: Vec<std::time::Instant>`, `open_until: Option<Instant>`, `respawn_in_flight: bool` | per-slot circuit breaker |
| `CrashVerdict` | `lib.rs:1037-1046` | `Respawn(Duration)` \| `CircuitOpen` \| `HalfOpenProbe` | breaker decision |
| `RespawnResult` | `lib.rs:1141-1147` | `index: usize`, `result: Result<(AcpClient, u32, String)>` | background respawn return |
| `SteerAckEvent` | `lib.rs:1158-1170` | `channel_id: Uuid`, `event_id: String`, `ack: Result<pool::SteerAck, RecvError>` | steer outcome routed to the main `select!` |
| `RespawnGuard` | `lib.rs:1172-1231` | `index`, `tx: mpsc::Sender<RespawnResult>`, `sent: bool` | RAII; `Drop` at `lib.rs:1211-1230` emits a synthetic failure so a panicked respawn task cannot pin `respawn_in_flight = true` forever |
| `PoolEvent` | `lib.rs:1700-1705` | `Result(Box<PromptResult>)` \| `Panic(JoinError)` \| `SteerAck(SteerAckEvent)` \| `Wake(u32, Result<AgentPool,String>)` | function-local enum used to split the `pool` borrow across `select!` arms |
| `LoopAction` | `lib.rs:2705-2710` | `Continue` \| `Exit` | main-loop control return |
| `PoolStartup` | `lib.rs:3717-3740` | `agents`, `command`, `args`, `extra_env`, `has_generated_codex_config`, `model`, `observer` | owned snapshot of `Config` so lazy-pool wake tasks need no `&Config` borrow |

`PromptResult` is boxed at `lib.rs:1702` (`Box<PromptResult>`) — the only explicitly boxed variant, so the enum is not sized to the largest payload.

#### Main-loop state (locals inside `tokio_main`, no struct wrapper)

| Local | Line | Type | Bound / eviction |
|---|---|---|---|
| `pool` | `lib.rs:1317` | `AgentPool` | fixed to `config.agents` slots |
| `pool_ready` | `lib.rs:1322` | `bool` | lazy-pool readiness gate |
| `pool_lifecycle` | `lib.rs:1323` | `PoolLifecycle<AgentPool>` | listening / waking / ready / failed |
| `startup_watermark` | `lib.rs:1330-1333` | `u64` unix secs | captured *before* relay connect |
| `owner_cache` | `lib.rs:1393` | `OwnerCache` | sibling map cleared wholesale at 256 entries (`lib.rs:184-186`) |
| `subscribed_channel_ids` | `lib.rs:1480` | `HashSet<Uuid>` | **unbounded** |
| `queue` | `lib.rs:1505-1506` | `EventQueue` | `compact_expired_state()` on the 30 s maintenance tick (`lib.rs:1753`) |
| `ctx` | `lib.rs:1530-1573` | `Arc<PromptContext>` | immutable for process life |
| `membership_newest_ts` | `lib.rs:1659` | `HashMap<Uuid,u64>` | **unbounded** |
| `seen_membership_current` / `_previous` | `lib.rs:1662-1663` | `HashSet<String>` | two-generation rotation at 1,000 entries (`lib.rs:1878-1881`) |
| `removed_channels` | `lib.rs:1681` | `HashSet<Uuid>` | entry removed on re-add (`lib.rs:1893`); otherwise **unbounded** |
| `crash_history` | `lib.rs:1688-1695` | `Vec<SlotCircuit>` | sized to `config.agents`, indexed by slot |
| `typing_channels` | `lib.rs:1602` | `HashMap<Uuid, ThreadTags>` | removed on turn completion (`lib.rs:2356`) and on membership removal (`lib.rs:1999`) |
| `heartbeat_in_flight` | `lib.rs:1581` | `bool` | at most one global heartbeat |

`crash_history` is indexed directly (`crash_history[rr.index]` at `lib.rs:1774`, `crash_history[index]` at `lib.rs:3266`), so its length must equal the configured pool capacity — the comment at `lib.rs:1684-1687` states this explicitly. It is sized from `config.agents`, not from `pool.live_count()`.

#### Session/turn identity

- Turn IDs are `Uuid::new_v4().to_string()` minted at dispatch (`lib.rs:2963`) and heartbeat (`lib.rs:3556`); they are harness-local, never derived from an event ID.
- ACP session state lives in `pool::SessionState` on `OwnedAgent`, not in `lib.rs`. `lib.rs` only invalidates it: `result.agent.state.invalidate_channel(ch)` (`lib.rs:3168`) and `pool.invalidate_channel_sessions(ch)` (`lib.rs:1993`, `lib.rs:2131`).
- `OwnedAgent` is constructed in three places with the same eight fields (`lib.rs:1777-1787`, `lib.rs:3799-3808`, `lib.rs:3873-3874` via `spawn_and_init`'s tuple return). `desired_model` is seeded from `config.model` / `startup.model`, and the comment at `lib.rs:3190-3195` records that it does **not** track `session/set_model` overrides — the harness's view of the active model can be stale.

#### Wire types

Nothing typed. Observer frames use `observer::ObserverEvent` (`observer.rs:59-79`), whose `payload` is `serde_json::Value`. Control frames inbound from the owner are decrypted into `serde_json::Value` and probed by string key (`lib.rs:874-893`): `type`, `channelId`, `modelId`. Nostr events are `nostr::Event` throughout; tag access is positional array indexing (`lib.rs:2711-2716`, `lib.rs:326-354`).

#### Size caps encoded as constants

| Constant | Line | Value |
|---|---|---|
| `MODELS_TIMEOUT` | `lib.rs:63` | 10 s |
| `AUTHENTICATE_TIMEOUT` | `lib.rs:67` | 600 s |
| `OBSERVER_PUBLISH_INTERVAL` | `lib.rs:364` | 167 ms |
| `OBSERVER_PUBLISH_LIMIT_PER_MINUTE` | `lib.rs:365` | 90 |
| `OBSERVER_CHUNK_MAX_TEXT_BYTES` | `lib.rs:544` | 60,000 |
| `OBSERVER_LEAF_RETAIN_BYTES` | `lib.rs:632` | 3,000 |
| `OBSERVER_CONTROL_FRESHNESS_SECS` | `lib.rs:835` | 300 |
| `CIRCUIT_BREAKER_THRESHOLD` | `lib.rs:1008` | 3 |
| `CIRCUIT_BREAKER_WINDOW` | `lib.rs:1010` | 60 s |
| `CIRCUIT_BREAKER_COOLDOWN` | `lib.rs:1012` | 300 s |
| `RESPAWN_BASE_DELAY` / `RESPAWN_MAX_DELAY` | `lib.rs:1014`, `lib.rs:1016` | 1 s / 30 s |

`OBSERVER_CHUNK_MAX_TEXT_BYTES` is a soft pre-flush that must be kept in sync with `OBSERVER_MAX_PLAINTEXT_LEN` in `buzz-core/src/observer.rs:25` — the coupling is documented in a comment (`lib.rs:539-543`) with no compile-time or test enforcement.
