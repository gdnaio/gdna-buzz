## Module: buzz-acp — harness core & orchestration (`crates/buzz-acp/src`)
### Aspect: Features

#### End-to-end startup sequence (`tokio_main`, `lib.rs:1239-1700`)

| Step | Line | Notes |
|---|---|---|
| install rustls ring provider | `lib.rs:1241-1243` | `.expect(...)` — panics if it fails |
| subcommand short-circuit | `lib.rs:1245-1274` | `models` / `auth-methods` / `authenticate` exit before tracing init |
| tracing init | `lib.rs:1276-1281` | `EnvFilter` from env, default `buzz_acp=info` |
| `Config::from_cli()` | `lib.rs:1290` | |
| setup-mode branch | `lib.rs:1298-1303` | `BUZZ_ACP_SETUP_PAYLOAD` present ⇒ `setup_mode::run_setup_listener`, agent pool never starts |
| observer handle | `lib.rs:1307-1316` | in-process only when `config.relay_observer` |
| agent pool | `lib.rs:1317-1321` | eager `initialize_agent_pool` unless `lazy_pool` |
| startup watermark | `lib.rs:1330-1333` | captured **before** relay connect |
| relay connect (NIP-42) | `lib.rs:1343-1346` | `HarnessRelay::connect` with optional NIP-OA tag |
| `set_startup_watermark` | `lib.rs:1352-1354` | best-effort; failure only loses the startup-window guard |
| membership subscription | `lib.rs:1358-1361` | fatal on error |
| owner resolution | `lib.rs:1370-1393` | |
| observer control subscription | `lib.rs:1395-1435` | only if owner resolved |
| channel discovery | `lib.rs:1437-1443` | fatal on error |
| rule construction | `lib.rs:1445-1474` | `mentions` / `all` / `config` |
| per-channel subscribe | `lib.rs:1480-1489` | per-channel failure is a warning, not fatal |
| presence `online` | `lib.rs:1511-1515` | published after subscriptions as a readiness boundary |
| build `PromptContext` | `lib.rs:1530-1573` | |

#### Wired capabilities

| Capability | Evidence | State |
|---|---|---|
| Mention-triggered turns | rules built `lib.rs:1445-1454`, matched `lib.rs:2172-2181` | wired |
| Per-channel batching (drain backlog → one prompt) | `queue.flush_next()` `lib.rs:2892` | wired |
| N parallel agent subprocesses (1–32) | `initialize_agent_pool` `lib.rs:3741-3846` | wired |
| Session affinity per channel | `pool.try_claim(Some(channel_id))` `lib.rs:2902`, `affinity_hit` logged `lib.rs:2911` | wired |
| Partial pool startup tolerated | `lib.rs:3826-3840` — zero live agents is fatal, fewer than requested is a warning | wired |
| Lazy pool (subscribe first, spawn on first accepted event) | `lib.rs:1317-1321`, wake path `lib.rs:1709-1748`, `PoolEvent::Wake` `lib.rs:2502-2545` | wired |
| Circuit-broken respawn with backoff | `SlotCircuit` `lib.rs:1048-1134`, `spawn_respawn_task` `lib.rs:3635-3684` | wired |
| 30 s maintenance tick (queue compaction + slot refill + retry flush) | `lib.rs:1750-1794` | wired |
| Panic recovery (requeue + unwedge channel + respawn) | `recover_panicked_agent` `lib.rs:3402-3499` | wired |
| Heartbeat prompts | `dispatch_heartbeat` `lib.rs:3537-3586` | wired, off by default (interval 0) |
| Presence 20001 + 60 s heartbeat + offline on exit | `lib.rs:77-91`, `1586-1592`, `2300-2320`, `2673-2685` | wired |
| Typing indicators 20002 on a 3 s tick | `lib.rs:1593-1599`, `2321-2341` | wired |
| 👀 reaction on accept, cleaned on drain | `lib.rs:2204-2213`, `1934-1944` | wired, fire-and-forget |
| Owner text commands `!shutdown` / `!cancel` / `!rotate` | `lib.rs:2033-2133` | wired |
| Encrypted observer telemetry (kind 24200, NIP-44 to owner) | `lib.rs:790-833` | wired, requires `--relay-observer` **and** a resolved owner |
| Encrypted observer control (`cancel_turn`, `switch_model`) | `lib.rs:837-1005` | wired |
| Non-cancelling ACP steer with cancel+merge fallback | `try_native_steer` `lib.rs:2803-2887`, ack arm `lib.rs:2417-2500` | wired |
| Auth-error fast dead-letter | `is_auth_error` `lib.rs:3003-3011`, used `lib.rs:3118` | wired |
| User-visible failure notices in-channel | `spawn_failure_notice` `lib.rs:3014-3031` | wired for hard timeout, retries-exhausted, and auth errors |
| Dynamic channel join/leave via membership notifications | `lib.rs:1861-1949` | wired |
| Graceful shutdown (SIGINT + SIGTERM, 30 s grace, child reaping) | `lib.rs:1635-1650`, `2588-2669` | wired |
| MCP server handed to the agent | `build_mcp_servers` `lib.rs:4141-4184` | wired, but only when `BUZZ_ACP_MCP_COMMAND` is non-empty — **default is empty** (`config.rs:261`), so a stock run gives the agent zero MCP servers |
| `models` / `auth-methods` / `authenticate` probe subcommands | `lib.rs:4005`, `3899`, `3947` | wired, undocumented in README |
| Setup-listener mode | `setup_mode::run_setup_listener` `lib.rs:1298-1303` | wired |

#### Declared but not reachable / not exercised

| Item | Evidence |
|---|---|
| `buzz-persona` dependency | declared `Cargo.toml:22`; `grep -rn 'buzz_persona' crates/buzz-acp/` returns only the manifest line. The `persona_env_vars` plumbing (`lib.rs:1762`, `3488`, `3666`, `3733`) is a plain `Vec<(String,String)>` built in `config.rs:945-999` — it does not call into the crate. |
| `pub use usage::TurnUsage` | `lib.rs:15`; no reference to `TurnUsage` anywhere else in `lib.rs`. Exported for external consumers only. |
| Owner re-resolution after startup | `OwnerCache.pubkey` has no setter (`lib.rs:161-163`). With `respond-to=owner-only` and no owner at boot, the harness drops every event for its whole life; the warning at `lib.rs:1379-1384` is the only signal. |
| Relay observer without an owner | `lib.rs:1421-1425` logs a warning and leaves the feature silently off, even though `--relay-observer` was explicitly requested. |
| `RespawnResult` third tuple element (`supports_goose_steer`) | Documented at `lib.rs:1142-1145` as "always `true`" because the supervisor uses try-and-tolerate. The field it describes no longer exists in the tuple, which is `(AcpClient, u32, String)` — the doc comment is stale. |
| `SubscribeMode::All` default kinds | `lib.rs:1456` yields an **empty** kinds vector when `--kinds` is absent, which per `AGENTS.md § Common Gotchas #2` trips the relay p-gate (403). Nothing in `lib.rs` warns about this. |

#### Recovery / resilience behaviours

- Relay stream end triggers `relay.reconnect()`; only a dead background task exits the loop (`lib.rs:2295-2303`).
- Requeued batches are re-flushed on three triggers so quiet channels don't stall: the maintenance tick (`lib.rs:1788-1793`), a completed respawn collection (`lib.rs:1795-1801`), and every pool result (`lib.rs:2390`). The comments at `lib.rs:1783-1787` and `1795-1798` state the failure mode this fixes.
- Shutdown reaps in four stages so `AcpClient::Drop` (start_kill + try_wait, best-effort) is never the primary path: drain `join_set` + `result_rx` under a 30 s grace (`lib.rs:2608-2636`), drain late results (`lib.rs:2643-2648`), reap idle slots (`lib.rs:2649-2661`), then abort and drain respawn tasks (`lib.rs:2663-2672`).
- Wake tasks are *drained*, not aborted, on shutdown — the comment at `lib.rs:2573-2589` explains that aborting mid-init would drop partially spawned `AcpClient`s and create exactly the zombies the eager path avoids. A 30 s timeout is the backstop before falling back to `shutdown()`.
