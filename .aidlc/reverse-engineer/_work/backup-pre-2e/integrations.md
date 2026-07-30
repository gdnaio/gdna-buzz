<!-- Analyzed: 2026-07-25T01:12:08Z | Scope: full project -->
# Integrations

> Status: initialized in Phase 1. External services, protocols, clients, retry/error
> handling, and failure modes are populated per-module during Phase 2 and consolidated in
> Phase 3.

## Summary

Scan-time inventory of external systems: PostgreSQL, Redis, S3/MinIO, AI agent
subprocesses (ACP over stdio), OTLP collector, Prometheus, Web Push services, and optional
iroh mesh peers.

Batch 2a (foundation crates) integrates with **no network service at all**:

| Module | Internal deps | External crates | Network | Filesystem |
|---|---|---|---|---|
| `buzz-core` | **0** | 14 | none | none |
| `buzz-sdk` | `buzz-core` | 6 | **zero endpoints called** | none |
| `buzz-persona` | **0** (leaf) | 4 + 1 dev | none | read-only |
| `buzz-ws-client` | **0** | 8 | WebSocket to one relay | none |

Two findings worth carrying forward:

**The zero-I/O rule on `buzz-core` is convention, not enforcement.** The prohibition is a
comment in the manifest (`crates/buzz-core/Cargo.toml:28`: "NO tokio, NO sqlx, NO redis, NO
axum"). There is no `cargo-deny` ban (`deny.toml:90-92` sets only `multiple-versions` and
`wildcards`), no `[workspace.lints]`, and no CI check. Nothing fails the build if `tokio` is
added to that manifest. The same "documented fence, not compiler fence" pattern is stated
explicitly for the tenant type (`crates/buzz-core/src/tenant.rs:23-30`).

**Two declared dependencies are unused at the code level:**
- `buzz-acp` declares `buzz-persona` (`crates/buzz-acp/Cargo.toml:22`) but contains **zero**
  `buzz_persona` references — so the persona crate's primary stated consumer is not wired.
  The receiving field exists (`crates/buzz-acp/src/config.rs:533-535`
  `persona_env_vars`), but nothing calls `resolve_pack`/`resolve_persona_by_name`.
- `buzz-sdk` declares `serde` (`crates/buzz-sdk/Cargo.toml:14`) with no `use serde` or
  derive in `src/` — needed only transitively via `serde_json`.

Batch 2b maps the actual service integrations. Each service crate touches exactly one
external system, and none calls another service crate — matching the architecture
principle at `ARCHITECTURE.md:97` that subsystems are isolated and only the relay
coordinates them.

| Module | External system | Client | Internal deps |
|---|---|---|---|
| `buzz-db` | PostgreSQL 17 | `sqlx` runtime queries (no offline cache ⇒ no compile-time SQL validation) | `buzz-core` |
| `buzz-search` | PostgreSQL FTS | `sqlx`, fully parameterized (`push_bind` everywhere) | `buzz-core` |
| `buzz-audit` | PostgreSQL | `sqlx` + per-community advisory lock | `buzz-core` |
| `buzz-media` | S3 / MinIO (Blossom) | `rust-s3` | `buzz-core` |
| `buzz-pubsub` | **Redis only** | `deadpool-redis` pool + 3 dedicated `redis::aio::PubSub` sockets | `buzz-core`, `buzz-auth` |
| `buzz-workflow` | Outbound HTTP (`call_webhook`) | HTTP client | `buzz-core`, `buzz-db` |
| `buzz-auth` | none | — | `buzz-core` |

Redis integration detail (`buzz-pubsub`): 8 key families, all community-prefixed except
the operator-global IP rate limit. Four connections minimum per pod — one injected
`deadpool` pool for all commands plus three dedicated stateful pub/sub sockets, because a
pub/sub connection cannot be pooled (`crates/buzz-pubsub/src/lib.rs:19-20`). Reconnect is
an exponential backoff of 1 s → 30 s, implemented three times over
(`crates/buzz-pubsub/src/subscriber.rs:16-19` and two mirrors).

Findings worth carrying forward from 2b:

**Redis is a hard availability dependency for authenticated reads, not just fan-out.** The
rate limiter fails closed — a Redis error becomes `AdmissionError::Unavailable`
(`crates/buzz-relay/src/admission.rs:29-33`) and the relay rejects the request
(`crates/buzz-relay/src/connection.rs:612-621`). Correct for a limiter, but there is no
circuit breaker, no in-process fallback, and no documented operator override, so a Redis
outage degrades to "no reads" rather than "no rate limiting".

**Tenant isolation in Redis is key-prefix naming inside one shared instance** — no logical
db separation, no ACLs. Two subscribers deliberately consume *all* tenants' control
traffic via `buzz:*` patterns (`crates/buzz-pubsub/src/cache_invalidation.rs:27`,
`src/conn_control.rs:30`) and demultiplex by parsing the channel name.

**Cross-pod control messages are unversioned and unauthenticated.** Neither
`CacheInvalidation` nor `ConnControl` carries a version, timestamp, origin-pod id, or
signature, and neither is `#[non_exhaustive]`. A rolling deploy that adds a variant makes
older pods `warn`-and-skip it — for `DisconnectPubkey` that means a ban is not live-enforced
on those pods until restart (`crates/buzz-pubsub/src/conn_control.rs:152-156`).

**One more unused declared dependency**, extending the 2a pattern: `buzz-pubsub` declares
`chrono` (`crates/buzz-pubsub/Cargo.toml:18`) with no `chrono::` path in any source file.
Also, `buzz-admin` and `buzz-conformance` both declare `buzz-pubsub` with no verified call
site.

**An outbound-HTTP gap:** workflow `call_webhook` accepts plain `http://` despite
documentation stating HTTPS, and its condition-evaluation timeout does not cancel the
`spawn_blocking` task it guards.

Agent subprocess, push, and mesh integrations arrive in batches 2c–2d.

### Batch 2c integrations (mesh transport, TLA+ specs, MeshLLM)

- **`iroh` version drift** — the manifest declares `1.0.0-rc.0` while `Cargo.lock:3902-3905`
  resolves 1.0.2.
- **Mesh gossip is unauthenticated** — peer records are accepted without signature or origin
  proof (`crates/buzz-relay-mesh/src/membership.rs:120-153`), and there is no peer eviction.
- **MeshLLM is a distinct, undocumented integration.** The 5 `mesh_*.rs` examples under
  `crates/buzz-relay/examples/` talk to `mesh_llm_sdk` via git dev-dependencies
  (`crates/buzz-relay/Cargo.toml:84-85`) — they are unrelated to `buzz-relay-mesh` despite the
  shared prefix.
- **The TLA+ relationship is documentary, not mechanical.** No build step, test, or CI job
  reads `docs/spec/MultiTenantRelay.tla`; the coupling is doc comments carrying spec line
  numbers, several of which have drifted (`crates/buzz-conformance/src/lib.rs:193` cites
  "line ~720" for `ReadHostFeedRows`, actual `tla:703`; `TRACE_SCHEMA.md:57`, `:69` cite 562
  and 612 against actual 559 and 606).
- **`docs/spec/GitOnObjectStore.tla` is a second, separate spec**, consumed by
  `crates/buzz-relay/src/api/git/cas_publish.rs` — unrelated to `buzz-conformance`.
- **Correction to batch-2b context:** `buzz-admin` does **not** depend on `buzz-conformance`.
  Only the workspace root (`Cargo.toml:125`) and `crates/buzz-relay/Cargo.toml:20` declare it.
- **Neither `ARCHITECTURE.md` nor `AGENTS.md` mentions the formal-methods lane** — grep for
  `buzz-conformance`, `conformance`, `MultiTenantRelay.tla`, `TLA`, and `formal` across
  `AGENTS.md`, `ARCHITECTURE.md`, and `CONTRIBUTING.md` returns no hits.

### Batch 2d integrations (LLM providers, MCP subprocesses, OAuth, multicall shims)

2d is where Buzz talks to things outside itself. Four integration classes appear here for the
first time: **outbound LLM provider HTTP**, **MCP child processes over stdio**, an **OAuth 2.0
PKCE flow with a loopback listener**, and **`argv[0]` multicall dispatch** between crates in this
workspace. The recurring theme is duplication where a dependency would do.

| ID | Finding | Location |
|---|---|---|
| INT-2d-1 | **Seven outbound LLM endpoints across four providers, all built by string concatenation from an unvalidated base URL.** Anthropic `POST {base}/v1/messages` with `x-api-key` + `anthropic-version` (`llm.rs:256-264`); OpenAI Responses `POST {base}/responses` and Chat `POST {base}/chat/completions`, both `Authorization: Bearer` (`:285`, `:290`, `:356`); legacy Databricks `POST {base}/serving-endpoints/{model}/invocations` (`:335-341`); and DatabricksV2's three gateway paths `/ai-gateway/{openai/v1/responses, anthropic/v1/messages, mlflow/v1/chat/completions}` (`:700-702`). Trailing slashes are trimmed at every construction site. TLS is reqwest+rustls default with no `danger_accept_invalid_certs` anywhere, but **nothing requires HTTPS** — see SEC-56. Bodies are serialized once before the retry loop and cloned per attempt (`:1053-1065`) specifically so a serialization failure cannot be mistaken for a retryable transport error. | `crates/buzz-agent/src/llm.rs:256-341`, `:700-702`, `:1053-1065` |
| INT-2d-2 | **`anthropic-version` is never sent on DatabricksV2's Anthropic route.** `post_openai` applies only `bearer_auth` (`llm.rs:356`), so the gateway's Anthropic Messages endpoint (`:701`) receives no version header. Whether it requires one is unverified and no test covers it. | `crates/buzz-agent/src/llm.rs:356`, `:701` |
| INT-2d-3 | **MCP children are spawned from client-supplied command and args with no allowlist**, in their own process group, with a curated environment. `McpRegistry::spawn_all` takes `command`/`args` verbatim from the `session/new` payload (`mcp.rs:711-712`), calls `env_clear()` then re-adds 17 allowlisted keys (`:39-63`, `:714-719`), applies client-supplied `env` **after** the allowlist so the payload wins (`:726-728`), and sets `Command::process_group(0)` (`:733`) so a timeout can kill the whole tree. The crate README claims `pre_exec` + `setpgid` — it is `process_group`, because `pre_exec` would require `unsafe`, which `lib.rs:1` forbids. Caps: 16 servers, 128 tools, 1 KiB descriptions, 4 KiB schemas (`:177-182`, `:232-236`, `:253-257`). | `crates/buzz-agent/src/mcp.rs:39-63`, `:711-733` |
| INT-2d-4 | **OAuth 2.0 PKCE against a discovery document, with a loopback HTTP listener.** Discovery at `{DATABRICKS_HOST}/oidc/.well-known/oauth-authorization-server` (`llm.rs:1188-1191`), `authorization_endpoint`/`token_endpoint` then taken verbatim from that document (`auth.rs:172-188`); an ephemeral `127.0.0.1:0` axum listener receives the callback (`:571`) with a 60 s budget and abort-on-drop (`:584-586`, `:601`). Token exchange carries the code plus `code_verifier` (`:609-616`); refresh carries the refresh token (`:206-213`). No scheme validation and no HTTP timeout on either client (`:153`). | `crates/buzz-agent/src/auth.rs:153`, `:172-213`, `:527-630` |
| INT-2d-5 | **The Databricks PKCE configuration is declared twice and cannot be shared.** Canonical at `llm.rs:1185-1198` using the private constants `DATABRICKS_CLIENT_ID` (`:19`) and `DATABRICKS_OAUTH_SCOPES` (`:20`); re-declared as bare string literals in `lib.rs:135-143`. Because both constants are private to a private module, `lib.rs` **structurally cannot** reference them. The cache filename hashes exactly those values (`auth.rs:446-454`), so a scope added to one side would make `buzz-agent auth databricks` cache a token the runtime never reads — with both sides reporting success. No test asserts they agree, and the oauth test re-implements the path derivation locally (`tests/databricks_oauth.rs:84-97`) so it would stay green. | `crates/buzz-agent/src/llm.rs:19-20`, `:1185-1198` vs `lib.rs:135-143` |
| INT-2d-6 | **`buzz-acp` reimplements `buzz-ws-client` rather than depending on it**, including a byte-identical debug string, and the copies have now diverged in wire coverage (7 inbound message types vs 6, `COUNT` absent from the acp copy). Full anchor list at DEBT-59. `buzz-cli`, by contrast, delegates the entire connect → NIP-42 → EVENT → OK → close sequence to the shared crate (`client.rs:1084`) and passes a 75 s outer budget with an in-code comment citing the three inner constants (20+20+30 s) — a cross-reference that is still accurate (`client.rs:1075-1085` vs `crates/buzz-ws-client/src/connection.rs:17-23`). | `crates/buzz-acp/src/relay.rs` vs `crates/buzz-ws-client/src/`; `crates/buzz-cli/src/client.rs:1075-1085` |
| INT-2d-7 | **`buzz-persona` is a declared dependency of `buzz-acp` with zero references** — `Cargo.toml:22`, the only internal dep declared by `path` rather than `workspace = true`, and no use anywhere under `crates/buzz-acp/src`. Its real CLI consumer is `pack.rs`, which reaches it at exactly four call sites (`:24`, `:28`, `:31`, `:62`) for `validate_pack`, the two `ValidationDiagnostic` variants, and `resolve_pack`. | `crates/buzz-acp/Cargo.toml:22`; `crates/buzz-cli/src/commands/pack.rs:24-62` |
| INT-2d-8 | **`argv[0]` multicall dispatch links four crates in this workspace.** `sprig` dispatches on `argv[0]` to `buzz_agent::run()` (`crates/sprig/src/main.rs:10-19`); `buzz-dev-mcp` calls `buzz_cli::run_from_args` when invoked as `buzz` (`crates/buzz-dev-mcp/src/lib.rs:167-170`); and `Shim::install` creates the symlinks that make that happen, in a `0700` tempdir prepended to the child `PATH` (`shim.rs:31-49`, dispatch `lib.rs:144-171`). Both `buzz-dev-mcp` and `buzz-cli` install the ring `CryptoProvider`, which is exactly the double-install that `buzz-cli`'s `let _ =` swallow accommodates — documented at `crates/buzz-cli/src/lib.rs:30-38`. | `crates/sprig/src/main.rs:10-19`; `crates/buzz-dev-mcp/src/lib.rs:144-171`; `crates/buzz-cli/src/lib.rs:30-38` |
| INT-2d-9 | **The desktop Tauri backend links `buzz-agent` for a single shared constant.** `desktop/src-tauri/Cargo.toml:91` pulls the crate in as `buzz_agent_pkg` purely to read `WINDOWS_SHELL_RESOLUTION_ENV` (`desktop/src-tauri/src/managed_agents/git_bash.rs:136`, `:438`) — deliberately sourced rather than copied, which is the right call and worth noting as a counterexample to the duplication above. | `desktop/src-tauri/Cargo.toml:91`; `crates/buzz-agent/src/lib.rs:22-30` |
| INT-2d-10 | **A compile-time coupling from a Rust crate into the desktop source tree.** `crates/buzz-agent/src/config.rs:2667` does `include_str!("../../../desktop/src/features/agents/ui/effortTable.fixture.json")`. Because `include_str!` is a compile-time macro, moving or renaming that desktop file breaks `cargo test -p buzz-agent` rather than merely failing an assertion. | `crates/buzz-agent/src/config.rs:2667` |
| INT-2d-11 | **`buzz-cli` reaches three relay HTTP endpoints outside the documented surface**, and the code carries the better rationale: `moderation.rs:8-13` argues that report and audit rows are structured queue rows, not public Nostr events, so serving them over a REQ filter would mean synthesizing fake events. Routed at `crates/buzz-relay/src/router.rs:113-116`, handled at `api/bridge.rs:2091-2145`, and correctly tenant-bound from the `Host` header before any lookup (`:2036-2043`). | `crates/buzz-cli/src/commands/moderation.rs:8-13`, `:110-128` |
| INT-2d-12 | **`buzz-cli`'s relationship to the relay's git surface is data-mediated, not protocol-level.** The CLI never speaks git wire protocol; `git-sign-nostr` and `git-credential-nostr` are peer tools invoked by `git` itself and are referenced nowhere in `buzz-cli` (zero matches). The coupling is: `repos create` publishes a kind-30617 announcement carrying `clone` URLs that point at the relay's smart-HTTP endpoints (`repos.rs:216-224`), and `repos protect set/remove` writes `buzz-protect` tags (`:64-90`) that the relay's policy hook reads back via `parse_protection_tags` + `evaluate_push` (`crates/buzz-relay/src/api/git/policy.rs:45`, `:285`) at `POST /internal/git/policy`. `buzz_core::git_perms` is the shared contract and is referenced by exactly three files repo-wide. The only place signing-tool output surfaces in the CLI is `patches send --commit-pgp-sig`, passed through untouched (`patches.rs:41`). | `crates/buzz-cli/src/commands/repos.rs:64-90`, `:216-224`; `crates/buzz-relay/src/api/git/policy.rs:45`, `:285` |
| INT-2d-13 | **`buzz-workflow` is reached only by document, not by dependency.** It is absent from `crates/buzz-cli/Cargo.toml`, and there is no YAML parser in the crate at all — so `workflows create`/`update` publish the definition verbatim with only the SDK's 64 KiB byte cap (`builders.rs:1468`, `:1486`) between argv and the relay. The `if:` conditions inside are evaluated by `evalexpr` in the relay's executor (`crates/buzz-workflow/src/executor.rs:15`, `:203-232`), whose variable namespace, underscore-not-dot naming rule (`:205-216`) and registered string helpers (`:232`) the CLI has no view of. The webhook surface `POST /hooks/{id}` (`crates/buzz-relay/src/router.rs:120`) is unreachable from any CLI command — `workflows trigger` uses the kind-46020 event path instead (`workflows.rs:156-190`). | `crates/buzz-cli/src/commands/workflows.rs:104`, `:127`, `:156-190`; `crates/buzz-workflow/src/executor.rs:203-232` |
| INT-2d-14 | **`buzz-dev-mcp` is an execution and filesystem integration with no containment.** `shell` invokes a real shell interpreter (`shell.rs:165-167`) overriding only `PATH` and `GIT_CONFIG_*` (`:169-174`); `resolve_path` accepts absolute paths and canonicalizes through symlinks with no root check (`paths.rs:20-45`). `rg.rs` integrates the external `ripgrep` binary when present and otherwise falls back to hand-rolled literal matching — permanently, on Windows, because the PATH split and probe name are Unix-shaped (DEBT-71). | `crates/buzz-dev-mcp/src/shell.rs:165-174`; `paths.rs:20-45`; `rg.rs:34-54` |
| INT-2d-15 | **Filesystem integration points, and which ones are bounded.** Bounded: `upload_file` stats then reads with 50 MiB / 500 MiB caps and magic-byte MIME sniffing (`client.rs:1102-1131`); `mem`'s stdin `.take(NIP44_PLAINTEXT_MAX + 1)` (`mem.rs:324-338`). Unbounded and unconfined: `read_file_or_stdin` (`validate.rs:195`, reached by `patches send --patch-file` and `pr --body-file`), `read_or_stdin` (`validate.rs:168-179`, reached by six commands), `mem patch --patch-file` (`mem.rs:581`), `media get -o` on the write side (`upload.rs:23`), `buzz-agent`'s `AGENTS.md` reads (`hints.rs:65-66`, cap applied after the read), and `load_skill`'s supporting-file reads (`builtin.rs:197-209`). Pack roots are `exists()`/`is_dir()`-checked but not confined (`pack.rs:16-22`), while every file *inside* a pack is resolved through `buzz_persona::safe_resolve` — so a malicious pack cannot escape its directory, but the operator can point the root anywhere. | `crates/buzz-cli/src/validate.rs:168-198`; `commands/mem.rs:581`; `crates/buzz-agent/src/builtin.rs:197-209` |
| INT-2d-16 | **No subprocess is spawned anywhere in `buzz-cli`** (zero `Command::new` matches across all 21 command modules and the core) — the CLI is itself the child that the ACP harness configures. Nor does it touch any OS keychain (zero `keychain`/`security_framework` matches); secrets arrive purely through env and flags. `pack inspect` prints MCP server `command`/`args` values but never executes them, and prints only the server **count** (`pack.rs:116`), keeping declared credentials out of the output. | verified by grep across `crates/buzz-cli/src` |
| INT-2d-17 | **Two intra-crate layering inversions worth recording.** `buzz-agent`'s `hints.rs:4` and `builtin.rs:10` import `truncate_at_boundary` from `mcp.rs`, so two filesystem-facing modules depend on the process registry for a byte-slicing helper. And `catalog.rs:17` imports `build_token_source` from `llm.rs`, so "model catalog" cannot be used without the LLM transport module — which is the entire reason `Config::for_discovery` (`config.rs:845-876`) exists, with ~30 inert fields (now 27 total, one of which — `prefer_mesh_for_auto` — the sync correctly pinned to `false` at `config.rs:854` rather than leaving inert-by-omission). Relatedly, `buzz-cli`'s `channels.rs:12` imports `fetch_archived_snapshot` from `commands::agents`, a cross-module dependency for a NIP-IA trust check. | `crates/buzz-agent/src/hints.rs:4`; `builtin.rs:10`; `catalog.rs:17`, `config.rs:845-876` |
| INT-2d-18 | **Third-party integrations and one dependency reached from exactly one place.** `reqwest` (rustls), `serde_json`, `tokio`, `tracing`, `getrandom` (used at a single site — `llm.rs:1012` — for backoff jitter, degrading silently to un-jittered delay on failure), `infer` for magic-byte MIME sniffing (`client.rs:1109-1111`), `sha2`+`hex`, both base64 alphabets (standard for NIP-98, URL-safe unpadded for Blossom), `url` for origin comparison, `uuid`, `chrono` (reached only from `agent_management.rs:97`, so `agents draft-*` is the sole reason it is a dependency), `rand` for jitter, `diffy` (31 occurrences, **all** in `mem.rs` — a dependency reached exclusively from one command), and `dirs` (used only by `channel_templates.rs`). No unused dependency was found in `buzz-cli`; two `Cargo.toml` comments are stale (CFG-2d-2). | `crates/buzz-cli/src/commands/mem.rs`; `client.rs:1109-1111`; `crates/buzz-agent/src/llm.rs:1012` |
| INT-2d-19 | **The desktop channel-template store is the CLI's only file-based configuration integration**, and it only knows the production bundle id. Default path is `dirs::data_dir()/xyz.block.buzz.app/templates/channel-templates.json` (`channel_templates.rs:17`, `:73-79`); `tauri.dev.conf.json` sets `xyz.block.buzz.app.dev`, so templates authored in a dev desktop build are invisible unless `--templates-file` is passed. The CLI's record is a deliberate narrowing of `desktop/src-tauri/src/templates/types.rs:5-21`, documented at `channel_templates.rs:1-8`; the dropped `role` field is harmless because both surfaces hardcode `bot` (`channels.rs:748-751`, `useApplyTemplate.ts:101`). | `crates/buzz-cli/src/commands/channel_templates.rs:1-8`, `:17`, `:73-79` |
| INT-2d-20 | **`buzz-acp` → `buzz-agent` contract points, all comment-documented rather than typed.** `usage_update` on `_goose/unstable/session/update` consumed by `UsageTracker` (`crates/buzz-acp/src/usage.rs:164`); `keepalive` classified as non-content so it resets the idle clock (`acp.rs:1623`, test `:2870`); `_meta.goose.activeRunId` cached as the steer target (`acp.rs:189`, `:769`, used `:1293-1313`); `stopReason` strings parsed at `acp.rs:61-72`. One of these has drifted — the `activeRunId: null` end-of-turn signal `acp.rs:185-186` documents is never emitted (DEBT-65). The goose wire extensions themselves are reimplemented from an external contract with the upstream reference cited only from the client side (`acp.rs:178-180`). | `crates/buzz-acp/src/acp.rs:61-72`, `:178-189`, `:1623` |
| INT-2d-21 | **Observability side effect of the harness integration:** `buzz-acp` forwards every frame it reads from the agent verbatim to its observer as `acp_read` (`acp.rs:1120`, `:1414`), so everything `buzz-agent` writes — including `rawInput`, the full tool arguments (`agent.rs:412`), and completed tool-result text (`:592`) — is republished, and the desktop transcript renders `update.rawInput` directly (`desktop/src/features/agents/ui/agentSessionTranscriptHelpers.ts:361`). Nothing on that path redacts or filters. | `crates/buzz-acp/src/acp.rs:1120`, `:1414`; `crates/buzz-agent/src/agent.rs:412`, `:592` |

## External Services

| Service | Direction | Protocol | Client | Purpose | Source |
|---|---|---|---|---|---|
| _pending_ | | | | | |

## Outbound HTTP

_Pending Phase 2 (workflow `call_webhook` with SSRF guard, push gateway delivery, media
storage)._

## Inbound Integrations

_Pending Phase 2 (workflow webhooks at `/hooks/{id}`, git smart HTTP, internal git policy
hook)._

## Error Handling & Resilience

_Pending Phase 2 (Redis reconnect backoff, agent crash recovery, slow-client handling,
bounded queues)._

---

# Phase 2 — Module Findings

## Module: buzz-core (`crates/buzz-core`)

### Aspect: Integrations

This crate has **no internal Buzz dependencies** — `crates/buzz-core/Cargo.toml:13-27` lists only third-party crates. It integrates with no network service, database, cache, or file system: there is no I/O code path anywhere in `src/`.

---

### 1. External crate dependencies

All versions come from `[workspace.dependencies]` in the repo-root `Cargo.toml`; `buzz-core` declares them as `{ workspace = true }` except `percent-encoding`, which is version-pinned locally.

| Crate | Declared at | Workspace version | Why it is used (code evidence) |
|-------|-------------|-------------------|-------------------------------|
| `nostr` | `crates/buzz-core/Cargo.toml:14` | `0.44`, features `["nip44", "nip98"]` (root `Cargo.toml:61`) | The core event/key model. Provides `Event`, `EventId`, `Filter`, `Keys`, `Kind`, `PublicKey`, `SecretKey`, `Tag`, `EventBuilder`, `Timestamp`, `SingleLetterTag`, `Alphabet` (re-exported at `src/lib.rs:42`); `verify_id`/`verify_signature`/`EventId::new` for verification (`src/verification.rs:12-25`); `secp256k1::Error` in the error enum (`src/error.rs:19`); `nips::nip44` v2 encrypt/decrypt (`src/observer.rs:73-95`, `src/engram.rs:457` and `:549`, `src/pairing/session.rs:566` and `:600`); `nips::nip44::v2::ConversationKey` for the engram K_c (`src/engram.rs:136-138`); `util::hkdf::{extract, expand}` for all NIP-AB derivations (`src/pairing/crypto.rs:24`, used in `hkdf32` at `:34-43`); `util::generate_shared_key` for ECDH (`src/pairing/session.rs:186`, `:292`); `hashes::Hash` trait import (`src/pairing/crypto.rs:23`) |
| `serde` | `Cargo.toml:15` | `1`, `derive` (root `:64`) | Derives on `PresenceStatus` (`src/presence.rs:9`), `TokenCounts`/`StopReason`/`AgentTurnMetricPayload` (`src/agent_turn_metric.rs:21`, `:51`, `:87`), `Listing` (`src/engram.rs:597`), all pairing message types (`src/pairing/types.rs:19`, `:61`, `:79`); manual `Deserialize` impl for `StopReason` (`src/agent_turn_metric.rs:66-77`); low-level `de::{DeserializeSeed, MapAccess, SeqAccess, Visitor}` for the strict-JSON parser (`src/engram.rs:284`) |
| `serde_json` | `Cargo.toml:16` | `1` (root `:69`) | Payload (de)serialization in observer/pairing (`src/observer.rs:63`, `:106`, `src/pairing/session.rs:565`, `:618`); `Value` + `Deserializer::from_slice` for duplicate-key-rejecting engram body parsing (`src/engram.rs:283-380`); JSON errors wrapped into `ObserverPayloadError::Json` (`src/observer.rs:35`) and `PairingError::Json` (`src/pairing/mod.rs:71`) |
| `thiserror` | `Cargo.toml:17` | `2` (root `:85`) | `#[derive(thiserror::Error)]` on `VerificationError` (`src/error.rs:2`), `EngramError` (`src/engram.rs:37`), `ObserverPayloadError` (`src/observer.rs:28`), `PairingError` (`src/pairing/mod.rs:34`), `NormalizeRelayUrlError` (`src/relay.rs:7`) |
| `uuid` | `Cargo.toml:18` | `1`, features `["v4", "serde"]` (root `:89`) | `StoredEvent.channel_id: Option<Uuid>` (`src/event.rs:17`), `CommunityId(Uuid)` (`src/tenant.rs:37`), `uuid::Uuid::new_v4()` in filter tests (`src/filter.rs:192`, `:213`, `:224`) |
| `chrono` | `Cargo.toml:19` | `0.4`, `serde` (root `:90`) | `DateTime<Utc>` receive timestamps and `Utc::now()` (`src/event.rs:6`, `:25`), test helper (`src/lib.rs:51`) |
| `hex` | `Cargo.toml:20` | `0.4` (root `:97`) | d-tag hex encoding (`src/engram.rs:155`), session-secret hex in the QR URI (`src/pairing/qr.rs:81`, `:163`), session id / transcript hash hex on the wire (`src/pairing/session.rs:179`, `:216`, `:318`, `:357`) |
| `hmac` | `Cargo.toml:21` | `0.13` (root `:98`) | `Hmac`, `Mac`, `digest::KeyInit` for the engram d-tag HMAC-SHA256 (`src/engram.rs:10-11`, `:147-154`) |
| `sha2` | `Cargo.toml:22` | `0.11` (root `:96`) | `Sha256` as the HMAC hash (`src/engram.rs:15`, `:147`) |
| `rand` | `Cargo.toml:23` | `0.10` (root `:101`) | `rand::fill` to generate the 32-byte pairing session secret (`src/pairing/session.rs:113-115`); `rand::random::<u64>()` for the 0–30 s `created_at` jitter (`src/pairing/session.rs:578`) |
| `subtle` | `Cargo.toml:24` | `2.6` (root `:102`) | `ConstantTimeEq` behind `ct_eq` for 32-byte secret comparisons (`src/pairing/crypto.rs:126-129`) |
| `zeroize` | `Cargo.toml:25` | `1.8` (root `:103`) | `Zeroize` on plaintext buffers (`src/observer.rs:66`, `:79`, `:101`, `:109`), `QrPayload::drop` (`src/pairing/qr.rs:52-56`), `PairingSession::drop` (`src/pairing/session.rs:731-739`), `Zeroizing<String>` payload type in the pairing API (`src/pairing/session.rs:227-247`, `:388-408`) |
| `percent-encoding` | `Cargo.toml:26` | **`"2.3"` pinned locally, not via workspace** | `utf8_percent_encode` / `percent_decode_str` with the `NON_ALPHANUMERIC` set for relay URLs inside the QR URI (`src/pairing/qr.rs:31`, `:220-236`) |
| `url` | `Cargo.toml:27` | `2` (root `:114`) | `Url` + `Host` parsing in `normalize_relay_url` (`src/relay.rs:4`, `:38-64`), `relay_url_authority` (`src/tenant.rs:156-172`), and relay-URL validation during QR decode (`src/pairing/qr.rs:185-204`) |

Transitive crypto note: no direct `secp256k1` dependency is declared; the crate reaches Schnorr/secp256k1 exclusively through `nostr` (`nostr::secp256k1::Error` at `src/error.rs:19`, `event.verify_signature()` at `src/verification.rs:27`).

---

### 2. The zero-I/O prohibition — how it is (and is not) enforced

| Mechanism | Present? | Evidence |
|-----------|----------|----------|
| Comment in the crate manifest | **yes** | `crates/buzz-core/Cargo.toml:28`: `# NO tokio, NO sqlx, NO redis, NO axum — zero I/O dependencies` |
| Absence of those crates in `[dependencies]` | **yes** | `Cargo.toml:13-27` lists 14 dependencies; none is `tokio`, `sqlx`, `redis`, or `axum` |
| Absence of `dev-dependencies` that would reintroduce a runtime | **yes** | the manifest has no `[dev-dependencies]` section at all (`Cargo.toml:1-29`) |
| Crate-level lint attributes | partial | `#![deny(unsafe_code)]`, `#![warn(missing_docs)]` (`src/lib.rs:1-2`) — these do not restrict dependencies |
| `cargo-deny` ban rules | **no** | root `deny.toml:90-92` `[bans]` sets only `multiple-versions = "warn"` and `wildcards = "allow"`; no per-crate dependency bans |
| `[workspace.lints]` / `[lints]` | **no** | neither section exists in the root `Cargo.toml` nor in `crates/buzz-core/Cargo.toml` |
| Automated CI check specific to buzz-core dependencies | **none found** | the only `buzz-core` references in tooling are test invocations: `Justfile:278` (`cargo nextest run -p buzz-core -p buzz-auth --lib`), `scripts/run-tests.sh:81-82` (`cargo test -p buzz-core --lib`) |

Conclusion as coded: the no-I/O rule is a **convention documented in the manifest comment and upheld by review**, not a mechanically enforced constraint. Nothing in the repo fails a build if `tokio` is added to `crates/buzz-core/Cargo.toml`.

The same "documented fence, not compiler fence" pattern is stated explicitly for tenant construction (`src/tenant.rs:23-30`), which names a "migration-lint harness" outside this crate as the enforcing mechanism.

---

### 3. Cargo features

| Feature | Declared at | Gates |
|---------|-------------|-------|
| `test-utils` | `crates/buzz-core/Cargo.toml:10-11` | `pub mod test_helpers` via `#[cfg(any(test, feature = "test-utils"))]` (`src/lib.rs:47`) |

No default features are declared. `test-utils` is enabled by one consumer: `crates/buzz-relay/Cargo.toml:89`, inside that crate's `[dev-dependencies]` section (`crates/buzz-relay/Cargo.toml:86`).

---

### 4. Consumers (inbound integration)

`buzz-core` is depended on by 15 workspace crates plus the Tauri desktop backend — all as `{ workspace = true }` path dependencies:

| Consumer | Manifest line |
|---|---|
| `buzz-acp` | `crates/buzz-acp/Cargo.toml:20` |
| `buzz-admin` | `crates/buzz-admin/Cargo.toml:16` |
| `buzz-audit` | `crates/buzz-audit/Cargo.toml:11` |
| `buzz-auth` | `crates/buzz-auth/Cargo.toml:15` |
| `buzz-cli` | `crates/buzz-cli/Cargo.toml:46` |
| `buzz-db` | `crates/buzz-db/Cargo.toml:11` |
| `buzz-dev-mcp` | `crates/buzz-dev-mcp/Cargo.toml:41` |
| `buzz-media` | `crates/buzz-media/Cargo.toml:11` |
| `buzz-pairing-cli` | `crates/buzz-pairing-cli/Cargo.toml:15` |
| `buzz-pubsub` | `crates/buzz-pubsub/Cargo.toml:11` |
| `buzz-relay` | `crates/buzz-relay/Cargo.toml:19` (+ `:89` under `[dev-dependencies]` with `test-utils`) |
| `buzz-sdk` | `crates/buzz-sdk/Cargo.toml:11` |
| `buzz-search` | `crates/buzz-search/Cargo.toml:11` |
| `buzz-test-client` | `crates/buzz-test-client/Cargo.toml:12` |
| `buzz-workflow` | `crates/buzz-workflow/Cargo.toml:11` |
| desktop Tauri backend | `desktop/src-tauri/Cargo.toml:88`, aliased as `buzz_core_pkg = { package = "buzz-core", path = "../../crates/buzz-core" }` |

`crates/buzz-conformance/Cargo.toml:14` mentions buzz-core in a comment about its "no Serde, no `From<Uuid>`" fence on `CommunityId` rather than as a dependency line at that position.

---

### 5. Protocol/spec integrations referenced by the code

These are specification integrations (behaviour contracts), all named in doc comments:

| Spec | Where the crate implements or references it |
|---|---|
| NIP-01 (event id/sig, filters, kind semantics) | `src/verification.rs:11-32`, `src/filter.rs:1-3`, `:62`, `src/kind.rs:3-5` |
| NIP-09, NIP-17, NIP-23, NIP-25, NIP-29, NIP-33, NIP-34, NIP-38, NIP-42, NIP-43, NIP-50, NIP-51, NIP-56, NIP-65, NIP-78, NIP-94, NIP-98, BUD-01 | kind registry doc comments, `src/kind.rs:8-563` |
| NIP-44 v2 (encryption + conversation key) | `src/observer.rs:66-95`, `src/engram.rs:457`, `src/pairing/session.rs:566` |
| NIP-AB (device pairing) | `src/pairing/**`; local spec files `src/pairing/NIP-AB.md` and a Tamarin model `src/pairing/NIP-AB.spthy` live beside the code |
| NIP-AE (agent engrams) | `src/engram.rs:1-6` (points to `docs/nips/NIP-AE.md`) |
| NIP-AM (agent turn metric) | `src/agent_turn_metric.rs:1-8` (points to `docs/nips/NIP-AM.md`) |
| NIP-AP (persona/team/managed agent), NIP-ER (event reminder), NIP-PL (push lease), NIP-DV (DM visibility), NIP-IA (identity archival), NIP-RS (read state) | kind doc comments `src/kind.rs:71-389` |
| RFC 1918 / 2544 / 3849 / 6598 (address ranges) — plus RFC 6052 / 8215 / 4380 / 3056 added by `c26bf594`, RFC 3986 (host case), RFC 8259 (JSON strings) | `src/network.rs:26-45` (blocked-range doc list; RFC 6052 cited at `:6`), `src/tenant.rs:110-116`, `src/engram.rs:261-262` |


## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: Integrations

---

### 1. Declared dependencies

`crates/buzz-sdk/Cargo.toml:10-16` (all workspace-pinned):

| Crate | Version (workspace) | Where declared | Usage in this crate |
|---|---|---|---|
| `buzz-core` | path `crates/buzz-core` (`Cargo.toml:124`) | `Cargo.toml:11` | kind constants, channel enums, `canonical_channel_name`, observer constants + `content_looks_like_nip44` |
| `nostr` | `0.44`, features `["nip44","nip98"]` (`Cargo.toml:61`) | `Cargo.toml:12` | `EventBuilder`, `Kind`, `Tag`, `EventId`, `Keys`, `PublicKey`, `SecretKey`, `SECP256K1`, `hashes::sha256`, `secp256k1::{Message, schnorr::Signature}`, `FromBech32` |
| `uuid` | `1`, features `["v4","serde"]` (`Cargo.toml:89`) | `Cargo.toml:13` | channel/workflow/action identifiers; `Uuid::parse_str` for validation |
| `serde` | `1`, feature `derive` (`Cargo.toml:64`) | `Cargo.toml:14` | **declared but no `use serde` / `#[derive(Serialize)]` appears in `src/`** — transitively needed only via `serde_json` |
| `serde_json` | `1` (`Cargo.toml:69`) | `Cargo.toml:15` | kind-0 content assembly, profile JSON parsing, auth-tag JSON encode/decode |
| `thiserror` | `2` (`Cargo.toml:85`) | `Cargo.toml:16` | `SdkError` derive (`lib.rs:87`) |

No `reqwest`, `tokio`, `axum`, `tungstenite`, or any async runtime. No dev-dependencies.

---

### 2. How each external crate is used

**`nostr` (0.44)**

| Import | Site | Purpose |
|---|---|---|
| `EventBuilder`, `Kind`, `Tag` | `builders.rs:23` | every builder's return value; `Kind::Custom(u16)` for all kinds; `Tag::parse(iter)` for tag construction |
| `Tag::parse` error mapping | `builders.rs:30-32`, `205-207` | mapped into `SdkError::InvalidTag` |
| `nostr::EventId` | `lib.rs:29-31`, `builders.rs:379`, `447`, `464`, `495`, `740` | typed event references; `.to_hex()` used for tag values |
| `nostr::Event` | `builders.rs:816` | `extract_channel_id` reads `event.tags` |
| `.allow_self_tagging()` | `builders.rs:1800`, `1821` | opt out of nostr 0.44's default same-pubkey `p`-tag stripping |
| `FromBech32`, `PublicKey` | `mentions.rs:32` | NIP-19 `npub` decoding in `extract_nostr_uris` |
| `hashes::sha256::Hash`, `hashes::Hash` | `nip_oa.rs:22-23` | SHA-256 of the NIP-OA preimage |
| `secp256k1::Message`, `secp256k1::schnorr::Signature`, `SECP256K1` | `nip_oa.rs:24-26` | BIP-340 Schnorr signing (`Keys::sign_schnorr`, `nip_oa.rs:170`) and verification (`SECP256K1.verify_schnorr`, `nip_oa.rs:241-243`) |
| `Keys`, `PublicKey::xonly()` | `nip_oa.rs:26`, `237-240` | owner key handling and x-only conversion for verification |
| `nostr::SecretKey`, `Keys::new` | `builders.rs:3756-3762` (test) | deterministic signing in the NIP-IA self-path test |

The `nip44` cargo feature of `nostr` is not used directly here — NIP-44
encryption lives in `buzz-core` (`crates/buzz-core/src/observer.rs:58`); the SDK
only length-checks ciphertext.

**`serde_json`**

| Site | Use |
|---|---|
| `builders.rs:541-561` | builds the kind-0 content object via `serde_json::Map` + `Value::String`, then `.to_string()` |
| `mentions.rs:183-190` | `serde_json::from_str::<Value>` on kind-0 `content` for name matching; parse failures are swallowed (`let Ok(..) else { continue }`) |
| `nip_oa.rs:124-133` | `parse_json_array` — `from_str` into `Value`, requires `Value::Array` |
| `nip_oa.rs:174` | `serde_json::json!([...])` to emit the auth tag |
| `builders.rs:2011-2016` (tests) | asserts on parsed kind-0 JSON |

**`uuid`**

| Site | Use |
|---|---|
| all channel-scoped builders | `Uuid` parameters rendered with `.to_string()` into `h`/`d`/`action_id` tags |
| `builders.rs:822` | `Uuid::parse_str` in `extract_channel_id` (invalid ⇒ `None`) |
| `builders.rs:1371-1373` | `Uuid::parse_str` validating and canonicalizing `GitPullRequestMeta.channel_id` |

**`buzz-core`**

| Import | Site | Use |
|---|---|---|
| 26 `KIND_*` constants | `builders.rs:6-19` | kind integers, cast `as u16` into `Kind::Custom` |
| `observer::{OBSERVER_AGENT_TAG, OBSERVER_FRAME_TAG, OBSERVER_FRAME_TELEMETRY, OBSERVER_FRAME_CONTROL, content_looks_like_nip44}` | `builders.rs:20-23` | observer-frame tag names, allowed frame values, ciphertext length gate |
| `channel::canonical_channel_name` | `builders.rs:623`, `636`, `675`; re-export `lib.rs:78` | channel-name normalization |
| `channel::{ChannelType, ChannelVisibility, MemberRole}` | re-exported `lib.rs:80-84`; used `builders.rs:566-578`, `674-696` | tag value vocabularies |
| `kind` module | re-exported `lib.rs:22` | so consumers avoid a direct `buzz-core` dependency |
| `observer::encrypt_observer_payload` | `builders.rs:1887-1892` (test only) | produces real NIP-44 ciphertext for the observer-frame happy path test |

---

### 3. HTTP / REST / WebSocket calls made

**None.** There are no HTTP methods, paths, URLs, sockets, or async functions in
this crate. The module doc states it explicitly
(`crates/buzz-sdk/src/lib.rs:13`), and the dependency set contains no HTTP or
WebSocket client (`crates/buzz-sdk/Cargo.toml:10-16`).

URLs appear only as **validated string values written into tags**, never as
request targets:

| Value | Validation | File:line |
|---|---|---|
| `DiffMeta.repo_url` | must start `http://`/`https://` | `builders.rs:317-321` |
| custom-emoji `url` | `http://`/`https://`, ≤2048 bytes | `builders.rs:152-170` |
| repo `clone_urls` | non-empty, ≤512 chars, scheme unchecked (so `ssh://`/`git@` forms are accepted — see test `builders.rs:2897-2925`) | `builders.rs:868-882` |
| repo `web_url` | `http://`/`https://`, ≤512 chars | `builders.rs:884-898` |
| repo `relays` | must start `ws://`/`wss://`, ≤256 chars | `builders.rs:900-919` |
| NIP-02 contact `relay_url` | ≤2048 bytes, scheme unchecked | `builders.rs:785-792` |
| `q`-tag relay hint | passed through unvalidated | `builders.rs:1266-1272` |
| PR/PR-update `clone_urls` | only non-emptiness of the list is checked; individual URLs unvalidated | `builders.rs:1344-1350`, `1425-1431` |

### 4. Error handling at integration boundaries

- All third-party errors are converted into `SdkError` variants rather than
  propagated: `Tag::parse` → `InvalidTag` (`builders.rs:30-32`), `Uuid::parse_str`
  → `InvalidInput` (`builders.rs:1371-1373`), `serde_json::from_str` →
  `InvalidInput` (`nip_oa.rs:125-127`), `PublicKey::from_hex` → `InvalidInput`
  (`nip_oa.rs:208-209`), `Signature::from_str` → `InvalidInput`
  (`nip_oa.rs:219-220`), `verify_schnorr` → `InvalidInput` (`nip_oa.rs:241-243`),
  `PublicKey::xonly` → `InvalidInput` (`nip_oa.rs:237-240`).
- Two integration points **swallow** errors deliberately: profile JSON that
  fails to parse is skipped (`mentions.rs:183-185`) and bech32 that fails to
  decode is skipped (`mentions.rs:381-386`). Both are documented as intentional.
- Signing errors are never handled here — `sign_with_keys` is called by the
  consumer, outside this crate.

---

### 5. Consumers (who integrates with this crate)

Declared `buzz-sdk` dependents (`grep` over workspace `Cargo.toml` files):
`crates/buzz-acp`, `crates/buzz-cli`, `crates/buzz-test-client`,
`crates/buzz-relay`, `desktop/src-tauri`.

Observed usage by symbol count in `src/`:

| Consumer | Distinct `buzz_sdk::*` paths referenced | Notes |
|---|---|---|
| `crates/buzz-cli` | 71 | the primary consumer |
| `crates/buzz-acp` | 7 | `build_message`, `build_reaction`, `build_remove_reaction`, `ThreadRef`, `nip_oa::{compute_auth_tag, parse_auth_tag, verify_auth_tag}` |
| `crates/buzz-relay` | 3 | `build_agent_observer_frame`, `ThreadRef`-adjacent paths, `nip_oa::verify_auth_tag` |
| `crates/buzz-test-client` | 0 `buzz_sdk::` paths in `src/` | declares the dependency but references it elsewhere (tests) or not at all |
| `desktop/src-tauri` | 0 `buzz_sdk::` paths in `src/` | builds its own events — `desktop/src-tauri/src/events.rs` contains 36 `EventBuilder::new` call sites, including its own `identity_archive_tags` (line 658) and `build_archive_identity_request` (line 716) |


## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: Integrations

### External crate dependencies

`crates/buzz-persona/Cargo.toml:9-16`. Note: unlike most crates in this workspace, these
are **direct version pins, not `workspace = true`** entries.

| Crate | Version | Features | Why (evidence) |
|---|---|---|---|
| `serde` | `1` | `derive` | `Serialize`/`Deserialize` derives on all public config types — `crates/buzz-persona/src/persona.rs:51`, `:68`, `:82`, `:99`; `crates/buzz-persona/src/manifest.rs:35`, `:47`, `:77` |
| `serde_json` | `1` | default | Parses `plugin.json` (`crates/buzz-persona/src/manifest.rs:153`) and `.mcp.json` (`crates/buzz-persona/src/pack.rs:192-197`); also used as the **internal merge medium** — `PersonaConfig` is round-tripped to `serde_json::Value` so the JSON-based merge can run (`crates/buzz-persona/src/pack.rs:406-409`, `crates/buzz-persona/src/merge.rs:47-83`) |
| `serde_yaml` | `0.9` | default | Parses persona YAML frontmatter (`crates/buzz-persona/src/persona.rs:220`) and `SKILL.md` frontmatter during advisory validation (`crates/buzz-persona/src/validate.rs:410-418`) |
| `thiserror` | `2` | default | All three error enums: `PersonaError` (`crates/buzz-persona/src/persona.rs:26`), `ManifestError` (`crates/buzz-persona/src/manifest.rs:22`), `PackError` (`crates/buzz-persona/src/pack.rs:25`) |

Dev-dependencies — `crates/buzz-persona/Cargo.toml:17-18`:

| Crate | Version | Why |
|---|---|---|
| `tempfile` | `3` | Temp pack directories in tests — `crates/buzz-persona/src/pack.rs:450`, `crates/buzz-persona/src/resolve.rs:739`, `crates/buzz-persona/src/validate.rs:462`, `crates/buzz-persona/tests/integration.rs:117`, `crates/buzz-persona/tests/e2e_env_flow.rs:37` |

No internal Buzz crates are depended on — `crates/buzz-persona/Cargo.toml` has no
`buzz-*` entries. The crate is a leaf.

`serde_yaml` 0.9 is unmaintained upstream (archived by its author); this is a supply-chain
observation recorded in the debt doc, not a code finding.

---

### Filesystem access

All I/O is synchronous `std::fs`. There is no async runtime dependency.

| Operation | Call site | Path source |
|---|---|---|
| `canonicalize` pack root | `crates/buzz-persona/src/pack.rs:126` | caller-supplied `pack_dir` |
| `exists()` manifest probe | `crates/buzz-persona/src/pack.rs:133` | `<root>/.plugin/plugin.json` |
| `read_to_string` | `crates/buzz-persona/src/pack.rs:367` (via `read_file`) | manifest, personas, instructions, `.mcp.json` |
| `metadata` size guard | `crates/buzz-persona/src/pack.rs:375`, `crates/buzz-persona/src/persona.rs:264` | any file about to be read |
| `canonicalize` declared paths | `crates/buzz-persona/src/pack.rs:357` | manifest-declared relative paths |
| `is_dir()` skills probe | `crates/buzz-persona/src/pack.rs:224`, `:277` | `<root>/skills` |
| `read_dir` skills enumeration | `crates/buzz-persona/src/pack.rs:278`, `crates/buzz-persona/src/validate.rs:377` | `<root>/skills` |
| `read_to_string` manifest (2nd read) | `crates/buzz-persona/src/validate.rs:213`, `:305` | advisory checks re-read `plugin.json` |
| `read_to_string` SKILL.md | `crates/buzz-persona/src/validate.rs:393` | `<root>/skills/<name>/SKILL.md` |

Writes: **none**. There is no `fs::write`, `create_dir`, `remove_*`, or `File::create`
in `crates/buzz-persona/src/` (all such calls are inside `#[cfg(test)]` blocks and the
`tests/` directory).

Process execution: **none**. No `std::process::Command`, no `exec`. Hooks are data only
(`crates/buzz-persona/src/resolve.rs:339-357`).

---

### Network access

**None.** No HTTP client, no socket, no URL fetching. `crates/buzz-persona/Cargo.toml`
declares no `reqwest`/`hyper`/`ureq`/`tokio`. `homepage` and `repository` manifest fields
(`crates/buzz-persona/src/manifest.rs:94`, and `"repository"` in
`KNOWN_MANIFEST_KEYS` at `crates/buzz-persona/src/validate.rs:108`) are stored as opaque
strings and never dereferenced.

`resolve.rs` states the design contract explicitly: "**Pure**: no env access, no network,
no side effects" (`crates/buzz-persona/src/resolve.rs:11`). That holds for `resolve.rs`
itself; `resolve_pack` does perform filesystem reads transitively via `pack::load_pack`
(`crates/buzz-persona/src/resolve.rs:109`).

---

### Declared consumers in the workspace

| Consumer | Dependency declaration | Actual code usage |
|---|---|---|
| `buzz-cli` | `crates/buzz-cli/Cargo.toml:70` — `buzz-persona = { path = "../buzz-persona" }` | **Yes** — `crates/buzz-cli/src/commands/pack.rs:24`, `:28`, `:31`, `:62` |
| `buzz-acp` | `crates/buzz-acp/Cargo.toml:22` — `buzz-persona = { path = "../buzz-persona" }` | **No** — a grep for `buzz_persona` across all of `crates/buzz-acp` returns zero matches. The dependency is declared but unused at the code level. |
| desktop Tauri backend | `desktop/src-tauri/Cargo.toml:89` — `buzz_persona_pkg = { package = "buzz-persona", path = "../../crates/buzz-persona" }` | **Yes, one call** — `desktop/src-tauri/src/migration.rs:1123` uses `buzz_persona_pkg::persona::split_frontmatter` |
| workspace membership | `Cargo.toml:23` — `"crates/buzz-persona"` | — |

#### `buzz-cli` consumption pattern

Two subcommands, dispatched at `crates/buzz-cli/src/lib.rs:1739-1740`:

- `buzz pack validate <path>` → `commands::pack::cmd_validate`
  (`crates/buzz-cli/src/commands/pack.rs:15-46`): checks the path exists and is a
  directory, calls `buzz_persona::validate::validate_pack`
  (`crates/buzz-cli/src/commands/pack.rs:24`), prints each `ValidationDiagnostic` to
  stderr by matching on the enum variants (`:26-35`), then maps `has_errors()` to
  `CliError::Usage` (`:37-38`). It does **not** use
  `ValidationReport::exit_code()` or the `Display` impl.
- `buzz pack inspect <path>` → `commands::pack::cmd_inspect`
  (`crates/buzz-cli/src/commands/pack.rs:52-152`): calls
  `buzz_persona::resolve::resolve_pack` (`:62`) and pretty-prints the fully resolved
  effective config per persona — recombining `llm_provider` + `model` for display
  (`:78-87`), triggers (`:101-114`), MCP server count (`:120-122`), skills (`:124-126`),
  a truncated system-prompt preview (`:132-145`), and `runtime_env_vars` as `K=V` pairs
  (`:147-155`).

#### desktop consumption pattern

`desktop/src-tauri/src/migration.rs:1121-1132` (`rewrite_legacy_persona_md_runtime`) uses
only the frontmatter splitter, then re-parses the YAML itself with `serde_yaml` to rewrite
`runtime: "sprout-agent"` → `"buzz-agent"` and re-emits `---\n{frontmatter}---\n{body}`.
It deliberately bypasses `parse_persona_md` — it must round-trip unknown/legacy keys that
`deny_unknown_fields` would reject.

#### How buzz-acp is *expected* to consume it

The dependency exists but no call sites do. The intended contract is documented rather
than exercised:

- `crates/buzz-persona/src/resolve.rs:1-14` — "`ResolvedPersona` maps 1:1 to ACP's needs";
  field-level comments name the ACP targets: `system_prompt` → `Config.system_prompt`
  (`:31`), `model` → `Config.model` (`:36`), `subscribe` → `Config.subscribe_mode +
  channels_override` (`:47`), `triggers` → "mapped to ACP filter rules at startup" (`:49`).
- `runtime_env_vars` (`crates/buzz-persona/src/resolve.rs:64`) is the projection buzz-acp
  is expected to inject at spawn. The matching consumer field exists on the ACP side:
  `crates/buzz-acp/src/config.rs:533-535` — `pub persona_env_vars: Vec<(String, String)>`
  with the comment "Populated from persona pack resolution. Empty when no pack is
  configured." It is passed to the spawn path at `crates/buzz-acp/src/lib.rs:3733`
  (`extra_env: config.persona_env_vars.clone()`).
- Operator-precedence filtering (level 1) is explicitly the consumer's job, not this
  crate's: `crates/buzz-persona/src/resolve.rs:359-364` — "ACP is responsible for
  filtering based on operator precedence (level 1)".
- `desktop/src-tauri/src/managed_agents/types.rs:52` references
  "ACP's `resolve_persona_by_name()`", indicating the desktop/ACP contract expects that
  entry point.

So: the *data contract* between this crate and buzz-acp is designed and the receiving
field exists on the ACP `Config`, but the wire-up that would call
`resolve_pack`/`resolve_persona_by_name` from buzz-acp is not present in the current tree.

#### Import-filter convention (consumer-side, mirrored in tests)

`crates/buzz-persona/tests/e2e_env_flow.rs:15-32` defines
`DERIVED_PROVIDER_MODEL_ENV_KEYS = ["GOOSE_MODEL", "GOOSE_PROVIDER", "BUZZ_AGENT_MODEL",
"BUZZ_AGENT_PROVIDER"]` and a `filter_derived` helper described as mirroring "desktop
import_persona_pack logic" (`crates/buzz-persona/tests/e2e_env_flow.rs:200`). The
test asserts that on import the derived provider/model keys are stripped while
`GOOSE_TEMPERATURE` survives (`:206-228`). This is a duplicated copy of consumer logic
living in this crate's test suite — the real implementation is outside this crate.

---

### Standards / external specs referenced

| Spec | Where referenced | Enforced in code? |
|---|---|---|
| Open Plugin Spec (`open-plugin-spec.org`) | `PERSONA_PACK_SPEC.md:6-7`, §2; `"$schema"` accepted in `KNOWN_MANIFEST_KEYS` (`crates/buzz-persona/src/validate.rs:101`) | Partially — OPS field names are accepted and unknown fields tolerated (`crates/buzz-persona/src/manifest.rs:123-130`), but no schema fetch or validation |
| Semver (`engines.buzz`, `version`) | `crates/buzz-persona/src/manifest.rs:33-40` doc; `version: String` (`:82`) | No — plain strings, no semver crate |
| ACP (Agent Client Protocol) | `crates/buzz-persona/src/resolve.rs:1-14` and field comments | No protocol code here; shape-only alignment |
| MCP (Model Context Protocol) | `McpServerConfig` (`crates/buzz-persona/src/persona.rs:70`), `.mcp.json` `mcpServers` key (`crates/buzz-persona/src/resolve.rs:285`) | Config shape only; no MCP client |


## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: Integrations

No internal workspace crates are depended on — the `[dependencies]` block
(`crates/buzz-ws-client/Cargo.toml:9`–`17`) lists only third-party crates, all
inherited via `workspace = true`. The single external system it talks to is a Nostr
relay over WebSocket.

---

### 1. External crates and how each is used

| Crate | Workspace version/features | Used for | file:line |
|---|---|---|---|
| `nostr` | `0.44`, features `nip44`, `nip98` (root `Cargo.toml:61`) | `Event`, `Keys`, `Tag` (`connection.rs:5`); `EventBuilder`, `RelayUrl` (`message.rs:1`); AUTH event construction + signing (`message.rs:181`, `:188`); `event.id.to_hex()` correlation keys (`connection.rs:80`, `:97`); `nostr::event::builder::Error` conversion (`error.rs:47`) | manifest `crates/buzz-ws-client/Cargo.toml:10` |
| `tokio` | `1`, features `rt-multi-thread, macros, net, time, sync, io-util, signal, process` (root `Cargo.toml:43`) | `tokio::time::timeout` (`connection.rs:7`, `:134`, `:187`, `:244`, `:284`); `tokio::time::Instant` deadlines (`connection.rs:176`, `:222`); `tokio::net::TcpStream` in the stream type (`connection.rs:14`) | `Cargo.toml:11` |
| `tokio-tungstenite` | `0.29`, features `rustls-tls-webpki-roots` (root `Cargo.toml:113`) | `connect_async` (`connection.rs:53`), `Message` frames (`connection.rs:124`, `:141`, `:148`, `:151`), `MaybeTlsStream`/`WebSocketStream` (`connection.rs:14`), `close(None)` (`connection.rs:116`), error type in `WsClientError::WebSocket` (`error.rs:8`) | `Cargo.toml:12` |
| `futures-util` | `0.3` (root `Cargo.toml:110`) | `SinkExt` for `send`, `StreamExt` for `next` on the WebSocket (`connection.rs:4`, `:124`, `:134`) | `Cargo.toml:13` |
| `serde_json` | `1` (root `Cargo.toml:69`) | `json!` frame construction (`connection.rs:6`, `:82`, `:98`), `to_string` (`connection.rs:122`), `from_str`/`from_value` parsing (`message.rs:63`, `:77`), `Value` as the `send_raw` parameter type (`connection.rs:121`) | `Cargo.toml:14` |
| `thiserror` | `2` (root `Cargo.toml:85`) | `#[derive(Error)]` and `#[error("…")]` messages on `WsClientError` (`error.rs:1`, `:4`–`:44`) | `Cargo.toml:15` |
| `url` | `2` (root `Cargo.toml:114`) | `url.parse::<url::Url>()` validation/normalization before dialing (`connection.rs:49`–`53`) | `Cargo.toml:16` |
| `tracing` | `0.1` (root `Cargo.toml:74`) | three `debug!` calls (`connection.rs:9`, `:57`, `:91`, `:123`) | `Cargo.toml:17` |

Not used: no `serde` derive dependency (all (de)serialization goes through
`serde_json::Value` plus the `nostr` crate's own impls), no `reqwest`/HTTP client,
no `rand`, no `anyhow`.

---

### 2. Transport and TLS configuration

- The stream type is `WebSocketStream<MaybeTlsStream<tokio::net::TcpStream>>`
  (`connection.rs:14`), i.e. TLS is optional per-connection and decided by
  `tokio-tungstenite` from the URL scheme.
- The crate calls the plain `connect_async` (`connection.rs:53`). It does **not**
  call `connect_async_tls_with_config`, does not construct a `Connector`, and passes
  no TLS config — verified: a search for `connect_async_tls_with_config`, `Connector`,
  `accept_invalid`, `rustls`, and `native_tls` inside `crates/buzz-ws-client/`
  returns no matches. So TLS behaviour is entirely the dependency default.
- Root-store selection therefore comes from the workspace feature choice
  `tokio-tungstenite = { version = "0.29", features = ["rustls-tls-webpki-roots"] }`
  (root `Cargo.toml:113`) — rustls with bundled webpki roots (not the OS trust store,
  not native-tls). This crate's own manifest inherits it with `workspace = true`
  (`crates/buzz-ws-client/Cargo.toml:12`) and adds no feature overrides.
- No WebSocket tuning is applied: no `WebSocketConfig`, no max-frame/max-message
  limits, no subprotocol or custom headers, no proxy support (verified — `connect_async`
  is invoked with a single URL argument only, `connection.rs:53`).

---

### 3. URL / scheme handling (ws vs wss)

| Behaviour | Evidence |
|---|---|
| Input is a `&str`; the only validation is `url::Url` parsing, which accepts any valid absolute URL scheme (including `http`/`https`) | `connection.rs:48`–`51` |
| **No scheme allow-list, no `ws`/`wss` check, no upgrade of `ws` → `wss`** anywhere in the crate | `connection.rs:48`–`65` (whole `connect`); no `starts_with("ws")` / scheme comparison exists |
| The dialed URL is the parsed/normalized form (`parsed.as_str()`), so `url` crate normalization (default port stripping, path defaulting to `/`) applies to the connection | `connection.rs:53` |
| The **original** string is what ends up in the AUTH event's relay tag, via `RelayUrl::parse(relay_url)` — so the AUTH-event relay value and the dialed URL can differ in normalization | store `connection.rs:63`; use `connection.rs:79` → `message.rs:180` |
| Rejection of non-WebSocket schemes is left to `tokio-tungstenite`, surfacing as `WsClientError::WebSocket` | `connection.rs:53`–`55` (behaviour of the dependency, not verified in this crate) |
| `nostr::RelayUrl::parse` may impose its own scheme rules; that logic lives in the `nostr` crate and is **not verifiable from these files** | `message.rs:180` |

---

### 4. Protocol integrations (wire level)

| Protocol surface | Direction | Evidence |
|---|---|---|
| NIP-01 `EVENT` (client→relay) | out | `connection.rs:98` |
| NIP-01 `OK` | in | `message.rs:87`–`104`; matched at `connection.rs:254` |
| NIP-01 `EVENT` (relay→client) | in | `message.rs:71`–`86` |
| NIP-01 `EOSE`, `CLOSED`, `NOTICE` | in | `message.rs:105`–`138` |
| NIP-42 `AUTH` challenge | in | `message.rs:139`–`146` |
| NIP-42 `AUTH` response (signed event) | out | `connection.rs:82` |
| "NIP-OA" authorization tag carried inside the AUTH event | out | `message.rs:169`–`186` (doc comment names it; example given as `["auth", "<token>"]`) |
| WebSocket Ping/Pong/Close control frames | both | `connection.rs:148`–`151` (and duplicates at `:208`–`:211`, `:262`–`:265`) |
| Anything else (`REQ`, `CLOSE`, `COUNT`) | out, untyped | via `send_raw` (`connection.rs:121`); e.g. `crates/buzz-test-client/src/lib.rs:154`, `:160` |

---

### 5. Consumers and parallel implementations in the repo

| Crate | Relationship | Evidence |
|---|---|---|
| `buzz-cli` | Depends on this crate; calls `publish_event` with a 75 s budget | `crates/buzz-cli/Cargo.toml:77`; `crates/buzz-cli/src/client.rs:1071`, `:1077`, `:1080` |
| `buzz-test-client` | Depends on this crate; wraps `NostrWsConnection` and re-exports its message/error types | `crates/buzz-test-client/Cargo.toml:13`; `crates/buzz-test-client/src/lib.rs:13`–`14`, `:85`, `:98` |
| `buzz-acp` | Does **not** depend on this crate; carries its own `connect_async` + NIP-42 + `RelayMessage` + parser implementation. No longer a byte-for-byte parallel: since the post-analysis sync the shared crate models a 7th inbound type (`Count`) that the ACP copy does not (`crates/buzz-acp/src/relay.rs:471`–`494` has 6 variants, no `"COUNT"` arm) | `crates/buzz-acp/src/relay.rs:3435`–`3461` (auth response), `:3610`–`3616` (AUTH parse), `:3843`–`3845` (handshake), `:2344`–`2350` (mid-session re-auth), `:471`–`494` (enum) |
| Other independent WebSocket clients in the repo (context for duplication, not dependencies) | separate implementations | `crates/buzz-relay/src/router.rs`, `crates/buzz-relay/src/audio/handler.rs`, `crates/buzz-pairing-cli/src/main.rs`, `crates/buzz-pair-relay/tests/integration.rs`, `desktop/src-tauri/src/native_websocket.rs` (all contain `connect_async`) |


## Module: buzz-db (`crates/buzz-db`)

### Aspect: Integrations

#### 1. Dependencies

Declared at `crates/buzz-db/Cargo.toml:11-25`; all versions come from the
workspace table in `Cargo.toml`.

| Crate | Workspace version / features | Used for |
|-------|------------------------------|----------|
| `buzz-core` | path `crates/buzz-core` (`Cargo.toml:124`) | `CommunityId`, `StoredEvent`, `kind::*` predicates and constants, `channel::{ChannelType, ChannelVisibility, MemberRole, canonical_channel_name}` |
| `sqlx` | `0.9`, features `runtime-tokio`, `tls-rustls`, `postgres`, `uuid`, `chrono`, `json` (`Cargo.toml:52-54`) | the only database driver |
| `tokio` | `1` (`Cargo.toml:43`) | `tokio::spawn` for the fence probe, `tokio::time::{interval, timeout}` |
| `serde` / `serde_json` | `1` (`Cargo.toml:64`, `:69`) | JSONB round-trips, `Serialize` on admin/feedback records, event reconstruction |
| `uuid` | `1`, `v4`+`serde` (`Cargo.toml:89`) | all UUID columns and `Uuid::new_v4()` id minting |
| `chrono` | `0.4`, `serde` (`Cargo.toml:90`) | `DateTime<Utc>` ↔ `TIMESTAMPTZ`, month arithmetic in the partition manager |
| `hex` | `0.4` (`Cargo.toml:97`) | pubkey/event-id hex encode for `event_mentions.pubkey_hex` and admin projections |
| `sha2` | `0.11` (`Cargo.toml:96`) | approval-token hashing (`crates/buzz-db/src/workflow.rs:33`), DM participant hash (`crates/buzz-db/src/dm.rs:48`), push advisory-lock key derivation (`crates/buzz-db/src/push.rs:221-230`) |
| `tracing` | `0.1` (`Cargo.toml:74`) | `info!` in the partition manager, `warn!`/`debug!` on best-effort paths |
| `thiserror` | `2` (`Cargo.toml:85`) | `DbError`, `ProbeError` |
| `nostr` | `0.44`, `nip44`+`nip98` (`Cargo.toml:61`) | `nostr::Event` in insert signatures; `EventBuilder`/`Keys`/`Tag`/`Kind` when the crate itself signs the NIP-43 snapshot |

Dev-dependencies: only `tokio` (`crates/buzz-db/Cargo.toml:23-24`). There is no
`[features]` section — the crate has **no cargo features**.

#### 2. Postgres / sqlx specifics

**Query construction.** Runtime `sqlx::query()` / `sqlx::query_as::<_, T>()` /
`sqlx::query_scalar::<_, T>()` only — there is **no** use of `sqlx::query!`,
`query_as!`, or `query_scalar!` anywhere in the crate, so no `.sqlx/` offline
cache is required (design note at `crates/buzz-db/src/lib.rs:10`). Dynamic
SQL uses either `sqlx::QueryBuilder` (`crates/buzz-db/src/event.rs:360`,
`:591`, `crates/buzz-db/src/feed.rs:91`, `crates/buzz-db/src/lib.rs:146`,
`crates/buzz-db/src/channel.rs:1337`, `crates/buzz-db/src/event.rs:877`, `:957`)
or `format!` + `sqlx::AssertSqlSafe` with all values still bound (15 sites; see
`buzz-db-security.md`).

**Pool configuration** (`crates/buzz-db/src/lib.rs:387-407`, defaults at `:236-249`):

| Knob | Default | Notes |
|------|---------|-------|
| `max_connections` | 20 | sized so 4 relay pods × (20 main + 5 audit) fit PG `max_connections=100` |
| `min_connections` | 2 | |
| `acquire_timeout_secs` | 3 | |
| `max_lifetime_secs` | 1800 | |
| `idle_timeout_secs` | 600 | |

The **writer** pool installs an `after_connect` hook that runs
`SELECT set_config('buzz.created_at_floor', $1, false)` with
`replica_fence::CREATED_AT_FLOOR_SECS` on every connection
(`crates/buzz-db/src/lib.rs:394-405`); the replica pool does not
(`arm_floor_guard = false` at `:363`).

**Read-replica handling.** `DbConfig::read_database_url` optionally connects a
second pool with identical sizing (`crates/buzz-db/src/lib.rs:222-234`, `:361-364`).
`Db::read()` returns the replica when configured, else the writer
(`:470-472`). The documented routing contract restricts replica use to
lag-tolerant reads; exactly two call sites route conditionally:
`get_thread_replies` (`:2004-2043`) and `get_channel_window` (`:2063-2077`).
A background probe (`replica_fence::run_probe`, spawned by
`Db::spawn_fence_probe` at `:449-467`) performs a writer→replica LSN handshake
every 5 s.

**Postgres features relied upon**

| Feature | Where |
|---------|-------|
| Declarative range partitioning + partition pruning | `migrations/0001_initial_schema.sql:235`, `:341` |
| Generated `STORED` columns | `events.search_tsv`, `migrations/0001_initial_schema.sql:222` |
| GIN indexes (`tsvector`, `jsonb_path_ops`) | `migrations/0001_initial_schema.sql:278`, `migrations/0004_events_tags_gin.sql:21` |
| Partial and expression indexes | e.g. `migrations/0001_initial_schema.sql:61`, `:102`, `:178`, `:269` |
| Enum types + `::text` / `::enum` casts | `migrations/0001_initial_schema.sql:28-37`; casts e.g. `crates/buzz-db/src/channel.rs:118` |
| `plpgsql` trigger functions, `CREATE CONSTRAINT TRIGGER … DEFERRABLE INITIALLY DEFERRED` | `migrations/0021_created_at_fence_floor.sql:70`, `migrations/0022_event_ttl_refresh.sql:37` |
| Session/transaction GUCs (`current_setting`, `set_config`) | `buzz.created_at_floor`, `buzz.nip_rs_hard_delete` |
| Advisory locks — transaction-scoped (`pg_advisory_xact_lock`, `_shared`) and session-scoped (`pg_try_advisory_lock`) | `crates/buzz-db/src/lib.rs:517-535`, `:3329`, `:3506`, `:3661`; `crates/buzz-db/src/push.rs:27`, `:232`, `:236`; `crates/buzz-db/src/relay_members.rs:446` |
| `hashtextextended(text, 0)` for lock keys derived in SQL | `migrations/0023_push_match_gate.sql:34-35`, `migrations/0024_…:31-32`, `crates/buzz-db/src/channel.rs:1132-1139` |
| `xmax = 0` upsert-winner detection | `crates/buzz-db/src/lib.rs:836` |
| `FOR UPDATE`, `FOR KEY SHARE`, `SKIP LOCKED` | `crates/buzz-db/src/relay_members.rs:460`, `crates/buzz-db/src/push.rs:259`, `:656`, `:864`, `:1044`; `migrations/0009_…:124` |
| `UNNEST(array…) AS t(...)` set-wise DML | `crates/buzz-db/src/push.rs:636-638`, `:673-681`, `:709-712` |
| `DISTINCT ON`, `FILTER (WHERE …)`, `ROW_NUMBER() OVER (PARTITION BY …)`, `json_agg`/`jsonb_build_object`, `jsonb_array_elements`, `array_position` | `crates/buzz-db/src/event.rs:1346`; `crates/buzz-db/src/usage.rs:47-48`; `crates/buzz-db/src/thread.rs:690-696`; `crates/buzz-db/src/channel.rs:900`; `crates/buzz-db/src/lib.rs:2813-2817`; `crates/buzz-db/src/channel.rs:840` |
| Catalog introspection (`pg_class`, `pg_namespace`, `pg_inherits`, `pg_trigger`, `pg_attrdef`, `information_schema.tables`, `to_regclass`) | `crates/buzz-db/src/partition.rs:108-121`; `crates/buzz-db/src/replica_fence.rs:147-172`; `migrations/0014_push_lease_fts.sql:15-21`; `crates/buzz-db/src/relay_members.rs:544-548`; `crates/buzz-db/src/migration.rs:36-38` |
| Replication views (`pg_stat_activity`, `pg_prepared_xacts`, `pg_current_wal_lsn`, `pg_last_wal_replay_lsn`, `pg_is_in_recovery`, `pg_lsn` casts) | `crates/buzz-db/src/replica_fence.rs:404-463` |
| `pgcrypto` extension for `gen_random_uuid()` | `migrations/0001_initial_schema.sql:24` |
| `LOCK TABLE … IN SHARE ROW EXCLUSIVE MODE` in migrations | `migrations/0007_nip_rs_retention.sql:12`, `migrations/0008_…:9` |
| `SET CONSTRAINTS ALL IMMEDIATE` (verification only) | `crates/buzz-db/src/replica_fence.rs:232` |

**TLS.** Supplied by sqlx's `tls-rustls` feature (`Cargo.toml:53`). The crate
itself never sets TLS options — mode is whatever the connection URL specifies.

**Migration runner.** `sqlx::migrate!("../../migrations")` embeds the SQL at
compile time (`crates/buzz-db/src/migration.rs:11`); `MIGRATOR.run(pool)` at
`:19`. Checksums are therefore frozen — every schema change must be a new
additive file, a constraint asserted throughout
`crates/buzz-db/src/migration.rs:559-830`.

#### 3. Non-Postgres I/O

None. The crate opens no sockets or files of its own, spawns no processes, and
makes no HTTP calls: the only network egress is the sqlx Postgres connection(s).
The only spawned task is `tokio::spawn(replica_fence::run_probe(...))`
(`crates/buzz-db/src/lib.rs:463-467`), which itself only talks to the two pools.
Environment variables are read **only** inside `#[cfg(test)]` modules
(see `buzz-db-configuration.md`).

#### 4. Upstream / downstream coupling

- **Upstream (consumed):** `buzz-core` only — no other Buzz crate is a
  dependency (`crates/buzz-db/Cargo.toml:11-25`).
- **Downstream (consumers of this crate):** `buzz-db` is declared in the
  workspace dependency table (`Cargo.toml:126`) and is imported by the relay and
  other service crates; per `ARCHITECTURE.md:97` the relay is the only
  orchestrator and sibling service crates do not call each other.
- **Cross-crate constant duplication:** the FTS exclusion/allowlist kind lists
  and the push-eligible kind allowlist are inlined in frozen SQL and must be
  kept in sync with `buzz_core::kind` by hand — stated at
  `migrations/0001_initial_schema.sql:214-221`,
  `migrations/0005_agent_turn_metric_fts.sql:20-24`, and
  `migrations/0018_push_match_queue.sql:22-24`. The moderation action vocabulary
  is duplicated in `crates/buzz-db/src/moderation.rs:104-118` and asserted
  against the SQL CHECK by a test at `crates/buzz-db/src/migration.rs:640-645`.
- **Sibling crate referenced from comments only:** `buzz-search`
  (`crates/buzz-search/tests/fts_integration.rs` is named as the place to add
  FTS regression tests — `migrations/0001_initial_schema.sql:220-221`).


## Module: buzz-auth (`crates/buzz-auth`)

### Aspect: Integrations

### Infrastructure reach — explicit answer

| Resource | Does this crate touch it? | Evidence |
|----------|---------------------------|----------|
| **Redis** | **No.** No `redis`, `deadpool-redis`, or any Redis client in the manifest (`crates/buzz-auth/Cargo.toml:14-26`). The crate only *formats Redis key strings* (`crates/buzz-auth/src/rate_limit.rs:201-215`, `crates/buzz-auth/src/nip98_replay.rs:114-121`) and *documents* Redis semantics that implementors must satisfy (`crates/buzz-auth/src/nip98_replay.rs:60-62`, `crates/buzz-auth/src/rate_limit.rs:3-4`). No command is ever issued from here. |
| **Postgres** | **No.** No `sqlx` and no `buzz-db` dependency (`crates/buzz-auth/Cargo.toml:14-26`). The `ChannelAccessChecker` trait exists precisely so the crate can enforce access "without depending on `buzz-db` directly" (`crates/buzz-auth/src/access.rs:3-4`, `:18-20`). |
| **Network (HTTP/WS/DNS)** | **No.** No `reqwest`, `axum`, `hyper`, `tokio-tungstenite`, or socket usage. NIP-42 verification is documented as "Pure cryptographic verification — no network calls, no JWT, no tokens" (`crates/buzz-auth/src/lib.rs:117`). `url` is used only for string parsing/normalisation (`crates/buzz-auth/src/nip42.rs:10`, `crates/buzz-auth/src/nip98.rs:28`) — it performs no resolution or I/O. `std::net::IpAddr` is used only as a key-formatting input (`crates/buzz-auth/src/rate_limit.rs:9`, `:213-215`). |
| **Filesystem** | **No.** No `std::fs`, no path handling, no config file reading. `AuthConfig` is a plain serde struct; the caller loads it (`crates/buzz-auth/src/lib.rs:89-95`, loaded at `crates/buzz-relay/src/config.rs:585`). |
| **Clock** | Yes — `nostr::Timestamp::now()` for skew checks (`crates/buzz-auth/src/nip42.rs:78`, `crates/buzz-auth/src/nip98.rs:78`). |
| **OS CSPRNG** | Yes — `rand::random::<[u8; 32]>()` for challenges (`crates/buzz-auth/src/nip42.rs:39`). |
| **Async runtime** | Yes — `tokio::task::spawn_blocking` for the CPU-bound Schnorr verify (`crates/buzz-auth/src/lib.rs:128-132`). |

Net: `buzz-auth` is a pure-compute library. Its only side effects are reading the
wall clock and drawing OS randomness.

---

### Internal dependencies

| Crate | Declared | What is used | Where |
|-------|----------|--------------|-------|
| `buzz-core` | `crates/buzz-auth/Cargo.toml:15` | `verify_event(&Event) -> Result<(), VerificationError>` — verifies event id hash then Schnorr signature (`crates/buzz-core/src/verification.rs:11-32`) | `crates/buzz-auth/src/nip42.rs:56`, `crates/buzz-auth/src/nip98.rs:74` |
| `buzz-core` | same | `TenantContext` (and `.community()`) for community scoping | `crates/buzz-auth/src/access.rs:9`, `crates/buzz-auth/src/rate_limit.rs:11`, `crates/buzz-auth/src/nip98_replay.rs:36` |
| `buzz-core` | same | `CommunityId` — test fixtures only | `crates/buzz-auth/src/rate_limit.rs:247`, `crates/buzz-auth/src/access.rs:156`, `crates/buzz-auth/src/nip98_replay.rs:144` |

`buzz-core` is the crate's only internal dependency. It does **not** depend on
`buzz-db`, `buzz-pubsub`, `buzz-relay`, or any other workspace crate.

---

### External crates and why

| Crate | Declared | Used for | Where |
|-------|----------|----------|-------|
| `nostr` | `crates/buzz-auth/Cargo.toml:16` | `Event`, `PublicKey`, `EventId`, `Kind` (`Authentication`, `HttpAuth`), `TagKind` (`Challenge`, `Relay`, `Method`, `Payload`), `SingleLetterTag`/`Alphabet` for the `u` tag, `Timestamp`, `SecretKey`, `Keys` | `crates/buzz-auth/src/nip42.rs:9`, `crates/buzz-auth/src/nip98.rs:26`, `crates/buzz-auth/src/lib.rs:66`, `crates/buzz-auth/src/nip98_replay.rs:37` |
| `serde` | `crates/buzz-auth/Cargo.toml:17` | `Serialize`/`Deserialize` on `AuthConfig`, `RateLimitConfig`, `LimitType`; `#[serde(default = ...)]` field fallbacks and `rename_all = "snake_case"` | `crates/buzz-auth/src/lib.rs:90`, `crates/buzz-auth/src/rate_limit.rs:13`, `:56-57`, `:85` |
| `serde_json` | `crates/buzz-auth/Cargo.toml:18` | parse the NIP-98 `Authorization` event JSON (`from_str`) | `crates/buzz-auth/src/nip98.rs:62`; also test serialisation `:185` |
| `tokio` | `crates/buzz-auth/Cargo.toml:19` | `task::spawn_blocking` to keep Schnorr verification off async worker threads; `#[tokio::test]` in tests | `crates/buzz-auth/src/lib.rs:128`, tests `:199`, `crates/buzz-auth/src/access.rs:163` |
| `sha2` | `crates/buzz-auth/Cargo.toml:22` | `Sha256` for the NIP-98 `payload` body hash and for the dev key derivation | `crates/buzz-auth/src/nip98.rs:27`, `:120`; `crates/buzz-auth/src/lib.rs:161-163` |
| `hex` | `crates/buzz-auth/Cargo.toml:23` | encode the 32-byte challenge and the computed body hash for comparison | `crates/buzz-auth/src/nip42.rs:40`, `crates/buzz-auth/src/nip98.rs:121` |
| `rand` | `crates/buzz-auth/Cargo.toml:24` | `rand::random::<[u8; 32]>()` challenge entropy | `crates/buzz-auth/src/nip42.rs:39` |
| `url` | `crates/buzz-auth/Cargo.toml:26` | `Url::parse` + path/host manipulation for both URL normalisers | `crates/buzz-auth/src/nip42.rs:10`, `:20-32`; `crates/buzz-auth/src/nip98.rs:28`, `:146-152` |
| `uuid` | `crates/buzz-auth/Cargo.toml:25` | `Uuid` channel ids in `AuthContext` and the access trait | `crates/buzz-auth/src/lib.rs:72`, `crates/buzz-auth/src/access.rs:11` |
| `thiserror` | `crates/buzz-auth/Cargo.toml:21` | derive `Error` + `Display` on `AuthError` | `crates/buzz-auth/src/error.rs:8` |
| `tracing` | `crates/buzz-auth/Cargo.toml:20` | **declared but unused** — no `tracing::` call, no `use tracing`, and no `#[instrument]` anywhere in `crates/buzz-auth/src/` | manifest `crates/buzz-auth/Cargo.toml:20` |

All dependencies use `{ workspace = true }` — no crate-local version pins
(`crates/buzz-auth/Cargo.toml:15-26`).

---

### Consumers of this crate

| Consumer | Declared | What it uses |
|----------|----------|--------------|
| `buzz-relay` | `crates/buzz-relay/Cargo.toml:22` (plus `features = ["dev"]` in `[dev-dependencies]`, `:90`) | `AuthService`/`AuthConfig` (`crates/buzz-relay/src/state.rs:19`, `:500`; `crates/buzz-relay/src/main.rs:368`), `AuthContext` (`crates/buzz-relay/src/connection.rs:17`, `:44`), `generate_challenge` + `verify_auth_event` (`crates/buzz-relay/src/handlers/auth.rs:87-89`, `crates/buzz-relay/src/audio/handler.rs:222`), `verify_nip98_event` (`crates/buzz-relay/src/api/bridge.rs:111`), `LimitType`/`RateLimiter` (`crates/buzz-relay/src/admission.rs:1`), `Nip98ReplayGuard` + `DEFAULT_REPLAY_TTL_SECS` (`crates/buzz-relay/src/api/bridge.rs:16`, `crates/buzz-relay/src/state.rs:582`), `Scope::all_known()` (`crates/buzz-relay/src/api/bridge.rs:829`), `RateLimitConfig` (`crates/buzz-relay/src/config.rs:284-316`) |
| `buzz-pubsub` | `crates/buzz-pubsub/Cargo.toml:12` | implements `RateLimiter` as `RedisRateLimiter` (`crates/buzz-pubsub/src/rate_limiter.rs:99-121`) and `Nip98ReplayGuard` as `RedisNip98ReplayGuard` (`crates/buzz-pubsub/src/nip98_replay.rs:34`); calls `rate_limit_key` / `ip_rate_limit_key` (`crates/buzz-pubsub/src/rate_limiter.rs:108`, `:118`) |
| `buzz-admin` | `crates/buzz-admin/Cargo.toml:17` | **nothing** — a grep of `crates/buzz-admin/` for `buzz_auth` returns zero matches; the dependency appears unused |

`crates/buzz-conformance/Cargo.toml:18` carries an explicit prohibition comment
naming `buzz-auth` among crates it must never depend on.

---

### Protocol / spec conformance surface

| Spec | Kind | Where implemented |
|------|------|-------------------|
| NIP-42 (client authentication of clients to relays) | `Kind::Authentication` = 22242 | `crates/buzz-auth/src/nip42.rs:52`; module doc describes the 3-step handshake at `:1-7` |
| NIP-98 (HTTP Auth) | `Kind::HttpAuth` = 27235 | `crates/buzz-auth/src/nip98.rs:66`; `Authorization: Nostr <base64(event)>` header shape documented at `:9-11` (base64 decoding is the caller's job, `:38`) |
| NIP-OA (owner attestation) | — | not implemented here; only the `AuthContext.agent_owner_pubkey` field is reserved for the relay to fill (`crates/buzz-auth/src/lib.rs:75-79`). The tag extraction/verification lives in `crates/buzz-relay/src/handlers/auth.rs:26-36`. |
| NIP-29 (relay-based groups) | — | referenced only as the justification for granting full scopes (`crates/buzz-auth/src/lib.rs:135`) |

The `nostr` workspace dependency enables the `nip98` feature explicitly
(`Cargo.toml:61`), which is what provides `TagKind::Method` / `TagKind::Payload`.


## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Integrations

---

### 1. External systems

| System | Protocol | Client | Usage |
|---|---|---|---|
| Redis | RESP over TCP | `deadpool-redis` pool (`Cargo.toml:14`) | All imperative commands: `PUBLISH`, `SET`, `GET`, `MGET`, `DEL`, `TTL`, `EXPIRE`, `EVALSHA` |
| Redis | RESP pub/sub mode | `redis::Client` → `get_async_pubsub()` (`Cargo.toml:13`) | Three independent stateful connections: events (`subscriber.rs:85-87`), cache-invalidate (`cache_invalidation.rs:135-139`), conn-control (`conn_control.rs:126-130`) |

Redis is the **only** external system. No Postgres, no HTTP client, no object
store, no message broker. The crate declares no `buzz-db`, `reqwest`, `axum`, or
`sqlx` dependency (`Cargo.toml:11-24`).

Redis command inventory (exhaustive, by call site):

| Command | Site |
|---|---|
| `PUBLISH` | `publisher.rs:32-36`; `lib.rs:276-280` (cache), `:296-300` (conn-control) |
| `SET … EX` | `presence.rs:37-43` |
| `SET … NX EX` | `nip98_replay.rs:68-78` |
| `DEL` | `presence.rs:53-56` |
| `GET` | `presence.rs:69` |
| `MGET` | `presence.rs:88` |
| `EXPIRE` (repair) | `rate_limiter.rs:60-65` |
| `INCR` + `EXPIRE` + `TTL` (in Lua) | `rate_limiter.rs:24-31`, invoked `:38-42` |
| `SUBSCRIBE` / `UNSUBSCRIBE` | `subscriber.rs:98`, `:119`, `:127` |
| `PSUBSCRIBE` | `cache_invalidation.rs:139`, `conn_control.rs:130` |
| `TTL` (assertions only) | `lib.rs:491`, `presence.rs:196` — test code |

Four connections per relay pod minimum: one pool (shared, size set by the caller)
plus three dedicated pub/sub sockets. The pool is **injected**, never created here —
every constructor takes `deadpool_redis::Pool` as a parameter (`lib.rs:117`, `:122`,
`rate_limiter.rs:94`, `nip98_replay.rs:29`), so pool sizing and TLS configuration
are the relay's concern. Only the pub/sub connections are opened from the crate's
own `redis_url` string (`lib.rs:105`).

### 2. Internal crate dependencies

| Dependency | Declared | What is actually used |
|---|---|---|
| `buzz-core` | `Cargo.toml:11` | `TenantContext` (`topic.rs:6`, `presence.rs:7`, `lib.rs:50`), `CommunityId` (`topic.rs:6`, `cache_invalidation.rs:17`, `conn_control.rs:20`) |
| `buzz-auth` | `Cargo.toml:12` | `RateLimiter` / `LimitType` / `RateLimitResult` / `rate_limit_key` / `ip_rate_limit_key` (`rate_limiter.rs:12-15`, `:110`, `:118`); `Nip98ReplayGuard` / `nip98_replay_key_for_scope` / `DEFAULT_REPLAY_TTL_SECS` / `MAX_REPLAY_TTL_SECS` (`nip98_replay.rs:8-14`, `:81`); `AuthError` (`rate_limiter.rs:13`, `nip98_replay.rs:9`) |
| `nostr` | `Cargo.toml:22` | `Event` + `JsonUtil` for wire (de)serialization (`publisher.rs:4`, `subscriber.rs:8`), `PublicKey` (`presence.rs:9`), `EventId` (`nip98_replay.rs:16`) |
| `uuid` | `Cargo.toml:17` | Channel and community ids (`topic.rs:7`) |
| `futures-util` | `Cargo.toml:23` | `StreamExt` for pub/sub stream consumption + sink/stream split (`subscriber.rs:7`, `:86`) |
| `serde` / `serde_json` | `Cargo.toml:15-16` | Control-message payloads (`cache_invalidation.rs:18`, `conn_control.rs:21`) |
| `tokio` | `Cargo.toml:14` | `broadcast`, `mpsc`, `Mutex`, `select!`, `sleep`, `spawn` |
| `tracing` | `Cargo.toml:20` | Structured logging throughout |
| `thiserror` | `Cargo.toml:21` | `PubSubError` derive (`error.rs:1`) |
| `chrono` | `Cargo.toml:18` | **Declared but no `chrono::` path appears in any source file** — unused |

`buzz-auth` is a genuine inbound coupling: this crate exists partly to provide the
Redis-backed implementations of two `buzz-auth` traits, which inverts the usual
layering (a lower-level transport crate depending on the auth crate). Noted as a
structural observation, not a defect — the alternative would be a `buzz-auth`
dependency on Redis.

### 3. Consumers

| Consumer | Manifest | Verified code usage |
|---|---|---|
| `buzz-relay` | `crates/buzz-relay/Cargo.toml` | Extensive — see table below |
| `buzz-admin` | `crates/buzz-admin/Cargo.toml` | Declared; no `buzz_pubsub::` path found in the relay-side grep sweep of `crates/**/*.rs` outside `buzz-relay` and `buzz-test-client` comments |
| `buzz-conformance` | `crates/buzz-conformance/Cargo.toml` | Declared; same — no verified call site |

Relay integration points (all verified by grep against `crates/**/*.rs`):

| Seam | Relay site |
|---|---|
| Manager construction | `buzz-relay/src/state.rs` (imports `state.rs:27`) |
| `run_conn_control_subscriber` spawn | `main.rs:366` |
| `subscribe_local` | `main.rs:822`, `handlers/event.rs:1644` |
| `subscribe_conn_control` | `main.rs:903`; dispatch `main.rs:908`, `:913` |
| `retain_topic` | `handlers/req.rs:256`, `handlers/event.rs:1683`, `:1687` |
| `release_topic` | `connection.rs:268`, `handlers/close.rs:21`, `handlers/req.rs:251`, `handlers/side_effects.rs:81` |
| `publish_cache_invalidation` | `state.rs:970` |
| `publish_conn_control` | `state.rs:1044`, `:1066` |
| `set_presence` / `clear_presence` | `handlers/event.rs:798` / `connection.rs:280`, `handlers/event.rs:793` |
| `get_presence_bulk` | `api/bridge.rs:1972` |
| `RedisRateLimiter` | import `state.rs:26`, field `state.rs:584`, construction `state.rs:712`, enforcement `connection.rs:593-648` via `admission.rs:14-34` |
| `RedisNip98ReplayGuard` | import `state.rs:27`, construction `state.rs:711`, two-pod tests `api/bridge.rs:2293-2294`, `:2304` |

Cross-node behaviour is additionally pinned from outside the crate by
`crates/buzz-test-client/tests/conformance_multitenant.rs:2371` and `:2484`, which
document that presence answers come from Redis via `get_presence_bulk` with **no DB
fallback**, and that community A's query must return only A's data.

### 4. Contract boundaries and coupling risks

- **Redis is a hard availability dependency for authenticated traffic.** A Redis
  outage makes `check_and_increment` return `AuthError::Internal`
  (`rate_limiter.rs:44-46`), which the relay maps to `AdmissionError::Unavailable`
  (`admission.rs:29-33`) and treats as denial — so every authenticated WS
  `EVENT`/`REQ`/`COUNT` is rejected (`connection.rs:612-621`). Fail-closed is the
  correct security posture, but it means Redis is on the critical path for reads,
  not just for fan-out.
- **Shared Redis across all tenants.** Isolation is by key prefix only
  (`topic.rs:13`), and two subscribers use cross-tenant `buzz:*` patterns
  (`cache_invalidation.rs:27`, `conn_control.rs:30`). Any process with access to the
  Redis instance sees every community's traffic.
- **No schema versioning on control payloads.** `CacheInvalidation`
  (`cache_invalidation.rs:57-80`) and `ConnControl` (`conn_control.rs:55-73`) are
  internally tagged with no version field and are not `#[non_exhaustive]`, so a
  rolling deploy that adds a variant produces `warn`-level deserialization failures
  on old pods (`cache_invalidation.rs:161-165`, `conn_control.rs:152-156`) —
  silently skipped, which for `ConnControl` means a ban may not be enforced on
  older pods until they restart. Mitigated by the DB backstop
  (`conn_control.rs:18-21`) but not by the transport.
- **Event payloads are whole Nostr events**, so Redis bandwidth scales with event
  size, and the pub/sub message size limit becomes an implicit cap on event size
  (`publisher.rs:31`).


## Module: buzz-search (`crates/buzz-search`)

### Aspect: Integrations

#### Dependencies — `crates/buzz-search/Cargo.toml`

| Dependency | Declaration | Line | Workspace version/features |
|---|---|---|---|
| `buzz-core` | `{ workspace = true }` | `Cargo.toml:11` | path `crates/buzz-core` (root `Cargo.toml:124`) |
| `sqlx` | `{ workspace = true }` | `Cargo.toml:12` | `0.9`, features `runtime-tokio, tls-rustls, postgres, uuid, chrono, json` (root `Cargo.toml:52-54`) |
| `uuid` | `{ workspace = true }` | `Cargo.toml:13` | `1`, features `v4, serde` (root `Cargo.toml:89`) |
| `thiserror` | `{ workspace = true }` | `Cargo.toml:14` | `2` (root `Cargo.toml:85`) |
| `tokio` (dev only) | `{ workspace = true }` | `Cargo.toml:17` | `1`, multi-thread/macros/etc. (root `Cargo.toml:43`) |

Four runtime dependencies total. No HTTP client, no serde, no tracing, no Redis, no
S3. Package description: "Postgres full-text search for Buzz, scoped by community"
(`Cargo.toml:8`).

#### buzz-core usage

| Item | Import | Use |
|---|---|---|
| `buzz_core::CommunityId` | `query.rs:14`, re-exported at `lib.rs:29` | `SearchQuery.community` field type (`query.rs:76`); `as_uuid()` for the SQL bind (`query.rs:241`, definition at `crates/buzz-core/src/tenant.rs:47-49`) |

That is the entire coupling to `buzz-core` in `src/`. Tests additionally import
`buzz_core::kind::{AUTHOR_ONLY_KINDS, KIND_AGENT_TURN_METRIC, KIND_MEMBER_ADDED_NOTIFICATION, KIND_MEMBER_REMOVED_NOTIFICATION, P_GATED_KINDS}`
and `buzz_core::kind::is_ephemeral` (`tests/fts_integration.rs:9-16`, `1382`) as
drift tripwires against the schema's hard-coded exclusion list.

#### sqlx / Postgres usage

| Aspect | Detail | Line |
|---|---|---|
| Imports | `sqlx::{PgPool, QueryBuilder, Row}` in query, `sqlx::PgPool` in lib | `query.rs:15`, `lib.rs:33` |
| Query style | Runtime `QueryBuilder<sqlx::Postgres>` — **not** the compile-time `sqlx::query!` macros; no `.sqlx/` offline cache is used by this crate | `query.rs:233` |
| Binding | `push_bind` for every dynamic value: community UUID (`241`), prefix/fulltext term (`144`, `168`), channel id vec (`257`, `262`), kinds vec (`270`), authors vec (`278`), since (`285`), until (`291`), limit (`296`), offset (`298`) | as listed |
| Execution | `qb.build().fetch_all(pool).await?` — single round trip, no transaction, no prepared-statement caching directives | `query.rs:300` |
| Row decoding | `Row::try_get` by column name for all six columns | `query.rs:304-318` |
| Postgres-specific SQL | `to_timestamp`, `EXTRACT(EPOCH FROM ...)::bigint`, `websearch_to_tsquery`, `to_tsvector`, `tsvector_to_array`, `ts_rank_cd`, `@@`, `= ANY(...)`, `CROSS JOIN LATERAL`, `regexp_split_to_table ... WITH ORDINALITY`, `string_agg`, `quote_literal`, `::tsquery` | `query.rs:143`, `154-176`, `234-298` |
| Types crossing the boundary | `Uuid` ↔ `uuid`, `Vec<u8>` ↔ `BYTEA`, `i32` ↔ `INT`, `i64` ↔ `BIGINT`, `f32` ↔ `real` (`ts_rank_cd` result), `Option<Uuid>` ↔ nullable `uuid` | `query.rs:304-318` |

#### Pool handling

`SearchService` stores an owned `PgPool` clone (`lib.rs:41`, `lib.rs:46-48`). The
crate never creates, configures, closes, or resizes a pool — no `PgPoolOptions` in
`src/`. `PgPool` is internally `Arc`-based, so `#[derive(Clone)]` on
`SearchService` (`lib.rs:39`) shares the same pool. The relay wraps it in
`Arc<SearchService>` in `AppState` and constructs it from the relay's pool
(`crates/buzz-relay/src/state.rs:1273`).

#### Non-Postgres I/O

None. No filesystem access, no network client, no environment reads, no process
spawning, no clock or RNG use in `src/`. The only `std::env::var` calls in the
crate are in the test harness (`tests/fts_integration.rs:33`, `92`).

#### Typesense status — explicit check

| Question | Answer | Evidence |
|---|---|---|
| Typesense dependency in `Cargo.toml`? | No | `Cargo.toml:10-18` lists only buzz-core, sqlx, uuid, thiserror, tokio |
| Typesense client/HTTP code anywhere in the crate? | No | grep for `typesense` (case-insensitive) across the crate returns 2 hits, both prose in doc comments |
| Remaining mentions | Two, historical | `query.rs:20` ("the legacy … matrix from the Typesense relay"), `query.rs:46` ("what the legacy Typesense `channel_id:=__global__` sentinel meant") |

Related historical prose outside the crate (not code): the schema comment
"Full-text search vector (Typesense → Postgres FTS)"
(`migrations/0001_initial_schema.sql:200`, `schema/schema.sql:199`).

#### Test-harness integrations

The integration test file couples directly to migration SQL at compile time via
`include_str!` (`tests/fts_integration.rs:22-32`): migrations `0001`–`0008` plus
`0014`, applied in order into a per-test schema (`tests/fts_integration.rs:57-84`).
It creates and drops a uniquely-named schema through a separate one-connection
admin pool (`tests/fts_integration.rs:36-46`, `87-103`) and passes the schema via
the connection URL option `options=-c search_path=<schema>`
(`tests/fts_integration.rs:48`). Adding a future FTS-affecting migration requires
editing this list — noted in-file at `tests/fts_integration.rs:55-56`.


## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Integrations

### Declared dependencies (`crates/buzz-audit/Cargo.toml:11-22`)

| Crate | Declared line | Workspace version (`Cargo.toml`) | Actually used in `src`? | Where |
|---|---|---|---|---|
| `buzz-core` | `Cargo.toml:11` | path `crates/buzz-core` (`Cargo.toml:130`) | Yes — `CommunityId` | `entry.rs:1`, `service.rs:7` |
| `sqlx` | `:12` | `0.9`, features `runtime-tokio, tls-rustls, postgres, uuid, chrono, json` (`Cargo.toml:52-54`) | Yes | `service.rs:3` (`Acquire, PgPool, Row`), `:84`, `:238` |
| `tokio` | `:13` | `1` (`Cargo.toml:43`) | **Tests only** | `service.rs:265` (`tokio::sync::Mutex`), `#[tokio::test]` at `:318,338,376,437,475,512` |
| `serde` | `:14` | `1` + derive (`Cargo.toml:64`) | Yes — derives | `action.rs:1,6-7`, `entry.rs:3,13` |
| `serde_json` | `:15` | `1` (`Cargo.toml:69`) | Yes | `entry.rs:34`, `hash.rs:80-116`, `error.rs:40` |
| `uuid` | `:16` | `1` + v4, serde (`Cargo.toml:89`) | Yes | `entry.rs:4,16`, `service.rs:5,246`, `hash.rs:45` (`Uuid::as_bytes`) |
| `chrono` | `:17` | `0.4` + serde (`Cargo.toml:90`) | Yes | `entry.rs:2,36`, `hash.rs:1,22-24`, `service.rs:1,21` |
| `tracing` | `:18` | `0.1` (`Cargo.toml:74`) | Yes | `service.rs:4` (`debug, instrument, warn`), `:52`, `:128`, `:159`, `:211`, `:241` |
| `thiserror` | `:19` | `2` (`Cargo.toml:85`) | Yes | `error.rs:1,11` |
| `sha2` | `:20` | `0.11` (`Cargo.toml:96`) | Yes | `hash.rs:2` (`Digest, Sha256`), `:43`, `:72` |
| `hex` | `:21` | `0.4` (`Cargo.toml:97`) | **No** — grep for `hex::` in `crates/buzz-audit/src` returns nothing | unused declaration |
| `futures-util` | `:22` | `0.3` (`Cargo.toml:110`) | Yes — `FutureExt::catch_unwind` | `service.rs:2`, `:68` |

No `[features]`, `[dev-dependencies]`, or `[build-dependencies]` sections exist in
`crates/buzz-audit/Cargo.toml`.

### Postgres / sqlx usage

- Connection handle is a `PgPool` held by value in `AuditService` (`service.rs:37-45`).
  The crate never creates the pool; the relay passes one in
  (`crates/buzz-relay/src/main.rs:321-330`, a dedicated pool with
  `max_connections(5)`, `min_connections(1)`).
- Write path pins a single pooled connection: `self.pool.acquire()` (`service.rs:54`),
  used for the lock (`:59-62`), the transaction (`:87`), and the unlock (`:71-74`) —
  necessary because advisory locks are session-scoped (`service.rs:49-51`).
- Transaction via `Acquire::begin` (`service.rs:87`) and `commit` (`:149`). No explicit
  rollback call; a dropped `Transaction` rolls back implicitly on the error paths
  (`service.rs:101`, `:126`, `:147`).
- Reads run directly on `&self.pool` with `fetch_all` (`service.rs:178`, `:231`).
- All statements are untyped `sqlx::query` with bind parameters (no `query!`/`query_as!`
  macros), so there is **no compile-time schema verification** and column access is
  runtime `Row::get` (`service.rs:105-106`, `:246-254`).
- Postgres-specific SQL used: `pg_advisory_lock`, `pg_advisory_unlock`,
  `hashtextextended($1, 0)` (`service.rs:59`, `:71`). `hashtextextended` is a Postgres
  internal hash function — this crate is not portable to other engines.
- Types crossing the boundary: `UUID`↔`Uuid`, `BIGINT`↔`i64`, `BYTEA`↔`Vec<u8>`,
  `VARCHAR`↔`String`/`&str`, `JSONB`↔`serde_json::Value`, `TIMESTAMPTZ`↔`DateTime<Utc>`
  (binds at `service.rs:137-145`; decodes at `:246-254`).

### Cryptography

`sha2::Sha256` only, used incrementally (`Sha256::new`, `update`, `finalize`) in
`compute_hash` (`hash.rs:43-72`). No HMAC, no signatures, no randomness, no key
material anywhere in the crate.

### Non-Postgres I/O

None. The crate performs no filesystem, network, HTTP, S3, Redis, or process I/O.
The only environment read is `DATABASE_URL` inside a test helper
(`service.rs:275-279`). Logging goes through `tracing` macros only.

### How the relay integrates it (fire-and-forget semantics)

| Step | Location | Behaviour |
|---|---|---|
| Construction gate | `crates/buzz-relay/src/main.rs:321-334` | `AuditService` built only when `config.audit_enabled`; otherwise `None` and an info log |
| Enabled gauge | `crates/buzz-relay/src/main.rs:139` | `buzz_audit_enabled` set to 1.0/0.0 |
| Queue | `crates/buzz-relay/src/state.rs:654` | bounded `mpsc::channel::<NewAuditEntry>(1000)`; `audit_tx: Option<Sender<...>>` (`state.rs:555`) |
| Producer (events) | `crates/buzz-relay/src/handlers/event.rs:540-577` | uses `send().await` (backpressure, not drop); on closed channel logs `error!` and increments `buzz_audit_send_errors_total` (`:575-577`) |
| Producer (media) | `crates/buzz-relay/src/api/media.rs:422-442` | same pattern; upload still returns `Ok` even if the audit send fails (`media.rs:443`) |
| Worker | `crates/buzz-relay/src/state.rs:657-690` | single task; select over `recv()` and a `CancellationToken`; on cancel closes the receiver and drains buffered entries |
| Failure handling | `crates/buzz-relay/src/state.rs:1199-1207` | `audit.log(entry)` error → `buzz_audit_log_errors_total` + `tracing::error!`; **no retry, no dead-letter**; success → `buzz_audit_log_seconds` histogram |
| Shutdown | `crates/buzz-relay/src/state.rs:632-636`, `:680-689` | `AuditShutdownHandle::drain()` flushes queued entries; a timeout path logs "Audit worker did not drain in time — exiting anyway" (`state.rs:1190-1191`) |

Consequences visible in code: a DB outage causes queued entries to be lost after one
failed attempt (`state.rs:1201-1203`), and because `log` is the only path that assigns
`seq`, a lost entry leaves **no gap** in the chain — the next successful append simply
takes the next `seq`. The chain therefore stays verifiable while being incomplete; the
crate offers no way to detect that an entry was dropped.

### Other repo touch points

- `crates/buzz-admin/Cargo.toml:20` declares the dependency, but grep for `audit` in
  `crates/buzz-admin/src` returns nothing — no operator CLI surface consumes
  `verify_chain`/`get_entries` today.
- `migrations/0023_push_match_gate.sql:21` references the `'buzz_audit:'` advisory-lock
  namespace in a comment about lock families.
- `crates/buzz-test-client/tests/conformance_multitenant.rs:2665-2710` documents that
  audit is deliberately *not* black-box testable over the wire and defers to this
  crate's own tests.
- `crates/buzz-conformance/Cargo.toml:19` explicitly excludes `buzz-audit` from the
  conformance checker's dependency set.


## Module: buzz-media (`crates/buzz-media`)

### Aspect: Integrations

### 1. Dependency inventory (`crates/buzz-media/Cargo.toml:10-35`)

| Crate | Version / spec | Purpose in this crate |
|---|---|---|
| `buzz-core` | workspace | `tenant::{CommunityId, TenantContext, normalize_host}` (`crates/buzz-media/src/storage.rs:6`, `crates/buzz-media/src/auth.rs:170`) |
| `nostr` | workspace | `Event`, `PublicKey`, `Timestamp`, `ToBech32` for Blossom auth + records (`crates/buzz-media/src/auth.rs:33`, `crates/buzz-media/src/upload_record.rs:145`) |
| `s3` = `rust-s3` | **0.37**, `default-features = false`, features `tokio-rustls-tls`, `fail-on-err`, `tags` (`crates/buzz-media/Cargo.toml:24`); resolved **0.37.2** (`Cargo.lock:7432-7434`) | all object storage |
| `infer` | **0.19** | magic-byte MIME sniffing (`crates/buzz-media/src/validation.rs:239`, `:176`) |
| `image` | **0.25**, `default-features = false`, features `jpeg`, `png`, `gif`, `webp` | full decode + thumbnail JPEG encode (`crates/buzz-media/src/thumbnail.rs:26-32`) |
| `imagesize` | **0.14** | header-only dimension parse for the pixel-bomb guard (`crates/buzz-media/src/validation.rs:270`) |
| `blurhash` | **0.2** | 4×3 blurhash from the thumbnail (`crates/buzz-media/src/thumbnail.rs:36-37`) |
| `mp4` | **0.14** | MP4 header/track parsing (`crates/buzz-media/src/validation.rs:307-386`) |
| `sha2` + `hex` | workspace | SHA-256 content addressing (`crates/buzz-media/src/upload.rs:4`, `:84`, `:397`) |
| `tempfile` | **3** | `NamedTempFile` staging for streamed video (`crates/buzz-media/src/upload.rs:307`) |
| `tokio` | workspace | async fs/IO, `spawn_blocking` (`crates/buzz-media/src/upload.rs:79`, `:410`, `:418`) |
| `tokio-util` | **0.7**, feature `io` | `StreamReader` to adapt the axum body stream (`crates/buzz-media/src/upload.rs:325`) |
| `futures-util` / `futures-core` | **0.3** | `StreamExt::map`, stream trait bounds (`crates/buzz-media/src/storage.rs:141`, `crates/buzz-media/src/upload.rs:298`) |
| `bytes` | **1** | `Bytes` bodies / chunks (`crates/buzz-media/src/upload.rs:3`) |
| `axum` | workspace | only `StatusCode`/`IntoResponse` (`crates/buzz-media/src/error.rs:3-4`) and `http::HeaderName` validation (`crates/buzz-media/src/config.rs:118`) |
| `ulid` | **1** | upload-record ids (`crates/buzz-media/src/upload_record.rs:150`) |
| `uuid` | workspace | community UUID parsing in key classification (`crates/buzz-media/src/bucket_index.rs:22`, `:112-127`) |
| `chrono` | workspace | `Utc::now().timestamp()` for `uploaded_at` (`crates/buzz-media/src/upload.rs:113`, `:132`) |
| `serde` / `serde_json` | workspace | sidecar + record JSON (`crates/buzz-media/src/storage.rs:199`, `:218`) |
| `thiserror` | workspace | `MediaError`, `SweepError` (`crates/buzz-media/src/error.rs:7`, `crates/buzz-media/src/bucket_index.rs:340`) |
| `tracing` | workspace | 3 log sites (`crates/buzz-media/src/upload.rs:135`, `crates/buzz-media/src/error.rs:135`, `:155`) |

Dev-dependencies: `tokio` with `test-util` only (`crates/buzz-media/Cargo.toml:33-34`). No `mockall`, no `wiremock`, no `testcontainers`.

---

### 2. S3 client configuration

| Aspect | Behaviour | file:line |
|---|---|---|
| Region/endpoint | `Region::Custom { region: config.s3_region, endpoint: config.s3_endpoint }` — always a custom region, even for real AWS | `crates/buzz-media/src/storage.rs:35-38` |
| Path style | **Always** `.with_path_style()` — unconditional, not gated by a flag | `crates/buzz-media/src/storage.rs:66-68` |
| Bucket | `Bucket::new(&config.s3_bucket, region, creds)` | `crates/buzz-media/src/storage.rs:66` |
| TLS | `tokio-rustls-tls` feature (no native-tls) | `crates/buzz-media/Cargo.toml:24` |
| HTTP error strictness | `fail-on-err` feature — non-2xx responses surface as `S3Error` rather than being silently returned | `crates/buzz-media/Cargo.toml:24` |
| Signing region correctness | Documented requirement: `s3_region` must match the endpoint's region for real AWS, else SigV4 credential scope is wrong; default `us-east-1` preserves MinIO behaviour | `crates/buzz-media/src/config.rs:29-37`, `crates/buzz-media/src/config.rs:11-13` |

**MinIO compatibility** is achieved by (a) unconditional path-style addressing, (b) a custom `Region` with an explicit endpoint, and (c) tolerating a non-meaningful region value ("Defaults to 'us-east-1' to preserve MinIO/local behavior, where the value is not meaningfully checked" — `crates/buzz-media/src/config.rs:33-37`). The live round-trip test targets `http://localhost:9000` with creds `buzz_dev`/`buzz_dev_secret`, bucket `buzz-media` (`crates/buzz-media/tests/static_creds_minio.rs:22-34`).

---

### 3. Credential sources

Two mutually exclusive modes, chosen by whether both static keys are non-empty (`crates/buzz-media/src/storage.rs:39-65`):

| Case | Behaviour | file:line |
|---|---|---|
| both keys non-empty | `Credentials::new(Some(access), Some(secret), None, None, None)` — static keys, no environment/metadata access | `crates/buzz-media/src/storage.rs:44-50` |
| both keys empty | `Credentials::default()` — AWS default chain: environment, shared profile, **web-identity token (IRSA on EKS, `AssumeRoleWithWebIdentity`)**, container, instance metadata | `crates/buzz-media/src/storage.rs:51-56`, documented `crates/buzz-media/src/storage.rs:25-33` |
| exactly one key set | hard error `StorageError("s3_access_key and s3_secret_key must be configured together, or both empty to use the AWS credential chain")` — never silently falls back | `crates/buzz-media/src/storage.rs:57-62`; test `crates/buzz-media/src/storage.rs:312-331` |

**Patched `aws-creds` fork** (workspace-level, applies transitively to this crate): `Cargo.toml:170-171` pins `aws-creds` to `git+https://github.com/tlongwell-block/rust-s3` rev `c9fce3620dd434c1f810101d672cf384268dbb0f` (`Cargo.lock:422-424`). The reason recorded at `Cargo.toml:163-169`: aws-creds 0.39.1 cannot read **EKS Pod Identity** credentials (`AWS_CONTAINER_CREDENTIALS_FULL_URI` + `AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE`), which the relay pod requires for S3 media + git storage; the fork adopts the aws-creds portion of `durch/rust-s3#449` (FULL_URI + token-file + Authorization header, refresh-safe, with a loopback allowlist for the auth token). Marked temporary — revert when #449 lands upstream.

No credential values are logged anywhere in the crate; the only `tracing` calls are the orphan-blob warning (`crates/buzz-media/src/upload.rs:135`) and the two error-mapping logs (`crates/buzz-media/src/error.rs:135`, `:155`).

---

### 4. Media-parsing integrations (in-process)

| Library | Where it runs | Input bound before it runs |
|---|---|---|
| `infer::get` | image path (`crates/buzz-media/src/validation.rs:239`), generic path (`crates/buzz-media/src/validation.rs:176`) | full buffered body |
| `imagesize::blob_size` | image path, before full decode (`crates/buzz-media/src/validation.rs:270`) | size cap already applied (`crates/buzz-media/src/validation.rs:259-268`) |
| `image::load_from_memory` (full pixel decode) | `generate_image_metadata_sync`, inside `spawn_blocking` (`crates/buzz-media/src/thumbnail.rs:26`, `crates/buzz-media/src/upload.rs:518-522`) | byte cap + 25 MP pixel cap |
| `image` thumbnail + JPEG encode | `crates/buzz-media/src/thumbnail.rs:30-32` | as above |
| `blurhash::encode` | `crates/buzz-media/src/thumbnail.rs:36-37` — failure swallowed via `unwrap_or_default()` | operates on the ≤320px thumbnail |
| `mp4::Mp4Reader::read_header` | `validate_video_file`, inside `spawn_blocking` (`crates/buzz-media/src/validation.rs:307`, `crates/buzz-media/src/upload.rs:416-419`) | byte cap, ISO-BMFF check, moov-order scan, box allowlist walk |

Custom hand-rolled parsers (no external crate): JPEG marker walk, PNG chunk walk, WebP RIFF walk, GIF block walk, MP4 box walk, top-level atom scan — `crates/buzz-media/src/validation.rs:502-928`.

---

### 5. Retry / timeout behaviour

| Aspect | Finding | file:line |
|---|---|---|
| Retries on S3 failure | **None** in this crate — every storage call maps the first error straight to `MediaError` | `crates/buzz-media/src/storage.rs:73-265` |
| Timeouts on S3 calls | **None set** in this crate; whatever `rust-s3`/`reqwest` defaults apply | `crates/buzz-media/src/storage.rs:34-70` (no timeout config on the client) |
| Sweep timeout | Only a *variant* to represent it — `SweepError::Timeout(Duration)`, constructed by the relay's sweep task which wraps the fold in `tokio::time::timeout` | `crates/buzz-media/src/bucket_index.rs:349-357` |
| Sweep object cap | Enforced by the caller-supplied `cap`, checked before folding each page | `crates/buzz-media/src/bucket_index.rs:394-398` |
| Listing page size | Caller-supplied `max_keys`, bounds one HTTP response only | `crates/buzz-media/src/storage.rs:236-241` |

---

### 6. Error handling on storage failures

| Situation | Result | file:line |
|---|---|---|
| Any `S3Error` | `MediaError::StorageError(e.to_string())` via `From` | `crates/buzz-media/src/error.rs:94-98` |
| 404 on `get`/`get_range` | `MediaError::NotFound` (special-cased on `HttpFailWithBody(404, _)`) | `crates/buzz-media/src/storage.rs:106-110`, `:119-123` |
| 404 on `head`/`head_with_metadata` | `Ok(false)` / `Ok(None)` — not an error | `crates/buzz-media/src/storage.rs:150-154`, `:168-174` |
| 404 on `get_stream` | checked via `response.status_code == 404` (different mechanism than the others) | `crates/buzz-media/src/storage.rs:136-139` |
| Sidecar read failure vs absence | Deliberately collapsed to `None` in `read_sidecar_mime` so a cross-community request cannot distinguish a foreign blob from a missing one | `crates/buzz-media/src/storage.rs:222-233` |
| Sidecar JSON parse failure | `MediaError::StorageError` (via `From<serde_json::Error>`) | `crates/buzz-media/src/error.rs:100-104` |
| `StorageError`/`Io`/`Internal` → HTTP | logged at `error` level, response body flattened to `"internal error"` | `crates/buzz-media/src/error.rs:154-158` |
| Blob PUT succeeded but metadata failed | Error propagates; blob intentionally left orphaned (no compensating delete) | `crates/buzz-media/src/upload.rs:131-141` |
| Upload-record PUT failed | Error propagates and the upload fails **before** the sidecar publish, so media is never served unscanned | `crates/buzz-media/src/upload.rs:154-172`, `crates/buzz-media/src/upload_record.rs:132-138` |
| `spawn_blocking` join failure | `MediaError::Internal` | `crates/buzz-media/src/upload.rs:87-88`, `:414`, `:418-419`, `:523-524` |
| Axum body-limit error inside the video stream | Detected by matching three Display substrings (`length limit`, `body limit`, `LengthLimitError`), converted to `ErrorKind::WriteZero` → `FileTooLarge` (413 instead of 500) | `crates/buzz-media/src/upload.rs:328-341`, `crates/buzz-media/src/upload.rs:358-366` |

---

### 7. Consumers / integration boundary

| Consumer | How it integrates | file:line |
|---|---|---|
| `buzz-relay` HTTP handlers | Owns routes `PUT /upload`, `PUT /media/upload`, `GET|HEAD /media/{sha256_ext}` and the `RequestBodyLimitLayer` | `crates/buzz-relay/src/router.rs:33-45` |
| `buzz-relay` upload dispatch | Sniffs 4096 bytes, uses `buzz_media::looks_like_iso_bmff` to route to the video pipeline, else buffers and picks image vs generic-file path | `crates/buzz-relay/src/api/media.rs:47-51`, `:317-399` |
| `buzz-relay` read path | `read_sidecar_mime`, `get_stream`, `get_range`, and `buzz_media::serve_inline` for `Content-Disposition` | `crates/buzz-relay/src/api/media.rs:633-740` |
| `buzz-relay` config | Constructs `MediaConfig` from env; also depends on `rust-s3` 0.37 directly | `crates/buzz-relay/src/config.rs:614-657`, `crates/buzz-relay/Cargo.toml:65` |
| `buzz-relay` storage sweep | Supplies the `fetch_page` closure over `MediaStorage::list_page` | `crates/buzz-media/src/bucket_index.rs:11-14` (documented), `crates/buzz-media/src/storage.rs:235-241` |
| `buzz-moderation` (external consumer) | Triggers on S3 `ObjectCreated` under `_uploads/` and parses `UploadRecord` instead of HEADing blobs | `crates/buzz-media/src/upload_record.rs:29-48` |
| `buzz-test-client` | Also depends on `rust-s3` 0.37 for E2E media tests | `crates/buzz-test-client/Cargo.toml:38` |


## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Integrations

---

### 1. Dependency inventory (`Cargo.toml:10-29`)

| Dependency | Version / source | Used for | Evidence |
|---|---|---|---|
| `buzz-core` | workspace | `CommunityId`, `StoredEvent`, `kind::{event_kind_u32, is_workflow_execution_kind, KIND_REACTION, KIND_STREAM_MESSAGE, KIND_STREAM_MESSAGE_DIFF}`, `network::is_private_ip` | `lib.rs:47-48`, `lib.rs:956`, `executor.rs:768` |
| `buzz-db` | workspace | `Db` handle, `workflow::RunStatus`, `WorkflowRecord`, `DbError` conversion | `lib.rs:49-50`, `error.rs:62-66` |
| `hex` | workspace | encode `owner_pubkey` for attribution | `executor.rs:558` |
| `serde` / `serde_json` / `serde_yaml` | workspace | schema derive, canonical JSON, YAML parsing | `schema.rs:7`, `schema.rs:263-266` |
| `dashmap` | workspace | `last_fired` interval anchors | `lib.rs:53`, `lib.rs:87` |
| `moka` | workspace | `moka::sync::Cache` for the per-channel workflow lookup | `lib.rs:104`, `lib.rs:118-121` |
| `evalexpr` | `"11"` (pinned major, direct dep — not workspace) | `if:`/filter condition evaluation, custom function registration | `executor.rs:17`, `executor.rs:224-316` |
| `cron` | `"0.16"` (direct dep) | cron expression parse + `after()` iteration | `schema.rs:239`, `lib.rs:691-694` |
| `nostr` | workspace | `PublicKey::from_hex`, `ToBech32` for the `npub` filter; test event builders | `executor.rs:18`, `executor.rs:194-198` |
| `uuid` | workspace | run/workflow ids, approval token generation, channel override parsing | `executor.rs:698-700`, `executor.rs:481` |
| `chrono` | workspace | `DateTime<Utc>` scheduling arithmetic | `lib.rs:52`, `lib.rs:692` |
| `tokio` | workspace | `Semaphore`, `spawn`, `spawn_blocking`, `time::{sleep, timeout}` | `lib.rs:54`, `executor.rs:370-373`, `executor.rs:1140` |
| `tracing` | workspace | structured logs throughout | `executor.rs:20` |
| `thiserror` | workspace | `WorkflowError`, `ActionSinkError` | `error.rs:3`, `action_sink.rs:12` |
| `reqwest` | workspace, **optional** (`features.reqwest = ["dep:reqwest"]`) | `call_webhook`, `add_reaction` | `Cargo.toml:27-29`, `executor.rs:781`, `executor.rs:888` |

No `sqlx` dependency — all Postgres access is via `buzz_db::Db` methods.

---

### 2. Outbound HTTP — `call_webhook`

| Concern | Implementation | Line |
|---|---|---|
| Client crate | `reqwest::Client`, built **per request** (required because `.resolve()` pins DNS per host) | `executor.rs:800-815` |
| Total timeout | `Duration::from_secs(10)` | `executor.rs:807` |
| Redirect policy | `reqwest::redirect::Policy::none()` — explicitly disabled so a redirect cannot bypass the SSRF check | `executor.rs:812` |
| Proxy bypass | `.no_proxy()` — **added after the original analysis**; without it a configured system proxy would do its own hostname resolution and bypass the pinned `safe_ip` | `executor.rs:810` |
| DNS pinning | `.resolve(host, SocketAddr::new(safe_ip, port))` with the IP validated by `check_ssrf`, closing the DNS-rebinding TOCTOU window | `executor.rs:813` |
| Method | `reqwest::Method::from_bytes(method)`; default `"POST"` when `method` is absent | `executor.rs:621`, `executor.rs:817-818` |
| Headers | caller-supplied map applied verbatim (names untemplated, values templated) | `executor.rs:822-826` |
| Body | raw string body, no content-type set automatically | `executor.rs:828-830` |
| Response cap | `WEBHOOK_MAX_RESPONSE_BYTES = 1024 * 1024` (1 MiB), enforced by chunked reads (`resp.chunk()`) with early abort | `executor.rs:778`, `executor.rs:841-863` |
| Response decode | `String::from_utf8_lossy` → `{status, body}` JSON | `executor.rs:865-868` |
| Port default | `port_or_known_default()`, fallback `80` — no scheme restriction, so plain `http://` targets are allowed despite the schema doc saying "must be a public HTTPS endpoint" (`schema.rs:120`) | `executor.rs:796-798` |
| Error mapping | every failure → `WorkflowError::WebhookError(String)` | `executor.rs:786`, `executor.rs:833-836`, `executor.rs:846-848` |

SSRF guard (`check_ssrf`, `executor.rs:745-776`): resolves `host:port` through the OS resolver on `spawn_blocking`; rejects on resolver error, on zero addresses, and if **any** returned address satisfies `buzz_core::network::is_private_ip`; returns `addrs[0]` for pinning. `is_private_ip` (`crates/buzz-core/src/network.rs:46-95`) covers IPv4 loopback/private/link-local/`0.0.0.0/8`/broadcast, CGNAT `100.64.0.0/10`, benchmarking `198.18.0.0/15`, and for IPv6 loopback/unspecified/`fc00::/7`/`fe80::/10`/`ff00::/8`/`2001:db8::/32` plus NAT64 local-use `64:ff9b:1::/48`, Teredo `2001::/32` and 6to4 `2002::/16` (added in `c26bf594`), with IPv4-mapped **and** IPv4-compatible addresses recursed through the IPv4 rules via `to_ipv4()` (`:62-65`), and NAT64 well-known `64:ff9b::/96` (`:69-73`) plus IPv4-translated `::ffff:0:0:0/96` (`:75-79`) recursed on their embedded IPv4.

---

### 3. Outbound HTTP — `add_reaction`

| Concern | Implementation | Line |
|---|---|---|
| Client | shared `LazyLock<reqwest::Client>` with a 10 s timeout, connection pooling retained | `executor.rs:874-885` |
| Target | `POST {BUZZ_RELAY_BASE_URL}/api/messages/{message_id}/reactions`, default base `http://localhost:3000` | `executor.rs:888-894` |
| Auth | `Authorization: Bearer {BUZZ_API_TOKEN}` if set, else `X-Pubkey: {BUZZ_RELAY_PUBKEY}` if set, else unauthenticated | `executor.rs:901-905` |
| SSRF guard | **none** on this path (no `check_ssrf`, no redirect policy, no response cap) — the URL comes from an env var rather than workflow YAML | `executor.rs:888-933` |
| Failure handling | non-2xx ⇒ `WebhookError` including the response body; body parse failure falls back to `{"raw": <text>}` | `executor.rs:914-922` |
| Reachability | the relay registers no `/api/messages/*` route (`crates/buzz-relay/src/router.rs:39-125`), so this call cannot succeed against the current relay |

---

### 4. Postgres via `buzz-db`

Access is exclusively through `buzz_db::Db` methods; no SQL is written in this crate.

| Db method | Purpose | Call site |
|---|---|---|
| `list_enabled_channel_workflows` | per-event workflow lookup (behind moka cache) | `lib.rs:301-306` |
| `list_all_enabled_workflows` | cron scan | `lib.rs:436` |
| `create_workflow_run` | run row for event + cron paths | `lib.rs:346-355`, `lib.rs:592-600` |
| `update_workflow_run` | `Running` / `Completed` / `Failed` transitions and trace writes | `executor.rs:985-994`, `executor.rs:1047-1056`, `lib.rs:201-215`, `lib.rs:220-238`, `lib.rs:244-261` |
| `get_workflow_run` | resolve `workflow_id` for `send_message`; read existing trace on resume | `executor.rs:537-543`, `executor.rs:1037` |
| `get_workflow` | `channel_id` + `owner_pubkey` for `send_message` | `executor.rs:546-552` |
| `claim_scheduled_workflow_fire` | cross-pod at-most-once cron claim | `lib.rs:547-568` |
| `latest_scheduled_workflow_fire` | restart anchor for interval schedules | `lib.rs:500-517` |
| `attach_scheduled_workflow_run` | best-effort claim→run link for audit | `lib.rs:617-628` |

DB errors convert via `From<buzz_db::error::DbError> for WorkflowError` → `Database(String)` (`error.rs:62-66`). In `finalize_run` DB failures are logged only (`lib.rs:207-213`, `:231-236`, `:256-260`); in `execute_run`/`execute_from_step` the initial `Running` write failure aborts the run (`executor.rs:995-1002`, `executor.rs:1057-1064`).

---

### 5. Relay integration (inbound)

| Integration point | Relay side | Line |
|---|---|---|
| Engine construction | `WorkflowEngine::new(db, WorkflowConfig::default())` | `crates/buzz-relay/src/main.rs:389-390`, `crates/buzz-relay/src/state.rs:1274-1276` |
| Side-effect sink | `RelayActionSink` registered via `set_action_sink` | `crates/buzz-relay/src/main.rs:594-595`, `crates/buzz-relay/src/workflow_sink.rs:13,159` |
| Scheduler | `tokio::spawn(async move { wf_cron.run().await })`, started only after the sink is wired | `crates/buzz-relay/src/main.rs:597-599` |
| Event hook | `workflow_engine.on_event(community, &stored_event)` spawned from the post-store fan-out | `crates/buzz-relay/src/handlers/event.rs:496-534` |
| Definition ingest | `WorkflowEngine::parse_yaml(&event.content)` in the workflow upsert command | `crates/buzz-relay/src/handlers/command_executor.rs:684` |
| Cache invalidation | `invalidate_channel_workflows` on upsert and on NIP-09 deletion | `crates/buzz-relay/src/handlers/command_executor.rs:787`, `crates/buzz-relay/src/handlers/side_effects.rs:2018,2039` |
| Manual trigger (kind-command) | ownership check, run creation, then `execute_from_step(..., 0, None)` + `finalize_run` | `crates/buzz-relay/src/handlers/command_executor.rs:826-936` |
| Inbound webhook | `POST /hooks/{id}` → host-bound tenant, `TriggerDef::Webhook` check, secret verification, body fields → `webhook_fields`, then `execute_from_step(..., 0, None)` | `crates/buzz-relay/src/router.rs:120`, `crates/buzz-relay/src/api/bridge.rs:1777-1911` |
| Approval grant/deny | relay handlers look up approvals by stored hash and call `resume_workflow` → `execute_from_step` | `crates/buzz-relay/src/handlers/command_executor.rs:1009-1169`, `:1244-1320` |
| Metrics | `buzz_workflow_runs_total{trigger,community}` incremented by the relay after a successful `on_event` | `crates/buzz-relay/src/handlers/event.rs:549-556` |

---

### 6. Error handling posture

| Boundary | Behaviour |
|---|---|
| Definition parse (per workflow, in `on_event` / cron) | warn + skip that workflow, other workflows continue (`lib.rs:331-337`, `lib.rs:449-459`) |
| Trigger filter error | warn + skip (fail-closed, no run created) (`lib.rs:838-845`) |
| Trigger-context serialization error | error log; `on_event` returns `Ok(())` (`lib.rs:319-326`); cron path `continue`s (`lib.rs:579-589`) |
| Run creation error | error log + `continue` to next workflow (`lib.rs:356-361`, `lib.rs:601-611`) |
| Step failure | aborts the run, `PartialProgress` preserved so the partial trace is persisted (`executor.rs:1113-1177`, `lib.rs:240-261`) |
| Spawned task | detached `tokio::spawn`; failures only surface through DB status + logs (`lib.rs:371-381`, `lib.rs:649-661`) |
| Sink not initialized | `WorkflowError::InvalidDefinition("action_sink not initialized …")` instead of a panic (`lib.rs:148-156`) |
| `set_action_sink` misuse | explicit `panic!("action_sink already initialized")` (`lib.rs:139-143`) — the only panic path in the crate outside tests |


## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Integrations

---

#### 1. External systems wired at startup, in order

The order below is the literal execution order of `main()` (`main.rs:83-1060`).

| Step | System | Call site | Fatal on failure? | Notes |
|------|--------|-----------|-------------------|-------|
| 1 | rustls `CryptoProvider` (ring) | `main.rs:88-90` | **yes** (`expect` panic) | Required before any TLS: `rediss://` to ElastiCache, `wss://`, S3 over TLS. Both `aws-lc-rs` and `ring` are in the tree transitively so rustls cannot auto-select (`main.rs:84-87`). |
| 2 | OTLP trace exporter (gRPC/tonic) | `telemetry.rs:85-88` via `main.rs:100` | **no** | Skipped entirely when `OTEL_EXPORTER_OTLP_ENDPOINT` is unset (`telemetry.rs:80-83`). A build failure returns `ExporterBuildFailed` and is logged as `warn` after the subscriber is up (`main.rs:118-120`). |
| 3 | Prometheus exporter HTTP listener | `metrics.rs:73`, spawned `metrics.rs:146` | **yes** (`expect` at `metrics.rs:143/145`) | Binds `0.0.0.0:metrics_port` (default 9102). Panics if the port is taken or a recorder already exists. |
| 4 | Postgres — main pool (+ optional read replica) | `main.rs:151-155` | **yes** | `DbConfig { database_url, read_database_url, ..default }` (`main.rs:145-149`) ⇒ max 20 / min 2 / acquire 3 s / lifetime 1800 s / idle 600 s (`crates/buzz-db/src/lib.rs:247-257`). Replica pool gets the *same* sizing (`crates/buzz-db/src/lib.rs:381-386`). |
| 5 | Postgres — migrations | `main.rs:164-168` | **yes**, but only runs when `BUZZ_AUTO_MIGRATE` is truthy (`main.rs:161-172`) | |
| 6 | Postgres — partition pre-creation (3 ahead) | `main.rs:173-176` | **no** — `error!` only | |
| 7 | Postgres — replica freshness fence probe | `main.rs:186-196` | **no** — `error!`, fence stays closed, all cursor reads go to the writer | Deliberately after step 5 so the commit-time floor guard is checked against the live schema (`main.rs:177-183`). |
| 8 | Postgres — community/owner/allowlist/d_tag bootstrap | `main.rs:220-320` | conditional: fatal iff `require_relay_membership` (steps for community, backfill, owner); `backfill_d_tags` always non-fatal | |
| 9 | Postgres — **audit pool** (separate) | `main.rs:325-329` | **yes** | `PgPoolOptions::new().max_connections(5).min_connections(1)`. No acquire timeout, lifetime, or idle timeout set. Only created when `audit_enabled`. |
| 10 | Redis — deadpool pool | `main.rs:336-341` | **yes** | `deadpool_redis::Config::from_url(redis_url)`, `PoolConfig::new(config.redis_pool_size)` (default 16). Cloned once for the readiness handler (`main.rs:333`, comment: cheap Arc clone). |
| 11 | Redis — `PubSubManager` (dedicated connection) | `main.rs:343-347` | **yes** | |
| 12 | Redis — 3 subscriber tasks | `main.rs:350-367` | n/a (spawned) | events, cache invalidation, conn-control. |
| 13 | Postgres — **search pool** (third pool) | `main.rs:378-382` | **yes** | `PgPoolOptions::new().connect(search_db_url)` — **no sizing knobs set at all**, sqlx defaults apply. Prefers `read_database_url` when set (`main.rs:373-377`). |
| 14 | Workflow engine (in-process, DB-backed) | `main.rs:389-390` | n/a | `WorkflowConfig::default()` — no env-driven workflow config is read here. |
| 15 | Relay signing keypair | `main.rs:392-414` | **yes** (`panic!` at `main.rs:409` when `require_auth_token` and no key) | Dev fallback uses hard-coded `0x…01` with a `warn` (`main.rs:396-408`). |
| 16 | S3 / MinIO — media storage | `main.rs:415-421` | **yes** | `config.media.validate()` then `MediaStorage::new`. |
| 17 | S3 / MinIO — git object store | inside `AppState::new`, `state.rs:694-701` | **yes** (`expect`, `state.rs:701`) | Reuses the same `media.s3_*` credentials/bucket/region. The `expect` message asserts media storage already validated the same config — true only because step 16 precedes `AppState::new`. |
| 18 | Local filesystem — git pack cache | `state.rs:702-709` | **yes** (`expect`, `state.rs:708`) | Directory itself was already `create_dir_all`'d during config load (`config.rs:390-397`). |
| 19 | Redis — NIP-98 replay guard | `state.rs:710-711` | n/a (lazy, uses the pool) | Doc is explicit: must stay Redis-backed and community-keyed; process-local caching would break cross-pod replay freshness (`state.rs:576-581`). |
| 20 | Redis — admission rate limiter | `state.rs:712` | n/a (lazy) | `RedisRateLimiter::new(redis_pool.clone())` — the real implementation lives at `crates/buzz-pubsub/src/rate_limiter.rs:88-99`. |
| 21 | Inter-relay QUIC mesh (UDP bind + Redis registry) | `main.rs:442-451` | **yes when enabled** (`?` at `main.rs:451`) | Off by default: `boot_mesh` returns `None` ⇒ nothing bound/published/spawned. Consumers wired at `main.rs:455-459` **before** the handle is published to `AppState.mesh`. |
| 22 | S3 / MinIO — A3 conformance probe | `main.rs:466-503` | **yes** | Runs by default (opt-out `BUZZ_GIT_CONFORMANCE_PROBE=false`). Races `BUZZ_GIT_PROBE_WRITERS` (default 32) writers over `BUZZ_GIT_PROBE_ROUNDS` (default 3) rounds against the pointer CAS. A backend that cannot do linearizable conditional writes aborts startup. |
| 23 | APNs push gateway (HTTPS, outbound) | matcher/worker spawned `main.rs:686-691`; timeout applied `push_runtime.rs:314` | **no** — the workers simply are not spawned when `push_gateway_delivery_url` is `None` | URL must be exactly `https://…/v1/deliveries/apns` (`config.rs:342-360`). Default when unset is the hard-coded `https://push.buzz.xyz/v1/deliveries/apns` (`config.rs:332`, `config.rs:752-757`) — **outbound push integration is on by default**. |
| 24 | TCP listeners: health then app | `main.rs:1116` then `main.rs:1157` | **yes** for both | Health binds first so probes answer as early as possible. |
| 25 | Unix domain socket (optional) | `main.rs:1178-1187` | **yes when configured** | Pre-existing non-socket file at the path is fatal (`main.rs:1168-1172`). |

#### 2. Postgres connection accounting

| Pool | Created | max | min | Instrumented? |
|------|---------|-----|-----|---------------|
| Main writer | `main.rs:151` → `crates/buzz-db/src/lib.rs:361` | 20 | 2 | yes (`main.rs:956-959`) |
| Read replica (if `READ_DATABASE_URL`) | `crates/buzz-db/src/lib.rs:363` | 20 | 2 | yes (`main.rs:962-966`) |
| Audit | `main.rs:325-329` | 5 | 1 | **no** |
| Search | `main.rs:378-382` | sqlx default (unset) | sqlx default | **no** |

**Verified doc drift:** `DbConfig::default()`'s doc says "At 20 main + 5 audit = 25/pod, four relay pods fit within the PG limit" (`crates/buzz-db/src/lib.rs:244-246`). `main.rs` opens a **third** pool for search (`main.rs:378-382`) whose size is never set, and a replica pool at the same 20. The actual per-pod ceiling is 20 + 5 + search-default, plus 20 more with a replica — not 25. The "four pods within PG max_connections=100" arithmetic therefore does not hold.

Note also: the search pool connects to the **replica** when one is configured (`main.rs:373-377`), deliberately bypassing the freshness fence because search is lag-tolerant (`main.rs:369-372`).

#### 3. Failure behaviour if each dependency is unavailable

| Dependency | At startup | At runtime |
|-----------|-----------|-----------|
| Postgres (main) | abort with `DB connection failed` (`main.rs:151-155`) | `/_readiness` → 503 with `{"postgres": false}` (`router.rs:352-373`); host→community binding fails closed ⇒ every new WS gets a generic 404 (`router.rs:288-299`); NIP-11 still served, `icon` omitted (`nip11.rs:277-286`) |
| Postgres (audit) | abort (`main.rs:329`) | per-entry `error!` + `buzz_audit_log_errors_total`; worker survives (`state.rs:1200-1206`); events are still accepted (audit is off the critical path) |
| Postgres (search) | abort (`main.rs:382`) | search REQ/`POST /query` fail; not surfaced in readiness |
| Redis | abort on pool creation or `PubSubManager::new` (`main.rs:336-347`) | `/_readiness` → 503 with `{"redis": false}` (`router.rs:353-373`); **rate-limit admission fails closed and denies** (`admission.rs:29-33`); NIP-98 replay guard unavailable ⇒ NIP-98 routes fail; cross-pod fan-out / cache invalidation / ban propagation silently stop |
| S3 / MinIO | abort — media storage init (`main.rs:419-421`), git store (`state.rs:701`), and A3 probe (`main.rs:488-495`) are all fatal | media + git request failures; the storage sweep is the only path with an explicit kill switch for missing `s3:ListBucket` (`main.rs:1436-1441`) |
| OTLP collector | never fatal (`telemetry.rs:80-90`) | batch exporter drops spans; shutdown error is `warn` (`main.rs:1054-1057`) |
| Mesh peers / Redis registry | fatal **only when `BUZZ_MESH=on`** (`main.rs:451`) | `/_mesh` reports peer state; fence-rejection counters exposed |
| APNs push gateway | not contacted at startup | per-request timeout `push_gateway_timeout` (default 2000 ms, `config.rs:759-773`) applied at `push_runtime.rs:314` |

#### 4. Retry / backoff inventory

There is **no exponential backoff anywhere in this file group**. Every retry is a fixed-interval loop or a single attempt.

| Path | Strategy | Cite |
|------|----------|------|
| Postgres connect (all 3 pools) | none — single attempt, abort | `main.rs:151`, `main.rs:329`, `main.rs:382` |
| Redis pool | deadpool acquires lazily per use; no relay-side retry | `main.rs:336-341` |
| Channel-event reconciliation | fixed 5 s × 24 attempts (≈2 min), then gives up silently | `main.rs:570-590` |
| NIP-43 snapshot reconciliation | fixed interval, default 60 s, indefinite | `main.rs:517-545` |
| Ephemeral reaper | fixed interval, default 60 s; failed tick `continue`s | `main.rs:609-620` |
| Reminder scheduler | fixed interval, default 10 s; failed publish releases the claim so the next tick retries — **unless the release itself fails, in which case the reminder is never retried** | `main.rs:701-798` |
| Community revalidator | fixed interval, default 30 s clamped to `1..=300`; a failed per-community lookup simply waits for the next tick | `main.rs:882-890`, `state.rs:1076-1087` |
| Pool metrics | fixed interval, default 10 s, `.max(1)` | `main.rs:945-949` |
| Usage metrics | fixed interval, default 300 s, `.max(5)`, first tick jittered by `rand % interval`; `MissedTickBehavior::Skip` | `main.rs:1009-1022` |
| Cross-pod publishes (cache invalidation, ban) | fire-and-forget, **no retry**; backstopped by ≤10 s cache TTL and the durable DB ban row respectively | `state.rs:964-978`, `state.rs:1039-1053` |
| Community-archive publish | **awaited** (not fire-and-forget) so the archive API can offer a retryable response; plus a periodic durable revalidation backstop | `state.rs:1055-1071`, `main.rs:869-890` |
| Redis broadcast consumers | `Lagged` is counted and tolerated; `Closed` **breaks the loop permanently** | `main.rs:834-843`, `:864-873`, `:925-934` |
| Storage sweep | single-flight with `BUZZ_STORAGE_SWEEP_TIMEOUT_SECS`; last cached snapshot re-emitted | `main.rs:1442-1477`, `storage_sweep.rs:56-72` |
| A3 conformance probe | `race_width` × `race_rounds` races, single overall attempt, fatal | `main.rs:472-502` |

#### 5. Crate-level dependency integration (`Cargo.toml`)

Internal crates depended on: `buzz-core`, `buzz-conformance`, `buzz-db`, `buzz-auth`, `buzz-pubsub`, `buzz-audit`, `buzz-search`, `buzz-relay-mesh`, `buzz-sdk`, `buzz-workflow` (with `reqwest` feature), `buzz-media` (`Cargo.toml:19-27`, `:66-68`).

Notable third-party pins that are **not** workspace-managed (each pinned locally):
- `rustls 0.23` with `default-features = false, features = ["ring","std"]` (`Cargo.toml:57`) — deliberate, paired with `main.rs:88-90`.
- `s3 = { version = "0.37", package = "rust-s3", features = ["tokio-rustls-tls","fail-on-err","tags"] }` (`Cargo.toml:61`).
- `async-trait 0.1` (`Cargo.toml:28`) — but `tenant.rs:29-30` explicitly says it uses native `async fn` in trait "no `async-trait` dependency", and `HostResolver` (`tenant.rs:31`) does. So the `async-trait` dep is used elsewhere in the crate, not by the tenancy seam.
- `base64 0.22`, `tempfile 3`, `bytes 1`, `infer 0.19`, `pulldown-cmark 0.13.4`, `async-compression 0.4.42` (`Cargo.toml:60`, `:62-64`, `:78-79`).
- `tokio-util` needs the extra `io` feature for git stdout streaming (`Cargo.toml:31-34`).

Dev-dependencies pull **two git-sourced crates from an external GitHub org**: `mesh-llm-sdk` and `mesh-llm-host-runtime`, both `git = "https://github.com/Mesh-LLM/mesh-llm.git", tag = "v0.73.1"` (`Cargo.toml:84-85`). These are test-only but make `cargo test -p buzz-relay` require network access to a third-party repo.

`buzz-relay` version is deliberately independent of the workspace: `0.2.0` (`Cargo.toml:7`, rationale `Cargo.toml:4-6`) and is what `NIP-11.version` reports (`nip11.rs:161`).

#### 6. Nothing depends on `buzz-relay` as a library

Verified: no crate in the workspace and nothing under `desktop/src-tauri/**` declares `buzz-relay` as a dependency (`buzz-admin`, `buzz-conformance`, `buzz-relay-mesh`, `git-sign-nostr` mention the name only in comments). The single external textual reference is a doc mention of `buzz_relay::handlers::event::tests::`. Consequence: the crate's entire `pub` surface exists solely for `main.rs` and in-crate consumers, and `lib.rs:53-55`'s three re-exports have no consumer at all.


## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Integrations

---

#### 1. `buzz-db` — every call from this group

| Call | Caller | Purpose | Failure handling |
|---|---|---|---|
| `is_community_active(community)` | `connection.rs:133` (closure passed to `run_registered_community_connection`) | durable community revalidation after socket registration | anything other than `Ok(true)` cancels the socket (`state.rs:149-152`) — fail closed |
| `moderation_restriction_state(community, pubkey)` | `auth.rs:119-130` | ban seam on the authenticated pubkey | `Err` → `BanOutcome::DbError` → deny with `error: internal …` |
| `moderation_restriction_state(community, owner)` | `auth.rs:136-156` | NIP-OA owner ban cascade | same |
| `is_pubkey_allowed(community, pubkey)` | `auth.rs:189-212` | allowlist gate (only when enabled **and** `Nip42`) | `Err` → `false` → deny (fail closed, `auth.rs:195-201`) |
| `is_agent_owner(community, agent, owner)` | `event.rs:1021-1025` | observer-frame publish authorisation | `Err` → `OK false "error: internal server error"` |
| `is_member(community, channel, pubkey)` | `req.rs:145-149` (REQ up-front confirm), `count.rs:126-141` (per-filter confirm) | uncached membership confirmation to repair a stale cache-negative | `Err` → `CLOSED "error: database error"` |
| `query_events(&EventQuery)` | `req.rs:313` (one per filter, concurrency 4), `count.rs:187`, `:256` (fallback) | historical/candidate rows | REQ → `EOSE` + return (**not** `CLOSED`, `req.rs:323-329`); COUNT → `CLOSED "error: {e}"` |
| `count_events(&EventQuery)` | `count.rs:176`, `:246` | exact fast-path count | `CLOSED "error: {e}"` |
| `get_events_by_ids(community, ids)` | `req.rs:641-645` | batch fetch of FTS hit ids | `warn!` + `break` out of the page loop (partial results, then EOSE) |
| `communities_of_channels(&[Uuid])` | `req.rs:355`, `:648` | conformance row-community projection **only** | `Err` → `warn!` + empty map; the emit's own guard-rail handles it |
| `db.clone()` | `req.rs:303` | cloned into the `buffered` stream | — |

Indirect (through `AppState` helpers, `state.rs`):

| Helper | Underlying | Caller | Cache |
|---|---|---|---|
| `get_accessible_channel_ids_cached` | `db.get_accessible_channel_ids` | `req.rs:94-98`, `count.rs:79-83` | moka, 10 s TTL, cap 10 000 (`state.rs:747-753`) |
| `is_member_cached` | `db.is_member` | `event.rs:209-212` (fan-out gate) | moka, 10 s TTL, cap 10 000 (`state.rs:740-746`) |
| `channel_visibility_cached` | `db.get_channel` | `event.rs:197-200` | moka, 10 s TTL — **caches only `"private"`** (`state.rs:1124-1150`) |

The read path is **not** replica-routed at this layer; `AppState.db` is a single `Db` and the optional `READ_DATABASE_URL` (`config.rs:57-59`) is handled inside `buzz-db`.

---

#### 2. `buzz-auth`

| Item | Used at |
|---|---|
| `generate_challenge()` | `connection.rs:157` |
| `AuthService::verify_auth_event(event, challenge, relay_url)` | `auth.rs:86-90` — the whole of NIP-42 verification; pure crypto |
| `AuthService::config().rate_limits` | `connection.rs:612` |
| `AuthContext` (`pubkey`, `scopes`, `channel_ids`, `auth_method`, `agent_owner_pubkey`) | stored in `AuthState::Authenticated` (`connection.rs:45`), read at `event.rs:634-654`, `req.rs:50-87`, `count.rs:37-51`, `connection.rs:604-609` |
| `AuthMethod::Nip42` | `auth.rs:191` (allowlist scoping) |
| `Scope::{MessagesRead, MessagesWrite}` | `req.rs:54`; `event.rs:681`, `:676` |
| `LimitType::{WsEvents, Messages}` | `connection.rs:619`, `:642` |
| `RateLimiter` trait (via `crate::admission::check_principal`) | `admission.rs:18-42` |
| `Nip98ReplayGuard` | not used by this group (HTTP only) |

`AuthService` is **not** consulted for authorization — only for verification and for reading rate-limit config. Every authz decision on the WS path is made in `handlers/*` against `buzz-db` and `buzz-core::kind` sets.

---

#### 3. `buzz-pubsub`

##### 3.1 Publish

| Topic | Caller | Preceded by `mark_local_event` |
|---|---|---|
| `EventTopic::Channel(ch)` | `event.rs:399` (persistent, when `stored_event.channel_id` is `Some`), `event.rs:874` (channel-scoped ephemeral) | yes (`:394`, `:824`) |
| `EventTopic::Global` | `event.rs:399` (persistent, channel-less), `event.rs:879` (channel-less ephemeral), `event.rs:1073` (observer frame) | yes (`:394`, `:852`, `:1046`) |

Every publish failure invalidates the local-echo mark before logging (`event.rs:400-405`, `:830-836`, `:858-864`, `:1052-1058`) — otherwise a failed publish would suppress a later legitimate delivery of the same id.

##### 3.2 `retain_topic` / `release_topic` lifecycle

Refcounting lives in `buzz-pubsub`: `desired_topics: HashMap<EventTopicKey, usize>`; `retain_topic` PSUBSCRIBEs only on the 0→1 transition (`buzz-pubsub/src/lib.rs:192-208`), `release_topic` schedules a debounced unsubscribe on the 1→0 transition and warns on an unmatched release (`:215-232`).

| Operation | Site | Topic |
|---|---|---|
| **retain** — after successful non-search REQ registration | `req.rs:254-257` | `topic_for_subscription(channel_id)` (`req.rs:1270-1275`) |
| **release** — the subscription that this REQ replaced | `req.rs:248-253` | `topic_for_subscription(replaced.channel_id)` |
| **release** — explicit CLOSE | `close.rs:20-25` | `topic_for_subscription(removed.channel_id)` (`close.rs:30-35`) |
| **release** — one per subscription at disconnect | `connection.rs:265-270` | `topic_for_subscription(removed.channel_id)` (`connection.rs:681-686`) |
| retain (test-only) | `event.rs:1706`, `:1687` | `EventTopic::Global`, inside `#[cfg(test)]` |

Balance audit:
- Every `register_scoped` is followed by exactly one `retain_topic`, and its `Option<RemovedSubscription>` return drives exactly one compensating `release_topic` — so a same-`sub_id` replacement is net-neutral when the scope is unchanged, and a correct swap when it changes (`req.rs:241-257`).
- `remove_connection` returns **one `RemovedSubscription` per subscription** (`subscription.rs:181-196`), and `connection.rs:265-270` releases once per element — so N subscriptions on the same topic produce N retains and N releases. Correct.
- **Three identical private copies** of `topic_for_subscription` exist (`req.rs:1270-1275`, `close.rs:30-35`, `connection.rs:681-686`). See the debt aspect.
- Search REQs never retain (they return at `req.rs:232`), so the counts stay balanced.

**Unbalanced-release risk found:** `close.rs:16` removes the entry from `conn.subscriptions` **before** `sub_registry.remove_subscription` at `:20`. Two concurrent CLOSE frames for the same `sub_id` cannot double-release, because `remove_subscription` is the one that returns `Some` and it is guarded by the DashMap entry removal (`subscription.rs:164-172`) — the second call returns `None`. Verified safe. The `conn.subscriptions` removal is not the guard.

##### 3.3 Subscribe (fan-in)

`main.rs:818-845` holds the only `subscribe_local()` consumer in production; it calls `fan_out_pubsub_event` (`event.rs:282`). Lag → `buzz_multinode_fanout_lag_total` and a warning (`main.rs:833-836`); a closed broadcast channel logs an error and ends the loop (`:840-842`) — **the loop is not restarted**, so a closed channel silently ends cross-node delivery for the process lifetime.

##### 3.4 Other pub/sub channels this group depends on

| Channel | Consumer | Effect on this group |
|---|---|---|
| cache invalidation | `main.rs:846-877` → `state.apply_cache_invalidation` | drops the membership / accessible-channels / visibility moka entries this group's gates read |
| conn control (`DisconnectPubkey`, `DisconnectCommunity`) | dispatched in `main.rs` → `conn_manager.disconnect_pubkey` / `community_connections.disconnect_community` | closes sockets owned by this group |
| presence (`set_presence` / `clear_presence`) | `event.rs:814-822`, `connection.rs:274-284` | Redis-side presence state |

---

#### 4. `buzz-search`

| Item | Used at |
|---|---|
| `SearchService::search(&SearchQuery)` | `req.rs:602-610` |
| `SearchQuery { community, q, channel_scope, kinds, authors, since, until, page, per_page, mode }` | built `req.rs:596-608` |
| `SearchMode::FullText` | `req.rs:607` — the only mode used |
| `ChannelScope::{Channels, ChannelsOrChannelLess, ChannelLessOnly}` | `req.rs:483-501`, `:580` |

Contract: search returns **event ids only** (`req.rs:637`); the full events are then fetched from Postgres (`req.rs:641-645`) and re-post-filtered (`req.rs:685-712`). This is why the sensitive-kind gates must run *before* the search branch (`req.rs:175-205`) — search hits skip the per-filter historical post-check chain by construction. A search error is non-fatal: `warn!` + `break` out of the page loop, then EOSE (`req.rs:611-616`).

---

#### 5. `buzz-conformance` (via `crate::conformance`)

| Emit | Site | Guard |
|---|---|---|
| `state_for_request(tenant, pubkey)` — builds the `AbstractState` once per REQ | `req.rs:110-116` | `None` only on malformed pubkey bytes |
| `record_req_authcheck(tracer, state, ch_id, member)` | `req.rs:141-148` | only on the DB-confirmation branch |
| `record_read_message_rows(tracer, state, per_filter_channel, &row_channels, &channel_communities)` | `req.rs:337-372` | non-search lane, per filter |
| `record_read_by_id_rows(tracer, state, None, &row_channels, &channel_communities)` | `req.rs:626-668` | search lane, per page |

Production binds `NoopTracer` (`state.rs:798`), so these are zero-cost — **except** the two `db.communities_of_channels` round-trips at `req.rs:355` and `:648`, which are issued **unconditionally** whenever `trace_state.is_some()` (i.e. always, in practice) regardless of whether the tracer is a no-op. That is a per-filter and per-search-page extra query on the hot read path with no production benefit. See the debt aspect.

No conformance emit exists on the write/fan-out side of this group; the comment at `event.rs:397-403` explains that acceptance is recorded at the ingest seam and fan-out surfaces as `ReadMessageRows` on the subscriber side.

---

#### 6. `buzz-core`

| Item | Used at |
|---|---|
| `filter::filters_match` | `subscription.rs:377`, `req.rs:372`, `:695`, `count.rs:198`, `:267` |
| `filter::reader_authorized_for_event` | **no longer called directly from these files** since `ab3af828` — the four call sites were folded into `handlers/req.rs::event_visible_to_reader` (`req.rs:1230`), reached from `req.rs:388`, `:705`, `count.rs:202`, `:271` |
| `verification::verify_event` | `event.rs:772` (ephemeral), `:927` (observer) — both on `spawn_blocking` |
| `event::StoredEvent::{new, with_received_at}` | `event.rs:292`, `:841`, `:869`, `:1060` |
| `kind::{event_kind_u32, is_ephemeral, is_workflow_execution_kind, is_command_kind, is_parameterized_replaceable}` | `event.rs:611`, `:675`, `:509-510`; `req.rs:777-782`, `:944-950` |
| `kind::{AUTHOR_ONLY_KINDS, P_GATED_KINDS, RESULT_GATED_KINDS}` | `event.rs:137`; `req.rs:1042`, `:1156`, `:1139`, `:1188`, `:1208` |
| `kind::{KIND_GIFT_WRAP, KIND_AUTH, KIND_AGENT_OBSERVER_FRAME, KIND_PRESENCE_UPDATE, KIND_DM_VISIBILITY, KIND_AGENT_TURN_METRIC, KIND_AGENT_ENGRAM, KIND_NIP43_MEMBERSHIP_LIST}` | `event.rs:659`, `:647`, `:657`, `:772`, `:438-439`; `req.rs:832`, `:1065`, `:1114` |
| `observer::{content_looks_like_nip44, OBSERVER_AGENT_TAG, OBSERVER_FRAME_TAG, OBSERVER_FRAME_TELEMETRY, OBSERVER_FRAME_CONTROL}` | `event.rs:1095-1113` |
| `tenant::{TenantContext, CommunityId}` | throughout |

---

#### 7. `buzz-audit`

Only via the bounded channel: `state.audit_tx` (capacity 1000, `state.rs:654`), written at `event.rs:574` with `send().await`. The worker (`state.rs:658-696`) does the DB write. Channel-closed → `error!` + `buzz_audit_send_errors_total` (`event.rs:574-577`); the event is **not** rejected. When `audit_enabled == false`, `audit_tx` is `None` and the enqueue short-circuits (`event.rs:571-573`).

---

#### 8. `buzz-workflow`

`workflow_engine.on_event(community, &stored_event)` is spawned from `dispatch_persistent_event_inner` (`event.rs:512-533`). The community is passed **explicitly** because `StoredEvent` does not carry it and the same channel UUID can exist in another tenant (rationale `event.rs:528-532`). Skipped for workflow-execution kinds, command kinds, relay-signed `buzz:workflow`-tagged events, and kind 1059 (`event.rs:526-534`).

---

#### 9. Slow-client / backpressure handling

Two sender surfaces, one shared counter:

| Surface | Method | Site |
|---|---|---|
| direct (handler → its own socket) | `ConnectionState::send(String)` → `send_tx.try_send(Text)` | `connection.rs:88-113` |
| fan-out (any task → any socket) | `ConnectionManager::send_to(String)` / `send_to_text_bytes(Arc<Bytes>)` → `try_send_ws_message` | `state.rs:436-438`, `:443-447`, `:449-474` |

Shared state: `backpressure_count: Arc<AtomicU8>` created at `connection.rs:164`, handed to both `ConnectionState` (`:178`) and `ConnEntry` (`:210`). Semantics (identical in both):

1. `Ok` → `store(0)` — a single successful send fully forgives accumulated strikes (`connection.rs:92`, `state.rs:456`).
2. `Full` → `fetch_add(1)+1`; `>= grace_limit` (15) → `warn!` + `buzz_ws_backpressure_disconnects_total` + `cancel()`; otherwise a graded warning (`connection.rs:95-107`, `state.rs:458-472`).
3. `Closed` → `debug!`, return `false`, **no** strike (`connection.rs:108-111`, `state.rs:473-476`).

Callers that react to a `false` return: `req.rs:398-400` and `:720-722` abandon delivery (and skip EOSE); `event.rs:81-91` counts drops into `drop_count` and logs an aggregate warning (`:249-255`, `:299-305`, `:474-479`).

Control-plane backpressure is treated as **terminal**, not graded: a full 8-slot `ctrl_tx` closes the connection in the heartbeat loop (`connection.rs:396-400`) and on a client Ping (`connection.rs:464-472`). Best-effort ctrl sends that do *not* close: ban reason (`auth.rs:177-179`), `disconnect_pubkey` reason (`state.rs:325-328`), restart close (`state.rs:359`, `:229`).

Read-side backpressure: there is none. `recv_loop` awaits `ws_recv.next()` and, for EVENT/REQ/COUNT, spawns a task under `handler_semaphore` and immediately loops (`connection.rs:530-536`, `:552-558`, `:573-579`). `AUTH` and `CLOSE` are awaited inline (`:506-508`, `:582`), which is the only inbound self-throttle.

---

#### 10. Integration failure-mode summary

| Dependency | On failure | Posture |
|---|---|---|
| Postgres — community active check | socket cancelled | fail closed |
| Postgres — ban state | deny with `error: internal …` | fail closed, cause-preserving |
| Postgres — allowlist | deny | fail closed |
| Postgres — accessible channels | `CLOSED "error: database error"` | fail closed |
| Postgres — membership confirm | `CLOSED "error: database error"` | fail closed |
| Postgres — visibility (fan-out) | recipient list emptied | fail closed |
| Postgres — membership (fan-out) | that recipient dropped | fail closed |
| Postgres — historical query | `EOSE`, subscription stays live | **fail open (silent)** |
| Postgres — COUNT | `CLOSED "error: {raw}"` | fail closed, **leaks error text** |
| Postgres — `get_events_by_ids` (search) | partial page, then EOSE | **fail open (silent)** |
| Postgres — `communities_of_channels` | empty map, trace-only | fail soft |
| Redis — admission limiter | frame rejected | fail closed |
| Redis — `publish_event` | echo mark invalidated, `warn!`, local fan-out still happens | fail open (single-node delivery only) |
| Redis — presence | ignored (`let _ =`) | fail open |
| Redis — cache-invalidation publish | spawned, warn-only | fail open, ≤10 s TTL backstop |
| Redis — broadcast lag | counted, events lost | fail open |
| Redis — broadcast closed | error logged, **loop exits permanently** | fail open |
| Search (FTS) | `warn!` + break, then EOSE | fail open (silent) |
| Audit channel closed | `error!` + counter, event still accepted | fail open |
| Workflow engine | `error!` in spawned task | fail open |


## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: Integrations

---

### 1. Crate dependency fan-out

| Crate | Reached from this group | How |
|---|---|---|
| `buzz-core` | all four files | kind constants + classification predicates (`ingest.rs:11-45`), `verify_event` (`:1491`), `TenantContext`, `CommunityId`, `StoredEvent`, `canonical_channel_name` (`ingest.rs:2069`, `side_effects.rs:466`) |
| `buzz-auth` | `ingest.rs:8` | `Scope` only — used by `required_scope_for_kind` (`:198-306`) and the (unreachable) gate at `:1551` |
| `buzz-db` | all except `imeta.rs` | 60+ distinct methods via `state.db` — see §2 |
| `buzz-media` | `imeta.rs` | `MediaStorage::get_sidecar` (`imeta.rs:246`, `:308`), `MediaStorage::head` (`:252`, `:290`, `:328`), `validation::mime_to_ext` (`imeta.rs:5`, used `:185`) |
| `buzz-pubsub` | `side_effects.rs` | `EventTopic` (`:26`); `publish_event` (`:701`, `:794`, `:873`); `release_topic` (`:81`) |
| `buzz-workflow` | `command_executor.rs` | `WorkflowEngine::parse_yaml` (`:719`), `TriggerDef::Webhook` match (`:738`), `WorkflowDef` deserialize (`:903`, `:1272`), `executor::execute_from_step` (`:925`, `:1317`), `engine.finalize_run` (`:934`, `:1325`), `engine.invalidate_channel_workflows` (`:783`); `side_effects.rs:2017`, `:2038` for the a-tag delete path |
| `buzz-conformance` | `ingest.rs` | `Tracer` trait (`:1414`), `TraceAction`, `EmitGuard`, `Verdict` via `crate::conformance` (`:47-50`) |
| `buzz-audit` | **not directly** | reached transitively through `dispatch_persistent_event` → `enqueue_event_created_audit` (`handlers/event.rs:359`, `:534-577`) |
| `buzz-search` | **never** | Postgres FTS is a generated `tsvector` column populated by the `insert_event` write itself; the old Typesense worker and `search_index_tx` are gone (`handlers/event.rs:501-507`). This module makes **zero** `buzz_search` calls. |
| `sqlx` | `command_executor.rs` | raw SQL — `pg_advisory_xact_lock` (`:170`), coordinate SELECT (`:176`), coordinate soft-delete (`:196`), `events` INSERT (`:196-232`). The only place in this group that bypasses the `buzz-db` API. |
| `metrics` | `ingest.rs`, `side_effects.rs`, `command_executor.rs` | 8 series — see configuration.md §4 |
| `nostr` | all | `Event`, `EventBuilder`, `Kind`, `Tag`, `Keys` |

`buzz-relay` internal reach-out: `crate::api::media::media_base_url_for_tenant`
(`ingest.rs:2238`) and `is_safe_ext` (`imeta.rs:379`); `crate::api::nip05::canonicalize_nip05`
(`side_effects.rs:1157`); `crate::api::git::{manifest, store, manifest_event}`
(`side_effects.rs:2616-2618`, `:2705`, `:2738`); `crate::protocol::RelayMessage`
(`side_effects.rs:23`); `crate::webhook_secret` (`command_executor.rs:27`);
`crate::handlers::{event, relay_admin, identity_archive, report, product_feedback,
moderation_commands, push_lease}`.

---

### 2. `buzz-db` call inventory (by concern)

| Concern | Methods called | Sites |
|---|---|---|
| Event read | `get_event_by_id`, `get_event_by_id_including_deleted`, `query_events` | `ingest.rs:361`, `:634`, `:665`, `:690`, `:790`, `:869`, `:2245`; `side_effects.rs:203`, `:552`, `:1600`, `:2114`, `:2201`, `:931`, `:2969`, `:3080` |
| Event write | `insert_event`, `insert_event_with_thread_metadata`, `insert_reaction_event_with_thread_metadata`, `replace_addressable_event`, `replace_parameterized_event` | `ingest.rs:2324`, `:2371`, `:2385`, `:2394`; `side_effects.rs:692`, `:868`, `:940`, `:2752`, `:2911`, `:3036`, `:3123`, `:3186` |
| Event delete | `soft_delete_event_and_update_thread`, `soft_delete_by_coordinate`, `soft_delete_discovery_events` | `side_effects.rs:1624`, `:2147`, `:2069`, `:1798` |
| Thread | `get_thread_metadata_by_event`, `get_thread_summary` | `ingest.rs:608`; `side_effects.rs:1616`, `:2131`, `:735` |
| Reaction | `remove_reaction_by_source_event_id`, `remove_reaction` | `side_effects.rs:2175`, `:2216` |
| Channel | `get_channel`, `create_channel`, `create_channel_with_id`, `update_channel`, `set_topic`, `set_purpose`, `archive_channel`, `unarchive_channel`, `soft_delete_channel`, `list_channels`, `open_dm`, `hide_dm`, `list_hidden_dms` | `ingest.rs:1765`, `:812`, `:2103`, `:2408`; `side_effects.rs:271`, `:545`, `:1345`, `:1372`, `:1387`, `:1416`, `:1466`, `:1485`, `:1499`, `:1789`, `:1846`, `:2952`, `:3062`; `command_executor.rs:398`, `:497`, `:534`, `:611`, `:633`, `:766` |
| Membership | `get_members`, `add_member`, `remove_member`, `is_agent_owner`, `get_agent_channel_policy` | `side_effects.rs:100`, `:298`, `:376`, `:391`, `:520`(via members), `:627`, `:647`, `:1216`, `:1275`, `:1293`, `:1526`, `:1932`; `ingest.rs:829`, `:1339`, `:2002`; `command_executor.rs:518` |
| User | `ensure_user`, `update_user_profile`, `set_channel_add_policy` | `side_effects.rs:1096`, `:1140`, `:1163`, `:1182`, `:1105`; `command_executor.rs:49` |
| Relay membership | `remove_relay_member`, `publish_nip43_membership_locked`, `nip43_membership_snapshot_needs_reconciliation`, `usage_community_hosts` | `ingest.rs:1884` (`remove_relay_member`), `:1909` (NIP-43 publish fan-out); `side_effects.rs:2866`, `:2789`, `:2777` |
| Archived identities | `list_archived` | `side_effects.rs:3009` |
| Moderation | `moderation_restriction_state` | `ingest.rs:1642` |
| Workflow | `get_workflow`, `upsert_workflow`, `create_workflow_run`, `get_workflow_run`, `update_workflow_run`, `delete_workflow_for_owner`, `find_workflow_by_owner_and_name`, `get_approval_by_stored_hash`, `update_approval_by_stored_hash` | `command_executor.rs:724`, `:775`, `:918`, `:1250`, `:1177`, `:1276`, `:1290`, `:1305`, `:1041`, `:1153`; `side_effects.rs:2000`, `:2011`, `:2026` |
| Git | `repo_name_owner`, `count_repos_for_owner`, `reserve_repo_name`, `release_repo_name` | `side_effects.rs:2470`, `:2491`, `:2500`, `:2543` |
| Transaction | `begin_transaction` | `command_executor.rs:106` |

---

### 3. Transaction boundaries and partial-failure semantics

**The core answer: ingest is *not* atomic beyond the single storage call.**

#### 3a. What *is* atomic

| Unit | Contents | Where |
|---|---|---|
| Generic event insert | `events` row + `thread_metadata` row + parent/root stubs + `reply_count`/`last_reply_at`/`descendant_count` updates | `buzz-db/src/event.rs:1004-1191`; the rationale ("could leave reply counters inconsistent if one succeeded and the other failed") is at `:1173-1177` |
| Reaction insert | `reactions` upsert + `events` row + `thread_metadata`, in a load-bearing order so an active duplicate returns before a duplicate kind:7 is stored | `buzz-db/src/event.rs:1201-1275`; ordering note `ingest.rs:2320-2323` |
| Soft delete | `events.deleted_at` + `reply_count`/`descendant_count` decrements | `soft_delete_event_and_update_thread`, called `side_effects.rs:1624`, `:2147` |
| Replaceable / NIP-33 replace | old-row supersede + new insert | inside `replace_addressable_event` / `replace_parameterized_event` |
| Relay-member removal | NotFound / IsOwner classification + delete | `remove_relay_member` (`ingest.rs:1883`) — the comment at `:1855` states it "handles both the NotFound and IsOwner cases atomically" |
| Git name reservation | `INSERT … ON CONFLICT DO NOTHING` as the TOCTOU race guard | `side_effects.rs:2500`, rationale `:2437-2447` |
| NIP-43 snapshot publish | read members + build event + replace, all inside one per-community advisory lock so a stale snapshot cannot win by arrival order | `publish_nip43_membership_locked` (`side_effects.rs:2866`), rationale `:2814-2824` |
| Command coordinate LWW | `pg_advisory_xact_lock` + head read + supersede + insert | `command_executor.rs:170-232` |

#### 3b. What is **not** atomic — the failure matrix

| Failure point | Event state | Domain state | Client sees | Evidence |
|---|---|---|---|---|
| `handle_side_effects` returns `Err` | **committed and fanned out** | not applied | `accepted: true, message: ""` | `ingest.rs:2460-2467` — `warn!` only, then `dispatch_persistent_event` at `:2513` runs unconditionally |
| 9000 `add_member` rejected by the DB's elevated-role guard (`buzz-db/src/channel.rs:385`, `:400`) | kind:9000 committed + fanned out | no membership change | success | same |
| 9002 `update_channel` fails mid-loop | kind:9002 committed; **earlier tags in the same event already applied** | partial metadata update | success | `side_effects.rs:1339-1552` — the per-tag loop uses `?`, so tag *n* failing leaves tags 1..n−1 applied |
| 9007 event insert fails after `create_channel_with_id` | not stored | channel soft-deleted by compensation | `Internal` | `ingest.rs:2430-2440` — manual compensation, itself `warn!`-only on failure |
| 30617 pointer seed fails | kind:30617 committed | name reservation released **only if this attempt created it** | success | `side_effects.rs:2528-2555` |
| 30617 kind:30618 emission fails | committed | pointer exists, subscribers miss the "repo exists" signal | success | `side_effects.rs:2588-2601` — explicitly "Non-fatal" |
| `emit_live_thread_summary` fails | committed, counters correct | live badge counts stale until the next page fetch | success | `side_effects.rs:724-815`, spawned task, `warn!` on every failure branch |
| `emit_system_message` insert fails | committed | no kind:40099 tombstone / notice | success (side effect returns `Ok`) | `side_effects.rs:690-697` — insert failure is `warn!`-ed and the function still returns `Ok(())` |
| `emit_membership_notification` fails | committed | agent never learns to resubscribe | success | `side_effects.rs:1248-1256`, `:1319-1327`, `:1766-1774` |
| `emit_group_discovery_events` fails | committed | 39000/39001/39002 stale | success | `side_effects.rs:1244`, `:1315`, `:1553`, `:1762`, `:1875` |
| Redis `publish_event` fails | committed | `local_event_ids` echo-dedupe entry **rolled back** so a later Redis retry is not swallowed | success | `side_effects.rs:790-800`, `:869-879`; same pattern in `handlers/event.rs:390` |
| Audit enqueue channel closed | committed | audit entry lost | success | `handlers/event.rs:597-600` — `error!` + `buzz_audit_send_errors_total` |
| Command mutation succeeds, `tx.commit()` fails | **not** stored | mutation persisted | `Internal("error: commit transaction: …")` | `command_executor.rs:92-98` documents this exact window; safety rests on `open_dm` / `hide_dm` / `update_approval` / `upsert_workflow` being idempotent so a retry converges |
| 28936 NIP-43 publishes fail | member already removed | 8001/13534 not published | `accepted: true, "info: you have left this relay"` | `ingest.rs:1909-1922` |

#### 3c. Fail-closed vs fail-open

Fail-**closed** (error propagates, write refused):
- restriction-state lookup (`ingest.rs:1633-1641`, comment: "a DB error must not let a
  banned/timed-out actor write");
- 9021 membership check (`side_effects.rs:1861-1866`, "Fail closed on DB errors rather
  than falling through to add_member");
- 9005 target lookup (`side_effects.rs:552-558`, "Fail closed: missing target → reject");
- 9002 `ttl` parse in the *side effect* (`side_effects.rs:1454-1464`, "a parse failure must
  reject, never silently clear the TTL to permanent");
- 30617 pointer establishment (`side_effects.rs:2528-2559`).

Fail-**open** (error swallowed):
- every side effect after storage (BR-IN-69);
- `insert_mentions` (`buzz-db/src/lib.rs:1394-1399`);
- `ensure_user` in the command executor (`command_executor.rs:60-63` — `warn!` and
  continue, even though the comment at `:44-46` says it exists to satisfy a foreign-key
  requirement; a subsequent FK violation would then surface as `Internal`);
- kind:0 NIP-05 UNIQUE collision, which retries the profile write **without** the handle
  rather than failing (`side_effects.rs:1174-1195`);
- kind:0 off-domain NIP-05, silently cleared because "the event is already persisted and
  can't be rejected" (`side_effects.rs:1150-1153`).

---

### 4. Fan-out path (via `handlers/event.rs`)

`dispatch_persistent_event` (`handlers/event.rs:326-367`) is called from
`ingest.rs:2375` (reaction), `:2513` (generic), and from `side_effects.rs:940`, `:2872`,
`:3045`, `:3132`, `:3195`. It:
1. awaits `enqueue_event_created_audit` (bounded mpsc, capacity 1000 — deliberate
   backpressure, `handlers/event.rs:540-548`);
2. spawns a task that marks the event local, publishes to Redis, then fans out to local
   subscribers, then spawns workflow evaluation.

Consequence: **NIP-01 `OK` is returned before Redis publish, local fan-out, or workflow
triggering complete** — stated explicitly at `handlers/event.rs:320-325`.

Workflow triggering is skipped for `is_workflow_execution_kind` (46001–46012),
`is_command_kind` (7 kinds), relay-signed events tagged `buzz:workflow`, and kind 1059
(`handlers/event.rs:497-502`). The workflow lookup is community-scoped, with the rationale
that a colliding channel UUID in another community must not trigger this community's
workflows (`handlers/event.rs:511-517`).

Three emitters deliberately bypass `dispatch_persistent_event` and call
`fan_out_event_to_local_subscribers` directly, skipping audit and workflow evaluation:
`emit_membership_notification` (`side_effects.rs:882-885`), `publish_nip43_delta`
(`:2905-2907`), `emit_initial_ref_state` (`:2755-2762`). The first documents this as
"Fan-out only — skip search indexing and workflow evaluation" (`side_effects.rs:855`).
`emit_live_thread_summary` does the same for a never-stored event (`:801-808`).

---

### 5. Subscription lifecycle coupling

`side_effects.rs` reaches directly into the connection and subscription registries — the
only non-transport module that does:

| Function | Effect | Trigger |
|---|---|---|
| `evict_live_channel_subscriptions` `:39` | closes a specific pubkey's channel subs across all their connections | 9001 (`:1295`), 9022 (`:1934`) |
| `evict_conn_channel_subscriptions` `:56` | removes from `sub_registry`, removes from the conn's local map, `release_topic`, sends `CLOSED restricted: channel access revoked` | the three above |
| `evict_non_member_channel_subscriptions` `:95` | closes subs for connections whose pubkey is not a current member | 9002 open→private (`:1437`) |
| `evict_all_channel_subscriptions` `:128` | closes every sub on a channel | ephemeral-channel reaper (`main.rs:672`) |

The reason string `channel access revoked` is chosen because it is in the client's
drop-set, so a connected agent drops one channel without reconnecting
(`side_effects.rs:118-125`).

---

### 6. Media integration (`imeta.rs`)

Read-only against `buzz_media::MediaStorage`: 3 `get_sidecar` reads and 3 `head` calls per
imeta tag worst case (`imeta.rs:246`, `:252`, `:290`, `:308`, `:328`). No writes, no
deletes, no retention interaction. Uploads happen out-of-band via Blossom; ingest only
proves the blob already exists and that the claimed metadata matches the sidecar.

Trust boundary: the sidecar is authoritative for `ext`, `mime_type`, `size`, and
`duration_secs`; the event's claims are checked *against* it, never the reverse
(`imeta.rs:259-278`). The upload validator's deny-list is therefore the real content gate,
and `validate_imeta_tags` only needs a structural MIME check — reasoned out at
`imeta.rs:71-76`.

Per-tenant media base URL: `media_base_url_for_tenant(&state.config.relay_url,
tenant.host())` (`ingest.rs:2211-2212`), so a tenant-host URL is accepted only when it
matches that tenant's base (tested at `imeta.rs:438-449`).

---

### 7. Git object store integration

`side_effects.rs` is the only handler that touches `state.git_store`:
`put_manifest` (`:2642`), `put_pointer(Precond::IfNoneMatchStar)` (`:2652`),
`get_pointer` (`:2664`, `:2714`). CAS semantics: `CasOutcome::Won` → success;
`CasOutcome::LostRace` → success **only if** the existing pointer names the same empty
manifest digest, otherwise a hard error rather than silently accepting a stale pointer
(`:2670-2691`). `ensure_manifest_pointer` (`:2704-2731`) is the tolerant re-announce
variant: any existing pointer is left untouched, an absent one is repaired.

The invariant maintained is "repo announced ⟺ pointer exists", so the read path can treat
pointer-absent as never-announced and keep `info_refs`'s fail-closed 404 unambiguous
(`side_effects.rs:2557-2571`).

---

### 8. Conformance-trace integration

`ingest_event` (`ingest.rs:1393`) wraps the pipeline in `EmitGuard::arm`
(`:1408-1412`), passes the counting tracer down, then maps terminal errors to a single
`SanitizedError` action (`:1436-1443`). Emitted actions: `AuthCheck` (`:1817-1825`),
`WriteInsert` / `WriteDuplicate` (`:2353-2376`, `:2493-2511`), `WriteInsertGlobal`
(`:2215-2222`, `:2506-2509`, plus `emit_product_feedback_success` `:133-154`).
Production tracer is `NoopTracer` (`state.rs:798`), so this is inert outside conformance
tests. Spec reference: `docs/spec/MultiTenantRelay.tla`, cited at `ingest.rs:1392`.


## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Integrations

---

#### 1. Dependency summary

| Collaborator | Reached via | Surface used |
|---|---|---|
| `buzz-db` | `state.db` | 27 distinct methods (§2) |
| `buzz-media` | `state.media_storage`, free fns | 5 storage methods + 6 pipeline/helper fns (§3) |
| `buzz-search` | `state.search` | `SearchService::search` (§4) |
| `buzz-pubsub` | `state.pubsub`, `state.nip98_replay`, `state.admission_rate_limiter` | presence bulk read, Redis replay seen-set, Redis rate limiter (§5) |
| `buzz-auth` | `state.auth`, free fns/traits | `verify_nip98_event`, `Scope::all_known`, `Nip98ReplayGuard`, `LimitType`, `DEFAULT_REPLAY_TTL_SECS`, `AuthConfig::rate_limits` (§6) |
| `buzz-core` | types | `TenantContext`, `CommunityId`, `StoredEvent`, `kind::*`, `filter::{filters_match, reader_authorized_for_event}`, `tenant::{normalize_host, relay_url_authority}` |
| `buzz-audit` | `state.audit_tx` | one `NewAuditEntry` for `MediaUploaded` |
| `buzz-workflow` | `state.workflow_engine` | `WorkflowDef`, `TriggerDef`, `TriggerContext`, `executor::execute_from_step`, `finalize_run` |
| `buzz-sdk` | free fn | `nip_oa::verify_auth_tag` |
| `buzz-relay-mesh` | `state.mesh()` | `RelayPeerTransport`, `ReliableStreamRouter`, `SessionDirectory` |
| `nostr` crate | ubiquitous | `Event`, `Filter`, `EventBuilder`, `Tag`, `PublicKey`, `EventId`, `SingleLetterTag` |
| `pulldown_cmark` | `invites.rs:132` | Markdown → HTML for policy pages |
| `url` crate | `admin/mod.rs:262-264` | parsing `imeta` attachment URLs |
| `infer` crate | `media.rs:50`, `:369-372`, `:380-382` | content sniffing |

**No outbound HTTP is issued from any file in this module group.** Every `reqwest` use in the
relay lives elsewhere (`buzz-workflow/src/executor.rs`). Grep for `reqwest` across the 13 assigned
files returns zero non-test hits.

---

#### 2. `buzz-db` call patterns

| Method | Called from | `file:line` | Notes |
|---|---|---|---|
| `is_relay_member(community, pubkey_hex)` | membership gate (agent, then owner) | `api/mod.rs:73-76`, `:92-96` | two round trips on the NIP-OA delegation path |
| `ensure_user(community, pubkey)` | NIP-OA backfill | `api/mod.rs:184-188` | called twice (agent + owner) before the FK-bearing write |
| `set_agent_owner` / `is_agent_owner` | NIP-OA backfill | `api/mod.rs:200-212` | first-write-wins, then confirm-same-owner |
| `query_events(&EventQuery)` | catch-all `/query`, aux closure | `bridge.rs:492-496`, `:1252-1255`, and count fallbacks `:1483`, `:1547` | `/query` catch-all runs **bounded-concurrent** with `futures_util::buffered(FILTER_QUERY_CONCURRENCY = 4)` and order-preserving post-processing |
| `count_events(&EventQuery)` | `/count` pushdown | `bridge.rs:1479`, `:1546` | only on the fully-pushable path; skipped entirely when the filter can match kind 30175 (`bridge.rs:1477`, `:1543`) |
| `get_channel_window(community, ch, limit, cursor, kinds)` | channel-window filter | `bridge.rs:466-476` | single call returning rows + `has_more` + `next_cursor` + per-row `thread_summary` |
| `get_thread_replies(community, root, depth, limit, cursor)` | thread filter | `bridge.rs:1153-1162` | keyset cursor encoded BE-i64 ‖ id bytes |
| `get_events_by_ids(community, &[&[u8]])` | FTS hydration | `bridge.rs:1701-1705` | batch fetch, then a `HashMap` to restore FTS relevance order |
| `query_feed_mentions` / `query_feed_needs_action` / `query_feed_activity` | feed filter | `bridge.rs:1066-1090` | one call per requested feed type, budget shared |
| `get_workflow(community, id)` | webhook | `bridge.rs:1809-1812` | any error ⇒ generic 404 |
| `create_workflow_run` / `update_workflow_run` | webhook | `bridge.rs:1879-1883`, `:1877-1886` | the update runs inside the detached task |
| `list_moderation_reports` / `list_moderation_actions` / `list_community_restrictions` | moderation reads | `bridge.rs:2102-2109`, `:2107-2111`, `:2124-2128` | limits clamped to ≤500 before the call |
| `get_user(community, pubkey)` | upload attribution | `media.rs:262-269` | best-effort; `.ok().flatten()` degrades to no display name |
| `get_user_by_nip05(community, name, domain)` | NIP-05 | `nip05.rs:50-54` | miss and error both fold into the empty response (`_ =>` arm at `:64`) |
| `get_relay_member(community, hex)` | invite mint authz | `invites.rs:234-239` | absent member ⇒ `role = ""` ⇒ 403 |
| `claim_relay_membership(community, hex, role, policy_version)` | invite claim | `invites.rs:325-338` | returns `was_inserted` for idempotency |
| `archive_community_owned_by(host, owner, deployment_host)` | operator | `operator.rs:234-239` | `None` ⇒ 404 |
| `unarchive_community_owned_by(host, owner)` | operator | `operator.rs:288-292` | `None` ⇒ 404 |
| `list_communities_owned_by(owner_hex)` | operator | `operator.rs:325-328` | **not** community-scoped (control plane) |
| `lookup_community_by_host_for_management(host)` | operator availability | `operator.rs:480-484` | separate from the admission lookup so archived rows are visible |
| `lookup_community_host(community)` | post-transfer publish | `operator.rs:437-441` | |
| `transfer_ownership(community, new, expected)` | operator | `operator.rs:392-396` | returns the `TransferResult` enum mapped to 200/404/409 |
| `admin_list_reports(...8 args)` / `admin_get_report(id)` / `admin_list_feedback(100)` / `admin_get_feedback(id)` | admin | `admin/mod.rs:101-111`, `:132-134`, `:155`, `:184-186` | **deployment-wide**, no `CommunityId` parameter on 3 of the 4 |
| `ensure_configured_community(host)` | tests only | `bridge.rs:3433`, `invites.rs:565`, `media.rs:1015` | |
| `ping()` | readiness (outside group) | `router.rs:249` | |

Cache reads/writes owned by `AppState` but driven from here:
`get_accessible_channel_ids_cached` (`bridge.rs:1002`, `:1425`),
`author_type_cache.insert` and `observer_owner_cache.insert` after a NIP-OA materialization
(`api/mod.rs:215-224`).

#### 3. `buzz-media` call patterns

| Call | From | `file:line` |
|---|---|---|
| `auth::verify_blossom_auth_event(event, Some(tenant.host()), 3600)` | upload extractor | `media.rs:177` |
| `auth::verify_blossom_get_auth(event, sha256, Some(tenant.host()), 3600)` | read gate | `media.rs:502` |
| `process_video_upload(storage, cfg, tenant, auth_event, stream, content_length, attribution)` | streaming video | `media.rs:344-353` |
| `process_upload(...)` (buffered image) | image path | `media.rs:375-383` |
| `process_file_upload(...)` (buffered generic) | generic path | `media.rs:390-398` |
| `MediaStorage::read_sidecar_mime(tenant, hash)` | serve + head | `media.rs:637-641`, `:648-652`, `:812-816`, `:822-826` |
| `MediaStorage::get_sidecar(tenant, hash)` | ext agreement + key resolution | `media.rs:653-657`, `:829-833`, `:868-871` |
| `MediaStorage::head_with_metadata(key)` | size for 200/206/HEAD | `media.rs:673-677`, `:705-710`, `:839` |
| `MediaStorage::get_stream(key)` | 200 body | `media.rs:678` |
| `MediaStorage::get_range(key, start, end)` | 206 body | `media.rs:730` |
| `looks_like_iso_bmff`, `serve_inline`, `parse_public_ip`, `parse_port` | helpers | `media.rs:51`, `:663`, `:279-280` |
| `MediaError` as the handler error type (its own `IntoResponse`) | all media handlers | `buzz-media/src/error.rs:107-162` |

**Key direction-of-dependency fact:** `verify_blossom_get_auth` is defined in `buzz-media`
(`auth.rs:207`) but its **only** call site in the whole repo is `media.rs:502` in this module —
i.e. `buzz-media` never gates reads itself; the gate lives here behind `require_media_get_auth`.

##### S3 access

All object access is indirect through `MediaStorage`; this module never constructs an S3 client and
never handles credentials. Object keys are content-addressed (`{sha256}.{ext}`) and derived from
either the request path (after `validate_media_path`) or the sidecar's `ext` (re-validated by
`is_safe_ext`, `media.rs:875-879`) — client input never reaches the key builder unvalidated.
Sidecar/`_uploads/` keys are structurally unreachable through the serve path
(`media.rs:547-583`; test `:1250-1264`).

#### 4. `buzz-search`

One integration point: `handle_bridge_search` (`bridge.rs:1616-1749`).

| Element | Detail | `file:line` |
|---|---|---|
| Entry | `state.search.search(&SearchQuery)` | `bridge.rs:1694-1698` |
| `SearchQuery` fields | `community`, `q`, `channel_scope`, `kinds`, `authors`, `since`, `until`, `page`, `per_page`, `mode` | `bridge.rs:1666-1704` |
| `ChannelScope` | `Channels(valid_uuids)` when `#h` present and intersects accessible, else `build_search_channel_scope_filter(accessible, include_global = true)`; `None` ⇒ early `Ok([])` | `bridge.rs:1618-1650` |
| `SearchMode` | `Prefix` on `search_mode`/`searchMode == "prefix"`, else `FullText` | `bridge.rs:368-378` |
| Post-filter | FTS returns only ids; full events are fetched from `buzz-db` and re-checked by `search_hit_accepted` (all non-pushed constraints + channel scope + reader auth) | `bridge.rs:1590-1607`, `:1717-1719` |
| Error mapping | `internal_error("search error: …")` and `internal_error("search fetch error: …")` → generic 500 | `bridge.rs:1697`, `:1694` |

#### 5. `buzz-pubsub` / Redis

| Integration | Detail | `file:line` |
|---|---|---|
| Presence bulk read | `state.pubsub.get_presence_bulk(tenant, &pubkeys)`; failure ⇒ `unwrap_or_default()` (empty) | `bridge.rs:1969-1975` |
| NIP-98 replay seen-set | `RedisNip98ReplayGuard` behind `Arc<dyn Nip98ReplayGuard>` (`state.rs:710-711`); community-scoped `try_mark` for bridge/invites, deployment-scoped `try_mark_in_scope("operator-management", …)` for operator | `bridge.rs:141`, `:158-161`; `operator.rs:108-122` |
| Rate limiter | `RedisRateLimiter` (`state.rs:712`) via `admission::check_principal` with `LimitType::ApiCalls`, 60 s window | `bridge.rs:31-38` |
| Cluster disconnect fan-out | `state.disconnect_community_clusterwide(&tenant)` after archive; failure ⇒ 503 retryable | `operator.rs:243-264` |
| NIP-43 side effects | `handlers::side_effects::publish_nip43_member_added` / `publish_nip43_membership_list` after invite claim and after ownership transfer — both best-effort | `invites.rs:344-355`; `operator.rs:445-455` |
| Mesh session directory | `SessionDirectory` (Redis fenced leases) via `ReliableStreamRouter::join` | `mesh_demo.rs:71-77`, `:100` |

Two limiters in this module are **not** Redis-backed and therefore not cluster-consistent:
`media_upload_rate_limiter` + `media_uploads_in_flight` (DashMap, `state.rs:592`, `:600`) and
`invite_claim_rate_limiter` (moka, `state.rs:597-598`).

#### 6. `buzz-auth`

| Item | Use | `file:line` |
|---|---|---|
| `verify_nip98_event(json, url, method, body)` | the actual signature/URL/method/payload check | `bridge.rs:110-111` |
| `Nip98ReplayGuard` trait | injected via `AppState`; four test doubles implement it (`AlwaysErrGuard`, `AlwaysFreshReplayGuard` ×3, `SeenOnceReplayGuard`) | `bridge.rs:2348-2356`, `:3268-3281`; `invites.rs:415-427`, `:1103-1128`; `operator.rs:551-563` |
| `DEFAULT_REPLAY_TTL_SECS` | TTL for both replay scopes | `bridge.rs:159`; `operator.rs:113` |
| `LimitType::ApiCalls` | rate-limit bucket | `bridge.rs:34` |
| `Scope::all_known()` | 16 scopes granted to every HTTP ingest | `bridge.rs:829` |
| `state.auth.config().rate_limits.human_api_calls_per_min` | the per-minute budget | `bridge.rs:29` |
| `AuthError` | surfaced only through guard failures, mapped to 401 | `bridge.rs:167-176` |
| `nip42::verify_nip42_event` | tests only in this module; production caller is `handlers/auth.rs:80-81` using `bridge::nip42_expected_relay_url` | `bridge.rs:2797-2804` |

#### 7. Reverse dependencies — who calls into this module

| Consumer | Symbol | `file:line` |
|---|---|---|
| `handlers/auth.rs` (WS NIP-42 door) | `relay_members::enforce_relay_membership`, `extract_nip_oa_owner`, `materialize_nip_oa_owner`, `bridge::nip42_expected_relay_url` | `handlers/auth.rs:217`, `:137`, `:246`, `:258`, `:81` |
| `audio/handler.rs` (huddle WS) | `relay_members::enforce_relay_membership`, `bridge::nip42_expected_relay_url` | `audio/handler.rs:244`, `:219` |
| `api/git/transport.rs` | `relay_members::enforce_relay_membership` | `git/transport.rs:211` |
| `handlers/ingest.rs` | `api::validate_imeta_tags`, `api::verify_imeta_blobs`, `api::media::media_base_url_for_tenant` | `ingest.rs:2239`, `:2215`, `:2212` |
| `handlers/product_feedback.rs` | same three | `product_feedback.rs:31-33` |
| `handlers/imeta.rs` | `api::media::is_safe_ext` | `imeta.rs:378`, `:406` |
| `handlers/side_effects.rs` | `api::nip05::canonicalize_nip05` | `side_effects.rs:1145` |
| `api/admin/mod.rs` | `api::media::serve_blob_for_tenant`, `api::media::is_safe_ext` | `admin/mod.rs:226`, `:283` |
| `handlers/command_executor.rs` | `webhook_secret::{generate_webhook_secret, inject_secret, extract_secret}` | `command_executor.rs:713-718` |
| `router.rs` | every routed handler + `api::admin::{router, is_admin_host}` | `router.rs:39-128`, `:59`, `:145`, `:264` |

**Doc delta:** `api/mod.rs:36-38` states the membership gate is "Called by `media.rs`, `bridge.rs`,
`git/transport.rs`, and `audio/handler.rs`." It omits `handlers/auth.rs:217`, which is the WebSocket
door — arguably the most important caller.

#### 8. Metrics emitted

| Metric | Labels | `file:line` |
|---|---|---|
| `buzz_admission_rejections_total` | `transport="http"`, `reason="quota"\|"unavailable"` | `bridge.rs:40`, `:49` |
| `buzz_events_rejected_total` | via `reject_with_transport("http", "invalid"\|"auth"\|"error")` | `bridge.rs:734`, `:857`, `:862`, `:868` |
| `buzz_count_fallback_rejections_total` | none | `bridge.rs:1492`, `:1558` |
| `buzz_media_upload_rejections_total` | `reason="rate_limit"\|"concurrency"` | `media.rs:216`, `:222` |
| `buzz_media_legacy_upload_route_total` | none | `media.rs:315` |
| `buzz_media_uploads_total` | `mime` (6-value allowlist), `community` (tenant host) | `media.rs:419-424` |
| `buzz_audit_send_errors_total` | none | `media.rs:443` |
| `buzz_users_created_total` | `community` | `api/mod.rs:189-193` |

Note `buzz_media_uploads_total` carries an unbounded-cardinality `community` label (one series per
tenant host) — acceptable at current tenant counts, a concern on a large multi-tenant deployment.


## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: Integrations

#### 1. Integration inventory

| Dependency | Direction | Surface | Criticality |
|---|---|---|---|
| S3 / MinIO (`rust-s3` 0.37) | out | 4 distinct bucket operations | **hard** — source of truth; startup probe is fatal |
| `git` binary (subprocess) | out | 10 production invocation sites, 9 distinct subcommands | **hard** — no in-process git |
| `buzz-db` (Postgres) | out | 4 read methods + 1 write | hard for push authz; the write is best-effort |
| `buzz-core` | in-proc | `TenantContext`, `CommunityId`, `MemberRole`, `git_perms::*` | hard |
| `buzz-auth` | in-proc | `nip98::verify_nip98_event` | hard |
| Relay event fan-out | out | `fan_out_event_to_local_subscribers` (bypasses `dispatch_persistent_event`) | best-effort |
| Local HTTP loopback | out+in | pre-receive hook → `POST /internal/git/policy` | **hard** — fail-closed |
| Local filesystem | out | tempdirs, tempfiles, pack cache | hard |
| `metrics` crate | out | 8 counters, 11 histograms, 3 gauges | observability |
| Runtime image | build | `curl` + `openssl` must be installed for the hook | **hard**, asserted by a test |

#### 2. Object store (every call)

Client construction — `store.rs:187-219`:
- `Region::Custom { region, endpoint }`, `Bucket::new(...).with_path_style()` (`store.rs:206-214`). Path-style is unconditional (MinIO compatibility; AWS accepts both).
- Credential selection (`store.rs:198-210`): both keys non-empty ⇒ `Credentials::new`; **both empty** ⇒ `Credentials::default()` (AWS chain: env, profile, IRSA web-identity, container, IMDS); exactly one empty ⇒ `StoreError::Config` (pinned `store.rs:961-982`).
- **Credentials and bucket are shared with `buzz-media`**: `state.rs:694-701` passes `config.media.s3_{endpoint,access_key,secret_key,bucket,region}` and `.expect("media storage was already constructed with this S3 config")`. There is no separate git bucket at runtime, despite `BUZZ_GIT_S3_*` appearing in probe/test helpers (`cas_publish.rs:1580-1601`).

| Bucket op | Site | Purpose | Headers / notes |
|---|---|---|---|
| `put_object_with_content_type_and_headers` | `store.rs:262-268` (`put_immutable`) | write `packs/<sha256>`, `manifests/<sha256>` | `If-None-Match: *`; 2xx ⇒ key, 412 ⇒ key (idempotent), other ⇒ `Backend` |
| `put_object_with_content_type_and_headers` | `store.rs:297-306` (`put_idx`) | write `idx/<pack-digest>` | `If-None-Match: *`; same 412 handling |
| `put_object_with_content_type_and_headers` | `store.rs:489-501` (`put_pointer`) | the CAS | `If-Match: <etag>` or `If-None-Match: *`; content type `application/json`; result routed through `classify_cas` (`:516-546`) |
| `put_object_with_content_type_and_headers` | `store.rs:886-899` (`put_immutable_raw`) | probe phase 3 only — returns the raw status so 412s are countable | `If-None-Match: *` |
| `get_object` | `store.rs:348-352` (`get`) | fetch any object; 404 ⇒ `NotFound` | no range requests anywhere — full-body GET |
| `get_object` | `store.rs:456-478` (`get_pointer`) | atomic `(ETag, body)` snapshot | fails if no `etag`/`ETag` header |
| `head_object` | `store.rs:407-425` (`get_limited`) | size pre-check before download | followed by a post-read length re-check (`:429-437`) |
| `delete_object` | `store.rs:618`, `:728`, `:871` | probe scratch cleanup only | **the protocol never deletes packs/manifests/pointers** (doc §Axioms A1 no-deletion rule) |

Read/write call graph:

```
receive_pack → hydrate_for_write → get_pointer, get_verified_limited(manifest),
                                    [pack_cache] get_verified_limited(pack), get_idx
             → cas_publish       → put_pack, put_idx, put_manifest, put_pointer
                                    [on LostRace] get_pointer, get_verified
info_refs fast → load_manifest_for_read → get_pointer, get_verified_limited
info_refs slow / upload_pack → hydrate_for_read → same as hydrate_for_write minus ParentState
announce (side_effects) → put_manifest, put_pointer, get_pointer
startup → run_conformance_probe → put_pack, get_verified, put_pointer, get_pointer,
                                   put_immutable_raw, delete_object
```

Note: `store.rs:25` carries a module-wide `#![allow(dead_code)]` whose comment ("wired in by the push path in a follow-up commit") is stale.

#### 3. `git` subprocess invocations

Exactly **10** production `Command::new("git")` sites (3 in `transport.rs`, 6 in `cas_publish.rs`, 1 shared helper in `hydrate.rs`). All go through `harden_git_env` except `hydrate::run_git`, which hand-rolls a *different* env.

##### Environment

`harden_git_env` (`transport.rs:294-310`):

| Var | Value |
|---|---|
| — | `env_clear()` first |
| `PATH` | inherited (`std::env::var("PATH").unwrap_or_default()` — empty string if unset) |
| `GIT_HTTP_EXPORT_ALL` | `1` |
| `GIT_CONFIG_NOSYSTEM` | `1` |
| `GIT_CONFIG_GLOBAL` | `/dev/null` |
| `HOME` | `/dev/null` |

`hydrate::run_git` (`hydrate.rs:451-465`): `env_clear()`, `PATH` (only if set), `GIT_CONFIG_NOSYSTEM=1`, `HOME=<cwd>`. **Missing `GIT_CONFIG_GLOBAL`** — its doc comment claims it "matches transport.rs's harden_git_env semantics" (`hydrate.rs:456-457`), which is inaccurate; the `HOME=<cwd>` trick is what makes the global lookup miss.

`receive_pack` adds, on top of `harden_git_env` (`transport.rs:917-931`):

| Var | Value |
|---|---|
| `BUZZ_HOOK_URL` | `http://127.0.0.1:<config.bind_addr.port()>/internal/git/policy` |
| `BUZZ_HOOK_SECRET` | `config.git_hook_hmac_secret` |
| `BUZZ_REPO_ID` | stripped repo id |
| `BUZZ_REPO_OWNER` | owner hex from the URL |
| `BUZZ_COMMUNITY_ID` | resolved community UUID |
| `BUZZ_PUSHER_PUBKEY` | authenticated pusher hex |
| `GIT_CONFIG_COUNT` | `1` |
| `GIT_CONFIG_KEY_0` | `core.hooksPath` |
| `GIT_CONFIG_VALUE_0` | `<workspace>/hooks` |

##### Invocation table

| # | Site | argv | cwd | stdin | stdout | stderr | Timeout | kill_on_drop |
|---|---|---|---|---|---|---|---|---|
| 1 | `transport.rs:645-653` | `git {upload-pack\|receive-pack} --stateless-rpc --advertise-refs <workspace>` | inherited | inherited | `NamedTempFile` in `git_repo_path` | tempfile (64 KiB prefix logged) | 120 s (`:660`) | yes |
| 2 | `transport.rs:1019-1027` | `git receive-pack --stateless-rpc <workspace>` | inherited | **piped** — request body pumped by a spawned task (`:1037-1064`) | tempfile | tempfile | 300 s (`:1067`), body task aborted on timeout | yes |
| 3 | `transport.rs:1423-1434` | `git upload-pack --stateless-rpc <workspace>` (`extra_args` always empty) | inherited | **piped** — body pumped by a detached task (`:1442-1467`) | **piped → HTTP body** | `null` | 300 s in-band deadline (`:1471`), child killed (`:1315-1331`) | yes |
| 4 | `cas_publish.rs:284-296` | `git for-each-ref --format=%(refname) %(objectname)` | workspace | inherited | tempfile in scratch | tempfile | none | no |
| 5 | `cas_publish.rs:337-345` | `git symbolic-ref --quiet HEAD` | workspace | inherited | in-memory (`.output()`) | in-memory | none | no |
| 6 | `cas_publish.rs:409-415` | `git index-pack <tmp>/pack-<digest>.pack` | private tempdir | inherited | in-memory | in-memory | none | no |
| 7 | `cas_publish.rs:524-541` | `git pack-objects --revs --stdout -q --threads=1 --window 10 --window-memory=67108864` | workspace | piped (rev-spec lines) | tempfile in scratch | tempfile | 300 s (`:433-460`) | yes |
| 8 | `cas_publish.rs:608-620` | `git count-objects -v` | workspace | piped (empty) | tempfile | tempfile | 300 s | yes |
| 9 | `cas_publish.rs:707-729` | `git pack-objects --revs -q --threads=1 --window 10 --window-memory=67108864 --max-pack-size=<n> <tmp>/compact` | workspace | piped (deduped tips) | tempfile in compaction tempdir | tempfile | 300 s inner, 600 s outer (`:1058`) | yes |
| 10 | `hydrate.rs:451-470` (`run_git`, 4 arg sets) | `git init --bare --quiet` (`:182`); `git symbolic-ref HEAD refs/heads/main` (`:183`); `git verify-pack <idx>` (`:393`); `git index-pack <pack>` (`:420`) | caller-supplied | inherited | in-memory | in-memory | **none** | yes |

Observations:

- Sites 4, 5, 6, 10 have **no timeout**. Site 10 covers `index-pack` on an attacker-influenced pack (through the pack cache), so a pathological pack can occupy a semaphore permit for an unbounded time.
- Sites 1, 4, 5, 6, 10 do not set `stdin`, so the child inherits the relay's stdin.
- All repo paths are passed as a **single argv element** (`Command::arg`, no shell), so `--stateless-rpc <path>` cannot be word-split. Paths are OS-generated tempdir paths, never user strings.
- `--threads=1` and `--window-memory` bound one `pack-objects`; total CPU is bounded only by `git_semaphore` (20) and the compaction semaphore (1).
- No `git gc`, `repack`, `fsck`, `prune`, or `update-ref` is ever invoked. Refs are written as loose files directly (`hydrate.rs:355-371`).
- Tests additionally shell out to `bash` twice (`policy.rs:682`, `:757`) to cross-verify the hook's HMAC against the Rust implementation.

#### 4. `buzz-db` (Postgres)

| Call | Site | Purpose |
|---|---|---|
| `query_events(EventQuery{ kinds:[30617], pubkey:owner, d_tag:repo_id, global_only:true, limit:1, ..for_community(community) })` | `policy.rs:254-284` | resolve the repo announcement for protection rules + channel binding. Direct DB query, so it bypasses the relay's `kinds`-required p-gate. |
| `get_channel(community, channel_id)` | `policy.rs:308-330` | archived-channel read-only check |
| `is_agent_owner(community, owner_bytes, pusher_bytes)` | `policy.rs:330-368` | NIP-OA managed-agent owner authority |
| `get_member_role(community, channel_id, pusher_bytes)` | `policy.rs:353-395` | channel role for non-owners |
| `insert_event(community, kind:30618, None)` | `transport.rs:1693-1700` | persist the derived ref-state event; `(_, false)` means DB dedup |
| `crate::tenant::bind_community(&state.db, raw_host)` | `transport.rs:128-130` | host → community (uses `db` as `HostResolver`) |

Every failure in the first four ⇒ 403 (fail-closed, `policy.rs:277-282`, `:322-326`, `:355-362`, `:388-392`). Failure of `insert_event` ⇒ warning only (`transport.rs:1727-1735`).

Not used by this module but part of the same feature: `git_repo_names` reservation and quota (`repo_name_owner`, `count_repos_for_owner`, `reserve_repo_name`, `release_repo_name`) live in `handlers/side_effects.rs:2463-2560`.

#### 5. `buzz-pubsub`

**Not used directly.** kind:30618 is fanned out through `crate::handlers::event::fan_out_event_to_local_subscribers` (`transport.rs:1701-1710`, and `handlers/side_effects.rs:2755-2761` for the announce path) — the *local* subscriber path only. It bypasses `dispatch_persistent_event`, so whatever cross-pod Redis publication that function performs does **not** happen for git ref-state events: a subscriber connected to a different pod will not see the 30618 in real time and must re-query. The code comments justify the bypass only in terms of the access gate no-op for `channel_id = None`.

#### 6. Relay event emission

| Event | Trigger | Signer | Path |
|---|---|---|---|
| kind:30618 (post-push) | successful CAS with `parent_digest != committed_digest` | relay keypair (`state.relay_keypair`) | `transport.rs:1662-1746` |
| kind:30618 (announce seed) | fresh kind:30617 reservation | relay keypair | `handlers/side_effects.rs:2728-2765` |

Tags: `["d", repo_id]`, one `[<refname>, <oid>]` per emittable ref, `["HEAD", "ref: <head>"]`, `["p", <actor-hex>]` (`manifest_event.rs:74-108`). Content is empty. Only `refs/heads/*` and `refs/tags/*` are emitted (`manifest_event.rs:117-127`), and refs with non-40/64-hex oids or malformed names are skipped silently (`:82-93`).

#### 7. The HMAC hook callback loop

```
receive_pack (transport.rs:858)
  └─ install_hook → <workspace>/hooks/pre-receive, 0o755        hook.rs:152-178
  └─ git receive-pack --stateless-rpc <workspace>               transport.rs:1019
        └─ pre-receive (bash)                                    hook.rs:32-150
             ├─ read "old new ref" lines from stdin              hook.rs:56
             ├─ git merge-base --is-ancestor old new             hook.rs:59-70  (quarantine env inherited)
             ├─ HMAC-SHA256 via `openssl dgst -sha256 -hmac`     hook.rs:118
             └─ curl --silent --max-time 10 -X POST $BUZZ_HOOK_URL
                                                                  hook.rs:129-139
                   └─ POST /internal/git/policy                   mod.rs:62 → policy.rs:173
                         ├─ require_localhost middleware          mod.rs:41-52
                         ├─ structural validation                  policy.rs:176-234
                         ├─ verify_hmac (constant-time)            policy.rs:159-171, :236-241
                         ├─ 30s TTL / 5s future skew               policy.rs:243-256
                         ├─ 4 buzz-db lookups                      policy.rs:254-395
                         └─ buzz_core::git_perms::evaluate_push    policy.rs:404-416
             └─ non-200 ⇒ echo body to stderr, exit 1 (deny)     hook.rs:141-148
```

Cross-boundary contract details:

- The bash side sets `LC_ALL=C` so `sort` order and `${#var}` byte lengths match Rust's byte comparison and `String::len()` (`hook.rs:36-38`).
- Refs are sorted by `ref_name` on both sides — bash `sort` on a `ref_name`-first line, Rust `sort_by_key(|r| r.ref_name.clone())` (`hook.rs:113-121`, `policy.rs:143-146`). Order-independence is pinned by `policy.rs:559-576`.
- The pre-image format agreement is verified by **two** tests that actually run `bash` + `openssl` and compare against `generate_hook_hmac`: `policy.rs:592-703` (two refs, unsorted input) and `:704-773` (single ref). These are the only tests of the module's most security-critical contract.
- `set -eo pipefail` plus `: "${VAR:?…}"` guards make a missing env var abort the hook (`hook.rs:35`, `:41-47`).
- Both `ref_name` and `repo_id` are JSON-escaped with `sed 's/\\/\\\\/g; s/"/\\"/g'` before interpolation (`hook.rs:74-76`, `hook.rs:126`).
- The relay's runtime container must ship `curl` and `openssl`; `hook.rs:184-206` parses the repo `Dockerfile`'s runtime stage and fails the unit test if either is missing.
- `is_ancestor` is *reported by the hook*, not recomputed by the relay, and it is HMAC-covered (pinned `policy.rs:512-520`). The relay trusts the hook's ancestry claim.

#### 8. Startup and shared state wiring

| Item | Site |
|---|---|
| `AppState.git_store` built with `.expect(...)` from the **media** S3 config | `state.rs:694-701` |
| `AppState.git_pack_cache` built with `.expect("git pack cache path must be available")` | `state.rs:702-709` |
| `AppState.git_semaphore = Semaphore::new(git_max_concurrent_ops)`; doc explicitly says it is **not** writer serialization | `state.rs:517-521`, `:729` |
| `git_router` + `git_policy_router` merged into the main router | `router.rs:48-50`, `:137-138` |
| Fatal A3 conformance probe before the listener opens | `main.rs:466-503` |
| `BUZZ_SERVE_GIT_WEB_GUI` gates SPA fallback for `/`, `/repos`, `/repos/*` | `router.rs:206-213` |
| DB-derived gauges `buzz_total_git_repos` / `buzz_community_git_repos` | `main.rs:1499`, `:1706-1725` |
| UDS listener uses `.into_make_service()` (no `ConnectInfo`) | `main.rs:1182` |
| `tokio-util` `io` feature enabled specifically to stream git stdout into the HTTP body | `crates/buzz-relay/Cargo.toml:31-34` |

#### 9. Metrics emitted

| Name | Type | Labels | Site |
|---|---|---|---|
| `buzz_git_semaphore_rejections_total` | counter | `operation` | `transport.rs:323-327` |
| `buzz_git_upload_pack_timeouts_total` | counter | — | `transport.rs:1364` |
| `buzz_git_upload_pack_stream_seconds` / `_bytes` | histogram | — | `transport.rs:1386-1389` |
| `buzz_git_hydrations_total` | counter | `outcome` ∈ {success, missing, invalid_pointer, manifest_error, store_error, hydrate_error, resource_limit} | `hydrate.rs:146` |
| `buzz_git_hydrate_seconds` | histogram | `outcome` | `hydrate.rs:147` |
| `buzz_git_hydrate_bytes` / `_packs` | histogram | — | `hydrate.rs:135-136` |
| `buzz_git_pack_cache_lookups_total` | counter | `result` ∈ {hit, miss, coalesced} | `pack_cache.rs:173`, `:197-200` |
| `buzz_git_pack_cache_populate_seconds` | histogram | `outcome` ∈ {success, bypass, error} | `pack_cache.rs:223-226` |
| `buzz_git_pack_cache_population_wait_seconds` | histogram | — | `pack_cache.rs:266` |
| `buzz_git_pack_cache_populations_active` | gauge | — | `pack_cache.rs:93`, `:268` |
| `buzz_git_pack_cache_bytes` / `_entries` | gauge | — | `pack_cache.rs:477-480` |
| `buzz_git_pack_cache_bypasses_total` | counter | — | `pack_cache.rs:302` |
| `buzz_git_pack_cache_copy_fallbacks_total` | counter | — | `pack_cache.rs:453` |
| `buzz_git_pack_cache_evictions_total` | counter | — | `pack_cache.rs:383` |
| `buzz_git_pack_compactions_total` | counter | `outcome` ∈ {success, fallback, cas_conflict, validation_error, publish_error} | `cas_publish.rs:968` |
| `buzz_git_pack_compaction_seconds` | histogram | `outcome` | `cas_publish.rs:969` |
| `buzz_git_pack_compaction_packs_before` / `_after` / `_bytes` | histogram | — | `cas_publish.rs:971-977` |
| `buzz_git_pack_compaction_required_failures_total` | counter | — | `cas_publish.rs:1129` |

**Gap:** no counter for push outcome (2xx / 409 conflict / 400 invalid-manifest / 413), and none for policy-endpoint allow/deny. CAS contention and authorization denials are therefore invisible in metrics — which matters because the design explicitly says "if contention ever shows up in metrics the fix is a short best-effort *local* lock" (doc §Scope). That signal does not currently exist.


## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: Integrations

---

#### 1. Dependency map

| Integration | Reached via | Files that touch it |
|---|---|---|
| `buzz-db` (Postgres) | `state.db` | all 12 except `moderation_authz` helpers' pure half |
| `buzz-pubsub` (Redis) | `state.pubsub`, indirectly via `state.disconnect_pubkey_clusterwide` / `dispatch_persistent_event` | `moderation_commands`, `moderation_notices`, `workflow_sink` |
| `buzz-audit` (hash chain) | `state.audit_tx` — **only transitively**, via `dispatch_persistent_event` | `moderation_notices`, `workflow_sink` (and `identity_archive` via ingest fall-through) |
| `buzz-media` / S3 | `state.media_storage` | `report` (sidecar lookup), `product_feedback` (blob verify), `storage_sweep` (bucket listing) |
| `buzz-search` | via `dispatch_persistent_event` | `moderation_notices`, `workflow_sink` |
| `buzz-workflow` | trait impl of `ActionSink` | `workflow_sink` |
| `buzz-core` | kinds, tenant, filter matching | all |
| `buzz-sdk` | `nip_oa::verify_auth_tag` | `identity_archive` |
| APNs push gateway | outbound HTTPS | `push_runtime` |
| `metrics` recorder (Prometheus) | `metrics::gauge!` / `counter!` | `storage_sweep`, `moderation_notices` |

---

#### 2. `buzz-db` (Postgres)

##### 2.1 Call inventory

| File | DB calls | Notable |
|---|---|---|
| `moderation_commands.rs` | `moderation_restriction_state` (`:105`), `ban_community_member` (`:169`), `unban_community_member` (`:248`), `timeout_community_member` (`:287`), `untimeout_community_member` (`:351`), `get_moderation_report_by_event` (`:414`), `resolve_moderation_report` (`:461`), `insert_moderation_action` (`:529`) | 8 distinct calls; **no transaction wraps them** |
| `moderation_authz.rs` | `get_relay_member` (`:98`, `:110`), `get_member_role` (`:126`) | 1–2 queries per authorization; `get_member_role` is unreachable |
| `moderation_notices.rs` | `open_dm` (`:102`), `unhide_dm` (`:127`), `query_events` (`:230`), `insert_event` (`:174`), `replace_addressable_event` (`:206`) | 5 calls per notice, minimum |
| `relay_admin.rs` | `get_relay_member` (`:135`, `:317`), `set_community_icon` (`:157`), `add_relay_member` (`:199`), `remove_relay_member_if_role` (`:243`), `remove_relay_member` (`:250`), `update_relay_member_role` (`:309`) | admin remove is a single conditional delete (TOCTOU-safe, `:235-241`) |
| `community_provisioning.rs` | `create_community_with_owner` (`:286`), `ensure_configured_community` (`:323`), `bootstrap_owner` (`:330`) | create-only path is one atomic call; convergence path is two |
| `report.rs` | `get_event_by_id` (`:56`), `insert_moderation_report` (`:79`) | |
| `product_feedback.rs` | `insert_product_feedback` (`:57`) | |
| `identity_archive.rs` | `get_relay_member` (`:241`), `query_events` (`:273`), `archive` (`:75`), `unarchive` (`:85`) | |
| `push_lease.rs` | `accept_push_lease_event` (`:563`) | single atomic transaction owning ordering + quota + endpoint namespace (`buzz-db/src/push.rs:206-212`) |
| `push_runtime.rs` | 13 distinct: `reap_exhausted_push_matches` (`:62`), `claim_due_push_match_batch` (`:72`), `active_push_match_leases` (`:102`), `membership_pairs` (`:115`), `enqueue_push_wakes` (`:184`), `complete_push_match_batch` (`:201`), `retry_push_match_batch` (`:207`/`:137`), `usage_community_hosts` (`:320`), `claim_due_push_wakes` (`:325`), `revalidate_push_wake` (`:356`/`:405`), `is_member` (`:375`), `disable_push_endpoint` (`:457`), `complete_push_wake` (`:438`/`:494`), `fail_push_wake` (`:363`/`:382`/`:412`/`:444`/`:471`/`:501`/`:535`), `retry_push_wake` (`:390`/`:541`) | heaviest DB consumer in the group |
| `workflow_sink.rs` | `lookup_community_host` (`:201`), `get_channel` (`:223`), `is_member_cached` (`:244`), `get_members` (`:275`), `get_users_bulk` (`:281`), `insert_event_with_thread_metadata` (`:339`) | 6 queries per workflow message |
| `storage_sweep.rs` | none directly — the host map is supplied by `main.rs` | |

##### 2.2 Failure modes

| Path | On DB error | Fail-open or fail-closed? |
|---|---|---|
| Restriction read (`moderation_commands.rs:107`) | `error: database error checking restriction state: {e}`, command rejected | **closed** |
| Ingest restriction gate (`ingest.rs:1645-1650`) | `IngestError::Internal` | **closed** |
| Auth-seam restriction read (`handlers/auth.rs:118-131`, `:145-152`) | `BanOutcome::DbError` ⇒ deny with `error: internal …`, distinguished from a real ban | **closed** |
| `authorize_moderation_action` (`moderation_authz.rs:99`, `:111`, `:127`) | `anyhow` error `?`-propagated, wrapped as `restricted: {e}` by `authz_denial` (`moderation_commands.rs:549`) | **closed, but the error string leaks a DB message under a `restricted:` prefix** |
| Ban/timeout write (`moderation_commands.rs:174`, `:292`) | `error: database error: {e}` | closed |
| Audit-row write (`moderation_commands.rs:544`) | `error: failed to write audit row: {e}` — **command fails after the ban already committed** | see §2.3 |
| Notice DM (any of 5 calls) | `anyhow` bubbles to `send_moderation_notice`'s caller, which logs and continues | **open by design** (`moderation_commands.rs:214-219`) |
| `relay_admin` any call | `database error: {e}`, wrapped by ingest as `invalid: database error: …` | closed |
| Report target resolution (`report.rs:58`) | `error: database error resolving report target: {e}` | closed |
| Push lease persistence (`push_lease.rs:572`) | `AcceptError::Internal("lease persistence failed")` — **the underlying `DbError` is discarded with `map_err(|_| …)`** | closed, but undiagnosable |
| Matcher context load (`push_runtime.rs:132-149`) | whole batch retried in 2 s | closed, retrying |
| Matcher per-job error (`:170-176`) | retry until `MAX_MATCH_ATTEMPTS = 8`, then discard as poison | bounded |
| Wake enqueue failure (`:186-198`) | contributing jobs retried; outbox dedup key absorbs partial commits | idempotent |
| `revalidate_push_wake` error (`:366-369`, `:414-417`) | `return` without touching the row — the claim lease simply expires | safe |
| `is_member` error during delivery (`:385-401`) | `retry_push_wake` in 2 s | closed |
| `usage_community_hosts` error (`:337`) | `error!` then the outer loop backs off | degraded, keeps looping |
| `workflow_sink` any call | `ActionSinkError::Database` → `WorkflowError::WebhookError` (`buzz-workflow/src/action_sink.rs:31-34`) → run fails | closed |
| `lookup_community_host` returns `None` (`workflow_sink.rs:205-210`) | `Database("workflow run community {id} is not mapped to a host")` | **closed by design** |

##### 2.3 Non-atomicity across the moderation write set

`handle_ban` performs four independent operations with no enclosing transaction: restriction read (`:105`), ban write (`:169`), audit write (`:180`), disconnect (`:195`), notice (`:204`). If the audit write fails, the ban is already durable but the command returns `error: failed to write audit row: …` (`:544`) — the client sees a failure for a ban that took effect. Same shape for timeout (`:287` then `:298`).

`handle_resolve` inverts the order deliberately — audit row **first** (`:453`), resolve **second** (`:461`) — with an in-code note that a lost-race resolve can leave an orphan audit row and that the residual window is tolerated (`:419-425`, `:469-474`).

---

#### 3. `buzz-pubsub` (Redis)

##### 3.1 Ban disconnect fan-out

`moderation_commands.rs:195-200` → `state.disconnect_pubkey_clusterwide` (`state.rs:1018-1050`):
1. Synchronous local socket close, fenced to the community (`state.rs:1025-1027`).
2. `tokio::spawn`ed `pubsub.publish_conn_control(&tenant, ConnControl::DisconnectPubkey { pubkey, event_id, reason })` (`state.rs:1043-1047`).

**Failure mode: fire-and-forget.** A Redis publish failure only emits `warn!("Failed to publish conn-control disconnect: {e}")` (`state.rs:1045`). The ban command still returns success. The in-code justification is that the durable ban row rejects the member again at auth (`state.rs:1039-1042`) and that the ingest write-path gate is the backstop for a surviving socket (`ingest.rs:1615-1622`).

**Consequence:** on another pod, a banned user's already-open socket keeps receiving events until (a) the Redis command arrives, (b) the socket reconnects and hits the auth seam, or (c) the user attempts a write and hits the ingest gate. Reads are not gated by the ingest path, so a missed publish means continued read access for the life of the socket.

The banning pod re-receives its own publish and no-ops; origin suppression was deliberately not added (`state.rs:1029-1031`).

##### 3.2 Notice and workflow fan-out

Both `moderation_notices.rs:178`/`:210` and `workflow_sink.rs:351` reach Redis only indirectly through `dispatch_persistent_event`. The workflow path discards the result entirely (`let _ =`, `workflow_sink.rs:351`), so a fan-out failure is invisible to the workflow run — the message is persisted and reported as sent.

Redis is also constructed directly inside two test helpers: `identity_archive.rs:445-448` and `workflow_sink.rs:580-588`. The latter deliberately points at `redis://127.0.0.1:1` (`workflow_sink.rs:578`) to prove the path works without a live Redis.

---

#### 4. `buzz-audit` (hash chain)

##### 4.1 Verified: no handler in this module writes an audit entry directly

`buzz-audit` declares **11** actions (`buzz-audit/src/action.rs:5-29`): `EventCreated`, `EventDeleted`, `ChannelCreated`, `ChannelUpdated`, `ChannelDeleted`, `MemberAdded`, `MemberRemoved`, `AuthSuccess`, `AuthFailure`, `RateLimitExceeded`, `MediaUploaded`.

Production emits exactly **2**: `EventCreated` (`handlers/event.rs:583`) and `MediaUploaded` (`api/media.rs:428`). **9 declared actions have zero producers.**

> **Documentation delta:** ARCHITECTURE.md:497 states "**10 audit actions**" and enumerates them without `MediaUploaded`. The enum has 11 (`action.rs:29`). The doc is stale by one variant, and its list is the set that is *declared*, not the set that is *emitted*.

Grep across the 12 assigned files: zero references to `buzz_audit`, `AuditAction`, or `state.audit_tx`. Confirmed.

##### 4.2 Which privileged mutations reach the hash chain, and how

| Mutation | Event stored? | Hash-chain entry? | Actor recorded |
|---|---|---|---|
| 9040 ban | no (`ingest.rs:1582-1586`) | **NO** | — |
| 9041 unban | no | **NO** | — |
| 9042 timeout | no | **NO** | — |
| 9043 untimeout | no | **NO** | — |
| 9044 resolve report | no | **NO** | — |
| 9030 add member | no (`ingest.rs:1811-1816`) | **NO** | — |
| 9031 remove member | no | **NO** | — |
| 9032 change role | no | **NO** | — |
| 9033 set workspace icon | no | **NO** | — |
| 1984 report | no (`ingest.rs:1563-1569`) | **NO** | — |
| 42000 product feedback | no (`ingest.rs:1567-1577`) | **NO** | — |
| 30350 push lease | yes (atomic with lease, `push_lease.rs:563`) | **NO** — ingest returns at `:2199` before the storage path that dispatches | — |
| **9035/9036 identity archive** | **yes** (falls through, `ingest.rs:1909-1912`) | **YES** — `EventCreated` | authenticated request signer |
| Moderation notice DM (kind:9) | yes (`moderation_notices.rs:174`) | **YES** — `EventCreated` | **relay pubkey** (`moderation_notices.rs:178`) |
| `"{host} Moderation"` kind:0 | yes, on first insert (`moderation_notices.rs:206`) | **YES** — `EventCreated`, only when `was_inserted` (`:207-211`) | relay pubkey |
| Workflow `send_message` (kind:9) | yes (`workflow_sink.rs:339`) | **YES** — `EventCreated` | **workflow owner**, not the relay key (`workflow_sink.rs:355`; rationale `handlers/event.rs:584-590`) |
| Operator community provisioning | n/a (HTTP) | **NO** | — |
| NIP-43 / NIP-IA announcement events emitted by `side_effects` | yes | yes, as `EventCreated` | per `side_effects` call |

**Net:** of the 14 privileged inbound kinds this module owns, **12 produce no hash-chain entry at all**. Bans, unbans, timeouts, role changes, member removals, ownership-affecting provisioning, and report resolutions are unauditable through `buzz-audit`. The only durable trail for moderation is the separate `moderation_actions` table (written by `moderation_commands.rs:529` only) — which is **not** hash-chained and therefore not tamper-evident. Relay-admin mutations (9030–9033) have **no** durable audit trail of any kind: no `moderation_actions` row, no event row, no hash-chain entry — only `tracing::info!` lines (`relay_admin.rs:164`, `:203-209`, `:268-272`, `:327-332`).

##### 4.3 Audit transport failure mode

`dispatch_persistent_event` sends into a bounded channel of capacity 1000 using `.send().await`, so backpressure propagates rather than silently dropping (`handlers/event.rs:548-576`). DB write failures inside the audit worker are logged but **not retried** (`:556-558`). A closed channel logs `Audit channel closed — entry lost` (`:575-577`). `state.audit_tx` being `None` skips auditing entirely (`:548-550`).

---

#### 5. `buzz-media` / S3

Three distinct integration points, all through `state.media_storage`:

| Consumer | Call | Purpose |
|---|---|---|
| `report.rs:66-71` | `get_sidecar(tenant, sha_hex)` | tenant-scoped blob existence check for `x`-tag report targets |
| `product_feedback.rs:35` | `verify_imeta_blobs(tenant, &imeta_tags, &state.media_storage)` | attachment verification before persisting feedback |
| `storage_sweep.rs` via `main.rs:1462-1470` | `list_page(token, 1000)` folded by `buzz_media::fold_bucket_listing` | whole-bucket inventory |

##### 5.1 Failure modes

**`report.rs`** — `map_err(|_| "invalid: report target blob not found")` (`:70`) collapses *every* failure class into "not found". Documented as a known Phase-1 limitation because the sidecar API exposes no typed not-found-vs-transient distinction (`:66-69`). A MinIO/S3 outage therefore tells reporters their blob does not exist.

**`product_feedback.rs`** — imeta errors propagate as `String` from `crate::api::validate_imeta_tags` / `verify_imeta_blobs` (`:34-35`) and reject the whole feedback submission.

**`storage_sweep`** — five distinct failure classes, all meaning "keep the old snapshot, never a partial one" (`buzz-media/src/bucket_index.rs:337-338`):

| `SweepError` | Cause | Site |
|---|---|---|
| `CapExceeded { seen, cap }` | cumulative object count exceeded `BUZZ_STORAGE_SWEEP_MAX_OBJECTS` | `bucket_index.rs:342`, raised `:394` |
| `Storage(MediaError)` | the S3/MinIO LIST call failed | `bucket_index.rs:345` |
| `Timeout(Duration)` | whole attempt exceeded `BUZZ_STORAGE_SWEEP_TIMEOUT_SECS`; **constructed by the relay**, not by the fold (`storage_sweep.rs:251`) | `bucket_index.rs:353` |
| `MalformedPage` | `is_truncated=true` with no continuation token — unresumable | `bucket_index.rs:357`, raised `:406` |
| task panic | `JoinError` | `storage_sweep.rs:194-202` |

All five increment `failures_total` and set `last_attempt.ok = false`; only `CapExceeded`/`Storage`/`Timeout`/`MalformedPage` log the operator-actionable hint "verify s3:ListBucket (or MinIO list) permission is granted on the bucket" (`storage_sweep.rs:176-181`).

##### 5.2 Credential coupling

The sweep uses the **same** `MediaStorage` instance as upload/download, therefore the same credentials: `BUZZ_S3_ACCESS_KEY` / `BUZZ_S3_SECRET_KEY` (`config.rs:622-625`). `list_page` is called with an empty prefix (`buzz-media/src/storage.rs:250`), i.e. whole-bucket listing across every tenant. There is no separate read-only or list-only credential and no per-tenant prefix restriction. Adding storage metrics therefore requires granting `s3:ListBucket` to the relay's read-write media key.

---

#### 6. APNs push gateway (outbound HTTPS)

##### 6.1 Client construction and destination

| Property | Value | Site |
|---|---|---|
| HTTP client | one `reqwest::Client` per worker, built once | `push_runtime.rs:313-316` |
| Timeout | `state.config.push_gateway_timeout` — applied as a whole-request `reqwest` timeout | `push_runtime.rs:314` |
| Timeout value | `BUZZ_PUSH_GATEWAY_TIMEOUT_MS`, default **2000 ms**, range-validated `100..=10000` | `config.rs:759-772` |
| Destination | `config.push_gateway_delivery_url` (`Option<url::Url>`) | `push_runtime.rs:422-424` |
| Default destination | `https://push.buzz.xyz/v1/deliveries/apns` | `config.rs:339`, `:755-758` |
| URL validation | scheme must be `https`, host required, no userinfo, path must be exactly `/v1/deliveries/apns`, no query, no fragment | `config.rs:341-361` |
| Disable | set `BUZZ_PUSH_GATEWAY_DELIVERY_URL` to an **empty** string | `config.rs:753` |
| Auth | NIP-98 kind-27235 event, base64'd into `Authorization: Nostr …`, with `u`/`method`/`payload`(sha256 of body)/`nonce` tags | `push_runtime.rs:551-565` |

**Failure mode: the client build `.expect("push HTTP client")` panics** (`push_runtime.rs:316`). This runs inside a `tokio::spawn`ed task, so a panic here silently kills the delivery worker while the matcher keeps enqueuing wakes — an unbounded outbox with no consumer, and no restart.

##### 6.2 Retry and invalidation semantics

| Condition | Action | Site |
|---|---|---|
| 2xx + `{"status":"accepted"}` | `complete_push_wake` | `:434-441` |
| 2xx + other/unparseable body | `fail_push_wake` (terminal) | `:442-447` |
| 410 + `invalid_endpoint{generation}` matching the wake | `disable_push_endpoint`, then `fail` | `:452-473` |
| 410 with mismatched generation | log only, then `fail` — **stale 410 cannot kill a rotated lease** | `:456-465` |
| 410 with unparseable body | `warn!("invalid closed-protocol 410 response")`, then `fail` | `:467` |
| 503 + `retry{retry_after_seconds>0}` | `retry_or_fail(that value)` | `:474-484` |
| 503 otherwise | `retry_or_fail(2)` | `:478-483` |
| 429 | `retry_or_fail(2)` — `Retry-After` header ignored | `:485-487` |
| 404 with `attempt > 1` | `complete_push_wake` — replay of a burned request id treated as delivered | `:488-497` |
| `is_timeout()` or `is_connect()` | `retry_or_fail(2)` | `:498` |
| anything else (incl. 4xx, TLS errors) | `fail_push_wake` (terminal) | `:499-503` |

Backoff: `delay * 2^(attempt-1)` clamped at `2^6 = 64×`, terminal at `MAX_ATTEMPTS = 8` (`push_runtime.rs:531-550`, const `:17`). Worst case with `delay=2`: 2, 4, 8, 16, 32, 64, 128, then fail — roughly 4 minutes.

**Every DB call in the response-handling path discards its result with `let _ =`** (`push_runtime.rs:436`, `:443`, `:455`, `:469`, `:492`, `:500`, `:534`, `:540`). A failed `complete_push_wake` therefore leaves the row claimed; recovery depends on the 30 s claim lease expiring and the wake being re-claimed — which is safe (idempotent via the request id) but invisible.

**SSRF assessment: not exposed.** The destination is operator config validated to a fixed path; the client-controlled `endpoint_grant` travels in the JSON body (`push_runtime.rs:507-515`), never in the URL. `reqwest` default redirect behaviour is not overridden, so a redirect from the configured gateway would be followed — a residual risk bounded by trusting the configured host.

##### 6.3 Counterpart crate

`crates/buzz-push-gateway/` implements the other side (its own `AppProfile` enum at `model.rs:14`, APNs client at `apns.rs`, grant model at `grant.rs`). It is not part of this module group; the wire contract between them is the `DeliveryRequest`/`DeliveryResponse` pair in `push_runtime.rs:31-51` and is not shared via a common crate — the two definitions are independent.

---

#### 7. `buzz-workflow`

`RelayActionSink` is the relay's implementation of `buzz_workflow::action_sink::ActionSink` (`workflow_sink.rs:172`). Wiring order matters and is documented: constructed after `AppState` (which owns `sub_registry` and `conn_manager`) and **before** the cron loop starts (`main.rs:591-597`).

Cycle avoidance: `AppState → WorkflowEngine → ActionSink → AppState` would be an `Arc` cycle, so the sink holds `Weak<AppState>` (`workflow_sink.rs:159-161`, rationale `:150-155`). Upgrade failure is mapped to `ActionSinkError::Database("relay is shutting down")` (`:186-188`).

Error surface: `ActionSinkError` has 6 variants (`buzz-workflow/src/action_sink.rs:11-29`) and **all of them collapse into a single `WorkflowError::WebhookError`** via `From` (`:31-34`). A channel-not-found, an archived channel, an access denial, and a genuine DB outage are therefore indistinguishable in workflow run output.

Failure modes reaching the workflow engine from this module:

| Cause | `ActionSinkError` | Site |
|---|---|---|
| shutting down | `Database` | `workflow_sink.rs:186-188` |
| community not mapped to a host | `Database` | `:205-210` |
| blank text | `EmptyContent` | `:212-214` |
| malformed channel UUID | `InvalidInput` | `:217-218` |
| channel missing | `ChannelNotFound` | `:225-230` |
| channel archived | `ChannelArchived` | `:232-236` |
| bad author pubkey hex | `InvalidInput` | `:238-240` |
| owner not a member of a non-open channel | `InvalidInput` | `:247-251` |
| any DB failure (5 calls) | `Database` | `:203`, `:229`, `:246`, `:277`, `:283`, `:342` |
| tag parse / signing failure | `EventBuild` | `:260-267`, `:292-294`, `:305` |

`buzz-workflow` also reaches the relay **outside** this sink for `add_reaction`, via its own HTTP client to `{BUZZ_RELAY_BASE_URL}/api/messages/{id}/reactions` (`buzz-workflow/src/executor.rs:885-919`). That route is not registered in `router.rs`, so this integration is permanently broken — see the features aspect.

---

#### 8. `buzz-search`

Reached only transitively through `dispatch_persistent_event` for the two relay-signed kind:9 paths (`moderation_notices.rs:178`, `workflow_sink.rs:351`). Indexing failures are absorbed inside `dispatch_persistent_event` and never surface here; `workflow_sink` additionally discards the whole result.

Consequence: a moderation notice DM is indexed into full-text search like any other message, so a moderator's `public_reason` becomes searchable by the recipient.

---

#### 9. `buzz-sdk` (NIP-OA)

Single integration point: `buzz_sdk::nip_oa::verify_auth_tag(auth_tag_json, &target_pubkey)` (`identity_archive.rs:320-326`), called twice per owner-consent request — once for the request's own `auth` tag (`:261`) and once for the target's live kind:0 `auth` tag (`:291`).

Failure mode: any verification error becomes a client-visible string via `e.to_string()` (`:324`), prefixed as `invalid request auth tag: {e}` (`:262`) or `invalid live kind:0 auth tag: {e}` (`:292`) — SDK-internal error text is exposed to the client.

Test helpers additionally use `nip_oa::compute_auth_tag` and `parse_auth_tag` (`identity_archive.rs:479-481`).

---

#### 10. Prometheus / `metrics`

| Metric | Type | Emitter | Labels |
|---|---|---|---|
| `buzz_channels_created_total` | counter | `moderation_notices.rs:113-118` | `community` (host), `type="dm"` |
| `buzz_storage_sweep_ok` | gauge | `storage_sweep.rs:293` | — |
| `buzz_storage_sweep_failures` | gauge (deliberately not `_total` — process-local, resets on failover, `:294-297`) | `storage_sweep.rs:298` | — |
| `buzz_storage_sweep_duration_seconds` | gauge | `:300` | — |
| `buzz_storage_sweep_age_seconds` | gauge | `:311-312` | — |
| `buzz_total_storage_bytes` / `_objects` | gauge | `:315-322` | `kind` ∈ {`physical`,`logical`} |
| `buzz_storage_orphan_blob_bytes`, `_orphan_blobs`, `_orphan_sidecars`, `_multi_variant_shas`, `_multi_variant_bytes`, `_unknown_key_bytes`, `_unknown_key_objects` | gauge | `:324-330` | — |
| `buzz_community_storage_bytes` / `_objects` | gauge | `:339-342`, zeroed at `:125-134` | `community` (host label) |
| `buzz_storage_unmapped_community_bytes` | gauge | `:347` | — |

`storage_sweep` emits **no** counters and no histograms — even sweep duration is a gauge, so percentile analysis across sweeps is not possible.

`push_runtime` emits **zero** metrics: no wake-enqueued counter, no delivery-outcome counter, no gateway-latency histogram. Delivery health is observable only through `warn!`/`error!` log lines.

---

#### 11. Integration risk summary

| Risk | Integration | Evidence |
|---|---|---|
| Delivery worker dies permanently on HTTP client build failure | reqwest | `push_runtime.rs:316` `.expect` inside a `tokio::spawn` |
| Every push DB result discarded (`let _ =`) — silent state divergence | `buzz-db` | 8 sites, `push_runtime.rs:436`…`:540` |
| Push lease DB error is undiagnosable | `buzz-db` | `push_lease.rs:572` `map_err(\|_\| …)` |
| 12 of 14 privileged kinds produce no hash-chain entry | `buzz-audit` | §4.2 |
| Relay-admin mutations have no durable audit trail at all | `buzz-audit` + `buzz-db` | `relay_admin.rs` writes only `tracing::info!` |
| Ban disconnect fan-out is fire-and-forget; reads stay open on other pods | `buzz-pubsub` | `state.rs:1043-1047` |
| S3 outage is reported to reporters as "blob not found" | `buzz-media` | `report.rs:66-70` |
| Storage sweep needs `s3:ListBucket` on the read-write media credential, whole-bucket | `buzz-media` | `config.rs:622-625`, `buzz-media/src/storage.rs:250` |
| All 6 `ActionSinkError` variants collapse to one `WorkflowError` | `buzz-workflow` | `action_sink.rs:31-34` |
| Workflow message fan-out failure invisible to the run | `buzz-pubsub`/`buzz-search` | `workflow_sink.rs:351` `let _ =` |
| Push wire contract duplicated, not shared | `buzz-push-gateway` | `push_runtime.rs:31-51` vs `buzz-push-gateway/src/model.rs` |
| Zero observability on push delivery | `metrics` | no `metrics::` calls in `push_runtime.rs` |


## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Integrations

---

#### 1. Dependency map

| Integration | Reached via | Call sites in this group | Failure mode |
|---|---|---|---|
| `buzz-relay-mesh` (iroh/QUIC) | `MeshHandle.transport: Arc<dyn RelayPeerTransport>`, `MeshRuntime`, `MeshEndpoint` | `mesh_boot.rs:423`, `:475`, `:502`, `:505`; `join.rs:1675`, `:1762`; `mesh.rs:277`; `reliable.rs:120` | §2 |
| `buzz-pubsub` / Redis pub-sub | `state.pubsub.publish_event` | `handler.rs:1322-1331` | §3.1 |
| Redis (deadpool) — session directory | `SessionDirectory` over `deadpool_redis::Pool` | `directory.rs:214`, `:249`, `:283`, `:313`, `:331`, `:359` | §3.2 |
| Redis — mesh ready registry | `ReadyRegistry` | `mesh_boot.rs:445`, `:461`, `:468` | §3.3 |
| `buzz-db` (Postgres) | `state.db.*` | `handler.rs:158`, `:389`, `:838`, `:1164`, `:1179`, `:1197`, `:1212`, `:1219`, `:1281` | §4 |
| `buzz-auth` | `generate_challenge`, `state.auth.verify_auth_event` | `handler.rs:176`, `:222` | §5 |
| `buzz-core` | `CommunityId`, `TenantContext`, `StoredEvent` | throughout | compile-time only |
| `buzz-conformance` | `Tracer`, `TraceStep`, `TraceAction`, `AbstractState`, … | `conformance/mod.rs:38-44`; `tracers.rs:12` | §6 |
| `nostr` crate | `EventBuilder`, `Kind`, `Tag`, `Keys` | `handler.rs:1240-1272`; `mesh_boot.rs:415`, `:452` | signing failure → `warn!` + skip (`handler.rs:1270-1273`) |
| `postcard` | `HuddleControlMsg` codec | `join.rs:1007`, `:1012` | `MeshError::Encode`/`Decode` |
| `dashmap` | rooms, peers, generation floor, owner registry | `room.rs:490`, `:163`; `mesh.rs:90`; `join.rs:601` | none (in-process) |
| `metrics` | one counter | `directory.rs:483` | none |
| `mesh-llm-sdk` / `mesh-llm-host-runtime` | **dev-dependencies only** | `Cargo.toml:87-88`; consumed by `examples/mesh_agent_e2e.rs:25,48`, `examples/mesh_serve_client_smoke.rs:29,44-45,53`, `examples/mesh_serve_smoke.rs:13`, `examples/mesh_stack_smoke.rs:53` | §7 |

**No audio codec or media library is linked.** `crates/buzz-relay/Cargo.toml` has no
`opus`, `webrtc`, `cpal`, `dasp`, `symphonia`, `stun`, or `turn` entry — verified by
grep. The relay treats every audio byte as opaque.

---

#### 2. `buzz-relay-mesh` — the deepest coupling

##### 2.1 Surfaces consumed

| Type / fn | Where used |
|---|---|
| `RelayPeerTransport::{send_datagram, open_session_stream, set_inbound}` | `join.rs:1675`, `:1762`; `mesh.rs:277`; `reliable.rs:120`; `mesh_boot.rs:505` |
| `InboundHandler::{on_datagram, on_session_stream}` | implemented by `MeshInboundDispatcher`, `mesh_boot.rs:91-130` |
| `MeshStream::{send_frame, recv_frame, finish}` | `join.rs:1010`, `:1179`, `:1783`; `reliable.rs:283`, `:317`, `:334` |
| `MeshStreamFrame` (4 variants) | `join.rs`, `reliable.rs`, `mesh_boot.rs` |
| `FencedHeader`, `MeshDatagram`, `Profile`, `GoodbyeReason`, `RuntimeId`, `StreamHello`, `StreamRole`, `MeshError` | throughout |
| `MeshEndpoint::bind`, `endpoint.ip_addrs()`, `endpoint.runtime_id()` | `mesh_boot.rs:423`, `:392`, `:432` |
| `MeshRuntime::{start, reconcile_now, membership, clone}` | `mesh_boot.rs:475`, `:478`, `:173`, `:501-502` |
| `MeshMembership::{new, with_expected_relay_pubkey}`, `RelayMeshMembership`, `MeshStatus` | `mesh_boot.rs:441-443`, `:501`, `:172-174` |
| `ReadyRegistry`, `ReadyRecord`, `GossipRecord`, `spawn_registry_heartbeat` | `mesh_boot.rs:445-472` |
| `WIRE_VERSION`, `wire::MAX_STREAM_FRAME` | `mesh_boot.rs:367`; `reliable.rs:945` |

`MeshHandle` is the sole gateway: `AppState::mesh()` returns `Option<&MeshHandle>`
(`state.rs:812-814`), and every consumer branches on that `Option`
(`handler.rs:306`, `:449`, `:577`, `:875`).

##### 2.2 Trust boundary between the two crates

- Inbound mesh connections are gated on `is_known_peer`, which requires a
  Redis ready-registry record (`buzz-relay-mesh/src/runtime.rs:275-283`, `:309-320`).
  Records are attested against the relay signing key
  (`MeshMembership::with_expected_relay_pubkey`, `mesh_boot.rs:442-443`).
- The `from: RuntimeId` handed to every handler is the **authenticated QUIC peer
  identity** (`runtime.rs:392-399`, `:412`), which is what lets
  `accept_inbound` assert `hello.sender == from` (`join.rs:1060-1065`,
  `reliable.rs:143-148`).
- The mesh layer itself does **no** Redis fencing. Every fence check in this group
  is performed by the *consumer*: `join.rs:1231-1245` (control frames),
  `reliable.rs:381-385` (reliable frames), `mesh.rs:212-220` (media, local floor
  only — see security).

##### 2.3 Failure modes

| Failure | Behaviour |
|---|---|
| `MeshEndpoint::bind` fails with mesh on | **Fatal at boot** — `anyhow` error propagated from `mesh_boot.rs:423-431` to `main.rs:442` |
| Peer unreachable when dialing an owner | `DialError::Mesh` → WS `huddle_owner_unreachable`; the joining client gets a clean error, and `cleanup_if_empty` runs (`handler.rs:487-503`) |
| `send_datagram` fails (disconnected peer, oversize) | `debug!` and continue — audio drop-on-error (`join.rs:1762-1765`, `mesh.rs:277-282`) |
| `send_frame` on the control stream fails | breaks the serve loop with `Err`, then unconditional peer teardown (`join.rs:1245-1254`, `:1345-1367`) |
| Owner pod dies mid-call | ingress reader sees a bare close → `StreamClosed` → cancel + `fence.forget` (`join.rs:1604-1610`, `handler.rs:707-714`) |
| Owner drains (SIGTERM) | `Goodbye(Draining)` → ingress rejoins; local owner clients closed by the drain watcher (`join.rs:1157-1161`, `handler.rs:735-748`) |
| Traffic arrives before a profile handler is registered | logged and dropped; the peer's fenced retry is safe (`mesh_boot.rs:52-55`, `:92-100`, `:122-129`) |
| `RealtimeMedia` arrives as a stream | rejected without routing (`mesh_boot.rs:113-121`) |
| MTU overflow on a media datagram | the sink drops it with a `debug!`; the comment explicitly says MTU prevention "is the ship-gate's job" (`mesh.rs:278-281`) — i.e. **no runtime MTU check exists** |

---

#### 3. Redis — three independent uses

##### 3.1 `buzz-pubsub` (lifecycle-event fan-out)

Single call: `state.pubsub.publish_event(tenant, EventTopic::Channel(parent), &event)`
(`handler.rs:1322-1325`). Topic is the **lifecycle parent channel**, not the
ephemeral huddle channel.

Failure: the event is already persisted and already fanned out locally, so a publish
error only means other pods miss the live delivery. `local_event_ids` is invalidated
so a later echo is not suppressed (`handler.rs:1326-1330`), then `warn!`.

##### 3.2 Session directory (ownership arbiter)

`deadpool_redis::Pool`, shared with the rest of the relay
(`state.redis_pool.clone()` → `mesh_boot.rs:442`, `:512`). Four Lua scripts
(`directory.rs:20-79`) + two plain `GET`s (`directory.rs:313`, `:331`).

| Failure | Behaviour |
|---|---|
| pool checkout fails | `DirectoryError::Pool` → `MeshError::Transport` at every `HuddleDirectory` boundary (`join.rs:114`, `:139`, `:158`, `:172`) |
| Redis unreachable during `resolve_join` | join fails → WS `join_rejected` (`handler.rs:342-355`). **Huddles become unjoinable when Redis is down and mesh is on** |
| Redis unreachable during renewal | `Err` → treated as **owner loss**, `lost` fires, every local owner client is closed for rejoin (`join.rs:521-529`, `handler.rs:756-765`). A Redis blip therefore drops every cross-pod huddle on the pod |
| Redis unreachable during owner-side `validate` | non-fence error → the whole `HuddleControl` stream is torn down (`join.rs:1240-1244`) |
| Malformed lease value in Redis | `MalformedLease` → `Transport` error → join failure; never a silent default (`directory.rs:495-531`) |
| Lease key expires while the pod still serves peers | Redis stops naming the pod; the next renew returns `Lost`. Between expiry and the next renew tick (up to 10 s) the pod keeps fanning out with a dead lease — local WS peers have **no per-frame fence** |

##### 3.3 Mesh ready registry

`ReadyRegistry::new(redis_pool, config.mesh.registry_refresh)` (`mesh_boot.rs:445`),
first `publish_ready` at `:461`, then `spawn_registry_heartbeat` gated on
`!shutting_down` (`mesh_boot.rs:466-472`).

Failure: the **first** publish failing is fatal to boot (`mesh_boot.rs:459-463`) —
"if Redis can't take the attested record, peers can never find us". Later heartbeat
failures are internal to `buzz-relay-mesh`.

---

#### 4. `buzz-db` (Postgres)

| Call | Line | Purpose | Failure mode |
|---|---|---|---|
| `is_community_active(community)` | `handler.rs:158` | community lifecycle gate | closure result drives `run_registered_community_connection`; a DB error there rejects the connection |
| `get_channel(community, channel)` | `handler.rs:1164` (in `ensure_membership`) | load channel, reject archived | `Err` → `"db error: {e}"` → WS `not a member` |
| `get_channel(community, channel)` | `handler.rs:389` | post-`get_or_create` archived re-check | `Err` → **fail closed**, silent teardown (`handler.rs:404-410`) |
| `huddle_started_link_exists(community, parent, channel, created_by)` | `handler.rs:1179-1186` | verify a creator-signed kind-48100 link before trusting a client-supplied parent | `Err` → `"db error"`; `false` → `"ephemeral channel is not linked to claimed parent"` |
| `is_member_cached(community, channel, pubkey)` ×2 | `handler.rs:1197`, `:1212` | membership fast path + parent check | `Err` → `"db error"` |
| `add_member(community, channel, pubkey, Member, Some(created_by))` | `handler.rs:1219-1227` | ephemeral auto-add | `Err` → `"auto-add failed: {e}"` → join refused |
| `invalidate_membership(tenant, channel, pubkey)` | `handler.rs:1228` | cache coherence after auto-add | infallible |
| `archive_channel(community, channel)` | `handler.rs:838` | auto-end | `Err` → `clear_ended()`, huddle stays alive, no 48103 (`handler.rs:840-845`) |
| `insert_event(community, &event, Some(parent))` | `handler.rs:1281-1284` | persist lifecycle event | duplicate → skip fan-out; `Err` → fan out from memory anyway (`handler.rs:1285-1307`) |

Note the **double `get_channel`** on every join (`:1164` and `:389`) — two round
trips for the same row, deliberate to close a race but uncached.

---

#### 5. `buzz-auth`

- `generate_challenge()` (`handler.rs:33`, `:176`) — the challenge nonce.
- `state.auth.verify_auth_event(event, &challenge, &relay_url)` (`handler.rs:220-238`)
  — full NIP-42 verification, identical to the main relay door. The returned
  `auth_ctx.pubkey` is the only identity used downstream (`handler.rs:240-242`).
- `crate::handlers::auth::extract_auth_tag_json(&event)` (`handler.rs:217`) —
  NIP-OA tag pulled out *before* the event is consumed by the verifier.
- `crate::api::bridge::nip42_expected_relay_url(&state.config.relay_url, &tenant)`
  (`handler.rs:219`) — per-tenant expected relay URL.
- `crate::api::relay_members::enforce_relay_membership` (`handler.rs:244-262`) —
  no-op unless `require_relay_membership` is on (`api/mod.rs:67`, `:130-131`).

Failure: any verifier rejection → WS `{"type":"error","message":"auth failed"}` and
close. No retry, no second challenge.

---

#### 6. `buzz-conformance`

Consumed as a pure schema + trait crate. `conformance/mod.rs:38-44` re-exports 11
items; `tracers.rs:12` imports `TraceStep` and `Tracer`.

| Direction | Detail |
|---|---|
| relay → crate | `AppState.tracer: Arc<dyn buzz_conformance::Tracer>` (`state.rs:620`); every `record` call |
| crate → relay | nothing — the checker (`buzz-conformance/src/checker.rs`) consumes JSONL offline |

Failure modes: **none can reach a request path.** `Tracer::record` returns `()`
(`buzz-conformance/src/lib.rs:317`), so an emit cannot fail, cannot block, and
cannot apply backpressure. `JsonlTracer` swallows every I/O error
(`tracers.rs:63-71`) and recovers a poisoned mutex via `into_inner`
(`tracers.rs:57-60`). Because production binds `NoopTracer` (`state.rs:798`), the
integration is inert at runtime — see features §4.1.

One duplication: `buzz_conformance::NoopTracer` (`buzz-conformance/src/lib.rs:323`)
exists and has **zero users**; the relay defines and uses its own
(`tracers.rs:20-24`).

---

#### 7. `mesh-llm` — a name collision, not a mesh integration

`crates/buzz-relay/Cargo.toml:87-88` pins two **dev-dependencies** to
`git = "https://github.com/Mesh-LLM/mesh-llm.git", tag = "v0.73.1"`:
`mesh-llm-sdk` (features `client`, `serving`) and `mesh-llm-host-runtime`
(feature `dynamic-native-runtime`).

Consumers are exactly the five files in `crates/buzz-relay/examples/`:

| Example | Uses |
|---|---|
| `mesh_agent_e2e.rs` | `mesh_llm_sdk::{serve, MeshDiscoveryMode}` (`:25`), `mesh_llm_host_runtime::initialize_host_runtime` (`:48`) |
| `mesh_serve_client_smoke.rs` | `mesh_llm_sdk::{client, serve, MeshDiscoveryMode}` (`:29`), `native_runtime_cache` / `CURRENT_MESH_VERSION` (`:44-45`), `initialize_host_runtime` (`:53`) |
| `mesh_serve_smoke.rs` | `mesh_llm_sdk::{serve, MeshDiscoveryMode}` (`:13`) |
| `mesh_stack_smoke.rs` | `mesh_llm_host_runtime::models::download_model_ref_with_progress_details` (`:53`) |
| `mesh_admission_smoke.rs` | process-global mesh-llm state note (`:16`); no direct import |

This is **local LLM inference / model serving**, entirely unrelated to
`buzz-relay-mesh` (the inter-relay QUIC mesh in this group). Two different things
called "mesh" inside one crate, both spelled `mesh_*` in file names. Risk profile:

- A `git`-pinned dependency by **tag** (not commit SHA) — tags are mutable, so the
  build is not reproducible against a retagged upstream.
- Present in `[dev-dependencies]`, so it does not ship in the relay binary, but it
  **does** enter the dev/CI dependency graph and lockfile for anyone running
  `cargo test -p buzz-relay`.
- `mesh_stack_smoke.rs:31` requires manual sync with
  `buzz_lib::mesh_llm::MESH_WORKER_STACK_SIZE` in the desktop crate — a
  cross-crate constant duplicated by comment, not by code.

---

#### 8. Inbound/outbound integration matrix for one cross-pod huddle frame

```
Client A (pod 1, non-owner)                        Client B (pod 2, OWNER)
   │ WS binary [v2 hdr][Opus]                          
   ├──────────────────────────────► handler.rs:1019 forward_media
   │                                 join.rs:1758 media_datagram
   │                                 [ownerIdx][v2 hdr][Opus] + FencedHeader
   │                                 transport.send_datagram ──► iroh/QUIC
   │                                                              │
   │                        mesh_boot.rs:242 dispatcher.on_datagram
   │                        mesh.rs:212 GenerationFloor::check   ◄─┘
   │                        mesh.rs:221 get_unambiguous_by_channel
   │                        mesh.rs:247 room.deliver_prefixed
   │                                     └─► B's audio_tx ─► WS binary
   │
   │  B speaks: room.broadcast_frame (room.rs:398) puts the prefixed frame
   │  into A's *remote* AudioPeer.audio_tx, drained by
   │  spawn_remote_peer_sink (mesh.rs:262) ─► datagram ─► pod 1
   │  ─► mesh.rs:247 deliver_prefixed ─► A's WS
```

Redis is consulted **only** at join (`resolve_join_owner_ready` → `owner_of` /
`acquire` / `validate`), at owner-side `RegisterPeer` (`validate`), and every 10 s
by the renewer. It is **never** consulted per media frame.


## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Integrations

Five external couplings: **iroh/QUIC**, **Redis**, **nostr/secp256k1**,
**postcard**, and **`buzz-relay`** (the only consumer). Notably **`buzz-core` is
NOT a dependency** — see §4.

Full dependency list (`Cargo.toml:11-26`): `tokio`, `serde`, `serde_json`,
`postcard`, `iroh`, `redis`, `deadpool-redis`, `thiserror`, `tracing`, `uuid`,
`hmac`, `sha2`, `hex`, `nostr`, `bytes`, `futures-util`.
Dev-deps (`:28-30`): `tokio` (test-util), `proptest`.

Verified usage census (`use`-site count per crate):

| Dep | `use` sites | Status |
|---|---|---|
| `tokio` | 36 | used |
| `tracing` | 28 | used |
| `iroh` | 12 | used |
| `nostr` | 11 | used |
| `uuid` | 7 | used |
| `postcard` | 6 | used |
| `redis` / `deadpool-redis` | 5 / 5 | used |
| `serde` / `serde_json` | 4 / 4 | used (`serde_json` only in `registry.rs`) |
| `sha2`, `hex`, `bytes`, `thiserror` | 1 each | used |
| **`hmac`** | **0** | **unused dependency** (`Cargo.toml:20`) |
| **`futures-util`** | **0** | **unused dependency** (`Cargo.toml:25`) |
| **`proptest`** | **0** | **unused dev-dependency** (`Cargo.toml:29`) |

---

#### 1. iroh / QUIC

##### Version — delta against the brief

The workspace **requirement** is `iroh = { version = "1.0.0-rc.0",
default-features = false, features = ["tls-ring"] }` (`Cargo.toml:68`, comment
`:67` "Inter-relay mesh transport (buzz-relay-mesh)"). The crate takes it via
`workspace = true` (`crates/buzz-relay-mesh/Cargo.toml:15`).

**`Cargo.lock` resolves iroh to `1.0.2` from crates.io**
(`Cargo.lock:3902-3905`, checksum `5fca9b4b462c…`), not to the rc. Because
`^1.0.0-rc.0` admits `1.0.2`, the pre-release string in the manifest is a
*floor*, not a pin: the built artifact uses a stable 1.0.x release. The manifest
string is nonetheless misleading and should be `"1.0"` — see `-debt.md` D-05.

`iroh` is the **only** crate in the workspace using it (grep: no other
`Cargo.toml` references it).

##### Surface consumed

| iroh item | Used at |
|---|---|
| `Endpoint`, `Endpoint::builder`, `.bind()` | `endpoint.rs:3`, `:33-41` |
| `iroh::endpoint::presets::Minimal` | `endpoint.rs:33` |
| `SecretKey::generate()` / `SecretKey::from_bytes` | `endpoint.rs:20`; tests `endpoint.rs:158` |
| `PublicKey::as_bytes` / `from_bytes` | `endpoint.rs:97`, `:101` |
| `RelayMode::Disabled` | `endpoint.rs:36` |
| `.alpns(vec![ALPN.to_vec()])` | `endpoint.rs:35` |
| `.bind_addr(SocketAddr)` | `endpoint.rs:37` |
| `EndpointAddr`, `EndpointAddr::from_parts`, `TransportAddr::Ip` | `endpoint.rs:3`, `:65-68`, `:105-108` |
| `endpoint.accept()` → `Incoming` → `Connection` | `endpoint.rs:74-81` |
| `endpoint.connect(addr, ALPN)` | `endpoint.rs:88` |
| `Connection::alpn / remote_id / max_datagram_size / open_bi / accept_bi / send_datagram / read_datagram` | `peer.rs:50`, `:58`, `:69`, `:80`, `:93`, `:112`, `:120` |
| `endpoint::SendStream::write_all / finish`, `RecvStream::read_exact` | `peer.rs:148-165`, `:171-188` |
| `endpoint::ReadExactError::FinishedEarly` | `peer.rs:172` |

##### Configuration choices with consequences

- **`RelayMode::Disabled`** (`endpoint.rs:36`): no iroh relay servers, no hole
  punching via relays, no DERP fallback. Peers must be **directly IP-reachable** —
  which is why the deployment story is pod-to-pod inside one cluster
  (`advertise_addrs` prefers `POD_IP`, `mesh_boot.rs:398-403`).
- **`presets::Minimal`** (`endpoint.rs:33`): the leanest iroh endpoint preset — no
  discovery services.
- **`tls-ring`** (`Cargo.toml:68`) with `default-features = false`: ring rather than
  aws-lc-rs; consistent with the rest of the workspace's `tls-ring` choices
  (`Cargo.toml:57` redis, `:79` otlp).
- **Identity = iroh node key.** `RuntimeId` *is* `PublicKey::as_bytes()`
  (`endpoint.rs:96-98`), so iroh's TLS peer authentication and the mesh's identity
  model are the same thing. `MeshPeer::from_connection` derives the remote's
  RuntimeId from `conn.remote_id()` (`peer.rs:58`) — an attacker cannot claim a
  RuntimeId they do not hold the key for.

##### Failure modes

| Failure | Behaviour | Site |
|---|---|---|
| bind fails (port in use, bad addr) | `MeshError::Transport` → **fatal relay boot** | `endpoint.rs:38-41`; `mesh_boot.rs:383-390`; `main.rs:442` `?` |
| inbound handshake fails | warn `"mesh: inbound connection failed"`, accept loop continues | `runtime.rs:277-279` |
| `endpoint.accept()` returns `None` (endpoint closed) | accept loop logs and **returns** — no restart, mesh is permanently deaf | `runtime.rs:271-274` |
| dial fails for an addr | warn, try next addr; all exhausted → mark Disconnected, retry in 5 s **with no backoff** | `runtime.rs:340-354` |
| ALPN mismatch on an established conn | `MeshError::Transport("unexpected mesh ALPN …")`, peer not installed | `peer.rs:50-55` |
| peer lacks datagram support | `Transport("peer does not support QUIC datagrams")` on every send | `peer.rs:109` |
| datagram or stream read error | `remove_peer` → tasks aborted, `ConnectionState::Disconnected`; membership entry **kept** | `runtime.rs:359-363`, `:379-383`, `:267-281` |
| oversize frame/datagram | typed `FrameTooLarge` / `DatagramTooLarge`, never truncated | `peer.rs:142-147`, `:178-183`, `lib.rs:218-223` |
| iroh error detail | **flattened to `String`** via `err.to_string()` at 12 sites — the structured iroh error type is discarded, so callers cannot match on cause | `endpoint.rs:38,39,79,91,101`; `peer.rs:82,95,113,123,151,155,163,174,187` |

`iroh` being a pre-1.0-string dependency with a 1.0.x lock resolution means an
`iroh` 1.1 release will be picked up silently by `cargo update`; the API surface
consumed here (`presets::Minimal`, `EndpointAddr`, `TransportAddr`,
`remote_id()`) is broad enough that this is a real upgrade-risk area.

---

#### 2. Redis (ready registry)

Clients: `redis` 1.0 (`Cargo.toml:57`, features `tokio-comp`,
`connection-manager`, `tokio-rustls-comp`) + `deadpool-redis` 0.23 (`:58`).
The crate never creates a pool — it receives one
(`ReadyRegistry::new(pool, refresh)`, `registry.rs:166-168`), and the relay passes
`state.redis_pool.clone()` (`mesh_boot.rs:447`, from `main.rs:443`). Same pool the
rest of the relay uses (`buzz-pubsub`, session directory), so mesh registry traffic
competes for the same connections.

| Operation | Command | Site | Cadence |
|---|---|---|---|
| publish ready | `SET mesh:ready:{id} <json> EX <ttl>` | `registry.rs:188-194` | once at boot (`mesh_boot.rs:459`) then every 15 s (`runtime.rs:602`) |
| clear ready | `DEL mesh:ready:{id}` | `registry.rs:201-204` | on ready→not-ready edge only (`registry.rs:299-302`) |
| scan | `SCAN <cur> MATCH mesh:ready:* COUNT 100` + `GET` per key | `registry.rs:217-228` | every 5 s (`runtime.rs:311`) **plus once per unknown inbound connection** (`runtime.rs:318`) |

Value codec is **JSON** (`serde_json::to_string`, `registry.rs:185` /
`from_str`, `registry.rs:232`) — the only non-postcard payload in the crate,
chosen for operator legibility.

Key namespace `mesh:ready:` (`registry.rs:19`) is **global**: not community-scoped,
not deployment-prefixed. Multi-deployment isolation rests entirely on the
`relay_pubkey` anchor check in `apply_ready_records` (`membership.rs:90-102`),
which counts rejects into `foreign_relay_rejections` (`status.rs:47`).

##### Failure modes

| Failure | Behaviour | Site |
|---|---|---|
| pool `get()` fails | `MeshError::Transport("redis pool: …")` | `registry.rs:270-272` |
| **first publish fails** | **fatal relay boot** — `anyhow!("mesh ready-registry publish failed: …")` | `mesh_boot.rs:456-463` |
| heartbeat publish fails | warn `"mesh: registry heartbeat tick failed"`, loop continues; the record then TTL-expires after 45 s and peers stop dialing us | `runtime.rs:601-603` |
| reconcile scan fails | warn `"mesh: registry scan failed"`, then dial from the (stale, never-evicted) membership table | `runtime.rs:288-292`, `:307-312` |
| inbound-admission scan fails | warn `"mesh: registry rescan on inbound failed"`, fall through to `has_peer` re-check → connection rejected | `runtime.rs:315-320` |
| malformed / key-mismatched / unattested entry | warn + skip, scan continues | `registry.rs:233-247` |
| Redis transport error mid-scan | whole scan aborts with `MeshError::Redis` (`#[from]`) | `registry.rs:225`, `:228` |
| clear fails on shutdown | error propagates from `tick`, logged as a heartbeat warn; record lingers up to TTL | `registry.rs:300`, `runtime.rs:601` |

Redis outage semantics: existing warm connections keep working (transport is
independent of Redis), but (a) no new peers can be discovered, (b) our record
TTL-expires so *new* pods can't find us, and (c) every unknown inbound connection
costs a failed scan. Nothing in this crate degrades gracefully to a static peer
list.

##### CPU/IO amplification

`scan_ready` performs one `GET` **per key** in a serial loop (`registry.rs:230-231`)
— no `MGET`, no pipelining — and one **secp256k1 schnorr verify per record**
(`registry.rs:233-238` → `registry.rs:70-80`). At 5 s cadence with N pods that is
N verifies per pod per 5 s, plus a full extra scan+verify pass for **every inbound
connection from an unrecognised runtime id** (`runtime.rs:309-320`). See
`-security.md` S-04.

---

#### 3. nostr / secp256k1 (attestation)

`nostr` 0.44 (`Cargo.toml:61`, features `nip44`, `nip98`) is used only in
`registry.rs`:

| Item | Site |
|---|---|
| `nostr::Keys::sign_schnorr` | `registry.rs:41` |
| `nostr::PublicKey::from_hex` / `.to_hex()` / `.xonly()` | `registry.rs:57`, `:39`, `:63` |
| `nostr::secp256k1::{Message, XOnlyPublicKey}`, `schnorr::Signature::from_str` | `registry.rs:11-12`, `:68` |
| `nostr::secp256k1::SECP256K1.verify_schnorr` | `registry.rs:81-82` |
| `sha2::Sha256::digest` over the textual preimage | `registry.rs:94` |

The signing key is the relay's Nostr identity, injected from the consumer:
`boot_mesh(..., relay_keypair: &nostr::Keys, ...)` (`mesh_boot.rs:415`) sourced from
`state.relay_keypair` (`main.rs:445`). The same key anchors acceptance
(`mesh_boot.rs:445` → `membership.rs:61-64`).

Failure modes: every parse/convert/verify error becomes a
`MeshError::Transport(format!(...))` string (`registry.rs:56-83`) — five distinct
failure causes collapse into one untyped variant, so a caller cannot distinguish
"bad hex" from "signature forged." Rejections are logged with `runtime_id`
(`membership.rs:96-101`, `:105-108`; `registry.rs:236-246`) and counted only for the
anchor-mismatch case (`foreign_relay_rejections`), not for signature failures.

---

#### 4. `buzz-core` — **not a dependency**

Verified: `crates/buzz-relay-mesh/Cargo.toml` has no `buzz-*` dependency at all, and
no `buzz_core` import exists in `src/`. The mesh crate is deliberately
Buzz-domain-free: no `CommunityId`, no event kinds, no NIP types, no tenant concept.

The tenant boundary is applied **outside** this crate — consumers thread
`buzz_core::CommunityId` through their own layers (`tunnel/reliable.rs:13`,
`audio/join.rs:41`, `api/mesh_demo.rs:29`) and the fenced session directory scopes
by community. Consequence: **the mesh wire format carries no tenant identifier.**
`FencedHeader` is `{session_id, generation, owner_runtime_id}` (`wire.rs:85-93`) —
community scoping is entirely inside the opaque `payload`, recovered by
`recv_validated`/`community_id()` on the relay side
(`mesh_boot.rs:334-341`, `tunnel/reliable.rs`). Cross-tenant isolation on the mesh
therefore depends on consumer discipline, not on the transport contract.

---

#### 5. postcard

`postcard` 1 with `default-features = false, features = ["use-std"]`
(`Cargo.toml:65`). Used at 6 sites: `wire.rs:178` (`to_extend`), `wire.rs:184`
(`from_bytes`), `gossip.rs:63`, `gossip.rs:67`, and the two error `#[source]`
bindings (`lib.rs:68`, `:70`).

Integration risk: postcard's enum encoding is a varint discriminant with no
unknown-variant escape hatch, and **no wire enum is `#[non_exhaustive]`**
(verified: zero occurrences in `src/`). Combined with the ALPN-per-version rule
(`wire.rs:34-36`) the design is "never mix versions," which is sound but leaves the
`WIRE_VERSION` byte and `GOSSIP_PAYLOAD_VERSION` field as belt-and-braces only.
The gossip payload is doubly-encoded (postcard `GossipMessage` inside a postcard
`MeshStreamFrame::Gossip.payload: Vec<u8>`), which costs a length prefix + a copy
per gossip frame but buys independent evolution (`wire.rs:139-141`).

---

#### 6. `buzz-relay` — how the consumer wires it

Declared at `crates/buzz-relay/Cargo.toml:26` (`buzz-relay-mesh = { workspace = true }`),
path-mapped at root `Cargo.toml:135`, member at `Cargo.toml:27`.

Boot sequence (`crates/buzz-relay/src/mesh_boot.rs:412-521`):

1. `config.mesh.enabled` false → `Ok(None)` (`:417-419`).
2. `MeshEndpoint::bind(config.mesh.bind_addr)` (`:383`) — fatal on error.
3. `advertise_addrs` (`:382-403`): `BUZZ_MESH_ADVERTISE_ADDR` → `POD_IP` + actual
   bound port → all endpoint IP addrs.
4. `GossipRecord::new(runtime_id, addrs, PROTO_VERSION)` + static capabilities
   (`:439-441`).
5. `MeshMembership::new(record).with_expected_relay_pubkey(relay_pubkey_hex)`
   (`:444-445`).
6. `ReadyRegistry::new(pool, registry_refresh)`; `ReadyRecord::new(...)`
   (`:447-453`).
7. `registry.publish_ready(&record)` — **fatal on error** (`:456-463`).
8. `spawn_registry_heartbeat(registry.clone(), record, || !shutting_down)`
   (`:467-471`).
9. `MeshRuntime::start(endpoint, membership, Some(registry))` (`:473`) — spawns 3
   loops.
10. `runtime.reconcile_now().await` (`:478`) — dial seeds immediately.
11. Drain watcher task: polls `shutting_down` every 500 ms, then `begin_drain()` +
    `owners.drain_all()` and returns (`:481-497`).
12. `Arc<dyn RelayMeshMembership>` (`:501`, never read) and
    `Arc<dyn RelayPeerTransport>` (`:502`) erased from the runtime.
13. `transport.set_inbound(Box::new(dispatcher.clone()))` (`:509`).
14. Return `MeshHandle` (`:511-520`).

Then `main.rs:455-459` calls `handle.wire_consumers(...)` **before**
`state.mesh.set(handle)` (`main.rs:457`), so the three profile consumers are
registered before the handle is observable; `main.rs:460` is
`unreachable!("mesh handle is set exactly once, right here")`.

Read path: `AppState::mesh()` (`state.rs:812`) — one caller, `router.rs:381`.

Consumer-side integration points:

| Consumer file | Uses |
|---|---|
| `mesh_boot.rs` | `MeshEndpoint`, `GossipRecord`, `ReadyRecord`/`ReadyRegistry`, `MeshMembership`, `MeshRuntime`, `spawn_registry_heartbeat`, `MeshStatus`, both seams, `InboundHandler`, `Profile`, `StreamRole`, `GoodbyeReason`, `WIRE_VERSION` |
| `audio/mesh.rs` | `FencedHeader`, `MeshDatagram`, `RelayPeerTransport`, `RuntimeId` (`audio/mesh.rs:37,56,260`) |
| `audio/join.rs` | full wire set + `MeshStream`, both half traits, `MeshError` fence variants; defines `HuddleControlMsg` as the opaque `Data` payload (`join.rs:797-801`) |
| `tunnel/reliable.rs` | wire set + `MeshStream`; chunks at 1 MiB against the 16 MiB cap (`reliable.rs:26-31`, assert `:945`) |
| `tunnel/directory.rs` | `MeshError` fence variants only (`:378,395,413,430,824,842,870,914`) |
| `api/mesh_demo.rs` | `RelayPeerTransport` + `MeshEndpoint`/`MeshPeer` in tests |
| `config.rs` | `MeshConfig` (`:136`, `:508`) |
| `audio/handler.rs` | reaches the handle via `AppState` (`:308`, `:446`) |

##### Consumer-side failure modes

- Handle absent (`mesh` off) → every consumer path is a no-op by contract
  (`state.rs:624-626`, `mesh_boot.rs:19-20`).
- Traffic arriving before a dispatcher slot is registered → logged and dropped
  (`mesh_boot.rs:99-104`, `:128-134`); documented as a bounded boot-window race made
  safe by the peer's fenced retry (`mesh_boot.rs:53-55`).
- Second registration on a slot → warn + ignored, first handler wins
  (`mesh_boot.rs:72-74`, `:79-81`, `:85-87`).
- `Profile::RealtimeMedia` arriving as a *stream* → rejected as a protocol violation
  (`mesh_boot.rs:118-126`).
- Mesh runtime loops are never shut down (`MeshRuntime::shutdown` uncalled), so on
  SIGTERM the accept/reconcile/gossip loops and the heartbeat task run until process
  exit.

---

#### 7. Not integrated (absences worth recording)

- **No metrics exporter.** The crate has no `metrics` dependency; the documented
  `mesh_fence_rejections_total` (`lib.rs:102-109`) does not exist. All mesh
  observability is the ad-hoc counters in `MeshStatus` plus `tracing` (28 sites).
- **No tracing spans / OTel.** `tracing` is used only for `info!/warn!/debug!`
  events; no `#[instrument]`, no span propagation across the mesh, so a cross-pod
  session cannot be traced end to end.
- **No `buzz-audit`.** Peer admission and rejection decisions are not audited, only
  logged.
- **No k8s API client.** Peer discovery is Redis-only; `POD_IP` comes from the
  Downward API via env, explicitly to need "zero RBAC" (`mesh_boot.rs:380-381`).
- **No `buzz-db` / Postgres.** The mesh is Redis-only.


## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: Integrations

Two integration edges only: **inward** from `buzz-relay` (the sole dependent crate) and
**outward** to a TLA+ spec file that is read by humans, never by code. No network client, no
external service, no message broker.

---

### Dependency graph

`buzz-conformance` is declared in exactly two manifests:

| Manifest | Line | Form |
|---|---|---|
| workspace root | `Cargo.toml:5` | workspace member |
| workspace root | `Cargo.toml:125` | `buzz-conformance = { path = "crates/buzz-conformance" }` |
| `crates/buzz-relay/Cargo.toml` | `:20` | `buzz-conformance = { workspace = true }` |

**`buzz-admin` does not declare this dependency.** Grep for `conformance` in
`crates/buzz-admin/Cargo.toml` returns nothing, and grep for `buzz_conformance` /
`conformance` across `crates/buzz-admin/src/` returns nothing. Repo-wide, the only
`Cargo.toml` files mentioning the crate are the workspace root and `buzz-relay`.

Outbound, the crate depends on `serde`, `serde_json`, `thiserror`, `uuid`
(`crates/buzz-conformance/Cargo.toml:26-29`) and `proptest` as a dev-dep (`:34`). The
"independence rule" comment (`:7-24`) enumerates the crates it must never depend on
(`buzz-db`, `buzz-relay`, `buzz-pubsub`, `buzz-auth`, `buzz-search`, `buzz-audit`) and the
manifest honors it — including the deliberate refusal to reuse `buzz_core::CommunityId`, so
that type's "no Serde, no `From<Uuid>`" fence stays intact (`:9-14`, restated
`src/lib.rs:47-63`).

---

### Relay call-in path

**1. Tracer binding.** `AppState` holds `pub tracer: Arc<dyn buzz_conformance::Tracer>`
(`crates/buzz-relay/src/state.rs:620`, doc `:615-619`). The constructor binds
`Arc::new(crate::conformance::NoopTracer)` (`crates/buzz-relay/src/state.rs:798`, comment
`:794-797`). Nothing else ever writes the field — grep for `tracer:` and `.tracer =` across
`crates/buzz-relay/src/` finds only this one assignment plus reads at
`handlers/ingest.rs:1383`, `handlers/req.rs:145`, `:356`, `:672`. The constructor comment
promises "Conformance tests overwrite this with a JsonlTracer … (see test helpers in
`crates/buzz-test-client` once those land)" (`:795-797`); those helpers do not exist —
`crates/buzz-test-client/tests/conformance_multitenant.rs` never references
`buzz_conformance`.

**2. Two tracer impls, relay-side.** `crates/buzz-relay/src/conformance/tracers.rs` declares
`NoopTracer` (`:16-20`, empty `record`) and `JsonlTracer` (`:30-45`) which serializes one JSON
object per line into a truncating-open file (`:37-43`) behind a `Mutex<BufWriter<File>>`
(`:31`). `JsonlTracer::record` (`:55-72`) recovers from lock poisoning by
`e.into_inner()` (`:60-63`) and swallows serialization/IO failures (`:68-70`). `JsonlTracer`
is never constructed anywhere in the repo.

The relay's `NoopTracer` shadows the identically-named one in the crate
(`crates/buzz-conformance/src/lib.rs:323-327`) and is what `mod.rs:46` re-exports, so the
crate's own no-op impl is dead.

**3. Ingest seam.** `handlers/ingest.rs:47-50` imports the helper set
(`self as conf, channel_label, claimed_community_from_event, emit, msg_id_label,
state_for_request, EmitGuard, TraceAction, Verdict`). `ingest_event`:

| Step | Line |
|---|---|
| build `AbstractState` | `:1407` |
| arm guard, receive counting tracer | `:1408-1412` |
| delegate to `ingest_event_inner(state, &tracer, …)` | `:1414` |
| map terminal `IngestError` → `SanitizedError` | `:1436-1443` |
| guard drops (implicit) | comment `:1445-1449` |

`ingest_event_inner` takes `tracer: &Arc<dyn buzz_conformance::Tracer>` (`:1455`) and emits at
`:1573` (via `emit_product_feedback_success`, `:133-147`), `:1820-1828`, `:2215-2222`,
`:2374`, `:2511`.

**4. REQ seam.** `handlers/req.rs:116-118` builds `trace_state: Option<AbstractState>` from
`PublicKey::from_slice(&pubkey_bytes).ok().map(…)`. It is `Option` **only** because the pubkey
bytes may be malformed (comment `:112-115`) — it has no relationship to which tracer is bound,
so `trace_state` is `Some` on essentially every authenticated REQ even under `NoopTracer`.

Plumbed by value into the search lane: `handle_search_req(..., trace_state.as_ref())`
(`:230`), parameter `trace_state: Option<&crate::conformance::AbstractState>` (`:514`).

Three gated blocks:

| Block | Gate | Extra DB query | Emit |
|---|---|---|---|
| membership confirmation | `:143` | none (reuses the `db.is_member` result from `:137-141`) | `record_req_authcheck` `:144-150` |
| non-search read | `:338` | `db.communities_of_channels` `:349` | `record_read_message_rows` `:359-365` |
| search read | `:649` | `db.communities_of_channels` `:661` | `record_read_by_id_rows` `:671-677` |

The two `communities_of_channels` calls are inside the `trace_state.is_some()` blocks, not
inside a tracer-type check — so both run and are awaited even though the resulting
`TraceStep` is immediately discarded by `NoopTracer::record`. On DB error the code substitutes
an empty `HashMap` (`:356`, `:668`), which turns every channel-scoped row into a
missing-lookup and yields a single `ImplBug` step for the whole page.

**5. Guard arm/disarm.** There is no disarm API. `EmitGuard::arm`
(`conformance/mod.rs:383-400`) hands back a `CountingTracer` (`:356-373`) and the caller
substitutes it for the duration; `Drop` (`:403-415`) decides based on the counter. The ingest
site names the seam `"ingest_event_exited_without_trace"` (`ingest.rs:1411`) and the
guard-drop test asserts that string flows through
(`conformance/mod.rs:516-521`). One guard exists in the whole relay — the REQ path arms none.

**6. Error-alphabet coupling.** `sanitized_reason_for` (`conformance/mod.rs:422-430`) is the
only place `buzz-relay`'s error type touches the schema, and it is an exhaustive match over
`crate::handlers::ingest::IngestError` — a new variant is a compile error, which is the
mechanism `crates/buzz-conformance/TRACE_SCHEMA.md:120-124` calls "closed".

---

### TLA+ spec relationship

The relationship is documentary: no build step, test, or CI job reads
`docs/spec/MultiTenantRelay.tla`. The coupling is doc comments carrying line numbers.

| Rust site | Cites | Actual spec line | Match |
|---|---|---|---|
| `src/lib.rs:186` | `WriteInsert` 514–550 | `:514` | yes |
| `src/lib.rs:187` | `WriteInsertGlobal` 559–595 | `:559` | yes |
| `src/lib.rs:188` | `WriteDuplicate` 606–637 | `:606` | yes |
| `src/lib.rs:189` | `SanitizedError` 778 | `:778` | yes |
| `src/lib.rs:190` | `AuthCheck` 794 | `:794` | yes |
| `src/lib.rs:191` | `ReadMessageRows` 643 | `:643` | yes |
| `src/lib.rs:192` | `ReadByIdRows` 681 | `:681` | yes |
| `src/lib.rs:193` | `ReadHostFeedRows` "line ~720" | `:703` | off by 17 |
| `src/transitions.rs:53` / `:296` | `Inv_NonInterference` "line ~983" | `:985` | ~approximate |
| `src/transitions.rs:54` | `Inv_ReadConfinement` "line ~1003" | `:999` | ~approximate |
| `src/lib.rs:115` | `AuthCheck` verdict alphabet, "spec line 794" | `:800` for `verdict ==` | close |

The relay repeats the citations: `ingest.rs:1803` ("Spec AuthCheck (line 794)"),
`:2355-2357` ("WriteInsert (line 514) / WriteDuplicate (line 606)"), `:2484-2492`
("WriteInsert (line 514) / WriteInsertGlobal (line 559) / WriteDuplicate (line 606) …
lines 559-595"), `conformance/mod.rs:422-423` ("spec line 778"). All resolve correctly.

`TRACE_SCHEMA.md` drifts from both: it cites `WriteInsertGlobal` at "line 562"
(`:57`) and `WriteDuplicate` at "line 612" (`:69`) — actual `:559` and `:606`.

**Spec surface not integrated.** `Next` has 23 disjuncts
(`docs/spec/MultiTenantRelay.tla:933-956`); the trace vocabulary covers 8. `Safety` conjoins
13 invariants (`:1129-1142`); the Rust checker enforces `Inv_NonInterference` for reads and a
fragment of `AuthCheck`. `docs/spec/MultiTenantRelay.cfg:26` declares a 9-element
`SanitizedErrors` set against the Rust enum's 3. `docs/spec/GitOnObjectStore.tla` is a
separate spec consumed by `crates/buzz-relay/src/api/git/cas_publish.rs` — unrelated to this
crate.

---

### Build / CI integration

| Surface | Where | Runs the crate? |
|---|---|---|
| `just test-unit` | `justfile:275-296`, conformance step at `:286-290` (`cargo nextest run -p buzz-conformance`) | **Yes** — all targets, lib + both integration tests |
| `just ci` | `justfile:266` → `test-unit` | Yes, transitively |
| `scripts/run-tests.sh unit` | `:78-103`, conformance at `:96-99` (`cargo test -p buzz-conformance`) | Yes (nextest-absent fallback) |
| relay-side emitter tests | `crates/buzz-relay/src/conformance/mod.rs:431-726`, `handlers/ingest.rs:2530-2565` | **No** — `buzz-relay` appears in neither unit list |
| GitHub Actions | grep `conformance` in `.github/workflows/` | No hits |

Contrast with the `buzz-relay-mesh` crate (`Cargo.toml:27`), which appears in no
`test-unit`/`run-tests.sh` step either — the same omission pattern, so `buzz-conformance`
being present in the unit gate is the exception rather than the rule for non-core crates.

**Documentation integration is absent.** Grep for `buzz-conformance`, `MultiTenantRelay.tla`,
`conformance`, `TLA`, and `formal` across `AGENTS.md`, `ARCHITECTURE.md`, and
`CONTRIBUTING.md` returns nothing. The crate is missing from `AGENTS.md`'s repo-structure
table (which lists `buzz-audit` at `AGENTS.md:46` and its neighbours but no
`buzz-conformance`), and `ARCHITECTURE.md` has no mention of the formal-methods lane at all.
The crate's own `LIMITS.md` (125 lines) and `TRACE_SCHEMA.md` (163 lines) are the only prose,
and neither is linked from any top-level doc.


## Module: buzz-acp — harness core & orchestration (`crates/buzz-acp/src`)
### Aspect: Integrations

#### Internal crate dependencies (`Cargo.toml:19-22`)

| Crate | Declared | Actually used by `lib.rs` |
|---|---|---|
| `buzz-core` | `Cargo.toml:20` | yes — `kind::*` (`lib.rs:23-26`, `lib.rs:82`), `observer::{decrypt_observer_payload, encrypt_observer_payload, OBSERVER_FRAME_TELEMETRY, OBSERVER_MAX_PLAINTEXT_LEN}` (`lib.rs:27-30`), `verify_event` (`lib.rs:845`) |
| `buzz-sdk` | `Cargo.toml:21` | yes — `nip_oa::verify_auth_tag` (`lib.rs:128`, `lib.rs:350`), `nip_oa::parse_auth_tag` (`lib.rs:1341`), `build_agent_observer_frame` (`lib.rs:810`) |
| `buzz-persona` | `Cargo.toml:22` | **no** — zero references. `grep -rn 'buzz_persona\|buzz-persona' crates/buzz-acp/` returns only `Cargo.toml:22`. The only `persona`-named thing in the crate is `Config::persona_env_vars` (`config.rs:533-535`), a plain `Vec<(String,String)>` populated in `config.rs:945-999` and threaded to `AcpClient::spawn` (`lib.rs:1762`, `3488`, `3666`, `3733`) without touching the crate. |

`buzz-persona` is also the only path-declared internal dep (`{ path = "../buzz-persona" }`) rather than `workspace = true`.

#### Relay: WebSocket primary, HTTP bridge secondary — both used

`buzz-acp` does **not** depend on `buzz-ws-client`. It reimplements the client stack in `relay.rs`: its own `connect_async` (`relay.rs:3825`), `RelayMessage` enum (`relay.rs:471`), AUTH challenge wait (`relay.rs:3865`), `send_auth_response` (`relay.rs:3433`), background task (`relay.rs:1534`), and reconnect state machine (`relay.rs:2893`, `3022`). `Cargo.toml` pulls `tokio-tungstenite` directly (`Cargo.toml:31`) instead of relying on the shared crate. It also diverges functionally: `relay.rs` re-authenticates on a mid-session AUTH challenge, where `buzz-ws-client` only records it.

`lib.rs`'s own contributions to that duplication:

| Concern | `lib.rs` site |
|---|---|
| Presence must go over WS, not HTTP | `publish_presence` `lib.rs:77-91`; the doc comment at `lib.rs:71-72` states ephemeral kinds 20000–29999 are rejected by the HTTP bridge |
| Typing indicators over WS with non-blocking `try_publish_event` | `lib.rs:2332-2340`; the comment at `lib.rs:2329-2331` cites issue #35 — they must not block the loop during reconnect |
| Observer frames over WS via `RelayEventPublisher` | `lib.rs:790-833` |
| Startup watermark handed to the background task so the first REQ uses `watermark − 5s` instead of `since=now` | `lib.rs:1348-1354` |
| Graceful WS close on exit (5 s wait, not abort) | `relay.shutdown()` `lib.rs:2690`; comment cites issue #40 |

HTTP bridge usage reached from `lib.rs`:

| Call | Site | Path |
|---|---|---|
| `rest_client.query(&[filter])` for a kind:0 profile during sibling verification | `lib.rs:310-315` | `POST /query` |
| `pool::reaction_add` / `reaction_remove` (👀) | `lib.rs:2204-2213`, `lib.rs:2000-2010` | via `RestClient` |
| `pool::post_failure_notice` | `lib.rs:3014-3031` | via `RestClient` |
| `ChannelInfoResolver` lazy kind:39000 fetch backing `is_dm_channel` | `lib.rs:273-286` → `pool::ChannelInfoResolver::resolve` | `POST /query` |
| channel discovery at startup | `relay.discover_channels()` `lib.rs:1437-1443` | REST |

`RestClient` signs every request with NIP-98 (`relay.rs:264-307`, header applied `relay.rs:380-385`) rather than relying on an `X-Pubkey` header, so the harness is not exposed to the unsigned-header impersonation path that `BUZZ_REQUIRE_AUTH_TOKEN=false` leaves open on the relay bridge. That said, the DM classification that the author gate fails closed on (`lib.rs:273-286`) depends on this HTTP path succeeding — a bridge outage degrades every non-startup channel to "treated as DM", collapsing `allowlist`/`anyone` to owner-only.

#### ACP subprocess protocol

`lib.rs` drives `acp::AcpClient` over stdio JSON-RPC. Sites:

| Operation | `lib.rs` site | `acp.rs` |
|---|---|---|
| spawn | `initialize_agent_pool` `lib.rs:3755-3760`, `spawn_and_init` `lib.rs:3856-3858`, `spawn_auth_client` `lib.rs:3885-3888` | `acp.rs:408-503` |
| `initialize` (60 s timeout in pool startup) | `lib.rs:3765`, `lib.rs:3861` | `acp.rs:539` |
| `session/new` (models probe only, in `lib.rs`) | `lib.rs:4048` | `acp.rs:563` |
| `authenticate` | `lib.rs:3985` | `acp.rs:549` |
| `session/prompt` | delegated to `pool::run_prompt_task` (`lib.rs:2971-2979`, `lib.rs:3563-3572`) | `acp.rs:654`, `676` |
| steer channel install | `agent.acp.install_steer_rx(rx)` `lib.rs:2934` | `acp.rs:800` |
| `shutdown` | `lib.rs:2610`, `2637`, `2645`, `3672`, `3698`, `3707`, `3711`, `3866` | `acp.rs:376` |

Protocol version is read as `init_result["protocolVersion"].as_u64().unwrap_or(1)` (`lib.rs:3785`, `lib.rs:3864`) — a missing or non-numeric field silently becomes legacy v1, which changes prompt composition (`pool::prepend_base_for_legacy`, pinned by tests at `lib.rs:4193-4216`).

Agent identity is normalized from `agentInfo.name` with `serverInfo.name` as fallback (`normalized_agent_name` `lib.rs:3686-3695`), lowercased and trimmed. `run_models` checks the same two keys in the **opposite** order (`serverInfo` first, `lib.rs:4062-4064`) — an agent that sets both would report different names depending on the code path.

`AcpClient::spawn` inherits the parent environment by default and only injects `extra_env` keys **not already set** in the parent (`acp.rs:456-461`, operator-wins). Consequence: the harness's own `BUZZ_PRIVATE_KEY` / `BUZZ_RELAY_URL` / `BUZZ_AUTH_TAG` are inherited by every agent subprocess implicitly, in addition to the explicit `mcpServers` env injection described below.

#### MCP server integration

`build_mcp_servers` (`lib.rs:4141-4184`) produces at most **one** `McpServer`, gated on `config.mcp_command` being non-empty (`lib.rs:4142-4144`). Default is `""` (`config.rs:261`), so an out-of-the-box run hands the agent zero MCP servers.

Server name is `Path::file_stem()` of the command, falling back to `"mcp"` (`lib.rs:4146-4150`). `args` is always empty (`lib.rs:4152`). Injected env:

| Var | Line | Value |
|---|---|---|
| `BUZZ_RELAY_URL` | `lib.rs:4155-4158` | `config.relay_url` |
| `BUZZ_PRIVATE_KEY` | `lib.rs:4159-4170` | `config.keys.secret_key().to_bech32()` — the raw `nsec1…` |
| `BUZZ_AUTH_TAG` | `lib.rs:4171-4180` | forwarded from the harness env, skipped when empty |

This `McpServer` list flows to `PromptContext.mcp_servers` (`lib.rs:1531`) → `pool.rs:832` → `acp.rs:566-571` where it becomes the `mcpServers` field of the `session/new` JSON-RPC request. **The private key travels over the agent's stdin pipe as part of that request body**, not via argv or the process environment of the ACP child itself. See the Security aspect for the observer-feed consequence.

#### Nostr / crypto libraries

`nostr` crate for `Event`, `EventBuilder`, `Keys`, `PublicKey`, `Filter`, `Tag`, `ToBech32` (`lib.rs:38`). `rustls` with the `ring` provider installed once at startup for `wss://` (`Cargo.toml:33`, `lib.rs:1241-1243`). NIP-44 encrypt/decrypt of observer payloads is delegated to `buzz-core` (`lib.rs:797`, `lib.rs:871`), so `lib.rs` holds no crypto primitives of its own.

#### Desktop / launcher integration

Two env vars read directly in `lib.rs` come from the desktop launcher, not the CLI:

| Var | Line | Role |
|---|---|---|
| `BUZZ_AUTH_TAG` | `lib.rs:125`, `lib.rs:1338-1341`, `lib.rs:4173` | NIP-OA owner attestation: owner resolution, relay membership delegation, MCP forwarding |
| `BUZZ_MANAGED_AGENT_START_NONCE` | `lib.rs:1503` | correlation ID stamped into every `managed_agent_runtime_lifecycle` observer event (`lib.rs:93-121`) |

`BUZZ_ACP_SETUP_PAYLOAD` (`setup_mode.rs:83`, read `setup_mode.rs:214`) is the desktop's signal that the agent is not ready; it diverts to the setup listener at `lib.rs:1298-1303`.

Lifecycle states emitted for the launcher: `listening` (`lib.rs:1517-1526`), `waking` (`lib.rs:1714-1721`), `ready` (`lib.rs:2514-2521`), `failed` (`lib.rs:2482-2489` and `lib.rs:2534-2541`).


## Module: buzz-acp — relay client & observer (`crates/buzz-acp/src`)
### Aspect: Integrations

#### Headline finding: `buzz-ws-client` is not a dependency

`crates/buzz-acp/Cargo.toml` lists `buzz-core`, `buzz-sdk`, and `buzz-persona`
as its only internal dependencies. `buzz-ws-client` is absent. The only crates
that depend on the shared NIP-42 client are `buzz-cli`
(`crates/buzz-cli/Cargo.toml:77`) and `buzz-test-client`
(`crates/buzz-test-client/Cargo.toml:13`).

`relay.rs` instead imports `tokio_tungstenite::{connect_async, tungstenite::Message,
MaybeTlsStream, WebSocketStream}` directly (`relay.rs:125`) — the same imports
`buzz-ws-client/src/connection.rs:9` uses — and reimplements the connect, the
`RelayMessage` enum, the frame parser, and the whole NIP-42 handshake.

#### Duplication table

| Behaviour | `crates/buzz-acp/src/relay.rs` | `crates/buzz-ws-client/src/connection.rs` | Divergence |
|---|---|---|---|
| `WsStream` type alias | `:525` | `:14` | Identical definition |
| URL parse before connect | `:3830-3832` (`url::Url`, error → `RelayError::Http`) | `:49-51` (`url::Url`, error → `WsClientError::Url`) | Neither asserts a `ws`/`wss` scheme |
| `connect_async` | `:3834-3837` | `:53-55` | acp wraps in `CONNECT_TIMEOUT` = 30 s (`:69`); ws-client has no connect timeout |
| `debug!("connected to relay at {url}")` | `:3838` | `:57` | Byte-identical log line |
| `RelayMessage` enum | `:470-495` (private) | `message.rs:8-41` (`pub`) | ws-client wraps OK in an `OkResponse` newtype variant; acp inlines the three fields |
| `OkResponse` struct | `:3917-3921` (private, no derives) | `message.rs:44-52` (`pub`) | Same three fields, duplicated shape |
| Frame parser | `parse_relay_message` `:3531-3620` | `parse_relay_message` `message.rs:55-144` | Two independent hand-written `Vec<Value>` parsers; acp's returns `RelayError`, ws-client's `WsClientError` |
| AUTH-challenge wait | `wait_for_auth_challenge` `:3865-3913` | `wait_for_auth_challenge` `:156-206` | **ws-client caps the challenge at 1024 bytes (`:198-201`); acp has no cap on any path** |
| Wait for AUTH OK | `wait_for_any_ok` `:3924-3982` | `wait_for_ok` `:208-263` | acp accepts the **first OK of any id** (comment concedes it, `:3846-3854`); ws-client matches the AUTH event id (`:226`, `:255`) |
| Build + send AUTH event | `send_auth_response` `:3433-3461` | `authenticate` `:70-93` + `build_auth_event` `message.rs:151-166` | acp hand-builds the tag vec when `auth_tag` is present (`:3442-3456`); ws-client always goes through `EventBuilder::auth` then appends the tag |
| AUTH challenge timeout | `AUTH_TIMEOUT` `:64` = 20 s | `AUTH_CHALLENGE_TIMEOUT_SECS` `:17` = 20 s | Same value, two constants; ws-client pins it with a compile-time test (`:279-281`) |
| AUTH OK timeout | reuses `AUTH_TIMEOUT` (20 s) at `:3849` | `AUTH_OK_TIMEOUT_SECS` `:20` = 20 s | Same value, separate constant |
| Ping → Pong in read loops | `:2370-2376`, `:3903-3907`, `:3963-3967` | `:148-150`, `:208-210`, `:262-264` | acp routes every send through `ws_send_timeout` (10 s, `:3312-3323`); ws-client uses bare `self.ws.send` |
| Non-target frame buffering | `VecDeque<RelayMessage>` threaded through `do_connect` (`:3833`, `:3902`, `:3960`) | `self.buffer: VecDeque<RelayMessage>` field (`:28`, `:205`, `:257`) | acp replays the buffer through `handle_ws_message` after re-serialising to text (`:2393-2450`); ws-client leaves it for `next_event` (`:105-112`) |
| Mid-session AUTH | Re-authenticates immediately, reconnects on failure (`:2344-2353`); rejects on `OK false auth*` (`:2359-2363`) | Records `pending_challenge` only (`:144`, `:256-258`) | **Functional divergence** |
| Outbound frame logging | Only `debug!("sent AUTH response for challenge")` (`:3459`) — no frame body | `send_raw` logs the full frame: `debug!("→ relay: {text}")` (`:123`) | acp **fixes** the ws-client token-leak: the AUTH event and any `auth_tag` bearer value never reach the log |
| Reconnect / retry | Three loops (`:3786`, `:2893`, `:3022`) | None | acp adds it |
| Rate-limit handling | Gate + paced drain (`:1182-1207`, `:1621-1699`) | None | acp adds it |

Verification of the line numbers cited in earlier analysis:
`relay.rs:2344-2350` is correct for the mid-session AUTH arm.
`relay.rs:3435-3461` is off by two — `send_auth_response` begins at
`relay.rs:3433` and ends at `relay.rs:3461`.
`relay.rs:3843-3845` is also shifted: `wait_for_auth_challenge` is called at
`relay.rs:3843` and `send_auth_response` at `relay.rs:3845`, with
`wait_for_any_ok` at `relay.rs:3849`.

#### Which `buzz-ws-client` weaknesses this copy repeats, fixes, or worsens

| Weakness | Status in `relay.rs` | Evidence |
|---|---|---|
| `ws://` accepted with no scheme check | **Repeated** | `:3830-3832` parses with `url::Url` and passes `parsed.as_str()` straight to `connect_async`; no scheme assertion. The `is_terminal_ws_error` test `do_connect_wrong_scheme_is_terminal` (`:5549-5557`) only confirms tungstenite's own `Url` error is treated as terminal — it does not add a check. `relay_ws_to_http` (`:3470-3475`) likewise accepts either. |
| AUTH frame logged in full at debug, including `auth_tag` | **Fixed** | `send_auth_response` logs a fixed string with no event body (`:3459`). There is no `send_raw`-equivalent that logs frames; the two other WS send helpers log only on error (`:2620`, `:3207`). |
| 1024-byte challenge cap on only 1 of 3 intake paths | **Made worse** | acp has the cap on **0 of 3** paths: `wait_for_auth_challenge` returns the challenge unchecked (`:3901`), `parse_relay_message`'s `"AUTH"` arm has no length guard (`:3610-3616`), and the mid-session arm passes it straight into `send_auth_response` (`:2344-2348`). |
| No retry / reconnect | **Fixed** | `retry_initial_connect` (`:3786-3822`), `try_autonomous_reconnect` (`:2893-3018`), `wait_for_reconnect` (`:3022-3151`), plus ping/pong death detection (`:1855-1927`). |

#### Relay protocol touch points

WebSocket (NIP-01 verbs implemented): outbound `AUTH`, `REQ`, `CLOSE`, `EVENT`,
`Ping`, `Pong`, `Close`; inbound `AUTH`, `EVENT`, `OK`, `EOSE`, `CLOSED`,
`NOTICE`. `COUNT` is not implemented in either direction.

HTTP bridge (used for all reads, so the WS/HTTP split is real, not incidental):

| Endpoint | Call site | Auth |
|---|---|---|
| `POST /query` | `relay.rs:399-406`, used at `:670`, `:705`, `engram_fetch.rs:91` | NIP-98 kind:27235 + optional `x-auth-tag` |
| `POST /events` | `relay.rs:411-423`, used by `pool.rs` reactions/metrics/notices | NIP-98 kind:27235 + optional `x-auth-tag` |

No `POST /count` call exists. The unauthenticated `X-Pubkey` header path — which
is exploitable when `BUZZ_REQUIRE_AUTH_TOKEN` defaults to false — is **not**
used: every bridge request signs a fresh NIP-98 event per attempt
(`relay.rs:378-381`).

#### `buzz-core`

| Import | Location | Use |
|---|---|---|
| `kind::{KIND_AGENT_OBSERVER_FRAME, KIND_MEMBER_ADDED_NOTIFICATION, KIND_MEMBER_REMOVED_NOTIFICATION, KIND_TYPING_INDICATOR}` | `relay.rs:118-123` | REQ kinds, publish kind, observer-frame detection |
| `kind::KIND_NIP29_GROUP_MEMBERS` | `relay.rs:665` | discovery filter |
| `kind::KIND_NIP29_GROUP_METADATA` | `relay.rs:700` | discovery filter |
| `kind::KIND_AGENT_ENGRAM` | `engram_fetch.rs:17`, used `:81` | engram filter |
| `engram::{conversation_key, d_tag, select_head, validate_and_decrypt, Body}` + `engram::CORE_SLUG` | `engram_fetch.rs:16`, `:77` | NIP-AE key derivation, LWW head selection, decrypt |

#### `buzz-sdk`

`relay.rs`, `observer.rs`, and `engram_fetch.rs` do **not** import `buzz-sdk`.
The crate is a dependency (`crates/buzz-acp/Cargo.toml`) but is consumed
elsewhere in `buzz-acp`: `nip_oa::verify_auth_tag` / `parse_auth_tag`
(`lib.rs:1370`), `build_agent_observer_frame` for the kind:24200 wrapper, and
`build_reaction` / `build_remove_reaction` / `build_message` in `pool.rs`. So the
event *builders* for the durable kinds this module transports live in the SDK,
while the two events this module builds itself — the NIP-42 AUTH event
(`relay.rs:3442-3457`) and the typing indicator (`relay.rs:866-868`) — bypass it
and use raw `nostr::EventBuilder`.

#### External crates

`tokio-tungstenite` + `futures-util` (`relay.rs:124-125`) for the socket;
`rustls` 0.23 with the `ring` provider, referenced directly for the terminal-TLS
downcast (`relay.rs:3724-3766`) — the doc comment notes this relies on a single
rustls version in the tree (`relay.rs:3762-3763`), a real coupling to the
workspace dependency graph. `reqwest` for the bridge, `sha2` + `base64` + `hex`
for NIP-98 (`relay.rs:268-269`, `:284`, `:299`), `url` for parsing,
`uuid`/`chrono` for ids and timestamps, `thiserror` for `RelayError`,
`serde`/`serde_json` throughout. `observer.rs` uses only
`tokio::sync::broadcast`, `std::sync::{Mutex, atomic}`, `serde::Serialize`,
`chrono`, and `uuid`. No `rand` — jitter is derived from
`SystemTime::subsec_nanos()` (`relay.rs:3337-3346`).

#### In-crate consumers

`lib.rs` is the only real consumer of `HarnessRelay`: `connect`
(`lib.rs:1344`), `set_startup_watermark` (`:1352`),
`subscribe_membership_notifications` (`:1359`), `event_publisher` (`:1364`,
`:1406`, `:1413`-adjacent), `subscribe_observer_controls` (`:1413`),
`take_observer_control_rx` (`:1416`), `discover_channels` (`:1432`),
`rest_client` (`:1551`, `:1552`), `subscribe_channel` / `subscribe_channel_from`,
`unsubscribe_channel`, `next_event`, `build_typing_event` + `try_publish_event`
(`:2333-2341`), `reconnect`, `shutdown`. `setup_mode.rs` uses
`event_publisher` (`:383`) and `publish_event` (`:641`). `pool.rs` consumes
`RestClient` and `ChannelInfo` (`pool.rs:41`) and calls
`engram_fetch::build_core_section` (`pool.rs:1387`).


## Module: buzz-acp — agent pool & lifecycle (`crates/buzz-acp/src`)
### Aspect: Integrations

#### ACP subprocess protocol (stdio, NDJSON JSON-RPC)

The pool does not own framing or the child handle — both live in `AcpClient` (`acp.rs`). `pool.rs` drives the protocol through typed methods on `agent.acp`:

| ACP method / call | Call site | Notes |
|---|---|---|
| `session/new` (via `session_new_full`) | `pool.rs:828-840` | passes `ctx.cwd`, `ctx.mcp_servers.clone()`, and the combined system prompt |
| `_goose/unstable/session/system-prompt/set` | `pool.rs:842-849` | probed once; `-32601` latches `goose_system_prompt_supported = Some(false)` (`pool.rs:852-858`) |
| `session/set_config_option` (model) | `pool.rs:953-960` via `ModelSwitchMethod::ConfigOption` | 5 s timeout |
| `session/set_model` (unstable) | `pool.rs:961-963` | 5 s timeout |
| `session/set_config_option` `configId: "mode"` | `pool.rs:1037-1041` | only when the mode is advertised in `session/new` (`pool.rs:924-928`) |
| `session/prompt` (with idle timeout) | `pool.rs:1611-1619` (initial message), `pool.rs:1832-1837` / `1843-1848` (blocks form) | idle + hard deadline both passed in |
| `session/cancel` (via `cancel_with_cleanup`) | `pool.rs:1646-1648`, `pool.rs:2075-2077` | timeout paths, grace = `ctx.idle_timeout` |
| `cancel_with_cleanup_grace` | `pool.rs:1863-1866` | control-signal path, fixed 5 s grace |
| Goose native steer | delivered as `SteerRequest` over a capacity-1 mpsc into the read loop (`pool.rs:646-662`) | read loop, not the pool, writes the JSON-RPC line |

Framing details the pool depends on but does not implement: line-delimited JSON with `LinesCodec::new_with_max_length(MAX_LINE_SIZE)` (`acp.rs:487`), request/response correlation by integer `id` with non-matching ids skipped (`acp.rs:979-1004`), and notifications distinguished by absence of `id` (`acp.rs:1038-1052`).

Protocol-version branching lives here: `has_system_prompt_support` treats `protocol_version >= 2` as system-prompt-capable except for `"goose"`, which requires its custom method to have succeeded (`pool.rs:173-183`). Legacy (`< 2`) agents get `[Base]` and `[Channel Canvas]` folded into the user message instead (`pool.rs:1090-1122`).

#### MCP servers

`PromptContext::mcp_servers: Vec<McpServer>` (`pool.rs:483`) is cloned into every `session/new` (`pool.rs:832`). The pool treats it as an opaque, already-built list: it neither validates the command, resolves it, nor spawns it. The list is built once in `lib.rs::build_mcp_servers` (`lib.rs:4141-4184`) from `config.mcp_command`, producing at most **one** server whose `name` is the command's file stem, `args` empty, and `env` carrying `BUZZ_RELAY_URL`, `BUZZ_PRIVATE_KEY` (bech32 nsec), and optionally `BUZZ_AUTH_TAG`. Empty `mcp_command` yields an empty vec (`lib.rs:4142-4144`).

Consequence: the MCP server process is spawned by the *agent*, from a spec the harness sent over stdio. Nothing in this module constrains that command — the only constraint anywhere is that `mcp_command` comes from harness config/CLI, not from a channel message or a persona pack (persona `mcp_servers` never reach this code path; see Features).

#### Relay (over `relay::RestClient`, HTTP)

Every relay interaction in this module is REST, not WebSocket, and every one is timeout-bounded and fail-open:

| Purpose | Function | Filter / endpoint | Timeout |
|---|---|---|---|
| Channel metadata (name, type) | `fetch_channel_info` (`pool.rs:2237-2304`) | `POST /query`, kind `KIND_NIP29_GROUP_METADATA`, `#d = channel uuid` | 3 s + 1 retry after 500 ms |
| Canvas revision | `fetch_canvas_section` (`pool.rs:2307-2363`) | `POST /query`, kind `KIND_CANVAS`, `#h`, `limit 1` | 3 s, no retry |
| Thread context | `fetch_thread_context` (`pool.rs:2676-2738`) | two filters: root by id, replies by `#e` + `#h`, kinds `KIND_STREAM_MESSAGE`/`_V2` | 3 s + 1 retry |
| DM context | `fetch_dm_context` (`pool.rs:2741-2784`) | `#h` + stream kinds, `limit` | 3 s + 1 retry |
| Author profiles | `fetch_prompt_profile_lookup` (`pool.rs:2632-2673`) | kind `Metadata` by authors | 3 s + 1 retry |
| Core memory (NIP-AE) | `engram_fetch::build_core_section` (`pool.rs:1387-1391`) | delegated | 3 s |
| Turn metric (NIP-AM) | `publish_agent_turn_metric` (`pool.rs:3416`) | `POST /events`, kind `44200` | 3 s |
| Reactions add/remove | `reaction_add` / `reaction_remove` (`pool.rs:3462`, `:3540`) | `POST /events` kind 7 / kind 5, plus a kind-7 lookup query | 500 ms add, 1 s query + 1 s delete |
| Dead-letter notice | `post_failure_notice` (`pool.rs:3495-3535`) | `POST /events` kind 9 | 5 s |

`fetch_with_retry` (`pool.rs:2219-2235`) is the shared one-retry wrapper: the closure is called at most twice with a 500 ms gap, so worst-case per fetch is ~6.5 s (`pool.rs:776-779`). Several of these run serially inside a single turn (channel info → context → profiles), so pre-prompt latency compounds.

Relay responses are not trusted blindly on the canvas path: `canvas_section_from_query_response` (`pool.rs:2366-2477`) deserializes a full `nostr::Event`, calls `event.verify()` (`pool.rs:2396-2405`), checks the kind is `KIND_CANVAS` (`:2408-2417`), and re-checks the `h` tag "to prevent a misbehaving relay from injecting a different channel's canvas" (`:2419-2432`). The other fetch paths (`fetch_channel_info`, thread/DM context, profiles) parse raw JSON fields with no signature verification (`pool.rs:2258-2287`, `:2892-2936`, `:2594-2630`).

#### `buzz-core`

| Item | Use | Site |
|---|---|---|
| `kind::KIND_NIP29_GROUP_METADATA` | channel-info filter | `pool.rs:2246` |
| `kind::KIND_CANVAS` | canvas filter + validation | `pool.rs:2312`, `:2400` |
| `kind::KIND_STREAM_MESSAGE`, `KIND_STREAM_MESSAGE_V2` | thread/DM filters | `pool.rs:2703-2706`, `:2751-2754` |
| `kind::KIND_AGENT_TURN_METRIC` | NIP-AM event | `pool.rs:3395` |
| `agent_turn_metric::{AgentTurnMetricPayload, TokenCounts, StopReason, encrypt_agent_turn_metric}` | metric build + encrypt | `pool.rs:3327`, `:3372-3390` |

`acp_stop_to_core` (`pool.rs:3305-3320`) maps ACP stop reasons onto the core enum, collapsing `MaxTurnRequests` and `Refusal` to `Unknown` — a lossy mapping that is asserted in `test_acp_stop_to_core_maps_all_variants` (`pool.rs:5098`).

#### `buzz-sdk` and `nostr`

`buzz_sdk::build_reaction` / `build_remove_reaction` / `build_message` / `ThreadRef` for the cosmetic and dead-letter events (`pool.rs:3470`, `:3595`, `:3514`). Signing always uses `rest.keys` for reactions/notices (`pool.rs:3477`, `:3521`, `:3602`) and `ctx.agent_keys` for the NIP-AM metric (`pool.rs:3402`). `nostr::EventBuilder`/`Tag::parse` build the metric event with `p` and `agent` tags (`pool.rs:3394-3400`).

#### Internal crate modules

`crate::acp` (client, `AcpError`, `McpServer`, `ModelSwitchMethod`, `StopReason`, catalog helpers) at `pool.rs:31-35`; `crate::config::{DedupMode, PermissionMode}` at `:36`; `crate::observer` at `:37`; `crate::queue` (batch, context, framing, `parse_thread_tags`, `format_prompt`, `base_section`, `slash_command_for_batch`) at `:38-41`; `crate::relay::{ChannelInfo, RestClient}` at `:42`; `crate::engram_fetch` at `:1387`. `pool_lifecycle.rs` imports nothing from the crate — only `std::time::Duration` and `tokio::time::Instant` (`pool_lifecycle.rs:7-8`), which is what makes it independently testable.

#### OS process APIs

None are called from `pool.rs` or `pool_lifecycle.rs`. All process control is in `acp.rs`, invoked by `lib.rs`:

- `tokio::process::Command::new(command)` with `.args(args)`, `stdin`/`stdout` piped, **`stderr` inherited**, `.kill_on_drop(true)` (`acp.rs:416-423`).
- `cmd.process_group(0)` on Unix so a group kill does not reach the harness's own group (`acp.rs:463-466`).
- `configure_no_window` for Windows console suppression (`acp.rs:469`, impl `acp.rs:1997`).
- `shutdown()`: `kill_process_group(pid)` when available, else `child.start_kill()`, then a 5 s bounded `child.wait()`; on expiry it logs and abandons the child (`acp.rs:372-397`).
- `Drop`: `start_kill()` only, no reap (`acp.rs:373`, `:1961`) — documented as best-effort.
- `nix` with the `signal` feature is a Unix-only dependency for the group kill (`Cargo.toml:56-58`).

The pool's contribution to process lifecycle is indirect: it decides *when* an agent is poisoned (via `PromptOutcome`) and hands the `OwnedAgent` back, and `lib.rs` then calls `agent.acp.shutdown()` inside the respawn task (`lib.rs:3672`) or the shutdown sequence (`lib.rs:2628`, `:2645`, `:3700-3712`).

#### Desktop / observer consumers

`session_config_captured` carries `configOptions`, `modes`, `models`, `modelOverridden`, and `relayUrl` so the desktop can key its cache by `(agent, relay)` (`pool.rs:906-922`). `AgentModelCapabilities` is documented as feeding the desktop's `get_agent_models` Tauri command (`pool.rs:70-73`); that command is a separate out-of-process Tauri handler (`desktop/src-tauri/src/commands/agent_models.rs:29`) and cannot read this struct — the in-process reader is `switch_idle_agent_model` (`pool.rs:748-756`).


## Module: buzz-acp — work queue, filtering & usage accounting (`crates/buzz-acp/src`)
### Aspect: Integrations

#### External crates

| Crate | Used for | Sites |
|---|---|---|
| `nostr` | `Event`, `Kind`, `Timestamp`, `EventId`, `PublicKey`, `ToBech32` | `queue.rs:16`; `event.kind.as_u16()` (`queue.rs:1089`, `filter.rs:47`, `:383`), `pubkey.to_hex()` (`queue.rs:1082`), `pubkey.to_bech32()` (`queue.rs:1083`), `event.id.to_hex()` (`queue.rs:1090`, `:629`, `:677`, `:709`, `:740`, `:746`), `tag.as_slice()` (`queue.rs:855`, `:1112`, `filter.rs:392`) |
| `uuid` | channel identity throughout | `queue.rs:19`, `filter.rs:44` |
| `evalexpr` | filter AST + evaluation (`Node`, `HashMapContext`, `Function`, `Value`) | `filter.rs:106`, `:197-262`, `:264-338` |
| `tokio` | `spawn_blocking`, `time::timeout`, `sync::Semaphore` | `filter.rs:182-183`, `:220-249` |
| `serde` / `serde_json` | `SubscriptionRule`/`ChannelScope` TOML deserialization; usage wire JSON; tag serialization inside prompts | `filter.rs:55`, `:82`; `usage.rs:57-93`; `queue.rs:1113` |
| `chrono` | RFC-3339 rendering of `created_at` in `[Event]` blocks | `queue.rs:1085-1087` |
| `thiserror` | `FilterError` | `filter.rs:15` |
| `tracing` | all logging | `queue.rs` (17 macro sites), `filter.rs:12` |

`buzz-core` is **not** imported by any of the three files. Kind constants (`KIND_STREAM_MESSAGE = 9`, `KIND_WORKFLOW_APPROVAL_REQUESTED = 46010`, `KIND_STREAM_REMINDER = 40007` — `buzz-core/src/kind.rs:343`, `:442`, `:355`) are pulled in by the *callers* that build `SubscriptionRule`s: `lib.rs:1447-1451`, `config.rs:1157-1162`, `config.rs:1241-1243`, `config.rs:1262-1268`, `setup_mode.rs:521-565`. `filter.rs` only ever sees raw `u32` kinds, so it has no knowledge of the Buzz kind registry.

#### Consumers inside `buzz-acp`

**`lib.rs` (orchestrator, 6,570 lines)**

| Interaction | Line |
|---|---|
| `use queue::{CancelReason, EventQueue, FlushBatch, QueuedEvent, ThreadTags}` | `lib.rs:44` |
| `use filter::SubscriptionRule` | `lib.rs:36` |
| `pub use usage::TurnUsage` (crate's only public re-export) | `lib.rs:15` |
| Builds `SubscriptionRule`s from `SubscribeMode` | `lib.rs:1439-1474` |
| Constructs the queue: `EventQueue::new(dedup_mode).with_in_flight_deadline(config.max_turn_duration_secs)` | `lib.rs:1505-1506` |
| `has_flushable_work()` — lazy-pool wake gate | `lib.rs:1715` |
| `compact_expired_state()` on the 30 s maintenance tick | `lib.rs:1746` (interval `lib.rs:1608`) |
| `has_flushable_work()` — flush retried batches on quiet channels | `lib.rs:1777` |
| `drain_channel()` when the agent loses channel access; returned ids drive 👀 cleanup | `lib.rs:1990` |
| `filter::match_event(...)` — the sole inbound gate after the author gate | `lib.rs:2174` |
| `push(QueuedEvent { channel_id, event, received_at: Instant::now(), prompt_tag })` | `lib.rs:2198` |
| `is_channel_in_flight()` — gates the mid-turn steer/interrupt fork | `lib.rs:2219` |
| `has_flushable_work()` — post-event dispatch | `lib.rs:2288` |
| `extend_in_flight_deadline` / `remove_event` / `release_native_steer` in the `SteerAck` arm | `lib.rs:2509`, `:2512`, `:2515` |
| `native_steer_framing()` + `format_event_block(channel_id, None, &be, None)` to render the steer body | `lib.rs:2824`, `:2831` |
| `mark_native_steer_pending()` before spawning the ack watcher | `lib.rs:2847` |
| `flush_next()` in `dispatch_pending` | `lib.rs:2896` |
| `parse_thread_tags` on the batch's last event → typing-indicator scope | `lib.rs:2904` |
| `pending_channels()` / `requeue_preserve_timestamps` / `mark_complete` on pool exhaustion | `lib.rs:2910-2913` |
| `recoverable_batch` clone gated on `DedupMode::Queue` | `lib.rs:2196-2199` |
| `parse_thread_tags` for the failure-notice thread anchor | `lib.rs:3023` |
| `requeue_as_cancelled` / `requeue` / `mark_complete` in `handle_prompt_result` | `lib.rs:3090`, `:3120`, `:3145`, `:3171` |
| `requeue` + `mark_complete` in panic recovery | `lib.rs:3428`, `:3440` |
| `set_retry_count_for_test` / `queued_event_count` in the `lib.rs` test module | `lib.rs:5733`; `:5507`, `:5612`, `:5808`, `:6231`, `:6316` |

**`pool.rs` (turn execution, 5,620 lines)**

| Interaction | Line |
|---|---|
| `use crate::queue::{CancelReason, ContextMessage, ConversationContext, FlushBatch, PromptChannelInfo, PromptProfile, PromptProfileLookup, ThreadTags}` | `pool.rs:37-41` |
| `base_section(bp)` for the initial/heartbeat prompt and the system-role prompt | `pool.rs:1097`, `:1145`, `:1147` |
| `slash_command_for_batch(b, &known_names)` | `pool.rs:1761` |
| `format_prompt(batch, &FormatPromptArgs { … })` | `pool.rs:1771-1773` |
| Builds `ConversationContext::{Thread, Dm}` from relay history (gated on `context_message_limit > 0`) | `pool.rs:1746`, `:2493-2502`, `:2964-2968` |
| `parse_thread_tags` for reply targets and mention fan-out | `pool.rs:2512`, `:2546` |
| `requeue_batch_if_queue` — drops the batch in `DedupMode::Drop` | `pool.rs:2971-2977` |
| `requeue_cancelled_batch` — maps `ControlSignal` → `CancelReason`, drops on `Cancel`/`Rotate` | `pool.rs:2981-2995` |
| `publish_agent_turn_metric(ctx, usage: Option<TurnUsage>, …)` | `pool.rs:3322-3420` |
| `TaskMeta.recoverable_batch: Option<FlushBatch>` for panic recovery | `pool.rs:45-46` |

**`acp.rs` (ACP client, 3,717 lines)** — the only owner of `UsageTracker`:

| Interaction | Line |
|---|---|
| `use crate::usage::{TurnUsage, UsageTracker}` | `acp.rs:17` |
| field `goose_usage: UsageTracker`, initialized `UsageTracker::default()` | `acp.rs:202`, `:498` |
| `begin_turn(session_id)` immediately before `session/prompt` | `acp.rs:687-690` |
| `handle_goose_usage_update` → `record(&notif.session_id, payload)`; malformed/other variants silently ignored | `acp.rs:1637-1666`; invoked from the read loop at `acp.rs:1141` and `:1488` |
| `take_turn_usage()` → `goose_usage.take()` | `acp.rs:783-785` |

**`config.rs`** — owns `SubscriptionRule` deserialization: `load_rules` reads TOML, enforces limits, and eagerly compiles each `filter` into `Arc<evalexpr::Node>` (`config.rs:1060-1129`). `resolve_channel_filters` (`config.rs:1134`) and `resolve_dynamic_channel_filter` (`config.rs:1236`) translate rules into relay REQ filters; `rule_applies_to_channel` reuses `ChannelScope` (`config.rs:1118`).

**`setup_mode.rs`** — second `filter::match_event` consumer, with its own rule builder (`setup_mode.rs:444-450`, `:521-565`) and its own `parse_thread_tags` call (`setup_mode.rs:605`).

#### Outbound effects

Neither `queue.rs` nor `filter.rs` performs I/O — no network, no filesystem, no database. Their only side channel is `tracing`. `usage.rs` likewise performs no I/O; the relay write happens in `pool.rs`, which encrypts the `AgentTurnMetricPayload` with `buzz_core::agent_turn_metric::encrypt_agent_turn_metric` and publishes kind 44200 tagged `["p", owner_hex]` and `["agent", agent_hex]` (`pool.rs:3376-3401`). `StopReason` is mapped from ACP to NIP-AM at `pool.rs:3305-3314`.

#### Relay-shape coupling carried in these files

- `flush_next`'s chronological re-sort exists because relay replay is `ORDER BY created_at DESC` (`queue.rs:346-350`).
- `parse_thread_tags` supports only marker-form NIP-10 `e` tags, justified by a cross-repo reference to `relay messages.rs:762-783` (`queue.rs:846-848`) — a stale pointer into `buzz-relay` that this crate cannot verify.
- `[Context]` hints hard-code `buzz` CLI invocations (`buzz messages thread --channel <UUID> --event <ID>`, `buzz messages get --channel <UUID>`) — an undeclared coupling to `buzz-cli`'s command surface with no compile-time check (`queue.rs:1253-1259`, `:1282-1284`, `:1307`).
- `[Context]` reply instructions hard-code `buzz messages send --reply-to <id>` (`queue.rs:1150-1180`).


## Module: buzz-acp — ACP protocol, config & setup mode (`crates/buzz-acp/src`)
### Aspect: Integrations

#### The agent subprocess contract

`AcpClient::spawn` (`acp.rs:408-497`) is the whole contract:

| Aspect | Value | Line |
|---|---|---|
| Launcher | `tokio::process::Command::new(command)` — no shell, argv passed via `.args(args)` | `acp.rs:416-417` |
| stdin | `Stdio::piped()` | `acp.rs:418` |
| stdout | `Stdio::piped()` | `acp.rs:419` |
| stderr | **`Stdio::inherit()`** — "so agent logs are visible in the harness terminal" | `acp.rs:420-421` |
| Drop behaviour | `.kill_on_drop(true)`, with a comment that callers must still `await shutdown()` | `acp.rs:422-424` |
| Process group | `cmd.process_group(0)` under `#[cfg(unix)]` — PID == PGID | `acp.rs:463-467` |
| Windows | `configure_no_window(&mut cmd)` sets `CREATE_NO_WINDOW` (0x0800_0000) | `acp.rs:469-471`, impl `acp.rs:1997-2006` |
| Working directory | **not set** — the child inherits the harness's cwd; the ACP `cwd` is only a `session/new` parameter | `acp.rs:416-471`, `acp.rs:566` |
| Missing pipes | `AcpError::Protocol("failed to open agent stdin"/"…stdout")` | `acp.rs:475-483` |

The command is resolved by the OS through the inherited `PATH`. The shipped default is the bare name `goose` — `config.rs:250` for the harness and `config.rs:191` for the auxiliary subcommands.

Five spawn call sites, differing only in whether persona env and the Codex signal are passed:

| Site | Args | Purpose |
|---|---|---|
| `lib.rs:3749` | full `extra_env`, real signal | pool agent spawn |
| `lib.rs:3856` | `command, args, extra_env, has_generated_codex_config` | shared spawn+initialize helper |
| `lib.rs:3887` | `&[], false` | `spawn_auth_client` for auth subcommands |
| `lib.rs:4017` | `&[], false` | `run_models` |
| `pool.rs:4983`, `pool.rs:5041` | test spawns | steer-invariant tests |
| `lib.rs:5178` | `AcpClient::spawn("cat", …)` | test double — `cat` echoes stdin back as stdout |

#### MCP server declaration — the credential channel

`build_mcp_servers` (`lib.rs:4141-4180`) is the only producer of `Vec<McpServer>`:

- Returns an **empty vec** when `config.mcp_command` is empty (`lib.rs:4142-4144`), which is the default (`config.rs:261`) — a stock run gives the agent zero MCP servers.
- Server `name` is derived from the command's file stem, defaulting to `"mcp"` (`lib.rs:4146-4150`); `args` is always empty (`lib.rs:4152`).
- `env` always carries `BUZZ_RELAY_URL` and `BUZZ_PRIVATE_KEY`, the latter as the **raw bech32 `nsec1…`** from `config.keys.secret_key().to_bech32()` with an `.expect(...)` (`lib.rs:4155-4170`).
- `BUZZ_AUTH_TAG` is forwarded from the harness's own environment when non-empty (`lib.rs:4171-4179`).

Flow to the wire: `pool.rs:832` clones the vec into each `session_new_full` call → `acp.rs:568-571` serializes it as `params.mcpServers` → `write_ndjson` (`acp.rs:951`) puts it on the child's stdin. `McpServer.mcp_servers` is stored on the per-agent context at `pool.rs:483`.

#### `lib.rs` / `pool.rs` call sites into this module

| Consumer | Calls | Line |
|---|---|---|
| Startup | `config::propagate_legacy_env_vars()` | `lib.rs:1234` |
| Startup | `setup_mode::SetupPayload::from_env()` then `run_setup_listener` | `lib.rs:1290-1295` |
| Startup | `config.summary()` for the boot log | `lib.rs:1296` |
| Startup | base-prompt selection from `config.no_base_prompt` / `base_prompt_content` / `include_str!` | `lib.rs:1529-1545` |
| Subcommands | `ModelsArgs::parse_from` → `run_models` | `lib.rs:1252-1253` |
| Subcommands | `AuthMethodsArgs::parse_from` → `run_auth_methods` | `lib.rs:1262-1263` |
| Subcommands | `AuthenticateArgs::parse_from` → `run_authenticate`, which calls `client.authenticate(&args.method_id)` under an `AUTHENTICATE_TIMEOUT` | `lib.rs:1272-1273`, `lib.rs:3983` |
| Spawn | `config.persona_env_vars.clone()` as `extra_env` | `lib.rs:1762`, `lib.rs:3488`, `lib.rs:3666`, `lib.rs:3733` |
| Handshake | `init_result["protocolVersion"].as_u64().unwrap_or(1) as u32` | `lib.rs:3776`, `lib.rs:3864` |
| Session | `session_new_full(&ctx.cwd, ctx.mcp_servers.clone(), session_new_system_prompt(...))` | `pool.rs:830-836` |
| Session | `session_set_goose_system_prompt`, gated on `goose_system_prompt_supported != Some(false)` | `pool.rs:838-846` |
| Session | `-32601` latch: `Err(AcpError::AgentError { code: -32601, .. })` marks the goose system-prompt method unsupported | `pool.rs:849`, documented `pool.rs:341` |
| Turn | `install_steer_rx` / `clear_steer_rx` | `pool.rs:5006`, `pool.rs:5032`, `pool.rs:5089`; cleared at `pool.rs:1243` |
| Filters | `config::resolve_channel_filters`, `resolve_dynamic_channel_filter`, `load_rules` | `setup_mode.rs:375`, `setup_mode.rs:578`, `setup_mode.rs:530` |

`setup_mode` reaches back into `lib.rs` for shared gates: `crate::resolve_agent_owner` (`setup_mode.rs:346`, defined `lib.rs:123`), `crate::OwnerCache::new` (`setup_mode.rs:347`), `crate::author_allowed` (`setup_mode.rs:433`), `crate::event_mentions_agent` (`setup_mode.rs:424`), `crate::is_dm_channel` (`setup_mode.rs:432`), `crate::queue::parse_thread_tags` (`setup_mode.rs:604`), and `crate::pool::ChannelInfoResolver::new` (`setup_mode.rs:383`).

#### `CODEX_CONFIG` — the Codex adapter integration

Two-part mechanism spanning both files:

- `config.rs:646-677` generates the env pair for normalized `codex` / `codex-acp` commands only, and only when the relay URL parses with a host. The doc comment (`config.rs:628-645`) explains the reason: Codex sandboxes MCP subprocesses (including `buzz-cli`) behind a macOS Seatbelt policy that blocks all outbound network, so `buzz-cli` cannot reach the relay WebSocket without it. The value is forwarded by the `@agentclientprotocol/codex-acp` adapter (1.x) as a session-level config override (`CODEX_CONFIG` → `thread/start config`), equivalent to the TOML `sandbox_workspace_write.network_access = true`, which sets `NetworkSandboxPolicy::Enabled` and emits `(allow network-outbound)` in the Seatbelt policy — full outbound TCP/TLS at the OS level.
- `acp.rs:257-345` merges persona entries, additional generated entries, and the parent process's own `CODEX_CONFIG` (parent wins on collisions at every nesting level), then **force-sets `sandbox_workspace_write.network_access = true`** as the final step (`acp.rs:329-343`). That single key is the only thing forced; everything else in the object is whatever the merge produced.

#### Goose-specific integrations

| Integration | Wire surface | Line |
|---|---|---|
| Custom notification opt-in | `_meta.goose.customNotifications = true` in `initialize` capabilities; without it goose suppresses the notification and no usage data arrives | `acp.rs:357-360` |
| Usage notifications | `_goose/unstable/session/update`, deserialized into `GooseSessionUpdateNotification` | `acp.rs:1637-1678` |
| System-prompt append | `_goose/unstable/session/system-prompt/set` with `mode:"append"`, `key:"buzz"` | `acp.rs:605-619` |
| Active-run tracking | `session_info_update` → `params.update._meta.goose.activeRunId`; the doc comment cites goose's own `crates/goose/src/acp/server.rs:2277 send_active_run_update` | `acp.rs:176-186`, `acp.rs:1590-1620` |
| Non-cancelling steer | `_goose/unstable/session/steer` with `expectedRunId` | `acp.rs:1338-1355` |

`acp.rs:1594-1597` notes that buzz-agent also emits `session_info_update` with the same field, and that other agents leave `active_run_id` as `None` so steer callers fall back to cancel+merge. The `_meta` position is documented as being on the update object itself (`params.update._meta`), not at params level (`acp.rs:1600-1604`).

A third non-standard capability, `_meta["terminal-auth"] = true`, is described as a claude-agent-acp extension for advertising the terminal login argv, relying on other adapters ignoring unknown `_meta` keys (`acp.rs:361-366`).

#### External crate dependencies exercised by this group

| Crate | Use | Line |
|---|---|---|
| `tokio` (`process`, `time`, `io`, `sync`) | subprocess, deadlines, `select!`, oneshot/mpsc | `acp.rs:13-14`, `acp.rs:1276-1360` |
| `tokio-util` `LinesCodec` / `FramedRead` | bounded NDJSON framing | `acp.rs:15`, `acp.rs:487` |
| `futures-util` `StreamExt` | `reader.next()` | `acp.rs:12` |
| `serde_json` | all wire construction and parsing | throughout |
| `thiserror` | `AcpError` (`acp.rs:78`), `ConfigError` (`config.rs:38`) |
| `nix` (`signal` feature, `default-features = false`) | `killpg` — **Unix-only target dependency** | `Cargo.toml:76-77`, used `acp.rs:1979-1986` |
| `clap` v4 (`derive`, `env`) | the whole CLI surface | `Cargo.toml:66` |
| `toml` 1.0 | `load_rules` | `Cargo.toml:69`, `config.rs:1065` |
| `evalexpr` | rule filter compilation at load time | `Cargo.toml:72`, `config.rs:1110` |
| `url` | relay-URL validation in `codex_network_env` | `config.rs:13`, `config.rs:654` |
| `uuid` | channel ids in filters | `config.rs:15` |
| `nostr` | `Keys`, `EventId`, `Tag`, event building | `config.rs:11`, `setup_mode.rs:40` |
| `anyhow` | setup-mode error type | `setup_mode.rs:36` |

`Cargo.toml:76-77` carries a comment pointing at the `#[cfg(not(unix))]` fallback in `acp.rs`, so the Unix-only scoping is deliberate. The `nix` dependency is pinned to `0.31` and the `signal` feature is chosen specifically so `killpg` can be called without `unsafe` (`acp.rs:1975-1977`).

#### Relay-side integration in setup mode

`run_setup_listener` uses `HarnessRelay` and `RelayEventPublisher` from `relay.rs` (`setup_mode.rs:73-77`):

| Step | Call | Line |
|---|---|---|
| Connect | `HarnessRelay::connect(&config.relay_url, &config.keys, &pubkey_hex, relay_auth_tag)` | `setup_mode.rs:330-333` |
| Watermark | `relay.set_startup_watermark(startup_watermark)` — failure warns only | `setup_mode.rs:334-336` |
| Membership | `relay.subscribe_membership_notifications()` — failure is fatal | `setup_mode.rs:338-341` |
| Discovery | `relay.discover_channels()` (REST) — failure is fatal | `setup_mode.rs:351-354` |
| Channel subs | `relay.subscribe_channel(*channel_id, filter.clone())` — per-channel failure warns only | `setup_mode.rs:379-382` |
| Publisher | `relay.event_publisher()`, `relay.rest_client()` | `setup_mode.rs:383-384` |
| Reconnect | `relay.reconnect()` when the event stream ends | `setup_mode.rs:394` |
| Publish | `buzz_sdk::build_message(...)` → `sign_with_keys(keys)` → `publisher.publish_event(signed)` | `setup_mode.rs:626-644` |

`BUZZ_AUTH_TAG` is parsed via `buzz_sdk::nip_oa::parse_auth_tag` and silently dropped when empty or unparseable (`setup_mode.rs:319-322`).


## Module: buzz-agent — core loop, ACP wire types & handoff (`crates/buzz-agent/src`)
### Aspect: Integrations
#### The ACP client (stdio, both directions)
Inbound frames are read from `tokio::io::stdin()` wrapped in `BufReader` (`lib.rs:171`) by `read_bounded_line` (`wire.rs:174-218`); outbound frames go through a single `mpsc::Sender<WireMsg>` with capacity 64 (`lib.rs:164`) into `writer_task`, the only writer of `tokio::io::stdout()` (`wire.rs:220-237`). All logging goes to stderr with ANSI disabled (`lib.rs:156-159`), keeping the stdout channel pure.

Failure behavior at the seam: if stdout write fails, `writer_task` returns (`wire.rs:232-234`) and every later `wire::send` silently no-ops because the result is discarded (`wire.rs:170-172`). The agent keeps looping without any output.

#### buzz-acp (the primary client)
This group is written against buzz-acp's expectations, and several contracts are cross-crate:

| Contract | buzz-agent side | buzz-acp side |
|---|---|---|
| `usage_update` on `_goose/unstable/session/update`, before the prompt response | `lib.rs:730-750` | `UsageTracker` (`crates/buzz-acp/src/usage.rs:164`), payload shape documented at `usage.rs:47` |
| `keepalive` resets the idle clock | `agent.rs:134-144` | classified as non-content (`crates/buzz-acp/src/acp.rs:1623`), test `keepalive_resets_idle_past_deadline` (`acp.rs:2870`) |
| `_meta.goose.activeRunId` seeds the steer target | `lib.rs:661-670` | cached in `AcpClient::active_run_id` (`acp.rs:189`, accessor `acp.rs:769`), used as `expectedRunId` (`acp.rs:1293-1313`) |
| `stopReason` strings | `types.rs:183-191` | parsed at `acp.rs:61-72` |
| `agent_thought_chunk` / `agent_message_chunk` | `agent.rs:185`, `agent.rs:199` | handled at `acp.rs:1566` and `acp.rs:1535` |
| `systemPrompt` on `session/new` (protocol ≥ 2) | consumed `lib.rs:361-370` | `[Base]` composition gated on version (`crates/buzz-acp/src/pool.rs:181`, `crates/buzz-acp/src/lib.rs:4191-4210`) |

Contract drift found: `crates/buzz-acp/src/acp.rs:185-186` documents that "goose/buzz-agent emit `activeRunId: null` at end of turn". buzz-agent does not — `grep -n 'activeRunId' src/*.rs` shows exactly one emission site (`lib.rs:665`), at prompt start; end-of-turn only clears the field internally (`lib.rs:707`). The client's cached run id therefore goes stale, and staleness is caught only by buzz-agent's own mismatch rejection (`lib.rs:588-598`).

Observability side effect: buzz-acp forwards every frame it reads from the agent verbatim to its observer as `acp_read` (`crates/buzz-acp/src/acp.rs:1120`, `acp.rs:1414`). Everything this group writes — including `rawInput` (the full tool arguments, `agent.rs:412`) and completed tool result text (`agent.rs:592`) — is therefore republished; the desktop transcript reads `update.rawInput` directly (`desktop/src/features/agents/ui/agentSessionTranscriptHelpers.ts:361`).

#### Sibling modules inside the crate
| Dependency | Used for | Call sites |
|---|---|---|
| `config` | `Config::from_env` and every tunable; `PROTOCOL_VERSION`, `MAX_*` constants | `lib.rs:41`, `lib.rs:160`, `agent.rs:8` |
| `llm` | `Llm::new`, `complete`, `summarize` | `lib.rs:161`, `agent.rs:124`, `handoff.rs:49-54` |
| `mcp` | `McpRegistry::spawn_all`, `tools`, `has`, `is_hook`, `call`, `call_hooks`, `server_of`, `kill_server`, `ResultBudget` | `lib.rs:390`, `agent.rs:115`, `agent.rs:224-231`, `agent.rs:330-337`, `agent.rs:509-537`, `handoff.rs:70-77`, `handoff.rs:177` |
| `builtin` | `LOAD_SKILL_TOOL`, `load_skill_def`, `call_load_skill` | `agent.rs:118`, `agent.rs:316-323` |
| `hints` | `build_hints_section`, `SkillEntry` | `lib.rs:356-359` |
| `catalog` | `discover_databricks_models`, `discovery_failure_fallback`, `ModelEntry` | `lib.rs:448-452`, `lib.rs:324` |
| `auth` | `PkceOAuthConfig`, `PkceOAuthTokenSource::interactive_login` | `lib.rs:136-142` |

#### External crates touched directly by this group
| Crate | Use | Site |
|---|---|---|
| `tokio` | runtime, stdio, `Mutex`, `watch`, `mpsc`, `Semaphore`, `JoinSet`, `OnceCell`, `interval`, `timeout_at` | `lib.rs:110-120`, `lib.rs:164`, `agent.rs:11-12`, `agent.rs:455` |
| `serde` / `serde_json` | all params deserialization and every outbound frame | `wire.rs:1-5`, `types.rs:1-2` |
| `getrandom` | 8-byte session/run/steer tokens | `lib.rs:821-825` |
| `tracing` / `tracing_subscriber` | stderr logging | `lib.rs:156-159` and throughout |

#### Cross-crate consumers of this group
- `sprig` multicall binary dispatches `argv[0] == "buzz-agent"` straight to `buzz_agent::run()` (`crates/sprig/src/main.rs:18`), so the argv contract in `run()` (`lib.rs:110-121`) is shared.
- The desktop Tauri backend links the crate as `buzz_agent_pkg` (`desktop/src-tauri/Cargo.toml:91`) purely to read `WINDOWS_SHELL_RESOLUTION_ENV` (`desktop/src-tauri/src/managed_agents/git_bash.rs:136`, `:438`) — a shared-constant integration, deliberately sourced from this crate rather than copied.

#### Subprocess and filesystem surface
This group spawns no processes itself; MCP children are created by `McpRegistry::spawn_all` with the client-supplied `cwd` and env (`lib.rs:390`). Its only filesystem interaction is indirect: hints/skills discovery under the session `cwd` (`lib.rs:356-359`) and the OAuth token cache written by `auth` (`lib.rs:140-144`). No direct file reads or writes exist here (`grep -c 'fs::write'`, `grep -c 'File::create'`, `grep -c 'std::fs'` → 0 across all six files).

#### Copied-rather-than-shared behavior
- The goose wire extensions (`_goose/unstable/session/steer`, `_goose/unstable/session/update`, `_meta.goose.*`) are reimplemented from goose's contract, documented in comments rather than by a shared type (`lib.rs:544-553`, `wire.rs:69-74`, `wire.rs:143-149`). The upstream reference is cited only from the client side (`crates/buzz-acp/src/acp.rs:178-180`).
- Three independent string-truncation helpers coexist: `types::clamp` (`types.rs:259-274`, used only by `mcp.rs:254` and `mcp.rs:630`), `handoff::clamp_bytes` (`handoff.rs:300-315`), and `mcp::truncate_at_boundary` / `mcp::truncate_middle` (`mcp.rs:866`, `mcp.rs:886`). `clamp` appends `\n[truncated]`, `clamp_bytes` appends `…` — same job, different markers, no shared implementation.


## Module: buzz-agent — LLM providers & configuration (`crates/buzz-agent/src`)
### Aspect: Integrations
`llm.rs` is the crate's only outbound-HTTP surface for model traffic. `config.rs` makes zero network calls — grep for `reqwest`, `Client`, `http`, and `await` in `config.rs` returned zero matches; its only I/O is one `std::fs::read_to_string` for `BUZZ_AGENT_SYSTEM_PROMPT_FILE` (`config.rs:796`).

#### Outbound HTTP: endpoints and auth headers
| Provider / route | Endpoint | Auth header | Extra headers | Site |
|---|---|---|---|---|
| Anthropic Messages | `POST {ANTHROPIC_BASE_URL}/v1/messages` | `x-api-key: {cfg.api_key}` | `anthropic-version: {cfg.anthropic_api_version}` | `llm.rs:330-338` |
| OpenAI Responses | `POST {OPENAI_COMPAT_BASE_URL}/responses` | `Authorization: Bearer {token}` via `bearer_auth` | — | `llm.rs:551`, `llm.rs:638` |
| OpenAI Chat Completions | `POST {OPENAI_COMPAT_BASE_URL}/chat/completions` | `Bearer` | — | `llm.rs:558`, `llm.rs:638` |
| **Mesh model catalog** | `GET {OPENAI_COMPAT_BASE_URL}/models` | `Bearer` via `bearer_auth` | — | `llm.rs:473`, `llm.rs:485-491` |
| Databricks legacy serving | `POST {DATABRICKS_HOST}/serving-endpoints/{model}/invocations` | `Bearer` | — | `llm.rs:608-616` |
| DatabricksV2 gateway (OpenAI) | `POST {DATABRICKS_HOST}/ai-gateway/openai/v1/responses` | `Bearer` | — | `llm.rs:985` |
| DatabricksV2 gateway (Anthropic) | `POST {DATABRICKS_HOST}/ai-gateway/anthropic/v1/messages` | `Bearer` | — | `llm.rs:986` |
| DatabricksV2 gateway (MLflow) | `POST {DATABRICKS_HOST}/ai-gateway/mlflow/v1/chat/completions` | `Bearer` | — | `llm.rs:987` |
| Databricks OIDC discovery (indirect) | `{DATABRICKS_HOST}/oidc/.well-known/oauth-authorization-server` | n/a | — | URL built at `llm.rs:1538-1541`, fetched inside `auth.rs` |

Every **POST** carries `content-type: application/json` and a pre-serialized body, applied uniformly in `post` (`llm.rs:1404-1408`). Body bytes are serialized **once** before the retry loop (`llm.rs:1400-1401`) and cloned per attempt (`llm.rs:1407`), so a serialization failure can never be mistaken for a retryable transport error — the rationale is stated at `llm.rs:1309-1310`.

The mesh catalog `GET` (`llm.rs:485-491`) does **not** go through `post` and therefore shares none of that machinery: no `content-type`, no retry loop, no `MAX_LLM_RESPONSE_BYTES` cap, no `terminal_llm_error` formatting, and no 401/403 refresh. It sets its own per-request `.timeout(MESH_AUTO_CATALOG_TIMEOUT)` (`llm.rs:488`) and decodes with `response.json::<Value>()` (`llm.rs:508`) rather than the bounded chunk loop `post` uses.

#### The mesh relay integration
This is a new *upstream* integration, not a new dependency. The counterpart is the mesh-llm router that Buzz's desktop app runs locally; `desktop/src-tauri/src/managed_agents/relay_mesh.rs` is the code that wires the two together, and it is the only production caller that turns the feature on:

| What the desktop provider sets | Value | Site |
|---|---|---|
| `BUZZ_AGENT_PROVIDER` | `openai` | `desktop/src-tauri/src/managed_agents/relay_mesh.rs:27` |
| `OPENAI_COMPAT_BASE_URL` | `http://127.0.0.1:9337/v1` (`RELAY_MESH_API_BASE_URL`, `desktop/src-tauri/src/managed_agents/relay_mesh.rs:1`) | `desktop/src-tauri/src/managed_agents/relay_mesh.rs:29-32` |
| `OPENAI_COMPAT_API_KEY` | `buzz-mesh-local` placeholder (`desktop/src-tauri/src/managed_agents/relay_mesh.rs:2`) | `desktop/src-tauri/src/managed_agents/relay_mesh.rs:33-36` |
| `OPENAI_COMPAT_API` | `chat` — pins Chat Completions, so the mesh path never auto-upgrades to `/responses` | `desktop/src-tauri/src/managed_agents/relay_mesh.rs:37` |
| `BUZZ_AGENT_MODEL` / `OPENAI_COMPAT_MODEL` | the requested model, defaulting to `auto` (`RELAY_MESH_AUTO_MODEL_ID`, `desktop/src-tauri/src/managed_agents/relay_mesh.rs:4`) | `desktop/src-tauri/src/managed_agents/relay_mesh.rs:27-33` |
| `BUZZ_AGENT_PREFER_MESH_FOR_AUTO` | `"1"` | `desktop/src-tauri/src/managed_agents/relay_mesh.rs:41-44`, constant `desktop/src-tauri/src/managed_agents/relay_mesh.rs:6` |
| `BUZZ_AGENT_MAX_OUTPUT_TOKENS` | `4096` — small local-model context windows | `desktop/src-tauri/src/managed_agents/relay_mesh.rs:48-51` |
| `BUZZ_AGENT_THINKING_EFFORT` | `none` | `desktop/src-tauri/src/managed_agents/relay_mesh.rs:52` |

The whole desktop side is behind `#[cfg(feature = "mesh-llm")]` (`desktop/src-tauri/src/managed_agents/relay_mesh.rs:5`, `desktop/src-tauri/src/managed_agents/relay_mesh.rs:11`), and its one test asserts the three tuning values including the new flag (`desktop/src-tauri/src/managed_agents/relay_mesh.rs:62-83`). Two integration contracts are worth recording because neither is enforced by a type or a shared constant:
1. **The virtual model names.** `MESH_VIRTUAL_MODEL_ID = "mesh"` (`llm.rs:28`) and `MESH_AUTO_MODEL_ID = "auto"` (`llm.rs:29`) must match the router's catalog ids. The desktop side independently declares `RELAY_MESH_AUTO_MODEL_ID = "auto"` (`desktop/src-tauri/src/managed_agents/relay_mesh.rs:4`); nothing links the two, and `"mesh"` is declared only in `llm.rs`.
2. **The MoA failure vocabulary.** `MESH_MOA_UNAVAILABLE_MESSAGE` (`llm.rs:34`) is the router's 503 message copied verbatim, and `"moa_failure"` (`llm.rs:1386`) is its structured error type. Both are string literals with no upstream reference; grep for either across `crates/` and `desktop/` returns only `llm.rs` and the `llm.rs` tests.

The catalog probe also assumes the router serves an OpenAI-compatible `GET /models` with a `data[].id` array (`llm.rs:48-52`) — the same shape `catalog.rs` uses for model discovery, but reimplemented here rather than shared. `mesh_catalog_supports_collective` is the only parser of that shape in `llm.rs`; `model_catalog` (`llm.rs:1746-1754`) is the test fixture that mints it.

#### TLS handling
The HTTP client is built once per `Llm` in `Llm::new` (`llm.rs:109-113`):
- `connect_timeout(10s)` (`llm.rs:110`)
- `read_timeout(cfg.llm_timeout)` (`llm.rs:111`)
- no `.timeout(...)`, no `danger_accept_invalid_certs`, no custom root store, no proxy configuration, no `.https_only(true)`

TLS comes from the `rustls` feature declared in `crates/buzz-agent/Cargo.toml:32` (`reqwest = { workspace = true, features = ["json", "rustls", "form"] }`). Certificate verification is therefore reqwest/rustls default — grep for `danger_accept_invalid`, `add_root_certificate`, and `use_native_tls` in `llm.rs` returned zero matches.

**Nothing in this group requires HTTPS.** `Config::validate` (`config.rs:878-959`) performs no scheme or host check on `base_url`; the only function that inspects the scheme is `is_openai_host` (`config.rs:1037-1047`), which explicitly accepts `http://` (`config.rs:1040`) and has a test asserting `http://eu.api.openai.com/v1 == true` (`config.rs:1281`). So `ANTHROPIC_BASE_URL=http://…` or `DATABRICKS_HOST=http://…` sends credentials in plaintext with no warning. Since `reqwest` has no `https_only` here, `.build()` will not reject it either. The relay-mesh integration exercises exactly that: it configures a plaintext loopback base URL (`desktop/src-tauri/src/managed_agents/relay_mesh.rs:1`), so the mesh catalog `GET` and every mesh inference POST are cleartext by design.

#### Crate dependencies actually used by these two files
| Crate | Used for | Site |
|---|---|---|
| `reqwest` | `Client`, `RequestBuilder`, `Response`, `Error`, `bearer_auth`, chunked body reads, `StatusCode` | `llm.rs:5`, `llm.rs:109`, `llm.rs:485-491`, `llm.rs:638`, `llm.rs:1268-1284`, `llm.rs:1447` |
| `serde_json` | every request body, every response parse, and both mesh body classifiers | `llm.rs:6`; `llm.rs:1365`, `llm.rs:1378`; `config.rs:129` (`use serde_json::json` inside `anthropic_thinking_config`) |
| `tokio` | `tokio::time::sleep` in backoff; **new:** `tokio::sync::Mutex` and `tokio::time::Instant` for the mesh state | `llm.rs:7-8`, `llm.rs:1301`, `llm.rs:98`, `llm.rs:398` |
| `tracing` | six `warn!` and six `debug!` sites in `llm.rs`, five `warn!` in `config.rs` | warn `llm.rs:364`, `llm.rs:382`, `llm.rs:666`, `llm.rs:1325`, `llm.rs:1415`, `llm.rs:1454`; debug `llm.rs:464`, `llm.rs:477`, `llm.rs:494`, `llm.rs:502`, `llm.rs:511`, `llm.rs:523`; `config.rs:149`, `config.rs:219`, `config.rs:512`, `config.rs:542`, `config.rs:566` |
| `getrandom` | jitter entropy (`getrandom::fill`) | `llm.rs:1295` |
| `std::sync::atomic` | the `auto_upgraded` latch | `llm.rs:2`, `llm.rs:117`, `llm.rs:544`, `llm.rs:665` |
| `std::collections::BTreeSet` | **new:** deduplicating physical model ids in the catalog probe | `llm.rs:1`, `llm.rs:50`, `llm.rs:60` |

`getrandom` (declared `Cargo.toml:35`) is used in exactly one place in the whole crate — `llm.rs:1295`. If jitter entropy is unavailable, backoff degrades silently to the un-jittered base delay (`llm.rs:1296-1300`).

`16d4ec33` is the commit that made `llm.rs` depend on `tokio`'s `sync` and `time` *types* rather than just `tokio::time::sleep`; both features were already enabled workspace-side for this crate (`Cargo.toml:28`), so no manifest change was needed — which also means nothing in the manifest records the new coupling.

Test-only dependencies pulled in by `llm.rs`'s test module: `tracing_subscriber` (`llm.rs:1577`, `llm.rs:3375-3403`), `async_trait` (`llm.rs:3563`), `tokio::net::TcpListener` + `tokio::io` (`llm.rs:1666-1667`, and again per socket test, e.g. `llm.rs:3125-3126`), and `std::collections::VecDeque` for the new sequenced stub server (`llm.rs:1665`). `config.rs`'s test module uses `serde::Deserialize` (`config.rs:2662`) and `serde_json::from_str` (`config.rs:2677`).

The mesh tests introduced a second, richer HTTP stub alongside the existing hand-rolled responders: `spawn_sequence_stub` (`llm.rs:1662-1745`) binds `127.0.0.1:0`, records every request as a `CapturedHttpRequest` (`llm.rs:1612-1617`) including method, path and parsed JSON body, and replays a queued `Vec<StubHttpResponse>` (`llm.rs:1619-1660`). It is the first stub in this file that can serve a *sequence* and be asserted on afterwards, which is what makes the multi-call hysteresis tests expressible.

#### Intra-crate dependencies
| Import | From | Used for |
|---|---|---|
| `PkceOAuthConfig`, `PkceOAuthTokenSource`, `StaticTokenSource`, `TokenSource` | `auth.rs` | `llm.rs:10`, consumed in `build_token_source` (`llm.rs:1529-1553`), `Llm.auth` (`llm.rs:104`), and now the catalog probe's `bearer()` call (`llm.rs:474`) |
| `is_openai_host`, `normalize_effort_for_anthropic_route`, `normalize_effort_for_openai_route`, `Config`, `OpenAiApi`, `Provider`, `ThinkingEffort` | `config.rs` | `llm.rs:11-14` |
| `crate::config::anthropic_thinking_config` | `config.rs` | called fully-qualified at `llm.rs:734` |
| `AgentError`, `HistoryItem`, `LlmResponse`, `ProviderStop`, `ToolCall`, `ToolDef`, `ToolResultContent` | `types.rs` | `llm.rs:15-17` |

`config.rs` has **no** intra-crate imports at all — it depends only on `std`, `serde_json`, and `tracing`. That makes it the crate's dependency root and explains why the effort tables live there rather than in `llm.rs`. `prefer_mesh_for_auto` (`config.rs:734`) preserved that property: the flag is a bare `bool`, so no mesh type crossed from `llm.rs` into `config.rs`.

Consumers of this group elsewhere in the crate:
| Consumer | What it uses |
|---|---|
| `lib.rs:41` | `Config`, `MAX_SYSTEM_PROMPT_BYTES`, `PROTOCOL_VERSION`; `Config::from_env()` at `lib.rs:160`, `Llm::new` at `lib.rs:161` |
| `agent.rs:8` | `Config`, `MAX_PROMPT_BYTES`, `MAX_TOOL_CALLS_PER_TURN`, `MAX_TOOL_RESULT_BYTES`; `Llm::complete` at `agent.rs:124` |
| `handoff.rs:3` | `HANDOFF_MAX_OUTPUT_TOKENS`, `HANDOFF_MAX_TOOL_NAMES`, `HANDOFF_ORIGINAL_TASK_MAX_BYTES`; `summarize` via `handoff.rs:51`, `handoff.rs:197` |
| `catalog.rs:17` | `llm::build_token_source`, called at `catalog.rs:117` |

No consumer changed. `agent.rs:124` still passes an `effective_model` and gets an `LlmResponse` back; the model substitution is invisible above `Llm::complete` (`llm.rs:123`), which means the ACP session, the desktop UI, and the audit trail all continue to report the *configured* model (`auto`) while the request may have been served by `mesh`. The only record of the substitution is the `debug` log at `llm.rs:464`.

#### Duplicated code that should be a dependency
`lib.rs:129-153` (`auth_subcommand`) re-declares the entire Databricks PKCE configuration that `build_token_source` builds at `llm.rs:1538-1551`, with the identifiers inlined as string literals rather than referencing the constants:

| Value | Canonical definition | Duplicated at |
|---|---|---|
| discovery URL template `{host}/oidc/.well-known/oauth-authorization-server` | `llm.rs:1538-1541` | `lib.rs:135-138` |
| client id `databricks-cli` | `DATABRICKS_CLIENT_ID`, `llm.rs:22` | `lib.rs:141` (bare literal) |
| scopes `all-apis`, `offline_access` | `DATABRICKS_OAUTH_SCOPES`, `llm.rs:23` | `lib.rs:142` (bare literals) |
| cache namespace `databricks` | `llm.rs:1549` | `lib.rs:143` |

Both constants are private module-level `const`s in a private module (`llm.rs:22-23`), so `lib.rs` structurally *cannot* reference them without a visibility change. The result is that `buzz-agent auth databricks` and the runtime token source can drift apart — e.g. a scope added to `DATABRICKS_OAUTH_SCOPES` would not be requested by the interactive login, producing a cached token that the runtime then rejects. No test covers the two staying in sync; grep for `DATABRICKS_OAUTH_SCOPES` across `crates/` returned only `llm.rs:23` and `llm.rs:1545`.

A second, smaller duplication: `strip_catalog_prefix` (`config.rs:89-97`) is re-implemented verbatim inside the test helper at `config.rs:2578-2590` instead of being called.

A third: `Llm::summarize`'s Anthropic body (`llm.rs:240-248`) duplicates the message/system/max_tokens shape that `anthropic_body` builds (`llm.rs:730-731`), and its Chat body (`llm.rs:270-278`) duplicates `openai_body`'s (`llm.rs:828-829`).

A fourth, new one: the OpenAI-compatible `/models` catalog shape is parsed in two places with two different implementations — `mesh_catalog_supports_collective` (`llm.rs:48-52`) reads `data[].id` for the viability test, while `catalog.rs` fetches and parses the same endpoint for model discovery using `build_token_source` (`catalog.rs:117`). Neither shares a type or a helper with the other.

#### Cross-language integration: the desktop effort table
`config.rs` is coupled to TypeScript through a shared JSON fixture. The test at `config.rs:2674-2708` does `include_str!("../../../desktop/src/features/agents/ui/effortTable.fixture.json")` (`config.rs:2675-2676`) — a compile-time path dependency from a Rust crate into the desktop app's source tree. The fixture exists (`desktop/src/features/agents/ui/effortTable.fixture.json`) and the TS side is `desktop/src/features/agents/ui/buzzAgentConfig.ts` with tests `buzzAgentConfig.test.mjs` and `effortTable.fixture.test.mjs`.

Two consequences worth flagging:
1. Moving or renaming that desktop file breaks the Rust build, not just a Rust test — `include_str!` is a compile-time macro inside `#[cfg(test)]`, so it breaks `cargo test -p buzz-agent`.
2. The Rust side of the guard compares the fixture against a **test-only** re-implementation (`valid_effort_values_for_provider_model`, `config.rs:2567-2658`), not against production functions. See Debt and Business Rules for why this can go green while production drifts.

There is now a *second* Rust↔desktop coupling in this group, and it is weaker than the first: `BUZZ_AGENT_PREFER_MESH_FOR_AUTO` is declared as a constant on the desktop side (`desktop/src-tauri/src/managed_agents/relay_mesh.rs:6`) and as a bare string literal on the agent side (`config.rs:807`), with no shared fixture, no `include_str!`, and no test spanning both. Renaming it on either side compiles cleanly and silently disables the feature.

#### Subprocesses
Neither file spawns a subprocess. grep for `Command`, `spawn`, and `std::process` in `llm.rs` and `config.rs` returned zero matches (`tokio::spawn` appears only in the test module, e.g. the sequenced stub server at `llm.rs:1675`). The one process-related fact worth recording for this group: `cfg.api_key` is never handed to a child environment — MCP children are spawned with `env_clear()` plus an explicit whitelist (`mcp.rs:714`, whitelist at `mcp.rs:41-47`), so the provider credential this group reads does not cross into tool subprocesses. `BUZZ_AGENT_PREFER_MESH_FOR_AUTO` travels the other direction — it is injected *into* this process by the desktop launcher (`desktop/src-tauri/src/managed_agents/relay_mesh.rs:41-44`), not exported from it.

#### Declared dependencies not used by this group
`crates/buzz-agent/Cargo.toml` declares `serde_yaml` (line 31), `rmcp` (line 33), `arc-swap` (line 34), `axum` (line 40), `base64` (line 41), `hex` (line 42), `sha2` (line 43), `urlencoding` (line 44), `webbrowser` (line 45), and `nix` (line 48). None of them are referenced from `llm.rs` or `config.rs` — grep for each in these two files returned zero matches. They belong to sibling modules (`auth.rs` for base64/sha2/hex/urlencoding/webbrowser/axum, `mcp.rs` for rmcp/nix, `hints.rs`/`persona` loading for serde_yaml). I did not audit whether any is genuinely unused crate-wide; that is outside this group's scope.


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

`fetch_v1_models` calls `{host}/api/2.0/serving-endpoints` (`catalog.rs:141`); `fetch_v2_models` calls `{host}/api/ai-gateway/v2/endpoints?page_size=100[&page_token=…]` (`catalog.rs:242`, `catalog.rs:248-254`). Both use a fresh `reqwest::Client::new()` created per call (`catalog.rs:120`) with `bearer_auth` (`catalog.rs:144`, `catalog.rs:257`) and no timeout configuration — contrast `llm.rs:53-57`, which sets `connect_timeout(10s)` and `read_timeout(cfg.llm_timeout)`. Re-verified after `8eb6e3eb`: **still no timeout** — `grep -n 'timeout' catalog.rs` returns zero matches, and the request-building code was not touched by that commit.

The v2 query string is built by a hand-rolled `percent_encode` (`catalog.rs:222-233`) justified by a comment that says it "avoids requiring the `query` reqwest feature in buzz-agent's Cargo.toml" (`catalog.rs:246-247`). The crate already depends on `urlencoding = "2"` (`Cargo.toml`) and uses it in `auth.rs:589-594`, so the duplication is avoidable regardless of the reqwest feature question. `reqwest` is declared as workspace `version = "0.13", features = ["json","rustls"]` (root `Cargo.toml:93`) with `features = ["json","rustls","form"]` added by this crate — `query` is not in either list, which is consistent with the comment's premise even though the same job is already covered by an existing dependency. Unchanged by `8eb6e3eb`.

Base URL for both paths is `cfg.base_url` (i.e. `DATABRICKS_HOST`) with trailing `/` trimmed (`catalog.rs:121`). Nothing validates the scheme, so an `http://` host is accepted (see Security).

The response *interpretation* did change in `8eb6e3eb`, without touching the request contract: the v2 integration now also reads `created_timestamp` off each endpoint object (`catalog.rs:320-325`) and drops endpoints whose name looks like an embedding model (`catalog.rs:370-373`). Both are tolerant of the remote shape — `created_timestamp` is accepted as a JSON string or a number and degrades to "sorts last" when absent (`catalog.rs:315-319`), so a gateway-side wire change cannot fail discovery, only reorder it.

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
| Percent-encoding | `catalog.rs:222-233` vs the `urlencoding` dependency used at `auth.rs:589-594` | two encoders with different reserved-set behaviour |
| OAuth cache-path derivation | `auth.rs:445-467` vs test helper `tests/databricks_oauth.rs:84-97` | the test re-implements the production rule, so a change to the hash inputs keeps the test green |
| Token-JSON on-disk shape | `auth.rs:110-118` vs test fixtures at `auth.rs:750-756`, `auth.rs:786-792`, `tests/databricks_oauth.rs:99-103` | same class of drift |

#### Declared dependencies and their use in this group

`async-trait` (`auth.rs:24`), `base64` (`auth.rs:26`), `sha2` (`auth.rs:29`), `hex` (`auth.rs:452`), `axum` (`auth.rs:528`), `urlencoding` (`auth.rs:589`), `webbrowser` (`auth.rs:599`), `reqwest` (`auth.rs:27`, `catalog.rs:13`), `arc-swap` (`mcp.rs:5`), `getrandom` (`mcp.rs:824`, `auth.rs:511`, `auth.rs:520`), `nix` (`mcp.rs:846-847`), `rmcp` (`mcp.rs:6-11`), `serde_yaml` (`hints.rs:93`), `tokio` (all five), `tracing` (`mcp.rs`, `auth.rs`), `serde`/`serde_json` (throughout). `tempfile` is dev-only and used by the unit tests in `hints.rs`, `builtin.rs`, and `auth.rs`.

The `form` reqwest feature is exercised only by this group (`auth.rs:218`, `auth.rs:617`); the `json` feature is used by both this group (`auth.rs:169`, `auth.rs:227`, `auth.rs:626`, `catalog.rs:157`, `catalog.rs:273`) and `llm.rs`. No declared dependency of the crate is unused by the crate as a whole, so there is nothing to report as dead in the manifest.


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


## Module: buzz-cli — dispatch, relay client & validation (`crates/buzz-cli/src`)
### Aspect: Integrations

#### Relay over HTTP

All HTTP goes through one `reqwest::Client` built in `BuzzClient::new`
(`client.rs:547-551`). Every request is signed per attempt; there is no session
or token.

| Method + path | Auth header(s) | Body | Site |
|---|---|---|---|
| `POST /query` | `Authorization: Nostr <b64 kind-27235>`, `Content-Type: application/json`, optional `x-auth-tag` | JSON array of filters | `client.rs:773-801` |
| `POST /count` | same | `[filter]` | `client.rs:803-834` (dead code) |
| `POST /events` | same | one signed event | `client.rs:873-1022` (moderation), `client.rs:1024-1071` (stored) |
| `PUT /upload` | `Authorization: Nostr <b64url kind-24242 t=upload>`, `Content-Type: <sniffed mime>`, `X-SHA-256`, optional `x-auth-tag` | raw file bytes | `client.rs:1152-1178` |
| `PUT /media/upload` | same | raw bytes | legacy fallback, `client.rs:1195-1226` |
| `GET /media/<sha256[.ext]>` | `Authorization: Nostr <b64url kind-24242 t=get>` + optional `x-auth-tag` | — | `client.rs:1229-1256` |
| `GET <path>` authed | NIP-98 GET over the full URL | — | `client.rs:836-856`; callers pass `/moderation/reports?…`, `/moderation/restricted`, `/moderation/audit?limit=…` (`commands/moderation.rs:114,120,127`) |
| `GET /` | none; `Accept: application/nostr+json` | — | `client.rs:753-765`; NIP-11 read from `commands/agents.rs:273` |

NIP-98 construction (`client.rs:84-110`): kind 27235, empty content, tags `u`
(full URL incl. query), `method`, `nonce` (UUIDv4), and `payload`
(hex SHA-256) when a body is present; header value is
`Nostr <base64-standard(event json)>`. No `expiration` tag is added
(`grep -n 'expiration' client.rs` shows it only on the two Blossom paths,
`client.rs:330` and `client.rs:364`).

Blossom (BUD-01 get / BUD-02 upload) uses kind 24242 with URL-safe,
unpadded base64 (`client.rs:325-348`, `client.rs:350-385`). Get auth carries
`t=get`, `expiration`, `server=<authority>` and deliberately **no `x` tag**
(asserted by `media_get_auth_header_is_server_scoped`, `client.rs:452-481`).
Upload auth carries `t=upload`, `x=<sha256>`, `expiration`, and `server` when the
authority is non-empty.

Response handling is centralized in `handle_response` (`client.rs:1258-1289`):
non-2xx bodies are parsed for an `error` or `message` field, and a 403 gets a
hint appended when `BUZZ_AUTH_TAG` is set in the environment
(`client.rs:1271-1279`).

#### Relay over WebSocket — shared client, not reimplemented

`publish_ephemeral_event` (`client.rs:1073-1098`) converts the HTTP base back to
`ws(s)://` (`to_ws_url`, `client.rs:1299-1305`) and delegates the entire
connect → NIP-42 AUTH → EVENT → OK → close sequence to
`buzz_ws_client::publish_event` (`client.rs:1084`), which is the shared crate
declared at `Cargo.toml:75-76`. The inner ceilings it relies on
(`AUTH_CHALLENGE_TIMEOUT_SECS = 20`, `AUTH_OK_TIMEOUT_SECS = 20`,
`PUBLISH_OK_TIMEOUT_SECS = 30` — `crates/buzz-ws-client/src/connection.rs:17-23`)
sum to 70 s, and the CLI passes a 75 s outer budget with a comment citing those
exact constants (`client.rs:1075-1085`) — a rare in-code cross-reference that is
still accurate.

There is **no duplicated WebSocket or NIP-42 logic** in this group:
`grep -n 'tokio_tungstenite\|WebSocket\|AUTH' client.rs` finds no connection
code, only the delegation. The one gap is error fidelity: any WS failure —
including NIP-42 auth rejection and timeout — collapses to
`CliError::Other(e.to_string())` (`client.rs:1084`), which exits **4**, whereas
the HTTP equivalents exit 2 or 3. A relay OK with `accepted:false` is mapped to
a synthetic `Relay{status:400}` (`client.rs:1090-1096`), i.e. exit 2.

#### Workspace crate dependencies

| Crate | Used for | Site |
|---|---|---|
| `buzz-ws-client` | ephemeral publish (kinds 20000-29999 are WS-only) | `client.rs:1084`, `Cargo.toml:75-76` |
| `buzz-core` | `tenant::relay_url_authority` for the Blossom `server` tag (`client.rs:113`); `observer::{encrypt_observer_payload, OBSERVER_FRAME_TELEMETRY}` (`agent_management.rs:3`) |
| `buzz-sdk` | `nip_oa::{parse_auth_tag, verify_auth_tag}` (`lib.rs:1757-1763`); `build_agent_observer_frame` (`agent_management.rs:113`); `SdkError` mapping (`validate.rs:155-160`) |
| `buzz-persona` | persona pack parse/validate — reached only via `commands/pack.rs`; declared at `Cargo.toml:67-68` |
| `nostr` | keys, event builders, tags, `Kind`, `Timestamp` | throughout `client.rs`, `lib.rs:10` |
| `rustls` (ring) | process-level CryptoProvider install | `lib.rs:39`, `Cargo.toml:78-84` |

Third-party integrations: `reqwest` (HTTP), `infer` (magic-byte MIME sniffing,
`client.rs:1109-1111`), `sha2` + `hex` (NIP-98 payload hash, Blossom hash —
`client.rs:106`, `client.rs:1133`), `base64` (both standard and URL-safe
alphabets — `client.rs:109`, `client.rs:326`), `url` (origin comparison,
`client.rs:277`, `client.rs:302`), `bytes` (request bodies and download return),
`uuid` (nonce + validation), `chrono` (RFC3339 observer timestamps,
`agent_management.rs:97`), `rand` (jitter, `client.rs:134`), `diffy`/`dirs`
(declared here, used by sibling command modules).

Every declared dependency resolves to a real use: `diffy` in `commands/mem.rs`,
`dirs` in `commands/channel_templates.rs`, `buzz-persona` in `commands/pack.rs`,
`tempfile`/`axum` in `#[cfg(test)]` code (`client.rs:2123`, `client.rs:1587-1591`).
I found no unused dependency. Two Cargo.toml *comments* are stale, though: line
19 claims clap's env support is "(BUZZ_API_TOKEN auto-wired)" — that variable is
never read by this crate (`grep -rn 'BUZZ_API_TOKEN' crates/buzz-cli/src` → no
matches; it exists only in `buzz-acp` and `buzz-workflow`) — and line 44
describes `nostr` as "used in `buzz auth`, auto-mint", but there is no `auth`
subcommand (`command_inventory_is_stable`, `lib.rs:1808-1829`, lists 21 groups
and none is `auth`).

#### Code duplicated instead of shared

- `extract_relay_message_field` (`client.rs:190-198`) is the named helper for
  pulling `error`/`message` out of a relay error body, yet the same logic is
  re-inlined twice: in `submit_moderation_event` (`client.rs:985-996`, with the
  comment "Map the body through `handle_response`'s error logic inline") and in
  `handle_response` itself (`client.rs:1261-1270`). Three copies of one rule.
- `resp_was_success` (`client.rs:217-219`) re-implements
  `reqwest::StatusCode::is_success` for `u16`, needed only because
  `submit_moderation_event` consumes the response before checking status.
- The 403 `BUZZ_AUTH_TAG` hint is duplicated verbatim in
  `submit_moderation_event` (`client.rs:991-997`) and `handle_response`
  (`client.rs:1271-1279`).

#### Subprocesses, filesystem, keychain

- **No subprocesses.** `grep -n 'Command::new\|process::Command' lib.rs client.rs validate.rs error.rs agent_management.rs main.rs`
  → zero matches; the only `spawn` calls are `tokio::spawn` inside test servers
  (`client.rs:1634`, `:1873`, `:1989`, `:2052`, `:2140`, `:2220`). Nothing in
  this group injects env vars into a child — the CLI is itself the child that
  the ACP harness configures.
- **Filesystem reads:** `upload_file` stats and slurps an arbitrary path
  (`client.rs:1102-1108`); `read_file_or_stdin` reads an arbitrary path
  (`validate.rs:189-192`); `read_or_stdin`/`read_file_or_stdin` read stdin
  (`validate.rs:164-170`, `:182-187`). No writes: `grep -n 'fs::write\|File::create' lib.rs client.rs validate.rs error.rs agent_management.rs`
  → matches only inside `#[cfg(test)]` (`validate.rs:492`).
- **No keychain or OS credential store.**
  `grep -rn 'keychain\|security_framework' crates/buzz-cli/src` → zero matches.
  Secrets arrive purely through env/flags.

#### Multicall integration with `buzz-dev-mcp`

`buzz-dev-mcp` re-exports this CLI: when its argv[0] resolves to `buzz` it calls
`buzz_cli::run_from_args` and exits with the returned code
(`crates/buzz-dev-mcp/src/lib.rs:167-170`). Both processes install the ring
provider (`crates/buzz-dev-mcp/src/lib.rs:165`, `lib.rs:39`), which is exactly
the double-install the `let _ =` swallow accommodates. This coupling is
documented in the code (`lib.rs:30-38`) and in `Cargo.toml:78-83`.


## Module: buzz-cli — channel, message & social commands (`crates/buzz-cli/src/commands`)
### Aspect: Integrations

#### Relay HTTP surface used

These files never touch a WebSocket and never construct a URL. All I/O goes
through `BuzzClient`, which owns the three endpoints below.

| Endpoint | Client method | Used by | Sites |
|----------|--------------|---------|-------|
| `POST /query` (single filter) | `query` (`client.rs:767`) | every read except the four below | e.g. `channels.rs:231`, `messages.rs:296`, `notes.rs:176`, `emoji.rs:82`, `dms.rs:16`, `feed.rs:41`, `reactions.rs:86`, `social.rs:77` |
| `POST /query` (ORed filters) | `query_multi` (`client.rs:773`) | `messages thread` only | `messages.rs:332` |
| `POST /query` (keyset-paginated) | `query_paginated` (`client.rs:715`) | `channels list`, `channels search` | `channels.rs:42`, `:65`, `:133` |
| `POST /query` (paginate to exhaustion) | `query_all` (`client.rs:724`) | template roster scan | `channels.rs:449` |
| `POST /events` | `submit_event` (`client.rs:863`) | every write | e.g. `messages.rs:576`, `channels.rs:326`, `notes.rs:545`, `emoji.rs:122`, `dms.rs:72`, `reactions.rs:29`, `social.rs:38` |
| Blossom media upload | `upload_file` (`client.rs:1100`) | `messages send --file` | `messages.rs:510` |
| `GET /` (NIP-11 info) | `get_public` (`client.rs:753`), reached via `agents::fetch_archived_snapshot` | `channels create --template` archive filter | `channels.rs:632`, implementation `agents.rs:272-282` |

`POST /count` (`client.rs:803`) and `get_authed` (`client.rs:836`) are never
called from these files. Every request carries a NIP-98 `Authorization` event and,
when configured, the `x-auth-tag` header (`client.rs:120-127`).

Relay-side extension fields the CLI injects into otherwise-standard Nostr filters
(all three are Buzz extensions, parsed out of the raw JSON before
`nostr::Filter` drops them — `bridge.rs:968-971`):

| Field | Set by | Consumed at |
|-------|--------|-------------|
| `before_id` | `social notes --before-id` (`social.rs:106-108`), and by `query_paginated`'s cursor (`client.rs:500-520`) | `bridge.rs:263-281`, `:1233-1247` |
| `depth_limit` | `messages thread --depth-limit` (`messages.rs:326-327`) | `bridge.rs:283-291`, `:1140-1146` |
| `feed_types` | `feed get --types` (`feed.rs:38`) | `bridge.rs:324-330`, `:1054-1111` |

The `feed_types` coupling is load-bearing and undocumented on the CLI side: if
`--types` is omitted the field is absent, `extract_feed_types` returns `None`, and
the relay `continue`s past its feed branch (`bridge.rs:1054-1057`), so the query
degrades to a plain "events p-tagging me" scan. `feed.rs:6`'s
`VALID_FEED_TYPES = ["mentions","needs_action","activity","agent_activity"]`
duplicates the relay's accepted set (`bridge.rs:1069-1111`, where
`agent_activity` canonicalizes to `activity`); the two lists agree today but
nothing keeps them in sync.

#### NIPs implemented or consumed

| NIP | Where | Sites |
|-----|-------|-------|
| NIP-01 (kinds 0/1/3, filters) | `social` reads/writes, profile lookups | `social.rs:34`, `:97`, `:118`; `messages.rs:407`; `notes.rs:214` |
| NIP-02 contact list | `social set-contacts` | `social.rs:63`, `builders.rs:764-810` |
| NIP-09 deletion | `reactions remove` (kind 5 on the reaction), `notes rm` (kind 5 with `a` tag) | `reactions.rs:71`; `notes.rs:712-715` |
| NIP-10 threading | root/reply marker parsing and emission | `messages.rs:25-56`, `builders.rs:171-181` |
| NIP-11 relay info | archive snapshot trust check | `agents.rs:272-282` via `channels.rs:632` |
| NIP-19 bech32 `naddr` / `npub` | note coordinates, `--author npub…` | `notes.rs:317-320`, `:255`; `messages.rs:400-403` |
| NIP-21 `nostr:` URI | accepted by `parse_naddr` | `notes.rs:249` (doc), `:255` |
| NIP-23 long-form | whole `notes` group | `notes.rs:1-31`, `:418-469` |
| NIP-25 reactions | `reactions add` | `reactions.rs:20-26`, `builders.rs:463-492` |
| NIP-27 inline mentions | `extract_nostr_uris` on message bodies | `messages.rs:527-529` |
| NIP-29 groups | whole `channels` group (`h` tags, kinds 9000-9008/9021/9022, 39000/39002) | `channels.rs` throughout; `builders.rs:563-730` |
| NIP-30 custom emoji | `emoji` group, custom emoji reactions | `emoji.rs:9`, `builders.rs:127-168`, `:479-492` |
| NIP-31 `alt` tag | `messages send-diff` | `messages.rs:620-626`, `builders.rs:371-373` |
| NIP-33 parameterized replaceable | 30023 / 30030 / 39000 / 39002 / 30176 / 30177 addressing, and the LWW conflict check | `notes.rs:271-274`, `emoji.rs:9`, `channels.rs:434-439`, `notes.rs:556-566` |
| NIP-50 search | `messages search --query`, both author-name resolvers | `messages.rs:365`, `:408`; `notes.rs:215` |
| NIP-51 lists/sets | `social set-list` / `social list` | `social.rs:127-139` |
| NIP-65 relay list metadata | kind 10002 accepted by the same pair | `social.rs:130` |
| NIP-92 `imeta` | media attachment tags | `client.rs:39-40` → `messages.rs:511` |
| NIP-98 HTTP auth | every request (transport) | `client.rs:84`, invoked in `query_multi`/`submit_event` |
| NIP-OA owner attestation | auth-tag injection + template owner derivation | `client.rs:588-596`; `channels.rs:645-651` |
| NIP-IA identity archive | template roster archive filter (kind 13535 snapshot, states 1/2/3) | `channels.rs:511-587`, `agents.rs:270-305` |
| NIP-CW cursor grammar | `until`+`before_id` composite cursor | `social.rs:102-108` (partially — see below), `client.rs:500-520` |
| NIP-AE engrams, NIP-AB pairing, NIP-17 gift wrap, NIP-34 git | **not** used here | `mem.rs` / `buzz-pairing-cli` / unused / `repos.rs`,`patches.rs`,`pr.rs`,`issues.rs` |

`social notes` exposes `--before` and `--before-id` independently
(`lib.rs:1000-1006`) but the relay requires both or neither
(`bridge.rs:1240-1246`); the CLI does not enforce the pairing, so
`--before-id` alone is a relay 400 rather than a local `Usage` error.

#### Crate dependencies used from these files

| Crate / module | What is used | Sites |
|----------------|-------------|-------|
| `buzz_sdk` (builders) | 24 `build_*` functions | `channels.rs:308,856,871,887,900,912,924,936,948,978,996,1058,747,729`; `messages.rs:542,549,558,659,682,714,745`; `reactions.rs:20,24,71`; `emoji.rs:119`; `dms.rs:118`; `social.rs:33,62`; `notes.rs` uses its own builder instead |
| `buzz_sdk::kind` | `KIND_DM_HIDE`, `KIND_AGENT_PROFILE`, `KIND_EMOJI_SET`, six social list kinds | `dms.rs:103`; `channels.rs:1040`; `emoji.rs:79`; `social.rs:1-4` |
| `buzz_sdk::mentions` | `extract_at_mentions_with_known`, `extract_nostr_uris`, `merge_mentions`, `strip_code_regions`, `MENTION_CAP` | `messages.rs:11-14`, used `:192`, `:527-531` |
| `buzz_sdk` types | `ThreadRef`, `DiffMeta`, `VoteDirection`, `DeleteMessageOptions`, `CustomEmoji`, `Visibility`, `ChannelKind`, `MemberRole` | `messages.rs:1`, `emoji.rs:6`, `channels.rs:301-320`, `:750` |
| `buzz_sdk::CUSTOM_EMOJI_SET_D_TAG` | the `buzz:custom-emoji` d-tag | `emoji.rs:9` |
| `buzz_core::kind` | `KIND_MANAGED_AGENT`, `KIND_TEAM` | `channels.rs:3` |
| `nostr` | `EventBuilder`, `Kind`, `Tag`, `EventId`, `PublicKey`, `Event`, `Timestamp`, `ToBech32`, `Coordinate` | `dms.rs:57`, `social.rs:5`, `channels.rs:1036`, `reactions.rs:3`, `notes.rs:36`, `messages.rs:2` |
| `uuid` | channel/DM UUID generation and parsing | `channels.rs:5`,`:294`,`:699`; `dms.rs:1`,`:60`; `messages.rs:3` |
| `serde` / `serde_json` | every filter, every output | throughout |
| `dirs` | platform app-data dir for the template store | `channel_templates.rs:73` |
| `tempfile` (dev) | template store fixtures | `channel_templates.rs:132` |
| `tokio` (dev) | the single async test | `channels.rs:1362` |

#### Intra-crate coupling

| Import | From | Purpose |
|--------|------|---------|
| `client::{normalize_events, normalize_write_response, print_create_response, extract_d_tag, extract_p_tags, extract_tag_value, build_imeta_tag, BuzzClient}` | `client.rs` | transport + output normalization |
| `validate::{parse_uuid, validate_uuid, validate_hex64, validate_content_size, parse_event_id, read_or_stdin, truncate_diff, infer_language, sdk_err, MAX_DIFF_BYTES}` | `validate.rs` | input validation |
| `error::CliError` | `error.rs` | error taxonomy |
| `crate::{OutputFormat, ChannelsCmd, MessagesCmd, …, EmojiScope}` | `lib.rs` | clap enums |
| `commands::agents::fetch_archived_snapshot` | `agents.rs:270` | **cross-command dependency**: `channels create --template` reaches into the agents module for the NIP-IA snapshot (`channels.rs:11`, called `channels.rs:632`) |
| `commands::channel_templates` | in-scope sibling | template store loading (`channels.rs:12`) |

`channels.rs` → `agents.rs` is the only cross-command-module call in scope. It
couples channel creation to the agent-management module's trust-check semantics
(states 1/2/3), which `channels.rs` then re-interprets as fail-open
(`channels.rs:511-524`).

#### Local filesystem integration

| Path | Direction | Site |
|------|-----------|------|
| `<platform-data-dir>/xyz.block.buzz.app/templates/channel-templates.json` | read | `channel_templates.rs:18`, resolved `:71-84`, read `:95-96` |
| any path from `--templates-file` | read | `channel_templates.rs:73-75` |
| any path from `emoji import --file` | read | `emoji.rs:166-167` |
| any path from `emoji export --file` | **write (overwrite)** | `emoji.rs:187-188` |
| stdin | read | `messages send --content -` and `send-diff --diff -` (`validate.rs:186-197`), `canvas set --content -` (`channels.rs:1055`), `notes set --content -` (`notes.rs:490-508`), `emoji import` default (`emoji.rs:169-183`) |
| stdout | write | every command |
| stderr | write | `emoji import --dry-run` marker (`emoji.rs:303`), template archive warning (`channels.rs:597`) |

The template path is derived from `dirs::data_dir()` joined with a hardcoded Tauri
bundle identifier (`channel_templates.rs:15-18`), i.e. the CLI reverse-engineers
the desktop app's storage location. The comment states this matches
`app.path().app_data_dir()` exactly; that equivalence is asserted only by a
suffix check in a test (`channel_templates.rs:143-147`) and cannot be verified
from this crate.

#### Logic duplicated across these files rather than shared

| Duplicated logic | Copies |
|------------------|--------|
| Compact-format event projection `{id,content,created_at}` | `messages.rs:249-257` and `feed.rs:50-59` (byte-for-byte equivalent); `channels.rs:98-106` is a third, channel-specific copy |
| Author-name resolution against kind:0 with NIP-50 | `messages.rs:394-467` and `notes.rs:204-252` — different accepted inputs and different ambiguity reporting (see Business Rules) |
| Newest-first sort by `created_at` | `messages.rs:377-381`, `feed.rs:44`, `notes.rs:388`, `notes.rs:178-181`, `notes.rs:361-363` |
| Read-modify-write on a NIP-33 own-set | `emoji.rs:128-157` (30030) and `notes.rs:528-534` (30023) |
| `accepted` / `message` extraction from a write response | `notes.rs:546-555` and `notes.rs:734-744` (twice in one file); `dms.rs:74-84` extracts `message` differently again; `client.rs:1407-1418` already provides `extract_relay_response_field` and is used by neither |
| DM pubkey count + hex validation | `dms.rs:52-56` re-implements `builders.rs:1544-1553` because the SDK builder lacks a `d`-tag parameter |
| Channel type/visibility string→enum mapping | `channels.rs:300-320` and `channels.rs:699-709` |
| Env-gate parsing for add policy | production `channels.rs:1022-1033`, test copy `channels.rs:1296-1307` |
| Profile-JSON parsing loop (`display_name` else `name`) | `messages.rs:167-190`, `messages.rs:451-459`, `notes.rs:222-234` — three implementations of the same rule |

#### Test coverage — integrations

No test in scope exercises a relay call: every `#[test]` is a pure-function test,
and the one `#[tokio::test]` (`channels.rs:1362`) constructs a `BuzzClient`
pointed at `ws://localhost:3000` (`channels.rs:1352-1358`) precisely because the
gate returns before any network I/O (`channels.rs:1345-1350` comment). Relay
contract coverage for these kinds lives outside this module in
`crates/buzz-test-client/tests/` (e.g. `e2e_long_form.rs` for kind 30023); there
is no e2e test that drives the `buzz` binary itself, and no
`crates/buzz-cli/tests/` directory exists.

The filesystem integration is the best-covered: `channel_templates.rs` has six
tests including override precedence, prod-path suffix, case-insensitive lookup,
missing-store `NotFound`, and full roster parsing
(`channel_templates.rs:137-197`). `emoji.rs`'s `read_source` / `write_output`
have no tests (`grep -n 'read_source\|write_output' emoji.rs` shows only
definitions at `:164`/`:185` and call sites at `:241`/`:230`).


## Module: buzz-cli — repo, agent, memory & moderation commands (`crates/buzz-cli/src/commands`)
### Aspect: Integrations

#### Event kinds published, per command

Every write in this group goes out as a signed Nostr event. Kind integers are owned by
`crates/buzz-core/src/kind.rs`; `buzz-sdk` re-exports that module wholesale
(`crates/buzz-sdk/src/lib.rs:22` — `pub use buzz_core::kind;`), so the builders name the
constant even where the CLI does not.

| Command | Kind | Constant (`kind.rs:LINE`) | Kind materialized at | Transport |
|---|---|---|---|---|
| `agents draft-create` / `draft-update` | 24200 | `KIND_AGENT_OBSERVER_FRAME`, `kind.rs:409` | `buzz-sdk/src/builders.rs:271` via `build_agent_observer_frame` (called `agent_management.rs:109`) | `publish_ephemeral_event` (WSS), `agents.rs:33`, `agents.rs:75` |
| `agents archive` | 9035 | `KIND_IA_ARCHIVE_REQUEST`, `kind.rs:348` | `builders.rs:1798` | `submit_event`, `agents.rs:110` |
| `agents unarchive` | 9036 | `KIND_IA_UNARCHIVE_REQUEST`, `kind.rs:350` | `builders.rs:1819` | `submit_event`, `agents.rs:140` |
| `mem set` / `patch` / `rm` | 30174 | `KIND_AGENT_ENGRAM`, `kind.rs:94` | `buzz_core::engram::build_event`, called `mem.rs:368`, `mem.rs:694`, `mem.rs:731` | `submit_engram` → `submit_event`, `mem.rs:93-108` |
| `repos create` | 30617 | `KIND_GIT_REPO_ANNOUNCEMENT`, `kind.rs:545` | `build_repo_announcement`, `repos.rs:216` | `submit_event`, `repos.rs:226` |
| `repos protect set` / `remove` | 30617 | same | `build_repo_announcement_with_tags`, `repos.rs:136` | `submit_repo_update`, `repos.rs:195-200` |
| `patches send` | 1617 | `KIND_GIT_PATCH`, `kind.rs:549` | `build_git_patch`, `patches.rs:50` | `submit_event`, `patches.rs:52` |
| `pr open` | 1618 | `KIND_GIT_PULL_REQUEST`, `kind.rs:551` | `build_git_pull_request`, `pr.rs:58` (kind set `builders.rs:1390`) | `submit_event`, `pr.rs:60` |
| `pr update` | 1619 | `KIND_GIT_PR_UPDATE`, `kind.rs:553` | `build_git_pr_update`, `pr.rs:100` | `submit_event`, `pr.rs:102` |
| `issues create` | 1621 | `KIND_GIT_ISSUE`, `kind.rs:555` | `build_git_issue`, `issues.rs:29` | `submit_event`, `issues.rs:31` |
| `patches status` / `pr status` / `issues status` | 1630 / 1631 / 1632 / 1633 | `KIND_GIT_STATUS_{OPEN,MERGED,CLOSED,DRAFT}`, `kind.rs:557-563` | `build_git_status`, `patches.rs:183`, `pr.rs:209`, `issues.rs:140`; word→kind map in `patches::parse_status`, `patches.rs:194-205` | `submit_event` |
| `users set-profile` | 0 | `KIND_PROFILE`, `kind.rs:8` | `build_profile`, `users.rs:200` | `submit_event`, `users.rs:211` |
| `users set-presence` | 20001 | `KIND_PRESENCE_UPDATE`, `kind.rs:403` | `build_presence_update`, `users.rs:299` (kind `builders.rs:1580`) | `publish_ephemeral_event` (WSS), `users.rs:302` |
| `workflows create` / `update` | 30620 | `KIND_WORKFLOW_DEF`, `kind.rs:382` | `build_workflow_def` / `build_workflow_update`, `workflows.rs:107`, `workflows.rs:129` | `submit_event` |
| `workflows delete` | 5 | `KIND_DELETION`, `kind.rs:56` | `build_workflow_delete`, `workflows.rs:144`; `a`-tag value is `"{KIND_WORKFLOW_DEF}:{pk}:{id}"` (`builders.rs:1505`, kind at `builders.rs:1507`) | `submit_event` |
| `workflows trigger` | 46020 | `KIND_WORKFLOW_TRIGGER`, `kind.rs:498` | **two paths** — `build_workflow_trigger` (`workflows.rs:184`) or, with `--inputs`, a hand-rolled `EventBuilder` naming `buzz_sdk::kind::KIND_WORKFLOW_TRIGGER` (`workflows.rs:175-178`) | `submit_event` |
| `workflows approve` | 46030 / 46031 | `KIND_APPROVAL_GRANT` / `KIND_APPROVAL_DENY`, `kind.rs:500`, `kind.rs:502` | `build_workflow_approval`, `workflows.rs:206` | `submit_event` |
| `moderation ban/unban/timeout/untimeout/resolve` | 9040-9044 | `KIND_MODERATION_*`, `kind.rs:298`, `:300`, `:303`, `:305`, `:310` | `build_moderation_*`, `moderation.rs:42`, `:53`, `:71`, `:81`, `:97` | `submit_event`, which routes 9040-9044 through the non-idempotent policy (`client.rs:863-870`) |

#### Event kinds queried, and the bare-literal audit

The task asked specifically for bare integers used in place of the constant. `grep -n '"kinds"'`
across the eleven files returns 20 filter sites; only **five** name a constant.

| Site | Filter kinds | Constant named? |
|---|---|---|
| `mem.rs:151`, `mem.rs:198` | `[KIND_AGENT_ENGRAM]` | yes |
| `agents.rs:285` | `[KIND_IA_ARCHIVED_LIST]` | yes |
| `repos.rs:21` | `[KIND_GIT_REPO_ANNOUNCEMENT]` | yes |
| `agents.rs:180` | `[0]` | no — bare literal for kind:0 (`KIND_PROFILE`, `kind.rs:8`) |
| `repos.rs:240`, `repos.rs:271` | `[30617]` | **no** — and this is the same file that imports the constant at `repos.rs:3` and uses it at `repos.rs:21` |
| `users.rs:42`, `users.rs:91`, `users.rs:223` | `[0]` | no |
| `users.rs:258` | `[40902]` | no (`KIND_PRESENCE_SNAPSHOT`, `kind.rs:443`) |
| `pr.rs:110`, `pr.rs:131` | `[1618]` | no (`kind.rs:551`) |
| `patches.rs:76`, `patches.rs:96` | `[1617]` | no (`kind.rs:549`) |
| `issues.rs:39`, `issues.rs:60` | `[1621]` | no (`kind.rs:555`) |
| `workflows.rs:16`, `workflows.rs:41` | `[30620]` | no (`kind.rs:382`) |
| `workflows.rs:74` | `[46001, 46002, 46003]` | no (`kind.rs:504-508`) |

A second class of bare literal is the NIP-34 `a`-tag address, hardcoded as
`format!("30617:{repo_owner}:{repo_id}")` in three files — `pr.rs:129`, `patches.rs:94`,
`issues.rs:58` — where the SDK's own `GitRepoCoord::to_a_tag_value` already exists
(`builders.rs:976`) and is used on the write side (`builders.rs:1024`, `:1097`, `:1240`,
`:1352`, `:1431`). And `repos.rs:411` builds
`Kind::Custom(30617)` in a test helper.

The `repos.rs` split (constant at `:21`, literal at `:240` and `:271`) is the clearest
internal inconsistency in the group: one file, one kind, two spellings.

#### HTTP paths reached, and one endpoint the docs do not name

`AGENTS.md:122` states the relay's HTTP surface is "NIP-11/NIP-05 metadata, `POST /events`,
`POST /query`, `POST /count`, workflow webhooks at `/hooks/{id}`, Blossom media, git smart
HTTP, git policy hooks, and health probes."

The `/moderation/*` triple is **not** on that list, and three commands here depend on it.
Verified on both sides:

| Path | CLI call site | Relay route | Relay handler |
|---|---|---|---|
| `GET /moderation/reports?limit=N[&status=S]` | `moderation.rs:110-114`, via `client.get_authed` (`client.rs:836-853`) | `crates/buzz-relay/src/router.rs:113` | `api/bridge.rs:2091-2115` |
| `GET /moderation/restricted` | `moderation.rs:120` | `router.rs:116` | `api/bridge.rs:2135-2145` |
| `GET /moderation/audit?limit=N` | `moderation.rs:126-128` | `router.rs:114` | `api/bridge.rs:2117-2133` |

`moderation.rs:8-13` documents and defends the choice ("reports and audit rows are
structured queue rows, not public nostr events — serving them over a REQ filter would mean
synthesizing fake events"). So this is a deliberate, argued exception that `AGENTS.md` has
not caught up with — a code-vs-docs contradiction with the code side carrying the better
rationale. Both endpoints keep the host-derived community boundary `AGENTS.md:122` promises:
`authorize_moderation_read` binds the tenant from the `Host` header before any lookup
(`api/bridge.rs:2032-2043`).

Everything else in the group stays inside the documented surface: `POST /query`
(`client.rs:767-771`), `POST /events` (`client.rs:863-870`), NIP-11 `GET /`
(`client.rs:753-765`, reached only from `agents.rs:272`), Blossom `PUT /upload` with legacy
`PUT /media/upload` fallback (`client.rs:1139`, `client.rs:1195`) and `GET /media/<sha256>`
(`client.rs:1230`), and WSS for ephemeral kinds (`client.rs:1073-1096`).

`POST /count` is implemented (`client.rs:803`) but reached by nothing here —
`grep -rn '\.count(' ` across the eleven files returns zero matches.

#### NIPs implemented or relied on

| NIP | Where, in this group |
|---|---|
| NIP-01 | event shape everywhere; `ev.verify()` before trusting a fetched event, `mem.rs:162-164`, `agents.rs:364-368` |
| NIP-11 | `agents archived` reads the relay info document for `self`, `agents.rs:271-284`; `normalize_relay_self_hex` guards it, `agents.rs:250-256` |
| NIP-33 | addressable `d`-tag replaceables for 30174/30617/30620; LWW-dominance detection at `mem.rs:105-108` and `repos.rs:187-192` |
| NIP-34 | git patches/PRs/issues/status — `patches.rs`, `pr.rs`, `issues.rs`, `repos.rs`; the `p`-tag-the-owner rule is spelled out at `patches.rs:152-155` and mirrored at `issues.rs:118-121`, `pr.rs:185-187` |
| NIP-42 | WSS auth for ephemeral publishes, delegated to `buzz_ws_client::publish_event`, `client.rs:1080-1082` |
| NIP-43 | moderation commands are modeled on the 9030-series relay-admin pattern, `moderation.rs:4-7` |
| NIP-44 v2 | engram body encryption; conversation key at `mem.rs:147`, plaintext cap `NIP44_PLAINTEXT_MAX = 65_535` (`buzz-core/src/engram.rs:28`) enforced at `mem.rs:333-338` and `mem.rs:650-656` |
| NIP-50 | `users get --name` sends a `search` filter, `users.rs:89-94`; the doc comment at `users.rs:80` notes it degrades to `[]` on relays without FTS |
| NIP-70 | `-` protected-event marker required exactly once on the 13535 snapshot, `agents.rs:337-358` |
| NIP-98 | every `/query`, `/events` and `/moderation/*` request is signed, `client.rs:769`, `client.rs:841`; NIP-98 is re-signed per retry attempt so the relay's replay guard stays happy (`client.rs:891-893`) |
| NIP-AE | the whole `mem` surface, `mem.rs:1-15` |
| NIP-IA | archive/unarchive + archived-identities snapshot, `agents.rs:93-156`, `agents.rs:259-317` |
| NIP-OA | owner-of-agent attestation; owner pubkey read from slot 1 of the `auth` tag (`client.rs:576-583`), consumed at `mem.rs:36-43` and `agents.rs:160-166` |

`grep -n 'NIP-' ` across the eleven files returns zero hits for NIP-05, NIP-29 or NIP-17 —
no command here is channel-scoped by `h` tag except `workflows list` (`workflows.rs:17`),
`workflows create`/`update` (`builders.rs:1471`, `:1489`) and `pr open --channel`
(`builders.rs:1373`).

#### Relationship to the relay's git surface

There is **no direct coupling** — the CLI never speaks git wire protocol. The connection is
data-mediated:

1. `repos create` publishes the 30617 announcement carrying `clone` URLs
   (`repos.rs:216-224`), which is what a client needs to find the smart-HTTP endpoint the
   relay serves at `/git/{owner}/{repo}/info/refs`, `/git-upload-pack`, `/git-receive-pack`
   (`crates/buzz-relay/src/api/git/transport.rs:1760-1762`).
2. `repos protect set/remove` writes `buzz-protect` tags onto that same event
   (`build_protection_tag`, `repos.rs:64-90`). The relay's git policy hook reads them back:
   `parse_protection_tags` + `evaluate_push` at
   `crates/buzz-relay/src/api/git/policy.rs:45` and `:285`, served at
   `POST /internal/git/policy` (`api/git/mod.rs:62`). `buzz_core::git_perms` is the shared
   contract — `grep -rln 'git_perms' crates/` returns exactly three files:
   `buzz-cli/src/commands/repos.rs`, `buzz-core/src/lib.rs`, and that policy module. So
   `repos.rs` is the **write** end of a rule the relay is the **enforce** end of, and the two
   agree only because both call into `buzz-core`.
3. `git-sign-nostr` and `git-credential-nostr` are **not** referenced from `buzz-cli` at all
   — `grep -rn 'git-sign-nostr\|git_sign_nostr\|git-credential-nostr\|git_credential_nostr'
   crates/buzz-cli/` returns zero matches. They are peer tools invoked by `git` itself:
   `git-sign-nostr` as a `gpg.x509.program` producing BIP-340 signatures over commits
   (`crates/git-sign-nostr/src/lib.rs:1-13`), `git-credential-nostr` as a credential helper
   minting kind:27235 NIP-98 events for push/fetch
   (`crates/git-credential-nostr/src/lib.rs:1-6`). The only place their output surfaces in
   this group is `patches send --commit-pgp-sig`, which the CLI passes through untouched
   (`patches.rs:41`).

#### `buzz-workflow` as reached from `workflows.rs`

`buzz-workflow` is **not a declared dependency** of `buzz-cli` — it is absent from
`crates/buzz-cli/Cargo.toml`. The integration is therefore entirely by document:

- `workflows create` / `update` read a YAML blob (`read_or_stdin`, `workflows.rs:104`,
  `workflows.rs:127`) and publish it verbatim as the kind:30620 content. **Nothing parses or
  validates the YAML client-side.** The only gate is the SDK's byte cap
  (`check_content(yaml, 64 * 1024)`, `builders.rs:1468`, `:1486`). A workflow with a
  malformed `if:` expression, an unknown step type, or a syntactically invalid document is
  accepted by `buzz create` and only fails later, relay-side.
- The `if:` conditions inside that YAML are evaluated by `evalexpr` in the relay's executor
  (`crates/buzz-workflow/src/executor.rs:15`, `:203-232`, `:317-318`). The CLI has no view
  of the variable namespace, the underscore-not-dot naming rule
  (`executor.rs:205-216`), or the `str_contains`/`str_starts_with`/`str_ends_with` helpers
  registered at `executor.rs:232`. So `buzz workflows create` cannot tell an operator that a
  condition references an undefined variable.
- The webhook side (`POST /hooks/{id}`, `crates/buzz-relay/src/router.rs:120`, handler
  `api/bridge.rs:1780-1795`) is **not reachable from any command in this group** —
  `grep -n '/hooks' ` across the eleven files returns zero matches. `workflows trigger`
  reaches the same executor through the kind:46020 event path instead
  (`workflows.rs:156-190`).
- `workflows runs` queries 46001/46002/46003 and its own doc comment
  (`workflows.rs:60-64`) says these events are never emitted — run history lives in the
  `workflow_runs` table. The command is wired to a surface that does not exist yet and will
  return `[]`.

#### `buzz-persona` as consumed by `pack.rs`

`buzz-persona` (declared `Cargo.toml:70-71`) is reached from **exactly four call sites, all
in `pack.rs`** — `grep -rn 'buzz_persona' crates/buzz-cli/src/` returns
`pack.rs:24`, `:28`, `:31`, `:62`:

| Call | Site | What it pulls in |
|---|---|---|
| `validate::validate_pack(pack_dir)` | `pack.rs:24` | full pack load + diagnostics, returned as `Vec<ValidationDiagnostic>` |
| `validate::ValidationDiagnostic::{Error,Warning}` | `pack.rs:28`, `pack.rs:31` | the two-level diagnostic enum, printed to stderr |
| `resolve::resolve_pack(pack_dir)` | `pack.rs:62` | `ResolvedPack` — post-merge, post-split effective config |

`pack.rs` reads a wide slice of `ResolvedPersona` (`runtime_env_vars` field declared at
`buzz-persona/src/resolve.rs:64`, projected by `resolve.rs:365`):
`name`, `display_name`, `description`, `avatar`, `llm_provider`, `model`, `temperature`,
`max_context_tokens`, `subscribe`, `triggers`, `thread_replies`, `broadcast_replies`,
`mcp_servers` (count only, `pack.rs:116`), `skills`, `system_prompt`, `runtime_env_vars`.
`pack_instructions`, `hooks` and `version`-per-persona are resolved by the crate but never
printed — the two `pack` commands are the only relay-free commands in the whole CLI
(`pack.rs:3` — "No relay connection needed"), and `lib.rs:1737-1740` short-circuits them
before a `BuzzClient` is built.

#### Filesystem interaction

`grep -rn 'std::fs::\|Path::new\|canonicalize'` across the eleven files returns four sites:

| Site | Operation | Bound? | Confined to a root? |
|---|---|---|---|
| `mem.rs:581` | `std::fs::read_to_string(patch_path)` for `mem patch --patch-file` | **no** — the `limit` computed at `mem.rs:577` is applied only in the stdin arm (`mem.rs:585`) | no |
| `upload.rs:23` | `std::fs::write(path, &bytes)` for `media get -o` | n/a | no |
| `pack.rs:16`, `pack.rs:53` | `Path::new(path)` + `exists()`/`is_dir()` for the pack root | n/a | no |
| (indirect) `validate.rs:189-192` | `read_file_or_stdin` → `std::fs::read_to_string`, used by `patches send --patch-file` (`patches.rs:26`) and `pr open/update/status --body-file` (`pr.rs:15`) | **no** | no |
| (indirect) `client.rs:1101-1110` | `metadata` + `read` for `upload file`; `is_file()` check and MIME/size gates at `client.rs:1103-1131` | yes — `MAX_IMAGE_BYTES` 50 MiB (`client.rs:73`) / `MAX_VIDEO_BYTES` 500 MiB (`client.rs:76`), selected at `client.rs:1120-1125` | no |

Memory "files" are virtual — a slug is a single virtual file (`mem.rs:610-613`) held in a
30174 event, never on disk. No command in this group writes a dotfile, cache, or config;
`dirs` (`Cargo.toml:73-75`) is used by `channels.rs`, not here.

#### Subprocesses

**None.** `grep -rn 'Command::new\|std::process\|tokio::process'` across the eleven files
returns zero matches. `pack inspect` prints MCP server `command`/`args` values but never
executes them (`pack.rs:116` prints only the count), and `buzz-persona` deliberately does
not resolve or run hook paths (`buzz-persona/src/resolve.rs:340-347` — "hooks are not
executed in this PR"). The only subprocess anywhere near this feature area is `git`'s own
invocation
of `git-credential-nostr`, which shells out to `git config`
(`crates/git-credential-nostr/src/lib.rs:16-19`) — outside this group.

#### Workspace crates and third-party dependencies used from this group

| Crate | Reached from | Notes |
|---|---|---|
| `buzz-core` | `mem.rs:22-25` (`engram`, `kind`), `agents.rs:1` (`kind`), `repos.rs:1-4` (`git_perms`, `kind`), `agent_management.rs:2` (`observer`) | the shared-contract crate |
| `buzz-sdk` | all eight write-bearing modules | typed builders; `buzz_sdk::kind` re-export used once, `workflows.rs:176` |
| `buzz-persona` | `pack.rs` only (4 sites) | **reached only from this group** |
| `buzz-ws-client` | indirectly, via `client.rs:1080` from `agents.rs:33`, `agents.rs:75`, `users.rs:302` | ephemeral publish |
| `nostr` | `mem.rs:26`, `agents.rs:3`, `repos.rs:5`, `moderation.rs:17`, `workflows.rs:174` | `Event`, `Keys`, `PublicKey`, `Tag`, `Timestamp`, `EventBuilder` |
| `diffy` 0.5 | `mem.rs` only — 31 occurrences; `grep -rn 'diffy' crates/buzz-cli/src/` outside `mem.rs` returns zero | **declared dependency reached exclusively from this group**, `Cargo.toml:52-53` |
| `sha2` | `mem.rs:20` (base-hash), `workflows.rs:1` (approval-token hash) — also `client.rs:7` | shared |
| `hex` | `mem.rs:381`, `workflows.rs:204` | shared with `client.rs` |
| `uuid` | `workflows.rs:105`, via `validate::parse_uuid` | shared |
| `chrono` | not from here — only `agent_management.rs:97`, which `agents.rs:6` pulls in transitively | so `agents draft-*` is the sole reason `chrono` is a dependency (`Cargo.toml:41-42`) |
| `serde_json` | everywhere | filters, responses, output |

Notably absent as dependencies given the surface: `buzz-workflow` (YAML/evalexpr — see
above), and any YAML parser at all (`grep -n 'serde_yaml\|yaml' crates/buzz-cli/Cargo.toml`
returns zero matches), which is why `workflows create` cannot pre-validate.

#### Logic duplicated across these eleven files rather than shared

| Duplicated logic | Copies | Comment |
|---|---|---|
| NIP-34 `a`-tag address string | `pr.rs:129`, `patches.rs:94`, `issues.rs:58` | three identical `format!("30617:{repo_owner}:{repo_id}")`; `GitRepoCoord::to_a_tag_value` (`builders.rs:1353`) already does this on the write side |
| `--repo-owner`/`--repo-id` must-pair match block | `pr.rs:169-183`, `patches.rs:136-150`, `issues.rs:99-113` | byte-identical including the error string |
| owner-first recipient list + dedupe | `pr.rs:185-196`, `patches.rs:152-165`, `issues.rs:118-127` | three copies of the same `p`-tag defaulting rule; each carries its own comment restating NIP-34's intent |
| relay "duplicate" → conflict detection | `mem.rs:93-108` (`submit_engram`), `repos.rs:173-192` (`validate_write_response`) | same two-clause rule (`== "duplicate"` ‖ `starts_with("duplicate:")`), two implementations, opposite clause order, different error text. Only the `repos.rs` copy is tested (`duplicate_write_response_is_a_conflict`, `repos.rs:619`) |
| `parse_events` JSON→`Vec<Event>` | `mem.rs:114-134`, `repos.rs:11-14` | **not** equivalent: `mem.rs` skips undeserializable entries by design (`mem.rs:120-128`), `repos.rs` fails the whole response. Same name, different failure semantics, same crate |
| `--format compact` projection | `users.rs:63-72`, `users.rs:134-143` | identical `{pubkey, display_name}` reduction inside one file |
| relay-response JSON reparse + field injection | `agents.rs:35-47`, `agents.rs:77-89` | two ~13-line copies differing only in which draft was sent |
| `pack` path precondition (`exists` / `is_dir`) | `pack.rs:16-22`, `pack.rs:54-59` | identical six-line preamble |

None of these have a shared helper, though `client.rs` already hosts the group's other
shared normalizers (`normalize_write_response` at `client.rs:1420`,
`print_create_response` at `client.rs:1401`).

#### Where I am uncertain

- I did not run the relay, so the `/moderation/*` request/response bodies are inferred from
  the handler signatures at `api/bridge.rs:2091-2145`, not observed.
- Whether the relay validates workflow YAML on ingest of kind:30620 (as opposed to at
  execution time) is not something I traced; I only established that the CLI does not.
- `workflows runs` returning `[]` is what `workflows.rs:60-64` asserts about the relay. I
  confirmed no 46001-46003 emission from `buzz-workflow`'s executor by absence of a publish
  call in the files I read, but I did not exhaustively grep the relay for every emission
  path.

