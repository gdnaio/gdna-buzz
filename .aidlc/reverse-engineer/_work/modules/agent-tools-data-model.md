## Module: buzz-agent — MCP registry, OAuth, hints & catalog (`crates/buzz-agent/src`)
### Aspect: Data Model

Scope: `mcp.rs` (1,139 lines), `auth.rs` (845), `hints.rs` (726), `builtin.rs` (575), `catalog.rs` (402). All types here are in-memory except one on-disk artifact (the OAuth token cache).

#### MCP registry types (`mcp.rs`)

| Type | Site | Fields / variants |
|---|---|---|
| `type Client` | `mcp.rs:83` | alias for `rmcp::service::RunningService<RoleClient, ()>` — the agent is a bare client with no server handlers |
| `ServerSpec` (private, `Clone`) | `mcp.rs:85-92` | `name: String`, `command: String`, `args: Vec<String>`, `env: Vec<(String,String)>`, `cwd: String` |
| `ClientState::Healthy` | `mcp.rs:95-99` | `client: Arc<Client>`, `pgid: Option<u32>`, `tools: Arc<Vec<String>>` (bare names from the last `tools/list`) |
| `ClientState::Dead` | `mcp.rs:100-106` | `attempts: u32`, `next_retry: Instant`, `reason: String`, `tools: Arc<Vec<String>>` (preserved from the last `Healthy` so `tools()` filtering stays accurate while dead — comment `mcp.rs:104`) |
| `Server` (private) | `mcp.rs:109-114` | `name`, `spec: ServerSpec`, `client: ArcSwap<ClientState>`, `restart_lock: AsyncMutex<()>` |
| `RestartCheck` | `mcp.rs:124-130` | `Healthy` \| `Ready { attempt_n: u32, prev_tools: Arc<Vec<String>> }` |
| `Entry` (private) | `mcp.rs:154-157` | `server_idx: usize` (index into `servers`), `bare: String` (unqualified tool name) |
| `McpRegistry` (pub in a private module) | `mcp.rs:159-170` | `by_qname: HashMap<String, Entry>`, `defs: Vec<ToolDef>`, `servers: Vec<Arc<Server>>`, `max_attempts: u32`, `backoff_base/backoff_max: Duration`, `init_timeout: Duration`, `hook_timeouts: std::sync::Mutex<HashMap<String,u32>>` |
| `ResultBudget` (`Clone, Copy`) | `mcp.rs:32-37` | `total: usize` (text + images), `text: usize` (text only) |

`ArcSwap<ClientState>` is the only mutable per-server cell (`mcp.rs:112`); every transition is a whole-state swap, and two transitions use `compare_and_swap` to avoid clobbering a concurrent restart (`mcp.rs:441-443`, `mcp.rs:475`). `restart_lock` (`mcp.rs:113`) serialises restarts per server. `hook_timeouts` is the only `std::sync::Mutex` (`mcp.rs:170`) — it is a plain counter map keyed by server name.

No type in `mcp.rs` derives `Serialize`/`Deserialize`; `ToolDef` (produced at `mcp.rs:252-257`) is defined in the sibling `types.rs`. `Server` has a `Drop` impl that kills the process group if the state is still `Healthy` with a pgid (`mcp.rs:116-122`).

#### Restart / health state machine (`mcp.rs`)

| Transition | Trigger | Site | Resulting `Dead` fields |
|---|---|---|---|
| `Healthy → Dead` | explicit kill (tool timeout, consecutive hook timeout) | `mcp.rs:421-451` | `attempts: 1` (`mcp.rs:432`), `next_retry = now + backoff(1)` (`mcp.rs:433`), `reason` = caller string |
| `Healthy → Dead` | transport error observed on a call | `mcp.rs:453-483` | `attempts: 1` (`mcp.rs:470`), `next_retry = now + backoff(1)` (`mcp.rs:471`) |
| `Dead → Healthy` | lazy restart succeeds | `mcp.rs:672-677` | n/a — the new `Healthy` carries **no** attempt count, so the counter resets |
| `Dead → Dead` | restart attempt fails | `mcp.rs:683-699` | `attempts: attempt_n`, `next_retry = now + backoff(attempt_n)`, or `now + 86_400s` when `attempt_n >= max_attempts` (`mcp.rs:685-690`) |
| `Dead → error` | `attempts >= max_attempts` | `mcp.rs:135-140` | terminal for the session: "unavailable (exhausted)" |
| `Dead → error` | `now < next_retry` | `mcp.rs:141-146` | transient: "is recovering (last error: …)" |

Consequence of the reset at `mcp.rs:672-677`: `attempts` only counts **consecutive spawn failures**. A server that starts cleanly but wedges on every call cycles `Healthy → kill(attempts=1) → restart OK → Healthy` without bound. See Business Rules and Security for the contradiction with the comment at `mcp.rs:414-419`.

`backoff` (`mcp.rs:813-821`) is `base << min(attempt-1, 20)`, capped at `backoff_max`, then jittered by ±20% (`jitter_percent`, `mcp.rs:823-827`) — so the realised ceiling is `1.2 × backoff_max`, not `backoff_max`. `jitter_percent` ignores RNG failure (`let _ = getrandom::fill`, `mcp.rs:825`), in which case the buffer stays zeroed and jitter is a deterministic −20%.

#### Truncation / budget types (`mcp.rs`)

| Constant | Value | Site | Applies to |
|---|---|---|---|
| `SEP` | `"__"` | `mcp.rs:19` | qualified name separator |
| `MAX_NAME_LEN` | 128 | `mcp.rs:20` | server name and bare tool name (`valid_name`, `mcp.rs:859-864`) |
| `MAX_QNAME_LEN` | 64 | `mcp.rs:21` | `server__tool` |
| `MAX_TOOLS_PER_SESSION` | 128 | `mcp.rs:22` | registry-wide tool count |
| `MAX_DESCRIPTION_BYTES` | 1024 | `mcp.rs:23` | tool description, via `clamp` |
| `MAX_SCHEMA_BYTES` | 4096 | `mcp.rs:24` | serialized input schema |
| `MARKER_FIELD_MAX` | 256 | `mcp.rs:25` | mime-type / URI text inside elision markers |
| `MAX_MCP_SERVERS` | 16 | `mcp.rs:26` | servers per session |
| `MAX_HOOK_RESULT_BYTES` | 16 KiB | `mcp.rs:27` | both budget fields for hook calls (`mcp.rs:355-358`) |
| `ELISION_MARKER_ALLOWANCE` | 128 | `mcp.rs:879` | slack reserved for the middle-elision marker |

`MAX_NAME_LEN` (128) is unreachable in practice for anything that produces a tool: a name long enough to matter makes every `qname` exceed `MAX_QNAME_LEN` (64) and fail registration at `mcp.rs:242-248`.

Truncation helpers are `truncate_at_boundary` (`mcp.rs:866-877`, head cut on a char boundary) and `truncate_middle` (`mcp.rs:886-911`, head+tail with a byte-count marker). `tool_result_content` (`mcp.rs:913-993`) assembles `Vec<ToolResultContent>` under both budgets, flushing accumulated text through `truncate_middle` and passing images whole or replacing them with a text marker.

#### OAuth token / PKCE state (`auth.rs`)

| Type | Site | Fields |
|---|---|---|
| `PkceOAuthConfig` (`Debug, Clone`, public) | `auth.rs:97-108` | `discovery_url`, `client_id`, `scopes: Vec<String>`, `cache_namespace`, `cache_dir_override: Option<PathBuf>` (test-only per doc `auth.rs:102-106`) |
| `CachedToken` (`Debug, Serialize, Deserialize, Clone`, private) | `auth.rs:110-118` | `access_token: String`, `refresh_token: Option<String>`, `expires_at: Option<u64>` (unix seconds) |
| `OidcEndpoints` (private) | `auth.rs:119-124` | `authorization_endpoint`, `token_endpoint` |
| `PkceOAuthTokenSource` (public) | `auth.rs:134-142` | `cfg`, `http: reqwest::Client`, `cache_path: PathBuf`, `state: tokio::sync::Mutex<Option<CachedToken>>` |
| `StaticTokenSource` (public tuple struct) | `auth.rs:76` | one `String` |
| `AbortOnDrop` (private) | `auth.rs:426-432` | wraps `JoinHandle<()>`, aborts on drop |

PKCE material is never stored in a struct: the verifier/challenge pair (`pkce_pair`, `auth.rs:509-516`, 48 random bytes → base64url, 64 chars) and the `state` (`random_state`, `auth.rs:518-522`, 16 random bytes → 22 chars) are locals inside `browser_pkce_flow` (`auth.rs:527-630`) and die with the call.

`Debug` is derived on `PkceOAuthConfig` (`auth.rs:97`) and on `CachedToken` (`auth.rs:110`) — so `{:?}` on a `CachedToken` prints the access and refresh tokens verbatim. No `Debug` impl is customised or redacted anywhere in the file (`grep -n 'impl.*Debug' auth.rs` → zero matches).

Constants: `TOKEN_REFRESH_LEEWAY = 60s` (`auth.rs:35`), `BROWSER_AUTH_TIMEOUT = 60s` (`auth.rs:39`).

#### On-disk token cache format

Path (`cache_path_for`, `auth.rs:445-467`):

```
$HOME/.config/buzz-agent/oauth/<cache_namespace>/<sha256_hex>.json
```

`<sha256_hex>` = `sha256(discovery_url | "|" | client_id | "|" | scopes.join(","))` hex-encoded (`auth.rs:446-454`). `$HOME` is read directly (`auth.rs:457-458`); `XDG_CONFIG_HOME` is **not** consulted even though the string `.config` is hardcoded (`auth.rs:460`). `cache_namespace` is `"databricks"` at both construction sites (`lib.rs:142`, `llm.rs:1193`).

Content is `serde_json::to_vec_pretty(&CachedToken)` (`auth.rs:192`), i.e. pretty-printed JSON with the raw bearer and refresh token as plaintext string fields. Write is `write tmp → rename` where tmp is `cache_path.with_extension("json.tmp")` (`auth.rs:195-200`). File mode is whatever the umask gives: `grep -rn 'set_permissions\|PermissionsExt\|0o600' crates/buzz-agent/src` returns zero matches (the repo does use `.mode(0o600)` elsewhere — `crates/buzz-dev-mcp/src/shim.rs:141`).

#### Skill / hint entry types (`hints.rs`)

| Item | Site | Notes |
|---|---|---|
| `SkillEntry` (`Clone`) | `hints.rs:14-25` | `name: String`, `description: String`, `path: PathBuf` (absolute `SKILL.md`), `supporting_files: Vec<PathBuf>` (absolute, sorted, pre-enumerated) |
| `SKILL_DIRS` | `hints.rs:8` | `[".agents/skills", ".goose/skills", ".claude/skills"]` — search order is precedence order |
| `MAX_HINTS_BYTES` | `hints.rs:6` | 128 KiB across the whole concatenated `AGENTS.md` chain |
| `MAX_SKILL_BODY_BYTES` | `hints.rs:7` | 32 KiB, public, consumed by `builtin.rs:9` |

There is no `SkillEntry` id, hash, or mtime — discovery is a one-shot snapshot taken at `session/new` (`lib.rs:356-357`) and stored on the session (`lib.rs:32-34`), then cloned per prompt (`lib.rs:795`, in `acquire_session`). Nothing invalidates it if the files change mid-session.

`parse_skill_frontmatter` (`hints.rs:86-108`) deserialises the YAML block into `HashMap<String, serde_yaml::Value>` and keeps only `name` (required, trimmed, non-empty) and `description` (optional, defaults to `""`). Every other frontmatter key is discarded — there is no schema type for skill metadata.

#### `load_skill` result shape (`builtin.rs`)

`call_load_skill` returns `ToolResult` (from `types.rs`) with `provider_id: String::new()` — the caller overwrites it (`agent.rs:321`). Two output shapes:

- plain skill name: frontmatter-stripped body (`builtin.rs:83`) plus an optional trailing supporting-files listing introduced by a literal `## Supporting Files` heading (`builtin.rs:88`), each line `- <rel> (load_skill(name: "<skill>/<rel>"))` (`builtin.rs:92-95`);
- `skill/rel/path`: `# Loaded: <skill>/<rel>` header, file content, and a `---\nFile loaded into context.` trailer (`builtin.rs:202-205`).

Both are capped at `MAX_SKILL_BODY_BYTES` with `truncate_at_boundary` (`builtin.rs:102-105`, `builtin.rs:206-209`) — a head cut with no marker, unlike MCP tool results which get an explicit elision marker (`mcp.rs:906-910`).

#### Model catalog types (`catalog.rs`)

| Item | Site | Notes |
|---|---|---|
| `ModelEntry` (`Debug, Clone, PartialEq, Eq`) | `catalog.rs:22-27` | `id`, `name`; both set to the endpoint `name` in v1 (`catalog.rs:167-170`) and v2 (`catalog.rs:254-257`) |
| `DATABRICKS_V2_KNOWN_MODELS` | `catalog.rs:31-33` | `["databricks-gpt-5-5", "databricks-claude-opus-4-7"]`, "Mirrors goose's `DATABRICKS_V2_KNOWN_MODELS`" |

No struct models the Databricks response; both parsers walk `serde_json::Value` by key (`catalog.rs:132-136`, `catalog.rs:266-272`). Pagination state is a local `Option<String>` page token (`catalog.rs:197`) bounded by a 20-iteration `for` loop (`catalog.rs:200`).

#### Test coverage — Data Model

Covered: `ResultBudget`-driven assembly and both truncation helpers (`tool_result_content_preserves_images` `mcp.rs:1035`, `tool_result_content_elides_images_over_budget` `mcp.rs:1053`, `oversized_text_is_middle_elided` `mcp.rs:1061`, `text_within_budget_is_untouched` `mcp.rs:1082`, `image_passes_whole_even_when_text_budget_is_small` `mcp.rs:1090`, `truncate_middle_respects_max_and_boundaries` `mcp.rs:1107`); `CachedToken` expiry semantics (`cached_token_no_expiry_is_not_expired` `auth.rs:646`, `cached_token_far_future_is_not_expired` `auth.rs:656`, `cached_token_within_leeway_is_expired` `auth.rs:671`); cache path shape (`cache_path_includes_namespace_and_hash` `auth.rs:686`); token-response parsing (`auth.rs:701`, `710`, `716`); `SkillEntry.supporting_files` population (`discover_skills_populates_supporting_files` `hints.rs:702`); `ModelEntry` extraction (`catalog.rs:307`, `348`).

Not covered: the `ClientState` state machine — `grep -rn 'ClientState\|RestartCheck\|check_restart_state\|backoff(' crates/buzz-agent/tests` returns zero matches, and no unit test in `mcp.rs` touches `Server`, `ClientState`, `backoff`, or `jitter_percent`. `MAX_HINTS_BYTES` has no test (`grep -rn 'MAX_HINTS_BYTES' src tests` → only `hints.rs:6` and `hints.rs:71`). `MAX_HOOK_RESULT_BYTES` has no test (`grep -rn 'MAX_HOOK_RESULT_BYTES' src tests` → only `mcp.rs:27,356,357`). The on-disk cache **format** is asserted only indirectly, by tests that write `CachedToken` JSON themselves (`auth.rs:750-756`, `auth.rs:786-792`) — they re-serialize the production struct, so they cannot detect a format change.
