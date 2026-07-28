## Module: buzz-acp — harness core & orchestration (`crates/buzz-acp/src`)
### Aspect: Configuration

Config comes from clap-with-`env` (`config.rs`), an optional TOML file, and four env vars read directly by `lib.rs`/`setup_mode.rs`. There are 44 `env = "…"` declarations in `config.rs` (`grep -c 'env = "'`).

#### Env vars read directly by `lib.rs` (bypassing `Config`)

| Var | Site | Default | Gates |
|---|---|---|---|
| `BUZZ_AUTH_TAG` | `lib.rs:125-137`, `lib.rs:1338-1341`, `lib.rs:4173-4179` | unset | (1) owner resolution via NIP-OA attestation, highest priority; (2) relay membership delegation tag on connect; (3) forwarded into the MCP server env. Empty string is filtered out at all three sites. |
| `BUZZ_MANAGED_AGENT_START_NONCE` | `lib.rs:1503` | `""` via `unwrap_or_default()` | correlation ID stamped into every `managed_agent_runtime_lifecycle` observer event (`lib.rs:93-121`) |
| `BUZZ_ACP_SETUP_PAYLOAD` | `setup_mode.rs:83` (const), read `setup_mode.rs:214` | unset | when present, `lib.rs:1298-1303` diverts to the setup listener and the agent pool never starts |
| `RUST_LOG` (implicit) | `EnvFilter::try_from_default_env()` `lib.rs:1278` | `buzz_acp=info` | tracing filter |

#### `Config` vars consumed by `lib.rs`

| Var | `config.rs` | Default | What `lib.rs` gates on it |
|---|---|---|---|
| `BUZZ_PRIVATE_KEY` | 243 | **required** | `config.keys` — relay auth, event signing, MCP injection (`lib.rs:4159`) |
| `BUZZ_RELAY_URL` | 240 | `ws://localhost:3000` | `HarnessRelay::connect` (`lib.rs:1343`), MCP env (`lib.rs:4156`) |
| `BUZZ_ACP_AGENT_COMMAND` | 250 | `goose` | every spawn path (`lib.rs:1762`, `3486`, `3664`, `3731`) |
| `BUZZ_ACP_AGENT_ARGS` | 255 | `acp` | same; comma-delimited |
| `BUZZ_ACP_MCP_COMMAND` | 261 | `""` | `build_mcp_servers` early-returns empty on `""` (`lib.rs:4142-4144`) — **default gives the agent no MCP servers** |
| `BUZZ_ACP_AGENTS` | 292 | `1` (1–32) | pool size (`lib.rs:1318`), `crash_history` length (`lib.rs:1688`), `respawn_tx` capacity (`lib.rs:1613`) |
| `BUZZ_ACP_IDLE_TIMEOUT` | 266 | `DEFAULT_IDLE_TIMEOUT_SECS` = **900** (`config.rs:27`) | `PromptContext.idle_timeout` (`lib.rs:1533`) |
| `BUZZ_ACP_MAX_TURN_DURATION` | 270 | `DEFAULT_MAX_TURN_DURATION_SECS` = 7200 (`config.rs:31`) | `PromptContext.max_turn_duration` (`lib.rs:1534`), `queue.with_in_flight_deadline` (`lib.rs:1506`), steer deadline extension (`lib.rs:2488`), user-facing timeout text (`lib.rs:3092`, `3106`, `3258`) |
| `BUZZ_ACP_TURN_LIVENESS_SECS` | 302 | `10` | `PromptContext.turn_liveness_interval` (`lib.rs:1535`) |
| `BUZZ_ACP_HEARTBEAT_INTERVAL` | 297 | `0` (disabled) | `heartbeat` interval (`lib.rs:1574-1582`); `0` leaves the arm permanently pending |
| `BUZZ_ACP_HEARTBEAT_PROMPT` / `_FILE` | 308, 316 | built-in | `PromptContext.heartbeat_prompt` (`lib.rs:1550`), falls back to `default_heartbeat_prompt()` (`lib.rs:3560`) |
| `BUZZ_ACP_SUBSCRIBE` | 326 | `mentions` | rule construction (`lib.rs:1445-1474`) |
| `BUZZ_ACP_KINDS` | 332 | unset | `kinds_override`; under `mentions` replaces the 9/46010/40007 default (`lib.rs:1447`), under `all` replaces an **empty** list (`lib.rs:1456`) |
| `BUZZ_ACP_CHANNELS` | 335 | unset | `resolve_channel_filters` (`lib.rs:1476`) |
| `BUZZ_ACP_NO_MENTION_FILTER` | 338 | `false` | `require_mention: !no_mention_filter` (`lib.rs:1451`) |
| `BUZZ_ACP_CONFIG` | 341 | `./buzz-acp.toml` | `config::load_rules` under `subscribe=config` (`lib.rs:1471`) |
| `BUZZ_ACP_DEDUP` | 344 | `queue` | `EventQueue::new` (`lib.rs:1505`); `Drop` also disables the panic-recovery batch copy (`lib.rs:2915-2918`) |
| `BUZZ_ACP_MULTIPLE_EVENT_HANDLING` | 355 | `queue` | `mode_gate_signal` (`lib.rs:2233`) |
| `BUZZ_ACP_NO_IGNORE_SELF` | 361 | `false` | self-authored event drop (`lib.rs:2027`) |
| `BUZZ_ACP_CONTEXT_MESSAGE_LIMIT` | 366 | `12` | `PromptContext.context_message_limit` (`lib.rs:1562`) |
| `BUZZ_ACP_MAX_TURNS_PER_SESSION` | 372 | `0` | `PromptContext.max_turns_per_session` (`lib.rs:1563`) |
| `BUZZ_ACP_NO_PRESENCE` | 377 | `false` | initial/heartbeat/offline presence (`lib.rs:1511`, `1583`, `2680`) |
| `BUZZ_ACP_NO_TYPING` | 381 | `false` | 3 s typing tick (`lib.rs:1593`) |
| `BUZZ_ACP_MEMORY` / `BUZZ_ACP_NO_MEMORY` | 395, 404 | memory on | `PromptContext.memory_enabled` (`lib.rs:1568`); disabled logs to target `engram::core` (`lib.rs:1575-1580`) |
| `BUZZ_ACP_NO_BASE_PROMPT` | 409 | `false` | drops the bundled `base_prompt.md` (`lib.rs:1541-1547`) |
| `BUZZ_ACP_BASE_PROMPT_FILE` | 416 | unset | replaces `include_str!("base_prompt.md")`; the loaded string is `Box::leak`ed (`lib.rs:1545`) |
| `BUZZ_ACP_MODEL` | 423 | unset | `OwnedAgent.desired_model` at spawn (`lib.rs:1781`, `3803`), `PoolStartup.model` (`lib.rs:3735`) |
| `BUZZ_ACP_PERMISSION_MODE` | 434 | see `config.rs` | `PromptContext.permission_mode` (`lib.rs:1566`) |
| `BUZZ_ACP_RESPOND_TO` | 444 | `owner-only` | `author_allowed` (`lib.rs:2155`); warnings at `lib.rs:1376-1390` |
| `BUZZ_ACP_RESPOND_TO_ALLOWLIST` | 452 | empty | allowlist set (`lib.rs:2156`) |
| `BUZZ_ACP_ALLOWED_RESPOND_TO` | 460 | empty | only reaches `config.summary()` (`config.rs:1019-1025`) — no `lib.rs` reference |
| `BUZZ_ACP_TEAM_INSTRUCTIONS` | 464 | unset | `PromptContext.team_instructions` (`lib.rs:1539`) |
| `BUZZ_ACP_SYSTEM_PROMPT` / `_FILE` | 279, 286 | unset | `PromptContext.system_prompt` (`lib.rs:1538`) |
| `BUZZ_ACP_INITIAL_MESSAGE` | 321 | unset | `PromptContext.initial_message` (`lib.rs:1532`) |
| `BUZZ_ACP_RELAY_OBSERVER` | 468 | `false` | in-process observer (`lib.rs:1307`), control subscribe + publisher (`lib.rs:1399-1435`) |
| `BUZZ_ACP_LAZY_POOL` | 472 | `false` | empty-slot pool + wake state machine (`lib.rs:1317-1322`, `1709-1748`) |
| `BUZZ_ACP_AGENT_OWNER` | 247 | unset | fallback owner when `BUZZ_AUTH_TAG` is absent or fails (`lib.rs:147`) |

#### Vars read outside `Config`

| Var | Site | Default | Notes |
|---|---|---|---|
| `BUZZ_ACP_EVENT_BUFFER` | `relay.rs:35-41` | `EVENT_CHANNEL_CAPACITY_DEFAULT` | parsed at connect time, clamped to ≥ 1 because `mpsc::channel` panics on 0; declared in `.env.example:221` but has **no clap flag** — env-only |
| `CODEX_CONFIG` | `acp.rs:439-462` | unset | recursively merged with persona-supplied values when `has_generated_codex_config` |

#### Legacy aliases

`propagate_legacy_env_vars()` (`config.rs:715-726`), called from `run()` before the tokio runtime starts (`lib.rs:1234`) because `set_var` needs a single-threaded process under Rust 2024:

| Legacy | Canonical |
|---|---|
| `BUZZ_ACP_PRIVATE_KEY` | `BUZZ_PRIVATE_KEY` |
| `BUZZ_ACP_API_TOKEN` | `BUZZ_API_TOKEN` |

`BUZZ_ACP_TURN_TIMEOUT` (`config.rs:274`, `hide = true`) is a third legacy alias handled by precedence rather than propagation: explicit `--idle-timeout` > `--turn-timeout` > default (`config.rs:848-866`).

#### Parsed-and-never-read

**`BUZZ_API_TOKEN` is dead.** `grep -rn 'BUZZ_API_TOKEN' crates/buzz-acp/src/` returns exactly one hit — the propagation table at `config.rs:718`. There is no clap flag, no `Config` field, no read site. Yet `crates/buzz-acp/README.md § Core` documents it as "API token (required if relay enforces token auth)". The harness propagates the legacy alias into an env var that nothing in the crate consumes.

**`allowed_respond_to` is display-only.** Declared at `config.rs:460`, stored on `Config`, and referenced only by `config.summary()` (`config.rs:1019-1025`). No `lib.rs` gate reads it; `author_allowed` takes `respond_to` and `respond_to_allowlist` only (`lib.rs:2154-2159`).

#### Missing from `.env.example`

`.env.example` documents 25 `BUZZ_ACP_*` / `BUZZ_*` vars (lines 121–226). Absent from it entirely:

`BUZZ_ACP_AGENT_OWNER`, `BUZZ_ACP_IDLE_TIMEOUT`, `BUZZ_ACP_MAX_TURN_DURATION`, `BUZZ_ACP_TURN_LIVENESS_SECS`, `BUZZ_ACP_MULTIPLE_EVENT_HANDLING`, `BUZZ_ACP_MEMORY`, `BUZZ_ACP_NO_MEMORY`, `BUZZ_ACP_NO_BASE_PROMPT`, `BUZZ_ACP_BASE_PROMPT_FILE`, `BUZZ_ACP_PERMISSION_MODE`, `BUZZ_ACP_RESPOND_TO`, `BUZZ_ACP_RESPOND_TO_ALLOWLIST`, `BUZZ_ACP_ALLOWED_RESPOND_TO`, `BUZZ_ACP_TEAM_INSTRUCTIONS`, `BUZZ_ACP_RELAY_OBSERVER`, `BUZZ_ACP_LAZY_POOL`, `BUZZ_AUTH_TAG`, `BUZZ_MANAGED_AGENT_START_NONCE`, `BUZZ_ACP_SETUP_PAYLOAD`.

That is 19 undocumented vars against 25 documented ones. Four of the missing ones are security-relevant (`RESPOND_TO`, `RESPOND_TO_ALLOWLIST`, `PERMISSION_MODE`, `AUTH_TAG`) and two control the entire startup mode (`LAZY_POOL`, `SETUP_PAYLOAD`).

Conversely, `.env.example:152` documents the **deprecated hidden** alias `BUZZ_ACP_TURN_TIMEOUT=320` rather than the canonical `BUZZ_ACP_IDLE_TIMEOUT`, and `.env.example:221` documents `BUZZ_ACP_EVENT_BUFFER=256`, which has no CLI flag.

#### Default-value drift

| Var | Code default | `README.md § Core` |
|---|---|---|
| `BUZZ_ACP_IDLE_TIMEOUT` | **900** (`config.rs:27`) | **620** |
| `BUZZ_ACP_MAX_TURN_DURATION` | 7200 (`config.rs:31`) | 7200 ✓ |
| `BUZZ_ACP_MCP_COMMAND` | `""` (`config.rs:261`) | `""` ✓ |

#### TOML file config

`--config` / `BUZZ_ACP_CONFIG` (default `./buzz-acp.toml`) is loaded only under `subscribe=config` (`lib.rs:1469-1472`). `config::load_rules` is documented at `lib.rs:1470` as already warning on a zero-rule file. Per-channel `[channel.UUID]` blocks with `kinds` and `require_mention` are documented in `README.md § Forum Channels`.

#### Config values that panic or abort on bad input

- `rustls` provider install: `.expect(...)` at `lib.rs:1243`.
- Malformed secret key: `.expect("secret key bech32 encoding should never fail")` at `lib.rs:4167`.
- Fatal-at-startup config errors (relay connect, membership subscribe, observer control subscribe, channel discovery) return `Err` from `tokio_main` (`lib.rs:1345`, `1360`, `1428`, `1440`).
- Zero live agents after eager pool startup is fatal (`lib.rs:3827-3832`); a partial pool is only a warning (`lib.rs:3833-3839`).
- No channel subscriptions resolved is only a warning — "agent will sit idle" (`lib.rs:1477-1479`).
