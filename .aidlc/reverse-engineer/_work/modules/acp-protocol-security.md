## Module: buzz-acp — ACP protocol, config & setup mode (`crates/buzz-acp/src`)
### Aspect: Security

#### The agent's private key reaches the child's stdin, the debug log, and the observer feed

The chain, end to end:

1. `build_mcp_servers` puts the raw bech32 `nsec1…` into an `EnvVar` inside `McpServer.env` (`lib.rs:4159-4170`), via `config.keys.secret_key().to_bech32()`.
2. `pool.rs:832` clones that vec into every `session_new_full` call, so it is re-sent on each session creation and each session rotation.
3. `session_new_full` serializes it as `params.mcpServers` (`acp.rs:568-571`).
4. `write_ndjson` writes the full frame to the child's stdin (`acp.rs:951-966`).

Two leaks branch off step 3/4:

- **Debug log.** `send_request` logs the entire outbound JSON before writing: `tracing::debug!(target: "acp::wire", "→ {}", &serde_json::to_string(&msg).unwrap_or_default())` (`acp.rs:994`). At `debug` on `acp::wire`, the `session/new` frame — key included — lands in whatever sink the harness's `tracing-subscriber` is configured with. Three sibling sites do the same for other frames (`acp.rs:703`, `:1059`, `:1317`).
- **Observer feed.** `write_ndjson` ends with `self.observe("acp_write", value.clone())` (`acp.rs:963`) — every outgoing frame, verbatim, with no filtering by method and no field redaction. `observe` forwards to `ObserverHandle::emit` (`acp.rs:524-533`).

The **inbound** direction has the same exposure: `self.observe("acp_read", msg.clone())` fires for every parsed inbound frame in both read loops (`acp.rs:1120`, `acp.rs:1414`), and `acp_parse_error` emits the raw offending line (`acp.rs:1105-1111`, `acp.rs:1399-1405`). So an agent that echoes its `mcpServers` config back — in an error message, a tool-call argument, a thought chunk — puts the key back on the observer feed a second time. Neither direction inspects content.

Reach of the observer feed is not local-only: `--relay-observer` / `BUZZ_ACP_RELAY_OBSERVER` (`config.rs:468`) publishes observer frames over the relay, and the doc comment for the flag says "Publish encrypted ACP observer frames over the relay" (`config.rs:467`). The encryption is applied downstream in `observer.rs`/`relay.rs`; `acp.rs` hands over plaintext.

No mitigation exists anywhere in the crate. `grep -rn 'redact\|Secret\|zeroize' crates/buzz-acp/src/` returns exactly two hits, both **comment text** at `config.rs:742-743` describing the best-effort string scrub of the CLI argument — there is no `Secret<T>` newtype, no custom `Debug`/`Serialize` impl, and no `zeroize` dependency in `Cargo.toml`. `McpServer` and `EnvVar` derive plain `Debug, Clone, serde::Serialize` (`acp.rs:27`, `acp.rs:36`).

The one defensive measure that does exist is narrow: `Config::from_args` overwrites the raw `--private-key` string with `0`s and clears it after parsing (`config.rs:742-748`), with a comment conceding that without the `zeroize` crate the allocator may retain copies.

Blast radius: `nsec1…` is the agent's full Nostr identity. Anyone reading a debug log, an observer stream, or the child's stdin can sign as the agent — publish messages, join channels, and (with `BUZZ_AUTH_TAG`, also forwarded at `lib.rs:4171-4179`) act with the owner's attestation.

#### Subprocess spawning

| Property | Finding | Line |
|---|---|---|
| Shell involvement | **None.** `tokio::process::Command::new(command)` with `.args(args)` — argv is passed as a vector, so no shell metacharacter interpretation | `acp.rs:416-417` |
| Command resolution | Bare name by default (`goose`), resolved through the **inherited `PATH`** | `config.rs:250`, `config.rs:191` |
| Argv source | `--agent-args` / `BUZZ_ACP_AGENT_ARGS`, comma-split by clap; trimmed and empty-filtered by `normalize_agent_args` | `config.rs:253-258`, `config.rs:679-706` |
| Working directory | **Not set.** No `.current_dir()` call — the child inherits the harness's cwd | `acp.rs:416-471` |
| Validation | Only "must not be empty" after trim | `config.rs:808-812` |

Because the command is a bare name resolved via `PATH`, a `PATH` entry earlier than the real binary's directory decides what executes. `normalize_agent_command_identity` (`config.rs:600-615`) reduces the command to a basename purely for *identity* matching (which default args, whether Codex env applies) — it does not constrain or canonicalise the path actually executed, so `/tmp/evil/goose` normalizes to `goose` and gets Codex/Goose treatment while running from `/tmp/evil`.

#### Environment inheritance

There is **no `env_clear()` anywhere in the crate** — `grep -rn 'env_clear' crates/buzz-acp/` returns nothing. `spawn` only ever *adds* variables (`acp.rs:449-461`). Consequences:

- The child inherits the harness's entire environment, which includes `BUZZ_PRIVATE_KEY` — the very variable clap read the key from (`config.rs:243`). So every agent subprocess receives the key implicitly through inheritance, **independently of** the explicit `mcpServers` path. Setting `BUZZ_ACP_MCP_COMMAND=""` (the default, `config.rs:261`) removes the stdin/log/observer exposure but not this one.
- `BUZZ_AUTH_TAG` (`lib.rs:125`, `lib.rs:1338`, `setup_mode.rs:319`) and every provider API key in the harness's environment are likewise inherited.
- Injection is **operator-wins**: `if std::env::var(key).is_err() { cmd.env(key, value) }` (`acp.rs:455-457`). A persona/generated value cannot override an already-set parent variable. The one exception is the merged `CODEX_CONFIG`, set unconditionally after the loop (`acp.rs:459-461`) — but even there the *parent's* keys win inside the merge (`acp.rs:311-327`).

#### `CODEX_CONFIG` widens the Codex sandbox by design

`build_codex_config_env` force-sets `sandbox_workspace_write.network_access = true` as the last merge step, explicitly overriding both persona and operator values (`acp.rs:329-343`; contract item 4 at `acp.rs:245-247`). Per the rationale at `config.rs:635-643`, this sets `NetworkSandboxPolicy::Enabled`, which puts `(allow network-outbound)` in the macOS Seatbelt policy — **full outbound TCP/TLS at the OS level**, not a relay-host allowlist. The relay host is parsed and logged (`config.rs:654-672`) but never narrows the grant.

The only guard is a fail-closed URL check: an unparseable relay URL or one without a host skips injection entirely rather than widening the sandbox for a malformed config (`config.rs:652-670`, rationale `config.rs:644-645`). Injection is also scoped to normalized `codex` / `codex-acp` commands (`config.rs:647-650`) — though that normalization is basename-only, per the `PATH` note above.

#### stderr inheritance

`.stderr(Stdio::inherit())` (`acp.rs:420-421`), chosen "so agent logs are visible in the harness terminal". Two consequences: the harness cannot filter, redact, or bound anything the agent writes to stderr, and agent stderr is interleaved into the harness's own stderr stream — so whatever collects harness logs also collects raw agent output, including any secrets the agent chooses to print.

#### Permission flow is auto-approve by default

`handle_permission_request` auto-selects the `allow_once` option for **every** `session/request_permission` (`acp.rs:1703-1718`), with `reject_once` used only when `allow_once` is absent (`acp.rs:1720-1726`). No prompt, no policy check, no per-tool allowlist. Combined with the `--permission-mode` default of `bypass-permissions` (`config.rs:435`), which tells supporting agents to skip the permission flow entirely (`config.rs:127`), the shipped posture is: agents execute tools without any human gate. `PermissionMode::DontAsk` and `Plan` exist as opt-ins (`config.rs:134`, `:138`) but are not the default.

#### The base prompt as prompt-injection substrate

`base_prompt.md` (136 lines, embedded at `lib.rs:1544`) is prepended to every prompt and instructs the agent to treat `buzz` CLI output as its working reality — read the feed, read channel messages, act on what it finds. Channel content is attacker-controllable by anyone the author gate admits, and the gate defaults to `owner-only` (`config.rs:445`) but supports `anyone` (`config.rs:97`). Specific instructions that convert injected text into action:

- Startup Recovery tells the agent to poll `buzz feed get` and `buzz messages get` and act on mentions and action items.
- The publishing rules make posting the *expected* outcome of a turn, so injected content that asks for a post is aligned with the standing instruction rather than in conflict with it.
- The agent-creation section supplies a ready `buzz agents draft-create --system-prompt …` invocation, so injected text can steer the content of a new agent's system prompt (mitigated by owner review before save).
- The Workspace Layout section grants read/write intent over `RESEARCH/`, `PLANS/`, `GUIDES/`, `WORK_LOGS/`, `OUTBOX/`, `REPOS/`, `.scratch/`.

There is no instruction anywhere in the file telling the agent to distrust message content or to treat channel text as untrusted data. The `--no-base-prompt` flag (`config.rs:409`) removes the orientation but not the CLI's capabilities.

Operator override surface: `--base-prompt-file` / `BUZZ_ACP_BASE_PROMPT_FILE` replaces the prompt wholesale from any readable path, bounded only by a 1 MB size check (`config.rs:780-790`). No signature, checksum, or content validation.

#### Setup mode auth posture

`run_setup_listener` runs the same identity and author gates as normal mode, not a relaxed set:

- Relay connection uses the real `config.keys` with NIP-42 via `HarnessRelay::connect` (`setup_mode.rs:330-333`), and `BUZZ_AUTH_TAG` is parsed for NIP-OA membership (`setup_mode.rs:318-322`).
- `author_allowed` is called with the production `respond_to` and allowlist, and with DM hardening — the call site comment states DMs fail closed on unknown channel type (`setup_mode.rs:428-442`).
- An explicit @mention is required even when `subscribe_mode = all` (`setup_mode.rs:422-426`, rationale `setup_mode.rs:514-519`).
- `ignore_self` is enforced by pubkey comparison (`setup_mode.rs:418-421`).
- Replay is deduped by event id so a reconnect cannot re-nudge (`setup_mode.rs:385-386`, `setup_mode.rs:506-509`).

The payload itself is **trusted without verification** by design — `setup_mode.rs:18-22` states desktop is the only readiness source and buzz-acp does not re-derive readiness. `BUZZ_ACP_SETUP_PAYLOAD` is an unsigned, unauthenticated env var (`setup_mode.rs:83`, read `setup_mode.rs:214`), and setting it diverts startup so the agent pool never runs (`lib.rs:1290-1295`). Anyone able to set the harness's environment can therefore silently disable the agent while it keeps posting plausible "needs configuration" replies. The threat model implicitly assumes environment control is already equivalent to full compromise — which the `BUZZ_PRIVATE_KEY` inheritance above supports.

Content published from the payload is attacker-influenced if the payload is: `diagnostic` (`setup_mode.rs:115`) and `setup_copy` (`setup_mode.rs:102`) strings go into the nudge markdown unescaped (`setup_mode.rs:184-188`, `setup_mode.rs:135`), and the whole payload is re-serialized into the `buzz:config-nudge` fenced block that the desktop parses into a card (`setup_mode.rs:296-302`).

#### Resource limits on the child

| Limit | Present? | Evidence |
|---|---|---|
| Max stdout line size | Yes — 10 MB, enforced at the codec level so the buffer never grows past it | `acp.rs:21`, `acp.rs:487`, `acp.rs:1076-1078` |
| Total stdout volume per turn | No | no accumulator in either read loop |
| Write blocking | Yes — 30 s `WRITE_TIMEOUT` | `acp.rs:952` |
| Non-prompt RPC wall clock | Yes — 60 s, ~90 s including the write phase | `acp.rs:968`, `acp.rs:974-976` |
| Turn idle silence | Yes — configurable, default 900 s | `acp.rs:1265-1268`, `config.rs:27` |
| Turn absolute wall clock | Yes — configurable, default 7200 s, ceiling 604 800 s | `acp.rs:1270-1273`, `config.rs:31`, `config.rs:36` |
| CPU / memory / fd / nproc rlimits | **No** — no `setrlimit`, no cgroup, no sandbox applied by the harness | `acp.rs:416-471` |
| Child count | Bounded indirectly by `--agents` 1..=32 | `config.rs:292-293` |
| Filesystem scope | **None** — no `current_dir`, no chroot; for Codex the harness actively *widens* the platform sandbox | `acp.rs:416-471`, `acp.rs:329-343` |

#### Process teardown can abandon children

`shutdown` (`acp.rs:376-397`) sends SIGKILL to the process group when a PID is available (`acp.rs:384-388`) — which reaches MCP servers and tool processes because the child was spawned with `process_group(0)` (`acp.rs:463-467`, `acp.rs:1970-1974`) — then waits at most 5 s (`acp.rs:394`). On expiry it logs `"child did not exit within 5s after SIGKILL — abandoning"` (`acp.rs:396`) and returns. The comment at `acp.rs:390-393` justifies the bound: an unbounded wait would wedge the harness during respawn.

`Drop` (`acp.rs:1953-1967`) does the same group-kill but can only manage a non-blocking `try_wait()` (`acp.rs:1965`), so a child that has not yet died is left as a zombie until the harness exits. `kill_on_drop(true)` (`acp.rs:424`) covers the direct child only; on non-Unix, `kill_process_group` returns `false` and the fallback `start_kill()` never reaches grandchildren (`acp.rs:1990-1992`). A crash-recovery loop that repeatedly hits the 5 s expiry can accumulate live MCP subprocesses holding the agent's credentials.
