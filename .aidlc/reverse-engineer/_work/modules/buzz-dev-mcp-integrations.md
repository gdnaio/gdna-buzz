## Module: buzz-dev-mcp (`crates/buzz-dev-mcp`)
### Aspect: Integrations

#### MCP server wiring

`rmcp` is pinned workspace-wide at `1.1.0` with features `server`,
`transport-io`, `macros` (`Cargo.toml:120`). The crate uses the derive-macro
route: `#[tool_router]` on `impl DevMcp` (`crates/buzz-dev-mcp/src/lib.rs:30`),
`#[tool(name = …, description = …)]` per method, and
`#[tool_handler(router = self.tool_router)]` on the `ServerHandler` impl
(`lib.rs:126-127`). Input schemas are generated from `schemars` (workspace `1`,
`Cargo.toml:121`) via `#[derive(JsonSchema)]` on each params struct.

Transport is **stdio only** — `DevMcp::new(state).serve(stdio())`
(`lib.rs:6-7`, `lib.rs:185`). There is no HTTP/SSE transport, no listening socket,
and no port binding anywhere in the crate. `service.waiting().await` blocks until
the client disconnects (`lib.rs:186`).

Capabilities advertised: `tools` only (`lib.rs:129-135`). No resources, no
prompts, no sampling.

#### Intra-repo crate dependencies

| Dependency | Declared | What it is used for |
|---|---|---|
| `buzz-cli` | `Cargo.toml:18` | the whole CLI is linked in and exposed as the `buzz` multicall personality: `buzz_cli::run_from_args(std::env::args()).await` (`lib.rs:168-171`); `run_from_args` is `crates/buzz-cli/src/lib.rs:23` |
| `git-credential-nostr` | `Cargo.toml:19` | `git_credential_nostr::run()` as the `git-credential-nostr` personality (`lib.rs:151`); the fn is `crates/git-credential-nostr/src/lib.rs:152` |
| `git-sign-nostr` | `Cargo.toml:20` | `git_sign_nostr::run()` as the `git-sign-nostr` personality (`lib.rs:152`); the fn is `crates/git-sign-nostr/src/lib.rs:1726` |
| `buzz-core` | `Cargo.toml:42` | exactly one call: `buzz_core::tenant::relay_url_authority` for normalising the Blossom `server` tag authority (`view_image.rs:278`, defined at `crates/buzz-core/src/tenant.rs:156`) |
| `nostr` (workspace `0.44`) | `Cargo.toml:20` of workspace (`Cargo.toml:61`) | `Keys::parse` + `public_key().to_bech32()` in the shim (`shim.rs:90-101`); `EventBuilder`/`Kind(24242)` signing in `view_image` (`view_image.rs:252-274`) |

**Why an MCP tool server depends on the CLI**: it is not a duplication — the
dependency exists so a single binary can *be* the `buzz` CLI. `Shim::install`
symlinks the running executable to `buzz` in a `0700` tempdir and prepends that
dir to the `PATH` handed to every `shell` child (`shim.rs:31-49`). The agent then
runs `buzz channels list` from the shell tool and hits the in-binary CLI, with no
separately installed `buzz` on the host. The same trick supplies `rg`, `tree`, and
the two git helpers. The `shell` tool description advertises exactly this
(`lib.rs:42`: "and `buzz` (Buzz relay CLI — run buzz --help for commands)").

Regarding `AGENTS.md:147` ("agent-facing operations go in `buzz-cli`… `buzz-dev-mcp`
is separate"): the boundary holds. This crate exposes no relay operation as an MCP
tool — no channel, message, or event tool exists in the router
(`lib.rs:40-125`). All relay interaction is reached through the bundled `buzz`
CLI inside the `shell` tool. The only relay-protocol code in the crate is the
Blossom `t=get` media-read signer in `view_image.rs:252-318`, which duplicates
the desktop client's signer by acknowledgement (`view_image.rs:51-53` cites the
desktop `MEDIA_GET_AUTH_EXPIRY_SECS`) rather than routing through `buzz-cli`.

#### Third-party crates

| Crate | Version | Use |
|---|---|---|
| `similar` | `3` (`Cargo.toml:31`) | unified diff generation and the `str_replace` miss-hint similarity score (`str_replace.rs:142-155`, `str_replace.rs:177-194`) |
| `tempfile` | `3` (`Cargo.toml:32`) | shim dir, session dir, and the atomic-write temp file (`shim.rs:26`, `shell.rs:41-44`, `str_replace.rs:130`) |
| `ignore` | `0.4.25` (`Cargo.toml:33`) | `WalkBuilder` for the `tree` personality's gitignore-aware walk (`tree.rs:41-50`) |
| `zeroize` | workspace `1.8` (`Cargo.toml:21`, `Cargo.toml:103`) | zeroing the in-memory copy of `NOSTR_PRIVATE_KEY` after the keyfile write (`shim.rs:65-68`) |
| `image` | `0.25`, features `jpeg png gif webp`, `default-features = false` (`Cargo.toml:40`) | decode/resize/encode in `view_image` (`view_image.rs:16-20`) |
| `base64` | `0.22` (`Cargo.toml:39`) | `STANDARD` for image payloads, `URL_SAFE_NO_PAD` for the Blossom auth header (`view_image.rs:99`, `view_image.rs:270-273`) |
| `reqwest` | workspace `0.13`, rustls (`Cargo.toml:38`, `Cargo.toml:93`) | http(s) image fetch (`view_image.rs:321-390`) |
| `rustls` | `0.23`, `ring` + `std`, `default-features = false` (`Cargo.toml:36`) | explicitly installed as the default crypto provider because the workspace pulls both `ring` and `aws-lc-rs` transitively (`Cargo.toml:34-36`, `lib.rs:164-166`) |
| `tokio-util` | workspace `0.7` (`Cargo.toml:24`) | `CancellationToken` for the `shell` cancel path (`shell.rs:14`) |
| `nix` | `0.31`, `signal` + `process`, Unix only (`Cargo.toml:45`) | `killpg` process-group termination (`shell.rs:706-724`) |
| `windows-sys` | `0.61`, five Win32 feature sets, Windows only (`Cargo.toml:52`) | Job Objects for process-tree kill, and the registry probe for Git for Windows (`shell.rs:475-539`, `shell.rs:759-844`) |

`tokio` is built as a **current-thread** runtime (`lib.rs:159-162`), so all tool
handling is single-threaded plus spawned reader tasks.

#### External binaries shelled out to

| Binary | Invoked from | Presence checked? |
|---|---|---|
| a shell (`bash`, or `BUZZ_SHELL`/`GIT_BASH`-selected `cmd`/`pwsh`/other) | `shell.rs:166-167` | Yes on Windows — six-step probe (`BUZZ_SHELL` → `GIT_BASH` → `bash.exe` on PATH excluding `%SystemRoot%` → sibling of `git.exe` → `ProgramFiles`/`ProgramFiles(x86)`/`LocalAppData` → HKLM/HKCU `SOFTWARE\GitForWindows\InstallPath`) with an actionable failure message (`shell.rs:409-459`). On Unix the resolver returns the bare string `"bash"` without any existence check when `BUZZ_SHELL` is unset (`shell.rs:363-388`), so a missing bash surfaces only as a spawn error (`shell.rs:183-190`) |
| system `ripgrep` (`rg`) | `rg.rs:18-29` | Yes — `which_rg` requires `is_file()` and, on Unix, an executable bit; falls back to the in-crate implementation when absent (`rg.rs:49-73`) |
| `git` | never invoked directly by this crate | n/a — the shim only *configures* git via `GIT_CONFIG_*`; `git.exe` is probed on Windows solely to locate its sibling `bin/bash.exe` (`shell.rs:437-441`, `shell.rs:464-470`) |
| anything the agent types | `shell.rs:166-167` | No |

Note the WSL-avoidance logic on Windows: `%SystemRoot%`-rooted PATH entries and
`%LOCALAPPDATA%\Microsoft\WindowsApps` alias stubs are skipped so `bash.exe`
never resolves to the WSL launcher (`shell.rs:566-643`).

#### Who spawns this crate

| Consumer | Mechanism |
|---|---|
| `buzz-acp` | constructs one `McpServer` spec from `config.mcp_command` and injects `BUZZ_RELAY_URL`, `BUZZ_PRIVATE_KEY` (bech32 `nsec1…`), and optionally `BUZZ_AUTH_TAG` (`crates/buzz-acp/src/lib.rs:4141-4184`). It does not spawn the process itself — the ACP agent does. |
| `buzz-agent` | `spawn_one` does `cmd.env_clear()` then re-adds only `PASSTHROUGH_ENV`, which includes `NOSTR_PRIVATE_KEY`, `BUZZ_PRIVATE_KEY`, `BUZZ_RELAY_URL`, `BUZZ_AUTH_TAG` (`crates/buzz-agent/src/mcp.rs:39-64`, `crates/buzz-agent/src/mcp.rs:708-727`); on Windows it also preserves `WINDOWS_SHELL_RESOLUTION_ENV` = `PATH`, `BUZZ_SHELL`, `GIT_BASH`, `SystemRoot`, `ProgramFiles`, `ProgramFiles(x86)`, `LOCALAPPDATA` (`crates/buzz-agent/src/lib.rs:22-30`) |
| Desktop app (Tauri) | `buzz-dev-mcp` is bundled as an external binary (`desktop/src-tauri/tauri.conf.json:58`) and set as the default `mcp_command` for discovered agents (`desktop/src-tauri/src/managed_agents/discovery.rs:138`, `:171`); the runtime process-tree reaper knows its name and multicall aliases (`desktop/src-tauri/src/managed_agents/runtime.rs:49-53`) |
| `sprig` | links the crate and routes every unmatched `argv[0]` to `buzz_dev_mcp::run()` (`crates/sprig/Cargo.toml:19`, `crates/sprig/src/main.rs:39-41`) |
| Desktop UI | classifies tool names by stripping the `buzz_dev_mcp_` prefix for display (`desktop/src/features/agents/ui/agentSessionToolClassifier.ts:338`, `:350`) |
| Relay E2E example | `mesh_agent_e2e.rs` wires `("dev", repo_bin("buzz-dev-mcp"))` as the agent's MCP server (`crates/buzz-relay/examples/mesh_agent_e2e.rs:170`) |

Combined with the `buzz-persona` finding that a persona's `mcp_servers` entry is
an arbitrary `command`/`args`/`env` subprocess spec with no allow-list, any of
these paths can point `mcp_command` at an arbitrary binary; conversely this crate
can be launched with an arbitrary environment.
