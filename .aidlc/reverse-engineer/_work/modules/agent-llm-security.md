## Module: buzz-agent — LLM providers & configuration (`crates/buzz-agent/src`)
### Aspect: Security
This group reads every provider credential the agent uses and constructs every outbound request that carries one. The trust model in `crates/buzz-agent/README.md:216` is "the operator who launched the agent … The harness, MCP server binaries, and API keys are all trusted", which is a reasonable framing for most of what follows — but two items below are weaker than that model implies.

#### Where each secret is read
| Secret | Env var | Read site | Stored as | Sent as |
|---|---|---|---|---|
| Anthropic API key | `ANTHROPIC_API_KEY` | `config.rs:741` (presence probe), `config.rs:759` (`req()`) | `Config.api_key: String` (`config.rs:723`) | `x-api-key` header (`llm.rs:259`) |
| OpenAI-compat API key | `OPENAI_COMPAT_API_KEY` | `config.rs:742`, `config.rs:769` | `Config.api_key: String` | `Authorization: Bearer` via `StaticTokenSource` (`llm.rs:1178`, `llm.rs:356`) |
| Databricks static token | `DATABRICKS_TOKEN` | `config.rs:779` (`unwrap_or_default`) | `Config.api_key: String`, `""` when unset | `Bearer` via `StaticTokenSource` (`llm.rs:1181-1183`) |
| Databricks OAuth access token | none — minted by PKCE flow | `llm.rs:1199` builds the source; token itself lives in `auth.rs` | `Arc<dyn TokenSource>` (`llm.rs:50`) | `Bearer` (`llm.rs:356`) |

`Config.api_key` is a plain `String`. There is **no** redaction, `Secret<T>`, `SecretString`, or `zeroize` wrapper anywhere in this group — `grep -rni 'zeroize\|secrecy\|redact\|Secret<' llm.rs config.rs` returned zero matches. The key therefore lives in unzeroed heap memory for the process lifetime and is cloned at `llm.rs:1178` and `llm.rs:1182`.

#### Can a secret reach a log line?
No, on the paths I could trace. All nine `tracing` events in this group were checked individually (`llm.rs:381`, `llm.rs:1040`, `llm.rs:1071`, `llm.rs:1100`; `config.rs:149`, `config.rs:219`, `config.rs:512`, `config.rs:542`, `config.rs:566`) and none takes `cfg`, `api_key`, `bearer`, or a request body as a field or format argument. `llm.rs:381-386` logs `provider_message = body` — but `body` there is the *provider's error text* (`llm.rs:374`), not the request body.

`Config` derives `Debug` (`config.rs:686`) with `api_key` as a normal field, so `{cfg:?}` would print the key verbatim. I found no site that does this: `grep -rn 'cfg:?\|?cfg\|{:?}", cfg\|debug!(.*cfg' --include='*.rs' crates/buzz-agent/` returned zero matches. This is a latent hazard rather than a live leak — a single `debug!(?cfg)` added anywhere would expose the key with no compiler or lint objection.

#### Can a secret reach an error message?
Not directly, but provider error bodies are propagated wholesale and those are attacker-influenced (see below). `read_error_body` (`llm.rs:983-998`) captures up to `MAX_LLM_ERROR_BODY_BYTES` = 4 KiB (`llm.rs:23`) of the response and embeds it verbatim into:
- `AgentError::LlmAuth(body)` for 401/403 (`llm.rs:1087`)
- `AgentError::Llm("exhausted retries: {status}: {body}")` (`llm.rs:1116-1119`)
- `AgentError::LlmModelNotFound("{status}: {body}")` (`llm.rs:1109-1112`)
- `AgentError::Llm("{status}: {body}")` for any other non-2xx (`llm.rs:1115-1118`)

Those strings then cross the process boundary to the ACP client as JSON-RPC error messages via `e.to_string()` at `lib.rs:392` and `lib.rs:760`. There is no scanning, masking, or allowlisting of the body content on that path. A misconfigured or hostile gateway that echoes the presented `Authorization` header into its error body would have that value forwarded to the client and to stderr. The 4 KiB cap bounds the volume but not the sensitivity.

#### Can a secret reach a serialized payload?
No. `Config` has no `Serialize` derive (`config.rs:686` is `Debug, Clone` only), and no body builder reads `cfg.api_key` — `grep -n 'api_key' llm.rs` returns only the header application at `llm.rs:259`, the two `StaticTokenSource::new` clones (`llm.rs:1178`, `llm.rs:1182`), the emptiness check (`llm.rs:1181`), and doc comments (`llm.rs:48`, `llm.rs:1167`). `anthropic_body`, `openai_body`, and `responses_body` take `&Config` but touch only `max_output_tokens` (`llm.rs:446`, `llm.rs:548`, `llm.rs:666`).

#### Can a secret reach a child process environment?
No. Neither file spawns a process — grep for `Command`, `spawn`, `std::process` in `llm.rs` and `config.rs` returned zero matches. MCP children are spawned with `env_clear()` (`mcp.rs:714`) plus an explicit whitelist (`mcp.rs:41-47`), so `ANTHROPIC_API_KEY` and friends are stripped. `crates/buzz-agent/README.md:221` documents this and is accurate, though the README's whitelist (`PATH`, `HOME`, `TERM`, `LANG`, `LC_ALL`, `TMPDIR`) omits `XDG_CONFIG_HOME`, which the code also passes (`mcp.rs:47`) — a docs gap in a sibling module, noted only because it is part of the same claim.

#### TLS / scheme validation on provider base URLs — the significant gap
`Config::validate` (`config.rs:870-951`) performs **no** validation of `base_url` whatsoever: no scheme check, no host allowlist, no rejection of `http://`, no rejection of a bare hostname or an IP literal. All 14 checks in `validate` are numeric or duration bounds (`config.rs:876-931`) plus the thinking-effort/provider compatibility rule (`config.rs:939-949`).

Consequences:
1. **Plaintext credential transmission is permitted.** `ANTHROPIC_BASE_URL=http://gateway.internal` causes `x-api-key: <key>` to be sent over cleartext HTTP at `llm.rs:257-260`. Same for `OPENAI_COMPAT_BASE_URL` (`llm.rs:285`, `llm.rs:290`) and `DATABRICKS_HOST` (`llm.rs:337`, `llm.rs:700-702`). The reqwest client is not built with `.https_only(true)` (`llm.rs:54-58`), so nothing downstream objects.
2. **An arbitrary host can receive the credential.** `base_url` is used unvalidated as a URL prefix at `llm.rs:257`, `llm.rs:337`, `llm.rs:343`, and `llm.rs:1190`. Whoever controls the environment controls where the key goes. Under the stated trust model ("the operator … is trusted") this is by design; it becomes a real risk where the environment is populated from a less-trusted source — and it is, in this repo: `buzz-persona` writes provider env vars for agent spawn (`crates/buzz-persona/src/resolve.rs:374` emits `BUZZ_AGENT_MODEL`; the persona env surface is described at `crates/buzz-acp/src/config.rs:533`). A persona that could set `ANTHROPIC_BASE_URL` would redirect the operator's key.
3. `is_openai_host` (`config.rs:1029-1039`) deliberately accepts `http://` (`config.rs:1032`) and there is a test asserting `http://eu.api.openai.com/v1 == true` (`config.rs:1273`). So the one function that does look at the scheme treats plaintext as equivalent to TLS for routing purposes.

The one hardening that *is* present in `is_openai_host` is lookalike resistance: the host is extracted up to the first `/` or `:` (`config.rs:1037`) and compared as an exact match or a `.openai.com` suffix (`config.rs:1038`), so `api.openai.com.evil.example` returns `false` — asserted at `config.rs:1277`. That prevents an attacker-controlled lookalike host from *causing endpoint selection to change*, but it does not prevent the request from being sent there.

Also worth noting: the Databricks OIDC discovery URL is built by string concatenation from `cfg.base_url` (`llm.rs:1188-1191`) with only a trailing-slash trim. An attacker-controlled `DATABRICKS_HOST` therefore controls the OAuth authorization-server metadata document, which in turn controls the authorization and token endpoints the PKCE flow uses.

#### Bearer-token handling
- The bearer is fetched once per call (`llm.rs:352`) and held in a local `String` (`llm.rs:352`, reassigned at `llm.rs:360`). It is not logged, not stored on `Llm`, and not embedded in a body.
- `refresh_now(&bearer)` (`llm.rs:360`) passes the *rejected* token to the refresh so the source can key its refresh decision off it — a reasonable design that avoids a thundering-herd refresh, but it means the rejected credential is passed across a trait boundary into `auth.rs` where I did not audit its handling.
- The refresh-once-per-call guard is a local `bool` (`llm.rs:353`), so it cannot be exhausted by an earlier turn (`llm.rs:345-348`). This bounds a credential-rejection loop to two requests per call — verified by `post_openai_persistent_401_propagates_after_one_retry` (`llm.rs:2740`) and `post_openai_persistent_403_propagates_after_one_retry` (`llm.rs:2768`).
- Treating **403** as refreshable (`llm.rs:1086-1088`) means a pure authorization denial triggers one unnecessary token refresh. For a PKCE source that could mean an unnecessary network round-trip per denied call; for a static source `refresh_now` is a no-op returning the same token (tested at `llm.rs:2819`).

#### Prompt-injection surface
Everything that flows into a request body from this group is untrusted-by-construction: tool results and model output. The relevant defences and their absences:

| Surface | Handling | Site |
|---|---|---|
| System prompt | taken from env or file, concatenated as-is; no escaping | `config.rs:785-792`; injected at `llm.rs:446`, `llm.rs:501` (as `system` role), `llm.rs:665` (as `instructions`) |
| Tool-result **text** | inserted verbatim into `role:"tool"` content / `tool_result` blocks / `function_call_output` | `llm.rs:532-534`, `llm.rs:429-431`, `llm.rs:621-625` |
| Tool-result **images** | base64 re-embedded as a `data:` URI built by `format!` with the tool-supplied `mime_type` | `llm.rs:572-575`, `llm.rs:641-644` |
| Tool names / descriptions / schemas | passed through unchanged into `tools[]` | `llm.rs:436-444`, `llm.rs:538-546`, `llm.rs:653-661` |
| Model-emitted tool name | validated only for non-emptiness | `llm.rs:964-966` |

The `mime_type` pass-through is the one input-validation gap I can point at concretely: `ToolResultContent::Image.mime_type` (`types.rs:136`) originates from a tool result and is interpolated straight into `data:{mime_type};base64,{data}` at `llm.rs:573` and `llm.rs:643`, and into the Anthropic `source.media_type` field at `llm.rs:468`. There is no allowlist of media types and no rejection of `;`, `,`, or whitespace, so a malicious MCP server can shape the data-URI prefix. The blast radius is limited to confusing the provider's own parser, not the agent, but it is unvalidated attacker-controlled input crossing into a structured field. grep for `image/png`, `image/jpeg`, or any media-type allowlist in `llm.rs` returned only test fixtures.

There is no prompt-injection mitigation in this group by design — no delimiter insertion, no instruction stripping, no separation marker around tool output. That is consistent with the README's framing that "Untrusted input — model output, tool results, prompts — is bounded" (`README.md:216`) meaning *size*-bounded, not *semantics*-sanitised.

#### Response-size and resource exhaustion
This is handled well and in two layers:

| Control | Value | Site |
|---|---|---|
| `Content-Length` precheck | reject if `> 16 MiB` before reading any body | `llm.rs:1121-1128` |
| Streaming accumulation cap | reject once buffered + incoming chunk `> 16 MiB` | `llm.rs:1129-1141` |
| Error-body cap | stop at 4 KiB, including a partial-chunk break | `llm.rs:983-998` |
| Attempt cap | 3 | `llm.rs:1000`, loop at `llm.rs:1055` |
| Backoff ceiling | 8 s | `llm.rs:1002` |
| Connect timeout | 10 s | `llm.rs:55` |
| Read (inactivity) timeout | `cfg.llm_timeout` | `llm.rs:56` |

The precheck is defence-in-depth: a lying `Content-Length` is still caught by the streaming cap (`llm.rs:1134-1138`). `read_error_body` correctly truncates mid-chunk rather than over-reading (`llm.rs:989-991`).

Two gaps:
1. **No total wall-clock timeout.** `Llm::new` sets only `connect_timeout` and `read_timeout` (`llm.rs:54-58`); `grep -n '\.timeout(' llm.rs` finds matches only inside the test module (`llm.rs:2226`, `llm.rs:2290`, `llm.rs:2341`, `llm.rs:2692`). A server that emits one byte every 239 s holds the call open indefinitely. `STALL_NOTICE_THRESHOLD` (`llm.rs:24`) only logs (`llm.rs:1039-1045`); it does not abort. Note that because the tests use `.timeout(5s)`, the production timeout configuration is never exercised by any test.
2. **No outbound body-size cap.** Nothing in `llm.rs` bounds the size of the request it constructs. `serde_json::to_vec` at `llm.rs:1053` will serialize a history of any size, and the bytes are then **cloned once per attempt** (`llm.rs:1065`) — so a 100 MiB body is materialised up to four times concurrently across the loop. The relevant bounds (`max_history_bytes`, `MAX_PROMPT_BYTES`, `MAX_TOOL_RESULT_BYTES`) are all defined in `config.rs` (`config.rs:700`, `config.rs:638`, `config.rs:643`) but enforced in `agent.rs`/`handoff.rs`, not here. grep for `max_history_bytes` in `llm.rs` returned only the test-fixture initializer at `llm.rs:1237`.

There is also no cap on the number of tool calls parsed out of a single response inside `llm.rs` — `parse_openai` (`llm.rs:934-948`) and `parse_responses` (`llm.rs:738-750`) will build an unbounded `Vec<ToolCall>`. `MAX_TOOL_CALLS_PER_TURN` = 64 is defined at `config.rs:650` and enforced downstream at `agent.rs:242-247`, after the whole vector already exists in memory.

#### Token-count caps
There is no token counting or pre-send token cap in this group. `max_context_tokens` (`config.rs:715`) is validated to exceed `max_output_tokens` (`config.rs:879-884`) and is documented as the handoff gate's budget, but `llm.rs` never reads it — grep for `max_context_tokens` in `llm.rs` returned only the test-fixture initializer at `llm.rs:1239`. Token totals are only ever read *from* the provider's response (`llm.rs:797-798`, `llm.rs:906-907`, `llm.rs:956-957`), never estimated before sending. So a request that will exceed the context window is sent and 400s, and the handoff fires on the *next* turn — which is exactly what the doc comment at `config.rs:710-714` describes ("the handoff fires when the previous request's … input tokens cross the handoff threshold … before the next request can exceed the window and 400").

#### Startup validation as a security control
`Config::validate` (`config.rs:870-951`) is the only fail-fast gate, and it is bypassable two ways:
- `Config::for_discovery` (`config.rs:838-868`) never calls it, and produces a struct that would fail it (`max_output_tokens: 1` at `config.rs:846`, `mcp_max_restart_attempts: 0` at `config.rs:851`).
- Every `Config` field is `pub` (`config.rs:688-733`) while `validate` is private (`config.rs:870`), so any in-crate or external caller can construct or mutate a `Config` past its invariants. `desktop`/`buzz-acp` set these values through env vars, so this is not a live bypass today, but the type offers no protection.

#### Test coverage for this aspect
Covered:
- Auth-failure recovery, all four combinations of 401/403 × recoverable/persistent (`llm.rs:2704`, `llm.rs:2740`, `llm.rs:2768`, `llm.rs:2796`), each asserting the exact refresh count.
- Static-source refresh is a safe no-op (`llm.rs:2819`).
- Lookalike host rejection (`config.rs:1277`) and malformed-URL rejection (`config.rs:1278`).
- Retry exhaustion bounding (`llm.rs:2306` asserts exactly `MAX_RETRIES` server-side accepts).

Not covered — each verified by grepping the test modules and finding zero matches:
- Neither response-size cap: no test for `MAX_LLM_RESPONSE_BYTES`, `response too large`, or `response exceeded`.
- The error-body 4 KiB truncation: no test for `read_error_body` or `MAX_LLM_ERROR_BODY_BYTES`.
- Any scheme/`http://` downgrade scenario for a *credential-bearing* request (the `is_openai_host` matrix tests routing, not credential exposure).
- Any assertion that `api_key` is absent from a serialized body or a log line.
- The `mime_type` pass-through with a hostile value.
- `Llm::new`'s timeout configuration (the tests build `Llm` by struct literal at `llm.rs:2689-2698` with a different client).
