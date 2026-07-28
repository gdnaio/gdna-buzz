## Module: buzz-agent — MCP registry, OAuth, hints & catalog (`crates/buzz-agent/src`)
### Aspect: API Surface

#### Module visibility gate

`lib.rs:2-12` declares: `pub mod auth`, `pub mod catalog`, `pub mod config`, `pub mod types`; and `mod agent`, `mod builtin`, `mod handoff`, `mod hints`, `mod llm`, `mod mcp`, `mod wire`. Everything marked `pub` inside `mcp.rs`, `hints.rs`, and `builtin.rs` is therefore reachable only inside the crate — the `pub` keyword there is decoration, not export. Only `auth.rs` and `catalog.rs` contribute to the real crate API.

Re-exports at `lib.rs:14`: `pub use catalog::{discover_databricks_models, ModelEntry, DATABRICKS_V2_KNOWN_MODELS};`. `catalog::discovery_failure_fallback` is public but not re-exported — external callers must path through `buzz_agent::catalog::`.

#### Genuinely public items from this group

| Item | Site | Signature / shape | Doc comment? |
|---|---|---|---|
| `auth::TokenSource` (trait) | `auth.rs:43-74` | `async fn bearer(&self) -> Result<String, AgentError>`; provided `bearer_no_browser`, `refresh_now(&self, rejected: &str)` | trait yes (`auth.rs:41-42`); `bearer` itself no |
| `auth::StaticTokenSource` | `auth.rs:76` | tuple struct over `String` | yes (`auth.rs:75`) |
| `StaticTokenSource::new` | `auth.rs:79-81` | `new(token: impl Into<String>) -> Self` | **no** |
| `auth::PkceOAuthConfig` | `auth.rs:97-108` | 5 pub fields | yes (`auth.rs:90-96`) |
| `auth::PkceOAuthTokenSource` | `auth.rs:134-142` | opaque; all fields private | yes (`auth.rs:126-133`) |
| `PkceOAuthTokenSource::new` | `auth.rs:144-158` | `-> Result<Arc<Self>, AgentError>` (returns `Arc`, not `Self`) | **no** |
| `PkceOAuthTokenSource::interactive_login` | `auth.rs:235-241` | `async -> Result<(), AgentError>` | yes (`auth.rs:233-234`) |
| `catalog::ModelEntry` | `catalog.rs:22-27` | `{ id, name }` | yes (`catalog.rs:20-21`) |
| `catalog::DATABRICKS_V2_KNOWN_MODELS` | `catalog.rs:31-33` | `&[&str; 2]` | yes (`catalog.rs:28-30`) |
| `catalog::discovery_failure_fallback` | `catalog.rs:48-66` | `(Provider, &str) -> Vec<ModelEntry>` | yes (`catalog.rs:35-47`) |
| `catalog::discover_databricks_models` | `catalog.rs:76-91` | `async (&Config) -> Result<Vec<ModelEntry>, AgentError>` | yes (`catalog.rs:68-75`) |

`TokenSource` uses `#[async_trait]` (`auth.rs:43`) and is object-safe — `llm.rs:49` holds `auth: Arc<dyn TokenSource>` and `build_token_source` returns one (`llm.rs:1175`). Two impls ship: `StaticTokenSource` (`auth.rs:84-89`, `bearer` only) and `PkceOAuthTokenSource` (`auth.rs:244-358`, overrides all three methods).

Trait-default semantics worth naming: `bearer_no_browser` defaults to `bearer` (`auth.rs:52-54`) — so a static source silently satisfies "must not open a browser", while `refresh_now`'s default returns the *same* rejected token (`auth.rs:70-72`), which makes the caller's 401 retry fail terminally rather than loop.

#### Crate-internal API (`pub` inside private modules)

| Item | Site | Notes |
|---|---|---|
| `mcp::MAX_MCP_SERVERS` | `mcp.rs:26` | `pub const`, unreachable externally |
| `mcp::ResultBudget` + fields | `mcp.rs:32-37` | consumed by `agent.rs:386-389`, `mcp.rs:355-358` |
| `McpRegistry::spawn_all` | `mcp.rs:172-176` | `async (&Config, &[McpServerStdio], &str) -> Result<Self, AgentError>` — **no doc comment** |
| `McpRegistry::server_of` | `mcp.rs:266-270` | `(&str) -> Option<&str>` — no doc comment |
| `McpRegistry::has` | `mcp.rs:272-274` | no doc comment |
| `McpRegistry::is_hook` | `mcp.rs:279-284` | documented (`mcp.rs:276-278`) |
| `McpRegistry::tools` | `mcp.rs:286-313` | no doc comment |
| `McpRegistry::call_hooks` | `mcp.rs:315-320` | `self: &Arc<Self>`; documented (`mcp.rs:307-314`) |
| `McpRegistry::kill_server` | `mcp.rs:421-425` | documented (`mcp.rs:414-420`) |
| `McpRegistry::call` | `mcp.rs:485-493` | no doc comment |
| `mcp::truncate_at_boundary` / `truncate_middle` | `mcp.rs:866`, `mcp.rs:886` | `pub(crate)`; imported by `hints.rs:4` and `builtin.rs:10` |
| `hints::build_hints_section` | `hints.rs:219-221` | `(&Path) -> (String, Vec<SkillEntry>)` — **no doc comment** |
| `hints::SkillEntry` + fields | `hints.rs:14-25` | no struct-level doc; two fields documented |
| `hints::MAX_SKILL_BODY_BYTES` | `hints.rs:7` | no doc comment |
| `hints::strip_frontmatter` | `hints.rs:254-263` | `pub(crate)`, documented |
| `builtin::LOAD_SKILL_TOOL` | `builtin.rs:13` | `"load_skill"` — no doc comment |
| `builtin::load_skill_def` | `builtin.rs:16-39` | `-> ToolDef`, documented |
| `builtin::call_load_skill` | `builtin.rs:41-116` | `async (&Value, &[SkillEntry]) -> ToolResult`, documented |
| `catalog::parse_v1_endpoints` / `parse_v2_endpoints_page` | `catalog.rs:131`, `catalog.rs:265` | `pub(crate)` purely for unit tests |

`AGENTS.md` states "New public API must have doc comments". The undocumented items above — most notably the two public constructors `StaticTokenSource::new` (`auth.rs:79`) and `PkceOAuthTokenSource::new` (`auth.rs:144`) — do not meet that bar.

#### The built-in tool contract (`builtin.rs`)

`load_skill_def()` (`builtin.rs:16-39`) emits a `ToolDef`:

- `name`: `"load_skill"` (`builtin.rs:13`, `builtin.rs:18`);
- `description`: instructs the model to call it before using a skill and documents the `"skill-name/relative/path"` form (`builtin.rs:19-25`);
- `input_schema`: `{"type":"object","properties":{"name":{"type":"string", …}},"required":["name"]}` (`builtin.rs:26-38`).

The schema declares exactly one property and no `additionalProperties: false`; extra arguments are ignored because only `name` is read (`builtin.rs:42`). The tool is appended to the LLM tool list only when at least one skill was discovered (`agent.rs:117-119`) and is dispatched in-process before any MCP lookup (`agent.rs:318-325`).

#### MCP JSON-RPC surface consumed

| Direction | Method / notification | Site | Bound |
|---|---|---|---|
| out (request) | `initialize` (implicit in `().serve(transport)`) | `mcp.rs:757-766` | `init_timeout` |
| out (request) | `tools/list` (`peer().list_all_tools()`) | `mcp.rs:767-780` | `init_timeout`; paginates internally inside `rmcp` |
| out (request) | `tools/call` (`CallToolRequest` via `send_cancellable_request`) | `mcp.rs:576-598` | `tool_timeout` applied by the caller (`agent.rs:509-520`) |
| out (notification) | `notifications/cancelled` (`handle.cancel(Some("session cancelled"))`) | `mcp.rs:788-800` | fire-and-forget in a detached task |
| in | none | — | the client is constructed as `()` (`mcp.rs:83`), so no server-initiated request or notification is handled |

Because the service handler is the unit type, MCP features that require client-side handlers — sampling, roots, elicitation, and `notifications/tools/list_changed` — are silently unsupported: `grep -rn 'list_changed\|sampling\|roots' crates/buzz-agent/src` returns zero matches. Tool-set drift is instead detected lazily by comparing the cached `tools` list on each call (`mcp.rs:500-506`, `mcp.rs:526-532`).

Responses other than `CallToolResult` are rejected as "unexpected response type" (`mcp.rs:601-605`). Application-level JSON-RPC errors are converted into a successful `ToolResult` with `is_error: true` and the text `Tool call rejected: {e}` (`mcp.rs:625-635`), while transport errors kill the server (`mcp.rs:616-624`, classifier at `mcp.rs:803-811`).

#### HTTP surface called (outbound)

| Endpoint | Method | Site | Auth |
|---|---|---|---|
| `cfg.discovery_url` (RFC 8414 document) | GET | `auth.rs:160-190` | none |
| `endpoints.token_endpoint` (refresh grant) | POST form | `auth.rs:205-231` | none (public client; `client_id` in body) |
| `endpoints.token_endpoint` (authorization_code grant) | POST form | `auth.rs:608-628` | none |
| `endpoints.authorization_endpoint` | opened in a browser | `auth.rs:588-599` | n/a |
| `{host}/api/2.0/serving-endpoints` | GET | `catalog.rs:96-129` | `bearer_auth` |
| `{host}/api/ai-gateway/v2/endpoints?page_size=100[&page_token=…]` | GET | `catalog.rs:195-243` | `bearer_auth` |

Local inbound listener: `browser_pkce_flow` binds an axum router with a single `GET /` route on `127.0.0.1:0` (`auth.rs:539-577`) for the redirect callback. It is not part of the crate's declared API and is torn down on every exit path via `AbortOnDrop` (`auth.rs:584-586`, `auth.rs:426-432`).

#### Error variants surfaced

All five files return `crate::types::AgentError`. Usage is uneven:

| Variant | Used by | Sites (verified by `grep -n`) |
|---|---|---|
| `Mcp` | every failure in `mcp.rs` (23 construction sites) | `mcp.rs:136,142,178,198,201,233,239,243,250,496,502,528,535,571,588,613,624,702,738,760,763,770,773` |
| `Cancelled` | `do_call` cancel branches | `mcp.rs:593`, `mcp.rs:602` |
| `Llm` | all OAuth failures (24 sites) and all catalog HTTP failures (8 sites) | `auth.rs:148,166,168,171,176,182,193,197,199,221,224,229,458,486,511,520,573,576,603,604,605,620,623,628`; `catalog.rs:107,112,118,136,221,227,233,272` |
| `LlmAuth` | refresh-token exhaustion and the no-browser path | `auth.rs:341`, `auth.rs:355`, `auth.rs:416` |
| `InvalidParams` | wrong provider passed to discovery | `catalog.rs:86-90` |

`AgentError::json_rpc_code()` (`types.rs:249-256`) maps `LlmAuth → -32001` and everything else in this group to `-32000` / `-32602`, so an OAuth failure raised as `Llm` (e.g. a dead discovery endpoint, `auth.rs:166`) is indistinguishable on the wire from a provider outage.

#### Test coverage — API Surface

Covered: the public `TokenSource` surface is exercised through the trait from an integration test (`tests/databricks_oauth.rs:20` imports `PkceOAuthConfig, PkceOAuthTokenSource, TokenSource`; `cache_hit_short_circuits_network` `:105`, `expired_cache_silently_refreshes` `:144`, `refreshed_token_is_persisted_to_disk` `:179`, `refresh_now_runs_grant_on_unexpired_rejected_token` `:212`, `refresh_now_coalesces_when_another_caller_already_refreshed` `:261`, `refresh_now_without_refresh_token_is_terminal` `:305`). `load_skill`'s schema-driven behaviour is covered by `builtin.rs:277-573` (12 `#[tokio::test]`s) plus one end-to-end path (`tests/hints_integration.rs:517 load_skill_tool_returns_body`). `catalog`'s pure parsers are covered by six unit tests (`catalog.rs:307-400`).

Not covered: `discover_databricks_models` itself, and both HTTP fetchers — `grep -rn 'discover_databricks_models\|fetch_v1_models\|fetch_v2_models\|percent_encode' crates/buzz-agent/tests` returns zero matches; the only test of the discovery *contract* lives in `lib.rs:832-878` and injects a future instead of calling the real function. `interactive_login` (`auth.rs:235`) has no test. `McpRegistry::call_hooks` is covered indirectly through the agent loop (`tests/regressions.rs:787`, `872`, `927`, `979`, `1035`, `1112`, `1514`) but never called directly, so its documented determinism ("results in config order", `mcp.rs:309-310`) is untested with more than one hook server.
