## Module: buzz-acp — agent pool & lifecycle (`crates/buzz-acp/src`)
### Aspect: Security

#### Where subprocess spawning actually happens

Neither `pool.rs` nor `pool_lifecycle.rs` calls a process API. Grepping both files for `Command`, `Stdio`, `env`, `kill`, or `spawn` returns only `tokio::spawn`/`JoinSet::spawn` for async tasks. All child-process creation is `AcpClient::spawn` (`acp.rs:408-497`), invoked by `lib.rs` at three sites: eager startup (`lib.rs:3749-3754`), respawn/refill (`lib.rs:3856`, via `spawn_and_init`), and the lazy wake (`lib.rs:1729-1741` → `initialize_agent_pool`). The pool's role is to decide *when* a child is poisoned and must be replaced.

#### Command resolution, PATH, and shell

- `tokio::process::Command::new(command)` with the raw configured string (`acp.rs:416`). Default is `"goose"` — a bare name (`config.rs:191`, `:250`), so resolution goes through the harness's inherited `PATH`. A `PATH` entry the harness user can write to is sufficient to substitute the agent binary.
- No shell is involved: args are passed as a vector via `cmd.args(args)` (`acp.rs:417`), so there is no word-splitting or metacharacter interpretation. Argument injection is limited to whatever `config.agent_args` contains.
- Command and args are harness configuration only (CLI flag, `BUZZ_ACP_AGENT_COMMAND`/`BUZZ_ACP_AGENT_ARGS`, or the TOML file). No path in this module lets a Nostr event, channel message, or remote peer influence the command or argv. There is **no allow-list** on the command — any resolvable binary is accepted, but the input is trusted config rather than caller-supplied data.
- No `current_dir()` is set on the `Command` (`acp.rs:416-470` contains no such call), so the child inherits the harness's working directory. `ctx.cwd` — `std::env::current_dir()` with `/` as the failure fallback (`lib.rs:1547-1550`) — is only sent as the ACP `session/new` `cwd` parameter (`pool.rs:831`) and rendered into the `[Workspace]` prompt section (`pool.rs:1165-1178`). Whether the agent honours it is the agent's choice; the harness does not enforce it. The `/` fallback is defended only at the prompt level: `workspace_section` refuses to name `/` because that "would actively encourage the `$HOME`-wide scan this section exists to prevent" (`pool.rs:1162-1164`).

#### Environment inheritance

The child environment is **inherited wholesale** — there is no `env_clear()` anywhere in the crate. On top of the inherited env, persona/config vars are injected with **operator-precedence**: a key already present in the parent is skipped (`acp.rs:452-458`). So the harness's own secrets in its environment (including `BUZZ_PRIVATE_KEY` if it was supplied that way, per `config.rs:243`) are visible to every agent subprocess and to everything the agent spawns. `CODEX_CONFIG` is the one key with special merge handling, and that merge force-sets `network_access = true` (`acp.rs:428-462`).

#### Credential propagation — how `BUZZ_PRIVATE_KEY` reaches the agent

This is the precise answer to the mechanism:

1. The private key is **not** in the agent child's argv and **not** in the agent child's environment set by the harness. `AcpClient::spawn` injects only `extra_env` = `config.persona_env_vars` (`lib.rs:3733`, `acp.rs:452-458`), which does not include `BUZZ_PRIVATE_KEY`.
2. The key travels **over the ACP stdio wire** as JSON. `build_mcp_servers` (`lib.rs:4141-4184`) constructs one `McpServer` whose `env` array contains `BUZZ_RELAY_URL`, `BUZZ_PRIVATE_KEY` = `config.keys.secret_key().to_bech32()` (the raw `nsec…`, `lib.rs:4160-4170`), and `BUZZ_AUTH_TAG` when non-empty (`lib.rs:4171-4180`). That vec becomes `PromptContext::mcp_servers`.
3. **This module is the transmitter.** `create_session_and_apply_model` clones it into every `session/new` request: `ctx.mcp_servers.clone()` at `pool.rs:832`, serialized into `params.mcpServers` (`acp.rs:568-571`) and written as one NDJSON line to the child's stdin. One line per session creation, per agent, for the process lifetime.
4. The agent then spawns the MCP server itself with that env. So the key is delivered to the MCP child's **environment**, but by the agent, from a spec the harness handed it in cleartext over a pipe.

Exposure consequences:

- **Not in `ps` output.** Nothing puts the nsec in argv, so a process listing does not reveal it. That specific risk does not apply here.
- **In debug logs.** `send_request` logs the entire outbound JSON before writing: `tracing::debug!(target: "acp::wire", "→ {}", serde_json::to_string(&msg))` (`acp.rs:994`). With `RUST_LOG` enabling `debug` for `acp::wire` (or globally), every `session/new` writes the agent's `nsec` into the harness log. There is no redaction: `McpServer`/`EnvVar` derive plain `Debug` and `Serialize` with no custom impl (`acp.rs:27-43`), and no `redact`/`Secret` wrapper exists anywhere in `crates/buzz-acp/src`.
- **In the agent's own transcript.** Whatever the agent logs or persists about its received `session/new` params carries the key; the harness has no control over that.
- **Panics on malformed key.** `build_mcp_servers` uses `.expect("secret key bech32 encoding should never fail")` (`lib.rs:4167`) with an inline justification that a bogus secret would cause delayed failures instead.
- **One identity for all agents.** `ctx.agent_keys` is a clone of the single harness `Keys` (`lib.rs:1557`), shared by every task and every agent, and also used to sign NIP-AM metrics (`pool.rs:3402`) and decrypt NIP-AE core memory. There is no per-agent credential scoping.

The pool never logs, prints, or `Debug`-renders `agent_keys` or `mcp_servers`: `PromptContext` has no `Debug` derive (`pool.rs:482`), and no tracing call in `pool.rs` references either field.

#### Content-trust boundary

The canvas path is the only relay-sourced input that is cryptographically verified before it is folded into a system prompt: full `nostr::Event` deserialization, `event.verify()`, kind check against `KIND_CANVAS`, and an `h`-tag re-check explicitly to stop "a misbehaving relay from injecting a different channel's canvas" (`pool.rs:2370-2432`). It also rejects out-of-range `created_at` via `i64::try_from` rather than `as i64` (`pool.rs:2435-2465`).

The other relay-sourced inputs that reach the prompt are **not** verified in this module: `fetch_channel_info` reads name and type from raw JSON tags (`pool.rs:2258-2281`), thread/DM context messages are taken from raw `content`/`pubkey`/`created_at` fields (`pool.rs:2861-2890`, `:2892-2971`), and profile display names / NIP-05 handles come from unverified kind-0 `content` (`pool.rs:2594-2630`). All of these are concatenated into the prompt body, so a hostile or compromised relay can shape agent instructions through display names and quoted history. `profile_event_is_agent` is explicitly labelled a routing heuristic and "not a verified security gate" (`pool.rs:2571-2578`).

Input validation that is present: `normalize_prompt_pubkey` accepts only 64-char lowercase hex (`pool.rs:2528-2534`), `fetch_thread_context` validates the root event id as 64 hex chars before querying (`pool.rs:2684-2695`), and `reaction_add` rejects non-hex event ids (`pool.rs:3463-3468`).

#### Permission posture

The shipped default is `bypass-permissions` (`config.rs:435`), i.e. the harness asks the agent to skip its per-tool-call permission prompt; `default` must be set explicitly to restore the agent's built-in gating (`config.rs:428-431`). On top of that, `ctx.permission_mode` is applied only when it is non-default **and** the agent advertised the mode in `session/new` (`pool.rs:924-928`). When the agent does not advertise it, the request is silently skipped and the harness falls back to "per-tool auto-approval in `handle_permission_request`" (`pool.rs:1028-1029`). Application-level failures to set the mode are logged at `warn` and the turn proceeds (`pool.rs:1064-1071`). Net effect: a configured restrictive permission mode can degrade to auto-approval without failing the turn or surfacing an error to the user.

#### Resource limits

Bounded: per-turn idle timeout and hard wall-clock cap (`pool.rs:1832-1837`), 5 s cancel-drain grace (`pool.rs:793`), 3 s core/canvas/metric fetches, 3 s + one retry for context fetches, 500 ms–1 s reaction calls, `REACTION_CONCURRENCY = 10` fan-out cap (`pool.rs:3618`), pool width capped at 32 by clap (`config.rs:293`).

Unbounded:
- `result_tx`/`result_rx` is `mpsc::unbounded_channel()` (`pool.rs:542`). In practice bounded by pool width, since only a checked-out agent can send.
- `SessionState`'s six maps grow one entry per channel with no cap or TTL (`pool.rs:86-104`); only explicit invalidation removes entries.
- No memory, CPU, file-descriptor, or process-count limit is applied to the child. No `setrlimit`, cgroup, or sandbox anywhere in the crate.
- Wake retries are unbounded in count (`pool_lifecycle.rs:122-131`), though the 300 s cap bounds the rate.

#### Zombie and orphan handling

- Children are spawned into their own process group on Unix (`acp.rs:463-466`), and `shutdown()` kills the whole group so MCP servers and tool processes are reaped rather than orphaned to init (`acp.rs:374-386`).
- `shutdown()`'s `child.wait()` is bounded at 5 s; on expiry it logs `"child did not exit within 5s after SIGKILL — abandoning"` and returns (`acp.rs:392-397`). An uninterruptible child is therefore intentionally leaked.
- `Drop` only does `start_kill()` without reaping (`acp.rs:373`, `:387`, `:1961`), so any path that drops an `AcpClient` instead of awaiting `shutdown()` can leave a zombie. `kill_on_drop(true)` (`acp.rs:423`) mitigates but does not reap.
- Paths that reach `Drop` rather than `shutdown()`: a panicked prompt task (its `OwnedAgent` is dropped in the unwind; `lib.rs:3461` states "The panicked task already dropped the AcpClient"), the overwrite branch of `return_agent` (`pool.rs:579-589`), and any task aborted when the 30 s shutdown grace expires (`lib.rs:2637-2639`, comment at `lib.rs:2609-2613`).
- `respawn_tasks.shutdown()` is called after `drop(pool)` specifically so a backoff-sleeping task cannot spawn a new child post-exit (`lib.rs:2657-2663`).

#### Can a crashed or hung child exhaust the host?

Partially bounded, with named gaps:

- A hung child is cut by the idle timeout (default 900 s) or the hard cap (default 7200 s) (`config.rs:27`, `:31`), then classified as poisoned and respawned. Respawn is rate-limited per slot by the circuit breaker: 3 crashes in 60 s opens the circuit for 300 s, with 1 s→30 s exponential backoff between attempts (`lib.rs:1008-1016`). So crash-looping cannot spin the host at full rate.
- The default hard cap is 2 hours, so a wedged-but-alive agent holds its slot — and, at `--agents 1`, the entire harness's throughput — for up to that long. `turn_liveness` frames make this observable but do not act on it.
- Abandoned children (the 5 s `wait()` expiry, or `Drop`-only paths) accumulate without limit across many respawns; nothing tracks or re-reaps them.
- Nothing in the harness constrains the child's own resource use, so a compromised or runaway agent can consume host memory/CPU/disk freely; the harness only stops sending it prompts.
