## Module: buzz-acp — agent pool & lifecycle (`crates/buzz-acp/src`)
### Aspect: Configuration

#### Direct env reads in this module

Zero. Neither `pool.rs` nor `pool_lifecycle.rs` calls `std::env::var`, `env!`, or reads a config file. Every knob arrives pre-resolved as a field on `PromptContext` (`pool.rs:482-532`), as a `PoolStartup` field, or as a compile-time `const`. All parsing happens in `config.rs` via clap `#[arg(long, env = ...)]`; `PromptContext` is assembled once at `lib.rs:1529-1563`.

#### Knobs that reach the pool

| Env var | Flag | Default | Parse site | What it gates in this module |
|---|---|---|---|---|
| `BUZZ_ACP_AGENTS` | `--agents` | `1`, range `1..=32` | `config.rs:292-295` | slot-vector length (`lib.rs:1318`, `:3747`); pool width |
| `BUZZ_ACP_LAZY_POOL` | `--lazy-pool` | `false` | `config.rs:471-473` | whether `PoolLifecycle` gates subprocess start (`lib.rs:1317-1322`, `:1714`) |
| `BUZZ_ACP_IDLE_TIMEOUT` | `--idle-timeout` | `900` s (`DEFAULT_IDLE_TIMEOUT_SECS`, `config.rs:27`) | `config.rs:266-267`, resolved `config.rs:849-866` | `ctx.idle_timeout` — per-turn silence cut and the grace passed to `cancel_with_cleanup` (`pool.rs:1616`, `:1648`, `:2077`) |
| `BUZZ_ACP_TURN_TIMEOUT` | `--turn-timeout` (hidden, deprecated) | none | `config.rs:274-275` | alias for idle timeout; `--idle-timeout` wins (`config.rs:848`) |
| `BUZZ_ACP_MAX_TURN_DURATION` | `--max-turn-duration` | `7200` s (`config.rs:31`) | `config.rs:270-271` | `ctx.max_turn_duration` — hard wall-clock cap (`pool.rs:1617`, `:1835`); must be `>` idle timeout or startup fails (`config.rs:894-898`) |
| `BUZZ_ACP_TURN_LIVENESS_SECS` | `--turn-liveness-secs` | `10` | `config.rs:302-303` | `ctx.turn_liveness_interval`; `0` disables emission (`pool.rs:3172-3174`) |
| `BUZZ_ACP_MAX_TURNS_PER_SESSION` | `--max-turns-per-session` | `0` (disabled) | `config.rs:372-375` | proactive rotation threshold (`pool.rs:1999-2022`) |
| `BUZZ_ACP_CONTEXT_MESSAGE_LIMIT` | `--context-message-limit` | `12`, range `0..=100` | `config.rs:366-369` | `0` skips context fetch entirely (`pool.rs:1746-1750`); otherwise the `limit` on thread/DM filters |
| `BUZZ_ACP_DEDUP` | `--dedup` | `queue` | `config.rs:344` | `requeue_batch_if_queue` — `Drop` discards the batch on every failure (`pool.rs:2973-2979`) and suppresses `recoverable_batch` for panic recovery (`lib.rs:2919-2922`) |
| `BUZZ_ACP_PERMISSION_MODE` | `--permission-mode` | **`bypass-permissions`** (`config.rs:435`) | `config.rs:432-438` | `ctx.permission_mode`; `Default` skips the request (`pool.rs:924-928`) |
| `BUZZ_ACP_MEMORY` / `BUZZ_ACP_NO_MEMORY` | `--memory` / `--no-memory` | memory on | `config.rs:395`, `:404` | `ctx.memory_enabled` — gates the NIP-AE core fetch (`pool.rs:1379`) |
| `BUZZ_ACP_MCP_COMMAND` | `--mcp-command` | `""` | `config.rs:261` | empty ⇒ `ctx.mcp_servers` is empty; otherwise the single server spec cloned into every `session/new` (`lib.rs:4142-4144`, `pool.rs:832`) |
| `BUZZ_ACP_SYSTEM_PROMPT` / `_FILE` | `--system-prompt(-file)` | none | `config.rs:279`, `:286` | `[System]` section (`pool.rs:1137-1157`) |
| `BUZZ_ACP_NO_BASE_PROMPT` / `BUZZ_ACP_BASE_PROMPT_FILE` | `--no-base-prompt` / `--base-prompt-file` | compiled-in `base_prompt.md` | `config.rs:409`, `:416` | `ctx.base_prompt`; `Box::leak`ed to `'static` (`lib.rs:1539-1544`) |
| `BUZZ_ACP_TEAM_INSTRUCTIONS` | `--team-instructions` | none | `config.rs:464` | `[Team Instructions]` section (`pool.rs:1180-1197`) |
| `BUZZ_ACP_INITIAL_MESSAGE` | `--initial-message` | none | `config.rs:321` | first prompt on brand-new channel sessions (`pool.rs:1580-1712`) |
| `BUZZ_ACP_MODEL` | `--model` | none | `config.rs:423` | seeds `OwnedAgent::desired_model` at every spawn/respawn (`lib.rs:1794`, `:3799`) |
| `BUZZ_ACP_AGENT_COMMAND` | `--agent-command` | `goose` | `config.rs:191`, `:250` | the spawned binary (via `PoolStartup`); also normalized into `ctx.harness_name` (`lib.rs:1561`) |
| `BUZZ_ACP_AGENT_ARGS` | `--agent-args` | see `config.rs:197` | `config.rs:197`, `:255` | child argv |
| `BUZZ_ACP_RELAY_OBSERVER` | `--relay-observer` | `false` | `config.rs:468` | whether an `ObserverHandle` exists; with none, `turn_started`/`turn_liveness`/`turn_completed` emit nothing (`pool.rs:3169-3171`, `:3291`) |
| `BUZZ_ACP_HEARTBEAT_INTERVAL` / `_PROMPT` / `_PROMPT_FILE` | `--heartbeat-*` | `0` = off | `config.rs:297`, `:308`, `:316` | `ctx.heartbeat_prompt`; heartbeat turns take `control_rx = None` (`pool.rs:1827-1836`) |
| `BUZZ_RELAY_URL` | `--relay-url` | `ws://localhost:3000` | `config.rs:240` | `ctx.relay_url` in observer payloads (`pool.rs:918`) and the MCP server env |
| `BUZZ_PRIVATE_KEY` | `--private-key` | none | `config.rs:243` | `ctx.agent_keys`; bech32-encoded into the MCP server env (`lib.rs:4160-4170`) |
| `BUZZ_AUTH_TAG` | — | none | read at `lib.rs:125`, `:1338`, `:4173` | forwarded into the MCP server env when non-empty |
| `BUZZ_ACP_PRIVATE_KEY` | — | — | alias mapped to `BUZZ_PRIVATE_KEY` (`config.rs:717`) | legacy compatibility |

Implicit, non-env inputs: `ctx.cwd` from `std::env::current_dir()` with a `/` fallback (`lib.rs:1547-1550`), and `ctx.agent_owner_pubkey` derived from the resolved owner (`lib.rs:1557-1559`) — when it is `None`, both NIP-AE core injection and NIP-AM metric publishing are skipped silently (`pool.rs:1380`, `:3336-3339`).

#### Hard-coded values with no configuration path

| Constant | Value | Site | Gates |
|---|---|---|---|
| `INITIAL_RETRY_DELAY` | 5 s | `pool_lifecycle.rs:11` | first lazy-wake retry delay |
| `MAX_RETRY_DELAY` | 300 s | `pool_lifecycle.rs:12` | wake retry ceiling |
| `RECENT_ACTIVITY_WINDOW` | 60 s | `pool.rs:45` | hard-cap `recently_active` classification (requeue vs dead-letter eligibility) |
| `CONTEXT_FETCH_TIMEOUT` | 3000 ms | `pool.rs:780` | every context/profile/channel-info fetch attempt |
| `CONTEXT_FETCH_RETRY_DELAY` | 500 ms | `pool.rs:783` | gap before the single retry |
| `MODEL_SWITCH_TIMEOUT` | 5 s | `pool.rs:786` | model-switch request |
| `CONTROL_CANCEL_GRACE` | 5 s | `pool.rs:793` | post-cancel drain deadline |
| `PERMISSION_MODE_TIMEOUT` | 5 s | `pool.rs:796` | permission-mode request |
| `CORE_FETCH_TIMEOUT` | 3 s | `pool.rs:1386` | NIP-AE core fetch |
| `CANVAS_FETCH_TIMEOUT` | 3 s | `pool.rs:2316` | canvas fetch |
| `METRIC_TIMEOUT` | 3 s | `pool.rs:3415` | NIP-AM publish |
| `REACTION_TIMEOUT` | 500 ms | `pool.rs:3437` | reaction add |
| `REACTION_CONCURRENCY` | 10 | `pool.rs:3618` | reaction fan-out width |
| reaction-remove query/delete | 1000 ms each | `pool.rs:3551`, `:3609` | inline, not named |
| failure-notice timeout | 5 s | `pool.rs:3528` | inline, not named |
| init timeout | 60 s per agent | `lib.rs:3759` | spawn+initialize per slot |
| shutdown grace | 30 s | `lib.rs:2608` | in-flight prompt drain |
| wake drain grace | 30 s | `lib.rs:2591` | lazy-wake task drain |
| circuit breaker | 3 crashes / 60 s, 300 s cooldown, 1 s→30 s backoff | `lib.rs:1008-1016` | per-slot respawn rate |

The 5 s / 300 s wake backoff is the module's most consequential un-tunable pair: with `--lazy-pool` and a persistently failing agent, first-response latency after a quiet period is capped at 5 minutes with no way to shorten or lengthen it.

#### Missing from `.env.example`

`.env.example` mentions `BUZZ_ACP_*` on 25 lines, spanning `.env.example:134-226`. Absent entirely, verified by grep:

`BUZZ_ACP_IDLE_TIMEOUT`, `BUZZ_ACP_MAX_TURN_DURATION`, `BUZZ_ACP_TURN_LIVENESS_SECS`, `BUZZ_ACP_LAZY_POOL`, `BUZZ_ACP_RELAY_OBSERVER`, `BUZZ_ACP_PERMISSION_MODE`, `BUZZ_ACP_NO_MEMORY`, `BUZZ_ACP_MULTIPLE_EVENT_HANDLING`.

Worse than absent: `.env.example:152` documents `BUZZ_ACP_TURN_TIMEOUT=320` — the **deprecated, `hide = true` alias** (`config.rs:274-275`) — while the current name `BUZZ_ACP_IDLE_TIMEOUT` appears nowhere. An operator following `.env.example` configures a hidden deprecated flag, and the value shown (320) does not match either code default (900 idle / 7200 hard cap).

`crates/buzz-acp/README.md:117-118` does document `--agents` and `--lazy-pool` correctly, including the lazy-wake retry behaviour, so the drift is specific to `.env.example`.

#### Parsed-and-never-read

`.env.example:221` documents `BUZZ_ACP_EVENT_BUFFER=256`, which is read by `relay.rs:36` and never reaches the pool — the only `BUZZ_ACP_*` var in `.env.example` that is not a clap arg. No pool knob is parsed and dropped: every `PromptContext` field is consumed somewhere in `pool.rs`. Two adjacent observations:

- `PromptContext::heartbeat_prompt` (`pool.rs:501`) is stored on the shared context but the heartbeat prompt text is composed in `lib.rs::dispatch_heartbeat` (`lib.rs:3537-3580`) and passed in as `prompt_text`; `pool.rs` never reads the field. It is a carried-but-unused field from this module's perspective.
- `PoolLifecycle::failed_error()` (`pool_lifecycle.rs:84-89`) has exactly one caller, a `debug_assert_eq!` (`lib.rs:2565`), so it is dead in release builds.

#### Validation performed at config time (relevant to pool behaviour)

`--agents` is range-checked `1..=32` by clap (`config.rs:293`). `context_message_limit` is range-checked `0..=100` (`config.rs:368`). `idle_timeout_secs` must be strictly less than `max_turn_duration_secs`, else `Config::from_cli` errors with the reason that otherwise the idle timeout would be "a dead letter" (`config.rs:891-899`). No validation exists for `turn_liveness_secs` relative to the timeouts, and none for `max_turns_per_session`.
