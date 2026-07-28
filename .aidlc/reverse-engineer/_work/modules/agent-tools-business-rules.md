## Module: buzz-agent — MCP registry, OAuth, hints & catalog (`crates/buzz-agent/src`)
### Aspect: Business Rules

#### Server registration and spawn sequencing (`mcp.rs:172-264`)

Rules applied in order by `spawn_all`, all of which abort `session/new` on violation (the error is returned to the client at `lib.rs:390-393`):

| # | Rule | Site | Error text |
|---|---|---|---|
| 1 | at most 16 servers per session | `mcp.rs:177-182` | `too many MCP servers: {n} > 16` |
| 2 | server name must be non-empty, ≤128 bytes, `[A-Za-z0-9_-]` only, and must not contain `__` | `mcp.rs:197-199`, `mcp.rs:859-864` | `invalid server name: {name}` |
| 3 | server names unique within the session | `mcp.rs:200-205` | `duplicate server name: {name}` |
| 4 | each server is spawned **and** initialised **and** `tools/list`-ed before the next one starts (sequential, `await` inside the loop) | `mcp.rs:217` | first failure aborts the whole session |
| 5 | at most 128 tools across all servers | `mcp.rs:232-236` | `too many tools (>128)` |
| 6 | bare tool name must pass `valid_name` and must not contain `__` | `mcp.rs:238-240` | `invalid tool name: {bare}` |
| 7 | `server__tool` must be ≤64 bytes | `mcp.rs:242-248` | `qualified tool name too long` |
| 8 | qualified names unique | `mcp.rs:249-251` | `duplicate tool: {qname}` |

Two consequences of the ordering: (a) spawning is serial, so 16 slow servers cost 16 × `mcp_init_timeout` in the worst case with no concurrency; (b) rules 5-8 fire *after* the child is already running, and the early `return Err` drops the partially built registry, relying on `Drop for Server` (`mcp.rs:116-122`) to kill the already-spawned process groups.

Per-server spawn (`spawn_one`, `mcp.rs:708-786`) sequence: build `Command` → `env_clear()` → re-add allowlist → add client-supplied env → `current_dir(cwd)` → inherit stderr → `process_group(0)` on unix → suppress console window on Windows → `TokioChildProcess::new` → record pgid → arm a `PgidGuard` → `().serve(transport)` under `init_timeout` → `list_all_tools()` under the same `init_timeout` → disarm the guard. The guard (`mcp.rs:741-756`, disarmed at `mcp.rs:781`) is what makes an init timeout kill the child rather than leak it.

#### Tool visibility and routing

`tools()` (`mcp.rs:286-313`) advertises a `ToolDef` only when all of: the qname resolves in `by_qname`; the bare name does **not** start with `_` (`mcp.rs:294-296`); and either the server is `Healthy` and still lists that bare name, or the server is `Dead` with `attempts < max_attempts` and previously listed it (`mcp.rs:298-306`). So a dead-but-retryable server keeps advertising its tools; an exhausted one disappears mid-session.

Routing is a prefix-free exact lookup, not a scan: the qname is the `HashMap` key and the entry holds `server_idx` + `bare` (`mcp.rs:154-157`, `mcp.rs:494-498`). Collisions cannot occur at call time because they are rejected at registration (rule 8). The `__` ban on both halves (rules 2 and 6) is what makes the split unambiguous — documented in the crate README ("Bare tool names containing `__` are rejected at registration").

Two guards run before a call reaches the child:
- tool-set drift: if the live `Healthy.tools` no longer contains the bare name, the call fails with "no longer available; the MCP server restarted with a different tool set" (`mcp.rs:500-506` for the fast path, `mcp.rs:526-532` after a restart);
- argument shape: `arguments` must be a JSON object or `null`; anything else is rejected locally (`mcp.rs:566-575`).

Hook tools are excluded from the LLM's view (`mcp.rs:294-296`) and direct invocation is rejected by the caller via `has() || is_hook()` (`agent.rs:330-335`), producing a synthetic `unknown tool: …` result. This matches `docs/MCP_DRIVEN_HOOKS.md` ("Hooks are filtered from the tool list sent to the LLM", "Hooks are rejected if the LLM attempts to call them directly").

#### Restart attempts and backoff

| Parameter | Source env | Default | Consumption |
|---|---|---|---|
| `max_attempts` | `BUZZ_AGENT_MCP_RESTART_MAX_ATTEMPTS` | 3 (`config.rs:809`) | `.max(1)` at `mcp.rs:188` |
| `backoff_base` | `BUZZ_AGENT_MCP_RESTART_BASE_MS` | 500 ms (`config.rs:810`) | `.max(1)` at `mcp.rs:189` |
| `backoff_max` | `BUZZ_AGENT_MCP_RESTART_MAX_MS` | 30 000 ms (`config.rs:811`) | `.max(1)` at `mcp.rs:190` |
| `init_timeout` | `BUZZ_AGENT_MCP_INIT_TIMEOUT_SECS` | 30 s (`config.rs:805-808`) | `mcp.rs:191` |

Restart is lazy and demand-driven: only `call` triggers it, via `maybe_restart` (`mcp.rs:646-706`). The check is performed twice — once unlocked, once under `restart_lock` — so concurrent callers do not double-spawn (`mcp.rs:647-660`). Backoff is `base << (attempt-1)` capped at 20 shifts, then `min(backoff_max)`, then ±20% jitter (`mcp.rs:813-827`); the jitter is applied *after* the cap, so the realised delay can reach `1.2 × backoff_max`.

Two rules stated only in comments, one of which the code does not implement:

- `mcp.rs:414-419` claims `kill_server` "Counts as one attempt toward the restart budget so that a pathological server (starts fine, deadlocks on every call) eventually exhausts." It does not: `kill_server` hardcodes `attempts: 1` (`mcp.rs:432`) and a successful restart replaces the state with `Healthy`, which carries no counter (`mcp.rs:672-677`). A server that spawns cleanly and wedges on every call therefore cycles kill → restart indefinitely; only *consecutive spawn failures* are budgeted (`mcp.rs:685-699`).
- `mcp.rs:687-690` sets `next_retry = now + 86_400s` when the budget is exhausted, but `check_restart_state` returns the terminal "exhausted" error before ever reading `next_retry` in that state (`mcp.rs:135-140`), so the 24-hour value is unreachable.

#### Hook dispatch rules (`call_hooks`, `mcp.rs:315-419`)

1. Short-circuit when `HookServers::None` (`mcp.rs:321-323`) — hooks are off unless `MCP_HOOK_SERVERS` is set (`config.rs:824`, parser at `config.rs:1083-1105`).
2. Targets are collected by walking `self.servers` in registration order and keeping servers that `allowed.allows(name)` and that expose `"{server}__{hook}"` (`mcp.rs:326-337`).
3. All targets are called concurrently in a `JoinSet` (`mcp.rs:341-364`), each bounded by `cfg.hook_timeout` and given a **dummy** cancel channel so session cancellation cannot interrupt hook evaluation (`mcp.rs:345-347`, rationale at `mcp.rs:342-344`).
4. Each hook result is budgeted at 16 KiB for both total and text (`mcp.rs:355-358`).
5. Fail-open filtering: join errors, timeouts, call errors, `is_error` results, and whitespace-only text are all dropped (`mcp.rs:367-385`).
6. Timeout escalation: a per-server consecutive-timeout counter is incremented on timeout and reset on any success; the server's process group is killed on the **second** consecutive timeout (`mcp.rs:386-405`). This matches `docs/MCP_DRIVEN_HOOKS.md` ("Server killed only on second consecutive timeout").
7. Results are re-sorted by the server's registration index before returning, so output order is deterministic regardless of completion order (`mcp.rs:407-411`).

Not bounded: the number of surviving hook outputs. Up to 16 servers × 16 KiB ≈ 256 KiB of hook text can enter history from one `_Stop` evaluation; `grep -n 'MAX_HOOK_RESULT_BYTES' mcp.rs` shows the cap is per-result only (`mcp.rs:27,356,357`).

#### Tool-result truncation rules (`mcp.rs:913-993`)

- Text blocks accumulate into one buffer joined by `\n` (`mcp.rs:944-949`, `mcp.rs:953`).
- On flush, the text budget is `min(max_text_bytes - text_used, max_bytes - used)` and the buffer is **middle**-elided so head and tail both survive, with an inline `[... N of M bytes elided from tool result ...]` marker (`mcp.rs:926-942`, `mcp.rs:886-911`).
- Images are emitted whole when `used + data.len() + mime.len() <= max_bytes`, otherwise replaced by a text marker naming the mime type and base64 length (`mcp.rs:954-973`). Images do **not** consume the text budget — the rationale is at `mcp.rs:28-32` and `config.rs:640-643`.
- Audio is always elided to a marker (`mcp.rs:974-981`); `ResourceLink` becomes `[resource: <uri>]` (`mcp.rs:982-984`); embedded `Resource` becomes `[resource elided]` (`mcp.rs:985`). Marker-embedded strings are cut to 256 bytes (`mcp.rs:923`, `mcp.rs:25`).
- Degenerate budget: when `max - 128 == 0` the middle elision degrades to a plain head cut (`mcp.rs:891-895`).

Budgets at the two call sites: regular tool calls get `total = 8 MiB` (`MAX_TOOL_RESULT_BYTES`) and `text = cfg.max_tool_result_text_bytes` (50 KiB default) — `agent.rs:386-389`; hooks get 16 KiB/16 KiB (`mcp.rs:355-358`).

#### Cancellation rules (`mcp.rs:556-644`)

An early `*cancel.borrow()` check happens after the request is sent, because `watch::changed()` only fires on new writes (`mcp.rs:583-587`). The response is awaited in a `biased` `select!` where the cancel branch wins ties (`mcp.rs:590-600`). On either cancel path the in-flight request handle is handed to a detached task that sends `notifications/cancelled` best-effort (`mcp.rs:788-800`) and the call returns `AgentError::Cancelled`.

#### OAuth PKCE rules (`auth.rs`)

`bearer()` (`auth.rs:246-297`) resolves in strict order, all under one `Mutex` hold (single-flight):

1. in-memory token, if `!is_expired` (`auth.rs:250-254`);
2. re-read disk, in case a sibling process refreshed (`auth.rs:257-263`);
3. discover endpoints (unconditionally at this point, `auth.rs:268`), then run the refresh-token grant if a refresh token exists (`auth.rs:269-280`);
4. re-read disk again after a failed refresh (`auth.rs:283-289`);
5. full browser flow (`auth.rs:292-296`).

`try_bearer_no_browser` (`auth.rs:367-423`) mirrors steps 1-4 and then returns `LlmAuth("no cached Databricks token; run `buzz-agent auth databricks` first")` instead of step 5 (`auth.rs:416-418`). It deliberately *defers* endpoint discovery until a refresh token is known to exist (`auth.rs:392-403`, rationale at `auth.rs:389-392`) so an unreachable discovery URL yields the graceful `LlmAuth` rather than a hard `Llm`. That divergence from `bearer()` is guarded by a regression test (`auth.rs:812`).

`refresh_now(rejected)` (`auth.rs:317-358`) does not trust the clock: it coalesces on token *identity* — if the cached in-memory or on-disk access token differs from `rejected`, that token is returned without spending a grant (`auth.rs:324-338`); otherwise the refresh grant runs unconditionally, and failure is terminal `LlmAuth` so a headless harness never falls through to the browser (`auth.rs:340-357`).

Expiry rule: `is_expired` returns `false` when `expires_at` is `None` (`auth.rs:434-437`) — a token with no advertised expiry is used forever and only replaced via the 401 path. Otherwise `now + 60s >= expires_at` (`auth.rs:438-443`, leeway at `auth.rs:35`).

Token-response parsing (`token_from_response`, `auth.rs:478-507`): `access_token` must be present and non-empty or the whole response is rejected (`auth.rs:479-485`, rationale at `auth.rs:474-477`); `refresh_token` falls back to the one just used, so a server that omits it on refresh does not silently lose the ability to refresh again (`auth.rs:486-491`); `expires_at = now + expires_in` when `expires_in` is a u64 (`auth.rs:492-500`).

Browser flow (`browser_pkce_flow`, `auth.rs:527-630`) steps: derive verifier+challenge → derive `state` → bind `127.0.0.1:0` → build `redirect_uri = http://localhost:{port}` → spawn the axum callback under an abort guard → build the authorize URL with `code_challenge_method=S256` → print it to stderr → open the browser → wait ≤60 s for the callback → exchange `code` + `code_verifier` + the same `redirect_uri` for a token. The callback accepts a code only when the returned `state` equals the generated one (`auth.rs:549-556`).

#### Skill and hint discovery precedence (`hints.rs`)

Hint chain (`load_hint_files_impl`, `hints.rs:40-85`):
1. locate the git root by walking up for a `.git` entry — file or directory, so worktrees work (`hints.rs:27-38`);
2. the chain is every ancestor of `cwd` inside that root, reversed to root→cwd; without a git root the chain is just `cwd` (`hints.rs:41-53`);
3. `$HOME` is prepended as a global layer unless it is already in the chain (`hints.rs:55-60`) — the dedupe covers both "cwd is home" and "home is the git root" (tests at `hints.rs:531`, `hints.rs:544`);
4. each directory contributes its `AGENTS.md`, concatenated with a blank line, most-general first, until 128 KiB is consumed (`hints.rs:62-84`).

Skill precedence (`discover_skills_impl`, `hints.rs:204-217`): `<cwd>/.agents/skills`, then `<cwd>/.goose/skills`, then `<cwd>/.claude/skills`, then `$HOME/.agents/skills`. First writer of a given `name` wins; later duplicates are skipped by the `seen` set (`hints.rs:127-131`). Within a directory, sub-directories are sorted so ordering is stable (`hints.rs:121`). Only `<dir>/SKILL.md` with parseable frontmatter *and* a non-empty `name` registers (`hints.rs:123-126`, `hints.rs:86-108`); `description` is optional.

Supporting-file enumeration (`collect_supporting_files`, `hints.rs:158-202`): recursive, sorted, excludes `SKILL.md`, does not descend into a subdirectory that itself contains a `SKILL.md` (that is a separate skill, `hints.rs:186-190`), and guards against symlink cycles with a canonicalised visited-set (`hints.rs:167-171`). Both walkers use `std::fs::metadata`, which follows symlinks, deliberately (`hints.rs:114-119`, `hints.rs:180-182`).

Prompt assembly (`build_hints_section_impl`, `hints.rs:223-250`): returns an empty string when there are no hints and no skills; otherwise emits an `# Additional Instructions` header, an optional project-hints section carrying the raw file text, and an optional skills section listing `- name: description` per skill plus one sentence telling the model to call `load_skill`. Skill **bodies** are never inlined — asserted by `build_hints_section_combined` (`hints.rs:445`, assertions at `hints.rs:479-495`).

An off-by-two in the 128 KiB cap: the `"\n\n"` separator is pushed *before* `remaining` is recomputed (`hints.rs:68-71`), so the result can end at `MAX_HINTS_BYTES + 2` with a dangling separator when a preceding file fills the budget exactly. Harmless, but the stated invariant is not exact.

#### `load_skill` resolution rules (`builtin.rs`)

1. `name` is required; a missing/non-string argument returns an error result (`builtin.rs:42-48`).
2. The first `/` splits the request into `skill-name` + relative path (`builtin.rs:52-54`) — so a skill whose *name* contains a slash can never be loaded in the plain form.
3. Plain form: exact `name` match against the session's skill list, else an error listing every available name (`builtin.rs:57-66`); read on a blocking pool (`builtin.rs:68-77`); strip frontmatter (`builtin.rs:81-83`); append the supporting-files listing when non-empty (`builtin.rs:85-98`); cap at 32 KiB (`builtin.rs:100-106`).
4. Supporting-file form: backslashes in the request are normalised to `/` (`builtin.rs:124`); the path must match a **pre-enumerated** `supporting_files` entry by its skill-dir-relative rendering (`builtin.rs:143-149`) — this is the primary containment rule, and it is what actually rejects `../secret` (comment at `builtin.rs:486-490`, test `call_load_skill_traversal_guard_rejects_escape` `builtin.rs:461`); on no match, the error lists available relative paths, or says "has no supporting files" when the list is empty (`builtin.rs:151-172`).
5. Secondary containment: the skill dir is canonicalised (hard failure if that fails — "a degraded guard is worse than no guard", `builtin.rs:175-186`) and the resolved file must satisfy `starts_with(canonical_skill_dir)` (`builtin.rs:196`), else "refusing to load … resolves outside the skill directory" (`builtin.rs:216-219`).
6. Output is capped at 32 KiB by a head cut with no marker (`builtin.rs:206-209`).

Note the interaction between rules 4-5 and symlink-following discovery: a symlinked *file* inside a skill directory is enumerated (rule 4 matches) but its canonical target is outside, so rule 5 refuses it — the tool advertises a file it will not read. Conversely a symlinked skill *directory* canonicalises to its target, so every file under that target is legitimately inside the "skill directory" (integration test `symlinked_skill_dir_is_discovered`, `tests/hints_integration.rs:469`).

#### Catalog discovery rules (`catalog.rs`)

- Auth is obtained with `bearer_no_browser()` (`catalog.rs:77-78`), so discovery never blocks on user interaction.
- Dispatch by provider; anything other than the two Databricks providers is `InvalidParams` (`catalog.rs:82-90`). `host` is `cfg.base_url` with trailing slashes trimmed (`catalog.rs:81`).
- v1 (`api/2.0/serving-endpoints`): a missing `endpoints` array is an error (`catalog.rs:132-140`); each entry needs a string `name` (`catalog.rs:145`); `state.ready` must equal `"READY"` **when present** and `task` must be `llm/v1/chat` or `llm/v1/completions` **when present** — absent fields include the endpoint ("prefer including over silently dropping", `catalog.rs:126-130`, logic at `catalog.rs:147-163`).
- v2 (`api/ai-gateway/v2/endpoints`): `page_size=100`, at most 20 pages (`catalog.rs:199-200`); the loop stops on an absent, empty, or repeated `next_page_token` (`catalog.rs:239-242`, `catalog.rs:275-279`); no filtering at all — every named endpoint is a model (`catalog.rs:250-257`); if the accumulated list is empty it is replaced with `DATABRICKS_V2_KNOWN_MODELS` (`catalog.rs:245-254`).
- Discovery-failure fallback (`discovery_failure_fallback`, `catalog.rs:48-66`) is provider-split on purpose: `DatabricksV2` → the two known gateway models; legacy `Databricks` → only the configured model, because gateway IDs may not be servable by `/serving-endpoints/{model}/invocations` (`catalog.rs:40-44`). Three tests in `lib.rs` lock the split (`lib.rs:882`, `lib.rs:917`, `lib.rs:941`).
- Caller-side rule: a *successful* discovery is cached in a `OnceCell`; a failure deliberately leaves the cell empty so the next `session/new` retries (`lib.rs:312-329`, test `lib.rs:840`).

Contradiction: the doc comment says "Returns a non-empty `Vec<ModelEntry>` on success" (`catalog.rs:70`), but the v1 path returns whatever `parse_v1_endpoints` produced, including an empty vector (`catalog.rs:129`; `v1_parse_empty_endpoints_returns_empty_vec`, `catalog.rs:341`). Only v2 has an empty-list fallback. Downstream, `session_new` would advertise an empty `availableModels` (`lib.rs:447-457`), while the desktop caller treats empty as an error (`desktop/src-tauri/src/commands/agent_models.rs:740-742`).

#### Test coverage — Business Rules

Covered: server-count cap (`tests/regressions.rs:420 mcp_server_count_cap`), tool-count/description caps (`tests/regressions.rs:354 tool_metadata_caps_enforced`, `681 description_clamping_enforced`), init timeout (`tests/regressions.rs:307 mcp_init_timeout_kills_child`), hook visibility and the `_Stop` budget/fail-open rules (`tests/regressions.rs:787, 872, 927, 979, 1035, 1112, 1514`), `notifications/cancelled` on cancel (`tests/regressions.rs:1573, 1710`), hint chain precedence and skill dedupe (`hints.rs:321, 384, 499, 531, 544, 576`), all six `load_skill` resolution rules (`builtin.rs:277-573`), the catalog parsers and the fallback split (`catalog.rs:307-400`, `lib.rs:882-950`), and the whole OAuth ladder except step 5 (`tests/databricks_oauth.rs:105, 144, 179, 212, 261, 305`; `auth.rs:722, 763, 812`).

Not covered: restart attempts and backoff (no test constructs a `Dead` state — `grep -rn 'RESTART_MAX_ATTEMPTS\|RESTART_BASE_MS' crates/buzz-agent/tests` returns zero matches), the `kill_server`→restart cycle described above, hook determinism with more than one server, the 128 KiB hint cap, the 16 KiB hook-result cap, v2 pagination and the 20-page ceiling, the v2 empty-list fallback, and the browser flow (`grep -rn 'browser_pkce_flow' crates/buzz-agent` matches only `auth.rs:237, 293, 527` — no test file).

One test is written to accept either of two contradictory behaviours: `tool_metadata_caps_enforced` (`tests/regressions.rs:348-352`, branch at `:383-390`) passes whether `spawn_all` rejects a 200-tool server or truncates it. The production rule is "reject" (`mcp.rs:232-236`), so a regression from reject to truncate would not be caught.
