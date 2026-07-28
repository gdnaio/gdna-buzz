## Module: buzz-acp — agent pool & lifecycle (`crates/buzz-acp/src`)
### Aspect: Features

#### Take-and-return pooling

The pool holds `1..=32` agent subprocesses (`config.rs:292-295`) in a fixed-length slot vector and moves ownership out on claim, back on return (`pool.rs:206-219`). `AcpClient` is not `Clone`, so the compiler enforces single-ownership of each child's stdio (`pool.rs:22`). Reachable and exercised on every dispatch (`lib.rs:2907`).

#### Channel affinity (session reuse)

`try_claim(Some(channel_id))` prefers an idle agent that already holds an ACP session for that channel, avoiding a `session/new` round-trip and re-fetch of core/canvas (`pool.rs:558-568`). `has_session_for` (`pool.rs:599-607`) computes the `affinity_hit` flag logged alongside `agent_claimed` (`lib.rs:2918`). Reachable. With the default `--agents 1` affinity is trivially satisfied by the single slot; the feature only has an effect at `agents > 1`.

#### Warm reuse of sessions across turns

A session survives between turns: `run_prompt_task` reuses `state.sessions[cid]` when present (`pool.rs:1468-1470`) and only creates a new one when absent. Cached alongside it are the rendered NIP-AE core block and the `[Channel Canvas]` block, both fetched "once per new channel session" and cleared on invalidation (`pool.rs:1379-1416`, `:1428-1449`). This is the module's only warm-start mechanism.

#### Deferred (lazy) pool start

`--lazy-pool` / `BUZZ_ACP_LAZY_POOL` (`config.rs:471-473`) constructs an all-`None` pool (`lib.rs:1318`) and defers subprocess spawning until the first accepted event is buffered; `PoolLifecycle` then runs exactly one initialization task with bounded exponential backoff on failure (`pool_lifecycle.rs:37-58`, `lib.rs:1714-1743`). Reachable, off by default. Runtime state is surfaced to the desktop as `managed_agent_runtime_lifecycle` frames with `waking`/`ready`/`failed` (`lib.rs:1721-1728`, `:2550-2557`, `:2566-2573`).

There is no partial or per-slot warm start: the wake initializes the whole pool in one task (`initialize_agent_pool`, `lib.rs:3741`), sequentially per slot, each spawn+init bounded at 60 s (`lib.rs:3766`). With `--agents 32` and slow-starting agents, first-response latency is up to 32 × 60 s.

#### Partial-pool tolerance

`initialize_agent_pool` pushes `None` for any slot that fails to spawn or initialize and continues (`lib.rs:3812-3823`), erroring only when zero agents came up (`lib.rs:3825-3830`) and warning on a reduced pool (`lib.rs:3832-3838`). `from_slots` preserves positions so index-to-slot alignment survives the gaps (`pool.rs:536-556`). Empty slots are later refilled by the maintenance pass when the slot's circuit allows (`lib.rs:1748-1768`).

#### Per-agent isolation

Isolation is per **process**, not per channel or per user:

- Each `OwnedAgent` owns its own child, stdio pipes, and `SessionState` (`pool.rs:150-171`).
- All agents in the pool run the same command, args, and env — one `PoolStartup` snapshot from `Config` (`lib.rs:3717-3737`) — so there is no per-agent policy, model, cwd, or credential differentiation.
- All N agents authenticate as the **same** Nostr identity (`README.md:197`); `ctx.agent_keys` is a single `nostr::Keys` clone shared by every task (`lib.rs:1557`).
- `ctx.cwd` is one process-wide value from `std::env::current_dir()` (`lib.rs:1547-1550`), so every agent and every channel shares one working directory.
- Cross-channel state does leak within one agent: `state.sessions`, `turn_counts`, `core_sections`, and `canvas_sections` are per-agent maps keyed by channel (`pool.rs:86-104`), and `agent_name`/`goose_system_prompt_supported`/`model_capabilities` are process-wide latches.

#### Mid-turn message delivery (steer)

Two paths. The native path sends a Goose `session/steer`-style request into the running read loop over a capacity-1 channel installed per turn (`pool.rs:646-662`, install at `lib.rs:2937-2938`), with the read loop filling `expectedRunId` at write time because it is a moving target (`pool.rs:296-318`). The universal fallback is `ControlSignal::Steer` — cancel the turn, requeue the batch stamped `CancelReason::Steer`, and re-prompt with merge framing (`pool.rs:2986-3004`). Both reachable; the steer channel is installed for **every** prompt task regardless of agent, using try-and-tolerate on `-32601` (`lib.rs:2930-2939`, `pool.rs:339-346`).

#### Runtime model switching

Two entry points. Busy path: `ControlSignal::SwitchModel(id)` sets `desired_model` + `model_overridden` before cancelling, so the requeued turn re-creates the session under the new model (`pool.rs:1855-1858`). Idle path: `switch_idle_agent_model` validates against the agent's cached catalog *before* invalidating, returning `UnsupportedModel` without disturbing the session (`pool.rs:732-762`). Both reachable from the observer control channel (`lib.rs:981`). Runtime-only — never persisted; reset on respawn because every `OwnedAgent` construction passes `desired_model: config.model.clone()` and `model_overridden: false` (`lib.rs:1793-1795`, `:3799-3801`).

Acknowledged gap in the code's own doc: an idle-path switch does not re-emit `session_config_captured`, so the desktop panel does not reflect it until the agent next runs a turn (`pool.rs:723-731`).

#### Proactive session rotation

Rotate on `MaxTokens`/`MaxTurnRequests`, or after `max_turns_per_session` successful turns when that knob is non-zero (`pool.rs:1994-2022`). Default is `0` = rotate only on stop-reason (`config.rs:372-375`), so the turn-count path is off unless configured.

#### Per-turn observability

`turn_started` (`pool.rs:1295`), `session_resolved` (`pool.rs:1573`), `session_config_captured` (`pool.rs:908`), `control_result` on an unsupported model (`pool.rs:888`), periodic `turn_liveness` (`pool.rs:3194`), and `turn_completed` from a drop guard covering every exit path including panic (`pool.rs:3291-3301`). All gated on `--relay-observer` producing an `ObserverHandle` (`lib.rs:1299-1301`); with the observer absent, `run_turn_liveness` parks forever and emits nothing (`pool.rs:3169-3171`).

Per-turn cost/token metrics are published as encrypted NIP-AM kind `44200` events, but only when the agent emitted usage **and** an owner pubkey is configured — otherwise `publish_agent_turn_metric` returns immediately (`pool.rs:3336-3339`).

#### Two-phase user-visible reaction lifecycle

👀 "seen" is added by `lib.rs` at queue-push time; 💬 "working" is added by the pool just before the prompt fires (`pool.rs:1802-1809`); both are removed by `ReactionGuard` on any exit path (`pool.rs:3125-3141`). Fan-out is chunked at `REACTION_CONCURRENCY = 10` (`pool.rs:3618-3648`). The code documents the accepted cosmetic race where a fast failure clears before the 👀 add lands (`pool.rs:3104-3110`).

#### Not present

- No idle-timeout eviction or scale-to-zero of individual agents; the only way a live agent's process ends is crash, poisoning, or harness shutdown (`pool.rs:534-763` contains no reaper).
- No dynamic resize: `agents` is allocated once and there is no add/remove-slot API.
- No persona-driven spawning. `buzz-persona` is a declared dependency (`Cargo.toml:22`) with zero references anywhere under `crates/buzz-acp/src`; the pool receives only `config.persona_env_vars` as opaque `(String, String)` pairs by way of `PoolStartup` (`lib.rs:3721`, `:3733`). Persona-defined `mcp_servers` never reach this module — `PromptContext::mcp_servers` is built solely from `config.mcp_command` (`lib.rs:4141-4184`).
- No per-agent MCP server sets: `ctx.mcp_servers` is one shared `Vec` cloned into every `session/new` (`pool.rs:832`).
