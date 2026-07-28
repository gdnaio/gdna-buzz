<!-- Analyzed: 2026-07-25T01:12:08Z | Scope: full project -->
# Configuration & Environment

## Summary

Configuration is **environment-variable-first**, loaded from a `.env` file (copied from
`.env.example` by `just bootstrap`) with `set dotenv-load := true` in the `Justfile`.
**140 distinct `BUZZ_*`/infra env vars are read by the Rust workspace**, plus 27 read by
the Tauri desktop shell (including 7 compile-time `BUZZ_BUILD_*` flags baked in by
`build.rs`) and 6 Vite `import.meta.env` flags in the frontends. All local-dev defaults
work out of the box against Docker Compose. Secrets management is **env-var based** with
no vault integration in-repo; the desktop app stores identity keys in the **OS keychain**
(`secret_store.rs`, `app_state_keyring.rs`) rather than env. A separate declarative
feature-flag file (`preview-features.json`) gates preview UI features per platform.
Notable drift: `.env.example` still documents **Typesense** (`TYPESENSE_API_KEY`,
`TYPESENSE_URL`) although search moved to Postgres FTS, and Keycloak runs in
docker-compose without any documented env surface.

### Batch 2b findings (service crates)

The service crates are almost entirely **configuration-free by design** — they receive
configuration as constructor arguments from the relay rather than reading the environment.
`buzz-pubsub` reads **zero** env vars in any production path (its single `env::var` is a
`REDIS_URL` lookup inside a `#[cfg(test)]` module at
`crates/buzz-pubsub/src/nip98_replay.rs:110`).

Three configuration defects surfaced:

**Three of seven `BUZZ_RATE_LIMIT_*` variables are dead.**
`AGENT_STANDARD_API_CALLS`, `AGENT_ELEVATED_MESSAGES`, and `AGENT_PLATFORM_MESSAGES` are
parsed at `crates/buzz-relay/src/config.rs:303-314` and read nowhere. Only four reach
enforcement. Operators can set the dead three and observe no behaviour change. This sits
alongside the larger documentation error that rate limiting isn't enforced at all — it is
(see `security.md` SEC-13).

**Media size limits are documented at the wrong layer.** `ARCHITECTURE.md`'s "50 MB limit"
is a *relay default for images only*; `buzz-media` itself enforces video 500 MB and file
100 MB. An operator reading the architecture doc would size infrastructure for the wrong
ceiling.

**`BUZZ_REQUIRE_MEDIA_GET_AUTH` defaults to false**, so Blossom media reads are
unauthenticated out of the box even though `verify_blossom_get_auth` is implemented.

Operationally significant items that are **not** configurable and arguably should be, all
in `buzz-pubsub`: presence TTL (90 s, `src/presence.rs:16`), all four channel capacities
(4096 each, `src/lib.rs:126-129`), and the reconnect backoff envelope (1 s → 30 s,
triplicated). One knob that *was* built is unreachable —
`PubSubConfig::with_unsubscribe_debounce` (`src/lib.rs:93`) has no production caller
because every construction goes through `PubSubManager::new`, which hardcodes the 500 ms
default (`src/lib.rs:117-119`).

Two smaller inconsistencies: the relay's Redis default is spelled `localhost:6379`
(`crates/buzz-relay/src/config.rs:419`) while `buzz-pubsub`'s test helper hardcodes
`127.0.0.1:6379` (`src/lib.rs:371-376`) and ignores `REDIS_URL` — different resolution on
IPv6-preferring hosts. And nothing anywhere asserts a Redis `noeviction` policy, which the
NIP-98 replay seen-set silently depends on.

### Batch 2c configuration (relay, mesh, conformance)

- **The most security-relevant default is undocumented.** `BUZZ_REQUIRE_AUTH_TOKEN` defaults
  to **false** (`crates/buzz-relay/src/config.rs:475-477`) and is **absent from the root
  `.env.example`**, so the header-impersonation path (SEC-29 in `security.md`) is open on a
  default deployment with no configuration signal. The one place it *is* surfaced is the
  compose bundle template, which sets it to `true`
  (`deploy/compose/.env.example:16`) — so a compose-based deployment is safe by default while
  a `.env.example`-derived one is not.
- **The health listener bind address is hard-coded `0.0.0.0`**
  (`crates/buzz-relay/src/main.rs:1116`, `:1119`) — not configurable — and it is what exposes
  the unauthenticated `GET /_mesh`.
- **`ARCHITECTURE.md:624`'s "50 MB" is actually 500 MB** in the router's configured limit
  (`crates/buzz-relay/src/router.rs:33-37`, applied at `:45`). Still true at `07d0265c`;
  neither file changed in the 2c sync — the earlier `:623` citation was off by one line and
  is corrected here to match §6 of this document.
- **`ARCHITECTURE.md:824` documents `/api/presence`, which does not exist.**
- **A subsystem whose entire purpose is switchable observation has no switch.**
  `buzz-conformance` has no `[features]` table (`crates/buzz-conformance/Cargo.toml`, 35
  lines), no `#[cfg(feature = …)]` anywhere in `src/`, and no env var or config field selecting
  a tracer. `buzz-relay`'s dependency is unconditional
  (`crates/buzz-relay/Cargo.toml:20`), so the crate compiles into every relay build. Two
  comments promise a feature gate that does not exist
  (`crates/buzz-conformance/src/lib.rs:321-322`,
  `crates/buzz-relay/src/conformance/tracers.rs:9-13`).
- **Switching to a recording tracer requires a code edit.** The only binding site is
  `crates/buzz-relay/src/state.rs:798`; `JsonlTracer::create` takes its output path as a plain
  argument with no default, env fallback, or directory convention
  (`conformance/tracers.rs:37-45`).
- **The one conformance env var is test-only and presence-checked, not value-parsed** —
  `BUZZ_CONFORMANCE_UPDATE` regenerates golden fixtures whether set to `1` or `0`
  (`crates/buzz-conformance/tests/replay_fixtures.rs:210-214`).
- **An observability path costs a Postgres round-trip per read, gated on the wrong
  condition.** `db.communities_of_channels` runs per REQ filter and per search page
  (`handlers/req.rs:345`, `:661`) gated on `trace_state.is_some()` (`:334`, `:649`), and
  `trace_state` depends only on whether the authenticated pubkey bytes parse (`:116-118`) —
  not on which tracer is bound. The `Tracer` trait has no `is_recording()` to short-circuit it
  (`crates/buzz-conformance/src/lib.rs:314-318`).

### Batch 2d configuration (agent surface: buzz-acp, buzz-agent, buzz-dev-mcp, buzz-cli)

The agent surface is configured almost entirely by environment variable, and it is the least
documented configuration surface in the repo. Counting only what these four crates read:
**19 `BUZZ_ACP_*` variables, all 36 `BUZZ_AGENT_*`-surface variables, `BUZZ_SHELL` / `GIT_BASH` /
`NOSTR_PRIVATE_KEY`, and `BUZZ_TIMEOUT_SECS` / `BUZZ_CONNECT_TIMEOUT_SECS` are absent from the
root `.env.example`.** Where documentation does exist it is frequently wrong about the default.
(`buzz-agent`'s count moved from 32 to 36 in `16d4ec33`, which added the mesh-routing knob
`BUZZ_AGENT_PREFER_MESH_FOR_AUTO` — see CFG-2d-6.)

| ID | Finding | Location |
|---|---|---|
| CFG-2d-1 | **`BUZZ_AGENT_MAX_HISTORY_BYTES` is documented as 16× smaller than it is.** Code default is `16 * 1024 * 1024` (`config.rs:821`); `crates/buzz-agent/README.md:155` says `1048576` / "1 MiB", repeated in the limits table at `:236`. An operator sizing a deployment from the README is off by a factor of 16. | `crates/buzz-agent/src/config.rs:821` vs `crates/buzz-agent/README.md:155`, `:236` |
| CFG-2d-2 | **`BUZZ_API_TOKEN` is parsed and never read.** `propagate_legacy_env_vars` writes it (`config.rs:718`); there are zero read sites. `crates/buzz-acp/README.md:107` still lists it as required, and `crates/buzz-cli/Cargo.toml:19` still claims clap "auto-wires" it. The variable is genuinely live in `buzz-workflow` (`executor.rs:901`) and the desktop agent runtime (`desktop/src-tauri/src/managed_agents/env_vars.rs:63`), so the claims are stale per-crate rather than repo-wide. | `crates/buzz-acp/src/config.rs:718`; `crates/buzz-acp/README.md:107`; `crates/buzz-cli/Cargo.toml:19` |
| CFG-2d-3 | **`.env.example:152` documents a hidden deprecated variable instead of the live one.** It shows `BUZZ_ACP_TURN_TIMEOUT=320`; the live knob is `BUZZ_ACP_IDLE_TIMEOUT`, default **900** (`config.rs:27`), which `crates/buzz-acp/README.md` reports as 620. Three values, three sources, no agreement. | `.env.example:152`; `crates/buzz-acp/src/config.rs:27` |
| CFG-2d-4 | **Nineteen `buzz-acp` variables are undocumented in `.env.example`**, including several that change the security posture: `BUZZ_ACP_PERMISSION_MODE` (default `bypass-permissions`, `config.rs:435`), `BUZZ_ACP_RESPOND_TO`, `BUZZ_ACP_RESPOND_TO_ALLOWLIST`, `BUZZ_AUTH_TAG`, `BUZZ_ACP_LAZY_POOL`, `BUZZ_ACP_SETUP_PAYLOAD`, `BUZZ_ACP_MAX_TURN_DURATION`, `BUZZ_ACP_MULTIPLE_EVENT_HANDLING`. | `crates/buzz-acp/src/config.rs:435` and siblings; verified by grep against `.env.example` |
| CFG-2d-5 | **`BUZZ_ACP_EVENT_BUFFER` silently sizes two channels while the docs describe one.** Declared at `relay.rs:35-42` with default 256, commented out at `.env.example:221`, and applied to both channels at `relay.rs:610-612`. | `crates/buzz-acp/src/relay.rs:35-42`, `:610-612`; `.env.example:221` |
| CFG-2d-6 | **`buzz-agent` reads 36 environment variables and no CLI flag; `.env.example` documents none of them** (zero `BUZZ_AGENT` matches). Eleven are absent from the crate README's own configuration table as well: `BUZZ_AGENT_MODEL` (`config.rs:755`), `BUZZ_AGENT_PREFER_MESH_FOR_AUTO` (`:807`), `BUZZ_AGENT_THINKING_EFFORT` (`:833`), `MCP_HOOK_SERVERS` (`:831`), `BUZZ_AGENT_NO_HINTS` (`:832`), `BUZZ_AGENT_HOOK_TIMEOUT_MS` (`:829`), `BUZZ_AGENT_STOP_MAX_REJECTIONS` (`:830`), and the four `BUZZ_AGENT_MCP_*` knobs (`:812`-`:818`). `MCP_HOOK_SERVERS` is the gate for the entire MCP-driven-hooks feature the README links to at `:254`, and it is also the only variable in `from_env` without the `BUZZ_AGENT_` prefix. `BUZZ_AGENT_THINKING_EFFORT` remains the most consequential of the pre-existing omissions: it has its own 13-line type doc, five operator-facing warnings that name it back, ~70 unit tests, and a desktop UI backed by a shared JSON fixture. The newest omission, `BUZZ_AGENT_PREFER_MESH_FOR_AUTO` (added `16d4ec33`), is worse in kind, not just undocumented: it silently changes *which model serves a turn* (mesh's virtual `mesh` model vs. the configured `auto`), has no pure parser to unit-test (inlined `parse_env(..., 0u8)? != 0` at `:807`, unlike `parse_thinking_effort`/`parse_hook_servers`), and is documented nowhere in the repo except a same-named Rust constant in the desktop crate (`desktop/src-tauri/src/managed_agents/relay_mesh.rs:6`) that nothing links to this one — renaming either side compiles clean and silently disables the feature. | `crates/buzz-agent/src/config.rs:742-837` vs `crates/buzz-agent/README.md:128-157` |
| CFG-2d-7 | **Three undocumented provider/API aliases are accepted silently**, with no indication which form is canonical: `openai-compat` (`config.rs:1003`), `databricks-v2` (`config.rs:1008`), and `chat-completions` / `chat_completions` for `OPENAI_COMPAT_API` (`config.rs:1024`). Re-verified: `grep -ni 'deprecat'` across `config.rs` and `llm.rs` still returns zero matches, so there is no deprecation-warning path either. | `crates/buzz-agent/src/config.rs:1003`, `:1008`, `:1024` |
| CFG-2d-8 | **An "inert default" that is not inert.** `openai_api` is hard-coded to `OpenAiApi::Chat` for both Databricks providers (`config.rs:789`) with a comment saying it is "only read by OpenAI/legacy Databricks dispatch" — it *is* read for legacy Databricks (`llm.rs:545-546`, `:563`), and the value permanently disables the chat→responses auto-upgrade there. `Provider::Anthropic` gets `Auto` instead (`config.rs:772`) for no stated reason. A real behavioural decision expressed as a placeholder. | `crates/buzz-agent/src/config.rs:772`, `:789`; `llm.rs:545-546`, `:563` |
| CFG-2d-9 | **`buzz-dev-mcp`'s three variables are undocumented**: `BUZZ_SHELL`, `GIT_BASH`, `NOSTR_PRIVATE_KEY` — none in `.env.example`, and the crate has no README to document them in. | `crates/buzz-dev-mcp/src/shell.rs`; verified by grep against `.env.example` |
| CFG-2d-10 | **`BUZZ_ACP_ALLOWED_CHANNEL_ADD_POLICIES` is a wholly undocumented deployment control whose absence means permit-all.** Read at `channels.rs:1021`, parsed as comma-split + trim + drop-empty (`:1022-1026`), with an empty or whitespace-only value also meaning "no restriction" (`:1027`). No case folding and no validation that the listed names are real policies. Zero matches in `.env.example`, `AGENTS.md` and `crates/buzz-cli/README.md`. `channels.rs:1014-1019` documents in-code that it is a client-side courtesy gate only — a direct kind-10100 submission bypasses it. | `crates/buzz-cli/src/commands/channels.rs:1014-1027` |
| CFG-2d-11 | **The CLI's only two operational tuning knobs are documented only in a Rust doc comment.** `BUZZ_TIMEOUT_SECS` (default 30 s) and `BUZZ_CONNECT_TIMEOUT_SECS` (default 15 s) are read via `env_duration_secs` (`client.rs:140-150`, applied `:547-549`) and appear in no `.md`, `.toml` or `.example` file. Both silently fall back to the default on a non-numeric value **and** on `0` — the latter deliberately, so timeouts cannot be disabled (`client.rs:139`). All four branches are pinned by one test (`client.rs:1554-1580`). | `crates/buzz-cli/src/client.rs:139-150`, `:535-540`, `:547-549` |
| CFG-2d-12 | **`.env.example` and the CLI disagree on the literal default relay URL.** `.env.example:130` shows `BUZZ_RELAY_URL=ws://localhost:3000` inside a section scoped to the ACP harness; the CLI's default and its `long_about` are `http://localhost:3000` (`lib.rs:81`, `:70`). Both work — `normalize_relay_url` rewrites the scheme (`client.rs:1291-1297`) — but the documented value never matches the documented default, and there is no buzz-cli section in `.env.example` at all. `BUZZ_AUTH_TAG` is absent from `.env.example` entirely (zero matches) and from the CLI README's auth table, despite being required by `agents draft-create`/`draft-update` (`agents.rs:156-162`) and by `mem` unless `--owner` is given (`mem.rs:37-42`). | `.env.example:130` vs `crates/buzz-cli/src/lib.rs:70`, `:81`, `:87-89` |
| CFG-2d-13 | **`--format` is parsed for every invocation and reaches 5 of 21 command groups.** Declared on the top-level `Cli` **without** `global = true` (`lib.rs:92-93`), so it is positionally sensitive: `buzz --format compact channels list` parses, `buzz channels list --format compact` exits 1. Forwarded only at `lib.rs:1772`, `:1773`, `:1778`, `:1780`, `:1790`; `moderation::dispatch` receives it and binds it as `_format` (`moderation.rs:133-137`). Within the three messaging dispatchers that do receive it, only `messages get/thread/search`, `feed get` and `channels list` honour it — and `channels list` emits a *different* compact shape (`channels.rs:95-109`) than `messages`/`feed` (`messages.rs:242-262`, `feed.rs:45-64`). `mem ls` has a redundant private `--json` instead (`lib.rs:1551-1552`). No test covers format propagation anywhere. | `crates/buzz-cli/src/lib.rs:92-93`, `:1772-1790`, `:1551-1552`; `commands/moderation.rs:133-137` |
| CFG-2d-14 | **Values silently ignored when they fail to parse.** `messages get --kinds abc` runs `filter_map(parse().ok())`, yields an empty list, fails the `!is_empty()` guard at `messages.rs:282`, and sends the request with the **default** kinds — exit 0, no warning; `--kinds 9,abc` silently degrades to `[9]`. `notes ls --author`'s handler-side `unwrap_or("me")` (`notes.rs:678`) is dead because clap always supplies `"me"` (`lib.rs:1069-1071`), making the `Option<String>` type misleading. | `crates/buzz-cli/src/commands/messages.rs:274-285`; `notes.rs:678` |
| CFG-2d-15 | **Three different stdin size caps coexist for the same "read a body from `-`" job, none documented.** Unbounded for `messages send` and `canvas set` (`validate.rs:168-181` applies no cap; `messages send` merely rejects afterwards at `messages.rs:480`, and `canvas set` never checks size at `channels.rs:1050`); 1 MiB for `notes set` (`SET_STDIN_MAX_BYTES`, `notes.rs:485`); 10 MB for `emoji import` (`STDIN_MAX_BYTES`, `emoji.rs:159`). `mem` is the only family that bounds *during* the read (`.take(NIP44_PLAINTEXT_MAX + 1)`, `mem.rs:324-338`) — and even there the `--patch-file` arm skips the bound (`mem.rs:577` computes `limit`, `:581` ignores it). | `crates/buzz-cli/src/validate.rs:168-181`; `commands/notes.rs:485`; `emoji.rs:159`; `mem.rs:324-338`, `:577-585` |
| CFG-2d-16 | **Hardcoded kind integers are the CLI's largest de-facto configuration surface, against `AGENTS.md § Common Gotchas #1`.** Of 20 filter literals in the dev-command group, only 5 name a constant. `repos.rs` is internally inconsistent — it imports `KIND_GIT_REPO_ANNOUNCEMENT` (`:3`), uses it at `:21`, then hardcodes `30617` at `:240` and `:271` — as is `workflows.rs`, which uses `buzz_sdk::kind::KIND_WORKFLOW_TRIGGER` at `:176` but literals `30620` at `:16`/`:41` and `46001, 46002, 46003` at `:74`. Also literal: `1618` (`pr.rs:110`, `:131`), `1617` (`patches.rs:76`, `:96`), `1621` (`issues.rs:39`, `:60`), `40902` (`users.rs:258`), `0` (`users.rs:42`, `:91`, `:223`; `agents.rs:180`). In the messaging group: `39002`, `39000`, `40100` (`channels.rs:38`, `:54`, `:265`), `7` (`reactions.rs:45`), `41001`/`41010` (`dms.rs:12`, `:69`), `1`/`3` (`social.rs:97`, `:118`), and `notes.rs:38` redeclares `KIND_LONG_FORM = 30023` which already exists at `kind.rs:66`. The NIP-34 `#a` coordinate `format!("30617:{owner}:{id}")` is hand-built in three files despite `GitRepoCoord::to_a_tag_value` (`builders.rs:976`). | `crates/buzz-cli/src/commands/repos.rs:240`, `:271`; `workflows.rs:16`, `:74`; `notes.rs:38` vs `crates/buzz-core/src/kind.rs:66` |
| CFG-2d-17 | **`buzz-acp`'s pool and queue tuning is mostly compile-time.** `OBSERVER_LEAF_RETAIN_BYTES = 3_000` (`lib.rs:632`), the 3 s typing tick (`lib.rs:2332-2341`), `MAX_TOOL_CALLS_PER_TURN = 64`, the 30 s keepalive interval (`agent.rs:129`), the 5 s post-cancel drain (`agent.rs:455`), the 64-slot wire channel (`lib.rs:164`), and `usage_update.contextLimit = 0` (`lib.rs:742`) are all literals. The keepalive interval is the consequential one: it exists to satisfy the harness idle clock, which *is* configurable (`.env.example:169-170` recommends raising it), so the two can be tuned out of alignment with no way to adjust the agent side. | `crates/buzz-acp/src/lib.rs:632`; `crates/buzz-agent/src/agent.rs:129`, `:455`; `lib.rs:164`, `:742` |
| CFG-2d-18 | **`DedupMode`'s documented default contradicts the CLI default.** `queue.rs:11-14` documents "Drop (default)"; the CLI default is `queue` (`config.rs:344`). And `DedupMode::Drop` loses batches on three paths including steers (`pool.rs:2971-2977`, `lib.rs:2196-2199`). | `crates/buzz-acp/src/queue.rs:11-14` vs `config.rs:344` |
| CFG-2d-19 | **`BUZZ_ACP_SETUP_PAYLOAD` diverts startup and the pool never starts**, and `setup_mode` silently ignores reminder events: it uses only `[STREAM_MESSAGE, WORKFLOW_APPROVAL_REQUESTED]` (`setup_mode.rs:526`) where `config.rs:1161-1165` uses a third kind (`STREAM_REMINDER`). The variable is read at `setup_mode.rs:214` (declared `:83`) and diverts at `lib.rs:1298-1303`. | `crates/buzz-acp/src/setup_mode.rs:83`, `:214`, `:526`; `config.rs:1161-1165`; `lib.rs:1298-1303` |
| CFG-2d-20 | **A new undocumented relay-side owner cap, surfaced by the post-sync reconciliation.** `BUZZ_MAX_COMMUNITIES_PER_OWNER` is read at `relay_members.rs:391` with default `MAX_COMMUNITIES_PER_OWNER = 3` (`:379`), cached in a `OnceLock` for the process lifetime (`:388-395`), and absent from both `.env.example` and `deploy/compose/.env.example`. Neither of the two owner-cap tests reads the effective limit — one loops on the literal `3` (`crates/buzz-db/src/lib.rs:4863`), the other on the constant (`crates/buzz-relay/src/api/operator.rs:1086`) — so setting the variable makes both fail spuriously and the knob has no end-to-end coverage. | `crates/buzz-db/src/relay_members.rs:379`, `:388-395`, `:391` |

Positive controls in 2d configuration: `buzz-agent`'s env parsers are pure functions taking
`Option<&str>` specifically so they are testable without mutating process state, with the impure
wrappers separated (`parse_thinking_effort` `config.rs:622`, `parse_openai_api` `:1022-1023`,
`parse_hook_servers` `:1091-1092`) — and each says so in its doc comment. `parse_env`
(`config.rs:1049-1056`) returns the default only when a variable is *absent*: a present but
unparseable value is a hard startup error prefixed `config: {key}: `, applied consistently at all
19 call sites (`config.rs:807-832` — one more than before the sync, since the new
`prefer_mesh_for_auto` field goes through the same helper at `:807`). `MCP_HOOK_SERVERS`
deliberately declines to treat a mixed `*,foo` as a wildcard so a typo cannot silently widen scope
(`config.rs:1104-1107`), with ten tests. And `buzz-cli`'s `env_duration_secs` refuses `0` on
purpose so timeouts can never be turned off (`client.rs:139-150`).

## Environment Variables

Grouped by concern. "Required" means the system will not function correctly without it in
a non-default deployment. Full canonical reference: [`.env.example`](../../.env.example).

### Core infrastructure

| Variable | Purpose | Module | Required | Default | Sensitive |
|---|---|---|---|---|---|
| `DATABASE_URL` | Postgres connection string | buzz-db, relay, admin | yes | `postgres://buzz:buzz_dev@localhost:5432/buzz` | yes |
| `READ_DATABASE_URL` | Optional read-replica DSN | buzz-db | no | unset (reads go to writer) | yes |
| `PGHOST`/`PGPORT`/`PGUSER`/`PGPASSWORD`/`PGDATABASE` | libpq-style overrides (scripts, CI) | scripts/CI | no | localhost/5432/buzz/buzz_dev/buzz | yes |
| `REDIS_URL` | Redis connection string | buzz-pubsub | yes | `redis://localhost:6379` | no |
| `BUZZ_REDIS_POOL_SIZE` | Shared Redis pool size | relay | no | 16 | no |
| `BUZZ_AUTO_MIGRATE` | Apply embedded migrations at boot | relay | no | (boot-time gate) | no |
| `BUZZ_TEST_DATABASE_URL` | Test-only DB DSN | tests | no | unset | yes |

### Relay server

| Variable | Purpose | Module | Required | Default | Sensitive |
|---|---|---|---|---|---|
| `BUZZ_BIND_ADDR` | App bind address (WS + REST) | relay | no | `0.0.0.0:3000` | no |
| `RELAY_URL` | Public WS URL used in NIP-42 challenges | relay | yes (prod) | `ws://localhost:3000` | no |
| `BUZZ_RELAY_BASE_URL` | Public base URL for links | relay | no | derived | no |
| `BUZZ_HEALTH_PORT` | Liveness/readiness port | relay | no | 8080 | no |
| `BUZZ_METRICS_PORT` | Prometheus metrics port | relay | no | 9102 | no |
| `BUZZ_RELAY_PRIVATE_KEY` | Stable relay signing key | relay | no (dev) / yes (prod) | ephemeral | **yes** |
| `BUZZ_RELAY_PUBKEY` | Relay identity pubkey | relay/clients | no | derived | no |
| `RELAY_OWNER_PUBKEY` | Relay owner identity | relay | no | unset | no |
| `BUZZ_MAX_CONNECTIONS` | Connection semaphore capacity | relay | no | (code default) | no |
| `BUZZ_MAX_CONCURRENT_HANDLERS` | EVENT/REQ handler semaphore | relay | no | 1024 | no |
| `BUZZ_MAX_FRAME_BYTES` | Max WS frame size | relay | no | 65536 | no |
| `BUZZ_SEND_BUFFER` | Per-connection send buffer | relay | no | (code default) | no |
| `BUZZ_SLOW_CLIENT_GRACE_LIMIT` | Full-buffer strikes before disconnect | relay | no | 3 | no |
| `BUZZ_CORS_ORIGINS` | Allowed CORS origins | relay | no | (code default) | no |
| `BUZZ_WEB_DIR` | Serve web SPA from directory | relay | no | unset (`/srv/buzz/web` in image) | no |
| `BUZZ_ADMIN_HOST` / `BUZZ_ADMIN_WEB_DIR` | Admin dashboard host + bundle | relay | no | unset (inert) | no |
| `BUZZ_SERVE_GIT_WEB_GUI` | Enable repo-browser web routes | relay | no | false | no |
| `BUZZ_REAPER_INTERVAL_SECS` | Ephemeral-channel reaper cadence | relay | no | 60 | no |
| `BUZZ_EPHEMERAL_TTL_OVERRIDE` | Force TTL for ephemeral channels | relay | no | unset | no |
| `BUZZ_RECONCILE_CHANNELS` | Emit missing channel discovery events at boot | relay | no | false | no |
| `BUZZ_NIP43_RECONCILE_INTERVAL_SECS` | Membership roster reconcile cadence | relay | no | (code default) | no |
| `BUZZ_COMMUNITY_REVALIDATE_INTERVAL_SECS` | Community/host map refresh | relay | no | (code default) | no |
| `BUZZ_HUDDLE_AUDIO_AVAILABLE` | Advertise huddle audio capability | relay | no | (code default) | no |

### Authentication & authorization

| Variable | Purpose | Module | Required | Default | Sensitive |
|---|---|---|---|---|---|
| `BUZZ_PRIVATE_KEY` | Client/agent Nostr identity key | acp, cli, agent, desktop | yes (agents) | unset | **yes** |
| `NOSTR_PRIVATE_KEY` | Alternate key var (git tooling) | git-sign-nostr | no | unset | **yes** |
| `BUZZ_API_TOKEN` / `BUZZ_AUTH_TAG` | API token + auth tag (harness-injected) | cli, acp | no | unset | **yes** |
| `BUZZ_REQUIRE_AUTH_TOKEN` | Require token auth | relay | no | (code default) | no |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | Gate access on relay roster membership | relay | no | false | no |
| `BUZZ_ALLOW_NIP_OA_AUTH` | Allow NIP-OA auth path | relay | no | false | no |
| `BUZZ_PUBKEY_ALLOWLIST` | Restrict to listed pubkeys | relay | no | unset | no |
| `BUZZ_TERMS_OF_SERVICE_MARKDOWN` / `BUZZ_PRIVACY_POLICY_MARKDOWN` / `BUZZ_AGE_ATTESTATION_REQUIRED` | Join-policy documents + age gate | relay | no | unset (policy off) | no |
| `BUZZ_TEST_OWNER_PRIVATE_KEY` / `BENCH_PRIVATE_KEY` | Test/benchmark identities | tests, benchmarks | no | unset | **yes** |

### Rate limiting (Redis-backed admission)

| Variable | Purpose | Module | Required | Default | Sensitive |
|---|---|---|---|---|---|
| `BUZZ_RATE_LIMIT_HUMAN_MESSAGES_PER_MIN` | Human message cap | relay | no | 60 | no |
| `BUZZ_RATE_LIMIT_HUMAN_API_CALLS_PER_MIN` | Human API cap | relay | no | 300 | no |
| `BUZZ_RATE_LIMIT_HUMAN_WS_EVENTS_PER_SEC` | Human WS event cap | relay | no | 10 | no |
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_MESSAGES_PER_MIN` | Agent (standard) message cap | relay | no | 120 | no |
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_API_CALLS_PER_MIN` | Agent (standard) API cap | relay | no | 600 | no |
| `BUZZ_RATE_LIMIT_AGENT_ELEVATED_MESSAGES_PER_MIN` | Agent (elevated) cap | relay | no | 300 | no |
| `BUZZ_RATE_LIMIT_AGENT_PLATFORM_MESSAGES_PER_MIN` | Agent (platform) cap | relay | no | 600 | no |

> ⚠️ `ARCHITECTURE.md` §9 states rate limiting is **not enforced** (only a test stub
> `AlwaysAllowRateLimiter` exists), while `.env.example` documents these as
> "shared Redis-backed admission limits" and CI sets them to 100000. To be verified
> against `buzz-auth`/`buzz-relay` code in Phase 2 — flagged in `debt.md`.

### Media (Blossom / S3)

| Variable | Purpose | Module | Required | Default | Sensitive |
|---|---|---|---|---|---|
| `BUZZ_S3_BUCKET` / `BUZZ_S3_ENDPOINT` / `BUZZ_S3_REGION` | Media object store target | buzz-media | yes (prod) | MinIO local | no |
| `BUZZ_S3_ACCESS_KEY` / `BUZZ_S3_SECRET_KEY` | S3 credentials | buzz-media | yes (prod) | dev creds | **yes** |
| `AWS_REGION` | AWS region fallback | buzz-media | no | unset | no |
| `BUZZ_MEDIA_BASE_URL` | Public media URL base | relay | no | derived | no |
| `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS` | Global upload concurrency | relay | no | 8 | no |
| `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS_PER_PUBKEY` | Per-identity upload concurrency | relay | no | 2 | no |
| `BUZZ_MEDIA_UPLOADS_PER_MINUTE` | Upload rate cap | relay | no | 30 | no |
| `BUZZ_REQUIRE_MEDIA_GET_AUTH` (alias `BUZZ_REQUIRE_MEDIA_READ_AUTH`) | Require Blossom auth on media GET/HEAD | relay | no | false | no |
| `BUZZ_MAX_FILE_BYTES` / `BUZZ_MAX_IMAGE_BYTES` / `BUZZ_MAX_VIDEO_BYTES` / `BUZZ_MAX_GIF_BYTES` | Per-type size caps | relay | no | (code defaults) | no |
| `BUZZ_MEDIA_UPLOAD_IP_HEADER` / `BUZZ_MEDIA_UPLOAD_PORT_HEADER` | Trusted client-IP headers | relay | no | unset | no |
| `BUZZ_MEDIA_UPLOAD_RECORDS` | Upload record retention/telemetry | relay | no | (code default) | no |
| `BUZZ_STORAGE_SWEEP_INTERVAL_SECS` / `_MAX_OBJECTS` / `_TIMEOUT_SECS` / `BUZZ_STORAGE_METRICS` | Orphan-object sweeper | relay | no | (code defaults) | no |

### Git hosting (NIP-34)

| Variable | Purpose | Module | Required | Default | Sensitive |
|---|---|---|---|---|---|
| `BUZZ_GIT_REPO_PATH` | Ephemeral workspace root | relay | no | `./repos` | no |
| `BUZZ_GIT_MAX_PACK_BYTES` / `BUZZ_GIT_MAX_REPO_BYTES` | Size limits | relay | no | 500MB / 1GB | no |
| `BUZZ_GIT_MAX_REPOS_PER_PUBKEY` / `BUZZ_GIT_MAX_CONCURRENT_OPS` | Per-identity + global caps | relay | no | (code defaults) | no |
| `BUZZ_GIT_PACK_CACHE_PATH` / `_MAX_BYTES` / `_MAX_CONCURRENT_POPULATIONS` | Immutable pack cache | relay | no | `./repos/.pack-cache`, 5GB, 2 | no |
| `BUZZ_GIT_S3_BUCKET` / `_ENDPOINT` / `_REGION` / `_ACCESS_KEY` / `_SECRET_KEY` | Git-on-object-storage backend | relay | no | unset | **yes** (keys) |
| `BUZZ_GIT_HOOK_HMAC_SECRET` | HMAC for internal git policy hook | relay | yes (git enabled) | unset | **yes** |
| `BUZZ_GIT_PROBE_WRITERS` / `BUZZ_GIT_PROBE_ROUNDS` / `BUZZ_GIT_S3_PROBE` / `BUZZ_GIT_CONFORMANCE_PROBE` | Boot deployment probes | relay | no | (code defaults) | no |
| `GIT_CREDENTIAL_NOSTR_BIN` / `GIT_BASH` / `GIT_CONFIG_COUNT` | Git tooling resolution | dev-mcp, tests | no | resolved | no |

### Agent harness (buzz-acp)

Full list in `.env.example`. Key vars: `BUZZ_RELAY_URL`, `BUZZ_ACP_AGENT_COMMAND`
(default `goose`), `BUZZ_ACP_AGENT_ARGS`, `BUZZ_ACP_MCP_COMMAND`, `BUZZ_ACP_AGENTS`
(1–32, default 1), `BUZZ_ACP_MODEL`, `BUZZ_ACP_TURN_TIMEOUT` (320s),
`BUZZ_ACP_MAX_TURNS_PER_SESSION` (0=off), `BUZZ_ACP_SYSTEM_PROMPT[_FILE]`,
`BUZZ_ACP_INITIAL_MESSAGE`, `BUZZ_ACP_HEARTBEAT_INTERVAL` (0 or ≥10) +
`BUZZ_ACP_HEARTBEAT_PROMPT[_FILE]`, `BUZZ_ACP_SUBSCRIBE`
(`mentions`|`all`|`config`), `BUZZ_ACP_KINDS`, `BUZZ_ACP_CHANNELS`,
`BUZZ_ACP_NO_MENTION_FILTER`, `BUZZ_ACP_CONFIG`, `BUZZ_ACP_DEDUP` (`queue`|`drop`),
`BUZZ_ACP_NO_IGNORE_SELF`, `BUZZ_ACP_CONTEXT_MESSAGE_LIMIT` (0–100, default 12),
`BUZZ_ACP_NO_PRESENCE`, `BUZZ_ACP_NO_TYPING`, `BUZZ_ACP_EVENT_BUFFER` (256),
`BUZZ_ACP_ALLOWED_CHANNEL_ADD_POLICIES`. Legacy alias: `BUZZ_ACP_PRIVATE_KEY` →
`BUZZ_PRIVATE_KEY`. Agent shell: `BUZZ_SHELL` (Windows requires Git Bash).

### Push gateway

| Variable | Purpose | Module | Required | Default | Sensitive |
|---|---|---|---|---|---|
| `BUZZ_PUSH_GATEWAY_DELIVERY_URL` | Gateway delivery endpoint | relay | no | unset | no |
| `BUZZ_PUSH_GATEWAY_TIMEOUT_MS` | Delivery timeout | relay | no | (code default) | no |
| `BUZZ_PUSH_EXECUTOR_KEY_ID` | Signing key id for push executor | push-gateway | no | unset | **yes** |
| `BUZZ_PUSH_RUNTIME_DATABASE_ROLE` | Least-privilege DB role | push-gateway | no | unset | no |

### Mesh (opt-in `mesh-llm` / relay mesh)

`BUZZ_MESH`, `BUZZ_MESH_BIND_ADDR`, `BUZZ_MESH_ADVERTISE_ADDR`, `BUZZ_MESH_DEMO_ECHO`,
`MESH_LLM_NATIVE_RUNTIME_CACHE_DIR`, `MESH_ROLE`, `MESH_STACK_ROLE`, `MESH_STACK_SIZE`,
`MESH_OPENAI_BASE`, `MESH_SMOKE_MODEL`, `MESH_E2E_MODEL`.

### Pairing (NIP-AB)

`BUZZ_PAIRING_RELAY_URL`, `BUZZ_PAIR_RELAY_BIND_ADDR`, `BUZZ_UDS_PATH`.

### Observability & misc

`RUST_LOG` (default `buzz_relay=debug,buzz_db=debug,buzz_auth=debug,buzz_pubsub=debug,tower_http=debug`),
`OTEL_EXPORTER_OTLP_ENDPOINT` (optional), `BUZZ_AUDIT_ENABLED`,
`BUZZ_POOL_METRICS_INTERVAL_SECS`, `BUZZ_USAGE_METRICS_INTERVAL_SECS`,
`BUZZ_USAGE_METRICS_IDLE_TIMEOUT_SECS`, `BUZZ_USAGE_METRICS_PER_COMMUNITY`,
`BUZZ_CONFORMANCE_UPDATE`, `BUZZ_MANAGED_AGENT_START_NONCE`,
`BUZZ_E2E_GIT_COMMUNITY_ID`, `DATABRICKS_HOST`, `CODEX_CONFIG`/`CODEX_HOME`,
`GOOSE_PATH_ROOT`, `FAKE_MCP_*` (test doubles).

### Stale / drifting variables

| Variable | Status |
|---|---|
| `TYPESENSE_API_KEY`, `TYPESENSE_URL` | ⚠️ Documented in `.env.example` but search moved to Postgres FTS (`buzz-search`); no Typesense service in `docker-compose.yml` |
| Keycloak (`KEYCLOAK_ADMIN`, `KC_DB`) | ⚠️ Set only in `docker-compose.yml`; no documented relay integration |
| `BUZZ_REQUIRE_MEDIA_READ_AUTH` | Legacy alias retained for `BUZZ_REQUIRE_MEDIA_GET_AUTH` |
| `BUZZ_ACP_PRIVATE_KEY` | Legacy alias retained for `BUZZ_PRIVATE_KEY` |
| `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS` | `sprout`-prefixed legacy naming (set in CI) |

## Configuration Files

| File | Purpose | Format | Environment-Specific |
|---|---|---|---|
| `.env.example` → `.env` | Canonical env template (dev defaults) | dotenv | yes — copied and edited per developer |
| `Cargo.toml` (workspace) | Rust deps, profiles (`ci`, `sprig`), dep patch | TOML | no |
| `rust-toolchain.toml` | Pinned Rust 1.95.0 | TOML | no |
| `Justfile` | Task runner; `set dotenv-load := true` | just | flags via `mesh=1`, `fresh=1` |
| `docker-compose.yml` | Local dev stack | YAML | dev only |
| `docker-compose.harness.yml` | Benchmark harness stack | YAML | benchmark |
| `deploy/compose/{compose,compose.dev,compose.caddy}.yml` + `.env.example` | Self-host compose + TLS | YAML/dotenv | yes |
| `deploy/charts/buzz/values.yaml` (+ `values.schema.json`) | Relay Helm values | YAML/JSON-Schema | yes — per cluster |
| `deploy/charts/buzz-push-gateway/values{,-production}.yaml` | Gateway Helm values | YAML | yes — prod overlay |
| `deploy/local/quickstart-ha-values.yaml` | Local HA quickstart | YAML | local |
| `migrations/*.sql` (24) | Forward-only schema migrations | SQL | no |
| `schema/schema.sql` | Desired-state schema (pgschema) | SQL | no |
| `prometheus.yml` | Scrape config | YAML | dev |
| `biome.json` | JS/TS lint + format | JSON | no |
| `deny.toml` | cargo-deny dependency policy | TOML | no |
| `lefthook.yml` | Pre-commit / pre-push hooks | YAML | no |
| `renovate.json` | Dependency update policy | JSON | no |
| `preview-features.json` | Preview feature registry | JSON | per-platform |
| `pnpm-workspace.yaml` | JS workspace, overrides, patches | YAML | no |
| `desktop/src-tauri/tauri.conf.json` | Tauri app config, sidecars, updater | JSON | via `BUZZ_TAURI_CONFIG` per instance |
| `mobile/pubspec.yaml` | Flutter deps | YAML | no |
| `buzz-acp.toml` (optional) | Rule-based ACP subscriptions | TOML | per agent |
| `ct.yaml` | Helm chart-testing config | YAML | CI |

## Feature Flags

| Flag | Purpose | Default | Location | Mechanism |
|---|---|---|---|---|
| `workflows` | YAML automations with approval gates | preview | `preview-features.json` | Declarative registry (desktop) |
| `projects` | Git repository browser | preview | `preview-features.json` | Declarative registry |
| `pulse` | Activity feed | preview | `preview-features.json` | Declarative registry |
| `forum` | Forum-style channels | preview | `preview-features.json` | Declarative registry |
| `agentManagedProfiles` | Agents manage own name/avatar | preview | `preview-features.json` | Declarative registry |
| `mesh-llm` | On-device LLM mesh compute | off | Cargo feature; `just mesh=1` | Compile-time cargo feature |
| `dev` (buzz-auth) | Dev-only key derivation | off | `buzz-auth` `#[cfg(feature="dev")]` | Compile-time — **must stay off in prod** |
| `test-utils` | Test stubs (e.g. AlwaysAllowRateLimiter) | off | `buzz-auth` | Compile-time |
| `BUZZ_BUILD_OBSERVER_ARCHIVE_DEFAULT` | Observer archive default on | false | `desktop/src-tauri/build.rs` | Compile-time env flag (CI-verified both states) |
| `BUZZ_BUILD_AUTO_CONNECT_DEFAULT_RELAY` | Auto-connect to default relay | false | `desktop/src-tauri/build.rs` | Compile-time env flag |
| `BUZZ_BUILD_AGENT_METRIC_ARCHIVE_DEFAULT` | Agent metric archive default | false | `build.rs` | Compile-time |
| `BUZZ_BUILD_AGENT_ENV` / `BUZZ_BUILD_BUZZ_AGENT_MODEL` / `_PROVIDER` / `BUZZ_BUILD_RELAY_RECONNECT_CMD` | Internal-build agent wiring | unset | `build.rs` | Compile-time |
| `BUZZ_SERVE_GIT_WEB_GUI` | Repo-browser web routes | false | relay env | Runtime env var |
| `BUZZ_REQUIRE_MEDIA_GET_AUTH` | Media read auth rollout gate | false | relay env | Runtime env var |
| `VITE_BUZZ_FORCE_FRESH_ONBOARDING` | Replay onboarding each launch (dev) | off | Vite env | Build-time frontend flag |
| `VITE_NOSTR_BIND_PREVIEW` / `VITE_SHOW_TRANSCRIPT_ACP_SOURCE` | Frontend preview toggles | off | Vite env | Build-time frontend flag |

## Environments Detected

| Environment | Config Source | Key Differences |
|---|---|---|
| local dev | `.env` (from `.env.example`), `docker-compose.yml`, `just dev` | Local Postgres/Redis/MinIO, debug `RUST_LOG`, `ws://localhost:3000`, dev keyring service, ephemeral relay key |
| desktop standalone | `just desktop-standalone` (no relay/DB) | No infra; app prompts for a community; isolated keyring per instance |
| CI | `.github/workflows/ci.yml` env blocks | `BUZZ_REQUIRE_AUTH_TOKEN=false`, rate limits at 100000, `BUZZ_RECONCILE_CHANNELS=true`, pgschema-applied schema, pre-seeded `localhost:3000` community |
| staging | `just staging` → `BUZZ_RELAY_URL=wss://sprout-oss.stage.blox.sqprod.co` | Release-mode sidecars, internal staging relay |
| production | `just production` → `BUZZ_RELAY_URL=wss://buzz.block.builderlab.xyz` | Release-mode sidecars, production relay |
| container/prod relay | `Dockerfile` ENV + Helm values | `BUZZ_WEB_DIR=/srv/buzz/web`, `BUZZ_ADMIN_WEB_DIR=/srv/buzz/admin-web`, ports 3000/8080/9102, non-root user |
| benchmark | `docker-compose.harness.yml`, `BENCH_PRIVATE_KEY` | Isolated compose project `buzz-benchmark` |

## Secrets Management

- **Approach**: environment variables (`.env` locally, Helm values/platform secrets in
  deployment). No Vault/AWS Secrets Manager integration in this repo. The optional
  GitHub MCP token path is documented as external/org-specific.
- **Desktop identity keys**: stored in the **OS keychain** via
  `desktop/src-tauri/src/secret_store.rs` and `app_state_keyring.rs`; dev instances use
  a per-instance service name (`BUZZ_DEV_KEYRING_SERVICE`).
- **Cloud credentials**: S3 access via `BUZZ_S3_*`/`BUZZ_GIT_S3_*`, or AWS EKS **Pod
  Identity** (`AWS_CONTAINER_CREDENTIALS_FULL_URI` + token file) through the patched
  `aws-creds` fork pinned in `Cargo.toml`.
- **In-repo test keys**: `Justfile` `mesh-dev-fresh` hardcodes a public test
  `BUZZ_PRIVATE_KEY` and a `0x…01` relay key, explicitly labelled local-only — must
  never be pointed at staging/prod (noted in `security.md`).
- **Rotation strategy**: none automated in-repo.
- **Access pattern**: loaded at process startup (dotenv + `env::var`); desktop keys read
  on demand from the keychain.

## Configuration Patterns

- **Loading**: `just` loads `.env` (`set dotenv-load := true`); Rust reads via
  `std::env::var` in per-crate config modules (e.g. `buzz-acp/src/config.rs`, 1,903 LOC
  covering CLI flags + env + TOML precedence). Tauri reads env plus compile-time
  `BUZZ_BUILD_*` values injected by `build.rs`.
- **Validation**: partial — documented constraints (positive integers for rate limits,
  `BUZZ_ACP_AGENTS` 1–32, heartbeat 0 or ≥10, context limit 0–100) suggest parse-time
  validation in `buzz-acp`; relay-wide startup schema validation not confirmed in this
  pass (to verify in Phase 2). Helm values are validated by `values.schema.json`.
- **Overrides**: precedence is CLI flag > env var > TOML config file > code default
  (documented for `buzz-acp`); env overrides `.env` defaults; per-worktree dev ports and
  identities are derived by `scripts/instance-env.sh`.
- **Location**: `.env.example` (canonical documentation), `crates/buzz-acp/src/config.rs`
  (richest config surface), `desktop/src-tauri/build.rs` (compile-time flags).

<!-- Per-module configuration findings are appended during Phase 2. -->

---

# Phase 2 — Module Findings

## Module: buzz-core (`crates/buzz-core`)

### Aspect: Configuration

---

### 1. Environment variables

**None.** A recursive search of `crates/buzz-core/src` for `env::var` and `std::env` returns zero matches. The crate performs no environment reads, no file reads, and no config parsing — consistent with the zero-I/O charter stated at `crates/buzz-core/Cargo.toml:29`.

Runtime configuration for anything built on this crate (relay URL, keys, DB, Redis) is read by the consuming crates, not here.

---

### 2. Cargo features

| Feature | Declared | Default? | What it gates | Enabled by |
|---------|----------|----------|---------------|-----------|
| `test-utils` | `crates/buzz-core/Cargo.toml:10-11` (`test-utils = []`) | no (no `default` key exists) | `pub mod test_helpers` — `make_event` (`src/lib.rs:55`), `make_event_with_keys` (`:64`), `make_stored_event` (`:72`); the gate is `#[cfg(any(test, feature = "test-utils"))]` at `src/lib.rs:47` | `crates/buzz-relay/Cargo.toml:89`, inside that crate's `[dev-dependencies]` (`:86`) |

No other `#[cfg(feature = …)]` appears anywhere in `src/` — the only conditional compilation in the crate is `#[cfg(test)]` and the single `#[cfg(any(test, feature = "test-utils"))]`.

Notably, this crate has **no `dev` feature** (unlike `buzz-auth`, whose `dev` feature is switched on from `crates/buzz-relay/Cargo.toml:84`/`:90`).

---

### 3. Package metadata (all inherited from the workspace)

| Key | Value / source | file:line |
|-----|----------------|-----------|
| `name` | `buzz-core` | `crates/buzz-core/Cargo.toml:2` |
| `version` | `version.workspace = true` → `0.1.0` | `Cargo.toml:3`; root `Cargo.toml:35` |
| `edition` | `edition.workspace = true` → `2021` | `Cargo.toml:4`; root `Cargo.toml:36` |
| `rust-version` | `rust-version.workspace = true` → `1.88.0` | `Cargo.toml:5`; root `Cargo.toml:37` |
| `license` | `license.workspace = true` → `Apache-2.0` | `Cargo.toml:6`; root `Cargo.toml:38` |
| `repository` | `repository.workspace = true` → `https://github.com/block/sprout` | `Cargo.toml:7`; root `Cargo.toml:39` |
| `description` | "Core types, event verification, and filter matching for Buzz" | `Cargo.toml:8` |

Dependency versions are all workspace-inherited except one local pin: `percent-encoding = "2.3"` (`crates/buzz-core/Cargo.toml:26`).

---

### 4. Compile-time configuration (lints and assertions)

| Setting | Effect | file:line |
|---------|--------|-----------|
| `#![deny(unsafe_code)]` | any `unsafe` in the crate is a hard compile error | `src/lib.rs:1` |
| `#![warn(missing_docs)]` | undocumented public items warn | `src/lib.rs:2` |
| `const _: () = assert!(...)` × 25 | kind-range and u16-fit invariants are enforced at compile time; violating one fails the build | `src/kind.rs:783-820` |

No `[lints]` table exists in this crate's manifest and no `[workspace.lints]` exists in the root manifest, so lint configuration is entirely via these crate-level attributes.

---

### 5. Tunable constants (the crate's de-facto configuration surface)

These are compile-time constants, not runtime config. Changing one requires a rebuild of every dependent crate.

| Constant | Value | Governs | file:line |
|---|---|---|---|
| `EPHEMERAL_KIND_MIN` / `MAX` | 20000 / 29999 | ephemeral classification | `src/kind.rs:397`, `:399` |
| `PARAM_REPLACEABLE_KIND_MIN` / `MAX` | 30000 / 39999 | NIP-33 addressable classification | `src/kind.rs:392`, `:394` |
| `NIP44_MIN_CONTENT_LEN` | 132 | minimum accepted NIP-44 ciphertext length | `src/observer.rs:21` |
| `NIP44_MAX_CONTENT_LEN` | 87_472 | maximum accepted NIP-44 ciphertext length | `src/observer.rs:23` |
| `OBSERVER_MAX_PLAINTEXT_LEN` | 65_535 | observer plaintext cap | `src/observer.rs:25` |
| `OBSERVER_AGENT_TAG` / `OBSERVER_FRAME_TAG` | `"agent"` / `"frame"` | observer frame tag names | `src/observer.rs:13`, `:15` |
| `OBSERVER_FRAME_TELEMETRY` / `OBSERVER_FRAME_CONTROL` | `"telemetry"` / `"control"` | observer frame direction values | `src/observer.rs:17`, `:19` |
| `CORE_SLUG` | `"core"` | reserved engram slug | `src/engram.rs:20` |
| `D_TAG_DOMAIN` | `b"agent-memory/v1/d-tag"` | HMAC domain separator; doc says future revisions MUST change it | `src/engram.rs:22-24` |
| `NIP44_PLAINTEXT_MAX` | 65_535 | engram body cap | `src/engram.rs:28` |
| `SLUG_MAX_LEN` | 255 | engram slug byte cap (per-segment cap of 64 is an inline literal at `src/engram.rs:98`) | `src/engram.rs:31` |
| `MEM_PREFIX` (private) | `"mem/"` | engram slug namespace | `src/engram.rs:34` |
| `MAX_PROTECTION_RULES` | 50 | `buzz-protect` rules per repo | `src/git_perms.rs:19` |
| `MAX_PATTERN_LENGTH` | 256 | ref-pattern character cap | `src/git_perms.rs:21` |
| `MAX_WILDCARDS_PER_PATTERN` | 3 | wildcard segments per pattern | `src/git_perms.rs:23` |
| `ZERO_OID` (private, fn-local) | 40 `0` chars | create/delete detection | `src/git_perms.rs:213` |
| `DEFAULT_TIMEOUT` (private) | `Duration::from_secs(120)` | pairing session lifetime | `src/pairing/session.rs:43` |
| `PAIRING_KIND` (private) | `KIND_PAIRING as u16` = 24134 | pairing event kind | `src/pairing/session.rs:46` |
| `INFO_SESSION_ID` / `INFO_SAS` / `INFO_TRANSCRIPT` (private) | `"nostr-pair-session-id"`, `"nostr-pair-sas-v1"`, `"nostr-pair-transcript-v1"` | HKDF domain separation | `src/pairing/crypto.rs:23-25` |
| SAS modulus (inline literal) | `1_000_000` | 6-digit SAS space | `src/pairing/crypto.rs:73` |
| QR URI length cap (inline literal) | 2048 | max scanned URI length | `src/pairing/qr.rs:106` |
| pairing content-length window (inline literals) | `132..=87472` | inbound ciphertext gate — duplicates `observer::NIP44_MIN/MAX_CONTENT_LEN` rather than referencing them | `src/pairing/session.rs:595` |
| pairing plaintext cap (inline literal) | `65_535` | decrypted plaintext gate — duplicates `OBSERVER_MAX_PLAINTEXT_LEN` / `NIP44_PLAINTEXT_MAX` | `src/pairing/session.rs:609` |
| jitter window (inline literal) | `% 31` → 0–30 s | `created_at` metadata jitter | `src/pairing/session.rs:578` |

Non-configurable by design: nothing in the crate reads these from the environment, and there is no builder or options struct that would let a consumer override them at runtime.


## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: Configuration

This crate has **no runtime configuration surface**. Findings are recorded so the
absence is explicit rather than assumed.

---

### 1. Environment variables

| Variable | Read where | Notes |
|---|---|---|
| — | — | A search for `env::var`, `env!`, `option_env!`, and `std::env` across `crates/buzz-sdk/src/` returns **no matches**. No SDK behavior is environment-dependent. |

The only `std::env` use in the crate is `std::env::args()` in the example binary,
which reads positional CLI arguments, not environment variables
(`crates/buzz-sdk/examples/compute_auth_tag.rs:12`).

Note for contrast: the agent-facing environment variables documented for Buzz
(`BUZZ_RELAY_URL`, `BUZZ_PRIVATE_KEY`, `BUZZ_AUTH_TAG`) are consumed by
`buzz-cli` / the ACP harness, **not** by this crate — `buzz-sdk` has no relay
URL, no key source, and no transport.

---

### 2. Cargo features

| Feature | Gates | Declared at |
|---|---|---|
| — | — | `crates/buzz-sdk/Cargo.toml` has **no `[features]` section** (`Cargo.toml:1-17`) |

No `#[cfg(feature = ...)]` attribute exists anywhere in `src/` or `examples/`.
The only conditional compilation is `#[cfg(test)]` on the three test modules
(`builders.rs:1825`, `mentions.rs:389`, `nip_oa.rs:301`).

Feature selection the crate *inherits* (workspace-pinned, not switchable here):

| Dependency | Features enabled by the workspace | Declared at |
|---|---|---|
| `nostr` `0.44` | `nip44`, `nip98` | `Cargo.toml:61` (root) |
| `uuid` `1` | `v4`, `serde` | `Cargo.toml:89` (root) |
| `serde` `1` | `derive` | `Cargo.toml:64` (root) |

All six dependencies are declared as `{ workspace = true }`, so versions and
feature sets are centrally controlled (`crates/buzz-sdk/Cargo.toml:10-16`).

---

### 3. Package metadata

Everything except `name` and `description` is inherited from the workspace
(`crates/buzz-sdk/Cargo.toml:1-8`):

| Key | Value |
|---|---|
| `name` | `buzz-sdk` |
| `version` / `edition` / `rust-version` / `license` / `repository` | `.workspace = true` |
| `description` | `"Typed Nostr event builders for Buzz operations"` |

There is no `[[bin]]`, no `[lib]` override, no `build.rs`, and no
`[dev-dependencies]` section.

---

### 4. Compile-time behavior switches

| Switch | Effect | File:line |
|---|---|---|
| `#![deny(unsafe_code)]` | build fails on any `unsafe` | `crates/buzz-sdk/src/lib.rs:1` |
| `#![warn(missing_docs)]` | undocumented public items warn | `crates/buzz-sdk/src/lib.rs:2` |

---

### 5. Effective "configuration" — hardcoded limits

Because nothing is configurable at runtime, the crate's tunable behavior is
fixed in constants and inline literals. Consumers cannot override them.

| Limit | Value | File:line |
|---|---|---|
| `MENTION_CAP` | 50 mention p-tags | `mentions.rs:38` |
| `MAX_CONTACTS` | 10 000 NIP-02 contacts | `builders.rs:751` |
| `MAX_REASON_BYTES` | 64 UTF-8 bytes | `builders.rs:1704` |
| `CUSTOM_EMOJI_SET_D_TAG` | `"buzz:custom-emoji"` | `builders.rs:503` |
| Default content cap | 64 KiB (13 call sites) | `builders.rs:227`, `284`, `299`, `383`, `742`, `1087`, `1227`, `1335`, `1421`, `1468`, `1486`, `1795`, `1816` |
| Diff / patch content cap | 60 KiB | `builders.rs:314`, `1023` |
| Reaction emoji cap | 64 chars | `builders.rs:467` |
| Emoji shortcode cap | 64 bytes | `builders.rs:136` |
| Emoji / contact URL cap | 2048 bytes | `builders.rs:157`, `787` |
| Repo name / description / clone URL / relay caps | 128 / 1024 / 512 / 256 chars | `builders.rs:845`, `856`, `875`, `907` |
| Repo clone-URL and relay counts | 5 / 10 | `builders.rs:868`, `900` |
| Subject cap (issue, PR) | 256 chars | `builders.rs:1092`, `1340` |
| Petname cap | 256 bytes | `builders.rs:794` |
| DM participants | 1–8 | `builders.rs:1546` |
| NIP-OA `kind` clause range | 0–65535 | `nip_oa.rs:63` |
| NIP-OA `created_at` clause range | 0–4294967295 | `nip_oa.rs:65`, `67` |


## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: Configuration

### 1. Environment variables read by this crate

**None.** There is no `std::env::var`, `env::var_os`, `envy`, or `dotenv` usage anywhere in
`crates/buzz-persona/src/` or `crates/buzz-persona/tests/` (verified by grep — the only
`std::env` occurrences in the crate are prose inside
`crates/buzz-persona/PERSONA_PACK_SPEC.md`).

This is a stated design property, not an accident:

- `crates/buzz-persona/src/resolve.rs:11` — "**Pure**: no env access, no network, no side effects."
- `crates/buzz-persona/src/resolve.rs:359-364` — "Pure function — does NOT read the current
  process env. ACP is responsible for filtering based on operator precedence (level 1): if
  the operator already set an env var, ACP skips injection so the operator's value wins."

Consequence: this crate cannot be configured through the environment. All behavior is a
function of the pack directory contents plus the arguments passed in.

---

### 2. Environment variables *emitted* by this crate

The crate produces env vars as **data** for a consumer to inject into an agent subprocess.
`runtime_env_vars` — `crates/buzz-persona/src/resolve.rs:365-397`.

| Emitted key | Source field | Condition | Line |
|---|---|---|---|
| `BUZZ_AGENT_MODEL` | model id (post colon-split) | `runtime == "buzz-agent"` | `resolve.rs:375` |
| `BUZZ_AGENT_PROVIDER` | model provider prefix | `runtime == "buzz-agent"` **and** model contains a colon | `resolve.rs:376-378` |
| `GOOSE_PROVIDER` | model provider prefix | any other runtime (incl. `None`, `"goose"`, `"claude"`) **and** model contains a colon | `resolve.rs:381-383` |
| `GOOSE_MODEL` | model id (post colon-split) | any other runtime | `resolve.rs:384` |
| `GOOSE_TEMPERATURE` | `temperature.to_string()` | `temperature` is `Some` — **regardless of runtime** | `resolve.rs:391-393` |
| `GOOSE_CONTEXT_LIMIT` | `max_context_tokens.to_string()` | `max_context_tokens` is `Some` — **regardless of runtime** | `resolve.rs:395-397` |

The runtime-independence of the last two is deliberate and commented: "temperature and
context_limit stay as GOOSE_* (only goose reads them)"
(`crates/buzz-persona/src/resolve.rs:389`).

Consumer-side handling of these keys (outside this crate): the ACP `Config` carries them as
`persona_env_vars: Vec<(String, String)>` (`crates/buzz-acp/src/config.rs:533-535`) and
passes them to spawn as `extra_env` (`crates/buzz-acp/src/lib.rs:3733`).

A related consumer convention is mirrored in this crate's tests: the desktop import path
strips the four derived provider/model keys while preserving the "knob" keys —
`DERIVED_PROVIDER_MODEL_ENV_KEYS` at `crates/buzz-persona/tests/e2e_env_flow.rs:15-20`,
asserted at `:206-228`.

---

### 3. Cargo features

**None declared.** `crates/buzz-persona/Cargo.toml:1-18` has no `[features]` section, and a
grep for `cfg(feature` across the crate returns zero matches. There is no `default`
feature, no optional dependency, and therefore nothing gated by features.

The only conditional compilation in the crate is platform and test based:

| Directive | Location | Gates |
|---|---|---|
| `#[cfg(windows)]` | `crates/buzz-persona/src/pack.rs:334` | Windows drive-letter rejection in `safe_resolve` |
| `#[cfg(unix)]` | `crates/buzz-persona/src/pack.rs:589` | the symlink-escape test |
| `#[cfg(test)]` | `manifest.rs:195`, `merge.rs:200`, `pack.rs:447`, `persona.rs:331`, `resolve.rs:400`, `validate.rs:436` | per-module test modules |

Dependency feature selection is minimal: only `serde` enables a feature (`derive`) —
`crates/buzz-persona/Cargo.toml:10`. `serde_json`, `serde_yaml`, `thiserror`, and
`tempfile` use defaults.

---

### 4. Package-level configuration

`crates/buzz-persona/Cargo.toml:1-8`:

| Key | Value | Note |
|---|---|---|
| `name` | `buzz-persona` | |
| `version` | `0.1.0` | hard-coded, **not** `version.workspace = true` — unlike e.g. `crates/buzz-acp/Cargo.toml:3` |
| `edition` | `2021` | hard-coded; no `rust-version` key |
| `description` | `Parser and loader for Buzz persona pack files (.persona.md)` | |
| `license` | `Apache-2.0` | hard-coded |
| `repository` | `https://github.com/block/sprout` | stale — the OSS repo is `block/buzz` per `AGENTS.md` |

Workspace membership: `Cargo.toml:23` (`"crates/buzz-persona"`).

---

### 5. Compile-time constants (the crate's tunables)

Because there are no env vars or features, all tunable behavior is expressed as constants.
Changing any of them requires a recompile.

| Constant | Value | Visibility | Location | Governs |
|---|---|---|---|---|
| `MAX_FRONTMATTER_BYTES` | `1_048_576` (1 MiB) | `pub` | `crates/buzz-persona/src/persona.rs:21` | YAML frontmatter cap; also a term in the pack-level file cap |
| `MAX_BODY_BYTES` | `262_144` (256 KiB) | `pub` | `crates/buzz-persona/src/persona.rs:24` | persona prompt cap |
| `DEFAULT_THREAD_REPLIES` | `true` | private | `crates/buzz-persona/src/merge.rs:38` | built-in default (precedence level 5) |
| `DEFAULT_BROADCAST_REPLIES` | `false` | private | `crates/buzz-persona/src/merge.rs:39` | built-in default (level 5) |
| `MAX_NAME_LEN` | `64` | fn-local | `crates/buzz-persona/src/validate.rs:168` | persona name length (validator path) |
| inline `64` | `64` | literal | `crates/buzz-persona/src/resolve.rs:146` | persona name length (resolve path) — duplicated, not shared with `MAX_NAME_LEN` |
| inline trigger built-ins | `mentions=true`, `keywords=[]`, `all_messages=false` | literals | `crates/buzz-persona/src/merge.rs:181`, `:191`, `:196` and again `crates/buzz-persona/src/resolve.rs:264-266` | built-in trigger defaults — declared twice |
| `+ 100` / `+ 200` slack in size guards | — | literals | `crates/buzz-persona/src/persona.rs:266` (`+ 100`), `crates/buzz-persona/src/pack.rs:154` and `:169` (`+ 200`) | delimiter/whitespace headroom above the two caps; the two paths use different slack values |
| `KNOWN_MANIFEST_KEYS` | 16 entries | private | `crates/buzz-persona/src/validate.rs:99-118` | which `plugin.json` top-level keys avoid an "unknown key" warning |
| `KNOWN_BEHAVIORAL_KEYS` | 8 entries (incl. legacy `respond_to`) | private | `crates/buzz-persona/src/validate.rs:121-130` | which `defaults` keys avoid a warning |
| `KNOWN_RESPOND_TO_KEYS` | `mentions`, `keywords`, `all_messages` | private | `crates/buzz-persona/src/validate.rs:133` | which `defaults.triggers` sub-keys avoid a warning |

`KNOWN_MANIFEST_KEYS` full list (`crates/buzz-persona/src/validate.rs:99-118`):
`$schema`, `id`, `name`, `version`, `description`, `author`, `license`, `homepage`,
`repository`, `keywords`, `engines`, `personas`, `defaults`, `pack_instructions`,
`hooks_config`, `mcp_config`.

Drift note: `KNOWN_MANIFEST_KEYS` includes `repository`
(`crates/buzz-persona/src/validate.rs:108`), but `PackManifest` has no `repository` field
(`crates/buzz-persona/src/manifest.rs:79-121`) — the key is warning-exempt yet discarded at
parse time.

---

### 6. Pack-file configuration surface (what a pack author can configure)

This is the crate's real "configuration" input. Paths are pack-relative and resolved
against the canonicalized pack root.

| Config location | Purpose | Discovery rule |
|---|---|---|
| `.plugin/plugin.json` | pack identity + persona list + pack-wide `defaults` | fixed path, required (`crates/buzz-persona/src/pack.rs:132-135`) |
| `<manifest>.personas[]` | which `.persona.md` files to load | explicit list only — no directory globbing; each must exist (`crates/buzz-persona/src/pack.rs:155-164`) |
| `<manifest>.pack_instructions` → else `instructions.md` | pack-level instruction text | explicit path preferred; implicit `instructions.md` at pack root as fallback (`crates/buzz-persona/src/pack.rs:167-188`) |
| `<manifest>.mcp_config` → else `.mcp.json` | shared MCP server map under key `mcpServers` | explicit path preferred; implicit `.mcp.json` at pack root as fallback (`crates/buzz-persona/src/pack.rs:191-220`); shape read at `crates/buzz-persona/src/resolve.rs:285` |
| `skills/<name>/SKILL.md` | skill definitions | presence of `skills/` recorded (`crates/buzz-persona/src/pack.rs:223-230`); directory entries enumerated by `resolve_skills` (`:275-293`) and by the validator (`crates/buzz-persona/src/validate.rs:375-386`) |
| `<manifest>.hooks_config` | pack-level lifecycle hooks | **parsed into `PackManifest` but never loaded** — dropped with an explicit comment at `crates/buzz-persona/src/pack.rs:111-112` |
| `<manifest>.defaults` | behavioral defaults (precedence level 4) | `crates/buzz-persona/src/manifest.rs:49-70`; re-encoded to JSON at `crates/buzz-persona/src/pack.rs:143-147` |
| `.persona.md` frontmatter | per-persona identity + behavioral config (level 3) | `crates/buzz-persona/src/persona.rs:176-196` |

Not honored / not searched:

- No user-level or global persona path. The crate never reads `~/.buzz/packs/`,
  `$XDG_CONFIG_HOME`, or any well-known location — the pack directory is always an
  explicit `&Path` argument (`load_pack`, `resolve_pack`, `resolve_persona_by_name`,
  `validate_pack`). The `~/.buzz/packs/<pack-id>/` convention appears only as prose in
  `crates/buzz-persona/PERSONA_PACK_SPEC.md:993-996` and is the consumer's responsibility.
- No config-file layering, no `.env`, no TOML/INI config for the crate itself.
- No persona directory auto-discovery: a `.persona.md` not listed in `personas[]` is
  invisible to `load_pack`.
- `engines.buzz` is parsed (`crates/buzz-persona/src/manifest.rs:39-40`) but never
  evaluated — no minimum-version gate exists in this crate.

---

### 7. The five-level precedence model, and which levels this crate implements

Documented at `crates/buzz-persona/src/merge.rs:1-9` and
`crates/buzz-persona/PERSONA_PACK_SPEC.md:625-637`.

| Level | Source | Implemented here? | Evidence |
|---|---|---|---|
| 1 | Operator env vars already set in the parent process | **No** — explicitly the consumer's job | `crates/buzz-persona/src/resolve.rs:359-364` |
| 2 | Desktop UI per-agent overrides | **No** — runtime concern | `crates/buzz-persona/src/merge.rs:9` |
| 3 | Per-persona frontmatter | **Yes** | persona arm of `merge_behavioral_config` (`crates/buzz-persona/src/merge.rs:68-73`) |
| 4 | Pack `defaults` in `plugin.json` | **Yes** | defaults iteration (`crates/buzz-persona/src/merge.rs:66-74`) |
| 5 | Built-in hardcoded defaults | **Yes** | constants `crates/buzz-persona/src/merge.rs:38-39`; trigger built-ins `:180-198`; fallback `:159-168` |

Fields with **no** level-5 default (they stay `None` and the consumer decides):
`model`, `temperature`, `max_context_tokens` (`crates/buzz-persona/src/merge.rs:95-97`),
and `subscribe` when neither side sets it (`:114-121`) — though `LoadedPersona.subscribe`
collapses `None` to an empty `Vec` at `crates/buzz-persona/src/pack.rs:432`, so the
distinction is lost past that point.


## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: Configuration

Summary: this crate has **no runtime configuration mechanism of its own**. Everything
configurable is a function parameter or a compile-time constant.

---

### 1. Environment variables

**None.** Verified by search across `crates/buzz-ws-client/`: no `env::var`, no
`std::env`, no `dotenv`, no `option_env!`. The crate never reads process
environment. Environment-driven values such as `BUZZ_RELAY_URL`,
`BUZZ_PRIVATE_KEY`, and `BUZZ_AUTH_TAG` are resolved by the consuming crates and
passed in as arguments — e.g. `buzz-cli` supplies `ws_url`, `self.keys`, and
`self.auth_tag` at `crates/buzz-cli/src/client.rs:1080`.

---

### 2. Cargo features

| Feature | Declared? | Notes |
|---|---|---|
| (none) | No `[features]` section exists in `crates/buzz-ws-client/Cargo.toml` (file is 17 lines: `[package]` `:1`–`7`, `[dependencies]` `:9`–`17`) | No `#[cfg(feature = …)]` anywhere in the source (verified) |

All dependency features are inherited from the workspace root via
`{ workspace = true }` (`crates/buzz-ws-client/Cargo.toml:10`–`17`). The two that
change this crate's behaviour:

| Inherited feature set | Declared at | Effect on this crate |
|---|---|---|
| `tokio-tungstenite = { version = "0.29", features = ["rustls-tls-webpki-roots"] }` | root `Cargo.toml:113` | Determines the TLS backend and trust anchors used by `connect_async` (`connection.rs:53`) |
| `tokio = { version = "1", features = ["rt-multi-thread", "macros", "net", "time", "sync", "io-util", "signal", "process"] }` | root `Cargo.toml:43` | `time` powers `timeout`/`Instant` (`connection.rs:7`, `:134`, `:176`); `net` provides `TcpStream` in the stream alias (`connection.rs:14`) |
| `nostr = { version = "0.44", features = ["nip44", "nip98"] }` | root `Cargo.toml:61` | Event/keys/builder types (`message.rs:1`); neither listed feature is referenced by this crate's code |

The only `#[cfg]` in the crate is the test gate `#[cfg(test)]`
(`connection.rs:296`).

---

### 3. Compile-time constants (the crate's tunables)

| Constant | Value | Visibility | Where applied | file:line |
|---|---|---|---|---|
| `AUTH_CHALLENGE_TIMEOUT_SECS` | `20` | `pub` | `authenticate` challenge wait | decl `connection.rs:17`; use `:76` |
| `AUTH_OK_TIMEOUT_SECS` | `20` | `pub` | `authenticate` OK wait | decl `connection.rs:20`; use `:85` |
| `PUBLISH_OK_TIMEOUT_SECS` | `30` | `pub` | `send_event` OK wait | decl `connection.rs:23`; use `:99` |
| Challenge byte cap | `1024` (inline literal, not a named constant) | n/a | `wait_for_auth_challenge` socket-read path | `connection.rs:198` |

These are `const`, so they are not overridable at runtime; changing a timeout requires
a code edit. Lower bounds are guarded by compile-time assertions in the test module
(`connection.rs:300`–`313`), so an edit that reduces them below 20/20/30 fails to
build.

---

### 4. Caller-supplied configuration (the actual configuration surface)

| Parameter | Type | Function | file:line | Notes |
|---|---|---|---|---|
| `url` / `relay_url` | `&str` | `connect`, `connect_authenticated`, `publish_event` | `connection.rs:38`, `:48`, `:278` | No default; no scheme restriction |
| `keys` | `&Keys` | `connect_authenticated`, `authenticate`, `publish_event`, `build_auth_event` | `connection.rs:39`, `:72`, `:280`; `message.rs:177` | Signing identity |
| `auth_tag` | `Option<&Tag>` | same four | `connection.rs:40`, `:73`, `:281`; `message.rs:178` | Optional single authorization tag added to the AUTH event (`message.rs:182`–`186`) |
| `timeout_dur` | `Duration` | `next_event` | `connection.rs:106` | Per-call read budget; no default provided by the crate |
| `timeout_secs` | `u64` | `publish_event` | `connection.rs:282` | Whole-operation budget; `buzz-cli` passes `75` (`crates/buzz-cli/src/client.rs:1080`) |

Note the composition consequence: `publish_event`'s outer budget must exceed the sum
of the inner fixed waits (20 s challenge + 20 s auth OK + 30 s publish OK = 70 s) for
the inner errors to surface rather than the outer `Timeout`; `buzz-cli`'s `75`
(`crates/buzz-cli/src/client.rs:1080`) sits just above that sum, with a comment
referencing the three constants at `crates/buzz-cli/src/client.rs:1077`.

---

### 5. Logging configuration

`tracing::debug!` is used at three sites (`connection.rs:57`, `:91`, `:123`). The
crate installs no subscriber and reads no log-level configuration — verbosity is
controlled entirely by the host binary's `tracing` setup.

---

### 6. Package metadata

All inherited from the workspace: `version.workspace`, `edition.workspace`,
`rust-version.workspace`, `license.workspace`, `repository.workspace`
(`crates/buzz-ws-client/Cargo.toml:3`–`7`). No `build.rs`, no `[lib]` section
(default lib name `buzz_ws_client`), no `[[bin]]`, no `[dev-dependencies]`.
Registered in the workspace at root `Cargo.toml:16` and exposed as a workspace
dependency at root `Cargo.toml:134`.


## Module: buzz-db (`crates/buzz-db`)

### Aspect: Configuration

#### 1. Environment variables

**Corrected by `2a051a40` (post-analysis sync).** The crate now reads exactly
**one** production environment variable, so the previous "reads no environment
variables in production code" statement no longer holds:

| Variable | Read at | Type / default | Semantics |
|----------|---------|----------------|-----------|
| `BUZZ_MAX_COMMUNITIES_PER_OWNER` | `crates/buzz-db/src/relay_members.rs:391` | positive `i64`; falls back to `MAX_COMMUNITIES_PER_OWNER` = `3` (`:379`) | per-owner community cap. Read **once per process** and cached in a `OnceLock` (`:387-396`), so a mid-process change has no effect. Parse/fallback is isolated in the pure `effective_owner_limit` (`:401-405`): missing, non-numeric, zero, and negative all resolve to `3`. Enforcement sites are unchanged — provisioning (`create_community_with_owner`) and ownership transfer inside the advisory-lock transaction. |

Two gaps come with it, both verified at `07d0265c`:

- **Undocumented in the canonical template.** The variable appears nowhere in
  the root `.env.example` (grep count 0), nor in `deploy/compose/.env.example`,
  so the only description of the knob is the Rust doc comment at
  `crates/buzz-db/src/relay_members.rs:381-386`. Same class of drift as the
  Typesense entries noted at `configuration.md:251`, but inverted: here the code
  has a variable the template does not.
- **No runtime reconfiguration path.** Because the `OnceLock` resolves on first
  use and is never invalidated (`relay_members.rs:388-395`), changing the cap
  requires a process restart. There is no admin event, HTTP endpoint, or SIGHUP
  path that re-reads it — consistent with the rest of the crate, but worth
  knowing before an operator raises the cap on a running relay.

Everything else still holds. Connection
details arrive as a `DbConfig` struct from the caller
(`crates/buzz-db/src/lib.rs:222-234`); the doc comments name the variables the
*relay* is expected to source them from, but the lookup happens outside this
crate:

| Variable | Referenced as | Where documented |
|----------|---------------|------------------|
| `DATABASE_URL` | source of `DbConfig::database_url` | `crates/buzz-db/src/lib.rs:225` |
| `READ_DATABASE_URL` | source of `DbConfig::read_database_url` (e.g. an Aurora `cluster-ro-` endpoint) | `crates/buzz-db/src/lib.rs:227-230` |
| `BUZZ_AUTO_MIGRATE` | relay-side switch that decides whether `Db::migrate` runs; affects fence-probe gating | `crates/buzz-db/src/lib.rs:437-441`; test at `:5946-5951` |

Test-only environment reads (all inside `#[cfg(test)] mod tests`):

| Variable | Precedence | Files |
|----------|-----------|-------|
| `BUZZ_TEST_DATABASE_URL` → `DATABASE_URL` → const default | first match wins | `crates/buzz-db/src/event.rs:1478-1480`, `crates/buzz-db/src/feed.rs:258-260`, `crates/buzz-db/src/git_repo.rs:190-192`, `crates/buzz-db/src/moderation.rs:639-641`, `crates/buzz-db/src/thread.rs:822-824`, `crates/buzz-db/src/push.rs:1270-1272`, `crates/buzz-db/src/workflow.rs:1702-1704`, `crates/buzz-db/src/migration.rs:1028-1030`, `crates/buzz-db/src/relay_members.rs:616-618`, `crates/buzz-db/src/product_feedback.rs:127-129` |
| `BUZZ_TEST_DATABASE_URL` → const default | no `DATABASE_URL` fallback | `crates/buzz-db/src/channel.rs:1794` |
| `TEST_DATABASE_URL` → const default | different name from the above | `crates/buzz-db/src/lib.rs:3936`, `:4655`, `:5228`; `crates/buzz-db/src/replica_fence.rs:515` |
| (none) | hard-coded const only | `crates/buzz-db/src/archived_identities.rs:130-135`, `crates/buzz-db/src/user.rs:406-408`, `crates/buzz-db/src/usage.rs:365-369`, `crates/buzz-db/src/api_token.rs` test setup |

The default test URL literal is
`postgres://buzz:buzz_dev@localhost:5432/buzz`, repeated as a
`const TEST_DB_URL` in 16 test modules.

#### 2. Cargo features

None. `crates/buzz-db/Cargo.toml` has no `[features]` section
(`crates/buzz-db/Cargo.toml:1-25`). All dependency feature selection is inherited
from the workspace table — most relevantly
`sqlx = { version = "0.9", features = ["runtime-tokio", "tls-rustls", "postgres", "uuid", "chrono", "json"] }`
(`Cargo.toml:52-54`). Package metadata (`version`, `edition`, `rust-version`,
`license`, `repository`) is all `.workspace = true`
(`crates/buzz-db/Cargo.toml:3-8`); the workspace pins `edition = "2021"` and
`rust-version = "1.88.0"` (`Cargo.toml:36-37`).

#### 3. Pool tunables (`DbConfig`)

Defined at `crates/buzz-db/src/lib.rs:222-234`, defaults at `:236-249`, applied
at `:387-393`.

| Field | Type | Default | Applied as |
|-------|------|---------|-----------|
| `database_url` | `String` | `postgres://buzz:buzz_dev@localhost:5432/buzz` | writer pool target |
| `read_database_url` | `Option<String>` | `None` (replica routing disabled) | replica pool target |
| `max_connections` | `u32` | `20` | `PgPoolOptions::max_connections` |
| `min_connections` | `u32` | `2` | `PgPoolOptions::min_connections` |
| `acquire_timeout_secs` | `u64` | `3` | `PgPoolOptions::acquire_timeout` |
| `max_lifetime_secs` | `u64` | `1800` | `PgPoolOptions::max_lifetime` |
| `idle_timeout_secs` | `u64` | `600` | `PgPoolOptions::idle_timeout` |

Sizing rationale recorded in the `Default` doc comment: staging measured 51 idle
+ 1 active out of 50, so 20 main + 5 audit = 25 connections per pod lets four
relay pods fit a Postgres `max_connections = 100`
(`crates/buzz-db/src/lib.rs:236-239`). Both pools use the **same** sizing
(`crates/buzz-db/src/lib.rs:361-364`), and `DbPoolStats::max` reports the
configured ceiling for either pool (`:494-511`).

`Db::from_pool` / `Db::from_pools` derive `max_connections` from
`pool.options().get_max_connections()` instead of a config value
(`crates/buzz-db/src/lib.rs:404-427`) and do **not** arm the floor guard, so
callers that build pools themselves are unguarded unless they set the GUC.

#### 4. Session GUCs (runtime configuration in Postgres)

| GUC | Set by | Scope | Effect |
|-----|--------|-------|--------|
| `buzz.created_at_floor` | writer-pool `after_connect` hook, value = `CREATED_AT_FLOOR_SECS` (`crates/buzz-db/src/lib.rs:394-405`) | session (`is_local = false`) | arms the deferred commit-time floor guard from `migrations/0021_created_at_fence_floor.sql:44-68`; unset/blank ⇒ guard is a no-op |
| `buzz.nip_rs_hard_delete` | `set_config(..., true)` inside the NIP-RS replacement transaction (`crates/buzz-db/src/lib.rs:3742-3747`) and inside the purge trigger (`migrations/0011_nip_rs_exact_tag_cardinality.sql:143`) | **transaction-local** | authorises the BEFORE DELETE fence at `migrations/0011_…:45-60`; verified not to leak across commit or rollback by test `crates/buzz-db/src/lib.rs:4194-4218` |

Tests arm the floor guard transaction-locally with
`set_config('buzz.created_at_floor', $1, true)`
(`crates/buzz-db/src/lib.rs:5811-5815`, `:5991-5995`).

#### 5. Compile-time constants

| Constant | Value | File:line | Meaning |
|----------|-------|-----------|---------|
| `replica_fence::CREATED_AT_FLOOR_SECS` | `960` | `crates/buzz-db/src/replica_fence.rs:51` | seconds of `created_at` history the commit-time guard tolerates; must exceed the relay's ±900 s ingest envelope. The GUC value and the fence subtrahend must never diverge. |
| `replica_fence::FENCE_CLOCK_MARGIN_SECS` | `5` | `crates/buzz-db/src/replica_fence.rs:59` | extra margin subtracted from the fence |
| `replica_fence::PROBE_INTERVAL` | `5 s` | `crates/buzz-db/src/replica_fence.rs:62` | probe cadence |
| `replica_fence::FENCE_STALENESS` | `30 s` | `crates/buzz-db/src/replica_fence.rs:66` | fence age after which it is treated as closed |
| `replica_fence::CLOSED` (private) | `i64::MIN` | `crates/buzz-db/src/replica_fence.rs:69` | closed sentinel |
| `feed::FEED_MAX_LIMIT` | `100` | `crates/buzz-db/src/feed.rs:29` | hard cap on every feed query |
| `workflow::LIST_DEFAULT_LIMIT` | `100` | `crates/buzz-db/src/workflow.rs:25` | default workflow list page |
| `workflow::LIST_MAX_LIMIT` | `1000` | `crates/buzz-db/src/workflow.rs:27` | workflow list ceiling |
| `admin_moderation::MAX_PAGE_SIZE` | `200` | `crates/buzz-db/src/admin_moderation.rs:15` | admin page clamp |
| `relay_members::MAX_COMMUNITIES_PER_OWNER` | `3` | `crates/buzz-db/src/relay_members.rs:379` | per-pubkey community ownership cap — **no longer hard-coded as of `2a051a40`**: it is now the *fallback* default, overridable per process by `BUZZ_MAX_COMMUNITIES_PER_OWNER` (read once and cached in `max_communities_per_owner`, `relay_members.rs:387-396`; parse/fallback rules in `effective_owner_limit`, `:401-405` — missing, unparseable, or non-positive values fall back to `3`) |
| `push::MAX_MATCH_ATTEMPTS` | `8` | `crates/buzz-db/src/push.rs:70` | matcher-job attempt ceiling |
| `push::PUSH_GATE_LOCK_NAMESPACE` (private) | `"buzz_push_gate:"` | `crates/buzz-db/src/push.rs:21` | advisory-lock key domain; must match `migrations/0023_push_match_gate.sql:34` |
| `push::PUSH_GATE_BACKFILL_SECS` (private) | `120` | `crates/buzz-db/src/push.rs:41` | activation backfill window over `received_at` |
| `event::D_TAG_MAX_LEN` | `1024` | `crates/buzz-db/src/event.rs:140` | documented ceiling, **not enforced in this crate** |
| `event::HUDDLE_LINK_CONTENT_MAX_BYTES` (private) | `512` | `crates/buzz-db/src/event.rs:147` | max candidate content size for huddle-link resolution |
| `event::HUDDLE_LINK_CANDIDATE_LIMIT` (private) | `32` | `crates/buzz-db/src/event.rs:150` | max candidate rows inspected |
| `partition::PARTITIONED_TABLES` (private) | `["events", "delivery_log"]` | `crates/buzz-db/src/partition.rs:12` | DDL allowlist |
| `moderation::MODERATION_ACTION_CHECK_VOCAB` | 12 strings | `crates/buzz-db/src/moderation.rs:104` | mirror of the SQL CHECK |

Hard-coded limits embedded directly in SQL rather than named constants:

| Limit | Value | Where |
|-------|-------|-------|
| Default query limit / clamp | 100 / 1000 | `crates/buzz-db/src/event.rs:350-352` |
| `list_channels`, `get_members`, `get_bot_members`, `get_accessible_channels` | `LIMIT 1000` | `crates/buzz-db/src/channel.rs:696`, `:713`, `:588`, `:903`, `:840` |
| `list_active_tokens` | `LIMIT 1000` | `crates/buzz-db/src/lib.rs:2396` |
| Active-token quota per (community, owner) | `< 10` | `crates/buzz-db/src/api_token.rs:119` |
| Thread/window participant cap | 10 | `crates/buzz-db/src/thread.rs:520`, `:698` |
| `list_dms_for_user` limit cap | 200 | `crates/buzz-db/src/dm.rs:233` |
| `search_users` limit clamp | `[1, 500]` | `crates/buzz-db/src/user.rs:233` |
| `list_workflow_runs` limit cap | 1000 | `crates/buzz-db/src/workflow.rs:824` |
| DM participant bounds | 2–9 | `crates/buzz-db/src/dm.rs:107-117` |
| `get_events_by_ids` debug bound | 500 | `crates/buzz-db/src/event.rs:997` |
| `UsageMetricsLeader::is_live` ping timeout | 5 s | `crates/buzz-db/src/lib.rs:216` |
| Push-eligible kinds (trigger allowlist) | `7, 9, 1059, 40007, 46010` | `migrations/0018_push_match_queue.sql:25`, `migrations/0023_push_match_gate.sql:26`, mirrored in Rust at `crates/buzz-db/src/push.rs:58` |
| FTS excluded / allowlisted kinds | see data-model §6 | `migrations/0001:223`, `0005:31`, `0008:15`, `0014:29` |
| TTL-refresh exempt kind | `9007` | `migrations/0024_event_ttl_refresh_shared_lock.sql:30` |
| NIP-29 discovery kinds soft-deleted together | `39000, 39001, 39002` | `crates/buzz-db/src/lib.rs:3287` |
| Declared partition ranges | 2026-01 … 2026-07 plus `_p_past`/`_p_future` | `migrations/0001_initial_schema.sql:237-252`, `:343-354` |

#### 6. Operational configuration expectations

- `ensure_future_partitions(months_ahead)` is meant to run on startup and monthly
  via cron; the caller chooses `months_ahead`
  (`crates/buzz-db/src/partition.rs:1-4`, `:15`).
- `spawn_fence_probe` must be called **after** the migration decision, otherwise
  an armed GUC with an unapplied migration 0021 could open a fence over an
  unenforced floor (`crates/buzz-db/src/lib.rs:434-448`).
- The probe role needs `pg_monitor`; without it the fence stays closed
  (`crates/buzz-db/src/replica_fence.rs:369-374`).
- Backfills of channel-bearing `events` rows must run on a connection **without**
  `buzz.created_at_floor` and with the replica breaker held closed
  (`migrations/0021_created_at_fence_floor.sql:26-42`).
- `prune_scheduled_workflow_fires_before`'s cutoff must be older than the largest
  supported interval schedule (`crates/buzz-db/src/workflow.rs:589-596`).
- Migration `0004` builds a GIN index on a partitioned parent without
  `CONCURRENTLY` (unsupported there), so brownfield application takes a share
  lock per partition and should run in a deploy window
  (`migrations/0004_events_tags_gin.sql:13-17`).


## Module: buzz-auth (`crates/buzz-auth`)

### Aspect: Configuration

### Environment variables read by this crate

**None.** A grep of `crates/buzz-auth/` for `std::env`, `env::var`, `env!`, and
`option_env!` returns zero matches. The crate is configured purely by the
`AuthConfig` value its caller passes to `AuthService::new`
(`crates/buzz-auth/src/lib.rs:106-108`).

The seven `BUZZ_RATE_LIMIT_*` variables that populate this crate's
`RateLimitConfig` are read in `buzz-relay`
(`crates/buzz-relay/src/config.rs:284-316`) — see the summary section below.

---

### Cargo features

Declared in `crates/buzz-auth/Cargo.toml:10-12`. There is **no** `default`
feature set, so both are opt-in.

| Feature | Declared | Deps enabled | What it gates |
|---------|----------|--------------|---------------|
| `test-utils` | `crates/buzz-auth/Cargo.toml:11` | none (empty list) | The three test doubles and their root re-exports, all via `#[cfg(any(test, feature = "test-utils"))]`: `MockAccessChecker` struct + inherent impl + `Default` + `ChannelAccessChecker` impl (`crates/buzz-auth/src/access.rs:107`, `:112`, `:128`, `:135`); `AlwaysAllowRateLimiter` struct + `RateLimiter` impl (`crates/buzz-auth/src/rate_limit.rs:218`, `:221`); `AlwaysFreshReplayGuard` struct + `Nip98ReplayGuard` impl (`crates/buzz-auth/src/nip98_replay.rs:126`, `:129`); re-exports (`crates/buzz-auth/src/lib.rs:46-51`) |
| `dev` | `crates/buzz-auth/Cargo.toml:12` | none (empty list) | Exactly one item: `derive_pubkey_from_username` (`crates/buzz-auth/src/lib.rs:159-167`) — the `SHA-256("buzz-test-key:{username}")` deterministic key derivation |

Both features use the `any(test, feature = "...")` form, so the gated code is
also compiled during this crate's own `cargo test` run without the flag.

#### Who enables them

| Feature | Enabled by | Where | Effect on production build |
|---------|-----------|-------|----------------------------|
| `dev` | `buzz-relay`'s own `dev` feature (`dev = ["buzz-auth/dev"]`) | `crates/buzz-relay/Cargo.toml:84` | none unless `--features dev` is passed; no invocation in `justfile`, CI, or Dockerfiles |
| `dev` | `buzz-relay` `[dev-dependencies]` entry `buzz-auth = { workspace = true, features = ["dev"] }` | `crates/buzz-relay/Cargo.toml:90` | none for a non-test build: workspace uses `resolver = "2"` (`Cargo.toml:32`), which does not unify dev-dependency features into normal builds. `Dockerfile:67` passes no feature flags |
| `test-utils` | nobody | — | The feature is never requested by any crate in the repo (grep of all `Cargo.toml` for `buzz-auth` shows plain `{ workspace = true }` at `crates/buzz-pubsub/Cargo.toml:12`, `crates/buzz-admin/Cargo.toml:17`, `crates/buzz-relay/Cargo.toml:22`, and `features = ["dev"]` only at `crates/buzz-relay/Cargo.toml:90`) |

`buzz-relay`'s `[features]` table contains only `dev`
(`crates/buzz-relay/Cargo.toml:83-84`) — no `default`.

---

### Compile-time constants

All five constants in the crate, with values:

| Constant | Value | Visibility | Purpose | file:line |
|----------|-------|-----------|---------|-----------|
| `TIMESTAMP_TOLERANCE_SECS` | `60` (`u64`) | private to `nip42` | NIP-42 `created_at` skew window (symmetric via `abs_diff`) | `crates/buzz-auth/src/nip42.rs:35` |
| `TIMESTAMP_TOLERANCE_SECS` | `60` (`u64`) | private to `nip98` | NIP-98 `created_at` skew window; separate definition with the same name/value | `crates/buzz-auth/src/nip98.rs:32` |
| `DEFAULT_REPLAY_TTL_SECS` | `120` (`u64`) | `pub`, re-exported at root | Floor for the NIP-98 replay window = 2× the ±60s tolerance; implementations MAY raise, MUST NOT lower | `crates/buzz-auth/src/nip98_replay.rs:46`, re-export `crates/buzz-auth/src/lib.rs:38` |
| `MAX_REPLAY_TTL_SECS` | `3600` (`u64`) | `pub`, re-exported at root | Ceiling for the replay window; implementations MUST clamp down. Rationale: 30× the natural maximum and safely inside Redis `EX`'s i64 range | `crates/buzz-auth/src/nip98_replay.rs:59`, re-export `crates/buzz-auth/src/lib.rs:39` |

Three of these are pinned by const-drift tripwire tests:
`DEFAULT_REPLAY_TTL_SECS >= 120` (`crates/buzz-auth/src/nip98_replay.rs:210-218`),
`DEFAULT_REPLAY_TTL_SECS < MAX_REPLAY_TTL_SECS` (`:220-230`),
`MAX_REPLAY_TTL_SECS <= i64::MAX as u64` (`:232-237`).

Related constant defined outside the crate but consumed against this crate's
config: `WS_BURST_WINDOW_SECS = 5` (`crates/buzz-relay/src/admission.rs:9`), which
multiplies `human_ws_events_per_sec` into a 5-second budget
(`crates/buzz-relay/src/admission.rs:40-45`).

---

### Configuration structs (deserialisation contract)

`AuthConfig` — `Serialize + Deserialize + Default`
(`crates/buzz-auth/src/lib.rs:90-95`). One field, `rate_limits: RateLimitConfig`
with `#[serde(default)]` (`:93-94`), so an empty `[auth]` section is valid. The
doc says it is "typically loaded from the relay's TOML config file"
(`crates/buzz-auth/src/lib.rs:89`); in practice the relay builds it from env vars.

`RateLimitConfig` — every field has a `#[serde(default = "fn")]` fallback, so any
subset of keys deserialises successfully
(`crates/buzz-auth/src/rate_limit.rs:85-108`):

| Field | Type | Default | Default fn |
|-------|------|---------|-----------|
| `human_messages_per_min` | `u64` | 60 | `crates/buzz-auth/src/rate_limit.rs:110-112` |
| `human_api_calls_per_min` | `u64` | 300 | `crates/buzz-auth/src/rate_limit.rs:113-115` |
| `human_ws_events_per_sec` | `u64` | 10 | `crates/buzz-auth/src/rate_limit.rs:116-118` |
| `agent_standard_messages_per_min` | `u64` | 120 | `crates/buzz-auth/src/rate_limit.rs:119-121` |
| `agent_standard_api_calls_per_min` | `u64` | 600 | `crates/buzz-auth/src/rate_limit.rs:122-124` |
| `agent_elevated_messages_per_min` | `u64` | 300 | `crates/buzz-auth/src/rate_limit.rs:125-127` |
| `agent_platform_messages_per_min` | `u64` | 600 | `crates/buzz-auth/src/rate_limit.rs:128-130` |

The manual `Default` impl reuses the same seven fns rather than repeating the
literals (`crates/buzz-auth/src/rate_limit.rs:132-144`), so each number has one
source of truth.

`LimitType` deserialises from snake_case strings (`messages`, `api_calls`,
`ws_events`, `ip_connections`) via
`#[serde(rename_all = "snake_case")]` (`crates/buzz-auth/src/rate_limit.rs:56-67`).

No other type in the crate is serde-annotated — `AuthContext`, `AuthMethod`,
`Scope`, and `RateLimitResult` are all plain in-memory types.

---

### Rate-limit configuration — summary of the verified state

Full analysis with per-element scoring lives in the security aspect under
`### Rate limiting — verified state`. Configuration-relevant summary:

`ARCHITECTURE.md:823` (§9 item 2) claims rate limiting is unimplemented and that
`AlwaysAllowRateLimiter` is the only `RateLimiter` implementation. **That is
stale.** The `.env.example:58` heading "Shared Redis-backed admission limits" is
the accurate description.

- A production `RateLimiter` exists: `RedisRateLimiter`
  (`crates/buzz-pubsub/src/rate_limiter.rs:88-121`), ungated, using an atomic
  `INCR`+`EXPIRE` Lua script (`crates/buzz-pubsub/src/rate_limiter.rs:24-31`).
- It is constructed unconditionally at relay startup
  (`crates/buzz-relay/src/state.rs:712`) and held on `AppState`
  (`crates/buzz-relay/src/state.rs:584`).
- Checks run **before** work is admitted: WebSocket EVENT/REQ/COUNT
  (`crates/buzz-relay/src/connection.rs:498-500` → `:594-653`) and HTTP
  `POST /events` / `/query` / `/count`
  (`crates/buzz-relay/src/api/bridge.rs:760`, `:955`, `:1386` → `:24-56`).
  Limiter errors fail closed (`crates/buzz-relay/src/admission.rs:33-36`).
- Media upload uses a **separate, per-pod, in-process** `DashMap` limiter keyed on
  `media_uploads_per_minute` (`crates/buzz-relay/src/api/media.rs:88-111`, window
  const `:66`) — not the Redis limiter, not any `BUZZ_RATE_LIMIT_*` var.
- `check_ip_connection` / `LimitType::IpConnections` is implemented
  (`crates/buzz-pubsub/src/rate_limiter.rs:112-120`) but has **no production
  caller** — there is no per-IP connection cap.

Env-var wiring (`crates/buzz-relay/src/config.rs`):

| Env var | Parsed at | Default | Reaches an enforcement point? |
|---------|-----------|---------|-------------------------------|
| `BUZZ_RATE_LIMIT_HUMAN_MESSAGES_PER_MIN` | `config.rs:287-290` | 60 | yes — `crates/buzz-relay/src/connection.rs:636` (WS EVENT) |
| `BUZZ_RATE_LIMIT_HUMAN_API_CALLS_PER_MIN` | `config.rs:291-294` | 300 | yes — `crates/buzz-relay/src/api/bridge.rs:29` (HTTP bridge) |
| `BUZZ_RATE_LIMIT_HUMAN_WS_EVENTS_PER_SEC` | `config.rs:295-298` | 10 | yes — `crates/buzz-relay/src/connection.rs:614` (×5s burst window) |
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_MESSAGES_PER_MIN` | `config.rs:299-302` | 120 | yes — `crates/buzz-relay/src/connection.rs:634` (NIP-OA agents) |
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_API_CALLS_PER_MIN` | `config.rs:303-306` | 600 | **no consumer** |
| `BUZZ_RATE_LIMIT_AGENT_ELEVATED_MESSAGES_PER_MIN` | `config.rs:307-310` | 300 | **no consumer** |
| `BUZZ_RATE_LIMIT_AGENT_PLATFORM_MESSAGES_PER_MIN` | `config.rs:311-314` | 600 | **no consumer** |

Parsing rules (`positive_u64_from_env`,
`crates/buzz-relay/src/config.rs:270-282`): unset → the `RateLimitConfig::default()`
value; zero or non-integer → `ConfigError::InvalidValue("{name} must be a positive
integer")`; non-Unicode → `ConfigError::InvalidValue`. So a value of `0` is a
startup failure, not "disabled" — pinned by test at
`crates/buzz-relay/src/config.rs:1109-1120`. Override plumbing is pinned by
`rate_limits_can_be_overridden` (`crates/buzz-relay/src/config.rs:1092-1106`).

CI raises three of the limits (`.github/workflows/ci.yml:492-494`:
`HUMAN_MESSAGES_PER_MIN=100000`, `HUMAN_API_CALLS_PER_MIN=100000`,
`HUMAN_WS_EVENTS_PER_SEC=10000`) so the integration suite does not trip them —
corroborating that enforcement is live rather than a no-op.

Documentation drift to fix: `ARCHITECTURE.md:390`, `ARCHITECTURE.md:460`, and
`ARCHITECTURE.md:823`.


## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Configuration

`buzz-pubsub` reads **no environment variables in any production path**. All
configuration is injected by the caller as constructor arguments. A repo-wide grep
for `env::var` inside `crates/buzz-pubsub/src/` returns exactly one hit, and it is
inside a `#[cfg(test)]` module.

---

### 1. Injected configuration

| Input | Type | Entry point | Supplied by |
|---|---|---|---|
| Redis URL (for pub/sub connections) | `String` | `PubSubManager::new` (`lib.rs:117`) → stored `lib.rs:105` | `buzz-relay/src/main.rs:344` passes `&config.redis_url` |
| Redis connection pool (for all commands) | `deadpool_redis::Pool` | `PubSubManager::new` (`lib.rs:117`), `RedisRateLimiter::new` (`rate_limiter.rs:94`), `RedisNip98ReplayGuard::new` (`nip98_replay.rs:29`) | `buzz-relay/src/state.rs:711-712`, `:717`; pool field `state.rs:494` |
| Unsubscribe debounce | `Duration` | `PubSubConfig::with_unsubscribe_debounce` (`lib.rs:93`) | **nobody** — see §3 |
| Rate-limit window + limit | `u64, u64` per call | `check_and_increment` args (`rate_limiter.rs:100-107`) | `buzz-relay/src/connection.rs:610-611`, `:628-646` from `state.auth.config().rate_limits` (`connection.rs:610`) |
| Replay TTL | `u64` per call, then clamped | `try_mark_in_scope` (`nip98_replay.rs:38`), clamp `:47` | `buzz-auth` constants `DEFAULT_REPLAY_TTL_SECS` / `MAX_REPLAY_TTL_SECS` (`nip98_replay.rs:11-12`) |

The Redis URL reaches the crate from `REDIS_URL`, resolved in the relay with a
default of `redis://localhost:6379` (`buzz-relay/src/config.rs:418-419`, field
declared `config.rs:60`). The crate never reads that variable itself.

Note the pool is **injected, not constructed** — so pool size, timeouts, and any TLS
settings are entirely the relay's concern; `buzz-pubsub` cannot influence them. Only
the three dedicated pub/sub sockets are opened from the crate's own stored URL
(`subscriber.rs:85-86`, `cache_invalidation.rs:135-136`, `conn_control.rs:126-127`),
each via `redis::Client::open` on every reconnect attempt.

### 2. Compile-time constants (the crate's real tunables)

| Constant | Value | Site | Configurable at runtime? |
|---|---|---|---|
| `PubSubConfig::DEFAULT_UNSUBSCRIBE_DEBOUNCE` | 500 ms | `lib.rs:82` | Only via the builder — unused in production |
| Event broadcast capacity | 4096 | `lib.rs:126` | No |
| Cache-invalidation broadcast capacity | 4096 | `lib.rs:127` | No |
| Conn-control broadcast capacity | 4096 | `lib.rs:128` | No |
| Subscription-command mpsc capacity | 4096 | `lib.rs:129` | No |
| `PRESENCE_TTL_SECS` | 90 | `presence.rs:16` | No |
| `BACKOFF_INITIAL_SECS` | 1 | `subscriber.rs:16`, `cache_invalidation.rs:91`, `conn_control.rs:81` | No — three separate copies |
| `BACKOFF_MAX_SECS` | 30 | `subscriber.rs:18`, `cache_invalidation.rs:93`, `conn_control.rs:83` | No — three separate copies |
| `BUZZ_PREFIX` | `"buzz"` | `topic.rs:13` | No |
| `CACHE_INVALIDATION_SUFFIX` / `_PATTERN` | `"cache-invalidate"` / `"buzz:*:cache-invalidate"` | `cache_invalidation.rs:23`, `:27` | No |
| `CONN_CONTROL_SUFFIX` / `_PATTERN` | `"conn-control"` / `"buzz:*:conn-control"` | `conn_control.rs:26`, `:30` | No |
| `RATE_LIMIT_SCRIPT` | Lua source | `rate_limiter.rs:24-31` | No |

Operationally significant: **presence TTL, all four channel capacities, and the
reconnect backoff envelope cannot be tuned without a recompile.** For a horizontally
scaled deployment these are exactly the knobs an operator would want — e.g. raising
broadcast capacity for pods with many concurrent sockets, or lengthening presence TTL
for clients on flaky networks.

### 3. The unsubscribe-debounce knob is unreachable

`PubSubConfig` exists (`lib.rs:73-97`) and `with_unsubscribe_debounce`
(`lib.rs:93`) works, but every production construction goes through
`PubSubManager::new` (`lib.rs:117-119`), which hardcodes
`PubSubConfig::new(redis_url)` and therefore always takes the 500 ms default.

All 11 `PubSubManager::new` call sites in the relay confirm this — the production one
is `buzz-relay/src/main.rs:344`; the other ten are test fixtures
(`state.rs:1267`, `api/operator.rs:591`, `api/bridge.rs:3309`, `api/invites.rs:536`,
`api/admin/mod.rs:348`, `api/media.rs:970`, `workflow_sink.rs:585`,
`handlers/identity_archive.rs:451`, `handlers/event.rs:1990`, `:2043`). Zero call
sites for `with_config` or `with_unsubscribe_debounce` exist outside this crate's own
tests (`lib.rs:514`, `:596`, `:621`).

### 4. Rate-limit configuration reaching this crate

The crate holds no limits of its own; window and limit arrive per call. The relay
derives them from `state.auth.config().rate_limits` (`buzz-relay/src/connection.rs:610`):

| Limit | Derivation | Site |
|---|---|---|
| WS admission | `human_ws_events_per_sec` converted to a 5 s window via `ws_admission_budget` → `(5, limit × 5)` | `buzz-relay/src/admission.rs:8`, `:35-40`; used `connection.rs:611-621` |
| Per-message (human) | `human_messages_per_min`, fixed 60 s window | `connection.rs:629-633`, `:641` |
| Per-message (agent) | `agent_standard_messages_per_min`, fixed 60 s window | `connection.rs:629-631`, `:641` |

The IP-connection limit path (`rate_limiter.rs:112-120`) receives no configuration
because it has no production caller.

### 5. Test-only configuration

| Item | Site | Behaviour |
|---|---|---|
| `REDIS_URL` | `nip98_replay.rs:110` | `std::env::var("REDIS_URL")` with fallback `redis://127.0.0.1:6379` — the only env read in the crate, and it is inside `#[cfg(test)] mod tests` |
| `test_util::make_test_pool` | `lib.rs:371-376` | Hardcodes `redis://127.0.0.1:6379`, ignoring `REDIS_URL` |
| `#[ignore = "requires Redis"]` | 11 tests (`lib.rs:400`, `:438`, `:476`, `:511`; `presence.rs:137`, `:160`, `:187`; `nip98_replay.rs:128`, `:145`, `:165`, `:180`) | Integration tests opt out of the default `cargo test` run |

Inconsistency: `nip98_replay.rs` honours `REDIS_URL` while `test_util::make_test_pool`
(`lib.rs:371-376`) and every test using it hardcode `127.0.0.1:6379`. Pointing CI at
a non-default Redis host would run some of this crate's tests against the wrong
endpoint. Also note the relay's default (`localhost`, `config.rs:419`) and the
crate's test default (`127.0.0.1`) differ in spelling, which matters on hosts where
`localhost` resolves to IPv6 first.

### 6. Not configured anywhere

No configuration exists for: Redis TLS or auth (must be embedded in the URL string),
Redis logical db index, `maxmemory`/eviction expectations for the replay seen-set,
pub/sub message size limits, or a cap on topics retained per connection.


## Module: buzz-search (`crates/buzz-search`)

### Aspect: Configuration

#### Environment variables

| Variable | Read in production code? | Where | Default | Purpose |
|---|---|---|---|---|
| `BUZZ_TEST_DATABASE_URL` | No — test harness only | `tests/fts_integration.rs:33`, `tests/fts_integration.rs:92` | `TEST_DB_URL` = `postgres://buzz:buzz_dev@localhost:5432/buzz` (`tests/fts_integration.rs:21`) | Postgres connection for the ignored integration tests |

**No environment variable is read anywhere in `src/`.** All runtime configuration
reaches the crate through the `PgPool` the caller passes to
`SearchService::new(pool)` (`crates/buzz-search/src/lib.rs:46-48`) — connection
string, pool sizing, TLS, and timeouts are entirely the caller's concern
(constructed by the relay, e.g. `crates/buzz-relay/src/state.rs:1273`).

#### Cargo features

| Feature | Status |
|---|---|
| Crate-defined features | None — `Cargo.toml` has no `[features]` section (`crates/buzz-search/Cargo.toml:1-18`) |
| `#[cfg(feature = ...)]` in code | None in any `.rs` file |
| `#[cfg(test)]` | One, gating the unit-test module (`src/query.rs:325`) |
| Inherited sqlx features (from workspace) | `runtime-tokio`, `tls-rustls`, `postgres`, `uuid`, `chrono`, `json` (root `Cargo.toml:52-54`) |
| Inherited uuid features | `v4`, `serde` (root `Cargo.toml:89`) |

Package metadata is fully workspace-inherited (`version`, `edition`,
`rust-version`, `license`, `repository`) — `crates/buzz-search/Cargo.toml:3-7`.

#### Compile-time constants (complete list)

| Constant | Value | Type | Line | Effect |
|---|---|---|---|---|
| `PER_PAGE_MAX` | `500` | `u32` | `src/query.rs:129` | Upper clamp on `SearchQuery.per_page` → `LIMIT` (`query.rs:224`) |
| `PER_PAGE_DEFAULT` | `100` | `u32` | `src/query.rs:130` | Substituted when `per_page == 0` (`query.rs:225-229`) |
| `SEARCH_TEXT_MAX_CHARS` | `4096` | `usize` | `src/query.rs:134` | Hard char cap on search text before it reaches Postgres' text-search parsers (`query.rs:185`) |
| `PAGE_MAX` | `1000` | `u32` | `src/query.rs:138` | Upper clamp on `page`, bounding `OFFSET = (page-1) * per_page` (`query.rs:220`, `230-231`) |

Effective caps composed with those constants: maximum reachable `OFFSET` is
`999 * 500 = 499_500` rows, and maximum rows returned per call is 500.

Test-file constants (not production configuration):

| Constant | Value | Line |
|---|---|---|
| `TEST_DB_URL` | `postgres://buzz:buzz_dev@localhost:5432/buzz` | `tests/fts_integration.rs:21` |
| `MIGRATION_0001_SQL` … `MIGRATION_0008_SQL`, `MIGRATION_0014_SQL` | `include_str!` of the corresponding `migrations/*.sql` | `tests/fts_integration.rs:22-32` |

#### Hard-coded behavioral settings (not configurable)

| Setting | Value | Line |
|---|---|---|
| Text-search configuration | `'simple'` (no stemming, no stopwords) — used for both `websearch_to_tsquery` and the prefix-mode `to_tsvector` | `query.rs:143`, `query.rs:173` |
| Ranking function | `ts_rank_cd` (no weights, no normalization flag) | `query.rs:236` |
| Ordering | `rank DESC, created_at DESC, id` | `query.rs:295` |
| Prefix-mode token split | `regexp_split_to_table(<term>, '\s+')` | `query.rs:167-170` |
| Prefix-mode conjunction | `' & '` | `query.rs:157` |
| Table queried | `events` (unqualified; resolved via the connection's `search_path`) | `query.rs:237` |

#### Caller-side settings that shape this crate's inputs (for reference)

| Setting | Value | Site |
|---|---|---|
| `MAX_SEARCH_PAGES` (WS pages iterated) | `10` | `crates/buzz-relay/src/handlers/req.rs:421` |
| WS `per_page` | `100` fixed | `crates/buzz-relay/src/handlers/req.rs:591` |
| Bridge `per_page` | `filter.limit.unwrap_or(100).min(500)` | `crates/buzz-relay/src/api/bridge.rs:1660` |
| Bridge `page` | raw `page` / `search_page` / `searchPage`, `> 0`, default `1` | `crates/buzz-relay/src/api/bridge.rs:345-353` |
| Bridge `mode` | `"prefix"` → `SearchMode::Prefix`, else `FullText` | `crates/buzz-relay/src/api/bridge.rs:337-345` |


## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Configuration

### Environment variables read by this crate

| Variable | Read at | Scope | Default |
|---|---|---|---|
| `DATABASE_URL` | `crates/buzz-audit/src/service.rs:276` | **Test-only** helper `test_pool()` inside `#[cfg(test)] mod tests` | `postgres://buzz:buzz_dev@localhost:5432/buzz` (`service.rs:277`) |

That is the **only** `std::env` access in the crate (grep for `env::var`/`std::env`
across `crates/buzz-audit/src` returns exactly `service.rs:276`).

`BUZZ_AUDIT_ENABLED` is **not** consumed here. It is read by the relay:

| Aspect | Location |
|---|---|
| Field declaration + doc | `crates/buzz-relay/src/config.rs:199-202` |
| Parse, default `true` | `crates/buzz-relay/src/config.rs:793` (`parse_bool("BUZZ_AUDIT_ENABLED", true)`) |
| Gate on service construction | `crates/buzz-relay/src/main.rs:321-334` — `Some(AuditService::new(pool))` or `None` + info log |
| Exposed as a gauge | `crates/buzz-relay/src/main.rs:139` (`buzz_audit_enabled` = 1.0/0.0) |
| Test | `crates/buzz-relay/src/config.rs:1041-1047` |

The relay's doc note states it does not control the separate `moderation_actions`
audit trail (`crates/buzz-relay/src/config.rs:200`).

### Runtime configuration surface of the crate

All configuration is by constructor argument, not environment:

| Input | Where | Notes |
|---|---|---|
| `PgPool` | `AuditService::new(pool)` (`service.rs:43-45`) | the crate never builds a pool; the relay supplies one sized `max_connections(5)`, `min_connections(1)` from `config.database_url` (`crates/buzz-relay/src/main.rs:322-328`) |
| `community: CommunityId` | `verify_chain` (`service.rs:162`), `get_entries` (`service.rs:214`) | per-call tenant scope |
| `from_seq` / `to_seq` / `limit` | `service.rs:163-164`, `:215-216` | caller-supplied; no defaults or caps in the crate |

### Cargo features

`crates/buzz-audit/Cargo.toml` declares **no `[features]` section**, no
`default-features` toggles on its own dependencies, and no `[dev-dependencies]`.
Every dependency is inherited via `workspace = true`
(`crates/buzz-audit/Cargo.toml:11-22`), so feature selection is entirely the
workspace's (e.g. sqlx features `runtime-tokio, tls-rustls, postgres, uuid, chrono,
json` at `Cargo.toml:52-54`).

Package metadata is all workspace-inherited except `description`
(`crates/buzz-audit/Cargo.toml:1-9`): `description = "Hash-chain audit log for Buzz"`
(`:8`).

### Compile-time constants

| Constant | Value | Visibility | Line |
|---|---|---|---|
| `GENESIS_HASH` | `[0u8; 32]` — 32 zero bytes | `pub` (re-exported at `lib.rs:34`) | `crates/buzz-audit/src/hash.rs:9` |
| `AUDIT_LOCK_NAMESPACE` | `"buzz_audit:"` | private to `service` module | `crates/buzz-audit/src/service.rs:29` |
| `AuditAction::ALL` | slice of all 11 variants | private (`const ALL: &'static [Self]`) | `crates/buzz-audit/src/action.rs:51-63` |

Other hard-coded values that behave as configuration:

| Value | Meaning | Line |
|---|---|---|
| `6` in `trunc_subsecs(6)` | timestamp precision = microseconds, matched to `TIMESTAMPTZ` | `hash.rs:23` |
| `0` in `hashtextextended($1, 0)` | Postgres hash seed for the advisory-lock key | `service.rs:59`, `:71` |
| `1` starting `seq` | derived from `prev_seq = 0` when a community has no rows | `service.rs:108-110` |
| presence tags `1u8` / `0u8` | optional-field framing in the pre-image | `hash.rs:55`, `:58`, `:62`, `:65` |

### Crate-level lint configuration

`#![deny(unsafe_code)]`, `#![warn(missing_docs)]` (`crates/buzz-audit/src/lib.rs:1-2`).
No `#[allow(...)]` attributes appear anywhere in the crate.

### Schema/DDL configuration

The crate ships no migrations — the `audit_log` table is owned by
`migrations/0001_initial_schema.sql:606-619`, stated explicitly at
`crates/buzz-audit/src/lib.rs:17-18`. The advisory-lock namespace `'buzz_audit:'` is
registered in migration commentary alongside other lock families
(`migrations/0023_push_match_gate.sql:21`).

### Test-environment configuration

- `test_pool()` returns `Option<PgPool>` so ignored tests skip silently when Postgres
  is unreachable (`service.rs:275-280`).
- `#[ignore = "requires Postgres"]` on all 6 async tests (`service.rs:319`, `:339`,
  `:377`, `:438`, `:476`, `:513`), so `cargo test` without `--ignored` needs no infra.
- DB tests require a `communities` row to satisfy the FK; `make_community()` inserts one
  with a unique host (`service.rs:296-305`).
- A `static OnceLock<Mutex<()>>` serializes DB tests that share the table
  (`service.rs:263-267`).


## Module: buzz-media (`crates/buzz-media`)

### Aspect: Configuration

### 1. Environment variables read *inside* this crate

**None in `src/`.** A scan of all 11 source files finds zero `std::env`/`env::var` uses. `MediaConfig` is a plain `serde::Deserialize` struct (`crates/buzz-media/src/config.rs:16-17`); the caller populates it. Env reading happens only in the ignored integration test:

| Var | Default in test | file:line |
|---|---|---|
| `BUZZ_S3_ENDPOINT` | `http://localhost:9000` | `crates/buzz-media/tests/static_creds_minio.rs:23-24` |
| `BUZZ_S3_ACCESS_KEY` | `buzz_dev` | `crates/buzz-media/tests/static_creds_minio.rs:25-26` |
| `BUZZ_S3_SECRET_KEY` | `buzz_dev_secret` | `crates/buzz-media/tests/static_creds_minio.rs:27-28` |
| `BUZZ_S3_BUCKET` | `buzz-media` | `crates/buzz-media/tests/static_creds_minio.rs:29` |

Indirect env consumption: `Credentials::default()` triggers the AWS credential chain, which reads `AWS_*` variables (env, profile, web-identity/IRSA, container — including the Pod Identity `AWS_CONTAINER_CREDENTIALS_FULL_URI` + `AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE` pair the patched `aws-creds` fork adds) inside the dependency, not in this crate (`crates/buzz-media/src/storage.rs:25-33`, `:51-56`; fork rationale `Cargo.toml:163-171`).

---

### 2. `MediaConfig` fields and the env vars that populate them

Env-var names and default values come from the relay (`crates/buzz-relay/src/config.rs:614-657`); the type and any serde default come from this crate.

| Field | Type | Serde default (this crate) | Env var (relay) | Relay default |
|---|---|---|---|---|
| `s3_endpoint` | `String` | required | `BUZZ_S3_ENDPOINT` | `http://localhost:9000` |
| `s3_access_key` | `String` | required | `BUZZ_S3_ACCESS_KEY` | `buzz_dev` |
| `s3_secret_key` | `String` | required | `BUZZ_S3_SECRET_KEY` | `buzz_dev_secret` |
| `s3_bucket` | `String` | required | `BUZZ_S3_BUCKET` | `buzz-media` |
| `s3_region` | `String` | `default_s3_region()` = `"us-east-1"` (`crates/buzz-media/src/config.rs:11-13`, `:24-27`) | `BUZZ_S3_REGION`, falling back to `AWS_REGION` | `us-east-1` |
| `max_image_bytes` | `u64` | **none — required** (`crates/buzz-media/src/config.rs:39-41`) | `BUZZ_MAX_IMAGE_BYTES` | `50 * 1024 * 1024` = 52_428_800 |
| `max_gif_bytes` | `u64` | **none — required** (`crates/buzz-media/src/config.rs:42-44`) | `BUZZ_MAX_GIF_BYTES` | `10 * 1024 * 1024` = 10_485_760 |
| `max_video_bytes` | `u64` | `default_max_video_bytes()` = `524_288_000` (500 MB) (`crates/buzz-media/src/config.rs:3-5`, `:45-47`) | `BUZZ_MAX_VIDEO_BYTES` | `500 * 1024 * 1024` = 524_288_000 |
| `max_file_bytes` | `u64` | `default_max_file_bytes()` = `104_857_600` (100 MB) (`crates/buzz-media/src/config.rs:7-9`, `:48-50`) | `BUZZ_MAX_FILE_BYTES` | `100 * 1024 * 1024` = 104_857_600 |
| `public_base_url` | `String` | required | `BUZZ_MEDIA_BASE_URL` | `http://localhost:3000/media` |
| `upload_records_enabled` | `bool` | `#[serde(default)]` → `false` (`crates/buzz-media/src/config.rs:49-50`) | `BUZZ_MEDIA_UPLOAD_RECORDS` (`"true"`/`"1"`) | `false` |
| `upload_ip_header` | `Option<String>` | `#[serde(default)]` → `None` (`crates/buzz-media/src/config.rs:55-56`) | `BUZZ_MEDIA_UPLOAD_IP_HEADER` (trimmed, lowercased) | unset |
| `upload_port_header` | `Option<String>` | `#[serde(default)]` → `None` (`crates/buzz-media/src/config.rs:60-61`) | `BUZZ_MEDIA_UPLOAD_PORT_HEADER` (trimmed, lowercased) | unset |

Related relay-side knobs that shape this crate's behaviour but are not `MediaConfig` fields: `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS` (default 8), `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS_PER_PUBKEY` (default 2, clamped to the global), `BUZZ_MEDIA_UPLOADS_PER_MINUTE` (default 30) (`crates/buzz-relay/src/config.rs:659-676`), and `BUZZ_REQUIRE_MEDIA_GET_AUTH` (default `false`, `crates/buzz-relay/src/config.rs:677-685`). `.env.example` documents the first three at `.env.example:85-87` and the read-auth flags at `.env.example:90-92`.

---

### 3. Startup validation rules (`MediaConfig::validate`, `crates/buzz-media/src/config.rs:66-122`)

| Check | Failure message | file:line |
|---|---|---|
| `public_base_url` must end with `/media` | `public_base_url must end with /media: got '…'` | `crates/buzz-media/src/config.rs:67-72` |
| `public_base_url` must not end with `/` | `public_base_url must not end with /: got '…'` | `crates/buzz-media/src/config.rs:73-78` |
| `max_image_bytes > 0` | `max_image_bytes must be > 0` | `crates/buzz-media/src/config.rs:79-81` |
| `max_gif_bytes > 0` and `<= max_image_bytes` | `max_gif_bytes must be > 0 and <= max_image_bytes` | `crates/buzz-media/src/config.rs:82-84` |
| `max_video_bytes > 0` | `max_video_bytes must be > 0` | `crates/buzz-media/src/config.rs:85-87` |
| `max_file_bytes > 0` | `max_file_bytes must be > 0` | `crates/buzz-media/src/config.rs:88-90` |
| IP header set without records enabled | long message ending "Enable upload records or unset the header." | `crates/buzz-media/src/config.rs:91-102` |
| Port header set without IP header | long message ending "Set the IP header or unset the port header." | `crates/buzz-media/src/config.rs:103-110` |
| Both header names must be valid HTTP header names (`axum::http::HeaderName::from_bytes`) | `{VAR} is not a valid header name: {h:?}` | `crates/buzz-media/src/config.rs:111-124` |

The two coherence checks are justified in-code as fail-loud choices: "an operator who set an IP header believes they are meeting a reporting obligation" (`crates/buzz-media/src/config.rs:91-95`).

---

### 4. Cargo features

The crate declares **no `[features]` section** (`crates/buzz-media/Cargo.toml`), and no `#[cfg(feature = …)]` appears in the source. Feature selection is only on dependencies:

| Dependency | Features |
|---|---|
| `s3` (`rust-s3` 0.37) | `default-features = false`, `tokio-rustls-tls`, `fail-on-err`, `tags` (`crates/buzz-media/Cargo.toml:24`) |
| `image` 0.25 | `default-features = false`, `jpeg`, `png`, `gif`, `webp` (`crates/buzz-media/Cargo.toml:26`) |
| `tokio-util` 0.7 | `io` (`crates/buzz-media/Cargo.toml:31`) |
| `tokio` (dev) | `test-util` (`crates/buzz-media/Cargo.toml:33-34`) |

Version/edition/license/rust-version are all inherited via `workspace = true` (`crates/buzz-media/Cargo.toml:3-8`).

---

### 5. Compile-time constants (all values)

| Constant | Value | Applies to | file:line |
|---|---|---|---|
| `ALLOWED_MIME_TYPES` | `["image/jpeg", "image/png", "image/gif", "image/webp"]` | image upload allowlist | `crates/buzz-media/src/validation.rs:15` |
| `MP4_BRANDS` | `isom iso2 iso3 iso4 iso5 iso6 iso7 iso8 iso9 mp41 mp42 avc1 dash "M4V "` (14 brands) | ISO-BMFF brand acceptance | `crates/buzz-media/src/validation.rs:17-20` |
| `BLOCKED_FILE_MIME_TYPES` | 14 entries: `text/html`, `application/xhtml+xml`, `image/svg+xml`, `application/javascript`, `text/javascript`, `application/x-msdownload`, `application/x-executable`, `application/vnd.microsoft.portable-executable`, `application/x-mach-binary`, `application/x-sharedlib`, `application/x-elf`, `application/x-msi`, `application/vnd.android.package-archive`, `application/x-apple-diskimage` | generic-file deny-list | `crates/buzz-media/src/validation.rs:75-95` |
| `MAX_PIXELS` | `25_000_000` (25 MP; comment: "100MB max RGBA decode") | image bomb guard | `crates/buzz-media/src/validation.rs:269` |
| duration limit | `600.0` seconds (literal, not a named const) | video length | `crates/buzz-media/src/validation.rs:357` |
| resolution limit | `3840` × `2160` (literals) | video resolution | `crates/buzz-media/src/validation.rs:364` |
| `MAX_ATOMS` | `1024` | top-level MP4 atoms scanned in the moov-order check | `crates/buzz-media/src/validation.rs:413` |
| `MAX_BOXES` | `100_000` | total boxes walked in the MP4 metadata check | `crates/buzz-media/src/validation.rs:832` |
| `MAX_BOX_DEPTH` | `32` | MP4 box nesting depth | `crates/buzz-media/src/validation.rs:833` |
| `EMPTY_FFMPEG_UDTA` | 53-byte literal sequence (`meta`/`hdlr`/`mdirappl`/`ilst`) | the only permitted `udta` payload | `crates/buzz-media/src/validation.rs:834-839` |
| `FORBIDDEN` (MP4 boxes) | `meta ilst keys data uuid "xml " bxml loci \xa9xyz name chap` (11) | metadata box denial | `crates/buzz-media/src/validation.rs:840-852` |
| `CONTAINERS` (MP4) | `moov trak mdia minf stbl edts dinf sinf schi` (9) | boxes walked recursively | `crates/buzz-media/src/validation.rs:853-855` |
| `ALLOWED` (MP4 boxes) | 35 box types (`ftyp moov mdat free skip wide trak mdia minf stbl edts dinf sinf schi udta mvhd tkhd mdhd hdlr vmhd smhd dref "url " "urn " stsd stts stss ctts stsc stsz stco co64 sgpd sbgp elst`) | box allowlist | `crates/buzz-media/src/validation.rs:856-864` |
| `PNG_SNAPSHOT_KEYWORDS` | `["buzz_agent_snapshot", "buzz_team_snapshot"]` | permitted `tEXt` keywords | `crates/buzz-media/src/validation.rs:579` |
| thumbnail max dimension | `320` × `320` (literals, aspect preserved) | thumbnail size | `crates/buzz-media/src/thumbnail.rs:30` |
| blurhash components | `4`, `3` (x, y) | blurhash resolution | `crates/buzz-media/src/thumbnail.rs:37` |
| thumbnail format | `ImageFormat::Jpeg` | thumbnail encoding | `crates/buzz-media/src/thumbnail.rs:31` |
| `MIN_SNIFF_BYTES` | `4096` | leading bytes retained for MP4 magic detection during streaming | `crates/buzz-media/src/upload.rs:351` |
| video read buffer | `64 * 1024` (64 KiB) | streaming chunk size | `crates/buzz-media/src/upload.rs:353` |
| buffered auth window | `600` seconds | image/file Blossom `created_at` age | `crates/buzz-media/src/upload.rs:85` |
| video auth window | `3600` seconds | video Blossom `created_at` age | `crates/buzz-media/src/upload.rs:412` |
| clock-skew tolerance | `5` seconds (future `created_at`) | Blossom auth | `crates/buzz-media/src/auth.rs:117` |
| Blossom auth kind | `24242` | auth event kind check | `crates/buzz-media/src/auth.rs:42` |
| `BUF` | `8 * 1024 * 1024` (8 MiB) | `put_file` read buffer | `crates/buzz-media/src/storage.rs:91` |
| `UPLOAD_RECORD_VERSION` | `1` | upload-record schema version | `crates/buzz-media/src/upload_record.rs:52` |
| sidecar key prefix | `_meta/` | tenant metadata namespace | `crates/buzz-media/src/storage.rs:184` |
| record key prefix | `_uploads/` | moderation namespace | `crates/buzz-media/src/upload_record.rs:182` |
| thumb key suffix | `.thumb.jpg` | derived artifact naming | `crates/buzz-media/src/upload.rs:531` |
| sha256 hex length | `64`, charset `[0-9a-f]` | key classification | `crates/buzz-media/src/bucket_index.rs:75-79` |
| blob ext bounds | 1–8 ASCII alphanumeric (uppercase allowed) | key classification | `crates/buzz-media/src/bucket_index.rs:84-87` |
| ULID length | `26`, uppercase Crockford base32 | record key classification | `crates/buzz-media/src/bucket_index.rs:93-105` |
| UUID form | exactly 36 chars, lowercase hex + hyphens at 8/13/18/23 | community segment parsing | `crates/buzz-media/src/bucket_index.rs:112-127` |

No concurrency-limit constants exist in this crate (semaphore sizes live in the relay); the only caller-supplied bounds are the sweep `cap` and `max_keys` arguments (`crates/buzz-media/src/bucket_index.rs:377-379`, `crates/buzz-media/src/storage.rs:242-246`).

---

### 6. Verification: where does "50 MB" actually live?

ARCHITECTURE.md states "PUT `/media/upload` — Upload media blob (Blossom, 50 MB limit)" (`ARCHITECTURE.md:624`). In this crate, 50 MB is **not** a production constant:

| Occurrence | Context |
|---|---|
| `crates/buzz-media/src/config.rs:34` | doc comment only: "Maximum upload size for images (bytes). Default: 50 MB." — the field itself has no serde default (`crates/buzz-media/src/config.rs:41`) |
| `crates/buzz-media/src/upload.rs:573` | test fixture |
| `crates/buzz-media/src/validation.rs:952`, `:1590` | test fixtures |
| `crates/buzz-media/src/storage.rs:288` | test fixture |
| `crates/buzz-media/tests/static_creds_minio.rs:33` | ignored integration test fixture |

The only place 50 MB is a real runtime default is the relay: `BUZZ_MAX_IMAGE_BYTES` → `unwrap_or(50 * 1024 * 1024)` (`crates/buzz-relay/src/config.rs:626-629`). It also applies to **images only** — video (500 MB) and generic files (100 MB) have larger caps, and the relay's request-body limit layer uses `max(max_image_bytes, max_video_bytes)` (`crates/buzz-relay/src/router.rs:33-45`).


## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Configuration

---

### 1. Environment variables

All three are read inside `add_reaction_impl` and only exist under `#[cfg(feature = "reqwest")]`. There is no config struct wiring, no `.env` loading, and no other `std::env::var` call in the crate (verified by grep).

| Variable | Read at | Default | Purpose |
|---|---|---|---|
| `BUZZ_RELAY_BASE_URL` | `executor.rs:889-890` | `"http://localhost:3000"` | Base URL for the reaction POST |
| `BUZZ_API_TOKEN` | `executor.rs:901` | none | If present, sent as `Authorization: Bearer {token}` |
| `BUZZ_RELAY_PUBKEY` | `executor.rs:903` | none | Fallback when `BUZZ_API_TOKEN` is absent, sent as `X-Pubkey` |

If neither auth variable is set, the request goes out unauthenticated (`executor.rs:901-905`).

---

### 2. Cargo features

| Feature | Definition | Effect | Enabled by |
|---|---|---|---|
| `reqwest` | `reqwest = ["dep:reqwest"]` (`Cargo.toml:28-29`), dependency declared `optional = true` (`Cargo.toml:27`) | Compiles the real HTTP paths: `check_ssrf` (`executor.rs:744-776`), `WEBHOOK_MAX_RESPONSE_BYTES` (`executor.rs:777-778`), `call_webhook_impl` (`executor.rs:780-869`), `shared_http_client` (`executor.rs:873-885`), `add_reaction_impl` (`executor.rs:887-933`). Disabled ⇒ `call_webhook` returns `{status:0, body:null, skipped:true}` (`executor.rs:636-647`) and `add_reaction` returns `{added:false, skipped:true}` (`executor.rs:606-616`) | `buzz-relay` (`crates/buzz-relay/Cargo.toml:63`: `features = ["reqwest"]`). `buzz-admin` depends without it (`crates/buzz-admin/Cargo.toml:21`) |

No default features are declared, so a bare `cargo build -p buzz-workflow` compiles the skip-stub variants.

---

### 3. Runtime configuration struct — `WorkflowConfig`

`lib.rs:57-71`. Every construction site in the repo uses `WorkflowConfig::default()`; no environment variable or CLI flag overrides these values.

| Field | Type | Default | Consumed at |
|---|---|---|---|
| `max_concurrent` | `usize` | `100` (`lib.rs:68`) | semaphore permits, `max(1)`-clamped (`lib.rs:110-111`) |
| `default_timeout_secs` | `u64` | `300` (`lib.rs:69`) | per-step timeout fallback when `step.timeout_secs` is absent (`executor.rs:1136-1138`) |

Construction sites: `crates/buzz-relay/src/main.rs:389-390` (production), `crates/buzz-relay/src/state.rs:1274-1276`, plus ~8 relay test fixtures — all `::default()`.

---

### 4. Compile-time constants (complete list, with values)

| Constant | Value | Where | Meaning |
|---|---|---|---|
| `EVAL_TIMEOUT` | `Duration::from_millis(100)` | `executor.rs:342` | Wall-clock cap on one evalexpr evaluation (does not cancel the blocking thread — `executor.rs:358-360`) |
| `MAX_EXPR_LEN` | `4096` (bytes) | `executor.rs:362` (fn-local const in `evaluate_condition`) | Rejects long condition expressions before evaluation |
| `MAX_DELAY_SECS` | `270` (seconds) | `executor.rs:676` (fn-local const in `dispatch_action`) | Upper bound on the `delay` action; chosen to stay under the 300 s default step timeout |
| `WEBHOOK_MAX_RESPONSE_BYTES` | `1024 * 1024` = 1 048 576 (1 MiB) | `executor.rs:778` (`#[cfg(feature = "reqwest")]`) | Outbound webhook response body cap |

---

### 5. Hard-coded values that behave like configuration (not named constants)

| Value | Where | Meaning |
|---|---|---|
| Semaphore permits = `config.max_concurrent.max(1)` → 100 by default | `lib.rs:110-111` | Concurrency admission; `try_acquire()` so overflow is rejected, not queued (`executor.rs:978`, `executor.rs:1028`) |
| Workflow cache `max_capacity(10_000)` | `lib.rs:119` | Max cached `(community_id, channel_id)` entries |
| Workflow cache `time_to_live(10s)` | `lib.rs:120` | TTL, documented as matching the relay's other moka caches and bounding cross-pod staleness (`lib.rs:92-103`) |
| Scheduler tick `sleep(Duration::from_secs(60))` | `lib.rs:432` | Cron/interval loop period; sleep happens before the first scan |
| Cron match window `60` (passed as `window_secs`) | `lib.rs:484` → `lib.rs:688-706` | Drift-tolerant lookback window for cron matching |
| Minimum allowed interval = `60` seconds | `schema.rs:220-224` | Definition-time rejection of sub-minute intervals, justified by the 60 s tick |
| HTTP client timeout `Duration::from_secs(10)` | `executor.rs:807` (per-request webhook client), `executor.rs:879` (shared reaction client) | Total request timeout |
| Redirect policy `Policy::none()` | `executor.rs:812` | Webhook redirects disabled |
| Default HTTP method `"POST"` | `executor.rs:621` | `call_webhook` when `method` is omitted |
| Default port fallback `80` | `executor.rs:798` | Used when `port_or_known_default()` yields `None` |
| Approval timeout default string `"24h"` | `executor.rs:653` | **Log-only.** The string is never parsed into a duration and no expiry is persisted, because no approval record is created (`executor.rs:663`). The documented "Defaults to 24h" in the schema (`schema.rs:138-140`) has no runtime effect today |
| Step-ID limits: non-empty, ≤ `64` chars, `[A-Za-z0-9_]` | `schema.rs:168-172` | Definition-time validation protecting evalexpr variable names |
| Cron field normalization: 5→7, 6→7 fields | `schema.rs:250-257` | `0 {expr} *` / `{expr} *` |
| Reaction `e`-tag length check `64` hex chars | `lib.rs:924-928` | Distinguishes hex event IDs from UUID channel refs when resolving `trigger.message_id` |

---

### 6. Not configurable

- The four evalexpr helper functions are registered unconditionally with no allow/deny list (`executor.rs:236-283`).
- The SSRF blocklist is compiled into `buzz-core` (`crates/buzz-core/src/network.rs:46-95`); there is no allowlist/bypass hook for internal webhook targets. Still true after `c26bf594` widened the blocklist — the added prefixes are also compile-time constants (`network.rs:7`, `:10`), not configuration.
- Scheduler tick period, cache TTL/capacity, delay cap, expression cap, evaluation timeout, HTTP timeouts, and the response cap are all literals — none are surfaced through `WorkflowConfig` or environment variables.


## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Configuration

**93 distinct environment variables** are read by the files in this group: 74 in `config.rs`, 15 in `main.rs` (14 explicit + `RUST_LOG` via `EnvFilter`), 1 in `nip11.rs`, 3 in `telemetry.rs` (2 explicit + `OTEL_RESOURCE_ATTRIBUTES` via the SDK detector). A further 4 are read by `storage_sweep.rs` and consumed only from `main.rs` (§4).

There is no config file, no CLI flag parsing, and no config reload. Everything is env-only, read once at boot — except two vars read **per request/per tick** (§5).

---

#### 1. `config.rs` — every env var (74)

##### 1a. Listeners and identity

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_BIND_ADDR` | `SocketAddr` | `0.0.0.0:3000` | `config.rs:407` | `main.rs:1157` (TCP bind). Invalid ⇒ **hard error** `InvalidBindAddr` (`config.rs:265-268`) |
| `RELAY_URL` | `String` | `ws://localhost:3000` | `config.rs:428` | `main.rs:239` (deployment community), `main.rs:568`, `nip11.rs:241`, `tenant.rs:130`. **Never validated as a URL** |
| `BUZZ_PAIRING_RELAY_URL` | `Option<String>` | `None` | `config.rs:430` | `nip11.rs:243` only. Must be `ws`/`wss` with a host ⇒ else hard error (`config.rs:432-446`) |
| `BUZZ_UDS_PATH` | `Option<String>` | `None` | `config.rs:604` | `main.rs:1162-1187`; a `warn` on non-unix (`main.rs:1207-1211`) |
| `BUZZ_HEALTH_PORT` | `u16` | `8080` | `config.rs:609` | `main.rs:1116` — bound to hard-coded `0.0.0.0`, ignores `BUZZ_BIND_ADDR`'s IP |
| `BUZZ_METRICS_PORT` | `u16` | `9102` | `config.rs:614` | `main.rs:138` → `metrics.rs:74` — also hard-coded `0.0.0.0` |
| `BUZZ_RELAY_PRIVATE_KEY` | `Option<String>` | `None` | `config.rs:602` | `main.rs:393-394` (parse or abort), `nip11.rs:302` (gates `self` + NIP-43). **Not validated at config load** — only when `main` parses it |
| `RELAY_OWNER_PUBKEY` | `Option<String>` (64 hex) | `None` | `config.rs:526` | `main.rs:201`, `main.rs:292-294`. Invalid ⇒ **warn-and-ignore, value echoed in the log** (`config.rs:538-542`) |

##### 1b. Datastores

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `DATABASE_URL` | `String` | `postgres://buzz:buzz_dev@localhost:5432/buzz` | `config.rs:410` | `main.rs:146` (main pool), `main.rs:328` (audit pool), `main.rs:375` (search fallback) |
| `READ_DATABASE_URL` | `Option<String>` | `None`; blank/whitespace normalised to `None` (`config.rs:413-416`) | `config.rs:413` | `main.rs:147` (replica pool), `main.rs:373` (search pool preference), `main.rs:384` (log) |
| `REDIS_URL` | `String` | `redis://localhost:6379` | `config.rs:419` | `main.rs:337`, `main.rs:344` |
| `BUZZ_REDIS_POOL_SIZE` | `usize` >0 | `16` | `config.rs:421` | `main.rs:338`. Zero/junk ⇒ silent fallback to 16 (tested `config.rs:988-1011`) |

##### 1c. Connection limits

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_MAX_CONNECTIONS` | `usize` | `10_000` | `config.rs:449` | `state.rs:649` → `conn_semaphore` (`state.rs:729`) |
| `BUZZ_MAX_CONCURRENT_HANDLERS` | `usize` | `1_024` | `config.rs:454` | `state.rs:650` → `handler_semaphore` (`state.rs:730`) |
| `BUZZ_SEND_BUFFER` | `usize` | `1_000` | `config.rs:459` | `connection.rs:159` |
| `BUZZ_MAX_FRAME_BYTES` | `usize` >0 | `524_288` (`config.rs:15`) | `config.rs:464` | `router.rs:302` → `router.rs:330-331`, `nip11.rs:242` (advertised `max_message_length`) |
| `BUZZ_SLOW_CLIENT_GRACE_LIMIT` | `u8` | `15` | `config.rs:470` | `connection.rs:179`, `connection.rs:212` |

##### 1d. Auth / membership

| Var | Grammar | Default | Read | Consumed |
|-----|---------|---------|------|----------|
| `BUZZ_REQUIRE_AUTH_TOKEN` | `=="true"\|\|=="1"` | `false` | `config.rs:475` | `api/bridge.rs`, `main.rs:395` (dev-key gate), `main.rs:409` (panic gate). Emits a startup `warn` when false (`config.rs:590-594`) |
| `BUZZ_PUBKEY_ALLOWLIST` | `=="true"\|\|=="1"` | `false` | `config.rs:479` | `handlers/auth.rs:187` |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | `=="true"\|\|=="1"` | `false` | `config.rs:483` | 22 sites; `main.rs:201/214/242/250/276/296/506`, `nip11.rs:304` |
| `BUZZ_ALLOW_NIP_OA_AUTH` | `=="true"\|\|=="1"` | `false` | `config.rs:520` | `api/mod.rs:81` |
| `RELAY_OPERATOR_PUBKEYS` | CSV of 64-hex | empty (⇒ provisioning disabled) | `config.rs:555` | `api/operator.rs:91`, `handlers/community_provisioning.rs:261`. Invalid entry ⇒ **hard error, entry echoed** (`config.rs:566-571`); deduped |
| `RELAY_OPERATOR_API_ORIGIN` | http(s) origin, no creds/path/query/fragment | `None` | `config.rs:549` | `api/operator.rs:70`. **Required** when `RELAY_OPERATOR_PUBKEYS` is non-empty (`config.rs:577-582`) |

##### 1e. Web / CORS / admin / policy

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_CORS_ORIGINS` | CSV | empty ⇒ **permissive CORS** | `config.rs:595` | `router.rs:190` → `router.rs:409-432` |
| `BUZZ_WEB_DIR` | `Option<PathBuf>` | `None` | `config.rs:843` | `router.rs:145-183`, `router.rs:320-330`. Must contain `index.html` ⇒ else hard error (`config.rs:850-856`) |
| `BUZZ_SERVE_GIT_WEB_GUI` | `=="true"\|\|=="1"` | `false` | `config.rs:848` | `router.rs:143`, `router.rs:320` |
| `BUZZ_ADMIN_HOST` | exact authority | `None` ⇒ admin surface absent | `config.rs:814` | `router.rs:47`, `api/admin::is_admin_host`. Rejects `/ \ @` (`config.rs:819-823`) |
| `BUZZ_ADMIN_WEB_DIR` | `Option<PathBuf>` | `None` | `config.rs:826` | `router.rs:49-52`, `router.rs:265-271`. Must contain `index.html` (`config.rs:830-836`) |
| `BUZZ_TERMS_OF_SERVICE_MARKDOWN` | `Option<String>` ≤256 KiB | `None` | `config.rs:790` (helper `:776`) | `api/invites::join_policy*` via `config.join_policy` |
| `BUZZ_PRIVACY_POLICY_MARKDOWN` | `Option<String>` ≤256 KiB | `None` | `config.rs:791` | same |
| `BUZZ_AGE_ATTESTATION_REQUIRED` | `parse_bool` strict | `false` | `config.rs:792` (helper `:379`) | same; invalid ⇒ hard error (tested `config.rs:1074-1090`) |

##### 1f. Media

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_S3_ENDPOINT` | `String` | `http://localhost:9000` | `config.rs:620` | `state.rs:695` (git store), `buzz_media::MediaStorage` |
| `BUZZ_S3_ACCESS_KEY` | `String` | `buzz_dev` | `config.rs:622` | same |
| `BUZZ_S3_SECRET_KEY` | `String` | `buzz_dev_secret` | `config.rs:624` | same |
| `BUZZ_S3_BUCKET` | `String` | `buzz-media` | `config.rs:626` | same |
| `BUZZ_S3_REGION` | `String` | falls back to `AWS_REGION`, then `us-east-1` | `config.rs:627` | same |
| `AWS_REGION` | `String` | — (fallback only) | `config.rs:628` | region resolution |
| `BUZZ_MAX_IMAGE_BYTES` | `u64` | `52_428_800` (50 MB) | `config.rs:630` | `router.rs:34` (body limit), media validation |
| `BUZZ_MAX_GIF_BYTES` | `u64` | `10_485_760` (10 MB) | `config.rs:634` | media validation |
| `BUZZ_MAX_VIDEO_BYTES` | `u64` | `524_288_000` (500 MB) | `config.rs:638` | `router.rs:35`, media validation |
| `BUZZ_MAX_FILE_BYTES` | `u64` | `104_857_600` (100 MB) | `config.rs:642` | media validation |
| `BUZZ_MEDIA_BASE_URL` | `String` | `http://localhost:3000/media` | `config.rs:646` | media URL generation |
| `BUZZ_MEDIA_UPLOAD_RECORDS` | `=="true"\|\|=="1"` | `false` | `config.rs:651` | `MediaConfig::validate` coherence check (`main.rs:417`) |
| `BUZZ_MEDIA_UPLOAD_IP_HEADER` | `Option<String>` lowercased | `None` | `config.rs:654` | same |
| `BUZZ_MEDIA_UPLOAD_PORT_HEADER` | `Option<String>` lowercased | `None` | `config.rs:658` | same |
| `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS` | `usize` >0 | `8` | `config.rs:664` | `state.rs:693` → `media_upload_semaphore` |
| `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS_PER_PUBKEY` | `u32` >0 | `2`, clamped to the above | `config.rs:670` | `api/media.rs` |
| `BUZZ_MEDIA_UPLOADS_PER_MINUTE` | `u32` >0 | `30` | `config.rs:676` | `api/media.rs` |
| `BUZZ_REQUIRE_MEDIA_GET_AUTH` | `true\|1\|yes\|on` (ci) | `false` | `config.rs:682` | `api/media.rs` GET/HEAD gate |

##### 1g. Git

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_GIT_REPO_PATH` | `PathBuf` (created) | `./repos` | `config.rs:705` | `api/git/*`; `create_dir_all` failure ⇒ hard error (`config.rs:384-397`) |
| `BUZZ_GIT_PACK_CACHE_PATH` | `PathBuf` (created) | `<git_repo_path>/.pack-cache` | `config.rs:709` | `state.rs:704` |
| `BUZZ_GIT_MAX_PACK_BYTES` | `u64` | `524_288_000` (500 MB) | `config.rs:713` | `api/git/transport.rs:1757` (body limit) |
| `BUZZ_GIT_MAX_REPO_BYTES` | `u64` | `git_max_pack_bytes * 2` (1 GB) | `config.rs:717` | `api/git/hydrate.rs` |
| `BUZZ_GIT_PACK_CACHE_MAX_BYTES` | `u64` | `git_max_repo_bytes * 5` (5 GB) | `config.rs:721` | `state.rs:705` |
| `BUZZ_GIT_PACK_CACHE_MAX_CONCURRENT_POPULATIONS` | `usize` >0 | `2` | `config.rs:726` | `state.rs:706` |
| `BUZZ_GIT_MAX_REPOS_PER_PUBKEY` | `u32` | `100` | `config.rs:731` | `handlers/side_effects.rs:2487` |
| `BUZZ_GIT_MAX_CONCURRENT_OPS` | `usize` | `20` | `config.rs:735` | `state.rs:692` → `git_semaphore` |
| `BUZZ_GIT_HOOK_HMAC_SECRET` | `String` ≥32 when set | random 32-byte hex **per process** | `config.rs:739`, re-read `config.rs:865` | `api/git/policy.rs`, `api/git/hook.rs` |

##### 1h. Push (NIP-PL)

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_PUSH_EXECUTOR_KEY_ID` | `String` 1..=64 | `relay-v1` | `config.rs:746` | `nip11.rs:241`. Empty or >64 ⇒ hard error (`config.rs:747-751`) |
| `BUZZ_PUSH_GATEWAY_DELIVERY_URL` | exact `https://…/v1/deliveries/apns` | **`https://push.buzz.xyz/v1/deliveries/apns`** (`config.rs:332`) — explicit empty string disables | `config.rs:752` | `main.rs:686` (worker gate), `nip11.rs:244/251`, `push_runtime.rs` |
| `BUZZ_PUSH_GATEWAY_TIMEOUT_MS` | `u64` in `100..=10000` | `2000` | `config.rs:759` | `push_runtime.rs:314`. Out of range ⇒ hard error (`config.rs:763-771`) |

##### 1i. Mesh

| Var | Grammar | Default | Read | Consumed |
|-----|---------|---------|------|----------|
| `BUZZ_MESH` | `on` (ci) \| `true` \| `1` | off | `config.rs:498` | `mesh_boot::boot_mesh` (`main.rs:442`) |
| `BUZZ_MESH_BIND_ADDR` | `SocketAddr` | `0.0.0.0:3478` | `config.rs:501` | `boot_mesh`. Invalid ⇒ hard error (`config.rs:502-506`) |
| `BUZZ_MESH_DEMO_ECHO` | `on` (ci) \| `true` \| `1` | off | `config.rs:516` | `main.rs:456` (`wire_consumers`), `api/mesh_demo.rs` |

##### 1j. Misc

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_HUDDLE_AUDIO_AVAILABLE` | `!(=="false"\|\|=="0")` (**inverted**) | `true` | `config.rs:489` | `audio/handler.rs:357` |
| `BUZZ_AUDIT_ENABLED` | `parse_bool` strict | `true` | `config.rs:793` | `main.rs:130/139/323`, `state.rs:763`. Invalid ⇒ hard error (tested `config.rs:1056-1072`) |
| `BUZZ_EPHEMERAL_TTL_OVERRIDE` | `i32` >0 | `None` | `config.rs:691` | `handlers/ingest.rs:2125`, `handlers/side_effects.rs:1681`. Emits a startup `warn` when set (`config.rs:696-702`) |

##### 1k. Rate limits (7 vars → `buzz_auth::RateLimitConfig`)

Read by `rate_limit_config_from_env` (`config.rs:284-317`) via `positive_u64_from_env` (`config.rs:270-282`). **All 7 hard-error on zero, junk, or non-Unicode** — no silent fallback.

| Var | Read | `RateLimitConfig` field | Production consumer |
|-----|------|-------------------------|---------------------|
| `BUZZ_RATE_LIMIT_HUMAN_MESSAGES_PER_MIN` | `config.rs:288` | `human_messages_per_min` | `connection.rs:636` |
| `BUZZ_RATE_LIMIT_HUMAN_API_CALLS_PER_MIN` | `config.rs:292` | `human_api_calls_per_min` | `api/bridge.rs:29` |
| `BUZZ_RATE_LIMIT_HUMAN_WS_EVENTS_PER_SEC` | `config.rs:296` | `human_ws_events_per_sec` | `connection.rs:614` |
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_MESSAGES_PER_MIN` | `config.rs:300` | `agent_standard_messages_per_min` | `connection.rs:634` |
| **`BUZZ_RATE_LIMIT_AGENT_STANDARD_API_CALLS_PER_MIN`** | `config.rs:304` | `agent_standard_api_calls_per_min` | **NONE — parsed, validated, stored, never read** |
| **`BUZZ_RATE_LIMIT_AGENT_ELEVATED_MESSAGES_PER_MIN`** | `config.rs:308` | `agent_elevated_messages_per_min` | **NONE** |
| **`BUZZ_RATE_LIMIT_AGENT_PLATFORM_MESSAGES_PER_MIN`** | `config.rs:312` | `agent_platform_messages_per_min` | **NONE** |

Confirmed by workspace grep across `crates/**` and `desktop/src-tauri/**`: the only occurrences of the three field names outside `config.rs` are their declarations (`crates/buzz-auth/src/rate_limit.rs:101,104,107`) and their default assignments (`:139,140,141`). No reader. **3 of 7 rate-limit vars are dead** — an operator setting them gets no effect and no warning. Two of the three name a tier ("elevated", "platform") for which no enforcement path exists at all.

---

#### 2. `main.rs` — 15 env vars read outside `Config`

These bypass `Config` entirely, so they are invisible to `Config`'s tests, invisible to the startup "Config loaded" log (`main.rs:128-136`), and not covered by any of `config.rs`'s validation helpers.

| Var | Type | Default | Read | Consumed | Invalid-value policy |
|-----|------|---------|------|----------|----------------------|
| `RUST_LOG` | filter directives | none (plus forced `buzz_relay=info`) | `main.rs:111` (`EnvFilter::from_default_env`) | log filtering | tracing-subscriber default |
| `BUZZ_AUTO_MIGRATE` | `true\|1\|yes\|on` trimmed/ci | off | `main.rs:162` (helper `main.rs:25-33`) | `main.rs:163-172` | anything else ⇒ off |
| `BUZZ_RECONCILE_CHANNELS` | **presence only** (`is_ok()`) | absent | `main.rs:549` | `main.rs:549-590` | any value incl. empty ⇒ on |
| `BUZZ_GIT_CONFORMANCE_PROBE` | `!= "false"` | on | `main.rs:469` | `main.rs:466-503` | anything but `false` ⇒ on |
| `BUZZ_GIT_PROBE_WRITERS` | `usize` | `32` | `main.rs:473` | `main.rs:485` | silent default |
| `BUZZ_GIT_PROBE_ROUNDS` | `usize` | `3` | `main.rs:477` | `main.rs:486` | silent default |
| `BUZZ_NIP43_RECONCILE_INTERVAL_SECS` | `u64` `.max(1)` | `60` | `main.rs:517` | `main.rs:523` | silent default |
| `BUZZ_REAPER_INTERVAL_SECS` | `u64` | `60` | `main.rs:609` | `main.rs:619` | silent default; **no floor — `0` yields a busy loop** |
| `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS` | `u64` | `10` | `main.rs:701` | `main.rs:721` | silent default; **no floor** |
| `SPROUT_REMINDER_SCHEDULER_BATCH_LIMIT` | `i64` | `100` | `main.rs:705` | `main.rs:727` | silent default |
| `BUZZ_COMMUNITY_REVALIDATE_INTERVAL_SECS` | `u64` `.clamp(1,300)` | `30` | `main.rs:882` | `main.rs:889` | silent default + clamp |
| `BUZZ_POOL_METRICS_INTERVAL_SECS` | `u64` `.max(1)` | `10` | `main.rs:945` | `main.rs:951` | silent default; floored because `interval` panics on zero (`main.rs:949`) |
| `BUZZ_USAGE_METRICS_INTERVAL_SECS` | `u64` `.max(5)` | `300` | `main.rs:1258` | `main.rs:133`, `main.rs:1013/1021` | silent default + floor |
| `BUZZ_USAGE_METRICS_IDLE_TIMEOUT_SECS` | `u64` | `900`, raised to `interval*3` | `main.rs:1267` | `main.rs:138` → `metrics.rs:77` | silent default |
| `BUZZ_USAGE_METRICS_PER_COMMUNITY` | `""\|all\|off` | `all` | `main.rs:58` | `main.rs:1002`, `main.rs:1335/1509/1467` | unknown ⇒ `warn` + `all` (`main.rs:66-72`) |

Two of these have **no lower bound** (`BUZZ_REAPER_INTERVAL_SECS`, `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS`), unlike the three neighbours that do — a `0` turns the corresponding task into a `sleep(0)` hot loop hammering Postgres. The two adjacent pollers were explicitly floored for exactly this reason (`main.rs:949`, `main.rs:1261`).

---

#### 3. `nip11.rs` and `telemetry.rs`

| Var | Type | Default | Read | Consumed | Note |
|-----|------|---------|------|----------|------|
| `SPROUT_MAX_NOT_BEFORE_DELTA` | `u64` | `31_536_000` (1 y) | `nip11.rs:97` | `nip11.rs:113` advertised `max_not_before_delta` | **read on every NIP-11 request**, not cached in `Config`; legacy `SPROUT_` prefix |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | presence | unset ⇒ tracing disabled | `telemetry.rs:81` | `telemetry.rs:79-90` | value itself is consumed by the OTLP SDK, not by this code |
| `OTEL_SERVICE_NAME` | non-empty `String` | `buzz-relay` | `telemetry.rs:50` | `telemetry.rs:56-59` | empty string falls back (tested `telemetry.rs:157-171`) |
| `OTEL_RESOURCE_ATTRIBUTES` | OTEL spec | — | `telemetry.rs:53` (`EnvResourceDetector::new()`) | overlaid **last**, so a `service.name` there wins over `OTEL_SERVICE_NAME` (`telemetry.rs:32`, `:48-52`) | |

**Documented-but-unimplemented (verified delta):** `telemetry.rs:23-24` lists `OTEL_TRACES_SAMPLER` (documented default `parentbased_always_on`) and `OTEL_TRACES_SAMPLER_ARG` as "Standard OTEL env vars honoured". No code in this crate reads either — `try_init_tracer` (`telemetry.rs:79-90`) and `classify_exporter_result` (`telemetry.rs:96-113`) build the provider with only `.with_resource(...)` and `.with_batch_exporter(...)`, never `.with_sampler(...)`. Whether they take effect therefore depends entirely on `opentelemetry_sdk` 0.32 internals (`Cargo.toml:78` → workspace `Cargo.toml:78`). I could not verify the SDK's behaviour offline; treat the doc claim as unproven and the sampler as effectively unconfigurable from this crate.

---

#### 4. Consumed by `main.rs` but declared in `storage_sweep.rs`

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_STORAGE_SWEEP_INTERVAL_SECS` | `u64` `.max(60)` | `3600` | `storage_sweep.rs:56` | `main.rs:1454` → `main.rs:1457` |
| `BUZZ_STORAGE_SWEEP_TIMEOUT_SECS` | `u64` | `120` | `storage_sweep.rs:61` | `main.rs:1458` |
| `BUZZ_STORAGE_SWEEP_MAX_OBJECTS` | `u64` | `1_000_000` | `storage_sweep.rs:65` | `main.rs:1454`, `main.rs:1462` |
| `BUZZ_STORAGE_METRICS` | `off` (trimmed, ci) disables; anything else incl. unset enables | enabled | `storage_sweep.rs:69` | `main.rs:1452` |

These live in a function-local `OnceLock` (`main.rs:1444-1451`) rather than on `Config`/`AppState`, with an explicit rationale: localized feature config, read once on the first leader tick, stable for the process lifetime.

---

#### 5. Config read outside boot (2 vars)

Everything else is read once. Two are not:

| Var | Frequency | Cite | Consequence |
|-----|-----------|------|-------------|
| `SPROUT_MAX_NOT_BEFORE_DELTA` | **every NIP-11 request** (`GET /`, `GET /info`) | `nip11.rs:96-100` | a `std::env::var` + parse per unauthenticated request; inconsistent with every other advertised limit, which comes from `Config` |
| `BUZZ_STORAGE_*` / `BUZZ_STORAGE_METRICS` | first leader tick only, then cached | `main.rs:1444-1451` | documented and intentional |

---

#### 6. Parsed-but-never-consumed summary (the known defect class)

| Item | Where parsed | Reader |
|------|--------------|--------|
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_API_CALLS_PER_MIN` | `config.rs:303-306` | **none** |
| `BUZZ_RATE_LIMIT_AGENT_ELEVATED_MESSAGES_PER_MIN` | `config.rs:307-310` | **none** |
| `BUZZ_RATE_LIMIT_AGENT_PLATFORM_MESSAGES_PER_MIN` | `config.rs:311-314` | **none** |

All three are fields of `buzz_auth::RateLimitConfig` nested inside `Config.auth` (`config.rs:82`), so the dead surface is *inside* a live field — which is why a top-level field audit misses it.

At the top level, every one of `Config`'s **51** fields (`config.rs:53-262`) has at least one production consumer — verified individually by grepping each field name across `crates/**` and `desktop/src-tauri/**` and discarding `#[cfg(test)]` hits. Two are consumed only through a continuation-line access that a naive `config.X` grep misses: `relay_operator_api_origin` (`api/operator.rs:70`) and `relay_operator_pubkeys` (`api/operator.rs:91`, `handlers/community_provisioning.rs:261`); both are live.

The near-misses worth recording:

| Field | Only production consumer | Note |
|-------|--------------------------|------|
| `pairing_relay_url` | `nip11.rs:243` | advertisement only — the relay never dials it |
| `pubkey_allowlist_enabled` | `handlers/auth.rs:187` | single site |
| `huddle_audio_available` | `audio/handler.rs:357` | single site |
| `allow_nip_oa_auth` | `api/mod.rs:81` | single site |
| `git_max_repos_per_pubkey` | `handlers/side_effects.rs:2487` | single site |
| `push_gateway_timeout` | `push_runtime.rs:314` | single site |
| `started_at` (AppState) | `router.rs:388` | single site (`/_status`) |

---

#### 7. `.env.example` coverage — 5 of 93

`.env.example` (233 lines) declares only 12 variable names, of which just **5 are read by this module**: `BUZZ_BIND_ADDR`, `DATABASE_URL`, `REDIS_URL`, `RELAY_URL`, `RUST_LOG`. The rest are `PG*` helpers for local tooling plus:

**Two dead entries**: `TYPESENSE_API_KEY` (`.env.example:40`) and `TYPESENSE_URL` (`.env.example:41`). No Rust code in the workspace reads either — search moved to Postgres FTS (`main.rs:369-372`, `handlers/event.rs:505-506` confirm "The old Typesense `index_event` worker and its `search_index_tx` mpsc are gone"). The only remaining `Typesense` mentions are historical comments (`crates/buzz-search/src/query.rs:20,46`).

**88 of 93 vars are undocumented in `.env.example`**, including every security-relevant switch: `BUZZ_REQUIRE_AUTH_TOKEN`, `BUZZ_REQUIRE_RELAY_MEMBERSHIP`, `BUZZ_PUBKEY_ALLOWLIST`, `BUZZ_REQUIRE_MEDIA_GET_AUTH`, `BUZZ_CORS_ORIGINS`, `BUZZ_RELAY_PRIVATE_KEY`, `RELAY_OWNER_PUBKEY`, `RELAY_OPERATOR_PUBKEYS`, `RELAY_OPERATOR_API_ORIGIN`, `BUZZ_AUDIT_ENABLED`, `BUZZ_AUTO_MIGRATE`, `BUZZ_ADMIN_HOST`. An operator following `AGENTS.md`'s `cp .env.example .env` gets a relay with every permissive default active and no indication that the switches exist.


## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Configuration

Every value this group reads comes from `AppState.config: Arc<Config>` (`state.rs:490`). Struct definition: `config.rs:51` onward; loader: `Config::from_env` at `config.rs:405`, called exactly once at `main.rs:122`.

> The WS path reads config through `state.config.*` on every frame (e.g. `connection.rs:420`, `:439`) rather than snapshotting it per connection — but there is no reload path, so the values are effectively immutable for the process lifetime.

---

#### 1. Env vars consumed directly by this group

| Env var | Config field | Default | Read at | Effect |
|---|---|---|---|---|
| `BUZZ_MAX_CONNECTIONS` | `max_connections` | **10 000** (`config.rs:449-452`) | `state.rs:727` (`Semaphore::new`), acquired `connection.rs:149` | hard cap on concurrent WS connections; exhaustion rejects with **no frame sent** (`connection.rs:151-154`) |
| `BUZZ_MAX_CONCURRENT_HANDLERS` | `max_concurrent_handlers` | **1024** (`config.rs:454-457`) | `state.rs:728`, acquired `connection.rs:513`/`:541`/`:563` | cap on in-flight EVENT/REQ/COUNT handler tasks; exhaustion → `rate-limited: too many concurrent requests` |
| `BUZZ_SEND_BUFFER` | `send_buffer_size` | **1 000** (`config.rs:459-462`) | `connection.rs:159` | per-connection outbound **message** queue depth (not bytes). Depth is what the backpressure strike counter measures |
| `BUZZ_MAX_FRAME_BYTES` | `max_frame_bytes` | **524 288** (512 KiB) — `DEFAULT_MAX_FRAME_BYTES` `config.rs:14`, parsed `:464-468` with a `>0` filter | parser `router.rs:334-342`; app re-check `connection.rs:420`, `:439` | max inbound frame; oversize → one `NOTICE` then disconnect. Doc'd as needing headroom over the 256 KiB content cap after JSON+NIP-44 overhead (`config.rs:11-13`) |
| `BUZZ_SLOW_CLIENT_GRACE_LIMIT` | `slow_client_grace_limit` (`u8`) | **15** (`config.rs:470-473`) | copied to `ConnectionState.grace_limit` `connection.rs:179` and `ConnEntry.grace_limit` `:212`; compared `connection.rs:100`, `state.rs:464` | consecutive buffer-full events tolerated before cancelling. **No `>0` filter** — `0` disconnects on the first full buffer; a value >255 fails to parse and silently falls back to 15 |
| `RELAY_URL` | `relay_url` | `ws://localhost:3000` (`config.rs:428`) | `auth.rs:80-81` via `nip42_expected_relay_url` | the URL a NIP-42 AUTH event's `relay` tag must match; tenant-adjusted per connection |
| `BUZZ_PUBKEY_ALLOWLIST` | `pubkey_allowlist_enabled` | **false** (`config.rs:479-481`) | `auth.rs:189` | when true, NIP-42 pubkey-only auth additionally requires a `pubkey_allowlist` row; DB error → deny |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | `require_relay_membership` | **false** (`config.rs:483-485`) | `auth.rs:245` | closed vs open relay. On an open relay the NIP-OA owner is still extracted for agent→owner backfill |
| `BUZZ_ALLOW_NIP_OA_AUTH` | `allow_nip_oa_auth` | **false** (`config.rs:520`) | indirectly, inside `enforce_relay_membership` (`auth.rs:217-223`) | permits NIP-OA owner attestation to *grant* membership on a closed relay |
| `BUZZ_RATE_LIMIT_HUMAN_WS_EVENTS_PER_SEC` | `auth.rate_limits.human_ws_events_per_sec` | **10** (`buzz-auth/src/rate_limit.rs:116-118`) | `connection.rs:614` → `admission.rs:44-49` | multiplied by the 5 s burst window → **50 frames per 5 s** for EVENT+REQ+COUNT combined |
| `BUZZ_RATE_LIMIT_HUMAN_MESSAGES_PER_MIN` | `auth.rate_limits.human_messages_per_min` | **60** (`rate_limit.rs:110-112`) | `connection.rs:636` | EVENT-only, 60 s window, human tier |
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_MESSAGES_PER_MIN` | `auth.rate_limits.agent_standard_messages_per_min` | **120** (`rate_limit.rs:119-121`) | `connection.rs:634` | EVENT-only, 60 s window, agent tier (selected by `agent_owner_pubkey.is_some()`) |

All `BUZZ_RATE_LIMIT_*` values go through `positive_u64_from_env` (`config.rs:270-283`), so `0` or a non-integer is a **hard startup error**, not a silent fallback — verified by `rate_limit_overrides_reject_zero`, `config.rs:1109`.

---

#### 2. Env vars consumed indirectly (services this group calls)

| Env var | Default | Reaches this group via |
|---|---|---|
| `REDIS_URL` | `redis://localhost:6379` (`config.rs:419`) | `admission_rate_limiter` (`state.rs:712`), `pubsub` publish/retain/release, presence |
| `BUZZ_REDIS_POOL_SIZE` | **16** (`config.rs:421-425`) — deadpool's own default (`CPU×2`) is called out as too small for a 2-vCPU pod (`config.rs:61-64`) | shared pool behind admission, presence, pub/sub |
| `DATABASE_URL` | `postgres://buzz:buzz_dev@localhost:5432/buzz` (`config.rs:410-411`) | every `state.db.*` call listed in the integrations aspect |
| `READ_DATABASE_URL` | unset → all reads on the writer (`config.rs:413-417`) | handled inside `buzz-db`; not visible to this group |
| `BUZZ_AUDIT_ENABLED` | **true** (`config.rs:793`) | when false, `audit_tx` is `None` and `enqueue_event_created_audit` short-circuits (`event.rs:571-573`) — removing the only awaited step in `dispatch_persistent_event` |
| `BUZZ_EPHEMERAL_TTL_OVERRIDE` | unset (`config.rs:691-695`) | `handlers::resolve_ttl` (`mod.rs:42-62`) — consumed by `ingest.rs:2125` / `side_effects.rs:1681`, not by the WS handlers themselves |
| `BUZZ_RELAY_PRIVATE_KEY` | unset → fresh keypair each start (`config.rs:602`) | `state.relay_keypair`, compared at `event.rs:517` to skip workflow triggering for relay-signed events |
| `BUZZ_CORS_ORIGINS` | empty → permissive (`config.rs:595-600`) | applied to the merged router (`router.rs:191`); does not gate the WS upgrade |
| `BUZZ_ADMIN_HOST` | unset (`config.rs:814-836`) | short-circuits `/` before NIP-11 or the WS upgrade (`router.rs:259-277`) — an admin-host request can never open a relay socket |

---

#### 3. Hard-coded values that are *not* configurable

These are the tuning knobs an operator cannot reach:

| Value | Constant | Site | Consequence of being fixed |
|---|---|---|---|
| **5 s** NIP-42 auth deadline | `AUTH_TIMEOUT` | `connection.rs:27` | a client on a high-latency link that cannot sign+round-trip in 5 s cannot connect |
| **30 s** heartbeat interval | — | `connection.rs:383` | not tunable for mobile radio-sleep profiles |
| **3** missed pongs | — | `connection.rs:389-393` | fixed 90 s dead-peer detection |
| **8** control-channel slots | — | `connection.rs:162` | a stalled writer becomes terminal after 8 queued control frames |
| **64** frames per flush | `MAX_WS_SEND_BATCH` | `connection.rs:33` | fan-out write batching |
| **1024** subscriptions/connection | `MAX_SUBSCRIPTIONS` | `req.rs:26` | matches the NIP-11 advertisement |
| **10** filters/REQ | `MAX_FILTERS_PER_REQ` | `protocol.rs:14` | NIP-11 advertised |
| **256 B** sub_id | `MAX_SUB_ID_LENGTH` | `protocol.rs:11` | NIP-11 advertised |
| **2 000** historical rows/filter | `MAX_HISTORICAL_LIMIT` | `req.rs:25` | also clamps the search per-filter limit (`req.rs:538`) |
| **4** concurrent filter queries | `FILTER_QUERY_CONCURRENCY` | `req.rs:35`; compile-time asserted 2..=8 at `:41` | intentionally locked; raising it requires re-running the relay bench (doc `req.rs:37-40`) |
| **10** search pages, **100** hits/page | `MAX_SEARCH_PAGES`, `per_page` | `req.rs:421`, `:589` | search result ceiling is 1000 hits/filter before post-filtering |
| **5 000** COUNT candidate rows | `COUNT_FALLBACK_CANDIDATE_LIMIT` | `req.rs:753` | a non-pushable COUNT over a large set is refused, not approximated |
| **100/s** observer telemetry per agent | — | `event.rs:912` | shared by every deployment size |
| **±300 s** observer freshness | — | `event.rs:952` | tighter than the ingest ±900 s drift window (`ingest.rs:1506`) |
| **±900 s** ingest timestamp drift | `MAX_TIMESTAMP_DRIFT_SECS` | `ingest.rs:1506` | — |
| **256 KiB** event content | `MAX_EVENT_CONTENT_BYTES` | `ingest.rs:1515` | the reason `max_frame_bytes` defaults to 512 KiB |
| **128 B** presence status truncation | — | `event.rs:780-785` | UTF-8-boundary-safe |
| **60 s / 10 000** local-echo cache | — | `state.rs:734-739` | cross-node dedup window |
| **10 s** membership / accessible-channels / visibility TTL | — | `state.rs:740-760` | upper bound on stale-access exposure when a cross-pod invalidation is dropped |
| **300 s / 1 000** observer owner cache | — | `state.rs:782-787` | safe because `agent_owner_pubkey` is immutable within a community (`state.rs:601-606`) |
| **1 000** audit channel capacity | — | `state.rs:654` | the back-pressure point for `dispatch_persistent_event` |
| **5 s** window / **×5** burst multiplier for `WsEvents` | `WS_BURST_WINDOW_SECS` | `admission.rs:10`, `:44-49` | fixed-window, not a token bucket — the code notes a Redis token bucket would be a better long-term fit (`admission.rs:5-8`) |
| **60 s** window for `Messages` | — | `connection.rs:643` | — |

---

#### 4. Startup-time validation relevant to this group

| Check | Behaviour | Site |
|---|---|---|
| `BUZZ_MAX_FRAME_BYTES` must be > 0 | invalid/zero → silent fallback to 512 KiB | `config.rs:464-468` |
| `BUZZ_RATE_LIMIT_*` must be a positive integer | invalid/zero → **`ConfigError::InvalidValue`, startup fails** | `config.rs:270-283` |
| `BUZZ_SLOW_CLIENT_GRACE_LIMIT` | **no validation** — unparsable → 15; `0` accepted verbatim | `config.rs:470-473` |
| `BUZZ_MAX_CONNECTIONS` / `BUZZ_MAX_CONCURRENT_HANDLERS` / `BUZZ_SEND_BUFFER` | **no `>0` filter** — `0` is accepted and would make `Semaphore::new(0)` / `mpsc::channel(0)`; `mpsc::channel(0)` **panics** at `connection.rs:159` | `config.rs:449-462` |
| `BUZZ_REQUIRE_AUTH_TOKEN=false` warns that REST bypasses token auth and states WS is unaffected | `warn!` only | `config.rs:588-593` |
| Defaults are asserted valid by a test | `config.rs:938-1000` — asserts `max_connections > 0`, `send_buffer_size > 0`, `slow_client_grace_limit > 0`, `max_frame_bytes == DEFAULT_MAX_FRAME_BYTES` | — |

The `defaults_are_valid` test (`config.rs:938`, 17 assertions through `:1000`) asserts the invariants that the **loader does not enforce**. It only covers the default path, so an operator setting `BUZZ_SEND_BUFFER=0` gets a runtime panic on the first connection rather than a config error.

---

#### 5. Configuration deltas against docs

| Claim | Source | Actual | Verdict |
|---|---|---|---|
| "Max frame size: 65,536 bytes" | `ARCHITECTURE.md:161` | `512 * 1024` (`config.rs:14`) | **wrong by 8×** |
| "Max historical results per filter: 500" | `ARCHITECTURE.md:161` | `2_000` (`req.rs:25`) | **wrong by 4×** |
| "After `SLOW_CLIENT_GRACE_LIMIT` (3) consecutive full-buffer events" | `ARCHITECTURE.md:208` | default **15** (`config.rs:473`); the value is a config field, not a constant | **wrong by 5×**, and misnames the mechanism |
| "`BUZZ_MAX_FRAME_BYTES` … default 65536" | `.aidlc/reverse-engineer/configuration.md:89` | 524 288 | **wrong** (inherits the ARCHITECTURE error) |
| "`BUZZ_SLOW_CLIENT_GRACE_LIMIT` … default 3" | `.aidlc/reverse-engineer/configuration.md:91` | 15 | **wrong** |
| "`BUZZ_SEND_BUFFER` … default (code default)" | `.aidlc/reverse-engineer/configuration.md:90` | 1 000 (`config.rs:462`) | resolvable — should be stated |
| "No rate limiting implementation … none are enforced" | `ARCHITECTURE.md:823` | `human_ws_events_per_sec`, `human_messages_per_min`, `agent_standard_messages_per_min` all enforced (`connection.rs:612-650`) | **wrong**; the accurate residual gap is `IpConnections` + the elevated/platform tiers |
| §3 Step 3 describes auth with no deadline | `ARCHITECTURE.md:181-187` | `AUTH_TIMEOUT` 5 s (`connection.rs:27`) | **omission** |
| §3 Step 5 cleanup lists 5 steps | `ARCHITECTURE.md:211-217` | 7 — adds per-subscription `release_topic` (`connection.rs:265-270`) and last-connection presence clear (`:274-285`) | **incomplete** |
| §4 pipeline step 10 "SEARCH INDEX — search_index_tx.send (bounded worker queue)" | `ARCHITECTURE.md:235` | removed; Postgres FTS makes the persisted row the searchable row, and both the worker and the mpsc are gone | **stale** — the code itself documents the removal at `event.rs:502-508` |
| `BUZZ_MAX_CONNECTIONS` / `BUZZ_MAX_CONCURRENT_HANDLERS` / rate-limit env vars | not in `.env.example` per `grep` of the documented table | present in the loader (`config.rs:449`, `:454`, `:288-320`) | **undocumented knobs** |

---

#### 6. Operational tuning notes derived from the code

- **Buffer sizing is coupled to the strike limit.** A 1000-message buffer with a 15-strike grace means a client must fail 15 *consecutive* sends; since any success resets to 0 (`connection.rs:92`), a client that drains slowly-but-steadily is never disconnected. Lowering `BUZZ_SLOW_CLIENT_GRACE_LIMIT` is the only lever, and there is no timeout-based eviction to complement it.
- **`max_frame_bytes` must exceed 256 KiB** or large legitimate events become unsendable: the ingest content cap is 256 KiB (`ingest.rs:1489`) and Nostr JSON + NIP-44 base64 roughly doubles it. The 512 KiB default is the documented reason (`config.rs:11-13`); a 65 536 setting (as `ARCHITECTURE.md` claims is the default) would silently break large messages.
- **The WS burst budget is not the advertised rate.** `human_ws_events_per_sec=10` yields a 5 s fixed window of 50, so a client can legitimately emit 50 frames in the first 10 ms of a window and then be starved for 4.99 s. Desktop startup is the stated reason (`admission.rs:5-7`).
- **Horizontal scaling requires `BUZZ_HUDDLE_AUDIO_AVAILABLE=false`** (`config.rs:114-129`, loader `config.rs:489-492`) — not a WS-protocol knob, but it shares the socket lifecycle registry this group registers into (`connection.rs:132`).
- **`BUZZ_MESH` defaults off** (loader `config.rs:498-500`) and is not consulted anywhere in this group; mesh-on and mesh-off WS behaviour is identical.


## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: Configuration

---

### 1. Config fields consumed (via `state.config`)

Exactly **four** fields of `crate::config::Config` are read across all 8 911 lines.

| Field | Env var | Default | Read at | Effect in this module |
|---|---|---|---|---|
| `require_relay_membership: bool` | `BUZZ_REQUIRE_RELAY_MEMBERSHIP` (`"true"` or `"1"` → true) | **`false`** (`config.rs:482-484`; asserted `config.rs:954-956`) | `ingest.rs:1847` | Gates kind 28936 (NIP-43 leave request). When `false` — the default — every 28936 is rejected with `invalid: relay membership is not enabled`. So **self-service relay leave is off out of the box.** |
| `ephemeral_ttl_override: Option<i32>` | `BUZZ_EPHEMERAL_TTL_OVERRIDE` (parsed `i32`, filtered `> 0`) | `None` (`config.rs:691-695`) | `ingest.rs:2125`, `side_effects.rs:1681` | Passed to `resolve_ttl` (`handlers/mod.rs:42-62`). When set, it **clobbers** any client-supplied `ttl` tag on kind 9007 rather than clamping it. Config emits a startup `warn!` when set (`config.rs:696-700`). Note: the 9002 `ttl` update path (`side_effects.rs:1449-1481`) does **not** apply the override — a client can change a channel's TTL post-creation to escape it. |
| `relay_url: String` | `RELAY_URL` (no `BUZZ_` prefix) | `"ws://localhost:3000"` (`config.rs:427-428`) | `ingest.rs:2238` | Input to `media_base_url_for_tenant(&state.config.relay_url, tenant.host())`, which yields the per-tenant `media_base_url` that `validate_imeta_tags` accepts absolute `imeta` URLs against (`imeta.rs:375-386`). A misconfigured `RELAY_URL` therefore rejects every absolute-form `imeta` URL while relative `/media/...` still works. |
| `git_max_repos_per_pubkey: u32` | `BUZZ_GIT_MAX_REPOS_PER_PUBKEY` | **`100`** (`config.rs:731-734`) | `side_effects.rs:2487` | Per-pubkey repo quota checked **before** claiming a name on kind 30617. Exceeded → `repo limit exceeded: {owned} >= {limit}`. Checked only on the fresh-claim path; a same-owner re-announce never grows the count (`side_effects.rs:2481-2490`). |

---

### 2. Env vars read directly (bypassing `Config`)

| Env var | Default | Read at | Effect |
|---|---|---|---|
| `SPROUT_MAX_NOT_BEFORE_DELTA` | **`31_536_000`** (1 year) | `ingest.rs:1325-1328` | Upper bound on how far in the future a kind:30300 `not_before` may be. Exceeded → `not_before too far in future`. |

⚠ Three problems with this one:
1. It is the **only** env var in the group read via `std::env::var` at request time instead
   of through `Config`. It is re-parsed on **every** kind:30300 ingest — no caching.
2. It uses the legacy `SPROUT_` prefix while every other var in this group uses `BUZZ_`
   (or bare `RELAY_URL`).
3. It is read in two places that must agree: here and `nip11.rs:97`, which advertises the
   value in the NIP-11 document. Both default to `31_536_000` independently; there is no
   shared constant. A change to one silently desynchronises the advertised limit from the
   enforced limit. The code comment at `ingest.rs:1298` asserts the coupling ("the same
   SPROUT_MAX_NOT_BEFORE_DELTA env var is advertised in NIP-11") without enforcing it.

---

### 3. Compile-time constants (effectively configuration, not tunable)

| Constant | Value | Declared | Governs |
|---|---|---|---|
| `MAX_TIMESTAMP_DRIFT_SECS` | `900` (±15 min) | `ingest.rs:1506` | Global created_at fence, all kinds |
| `MAX_EVENT_CONTENT_BYTES` | `256 * 1024` | `ingest.rs:1515` | Global content size cap |
| `MAX_REACTION_EMOJI_CHARS` | `64` | `ingest.rs:2309` | kind:7 emoji length (Unicode chars, not bytes) |
| `MAX_NOT_BEFORE` | `9_007_199_254_740_991` (`Number.MAX_SAFE_INTEGER`) | `ingest.rs:1250` | kind:30300 `not_before` ceiling |
| diff content cap | `61_440` (60 KiB) | `ingest.rs:898` | kind:40008 |
| thread depth cap | `100` | `ingest.rs:647` | NIP-10 nesting |
| 28936 freshness window | `120` s | `ingest.rs:1834` | NIP-43 leave request |
| DM participant cap | `8` others / `9` total | `command_executor.rs:322`, `:544` | kinds 41010, 41011 |
| NIP-44 v2 minimum decoded length | `99` bytes | `ingest.rs:1130` | kinds 30174, 44200 |
| `D_TAG_MAX_LEN` | `1024` | `buzz-db/src/event.rs:140`, used `ingest.rs:2404`, `command_executor.rs:141` | all NIP-33 kinds |
| persona slug max | `64` chars | `ingest.rs:1055` | kind 30175 |
| repo id max | `64` chars | `side_effects.rs:2393` | kind 30617 |
| imeta `filename` max | `255` chars | `imeta.rs:143` | all imeta |
| imeta MIME max | `255` chars | `imeta.rs:345` | all imeta |
| imeta duration tolerance | `0.1` s | `imeta.rs:272` | video imeta |
| `DEFAULT_HEAD` | `"refs/heads/main"` | `side_effects.rs:2606` | seeded git manifest + initial kind:30618 |
| audit channel capacity | `1000` | `state.rs:654`, referenced `handlers/event.rs:540` | audit backpressure on the ingest path |

None of these 17 are configurable. The 256 KiB content cap and the ±15 min drift window are
the two most likely to need per-deployment tuning and are hard-coded.

---

### 4. Metrics emitted (observability configuration surface)

| Metric | Type | Labels | Emitted at |
|---|---|---|---|
| `buzz_events_stored_total` | counter | `kind` (bounded), `author_type` (`agent`/`human`) | `ingest.rs:1422-1427` — at the shared seam so WS and HTTP count identically |
| `buzz_events_rejected_total` | counter | `transport` (`ws`/`http`), `reason` (`auth`/`invalid`/`scope`/`error`) | `reject_with_transport` `ingest.rs:157-163`; called from the transports, not from ingest |
| `buzz_channels_created_total` | counter | `community` (host), `type` | `ingest.rs:2150-2155`; `side_effects.rs:1704`, `:1731`; `command_executor.rs:373`, `:527` |
| `buzz_users_created_total` | counter | `community` (host) | `side_effects.rs:1097-1101`, `:1141-1145`; `command_executor.rs:51-56` |
| `buzz_nip43_membership_publications_total` | counter | `result` (`attempted`/`succeeded`/`failed`) | `side_effects.rs:2828`, `:2834-2838` |
| `buzz_nip43_membership_publication_seconds` | histogram | — | `side_effects.rs:2832` |
| `buzz_nip43_membership_reconciliations_total` | counter | — | `side_effects.rs:2818` |
| `buzz_nip43_membership_reconciliation_failures_total` | counter | — | `side_effects.rs:2811` |

Reached transitively via `dispatch_persistent_event`:
`buzz_post_commit_dispatch_scheduled_total` (`handlers/event.rs:371`),
`buzz_audit_send_errors_total` (`handlers/event.rs:599`),
`buzz_workflow_runs_total` with `trigger` + `community` (`handlers/event.rs:547-552`).

⚠ Gaps: there is **no** counter for side-effect failures, despite 32 `warn!` sites in
`side_effects.rs` that each represent a silently-degraded write (BR-IN-69). There is also no
histogram for ingest latency in this module, and `buzz_channels_created_total` is emitted
from five sites whose non-double-counting relies on a prose argument
(`side_effects.rs:1670-1679`) rather than a single emit point.

`author_type_label` (`ingest.rs:1328-1361`) performs a DB lookup
(`get_agent_channel_policy`) on cache miss purely to compute a metric label, cached in
`state.author_type_cache` (`state.rs:613`, `:788`). It is explicitly fail-open — unknown
pubkeys and lookup errors count as `"human"` so "the label must not add a failure path to
ingest" (`ingest.rs:1322-1324`). That cache is never invalidated, so an agent registered
after its first event is mislabelled `human` for the cache's lifetime.

---

### 5. Runtime-state dependencies (not env config, but deployment-shaping)

| `AppState` field | Used at | Note |
|---|---|---|
| `db` | everywhere | Postgres, required |
| `pubsub` | `side_effects.rs:81`, `:701`, `:794`, `:873` | Redis; publish failures are `warn!`-only |
| `media_storage` | `ingest.rs:2241` | S3/Blossom; required for any event with `imeta` |
| `git_store` | `side_effects.rs:2642-2726` | object store; required for kind 30617 |
| `workflow_engine` | `command_executor.rs:783`, `:900`, `:1076`; `side_effects.rs:2017`, `:2038` | in-process |
| `relay_keypair` | 20+ sites in `side_effects.rs` | signs every relay-minted event and is the sentinel in `effective_message_author` (`ingest.rs:731`) |
| `conn_manager`, `sub_registry` | `side_effects.rs:44-140` | subscription eviction |
| `tracer` | `ingest.rs:1409` | `NoopTracer` in production (`state.rs:798`) |
| `author_type_cache` | `ingest.rs:1363-1379` | metric labelling only |
| `audit_tx` | via `handlers/event.rs:539` | `None` disables auditing entirely (`main.rs:330` decides) |

---

### 6. `.env.example` coverage check

| Var this module reads | In `.env.example`? | In `Config`? |
|---|---|---|
| `RELAY_URL` | ✅ `.env.example:49` | ✅ `config.rs:68`, `:427-428` |
| `BUZZ_EPHEMERAL_TTL_OVERRIDE` | ✅ commented out, `.env.example:100` | ✅ `config.rs:209`, `:691-695` |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | ❌ **absent** | ✅ `config.rs:112`, `:482-484` |
| `BUZZ_GIT_MAX_REPOS_PER_PUBKEY` | ❌ **absent** | ✅ `config.rs:234`, `:731-734` |
| `SPROUT_MAX_NOT_BEFORE_DELTA` | ❌ **absent** | ❌ **absent** — only two independent `std::env::var` reads |

Two actionable rows:
- `BUZZ_REQUIRE_RELAY_MEMBERSHIP` gates a whole feature (self-service relay leave, kind
  28936) and defaults off, yet is not discoverable from `.env.example`.
- `SPROUT_MAX_NOT_BEFORE_DELTA` changes protocol-visible behaviour (the NIP-11-advertised
  reminder horizon) and exists only as two independent `std::env::var` reads
  (`ingest.rs:1299`, `nip11.rs:97`). It never appears in the config struct, so it is absent
  from any config dump, validation, or startup log — and the enforced value can silently
  diverge from the advertised one.


## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Configuration

All values are read from `crate::config::Config` (constructed once in
`Config::from_env`, `crates/buzz-relay/src/config.rs`) and reached through
`state.config`. No file in this module group calls `std::env::var` outside
`#[cfg(test)]`.

---

#### 1. Env vars consumed by this module group

##### Authentication / admission

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `BUZZ_REQUIRE_AUTH_TOKEN` | `require_auth_token` | **`false`** | `bridge.rs:640`, `:908`, `:1341`, `:2033` | ⚠️ **Permissive.** `false` enables the unsigned `X-Pubkey` impersonation path (`bridge.rs:118-127`). Parsed at `config.rs:475-477`; startup warning at `config.rs:588-593`. **Not present in `.env.example`.** |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | `require_relay_membership` | **`false`** | `api/mod.rs:67`, `bridge.rs:808`, `operator.rs:435` | ⚠️ Open relay by default — any valid key may publish, read, and upload. `config.rs:483-485`; default asserted `config.rs:953-956`. |
| `BUZZ_ALLOW_NIP_OA_AUTH` | `allow_nip_oa_auth` | **`false`** | `api/mod.rs:81` | Gates NIP-OA owner delegation *for admission*. The owner-**backfill** path is intentionally unflagged (`api/mod.rs:151-156`). `config.rs:520-522`. |
| `BUZZ_RATE_LIMIT_HUMAN_API_CALLS_PER_MIN` | `auth.rate_limits.human_api_calls_per_min` | **`300`** | `bridge.rs:29` | Window is a fixed 60 s (`bridge.rs:35`). Default `buzz-auth/src/rate_limit.rs:113-115`; env parse `config.rs:291-294`; documented `.env.example:61`. |

##### Relay identity (used for scheme derivation and HMAC keys)

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `BUZZ_RELAY_URL` (see `config.rs`) | `relay_url` | — | `bridge.rs:635`, `:903`, `:1336`, `:2031`; `media.rs:404`, `:449`; `invites.rs:212`, `:267`; `nip05.rs:55`, `:107`; `operator.rs:219` | **Only its scheme prefix is used** for URL construction (`wss://`/`https://` ⇒ TLS, else plaintext). Its *host* is deliberately never used as an identity — see `bridge.rs:184-193`. `operator.rs:219` is the one exception: `relay_url_authority` extracts the deployment host to protect it from archival. |
| `BUZZ_RELAY_PRIVATE_KEY` | → `state.relay_keypair` | generated if absent (`config.rs:594`) | `bridge.rs:526`, `:1972`; `invites.rs:185`, `:259`, `:300` | Signs 39005/39006/20001 overlays **and** derives the invite/policy HMAC key (`invite_token.rs:112-117`). Rotating it invalidates every outstanding invite — the documented blast-radius control (`invite_token.rs:24-30`). |

##### Media

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `BUZZ_REQUIRE_MEDIA_GET_AUTH` | `require_media_get_auth` | **`false`** | `media.rs:491`, `:517`, `:626`, `:805` | ⚠️ **Permissive.** Off ⇒ `GET`/`HEAD /media/*` unauthenticated and `Cache-Control: public`. Accepts `true`/`1`/`yes`/`on` case-insensitively (`config.rs:682-689`). Default explicitly justified "for staged client rollout" (`config.rs:973-976`). Documented `.env.example:90`. |
| `BUZZ_MEDIA_UPLOADS_PER_MINUTE` | `media_uploads_per_minute` | `30` | `media.rs:96` | Per `(community, pubkey)`, per pod. Must be > 0 (`config.rs:676-680`). `.env.example:87`. |
| `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS` | `media_max_concurrent_uploads` | `8` | `state.rs:730` (`Semaphore`), read at `media.rs:117` | Global per pod. `config.rs:663-668`. `.env.example:85`. |
| `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS_PER_PUBKEY` | `media_max_concurrent_uploads_per_pubkey` | `2`, clamped to the global | `media.rs:126` | `config.rs:669-675`. `.env.example:86`. |
| `BUZZ_MAX_IMAGE_BYTES` | `media.max_image_bytes` | `50 MB` | `media.rs:363`; `router.rs:34` | Contributes to the buffered non-video ceiling **and** to the route body limit. |
| `BUZZ_MAX_VIDEO_BYTES` | `media.max_video_bytes` | `500 MB` | `router.rs:35` | Sets the media-router `RequestBodyLimitLayer` to `max(image, video)` = **500 MB** by default. |
| `BUZZ_MAX_FILE_BYTES` | `media.max_file_bytes` | `100 MB` | `media.rs:364` | Buffered non-video ceiling is `max(image, file)` = **100 MB** in RAM per upload. |
| `BUZZ_MEDIA_UPLOAD_RECORDS` | `media.upload_records_enabled` | **`false`** | `media.rs:247` | Off ⇒ `upload_attribution` returns `None` and no `_uploads/` record is written. `config.rs:651-653`. |
| `BUZZ_MEDIA_UPLOAD_IP_HEADER` | `media.upload_ip_header` | `None` | `media.rs:279` | Trusted-edge header name. **Fail-empty**: missing/malformed/non-public ⇒ nothing recorded; the socket address is never used (`media.rs:243-251`). `config.rs:654-657`. |
| `BUZZ_MEDIA_UPLOAD_PORT_HEADER` | `media.upload_port_header` | `None` | `media.rs:280` | Only kept alongside a valid IP. `config.rs:658-661`. |

##### Join policy (deployment-global, not per-community)

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `BUZZ_TERMS_OF_SERVICE_MARKDOWN` | `join_policy.terms_markdown` | `None` | `invites.rs:82`, `:98` | Size-capped by `MAX_POLICY_MARKDOWN_BYTES` (`config.rs:780-787`). |
| `BUZZ_PRIVACY_POLICY_MARKDOWN` | `join_policy.privacy_markdown` | `None` | `invites.rs:83`, `:107` | same |
| `BUZZ_AGE_ATTESTATION_REQUIRED` | `join_policy.age_attestation_required` | `false` | `invites.rs:84`, `:177` | `config.rs:792` |
| *(derived)* | `join_policy.version` | `sha256(terms ‖ 0x00 ‖ privacy ‖ 0x00‖age)` | `invites.rs:85`, `:178`, `:186`, `:319`, `:333` | Any edit invalidates every outstanding receipt (`config.rs:799-810`). |
| *(derived)* | `join_policy: Option<…>` | `None` when all three inputs are unset | `invites.rs:76`, `:169`, `:314` | `None` ⇒ the claim gate is skipped entirely and all three policy endpoints 404. Default asserted `config.rs:977-980`. **None of these three vars appear in `.env.example`.** |

##### Operator control plane

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `RELAY_OPERATOR_API_ORIGIN` | `relay_operator_api_origin` | `None` | `operator.rs:68` | Deliberately **un-prefixed** (shared relay-identity family, `config.rs:544-548`). Unset ⇒ every operator request **500s** (`operator.rs:69-72`). Parsed by `parse_operator_api_origin` (`config.rs:549-553`). |
| `RELAY_OPERATOR_PUBKEYS` | `relay_operator_pubkeys` | empty | `operator.rs:89-95` | Comma-separated 64-hex, lowercased, deduped. An invalid entry is a **hard config error** (unlike `RELAY_OWNER_PUBKEY`, which warns and ignores) because silently dropping an operator would silently disable provisioning (`config.rs:555-576`). Non-empty without the origin is also a hard error (`config.rs:577-581`). Default asserted `config.rs:961-964`. **Neither var appears in `.env.example`.** |

##### Deployment-admin surface

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `BUZZ_ADMIN_HOST` | `admin.host` | `None` | `admin/auth.rs:8-14`, `:18-21`; `router.rs:56-59`, `:264` | ⚠️ **This single value is the entire authorization boundary** for 5 cross-tenant read endpoints (see security SEC-02). Must be a bare authority — `/`, `\`, `@` are rejected (`config.rs:819-825`). Unset ⇒ the sub-router (and the only security-header middleware in the app) is never mounted. **Not in `.env.example`.** |
| `BUZZ_ADMIN_WEB_DIR` | `admin.web_dir` | `None` | `router.rs:52-55`, `:159-176`, `:266-275` | Validated at startup: must contain `index.html` (`config.rs:826-837`). |

##### Mesh demo

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `BUZZ_MESH` | `mesh.enabled` → `state.mesh()` | **`false`** | `mesh_demo.rs:64` | Strict parse: `on`/`true`/`1` only (`config.rs:509-512`). |
| `BUZZ_MESH_DEMO_ECHO` | `mesh_demo_echo` | **`false`** | `mesh_demo.rs:67` | Same strict parse (`config.rs:514-518`). Both must be on or the route 404s. **Not in `.env.example`.** |

##### Infrastructure reached indirectly

| Env var | Used by this module via | `file:line` |
|---|---|---|
| `REDIS_URL` / `redis_url` | `state.nip98_replay` (replay seen-set), `state.admission_rate_limiter`, `state.pubsub` (presence), mesh `SessionDirectory` | `state.rs:710-712`; `bridge.rs:141`, `:31`, `:1951` |
| `DATABASE_URL` / `database_url` | `state.db` — every handler | throughout |
| `BUZZ_S3_*` | `state.media_storage` | `media.rs:637-679` |
| `BUZZ_CORS_ORIGINS` | `build_cors_layer` — applies to **every** route including this module's | `router.rs:188`, `:373-397` |
| `BUZZ_WEB_DIR`, `BUZZ_SERVE_GIT_WEB_GUI` | SPA fallback that serves `/invite/{code}` | `router.rs:155-186`, `:191-198` |

---

#### 2. Permissive security-relevant defaults (flagged)

| # | Var | Default | Consequence | Cited at |
|---|---|---|---|---|
| 1 | `BUZZ_REQUIRE_AUTH_TOKEN` | `false` | Unsigned `X-Pubkey` header impersonates any pubkey on `/events`, `/query`, `/count`, and all three `/moderation/*` reads; NIP-98 replay protection is bypassed on that path. **Highest-impact default in the module.** | `config.rs:475-477`; `bridge.rs:118-127`, `:150-153` |
| 2 | `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | `false` | Open relay: any valid key may publish, read, and upload media. Also means the unbounded media limiter map (`state.rs:774`) is keyed by an attacker-mintable identity. | `config.rs:483-485`; `api/mod.rs:67-69`; `media.rs:196-206` |
| 3 | `BUZZ_REQUIRE_MEDIA_GET_AUTH` | `false` | `GET`/`HEAD /media/*` unauthenticated; blob responses cached as `public`. `verify_blossom_get_auth` is dead code in this configuration. | `config.rs:682-689`; `media.rs:491-494`, `:517-523` |
| 4 | `BUZZ_CORS_ORIGINS` | empty ⇒ `CorsLayer::permissive()` | Any origin may read cross-origin responses from every route in this module, including `/query` results and `/moderation/*`. Mitigated for the *authenticated* routes by the NIP-98 requirement (no cookies/ambient credentials), but a real concern for `/media/*` when read auth is off. Note the failure path is correct: an unparseable explicit list falls back to **deny**, not permissive (`router.rs:383-392`). | `router.rs:373-397`; `config.rs:595-600` |
| 5 | `BUZZ_ADMIN_HOST` | `None` (safe) — but when **set**, it is the whole boundary | Setting it exposes 5 cross-tenant read endpoints with no credential; a spoofable `Host` header is sufficient. Safe-by-default, unsafe-once-enabled. | `admin/auth.rs:16-33`; `docs/admin/README.md:64-70` |
| 6 | `BUZZ_MAX_VIDEO_BYTES` = 500 MB | route body limit becomes `max(image, video)` | The media router's `RequestBodyLimitLayer` is **500 MB** — 10× the "50 MB limit" claimed in `ARCHITECTURE.md:623`. Bounded in practice by the pre-body auth extractor (`media.rs:29-38`) and the concurrency semaphore. | `router.rs:33-36` |
| 7 | `BUZZ_MAX_FILE_BYTES` = 100 MB | non-video buffered ceiling | Worst case per pod ≈ `max(image, file) × media_max_concurrent_uploads` = 100 MB × 8 = **800 MB** resident. | `media.rs:362-367`; `state.rs:730` |
| 8 | `join_policy` | `None` | The consent gate does not exist unless explicitly configured; `claim` accepts without any receipt. | `config.rs:794-799`; `invites.rs:314-316` |

**Non-permissive defaults worth noting as correct:** `mesh`/`mesh_demo_echo` off with a strict
`on`/`true`/`1` parse so typos fail closed (`config.rs:509-518`); `upload_records_enabled` off with
fail-empty IP capture (`media.rs:243-251`); `allow_nip_oa_auth` off; `relay_operator_pubkeys` empty
so provisioning is disabled until explicitly configured, with a startup hard-error if the pubkeys
are set without the origin (`config.rs:577-581`).

---

#### 3. Hard-coded constants (not configurable)

| Constant | Value | Purpose | `file:line` |
|---|---|---|---|
| `BRIDGE_FEED_MAX_LIMIT` | 100 | feed page cap | `bridge.rs:270` |
| `BRIDGE_THREAD_MAX_LIMIT` | 500 | thread page cap | `bridge.rs:271` |
| `BRIDGE_WINDOW_DEFAULT_LIMIT` / `_MAX_LIMIT` | 50 / 200 | channel-window rows | `bridge.rs:374-375` |
| aux-closure per-hop limit | 1000 | reaction/deletion closure | `bridge.rs:492` |
| FTS `per_page` ceiling | 500 | `limit.min(500)` | `bridge.rs:1665` |
| `REJECT_REASON_MAX_BYTES` | 256 | log truncation of attacker-controlled text | `bridge.rs:595` |
| `MODERATION_READ_LIMIT` | 500 | moderation row cap | `bridge.rs:2077` |
| `MAX_RANGE_CHUNK` | 16 MiB | 206 response cap | `media.rs:587` |
| `MEDIA_UPLOAD_RATE_WINDOW` | 60 s | upload limiter window | `media.rs:66` |
| upload-extractor Blossom window | 3600 s | pre-type-detection permissive window | `media.rs:177` |
| media-read Blossom window | 3600 s | `verify_blossom_get_auth` max age | `media.rs:502` |
| `SNIFF_BYTES` | 4096 | content-sniff prefix | `media.rs:321` |
| `is_safe_ext` bounds | 1–8 chars, `[a-z0-9]` | path-segment gate | `media.rs:533-535` |
| `CLAIM_RATE_WINDOW` / `CLAIM_RATE_LIMIT` / `CLAIM_RATE_CACHE_CAPACITY` | 60 s / 10 / 10 000 | invite-claim limiter | `invites.rs:34`, `:37`, `:41` |
| `DEFAULT_INVITE_TTL_SECS` / `MAX_INVITE_TTL_SECS` / mint TTL floor | 72 h / 30 d / 60 s | invite lifetime | `invite_token.rs:52`, `:55`, `:129` |
| `MAX_CODE_LEN` | 1024 | pre-parse input bound | `invite_token.rs:57` |
| policy-receipt max length / TTL | 2048 bytes / 10 min | receipt bounds | `invite_token.rs:369`, `:350` |
| `KEY_DERIVATION_LABEL` | `b"buzz-invite-v1"` | HMAC domain separation | `invite_token.rs:58` |
| `OPERATOR_REPLAY_SCOPE` | `"operator-management"` | replay namespace | `operator.rs:55` |
| admin `limit` range | 1–200, default 50 | report page cap | `admin/mod.rs:75-83` |
| admin feedback limit | **100, hard-coded** | no client control | `admin/mod.rs:155` |
| admin `summarize_body` cap | 240 chars | summary truncation | `admin/mod.rs:296` |
| admin body limit | 1024 bytes | `RequestBodyLimitLayer` | `admin/mod.rs:39` |
| `ECHO_TIMEOUT` | 10 s | mesh demo wait | `mesh_demo.rs:44` |
| `FILTER_QUERY_CONCURRENCY` | 4 (compile-time asserted 2..=8) | bounded-concurrent DB reads | `handlers/req.rs:35`, `:41` |
| bridge/api body limit | 1 MiB | `RequestBodyLimitLayer` | `router.rs:130` |

Candidates for promotion to config: `MODERATION_READ_LIMIT`, `MAX_RANGE_CHUNK`, the invite TTLs, and
the admin feedback limit — all are operational tuning knobs currently requiring a rebuild.

---

#### 4. `.env.example` coverage gap

Documented (`.env.example`): `BUZZ_RATE_LIMIT_HUMAN_API_CALLS_PER_MIN` (:61),
`BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS` (:85), `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS_PER_PUBKEY` (:86),
`BUZZ_MEDIA_UPLOADS_PER_MINUTE` (:87), `BUZZ_REQUIRE_MEDIA_GET_AUTH` (:90).

**Undocumented but consumed by this module — 11 vars, including the four most
security-significant ones:**

| Var | Why it matters |
|---|---|
| `BUZZ_REQUIRE_AUTH_TOKEN` | the unsigned-`X-Pubkey` switch (SEC-01) |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | open-vs-closed relay |
| `BUZZ_ADMIN_HOST` | the sole authorization boundary for 5 cross-tenant read routes (SEC-02) |
| `BUZZ_ADMIN_WEB_DIR` | admin SPA |
| `RELAY_OPERATOR_API_ORIGIN` | operator NIP-98 audience |
| `RELAY_OPERATOR_PUBKEYS` | operator allowlist |
| `BUZZ_MESH_DEMO_ECHO` | unauthenticated, body-tenanted demo route (SEC-03) |
| `BUZZ_ALLOW_NIP_OA_AUTH` | agent→owner delegation |
| `BUZZ_TERMS_OF_SERVICE_MARKDOWN` | join-policy gate |
| `BUZZ_PRIVACY_POLICY_MARKDOWN` | join-policy gate |
| `BUZZ_AGE_ATTESTATION_REQUIRED` | join-policy gate |

An operator following `.env.example` alone ships a relay where `/events`, `/query`, `/count`, and the
moderation reads accept unsigned `X-Pubkey` impersonation, with no hint in the template that the
knob exists.

---

#### 5. Startup validation performed for this surface

| Check | Behaviour | `file:line` |
|---|---|---|
| `RELAY_OPERATOR_PUBKEYS` entry not 64-hex | **hard error**, refuses to start | `config.rs:565-570` |
| `RELAY_OPERATOR_PUBKEYS` set without `RELAY_OPERATOR_API_ORIGIN` | **hard error** | `config.rs:577-581` |
| `BUZZ_ADMIN_HOST` contains `/`, `\`, or `@` | **hard error** | `config.rs:819-825` |
| `BUZZ_ADMIN_WEB_DIR` without `index.html` | **hard error** | `config.rs:830-836` |
| policy Markdown over `MAX_POLICY_MARKDOWN_BYTES` | **hard error** | `config.rs:780-787` |
| `BUZZ_REQUIRE_AUTH_TOKEN=false` | warning only | `config.rs:588-593` |
| `BUZZ_CORS_ORIGINS` set but nothing parseable | error log + deny-all CORS (not permissive) | `router.rs:383-392` |
| media knobs ≤ 0 | silently fall back to the default via `.filter(\|&v\| v > 0)` | `config.rs:663-680` |
| `BUZZ_MESH*` typos (e.g. `BUZZ_MESH=yes`) | silently **off** | `config.rs:509-518` |
| `BUZZ_REQUIRE_MEDIA_GET_AUTH` accepts `yes`/`on` | more lenient than the other booleans, which take only `true`/`1` | `config.rs:682-689` vs `:475-477` |

The boolean-parsing inconsistency is worth noting: `require_media_get_auth` accepts four spellings,
`require_auth_token` / `require_relay_membership` / `allow_nip_oa_auth` accept two, and the mesh vars
accept three (case-insensitive `on` plus `true`/`1`). An operator writing `BUZZ_REQUIRE_AUTH_TOKEN=yes`
silently gets `false`.


## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: Configuration

#### 1. `BUZZ_GIT_*` environment variables — complete inventory

Ten `BUZZ_GIT_*` names are read anywhere in the shipped binary. Three of them (`_S3_PROBE`, plus the `_S3_*` family) are **test-only**.

| Var | Type | Default | Read at | Stored as | Consumed at |
|---|---|---|---|---|---|
| `BUZZ_GIT_REPO_PATH` | `PathBuf`, `create_dir_all`'d | `./repos` | `config.rs:704-706` | `Config::git_repo_path` (`config.rs:218`) | scratch root for every tempdir/tempfile: `transport.rs:604`, `:622`, `:626`, `:797`, `:884`, `:940`; via `repo_path.parent()` in `cas_publish.rs:1028-1030`; `hydrate.rs:294`, `:229` |
| `BUZZ_GIT_PACK_CACHE_PATH` | `PathBuf`, `create_dir_all`'d | `<git_repo_path>/.pack-cache` | `config.rs:707-712` | `Config::git_pack_cache_path` (`:220`) | `state.rs:704` → `GitPackCache::new` (`pack_cache.rs:107-160`) |
| `BUZZ_GIT_MAX_PACK_BYTES` | `u64` | `500 * 1024 * 1024` = 524 288 000 | `config.rs:713-716` | `Config::git_max_pack_bytes` (`:222`) | router body limit `transport.rs:1757`; receive-pack decoded cap `:861`; `HydrationOptions.max_pack_bytes` `:607`, `:801`, `:887`; `PublishLimits.max_pack_bytes` `:1590`; per-pack + per-idx caps `hydrate.rs:305-317`, `:431-446`; `--max-pack-size` `cas_publish.rs:703` |
| `BUZZ_GIT_MAX_REPO_BYTES` | `u64` | **derived**: `git_max_pack_bytes * 2` (saturating) = 1 048 576 000 | `config.rs:717-720` | `Config::git_max_repo_bytes` (`:227`) | `HydrationOptions.max_repo_bytes` `transport.rs:608`, `:802`, `:888`; `hydrate.rs:318-329`; `PublishLimits.max_repo_bytes` `:1591` → `cas_publish.rs:1136-1146`, `:774-782` |
| `BUZZ_GIT_PACK_CACHE_MAX_BYTES` | `u64` | **derived**: `git_max_repo_bytes * 5` (saturating) = 5 242 880 000 | `config.rs:721-724` | `Config::git_pack_cache_max_bytes` (`:230`) | `state.rs:705` → `GitPackCache.max_bytes` (`pack_cache.rs:71`), used by `prune` (`:365-390`) and the bypass check (`:299-313`) |
| `BUZZ_GIT_PACK_CACHE_MAX_CONCURRENT_POPULATIONS` | `usize`, filtered `> 0` | `2` | `config.rs:725-730` | `Config::git_pack_cache_max_concurrent_populations` (`:232`) | `state.rs:706` → `population_semaphore` (`pack_cache.rs:110-113`, acquired `:253-269`) |
| `BUZZ_GIT_MAX_REPOS_PER_PUBKEY` | `u32` | `100` | `config.rs:731-734` | `Config::git_max_repos_per_pubkey` (`:234`) | `handlers/side_effects.rs:2486-2492` (announce-time quota) — **not** consumed inside this module |
| `BUZZ_GIT_MAX_CONCURRENT_OPS` | `usize` | `20` | `config.rs:735-738` | `Config::git_max_concurrent_ops` (`:236`) | `state.rs:693`, `:729` → `git_semaphore` (`state.rs:517-521`), acquired `transport.rs:322` |
| `BUZZ_GIT_HOOK_HMAC_SECRET` | `String` | **random 32 bytes hex per process** | `config.rs:739-744`; length validated `:863-871` | `Config::git_hook_hmac_secret` (`:239`) | `transport.rs:919-921` (`BUZZ_HOOK_SECRET` env for the hook); `policy.rs:236-241` (verification key) |
| `BUZZ_GIT_S3_PROBE` | `"1"` gate | unset | `store.rs:997`, `hydrate.rs:580`, `cas_publish.rs:1582` | — | **test-only**: enables the live-MinIO test bodies |
| `BUZZ_GIT_S3_{ENDPOINT,ACCESS_KEY,SECRET_KEY,BUCKET,REGION}` | `String` | falls back to `BUZZ_S3_*`, then hardcoded MinIO dev values | `cas_publish.rs:1585-1601` (test helper); `crates/buzz-test-client/tests/e2e_git.rs:118-131` | — | **test-only.** `config.rs` never reads them. |

##### Two `BUZZ_GIT_*` names read outside `config.rs`

| Var | Type | Default | Read at | Effect |
|---|---|---|---|---|
| `BUZZ_GIT_CONFORMANCE_PROBE` | `!= "false"` | enabled | `main.rs:470-472` | when `false`, skips the fatal A3 admission gate entirely |
| `BUZZ_GIT_PROBE_WRITERS` | `usize` | `32` | `main.rs:474-477` | `ProbeConfig.race_width`; must be ≥ 2 (`store.rs:572-586`) |
| `BUZZ_GIT_PROBE_ROUNDS` | `usize` | `3` | `main.rs:478-481` | `ProbeConfig.race_rounds`; must be ≥ 1 |

`ProbeConfig::default()` is `{ race_width: 32, race_rounds: 3 }` (`store.rs:114-121`) — matching the `main.rs` defaults, though `main.rs` builds the struct literally rather than using `Default`.

#### 2. Non-`BUZZ_GIT_*` configuration this module depends on

| Config | Source | Used for |
|---|---|---|
| `config.media.s3_{endpoint,access_key,secret_key,bucket,region}` | `BUZZ_S3_*` | **the git object store.** `state.rs:694-701` builds `GitStore` from the *media* config with `.expect("media storage was already constructed with this S3 config")`. There is no dedicated git bucket at runtime. |
| `config.bind_addr` | `BUZZ_BIND_ADDR`, default `0.0.0.0:3000` (`config.rs:365-368`) | only its **port** — the hook callback URL is `http://127.0.0.1:<port>/internal/git/policy` (`transport.rs:911-915`) |
| `config.relay_url` | `RELAY_URL`, default `ws://localhost:3000` | **scheme only** for the NIP-98 `u` derivation (`transport.rs:236-241`); the host always comes from the resolved tenant |
| `config.require_relay_membership` | `BUZZ_REQUIRE_RELAY_MEMBERSHIP`, **default false** (`config.rs:483-485`) | when false, the membership gate in `GitAuth` is a no-op (`api/mod.rs:130-131`) |
| `config.serve_git_web_gui` | `BUZZ_SERVE_GIT_WEB_GUI`, default false (`config.rs:848-850`) | serves the web bundle at `/`, `/repos`, `/repos/*` (`router.rs:206-213`) |
| `state.relay_keypair` | `BUZZ_RELAY_PRIVATE_KEY` | signs kind:30618 (`transport.rs:1690`) |
| `config.uds_path` | `BUZZ_UDS_PATH` | when set, a second listener without `ConnectInfo` — makes `/internal/git/policy` return 403 over UDS (`main.rs:1179-1188`, `mod.rs:41-49`) |
| `PATH` | process env | inherited into every git subprocess (`transport.rs:304`, `hydrate.rs:458-460`) |

#### 3. Derived-default chain

```
BUZZ_GIT_MAX_PACK_BYTES                 = 500 MiB                     (config.rs:713-716)
  └─ BUZZ_GIT_MAX_REPO_BYTES            = pack  × 2  =   1 GiB        (config.rs:717-720)
       └─ BUZZ_GIT_PACK_CACHE_MAX_BYTES = repo  × 5  =   5 GiB        (config.rs:721-724)

BUZZ_GIT_REPO_PATH                      = ./repos                     (config.rs:704-706)
  └─ BUZZ_GIT_PACK_CACHE_PATH           = <repo_path>/.pack-cache     (config.rs:707-712)
```

Both multiplications use `saturating_mul`, so an extreme `max_pack_bytes` saturates instead of wrapping. Consequences of the chain that are easy to miss:

- Raising `BUZZ_GIT_MAX_PACK_BYTES` alone **silently multiplies the pack-cache disk budget by 10×** unless the two downstream vars are also pinned. Setting `pack=2 GiB` yields a 20 GiB cache budget.
- The pack cache lives **inside** the scratch root by default, so a single volume holds ephemeral workspaces, subprocess tempfiles, compaction tempdirs, and up to 5 GiB of cache — and `git_max_repo_bytes` (1 GiB) is the per-request workspace bound on the *same* volume.
- The Helm chart breaks the co-location by pinning `packCachePath` to its own `emptyDir` (`deploy/charts/buzz/templates/deployment.yaml:148`, `:226-241`).

#### 4. Constants that are **not** configurable

These behave like configuration but are compile-time literals.

| Constant | Value | Site |
|---|---|---|
| `INFO_REFS_TIMEOUT` | 120 s | `transport.rs:42` |
| `PACK_OPS_TIMEOUT` | 300 s | `transport.rs:44` |
| `RECEIVE_PACK_MAX_OUTPUT_BYTES` | 1 MiB | `transport.rs:50` |
| `INFO_REFS_MAX_OUTPUT_BYTES` | 4 MiB | `transport.rs:52` |
| `UPLOAD_PACK_MAX_DECODED_BYTES` | 64 MiB | `transport.rs:59` |
| `MAX_FAST_PATH_REFNAME_LEN` | 4096 | `transport.rs:378` |
| `PKT_LINE_MAX_PAYLOAD` | `0xffff - 4` | `transport.rs:422` |
| `PACK_CAPTURE_TIMEOUT` | 300 s | `cas_publish.rs:81` |
| `PACK_COMPACTION_OPERATION_TIMEOUT` | 600 s | `cas_publish.rs:82` |
| `PACK_OBJECTS_WINDOW_MEMORY_BYTES` | 64 MiB | `cas_publish.rs:83` |
| `PACK_OBJECTS_WINDOW` | `"10"` | `cas_publish.rs:84` |
| `MAX_COMPACTION_OBJECTS` | 1 000 000 | `cas_publish.rs:85` |
| `PACK_COMPACTION_SEMAPHORE` | 1 permit | `cas_publish.rs:86` |
| `MAX_REF_SNAPSHOT_BYTES` | 4 MiB | `cas_publish.rs:270` |
| `MAX_COUNT_OBJECTS_OUTPUT_BYTES` | 64 KiB | `cas_publish.rs:595` |
| `MANIFEST_VERSION` | 1 | `manifest.rs:34` |
| `MAX_MANIFEST_PACKS` | 128 | `manifest.rs:39` |
| `PACK_COMPACTION_THRESHOLD` | 96 (`= 128 * 3 / 4`) | `manifest.rs:45` |
| `MAX_MANIFEST_REFS` | 10 000 | `manifest.rs:47` |
| `MAX_MANIFEST_BYTES` | 4 MiB | `hydrate.rs:45` |
| `HEARTBEAT_INTERVAL` | 60 s | `pack_cache.rs:20` |
| `STALE_SESSION_AGE` | 10 min | `pack_cache.rs:21` |
| `MAX_CALLBACK_AGE_SECS` | 30 s | `policy.rs:51` |
| future-skew tolerance | 5 s | `policy.rs:253` |
| policy body limit | 1 MiB | `mod.rs:63` |
| policy ref-update count | 1..=500 | `policy.rs:213` |
| policy `ref_name` max | 256 | `policy.rs:227` |
| hook `curl --max-time` | 10 s | `hook.rs:129` |
| stderr capture prefix | 64 KiB | `transport.rs:678`, `:1122`; `cas_publish.rs:288`, `:551`, `:761` |
| `MAX_PROTECTION_RULES` / `MAX_PATTERN_LENGTH` / `MAX_WILDCARDS_PER_PATTERN` | 50 / 256 / 3 | `buzz-core/src/git_perms.rs:19-23` |
| `DEFAULT_HEAD` | `refs/heads/main` | `handlers/side_effects.rs:2611` |

The hook's 10 s `curl` timeout sits well inside the 300 s `receive-pack` timeout, so a slow policy endpoint fails the push rather than hanging it.

#### 5. Validation and failure behaviour at load

| Check | Behaviour |
|---|---|
| `BUZZ_GIT_REPO_PATH` / `BUZZ_GIT_PACK_CACHE_PATH` | `create_dir_all`; failure is a hard `ConfigError::InvalidValue` naming the setting (`config.rs:388-400`); pinned by `config.rs:1304-1341` |
| Numeric vars | `.ok().and_then(parse)` — **an unparseable value silently falls back to the default**, no warning. Applies to `_MAX_PACK_BYTES`, `_MAX_REPO_BYTES`, `_PACK_CACHE_MAX_BYTES`, `_MAX_REPOS_PER_PUBKEY`, `_MAX_CONCURRENT_OPS`, and both probe vars (`config.rs:713-738`, `main.rs:474-481`) |
| `_PACK_CACHE_MAX_CONCURRENT_POPULATIONS` | additionally `.filter(|v| *v > 0)`, so `0` falls back to 2 (`config.rs:729`); `GitPackCache::new` rejects 0 defensively (`pack_cache.rs:110-113`) |
| `BUZZ_GIT_HOOK_HMAC_SECRET` | if **set**, must be ≥ 32 chars, else hard error (`config.rs:863-871`); if unset, a 64-hex random value is generated with no warning |
| Pack-cache parent is a symlink | hard error at `GitPackCache::new` → `.expect()` in `state.rs:702-709` ⇒ process abort |
| S3 config half-set (one key empty) | `StoreError::Config` → `.expect()` in `state.rs:694-701` ⇒ process abort |
| A3 conformance probe | any phase failure ⇒ `anyhow` error ⇒ startup aborts (`main.rs:490-495`); `race_width < 2` or `race_rounds == 0` fails the `config` phase (`store.rs:572-586`) |
| `BUZZ_GIT_MAX_PACK_BYTES` on 32-bit | `as usize` cast for the body limit (`transport.rs:1757`) would truncate; irrelevant on 64-bit targets |

Zero of the `BUZZ_GIT_*` numeric vars log a warning on a parse failure, and none is range-checked. A typo like `BUZZ_GIT_MAX_CONCURRENT_OPS=20x` silently yields 20.

#### 6. Deployment surfaces

##### `.env.example` (`.env.example:71-79`)

Documents 5 of the 8 real config vars: `_REPO_PATH`, `_MAX_PACK_BYTES`, `_MAX_REPO_BYTES`, `_PACK_CACHE_PATH`, `_PACK_CACHE_MAX_BYTES`, `_PACK_CACHE_MAX_CONCURRENT_POPULATIONS`. **Missing:** `BUZZ_GIT_MAX_REPOS_PER_PUBKEY`, `BUZZ_GIT_MAX_CONCURRENT_OPS`, `BUZZ_GIT_HOOK_HMAC_SECRET`, and all three probe vars.

##### Helm chart (`deploy/charts/buzz/templates/deployment.yaml`)

| Env | Source |
|---|---|
| `BUZZ_GIT_REPO_PATH` | `.Values.persistence.git.mountPath` (`:146`) |
| `BUZZ_GIT_MAX_PACK_BYTES` | `.Values.git.maxPackBytes` (`:147`) |
| `BUZZ_GIT_PACK_CACHE_PATH` | `.Values.git.packCachePath` (`:148`) |
| `BUZZ_GIT_PACK_CACHE_MAX_BYTES` | `.Values.git.packCacheMaxBytes` (`:149`) |
| `BUZZ_GIT_PACK_CACHE_MAX_CONCURRENT_POPULATIONS` | `.Values.git.packCacheMaxConcurrentPopulations` (`:150`) |
| `BUZZ_GIT_MAX_REPOS_PER_PUBKEY` | `.Values.git.maxReposPerPubkey` (`:151`) |
| `BUZZ_GIT_MAX_CONCURRENT_OPS` | `.Values.git.maxConcurrentOps` (`:152`) |
| `BUZZ_GIT_HOOK_HMAC_SECRET` | Kubernetes Secret key (`:168-172`) |
| `BUZZ_S3_*` | ConfigMap + Secret (`:157-159`, `:191-201`) |

Volumes: `git-repos` (PVC or `emptyDir` with `sizeLimit`) at the repo path, and `git-pack-cache` as a dedicated `emptyDir` with `packCacheVolumeSize` (`:226-241`). **`BUZZ_GIT_MAX_REPO_BYTES` is not set by the chart**, so it always takes the `pack × 2` derived value — and if an operator raises `maxPackBytes`, both the repo bound and the cache bound move with it while `packCacheVolumeSize` does not. A `packCacheMaxBytes` larger than `packCacheVolumeSize` results in ENOSPC rather than eviction.

#### 7. Deltas found

| Claim | Source | Reality |
|---|---|---|
| `BUZZ_GIT_S3_BUCKET/_ENDPOINT/_REGION/_ACCESS_KEY/_SECRET_KEY` listed as relay configuration | `.aidlc/reverse-engineer/configuration.md:159` | **Not configuration.** No `config.rs` read exists; the names appear only in `#[cfg(test)]` helpers (`cas_publish.rs:1585-1601`) and the e2e test (`crates/buzz-test-client/tests/e2e_git.rs:118-131`). At runtime the git store reuses `config.media.s3_*` (`state.rs:694-701`). |
| `BUZZ_GIT_HOOK_HMAC_SECRET` "required (git enabled)" | `.aidlc/reverse-engineer/configuration.md:160` | Optional — a random per-process secret is generated when unset (`config.rs:739-744`), and that is functionally safe because the hook always dials `127.0.0.1` into the same process (`transport.rs:911-915`). |
| `BUZZ_GIT_MAX_REPO_BYTES` "consumed at `api/git/hydrate.rs`" | `.aidlc/reverse-engineer/_work/modules/relay-core-configuration.md:98` | Also consumed on the publish side via `PublishLimits` (`transport.rs:1591` → `cas_publish.rs:1136-1146`, `:774-782`). |
| `BUZZ_GIT_MAX_PACK_BYTES` "consumed at `transport.rs:1757` (body limit)" | `relay-core-configuration.md:97` | Six additional consumers: decoded-gzip cap (`transport.rs:861`), three `HydrationOptions` sites, `PublishLimits`, per-pack/per-idx caps (`hydrate.rs:305-317`, `:431-446`), and `--max-pack-size` (`cas_publish.rs:703`). |
| `BUZZ_GIT_PACK_CACHE_MAX_BYTES` default "5 GB" | `.env.example:78` shows `5368709120` | Consistent: `1 048 576 000 × 5 = 5 242 880 000`. The `.env.example` literal `5368709120` (= 5 × 2³⁰) is **not** the derived default — the derived value is 5 242 880 000. Setting the documented value changes behaviour by ~125 MB. |
| Doc: "Each process may retain a byte-bounded, process-lifetime cache … Deployment mounts that cache on a per-pod ephemeral volume" | `docs/git-on-object-storage.md` §v1 | Matches the chart (`deployment.yaml:238-241`) but **not** the code default, which nests the cache inside the scratch root (`config.rs:707-712`). |


## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: Configuration

---

#### 1. Environment variables read *directly* by the 12 assigned files

Exactly **4** env vars are read inside this module's own code, all in `storage_sweep.rs`. Every other configuration input arrives pre-parsed on `state.config`.

| Variable | Parse | Default | Floor/validation | Read at | Consumed at |
|---|---|---|---|---|---|
| `BUZZ_STORAGE_SWEEP_INTERVAL_SECS` | `u64`, unparseable ⇒ default | `3600` | `.max(60)` — a misconfigured value cannot become a listing busy-loop (`:36-38`) | `storage_sweep.rs:56-60` | `main.rs:1462` → `maybe_spawn_sweep(interval)` → `should_spawn` (`storage_sweep.rs:105-127`) |
| `BUZZ_STORAGE_SWEEP_TIMEOUT_SECS` | `u64`, unparseable ⇒ default | `120` | none | `storage_sweep.rs:61-64` | `main.rs:1463` → `tokio::time::timeout` (`storage_sweep.rs:249-252`) |
| `BUZZ_STORAGE_SWEEP_MAX_OBJECTS` | `u64`, unparseable ⇒ default | `1_000_000` | none | `storage_sweep.rs:65-68` | `main.rs:1458`, `:1464-1467` → `buzz_media::fold_bucket_listing` cap check (`bucket_index.rs:392-395`) |
| `BUZZ_STORAGE_METRICS` | trimmed + lowercased; `"off"` ⇒ disabled, anything else **including unset** ⇒ enabled | enabled | none | `storage_sweep.rs:69-73` | `main.rs:1454-1456` — early return suppressing the sweep *and every* storage gauge including health ones |

##### 1.1 Read timing and caching

All four are read **once**, on the first leader tick, behind a function-local `OnceLock` in `main.rs`:

```
static SWEEP_CONFIG: OnceLock<StorageSweepConfig> = OnceLock::new();   // main.rs:1447-1448
let config = *SWEEP_CONFIG.get_or_init(StorageSweepConfig::from_env);   // main.rs:1453
```

The `OnceLock` is deliberately function-local rather than on `Config`/`AppState`, with the in-code rationale that this is localized feature config with a single consumer, stable for the process lifetime because env is immutable (`main.rs:1449-1452`). Consequence: these four vars are **not** validated at boot — a typo is silently absorbed into the default, and nothing is discoverable until the first leader tick.

`StorageSweepConfig` is `Copy` (`storage_sweep.rs:33`), so the deref-copy at `main.rs:1453` is cheap.

##### 1.2 Silent-default behaviour

All four use the `.ok().and_then(parse).unwrap_or(default)` idiom, so `BUZZ_STORAGE_SWEEP_TIMEOUT_SECS=abc` yields 120 with **no warning**. This is the opposite of the crate's `Config::from_env` convention, where an invalid value is a hard `ConfigError` (`config.rs:764-771` for `BUZZ_PUSH_GATEWAY_TIMEOUT_MS`, `:747-751` for `BUZZ_PUSH_EXECUTOR_KEY_ID`, `:563-568` for `RELAY_OPERATOR_PUBKEYS`). Two config subsystems, two error philosophies.

##### 1.3 Kill-switch semantics

Only the exact (trimmed, case-insensitive) string `off` disables (`storage_sweep.rs:69-73`). `BUZZ_STORAGE_METRICS=false`, `=0`, `=disabled` all **enable** the feature. This differs from `config.rs`'s `parse_bool` helper, which accepts `true|1|on` / `false|0|off|""` (`config.rs:364-378`) — and `storage_sweep` does not use it. A unit test pins the asymmetry (`storage_sweep.rs:379-397`, helper `:395-397`).

Documented for operators at `deploy/charts/buzz/values.yaml:308-313`.

---

#### 2. `Config` fields consumed by this module

Read from `state.config`, parsed at boot in `crates/buzz-relay/src/config.rs`.

| Field | Env var | Default | Boot validation | Consumed in this module |
|---|---|---|---|---|
| `relay_operator_pubkeys: Vec<String>` | `RELAY_OPERATOR_PUBKEYS` (comma-separated 64-hex, deduped, lowercased) | **empty ⇒ provisioning disabled** (`config.rs:962-963`) | each entry must be valid 64-hex (`config.rs:563-568`); **must be paired with `RELAY_OPERATOR_API_ORIGIN` or boot fails** (`config.rs:577-580`) | `community_provisioning.rs:258-266` — the operator gate |
| `require_relay_membership: bool` | `REQUIRE_RELAY_MEMBERSHIP` | — | — | `community_provisioning.rs:215` — gates NIP-43 snapshot publication; also overridden to `false` in two test helpers (`workflow_sink.rs:577`) |
| `push_gateway_delivery_url: Option<url::Url>` | `BUZZ_PUSH_GATEWAY_DELIVERY_URL` | **`https://push.buzz.xyz/v1/deliveries/apns`** (`config.rs:339`, `:755-758`) | exact HTTPS + host + no userinfo + path `/v1/deliveries/apns` + no query + no fragment (`config.rs:341-361`) — else boot error | `push_lease.rs:480-482` (gates lease acceptance), `push_runtime.rs:422-424` (delivery target), `main.rs:686` (gates both worker spawns) |
| `push_gateway_timeout: Duration` | `BUZZ_PUSH_GATEWAY_TIMEOUT_MS` | `2000` ms (`config.rs:771`) | integer in `100..=10_000`, else boot error (`config.rs:759-770`) | `push_runtime.rs:314` — whole-request `reqwest` timeout |
| `push_executor_key_id: String` | `BUZZ_PUSH_EXECUTOR_KEY_ID` | `"relay-v1"` (`config.rs:745-746`) | 1..=64 bytes, else boot error (`config.rs:747-751`) | `push_lease.rs:484-486` — must equal the lease's `exec` tag |
| `relay_url: String` | `RELAY_URL` | — | — | `push_lease.rs:494` → `canonical_origin` scheme derivation (`:585-596`); `product_feedback.rs:29` → tenant media base |
| `media: MediaConfig` | `BUZZ_S3_*`, `BUZZ_MEDIA_*` | see §3 | — | via `state.media_storage` in `report.rs:69`, `product_feedback.rs:35`, and the sweep's `list_page` (`main.rs:1462-1470`) |
| `database_url`, `redis_url`, `auth` | — | — | — | **test-only** construction in `identity_archive.rs:441-448` and `workflow_sink.rs:576-596` |

##### 2.1 The push-gateway default is the most consequential setting in this module

Because unset falls back to a concrete third-party URL rather than `None`:

| Consequence | Site |
|---|---|
| Lease acceptance is **on** by default | `push_lease.rs:480-482` returns `push not supported` only when the URL is `None` |
| The matcher **and** delivery worker are spawned by default | `main.rs:686-692` — "enabled as one unit" |
| NIP-11 advertises push support by default | `nip11.rs:208-209` |
| Outbound HTTPS to `push.buzz.xyz` is attempted by default | `push_runtime.rs:422-428` |

Disabling requires setting the variable to an **empty** string (`config.rs:753` matches `Ok(raw) if raw.trim().is_empty() => None`). Unsetting it does the opposite of what an operator would expect. A unit test pins both behaviours (`config.rs:1189-1212`).

---

#### 3. Indirect configuration (consumed via `state`, not read here)

| Setting | Env var | Default | Why this module cares |
|---|---|---|---|
| Usage-tick cadence | `BUZZ_USAGE_METRICS_INTERVAL_SECS` | `300`, `.max(5)` (`main.rs:1257-1262`) | **This is the storage sweep's real retry cadence after a failure** — `should_spawn` returns `true` unconditionally on `!ok` (`storage_sweep.rs:110`), so a permanently failing sweep retries every tick, not every `interval` (documented `:89-103`) |
| Gauge idle timeout | `BUZZ_USAGE_METRICS_IDLE_TIMEOUT_SECS` | `900` (`main.rs:1264-1269`) | the reason `emit_storage_metrics` explicitly zeroes disappearing per-community series instead of relying on idle eviction (`storage_sweep.rs:284-291`) |
| S3 endpoint / credentials / bucket / region | `BUZZ_S3_ENDPOINT`, `BUZZ_S3_ACCESS_KEY`, `BUZZ_S3_SECRET_KEY`, `BUZZ_S3_BUCKET` (default `buzz-media`), `BUZZ_S3_REGION` (falls back to `AWS_REGION`) | `config.rs:620-629` | the sweep lists the whole bucket with these credentials — needs `s3:ListBucket` added to the read-write media key |
| Relay keypair | `RELAY_PRIVATE_KEY` (random if unset, `config.rs:740-744`) | — | signs notice DMs (`moderation_notices.rs:165`, `:198`), workflow messages (`workflow_sink.rs:304`), the outbound NIP-98 header (`push_runtime.rs:562`), and decrypts lease plaintext (`push_lease.rs:488`) |
| Relay owner bootstrap | `RELAY_OWNER_PUBKEY` | — | the **only** supported path to owner promotion, since 9030 and 9032 both refuse `role=owner` (`relay_admin.rs:184-186`, `:300-302`); referenced in-code at `relay_admin.rs:298` and `community_provisioning.rs:38`, `:245` |
| Operator API origin | `RELAY_OPERATOR_API_ORIGIN` | — | NIP-98 URL binding for `POST /operator/communities` (`api/operator.rs:155-163`); required whenever the allowlist is non-empty (`config.rs:577-580`) |
| `BUZZ_RELAY_BASE_URL`, `BUZZ_API_TOKEN`, `BUZZ_RELAY_PUBKEY` | — | `http://localhost:3000` | read by `buzz-workflow`'s `add_reaction_impl` (`buzz-workflow/src/executor.rs:886`, `:895`, `:897`), **not** by `workflow_sink`. That action targets an unregistered route, so these three vars configure a permanently broken path |

---

#### 4. Hard-coded constants that behave like configuration

None of these are env-tunable. They are policy that requires a rebuild to change.

##### 4.1 Freshness / skew

| Constant | Value | Site |
|---|---|---|
| `MAX_COMMAND_SKEW_SECS` (9040–9044) | `120` | `moderation_commands.rs:81` |
| relay-admin skew (bare literal) | `120` | `relay_admin.rs:125` |
| identity-archive skew (bare literal) | `120` | `identity_archive.rs:147` |
| `ALLOWED_SKEW` (lease expiration) | `120` | `push_lease.rs:476` |

Four independent declarations of the same policy value; only one is named.

##### 4.2 Push-lease policy

| Constant | Value | Site |
|---|---|---|
| `MAX_LEASE_TTL` | 30 days (`30*24*60*60`) | `push_lease.rs:475` |
| `MAX_CONTENT` (ciphertext) | 65,536 bytes | `:477` |
| `MAX_PLAINTEXT` | 32,768 bytes | `:478` |
| `MAX_ACTIVE_LEASES` per author | 16 | `:479` |
| `PUSH_KINDS` | `[7, 9, 1059, 40007, 46010]` | `:15` — pinned to `migrations/0018_push_match_queue.sql` by a test (`:696-710`) |
| `URGENT_KINDS` | `[]` (empty) | `:16` — advertised in NIP-11 (`nip11.rs:209`) |
| `MAX_SAFE_JSON_INTEGER` | `2^53 - 1` | `:21` |
| app profiles | `buzz-ios-production`, `buzz-ios-sandbox` (both `apns`) | `:504-512` |
| supported classes | `silent`, `default`, `time_sensitive` | `:509` |
| `max_subscriptions` / `max_kinds` | 16 / 16 | `:513-514` |
| `max_authors` / `max_h` / `max_tag_values` / `max_ignore` | 20 / 50 / 20 / 8 | `:515-518` |
| `max_endpoint_len` / `max_string_len` | 4096 / 512 | `:519-520` |
| `d`-tag length bound | 1..=64 | `:139-141` |

##### 4.3 Push runtime

| Constant | Value | Site |
|---|---|---|
| `CLAIM_SECS` | 30 | `push_runtime.rs:15` |
| `EVENT_USEFUL_SECS` | 3600 | `:16` |
| `MAX_ATTEMPTS` (delivery) | 8 | `:17` |
| `MATCH_BATCH_LIMIT` | 64 | `:20` |
| `IDLE_POLL_FLOOR` / `IDLE_POLL_CEILING` (matcher) | 250 ms / 2 s | `:24-25` |
| delivery-worker idle floor | 500 ms (inline) | `:317`, `:341` |
| `REAP_INTERVAL` | 30 s | `:29` |
| wake claim batch size | 16 (inline) | `:325` |
| default retry delay | 2 s (inline, 4 sites) | `:392`, `:481`, `:486`, `:498` |
| backoff shift clamp | `2^6` | `:544` |
| `MAX_MATCH_ATTEMPTS` | 8 (`buzz-db/src/push.rs:70`) | referenced `:172`, `:190` |
| batch-retry delay | 2 s | `:141`, `:210` |
| claim-error sleep | 2 s | `:84` |

##### 4.4 Moderation / admin / feedback limits

| Constant | Value | Site |
|---|---|---|
| `MAX_WORKSPACE_ICON_URL_LEN` | 2048 | `relay_admin.rs:60` |
| `MAX_WORKSPACE_ICON_DATA_URL_LEN` | 98,304 (~72 KB image) | `relay_admin.rs:64` |
| `MAX_HOST_LEN` | 255 (matches `communities.host VARCHAR(255)`) | `community_provisioning.rs:39` |
| DNS total / label length | 253 / 63 | `community_provisioning.rs:151`, `:158` |
| notice idempotency scan limit | 1000 (the query clamp) | `moderation_notices.rs:240`, rationale `:222-226` |
| `MAX_BODY_BYTES` (feedback) | 32,768 | `product_feedback.rs:12` |
| `MAX_TAGS_BYTES` (feedback) | 65,536 | `product_feedback.rs:13` |
| feedback categories | `bug`, `praise`, `needs-work` | `product_feedback.rs:11` |
| `REPORT_TYPES` | 7 NIP-56 values | `report.rs:29-37` |
| report note size | **uncapped** | `report.rs:222-228` |
| `MODERATION_READ_LIMIT` | 500 | `api/bridge.rs:2089` |
| `MODERATION_SOURCE_TAG` | `"moderation_source"` | `moderation_notices.rs:38` |
| `MODERATION_ACTION_CHECK_VOCAB` | 12 values, must match `migrations/0006_moderation.sql` | `buzz-db/src/moderation.rs:104-117`, pinned by `moderation_commands.rs:659-667` |

**Notable gaps:** ban/timeout durations have no configurable maximum (any `expiration` in `i64` range is accepted, `moderation_commands.rs:592-606`); the report note has no size cap while feedback does; the push quotas are the only limits documented as "advertised descriptor policy" yet none of them are actually advertised via NIP-11 except `push_kinds`/`urgent_kinds`.

---

#### 5. Legacy `SPROUT_*` names

**Zero `SPROUT_*` variables are read by any of the 12 assigned files.** Verified by grep.

For completeness, the legacy names surviving elsewhere in `buzz-relay`:

| Variable | Read at | Purpose |
|---|---|---|
| `SPROUT_MAX_NOT_BEFORE_DELTA` | `nip11.rs:97`, `ingest.rs:1325` | advertised in NIP-11 and enforced at ingest |
| `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS` | `main.rs:701` | NIP-ER reminder scheduler cadence |
| `SPROUT_REMINDER_SCHEDULER_BATCH_LIMIT` | `main.rs:705` | NIP-ER batch size |

All three belong to other module groups. This module group is fully on the `BUZZ_*` prefix.

---

#### 6. Configuration-related findings

| # | Severity | Finding |
|---|---|---|
| C-1 | **HIGH** | `BUZZ_PUSH_GATEWAY_DELIVERY_URL` unset ⇒ `Some("https://push.buzz.xyz/v1/deliveries/apns")` (`config.rs:755-758`), which enables lease acceptance (`push_lease.rs:480`), spawns both background workers (`main.rs:686-692`), advertises push in NIP-11 (`nip11.rs:208`), and attempts outbound HTTPS carrying the device token to a host the operator never chose. Disabling requires an explicitly **empty** value (`config.rs:753`) — unsetting does the opposite of the intuitive thing. |
| C-2 | MEDIUM | The four `BUZZ_STORAGE_*` vars silently absorb invalid values into defaults (`storage_sweep.rs:56-73`), unlike every `Config::from_env` field which hard-fails at boot (`config.rs:747-751`, `:764-771`, `:563-568`). A typo is undiscoverable. |
| C-3 | MEDIUM | Those same four vars are read only on the **first leader tick** behind a function-local `OnceLock` (`main.rs:1447-1453`), so a non-leader pod never validates them and a misconfiguration surfaces late and only on one pod. |
| C-4 | MEDIUM | The storage sweep's effective retry cadence after a failure is `BUZZ_USAGE_METRICS_INTERVAL_SECS` (default 300 s), not `BUZZ_STORAGE_SWEEP_INTERVAL_SECS` (`storage_sweep.rs:105-127`). The coupling is documented in-code (`:89-103`) but the variable name implies otherwise. |
| C-5 | LOW | `BUZZ_STORAGE_METRICS` uses bespoke `off`-only parsing rather than the crate's `parse_bool` (`config.rs:364-378`), so `false`/`0`/`disabled` all **enable** the feature. Pinned by a test that documents the asymmetry (`storage_sweep.rs:379-397`). |
| C-6 | LOW | `BUZZ_STORAGE_SWEEP_TIMEOUT_SECS` and `_MAX_OBJECTS` have no floor or ceiling. `TIMEOUT_SECS=0` yields an immediately-timing-out sweep that respawns on every usage tick forever (`should_spawn` returns `true` on `!ok`, `storage_sweep.rs:110`). Only `INTERVAL_SECS` is floored. |
| C-7 | LOW | Owner promotion has no event-based path — 9030 and 9032 both refuse `owner` (`relay_admin.rs:184-186`, `:300-302`) — so it is config-only via `RELAY_OWNER_PUBKEY` or the operator endpoint. 9030's error message "use kind:9032 to promote to owner" (`:185`) points at a path that also refuses. |
| C-8 | LOW | 12 push-lease quotas plus 13 push-runtime timing constants are compile-time only (`push_lease.rs:475-521`, `push_runtime.rs:15-29`). Tuning batch size, claim lease, retry ceiling, or per-author lease quota for a differently-sized deployment requires a rebuild. |
| C-9 | LOW | `moderation_notices.rs:240` hard-codes the idempotency scan limit to 1000 with the reasoning that "1000 moderation notices to one user in one community is a practical ceiling" (`:225-226`). Beyond that, crash-retry can re-send a duplicate notice. |
| C-10 | INFO | Only `BUZZ_STORAGE_*` appears in `deploy/charts/buzz/values.yaml` (`:308-313`) among this module's direct env surface. The push-gateway vars are not documented in the chart. |


## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Configuration

---

#### 1. Environment variables consumed (directly or via `Config`)

| Env var | Parse rule | Default | Read at | Consumed in this group at |
|---|---|---|---|---|
| `BUZZ_HUDDLE_AUDIO_AVAILABLE` | `!(v == "false" \|\| v == "0")` — i.e. **anything except `false`/`0` is true**, including typos | **`true`** | `config.rs:487-491` | `handler.rs:357` |
| `BUZZ_MESH` | `eq_ignore_ascii_case("on") \|\| v == "true" \|\| v == "1"` | **`false`** (absent, `off`, or a typo → off) | `config.rs:494-500` | `mesh_boot.rs:418-421` |
| `BUZZ_MESH_BIND_ADDR` | `SocketAddr::parse`; a parse failure is a **hard `ConfigError::InvalidValue`**; an *absent* var falls back to the default | `0.0.0.0:3478` | `config.rs:501-508` | `mesh_boot.rs:423-431`, logged `:436` |
| `BUZZ_MESH_DEMO_ECHO` | same strict `on`/`true`/`1` as `BUZZ_MESH` | **`false`** | `config.rs:514-518` | `mesh_boot.rs:293-297`; gate `api/mesh_demo.rs:70-72`; passed at `main.rs:456` |
| `BUZZ_MESH_ADVERTISE_ADDR` | trimmed; empty string ignored | unset | **read directly from the environment**, not via `Config` | `mesh_boot.rs:384-389` |
| `POD_IP` | trimmed; used only when a bound port is known | unset | **read directly** | `mesh_boot.rs:394-399` |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | `v == "true" \|\| v == "1"` | **`false`** | `config.rs:481-483` | reached via `enforce_relay_membership` at `handler.rs:244`; the gate itself short-circuits at `api/mod.rs:67` |
| `REDIS_URL` | used only by test helpers | `redis://127.0.0.1:6379` | `directory.rs:593`, `reliable.rs:708`, `api/mesh_demo.rs:170` | test-only |

`BUZZ_MESH_ADVERTISE_ADDR` and `POD_IP` are the only two env vars in this group read
with a bare `std::env::var` instead of going through `Config`
(`mesh_boot.rs:384`, `:394`). They are therefore invisible to `Config::from_env`'s
validation and to any config-dump/diagnostic surface.

##### 1.1 Advertise-address preference chain (`mesh_boot.rs:379-402`)

1. `BUZZ_MESH_ADVERTISE_ADDR` if set and non-empty → used verbatim, **no
   `SocketAddr` validation** (`mesh_boot.rs:384-389`). A malformed value is
   published to the ready registry and silently breaks peer dialing.
2. `POD_IP` + the actual bound port, when both are usable
   (`mesh_boot.rs:392-399`). Comment: "k8s Downward API, zero RBAC".
3. Every IP transport address the endpoint reports (`mesh_boot.rs:401`).

The bound port is taken from `ip_addrs().first()` (`mesh_boot.rs:392-393`); if that
is empty the port is `0` and branch 2 is skipped, falling through to an empty
vector from branch 3 — publishing a ready record with **no addresses** and no error.

---

#### 2. `Config` fields consumed

| Field | Type | Default | Definition | Read at |
|---|---|---|---|---|
| `huddle_audio_available` | `bool` | `true` | `config.rs:114-129` | `handler.rs:357` |
| `mesh: buzz_relay_mesh::MeshConfig` | struct | see §2.1 | `config.rs:131-136` | `mesh_boot.rs:418`, `:423`, `:445` |
| `mesh_demo_echo` | `bool` | `false` | `config.rs:138-144` | `main.rs:456` → `mesh_boot.rs:293` |
| `relay_url` | `String` | — | (elsewhere) | `handler.rs:219` via `nip42_expected_relay_url` |
| `require_relay_membership` | `bool` | `false` | `config.rs:110-112` | indirectly, `api/mod.rs:67` |

##### 2.1 `MeshConfig` (`buzz-relay-mesh/src/lib.rs:54-63`)

| Field | Source | Default | Tunable? |
|---|---|---|---|
| `enabled` | `BUZZ_MESH` | `false` | yes |
| `bind_addr` | `BUZZ_MESH_BIND_ADDR` | `0.0.0.0:3478` | yes |
| `registry_refresh` | **hard-coded** `Duration::from_secs(15)` | 15 s | **no** — `config.rs:511` writes a literal |

`registry_refresh` is a `MeshConfig` field with no env var behind it. It controls the
ready-registry heartbeat cadence (`mesh_boot.rs:445`), so ready-record staleness on
a mesh deployment is not operator-tunable.

---

#### 3. Other `AppState` inputs this group depends on

| Input | Where set | Consumed |
|---|---|---|
| `conn_semaphore: Arc<Semaphore>` sized by `max_connections` | `state.rs:727` | `handler.rs:90` — **shared with ordinary relay WebSockets**, so `max_connections` is the only operator-tunable audio limit |
| `community_connections: Arc<CommunityConnectionRegistry>` | `state.rs:724` | `handler.rs:153-164` |
| `audio_rooms: Arc<AudioRoomManager>` | `state.rs:768` | throughout |
| `relay_keypair: nostr::Keys` | `state.rs` | `handler.rs:1268` (lifecycle-event signing); `mesh_boot.rs:441-443`, `:452` (mesh attestation) |
| `redis_pool: deadpool_redis::Pool` | `state.rs` | `main.rs:444` → `mesh_boot.rs:442`, `:512` |
| `shutting_down: Arc<AtomicBool>` | `state.rs:769` | `main.rs:445`, `:457` → `mesh_boot.rs:466-472`, `:481-496`, `:317` |
| `tracer: Arc<dyn Tracer>` | `state.rs:794-798` | **unconditionally `NoopTracer`** |
| `mesh: Arc<OnceLock<MeshHandle>>` | `state.rs:627`, set `main.rs:458` | `state.rs:812-814` |

---

#### 4. `BUZZ_HUDDLE_AUDIO_AVAILABLE` — verifying the documented single-pod constraint

##### 4.1 What the doc says

`config.rs:114-129`:

> Huddle audio frames are relayed peer-to-peer *within a single pod*
> (`AudioRoomManager` is an in-process map; only huddle lifecycle events cross pods
> via Redis). Under horizontal scaling … two peers in the same huddle can land on
> different pods and never hear each other. … Operators running multiple relay pods
> MUST set `BUZZ_HUDDLE_AUDIO_AVAILABLE=false` until the out-of-relay media/SFU
> service lands.

##### 4.2 What actually breaks with more than one pod, mesh off

Verified accurate. `AudioRoomManager` is a per-process `DashMap`
(`room.rs:490-492`), constructed once per `AppState` (`state.rs:768`). Frame
fan-out (`room.rs:398-411`) iterates only that map. Nothing crosses pods except the
three lifecycle events, which go through `pubsub.publish_event`
(`handler.rs:1322-1325`). So two peers on different pods:

- each create their own `Room` for the same `(community, channel)` key;
- each independently pin a protocol version;
- each independently allocate `peer_index` from 0, so indices collide across pods;
- see a `joined` roster containing only their own pod's peers
  (`handler.rs:614-619` from `room.peer_pubkeys()`);
- hear nothing from the other pod;
- **each become "the last peer"** on leave and each run
  `remove_peer_and_check_ended` → `archive_channel` (`handler.rs:833-866`), so the
  channel is archived while the other pod's peer is still connected, and two 48103
  events are emitted (the second de-duplicated only if the event ids collide, which
  they will not — different `created_at`/content ordering).

The failure is worse than "never hear each other": the auto-end/archive path is
actively wrong under multi-pod. So the `MUST` in the doc is well-founded.

##### 4.3 Does the mesh path change that? Yes — and it also bypasses the gate

`handler.rs:306-378` is a two-arm match on `state.mesh()`:

```rust
match state.mesh() {
    Some(mesh) => { /* drain check, resolve_join_owner_ready */ }
    None => { if !state.config.huddle_audio_available { /* reject */ } }
}
```

**`huddle_audio_available` is only consulted on the `None` arm.** With `BUZZ_MESH=on`,
setting `BUZZ_HUDDLE_AUDIO_AVAILABLE=false` has **no effect at all** — joins are
accepted and routed through the mesh. This is intentional (the comment at
`handler.rs:289-296` says "When the mesh is off, we keep today's behavior exactly —
including the `huddle_audio_available=false` rejection under a non-mesh horizontal
deployment") but the *field documentation* at `config.rs:120-128` was written before
the mesh landed and still says operators "MUST set
`BUZZ_HUDDLE_AUDIO_AVAILABLE=false`" with no mention of the mesh. **Documented
delta.** An operator following `config.rs` on a mesh-enabled multi-pod deployment
would set a flag that does nothing.

With the mesh on, multi-pod huddles work correctly: one pod owns the room (Redis
CAS, `join.rs:317-379`), that pod is the sole `peer_index` allocator
(`mesh.rs:18-24`), and ingress pods never archive (`handler.rs:803-810`). So the
mesh does remove the single-pod constraint — the config doc has simply not caught up.

##### 4.4 The `false` value is nearly unreachable

The parse rule is `!(v == "false" || v == "0")` (`config.rs:489-491`). Only the exact
lowercase `false` or `0` disables it. `False`, `FALSE`, `no`, `off`, `disabled`, and
an empty string all evaluate to **true**. Contrast `BUZZ_MESH`, which uses
`eq_ignore_ascii_case("on")` (`config.rs:498-500`) — three flags in the same file
with three different parse conventions:

| Flag | Convention | Case-insensitive? |
|---|---|---|
| `BUZZ_HUDDLE_AUDIO_AVAILABLE` | deny-list (`false`/`0` → off) | no |
| `BUZZ_MESH` | allow-list (`on`/`true`/`1` → on) | `on` only |
| `BUZZ_MESH_DEMO_ECHO` | allow-list (`on`/`true`/`1` → on) | `on` only |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | allow-list (`true`/`1` → on) | no |

##### 4.5 Test coverage of the defaults

| Assertion | Line |
|---|---|
| `huddle_audio_available` defaults true | `config.rs:980-984` |
| `BUZZ_HUDDLE_AUDIO_AVAILABLE=false` disables it | `config.rs:1256-1265` |
| `BUZZ_MESH` absent → mesh off | `mesh_boot.rs:546-556` (skips if externally forced) |
| mesh off boots nothing (unroutable Redis proves it) | `mesh_boot.rs:527-541` |

---

#### 5. `BUZZ_MESH` / `BUZZ_MESH_BIND_ADDR` gating — verified behaviour

| Condition | Behaviour |
|---|---|
| `BUZZ_MESH` absent/off | `boot_mesh` returns `Ok(None)` at `mesh_boot.rs:418-421` before touching anything. `state.mesh()` stays `None`, so every consumer takes its single-pod branch (`handler.rs:306`, `:449`, `:577`, `:875`). `/_mesh` returns `{"enabled": false}` (`router.rs:404`). `/_mesh/demo/echo` 404s (`api/mesh_demo.rs:66-68`). No UDP port bound, no Redis key written, no task spawned — pinned by `mesh_boot.rs:527-541` using an unroutable Redis pool |
| `BUZZ_MESH=on`, bind succeeds, publish succeeds | endpoint bound on `bind_addr`, boot-unique `RuntimeId` (`mesh_boot.rs:423-432`), ready record published (`:459-464`), heartbeat spawned (`:466-472`), `MeshRuntime` started (`:475`), immediate `reconcile_now()` (`:478`), drain watcher spawned (`:481-496`), dispatcher installed as the single inbound slot (`:505`) |
| `BUZZ_MESH=on`, bind fails | **fatal** — `anyhow!("mesh endpoint bind on {} failed: {e}")` propagates to `main.rs:442` and aborts startup (`mesh_boot.rs:423-431`) |
| `BUZZ_MESH=on`, first ready-registry publish fails | **fatal** — `anyhow!("mesh ready-registry publish failed: {e}")` (`mesh_boot.rs:459-463`). Rationale at `:457-458`: "if Redis can't take the attested record, peers can never find us — fail loudly now, not quietly forever" |
| `BUZZ_MESH_BIND_ADDR` malformed | **fatal at config load** — `ConfigError::InvalidValue` (`config.rs:501-508`) |
| `BUZZ_MESH_BIND_ADDR` absent | default `0.0.0.0:3478` (`config.rs:508`) — binds on **all interfaces**. §7 |

`main.rs:454-459` wires consumers, then publishes the handle, then logs. The
`OnceLock::set` failure is `unreachable!("mesh handle is set exactly once, right
here")` (`main.rs:459`) — a genuine single-writer invariant, not a swallowed error.

Ordering note: consumers are registered **before** `state.mesh.set(handle)`, so
inbound mesh traffic can be served before any local join on this pod can resolve
ownership. That is the correct order (a peer must not hit an unregistered slot), and
`mesh_boot.rs:47-55` documents the residual boot-window drop as bounded and safe
because of fencing.

---

#### 6. Hard-coded values that should be tunable

Grouped by operational consequence.

##### 6.1 Capacity / QoS — no env var exists

| Constant | Value | Line | Why it should be tunable |
|---|---|---|---|
| `MAX_PEERS_PER_ROOM` | 25 | `room.rs:50` | The doc itself calls it "the soft one" and shows the `N×(N−1)` math (`room.rs:47-50`). A deployment with fatter pods or smaller huddles has no way to move it. This is the single most likely value an operator would want to change |
| `AUDIO_CHANNEL_CAPACITY` | 8 (≈160 ms) | `room.rs:40` | Jitter tolerance vs. latency tradeoff; hardware- and network-dependent |
| `CTRL_CHANNEL_CAPACITY` | 32 | `room.rs:45` | Sized by a comment ("even 30 simultaneous join/leave events fit") that is tied to `MAX_PEERS_PER_ROOM`; if the peer cap moved, this would need to move with it — and neither can |
| `MAX_AUDIO_FRAME_BYTES` | 4096 | `handler.rs:44` | Caps Opus bitrate implicitly. A higher-fidelity or multi-channel configuration would need this raised |
| WS data / control queue depths | 16 / 8 | `handler.rs:659-660` | Same tradeoff, second layer |
| `roster_tx` broadcast capacity | 64 | `room.rs:179` | Determines how much roster churn a slow control stream can absorb before a full resync |

##### 6.2 Timing — no env var exists

| Constant | Value | Line | Consequence |
|---|---|---|---|
| `HEARTBEAT_INTERVAL` | 30 s | `handler.rs:55` | With `MAX_MISSED_PONGS=3` this fixes dead-peer detection at 60–90 s. Behind an aggressive LB idle timeout this is too slow; on a lossy mobile link, too aggressive |
| `MAX_MISSED_PONGS` | 3 | `handler.rs:58` | as above |
| `AUTH_TIMEOUT` | 5 s | `handler.rs:61` | Also the window during which an unauthenticated client holds a shared `conn_semaphore` permit (see security §6.1) — the one timing constant with a direct DoS consequence |
| `DEFAULT_LEASE_TTL` | 30 s | `directory.rs:17` | `with_lease_ttl` exists (`directory.rs:187-189`) and tests use it (`directory.rs:601`), but production calls `SessionDirectory::new` (`mesh_boot.rs:512`), so the TTL is fixed. TTL is the failover-latency dial for cross-pod huddles |
| `DEFAULT_HUDDLE_RENEW_INTERVAL` | 10 s | `join.rs:452` | Must stay ≪ TTL; coupled to a value that is also fixed |
| `DEFAULT_RENEW_INTERVAL` (reliable) | 10 s | `reliable.rs:34` | duplicate of the above, in a second file |
| `OWNER_READY_RETRY_INTERVAL` / `_MAX_ATTEMPTS` | 20 ms / 25 | `join.rs:387-388` | Bounds a join's worst-case latency at ~500 ms and its Redis round trips at 25 |
| `registry_refresh` | 15 s | `config.rs:511` | A `MeshConfig` field with no env var — see §2.1 |
| drain-watcher poll | 500 ms | `mesh_boot.rs:495` | SIGTERM→drain latency |
| demo-echo drain poll | 100 ms | `mesh_boot.rs:315` | demo only |
| `ECHO_TIMEOUT` | 10 s | `api/mesh_demo.rs:45` | demo only |

##### 6.3 Protocol / wire — arguably correct to hard-code

| Constant | Value | Line | Note |
|---|---|---|---|
| `CURRENT_PROTOCOL_VERSION` | 2 | `handler.rs:123-124` | Correct as a compile-time constant, but there is **no way to pin a deployment to v1** during a staged rollout even though the doc says "older versions stay supported indefinitely for staged rollouts" (`handler.rs:120-122`). A `BUZZ_HUDDLE_MAX_PROTOCOL_VERSION` would make that claim actionable |
| `default_protocol_version()` | 1 | `handler.rs:140-142` | Correct — backwards compatibility |
| `V2_HEADER_LEN` / `FLAG_DTX` | 8 / `0x01` | `wire.rs:29`, `:33` | Wire format; must not be tunable. Duplicated in `desktop/src-tauri/src/huddle/wire.rs:48` — two copies of one protocol constant |
| `MAX_RELIABLE_PAYLOAD_BYTES` | 1 MiB | `reliable.rs:31` | Justified against goose's 50 MiB bodies (`reliable.rs:26-30`) and asserted against the 16 MiB wire cap (`reliable.rs:945`) |
| `ReliableWireFrame::VERSION` | 1 | `reliable.rs:423` | wire format |
| `PROTO_VERSION` | `WIRE_VERSION as u16` | `mesh_boot.rs:367` | derived, correct |
| `capabilities()` | all three profiles, static | `mesh_boot.rs:369-377` | Correct — "All three tunnel profiles ship in the same binary". But it means a pod cannot advertise *not* serving huddles, so there is no capability-based way to keep huddle ownership off a given pod |

##### 6.4 Conformance — no configuration at all

| Item | Status |
|---|---|
| Tracer selection | **unconditional** `NoopTracer` at `state.rs:794-798`. No env var, no config field, no cargo feature. `crates/buzz-relay/Cargo.toml` `[features]` contains only `dev = ["buzz-auth/dev"]` |
| `JsonlTracer` output path | `create<P: AsRef<Path>>` (`tracers.rs:36`) takes a path — but nothing calls it, so no path is ever configured |
| Feature-gated elision | `tracers.rs:11-13` says "the build can have the compiler eliminate them entirely behind a feature flag if the cost ever shows up in benches". No such flag exists, so the emit arguments are constructed and discarded on every ingest and REQ |
| Seam names | `"ingest_event_exited_without_trace"` (`ingest.rs:1411`), `"row_community_lookup_missing"` (`conformance/mod.rs:251`) — string literals, correctly not configurable |

Making the tracer selectable (`BUZZ_CONFORMANCE_TRACE=<path>`) is the single change
that would turn this module from inert scaffolding into an operable diagnostic.

---

#### 7. Deployment-facing configuration notes

- **`0.0.0.0:3478` binds all interfaces by default.** `config.rs:508`. On a
  mesh-enabled deployment the mesh QUIC endpoint is reachable from anywhere the pod
  is. Admission is gated on the Redis ready registry
  (`buzz-relay-mesh/src/runtime.rs:275-283`), so an unattested dialer is rejected,
  but the port is open and the default is not a loopback or pod-network address.
  `3478` is the IANA STUN port — coincidental, and potentially confusing to a
  network operator reading a port scan.
- **`/_mesh/demo/echo` is on the public router** (`router.rs:123`), not the admin
  host, and is unauthenticated when both mesh flags are on. See security §5.
- **`/_mesh` is on the health router** (`router.rs:230`) which
  `router.rs:222-224` documents as having "No metrics middleware, no auth, no CORS,
  no body limit". It returns peer runtime ids and addresses.
- **`max_connections` is the only operator dial that affects huddle capacity**, and
  it is shared with every other relay WebSocket (`state.rs:727`, `handler.rs:90`).
  There is no way to reserve or cap the audio share of that budget.
- **Nothing in `.env.example` or `deploy/compose/.env.example`** was found to
  document `BUZZ_HUDDLE_AUDIO_AVAILABLE`, `BUZZ_MESH`, `BUZZ_MESH_BIND_ADDR`,
  `BUZZ_MESH_DEMO_ECHO`, `BUZZ_MESH_ADVERTISE_ADDR`, or `POD_IP` — operators
  discover them only from `config.rs` doc comments and `mesh_boot.rs`.
- **Redis is a hard dependency for mesh-enabled huddles**, not a cache: joins fail
  (`join_rejected`) when the directory is unreachable, and a renewal failure is
  treated as owner loss, closing every local owner client on that pod
  (`join.rs:521-529` → `handler.rs:756-765`).


## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Configuration

The crate reads **no environment variables itself** — verified: `std::env` appears
nowhere in `crates/buzz-relay-mesh/src`. All configuration is resolved by
`buzz-relay` and passed in as `MeshConfig` (`lib.rs:53-64`) plus two direct env
reads in `mesh_boot.rs`.

---

#### 1. Environment variables

Complete set (grep for `BUZZ_MESH` + `POD_IP` across the repo, excluding the
unrelated MeshLLM `BUZZ_MESH_API_PORT`/`BUZZ_MESH_CONSOLE_PORT`/
`BUZZ_MESH_IROH_RELAYS` desktop vars, `desktop/src-tauri/src/mesh_llm/mod.rs:37-47`):

| Variable | Default | Accepted values | Resolved at | Consumed at |
|---|---|---|---|---|
| `BUZZ_MESH` | **off** | `on` (case-insensitive), `true`, `1` → enabled; **absent, `off`, or any other value → disabled** | `config.rs:498-500` | `mesh_boot.rs:417` |
| `BUZZ_MESH_BIND_ADDR` | `0.0.0.0:3478` | any parseable `SocketAddr`; parse failure is a **fatal `ConfigError::InvalidValue`** | `config.rs:501-507` | `MeshEndpoint::bind`, `mesh_boot.rs:383`; logged `mesh_boot.rs:394` |
| `BUZZ_MESH_ADVERTISE_ADDR` | unset | free-form string, trimmed; empty → ignored. **Not validated as a `SocketAddr` at boot** — a typo is only discovered when a *peer* fails to parse it (`runtime.rs:328-334`) | read inline, `mesh_boot.rs:384-389` | published in `ReadyRecord.endpoint_addrs` / `GossipRecord.endpoint_addrs` |
| `POD_IP` | unset | IP string, trimmed; used only when non-empty **and** a bound port was resolved | read inline, `mesh_boot.rs:398-403` | same |
| `BUZZ_MESH_DEMO_ECHO` | **off** | `on`/`true`/`1` only (same strict parser) | `config.rs:516-518` | `main.rs:456` → `wire_consumers(demo_echo)`; route gate `api/mesh_demo.rs`; owner-side echo `mesh_boot.rs:290-292` |

Two more relay-level vars materially affect the mesh's exposure:

| Variable | Default | Relevance |
|---|---|---|
| `BUZZ_HEALTH_PORT` (`config.health_port`) | 8080 | the listener carrying `GET /_mesh` (`router.rs:230`), bound to **hard-coded `0.0.0.0`** at `main.rs:1119`, ignoring `BUZZ_BIND_ADDR` |
| `BUZZ_RELAY_PRIVATE_KEY` (→ `state.relay_keypair`) | — | the secp256k1 key that signs and anchors every ready-record attestation (`mesh_boot.rs:445`, `:450`) |

##### Parser semantics (verified)

```rust
// config.rs:498-500
let mesh_enabled = std::env::var("BUZZ_MESH")
    .map(|v| v.eq_ignore_ascii_case("on") || v == "true" || v == "1")
    .unwrap_or(false);
```

Note the asymmetry: `on` is case-insensitive, `true`/`1` are exact. `TRUE`, `On `
(trailing space), `yes`, `enabled` all evaluate to **off**. Deliberate — rationale at
`config.rs:494-497`: "an image upgrade with untouched env must not bind a new UDP
port or write a new Redis key… anything else (absent, `off`, other values) keeps
exact single-instance behavior." `BUZZ_MESH_DEMO_ECHO` uses the identical parser
(`config.rs:514-518`, comment "same strict pattern as BUZZ_MESH — explicit
`on`/`true`/`1` only, anything else (absent, `off`, typos) is off"). The cost of the
strictness is silent misconfiguration: `BUZZ_MESH=ON` yields a meshless relay with
no warning.

`BUZZ_MESH_BIND_ADDR` uses `unwrap_or_else(|_| Ok(default))?` (`config.rs:502-507`),
so a **missing** var falls back to the default while a **malformed** var is fatal —
the correct split.

---

#### 2. `MeshConfig` fields

`lib.rs:53-64`, `#[derive(Clone, Debug)]`, no `Default`, no `Deserialize`.
Constructed only at `config.rs:508-512`; stored as `Config.mesh` (`config.rs:136`).

| Field | Type | Value source | Default | Read at |
|---|---|---|---|---|
| `enabled` | `bool` | `BUZZ_MESH` | `false` | `mesh_boot.rs:417` |
| `bind_addr` | `std::net::SocketAddr` | `BUZZ_MESH_BIND_ADDR` | `0.0.0.0:3478` | `mesh_boot.rs:383`, `:396` |
| `registry_refresh` | `std::time::Duration` | **hardcoded** | `15s` (`config.rs:511`) | `mesh_boot.rs:447` → `ReadyRegistry::new` |

**`registry_refresh` has no environment variable.** `config.rs:511` is a literal
`std::time::Duration::from_secs(15)`. The crate exposes
`DEFAULT_REGISTRY_REFRESH = 15s` (`registry.rs:20`) for exactly this purpose and the
relay does not use it (grep: zero external callers). Operators cannot tune heartbeat
cadence, and therefore cannot tune the derived **45 s registry TTL**
(`REGISTRY_EXPIRY_MULTIPLIER = 3`, `registry.rs:21`; `expiry_for`, `:154-156`) — the
window in which a crashed pod remains dialable.

##### Documented-default delta

`lib.rs:55-56` documents `BUZZ_MESH` as "`on` (default when replicas can exist) |
`off` kill switch." The implementation defaults it **off**, unconditionally, with no
replica detection. `config.rs:131-135` and `mesh_boot.rs:4-6` state the correct
behaviour, and `mesh_boot.rs:544-555` tests it
(`mesh_defaults_off_when_env_absent`). The `lib.rs` doc comment is stale — it
describes the design as originally proposed, before the review blocker noted at
`mesh_boot.rs:544-546` ("Blocker fix (Wren review of 8b077fdb): absent `BUZZ_MESH`,
the mesh is OFF"). Also, "kill switch" implies default-on semantics that no longer
apply.

---

#### 3. Compile-time constants (the real tuning surface)

None of these are configurable at runtime; all require a rebuild.

| Constant | Value | Location | Overridable? |
|---|---|---|---|
| `DEFAULT_GOSSIP_INTERVAL` | 2 s | `runtime.rs:44` | only via `MeshRuntime::start_with_intervals` (`:102`) — **zero external callers**, tests only |
| `DEFAULT_RECONCILE_INTERVAL` | 5 s | `runtime.rs:42` | same |
| `CONTROL_QUEUE_DEPTH` | 64 | `runtime.rs:46` | no — private const |
| `DEFAULT_PHI_SUSPECT_THRESHOLD` | 8.0 | `membership.rs:17` | only via `with_phi_suspect_threshold` (`:66`) — **zero callers**; the relay never overrides it (`mesh_boot.rs:444-445`) |
| `PhiAccrual` sample window | 100 | `gossip.rs:176` (`Default::new(100)`) | only via `PhiAccrual::new` — zero external callers |
| `DEFAULT_REGISTRY_REFRESH` | 15 s | `registry.rs:20` | unused by the relay (hardcoded duplicate at `config.rs:511`) |
| `REGISTRY_EXPIRY_MULTIPLIER` | 3 | `registry.rs:21` | no |
| `READY_KEY_PREFIX` | `"mesh:ready:"` | `registry.rs:19` | no |
| `ATTESTATION_CONTEXT` | `"buzz-relay-mesh-ready-v1"` | `registry.rs:22` | no |
| `GOSSIP_PAYLOAD_VERSION` | 1 | `gossip.rs:13` | no |
| `ALPN` | `b"buzz/mesh/1"` | `wire.rs:37` | no |
| `WIRE_VERSION` | 1 | `wire.rs:42` | no |
| `MAX_STREAM_FRAME` | 16 MiB | `wire.rs:46` | no |
| `PROTO_VERSION` (relay) | `WIRE_VERSION as u16` = 1 | `mesh_boot.rs:367` | no |
| capabilities list | `["reliable-stream","realtime-media","huddle-control"]` | `mesh_boot.rs:371-377` | no — static, "all three tunnel profiles ship in the same binary" |
| drain-watcher poll | 500 ms | `mesh_boot.rs:495` | no |
| demo-echo drain tick | 100 ms | `mesh_boot.rs:319` | no |
| demo-echo `ECHO_TIMEOUT` | 10 s | `api/mesh_demo.rs:~41` | no |

Derived timings worth recording: with the shipped values a peer is marked
`Suspect` after `8 × ln(10) × 2s ≈ 36.8 s` of gossip silence (BR-MESH-34), its
registry record expires after 45 s, and a dead peer is re-dialed every 5 s
indefinitely with no backoff.

---

#### 4. Address advertisement resolution

`advertise_addrs(&MeshEndpoint)` (`mesh_boot.rs:382-403`), documented at
`mesh_boot.rs:379-381`. Strict precedence, first match wins:

1. **`BUZZ_MESH_ADVERTISE_ADDR`** (`:384-389`) — trimmed; empty string falls through.
   Returns a single-element list. Intended for "explicit, classic-LB shapes."
2. **`POD_IP` + actual bound port** (`:391-403`) — the port comes from
   `endpoint.ip_addrs().first().port()`, i.e. the *real* bound port, so
   `BUZZ_MESH_BIND_ADDR=0.0.0.0:0` works. Requires `bound_port != 0`. Intended for
   k8s Downward API with "zero RBAC."
3. **All endpoint IP transport addrs** (`:403`) — `endpoint.ip_addrs()`
   (`endpoint.rs:61-71`, filters `TransportAddr::Ip`, drops relay paths). Dev/local
   fallback.

Consumed identically by both the Redis record (`ReadyRecord::new(…, addrs, …)`,
`mesh_boot.rs:448-453`) and the gossip record (`GossipRecord::new(runtime_id, addrs,
PROTO_VERSION)`, `mesh_boot.rs:439`).

Gaps:

- `BUZZ_MESH_ADVERTISE_ADDR` is **never parsed at boot**. It is stored as a `String`
  (`registry.rs:105-107` explains the string choice as layer independence) and only
  parsed by the *dialing peer* (`runtime.rs:329-335`), where a bad value produces
  `warn!("mesh: bad peer addr")` on every remote pod every 5 s. A typo is silently
  self-isolating.
- Path 3 can advertise loopback or a container-internal address in environments
  where neither env var is set, producing a mesh that forms only within one host.
- The advertised list is captured **once at boot** (`mesh_boot.rs:385`) and republished
  verbatim by the heartbeat (`ReadyRecord` is moved into `spawn_registry_heartbeat`,
  `mesh_boot.rs:467-471`). If the pod's IP changes, the record never updates.

---

#### 5. Deployment configuration state

- **No deployment in this repo enables the mesh.** grep for `BUZZ_MESH` across
  `.env.example`, `deploy/`, and the helm charts returns nothing. There is likewise
  no `3478` port declaration and no `POD_IP` Downward-API stanza anywhere in the repo.
  The k8s wiring the code anticipates (`lib.rs:59-61` "Excluded from istio sidecar
  capture in k8s"; `mesh_boot.rs:380` Downward API) lives in the external
  `squareup/block-coder-tf-stacks` repo, if at all.
- **`BUZZ_HUDDLE_AUDIO_AVAILABLE`** is the current answer to horizontal scaling:
  `config.rs:120-129` instructs operators running multiple relay pods to set it
  `false` "until the out-of-relay media/SFU service lands." That is the shipped
  multi-pod story; the mesh is the not-yet-enabled replacement.
- `BUZZ_MESH_BIND_ADDR` defaults to **`0.0.0.0`** — all interfaces. In any
  environment where the mesh is enabled without network policy, the QUIC endpoint is
  publicly reachable (see `-security.md` F-04, F-06).

---

#### 6. Configuration-related test coverage

| Test | Asserts | Location |
|---|---|---|
| `mesh_off_boots_nothing` | `boot_mesh` returns `None` and never touches Redis (pool points at unroutable `redis://127.0.0.1:1`) | `mesh_boot.rs:526-541` |
| `mesh_defaults_off_when_env_absent` | absent `BUZZ_MESH` ⇒ `config.mesh.enabled == false`; self-skips if the var is externally forced | `mesh_boot.rs:544-555` |
| `expiry_is_three_refreshes` | `expiry_for(15s) == 45s` | `registry.rs:324-327` |
| `ready_key_is_stable_and_namespaced` | `ready_key(rid(0xAB)) == "mesh:ready:" + "ab"*32` | `registry.rs:317-322` |

Untested: `BUZZ_MESH_BIND_ADDR` parse-failure fatality, `advertise_addrs`'
three-way precedence (all three branches), `BUZZ_MESH_DEMO_ECHO` gating, and the
`registry_refresh` value flowing through to the Redis TTL.

---

#### 7. Summary of configuration findings

1. `registry_refresh` (and hence the 45 s TTL) is unreachable from the environment
   despite `MeshConfig` exposing the field and `registry.rs:20` providing the default
   constant.
2. Gossip interval, reconcile interval, phi threshold, and phi sample window are all
   API-overridable but **have zero callers** — effectively compile-time constants.
3. `lib.rs:55-56` documents the wrong default for `BUZZ_MESH` (says on, is off).
4. `BUZZ_MESH_ADVERTISE_ADDR` accepts an unvalidated string; failures surface on
   remote pods, not at boot.
5. The strict-opt-in parsers silently ignore near-miss values (`ON`, `yes`) with no
   warning.
6. `/_mesh` rides a listener whose bind address is hard-coded `0.0.0.0`
   (`main.rs:1119`) and cannot be restricted by `BUZZ_BIND_ADDR`.


## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: Configuration

This aspect is genuinely thin, and the thinness is the finding: **a subsystem whose entire
purpose is switchable observation has no switch.** There is no environment variable, no config
struct field, no CLI flag, and no Cargo feature that selects a tracer. The only runtime
selector is a hard-coded constructor line.

---

### Cargo features

**None.** `crates/buzz-conformance/Cargo.toml` (35 lines) has `[package]`
(`:1-8`), `[dependencies]` (`:25-29`), and `[dev-dependencies]` (`:33-34`). There is no
`[features]` table — verified by grep. No `#[cfg(feature = ...)]` appears anywhere in `src/`.

Two comments promise a feature that does not exist:

| Claim | Location |
|---|---|
| "Zero cost: the build can omit emission entirely behind a feature" | `src/lib.rs:321-322` |
| "the build can have the compiler eliminate them entirely behind a feature flag if the cost ever shows up in benches" | `crates/buzz-relay/src/conformance/tracers.rs:9-13` |

`buzz-relay`'s dependency is unconditional: `buzz-conformance = { workspace = true }`
(`crates/buzz-relay/Cargo.toml:20`) with no `optional = true` and no feature gate, so the crate
compiles into every relay build.

---

### Environment variables

| Variable | Scope | Read at | Effect |
|---|---|---|---|
| `BUZZ_CONFORMANCE_UPDATE` | test-only | `tests/replay_fixtures.rs:210` (`std::env::var(...).is_ok()`) | when set to anything, `assert_fixture_matches` writes the golden JSONL instead of comparing (`:211-214`) |

That is the complete list. Grep for `CONFORMANCE`, `TRACER`, and `BUZZ_TRACE` across
`.env.example` and `crates/buzz-relay/src/config.rs` returns nothing. `.env.example` has no
tracing/conformance section.

Note the variable is presence-checked, not value-parsed — `BUZZ_CONFORMANCE_UPDATE=0`
regenerates the fixtures just as `=1` does.

---

### How a non-`Noop` tracer would be selected

There is no selection mechanism. The single binding site is:

```
tracer: Arc::new(crate::conformance::NoopTracer),
```
`crates/buzz-relay/src/state.rs:798`, inside `AppState`'s constructor. The field is
`pub tracer: Arc<dyn buzz_conformance::Tracer>` (`:620`), so it is publicly readable and
writable — but grep across `crates/buzz-relay/src/` finds exactly one assignment (`:798`) and
four reads (`handlers/ingest.rs:1383`, `handlers/req.rs:145`, `:356`, `:672`).

Switching to `JsonlTracer` therefore requires one of:

1. Mutating `AppState.tracer` after construction — the path the constructor comment envisions:
   "Conformance tests overwrite this with a JsonlTracer after construction (see test helpers in
   `crates/buzz-test-client` once those land)" (`state.rs:794-797`). Those helpers do not exist;
   `crates/buzz-test-client/tests/conformance_multitenant.rs` never references
   `buzz_conformance` (verified by grep).
2. Editing `state.rs:798` and rebuilding.

`JsonlTracer::create` (`crates/buzz-relay/src/conformance/tracers.rs:37-45`) takes the output
path as a plain `P: AsRef<Path>` argument — no default, no env fallback, no directory
convention. It is never called anywhere in the repo.

---

### Constants that behave like configuration

| Constant | Value | Line | Notes |
|---|---|---|---|
| `SCHEMA_VERSION` | `1` | `src/lib.rs:86` | compared at `src/checker.rs:85`, `:97`; stamped by `TraceStep::new` (`:305`) |
| `ProptestConfig::with_cases` | `128` | `tests/proptest_checker.rs:193` | per-property case count; not overridable via env in this file |
| `POOL` | `3` | `tests/proptest_checker.rs:51` | community/channel pool width, chosen so foreign-vs-resolved collisions are frequent (`:44-49`) |
| `EmitGuard` seam name | `"ingest_event_exited_without_trace"` | `crates/buzz-relay/src/handlers/ingest.rs:1411` | `&'static str`, passed as `kind` |
| projection breach tag | `"row_community_lookup_missing"` | `crates/buzz-relay/src/conformance/mod.rs:250` | `&'static str` |

`PROPTEST_CASES` and the other standard proptest env overrides are honoured by the proptest
library itself, but the crate ships no `proptest-regressions/` directory and no
`proptest.toml` — verified by `ls crates/buzz-conformance` (only `Cargo.toml`, `LIMITS.md`,
`TRACE_SCHEMA.md`, `src/`, `tests/`).

---

### Test-invocation configuration

| Entry point | Command | Line |
|---|---|---|
| `just test-unit` (nextest present) | `cargo nextest run -p buzz-conformance` | `justfile:290` |
| `just test-unit` (fallback) → `scripts/run-tests.sh unit` | `cargo test -p buzz-conformance -- --nocapture` | `scripts/run-tests.sh:98-99` |
| `just ci` | `check test-unit desktop-test …` | `justfile:266` |

Both run all targets (lib + `proptest_checker` + `replay_fixtures`); the justfile comment at
`:286-289` states the intent ("Run all targets (lib + the tests/replay_fixtures.rs integration
test), not just --lib"). No `--ignored` tests exist in this crate, so nothing is gated behind a
second invocation.

`crates/buzz-conformance/LIMITS.md:88-118` documents a three-command CI contract
(`cargo test -p buzz-conformance --lib`, `--test replay_fixtures`, and
`cargo test -p buzz-relay --lib conformance::`) and totals it as "9 + 5 + 2 = 16 tests". The
actual counts are 9 lib + 6 `replay_fixtures` + 7 `proptest_checker` = 22 in this crate, plus 9
in `crates/buzz-relay/src/conformance/mod.rs` and 1 in `handlers/ingest.rs`. The proptest lane
is absent from that doc's command list entirely, and no justfile recipe or GitHub workflow runs
the `-p buzz-relay --lib conformance::` command it calls mandatory (grep `conformance` in
`.github/workflows/` returns nothing).


## Module: buzz-acp — harness core & orchestration (`crates/buzz-acp/src`)
### Aspect: Configuration

Config comes from clap-with-`env` (`config.rs`), an optional TOML file, and four env vars read directly by `lib.rs`/`setup_mode.rs`. There are 44 `env = "…"` declarations in `config.rs` (`grep -c 'env = "'`).

#### Env vars read directly by `lib.rs` (bypassing `Config`)

| Var | Site | Default | Gates |
|---|---|---|---|
| `BUZZ_AUTH_TAG` | `lib.rs:125-137`, `lib.rs:1338-1341`, `lib.rs:4173-4179` | unset | (1) owner resolution via NIP-OA attestation, highest priority; (2) relay membership delegation tag on connect; (3) forwarded into the MCP server env. Empty string is filtered out at all three sites. |
| `BUZZ_MANAGED_AGENT_START_NONCE` | `lib.rs:1503` | `""` via `unwrap_or_default()` | correlation ID stamped into every `managed_agent_runtime_lifecycle` observer event (`lib.rs:93-121`) |
| `BUZZ_ACP_SETUP_PAYLOAD` | `setup_mode.rs:83` (const), read `setup_mode.rs:214` | unset | when present, `lib.rs:1298-1303` diverts to the setup listener and the agent pool never starts |
| `RUST_LOG` (implicit) | `EnvFilter::try_from_default_env()` `lib.rs:1278` | `buzz_acp=info` | tracing filter |

#### `Config` vars consumed by `lib.rs`

| Var | `config.rs` | Default | What `lib.rs` gates on it |
|---|---|---|---|
| `BUZZ_PRIVATE_KEY` | 243 | **required** | `config.keys` — relay auth, event signing, MCP injection (`lib.rs:4159`) |
| `BUZZ_RELAY_URL` | 240 | `ws://localhost:3000` | `HarnessRelay::connect` (`lib.rs:1343`), MCP env (`lib.rs:4156`) |
| `BUZZ_ACP_AGENT_COMMAND` | 250 | `goose` | every spawn path (`lib.rs:1762`, `3486`, `3664`, `3731`) |
| `BUZZ_ACP_AGENT_ARGS` | 255 | `acp` | same; comma-delimited |
| `BUZZ_ACP_MCP_COMMAND` | 261 | `""` | `build_mcp_servers` early-returns empty on `""` (`lib.rs:4142-4144`) — **default gives the agent no MCP servers** |
| `BUZZ_ACP_AGENTS` | 292 | `1` (1–32) | pool size (`lib.rs:1318`), `crash_history` length (`lib.rs:1688`), `respawn_tx` capacity (`lib.rs:1613`) |
| `BUZZ_ACP_IDLE_TIMEOUT` | 266 | `DEFAULT_IDLE_TIMEOUT_SECS` = **900** (`config.rs:27`) | `PromptContext.idle_timeout` (`lib.rs:1533`) |
| `BUZZ_ACP_MAX_TURN_DURATION` | 270 | `DEFAULT_MAX_TURN_DURATION_SECS` = 7200 (`config.rs:31`) | `PromptContext.max_turn_duration` (`lib.rs:1534`), `queue.with_in_flight_deadline` (`lib.rs:1506`), steer deadline extension (`lib.rs:2488`), user-facing timeout text (`lib.rs:3092`, `3106`, `3258`) |
| `BUZZ_ACP_TURN_LIVENESS_SECS` | 302 | `10` | `PromptContext.turn_liveness_interval` (`lib.rs:1535`) |
| `BUZZ_ACP_HEARTBEAT_INTERVAL` | 297 | `0` (disabled) | `heartbeat` interval (`lib.rs:1574-1582`); `0` leaves the arm permanently pending |
| `BUZZ_ACP_HEARTBEAT_PROMPT` / `_FILE` | 308, 316 | built-in | `PromptContext.heartbeat_prompt` (`lib.rs:1550`), falls back to `default_heartbeat_prompt()` (`lib.rs:3560`) |
| `BUZZ_ACP_SUBSCRIBE` | 326 | `mentions` | rule construction (`lib.rs:1445-1474`) |
| `BUZZ_ACP_KINDS` | 332 | unset | `kinds_override`; under `mentions` replaces the 9/46010/40007 default (`lib.rs:1447`), under `all` replaces an **empty** list (`lib.rs:1456`) |
| `BUZZ_ACP_CHANNELS` | 335 | unset | `resolve_channel_filters` (`lib.rs:1476`) |
| `BUZZ_ACP_NO_MENTION_FILTER` | 338 | `false` | `require_mention: !no_mention_filter` (`lib.rs:1451`) |
| `BUZZ_ACP_CONFIG` | 341 | `./buzz-acp.toml` | `config::load_rules` under `subscribe=config` (`lib.rs:1471`) |
| `BUZZ_ACP_DEDUP` | 344 | `queue` | `EventQueue::new` (`lib.rs:1505`); `Drop` also disables the panic-recovery batch copy (`lib.rs:2915-2918`) |
| `BUZZ_ACP_MULTIPLE_EVENT_HANDLING` | 355 | `queue` | `mode_gate_signal` (`lib.rs:2233`) |
| `BUZZ_ACP_NO_IGNORE_SELF` | 361 | `false` | self-authored event drop (`lib.rs:2027`) |
| `BUZZ_ACP_CONTEXT_MESSAGE_LIMIT` | 366 | `12` | `PromptContext.context_message_limit` (`lib.rs:1562`) |
| `BUZZ_ACP_MAX_TURNS_PER_SESSION` | 372 | `0` | `PromptContext.max_turns_per_session` (`lib.rs:1563`) |
| `BUZZ_ACP_NO_PRESENCE` | 377 | `false` | initial/heartbeat/offline presence (`lib.rs:1511`, `1583`, `2680`) |
| `BUZZ_ACP_NO_TYPING` | 381 | `false` | 3 s typing tick (`lib.rs:1593`) |
| `BUZZ_ACP_MEMORY` / `BUZZ_ACP_NO_MEMORY` | 395, 404 | memory on | `PromptContext.memory_enabled` (`lib.rs:1568`); disabled logs to target `engram::core` (`lib.rs:1575-1580`) |
| `BUZZ_ACP_NO_BASE_PROMPT` | 409 | `false` | drops the bundled `base_prompt.md` (`lib.rs:1541-1547`) |
| `BUZZ_ACP_BASE_PROMPT_FILE` | 416 | unset | replaces `include_str!("base_prompt.md")`; the loaded string is `Box::leak`ed (`lib.rs:1545`) |
| `BUZZ_ACP_MODEL` | 423 | unset | `OwnedAgent.desired_model` at spawn (`lib.rs:1781`, `3803`), `PoolStartup.model` (`lib.rs:3735`) |
| `BUZZ_ACP_PERMISSION_MODE` | 434 | see `config.rs` | `PromptContext.permission_mode` (`lib.rs:1566`) |
| `BUZZ_ACP_RESPOND_TO` | 444 | `owner-only` | `author_allowed` (`lib.rs:2155`); warnings at `lib.rs:1376-1390` |
| `BUZZ_ACP_RESPOND_TO_ALLOWLIST` | 452 | empty | allowlist set (`lib.rs:2156`) |
| `BUZZ_ACP_ALLOWED_RESPOND_TO` | 460 | empty | only reaches `config.summary()` (`config.rs:1019-1025`) — no `lib.rs` reference |
| `BUZZ_ACP_TEAM_INSTRUCTIONS` | 464 | unset | `PromptContext.team_instructions` (`lib.rs:1539`) |
| `BUZZ_ACP_SYSTEM_PROMPT` / `_FILE` | 279, 286 | unset | `PromptContext.system_prompt` (`lib.rs:1538`) |
| `BUZZ_ACP_INITIAL_MESSAGE` | 321 | unset | `PromptContext.initial_message` (`lib.rs:1532`) |
| `BUZZ_ACP_RELAY_OBSERVER` | 468 | `false` | in-process observer (`lib.rs:1307`), control subscribe + publisher (`lib.rs:1399-1435`) |
| `BUZZ_ACP_LAZY_POOL` | 472 | `false` | empty-slot pool + wake state machine (`lib.rs:1317-1322`, `1709-1748`) |
| `BUZZ_ACP_AGENT_OWNER` | 247 | unset | fallback owner when `BUZZ_AUTH_TAG` is absent or fails (`lib.rs:147`) |

#### Vars read outside `Config`

| Var | Site | Default | Notes |
|---|---|---|---|
| `BUZZ_ACP_EVENT_BUFFER` | `relay.rs:35-41` | `EVENT_CHANNEL_CAPACITY_DEFAULT` | parsed at connect time, clamped to ≥ 1 because `mpsc::channel` panics on 0; declared in `.env.example:221` but has **no clap flag** — env-only |
| `CODEX_CONFIG` | `acp.rs:439-462` | unset | recursively merged with persona-supplied values when `has_generated_codex_config` |

#### Legacy aliases

`propagate_legacy_env_vars()` (`config.rs:715-726`), called from `run()` before the tokio runtime starts (`lib.rs:1234`) because `set_var` needs a single-threaded process under Rust 2024:

| Legacy | Canonical |
|---|---|
| `BUZZ_ACP_PRIVATE_KEY` | `BUZZ_PRIVATE_KEY` |
| `BUZZ_ACP_API_TOKEN` | `BUZZ_API_TOKEN` |

`BUZZ_ACP_TURN_TIMEOUT` (`config.rs:274`, `hide = true`) is a third legacy alias handled by precedence rather than propagation: explicit `--idle-timeout` > `--turn-timeout` > default (`config.rs:848-866`).

#### Parsed-and-never-read

**`BUZZ_API_TOKEN` is dead.** `grep -rn 'BUZZ_API_TOKEN' crates/buzz-acp/src/` returns exactly one hit — the propagation table at `config.rs:718`. There is no clap flag, no `Config` field, no read site. Yet `crates/buzz-acp/README.md § Core` documents it as "API token (required if relay enforces token auth)". The harness propagates the legacy alias into an env var that nothing in the crate consumes.

**`allowed_respond_to` is display-only.** Declared at `config.rs:460`, stored on `Config`, and referenced only by `config.summary()` (`config.rs:1019-1025`). No `lib.rs` gate reads it; `author_allowed` takes `respond_to` and `respond_to_allowlist` only (`lib.rs:2154-2159`).

#### Missing from `.env.example`

`.env.example` documents 25 `BUZZ_ACP_*` / `BUZZ_*` vars (lines 121–226). Absent from it entirely:

`BUZZ_ACP_AGENT_OWNER`, `BUZZ_ACP_IDLE_TIMEOUT`, `BUZZ_ACP_MAX_TURN_DURATION`, `BUZZ_ACP_TURN_LIVENESS_SECS`, `BUZZ_ACP_MULTIPLE_EVENT_HANDLING`, `BUZZ_ACP_MEMORY`, `BUZZ_ACP_NO_MEMORY`, `BUZZ_ACP_NO_BASE_PROMPT`, `BUZZ_ACP_BASE_PROMPT_FILE`, `BUZZ_ACP_PERMISSION_MODE`, `BUZZ_ACP_RESPOND_TO`, `BUZZ_ACP_RESPOND_TO_ALLOWLIST`, `BUZZ_ACP_ALLOWED_RESPOND_TO`, `BUZZ_ACP_TEAM_INSTRUCTIONS`, `BUZZ_ACP_RELAY_OBSERVER`, `BUZZ_ACP_LAZY_POOL`, `BUZZ_AUTH_TAG`, `BUZZ_MANAGED_AGENT_START_NONCE`, `BUZZ_ACP_SETUP_PAYLOAD`.

That is 19 undocumented vars against 25 documented ones. Four of the missing ones are security-relevant (`RESPOND_TO`, `RESPOND_TO_ALLOWLIST`, `PERMISSION_MODE`, `AUTH_TAG`) and two control the entire startup mode (`LAZY_POOL`, `SETUP_PAYLOAD`).

Conversely, `.env.example:152` documents the **deprecated hidden** alias `BUZZ_ACP_TURN_TIMEOUT=320` rather than the canonical `BUZZ_ACP_IDLE_TIMEOUT`, and `.env.example:221` documents `BUZZ_ACP_EVENT_BUFFER=256`, which has no CLI flag.

#### Default-value drift

| Var | Code default | `README.md § Core` |
|---|---|---|
| `BUZZ_ACP_IDLE_TIMEOUT` | **900** (`config.rs:27`) | **620** |
| `BUZZ_ACP_MAX_TURN_DURATION` | 7200 (`config.rs:31`) | 7200 ✓ |
| `BUZZ_ACP_MCP_COMMAND` | `""` (`config.rs:261`) | `""` ✓ |

#### TOML file config

`--config` / `BUZZ_ACP_CONFIG` (default `./buzz-acp.toml`) is loaded only under `subscribe=config` (`lib.rs:1469-1472`). `config::load_rules` is documented at `lib.rs:1470` as already warning on a zero-rule file. Per-channel `[channel.UUID]` blocks with `kinds` and `require_mention` are documented in `README.md § Forum Channels`.

#### Config values that panic or abort on bad input

- `rustls` provider install: `.expect(...)` at `lib.rs:1243`.
- Malformed secret key: `.expect("secret key bech32 encoding should never fail")` at `lib.rs:4167`.
- Fatal-at-startup config errors (relay connect, membership subscribe, observer control subscribe, channel discovery) return `Err` from `tokio_main` (`lib.rs:1345`, `1360`, `1428`, `1440`).
- Zero live agents after eager pool startup is fatal (`lib.rs:3827-3832`); a partial pool is only a warning (`lib.rs:3833-3839`).
- No channel subscriptions resolved is only a warning — "agent will sit idle" (`lib.rs:1477-1479`).


## Module: buzz-acp — relay client & observer (`crates/buzz-acp/src`)
### Aspect: Configuration

#### Environment variables read in these three files

Exactly one:

| Variable | Parse site | Type | Default | Gates | In `.env.example`? |
|---|---|---|---|---|---|
| `BUZZ_ACP_EVENT_BUFFER` | `relay.rs:35-42` | `usize`, `.max(1)` | `EVENT_CHANNEL_CAPACITY_DEFAULT` = 256 (`relay.rs:29`) | capacity of **both** the `event_tx` channel (`relay.rs:610`) and the `observer_control_tx` channel (`relay.rs:611-612`) | Yes — `.env.example:221`, commented out: `# BUZZ_ACP_EVENT_BUFFER=256` |

`event_channel_capacity()` is called once per `HarnessRelay::connect`
(`relay.rs:610-612`), so the value is read at connect time, not process start.
The `.max(1)` guard exists because `mpsc::channel` panics on capacity 0
(`relay.rs:40`). A non-numeric value silently falls back to 256 — no warning is
emitted.

The doc comment at `relay.rs:27-28` says the variable overrides "the event
channel" capacity; it is actually applied to two channels. `.env.example:219-220`
describes it as "Event channel buffer capacity (WebSocket → harness)", which
matches the doc comment and understates the observer-control effect.

`observer.rs` and `engram_fetch.rs` read no environment variables.

#### Variables consumed by this module but read elsewhere

These reach `relay.rs` as function arguments, not through `std::env` in these
files:

| Variable | Read at | Path into this module |
|---|---|---|
| `BUZZ_RELAY_URL` (via `Config.relay_url`) | `config.rs` clap `env` attribute | `HarnessRelay::connect(&config.relay_url, …)` (`lib.rs:1344`), stored at `relay.rs:545`, used by `do_connect` (`relay.rs:3830`), `send_auth_response` (`relay.rs:3440`, `:3446`), and `relay_ws_to_http` for `RestClient.base_url` (`relay.rs:722`) |
| `BUZZ_PRIVATE_KEY` (via `Config.keys`) | `config.rs` | `HarnessRelay::connect(…, &config.keys, …)`, cloned into `bg_keys` (`relay.rs:615`) and `RestClient.keys` (`relay.rs:723`) |
| `BUZZ_AUTH_TAG` | `lib.rs:1370-1373` (`std::env::var` + `buzz_sdk::nip_oa::parse_auth_tag`) | `auth_tag: Option<nostr::Tag>` argument, stored at `relay.rs:549`, serialised to `auth_tag_json` for the `x-auth-tag` header (`relay.rs:724-727`), and injected into the AUTH event (`relay.rs:3444-3456`) |

`BUZZ_ACP_NO_TYPING` (`.env.example:216`) gates `config.typing_enabled`, which
controls whether the 3-second refresh tick that calls `build_typing_event` is
created at all (`lib.rs:1593-1596`). `BUZZ_ACP_NO_MEMORY` gates
`ctx.memory_enabled`, which controls whether `engram_fetch::build_core_section`
is reached (`pool.rs:1382`). Neither is read in these files.

#### Compile-time constants — `relay.rs`

| Constant | Value | Location | Effect |
|---|---|---|---|
| `EVENT_CHANNEL_CAPACITY_DEFAULT` | 256 | `:29` | fallback for `BUZZ_ACP_EVENT_BUFFER` |
| `CMD_CHANNEL_CAPACITY` | 64 | `:31` | harness → bg command channel; not env-tunable |
| `SEEN_ID_LIMIT` | 12_000 | `:45` | dedup set, rotates at 6,000 |
| `PING_INTERVAL` | 30 s | `:48` | client-initiated ping |
| `PONG_TIMEOUT` | 10 s | `:51` | no-pong → reconnect |
| `WS_SEND_TIMEOUT_SECS` | 10 | `:54` | every `ws.send()` |
| `STABLE_CONNECTION_SECS` | 60 | `:59` | resets `backoff_step` to 0 |
| `SINCE_SKEW_SECS` | 5 | `:57` | subtracted from `since` on resubscribe |
| `AUTH_TIMEOUT` | 20 s | `:64` | both NIP-42 steps; comment records the raise from 5 s |
| `CONNECT_TIMEOUT` | 30 s | `:69` | TCP + WS handshake; comment records the raise from 10 s |
| `STARTUP_CONNECT_BACKOFFS` | `[1,2,4,8,16] s` | `:83-89` | shared by `retry_initial_connect` and `try_autonomous_reconnect` |
| `DNS_RETRY_INTERVAL` | 2 s | `:98` | flat DNS retry, no ladder rung |
| `REQ_PACING_INTERVAL` | 125 ms | `:107` | ≈8 REQ/s |
| `DRAIN_BUDGET_PER_ITER` | 1 | `:113` | frames per select tick |
| `GATED_OBSERVER_QUEUE_CAP` | 256 | `:116` | parked + in-flight observer frames |
| `REST_RETRY_BASE_DELAYS` | `[500, 1000, 2000] ms` | `:249-253` | HTTP bridge retry |
| `MEMBERSHIP_NOTIF_SUB_ID` | `"membership-notif"` | `:497` | subscription id |
| `OBSERVER_CONTROL_SUB_ID` | `"agent-observer-control"` | `:499` | subscription id |
| `CHANNEL_ACCESS_DENIED_REASONS` | two exact strings | `:3498-3501` | drop-channel-not-connection |

Local (function-scope) constants: `MAX_DNS_FLAT_RETRIES` = 10
(`relay.rs:2914`) bounds the DNS brownout retry in `try_autonomous_reconnect`;
the `wait_for_reconnect` ladder `[1,2,4,8,16,32] s` with a 60 s tail is an inline
array (`relay.rs:3055-3062`, `:3128-3132`) rather than a named constant, so it
cannot be tuned or asserted alongside `STARTUP_CONNECT_BACKOFFS`.

None of these are env-overridable. Notably `CMD_CHANNEL_CAPACITY` (64) bounds
`try_publish_event`, which is the typing-indicator path — a full command channel
silently drops the event (`relay.rs:834-840`) — and there is no way to raise it
without a rebuild.

#### Compile-time constants — `observer.rs` / `engram_fetch.rs`

| Constant | Value | Location | Effect |
|---|---|---|---|
| `OBSERVER_BUFFER_CAP` | 1_000 | `observer.rs:19` | sizes both the broadcast channel (`:46`) and the replay `VecDeque` (`:51`) |
| `SECTION_LABEL` | `"Agent Memory — core"` | `engram_fetch.rs:23` | prompt section header |
| `ONBOARDING_NUDGE` | fixed sentence | `engram_fetch.rs:29-31` | rendered on confirmed absence |

The engram query's `limit(16)` is a bare literal at `engram_fetch.rs:88`, not a
named constant.

#### Parsed-and-never-read / unused configuration

- `send_subscribe` takes `_state: &BgState` and never reads it (`relay.rs:3162`) — a vestigial parameter threaded through six call sites (`relay.rs:1390`, `:2306-2313`, `:2544`, `:2708`, `:2765`).
- `HarnessRelay::publish_event` is `#[allow(dead_code)]` (`relay.rs:820-821`); no in-repo caller uses it.
- `relay.rs:3849-3854` computes `event_id` from `wait_for_any_ok` and uses it only in a debug log (`relay.rs:3856`) — the value it was intended to match against is never derived.
- No env var in this module is parsed and then discarded; `BUZZ_ACP_EVENT_BUFFER` is the only one and it is used.

#### Documentation drift on configuration

`.env.example:219-220` and the doc comment at `relay.rs:27-28` both describe
`BUZZ_ACP_EVENT_BUFFER` as sizing the WebSocket → harness event channel only.
It also sizes the observer-control channel (`relay.rs:611-612`), so an operator
tuning it for high message throughput silently changes observer-control
backpressure behaviour too — and that channel drops on full with a warning
(`relay.rs:2072-2074`) rather than triggering the replay machinery that channel
events get.

`ARCHITECTURE.md:452-458` documents typing indicators as a Redis sorted-set
protocol; the kind:20002 ephemeral event this module builds
(`relay.rs:866-868`, published from `lib.rs:2333-2341`) is handled by the
generic ephemeral fan-out instead — there is no `KIND_TYPING_INDICATOR`
reference in `crates/buzz-relay/src` or `crates/buzz-pubsub/src`. Any operator
reading that section to size Redis is sizing something this path does not use.


## Module: buzz-acp — agent pool & lifecycle (`crates/buzz-acp/src`)
### Aspect: Configuration

#### Direct env reads in this module

Zero. Neither `pool.rs` nor `pool_lifecycle.rs` calls `std::env::var`, `env!`, or reads a config file. Every knob arrives pre-resolved as a field on `PromptContext` (`pool.rs:482-532`), as a `PoolStartup` field, or as a compile-time `const`. All parsing happens in `config.rs` via clap `#[arg(long, env = ...)]`; `PromptContext` is assembled once at `lib.rs:1529-1563`.

#### Knobs that reach the pool

| Env var | Flag | Default | Parse site | What it gates in this module |
|---|---|---|---|---|
| `BUZZ_ACP_AGENTS` | `--agents` | `1`, range `1..=32` | `config.rs:292-295` | slot-vector length (`lib.rs:1318`, `:3747`); pool width |
| `BUZZ_ACP_LAZY_POOL` | `--lazy-pool` | `false` | `config.rs:471-473` | whether `PoolLifecycle` gates subprocess start (`lib.rs:1317-1322`, `:1714`) |
| `BUZZ_ACP_IDLE_TIMEOUT` | `--idle-timeout` | `900` s (`DEFAULT_IDLE_TIMEOUT_SECS`, `config.rs:27`) | `config.rs:266-267`, resolved `config.rs:849-866` | `ctx.idle_timeout` — per-turn silence cut and the grace passed to `cancel_with_cleanup` (`pool.rs:1616`, `:1648`, `:2077`) |
| `BUZZ_ACP_TURN_TIMEOUT` | `--turn-timeout` (hidden, deprecated) | none | `config.rs:274-275` | alias for idle timeout; `--idle-timeout` wins (`config.rs:848`) |
| `BUZZ_ACP_MAX_TURN_DURATION` | `--max-turn-duration` | `7200` s (`config.rs:31`) | `config.rs:270-271` | `ctx.max_turn_duration` — hard wall-clock cap (`pool.rs:1617`, `:1835`); must be `>` idle timeout or startup fails (`config.rs:894-898`) |
| `BUZZ_ACP_TURN_LIVENESS_SECS` | `--turn-liveness-secs` | `10` | `config.rs:302-303` | `ctx.turn_liveness_interval`; `0` disables emission (`pool.rs:3172-3174`) |
| `BUZZ_ACP_MAX_TURNS_PER_SESSION` | `--max-turns-per-session` | `0` (disabled) | `config.rs:372-375` | proactive rotation threshold (`pool.rs:1999-2022`) |
| `BUZZ_ACP_CONTEXT_MESSAGE_LIMIT` | `--context-message-limit` | `12`, range `0..=100` | `config.rs:366-369` | `0` skips context fetch entirely (`pool.rs:1746-1750`); otherwise the `limit` on thread/DM filters |
| `BUZZ_ACP_DEDUP` | `--dedup` | `queue` | `config.rs:344` | `requeue_batch_if_queue` — `Drop` discards the batch on every failure (`pool.rs:2973-2979`) and suppresses `recoverable_batch` for panic recovery (`lib.rs:2919-2922`) |
| `BUZZ_ACP_PERMISSION_MODE` | `--permission-mode` | **`bypass-permissions`** (`config.rs:435`) | `config.rs:432-438` | `ctx.permission_mode`; `Default` skips the request (`pool.rs:924-928`) |
| `BUZZ_ACP_MEMORY` / `BUZZ_ACP_NO_MEMORY` | `--memory` / `--no-memory` | memory on | `config.rs:395`, `:404` | `ctx.memory_enabled` — gates the NIP-AE core fetch (`pool.rs:1379`) |
| `BUZZ_ACP_MCP_COMMAND` | `--mcp-command` | `""` | `config.rs:261` | empty ⇒ `ctx.mcp_servers` is empty; otherwise the single server spec cloned into every `session/new` (`lib.rs:4142-4144`, `pool.rs:832`) |
| `BUZZ_ACP_SYSTEM_PROMPT` / `_FILE` | `--system-prompt(-file)` | none | `config.rs:279`, `:286` | `[System]` section (`pool.rs:1137-1157`) |
| `BUZZ_ACP_NO_BASE_PROMPT` / `BUZZ_ACP_BASE_PROMPT_FILE` | `--no-base-prompt` / `--base-prompt-file` | compiled-in `base_prompt.md` | `config.rs:409`, `:416` | `ctx.base_prompt`; `Box::leak`ed to `'static` (`lib.rs:1539-1544`) |
| `BUZZ_ACP_TEAM_INSTRUCTIONS` | `--team-instructions` | none | `config.rs:464` | `[Team Instructions]` section (`pool.rs:1180-1197`) |
| `BUZZ_ACP_INITIAL_MESSAGE` | `--initial-message` | none | `config.rs:321` | first prompt on brand-new channel sessions (`pool.rs:1580-1712`) |
| `BUZZ_ACP_MODEL` | `--model` | none | `config.rs:423` | seeds `OwnedAgent::desired_model` at every spawn/respawn (`lib.rs:1794`, `:3799`) |
| `BUZZ_ACP_AGENT_COMMAND` | `--agent-command` | `goose` | `config.rs:191`, `:250` | the spawned binary (via `PoolStartup`); also normalized into `ctx.harness_name` (`lib.rs:1561`) |
| `BUZZ_ACP_AGENT_ARGS` | `--agent-args` | see `config.rs:197` | `config.rs:197`, `:255` | child argv |
| `BUZZ_ACP_RELAY_OBSERVER` | `--relay-observer` | `false` | `config.rs:468` | whether an `ObserverHandle` exists; with none, `turn_started`/`turn_liveness`/`turn_completed` emit nothing (`pool.rs:3169-3171`, `:3291`) |
| `BUZZ_ACP_HEARTBEAT_INTERVAL` / `_PROMPT` / `_PROMPT_FILE` | `--heartbeat-*` | `0` = off | `config.rs:297`, `:308`, `:316` | `ctx.heartbeat_prompt`; heartbeat turns take `control_rx = None` (`pool.rs:1827-1836`) |
| `BUZZ_RELAY_URL` | `--relay-url` | `ws://localhost:3000` | `config.rs:240` | `ctx.relay_url` in observer payloads (`pool.rs:918`) and the MCP server env |
| `BUZZ_PRIVATE_KEY` | `--private-key` | none | `config.rs:243` | `ctx.agent_keys`; bech32-encoded into the MCP server env (`lib.rs:4160-4170`) |
| `BUZZ_AUTH_TAG` | — | none | read at `lib.rs:125`, `:1338`, `:4173` | forwarded into the MCP server env when non-empty |
| `BUZZ_ACP_PRIVATE_KEY` | — | — | alias mapped to `BUZZ_PRIVATE_KEY` (`config.rs:717`) | legacy compatibility |

Implicit, non-env inputs: `ctx.cwd` from `std::env::current_dir()` with a `/` fallback (`lib.rs:1547-1550`), and `ctx.agent_owner_pubkey` derived from the resolved owner (`lib.rs:1557-1559`) — when it is `None`, both NIP-AE core injection and NIP-AM metric publishing are skipped silently (`pool.rs:1380`, `:3336-3339`).

#### Hard-coded values with no configuration path

| Constant | Value | Site | Gates |
|---|---|---|---|
| `INITIAL_RETRY_DELAY` | 5 s | `pool_lifecycle.rs:11` | first lazy-wake retry delay |
| `MAX_RETRY_DELAY` | 300 s | `pool_lifecycle.rs:12` | wake retry ceiling |
| `RECENT_ACTIVITY_WINDOW` | 60 s | `pool.rs:45` | hard-cap `recently_active` classification (requeue vs dead-letter eligibility) |
| `CONTEXT_FETCH_TIMEOUT` | 3000 ms | `pool.rs:780` | every context/profile/channel-info fetch attempt |
| `CONTEXT_FETCH_RETRY_DELAY` | 500 ms | `pool.rs:783` | gap before the single retry |
| `MODEL_SWITCH_TIMEOUT` | 5 s | `pool.rs:786` | model-switch request |
| `CONTROL_CANCEL_GRACE` | 5 s | `pool.rs:793` | post-cancel drain deadline |
| `PERMISSION_MODE_TIMEOUT` | 5 s | `pool.rs:796` | permission-mode request |
| `CORE_FETCH_TIMEOUT` | 3 s | `pool.rs:1386` | NIP-AE core fetch |
| `CANVAS_FETCH_TIMEOUT` | 3 s | `pool.rs:2316` | canvas fetch |
| `METRIC_TIMEOUT` | 3 s | `pool.rs:3415` | NIP-AM publish |
| `REACTION_TIMEOUT` | 500 ms | `pool.rs:3437` | reaction add |
| `REACTION_CONCURRENCY` | 10 | `pool.rs:3618` | reaction fan-out width |
| reaction-remove query/delete | 1000 ms each | `pool.rs:3551`, `:3609` | inline, not named |
| failure-notice timeout | 5 s | `pool.rs:3528` | inline, not named |
| init timeout | 60 s per agent | `lib.rs:3759` | spawn+initialize per slot |
| shutdown grace | 30 s | `lib.rs:2608` | in-flight prompt drain |
| wake drain grace | 30 s | `lib.rs:2591` | lazy-wake task drain |
| circuit breaker | 3 crashes / 60 s, 300 s cooldown, 1 s→30 s backoff | `lib.rs:1008-1016` | per-slot respawn rate |

The 5 s / 300 s wake backoff is the module's most consequential un-tunable pair: with `--lazy-pool` and a persistently failing agent, first-response latency after a quiet period is capped at 5 minutes with no way to shorten or lengthen it.

#### Missing from `.env.example`

`.env.example` mentions `BUZZ_ACP_*` on 25 lines, spanning `.env.example:134-226`. Absent entirely, verified by grep:

`BUZZ_ACP_IDLE_TIMEOUT`, `BUZZ_ACP_MAX_TURN_DURATION`, `BUZZ_ACP_TURN_LIVENESS_SECS`, `BUZZ_ACP_LAZY_POOL`, `BUZZ_ACP_RELAY_OBSERVER`, `BUZZ_ACP_PERMISSION_MODE`, `BUZZ_ACP_NO_MEMORY`, `BUZZ_ACP_MULTIPLE_EVENT_HANDLING`.

Worse than absent: `.env.example:152` documents `BUZZ_ACP_TURN_TIMEOUT=320` — the **deprecated, `hide = true` alias** (`config.rs:274-275`) — while the current name `BUZZ_ACP_IDLE_TIMEOUT` appears nowhere. An operator following `.env.example` configures a hidden deprecated flag, and the value shown (320) does not match either code default (900 idle / 7200 hard cap).

`crates/buzz-acp/README.md:117-118` does document `--agents` and `--lazy-pool` correctly, including the lazy-wake retry behaviour, so the drift is specific to `.env.example`.

#### Parsed-and-never-read

`.env.example:221` documents `BUZZ_ACP_EVENT_BUFFER=256`, which is read by `relay.rs:36` and never reaches the pool — the only `BUZZ_ACP_*` var in `.env.example` that is not a clap arg. No pool knob is parsed and dropped: every `PromptContext` field is consumed somewhere in `pool.rs`. Two adjacent observations:

- `PromptContext::heartbeat_prompt` (`pool.rs:501`) is stored on the shared context but the heartbeat prompt text is composed in `lib.rs::dispatch_heartbeat` (`lib.rs:3537-3580`) and passed in as `prompt_text`; `pool.rs` never reads the field. It is a carried-but-unused field from this module's perspective.
- `PoolLifecycle::failed_error()` (`pool_lifecycle.rs:84-89`) has exactly one caller, a `debug_assert_eq!` (`lib.rs:2565`), so it is dead in release builds.

#### Validation performed at config time (relevant to pool behaviour)

`--agents` is range-checked `1..=32` by clap (`config.rs:293`). `context_message_limit` is range-checked `0..=100` (`config.rs:368`). `idle_timeout_secs` must be strictly less than `max_turn_duration_secs`, else `Config::from_cli` errors with the reason that otherwise the idle timeout would be "a dead letter" (`config.rs:891-899`). No validation exists for `turn_liveness_secs` relative to the timeouts, and none for `max_turns_per_session`.


## Module: buzz-acp — work queue, filtering & usage accounting (`crates/buzz-acp/src`)
### Aspect: Configuration

None of the three files reads an environment variable or a file. Every knob is parsed in `config.rs` (clap + `env`) or `config::load_rules` (TOML) and passed in as a value.

#### Knobs that reach `EventQueue`

| Knob | Env / flag | Default | Parse site | Effect on the queue |
|---|---|---|---|---|
| `dedup_mode` | `--dedup` / `BUZZ_ACP_DEDUP` | `queue` | `config.rs:344`, enum `config.rs:57-61` | `EventQueue::new(dedup_mode)` (`lib.rs:1505`). `Drop` discards events for in-flight channels at `queue.rs:231-241`; also gates `recoverable_batch` (`lib.rs:2196-2199`) and `requeue_batch_if_queue` (`pool.rs:2971-2977`) |
| `max_turn_duration_secs` | `--max-turn-duration` / `BUZZ_ACP_MAX_TURN_DURATION` | 7200 (`config.rs:30`) | `config.rs:270`, validated `config.rs:876-899`, ceiling `MAX_TURN_DURATION_CEILING_SECS = 604_800` (`config.rs:36`) | `with_in_flight_deadline(max_turn + 100)` (`lib.rs:1506`, `queue.rs:197-201`); also the argument to `extend_in_flight_deadline` on a successful steer (`lib.rs:2509`) |
| `multiple_event_handling` | `--multiple-event-handling` / `BUZZ_ACP_MULTIPLE_EVENT_HANDLING` | `steer` (`config.rs:356`) | `config.rs:352-359`, enum `config.rs:65-84` | selects the mid-turn signal (`mode_gate_signal`, `lib.rs:2224` → `mode_gate_signal` `lib.rs:2741`) which becomes the `CancelReason` stamped by `requeue_cancelled_batch` (`pool.rs:2985-2995`) and consumed by `requeue_as_cancelled` (`queue.rs:542-548`) |

Note the coupling between the last two: `MultipleEventHandling::Steer` is documented as requiring `DedupMode::Queue` (`config.rs:74`, `:79`, `:83`). With `--dedup drop`, a steer still cancels the turn but `requeue_batch_if_queue` returns `None` and the batch is lost (`pool.rs:2973-2976`).

#### Knobs that reach `filter.rs`

| Knob | Env / flag | Default | Parse site | Effect |
|---|---|---|---|---|
| `subscribe_mode` | `--subscribe` / `BUZZ_ACP_SUBSCRIBE` | `mentions` | `config.rs:326-331` | picks which `SubscriptionRule` set is built (`lib.rs:1439-1474`) |
| `kinds_override` | `--kinds` / `BUZZ_ACP_KINDS` (comma-delimited) | `None` | `config.rs:332-333` | `Mentions` → replaces `[9, 46010, 40007]`; `All` → `unwrap_or_default()`, i.e. **empty vec** when unset (`lib.rs:1447-1451`, `:1462`) |
| `no_mention_filter` | `--no-mention-filter` / `BUZZ_ACP_NO_MENTION_FILTER` | `false` | `config.rs:338-339` | `require_mention = !no_mention_filter` (`lib.rs:1452`) |
| `channels_override` | `--channels` / `BUZZ_ACP_CHANNELS` | `None` | `config.rs:335-336` | not read by `filter.rs`; consumed by `resolve_channel_filters` / `resolve_dynamic_channel_filter` (`config.rs:1134`, `:1249-1258`) and ignored entirely in `Config` mode |
| `config_path` | `--config` / `BUZZ_ACP_CONFIG` | `./buzz-acp.toml` | `config.rs:341-342` | `load_rules(&config.config_path)` in `Config` mode (`lib.rs:1472`) |

`buzz-acp.toml` rule keys (deserialized into `SubscriptionRule`, `filter.rs:83-114`):

| Key | Type | Default | Validation |
|---|---|---|---|
| `name` | string | required | non-empty, unique across rules (`config.rs:1084-1093`) |
| `channels` | `"all"` or `[uuid, …]` | required | `"all"` must be exact lowercase (`config.rs:1118-1124`) |
| `kinds` | `[u32]` | `[]` = wildcard | none |
| `require_mention` | bool | `false` | none |
| `filter` | evalexpr string | `None` | ≤4096 bytes, compiled at load (`config.rs:1097-1116`) |
| `prompt_tag` | string | falls back to `name` | none (`filter.rs:451`) |

Global rule-file limit: 100 rules (`config.rs:1067-1072`); zero rules emits a `WARN` and the agent receives nothing (`config.rs:1075-1080`).

#### Knobs that reach the prompt formatter

| Knob | Env / flag | Default | Effect on `format_prompt` |
|---|---|---|---|
| `context_message_limit` | `--context-message-limit` / `BUZZ_ACP_CONTEXT_MESSAGE_LIMIT` | 12, range 0..=100 (`config.rs:364-368`) | `0` disables context fetching entirely, so `conversation_context` is `None` and no `[Thread Context]`/`[Conversation Context]` section is emitted (`pool.rs:1746`, `:2493-2502`) |
| `base_prompt_content` | `--base-prompt` family (`config.rs`) | built-in `base_prompt.md` | `FormatPromptArgs::base_prompt`; rendered as `[Base]` only for legacy agents (`queue.rs:1435-1437`) |
| `system_prompt` / file | `--system-prompt` / `BUZZ_ACP_SYSTEM_PROMPT`, `BUZZ_ACP_SYSTEM_PROMPT_FILE` | none | `[System]` section, legacy agents only (`queue.rs:1438-1440`) |
| protocol version (not a knob — negotiated) | — | — | `has_system_prompt_support` gates `[Base]`/`[System]`/`[Team Instructions]`/core/canvas (`queue.rs:1432-1462`) |

#### Hard-coded values with no configuration path

These are compile-time constants that operators cannot change:

| Constant | Value | Line |
|---|---|---|
| `MAX_PENDING_PER_CHANNEL` | 500 | `queue.rs:24` |
| `MAX_BATCH_EVENTS` | 50 | `queue.rs:27` |
| `MAX_RETRIES` | 10 | `queue.rs:30` |
| `BASE_RETRY_DELAY_SECS` | 5 | `queue.rs:33` |
| `MAX_RETRY_DELAY_SECS` | 300 | `queue.rs:36` |
| `IN_FLIGHT_DEADLINE_BUFFER_SECS` | 100 | `queue.rs:39` |
| `DEFAULT_IN_FLIGHT_DEADLINE_SECS` | 7300 | `queue.rs:42` |
| `MAX_PROMPT_LABEL_LEN` | 64 | `queue.rs:1023` |
| `MAX_EXPR_LEN` | 4096 | `filter.rs:162` |
| `EVAL_TIMEOUT` | 100 ms | `filter.rs:165` |
| `MAX_CONCURRENT_FILTER_EVALS` | 4 | `filter.rs:173` |
| `MAX_CONSECUTIVE_TIMEOUTS` | 5 | `filter.rs:341` |

`DEFAULT_IN_FLIGHT_DEADLINE_SECS = 7300` duplicates `DEFAULT_MAX_TURN_DURATION_SECS (7200) + IN_FLIGHT_DEADLINE_BUFFER_SECS (100)` as a literal rather than computing it; `config.rs:30` and `queue.rs:42` must be edited together. The test `default_in_flight_deadline_exceeds_default_max_turn_duration` (`queue.rs:4530`) is the only guard.

`MAX_EXPR_LEN = 4096` (`filter.rs:162`) is duplicated as a bare `4096` literal in `load_rules` (`config.rs:1098`) — the two can drift.

#### Parsed and never read

| Item | Evidence |
|---|---|
| `UsageUpdatePayload::used` | `#[serde(default)] #[allow(dead_code)]`, `usage.rs:78-81`; no reader anywhere — `pool.rs` never populates a context-utilization field |
| `UsageUpdatePayload::context_limit` | same, `usage.rs:82-85` |
| `MatchedRule::rule_index` | `#[cfg_attr(not(test), allow(dead_code))]`, `filter.rs:152-153`; `lib.rs:2175` and `setup_mode.rs:449` both discard it |

Both usage fields are tested for wire-compatibility (`notification_deserializes_without_used_and_context_limit`, `usage.rs:691`; `buzz_agent_payload_no_context_fields_processes_correctly`, `usage.rs:797`) but never surface in a published metric — `TokenCounts.total_tokens`, `cache_read_tokens`, `cache_write_tokens` are hard-`None` at `pool.rs:3339-3342` and `:3358-3360`.

#### Missing from `.env.example`

`.env.example` documents 5 of the queue/filter-relevant vars — `BUZZ_ACP_SUBSCRIBE` (`.env.example:186`), `BUZZ_ACP_KINDS` (`:189`), `BUZZ_ACP_CHANNELS` (`:192`), `BUZZ_ACP_NO_MENTION_FILTER` (`:195`), `BUZZ_ACP_CONFIG` (`:198`), `BUZZ_ACP_DEDUP` (`:202`), `BUZZ_ACP_CONTEXT_MESSAGE_LIMIT` (`:209`) — and omits:

| Missing var | Reaches | Why it matters |
|---|---|---|
| `BUZZ_ACP_MAX_TURN_DURATION` | `with_in_flight_deadline` (`lib.rs:1506`) | directly sets the in-flight backstop; the only knob that can wedge or prematurely release a channel |
| `BUZZ_ACP_MULTIPLE_EVENT_HANDLING` | cancel/steer path (`lib.rs:2224` → `mode_gate_signal` `lib.rs:2741`) | selects whether mid-turn mentions cancel the turn at all, and which framing the merged prompt uses |
| `BUZZ_ACP_IDLE_TIMEOUT` | validated against `max_turn_duration` (`config.rs:894-899`) | the pair is cross-validated at startup; documenting one without the other is misleading |

Worse, the one timeout variable `.env.example` **does** document — `BUZZ_ACP_TURN_TIMEOUT=320` (`.env.example:152`) — is marked `hide = true` in clap and warns "deprecated and ignored" at runtime (`config.rs:274`, `:853-862`). So the only timeout knob visible to an operator reading `.env.example` is the deprecated one, while the two live knobs that actually set the queue's in-flight deadline are undocumented.


## Module: buzz-acp — ACP protocol, config & setup mode (`crates/buzz-acp/src`)
### Aspect: Configuration

Configuration is CLI-first: 43 `env = "…"` attributes in `config.rs`, every one paired with a long flag (`config.rs:1-4`). A TOML file supplies subscription rules only. Precedence is clap's: explicit flag > env var > `default_value`.

#### Authoritative env-var table — clap-parsed (`CliArgs`, `config.rs:234-474`)

| Env var | Flag | Default | Validation / effect |
|---|---|---|---|
| `BUZZ_RELAY_URL` | `--relay-url` | `ws://localhost:3000` (`:240`) | Not validated as a URL here; only `codex_network_env` parses it (`:654`), and a parse failure there just skips Codex injection |
| `BUZZ_PRIVATE_KEY` | `--private-key` | **required** (`:243`) | `Keys::parse` → `ConfigError::KeyParse` (`:741`); raw string then zeroed + cleared (`:742-748`) |
| `BUZZ_ACP_AGENT_OWNER` | `--agent-owner` | — (`:247`) | Trimmed + lowercased (`:1003`); **no hex/length validation**, unlike the allowlist |
| `BUZZ_ACP_AGENT_COMMAND` | `--agent-command` | `goose` (`:250`) | Trim-empty → error (`:808-812`); resolved via inherited `PATH` |
| `BUZZ_ACP_AGENT_ARGS` | `--agent-args` | `acp`, comma-delimited (`:253-258`) | `normalize_agent_args` (`:679-706`) |
| `BUZZ_ACP_MCP_COMMAND` | `--mcp-command` | `""` (`:261`) | Empty → `build_mcp_servers` returns `[]` (`lib.rs:4142-4144`), so a stock run gives the agent **zero MCP servers** |
| `BUZZ_ACP_IDLE_TIMEOUT` | `--idle-timeout` | unset → `900` (`:266`, `:27`) | `0` → warn + clamp to `1` (`:868-873`); must be `<` max turn duration (`:894-899`) |
| `BUZZ_ACP_MAX_TURN_DURATION` | `--max-turn-duration` | `7200` (`:270`, `:31`) | `0` → warn + clamp to `60`; `>604_800` → **error** (`:876-890`) |
| `BUZZ_ACP_TURN_TIMEOUT` | `--turn-timeout` | — (`:274`) | **Hidden (`hide = true`) and deprecated.** Alias for idle timeout; loses to `--idle-timeout`, warns either way (`:849-874`) |
| `BUZZ_ACP_SYSTEM_PROMPT` | `--system-prompt` | — (`:277-281`) | `conflicts_with = "system_prompt_file"`; also wins in code (`:750-756`) |
| `BUZZ_ACP_SYSTEM_PROMPT_FILE` | `--system-prompt-file` | — (`:284-288`) | Read with `fs::read_to_string` → `ConfigError::Io`; **no size cap** (unlike base prompt) |
| `BUZZ_ACP_AGENTS` | `--agents` | `1`, range `1..=32` (`:292-293`) | Enforced by clap `value_parser` |
| `BUZZ_ACP_HEARTBEAT_INTERVAL` | `--heartbeat-interval` | `0` = disabled (`:297`) | `>0 && <10` → **error** (`:758-762`); `>86400` → warn + clamp (`:826-834`) |
| `BUZZ_ACP_TURN_LIVENESS_SECS` | `--turn-liveness-secs` | `10` (`:302`) | `>0 && <5` → **error** (`:764-768`); `>86400` → warn + clamp (`:836-845`) |
| `BUZZ_ACP_HEARTBEAT_PROMPT` | `--heartbeat-prompt` | — (`:306-310`) | Conflicts with the file variant; wins in code (`:770-776`) |
| `BUZZ_ACP_HEARTBEAT_PROMPT_FILE` | `--heartbeat-prompt-file` | — (`:314-318`) | No size cap |
| `BUZZ_ACP_INITIAL_MESSAGE` | `--initial-message` | — (`:321`) | Passed through unvalidated (`:980`) |
| `BUZZ_ACP_SUBSCRIBE` | `--subscribe` | `mentions` (`:324-329`) | `mentions` / `all` / `config` |
| `BUZZ_ACP_KINDS` | `--kinds` | — comma (`:332`) | Ignored in config mode (warns, `:794-796`). Absent + `--subscribe all` → `kinds: None` wildcard (`:1180`, `:1272`) |
| `BUZZ_ACP_CHANNELS` | `--channels` | — comma (`:335`) | Non-UUID entries **warn and are dropped**, never error (`:816-824`); ignored in config mode (warns, `:797-799`) |
| `BUZZ_ACP_NO_MENTION_FILTER` | `--no-mention-filter` | `false` (`:338`) | Inverted into `require_mention` (`:1168`, `:1269`); ignored in config mode (warns, `:800-802`) |
| `BUZZ_ACP_CONFIG` | `--config` | `./buzz-acp.toml` (`:341`) | Read only in `SubscribeMode::Config` via `load_rules` (`:1060`) |
| `BUZZ_ACP_DEDUP` | `--dedup` | `queue` (`:344`) | `drop` + any cancel mode → **error** (`:579-597`) |
| `BUZZ_ACP_MULTIPLE_EVENT_HANDLING` | `--multiple-event-handling` | `steer` (`:353-358`) | `queue` / `steer` / `interrupt` / `owner-interrupt` |
| `BUZZ_ACP_NO_IGNORE_SELF` | `--no-ignore-self` | `false` (`:361`) | Inverted → `ignore_self` (`:977`) |
| `BUZZ_ACP_CONTEXT_MESSAGE_LIMIT` | `--context-message-limit` | `12`, range `0..=100` (`:366-367`) | Enforced by clap; `0` disables context fetching (`:364-365`) |
| `BUZZ_ACP_MAX_TURNS_PER_SESSION` | `--max-turns-per-session` | `0` = disabled (`:372-373`) | `value_parser!(u32)` with **no range** — any `u32` accepted |
| `BUZZ_ACP_NO_PRESENCE` | `--no-presence` | `false` (`:377`) | Inverted → `presence_enabled` (`:983`) |
| `BUZZ_ACP_NO_TYPING` | `--no-typing` | `false` (`:381`) | Inverted → `typing_enabled` (`:984`) |
| `BUZZ_ACP_MEMORY` | `--memory` | `true` (`:393-398`) | `conflicts_with = "no_memory"`; combined as `memory && !no_memory` (`:985`) |
| `BUZZ_ACP_NO_MEMORY` | `--no-memory` | `false` (`:404`) | The documented opt-out |
| `BUZZ_ACP_NO_BASE_PROMPT` | `--no-base-prompt` | `false` (`:409`) | Suppresses the `[Base]` section entirely (`lib.rs:1539-1540`) |
| `BUZZ_ACP_BASE_PROMPT_FILE` | `--base-prompt-file` | — (`:414-418`) | `conflicts_with = "no_base_prompt"`; content read at startup, **`>1_048_576` bytes → error** (`:778-791`) |
| `BUZZ_ACP_MODEL` | `--model` | — (`:423`) | Applied after every `session_new_full`; resolved against the catalog (`acp.rs:1876`) |
| `BUZZ_ACP_PERMISSION_MODE` | `--permission-mode` | **`bypass-permissions`** (`:435`) | Five values, each with a camelCase alias (`:122-139`) |
| `BUZZ_ACP_RESPOND_TO` | `--respond-to` | `owner-only` (`:445`) | Checked against `allowed_respond_to` when that is non-empty (`:926-933`) |
| `BUZZ_ACP_RESPOND_TO_ALLOWLIST` | `--respond-to-allowlist` | — comma (`:452`) | Required non-empty in `allowlist` mode (`:903-907`); 64-hex per entry, trimmed + lowercased + deduped (`:558-572`, applied `:908`); **warn-and-discard** in any other mode (`:909-915`) |
| `BUZZ_ACP_ALLOWED_RESPOND_TO` | `--allowed-respond-to` | — comma (`:460`) | Each entry must parse as a `RespondTo` (`:921-927`); non-empty list not containing the active mode → **error**. Afterwards **display-only** |
| `BUZZ_ACP_TEAM_INSTRUCTIONS` | `--team-instructions` | — (`:464`) | Trimmed; empty → `None` (`:967-972`) |
| `BUZZ_ACP_RELAY_OBSERVER` | `--relay-observer` | `false` (`:468`) | Gates `ObserverHandle` creation (`lib.rs:1298-1300`) |
| `BUZZ_ACP_LAZY_POOL` | `--lazy-pool` | `false` (`:472`) | Defers subprocess startup until accepted work arrives |

`AuthAgentArgs` re-declares two of these for the auxiliary subcommands, with the same env names and defaults: `BUZZ_ACP_AGENT_COMMAND` → `goose` (`config.rs:191`) and `BUZZ_ACP_AGENT_ARGS` → `acp` (`config.rs:194-199`). `AuthenticateArgs::method_id` is the only required flag with **no** env fallback (`config.rs:230-231`).

#### Env vars read outside clap

| Env var | Read site | Effect |
|---|---|---|
| `BUZZ_ACP_SETUP_PAYLOAD` | `setup_mode.rs:83` (const), `setup_mode.rs:214` | Non-empty valid JSON diverts startup to the nudge listener; the agent pool never starts (`lib.rs:1290-1295`). Malformed → startup error |
| `BUZZ_AUTH_TAG` | `lib.rs:125`, `lib.rs:1338`, `lib.rs:4173`, `setup_mode.rs:319` | NIP-OA owner attestation; parsed by `buzz_sdk::nip_oa::parse_auth_tag`, silently dropped when empty or unparseable. Also forwarded into `McpServer.env` |
| `CODEX_CONFIG` | `acp.rs:437` | The parent's own value; deep-merged with **parent-wins** precedence into the generated config (`acp.rs:311-327`) |
| `BUZZ_MANAGED_AGENT_START_NONCE` | `lib.rs:1503` | `unwrap_or_default()` — no validation |
| `BUZZ_ACP_EVENT_BUFFER` | `relay.rs:36` | Relay event-channel capacity. Documented at `relay.rs:28`. Outside this module group but part of the crate's env surface |
| *(any key in `persona_env_vars`)* | `acp.rs:455` | Injected into the child **only if not already set** in the parent |

#### Legacy aliases

`propagate_legacy_env_vars` (`config.rs:715-726`) copies, only when the canonical name is unset (`config.rs:720`):

| Legacy | Canonical | Status |
|---|---|---|
| `BUZZ_ACP_PRIVATE_KEY` | `BUZZ_PRIVATE_KEY` | Live — feeds `CliArgs::private_key` (`config.rs:243`) |
| `BUZZ_ACP_API_TOKEN` | `BUZZ_API_TOKEN` | **Parsed and never read.** `BUZZ_API_TOKEN` appears nowhere else in the crate's source; the only other repo mention is the README table row (`crates/buzz-acp/README.md:107`) |

`BUZZ_ACP_TURN_TIMEOUT` is the third legacy name, handled inside clap as a hidden flag rather than by this function (`config.rs:274`).

Ordering constraint: this function uses `std::env::set_var` and must run before the tokio runtime starts, because Rust 2024 requires a single-threaded process for that call (`config.rs:708-714`). It is invoked from the sync wrapper at `lib.rs:1234`; `Config::from_cli` explicitly does not call it (`config.rs:730-732`).

#### Parsed-and-never-read / effectively-inert configuration

| Item | Evidence |
|---|---|
| `BUZZ_API_TOKEN` | written at `config.rs:718`, no read site in the crate |
| `Config::allowed_respond_to` | validated `config.rs:919-937`, then referenced only by `summary()` (`config.rs:1019-1025`, emitted `:1049`) and two test-fixture initialisers (`lib.rs:4979`, `lib.rs:5145`) |
| `Config::persona_env_vars` as a persona mechanism | only ever receives the one generated `CODEX_CONFIG` pair (`config.rs:945-955`); `buzz-persona` (declared `Cargo.toml:22`) has zero references under `crates/buzz-acp/src` |
| `--respond-to-allowlist` outside `allowlist` mode | warn-and-discard (`config.rs:909-915`) |
| `--kinds` / `--channels` / `--no-mention-filter` in config mode | warn-and-ignore (`config.rs:793-804`) |
| `handle_setup_membership`'s `_initial_channel_ids` | accepted, unused (`setup_mode.rs:568`) |

#### TOML configuration file

Path from `BUZZ_ACP_CONFIG` (default `./buzz-acp.toml`, `config.rs:341`), loaded only in `SubscribeMode::Config`. Schema is a single `rules` array of `SubscriptionRule` (`config.rs:1053-1057`, `#[serde(default)]` so an empty file parses).

Constraints enforced by `load_rules` (`config.rs:1060-1132`): ≤100 rules; non-empty unique `name`; `filter` expression ≤4096 bytes and compiled by `evalexpr::build_operator_tree` at load time; `channels` must be the literal `"all"` or a list. Zero rules is a **warning only** — "agent will receive no events in Config mode" (`config.rs:1075-1080`).

#### Documentation drift

| Claim | Source | Actual |
|---|---|---|
| `BUZZ_ACP_IDLE_TIMEOUT` default `620` | `crates/buzz-acp/README.md:105` | `DEFAULT_IDLE_TIMEOUT_SECS = 900` (`config.rs:27`) |
| `BUZZ_API_TOKEN` "required if relay enforces token auth" | `crates/buzz-acp/README.md:107` | Never read by the crate |
| `BUZZ_ACP_TURN_TIMEOUT=320` presented as the timeout knob | `.env.example:152` | Hidden + deprecated (`config.rs:274`); `.env.example` never mentions `BUZZ_ACP_IDLE_TIMEOUT` or `BUZZ_ACP_MAX_TURN_DURATION` |
| `.env.example` coverage | `.env.example:121-226` | Documents ~20 of the 43 env vars. Missing entirely: `BUZZ_ACP_AGENT_OWNER`, `BUZZ_ACP_IDLE_TIMEOUT`, `BUZZ_ACP_MAX_TURN_DURATION`, `BUZZ_ACP_TURN_LIVENESS_SECS`, `BUZZ_ACP_MULTIPLE_EVENT_HANDLING`, `BUZZ_ACP_MEMORY` / `BUZZ_ACP_NO_MEMORY`, `BUZZ_ACP_NO_BASE_PROMPT`, `BUZZ_ACP_BASE_PROMPT_FILE`, `BUZZ_ACP_PERMISSION_MODE`, `BUZZ_ACP_RESPOND_TO`, `BUZZ_ACP_RESPOND_TO_ALLOWLIST`, `BUZZ_ACP_ALLOWED_RESPOND_TO`, `BUZZ_ACP_TEAM_INSTRUCTIONS`, `BUZZ_ACP_RELAY_OBSERVER`, `BUZZ_ACP_LAZY_POOL`, `BUZZ_ACP_SETUP_PAYLOAD`, `BUZZ_AUTH_TAG`, `BUZZ_MANAGED_AGENT_START_NONCE` |
| `BUZZ_ACP_EVENT_BUFFER=256` | `.env.example:221` | Real, but read in `relay.rs:36`, not in this group — it is the one env var `.env.example` documents that no `config.rs` flag covers |

The two security-relevant defaults — `--permission-mode bypass-permissions` (`config.rs:435`) and the auto-approve permission handler (`acp.rs:1703-1718`) — are absent from both `.env.example` and the README's env table.


## Module: buzz-agent — core loop, ACP wire types & handoff (`crates/buzz-agent/src`)
### Aspect: Configuration
#### How this group gets configured
All environment parsing lives in `config.rs` (a sibling agent's scope); this group receives an immutable `Config` built once at startup (`lib.rs:160`) and stored in `App.cfg` (`lib.rs:53`). The only direct env read in the group is `DATABRICKS_HOST` inside the auth subcommand (`lib.rs:135-136`) — `grep -c 'std::env::var'` returns 1 for `lib.rs` and 0 for the other five files. There are no CLI flags: `run()` recognizes exactly one argument, `auth` (`lib.rs:111`), and ignores everything else (`lib.rs:117-120`).

#### Config fields consumed in this group
| Field | Env var | Default (site) | Read site(s) in this group |
|---|---|---|---|
| `max_line_bytes` | `BUZZ_AGENT_MAX_LINE_BYTES` | 4 MiB (`config.rs:813`) | `lib.rs:162` → `read_loop`/`read_bounded_line` (`wire.rs:189`) |
| `max_sessions` | `BUZZ_AGENT_MAX_SESSIONS` | `usize::MAX` (`config.rs:812`) | `lib.rs:349`, `lib.rs:401` |
| `hints_enabled` | `BUZZ_AGENT_NO_HINTS` (inverted: `0` → enabled) | enabled (`config.rs:825`) | `lib.rs:356` |
| `system_prompt` | `BUZZ_AGENT_SYSTEM_PROMPT` / `_FILE` | built-in default (`config.rs:658`, resolved `config.rs:786-792`) | `lib.rs:365` (fallback only) |
| `model` | `BUZZ_AGENT_MODEL` > provider-specific var | required (`config.rs:749`, `config.rs:757-785`) | `lib.rs:470`, `lib.rs:672` |
| `provider` | `BUZZ_AGENT_PROVIDER` | required, no inference (`config.rs:982`, `config.rs:1005-1008`) | `lib.rs:445` |
| `max_rounds` | `BUZZ_AGENT_MAX_ROUNDS` | `0` = unlimited (`config.rs:801`) | `agent.rs:88` |
| `max_history_bytes` | `BUZZ_AGENT_MAX_HISTORY_BYTES` | **16 MiB** (`config.rs:814`) | `agent.rs:111`, `handoff.rs:125` |
| `max_parallel_tools` | `BUZZ_AGENT_MAX_PARALLEL_TOOLS` | 8 (`config.rs:821`) | `agent.rs:371` |
| `tool_timeout` | `BUZZ_AGENT_TOOL_TIMEOUT_SECS` | 660 s (`config.rs:804`) | `agent.rs:383`, `agent.rs:509` |
| `max_tool_result_text_bytes` | `BUZZ_AGENT_MAX_TOOL_RESULT_TEXT_BYTES` | 50 KiB (`config.rs:815-818`) | `agent.rs:387` |
| `stop_max_rejections` | `BUZZ_AGENT_STOP_MAX_REJECTIONS` | 3 (`config.rs:823`) | `agent.rs:220` |
| `hook_timeout` | `BUZZ_AGENT_HOOK_TIMEOUT_MS` | 2500 ms (`config.rs:822`) | `agent.rs:228`, `handoff.rs:75` |
| `hook_servers` | `MCP_HOOK_SERVERS` | `None` = hooks off (`config.rs:824`) | `agent.rs:229`, `handoff.rs:76` |
| `max_context_tokens` | `BUZZ_AGENT_MAX_CONTEXT_TOKENS` | 200 000 (`config.rs:819`) | `handoff.rs:113`, `handoff.rs:123`, `handoff.rs:196` |
| `max_output_tokens` | `BUZZ_AGENT_MAX_OUTPUT_TOKENS` | 32 768 (`config.rs:802`) | `handoff.rs:113`, `handoff.rs:124` |
| `max_handoffs` | `BUZZ_AGENT_MAX_HANDOFFS` | 10 (`config.rs:820`) | `handoff.rs:34-38` |
| `llm_timeout`, `mcp_*`, `thinking_effort`, `api_key`, `base_url`, `openai_api`, `anthropic_api_version` | various | — | not read here; passed through `cfg` to `llm`/`mcp` |

`DATABRICKS_HOST` is read directly at `lib.rs:135-136` (auth subcommand only) and is documented in the crate README (`README.md:145`).

#### Compile-time constants that behave like configuration
| Constant | Value | Site | Tunable? |
|---|---|---|---|
| `PROTOCOL_VERSION` | 2 | `config.rs:3`, used `lib.rs:284` | no |
| `MAX_PROMPT_BYTES` | 1 MiB | `config.rs:638`, used `agent.rs:69` | no |
| `MAX_SYSTEM_PROMPT_BYTES` | 512 KiB | `config.rs:639`, used `lib.rs:375` | no |
| `MAX_TOOL_RESULT_BYTES` | 8 MiB | `config.rs:643`, used `agent.rs:386` | no |
| `MAX_TOOL_CALLS_PER_TURN` | 64 | `config.rs:650`, used `agent.rs:242` | no |
| `HANDOFF_MAX_OUTPUT_TOKENS` | 8192 | `config.rs:652`, used `handoff.rs:51`, `handoff.rs:197` | no |
| `HANDOFF_ORIGINAL_TASK_MAX_BYTES` | 16 KiB | `config.rs:654`, used `handoff.rs:175` | no |
| `HANDOFF_MAX_TOOL_NAMES` | 20 | `config.rs:656`, used `handoff.rs:182` | no |
| `IMAGE_CONTEXT_TOKEN_EQUIV` | 16 KiB | `types.rs:14` | no |
| `CONSERVATIVE_BYTES_PER_TOKEN` | 1 | `handoff.rs:326` | no |
| keepalive interval | 30 s | `agent.rs:129` (literal) | no |
| post-cancel drain bound | 5 s | `agent.rs:455` (literal) | no |
| wire channel depth | 64 | `lib.rs:164` (literal) | no |
| session/run/steer token width | 8 bytes | `lib.rs:822` (literal) | no |
| `usage_update.contextLimit` | `0` | `lib.rs:742` (literal) | no |

The keepalive interval is the most consequential hardcoded value: it exists to keep the harness idle clock alive, and the harness side *is* configurable (`.env.example:169-170` recommends raising the ACP idle timeout to 60 s), so the two can be tuned out of alignment with no way to adjust this side.

#### Documentation status of every env var this group depends on
`.env.example` documents zero of them: `grep -c 'BUZZ_AGENT' .env.example` → 0 (the file covers relay and `BUZZ_ACP_*` harness config only, `.env.example:114-170`). The authoritative reference is the crate README table (`crates/buzz-agent/README.md:130-158`).

| Env var | crate README | `docs/MCP_DRIVEN_HOOKS.md` | root `.env.example` | AGENTS.md |
|---|---|---|---|---|
| `BUZZ_AGENT_PROVIDER` | yes (`:132`) | — | no | no |
| `BUZZ_AGENT_MAX_ROUNDS` | yes (`:149`) | — | no | no |
| `BUZZ_AGENT_MAX_OUTPUT_TOKENS` | yes (`:150`) | — | no | no |
| `BUZZ_AGENT_MAX_CONTEXT_TOKENS` | yes (`:151`) | — | no | no |
| `BUZZ_AGENT_MAX_HANDOFFS` | yes (`:152`) | — | no | no |
| `BUZZ_AGENT_TOOL_TIMEOUT_SECS` | yes (`:154`) | — | no | no |
| `BUZZ_AGENT_MAX_PARALLEL_TOOLS` | yes (`:155`) | — | no | no |
| `BUZZ_AGENT_MAX_SESSIONS` | yes (`:156`) | — | no | no |
| `BUZZ_AGENT_MAX_LINE_BYTES` | yes (`:157`) | — | no | no |
| `BUZZ_AGENT_MAX_HISTORY_BYTES` | yes but **wrong default** (`:158`) | — | no | no |
| `BUZZ_AGENT_MAX_TOOL_RESULT_TEXT_BYTES` | yes (`:159`) | — | no | no |
| `BUZZ_AGENT_SYSTEM_PROMPT` / `_FILE` | yes (`:147-148`) | — | no | no |
| `MCP_HOOK_SERVERS` | no | yes (`:61`) | no | no |
| `BUZZ_AGENT_HOOK_TIMEOUT_MS` | no | yes (`:62`) | no | no |
| `BUZZ_AGENT_STOP_MAX_REJECTIONS` | no | yes (`:63`) | no | no |
| `BUZZ_AGENT_NO_HINTS` | no | no | no | no |
| `BUZZ_AGENT_MODEL` | no | no | no | no |
| `BUZZ_AGENT_THINKING_EFFORT` | no | no | no | no |
| `BUZZ_AGENT_MCP_INIT_TIMEOUT_SECS` | no | no | no | no |
| `BUZZ_AGENT_MCP_RESTART_MAX_ATTEMPTS` / `_BASE_MS` / `_MAX_MS` | no | no | no | no |

Documented default that contradicts the code: README says `BUZZ_AGENT_MAX_HISTORY_BYTES` defaults to `1048576` / "1 MiB" (`crates/buzz-agent/README.md:158`, repeated in the limits table at `:236`), but the code default is `16 * 1024 * 1024` (`config.rs:814`). Anyone sizing a deployment from the README is off by 16×. `BUZZ_AGENT_NO_HINTS` is undocumented anywhere yet is the only switch for the hints/skills injection this group performs at `lib.rs:356-359` (it *is* exercised by `hints_suppressed_with_env_var`, `tests/hints_integration.rs:223`).

#### Parsed but never read
- `InitializeParams::_client_capabilities` — deserialized with `#[serde(default)]` and discarded (`wire.rs:41-42`). The agent advertises capabilities but never adapts to the client's.
- `negotiated_version` — computed as `min(client, 2)` and echoed, never stored or consulted (`lib.rs:284-290`). The `[Base]` note at `lib.rs:255-259` implies behavior *should* depend on it; that dependency lives in buzz-acp (`crates/buzz-acp/src/pool.rs:181`) instead.
- `SessionNewParams` tolerates and drops unknown fields by design (test `session_new_params_ignores_unknown_fields`, `wire.rs:270`).

#### Config-driven behavior with no test coverage
`max_sessions` (both rejection paths, `lib.rs:346-355` and `lib.rs:399-409`) has no test — `grep -n 'max sessions reached' tests/` returns zero matches. `max_rounds` reaching the cap (`agent.rs:88-90`, returning `max_turn_requests`) likewise has no test in `crates/buzz-agent/tests/` (`grep -n 'MAX_ROUNDS\|max_turn_requests' tests/` → 0 matches). By contrast the handoff-related settings are covered by both unit tests (`handoff.rs:378-428`) and integration tests (`tests/regressions.rs:1224-1512`).


## Module: buzz-agent — LLM providers & configuration (`crates/buzz-agent/src`)
### Aspect: Configuration
`Config::from_env` (`config.rs:742-837`) is the crate's entire configuration surface: **36** environment variables, no config file, no CLI flag. (Count re-derived by extracting every quoted upper-case env-var name in `config.rs:742-837`; it was 35 before `16d4ec33` added `BUZZ_AGENT_PREFER_MESH_FOR_AUTO` at `config.rs:807`. The previous revision of this document said "32", which undercounted — the four tables below have always enumerated 35.) Documentation lives only in `crates/buzz-agent/README.md:130-156`; **none of these variables appear in the repo-root `.env.example`, `README.md`, `AGENTS.md`, `ARCHITECTURE.md`, or `CONTRIBUTING.md`** — `grep -rn 'BUZZ_AGENT_' .env.example README.md AGENTS.md ARCHITECTURE.md CONTRIBUTING.md` returned zero matches, and `grep -n 'AGENT\|ANTHROPIC\|OPENAI\|DATABRICKS' deploy/compose/.env.example` returned zero matches for these vars.

#### CLI flags read by this group
**None.** `grep` for `std::env::args`, `clap`, `argh`, `structopt` in `llm.rs` and `config.rs` returned zero matches. The crate's only argument handling is the `auth` subcommand dispatch at `lib.rs:111-117`, outside this group.

#### Provider / credential variables
| Env var | Default | Read site | Validation | In crate README? |
|---|---|---|---|---|
| `BUZZ_AGENT_PROVIDER` | none — **required** | `config.rs:746` → `resolve_provider` `config.rs:990` | absent/blank → error (`config.rs:1015-1018`); accepts `anthropic`, `openai`, `openai-compat`, `databricks`, `databricks_v2`, `databricks-v2` case-insensitively (`config.rs:996-1010`) | yes (`README.md:132`) — but the `openai-compat` and `databricks-v2` aliases are **undocumented** |
| `ANTHROPIC_API_KEY` | none — required for `anthropic` | `config.rs:747`, `config.rs:765` | non-empty after trim (`config.rs:999`, `config.rs:986-988`) | yes (`README.md:133`) |
| `ANTHROPIC_MODEL` | none | `config.rs:768` | required unless `BUZZ_AGENT_MODEL` set (`config.rs:766-770`) | yes (`README.md:134`) |
| `ANTHROPIC_BASE_URL` | `https://api.anthropic.com` | `config.rs:771` | **none** — no scheme or host check | yes (`README.md:135`) |
| `ANTHROPIC_API_VERSION` | `2023-06-01` | `config.rs:805` | none | yes (`README.md:136`) |
| `OPENAI_COMPAT_API_KEY` | none — required for `openai` | `config.rs:748`, `config.rs:775` | non-empty after trim (`config.rs:1003`) | yes (`README.md:137`) |
| `OPENAI_COMPAT_MODEL` | none | `config.rs:778` | required unless `BUZZ_AGENT_MODEL` set (`config.rs:776-780`) | yes (`README.md:138`) |
| `OPENAI_COMPAT_BASE_URL` | `https://api.openai.com/v1` | `config.rs:781` | **none** | yes (`README.md:139`) |
| `OPENAI_COMPAT_API` | `auto` | `config.rs:782` → `parse_openai_api` `config.rs:1022` | accepts `auto`, `""`, `chat`, `chat-completions`, `chat_completions`, `responses` (`config.rs:1023-1026`); anything else is an error | yes (`README.md:140`) — the `chat-completions`/`chat_completions` aliases are **undocumented** |
| `DATABRICKS_HOST` | none — required for both Databricks providers | `config.rs:743`, `config.rs:788` | presence only (`config.rs:788`); **no scheme check** | yes (`README.md:141`) |
| `DATABRICKS_MODEL` | none | `config.rs:744`, `config.rs:786` | required unless `BUZZ_AGENT_MODEL` set (`config.rs:786-787`) | yes (`README.md:142`) |
| `DATABRICKS_TOKEN` | `""` (empty ⇒ OAuth PKCE) | `config.rs:785` | none; emptiness is meaningful (`llm.rs:1535`) | yes (`README.md:143`) |
| `BUZZ_AGENT_MODEL` | none | `config.rs:755` | wins over the provider-specific model var (`config.rs:979-985`) | **NO** |

`BUZZ_AGENT_MODEL` is the highest-precedence model selector, is set programmatically by the persona layer (`crates/buzz-persona/src/resolve.rs:374`), is asserted in persona E2E tests (`crates/buzz-persona/tests/e2e_env_flow.rs:137-139`, `:308`), is surfaced in the desktop UI (`desktop/tests/e2e/edit-agent.spec.ts:8`), is set by the relay-mesh launcher (`desktop/src-tauri/src/managed_agents/relay_mesh.rs:28`), and appears in the benchmark harness docs (`benchmarks/harbor-buzz-orchestra/testbed/endpoints/README.md:19`) — yet it is missing from the one table that claims to enumerate the agent's configuration (`crates/buzz-agent/README.md:130-156`). It is also now load-bearing for a *behavioural* switch, not just model naming: the mesh policy fires only when the effective model resolves to the literal `auto` (`llm.rs:413`).

#### Mesh routing variable (new in `16d4ec33`)
| Env var | Default | Read site | Validation | Documented anywhere? |
|---|---|---|---|---|
| `BUZZ_AGENT_PREFER_MESH_FOR_AUTO` | `0` ⇒ disabled | `config.rs:807` | parsed as `u8` via `parse_env`; `prefer_mesh_for_auto = value != 0`, so any non-zero integer enables it and a non-numeric value (`true`, `yes`, `on`) is a **startup error**; no cross-field check against `provider` or `model` | **NO** — see below |

Documentation status, verified by `grep -rn 'BUZZ_AGENT_PREFER_MESH_FOR_AUTO'` across the repo. Exactly three occurrences exist, all in code:
1. `crates/buzz-agent/src/config.rs:732` — inside the field's own doc comment (`config.rs:729-733`), which is the only prose description anywhere. It states the intent ("Prefer mesh-llm's virtual `mesh` model when the configured/effective OpenAI model is `auto` and the live model catalog advertises it"), names the expected setter ("Buzz's relay-mesh provider"), and gives the expected value (`=1`).
2. `crates/buzz-agent/src/config.rs:807` — the parse site.
3. `desktop/src-tauri/src/managed_agents/relay_mesh.rs:6` — `RELAY_MESH_PREFER_MESH_FOR_AUTO_ENV`, the desktop constant, set to `"1"` at `desktop/src-tauri/src/managed_agents/relay_mesh.rs:41-44` behind `#[cfg(feature = "mesh-llm")]` (`desktop/src-tauri/src/managed_agents/relay_mesh.rs:5`).

It appears in **none** of: the repo-root `.env.example`, `crates/buzz-agent/README.md` (its configuration table ends at `README.md:156`), `AGENTS.md`, the root `README.md`, `ARCHITECTURE.md`, `CONTRIBUTING.md`, or `deploy/compose/.env.example`. So the crate's own README describes an `auto` semantics for `OPENAI_COMPAT_API` (`README.md:140`) with no mention that a second `auto`-related switch now exists and changes which model is actually invoked. This is the same documentation gap the ten pre-existing undocumented variables have, with one mitigation the others lack: the field doc comment (`config.rs:729-733`) is unusually complete, so a reader of `config.rs` is not left guessing.

Behavioural notes worth recording, all verified against `llm.rs`:
- The variable is **not** provider-scoped at parse time (`config.rs:807` sits in the shared struct literal), so setting it with `BUZZ_AGENT_PROVIDER=anthropic` is accepted silently and has no effect (`llm.rs:411`).
- Setting it without an `auto` model is likewise accepted and inert (`llm.rs:413`).
- Setting it against a non-mesh OpenAI-compatible endpoint is accepted and **not** inert: the agent will issue `GET {OPENAI_COMPAT_BASE_URL}/models` once per 5 s window (`llm.rs:473`, TTL `llm.rs:30`) for the process lifetime. Nothing warns about this.
- Its tuning is entirely non-configurable: the TTL, probe timeout, cooldown, and confirmation count are `const`s in `llm.rs:30-33` with no env override — grep for `MESH_AUTO` in `config.rs` returns nothing.

#### Prompt variables
| Env var | Default | Read site | Validation | In crate README? |
|---|---|---|---|---|
| `BUZZ_AGENT_SYSTEM_PROMPT` | `DEFAULT_SYSTEM_PROMPT` (`config.rs:658-659`) | `config.rs:792` | mutually exclusive with `_FILE` (`config.rs:792-794`) | yes (`README.md:144`) |
| `BUZZ_AGENT_SYSTEM_PROMPT_FILE` | none | `config.rs:792` | file read at startup; IO error → `config: read {p}: {e}` (`config.rs:796`) | yes (`README.md:145`) |

Neither is length-checked here. `MAX_SYSTEM_PROMPT_BYTES` (512 KiB, `config.rs:639`) is enforced in `lib.rs:375-383` against a per-session prompt override, **not** against the env-supplied prompt — so `BUZZ_AGENT_SYSTEM_PROMPT_FILE` pointing at a 100 MiB file is accepted at startup with no bound. `grep -n 'MAX_SYSTEM_PROMPT_BYTES' config.rs` returns only the definition at `config.rs:639`.

#### Numeric / duration variables
All go through `parse_env` (`config.rs:1049-1056`), which returns the default only when the variable is **absent**; a present-but-unparseable value is a hard startup error prefixed `config: {key}: `.

| Env var | Default (code) | Read site | Validation | README default | Match? |
|---|---|---|---|---|---|
| `BUZZ_AGENT_MAX_ROUNDS` | `0` | `config.rs:808` | none | `0` (`README.md:146`) | yes |
| `BUZZ_AGENT_MAX_OUTPUT_TOKENS` | `32_768` | `config.rs:809` | `>= 1` (`config.rs:884`); `< max_context_tokens` (`config.rs:887`) | `32768` (`README.md:147`) | yes |
| `BUZZ_AGENT_MAX_CONTEXT_TOKENS` | `200_000` | `config.rs:826` | `> max_output_tokens` (`config.rs:887`) | `200000` (`README.md:148`) | yes |
| `BUZZ_AGENT_MAX_HANDOFFS` | `10` | `config.rs:827` | none | `10` (`README.md:149`) | yes |
| `BUZZ_AGENT_LLM_TIMEOUT_SECS` | `240` | `config.rs:810` | `>= 1s` (`config.rs:916`) | `240` (`README.md:150`) | yes |
| `BUZZ_AGENT_TOOL_TIMEOUT_SECS` | `660` | `config.rs:811` | `>= 1s` (`config.rs:919`) | `660` (`README.md:151`) | yes |
| `BUZZ_AGENT_MAX_PARALLEL_TOOLS` | `8` | `config.rs:828` | `>= 1` (`config.rs:925`) | `8` (`README.md:152`) | yes |
| `BUZZ_AGENT_MAX_SESSIONS` | `usize::MAX` | `config.rs:819` | none | "unlimited" (`README.md:153`) | yes |
| `BUZZ_AGENT_MAX_LINE_BYTES` | `4 * 1024 * 1024` | `config.rs:820` | `>= 1024` (`config.rs:904`) | `4194304` (`README.md:154`) | yes |
| `BUZZ_AGENT_MAX_HISTORY_BYTES` | **`16 * 1024 * 1024`** | `config.rs:821` | `>= 4096` (`config.rs:893`) **and** `>= MAX_PROMPT_BYTES` (1 MiB, `config.rs:898`) | **`1048576` / "1 MiB"** (`README.md:155`, repeated `README.md:236`) | **NO — 16× discrepancy** |
| `BUZZ_AGENT_MAX_TOOL_RESULT_TEXT_BYTES` | `DEFAULT_TOOL_RESULT_TEXT_BYTES` = 50 KiB (`config.rs:649`) | `config.rs:823` | `1024 ..= MAX_TOOL_RESULT_BYTES` (8 MiB) (`config.rs:909-915`) | `51200` (`README.md:156`) | yes |
| `BUZZ_AGENT_MCP_INIT_TIMEOUT_SECS` | `30` | `config.rs:812-815` | `>= 1s` (`config.rs:922`) | — | **NO** |
| `BUZZ_AGENT_MCP_RESTART_MAX_ATTEMPTS` | `3` | `config.rs:816` | `>= 1` (`config.rs:928`) | — | **NO** |
| `BUZZ_AGENT_MCP_RESTART_BASE_MS` | `500` | `config.rs:817` | `>= 1` (`config.rs:931`) | — | **NO** |
| `BUZZ_AGENT_MCP_RESTART_MAX_MS` | `30_000` | `config.rs:818` | `>= mcp_restart_base_ms` (`config.rs:934`) | — | **NO** |
| `BUZZ_AGENT_HOOK_TIMEOUT_MS` | `2500` | `config.rs:829` | none | — | **NO** |
| `BUZZ_AGENT_STOP_MAX_REJECTIONS` | `3` | `config.rs:830` | none; `0` disables `_Stop` hooks (`config.rs:716-717`) | — | **NO** |

The `BUZZ_AGENT_MAX_HISTORY_BYTES` discrepancy is the sharpest doc contradiction in this group. Code: `16 * 1024 * 1024` at `config.rs:821`. README states `1048576` with the note "1 MiB. Old turns are evicted past this" at `README.md:155`, and repeats "History window | 1 MiB" in the Bounded Everything table at `README.md:236`. The code-side default is also confirmed by the `for_discovery` fixture (`config.rs:865`) and the `llm.rs` test fixture (`llm.rs:1593`, which sets `max_history_bytes: 16 * 1024 * 1024`). An operator reading the README would size their history budget 16× too small.

#### Non-numeric behavioural variables
| Env var | Default | Read site | Parse rules | In crate README? |
|---|---|---|---|---|
| `MCP_HOOK_SERVERS` | unset ⇒ `HookServers::None` (hooks off) | `config.rs:831` → `parse_hook_servers_env` `config.rs:1085` → `parse_hook_servers` `config.rs:1091` | comma-separated; entries trimmed, empties dropped (`config.rs:1096-1101`); all-empty ⇒ `None` (`config.rs:1101-1103`); lone `*` ⇒ `All` (`config.rs:1108-1109`); `*,foo` ⇒ literal `Only(["*","foo"])` (`config.rs:1104-1111`) | **NO** |
| `BUZZ_AGENT_NO_HINTS` | `0` ⇒ hints enabled | `config.rs:832` | parsed as `u8`; `hints_enabled = value == 0`, so any non-zero disables; a non-numeric value (`true`, `yes`) is a **startup error** | **NO** |
| `BUZZ_AGENT_THINKING_EFFORT` | unset ⇒ `None` (no thinking config sent) | `config.rs:833` → `parse_thinking_effort` `config.rs:622` | trimmed, lowercased; accepts `none`, `minimal`, `low`, `medium`, `high`, `xhigh`, `max`; `""` and whitespace-only ⇒ `None`; anything else is an error naming the value and the legal set (`config.rs:633-636`); Anthropic additionally rejects `none`/`minimal` at startup (`config.rs:947-957`) | **NO** |

`MCP_HOOK_SERVERS` is the odd one out on naming — it is the only variable in `from_env` without the `BUZZ_AGENT_` prefix (`config.rs:831`). It is also the gate for the entire MCP-driven-hooks feature that `README.md:254` links to (`docs/MCP_DRIVEN_HOOKS.md`), and it is used in six integration tests (`crates/buzz-agent/tests/regressions.rs:805`, `:886`, `:939`, `:993`, `:1133`, `:1519`) — a documented feature whose only enabling switch is absent from the configuration table.

`BUZZ_AGENT_THINKING_EFFORT` is the most consequential undocumented variable in the crate. It has its own 13-line type doc (`config.rs:5-17`), five dedicated warning messages that name it back to the operator (`config.rs:153`, `config.rs:223`, `config.rs:516`, `config.rs:545`, `config.rs:568`), roughly 70 unit tests across `config.rs` and `llm.rs`, a desktop UI with a shared JSON fixture (`desktop/src/features/agents/ui/effortTable.fixture.json`), a desktop-specific guidance note (`desktop/src/features/agents/AGENTS.md:35`), is set explicitly by the relay-mesh launcher to `none` (`desktop/src-tauri/src/managed_agents/relay_mesh.rs:52`), and is set in a relay example (`crates/buzz-relay/examples/mesh_agent_e2e.rs:281`). It appears nowhere in `crates/buzz-agent/README.md`.

`BUZZ_AGENT_NO_HINTS` is documented only implicitly, via the integration test at `crates/buzz-agent/tests/hints_integration.rs:230`.

#### Summary of documentation gaps
**Eleven** of the 36 variables read by `from_env` are absent from the crate README's configuration table (`README.md:130-156`) — the ten previously recorded plus the new mesh flag:

`BUZZ_AGENT_MODEL` (`config.rs:755`), `BUZZ_AGENT_PREFER_MESH_FOR_AUTO` (`config.rs:807`), `BUZZ_AGENT_MCP_INIT_TIMEOUT_SECS` (`config.rs:812`), `BUZZ_AGENT_MCP_RESTART_MAX_ATTEMPTS` (`config.rs:816`), `BUZZ_AGENT_MCP_RESTART_BASE_MS` (`config.rs:817`), `BUZZ_AGENT_MCP_RESTART_MAX_MS` (`config.rs:818`), `BUZZ_AGENT_HOOK_TIMEOUT_MS` (`config.rs:829`), `BUZZ_AGENT_STOP_MAX_REJECTIONS` (`config.rs:830`), `MCP_HOOK_SERVERS` (`config.rs:831`), `BUZZ_AGENT_NO_HINTS` (`config.rs:832`), `BUZZ_AGENT_THINKING_EFFORT` (`config.rs:833`).

Plus one wrong default (`BUZZ_AGENT_MAX_HISTORY_BYTES`) and three undocumented aliases (`openai-compat`, `databricks-v2`, `chat-completions`/`chat_completions`).

#### Parsed-but-never-read configuration
| Field | Parsed at | Read anywhere? |
|---|---|---|
| `Config.model` | `config.rs:763-789` | Yes, but **not** by `llm.rs` — every body builder takes an explicit `effective_model: &str` parameter (`llm.rs:681`, `llm.rs:766`, `llm.rs:879`) and the `Config.model` field is never read there. `grep -n 'cfg.model' llm.rs` returned zero matches. The field is consumed by the session layer (`wire.rs:85` documents `model_id` overriding `BUZZ_AGENT_MODEL`), which resolves the per-session effective model. Two tests explicitly assert that `effective_model` wins over `cfg.model` (`llm.rs:2837`, `llm.rs:2850`). |
| `Config.prefer_mesh_for_auto` | `config.rs:807` | Yes, at exactly one site: `llm.rs:412`, inside `resolve_openai_model`'s eligibility guard. It is the only `Config` field whose sole reader is `llm.rs`, and the only one that is inert for three of the four providers |
| `Config.openai_api` for `Provider::Anthropic` | hard-coded `Auto` at `config.rs:772` | Not read on the Anthropic path (`llm.rs:133-146`) — genuinely inert, and commented as such |
| `Config.openai_api` for `Provider::Databricks` | hard-coded `Chat` at `config.rs:789`, commented "only read by OpenAI/legacy Databricks dispatch" | **Is** read, at `llm.rs:545-546` and `llm.rs:563`. Hard-coding `Chat` permanently disables the chat→responses auto-upgrade for legacy Databricks. That is a behavioural decision expressed as an inert default with no comment saying so. |
| `Config.anthropic_api_version` | `config.rs:805` | Read only on the Anthropic path (`llm.rs:334`). For `Provider::DatabricksV2`'s Anthropic Messages route (`llm.rs:986`) it is **never sent** — `post_openai` applies only `bearer_auth` (`llm.rs:638`), so the gateway's Anthropic endpoint receives no `anthropic-version` header. Whether the gateway requires one is unverified; no test covers it. |

Fields validated in `config.rs` but enforced elsewhere (not a defect, just a boundary worth recording): `max_rounds`, `max_history_bytes`, `max_line_bytes`, `max_tool_result_text_bytes`, `max_context_tokens`, `max_handoffs`, `max_parallel_tools`, `tool_timeout`, `hook_timeout`, `stop_max_rejections`, `hook_servers`, `hints_enabled`, `max_sessions`, and the four `mcp_*` fields are all read by `agent.rs`, `mcp.rs`, `handoff.rs`, `hints.rs`, or `lib.rs`. Of `Config`'s 27 fields, `llm.rs` reads exactly **nine**: `provider`, `base_url`, `api_key`, `anthropic_api_version`, `openai_api`, `prefer_mesh_for_auto`, `max_output_tokens`, `thinking_effort`, and `llm_timeout` (verified by grepping `cfg\.` in `llm.rs` lines 1-1570; `prefer_mesh_for_auto` at `llm.rs:412` is the ninth, added by `16d4ec33`).

#### Deprecated aliases
No variable is marked deprecated and there is no deprecation-warning path — `grep -ni 'deprecat' config.rs llm.rs` returned zero matches. The provider aliases (`openai-compat` at `config.rs:1003`, `databricks-v2` at `config.rs:1008`) and the `OPENAI_COMPAT_API` aliases (`chat-completions`, `chat_completions` at `config.rs:1023`) are accepted silently and equally, with no indication of which form is canonical.

#### Test coverage for this aspect
Covered:
- `parse_openai_api` including the empty-string, whitespace, and mixed-case forms, plus the error path (`config.rs:1206-1222`).
- `resolve_provider` for present/missing keys, absent provider, unsupported value with casing preserved (`config.rs:1224-1272`).
- `resolve_model` all three precedence cases (`config.rs:1293`, `config.rs:1299`, `config.rs:1305`).
- `parse_thinking_effort` round-trip, empty/whitespace, case-insensitivity, and the error message content (`config.rs:1311-1356`).
- `parse_hook_servers` — ten tests covering unset, empty, whitespace-only, `*`, `*` with padding, named list, trimming, the mixed `*,foo` case, and `allows()` semantics (`config.rs:1119-1203`).
- `validate`'s thinking-effort rule for all four providers × 7 levels (`config.rs:1867-1991`).
- **New:** `prefer_mesh_for_auto`'s *effect* is well covered from the `llm.rs` side — `true` with model `auto` (`llm.rs:1826`), `false` with model `auto` (`llm.rs:2121`), `true` with an explicit model (`llm.rs:2155`), and `true` with an explicit `mesh` (`llm.rs:2136`). The desktop side asserts the env var is emitted with value `"1"` (`desktop/src-tauri/src/managed_agents/relay_mesh.rs:79-82`).

Not covered:
- **`Config::from_env` has zero tests.** No test reads or mutates process env; every fixture is a struct literal (`llm.rs:1579-1610`) or a patched `for_discovery` (`config.rs:1844-1864`). So none of the 36 defaults, none of the required-variable error messages, the `BUZZ_AGENT_SYSTEM_PROMPT` / `_FILE` mutual exclusion (`config.rs:792-794`), the file-read error path (`config.rs:796`), the `BUZZ_AGENT_NO_HINTS` `== 0` inversion (`config.rs:832`), or the new `BUZZ_AGENT_PREFER_MESH_FOR_AUTO` `!= 0` inversion (`config.rs:807`) is verified in a unit test. Unlike `parse_thinking_effort` and `parse_hook_servers`, the mesh flag has no pure parser to test — it is inlined in the struct literal — so its parse semantics (non-zero enables, non-numeric is fatal) are asserted nowhere in this crate.
- The 13 numeric/duration invariants at `config.rs:884-939` are never asserted — the only `validate()` tests (`config.rs:1867-1991`) construct a config that already satisfies them via `make_config_for_validation` (`config.rs:1844-1864`) and vary only `thinking_effort`. That helper even has to *undo* `for_discovery`'s invalid values (`config.rs:1855-1862`) to reach the effort check, which is direct evidence the numeric branches are unexercised.
- No cross-field validation of `prefer_mesh_for_auto` exists, so there is nothing to test: enabling it on a non-OpenAI provider, or with a non-`auto` model, or against a base URL with no `/models` endpoint, are all silently accepted at startup (`config.rs:878-959` contains no reference to the field).
- `HookServers::is_disabled` (`config.rs:1080`) — no test.


## Module: buzz-agent — MCP registry, OAuth, hints & catalog (`crates/buzz-agent/src`)
### Aspect: Configuration

Documentation columns below mean: **README** = `crates/buzz-agent/README.md` config table; **.env.example** = repo-root `.env.example`; **AGENTS.md** = repo-root `AGENTS.md`; **hooks doc** = `docs/MCP_DRIVEN_HOOKS.md`. Counts come from `grep -c '<VAR>' <file>` run per variable.

#### Environment read directly inside this group

| Variable | Read site | Purpose | Default / fallback | Documented |
|---|---|---|---|---|
| `HOME` | `hints.rs:11` (`home_dir`) | locate `~/AGENTS.md` and `~/.agents/skills` | `None` → global layer silently skipped (`hints.rs:55-60`, `hints.rs:212-214`) | no |
| `HOME` | `auth.rs:457-458` (`cache_path_for`) | OAuth cache root | none — hard error `oauth cache: $HOME not set` | no |
| every key in `PASSTHROUGH_ENV` | `mcp.rs:716` (`std::env::var(k)`) | forwarded into each MCP child after `env_clear()` | absent keys are simply not forwarded | partially, and inaccurately (see below) |
| `PASSTHROUGH_ENV_WINDOWS` + `WINDOWS_SHELL_RESOLUTION_ENV` | `mcp.rs:722` | Windows child temp/profile/shell resolution | as above | no |

The forwarded set (`mcp.rs:39-63`, plus `mcp.rs:70-71` and `lib.rs:19-30` on Windows):

| Key | Forwarded from | Notes |
|---|---|---|
| `PATH`, `HOME`, `TERM`, `LANG`, `LC_ALL`, `TMPDIR`, `XDG_CONFIG_HOME` | agent env | `XDG_CONFIG_HOME` is forwarded to children but **not** honoured by this crate's own cache path, which hardcodes `.config` (`auth.rs:460`) |
| `SSH_AUTH_SOCK`, `SSH_AGENT_PID` | agent env | git-over-SSH |
| `GIT_ASKPASS`, `GIT_SSH_COMMAND`, `GIT_CONFIG_GLOBAL` | agent env | operator git overrides |
| `NOSTR_PRIVATE_KEY`, `BUZZ_PRIVATE_KEY`, `BUZZ_RELAY_URL`, `BUZZ_AUTH_TAG` | agent env | `AGENTS.md:162-164` documents the last three as harness-injected; `NOSTR_PRIVATE_KEY` appears in neither `AGENTS.md` nor the crate README |
| `TMP`, `TEMP`, `USERPROFILE`, `APPDATA` (Windows) | agent env | needed because `std::env::temp_dir()` otherwise resolves to an unwritable location (`mcp.rs:66-69`) |
| `PATH`, `BUZZ_SHELL`, `GIT_BASH`, `SystemRoot`, `ProgramFiles`, `ProgramFiles(x86)`, `LOCALAPPDATA` (Windows) | agent env | contract shared with Doctor (`lib.rs:16-30`) |

Client-supplied `env` from the `session/new` payload is applied *after* the allowlist (`mcp.rs:726-728`), so a payload entry wins over an inherited value. That channel is per-server configuration, not process configuration, and is not covered by any env-var documentation.

#### Configuration consumed by this group but read in `config.rs`

| Variable | Read site (`config.rs`) | Default | Consumed here | README | .env.example | hooks doc |
|---|---|---|---|---|---|---|
| `BUZZ_AGENT_MCP_INIT_TIMEOUT_SECS` | `805-808` | 30 s | `mcp.rs:191` → `spawn_one` init + `tools/list` (`mcp.rs:757`, `:767`) | no (0) | no (0) | no |
| `BUZZ_AGENT_MCP_RESTART_MAX_ATTEMPTS` | `809` | 3 | `mcp.rs:188` (`.max(1)`), `mcp.rs:135`, `mcp.rs:303` | no (0) | no (0) | no |
| `BUZZ_AGENT_MCP_RESTART_BASE_MS` | `810` | 500 ms | `mcp.rs:189` (`.max(1)`) → `backoff` (`mcp.rs:813`) | no (0) | no (0) | no |
| `BUZZ_AGENT_MCP_RESTART_MAX_MS` | `811` | 30 000 ms | `mcp.rs:190` (`.max(1)`) → `backoff` cap | no (0) | no (0) | no |
| `BUZZ_AGENT_MAX_TOOL_RESULT_TEXT_BYTES` | `815-819` | 51 200 (`config.rs:649`) | `agent.rs:388` → `ResultBudget.text` (`mcp.rs:933-935`) | yes | no (0) | no |
| `BUZZ_AGENT_HOOK_TIMEOUT_MS` | `822` | 2 500 ms | `agent.rs:228`, `handoff.rs:78` → `mcp.rs:352` | no (0) | no (0) | yes, default matches |
| `BUZZ_AGENT_STOP_MAX_REJECTIONS` | `823` | 3 | `agent.rs:228-236` gate around `call_hooks` | no (0) | no (0) | yes, default matches |
| `MCP_HOOK_SERVERS` | `824` (parser `1083-1105`) | unset → `HookServers::None` | `mcp.rs:321-331` | no (0) | no (0) | yes, default matches |
| `BUZZ_AGENT_NO_HINTS` | `825` | `0` (hints on) | `lib.rs:355-360` → `hints::build_hints_section` | no (0) | no (0) | no |
| `BUZZ_AGENT_PROVIDER` | `739-743` | required | `catalog.rs:123-130` dispatch, `llm.rs:1176-1198` token-source choice | yes | no (0) | no |
| `DATABRICKS_HOST` | `737`, `782` | required for Databricks | `catalog.rs:121` (`cfg.base_url`), OAuth discovery URL (`llm.rs:1183-1186`) | yes | no (0) | no |
| `DATABRICKS_TOKEN` | `779` | empty → use PKCE | empty/non-empty selects `StaticTokenSource` vs `PkceOAuthTokenSource` (`llm.rs:1179-1197`) | yes | no (0) | no |
| `DATABRICKS_MODEL` | `738`, `781` | required for Databricks | `discovery_failure_fallback(provider, cfg.model)` (`catalog.rs:51-80`, `lib.rs:447-452`) | yes | no (0) | no |
| `BUZZ_AGENT_MODEL` | `748` | none | overrides `DATABRICKS_MODEL`, so it changes the fallback catalog | no (0) | no (0) | no |

Whitespace handling of the resolved model changed in `8eb6e3eb`: `discovery_failure_fallback` now `trim()`s the configured model on entry (`catalog.rs:55`) precisely because `resolve_model` does not (`catalog.rs:52-54`). So `DATABRICKS_MODEL=" databricks-gpt-5-5 "` no longer produces a duplicated picker entry, and a value that is empty after trimming is dropped from the fallback catalog entirely rather than advertised as an empty-string model id (`catalog.rs:63-65`). Both behaviours are pinned by unit tests (`catalog.rs:612`, `catalog.rs:621`). Note this is a *local* normalisation inside the fallback helper — `cfg.model` itself is still untrimmed everywhere else it is used, including the `currentModelId` echoed by `session/new` (`lib.rs:470`).

`MCP_HOOK_SERVERS` semantics (`config.rs:1083-1105`): unset / empty / whitespace-only → `None` (hooks off); a lone `*` → `All`; otherwise a trimmed comma list → `Only([...])`. A mixed value like `*,foo` is deliberately *not* a wildcard — `*` cannot pass `valid_name` (`mcp.rs:859-864`) so it never matches a real server (comment `config.rs:1096-1099`, tests `config.rs:1128`, `config.rs:1176`).

#### CLI surface

| Invocation | Site | Behaviour |
|---|---|---|
| `buzz-agent` (no args) | `lib.rs:109-122` | runs the ACP stdio loop |
| `buzz-agent auth <provider>` | `lib.rs:111-116`, `lib.rs:126-153` | one-shot interactive OAuth; `databricks`, `databricks_v2`, `databricks-v2` accepted (`lib.rs:131`); requires `DATABRICKS_HOST` (`lib.rs:133-134`); prints the cache directory on success (`lib.rs:147`) |
| `buzz-agent auth` (no provider) | `lib.rs:151` | error: "provider required (try: buzz-agent auth databricks)" |
| `buzz-agent auth <other>` | `lib.rs:150` | error: `auth: unknown provider "<other>"` |
| anything else, e.g. `--help`, `--version` | `lib.rs:110-121` | **not** a flag — argv is only inspected for the literal `auth`, so any other argument falls through to the stdio loop and the process waits on stdin |

There is no flag parser and no `clap` dependency (`grep -n 'clap' crates/buzz-agent/Cargo.toml` → zero matches). This matches the README's stance ("Everything is environment variables. No flags, no config files") except that the README does not mention the `auth` subcommand at all.

The `auth` subcommand hardcodes its own OAuth parameters instead of reusing the constants the runtime uses: `client_id: "databricks-cli"` and `scopes: ["all-apis","offline_access"]` at `lib.rs:139-141` versus `DATABRICKS_CLIENT_ID` / `DATABRICKS_OAUTH_SCOPES` at `llm.rs:19-20`. Since the cache filename hashes those exact values (`auth.rs:446-454`), a divergence would send the CLI's token to a file the runtime never reads — a configuration coupling with no compile-time or test-time guard.

#### Configuration that exists in code but is not reachable in production

| Item | Site | Why |
|---|---|---|
| `PkceOAuthConfig::cache_dir_override` | `auth.rs:102-107` | documented as test-only; both production constructors pass `None` (`lib.rs:143`, `llm.rs:1194`). There is no env var to relocate the token cache |
| `PkceOAuthConfig::cache_namespace` | `auth.rs:101` | a real knob, but hardcoded to `"databricks"` at both sites (`lib.rs:142`, `llm.rs:1193`) |
| `MAX_NAME_LEN = 128` | `mcp.rs:20` | tighter `MAX_QNAME_LEN = 64` makes it unreachable for any name that produces a tool (`mcp.rs:242-248`) |
| `next_retry = now + 86_400s` on exhaustion | `mcp.rs:687-690` | never consulted; the exhausted branch errors earlier (`mcp.rs:135-140`) |

#### Hardcoded values that arguably should be configurable

| Value | Site | Impact |
|---|---|---|
| `MAX_MCP_SERVERS = 16`, `MAX_TOOLS_PER_SESSION = 128` | `mcp.rs:26`, `mcp.rs:22` | hard session-creation failure when exceeded; documented in the README's "Bounded Everything" table but not tunable |
| `MAX_HOOK_RESULT_BYTES = 16 KiB` | `mcp.rs:27` | per-hook output cap; not in the README or hooks doc |
| `MAX_HINTS_BYTES = 128 KiB` | `hints.rs:6` | hint-chain cap; undocumented anywhere |
| `MAX_SKILL_BODY_BYTES = 32 KiB` | `hints.rs:7` | `load_skill` output cap; undocumented anywhere |
| `SKILL_DIRS` | `hints.rs:8` | the three project skill directories and their precedence are not configurable and not documented |
| `TOKEN_REFRESH_LEEWAY = 60s`, `BROWSER_AUTH_TIMEOUT = 60s` | `auth.rs:35`, `auth.rs:39` | the browser window in particular is a UX-visible timeout with no override |
| 20-page / `page_size=100` catalog ceiling | `catalog.rs:244-247` | silently caps discovery at 2 000 endpoints |
| `DATABRICKS_V2_KNOWN_MODELS` | `catalog.rs:32-33` | the fallback model list is compiled in; operators cannot extend it |
| no HTTP timeouts for OAuth/catalog | `auth.rs:153`, `catalog.rs:120` | `Client::new()` means neither a default nor a configurable timeout, unlike `llm.rs:53-57` |
| embedding-endpoint reject vocabulary — the substring `embedding` plus the segments `bge` and `gte` | `catalog.rs:99-105` | new in `8eb6e3eb`. Compiled in, so an operator whose workspace serves an embedding family named outside that vocabulary cannot exclude it from the picker, and an operator whose chat endpoint happens to contain `embedding` in its name cannot re-include it. There is no env override and no logging of a drop (`grep -n 'tracing::' catalog.rs` → zero matches) |
| newest-first catalog ordering | `catalog.rs:337-344`, applied `catalog.rs:298` | new in `8eb6e3eb`. Sort order is fixed; there is no way to ask for the gateway's own ordering back |

#### Documentation status summary

- Repo-root `.env.example` contains **no** agent configuration at all: `grep -c 'BUZZ_AGENT\|DATABRICKS\|MCP_HOOK' .env.example` → 0 (the file is scoped to the relay/backend, header at `.env.example:1-16`). `deploy/compose/.env.example` likewise → 0. So every variable in this group is undocumented in the canonical env template.
- The crate README documents provider/Databricks selection and the tool-result text cap, but none of the four `BUZZ_AGENT_MCP_*` knobs, none of the three hook knobs, and neither hints nor skills configuration.
- `docs/MCP_DRIVEN_HOOKS.md` is the only place the hook configuration is documented, and its three defaults (unset, 2500, 3) agree with `config.rs:822-824`.
- Independent evidence that the README's config table is not maintained in lockstep with the code: it lists `BUZZ_AGENT_MAX_HISTORY_BYTES` as `1048576` / "1 MiB" (`crates/buzz-agent/README.md:155`, repeated at `:236`) while `config.rs:815` defaults it to `16 * 1024 * 1024`. That variable is not consumed by this group, but it calibrates how much the "documented" column above can be trusted.

#### Test coverage — Configuration

Covered: `MCP_HOOK_SERVERS` parsing has twelve dedicated unit tests (`config.rs:1111`, `1116`, `1121`, `1129`, `1134`, `1142`, `1150`, `1158`, `1168`, `1176`, `1181`, `1186`); `BUZZ_AGENT_NO_HINTS` is covered end-to-end (`tests/hints_integration.rs:223 hints_suppressed_with_env_var`); `MCP_HOOK_SERVERS` reaching the registry is covered by six hook integration tests that set it on the agent process (`tests/regressions.rs:805`, `886`, `939`, `993`, `1133`, `1519`) plus one that deliberately leaves it unset (`:1044`); `BUZZ_AGENT_MCP_INIT_TIMEOUT_SECS` is exercised implicitly by `mcp_init_timeout_kills_child` (`tests/regressions.rs:307`).

Not covered: the three restart knobs (`grep -rn 'RESTART_MAX_ATTEMPTS\|RESTART_BASE_MS\|RESTART_MAX_MS' crates/buzz-agent/tests` → zero matches); `HOME`-absent behaviour for the OAuth cache (`auth.rs:457-458` returns an error no test triggers) versus `HOME`-absent behaviour for hints (covered, `hints.rs:523 load_hint_files_no_home_dir`); the `auth` CLI argument parsing (`grep -rn '"auth"' crates/buzz-agent/tests` → zero matches); and the claim that `cache_dir_override` is production-unused, which nothing enforces.


## Module: buzz-dev-mcp (`crates/buzz-dev-mcp`)
### Aspect: Configuration

There is no config file, no CLI flag, and no `clap` parser in MCP-server mode. All
configuration is environment variables read at startup or per call, plus the
process's `argv[0]` and current working directory.

#### Environment variables read by this crate

| Variable | Default | Parse site | Effect |
|---|---|---|---|
| `BUZZ_SHELL` | unset → `bash` | `shell.rs:366` (Unix), `shell.rs:411` (Windows) | selects the shell binary. A value with more than one path component (or a root) must exist as a file, else it is ignored and the resolver falls through; a bare name is searched on PATH. On Windows this branch deliberately skips the `%SystemRoot%` exclusion so `cmd`/`powershell` resolve (`shell.rs:411-423`). The resolved stem also drives the flag: `cmd`→`/C`, `powershell`/`pwsh`→`-Command`, else `-c` (`shell.rs:336-348`) |
| `GIT_BASH` | unset | `shell.rs:424` | Windows-only legacy override; must be an existing file |
| `PATH` | inherited | `shim.rs:42-49` (rebuild), `shell.rs:377` (Unix `BUZZ_SHELL` bare-name scan), `rg.rs:32` (`clean_path`) | the shim dir is prepended with `std::env::join_paths` and the result is set as the child's `PATH` (`shell.rs:169`). `rg.rs` re-reads the raw `PATH` and splits it on hardcoded `':'` (`rg.rs:34`, `rg.rs:50`) |
| `NOSTR_PRIVATE_KEY` | unset | `shim.rs:54` | if set and parseable, written to a `0600` keyfile and used to build ten `GIT_CONFIG_*` settings; then **removed from the process env unconditionally** (`shim.rs:51-68`). An unparseable value logs a warning to stderr and disables git auth/signing (`shim.rs:90-99`) |
| `BUZZ_PRIVATE_KEY` | unset | `shell.rs:78` (presence only), `view_image.rs:299` (value) | presence gates the "Buzz relay configured" bootstrap line; value is parsed as `nostr::Keys` to sign Blossom `t=get` media-read tokens. Also inherited verbatim by every `shell` child (`shell.rs:171`) |
| `BUZZ_RELAY_URL` | unset | `shell.rs:78` (presence), `shim.rs:155` (host extraction), `view_image.rs:294` (URL parse) | presence gates the bootstrap hint; host becomes the `user.email` domain unless it is empty/localhost/`127.*` (`shim.rs:154-172`); parsed URL is the authority the relay-media auth gate compares against (`view_image.rs:236-247`) |
| `BUZZ_AUTH_TAG` | unset | `view_image.rs:341` | when a relay-media token is attached and the value is non-blank after trim, sent as the `x-auth-tag` header (`view_image.rs:341-345`) |
| `GIT_CONFIG_COUNT` | `0` | `shim.rs:200-204` | existing count is used as the index base so the shim's ten entries append rather than clobber |
| `SystemRoot` | unset | `shell.rs:431` | Windows-only: PATH entries under it are excluded from the implicit `bash.exe` scan (WSL avoidance) |
| `ProgramFiles`, `ProgramFiles(x86)`, `LocalAppData` | unset | `shell.rs:544-546` | Windows-only: standard Git-for-Windows install-location probes |

Per-call inputs that behave as configuration: `workdir` (`shell.rs:145-149`,
`read_file.rs:20-21`, `str_replace.rs:22-23`, `view_image.rs:74-76`) and
`timeout_ms` (`shell.rs:141-144`).

Implicit configuration from process state: `std::env::current_dir()` becomes
`SharedState.cwd`, the default workspace root for every tool (`lib.rs:179-181`);
`argv[0]`'s file stem selects the multicall personality (`lib.rs:139-153`);
`std::env::current_exe()` is the symlink target for the shim
(`shim.rs:29`, `rg.rs:19-21`).

#### Sandbox-root configuration

**There is none.** `SharedState.cwd` (`shell.rs:27`) is only the default value
used when `workdir` is omitted; it is not a boundary. No env var, flag, or
parameter restricts which directories the tools may touch — see the Security
aspect and the explicit statement at `paths.rs:3-6`.

#### Compile-time configuration

| Setting | Value | Site |
|---|---|---|
| `unsafe` policy | `forbid` on non-Windows, `deny` on Windows | `lib.rs:1-2` |
| tokio runtime | `new_current_thread().enable_all()` | `lib.rs:159-162` |
| tracing | stderr writer, ANSI disabled | `lib.rs:174-177` |
| TLS provider | `rustls::crypto::ring` installed as default (repeat installs ignored) | `lib.rs:164-166`, `Cargo.toml:34-36` |
| `image` codecs | `jpeg`, `png`, `gif`, `webp` only — `default-features = false` | `Cargo.toml:40` |
| `reqwest` | rustls, no default features | `Cargo.toml:93` (workspace) |
| Windows `windows-sys` features | `Win32_Foundation`, `Win32_Security`, `Win32_System_JobObjects`, `Win32_System_Registry`, `Win32_System_Threading` | `Cargo.toml:47-52` |
| `[lib]` / `[[bin]]` | lib `buzz_dev_mcp` at `src/lib.rs`, bin `buzz-dev-mcp` at `src/main.rs` | `Cargo.toml:8-15` |

Everything else that could plausibly be configurable is a hardcoded `const`. The
following have **no** environment or flag override:

| Constant | Value | Site |
|---|---|---|
| `DEFAULT_TIMEOUT_MS` / `MAX_TIMEOUT_MS` | 120 s / 600 s | `shell.rs:16-17` |
| `MAX_COMMAND_BYTES` | 1 MB | `shell.rs:18` |
| `CAPTURE_CAP` / `MAX_BYTES` / `MAX_LINES` / `TAIL_BYTES` | 10 MiB / 50 KiB / 2000 / 8 KiB | `shell.rs:19-22` |
| `ARTIFACT_RING_SIZE` / `READ_CHUNK` | 8 / 16 KiB | `shell.rs:23-24` |
| `MAX_FILE_BYTES` | 10 MiB | `paths.rs:15` |
| `DEFAULT_LIMIT` (read_file) | 2000 lines | `read_file.rs:6` |
| `MAX_INPUT_BYTES` / `HINT_SCAN_LINE_LIMIT` / `MAX_DIFF_BYTES` | 1 MiB / 200 / 64 KiB | `str_replace.rs:9-10`, `str_replace.rs:140` |
| `MAX_ITEMS` / `MAX_TEXT_CHARS` | 50 / 200 | `todo.rs:20-21` |
| `MAX_SOURCE_BYTES` / `MAX_FINAL_RAW_BYTES` / `DEFAULT_MAX_DIM` / `MIN_MAX_DIM` / `MAX_MAX_DIM` / `MAX_PIXELS` / `MAX_DECODER_ALLOC` / `FETCH_TIMEOUT` / `MEDIA_GET_AUTH_EXPIRY_SECS` | 20 MiB / 3 MiB / 1568 / 64 / 2048 / 64 Mpx / 256 MiB / 10 s / 600 s | `view_image.rs:31-53` |
| `rg` caps | 1 MiB line / 50 KiB / 2000 lines / context 100 / depth 50 | `rg.rs:5-9` |
| `tree` caps | 50 KiB / 2000 lines / depth 50 / 10 MiB per-file | `tree.rs:6-9` |
| `rg` fallback ignore list | `target`, `node_modules`, `dist`, `build` | `rg.rs:375` |
| `detect_stack` marker list | 9 filenames | `shell.rs:95-105` |
| shim symlink names | `rg`, `tree`, `buzz`, `git-credential-nostr`, `git-sign-nostr` | `shim.rs:33-39` |

#### Parsed-but-unused / partially-used variables

- `BUZZ_AUTH_TAG` is read at exactly one site (`view_image.rs:341`) and applied
  only to relay-hosted `/media/` fetches. It is not used by any other tool in this
  crate even though `buzz-acp` injects it (`crates/buzz-acp/src/lib.rs:4170-4180`)
  and `buzz-agent` passes it through (`crates/buzz-agent/src/mcp.rs:63`). It reaches
  the bundled `buzz` CLI and `git-credential-nostr` only by env inheritance.
- `BUZZ_PRIVATE_KEY` and `BUZZ_RELAY_URL` are read for presence at `shell.rs:78`
  purely to decide whether to append one sentence to the bootstrap string; the
  values are discarded there.
- `GIT_CONFIG_COUNT` composition (`shim.rs:199-204`) is documented as reachable
  only when dev-mcp is run directly, because `buzz-agent` clears the env
  (`shim.rs:174-177`) — dead in the primary launch path.

#### `.env.example` coverage

`.env.example` documents `BUZZ_PRIVATE_KEY` (`:125`), `BUZZ_RELAY_URL` (`:130`),
and the `BUZZ_ACP_PRIVATE_KEY` → `BUZZ_PRIVATE_KEY` rename (`:226`).

Missing from `.env.example` entirely:

| Variable | Why it matters |
|---|---|
| `BUZZ_SHELL` | the only supported way to change the interpreter, and the documented Windows escape hatch in the resolver's own error text (`shell.rs:455-457`); also part of `buzz-agent`'s `WINDOWS_SHELL_RESOLUTION_ENV` public contract (`crates/buzz-agent/src/lib.rs:22-30`) |
| `GIT_BASH` | Windows fallback override (`shell.rs:424`) |
| `NOSTR_PRIVATE_KEY` | drives the entire ephemeral git-identity feature (`shim.rs:54`) and is on `buzz-agent`'s passthrough list (`crates/buzz-agent/src/mcp.rs:60`) |
| `BUZZ_AUTH_TAG` | referenced only indirectly at `.env.example:226`; never given its own entry despite being injected by `buzz-acp` |


## Module: buzz-cli — dispatch, relay client & validation (`crates/buzz-cli/src`)
### Aspect: Configuration

#### Environment variables read by this group

| Var | Read site | Default | Effect | `.env.example` | `AGENTS.md` | `crates/buzz-cli/README.md` |
|---|---|---|---|---|---|---|
| `BUZZ_RELAY_URL` | clap `env` (`lib.rs:81`) | `http://localhost:3000` | relay base URL; `ws/wss` normalized to `http/https` (`client.rs:1291-1297`) | yes, but as `ws://localhost:3000` under the ACP section (`.env.example:130`) | yes (`AGENTS.md:162`) | yes ("defaults to http://localhost:3000") |
| `BUZZ_PRIVATE_KEY` | clap `env` (`lib.rs:85`) | none — required for every group except `pack` (`lib.rs:1746-1748`) | the identity; hex or nsec | yes (`.env.example:125`) | yes (`AGENTS.md:162`) | yes (auth table) |
| `BUZZ_AUTH_TAG` | clap `env` (`lib.rs:89`) | none; empty string treated as unset (`lib.rs:1755`) | NIP-OA owner attestation: verified locally, injected into every signed event, sent as `x-auth-tag` | **no** (`grep -c 'BUZZ_AUTH_TAG' .env.example` → 0) | yes (`AGENTS.md:162`) | no |
| `BUZZ_AUTH_TAG` (second read) | `std::env::var` (`client.rs:991`, `client.rs:1271`) | — | appends a "may be stale or revoked" hint to 403 errors | as above | no | no |
| `BUZZ_TIMEOUT_SECS` | `env_duration_secs` (`client.rs:548`) | 30 s; `0` and unparseable fall back (`client.rs:140-150`) | per-request total timeout | **no** | **no** | **no** |
| `BUZZ_CONNECT_TIMEOUT_SECS` | `env_duration_secs` (`client.rs:549`) | 15 s; same fallback rule | TCP connect timeout | **no** | **no** | **no** |

The two timeout vars are documented **only** in a Rust doc comment
(`client.rs:535-540`). Repo-wide check:
`grep -rn 'BUZZ_TIMEOUT_SECS\|BUZZ_CONNECT_TIMEOUT_SECS' --include='*.md' --include='*.toml' --include='*.example' .`
(excluding `target/`) returns hits only in `client.rs` — zero documentation
matches. They are the CLI's only operational tuning knobs and an operator cannot
discover them without reading the source.

One further env var is read by the sibling-owned command layer and worth naming
so the configuration picture is complete:
`BUZZ_ACP_ALLOWED_CHANNEL_ADD_POLICIES` (`commands/channels.rs:1021`). Nothing
in the six in-scope files reads it.

The double read of `BUZZ_AUTH_TAG` is a genuine configuration inconsistency: the
403 hint keys off the *environment* rather than the *effective* value, so
`--auth-tag <json>` with no env var set produces no hint, and an env var
overridden by `--auth-tag` still produces one.

#### CLI flags at this level

| Flag | Type | Default | Read site | Documented |
|---|---|---|---|---|
| `--relay` | `String` | `http://localhost:3000` | `lib.rs:80-82` | `lib.rs:66` long_about, README |
| `--private-key` | `Option<String>` | none | `lib.rs:84-85` | `lib.rs:67`, README |
| `--auth-tag` | `Option<String>` | none | `lib.rs:87-89` | `lib.rs:68` |
| `--format` | `OutputFormat` | `json` | `lib.rs:92-93` | `AGENTS.md:192-193` |
| `-h`/`--help` | flag | — | clap builtin | — |

Precedence is clap's standard "explicit arg beats env" — verified empirically:
with `BUZZ_RELAY_URL=http://127.0.0.1:9` and `--relay notaurl`, the failure comes
from `notaurl` (`builder error: relative URL without a base`), proving the flag
won.

#### Parsed but not read

`--format` is parsed for every invocation but forwarded to only 5 of 21
dispatchers (`lib.rs:1772`, `:1773`, `:1778`, `:1780`, `:1790`). For the other 16
groups — `agents`, `canvas`, `reactions`, `emoji`, `dms`, `workflows`, `social`,
`notes`, `repos`, `patches`, `issues`, `pr`, `media`, `upload`, `mem`, `pack` — it
is accepted and silently discarded. `grep -n 'cli.format' lib.rs` returns exactly
those five lines.

`--relay` is also parsed on the `pack` path and then never used, because `pack`
short-circuits before `BuzzClient::new` (`lib.rs:1734-1743`) — harmless, but it
means `buzz --relay <garbage> pack validate .` succeeds.

#### Configuration that only exists as constants

None of the following can be changed without a rebuild:

| Constant | Value | Site |
|---|---|---|
| `RETRY_MAX_ATTEMPTS` | 3 | `client.rs:122` |
| `RETRY_BASE_SECS` | `[0.5, 1.5]` s jitter ceilings | `client.rs:126` |
| `RETRY_IN_MAX_SECS` | 30 s cap on relay `retry in Ns` hints | `client.rs:130` |
| `QUERY_PAGE_SIZE` | 500 events/page | `client.rs:498` |
| upload timeouts | 120 s image, 600 s video | `client.rs:1141-1145` |
| download timeout | 120 s | `client.rs:1233` |
| WS publish budget | 75 s | `client.rs:1084` |
| Blossom auth expiry | 600 s get; 600/3600 s upload | `client.rs:329-330`, `:359-363` |
| `ALLOWED_MIMES` | jpeg, png, gif, webp, mp4 | `client.rs:64-71` |
| `MAX_IMAGE_BYTES` / `MAX_VIDEO_BYTES` | 50 MiB / 500 MiB | `client.rs:73-76` |
| `MAX_CONTENT_BYTES` / `MAX_DIFF_BYTES` | 64 KiB / 60 KiB | `validate.rs:4`, `:7` |
| agent draft caps | 120 name / 20,000 prompt / 300 other chars | `agent_management.rs:10-11`, `:83-85` |

#### No config file, no profile

This layer reads no file for configuration: `grep -rn 'dirs::\|home_dir\|config.toml\|\.buzzrc' lib.rs client.rs validate.rs error.rs agent_management.rs main.rs`
returns zero matches. The `dirs` dependency (`Cargo.toml:70-72`) is used only by
`commands/channel_templates.rs` to locate the desktop app's
`channel-templates.json`, which `channels create --template/--templates-file`
consumes (`lib.rs:565-575`) — that is the only file-based configuration reachable
from the CLI, and it is sibling-owned.

#### Documentation drift on configuration

1. `.env.example:130` presents `BUZZ_RELAY_URL=ws://localhost:3000` while
   `--relay`'s help says "Relay URL (http:// or https://)" (`lib.rs:80`). Both
   work — `normalize_relay_url` converts (`client.rs:1291-1297`) — but the flag
   help understates what is accepted, and `.env.example` documents the var only
   inside the "ACP (Agent Communication Protocol — buzz-acp harness)" block
   (`.env.example:113-130`), with no buzz-cli section at all.
2. `BUZZ_AUTH_TAG` is absent from `.env.example` even though `AGENTS.md:161-164`
   names it as harness-injected and this group both verifies and transmits it.
3. `Cargo.toml:19` credits clap's env feature with "(BUZZ_API_TOKEN
   auto-wired)"; that variable is read nowhere in this crate
   (`grep -rn 'BUZZ_API_TOKEN' crates/buzz-cli/src` → zero matches) and belongs
   to `buzz-acp`/`buzz-workflow`.
4. `crates/buzz-cli/TESTING.md:37-73` still describes a token/scope
   configuration model (`BUZZ_REQUIRE_AUTH_TOKEN=false`,
   `buzz-admin mint-token --scopes …`, a per-command scope table) that this layer
   contradicts in code: "The keypair IS the identity — no tokens, no other auth"
   (`lib.rs:1745-1746`). The same section points at a stale checkout path
   (`cd REPOS/buzz-nostr`, `TESTING.md:26`).
5. The `x-auth-tag` HTTP header this group sends (`client.rs:616-621`) appears in
   no markdown doc: `grep -rln 'x-auth-tag' --include='*.md' .` (excluding
   `.aidlc/`) returns nothing.


## Module: buzz-cli — channel, message & social commands (`crates/buzz-cli/src/commands`)
### Aspect: Configuration
These ten files are almost entirely *flag*-configured. They read exactly **one**
environment variable directly (`channels.rs:1021`); everything else arrives as a
clap argument, a compile-time constant, or one local JSON file. Proof of the
absence: `grep -nE 'env::|env!|var_os|vars\(\)|option_env' channels.rs notes.rs
messages.rs emoji.rs social.rs channel_templates.rs reactions.rs dms.rs feed.rs
mod.rs` returns three lines, all in `channels.rs` — the production read at
`:1021` and `set_var`/`remove_var` inside the test module at `:1363`/`:1366`.
`grep -nE 'dirs::|home_dir|XDG|std::env::current_dir'` over the same ten files
returns one production line, `channel_templates.rs:73`.
#### Environment variables read inside the ten files
| Var | Read site | Default when unset | Parse rules | `.env.example` | `AGENTS.md` | `buzz-cli/README.md` |
|---|---|---|---|---|---|---|
| `BUZZ_ACP_ALLOWED_CHANNEL_ADD_POLICIES` | `channels.rs:1021` (`cmd_set_add_policy`) | absent ⇒ **no restriction**; every policy allowed | `split(',')` → `str::trim` → drop empty segments (`channels.rs:1022-1026`). An empty/whitespace-only value yields an empty list, which is also treated as "no restriction" (`channels.rs:1027`). No case folding, no validation that the listed names are real policies. | **No** | **No** | **No** |
Verifying the lead: the site, default, and parse rules above are all as
described. On the documentation half, `grep -rn 'BUZZ_ACP_ALLOWED_CHANNEL_ADD_POLICIES'`
across the repo returns only `channels.rs` (lines 1021, 1030, 1294, 1305, 1349,
1363, 1366) plus this reverse-engineering corpus. `.env.example` never mentions
it (its BUZZ_ACP block is around `.env.example:121-130`, listing only
`BUZZ_PRIVATE_KEY` and `BUZZ_RELAY_URL`), and neither does `AGENTS.md` or
`crates/buzz-cli/README.md`. It is a **completely undocumented deployment
control** whose absence silently means "permit everything", and
`channels.rs:1014-1019` documents in-code that it is a client-side courtesy
gate only — a direct kind:10100 submission bypasses it.
#### Environment variables that reach these files indirectly
Set on `Cli` in `lib.rs` and threaded in as an already-constructed `BuzzClient`,
so none of the ten files parse them:
| Var | Declaration | Default | Consumed in scope via |
|---|---|---|---|
| `BUZZ_RELAY_URL` | `lib.rs:79` (`#[arg(long, env = …, default_value = "http://localhost:3000")]`) | `http://localhost:3000` | every `client.query` / `client.submit_event` call |
| `BUZZ_PRIVATE_KEY` | `lib.rs:82` | none — hard error at `lib.rs:1744-1746` | `client.keys().public_key()` (`channels.rs:33`, `messages.rs`, `dms.rs:9`, `feed.rs:15`, `emoji.rs:101`, `notes.rs:169`, `reactions.rs:43`) |
| `BUZZ_AUTH_TAG` | `lib.rs:85` | unset | `client.auth_tag_owner_hex()` — the template owner invariant at `channels.rs:646-648` |
Defaults-vs-docs disagreement: `AGENTS.md § Agent CLI` and `.env.example:121`
both show `BUZZ_RELAY_URL=ws://localhost:3000`, but the CLI's own default and
its `long_about` (`lib.rs:70`) are `http://localhost:3000`. The `ws://` form is
the relay/harness convention; the CLI speaks HTTP to `POST /query` and
`POST /events`. A reader following `AGENTS.md` verbatim gets a scheme the CLI
then has to normalize (`client::normalize_relay_url`, `lib.rs:1733`). Not a bug,
but the docs and the code disagree on the literal default string.
#### `--format`: declared non-global, honored by 4 of 26 read commands
`--format` is declared on the top-level `Cli` struct at `lib.rs:92-94` with
`#[arg(long, value_enum, default_value = "json")]` — **no `global = true`**.
Confirmed empirically against the prebuilt `target/debug/buzz`:
`buzz channels list --format compact` → `{"error":"user_error","message":"error:
unexpected argument '--format' found …"}`, while `buzz --format compact channels
list` parses and fails later on the missing key. So `AGENTS.md`'s instruction
("`--format compact` is a **global** flag — it goes before the subcommand") is
right about *position* and loose about *mechanism*: it is a top-level-only
argument, not a clap global.
Forwarding, from `lib.rs:1771-1791`:
| Dispatcher | Receives `&cli.format`? | Site |
|---|---|---|
| `messages::dispatch` | yes | `lib.rs:1772` |
| `channels::dispatch` | yes | `lib.rs:1773` |
| `feed::dispatch` | yes | `lib.rs:1780` |
| `channels::dispatch_canvas` | **no** | `lib.rs:1774` |
| `reactions::dispatch` | **no** | `lib.rs:1775` |
| `emoji::dispatch` | **no** | `lib.rs:1776` |
| `dms::dispatch` | **no** | `lib.rs:1777` |
| `social::dispatch` | **no** | `lib.rs:1781` |
| `notes::dispatch` | **no** | `lib.rs:1782` |
Within the three dispatchers that do receive it, forwarding is again partial:
| Command | Honors `--format compact`? | Evidence |
|---|---|---|
| `messages get` | yes | `messages.rs:300` → `format_events` (`messages.rs:242-262`) |
| `messages thread` | yes | `messages.rs:336` |
| `messages search` | yes | `messages.rs:383` |
| `feed get` | yes | `feed.rs:45-65` (inline projection) |
| `channels list` | yes | `channels.rs:95-109` — but a *different* compact shape (`channel_id`, `name`) |
| `channels get` | **no** | `cmd_get_channel` takes no format param (`channels.rs:224`) |
| `channels search` | **no** | `cmd_search_channels` takes no format param (`channels.rs:119-125`); `dispatch` drops it at `channels.rs:1083-1088` |
| `channels members` | **no** | `channels.rs:244-247` |
| `canvas get` | **no** | `channels.rs:262` |
| `reactions get`, `emoji list`/`export`, `dms list`, `social event`/`notes`/`contacts`/`list`, `notes get`/`ls` | **no** | none of these functions has an `OutputFormat` parameter |
Net effect: `buzz --format compact <anything except messages get/thread/search,
feed get, channels list>` is accepted, exits 0, and silently emits full JSON.
There is no warning and no test — `grep -n 'format_events' messages.rs` returns
only the definition (`:242`) and three production call sites (`:300`, `:336`,
`:383`), all above the `#[cfg(test)]` boundary at `messages.rs:877`.
#### Local filesystem configuration
Only `channels create --template` touches the filesystem for config.
| Item | Value | Site |
|---|---|---|
| Override flag | `--templates-file <PATH>`, taken verbatim, no existence check at resolve time | `lib.rs:568-571`, `channel_templates.rs:70-72` |
| Default path | `dirs::data_dir()` / `xyz.block.buzz.app` / `templates` / `channel-templates.json` | `channel_templates.rs:73-79` |
| Bundle identifier constant | `PROD_BUNDLE_IDENTIFIER = "xyz.block.buzz.app"` | `channel_templates.rs:17` |
| Missing-file behavior | `CliError::NotFound` naming the path and suggesting `--templates-file` | `channel_templates.rs:83-89` |
| Parse failure | `CliError::Other` (exit 4), not `Usage` | `channel_templates.rs:94-95` |
| Template lookup | exact match after `to_ascii_lowercase()` on both sides | `channel_templates.rs:101-110` |
| Store re-read | `load_templates` runs **twice** on the not-found path — once in `find_template`, once in `available_names` | `channel_templates.rs:102`, `channel_templates.rs:114` |
Configuration gap worth flagging: the constant covers only the **production**
bundle id. `desktop/src-tauri/tauri.dev.conf.json:2` sets
`xyz.block.buzz.app.dev`, so templates authored in a dev desktop build are
invisible to `buzz channels create --template` unless `--templates-file` is
passed. The doc comment at `channel_templates.rs:29-31` acknowledges the
override is "useful for the dev store or tests" but the dev path itself is
nowhere derived, and neither `AGENTS.md` nor `crates/buzz-cli/README.md`
mentions channel templates at all (`grep -n 'templates-file\|channel-templates'
crates/buzz-cli/README.md` → zero matches).
Consumed fields of `channel-templates.json`, with defaults supplied by serde:
| Field | Default | Site |
|---|---|---|
| `name` | required | `channel_templates.rs:21` |
| `description` | `None` | `channel_templates.rs:22-23` |
| `channel_type` | `"stream"` via `default_channel_type` | `channel_templates.rs:24-25`, `:57-59` |
| `visibility` | `"open"` via `default_visibility` | `channel_templates.rs:26-27`, `:61-63` |
| `canvas_template` | `None` | `channel_templates.rs:28-29` |
| `agents.personas[].personaId` | `[]` | `channel_templates.rs:30-31`, `:38-41` |
| `agents.teams[].teamId` | `[]` | `channel_templates.rs:43-47` |
The CLI's record is a deliberate narrowing of
`desktop/src-tauri/src/templates/types.rs:5-21`, documented at
`channel_templates.rs:1-8`. Dropped fields: `id`, `is_builtin`, `created_at`,
`updated_at`, and per-entry `runtime` / `model` / `role` / `backend`
(`types.rs:35-43`, `:49-55`). The dropped `role` is harmless in practice —
`channels.rs:748-751` hardcodes `MemberRole::Bot`, and desktop's
`useApplyTemplate.ts:101` and `:126` also hardcode `role: "bot"`, so both
surfaces ignore the field identically. Placeholder substitution in
`canvas_template` supports exactly two tokens, `{channel.name}` and
`{template.name}` (`channels.rs:726-728`); any other `{…}` is emitted literally.
#### CLI flags with defaults, clamps, or silently-ignored values
| Flag | Declared default | Effective default / clamp | Sites |
|---|---|---|---|
| `channels list --limit` | none (`Option<u32>`), doc comment says `[default: 500]` | `unwrap_or(500)`, no cap | `lib.rs:514-516`, `channels.rs:32` |
| `channels search --limit` | `default_value_t = 1000` | passed straight to `query_paginated`, **no clamp** | `lib.rs:538-539`, `channels.rs:131` |
| `channels create --ttl` | none | must be `> 0` and fit `i32` | `lib.rs:559-561`, `channels.rs:820-830` |
| `messages get --limit` | none | `unwrap_or(50).min(200)` | `lib.rs:443-445`, `messages.rs:271` |
| `messages get --kinds` | none | defaults to `[9, 40002, 40008, 45001, 45003]`; override only if ≥1 segment parses | `lib.rs:451-453`, `messages.rs:274-285` |
| `messages thread --limit` | none | `unwrap_or(100).min(500)` | `messages.rs:312` |
| `messages thread --depth-limit` | none | passed through as the relay extension field `depth_limit` | `messages.rs:322-324` |
| `messages search --limit` | none | `unwrap_or(20).min(100)` | `messages.rs:352` |
| `feed get --limit` | none | `unwrap_or(20).min(50)` | `feed.rs:17` |
| `feed get --types` | none | validated against `VALID_FEED_TYPES`; sent as relay extension `feed_types` | `feed.rs:6`, `:29-40` |
| `dms list --limit` | none | `unwrap_or(50).min(200)` | `dms.rs:10` |
| `social notes --limit` | none | `unwrap_or(50).min(100)` | `social.rs:94` |
| `social list --limit` | not exposed | hardcoded `10` | `social.rs:201` |
| `notes ls --limit` | none | `unwrap_or(50).min(200)` | `notes.rs:677` |
| `notes ls --author` | `default_value = "me"` (clap) | `unwrap_or("me")` again in the handler — unreachable | `lib.rs:1069-1071`, `notes.rs:678` |
| `notes get --name` fan-out | not exposed | hardcoded `limit: 50` cross-author | `notes.rs:191` |
| `emoji export --scope` | `default_value = "own"` | `own` \| `workspace` | `lib.rs:757-759` |
| `emoji import --replace` / `--dry-run` | `default_value_t = false` | merge, publish | `lib.rs:765-770` |
| `notes set --clear-tags` / `--allow-empty` | `default_value_t = false` | carry-forward, refuse-empty | `lib.rs:1035-1049` |
Two silent-ignore behaviors in this table are worth calling out explicitly:
- `messages get --kinds abc` → `filter_map(…parse().ok())` yields an empty list,
  the `if !kind_list.is_empty()` guard at `messages.rs:282` fails, and the
  request goes out with the **default** kinds. Exit 0, no warning. Mixed input
  like `--kinds 9,abc` silently degrades to `[9]`.
- `notes ls --author` can never be `None` at the handler (clap always supplies
  `"me"`), so the `unwrap_or("me")` at `notes.rs:678` is dead and the flag's
  `Option<String>` type is misleading.
#### Compile-time constants behaving like configuration
| Constant | Value | Site | What it configures |
|---|---|---|---|
| `KIND_LONG_FORM` | `30023` | `notes.rs:38` | every `notes` filter and the `set`/`rm` builders. **Redeclared** — `buzz_core::kind::KIND_LONG_FORM` already exists at `kind.rs:66` |
| `SLUG_MAX_LEN` | `80` | `notes.rs:42` | `parse_slug` upper bound (`notes.rs:55-60`) |
| `SET_STDIN_MAX_BYTES` | `1_048_576` (1 MiB) | `notes.rs:485` | `notes set --content -` stdin cap (`notes.rs:502-508`) |
| `STDIN_MAX_BYTES` | `10_000_000` | `emoji.rs:159` | `emoji import` stdin cap (`emoji.rs:171`) |
| `MAX_CONTENT_BYTES` | `65_536` | `validate.rs:4` | `messages send` / `messages edit` (`messages.rs:480`, `:705`) |
| `MAX_DIFF_BYTES` | `61_440` (60 KiB) | `validate.rs:7` | `messages send-diff` truncation at a hunk boundary (`messages.rs:606`) |
| `MENTION_CAP` | `50` | `buzz-sdk/src/mentions.rs:38` | mention-tag ceiling for `messages send` (`messages.rs:536`) |
| `CUSTOM_EMOJI_SET_D_TAG` | re-export of `buzz_sdk::CUSTOM_EMOJI_SET_D_TAG` | `emoji.rs:9` | `#d` on every kind:30030 filter (`emoji.rs:80`, `:103`, `:221`) |
| `VALID_FEED_TYPES` | `["mentions","needs_action","activity","agent_activity"]` | `feed.rs:6` | `feed get --types` allowlist |
| `PROD_BUNDLE_IDENTIFIER` | `"xyz.block.buzz.app"` | `channel_templates.rs:17` | template store path |
| `QUERY_PAGE_SIZE` | `500` | `client.rs:498` | page size for `query_paginated` / `query_all`, used by `channels.rs:41`, `:57`, `:62`, `:131`, `:452` |
Hardcoded kind-integer literals are the module's largest de-facto config
surface. `channels.rs` embeds `39002` (`:38`, `:250`), `39000` (`:54`, `:62`,
`:131`, `:227`) and `40100` (`:265`); `messages.rs` embeds `39002` (`:139`),
`0` (`:150`, `:407`) and the read sets at `:276`, `:320`, `:361`;
`reactions.rs` embeds `7` (`:45`, `:83`); `dms.rs` embeds `41001` (`:12`) and
`41010` (`:69`); `social.rs` embeds `1` (`:97`) and `3` (`:118`). This
contradicts `AGENTS.md § Common Gotchas #1` ("All kinds defined in
`buzz-core/src/kind.rs`") — the constants exist (`kind.rs:447-453` for the DM
kinds alone) and are used correctly by `dms.rs:104`, `emoji.rs:80`,
`channels.rs:3`, `social.rs:1-5`, so the module knows the pattern and applies
it inconsistently.
Three different stdin caps coexist for the same "read a body from `-`" job:
unbounded for `messages send` / `canvas set` (`validate.rs:168-181` has no cap;
`messages send` merely rejects the result afterwards at `messages.rs:480`, and
`canvas set` at `channels.rs:1050` never checks size at all), 1 MiB for
`notes set`, 10 MB for `emoji import`. None of the three is documented.
#### Stated uncertainties
- `channels search --limit 1000` (`lib.rs:539`) is a *fetch* budget, not a
  result count — the name-matching happens client-side at `channels.rs:134-138`.
  Whether the relay itself clamps that limit on `POST /query` is out of scope
  here; I did not trace it.
- `messages thread --depth-limit` and `social notes --before-id` are relay
  *extension* fields injected into the filter JSON (`messages.rs:323`,
  `social.rs:106`). `crates/buzz-relay/src/api/bridge.rs:283` and `:263` do read
  them, so they are real, but I did not verify their end-to-end semantics.
- The `--format` empirical checks above ran against `target/debug/buzz` built
  2025-07-26, which predates the module docs in this corpus. Both facts also
  follow directly from the source I read (`lib.rs:92-94` has no `global`;
  `lib.rs:476-489` declares no `kinds` on `messages search`), so I am confident
  in them, but they are not from a fresh build.


## Module: buzz-cli — repo, agent, memory & moderation commands (`crates/buzz-cli/src/commands`)
### Aspect: Configuration

Scope: `mem.rs`, `agents.rs`, `repos.rs`, `users.rs`, `pr.rs`, `patches.rs`,
`workflows.rs`, `issues.rs`, `moderation.rs`, `pack.rs`, `upload.rs`. `lib.rs`,
`client.rs` and `validate.rs` are cited only where a value these files consume is
declared or read there.

#### Environment variables

None of the eleven files calls `std::env::var` directly:
`grep -n 'env::var' ` across all eleven returns zero matches. Every env var reaches
them either through the clap `env =` attribute on the root `Cli` struct or through
`BuzzClient`.

| Var | Default | Read site | Reaches these files via | `.env.example` | `AGENTS.md` | `buzz-cli/README.md` |
|---|---|---|---|---|---|---|
| `BUZZ_RELAY_URL` | `http://localhost:3000` (`lib.rs:81`) | clap `env =` (`lib.rs:81`), normalized by `client::normalize_relay_url` (`client.rs:1291-1296`) at `lib.rs:1734` | `client.relay_url()` | yes but **as `ws://localhost:3000`** (`.env.example:130`), and only inside the ACP-harness section | yes (`AGENTS.md:164`, `:166`) | yes (`README.md:29`) |
| `BUZZ_PRIVATE_KEY` | none — required (`lib.rs:85`, enforced `lib.rs:1746-1748`) | clap `env =` (`lib.rs:85`) | `client.keys()` — used in every file except `pack.rs`/`upload.rs` | yes, ACP section only (`.env.example:125`) | yes (`AGENTS.md:164`, `:166`) | yes (`README.md:15`, `:19`) |
| `BUZZ_AUTH_TAG` | none — optional (`lib.rs:89`) | clap `env =` (`lib.rs:89`); parsed + signature-verified at `lib.rs:1755-1765` | `client.auth_tag_owner_hex()` at `mem.rs:38`, `agents.rs:159` | **no** (`grep -c 'BUZZ_AUTH_TAG' .env.example` → `0`) | yes (`AGENTS.md:164`) | **no** — the auth table lists only `BUZZ_PRIVATE_KEY` (`README.md:13-15`) |
| `BUZZ_TIMEOUT_SECS` | `30` s (`client.rs:548`) | `env_duration_secs` (`client.rs:140-146`) | every relay call | **no** | **no** | **no** |
| `BUZZ_CONNECT_TIMEOUT_SECS` | `15` s (`client.rs:549`) | `env_duration_secs` (`client.rs:140-146`) | every relay call | **no** | **no** | **no** |

`grep -rn 'BUZZ_TIMEOUT_SECS\|BUZZ_CONNECT_TIMEOUT_SECS' .env.example AGENTS.md crates/buzz-cli/README.md CONTRIBUTING.md`
returns zero matches — both timeout knobs are documented only in the `BuzzClient::new`
doc comment (`client.rs:537-538`).

Two silent-ignore behaviours in that read path: `env_duration_secs` discards a value
that fails `parse::<u64>()` **and** discards `0`, falling back to the default rather
than erroring (`client.rs:140-146`; the doc comment at `client.rs:139` states `0` is
treated as invalid deliberately, to prevent disabling timeouts). So
`BUZZ_TIMEOUT_SECS=abc` and `BUZZ_TIMEOUT_SECS=0` both silently mean 30 s. All four
branches (valid / non-numeric / zero / unset) are pinned by the single test
`env_duration_secs_parsing` (`client.rs:1554-1569`).

`BUZZ_AUTH_TAG` is also read a second time, out of band, purely to improve a 403
message (`client.rs:991`, `client.rs:1271`) — `std::env::var("BUZZ_AUTH_TAG").is_ok()`.
That path bypasses `--auth-tag`, so a caller who passes the tag as a flag rather than an
env var gets the less helpful error text.

Documentation contradiction worth flagging: `.env.example:130` shows
`BUZZ_RELAY_URL=ws://localhost:3000` while the CLI flag default is
`http://localhost:3000` (`lib.rs:81`). Functionally both work —
`normalize_relay_url` rewrites `ws://`→`http://` and `wss://`→`https://`
(`client.rs:1292-1293`) — but the documented value never matches the documented
default, and `.env.example`'s comment scopes the var to the ACP harness
(`.env.example:113-121`), not to `buzz`.

#### Global flags and the `--format` matrix

`--format` is declared on the root parser **without `global = true`**
(`lib.rs:92-93`, `default_value = "json"`). It is therefore accepted only before the
subcommand, which matches the usage `AGENTS.md § Agent CLI` documents
(`buzz --format compact channels list`). `lib.rs:1770-1791` forwards `&cli.format` to
only five dispatchers; the rest never receive it.

| In-scope dispatcher | Receives `OutputFormat`? | Honors it? | Site |
|---|---|---|---|
| `users::dispatch` | yes | **yes** — `Compact` projects `{pubkey, display_name}` in both `cmd_get_users` (`users.rs:63-74`) and `search_by_name` (`users.rs:134-145`) | `lib.rs:1778`, `users.rs:307-311` |
| `moderation::dispatch` | yes | **no** — bound as `_format` and never read (`moderation.rs:133-137`) | `lib.rs:1790` |
| `agents::dispatch` | no | n/a | `lib.rs:1771` |
| `workflows::dispatch` | no | n/a | `lib.rs:1779` |
| `repos::dispatch` | no | n/a | `lib.rs:1783` |
| `patches::dispatch` | no | n/a | `lib.rs:1784` |
| `issues::dispatch` | no | n/a | `lib.rs:1785` |
| `pr::dispatch` | no | n/a | `lib.rs:1786` |
| `upload::dispatch_media` / `dispatch` | no | n/a | `lib.rs:1787-1788` |
| `mem::dispatch` | no | n/a — has its own per-subcommand `--json` bool instead (`lib.rs:1550-1552`, honored at `mem.rs:193`, `mem.rs:262-270`) | `lib.rs:1789` |
| `pack` | no — dispatched before the client is built (`lib.rs:1737-1742`) | n/a | `lib.rs:1737` |

Net effect: `--format compact` is **parsed and silently ignored** for 10 of the 11
in-scope groups. `moderation` is the sharpest case because it accepts the argument
and discards it at the function boundary; the others never plumb it. `mem ls --json`
is a redundant second output-format control that the global flag does not reach.

#### Flag defaults, clamps, and values with no client-side validation

| Flag | Type / default | Site | Notes |
|---|---|---|---|
| `mem ls --json` | `bool`, `false` | `lib.rs:1551-1552` | see matrix above |
| `mem set --allow-empty` | `bool`, `false` | `lib.rs:1581-1582` | gates the zero-byte-stdin refusal (`mem.rs:339-346`); a literal `""` positional is always allowed (`mem.rs:348-349`) |
| `mem patch --no-base-hash` | `bool`, `false` | `lib.rs:1603-1604` | mutually exclusive with `--base-hash` and one of the two is mandatory (`mem.rs:551-567`) — enforced in code, **not** by a clap `conflicts_with` |
| `mem patch --base-hash` | `Option<String>` | `lib.rs:1599` | must be exactly 64 ASCII hex (`mem.rs:568-574`), compared case-insensitively (`mem.rs:607`) |
| `mem patch --dry-run` / `--allow-empty` | `bool`, `false` | `lib.rs:1606-1607`, `lib.rs:1609-1610` | |
| `mem ls/get/hash --owner` / `--agent` | `Option<String>` | `lib.rs:1546`, `lib.rs:1549` (plus `:1558`/`:1561`, `:1567`/`:1570`) | mutually exclusive for reads (`mem.rs:60-63`); `--agent` must differ from the CLI identity (`mem.rs:67-71`) |
| `repos protect set --no-force-push` / `--no-delete` / `--require-patch` | `bool`, `default_value_t = false` | `lib.rs:1161`, `:1164`, `:1167` | at least one rule (incl. `--push`) is required — otherwise `build_protection_tag` errors via `parse_protection_tag` (`repos.rs:79-81`; test `protection_set_requires_at_least_one_rule`, `repos.rs:521`) |
| `patches send --root` / `--root-revision` | `bool`, `default_value_t = false` | `lib.rs:1218`, `lib.rs:1221` | no `conflicts_with`, and no runtime check in `patches.rs`; both are handed straight to `GitPatchMeta` (`patches.rs:36-37`) |
| `workflows approve --approved` | `bool`, `default_value_t = true`, `ArgAction::Set` | `lib.rs:914` | **defaults to approve** — omitting the flag grants |
| `workflows runs --limit` | `Option<u32>` → `limit.unwrap_or(20).min(100)` | `workflows.rs:72` | the only client-side clamp in the group |
| `moderation reports --limit`, `moderation audit --limit` | `i64`, `default_value_t = 50` | `lib.rs:1654-1655`, `lib.rs:1728-1729` | no client-side validation; a negative value is interpolated verbatim into the query string (`moderation.rs:110`, `moderation.rs:127`) |
| `moderation ban/timeout --expires-in` | `Option<u64>`, `conflicts_with = "expires_at"` | `lib.rs:1684`, `lib.rs:1708` | `--expires-in` wins when both are somehow set (`moderation.rs:27-32`); `timeout` requires one of them (`moderation.rs:69-70`), `ban` does not (permanent) |
| `moderation reports --status`, `resolve --status/--action` | free-form `String` | `lib.rs:1651-1652`, `lib.rs:1670-1676` | no `value_parser`; the accepted vocabularies exist only in help text and are validated relay-side. `--status` is spliced into a URL query with no escaping (`moderation.rs:110-113`) — the crate's only percent-encoder is `#[cfg(test)]`-gated (`validate.rs:76-77`) and so unavailable to production code |
| `patches status --status`, `issues status --status` | `String` with `value_parser = [...]` | `lib.rs:1263`, `lib.rs:1495` | clap restricts the words, then `patches::parse_status` re-validates (`patches.rs:194-206`). The two clap lists differ (`merged` vs `resolved`) while `parse_status` accepts both as synonyms for kind 1631 |
| `agents archive/unarchive --content` | `String`, `default_value = ""` | `lib.rs:319`, `lib.rs:332` | |
| `repos list --limit`, `patches list --limit`, `pr list --limit`, `issues list --limit` | `Option<u32>` | `lib.rs:1133`, `:1255`, `:1406`, `:1487` | injected into the filter unvalidated (`repos.rs:275-277`, `patches.rs:105-107`, `pr.rs:141-143`, `issues.rs:71-73`); server-side clamping is covered in Business Rules |
| `users get --pubkey` | `Vec<String>`, max 200 | `lib.rs:806-807`, enforced `users.rs:32-34` | the only explicitly-capped repeated flag in the group |
| `pr open/update --body` / `--body-file` | `Option<String>`, `conflicts_with` each other | `lib.rs:1315-1319`, `lib.rs:1369-1373`, `lib.rs:1417-1421` | clap already rejects the pair, so the runtime check in `read_optional_body` (`pr.rs:11-13`) is unreachable in practice — its test calls the function directly (`pr.rs:334`) |
| `media get --output` | `Option<String>`; `-` or absent ⇒ stdout | `lib.rs:1535`, `upload.rs:22-36` | |
| `upload file --file` | `String`, required | `lib.rs:1523` | |
| `pack validate/inspect <path>` | positional `String`, required | `lib.rs:1628`, `lib.rs:1633` | |

`--relay`, `--private-key` and `--auth-tag` are the only other root flags
(`lib.rs:80-90`); all three override their env var per clap precedence, as the
`long_about` states (`lib.rs:68-71`).

#### Local filesystem inputs

| Command | Path source | Resolution | Failure mode |
|---|---|---|---|
| `mem patch --patch-file <path>` | flag (`lib.rs:1593`) | `std::fs::read_to_string(path)` — relative to CWD, no canonicalization (`mem.rs:581-582`) | `Usage` (exit 1) |
| `mem set <slug> -`, `mem patch` (no `--patch-file`) | stdin | `std::io::stdin().take(NIP44_PLAINTEXT_MAX + 1)` (`mem.rs:327-332`, `mem.rs:579`, `mem.rs:583-588`) | oversize ⇒ `Usage`; empty ⇒ `Usage` unless `--allow-empty` (set) / always for patch (`mem.rs:589-595`) |
| `patches send --patch-file <path>` | flag (`lib.rs:1207`) | `validate::read_file_or_stdin` → `std::fs::read_to_string` or stdin on `-` (`validate.rs:178-192`), called at `patches.rs:26` | `Usage` naming the path (`validate.rs:190`) |
| `pr open/update --body-file <path>` | flag | `read_file_or_stdin` via `read_optional_body` (`pr.rs:10-18`) | `Usage`; `--body` + `--body-file` together is also `Usage` (`pr.rs:11-13`) |
| `issues create --content -`, `patches status --content -`, `issues status --content -`, `agents draft-* --system-prompt -`, `workflows create/update --yaml -` | positional/flag value `-` | `validate::read_or_stdin` (`validate.rs:163-175`) — treats a non-`-` value as **literal content**, never a path (`issues.rs:19`, `patches.rs:132`, `issues.rs:95`, `agents.rs:29`, `workflows.rs:106`) | unbounded stdin read (no `.take()`, unlike `mem`) |
| `pack validate/inspect <path>` | positional (`lib.rs:1628`, `lib.rs:1633`) | `Path::new(path)`, existence + is-dir pre-checks (`pack.rs:16-22`, `pack.rs:53-59`), then `buzz_persona::validate::validate_pack` / `resolve::resolve_pack` | `Usage` for missing/non-dir; `Other` for resolve failure (`pack.rs:62-63`) |
| `upload file --file <path>` | flag (`lib.rs:1523`) | delegated to `client.upload_file` (`upload.rs:7`) | handled in `client.rs` |
| `media get --output <path>` | flag | `std::fs::write(path, &bytes)` (`upload.rs:26-27`) | `Other` (exit 4) |

Pack directory layout consumed by `pack.rs` (resolved inside `buzz-persona`, not
here): manifest at `<pack>/.plugin/plugin.json` (`crates/buzz-persona/src/validate.rs:211`,
`:303`), persona files listed in the manifest's `personas` array
(`crates/buzz-persona/src/validate.rs:113`), skills at `<pack>/skills/<name>/SKILL.md`
(`crates/buzz-persona/src/validate.rs:369`, `:392`).

Note the asymmetry: `mem` bounds its stdin reads, every other stdin path in the group
uses the unbounded `read_or_stdin`. And `read_or_stdin` vs `read_file_or_stdin` is a
live footgun — `validate.rs:172-176`'s doc comment and the regression test
`read_file_or_stdin_does_not_treat_path_as_literal_content` (`validate.rs:499`) exist
because `--patch-file` once used the literal-content variant.

#### Compile-time constants that behave like configuration

| Constant | Value | Declared | Consumed here |
|---|---|---|---|
| `engram::NIP44_PLAINTEXT_MAX` | `65_535` | `crates/buzz-core/src/engram.rs:28` | stdin/patch bounds and result-size caps (`mem.rs:322`, `:326-329`, `:562`, `:630-636`) |
| `engram::CORE_SLUG` | `"core"` | `crates/buzz-core/src/engram.rs:20` | `Body::Core` selection (`mem.rs:352-359`, `:640-648`) and the `mem rm core` refusal (`mem.rs:713-716`) |
| `engram::D_TAG_DOMAIN` | `b"agent-memory/v1/d-tag"` | `crates/buzz-core/src/engram.rs:24` | indirectly, via `engram::d_tag(&k_c, slug)` (`mem.rs:148`) — the `d` tag is `HMAC`-derived from the agent↔owner conversation key, not the slug, so it is opaque and per-pair |
| `engram::SLUG_MAX_LEN` | `255` | `crates/buzz-core/src/engram.rs:31` | via `normalize_slug` (`mem.rs:284`, `:322`, `:515`, `:549`, `:712`) |
| `mem ls` filter limit | `5000` literal | `mem.rs:201` | server-clamped — see Business Rules |
| `mem` head-fetch limit | `16` literal | `mem.rs:155` | |
| `users get --name` search limit | `100` literal | `users.rs:93` | |
| `users get --pubkey` cap | `200` literal | `users.rs:32-34` | |
| `workflows runs` default/cap | `20` / `100` literals | `workflows.rs:72` | |
| repo protection-rule ceiling | `50` per repo | enforced in `buzz_core::git_perms::parse_protection_tags`, surfaced at `repos.rs:120-125`; pinned by `protection_update_enforces_repository_rule_limit` (`repos.rs:551`) | |
| `validate::MAX_CONTENT_BYTES` / `MAX_DIFF_BYTES` | `65_536` / `61_440` | `validate.rs:5`, `validate.rs:8` | **not used by any of the eleven** — `grep -n 'validate_content_size\|MAX_CONTENT_BYTES\|MAX_DIFF_BYTES'` across all eleven returns zero matches. Content-size enforcement in this group is either `mem`'s NIP-44 cap or relay-side |
| `client.rs` retry/upload constants (`RETRY_MAX_ATTEMPTS = 3`, `RETRY_BASE_SECS`, `RETRY_IN_MAX_SECS = 30`, `MAX_IMAGE_BYTES = 50 MiB`, `MAX_VIDEO_BYTES = 500 MiB`, `QUERY_PAGE_SIZE = 500`, upload timeouts 600 s video / 120 s other) | `client.rs:122-134`, `:73`, `:76`, `:498`, `:1139-1142` | inherited by every relay call these files make; `upload file` is the only in-scope consumer of the media caps and upload timeouts (`upload.rs:7`) |

`d`-tag conventions written by this group: repo announcements use the raw `repo_id`
(`repos.rs:36-44` reads it back; `buzz_sdk::build_repo_announcement*` writes it),
workflow definitions use the workflow UUID (`workflows.rs:41-44`,
`workflows.rs:169-171`), and workflow **approvals** use
`hex(SHA256(approval_token))` rather than the raw UUID — an easily-missed convention
documented in a one-line comment and implemented at `workflows.rs:204-206`.
Engram `d` tags are the derived opaque value above.

#### Hardcoded kind integers vs `crates/buzz-core/src/kind.rs`

`AGENTS.md § Common Gotchas #1` states all event-kind integers are defined in
`crates/buzz-core/src/kind.rs`. Three of the eleven files import and use those
constants; the rest inline bare integers even though a matching constant exists.

| Site | Bare literal | Existing constant it should use |
|---|---|---|
| `repos.rs:240` (`cmd_get_repo`) | `30617` | `KIND_GIT_REPO_ANNOUNCEMENT` (`kind.rs:545`) — already imported at `repos.rs:1-4` and used at `repos.rs:21` |
| `repos.rs:271` (`cmd_list_repos`) | `30617` | same |
| `pr.rs:129`, `patches.rs:94`, `issues.rs:58` | `"30617:{owner}:{id}"` in the `#a` coordinate | same |
| `pr.rs:110`, `pr.rs:131` | `1618` | `KIND_GIT_PULL_REQUEST` (`kind.rs:551`) |
| `patches.rs:76`, `patches.rs:96` | `1617` | `KIND_GIT_PATCH` (`kind.rs:549`) |
| `issues.rs:39`, `issues.rs:60` | `1621` | `KIND_GIT_ISSUE` (`kind.rs:555`) |
| `workflows.rs:16`, `workflows.rs:41` | `30620` | `KIND_WORKFLOW_DEF` (`kind.rs:382`) — the same file *does* use `buzz_sdk::kind::KIND_WORKFLOW_TRIGGER` at `workflows.rs:176` |
| `workflows.rs:74` | `46001, 46002, 46003` | `KIND_WORKFLOW_TRIGGERED` / `KIND_WORKFLOW_STEP_STARTED` / `KIND_WORKFLOW_STEP_COMPLETED` (`kind.rs:504-508`) |
| `users.rs:258` | `40902` | `KIND_PRESENCE_SNAPSHOT` (`kind.rs:443`) |
| `users.rs:42`, `users.rs:91`, `users.rs:223`, `agents.rs:180` | `0` | `KIND_PROFILE` (`kind.rs:9`) |

Compliant sites, for contrast: `mem.rs:24` imports `KIND_AGENT_ENGRAM`,
`agents.rs:1` imports `KIND_IA_ARCHIVED_LIST`, `repos.rs:3` imports
`KIND_GIT_REPO_ANNOUNCEMENT`. `repos.rs` and `workflows.rs` are internally
inconsistent — constant in one function, literal in the next.

`patches.rs`/`issues.rs` doc strings mention `kind:1630-1633` for statuses
(`lib.rs:1252`, `lib.rs:1489`) but no integer is written in those files; the
status kind is chosen inside `buzz_sdk::build_git_status` from the `GitStatus`
enum (`patches.rs:184`, `issues.rs:139`, `pr.rs:215`).

#### Configuration absent by design

- No config file. `grep -rn 'config\.toml\|\.buzzrc\|dirs::' ` across the eleven
  returns zero matches; `dirs` is a crate dependency
  (`crates/buzz-cli/Cargo.toml`, "locates the desktop app's channel-templates.json
  store for `channels create --template`") but is used only by `channels.rs`, out of
  scope here.
- No per-community or per-channel config: `moderation` derives its tenant purely
  from the relay host, documented at `moderation.rs:14-15` and `lib.rs:1636-1641`.
- `pack.rs` reads no environment or relay configuration at all — it is dispatched
  before `BuzzClient` construction (`lib.rs:1737-1742`), which is why `buzz pack`
  works with no `BUZZ_PRIVATE_KEY` (`lib.rs:1746-1748` would otherwise reject it).

#### Test coverage of configuration behaviour

Covered: `env_duration_secs` parse/zero/absent fallbacks
(`env_duration_secs_parsing`, `client.rs:1554`); the `--base-hash` digest convention
via `sha256_hex_*` vectors (`mem.rs:847`, `:855`, `:863`);
`--owner`/`--agent` resolution and both mutual-exclusion rules
(`resolve_reader_defaults_to_agent_identity` `mem.rs:793`,
`resolve_reader_agent_flag_uses_cli_identity_as_owner` `mem.rs:807`,
`resolve_reader_rejects_owner_with_agent_flag` `mem.rs:821`,
`resolve_reader_rejects_agent_flag_matching_cli_identity` `mem.rs:837`);
`parse_status`'s flag vocabulary (`patches.rs:304`, `:319`).

Not covered: `--format` propagation for any command
(`grep -n 'OutputFormat' ` in test modules across the eleven returns zero matches);
`resolve_expiry`'s `--expires-in`/`--expires-at` precedence (`moderation.rs:26-35`);
`workflows runs`'s `min(…, 100)` clamp (`workflows.rs:72`); the `--approved` default;
the `moderation` URL/query construction; every filesystem-path branch in `pack.rs`
(`grep -n '#\[test\]' crates/buzz-cli/src/commands/pack.rs` → zero matches, as for
`upload.rs`, `workflows.rs`, `issues.rs` and `moderation.rs`).

One caveat on the `mem` test client: `test_client` builds a `BuzzClient` with
`None` for both auth-tag slots (`mem.rs:787-789`), so the
`BUZZ_AUTH_TAG`-derived owner path (`mem.rs:38-41`) is never exercised — only the
explicit `--owner` and `--agent` branches are.

