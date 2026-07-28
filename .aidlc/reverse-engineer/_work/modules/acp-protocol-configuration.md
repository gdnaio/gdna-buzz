## Module: buzz-acp — ACP protocol, config & setup mode (`crates/buzz-acp/src`)
### Aspect: Configuration

Configuration is CLI-first: 43 `env = "…"` attributes in `config.rs`, every one paired with a long flag (`config.rs:1-4`). A TOML file supplies subscription rules only. Precedence is clap's: explicit flag > env var > `default_value`.

#### Authoritative env-var table — clap-parsed (`CliArgs`, `config.rs:234-474`)

| Env var | Flag | Default | Validation / effect |
|---|---|---|---|
| `BUZZ_RELAY_URL` | `--relay-url` | `ws://localhost:3000` (`:240`) | Not validated as a URL here; only `codex_network_env` parses it (`:654`), and a parse failure there just skips Codex injection |
| `BUZZ_PRIVATE_KEY` | `--private-key` | **required** (`:243`) | `Keys::parse` → `ConfigError::KeyParse` (`:741`); raw string then zeroed + cleared (`:742-748`) |
| `BUZZ_ACP_AGENT_OWNER` | `--agent-owner` | — (`:247`) | Trimmed + lowercased (`:1003`); **no hex/length validation**, unlike the allowlist |
| `BUZZ_ACP_AGENT_COMMAND` | `--agent-command` | `goose` (`:250`) | Trim-empty → error (`:808-812`); resolved via inherited `PATH` |
| `BUZZ_ACP_AGENT_ARGS` | `--agent-args` | `acp`, comma-delimited (`:253-258`) | `normalize_agent_args` (`:679-706`) |
| `BUZZ_ACP_MCP_COMMAND` | `--mcp-command` | `""` (`:261`) | Empty → `build_mcp_servers` returns `[]` (`lib.rs:4142-4144`), so a stock run gives the agent **zero MCP servers** |
| `BUZZ_ACP_IDLE_TIMEOUT` | `--idle-timeout` | unset → `900` (`:266`, `:27`) | `0` → warn + clamp to `1` (`:868-873`); must be `<` max turn duration (`:894-899`) |
| `BUZZ_ACP_MAX_TURN_DURATION` | `--max-turn-duration` | `7200` (`:270`, `:31`) | `0` → warn + clamp to `60`; `>604_800` → **error** (`:876-890`) |
| `BUZZ_ACP_TURN_TIMEOUT` | `--turn-timeout` | — (`:274`) | **Hidden (`hide = true`) and deprecated.** Alias for idle timeout; loses to `--idle-timeout`, warns either way (`:849-874`) |
| `BUZZ_ACP_SYSTEM_PROMPT` | `--system-prompt` | — (`:277-281`) | `conflicts_with = "system_prompt_file"`; also wins in code (`:750-756`) |
| `BUZZ_ACP_SYSTEM_PROMPT_FILE` | `--system-prompt-file` | — (`:284-288`) | Read with `fs::read_to_string` → `ConfigError::Io`; **no size cap** (unlike base prompt) |
| `BUZZ_ACP_AGENTS` | `--agents` | `1`, range `1..=32` (`:292-293`) | Enforced by clap `value_parser` |
| `BUZZ_ACP_HEARTBEAT_INTERVAL` | `--heartbeat-interval` | `0` = disabled (`:297`) | `>0 && <10` → **error** (`:758-762`); `>86400` → warn + clamp (`:826-834`) |
| `BUZZ_ACP_TURN_LIVENESS_SECS` | `--turn-liveness-secs` | `10` (`:302`) | `>0 && <5` → **error** (`:764-768`); `>86400` → warn + clamp (`:836-845`) |
| `BUZZ_ACP_HEARTBEAT_PROMPT` | `--heartbeat-prompt` | — (`:306-310`) | Conflicts with the file variant; wins in code (`:770-776`) |
| `BUZZ_ACP_HEARTBEAT_PROMPT_FILE` | `--heartbeat-prompt-file` | — (`:314-318`) | No size cap |
| `BUZZ_ACP_INITIAL_MESSAGE` | `--initial-message` | — (`:321`) | Passed through unvalidated (`:980`) |
| `BUZZ_ACP_SUBSCRIBE` | `--subscribe` | `mentions` (`:324-329`) | `mentions` / `all` / `config` |
| `BUZZ_ACP_KINDS` | `--kinds` | — comma (`:332`) | Ignored in config mode (warns, `:794-796`). Absent + `--subscribe all` → `kinds: None` wildcard (`:1180`, `:1272`) |
| `BUZZ_ACP_CHANNELS` | `--channels` | — comma (`:335`) | Non-UUID entries **warn and are dropped**, never error (`:816-824`); ignored in config mode (warns, `:797-799`) |
| `BUZZ_ACP_NO_MENTION_FILTER` | `--no-mention-filter` | `false` (`:338`) | Inverted into `require_mention` (`:1168`, `:1269`); ignored in config mode (warns, `:800-802`) |
| `BUZZ_ACP_CONFIG` | `--config` | `./buzz-acp.toml` (`:341`) | Read only in `SubscribeMode::Config` via `load_rules` (`:1060`) |
| `BUZZ_ACP_DEDUP` | `--dedup` | `queue` (`:344`) | `drop` + any cancel mode → **error** (`:579-597`) |
| `BUZZ_ACP_MULTIPLE_EVENT_HANDLING` | `--multiple-event-handling` | `steer` (`:353-358`) | `queue` / `steer` / `interrupt` / `owner-interrupt` |
| `BUZZ_ACP_NO_IGNORE_SELF` | `--no-ignore-self` | `false` (`:361`) | Inverted → `ignore_self` (`:977`) |
| `BUZZ_ACP_CONTEXT_MESSAGE_LIMIT` | `--context-message-limit` | `12`, range `0..=100` (`:366-367`) | Enforced by clap; `0` disables context fetching (`:364-365`) |
| `BUZZ_ACP_MAX_TURNS_PER_SESSION` | `--max-turns-per-session` | `0` = disabled (`:372-373`) | `value_parser!(u32)` with **no range** — any `u32` accepted |
| `BUZZ_ACP_NO_PRESENCE` | `--no-presence` | `false` (`:377`) | Inverted → `presence_enabled` (`:983`) |
| `BUZZ_ACP_NO_TYPING` | `--no-typing` | `false` (`:381`) | Inverted → `typing_enabled` (`:984`) |
| `BUZZ_ACP_MEMORY` | `--memory` | `true` (`:393-398`) | `conflicts_with = "no_memory"`; combined as `memory && !no_memory` (`:985`) |
| `BUZZ_ACP_NO_MEMORY` | `--no-memory` | `false` (`:404`) | The documented opt-out |
| `BUZZ_ACP_NO_BASE_PROMPT` | `--no-base-prompt` | `false` (`:409`) | Suppresses the `[Base]` section entirely (`lib.rs:1539-1540`) |
| `BUZZ_ACP_BASE_PROMPT_FILE` | `--base-prompt-file` | — (`:414-418`) | `conflicts_with = "no_base_prompt"`; content read at startup, **`>1_048_576` bytes → error** (`:778-791`) |
| `BUZZ_ACP_MODEL` | `--model` | — (`:423`) | Applied after every `session_new_full`; resolved against the catalog (`acp.rs:1876`) |
| `BUZZ_ACP_PERMISSION_MODE` | `--permission-mode` | **`bypass-permissions`** (`:435`) | Five values, each with a camelCase alias (`:122-139`) |
| `BUZZ_ACP_RESPOND_TO` | `--respond-to` | `owner-only` (`:445`) | Checked against `allowed_respond_to` when that is non-empty (`:926-933`) |
| `BUZZ_ACP_RESPOND_TO_ALLOWLIST` | `--respond-to-allowlist` | — comma (`:452`) | Required non-empty in `allowlist` mode (`:903-907`); 64-hex per entry, trimmed + lowercased + deduped (`:558-572`, applied `:908`); **warn-and-discard** in any other mode (`:909-915`) |
| `BUZZ_ACP_ALLOWED_RESPOND_TO` | `--allowed-respond-to` | — comma (`:460`) | Each entry must parse as a `RespondTo` (`:921-927`); non-empty list not containing the active mode → **error**. Afterwards **display-only** |
| `BUZZ_ACP_TEAM_INSTRUCTIONS` | `--team-instructions` | — (`:464`) | Trimmed; empty → `None` (`:967-972`) |
| `BUZZ_ACP_RELAY_OBSERVER` | `--relay-observer` | `false` (`:468`) | Gates `ObserverHandle` creation (`lib.rs:1298-1300`) |
| `BUZZ_ACP_LAZY_POOL` | `--lazy-pool` | `false` (`:472`) | Defers subprocess startup until accepted work arrives |

`AuthAgentArgs` re-declares two of these for the auxiliary subcommands, with the same env names and defaults: `BUZZ_ACP_AGENT_COMMAND` → `goose` (`config.rs:191`) and `BUZZ_ACP_AGENT_ARGS` → `acp` (`config.rs:194-199`). `AuthenticateArgs::method_id` is the only required flag with **no** env fallback (`config.rs:230-231`).

#### Env vars read outside clap

| Env var | Read site | Effect |
|---|---|---|
| `BUZZ_ACP_SETUP_PAYLOAD` | `setup_mode.rs:83` (const), `setup_mode.rs:214` | Non-empty valid JSON diverts startup to the nudge listener; the agent pool never starts (`lib.rs:1290-1295`). Malformed → startup error |
| `BUZZ_AUTH_TAG` | `lib.rs:125`, `lib.rs:1338`, `lib.rs:4173`, `setup_mode.rs:319` | NIP-OA owner attestation; parsed by `buzz_sdk::nip_oa::parse_auth_tag`, silently dropped when empty or unparseable. Also forwarded into `McpServer.env` |
| `CODEX_CONFIG` | `acp.rs:437` | The parent's own value; deep-merged with **parent-wins** precedence into the generated config (`acp.rs:311-327`) |
| `BUZZ_MANAGED_AGENT_START_NONCE` | `lib.rs:1503` | `unwrap_or_default()` — no validation |
| `BUZZ_ACP_EVENT_BUFFER` | `relay.rs:36` | Relay event-channel capacity. Documented at `relay.rs:28`. Outside this module group but part of the crate's env surface |
| *(any key in `persona_env_vars`)* | `acp.rs:455` | Injected into the child **only if not already set** in the parent |

#### Legacy aliases

`propagate_legacy_env_vars` (`config.rs:715-726`) copies, only when the canonical name is unset (`config.rs:720`):

| Legacy | Canonical | Status |
|---|---|---|
| `BUZZ_ACP_PRIVATE_KEY` | `BUZZ_PRIVATE_KEY` | Live — feeds `CliArgs::private_key` (`config.rs:243`) |
| `BUZZ_ACP_API_TOKEN` | `BUZZ_API_TOKEN` | **Parsed and never read.** `BUZZ_API_TOKEN` appears nowhere else in the crate's source; the only other repo mention is the README table row (`crates/buzz-acp/README.md:107`) |

`BUZZ_ACP_TURN_TIMEOUT` is the third legacy name, handled inside clap as a hidden flag rather than by this function (`config.rs:274`).

Ordering constraint: this function uses `std::env::set_var` and must run before the tokio runtime starts, because Rust 2024 requires a single-threaded process for that call (`config.rs:708-714`). It is invoked from the sync wrapper at `lib.rs:1234`; `Config::from_cli` explicitly does not call it (`config.rs:730-732`).

#### Parsed-and-never-read / effectively-inert configuration

| Item | Evidence |
|---|---|
| `BUZZ_API_TOKEN` | written at `config.rs:718`, no read site in the crate |
| `Config::allowed_respond_to` | validated `config.rs:919-937`, then referenced only by `summary()` (`config.rs:1019-1025`, emitted `:1049`) and two test-fixture initialisers (`lib.rs:4979`, `lib.rs:5145`) |
| `Config::persona_env_vars` as a persona mechanism | only ever receives the one generated `CODEX_CONFIG` pair (`config.rs:945-955`); `buzz-persona` (declared `Cargo.toml:22`) has zero references under `crates/buzz-acp/src` |
| `--respond-to-allowlist` outside `allowlist` mode | warn-and-discard (`config.rs:909-915`) |
| `--kinds` / `--channels` / `--no-mention-filter` in config mode | warn-and-ignore (`config.rs:793-804`) |
| `handle_setup_membership`'s `_initial_channel_ids` | accepted, unused (`setup_mode.rs:568`) |

#### TOML configuration file

Path from `BUZZ_ACP_CONFIG` (default `./buzz-acp.toml`, `config.rs:341`), loaded only in `SubscribeMode::Config`. Schema is a single `rules` array of `SubscriptionRule` (`config.rs:1053-1057`, `#[serde(default)]` so an empty file parses).

Constraints enforced by `load_rules` (`config.rs:1060-1132`): ≤100 rules; non-empty unique `name`; `filter` expression ≤4096 bytes and compiled by `evalexpr::build_operator_tree` at load time; `channels` must be the literal `"all"` or a list. Zero rules is a **warning only** — "agent will receive no events in Config mode" (`config.rs:1075-1080`).

#### Documentation drift

| Claim | Source | Actual |
|---|---|---|
| `BUZZ_ACP_IDLE_TIMEOUT` default `620` | `crates/buzz-acp/README.md:105` | `DEFAULT_IDLE_TIMEOUT_SECS = 900` (`config.rs:27`) |
| `BUZZ_API_TOKEN` "required if relay enforces token auth" | `crates/buzz-acp/README.md:107` | Never read by the crate |
| `BUZZ_ACP_TURN_TIMEOUT=320` presented as the timeout knob | `.env.example:152` | Hidden + deprecated (`config.rs:274`); `.env.example` never mentions `BUZZ_ACP_IDLE_TIMEOUT` or `BUZZ_ACP_MAX_TURN_DURATION` |
| `.env.example` coverage | `.env.example:121-226` | Documents ~20 of the 43 env vars. Missing entirely: `BUZZ_ACP_AGENT_OWNER`, `BUZZ_ACP_IDLE_TIMEOUT`, `BUZZ_ACP_MAX_TURN_DURATION`, `BUZZ_ACP_TURN_LIVENESS_SECS`, `BUZZ_ACP_MULTIPLE_EVENT_HANDLING`, `BUZZ_ACP_MEMORY` / `BUZZ_ACP_NO_MEMORY`, `BUZZ_ACP_NO_BASE_PROMPT`, `BUZZ_ACP_BASE_PROMPT_FILE`, `BUZZ_ACP_PERMISSION_MODE`, `BUZZ_ACP_RESPOND_TO`, `BUZZ_ACP_RESPOND_TO_ALLOWLIST`, `BUZZ_ACP_ALLOWED_RESPOND_TO`, `BUZZ_ACP_TEAM_INSTRUCTIONS`, `BUZZ_ACP_RELAY_OBSERVER`, `BUZZ_ACP_LAZY_POOL`, `BUZZ_ACP_SETUP_PAYLOAD`, `BUZZ_AUTH_TAG`, `BUZZ_MANAGED_AGENT_START_NONCE` |
| `BUZZ_ACP_EVENT_BUFFER=256` | `.env.example:221` | Real, but read in `relay.rs:36`, not in this group — it is the one env var `.env.example` documents that no `config.rs` flag covers |

The two security-relevant defaults — `--permission-mode bypass-permissions` (`config.rs:435`) and the auto-approve permission handler (`acp.rs:1703-1718`) — are absent from both `.env.example` and the README's env table.
