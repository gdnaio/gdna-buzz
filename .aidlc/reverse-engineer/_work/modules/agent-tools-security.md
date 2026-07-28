## Module: buzz-agent — MCP registry, OAuth, hints & catalog (`crates/buzz-agent/src`)
### Aspect: Security

The crate's stated trust boundary is "the operator who launched the agent"; harness, MCP binaries, and API keys are trusted, while model output, tool results, and prompts are treated as untrusted-but-bounded (`crates/buzz-agent/README.md`, "Security Model"). The findings below are scoped to how well this group holds that line.

#### Child-process environment: allowlist and what survives `env_clear()`

`spawn_one` calls `cmd.env_clear()` (`mcp.rs:714`) and then re-adds only allowlisted keys (`mcp.rs:715-719`), so an unlisted variable — including `ANTHROPIC_API_KEY`, `OPENAI_COMPAT_API_KEY`, and `DATABRICKS_TOKEN` — cannot reach an MCP child. Verified: `grep -n 'ANTHROPIC_API_KEY\|OPENAI_COMPAT_API_KEY\|DATABRICKS_TOKEN' mcp.rs` returns zero matches.

What *does* survive (`PASSTHROUGH_ENV`, `mcp.rs:39-63`):

| Key | Secret? | Why it is there (source comment) |
|---|---|---|
| `PATH`, `HOME`, `TERM`, `LANG`, `LC_ALL`, `TMPDIR`, `XDG_CONFIG_HOME` | no | "Core" (`mcp.rs:40`) |
| `SSH_AUTH_SOCK` | **capability handle** | git over SSH (`mcp.rs:49`) — grants the child use of every key in the agent's ssh-agent |
| `SSH_AGENT_PID` | no | same group |
| `GIT_ASKPASS`, `GIT_SSH_COMMAND` | **code-execution vectors** | "operator-configured helpers and transport overrides" (`mcp.rs:53`) — both name programs git will execute |
| `GIT_CONFIG_GLOBAL` | config path | same group |
| `NOSTR_PRIVATE_KEY` | **yes, private key** | comment says dev-mcp writes it to a keyfile and removes it from its own env "(children never see it)" (`mcp.rs:54-56`) |
| `BUZZ_PRIVATE_KEY` | **yes, private key** | "kept for the buzz CLI" (`mcp.rs:56-57`) |
| `BUZZ_RELAY_URL` | no | same |
| `BUZZ_AUTH_TAG` | signed attestation | "non-secret signed ownership attestation … MCP subprocesses are trusted like the agent runtime" (`mcp.rs:57-62`) |

So every MCP child the client asks for — command and args are taken verbatim from the `session/new` payload (`mcp.rs:711-712`), with no allowlist on the binary — receives the agent's Nostr identity key, its Buzz owner attestation, and its ssh-agent socket. That is a deliberate decision, argued in the comment at `mcp.rs:57-62`, but it is a much wider grant than the crate README advertises: the README's Security Model table lists the whitelist as "`PATH`, `HOME`, `TERM`, `LANG`, `LC_ALL`, `TMPDIR`" and reassures the reader that "Your `ANTHROPIC_API_KEY` does not leak into MCP children" — true, while omitting that two private keys do. This is the single largest documentation/security drift in the group.

Ordering detail: client-supplied `env` entries are applied *after* the allowlist (`mcp.rs:726-728`), so a client can override `PATH`, `GIT_SSH_COMMAND`, or `BUZZ_PRIVATE_KEY` for the child. Since the client is the ACP harness, that is inside the trust boundary, but it means the last word belongs to the payload, not to the allowlist.

Windows: `env_clear()` is followed by the same allowlist plus `TMP`, `TEMP`, `USERPROFILE`, `APPDATA` (`mcp.rs:70-71`) and the shell-resolution keys shared with Doctor (`mcp.rs:74-81`, `lib.rs:19-30`). The Windows list is asserted by `windows_passthrough_includes_shell_resolution_vars` (`mcp.rs:1017`) — but it is `#[cfg(windows)]`, so it does not run on the CI targets this repo builds. The only allowlist assertion that runs everywhere is a single containment check for `BUZZ_AUTH_TAG` (`mcp.rs:1010-1014`). No test anywhere verifies that a spawned child's *actual* environment matches the allowlist: `grep -rn 'PASSTHROUGH\|env_clear' crates/buzz-agent/tests` returns zero matches.

#### Secret exposure paths

| Secret | Can it reach a log line? | An error message? | Serialized output? | A child env? |
|---|---|---|---|---|
| provider API keys (`ANTHROPIC_API_KEY`, `OPENAI_COMPAT_API_KEY`) | not from this group | no | no | no (`mcp.rs:39-63`) |
| `DATABRICKS_TOKEN` (static bearer) | not directly; but see the caller's defensive redaction below | only if the server echoes it in a response body (`catalog.rs:111-114`, `:226-229`) | no | no |
| OAuth access token | no — never logged; `tracing::warn!(error = %e, …)` at `auth.rs:278`/`:401` carries only the error | not constructed into any message | **yes, to disk in plaintext** (`auth.rs:192-200`) | no |
| OAuth refresh token | no | possible via echoed token-endpoint body (`auth.rs:223-224`, `auth.rs:622-623`) | **yes, to disk in plaintext** | no |
| PKCE `code_verifier` | no (`grep -n 'verifier' auth.rs` shows production uses only at `auth.rs:512-515`, `auth.rs:537`, `auth.rs:612` — generation and the form field) | no | no | no |
| `state` parameter | **yes** — printed inside the full authorize URL to stderr (`auth.rs:589-598`) | echoed into the callback HTML on mismatch (`auth.rs:552`, `auth.rs:565`) | no | no |
| `BUZZ_PRIVATE_KEY` / `NOSTR_PRIVATE_KEY` | no | no | no | **yes, by design** (`mcp.rs:55-57`) |

Two concrete gaps:

1. **Unbounded, unredacted server bodies in errors.** `auth.rs:223-224` and `auth.rs:622-623` do `resp.text()` and interpolate the whole body into `AgentError::Llm`; `catalog.rs:111-114` and `catalog.rs:226-229` do the same. There is no length cap and no redaction, in a crate that already has the pattern — `llm.rs:23` defines `MAX_LLM_ERROR_BODY_BYTES = 4 KiB` and enforces it while streaming (`llm.rs:985-988`). Because `session_new` only logs discovery failures (`lib.rs:317-321`), the blast radius for catalog errors is the stderr log; for OAuth errors raised during a prompt the message travels to the ACP client via `wire::err`.
2. **Redaction is a caller responsibility, not this group's.** The desktop wraps the same `discover_databricks_models` call and scrubs `DATABRICKS_TOKEN` out of any error string before surfacing it (`desktop/src-tauri/src/commands/agent_models.rs:726-737`). Nothing equivalent exists inside `catalog.rs` or `auth.rs`; the second consumer (`session_new`) does not redact.

Also note `CachedToken` derives `Debug` (`auth.rs:110`), so any future `{:?}` of it prints both tokens; there is no redacting `Debug` impl (`grep -n 'impl.*Debug' auth.rs` → zero matches).

#### OAuth token cache file

| Property | Value | Site |
|---|---|---|
| path | `$HOME/.config/buzz-agent/oauth/<namespace>/<sha256>.json` | `auth.rs:455-467` |
| directory creation | `create_dir_all`, default mode (umask) | `auth.rs:146-149` |
| file creation | `fs::write` to `<hash>.json.tmp` then `fs::rename` | `auth.rs:195-200` |
| permissions | **not set** — inherits umask, typically `0644` | `grep -rn 'set_permissions\|PermissionsExt\|0o600' crates/buzz-agent/src` → zero matches |
| content | pretty-printed JSON with `access_token` and `refresh_token` in cleartext | `auth.rs:192`, `auth.rs:110-118` |
| removal | never; no expiry-based cleanup or `revoke` call | `grep -n 'remove_file\|revoke' auth.rs` → zero matches |

The repo already knows how to do this: `crates/buzz-dev-mcp/src/shim.rs:141` writes its keyfile with `.mode(0o600)`. Long-lived Databricks refresh tokens sitting world-readable under `~/.config` is the highest-value finding in this file after the env allowlist.

Second issue with the same code: the temp filename is derived from the cache path alone (`auth.rs:195`), so two processes refreshing concurrently share one `<hash>.json.tmp`. The comment claims "Atomic rename so a concurrent reader never sees a partial write" (`auth.rs:194`) — correct for readers, but two concurrent *writers* can interleave `write`/`rename` on the shared temp path. The single-flight `Mutex` (`auth.rs:141`) only serialises within one process, and the design explicitly anticipates sibling processes (`auth.rs:256-257`, `auth.rs:283-284`).

#### PKCE correctness

| Requirement | Implementation | Verdict |
|---|---|---|
| verifier entropy | 48 bytes from `getrandom::fill` → base64url, 64 chars (`auth.rs:509-516`) | meets RFC 7636 §4.1 (43-128 chars); RNG failure is propagated as an error, not ignored (`auth.rs:511`) |
| challenge method | `S256`, `base64url(sha256(verifier))`, advertised as `code_challenge_method=S256` (`auth.rs:512-515`, `auth.rs:589-596`) | correct; asserted by `pkce_pair_produces_valid_challenge` (`auth.rs:637`) |
| `state` generation | 16 bytes → base64url, 22 chars (`auth.rs:518-522`) | adequate entropy; RNG failure propagated |
| `state` comparison | `st == &expected` inside the callback (`auth.rs:551`) | present and correct; not constant-time, which is immaterial for a single-use CSRF token |
| mismatch handling | `Err("state mismatch")` is sent to the waiter, aborting the flow (`auth.rs:552`, `auth.rs:557-559`) | correct — a mismatched callback cannot yield a code |
| redirect-URI binding | the same `redirect_uri` string is used in the authorize URL and the token exchange (`auth.rs:578`, `auth.rs:592`, `auth.rs:612`) | correct |
| loopback binding | `127.0.0.1:0` only (`auth.rs:571`) | correct — not `0.0.0.0` |
| listener lifetime | aborted by `AbortOnDrop` on every exit path (`auth.rs:584-586`, `auth.rs:426-432`), bounded by a 60 s timeout (`auth.rs:39`, `auth.rs:601`) | correct |
| single-use callback | the oneshot sender is `take()`n, so only the first request can deliver a code (`auth.rs:558-560`) | correct |

Three residual weaknesses:

1. **Reflected, unescaped HTML on the loopback page.** The failure branch renders `Html(format!("<h2>Buzz auth failed</h2><pre>{e}</pre>"))` (`auth.rs:565`), and `e` can be the raw `error` query parameter supplied by whoever hits the endpoint (`auth.rs:553-556`). For the ~60 s the listener is up, any page the user visits can navigate to `http://localhost:<port>/?error=<markup>` and get it injected into a same-origin document. Impact is limited (ephemeral port, no cookies or secrets on that origin, no persistence) but the reflection is real and one `html_escape` call away from being closed.
2. **`localhost` vs `127.0.0.1`.** The listener binds the IPv4 literal (`auth.rs:571`) while the redirect URI names `localhost` (`auth.rs:578`). On an IPv6-first host the browser may hit `[::1]:port` and time out; RFC 8252 §7.3 recommends the literal for this reason.
3. **Anyone local can talk to the listener.** There is no loopback-peer or `Origin` check beyond `state`. `state` is what protects the flow, and it is printed to stderr as part of the authorize URL (`auth.rs:598`) — a local process able to read the agent's stderr could forge a callback. Given the trust model (same operator), this is a note rather than a defect.

#### TLS / scheme validation

None. `grep -n 'https\|scheme' auth.rs catalog.rs` matches only test fixtures (`auth.rs:689`, `725`, `766`, `818`). Consequences:

- `discovery_url` is built by string-concatenating `DATABRICKS_HOST` (`llm.rs:1183-1186`, `lib.rs:137-140`), so `DATABRICKS_HOST=http://…` produces a plaintext discovery fetch (`auth.rs:161-165`).
- `authorization_endpoint` and `token_endpoint` are taken verbatim from that document (`auth.rs:172-188`) with no check that they are `https`, nor that they share the discovery host. A hostile or MITM'd discovery document therefore redirects the token exchange — which carries the authorization `code` and the `code_verifier` (`auth.rs:609-616`), and on the refresh path the refresh token itself (`auth.rs:206-213`) — to an arbitrary URL.
- The catalog endpoints inherit the same unvalidated `cfg.base_url` (`catalog.rs:81`, `catalog.rs:101`, `catalog.rs:202`) and send `Authorization: Bearer` to it (`catalog.rs:104`, `catalog.rs:217`).
- Transport is `reqwest` with `rustls` (root `Cargo.toml:93`), and no `danger_accept_invalid_certs` anywhere (`grep -rn 'danger' crates/buzz-agent` → zero matches), so certificate validation itself is intact — the gap is purely that `http://` is never refused.

For contrast, the crate does harden a similar decision elsewhere: `is_openai_host` (`config.rs:1029-1044`) is explicitly lookalike-safe and tested (`is_openai_host_matrix`, `config.rs:1267`).

#### Path traversal and symlinks

`load_skill`'s supporting-file path has two layers (`builtin.rs:118-230`): the request must match a **pre-enumerated** absolute path by its skill-dir-relative rendering (`builtin.rs:143-149`), and the resolved file must canonicalise inside the canonicalised skill directory (`builtin.rs:178-196`). The first layer is what actually stops `../secret` — the test's own comment says so (`builtin.rs:486-490`, test at `builtin.rs:461`). Canonicalisation failure of the skill dir is a hard error rather than a bypass (`builtin.rs:178-186`, rationale `builtin.rs:176-177`).

Compared with `buzz-persona::safe_resolve` (`crates/buzz-persona/src/pack.rs:323-364`), this group is missing three of its four steps:

| `safe_resolve` step | `pack.rs` | `builtin.rs` equivalent |
|---|---|---|
| reject absolute paths (`/`, Windows drive) | `:325-331` | none — absolute input simply fails to match the enumeration |
| reject `..` components eagerly | `:333-339` | none — relies on enumeration mismatch |
| canonicalise, then require `starts_with(root)` | `:341-361` | present (`builtin.rs:178-196`) |
| stat-then-read size bound (`read_bounded_file`, `pack.rs:374-386`) | present | **absent** — full `read_to_string`, cap applied after (`builtin.rs:197-209`) |

Where containment genuinely does not hold:

- **The plain skill form has no containment check at all.** `call_load_skill` reads `entry.path` directly (`builtin.rs:68-71`) with no canonicalisation. It is safe only because that path came from discovery — the property is "trusted producer", not "verified path".
- **Discovery follows symlinks on purpose.** Both walkers use `std::fs::metadata` instead of `DirEntry::file_type` specifically so symlinks resolve (`hints.rs:112-119`, `hints.rs:178-182`), and an integration test locks that in (`symlinked_skill_dir_is_discovered`, `tests/hints_integration.rs:469`). A symlink at `.agents/skills/x → /any/dir` therefore makes every file under that directory a "supporting file" of skill `x`, and the containment check passes because the skill dir canonicalises to the symlink target (`builtin.rs:178`, `builtin.rs:196`). Anyone who can create a symlink in a skills directory can make arbitrary readable files loadable through the tool.
- **Asymmetry the model will notice:** a symlinked *file* inside a real skill directory is enumerated and advertised in the supporting-files listing (`builtin.rs:88-98`) but then refused by the canonicalisation check (`builtin.rs:216-219`).
- `hints.rs` performs no containment at all — it reads `AGENTS.md` from `$HOME` and from every directory between the git root and `cwd` (`hints.rs:41-66`), following symlinks implicitly via `read_to_string`. Since `cwd` comes from the client and is only checked for absoluteness (`lib.rs:334-341`), any `AGENTS.md` on the filesystem can be loaded into the system prompt by pointing a session at its directory.

#### Prompt-injection surface

Three inbound channels, at three different trust levels:

| Channel | Lands as | Sanitisation |
|---|---|---|
| `AGENTS.md` contents | **system prompt** (`hints.rs:233-237`, `lib.rs:361-388`) | none — raw file text, only byte-capped at 128 KiB |
| skill `name` + `description` from YAML frontmatter | **system prompt** bullet list (`hints.rs:239-247`) | `trim()` only (`hints.rs:96-105`); interior newlines and markdown survive; no length cap |
| `load_skill` output (body + supporting files) | tool result (`builtin.rs:108-115`) | frontmatter stripped, 32 KiB head cut, no structural escaping |
| MCP tool results | tool result (`mcp.rs:913-990`) | budgeted and marked, no escaping |
| hook output | `_Stop`: tool result, JSON-encoded (`agent.rs:641-649`); `_PostCompact`: plain text inside a synthetic user message (`handoff.rs:87-92`, `handoff.rs:247-253`) | inconsistent |

The skill-metadata path is the sharpest: a `SKILL.md` frontmatter `description` is unbounded, unescaped, and injected at *system-prompt* trust — strictly above the tool-result trust level that `docs/MCP_DRIVEN_HOOKS.md` insists on for hook output ("Hook responses are injected as tool-result messages (lower trust than system)", "Hook output is JSON-encoded for prompt-injection safety"). The repo therefore applies a stricter rule to a server it spawns than to a file it finds on disk. And that hooks doc claim is itself only half-true: `_Stop` output is JSON-encoded (`agent.rs:641-649`), while `_PostCompact` output is concatenated as plain text into a user message labelled "[Post-compact hook output — untrusted]" (`handoff.rs:88-91`).

#### Resource exhaustion

| Vector | Bounded? | Detail |
|---|---|---|
| MCP servers / tools per session | yes | 16 / 128 (`mcp.rs:177-182`, `mcp.rs:232-236`) |
| per-result tool output | yes | 8 MiB total, 50 KiB text default (`agent.rs:386-389`) |
| tool description / schema | yes | 1 KiB clamp, 4 KiB → replaced with `{}` (`mcp.rs:253-257`, `mcp.rs:833-843`) |
| per-hook result | yes (16 KiB) | but the *number* of hook results is unbounded — 16 servers × 16 KiB per `_Stop` (`mcp.rs:355-358`) |
| restart attempts | **only per consecutive spawn failure** | a server that starts fine and wedges every call cycles kill→restart forever (`mcp.rs:432`, `mcp.rs:672-677`; the comment at `mcp.rs:414-419` claims otherwise) |
| `AGENTS.md` read | **no** | `read_to_string` reads the whole file before the 128 KiB cap is applied (`hints.rs:65-66`, `hints.rs:71-81`), synchronously on the `session/new` task |
| skill count / description length | **no** | no cap on discovered skills or on frontmatter description length (`hints.rs:239-243`); overflow is caught only by the 512 KiB system-prompt check, which *fails the session* (`lib.rs:375-388`) |
| `load_skill` reads | **no** | full `read_to_string` then truncate (`builtin.rs:68-77`, `builtin.rs:197-209`) — a 1 GiB supporting file is fully read into memory |
| supporting-file enumeration | **no** | unbounded recursive walk at session creation; cycles are broken but breadth is not (`hints.rs:158-202`) |
| OAuth / catalog HTTP | **no timeouts** | `Client::new()` with no connect/read timeout (`auth.rs:153`, `catalog.rs:80`), versus `llm.rs:53-57` which sets both. A hung Databricks host stalls `session/new` indefinitely, because discovery is awaited inline (`lib.rs:440-458`) |
| catalog pagination | yes | 20 pages × 100 (`catalog.rs:199-200`), silently truncating with no log |
| MCP child memory/CPU | no | no rlimits, cgroups, or nice values — `grep -rn 'rlimit\|setrlimit' crates/buzz-agent/src` → zero matches |

#### Test coverage — Security

Covered: hook tools hidden from the model and rejected on direct call (`tests/regressions.rs:1035 hook_tools_hidden_from_llm`); hook timeout fail-open (`:1514`); cancellation reaching the child as `notifications/cancelled` (`:1573`, `:1710`); tool-metadata caps (`:354`, `:681`); server/tool count caps (`:420`, `:606`); `load_skill` traversal rejection (`builtin.rs:461`) and both size caps (`builtin.rs:504`, `:540`); symlink-cycle safety in enumeration (`hints.rs:686`); PKCE challenge derivation (`auth.rs:637`); refresh coalescing and terminal failure (`tests/databricks_oauth.rs:212`, `:261`, `:305`).

Not covered — each of these is a security property with no test:
- the child env allowlist as *applied to a real child* (`grep -rn 'PASSTHROUGH\|env_clear' crates/buzz-agent/tests` → zero matches; the only unit test checks the constant's contents, `mcp.rs:1010`);
- process-group kill of grandchildren, despite `FAKE_MCP_SPAWN_GRANDCHILD` existing in the fake server (`tests/bin/fake_mcp.rs:228`) and no test referencing it;
- token-cache file permissions (no test asserts a mode);
- `state` mismatch rejection, the callback HTML, and the 60 s browser timeout (`browser_pkce_flow` has no test at all);
- absence of `https` enforcement (nothing asserts a rejection, because there is no rejection to assert);
- restart exhaustion / unbounded restart cycling;
- prompt-injection containment for skill metadata (the only related assertion is that skill *bodies* are not inlined, `hints.rs:479-495`).
