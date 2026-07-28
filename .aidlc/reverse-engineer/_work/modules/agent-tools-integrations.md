## Module: buzz-agent — MCP registry, OAuth, hints & catalog (`crates/buzz-agent/src`)
### Aspect: Integrations

#### MCP child processes

Spawn recipe (`spawn_one`, `mcp.rs:708-786`):

| Step | Site | Detail |
|---|---|---|
| command + args | `mcp.rs:711-712` | taken verbatim from the client's `session/new` payload (`types.rs:McpServerStdio`); no allowlist, no PATH resolution of the binary by this crate |
| clear env | `mcp.rs:714` | `env_clear()` — the child starts from nothing |
| re-add allowlist | `mcp.rs:715-719` | 17 keys from `PASSTHROUGH_ENV` (`mcp.rs:39-63`), each only if present in the agent's own env |
| re-add Windows keys | `mcp.rs:720-725` | `PASSTHROUGH_ENV_WINDOWS` (`mcp.rs:71`) chained with `WINDOWS_SHELL_RESOLUTION_ENV` (`lib.rs:19-30`) via `windows_child_passthrough_env` (`mcp.rs:74-81`) |
| add client-supplied env | `mcp.rs:726-728` | applied **after** the allowlist, so a client value overrides an inherited one |
| working directory | `mcp.rs:729` | the session `cwd`, validated as absolute by the caller (`lib.rs:334-341`) |
| stderr | `mcp.rs:730` | `Stdio::inherit()` — child stderr is interleaved into the agent's stderr |
| stdin/stdout | `mcp.rs:737` | owned by `rmcp::transport::TokioChildProcess` (piped) |
| process group | `mcp.rs:732-733` | `process_group(0)` on unix only |
| console window | `mcp.rs:735`, `mcp.rs:992-1003` | `CREATE_NO_WINDOW` on Windows |

Env allowlist as shipped (`mcp.rs:39-63`):

| Group | Keys | Comment in source |
|---|---|---|
| Core | `PATH`, `HOME`, `TERM`, `LANG`, `LC_ALL`, `TMPDIR`, `XDG_CONFIG_HOME` | `mcp.rs:40-48` |
| SSH | `SSH_AUTH_SOCK`, `SSH_AGENT_PID` | "required for git clone/push over SSH" (`mcp.rs:49`) |
| Git | `GIT_ASKPASS`, `GIT_SSH_COMMAND`, `GIT_CONFIG_GLOBAL` | "operator-configured helpers and transport overrides" (`mcp.rs:53`) |
| Buzz identity | `NOSTR_PRIVATE_KEY`, `BUZZ_PRIVATE_KEY`, `BUZZ_RELAY_URL`, `BUZZ_AUTH_TAG` | "MCP subprocesses are trusted like the agent runtime" (`mcp.rs:57-62`) |

Signal handling: SIGKILL to the process group, via `nix::sys::signal::killpg` (`mcp.rs:844-853`); `nix` is a unix-only dependency (`Cargo.toml`, `[target.'cfg(unix)'.dependencies]`). Four call sites: `Drop for Server` (`mcp.rs:119`), `PgidGuard::drop` (`mcp.rs:748`), `kill_server` (`mcp.rs:446`), transport failure (`mcp.rs:465`). No SIGTERM-then-SIGKILL escalation and no `wait()`/reap after `killpg` — reaping is left to `rmcp`'s `TokioChildProcess` drop and to the OS.

#### rmcp (MCP client library)

`rmcp = { version = "1", default-features = false, features = ["client", "transport-child-process"] }` (`Cargo.toml`). Types used: `CallToolRequestParams` / `CallToolRequest` / `ClientRequest` / `ServerResult` (`mcp.rs:5`, `mcp.rs:577-579`), `RunningService<RoleClient, ()>` (`mcp.rs:83`), `TokioChildProcess` (`mcp.rs:7`, `mcp.rs:737`), `PeerRequestOptions` + `RequestHandle` (`mcp.rs:578`, `mcp.rs:789`), `ServiceError` (`mcp.rs:8`, classified at `mcp.rs:803-811`), `rmcp::model::{Content, RawContent, Tool}` (`mcp.rs:914-915`, `mcp.rs:711`).

Two integration details worth naming: the code reaches past the high-level API to poll the inner oneshot directly (`&mut handle.rx`, `mcp.rs:604`) so it can still own the handle in the cancel branch (comment `mcp.rs:596-597`) — that couples this file to `rmcp`'s `RequestHandle` internals. And pagination of `tools/list` is delegated to `list_all_tools()` (`mcp.rs:767`), so any page cap is `rmcp`'s, not this crate's.

#### OAuth authorization server

| Interaction | Site | Notes |
|---|---|---|
| discovery document (RFC 8414) | `auth.rs:160-190` | plain `GET`, JSON; only `authorization_endpoint` and `token_endpoint` are read, both required |
| authorization request | `auth.rs:588-599` | query string hand-built with `urlencoding::encode` per parameter; `response_type=code`, `code_challenge_method=S256` |
| token exchange | `auth.rs:608-628` | `POST` form: `grant_type=authorization_code`, `code`, `redirect_uri`, `code_verifier`, `client_id` |
| refresh | `auth.rs:205-231` | `POST` form: `grant_type=refresh_token`, `refresh_token`, `client_id` — no `scope`, no client secret (public client) |

Provider parameters for Databricks come from two places that must agree: `llm.rs:19-20` (`DATABRICKS_CLIENT_ID = "databricks-cli"`, `DATABRICKS_OAUTH_SCOPES = ["all-apis","offline_access"]`, used at `llm.rs:1190-1195`) and a hand-copied duplicate inside the `auth` subcommand (`lib.rs:135-144`). Because the cache filename hashes `discovery_url|client_id|scopes` (`auth.rs:446-454`), any divergence between those two sites would silently write the token to a different file than the runtime reads — the CLI would report success and the agent would still open a browser. The discovery URL template is likewise duplicated (`lib.rs:137-140` vs `llm.rs:1183-1186`).

#### Local loopback callback listener

`browser_pkce_flow` (`auth.rs:527-630`) starts an `axum::Router` with a single `GET /` handler (`auth.rs:539-568`) on a `tokio::net::TcpListener` bound to `127.0.0.1:0` (`auth.rs:571-573`). The ephemeral port is read back (`auth.rs:574-577`) and used to build `redirect_uri = http://localhost:{port}` (`auth.rs:578`), which is sent both in the authorize URL and in the token exchange — so the redirect URI is bound to the same value on both legs. The serve task is wrapped in `AbortOnDrop` (`auth.rs:584-586`) so it dies on every exit path; lifetime is bounded by `BROWSER_AUTH_TIMEOUT` (60 s, `auth.rs:39`, applied `auth.rs:601`).

Mismatch worth noting: the listener binds the IPv4 loopback literal, but the redirect URI uses the name `localhost`. On a host where `localhost` resolves to `::1` first, the browser can fail to reach the listener; RFC 8252 §7.3 recommends the literal address for exactly this reason. `grep -n '127.0.0.1\|localhost' auth.rs` shows the two forms at `auth.rs:571` and `auth.rs:578`.

The browser is launched through `webbrowser::open` (`auth.rs:599`) with the result discarded — the URL is always printed to stderr first (`auth.rs:598`) so a headless operator can copy it.

#### Databricks catalog endpoint

`fetch_v1_models` calls `{host}/api/2.0/serving-endpoints` (`catalog.rs:101`); `fetch_v2_models` calls `{host}/api/ai-gateway/v2/endpoints?page_size=100[&page_token=…]` (`catalog.rs:202`, `catalog.rs:207-214`). Both use a fresh `reqwest::Client::new()` created per call (`catalog.rs:80`) with `bearer_auth` (`catalog.rs:104`, `catalog.rs:217`) and no timeout configuration — contrast `llm.rs:53-57`, which sets `connect_timeout(10s)` and `read_timeout(cfg.llm_timeout)`.

The v2 query string is built by a hand-rolled `percent_encode` (`catalog.rs:182-193`) justified by a comment that says it "avoids requiring the `query` reqwest feature in buzz-agent's Cargo.toml" (`catalog.rs:201-202`). The crate already depends on `urlencoding = "2"` (`Cargo.toml`) and uses it in `auth.rs:589-594`, so the duplication is avoidable regardless of the reqwest feature question. `reqwest` is declared as workspace `version = "0.13", features = ["json","rustls"]` (root `Cargo.toml:93`) with `features = ["json","rustls","form"]` added by this crate — `query` is not in either list, which is consistent with the comment's premise even though the same job is already covered by an existing dependency.

Base URL for both paths is `cfg.base_url` (i.e. `DATABRICKS_HOST`) with trailing `/` trimmed (`catalog.rs:81`). Nothing validates the scheme, so an `http://` host is accepted (see Security).

#### Filesystem

| Path / pattern | Purpose | Site |
|---|---|---|
| `<dir>/AGENTS.md` for `$HOME` + git-root→cwd | project hints | `hints.rs:64-66` |
| `<dir>/.git` (file or directory) | git-root detection | `hints.rs:30` |
| `<cwd>/.agents/skills`, `<cwd>/.goose/skills`, `<cwd>/.claude/skills` | skill discovery | `hints.rs:8`, `hints.rs:208-210` |
| `$HOME/.agents/skills` | global skills | `hints.rs:212-214` |
| `<skill>/SKILL.md` | skill manifest + body | `hints.rs:129`, `builtin.rs:69` |
| every other file under `<skill>/` | supporting files | `hints.rs:158-202`, read at `builtin.rs:197` |
| `$HOME/.config/buzz-agent/oauth/<ns>/<sha256>.json` | OAuth token cache | `auth.rs:445-467`; created with `create_dir_all` (`auth.rs:146-149`), written `tmp`+`rename` (`auth.rs:195-200`) |

Symlink policy: both directory walkers use `std::fs::metadata` rather than `DirEntry::file_type`, deliberately, so symlinked skill directories and files are followed (`hints.rs:112-119`, `hints.rs:178-182`). Cycles are broken by a canonicalised visited set (`hints.rs:167-171`). All reads are `std::fs::read_to_string` with no size pre-check; `builtin.rs` at least moves them onto `spawn_blocking` (`builtin.rs:69`, `builtin.rs:197`), while `hints.rs:65-67` reads synchronously — that call happens on the `session/new` task (`lib.rs:356-357`), so a slow or huge `AGENTS.md` blocks a Tokio worker.

#### Intra-crate dependencies

| From | To | Site |
|---|---|---|
| `mcp.rs` | `config::{Config, HookServers}` | `mcp.rs:16` |
| `mcp.rs` | `types::{clamp, AgentError, McpServerStdio, ToolDef, ToolResult, ToolResultContent}` | `mcp.rs:17` |
| `mcp.rs` | `crate::WINDOWS_SHELL_RESOLUTION_ENV` (Windows only) | `mcp.rs:77-79` |
| `hints.rs` | `mcp::truncate_at_boundary` | `hints.rs:4` |
| `builtin.rs` | `hints::{strip_frontmatter, SkillEntry, MAX_SKILL_BODY_BYTES}`, `mcp::truncate_at_boundary`, `types::{ToolDef, ToolResult, ToolResultContent}` | `builtin.rs:9-11` |
| `catalog.rs` | `config::{Config, Provider}`, `llm::build_token_source`, `types::AgentError` | `catalog.rs:15-19` |
| `auth.rs` | `types::AgentError` only | `auth.rs:31` |
| consumers | `agent.rs` (registry + `load_skill` + `ResultBudget`), `handoff.rs` (`_PostCompact`), `lib.rs` (spawn, hints, catalog, `auth` CLI) | `agent.rs:13`, `agent.rs:118-120`, `agent.rs:225`, `handoff.rs:73-81`, `lib.rs:135-146`, `lib.rs:356-357`, `lib.rs:390`, `lib.rs:447-452` |

Note the layering oddity: text-truncation helpers live in `mcp.rs` and are imported by two modules that have nothing to do with MCP (`hints.rs:4`, `builtin.rs:10`). `catalog.rs` depends on `llm.rs` for auth construction, so the "catalog" module cannot be used without the LLM transport module.

#### External consumers of this group

- `desktop/src-tauri/src/commands/agent_models.rs:709-758` calls `buzz_agent_pkg::discover_databricks_models` with a `Config::for_discovery` (`config.rs:840-871`), maps `LlmAuth` to "fall through to subprocess" (`agent_models.rs:731-734`), treats an empty list as an error (`agent_models.rs:740-742`), and **redacts** `DATABRICKS_TOKEN` out of any surfaced error string (`agent_models.rs:727-728`, `:736`).
- `tests/databricks_oauth.rs:20` imports the `auth` API directly and stands up a fake OIDC server plus token endpoint (`:36-82`), re-deriving the cache path with its own copy of the hashing logic (`:84-97`).

#### Duplicated rather than depended upon

| Duplicate | Sites | Risk |
|---|---|---|
| Databricks OAuth client id, scopes, discovery URL template | `lib.rs:135-144` vs `llm.rs:19-20`, `llm.rs:1183-1195` | divergence silently changes the cache filename |
| Percent-encoding | `catalog.rs:182-193` vs the `urlencoding` dependency used at `auth.rs:589-594` | two encoders with different reserved-set behaviour |
| OAuth cache-path derivation | `auth.rs:445-467` vs test helper `tests/databricks_oauth.rs:84-97` | the test re-implements the production rule, so a change to the hash inputs keeps the test green |
| Token-JSON on-disk shape | `auth.rs:110-118` vs test fixtures at `auth.rs:750-756`, `auth.rs:786-792`, `tests/databricks_oauth.rs:99-103` | same class of drift |

#### Declared dependencies and their use in this group

`async-trait` (`auth.rs:24`), `base64` (`auth.rs:26`), `sha2` (`auth.rs:29`), `hex` (`auth.rs:452`), `axum` (`auth.rs:528`), `urlencoding` (`auth.rs:589`), `webbrowser` (`auth.rs:599`), `reqwest` (`auth.rs:27`, `catalog.rs:13`), `arc-swap` (`mcp.rs:5`), `getrandom` (`mcp.rs:824`, `auth.rs:511`, `auth.rs:520`), `nix` (`mcp.rs:846-847`), `rmcp` (`mcp.rs:6-11`), `serde_yaml` (`hints.rs:93`), `tokio` (all five), `tracing` (`mcp.rs`, `auth.rs`), `serde`/`serde_json` (throughout). `tempfile` is dev-only and used by the unit tests in `hints.rs`, `builtin.rs`, and `auth.rs`.

The `form` reqwest feature is exercised only by this group (`auth.rs:218`, `auth.rs:617`); the `json` feature is used by both this group (`auth.rs:169`, `auth.rs:227`, `auth.rs:626`, `catalog.rs:117`) and `llm.rs`. No declared dependency of the crate is unused by the crate as a whole, so there is nothing to report as dead in the manifest.
