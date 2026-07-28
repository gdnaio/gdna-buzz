## Module: buzz-acp — agent pool & lifecycle (`crates/buzz-acp/src`)
### Aspect: API Surface

#### Visibility reality

Both files are private modules: `mod pool;` (`lib.rs:7`) and `mod pool_lifecycle;` (`lib.rs:8`). The crate's only public re-export is `pub use usage::TurnUsage` (`lib.rs:13`). Every `pub` item in `pool.rs` is therefore reachable only inside `buzz_acp` — the `pub` keyword is decorative at the crate boundary, and `pool.rs` mixes `pub`, `pub(crate)`, and private for items with identical reach (e.g. `pub fn try_claim` at `pool.rs:558` vs `pub(crate) fn prepend_base_for_legacy` at `pool.rs:1090`).

`pool_lifecycle.rs` is uniformly `pub(crate)` (`pool_lifecycle.rs:14`, `:28`, `:37`, `:60`, `:70`, `:77`, `:84`, `:91`, `:99`).

#### `AgentPool` inherent methods (`impl` block `pool.rs:534-763`)

| Method | Site | Signature summary |
|---|---|---|
| `from_slots` | `pool.rs:541` | `Vec<Option<OwnedAgent>> -> Self` — only constructor |
| `try_claim` | `pool.rs:558` | `Option<Uuid> -> Option<OwnedAgent>`; two-pass (affinity, then any idle) |
| `return_agent` | `pool.rs:577` | `OwnedAgent -> ()`; logs `BUG:` and overwrites on double-return |
| `any_idle` | `pool.rs:593` | `-> bool` |
| `has_session_for` | `pool.rs:599` | `Uuid -> bool`; used to compute `affinity_hit` before claiming |
| `live_count` | `pool.rs:610` | `-> usize` = idle slots + `task_map.len()` |
| `task_map` / `task_map_mut` | `pool.rs:616`, `:620` | raw `&`/`&mut HashMap<task::Id, TaskMeta>` |
| `send_steer` | `pool.rs:646` | `(Uuid, SteerRequest) -> Result<(), SteerError>` |
| `result_tx` | `pool.rs:664` | clone of the unbounded sender for a new task |
| `rx_and_join_set` | `pool.rs:671` | split-borrow `(&mut Receiver, &mut JoinSet)` for one `select!` |
| `result_rx_try_recv` | `pool.rs:679` | non-blocking drain, shutdown-only |
| `slot_alive` | `pool.rs:686` | `usize -> bool` (idle **or** checked out) |
| `agents_mut` | `pool.rs:695` | `&mut Vec<Option<OwnedAgent>>` |
| `invalidate_channel_sessions` | `pool.rs:707` | `Uuid -> usize`; idle agents only |
| `switch_idle_agent_model` | `pool.rs:732` | `(Uuid, &str) -> IdleSwitchResult` |

`task_map_mut` and `agents_mut` hand the caller unrestricted mutable access to the pool's two invariant-bearing structures, so the slot/index invariant is enforced by convention in `lib.rs`, not by the type.

#### Free functions exported from `pool.rs`

| Item | Site | Consumer |
|---|---|---|
| `pub async fn run_prompt_task` | `pool.rs:1265` | spawned by `dispatch_pending` (`lib.rs:2947-2957`) and `dispatch_heartbeat` (`lib.rs:3537-3580`) |
| `pub(crate) fn prepend_base_for_legacy` | `pool.rs:1090` | `lib.rs` heartbeat path + own tests |
| `pub(crate) fn prepend_canvas_for_legacy` | `pool.rs:1110` | `run_prompt_task` initial-message path |
| `pub(crate) async fn fetch_channel_info` | `pool.rs:2237` | `ChannelInfoResolver::resolve` and `lib.rs` |
| `pub(crate) fn canvas_section_from_query_response` | `pool.rs:2366` | `fetch_canvas_section`, unit tests |
| `pub(crate) fn render_canvas_section` | `pool.rs:2479` | canvas rendering, unit tests |
| `pub(crate) async fn reaction_add` | `pool.rs:3462` | `react_working`, `lib.rs` 👀 path |
| `pub(crate) async fn reaction_remove` | `pool.rs:3540` | `clear_reactions` |
| `pub(crate) async fn post_failure_notice` | `pool.rs:3495` | `spawn_failure_notice` (`lib.rs:3014`) |
| `pub(crate) fn OwnedAgent::has_system_prompt_support` | `pool.rs:198` | prompt framing decisions |

`run_prompt_task` is the module's real entry point: 948 lines (`pool.rs:1265-2212`) with 20 distinct `send_prompt_result` exit sites (`pool.rs:1496`, `1509`, `1538`, `1549`, `1630`, `1656`, `1674`, `1692`, `1708`, `1789`, `1884`, `1920`, `1975`, `2038`, `2060`, `2094`, `2120`, `2145`, `2174`, `2201`).

Twelve private helpers are internal to the turn path and never named outside the file: `create_session_and_apply_model` (`pool.rs:804`), `apply_model_switch` (`:939`), `agent_supports_mode` (`:1015`), `apply_permission_mode` (`:1032`), `framed_system_prompt` (`:1137`), `workspace_section` (`:1165`), `with_team`/`with_core`/`with_canvas` (`:1180`, `:1199`, `:1213`), `send_prompt_result` (`:1235`), `requeue_batch_if_queue` (`:2973`), `requeue_cancelled_batch` (`:2986`), `classify_control_cancel_failure` (`:3029`), `publish_agent_turn_metric` (`:3322`).

#### `PoolLifecycle<P>` surface

| Method | Site | Contract |
|---|---|---|
| `listening()` | `pool_lifecycle.rs:28` | initial state |
| `start_wake_if_due(has_pending_work, now) -> Option<u32>` | `pool_lifecycle.rs:37` | returns the attempt token exactly once per entry into `Waking` |
| `take_ready() -> Option<P>` | `pool_lifecycle.rs:60` | moves the pool out, resets to `Listening` |
| `waking_attempt() -> Option<u32>` | `pool_lifecycle.rs:70` | read-only probe |
| `retry_at() -> Option<Instant>` | `pool_lifecycle.rs:77` | drives the retry sleep arm |
| `failed_error() -> Option<&str>` | `pool_lifecycle.rs:84` | only consumer is a `debug_assert_eq!` (`lib.rs:2565`) |
| `cancel_wake(attempt, error, now) -> bool` | `pool_lifecycle.rs:91` | thin wrapper over `complete_wake(.., Err(..))` |
| `complete_wake(attempt, Result<P,String>, now) -> Result<(), &'static str>` | `pool_lifecycle.rs:99` | rejects stale/duplicate results |

The two rejection strings are part of the contract and asserted verbatim in tests: `"wake result attempt did not match Waking attempt"` (`pool_lifecycle.rs:106`) and `"wake completed while lifecycle was not Waking"` (`pool_lifecycle.rs:107`).

#### How `lib.rs` drives the module

Construction forks on `config.lazy_pool` (`lib.rs:1317-1321`): the lazy path builds an all-`None` pool of `config.agents` slots, the eager path calls `initialize_agent_pool` (`lib.rs:3741-3841`) which spawns and initializes each agent before `from_slots`.

Main-loop wiring (`lib.rs:1698-1880`): a local `enum PoolEvent { Result, Panic, SteerAck, Wake }` (`lib.rs:1703-1708`) is produced by a single `biased` `select!` over `pool.rx_and_join_set()` (`lib.rs:1821`), the steer-ack channel, the wake channel, and the `retry_at` sleep. Dispatch of the resulting event goes to `handle_prompt_result` (`lib.rs:3034`), `recover_panicked_agent` (`lib.rs:3402`), or the `Wake` arm (`lib.rs:2535-2575`).

Turn dispatch (`lib.rs:2889-2981`): `has_session_for` → `try_claim(Some(channel_id))` → clone batch for `recoverable_batch` when `DedupMode::Queue` → install a capacity-1 steer channel on the client (`lib.rs:2937-2938`) → `join_set.spawn(run_prompt_task(..))` → insert `TaskMeta` keyed by the returned `AbortHandle::id()` (`lib.rs:2960-2970`). Heartbeats take the same shape with `try_claim(None)` and `control_rx = None` (`lib.rs:3538-3580`).

Shutdown ordering (`lib.rs:2590-2678`): fire the shutdown watch → drain `wake_tasks` under a 30 s timeout → drain `join_set` + `result_rx` under a 30 s grace, calling `acp.shutdown()` on each returned agent → `join_set.shutdown()` if the grace expires → `result_rx_try_recv` sweep → `acp.shutdown()` on remaining idle slots → `drop(pool)` → `respawn_tasks.shutdown()`.
