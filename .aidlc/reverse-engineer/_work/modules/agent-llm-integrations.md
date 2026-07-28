## Module: buzz-agent — LLM providers & configuration (`crates/buzz-agent/src`)
### Aspect: Integrations
`llm.rs` is the crate's only outbound-HTTP surface for model traffic. `config.rs` makes zero network calls — grep for `reqwest`, `Client`, `http`, and `await` in `config.rs` returned zero matches; its only I/O is one `std::fs::read_to_string` for `BUZZ_AGENT_SYSTEM_PROMPT_FILE` (`config.rs:790`).

#### Outbound HTTP: endpoints and auth headers
| Provider / route | Endpoint | Auth header | Extra headers | Site |
|---|---|---|---|---|
| Anthropic Messages | `POST {ANTHROPIC_BASE_URL}/v1/messages` | `x-api-key: {cfg.api_key}` | `anthropic-version: {cfg.anthropic_api_version}` | `llm.rs:256-264` |
| OpenAI Responses | `POST {OPENAI_COMPAT_BASE_URL}/responses` | `Authorization: Bearer {token}` via `bearer_auth` | — | `llm.rs:285`, `llm.rs:356` |
| OpenAI Chat Completions | `POST {OPENAI_COMPAT_BASE_URL}/chat/completions` | `Bearer` | — | `llm.rs:290`, `llm.rs:356` |
| Databricks legacy serving | `POST {DATABRICKS_HOST}/serving-endpoints/{model}/invocations` | `Bearer` | — | `llm.rs:335-341` |
| DatabricksV2 gateway (OpenAI) | `POST {DATABRICKS_HOST}/ai-gateway/openai/v1/responses` | `Bearer` | — | `llm.rs:700` |
| DatabricksV2 gateway (Anthropic) | `POST {DATABRICKS_HOST}/ai-gateway/anthropic/v1/messages` | `Bearer` | — | `llm.rs:701` |
| DatabricksV2 gateway (MLflow) | `POST {DATABRICKS_HOST}/ai-gateway/mlflow/v1/chat/completions` | `Bearer` | — | `llm.rs:702` |
| Databricks OIDC discovery (indirect) | `{DATABRICKS_HOST}/oidc/.well-known/oauth-authorization-server` | n/a | — | URL built at `llm.rs:1188-1191`, fetched inside `auth.rs` |

Every request carries `content-type: application/json` and a pre-serialized body, applied uniformly in `post` (`llm.rs:1063-1065`). Body bytes are serialized **once** before the retry loop (`llm.rs:1053-1054`) and cloned per attempt (`llm.rs:1065`), so a serialization failure can never be mistaken for a retryable transport error — the rationale is stated at `llm.rs:1023-1025`.

#### TLS handling
The HTTP client is built once per `Llm` in `Llm::new` (`llm.rs:54-58`):
- `connect_timeout(10s)` (`llm.rs:55`)
- `read_timeout(cfg.llm_timeout)` (`llm.rs:56`)
- no `.timeout(...)`, no `danger_accept_invalid_certs`, no custom root store, no proxy configuration, no `.https_only(true)`

TLS comes from the `rustls` feature declared in `crates/buzz-agent/Cargo.toml:29` (`reqwest = { workspace = true, features = ["json", "rustls", "form"] }`). Certificate verification is therefore reqwest/rustls default — grep for `danger_accept_invalid`, `add_root_certificate`, and `use_native_tls` in `llm.rs` returned zero matches.

**Nothing in this group requires HTTPS.** `Config::validate` (`config.rs:870-951`) performs no scheme or host check on `base_url`; the only function that inspects the scheme is `is_openai_host` (`config.rs:1029-1039`), which explicitly accepts `http://` (`config.rs:1032`) and has a test asserting `http://eu.api.openai.com/v1 == true` (`config.rs:1273`). So `ANTHROPIC_BASE_URL=http://…` or `DATABRICKS_HOST=http://…` sends credentials in plaintext with no warning. Since `reqwest` has no `https_only` here, `.build()` will not reject it either.

#### Crate dependencies actually used by these two files
| Crate | Used for | Site |
|---|---|---|
| `reqwest` | `Client`, `RequestBuilder`, `Response`, `Error`, `bearer_auth`, chunked body reads | `llm.rs:4`, `llm.rs:54`, `llm.rs:356`, `llm.rs:983-998`, `llm.rs:1132` |
| `serde_json` | every request body and every response parse | `llm.rs:5`; `config.rs:129` (`use serde_json::json` inside `anthropic_thinking_config`) |
| `tokio` | `tokio::time::sleep` in backoff | `llm.rs:1017` |
| `tracing` | five `warn!` sites in `llm.rs`, five in `config.rs` | `llm.rs:381`, `llm.rs:1040`, `llm.rs:1071`, `llm.rs:1100`; `config.rs:149`, `config.rs:219`, `config.rs:512`, `config.rs:542`, `config.rs:566` |
| `getrandom` | jitter entropy (`getrandom::fill`) | `llm.rs:1012` |
| `std::sync::atomic` | the `auto_upgraded` latch | `llm.rs:1`, `llm.rs:62`, `llm.rs:278`, `llm.rs:385` |

`getrandom` (declared `Cargo.toml:32`) is used in exactly one place in the whole crate — `llm.rs:1012`. If jitter entropy is unavailable, backoff degrades silently to the un-jittered base delay (`llm.rs:1013-1015`).

Test-only dependencies pulled in by `llm.rs`'s test module: `tracing_subscriber` (`llm.rs:1223`, `llm.rs:2440-2468`), `async_trait` (`llm.rs:2618`), `tokio::net::TcpListener` + `tokio::io` (`llm.rs:2180-2182`, `llm.rs:2246-2248`, etc.). `config.rs`'s test module uses `serde::Deserialize` (`config.rs:2652`) and `serde_json::from_str` (`config.rs:2669`).

#### Intra-crate dependencies
| Import | From | Used for |
|---|---|---|
| `PkceOAuthConfig`, `PkceOAuthTokenSource`, `StaticTokenSource`, `TokenSource` | `auth.rs` | `llm.rs:7`, consumed in `build_token_source` (`llm.rs:1175-1204`) and `Llm.auth` (`llm.rs:50`) |
| `is_openai_host`, `normalize_effort_for_anthropic_route`, `normalize_effort_for_openai_route`, `Config`, `OpenAiApi`, `Provider`, `ThinkingEffort` | `config.rs` | `llm.rs:8-11` |
| `crate::config::anthropic_thinking_config` | `config.rs` | called fully-qualified at `llm.rs:451` |
| `AgentError`, `HistoryItem`, `LlmResponse`, `ProviderStop`, `ToolCall`, `ToolDef`, `ToolResultContent` | `types.rs` | `llm.rs:12-14` |

`config.rs` has **no** intra-crate imports at all — it depends only on `std`, `serde_json`, and `tracing`. That makes it the crate's dependency root and explains why the effort tables live there rather than in `llm.rs`.

Consumers of this group elsewhere in the crate:
| Consumer | What it uses |
|---|---|
| `lib.rs:41` | `Config`, `MAX_SYSTEM_PROMPT_BYTES`, `PROTOCOL_VERSION`; `Config::from_env()` at `lib.rs:160`, `Llm::new` at `lib.rs:161` |
| `agent.rs:8` | `Config`, `MAX_PROMPT_BYTES`, `MAX_TOOL_CALLS_PER_TURN`, `MAX_TOOL_RESULT_BYTES`; `Llm::complete` at `agent.rs:124` |
| `handoff.rs:3` | `HANDOFF_MAX_OUTPUT_TOKENS`, `HANDOFF_MAX_TOOL_NAMES`, `HANDOFF_ORIGINAL_TASK_MAX_BYTES`; `summarize` via `handoff.rs:51`, `handoff.rs:197` |
| `catalog.rs:17` | `llm::build_token_source`, called at `catalog.rs:77` |

#### Duplicated code that should be a dependency
`lib.rs:129-153` (`auth_subcommand`) re-declares the entire Databricks PKCE configuration that `build_token_source` builds at `llm.rs:1185-1198`, with the identifiers inlined as string literals rather than referencing the constants:

| Value | Canonical definition | Duplicated at |
|---|---|---|
| discovery URL template `{host}/oidc/.well-known/oauth-authorization-server` | `llm.rs:1188-1191` | `lib.rs:135-138` |
| client id `databricks-cli` | `DATABRICKS_CLIENT_ID`, `llm.rs:19` | `lib.rs:141` (bare literal) |
| scopes `all-apis`, `offline_access` | `DATABRICKS_OAUTH_SCOPES`, `llm.rs:20` | `lib.rs:142` (bare literals) |
| cache namespace `databricks` | `llm.rs:1194` | `lib.rs:143` |

Both constants are private module-level `const`s in a private module (`llm.rs:19-20`), so `lib.rs` structurally *cannot* reference them without a visibility change. The result is that `buzz-agent auth databricks` and the runtime token source can drift apart — e.g. a scope added to `DATABRICKS_OAUTH_SCOPES` would not be requested by the interactive login, producing a cached token that the runtime then rejects. No test covers the two staying in sync; grep for `DATABRICKS_OAUTH_SCOPES` across `crates/` returned only `llm.rs:20` and `llm.rs:1195`.

A second, smaller duplication: `strip_catalog_prefix` (`config.rs:89-97`) is re-implemented verbatim inside the test helper at `config.rs:2570-2582` instead of being called.

A third: `Llm::summarize`'s Anthropic body (`llm.rs:164-173`) duplicates the message/system/max_tokens shape that `anthropic_body` builds (`llm.rs:446-447`), and its Chat body (`llm.rs:192-203`) duplicates `openai_body`'s (`llm.rs:547-548`).

#### Cross-language integration: the desktop effort table
`config.rs` is coupled to TypeScript through a shared JSON fixture. The test at `config.rs:2666-2700` does `include_str!("../../../desktop/src/features/agents/ui/effortTable.fixture.json")` (`config.rs:2667-2668`) — a compile-time path dependency from a Rust crate into the desktop app's source tree. The fixture exists (7,790 bytes, `desktop/src/features/agents/ui/effortTable.fixture.json`) and the TS side is `desktop/src/features/agents/ui/buzzAgentConfig.ts` with tests `buzzAgentConfig.test.mjs` and `effortTable.fixture.test.mjs`.

Two consequences worth flagging:
1. Moving or renaming that desktop file breaks the Rust build, not just a Rust test — `include_str!` is a compile-time macro inside `#[cfg(test)]`, so it breaks `cargo test -p buzz-agent`.
2. The Rust side of the guard compares the fixture against a **test-only** re-implementation (`valid_effort_values_for_provider_model`, `config.rs:2559-2650`), not against production functions. See Debt and Business Rules for why this can go green while production drifts.

#### Subprocesses
Neither file spawns a subprocess. grep for `Command`, `spawn`, and `std::process` in `llm.rs` and `config.rs` returned zero matches. The one process-related fact worth recording for this group: `cfg.api_key` is never handed to a child environment — MCP children are spawned with `env_clear()` plus an explicit whitelist (`mcp.rs:714`, whitelist at `mcp.rs:41-47`), so the provider credential this group reads does not cross into tool subprocesses.

#### Declared dependencies not used by this group
`crates/buzz-agent/Cargo.toml` declares `serde_yaml` (line 27), `rmcp` (line 30), `arc-swap` (line 31), `axum` (line 37), `base64` (line 38), `hex` (line 39), `sha2` (line 40), `urlencoding` (line 41), `webbrowser` (line 42), and `nix` (line 45). None of them are referenced from `llm.rs` or `config.rs` — grep for each in these two files returned zero matches. They belong to sibling modules (`auth.rs` for base64/sha2/hex/urlencoding/webbrowser/axum, `mcp.rs` for rmcp/nix, `hints.rs`/`persona` loading for serde_yaml). I did not audit whether any is genuinely unused crate-wide; that is outside this group's scope.
