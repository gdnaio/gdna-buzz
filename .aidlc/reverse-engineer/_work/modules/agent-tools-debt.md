## Module: buzz-agent — MCP registry, OAuth, hints & catalog (`crates/buzz-agent/src`)
### Aspect: Debt

#### Correctness debt: comments that describe behaviour the code does not have

| Claim | Claim site | Code reality |
|---|---|---|
| `kill_server` "Counts as one attempt toward the restart budget so that a pathological server (starts fine, deadlocks on every call) eventually exhausts" | `mcp.rs:414-419` | `attempts` is hardcoded to `1` on every kill (`mcp.rs:432`) and a successful restart replaces the state with `Healthy`, which stores no counter (`mcp.rs:672-677`). Kill→restart cycles are unbounded; only consecutive *spawn* failures are budgeted (`mcp.rs:685-699`) |
| discovery "Returns a non-empty `Vec<ModelEntry>` on success" | `catalog.rs:70` | the v1 path returns `parse_v1_endpoints` verbatim, which can be empty (`catalog.rs:129`; `v1_parse_empty_endpoints_returns_empty_vec`, `catalog.rs:341`). Only v2 has an empty-list fallback (`catalog.rs:245-254`) |
| "Atomic rename so a concurrent reader never sees a partial write" | `auth.rs:194` | true for readers; the temp path is shared across processes (`auth.rs:195`), so two concurrent writers can interleave on `<hash>.json.tmp` |
| hints are capped at `MAX_HINTS_BYTES` | `hints.rs:6`, `hints.rs:71-81` | the `"\n\n"` separator is appended before `remaining` is recomputed (`hints.rs:68-71`), so the result can end at cap + 2 bytes with a dangling separator |
| dev-mcp removes `NOSTR_PRIVATE_KEY` from its env so "children never see it" | `mcp.rs:54-56` | accurate for dev-mcp's own children, but the comment sits in the allowlist that hands `NOSTR_PRIVATE_KEY` to *every* MCP child this crate spawns (`mcp.rs:63`, `mcp.rs:716`) — the reassurance reads broader than it is |

#### Unreachable / vestigial code

| Item | Site | Why it is dead |
|---|---|---|
| `next_retry = now + 86_400s` on exhaustion | `mcp.rs:687-690` | `check_restart_state` returns the terminal "exhausted" error before reading `next_retry` in that state (`mcp.rs:135-140`) |
| `MAX_NAME_LEN = 128` | `mcp.rs:20` | any name long enough to hit it fails the 64-byte qname check first (`mcp.rs:242-248`) |
| `PkceOAuthConfig::cache_dir_override` | `auth.rs:102-107` | production always passes `None` (`lib.rs:143`, `llm.rs:1194`); a test-only field on a public struct |
| `cache_namespace` as a knob | `auth.rs:101` | hardcoded to `"databricks"` at both construction sites |
| `discovery_failure_fallback`'s wildcard arm | `catalog.rs:61-64` | the `_` arm duplicates the `Provider::Databricks` arm and is only reachable if a non-Databricks provider is passed, which `discover_databricks_models` rejects first (`catalog.rs:86-90`) |
| `configure_no_window_compiles_and_applies_flag_on_windows` | `mcp.rs:1128-1140` | a `#[cfg(windows)]` test with **no assertion**; its own comment concedes "The real protection is the cfg-gated production path" |

Notably, none of this is masked with `#[allow(dead_code)]` — the group's only `#[allow]` is `clippy::too_many_arguments` on `do_call` (`mcp.rs:555`). `grep -n '#\[allow' mcp.rs auth.rs hints.rs builtin.rs catalog.rs` returns exactly one match. There are also zero `TODO`/`FIXME`/`XXX`/`HACK` markers in all five files (`grep -n 'TODO\|FIXME\|XXX\|HACK' mcp.rs auth.rs hints.rs builtin.rs catalog.rs` → exit 1, no matches), so known gaps are recorded in prose comments rather than in greppable markers — which is why several of the contradictions above went unnoticed.

#### Duplication

| Duplicated logic | Sites | Failure mode if they drift |
|---|---|---|
| Databricks OAuth `client_id`, scopes, discovery-URL template | `lib.rs:135-144` (hand-copied) vs `llm.rs:19-20`, `llm.rs:1183-1195` | the cache filename hashes all three (`auth.rs:446-454`), so `buzz-agent auth databricks` would write a token the runtime never reads — and both sides would report success |
| percent-encoding | `catalog.rs:182-193` (hand-rolled) vs the already-declared `urlencoding` crate used at `auth.rs:589-594` | two encoders with different reserved sets; the justifying comment (`catalog.rs:201-202`) cites a reqwest feature but the crate already ships a URL encoder |
| the `bearer()` cache-and-refresh ladder | `auth.rs:246-297` vs `try_bearer_no_browser` `auth.rs:367-423` | steps 1, 2, 4 are copied verbatim; a fix to the disk-reread logic must be applied twice. The documented divergence is only in step 3's lazy discovery and step 5's absence (`auth.rs:389-392`) |
| skill-not-found error construction | `builtin.rs:59-65` vs `builtin.rs:127-133` | same message, two copies |
| output size-cap + `truncate_at_boundary` | `builtin.rs:102-106` vs `builtin.rs:206-209` | same pattern twice, both silent (no marker), unlike the MCP path which always marks (`mcp.rs:906-910`) |

#### Layering debt

- `hints.rs:4` and `builtin.rs:10` import `truncate_at_boundary` from `mcp.rs`. Generic string truncation lives in the MCP transport module, so two filesystem-facing modules depend on the process registry for a byte-slicing helper.
- `catalog.rs:17` imports `build_token_source` from `llm.rs`, so "model catalog" cannot be used without the LLM transport module; `Config::for_discovery` (`config.rs:840-871`) exists solely to satisfy that coupling with ~30 inert fields.
- `mcp.rs` mixes four concerns in one 1,139-line file: child-process lifecycle, tool registry, hook dispatch, and result truncation. The truncation section (`mcp.rs:866-990`) has no MCP dependency at all and would move cleanly.

#### Oversized files and functions

| Unit | Size | Note |
|---|---|---|
| `mcp.rs` | 1,139 lines (production ≈1,004; tests from `mcp.rs:1005`) | over the 1,000-line ceiling `AGENTS.md` documents for `mobile/`; no equivalent guard exists for Rust — `grep -rn 'check-file-sizes' justfile` matches one line, inside the mobile recipe (`justfile:617`) |
| `call_hooks` | `mcp.rs:315-419`, 105 lines | dispatch, timeout accounting, kill escalation, and ordering in one function |
| `browser_pkce_flow` | `auth.rs:527-630`, 104 lines | includes an inline axum router, the HTML responses, and the token exchange |
| `spawn_all` | `mcp.rs:172-264`, 93 lines | eight validation branches interleaved with spawning |
| `load_supporting_file` | `builtin.rs:118-230`, 113 lines | doubly-nested `spawn_blocking` match with four error arms |
| `tool_result_content` | `mcp.rs:913-990`, 78 lines | three nested closures capturing four mutable locals |

For context, the sibling `config.rs` in the same crate is 2,701 lines, so this is a crate-wide pattern rather than a local lapse.

#### Untested critical paths

| Path | Evidence of absence |
|---|---|
| child env allowlist as applied to a real child | `grep -rn 'PASSTHROUGH\|env_clear' crates/buzz-agent/tests` → zero matches; the only unit test checks that the constant array contains `BUZZ_AUTH_TAG` (`mcp.rs:1010-1014`) |
| process-group kill of grandchildren | `grep -rn 'FAKE_MCP_SPAWN_GRANDCHILD' --include='*.rs' .` matches only the fake server's own doc comment and implementation (`tests/bin/fake_mcp.rs:17`, `:228`) — no test drives it |
| restart / backoff state machine | `grep -rn 'ClientState\|RestartCheck\|check_restart_state\|backoff(' crates/buzz-agent/tests` → zero matches |
| `browser_pkce_flow`, `interactive_login` | `grep -rn 'browser_pkce_flow' crates/buzz-agent` matches only `auth.rs:237`, `:293`, `:527` |
| token-cache file permissions | no test asserts a mode anywhere in the crate |
| catalog HTTP layer (`fetch_v1_models`, `fetch_v2_models`, `percent_encode`, pagination, 20-page cap, v2 empty fallback) | `grep -rn 'discover_databricks_models\|fetch_v1_models\|fetch_v2_models\|percent_encode' crates/buzz-agent/tests` → zero matches |
| `MAX_HINTS_BYTES` truncation | `grep -rn 'MAX_HINTS_BYTES' crates/buzz-agent/src crates/buzz-agent/tests` → 2 matches, both in `hints.rs` production code |
| `MAX_HOOK_RESULT_BYTES` truncation | `grep -rn 'MAX_HOOK_RESULT_BYTES' …` → 3 matches, all in `mcp.rs` production code |
| `auth` CLI argument handling | `grep -rn '"auth"' crates/buzz-agent/tests` → zero matches |
| `https`/scheme rejection | nothing to test — no such check exists (`grep -n 'https\|scheme' auth.rs catalog.rs` matches only test fixtures) |

#### Tests that can drift green

- `tests/databricks_oauth.rs:84-97` re-implements `cache_path_for` locally (its own `Sha256` + `hex::encode` + `join`) instead of calling the production function. Change the hash inputs in `auth.rs:446-454` and this test still passes while every cached token becomes unreachable.
- `mcp_init_timeout_kills_child` (`tests/regressions.rs:307-341`) documents "the child process must be killed (not lingering)" (`:303-304`) but asserts only that the error message contains "timeout" and that the call returned within 8 s. The kill is never observed.
- `tool_metadata_caps_enforced` (`tests/regressions.rs:353-411`) explicitly accepts *either* rejection or truncation (`:383-390`). Production rejects (`mcp.rs:232-236`); a change to truncation would not be caught.
- `find_git_root_none` (`hints.rs:288-298`) weakens its own assertion because CI may run inside a real repo — it asserts only that any discovered root is not under the temp dir, so the "no git root" branch is not actually pinned.
- `configure_no_window_is_a_noop_on_non_windows` (`mcp.rs:1118-1125`) states in its own comment that "the only assertion is 'didn't crash'".
- The OAuth on-disk format is asserted only by tests that re-serialize the production `CachedToken` (`auth.rs:750-756`, `auth.rs:786-792`), so a field rename would keep them green while orphaning existing caches.

#### Documentation drift

| Doc statement | Doc site | Code |
|---|---|---|
| MCP child env whitelist is "`PATH`, `HOME`, `TERM`, `LANG`, `LC_ALL`, `TMPDIR`" | `crates/buzz-agent/README.md`, Security Model table | 17 keys including `SSH_AUTH_SOCK`, `GIT_ASKPASS`, `GIT_SSH_COMMAND`, `NOSTR_PRIVATE_KEY`, `BUZZ_PRIVATE_KEY`, `BUZZ_AUTH_TAG` (`mcp.rs:39-63`) |
| "Process group via `setpgid(0,0)` in `pre_exec`" | same table | `Command::process_group(0)` (`mcp.rs:733`); `pre_exec` would require `unsafe`, forbidden at `lib.rs:1` |
| "The full server is hand-rolled in `main.rs`" | README, ACP section | `main.rs` is 6 lines (`src/main.rs`); the server is `lib.rs` |
| `session/new` result shape shows only `sessionId` | README ACP transcript | the response also carries a `models` object (`lib.rs:462-473`) |
| hints / skills / `load_skill` | — | entirely undocumented: `grep -c 'load_skill\|skills\|AGENTS.md' crates/buzz-agent/README.md` → 0; only `CHANGELOG.md:738` records the feature |
| `buzz-agent auth`, the token cache path, the 60 s browser window | — | undocumented; the README's only OAuth references are `README:53` and the `DATABRICKS_TOKEN` row (`README:143`) |
| all four `BUZZ_AGENT_MCP_*` knobs, `BUZZ_AGENT_NO_HINTS`, `BUZZ_AGENT_MODEL` | — | absent from the README config table and from `.env.example` (`grep -c 'BUZZ_AGENT\|DATABRICKS\|MCP_HOOK' .env.example` → 0) |
| "Hook output is JSON-encoded for prompt-injection safety" | `docs/MCP_DRIVEN_HOOKS.md`, Convention list | true for `_Stop` (`agent.rs:641-649`), false for `_PostCompact`, which is concatenated as plain text into a user message (`handoff.rs:87-92`, `handoff.rs:247-253`) |
| `BUZZ_AGENT_MAX_HISTORY_BYTES` default "1048576 / 1 MiB" | `crates/buzz-agent/README.md:155`, `:236` | `16 * 1024 * 1024` (`config.rs:815`) — outside this group, but it calibrates how far the README's table can be trusted |

#### Stale in-code cross-references

`grep -n '\.rs:[0-9]' mcp.rs auth.rs hints.rs builtin.rs catalog.rs` returns zero matches, so there are no hardcoded `file:line` pointers to go stale. Cross-references are by identifier instead (`[\`build_token_source\`](crate::llm::build_token_source)` at `catalog.rs:6`, `[\`RunCtx::should_handoff\`]` style links elsewhere in the crate), which rustdoc will fail on if the target disappears — a better pattern. Two references point at other repos' behaviour and cannot be verified from here: "Mirrors goose's `DATABRICKS_V2_KNOWN_MODELS`" (`catalog.rs:31`) and "the same shape goose uses for Databricks" (`auth.rs:9-11`); the two model IDs in that list (`catalog.rs:32-33`) are compile-time constants with no freshness mechanism.

#### Missing observability

`hints.rs`, `builtin.rs`, and `catalog.rs` contain no logging at all (`grep -n 'tracing::' hints.rs builtin.rs catalog.rs` → zero matches). Silent-by-design outcomes with no diagnostic trail: a `SKILL.md` skipped for missing `name` (`hints.rs:124-126`), a skill name shadowed by an earlier directory (`hints.rs:127-131`), a hint chain truncated at 128 KiB (`hints.rs:77-81`), a `load_skill` result cut at 32 KiB (`builtin.rs:102-105`, `:206-209`), and a catalog walk stopped at 20 pages (`catalog.rs:200`). The first two are the ones most likely to be reported as "my skill isn't showing up".

#### Suggested order of repair

1. Set `0o600` on the OAuth token cache and use a unique temp filename (`auth.rs:146-200`) — smallest change, largest security return.
2. Correct or implement the `kill_server` restart-budget comment (`mcp.rs:414-419` vs `mcp.rs:432`, `mcp.rs:672-677`).
3. Add timeouts to the OAuth and catalog HTTP clients (`auth.rs:153`, `catalog.rs:80`), matching `llm.rs:53-57`; a hung host currently stalls `session/new`.
4. Reconcile the README's env-allowlist and `pre_exec` claims with `mcp.rs:39-63` and `mcp.rs:733`, and document `NOSTR_PRIVATE_KEY` / `BUZZ_PRIVATE_KEY` passthrough explicitly.
5. Bound or escape skill `name`/`description` before they enter the system prompt (`hints.rs:239-243`), bringing them in line with the trust rules the hooks doc already states.
6. Reject non-`https` OAuth and catalog URLs, or document the decision to allow `http` (`auth.rs:161-190`, `catalog.rs:81`).
7. Escape the reflected error in the loopback callback page (`auth.rs:565`).
8. Give the four untested critical paths a test each: applied child env, grandchild kill, restart/backoff, catalog HTTP.
