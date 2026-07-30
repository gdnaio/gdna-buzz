<!-- Analyzed: 2026-07-25T01:12:08Z | Scope: full project -->
# API Surface

> Status: initialized in Phase 1. Endpoints, wire messages, event kinds, request/response
> shapes, and auth requirements are populated per-module during Phase 2 and verified in
> Phase 3.

## Summary

Buzz's primary API is **NIP-01/NIP-29 over WebSocket** (signed Nostr events), with a
deliberately narrow HTTP bridge. New capabilities are added as event kinds rather than
endpoints.

Batch 2a (foundation crates) complete. Verified surface so far:

| Module | Public API shape |
|---|---|
| `buzz-core` | 15 public modules, 33 public types, 152 data constants, 94 fns/methods; **130 kind constants**, `ALL_KINDS` = **127** entries, 11 classification fns, 25 compile-time assertions |
| `buzz-sdk` | 61 public fns (51 event builders + 10 helpers), 15 types; covers **45 kind integers**; zero network calls |
| `buzz-persona` | 6 modules, 21 types, 16 fns; **zero built-in personas** (pure parser/loader over caller-supplied pack dirs) |
| `buzz-ws-client` | 4 types, 3 constants, 10 fns; handles **7** inbound wire messages (was 6 — `07d0265c` added the `COUNT` arm, `message.rs:147-162`), 2 typed outbound + `send_raw` escape hatch |

⚠️ **Correction to `ARCHITECTURE.md`** (verified against code): the doc's claim of
"all 81 kinds" (`ARCHITECTURE.md:142`) and "`ALL_KINDS` // 80 entries"
(`ARCHITECTURE.md:346`) are both stale. Actual: **130 kind constants**, **127 in
`ALL_KINDS`** (`crates/buzz-core/src/kind.rs:566-693`). Three constants are excluded
from `ALL_KINDS` — `KIND_AUTH` (22242), plus two the docs never mention:
`KIND_NOSTR_IDENTITY_BINDING` (24243) and `KIND_PUSH_LEASE` (30350). The implied
"never stored ⇒ excluded" rule does not hold — `KIND_BLOSSOM_AUTH` and `KIND_HTTP_AUTH`
carry "not stored" doc comments yet are present in `ALL_KINDS`.

Batch 2b (service crates) complete. All seven are **library-only** — none exposes an HTTP
route, WebSocket handler, or CLI of its own; the relay is the sole integrator.

| Module | Public API shape |
|---|---|
| `buzz-db` | **409 public API items** across 22 files; `lib.rs` alone is 6,106 lines. Runtime `sqlx::query()` throughout — no compile-time SQL validation |
| `buzz-auth` | Scope enum + session types. `Scope::all_known()` = **16** scopes is what NIP-42 grants; `all_non_admin()` (14) has **zero callers**. `ChannelAccessChecker` has **zero implementors repo-wide** — `check_read_access`, `check_write_access`, `require_scope`, and `derive_pubkey_from_username` all have no production caller |
| `buzz-search` | A single query shape, `community_id` always the first predicate, zero authorization (callers must re-authorize every hit) |
| `buzz-audit` | **11** audit actions, not the 10 documented (`MediaUploaded` is missing from the doc). `AuditError::AuthEventForbidden` **does not exist** despite being documented. `verify_chain`/`get_entries` have no production caller |
| `buzz-media` | Blossom upload/download surface. `verify_blossom_get_auth` exists but is never called from this crate — the gate lives in the relay behind a flag that defaults to false |
| `buzz-workflow` | **5** triggers, not 4 (`diff_posted` is undocumented). Three actions are non-functional: `send_dm`, `set_channel_topic` (both return `NotImplemented`) and `add_reaction`, which POSTs to a route the relay never registers. `execute_from_step` has **3** live callers (docs say it is for future use); `execute_run` has **none** |
| `buzz-pubsub` | ~55 public items over 9 modules; `PubSubManager` facade of 18 methods. `pub mod subscriber` is an **empty public namespace** (all items `pub(crate)`). `channel_key`/`global_key` are reachable by three distinct paths. No public enum is `#[non_exhaustive]`, so adding a variant is semver-breaking |

⚠️ **Second correction to `ARCHITECTURE.md`**: §9 #2 (`ARCHITECTURE.md:823`), plus `:390`
and `:460`, state that rate limiting is not enforced and only a test stub exists. That is
false. `RedisRateLimiter` (`crates/buzz-pubsub/src/rate_limiter.rs:99`) is constructed
unconditionally (`crates/buzz-relay/src/state.rs:712`) and enforced before work on WS
`EVENT`/`REQ`/`COUNT` (`crates/buzz-relay/src/connection.rs:498-500` → `:594-653`) and on
`POST /events`, `/query`, `/count` (`crates/buzz-relay/src/api/bridge.rs:760`, `:955`,
`:1386`).

Relay HTTP endpoints, CLI surfaces, MCP tools, and Tauri IPC commands are populated in
batches 2c–2f.

### Batch 2c API surface (relay HTTP, WebSocket, git, mesh)

This batch is where the actual public surface lives, and it is materially wider than
`AGENTS.md` describes.

- **14 routed endpoints fall outside the "narrow HTTP surface"** `AGENTS.md` claims (NIP-11/05
  metadata, `POST /events|/query|/count`, `/hooks/{id}`, Blossom, git smart HTTP, git policy
  hooks, health). The full route table is in `crates/buzz-relay/src/router.rs`.
- **`GET /_mesh` is unauthenticated** and bound on the hard-coded `0.0.0.0` health listener
  (`crates/buzz-relay/src/main.rs:1116`, `:1119`); it discloses peer `endpoint_addrs`
  (`crates/buzz-relay-mesh/src/status.rs:20`).
- **`POST /api/invites/accept-policy` is an unauthenticated HMAC oracle** — no credential is
  required to exercise it.
- **`/_mesh/demo/echo` takes `community_id` from the request body**, not from the host-derived
  `TenantContext` — the one place the tenancy fence is bypassed by construction.
- **`crates/buzz-relay/src/api/events.rs` is 100% dead** — no route references it.
- **Git smart HTTP performs no read authorization** on either the info/refs or upload-pack
  path (`crates/buzz-relay/src/api/git/transport.rs:539-594`, `:786-827`).
- **`buzz-conformance` has no transport surface at all** — no HTTP route, no WS frame, no CLI
  subcommand, no event kind; its dependencies (`serde`, `serde_json`, `thiserror`, `uuid`,
  `crates/buzz-conformance/Cargo.toml:26-29`) carry no transport. Its entire API is the
  `Tracer` trait (`src/lib.rs:314-318`) plus `check_trace` (`src/checker.rs:74`).
- **HTTP is stricter than WebSocket on one authorization check** — the p-gate at
  `crates/buzz-relay/src/api/bridge.rs:981` has no WS counterpart for channel-scoped REQ
  (`crates/buzz-relay/src/handlers/req.rs:182`).

### Batch 2d API surface (ACP JSON-RPC, MCP tools, agent CLI)

2d adds three API surfaces that are not Nostr and not HTTP: the **ACP JSON-RPC 2.0 line protocol**
over stdin/stdout, the **MCP tool surface** `buzz-dev-mcp` exposes to the model, and the **`buzz`
CLI** itself — 21 command groups and roughly 100 subcommands, which is the largest single API
surface in the repo by leaf count.

| ID | Finding | Location |
|---|---|---|
| API-156 | **`buzz-agent` handles six ACP request methods, not the three its README documents.** `initialize`, `session/new`, `session/prompt`, `session/set_model` (`lib.rs:234`), `session/cancel` as a request (`:237`), and `_goose/unstable/session/steer` (`:245`). Dispatch is a flat `match` on the method string (`lib.rs:224-265`) with **no state machine** — `initialize` is not required before `session/new` or `session/prompt`, and `negotiated_version` is computed and dropped (`:284`). Unknown methods return `-32601`. | `crates/buzz-agent/src/lib.rs:224-265`, `:284` |
| API-157 | **Ten outbound notification variants exist against three documented.** `session/update` carries `session_info_update` with `_meta.goose.activeRunId` (`lib.rs:661-670`) and with `queuedSteer` (`:612-620`), plus `keepalive` (`agent.rs:134-144`), `agent_thought_chunk` (`:179-192`), `agent_message_chunk` (`:194-206`), `tool_call` pending (`:552-568`) and `tool_call_update` in-progress/completed/failed (`:570-616`); `usage_update` uses a separate top-level method `_goose/unstable/session/update` (`lib.rs:730-750`). The `_meta` envelope is nested **inside** `update`, not beside it (`wire.rs:157-169`), deliberately matching goose's layout. | `crates/buzz-agent/src/lib.rs:612-620`, `:661-670`, `:730-750`; `agent.rs:134-206`, `:552-616` |
| API-158 | **One ordering guarantee is load-bearing across the crate boundary and enforced only by statement order.** `usage_update` must be sent before the `session/prompt` response, because `buzz-acp`'s `UsageTracker` must observe it while the turn is still open. `lib.rs:714-752` precedes `:753-759`; the contract is a comment plus ordering, with no assertion or type. Three integration tests do catch a reorder (`tests/fake_llm.rs:801`, `:888`, `:926`). | `crates/buzz-agent/src/lib.rs:714-759` |
| API-159 | **The ACP error-code table is complete and consistent**, mapped from `AgentError::json_rpc_code` (`types.rs:249-256`): `-32700` parse, `-32600` invalid request, `-32601` method not found, `-32602` invalid params, `-32001` LLM auth failure, `-32002` model not found, `-32000` everything else. Five `stopReason` strings (`end_turn`, `cancelled`, `max_tokens`, `max_turn_requests`, `refusal`) are emitted by `StopReason::as_wire` (`types.rs:183-191`) and all five are parsed by the client (`crates/buzz-acp/src/acp.rs:66-71`, test `:2012`). | `crates/buzz-agent/src/types.rs:183-191`, `:249-256` |
| API-160 | **Advertised capabilities match enforcement.** `initialize` advertises `loadSession: false` (`lib.rs:292` — no `session/load` handler exists), `promptCapabilities.image: false` (`:293` — `ContentBlock` accepts only `text` and `resource_link`, `types.rs:210-221`, enforced `agent.rs:623-628`), and `mcpCapabilities.http/sse: false` (`:294` — only stdio server specs are accepted, `types.rs:195-202`). Version negotiation is `min(client, PROTOCOL_VERSION = 2)` (`lib.rs:284`, `config.rs:3`), tested for both 99→2 and 1→1 (`tests/golden_transcripts.rs:288`). | `crates/buzz-agent/src/lib.rs:284-294` |
| API-161 | **`buzz-acp`'s only public re-export has no external consumer.** `pub use usage::TurnUsage` (`lib.rs:15`) is the crate's sole public re-export; `sprig` calls `buzz_acp::run()` only. `run()` itself (`lib.rs:1233`) has no doc comment. `buzz-dev-mcp`'s only public item is `pub fn run()` (`lib.rs:138`), also undocumented, with no `//!` module doc on the crate. | `crates/buzz-acp/src/lib.rs:15`, `:1233`; `crates/buzz-dev-mcp/src/lib.rs:138` |
| API-162 | **`buzz-dev-mcp` exposes shell and file-edit tools and deliberately no relay operations**, so the `AGENTS.md:147` boundary ("agent-facing relay operations go in `buzz-cli`") holds. The tool list is at `lib.rs:40-125`. Relay access is instead provided by `Shim::install`, which symlinks the running executable to `rg`, `tree`, `buzz`, `git-credential-nostr` and `git-sign-nostr` inside a `0700` tempdir prepended to the child `PATH` (`shim.rs:31-49`), dispatched by `argv[0]` (`lib.rs:144-171`) — which is why the crate depends on `buzz-cli`. | `crates/buzz-dev-mcp/src/lib.rs:40-125`, `:144-171`; `shim.rs:31-49` |
| API-163 | **The `buzz` CLI surface: 21 command groups, ~100 subcommands, four global flags, and no `--version`.** Globals are `--relay`, `--private-key`, `--auth-tag`, `--format` (`lib.rs:80-93`), all top-level-only. `--version` is handled by a branch at `lib.rs:48-52` that can never fire, because no `version` attribute is declared on `#[command(...)]` (`lib.rs:62-78`) — verified: `buzz --version` exits **1** as an unknown argument. There are no shell completions, no man page, no config file, and no verbosity/quiet/colour controls; the crate has no logging framework at all. | `crates/buzz-cli/src/lib.rs:48-52`, `:62-93`, `:175-240` |
| API-164 | **`messages search` has no `--kinds` flag**, so `AGENTS.md` gotcha #3 is unfollowable. `MessagesCmd::Search` declares only `--query`, `--author`, `--since`, `--limit` (`lib.rs:472-489`); verified `--kinds 9` → exit 1. Kinds are hardcoded downstream at `commands/messages.rs:361`. `--kinds` exists only on `messages get` (`lib.rs:453-455`), where it is optional with downstream defaults. | `crates/buzz-cli/src/lib.rs:472-489` vs `AGENTS.md` § Common Gotchas #3 |
| API-165 | **The CLI exit-code contract is implemented as documented for the six named classes but has two undocumented collisions and no unit test.** `exit_code` (`error.rs:92-107`): 0 ok, 1 `Usage` **and `NotFound`**, 2 `Relay{other}`/`Network` **and `DeliveryUnknown`**, 3 `Relay{401,403}`/`Auth`/`Key`, 4 `Other`, 5 `Conflict`; clap usage errors also exit 1 (`lib.rs:44-47`). Because `DeliveryUnknown` shares 2 with `Network`, an agent cannot distinguish "the write may already have landed" from "the connection failed" by exit code alone — only by the `error` field on stderr (`error.rs:117-125`). `exit_code` itself has zero tests. Six mappings were verified empirically against the built binary. | `crates/buzz-cli/src/error.rs:92-107`, `:117-125`; `lib.rs:44-47`, `:74` |
| API-166 | **The CLI reaches three relay endpoints the house docs do not describe, plus one undocumented request header.** `GET /moderation/reports?limit=N[&status=S]`, `GET /moderation/restricted` and `GET /moderation/audit?limit=N` via `client.get_authed` (`moderation.rs:110-128`, client `client.rs:836-856`), routed at `crates/buzz-relay/src/router.rs:113-116` and handled at `api/bridge.rs:2091-2145`. `moderation.rs:8-13` argues the exception — these are structured queue rows, not public Nostr events — so the code carries the better rationale and `AGENTS.md:122` simply has not caught up. The `x-auth-tag` header (`client.rs:616-621`) appears in no markdown document. | `crates/buzz-cli/src/commands/moderation.rs:8-13`, `:110-128`; `client.rs:616-621`, `:836-856` |
| API-167 | **`POST /count` is fully implemented in the CLI client and unreachable from any subcommand.** `BuzzClient::count` (`client.rs:802-834`) has retry wrapping and zero callers, kept alive by `#[allow(dead_code)]`; so COUNT filters, a documented relay capability (`AGENTS.md:145`), are not exposed to users. `relay_url()` (`client.rs:567`) is dead the same way. | `crates/buzz-cli/src/client.rs:567`, `:802-834` |
| API-168 | **`buzz-cli`'s 22 `*Cmd` enums are `pub` but structurally unreachable.** `Cli` (`lib.rs:79`) and `Cmd` (`lib.rs:175`) are private and no public function accepts or returns a `*Cmd`, so an external crate can name `buzz_cli::MessagesCmd` but cannot feed one in. The only real entry point is `run_from_args` (`lib.rs:23-60`), and the only cross-crate consumer uses exactly that (`crates/buzz-dev-mcp/src/lib.rs:170`). Meanwhile the public data module `types` in `buzz-agent` exports 20 items of which 18 carry no doc comment, and 11 of the CLI's `*Cmd` enums are undocumented — both against the `AGENTS.md` rule that new public API must have doc comments. | `crates/buzz-cli/src/lib.rs:23-60`, `:79`, `:175`; `crates/buzz-agent/src/types.rs` |
| API-169 | **The write/read output envelopes are produced by shared helpers that most commands bypass.** `normalize_events` strips `sig` (`client.rs:1307-1323`) and is called by exactly two command modules (`feed.rs`, `messages.rs`); `normalize_write_response` (`client.rs:1420-1434`) has 47 call sites but is skipped by `mem`, `repos create` and the whole NIP-34 family; `print_create_response` (`client.rs:1401-1403`) is used by one command (`workflows create`). Consequence: `BuzzClient::submit_event` returns a raw `String` rather than a typed result, so normalization is each caller's choice. See DEBT-73 for the resulting contract drift. | `crates/buzz-cli/src/client.rs:1307-1323`, `:1401-1403`, `:1420-1434` |
| API-170 | **`buzz-acp`'s relay surface is a private reimplementation rather than a shared client.** `RestClient` (`relay.rs:261-424`) and a full WebSocket/AUTH stack (`relay.rs:3533-3620`, `:3865-3913`) duplicate `buzz-ws-client` without depending on it, and the two copies now differ in wire coverage — `buzz-ws-client` parses 7 inbound message types including `COUNT`, the acp copy declares 6 and never matches `"COUNT"`. See DEBT-59. | `crates/buzz-acp/src/relay.rs:261-424`, `:3533-3620` vs `crates/buzz-ws-client/src/message.rs` |
| API-171 | **Three new public classifiers were added to `buzz-core::kind` by the post-sync range and are worth noting for API stability**: `is_persona_shared_kind` (`kind.rs:192-194`), `is_unshared_persona_event` (`:205-216`), `persona_event_is_shared` (`:226-241`). All three are plain `fn`, breaking the `const fn` convention every other classifier in that module follows. | `crates/buzz-core/src/kind.rs:192-241` |
| API-172 | **A second post-sync change, `16d4ec33`, adds a mesh "Auto" collective-routing subsystem to `llm.rs` entirely below `buzz-agent`'s public surface — API-161's point about the crate's tiny public API still holds.** Every new item (`MeshCatalogObservation`, `MeshAutoState`, `PostError`, `resolve_openai_model`, `observe_mesh_virtual_model`, `cool_down_collective`, `openai_request_for_model`, `post_anthropic`, four free functions, seven constants) is private to `llm.rs`, which is itself a private module (`lib.rs:9`) — none of it is reachable as `buzz_agent::*`. `Config` did gain one new `pub` field, `prefer_mesh_for_auto: bool` (`config.rs:734`), so `buzz_agent::config::Config` now has 27 fields instead of 26; that field is the only externally-visible surface change from this commit. | `crates/buzz-agent/src/llm.rs:28-105`, `:1344-1362`; `config.rs:734` |

## WebSocket Protocol (NIP-01)

| Direction | Message | Purpose | Source |
|---|---|---|---|
| _pending verification in Phase 2_ | | | |

## HTTP Endpoints

| Method | Path | Auth | Handler | Source |
|---|---|---|---|---|
| _pending verification in Phase 2_ | | | | |

## Event Kinds

_Pending Phase 2 — full registry to be extracted from `crates/buzz-core/src/kind.rs`._

## CLI Surfaces

_Pending Phase 2 (`buzz-cli`, `buzz-admin`, `buzz-pairing-cli`, `buzz-test-client`)._

## MCP Tools

_Pending Phase 2 (`buzz-dev-mcp`)._

## Tauri IPC Commands

_Pending Phase 2 (`desktop/src-tauri/src/commands/`)._

---

# Phase 2 — Module Findings

## Module: buzz-core (`crates/buzz-core`)

### Aspect: API Surface

`buzz-core` is a library-only crate (no `main.rs`, no `src/bin/`). Its API is the `pub` surface of `crates/buzz-core/src/lib.rs` plus the 15 public modules it declares. Crate-level lints: `#![deny(unsafe_code)]` and `#![warn(missing_docs)]` (`crates/buzz-core/src/lib.rs:1-2`).

---

### 1. Public modules (`crates/buzz-core/src/lib.rs:9-38`)

| Module | Declared at | Purpose (per doc comment) |
|--------|-------------|---------------------------|
| `agent_turn_metric` | `lib.rs:9` | NIP-AM turn metric payload + encrypt/decrypt |
| `channel` | `lib.rs:11` | channel + membership enums |
| `engram` | `lib.rs:14` | NIP-AE slug grammar, conversation key, d-tag, body parse, envelope, head selection |
| `error` | `lib.rs:16` | relay-side error types |
| `event` | `lib.rs:18` | event wrapper with verification tracking |
| `filter` | `lib.rs:20` | NIP-01 subscription filter matching |
| `git_perms` | `lib.rs:22` | git ref patterns, protection rules, policy evaluation |
| `kind` | `lib.rs:24` | Buzz kind-number registry |
| `network` | `lib.rs:26` | SSRF-safe IP classification |
| `observer` | `lib.rs:28` | agent observer frame helpers |
| `pairing` | `lib.rs:30` | NIP-AB device pairing |
| `presence` | `lib.rs:32` | presence status types |
| `relay` | `lib.rs:34` | canonical relay runtime identities |
| `tenant` | `lib.rs:36` | tenant identity (community key) |
| `verification` | `lib.rs:38` | Schnorr signature + event id verification |

### 2. Crate-root re-exports (`lib.rs:40-45`)

| Re-export | Source |
|-----------|--------|
| `VerificationError` | `error::VerificationError` (`lib.rs:40`) |
| `StoredEvent` | `event::StoredEvent` (`lib.rs:41`) |
| `Event`, `EventId`, `Filter`, `Keys`, `Kind`, `PublicKey` | **re-exported from `nostr`** (`lib.rs:42`) |
| `PresenceStatus` | `presence::PresenceStatus` (`lib.rs:43`) |
| `normalize_host`, `CommunityId`, `TenantContext` | `tenant::…` (`lib.rs:44`) |
| `verify_event` | `verification::verify_event` (`lib.rs:45`) |

Feature-gated module `test_helpers` (`lib.rs:47-74`), active under `#[cfg(any(test, feature = "test-utils"))]`: `make_event(Kind)` (`lib.rs:55`), `make_event_with_keys(&Keys, Kind)` (`lib.rs:64`), `make_stored_event(Kind, Option<Uuid>)` (`lib.rs:72`).

---

### 3. Kind registry — complete constant table

**Verified counts (from `crates/buzz-core/src/kind.rs`):**

| Measure | Actual value | Evidence |
|---------|--------------|----------|
| `pub const … : u32` declarations in `kind.rs` | **134** | `grep "^pub const .*: u32 = "` over `kind.rs` |
| of those, range-boundary constants (not kinds) | **4** | `PARAM_REPLACEABLE_KIND_MIN` `kind.rs:392`, `PARAM_REPLACEABLE_KIND_MAX` `kind.rs:394`, `EPHEMERAL_KIND_MIN` `kind.rs:397`, `EPHEMERAL_KIND_MAX` `kind.rs:399` |
| **actual `KIND_*` / `RELAY_ADMIN_*` kind constants** | **130** | 134 − 4 |
| **entries in `ALL_KINDS`** | **127** | `kind.rs:566-693` (list body) |
| kind constants **absent** from `ALL_KINDS` | **3** | `KIND_AUTH` (22242, `kind.rs:77`), `KIND_NOSTR_IDENTITY_BINDING` (24243, `kind.rs:81`), `KIND_PUSH_LEASE` (30350, `kind.rs:109`) |
| duplicate names in `ALL_KINDS` | 0 | set comparison of the list body |
| duplicate integer values across all 130 constants | 0 | value-frequency check over the constant declarations |

Every kind constant, sorted by integer value. "In `ALL_KINDS`" = whether the name appears in the `ALL_KINDS` slice literal (`kind.rs:566-693`).

| Constant | Value | In `ALL_KINDS` | Declared at |
|---|---|---|---|
| `KIND_PROFILE` | 0 | yes | src/kind.rs:9 |
| `KIND_TEXT_NOTE` | 1 | yes | src/kind.rs:11 |
| `KIND_CONTACT_LIST` | 3 | yes | src/kind.rs:13 |
| `KIND_DELETION` | 5 | yes | src/kind.rs:56 |
| `KIND_REACTION` | 7 | yes | src/kind.rs:58 |
| `KIND_STREAM_MESSAGE` | 9 | yes | src/kind.rs:419 |
| `KIND_CHANNEL_METADATA` | 41 | yes | src/kind.rs:54 |
| `KIND_GIFT_WRAP` | 1059 | yes | src/kind.rs:60 |
| `KIND_FILE_METADATA` | 1063 | yes | src/kind.rs:62 |
| `KIND_GIT_PATCH` | 1617 | yes | src/kind.rs:549 |
| `KIND_GIT_PULL_REQUEST` | 1618 | yes | src/kind.rs:551 |
| `KIND_GIT_PR_UPDATE` | 1619 | yes | src/kind.rs:553 |
| `KIND_GIT_ISSUE` | 1621 | yes | src/kind.rs:555 |
| `KIND_GIT_STATUS_OPEN` | 1630 | yes | src/kind.rs:557 |
| `KIND_GIT_STATUS_MERGED` | 1631 | yes | src/kind.rs:559 |
| `KIND_GIT_STATUS_CLOSED` | 1632 | yes | src/kind.rs:561 |
| `KIND_GIT_STATUS_DRAFT` | 1633 | yes | src/kind.rs:563 |
| `KIND_REPORT` | 1984 | yes | src/kind.rs:267 |
| `KIND_NIP43_MEMBER_ADDED` | 8000 | yes | src/kind.rs:340 |
| `KIND_NIP43_MEMBER_REMOVED` | 8001 | yes | src/kind.rs:342 |
| `KIND_IA_ARCHIVED` | 8002 | yes | src/kind.rs:354 |
| `KIND_IA_UNARCHIVED` | 8003 | yes | src/kind.rs:356 |
| `KIND_NIP29_PUT_USER` | 9000 | yes | src/kind.rs:275 |
| `KIND_NIP29_REMOVE_USER` | 9001 | yes | src/kind.rs:277 |
| `KIND_NIP29_EDIT_METADATA` | 9002 | yes | src/kind.rs:279 |
| `KIND_NIP29_DELETE_EVENT` | 9005 | yes | src/kind.rs:281 |
| `KIND_NIP29_CREATE_GROUP` | 9007 | yes | src/kind.rs:283 |
| `KIND_NIP29_DELETE_GROUP` | 9008 | yes | src/kind.rs:285 |
| `KIND_NIP29_CREATE_INVITE` | 9009 | yes | src/kind.rs:287 |
| `KIND_NIP29_JOIN_REQUEST` | 9021 | yes | src/kind.rs:289 |
| `KIND_NIP29_LEAVE_REQUEST` | 9022 | yes | src/kind.rs:291 |
| `RELAY_ADMIN_ADD_MEMBER` | 9030 | yes | src/kind.rs:329 |
| `RELAY_ADMIN_REMOVE_MEMBER` | 9031 | yes | src/kind.rs:331 |
| `RELAY_ADMIN_CHANGE_ROLE` | 9032 | yes | src/kind.rs:333 |
| `RELAY_ADMIN_SET_WORKSPACE_PROFILE` | 9033 | yes | src/kind.rs:335 |
| `KIND_IA_ARCHIVE_REQUEST` | 9035 | yes | src/kind.rs:348 |
| `KIND_IA_UNARCHIVE_REQUEST` | 9036 | yes | src/kind.rs:350 |
| `KIND_MODERATION_BAN` | 9040 | yes | src/kind.rs:298 |
| `KIND_MODERATION_UNBAN` | 9041 | yes | src/kind.rs:300 |
| `KIND_MODERATION_TIMEOUT` | 9042 | yes | src/kind.rs:303 |
| `KIND_MODERATION_UNTIMEOUT` | 9043 | yes | src/kind.rs:305 |
| `KIND_MODERATION_RESOLVE_REPORT` | 9044 | yes | src/kind.rs:310 |
| `KIND_MUTE_LIST` | 10000 | yes | src/kind.rs:17 |
| `KIND_PIN_LIST` | 10001 | yes | src/kind.rs:22 |
| `KIND_NIP65_RELAY_LIST_METADATA` | 10002 | yes | src/kind.rs:27 |
| `KIND_BOOKMARK_LIST` | 10003 | yes | src/kind.rs:32 |
| `KIND_EMOJI_LIST` | 10030 | yes | src/kind.rs:34 |
| `KIND_AGENT_PROFILE` | 10100 | yes | src/kind.rs:87 |
| `KIND_NIP43_MEMBERSHIP_LIST` | 13534 | yes | src/kind.rs:338 |
| `KIND_IA_ARCHIVED_LIST` | 13535 | yes | src/kind.rs:358 |
| `KIND_PRESENCE_UPDATE` | 20001 | yes | src/kind.rs:403 |
| `KIND_TYPING_INDICATOR` | 20002 | yes | src/kind.rs:407 |
| `KIND_AUTH` | 22242 | **NO** | src/kind.rs:77 |
| `KIND_PAIRING` | 24134 | yes | src/kind.rs:405 |
| `KIND_AGENT_OBSERVER_FRAME` | 24200 | yes | src/kind.rs:409 |
| `KIND_BLOSSOM_AUTH` | 24242 | yes | src/kind.rs:79 |
| `KIND_NOSTR_IDENTITY_BINDING` | 24243 | **NO** | src/kind.rs:81 |
| `KIND_HUDDLE_REACTION` | 24810 | yes | src/kind.rs:412 |
| `KIND_HTTP_AUTH` | 27235 | yes | src/kind.rs:83 |
| `KIND_NIP43_LEAVE_REQUEST` | 28936 | yes | src/kind.rs:344 |
| `KIND_FOLLOW_SET` | 30000 | yes | src/kind.rs:39 |
| `KIND_BOOKMARK_SET` | 30003 | yes | src/kind.rs:43 |
| `KIND_LONG_FORM` | 30023 | yes | src/kind.rs:66 |
| `KIND_EMOJI_SET` | 30030 | yes | src/kind.rs:52 |
| `KIND_READ_STATE` | 30078 | yes | src/kind.rs:75 |
| `KIND_AGENT_ENGRAM` | 30174 | yes | src/kind.rs:94 |
| `KIND_PERSONA` | 30175 | yes | src/kind.rs:183 |
| `KIND_TEAM` | 30176 | yes | src/kind.rs:250 |
| `KIND_MANAGED_AGENT` | 30177 | yes | src/kind.rs:259 |
| `KIND_EVENT_REMINDER` | 30300 | yes | src/kind.rs:102 |
| `KIND_USER_STATUS` | 30315 | yes | src/kind.rs:70 |
| `KIND_PUSH_LEASE` | 30350 | **NO** | src/kind.rs:109 |
| `KIND_GIT_REPO_ANNOUNCEMENT` | 30617 | yes | src/kind.rs:545 |
| `KIND_GIT_REPO_STATE` | 30618 | yes | src/kind.rs:547 |
| `KIND_WORKFLOW_DEF` | 30620 | yes | src/kind.rs:382 |
| `KIND_DM_VISIBILITY` | 30622 | yes | src/kind.rs:389 |
| `KIND_NIP29_GROUP_METADATA` | 39000 | yes | src/kind.rs:362 |
| `KIND_NIP29_GROUP_ADMINS` | 39001 | yes | src/kind.rs:364 |
| `KIND_NIP29_GROUP_MEMBERS` | 39002 | yes | src/kind.rs:366 |
| `KIND_NIP29_GROUP_ROLES` | 39003 | yes | src/kind.rs:368 |
| `KIND_THREAD_SUMMARY` | 39005 | yes | src/kind.rs:375 |
| `KIND_WINDOW_BOUNDS` | 39006 | yes | src/kind.rs:379 |
| `KIND_STREAM_MESSAGE_V2` | 40002 | yes | src/kind.rs:421 |
| `KIND_STREAM_MESSAGE_EDIT` | 40003 | yes | src/kind.rs:423 |
| `KIND_STREAM_MESSAGE_PINNED` | 40004 | yes | src/kind.rs:425 |
| `KIND_STREAM_MESSAGE_BOOKMARKED` | 40005 | yes | src/kind.rs:427 |
| `KIND_STREAM_MESSAGE_SCHEDULED` | 40006 | yes | src/kind.rs:429 |
| `KIND_STREAM_REMINDER` | 40007 | yes | src/kind.rs:431 |
| `KIND_STREAM_MESSAGE_DIFF` | 40008 | yes | src/kind.rs:433 |
| `KIND_SYSTEM_MESSAGE` | 40099 | yes | src/kind.rs:437 |
| `KIND_CANVAS` | 40100 | yes | src/kind.rs:435 |
| `KIND_CHANNEL_SUMMARY` | 40901 | yes | src/kind.rs:441 |
| `KIND_PRESENCE_SNAPSHOT` | 40902 | yes | src/kind.rs:443 |
| `KIND_DM_CREATED` | 41001 | yes | src/kind.rs:453 |
| `KIND_DM_OPEN` | 41010 | yes | src/kind.rs:447 |
| `KIND_DM_ADD_MEMBER` | 41011 | yes | src/kind.rs:449 |
| `KIND_DM_HIDE` | 41012 | yes | src/kind.rs:451 |
| `KIND_PRODUCT_FEEDBACK` | 42000 | yes | src/kind.rs:271 |
| `KIND_JOB_REQUEST` | 43001 | yes | src/kind.rs:458 |
| `KIND_JOB_ACCEPTED` | 43002 | yes | src/kind.rs:460 |
| `KIND_JOB_PROGRESS` | 43003 | yes | src/kind.rs:462 |
| `KIND_JOB_RESULT` | 43004 | yes | src/kind.rs:464 |
| `KIND_JOB_CANCEL` | 43005 | yes | src/kind.rs:466 |
| `KIND_JOB_ERROR` | 43006 | yes | src/kind.rs:468 |
| `KIND_MEMBER_ADDED_NOTIFICATION` | 44100 | yes | src/kind.rs:472 |
| `KIND_MEMBER_REMOVED_NOTIFICATION` | 44101 | yes | src/kind.rs:476 |
| `KIND_AGENT_TURN_METRIC` | 44200 | yes | src/kind.rs:485 |
| `KIND_FORUM_POST` | 45001 | yes | src/kind.rs:490 |
| `KIND_FORUM_VOTE` | 45002 | yes | src/kind.rs:492 |
| `KIND_FORUM_COMMENT` | 45003 | yes | src/kind.rs:494 |
| `KIND_WORKFLOW_TRIGGERED` | 46001 | yes | src/kind.rs:504 |
| `KIND_WORKFLOW_STEP_STARTED` | 46002 | yes | src/kind.rs:506 |
| `KIND_WORKFLOW_STEP_COMPLETED` | 46003 | yes | src/kind.rs:508 |
| `KIND_WORKFLOW_STEP_FAILED` | 46004 | yes | src/kind.rs:510 |
| `KIND_WORKFLOW_COMPLETED` | 46005 | yes | src/kind.rs:512 |
| `KIND_WORKFLOW_FAILED` | 46006 | yes | src/kind.rs:514 |
| `KIND_WORKFLOW_CANCELLED` | 46007 | yes | src/kind.rs:516 |
| `KIND_WORKFLOW_APPROVAL_REQUESTED` | 46010 | yes | src/kind.rs:518 |
| `KIND_WORKFLOW_APPROVAL_GRANTED` | 46011 | yes | src/kind.rs:520 |
| `KIND_WORKFLOW_APPROVAL_DENIED` | 46012 | yes | src/kind.rs:522 |
| `KIND_WORKFLOW_TRIGGER` | 46020 | yes | src/kind.rs:498 |
| `KIND_APPROVAL_GRANT` | 46030 | yes | src/kind.rs:500 |
| `KIND_APPROVAL_DENY` | 46031 | yes | src/kind.rs:502 |
| `KIND_AUDIT_ENTRY` | 48001 | yes | src/kind.rs:528 |
| `KIND_HUDDLE_STARTED` | 48100 | yes | src/kind.rs:530 |
| `KIND_HUDDLE_PARTICIPANT_JOINED` | 48101 | yes | src/kind.rs:532 |
| `KIND_HUDDLE_PARTICIPANT_LEFT` | 48102 | yes | src/kind.rs:534 |
| `KIND_HUDDLE_ENDED` | 48103 | yes | src/kind.rs:536 |
| `KIND_HUDDLE_GUIDELINES` | 48106 | yes | src/kind.rs:538 |
| `KIND_MEDIA_UPLOAD` | 49001 | yes | src/kind.rs:542 |

#### Range-boundary constants

| Constant | Value | Declared at |
|---|---|---|
| `PARAM_REPLACEABLE_KIND_MIN` | 30000 | src/kind.rs:392 |
| `PARAM_REPLACEABLE_KIND_MAX` | 39999 | src/kind.rs:394 |
| `EPHEMERAL_KIND_MIN` | 20000 | src/kind.rs:397 |
| `EPHEMERAL_KIND_MAX` | 29999 | src/kind.rs:399 |

#### Kind-set constants (`&[u32]` slices)

| Constant | Members | Declared at |
|---|---|---|
| `AUTHOR_ONLY_KINDS` | `KIND_EVENT_REMINDER` (30300), `KIND_PUSH_LEASE` (30350) | src/kind.rs:120 |
| `RESULT_GATED_KINDS` | `KIND_DM_VISIBILITY` (30622), `KIND_AGENT_TURN_METRIC` (44200) | src/kind.rs:129 |
| `P_GATED_KINDS` | `KIND_AGENT_OBSERVER_FRAME` (24200), `KIND_MEMBER_ADDED_NOTIFICATION` (44100), `KIND_MEMBER_REMOVED_NOTIFICATION` (44101), `KIND_GIFT_WRAP` (1059), `KIND_DM_VISIBILITY` (30622), `KIND_AGENT_TURN_METRIC` (44200) — 6 entries | src/kind.rs:146-156 |
| `ALL_KINDS` | 127 entries (see table above) | src/kind.rs:566-693 |

#### Classification / helper functions (`kind.rs`)

| Function | Signature | Rule as coded | file:line |
|---|---|---|---|
| `is_persona_shared_kind` **(added by `07d0265c`)** | `fn(u32) -> bool` | `kind == KIND_PERSONA` (30175) — not `const fn` | `kind.rs:192-194` |
| `is_unshared_persona_event` **(added by `07d0265c`)** | `fn(&nostr::Event, &[u8]) -> bool` | true only when all three hold: kind is 30175, requester ≠ author, and the event is not shared | `kind.rs:205-216` |
| `persona_event_is_shared` **(added by `07d0265c`)** | `fn(&nostr::Event) -> bool` | exactly one `["shared","true"]` tag of **exactly two** elements; any other `shared` shape or value fails closed | `kind.rs:226-241` |
| `is_moderation_command_kind` | `const fn(u32) -> bool` | matches 9040, 9041, 9042, 9043, 9044 | `kind.rs:316-325` |
| `is_ephemeral` | `const fn(u32) -> bool` | `kind >= 20000 && kind <= 29999` | `kind.rs:697-699` |
| `is_replaceable` | `const fn(u32) -> bool` | `matches!(kind, 0 \| 3 \| 41 \| 10000..=19999)` | `kind.rs:704-706` |
| `is_parameterized_replaceable` | `const fn(u32) -> bool` | `kind >= 30000 && kind <= 39999` | `kind.rs:711-713` |
| `is_workflow_execution_kind` | `const fn(u32) -> bool` | `46001 ..= 46012` (`KIND_WORKFLOW_TRIGGERED..=KIND_WORKFLOW_APPROVAL_DENIED`) | `kind.rs:717-719` |
| `is_relay_admin_kind` | `const fn(u32) -> bool` | matches 9030, 9031, 9032, 9033 | `kind.rs:723-732` |
| `is_identity_archive_request_kind` | `const fn(u32) -> bool` | matches 9035, 9036 only (relay-signed 8002/8003/13535 excluded per doc `kind.rs:734-737`) | `kind.rs:738-740` |
| `is_command_kind` | `const fn(u32) -> bool` | matches `KIND_WORKFLOW_DEF` 30620, `KIND_DM_OPEN` 41010, `KIND_DM_ADD_MEMBER` 41011, `KIND_DM_HIDE` 41012, `KIND_WORKFLOW_TRIGGER` 46020, `KIND_APPROVAL_GRANT` 46030, `KIND_APPROVAL_DENY` 46031 | `kind.rs:743-755` |
| `is_relay_only_kind` | `const fn(u32) -> bool` | matches 13534, 40901, 40902, 30622, 39005, 39006 | `kind.rs:758-769` |
| `event_kind_u32` | `fn(&nostr::Event) -> u32` | `event.kind.as_u16() as u32` | `kind.rs:772-774` |
| `event_kind_i32` | `fn(&nostr::Event) -> i32` | `event.kind.as_u16() as i32` | `kind.rs:778-780` |

Compile-time assertions in the module body (not functions, but part of the contract): `kind.rs:783-821` — range membership for `KIND_AGENT_PROFILE`, `KIND_PERSONA`, `KIND_TEAM`, `KIND_MANAGED_AGENT`, `KIND_WORKFLOW_DEF`, `KIND_EVENT_REMINDER`, `KIND_DM_VISIBILITY`, `KIND_THREAD_SUMMARY`, `KIND_WINDOW_BOUNDS`, the two NIP-34 addressable kinds, u16 fits for `KIND_AUTH`/`KIND_CANVAS`/`KIND_HUDDLE_GUIDELINES`/`KIND_AGENT_TURN_METRIC`/`KIND_REPORT`/`KIND_MODERATION_RESOLVE_REPORT`, and the moderation-command classification.

---

### 4. Verification + event API

| Item | Signature | file:line |
|---|---|---|
| `verify_event` | `fn(&Event) -> Result<(), VerificationError>` | `src/verification.rs:11-32` |
| `StoredEvent::new` | `fn(nostr::Event, Option<Uuid>) -> Self` | `src/event.rs:23-30` |
| `StoredEvent::with_received_at` | `fn(nostr::Event, DateTime<Utc>, Option<Uuid>, bool) -> Self` | `src/event.rs:38-48` |
| `StoredEvent::is_verified` | `fn(&self) -> bool` | `src/event.rs:33-35` |

### 5. Filter API (`src/filter.rs`)

| Item | Signature | file:line |
|---|---|---|
| `filters_match` | `fn(&[Filter], &StoredEvent) -> bool` | `filter.rs:10-13` |
| `reader_authorized_for_event` | `fn(&nostr::Event, &str) -> bool` | `filter.rs:23-33` |
| `filter_match_one` | private | `filter.rs:35-104` |

### 6. Network API (`src/network.rs`)

`is_private_ip(&std::net::IpAddr) -> bool` (`network.rs:46-95`) — the crate's only network helper; no other public items. `c26bf594` added a private helper `embedded_ipv4` (`network.rs:15-20`) and two private prefix consts (`:7`, `:10`), so the *public* surface is still exactly one function.

### 7. Tenant API (`src/tenant.rs`)

| Item | Signature | file:line |
|---|---|---|
| `CommunityId::from_uuid` | `const fn(Uuid) -> Self` | `tenant.rs:45-47` |
| `CommunityId::as_uuid` | `const fn(&self) -> &Uuid` | `tenant.rs:50-52` |
| `TenantContext::resolved` | `fn(CommunityId, impl Into<String>) -> Self` | `tenant.rs:79-84` |
| `TenantContext::community` | `const fn(&self) -> CommunityId` | `tenant.rs:87-89` |
| `TenantContext::host` | `fn(&self) -> &str` | `tenant.rs:95-97` |
| `normalize_host` | `#[must_use] fn(&str) -> String` | `tenant.rs:121-139` |
| `relay_url_authority` | `#[must_use] fn(&str) -> String` | `tenant.rs:156-172` |

### 8. Relay identity API (`src/relay.rs`)

`normalize_relay_url(&str) -> Result<String, NormalizeRelayUrlError>` (`relay.rs:37-78`). Doc explicitly states it is **not** the NIP-42 AUTH comparison helper (that lives in `buzz-auth`) — `relay.rs:28-32`.

### 9. Channel API (`src/channel.rs`)

`canonical_channel_name(&str) -> &str` (`channel.rs:15-19`); `ChannelVisibility`, `ChannelType`, `MemberRole` with `as_str`, `Display`, `FromStr` on each, plus `MemberRole::is_elevated` (`:134-136`), `permission_level` (`:142-150`), `has_at_least` (`:155-157`).

### 10. Presence API (`src/presence.rs`)

`PresenceStatus` + `as_str(&self) -> &'static str` (`presence.rs:22-28`) + `Display` (`:31-35`).

### 11. Observer API (`src/observer.rs`)

| Item | Value / signature | file:line |
|---|---|---|
| `OBSERVER_AGENT_TAG` | `"agent"` | `observer.rs:13` |
| `OBSERVER_FRAME_TAG` | `"frame"` | `observer.rs:15` |
| `OBSERVER_FRAME_TELEMETRY` | `"telemetry"` | `observer.rs:17` |
| `OBSERVER_FRAME_CONTROL` | `"control"` | `observer.rs:19` |
| `NIP44_MIN_CONTENT_LEN` | `132` | `observer.rs:21` |
| `NIP44_MAX_CONTENT_LEN` | `87_472` | `observer.rs:23` |
| `OBSERVER_MAX_PLAINTEXT_LEN` | `65_535` | `observer.rs:25` |
| `content_looks_like_nip44` | `fn(&str) -> bool` | `observer.rs:53-55` |
| `encrypt_observer_payload` | `fn<T: Serialize>(&Keys, &PublicKey, &T) -> Result<String, ObserverPayloadError>` | `observer.rs:58-81` |
| `decrypt_observer_payload` | `fn<T: DeserializeOwned>(&Keys, &Event) -> Result<T, ObserverPayloadError>` | `observer.rs:84-110` |

### 12. Agent turn metric API (`src/agent_turn_metric.rs`)

| Item | Signature | file:line |
|---|---|---|
| `AgentTurnMetricError` | alias of `ObserverPayloadError` | `agent_turn_metric.rs:15` |
| `AgentTurnMetricPayload::validate` | `fn(&self) -> Result<(), ObserverPayloadError>` | `:140-169` |
| `encrypt_agent_turn_metric` | `fn(&Keys, &PublicKey, &AgentTurnMetricPayload) -> Result<String, ObserverPayloadError>` | `:169-176` |
| `decrypt_agent_turn_metric` | `fn(&Keys, &Event) -> Result<AgentTurnMetricPayload, ObserverPayloadError>` | `:185-191` |

### 13. Engram API (`src/engram.rs`)

| Item | Value / signature | file:line |
|---|---|---|
| `CORE_SLUG` | `"core"` | `engram.rs:20` |
| `D_TAG_DOMAIN` | `b"agent-memory/v1/d-tag"` | `engram.rs:24` |
| `NIP44_PLAINTEXT_MAX` | `65_535` | `engram.rs:28` |
| `SLUG_MAX_LEN` | `255` | `engram.rs:31` |
| `validate_slug` | `fn(&str) -> Result<(), EngramError>` | `:67-92` |
| `normalize_slug` | `fn(&str) -> Result<String, EngramError>` | `:123-131` |
| `conversation_key` | `fn(&SecretKey, &PublicKey) -> ConversationKey` | `:136-138` |
| `d_tag` | `fn(&ConversationKey, &str) -> String` | `:144-155` |
| `Body::slug` / `is_tombstone` / `to_json_bytes` / `from_json_bytes` | see data-model doc | `:175`, `:183`, `:189`, `:216` |
| `extract_refs` | `fn(&str) -> Vec<String>` | `:384-430` |
| `build_event` | `fn(&Keys, &PublicKey, &Body, u64) -> Result<Event, EngramError>` | `:435-476` |
| `validate_and_decrypt` | `fn(&Event, &PublicKey, &PublicKey, &SecretKey, &PublicKey) -> Result<Body, EngramError>` | `:488-558` |
| `select_head` | `fn<I: IntoIterator<Item = Event>>(I) -> Option<Event>` | `:564-583` |
| `monotonic_created_at` | `fn(u64, Option<u64>) -> u64` | `:588-593` |
| `Listing` | struct | `:598-603` |

Private helpers (not exported): `validate_segment` (`:94-112`), `is_lower_alnum` (`:114-116`), `write_json_string` (`:263-281`), `parse_strict_json` (`:283-380`).

### 14. Git permissions API (`src/git_perms.rs`)

| Item | Value / signature | file:line |
|---|---|---|
| `MAX_PROTECTION_RULES` | `50` | `git_perms.rs:19` |
| `MAX_PATTERN_LENGTH` | `256` | `git_perms.rs:21` |
| `MAX_WILDCARDS_PER_PATTERN` | `3` | `git_perms.rs:23` |
| `RefPattern::parse` | `fn(&str) -> Result<Self, PatternError>` | `:83-146` |
| `RefPattern::matches` | `fn(&self, &str) -> bool` | `:147-178` |
| `RefPattern::as_str` | `fn(&self) -> &str` | `:181-183` |
| `UpdateKind::classify` | `fn(&str, &str, bool) -> Self` | `:212-221` |
| `parse_protection_tag` | `fn(&[&str]) -> Result<ProtectionRule, RuleParseError>` | `:297-300` |
| `parse_protection_tag_with_warnings` | `fn(&[&str]) -> Result<(ProtectionRule, Vec<String>), RuleParseError>` | `:303-362` |
| `parse_protection_tags` | `fn(&[Vec<String>]) -> Result<ParsedProtection, RuleParseError>` | `:379-399` |
| `default_min_role` | `fn(&str, UpdateKind) -> MemberRole` | `:403-425` |
| `EffectiveRules::for_ref` | `fn(&str, &[ProtectionRule]) -> Self` | `:447-475` |
| `evaluate_ref_update` | `fn(&RefUpdate, MemberRole, &[ProtectionRule]) -> Result<(), Denial>` | `:508-578` |
| `evaluate_push` | `fn(&[RefUpdate], MemberRole, &[ProtectionRule]) -> Result<(), Vec<Denial>>` | `:584-597` |

### 15. Pairing API (`src/pairing/`)

Re-exports from `pairing/mod.rs:27-29`: `QrPayload`, `PairingSession`, `Role`, `SessionState`, `AbortReason`, `PairingMessage`, `PayloadType`; plus `PairingError` (`mod.rs:35-71`) and submodules `crypto`, `qr`, `session`, `types` (`mod.rs:22-25`).

`pairing::crypto` (all pure):

| Function | Signature | file:line |
|---|---|---|
| `derive_session_id` | `fn(&[u8; 32]) -> [u8; 32]` | `crypto.rs:54-56` |
| `derive_sas` | `fn(&[u8; 32], &[u8; 32]) -> (u32, [u8; 32])` | `crypto.rs:70-75` |
| `derive_transcript_hash` | `fn(&[u8;32], &[u8;32], &[u8;32], &[u8;32], &[u8;32]) -> [u8;32]` | `crypto.rs:89-105` |
| `format_sas` | `fn(u32) -> String` (zero-padded 6 digits; doctest at `:110-114`) | `crypto.rs:116-118` |
| `ct_eq` | `fn(&[u8;32], &[u8;32]) -> bool` (constant-time) | `crypto.rs:126-129` |

`pairing::qr`: `encode_qr(&QrPayload) -> String` (`qr.rs:79-93`, doctest `:66-77`), `decode_qr(&str) -> Result<QrPayload, PairingError>` (`qr.rs:104-206`).

`pairing::session::PairingSession` — 22 public methods:

| Method | Role | Signature | file:line |
|---|---|---|---|
| `new_source` | Source | `fn(String) -> (Self, QrPayload)` | `session.rs:112-146` |
| `handle_offer` | Source | `fn(&mut self, &Event) -> Result<String, PairingError>` | `:149-197` |
| `confirm_sas` | Source | `fn(&mut self) -> Result<Event, PairingError>` | `:200-224` |
| `send_payload` | Source | `fn(&mut self, PayloadType, Zeroizing<String>) -> Result<Event, PairingError>` | `:227-251` |
| `handle_complete` | Source | `fn(&mut self, &Event) -> Result<(), PairingError>` | `:254-281` |
| `new_target` | Target | `fn(&QrPayload) -> Result<(Self, Event), PairingError>` | `:286-326` |
| `handle_sas_confirm` | Target | `fn(&mut self, &Event) -> Result<String, PairingError>` | `:329-373` |
| `confirm_target_sas` | Target | `fn(&mut self) -> Result<(), PairingError>` | `:376-382` |
| `handle_payload` | Target | `fn(&mut self, &Event) -> Result<(PayloadType, Zeroizing<String>), PairingError>` | `:388-409` |
| `send_complete` | Target | `fn(&mut self) -> Result<Event, PairingError>` | `:412-421` |
| `abort` | both | `fn(&mut self, AbortReason) -> Result<Option<Event>, PairingError>` | `:430-445` |
| `handle_abort` | both | `fn(&mut self, &Event) -> Result<AbortReason, PairingError>` | `:448-473` |
| `is_expired` / `state` / `role` / `pubkey` / `relay_urls` / `sas_code` / `sign_event` / `qr_uri` | accessors | — | `:476`, `:481`, `:486`, `:491`, `:496`, `:501`, `:510`, `:517` |

Test-only methods in a `#[cfg(test)] impl PairingSession` block (`session.rs:530-544`): `has_processed` (`:536`), `set_timeout` (`:541`) — not part of the published API. The type is split across four `impl` blocks: source-side (`:109`), target-side (`:282`), shared/accessors (`:424`), and private helpers (`:546`).

---

### 16. ARCHITECTURE.md verification

| Doc claim | Location | Actual code | Verdict |
|---|---|---|---|
| "`buzz-core` defines all 81 kinds as `pub const KIND_*: u32`" | `ARCHITECTURE.md:142` | 130 kind constants (134 `u32` consts − 4 range bounds) in `crates/buzz-core/src/kind.rs` | **stale** — undercounts by 49 |
| "`pub const ALL_KINDS: &[u32]  // 80 entries (KIND_AUTH excluded — never stored)`" | `ARCHITECTURE.md:346` | `ALL_KINDS` has **127** entries (`kind.rs:566-693`) | **stale** count |
| "KIND_AUTH excluded" | `ARCHITECTURE.md:346` | Correct: `KIND_AUTH` (22242, `kind.rs:77`) is **not** in `ALL_KINDS` | **accurate**, but incomplete — `KIND_NOSTR_IDENTITY_BINDING` (24243, `kind.rs:81`) and `KIND_PUSH_LEASE` (30350, `kind.rs:109`) are also excluded and are not mentioned |

No code comment in `kind.rs` explains why `KIND_NOSTR_IDENTITY_BINDING` or `KIND_PUSH_LEASE` are omitted from `ALL_KINDS`; their doc comments say "ephemeral, not stored" (`kind.rs:80`) and "parameterized replaceable, author-only" (`kind.rs:104-108`) respectively. Note that `KIND_BLOSSOM_AUTH` (24242) and `KIND_HTTP_AUTH` (27235) carry similar "not stored" doc comments (`kind.rs:78`, `:82`) yet **are** present in `ALL_KINDS` (`kind.rs:626`, `:629`), so "never stored" is not the discriminating rule in the code as written.


## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: API Surface

Crate root declares three public modules and glob-re-exports the builders
(`crates/buzz-sdk/src/lib.rs:15-19`):

```rust
pub mod builders;   // 51 public builder/helper fns
pub mod mentions;   // 7 public fns + 1 struct + 1 const
pub mod nip_oa;     // 3 public fns
pub use builders::*;
pub use buzz_core::kind;               // lib.rs:22
```

Crate-level lints: `#![deny(unsafe_code)]`, `#![warn(missing_docs)]`
(`crates/buzz-sdk/src/lib.rs:1-2`).

**No REST/WS helpers exist.** The crate has no HTTP or WebSocket client, no
async, and no network dependency; the module doc states "No keys are held here.
No network calls are made." (`crates/buzz-sdk/src/lib.rs:13`). Dependency list
confirms it (`crates/buzz-sdk/Cargo.toml:10-16`).

---

### 1. Builder → kind → tags → content

Signature abbreviations: all builders return `Result<EventBuilder, SdkError>`.
`R` = required, `O` = optional.

| # | Builder (signature) | Kind | Required tags | Optional tags | Content |
|---|---|---|---|---|---|
| 1 | `build_message(channel_id: Uuid, content: &str, thread_ref: Option<&ThreadRef>, mentions: &[&str], broadcast: bool, media_tags: &[Vec<String>])` `builders.rs:219` | 9 | `h` | `e`(NIP-10), `p`*, `broadcast`, imeta | text ≤64 KiB |
| 2 | `build_agent_observer_frame(recipient_pubkey: &str, agent_pubkey: &str, frame: &str, encrypted_content: &str)` `builders.rs:245` | 24200 | `p`, `agent`, `frame` | — | NIP-44 v2 ciphertext |
| 3 | `build_forum_post(channel_id, content, mentions, media_tags)` `builders.rs:278` | 45001 | `h` | `p`*, imeta | text ≤64 KiB |
| 4 | `build_forum_comment(channel_id, content, thread_ref: &ThreadRef, mentions, media_tags)` `builders.rs:292` | 45003 | `h`, `e` | `p`*, imeta | text ≤64 KiB |
| 5 | `build_diff_message(channel_id, content, diff_meta: &DiffMeta, thread_ref: Option<&ThreadRef>)` `builders.rs:308` | 40008 | `h`, `repo`, `commit` | `file`, `parent-commit`, `branch`, `pr`, `l`, `description`, `truncated`, `alt`, `e` | diff ≤60 KiB |
| 6 | `build_edit(channel_id, target_event_id: EventId, new_content)` `builders.rs:378` | 40003 | `h`, `e` | — | text ≤64 KiB |
| 7 | `build_delete_message(channel_id, target_event_id)` `builders.rs:403` | 9005 | `h`, `e` | — | `""` |
| 8 | `build_delete_message_with_options(channel_id, target_event_id, options: DeleteMessageOptions)` `builders.rs:411` | 9005 | `h`, `e` | `action_id`, `reason_code`, `public_reason` | `""` |
| 9 | `build_delete_compat(channel_id, target_event_id)` `builders.rs:434` | 5 | `h`, `e` | — | `""` |
| 10 | `build_vote(channel_id, target_event_id, direction: VoteDirection)` `builders.rs:446` | 45002 | `h`, `e` | — | `"+"`/`"-"` |
| 11 | `build_reaction(target_event_id, emoji: &str)` `builders.rs:463` | 7 | `e` | — | emoji ≤64 chars |
| 12 | `build_custom_emoji_reaction(target_event_id, shortcode, url)` `builders.rs:479` | 7 | `e`, `emoji` | — | `":shortcode:"` |
| 13 | `build_remove_reaction(reaction_event_id)` `builders.rs:495` | 5 | `e` | — | `""` |
| 14 | `build_custom_emoji_set(emojis: &[CustomEmoji])` `builders.rs:511` | 30030 | `d`=`buzz:custom-emoji` | `emoji`* | `""` |
| 15 | `build_set_canvas(channel_id, content)` `builders.rs:529` | 40100 | `h` | — | canvas text (unbounded) |
| 16 | `build_profile(display_name, name, picture, about, nip05: Option<&str>)` `builders.rs:537` | 0 | — | — | JSON object of present fields |
| 17 | `build_add_member(channel_id, target_pubkey, role: Option<MemberRole>)` `builders.rs:565` | 9000 | `h`, `p` | `role` | `""` |
| 18 | `build_remove_member(channel_id, target_pubkey)` `builders.rs:582` | 9001 | `h`, `p` | — | `""` |
| 19 | `build_leave(channel_id)` `builders.rs:595` | 9022 | `h` | — | `""` |
| 20 | `build_update_channel(channel_id, name, about, visibility: Option<&str>, ttl: Option<Option<i32>>)` `builders.rs:604` | 9002 | `h` + ≥1 of the four | `name`, `about`, `visibility`, `ttl` | `""` |
| 21 | `build_set_topic(channel_id, topic)` `builders.rs:652` | 9002 | `h`, `topic` | — | `""` |
| 22 | `build_set_purpose(channel_id, purpose)` `builders.rs:661` | 9002 | `h`, `purpose` | — | `""` |
| 23 | `build_create_channel(channel_id, name, visibility: Option<Visibility>, channel_type: Option<ChannelKind>, about, ttl: Option<i32>)` `builders.rs:674` | 9007 | `h`, `name` | `visibility`, `channel_type`, `about`, `ttl` | `""` |
| 24 | `build_join(channel_id)` `builders.rs:703` | 9021 | `h` | — | `""` |
| 25 | `build_archive(channel_id)` `builders.rs:709` | 9002 | `h`, `archived=true` | — | `""` |
| 26 | `build_unarchive(channel_id)` `builders.rs:718` | 9002 | `h`, `archived=false` | — | `""` |
| 27 | `build_delete_channel(channel_id)` `builders.rs:727` | 9008 | `h` | — | `""` |
| 28 | `build_note(content, reply_to_event_id: Option<EventId>)` `builders.rs:738` | 1 | — | `e` (`reply` marker) | text ≤64 KiB |
| 29 | `build_contact_list(contacts: &[(&str, Option<&str>, Option<&str>)])` `builders.rs:764` | 3 | — | `p`* (≤10 000) | `""` |
| 30 | `build_repo_announcement(repo_id, name, description, clone_urls: &[&str], web_url, relays: &[&str])` `builders.rs:834` | 30617 | `d` | `name`, `description`, `clone`, `web`, `relays` | `""` |
| 31 | `build_repo_announcement_with_tags(repo_id, content, tags: Vec<Tag>)` `builders.rs:952` | 30617 | `d` | all caller tags preserved except `d` | caller content |
| 32 | `build_git_patch(repo: &GitRepoCoord, content, meta: &GitPatchMeta)` `builders.rs:1013` | 1617 | `a`, `p`(owner) | `r`(euc), `p`*, `e`, `t`, `commit`, `r`, `parent-commit`, `commit-pgp-sig`, `committer` | patch ≤60 KiB, non-blank |
| 33 | `build_git_issue(repo, subject, content, meta: &GitIssueMeta)` `builders.rs:1081` | 1621 | `a`, `p`(owner), `subject` | `p`*, `t`* | markdown ≤64 KiB |
| 34 | `build_git_status(status: GitStatus, content, meta: &GitStatusMeta)` `builders.rs:1222` | 1630/1631/1632/1633 | `e`(root) | `e`(reply), `p`*, `a`, `r`, `q`*, `merge-commit`, `applied-as-commits` | markdown ≤64 KiB |
| 35 | `build_git_pull_request(repo, content, meta: &GitPullRequestMeta)` `builders.rs:1330` | 1618 | `a`, `p`(owner), `subject`, `c`, `clone` | `r`, `p`*, `t`*, `h`, `branch-name`, `merge-base`, `e` | markdown ≤64 KiB |
| 36 | `build_git_pr_update(repo, content, meta: &GitPrUpdateMeta)` `builders.rs:1416` | 1619 | `a`, `p`(owner), `E`, `P`, `c`, `clone` | `r`, `p`*, `merge-base` | markdown ≤64 KiB |
| 37 | `build_workflow_def(channel_id, workflow_id, yaml)` `builders.rs:1463` | 30620 | `d`, `h` | — | YAML ≤64 KiB |
| 38 | `build_workflow_update(channel_id, workflow_id, yaml)` `builders.rs:1481` | 30620 | `d`, `h` | — | YAML ≤64 KiB |
| 39 | `build_workflow_delete(author_pubkey, workflow_id)` `builders.rs:1498` | 5 | `a` (`30620:pk:uuid`) | — | `""` |
| 40 | `build_workflow_trigger(workflow_id)` `builders.rs:1511` | 46020 | `d` | — | `""` |
| 41 | `build_workflow_approval(token_hash, approved: bool, note)` `builders.rs:1522` | 46030 / 46031 | `d` | — | note (unbounded) |
| 42 | `build_dm_open(pubkeys: &[&str])` `builders.rs:1544` | 41010 | `p` ×1–8 | — | `""` |
| 43 | `build_dm_add_member(channel_id, pubkey)` `builders.rs:1559` | 41011 | `h`, `p` | — | `""` |
| 44 | `build_presence_update(status: &str)` `builders.rs:1570` | 20001 | `status` | — | status string |
| 45 | `build_moderation_ban(target_pubkey, expires_at: Option<u64>, reason: Option<&str>)` `builders.rs:1597` | 9040 | `p` | `expiration`, `reason` | `""` |
| 46 | `build_moderation_unban(target_pubkey)` `builders.rs:1614` | 9041 | `p` | — | `""` |
| 47 | `build_moderation_timeout(target_pubkey, expires_at: u64, reason)` `builders.rs:1623` | 9042 | `p`, `expiration` | `reason` | `""` |
| 48 | `build_moderation_untimeout(target_pubkey)` `builders.rs:1640` | 9043 | `p` | — | `""` |
| 49 | `build_moderation_resolve_report(report_event_id, status, action, reason)` `builders.rs:1654` | 9044 | `report`, `status`, `action` | `reason` | `""` |
| 50 | `build_archive_identity_request(target_pubkey, content, reason, replaced_by, auth: Option<&[String;4]>)` `builders.rs:1788` | 9035 | `-`, `p` | `reason`, `replaced-by`, `auth` | text ≤64 KiB |
| 51 | `build_unarchive_identity_request(target_pubkey, content, reason, auth)` `builders.rs:1810` | 9036 | `-`, `p` | `reason`, `auth` | text ≤64 KiB |

Kind integers are sourced from `buzz_core::kind` for 26 of the builders
(`crates/buzz-sdk/src/builders.rs:6-19`); the remainder pass literals directly
to `Kind::Custom` (9, 45001, 45003, 40008, 40003, 9005, 5, 45002, 7, 40100, 0,
9000, 9001, 9022, 9002, 9007, 9021, 9008, 1, 3).

---

### 2. Non-builder public API in `builders.rs`

| Item | Signature | Purpose | File:line |
|---|---|---|---|
| `normalize_custom_emoji_shortcode` | `fn(&str) -> Result<String, SdkError>` | trims `:`/whitespace, validates charset+length, lowercases | `builders.rs:127-150` |
| `extract_channel_id` | `fn(&nostr::Event) -> Option<Uuid>` | reads first `h` tag as UUID | `builders.rs:816-826` |
| `GitAppliedPatchRef::parse` | `fn(&str) -> Result<Self, SdkError>` | parses `<id>[:<relay>[:<pubkey>]]` | `builders.rs:1143-1185` |
| `CUSTOM_EMOJI_SET_D_TAG` | `pub const &str` | `"buzz:custom-emoji"` | `builders.rs:503` |
| `DeleteMessageOptions`, `GitRepoCoord`, `GitPatchMeta`, `GitIssueMeta`, `GitStatus`, `GitAppliedPatchRef`, `GitStatusMeta`, `GitPullRequestMeta`, `GitPrUpdateMeta` | public structs/enums | builder inputs | see data-model doc |

`GitRepoCoord::to_a_tag_value` and `GitStatus::kind` are **private** inherent
methods (`builders.rs:976`, `builders.rs:1188`).

---

### 3. `mentions` module public API

| Item | Signature | Behavior | File:line |
|---|---|---|---|
| `MENTION_CAP` | `pub const usize = 50` | hard cap on mention p-tags | `mentions.rs:38` |
| `MentionProfile<'a>` | struct | `{ pubkey, content_json }` | `mentions.rs:46-51` |
| `extract_at_names` | `fn(&str) -> Vec<String>` | single-word `@name` scan; `@` must be at start or after ASCII whitespace; charset `[A-Za-z0-9._-]`; lowercased, deduped, first-seen order | `mentions.rs:64-104` |
| `extract_at_mentions_with_known` | `fn(&str, known_names: &[&str]) -> Vec<String>` | longest-known-name-first matching with word-boundary check, falls back to single-word tokenizer | `mentions.rs:107-152` |
| `match_names_to_profiles` | `fn(&[String], &[MentionProfile]) -> Vec<String>` | matches `display_name` (fallback `name`) case-insensitively; returns pubkeys in **profile order** | `mentions.rs:179-206` |
| `merge_mentions` | `fn(&mut Vec<String>, &[String], cap: usize)` | appends non-duplicate auto-resolved pubkeys up to `cap` | `mentions.rs:208-220` |
| `normalize_mention_pubkeys` | `fn(&[String], sender_pubkey: Option<&str>) -> Vec<String>` | lowercases, dedupes, drops self-mention | `mentions.rs:228-241` |
| `strip_code_regions` | `fn(&str) -> String` | replaces fenced blocks and inline spans with a single space | `mentions.rs:244-341` |
| `extract_nostr_uris` | `fn(&str) -> Vec<String>` | finds `nostr:npub1` + 58 bech32 chars, decodes to hex, dedupes | `mentions.rs:353-387` |

---

### 4. `nip_oa` module public API

| Item | Signature | Behavior | File:line |
|---|---|---|---|
| `compute_auth_tag` | `fn(&Keys, &PublicKey, conditions: &str) -> Result<String, SdkError>` | rejects self-attestation, validates conditions, BIP-340 Schnorr-signs `SHA256("nostr:agent-auth:<agent_hex>:<conditions>")`, returns JSON array string | `nip_oa.rs:146-176` |
| `verify_auth_tag` | `fn(auth_tag_json: &str, &PublicKey) -> Result<PublicKey, SdkError>` | full structural + cryptographic verification; returns owner pubkey | `nip_oa.rs:179-249` |
| `parse_auth_tag` | `fn(json_str: &str) -> Result<Tag, SdkError>` | structure-only fast path (no crypto); requires 64-hex lowercase owner, 128-hex lowercase sig | `nip_oa.rs:252-299` |

Private helpers: `validate_conditions` (`nip_oa.rs:36-59`), `validate_clause`
(`61-73`), `validate_canonical_decimal` (`75-107`), `build_preimage` (`109-111`),
`hash_preimage` (`113-116`), `is_lowercase_hex` (`120-122`), `parse_json_array`
(`124-133`).

---

### 5. Example binary

`examples/compute_auth_tag.rs` — CLI wrapper: takes `<owner_secret_hex>
<agent_pubkey_hex> [conditions]` from `std::env::args`, prints the auth-tag JSON
to stdout (`crates/buzz-sdk/examples/compute_auth_tag.rs:11-28`). There are no
`src/bin/*.rs` entry points.


## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: API Surface

### Crate root

`crates/buzz-persona/src/lib.rs:1-6` — six public modules, no re-exports, no prelude,
no crate-level doc comment:

```rust
pub mod manifest;
pub mod merge;
pub mod pack;
pub mod persona;
pub mod resolve;
pub mod validate;
```

Every item below is reached by its full module path (e.g. `buzz_persona::resolve::resolve_pack`).

---

### Built-in personas shipped in the crate

**None.** There are zero built-in / bundled personas.

Verified by: full file listing of `crates/buzz-persona` (only `Cargo.toml`,
`PERSONA_PACK_SPEC.md`, 7 files in `src/`, 2 files in `tests/`); repo-wide grep for
`include_str!`, `include_dir!`, `include_bytes!` inside the crate returns no matches;
no `[build-dependencies]`, `build.rs`, or asset directory in
`crates/buzz-persona/Cargo.toml:1-16`.

Persona names that appear in the crate are **test fixtures only**, not shipped
artifacts:

| Fixture name | Where | Role in the test |
|---|---|---|
| `berry` | `crates/buzz-persona/src/pack.rs:481-487` (`SIMPLE_PERSONA`) | minimal valid persona for loader tests |
| `pip`, `lep` | `crates/buzz-persona/tests/integration.rs:57-93` | pack-defaults override vs inherit |
| `alpha`, `beta`, `gamma` | `crates/buzz-persona/tests/integration.rs:471-490` | 3-persona defaults matrix |
| `bot`, `test-agent`, `my-bot`, `full-bot`, `t`, `goose-bot`, `buzz-bot` | `crates/buzz-persona/src/persona.rs:334-336`, `crates/buzz-persona/src/resolve.rs:461`, `crates/buzz-persona/tests/e2e_env_flow.rs:129`, `crates/buzz-persona/tests/e2e_env_flow.rs:277-289` | parser/env-projection unit tests |

The `PERSONA_PACK_SPEC.md` examples (`pip`, `lep`, `thistle`, `berry` as a "Meadow
Security Team") are documentation only — no corresponding files exist in the crate.

---

### `persona` module — `crates/buzz-persona/src/persona.rs`

| Item | Kind | Signature / definition | Line |
|---|---|---|---|
| `MAX_FRONTMATTER_BYTES` | `pub const usize` | `1_048_576` | `:21` |
| `MAX_BODY_BYTES` | `pub const usize` | `262_144` | `:24` |
| `PersonaError` | `pub enum` (thiserror) | see variants below | `:27` |
| `RespondTo` | `pub struct` | `mentions: Option<bool>`, `keywords: Vec<String>`, `all_messages: Option<bool>` | `:53` |
| `McpServerConfig` | `pub struct` | `name: String`, `command: String`, `args: Vec<String>`, `env: HashMap<String,String>` | `:70` |
| `Hooks` | `pub struct` | `on_start`/`on_stop`/`on_message`: `Option<String>` | `:84` |
| `PersonaConfig` | `pub struct` | 18 fields (see data-model doc) | `:101` |
| `parse_persona_md` | `pub fn` | `fn(content: &str) -> Result<PersonaConfig, PersonaError>` | `:208` |
| `parse_persona_file` | `pub fn` | `fn(path: &Path) -> Result<PersonaConfig, PersonaError>` | `:262` |
| `split_frontmatter` | `pub fn` | `fn(content: &str) -> Result<(&str, &str), PersonaError>` | `:277` |
| `split_model` | `pub fn` | `fn(model: &str) -> (Option<&str>, &str)` | `:324` |

`PersonaError` variants — `crates/buzz-persona/src/persona.rs:27-48`:

| Variant | Payload | Display message | Line |
|---|---|---|---|
| `Io` | `#[from] std::io::Error` | `failed to read file: {0}` | `:28-29` |
| `NoFrontmatter` | — | `missing \`---\` frontmatter delimiters` | `:31-32` |
| `FrontmatterTooLarge` | — | `frontmatter exceeds {MAX_FRONTMATTER_BYTES} bytes` | `:34-35` |
| `BodyTooLarge` | — | `body exceeds {MAX_BODY_BYTES} bytes` | `:37-38` |
| `TooLarge` | `String` | `file too large: {0}` | `:40-41` |
| `Yaml` | `#[from] serde_yaml::Error` | `failed to parse YAML frontmatter: {0}` | `:43-44` |
| `MissingField` | `String` | `missing required field: {0}` | `:46-47` |

---

### `manifest` module — `crates/buzz-persona/src/manifest.rs`

| Item | Kind | Signature / definition | Line |
|---|---|---|---|
| `ManifestError` | `pub enum` (thiserror) | `Io(#[from] io::Error)`, `Json(#[from] serde_json::Error)`, `MissingField(String)` | `:23-31` |
| `Engines` | `pub struct` | `buzz: Option<String>` | `:37` |
| `BehavioralDefaults` | `pub struct` | 7 optional behavioral fields | `:49` |
| `PackManifest` | `pub struct` | 14 fields | `:79` |
| `parse_manifest` | `pub fn` | `fn(content: &str) -> Result<PackManifest, ManifestError>` | `:152` |
| `parse_manifest_file` | `pub fn` | `fn(path: &Path) -> Result<PackManifest, ManifestError>` | `:190` |

---

### `pack` module — `crates/buzz-persona/src/pack.rs`

| Item | Kind | Signature / definition | Line |
|---|---|---|---|
| `PackError` | `pub enum` (thiserror) | 8 variants, see below | `:26` |
| `LoadedPack` | `pub struct` | `manifest`, `personas`, `pack_instructions`, `shared_mcp_config`, `skills_dir` | `:64` |
| `LoadedPersona` | `pub struct` | 18 fields | `:77` |
| `PackManifestData` | `pub struct` | 8 fields | `:102` |
| `load_pack` | `pub fn` | `fn(pack_dir: &Path) -> Result<LoadedPack, PackError>` | `:125` |
| `resolve_skills` | `pub fn` | `fn(pack_dir: &Path, personas: &[LoadedPersona]) -> HashMap<String, Vec<String>>` | `:249` |
| `impl From<ManifestError> for PackError` | trait impl | flattens to `PackError::ManifestParse(e.to_string())` | `:56-60` |

`PackError` variants — `crates/buzz-persona/src/pack.rs:26-54`:

| Variant | Payload | Display | Line |
|---|---|---|---|
| `ManifestNotFound` | `PathBuf` | `manifest not found at {0}` | `:27-28` |
| `Io` | `{ path: PathBuf, source: io::Error }` | `failed to read {path}: {source}` | `:30-35` |
| `ManifestParse` | `String` | `failed to parse manifest: {0}` | `:37-38` |
| `PersonaNotFound` | `PathBuf` | `persona file not found: {0}` | `:40-41` |
| `FileParse` | `{ path: PathBuf, reason: String }` | `invalid file {path}: {reason}` | `:43-44` |
| `PathTraversal` | `String` | `path traversal rejected: {0}` | `:46-47` |
| `PathEscape` | `PathBuf` | `path escapes pack root: {0}` | `:49-50` |
| `McpConfigParse` | `{ path: PathBuf, reason: String }` | `failed to parse .mcp.json at {path}: {reason}` | `:52-53` |

Private helpers (not public API): `safe_resolve` (`:323`), `read_file` (`:366`),
`read_bounded_file` (`:374`), `parse_persona_file` (`:392` — note: name collides with
the public `persona::parse_persona_file` but is a distinct private fn).

---

### `merge` module — `crates/buzz-persona/src/merge.rs`

| Item | Kind | Signature / definition | Line |
|---|---|---|---|
| `TriggersData` | `pub struct` | `mentions: bool`, `keywords: Vec<String>`, `all_messages: bool` | `:11` |
| `HooksData` | `pub struct` | three `Option<String>` | `:18` |
| `ResolvedConfig` | `pub struct` | 7 fields | `:25` |
| `merge_behavioral_config` | `pub fn` | `fn(persona_config: &serde_json::Value, pack_defaults: &serde_json::Value) -> serde_json::Value` | `:47-50` |
| `resolve_persona_config` | `pub fn` | `fn(persona_frontmatter: &serde_json::Value, pack_defaults: Option<&serde_json::Value>) -> ResolvedConfig` | `:85-88` |

Private: `string_field` (`:173`), `parse_triggers` (`:177`). Constants
`DEFAULT_THREAD_REPLIES` / `DEFAULT_BROADCAST_REPLIES` are private
(`crates/buzz-persona/src/merge.rs:38-39`).

---

### `resolve` module — `crates/buzz-persona/src/resolve.rs`

| Item | Kind | Signature / definition | Line |
|---|---|---|---|
| `ResolvedPersona` | `pub struct` | 19 fields | `:23` |
| `ResolvedMcpServer` | `pub struct` | `name`, `command`, `args`, `env: Vec<(String,String)>` | `:69` |
| `ResolvedHooks` | `pub struct` | three `Option<String>` | `:78` |
| `ResolvedTriggers` | `pub struct` | `mentions: bool`, `keywords: Vec<String>`, `all_messages: bool` | `:86` |
| `ResolvedPack` | `pub struct` | `id`, `name`, `version`, `description`, `personas` | `:94` |
| `resolve_pack` | `pub fn` | `fn(pack_dir: &Path) -> Result<ResolvedPack, PackError>` | `:108` |
| `resolve_loaded_pack` | `pub fn` | `fn(loaded: &LoadedPack) -> Result<ResolvedPack, PackError>` | `:118` |
| `resolve_persona_by_name` | `pub fn` | `fn(pack_dir: &Path, name: &str) -> Result<ResolvedPersona, PackError>` | `:186` |

Private: `resolve_one_persona` (`:194`), `resolve_triggers` (`:255`),
`merge_mcp_servers` (`:277`), `parse_mcp_server_config` (`:311`), `resolve_hooks` (`:347`),
`runtime_env_vars` (`:365`).

---

### `validate` module — `crates/buzz-persona/src/validate.rs`

| Item | Kind | Signature / definition | Line |
|---|---|---|---|
| `ValidationDiagnostic` | `pub enum` | `Error(String)`, `Warning(String)` | `:19` |
| `ValidationReport` | `pub struct` | `diagnostics: Vec<ValidationDiagnostic>` | `:35` |
| `ValidationReport::error` | `pub fn` | `fn(&mut self, msg: impl Into<String>)` | `:40` |
| `ValidationReport::warn` | `pub fn` | `fn(&mut self, msg: impl Into<String>)` | `:45` |
| `ValidationReport::has_errors` | `pub fn` | `fn(&self) -> bool` | `:50` |
| `ValidationReport::has_warnings` | `pub fn` | `fn(&self) -> bool` | `:56` |
| `ValidationReport::exit_code` | `pub fn` | `fn(&self) -> i32` — 0 clean / 1 errors / 2 warnings-only | `:63` |
| `validate_pack` | `pub fn` | `fn(pack_dir: &Path) -> ValidationReport` | `:143` |
| `impl Display for ValidationDiagnostic` | trait impl | `ERROR: {msg}` / `WARN:  {msg}` | `:24-33` |
| `impl Display for ValidationReport` | trait impl | `✓ Pack is valid.` or list + `{errors} error(s), {warnings} warning(s).` | `:74-96` |

Private constants: `KNOWN_MANIFEST_KEYS` (`:99-118`), `KNOWN_BEHAVIORAL_KEYS` (`:121-130`),
`KNOWN_RESPOND_TO_KEYS` (`:133`). Private fns: `validate_persona_name` (`:167`),
`semantic_check_personas` (`:187`), `advisory_check_respond_to_types` (`:210`),
`check_respond_to_value` (`:236`), `value_type_name` (`:289`),
`advisory_check_manifest_keys` (`:302`), `advisory_check_skill_names` (`:354`).

---

### Traits

No traits are **defined** by this crate. Derived/implemented trait surface:

| Trait | On | Where |
|---|---|---|
| `Serialize` + `Deserialize` | `RespondTo`, `McpServerConfig`, `Hooks`, `PersonaConfig`, `Engines`, `BehavioralDefaults`, `PackManifest` | `persona.rs:51`, `:68`, `:82`, `:99`; `manifest.rs:35`, `:47`, `:77` |
| `Deserialize` only | private `Frontmatter`, `RawManifest` | `persona.rs:174`, `manifest.rs:130` |
| `std::error::Error` (via `thiserror::Error`) | `PersonaError`, `ManifestError`, `PackError` | `persona.rs:26`, `manifest.rs:22`, `pack.rs:25` |
| `From<ManifestError>` | `PackError` | `pack.rs:56` |
| `Display` | `ValidationDiagnostic`, `ValidationReport` | `validate.rs:24`, `:74` |
| `Debug` | all public types | derives on each struct/enum |
| `Clone` | `RespondTo`, `McpServerConfig`, `Hooks`, `PersonaConfig`, `Engines`, `BehavioralDefaults`, `PackManifest`, `TriggersData`, `HooksData`, `ResolvedConfig`, `ResolvedPersona`, `ResolvedMcpServer`, `ResolvedHooks`, `ResolvedTriggers`, `ValidationDiagnostic` | respective derives |
| `PartialEq` | `TriggersData`, `HooksData`, `ResolvedConfig`, `ResolvedMcpServer`, `ResolvedHooks`, `ResolvedTriggers` | `merge.rs:10`, `:17`, `:24`; `resolve.rs:68`, `:77`, `:85` |
| `Default` | `ValidationReport` | `validate.rs:34` |

Not implemented anywhere: `Copy`, `Hash`, `Ord`, `Send`/`Sync` (auto), custom
`Deserialize` impls, builder types.

---

### Entry-point summary (three practical call paths)

| Goal | Call | Returns |
|---|---|---|
| Validate a pack directory | `validate::validate_pack(&Path)` | `ValidationReport` (never `Err`) |
| Get ACP-ready output for all personas | `resolve::resolve_pack(&Path)` | `Result<ResolvedPack, PackError>` |
| Get one persona by name | `resolve::resolve_persona_by_name(&Path, &str)` | `Result<ResolvedPersona, PackError>` |
| Parse a single standalone `.persona.md` | `persona::parse_persona_md(&str)` / `parse_persona_file(&Path)` | `Result<PersonaConfig, PersonaError>` |
| Split frontmatter from any markdown file | `persona::split_frontmatter(&str)` | `Result<(&str, &str), PersonaError>` |
| Map skills → personas | `pack::resolve_skills(&Path, &[LoadedPersona])` | `HashMap<String, Vec<String>>` |


## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: API Surface

Crate root: `crates/buzz-ws-client/src/lib.rs`. Three public modules
(`connection`, `error`, `message` — `lib.rs:3`–`5`) plus a flat re-export set
(`lib.rs:7`–`9`). No `#[doc(hidden)]`, no `pub(crate)` gating, no traits, no
macros. Library only — no `main.rs`, no binaries in `Cargo.toml`.

---

### 1. Re-exports (the intended entry surface)

| Re-export | Source | file:line |
|---|---|---|
| `publish_event`, `NostrWsConnection` | `connection` | `crates/buzz-ws-client/src/lib.rs:7` |
| `WsClientError` | `error` | `crates/buzz-ws-client/src/lib.rs:8` |
| `build_auth_event`, `parse_relay_message`, `OkResponse`, `RelayMessage` | `message` | `crates/buzz-ws-client/src/lib.rs:9` |

Because the modules themselves are `pub`, everything below is also reachable via
its module path (e.g. `buzz_ws_client::connection::AUTH_OK_TIMEOUT_SECS`, which is
**not** in the flat re-export list).

---

### 2. Public constants

| Constant | Type | Value | file:line |
|---|---|---|---|
| `AUTH_CHALLENGE_TIMEOUT_SECS` | `u64` | `20` | `crates/buzz-ws-client/src/connection.rs:17` |
| `AUTH_OK_TIMEOUT_SECS` | `u64` | `20` | `crates/buzz-ws-client/src/connection.rs:20` |
| `PUBLISH_OK_TIMEOUT_SECS` | `u64` | `30` | `crates/buzz-ws-client/src/connection.rs:23` |

---

### 3. Public functions and methods (full signatures)

```rust
// crates/buzz-ws-client/src/connection.rs:37
pub async fn NostrWsConnection::connect_authenticated(
    url: &str, keys: &Keys, auth_tag: Option<&Tag>,
) -> Result<Self, WsClientError>

// crates/buzz-ws-client/src/connection.rs:48
pub async fn NostrWsConnection::connect(url: &str) -> Result<Self, WsClientError>

// crates/buzz-ws-client/src/connection.rs:70
pub async fn NostrWsConnection::authenticate(
    &mut self, keys: &Keys, auth_tag: Option<&Tag>,
) -> Result<(), WsClientError>

// crates/buzz-ws-client/src/connection.rs:96
pub async fn NostrWsConnection::send_event(&mut self, event: Event)
    -> Result<OkResponse, WsClientError>

// crates/buzz-ws-client/src/connection.rs:104
pub async fn NostrWsConnection::next_event(&mut self, timeout_dur: Duration)
    -> Result<RelayMessage, WsClientError>

// crates/buzz-ws-client/src/connection.rs:115
pub async fn NostrWsConnection::disconnect(mut self) -> Result<(), WsClientError>

// crates/buzz-ws-client/src/connection.rs:121
pub async fn NostrWsConnection::send_raw(&mut self, value: &Value)
    -> Result<(), WsClientError>

// crates/buzz-ws-client/src/connection.rs:277
pub async fn publish_event(
    relay_url: &str, event: Event, keys: &Keys,
    auth_tag: Option<&Tag>, timeout_secs: u64,
) -> Result<OkResponse, WsClientError>

// crates/buzz-ws-client/src/message.rs:62
#[allow(clippy::result_large_err)]
pub fn parse_relay_message(text: &str) -> Result<RelayMessage, WsClientError>

// crates/buzz-ws-client/src/message.rs:174
pub fn build_auth_event(
    challenge: &str, relay_url: &str, keys: &Keys, auth_tag: Option<&Tag>,
) -> Result<Event, WsClientError>
```

Private helpers (not part of the surface): `recv_one`
(`crates/buzz-ws-client/src/connection.rs:128`), `wait_for_auth_challenge`
(`:157`), `wait_for_ok` (`:217`).

---

### 4. Public types

| Type | Kind | Public members | file:line |
|---|---|---|---|
| `NostrWsConnection` | struct | none (all 4 fields private) | `connection.rs:26` |
| `RelayMessage` | enum, `Debug + Clone` | **7** variants: `Event`, `Ok`, `Eose`, `Closed`, `Notice`, `Auth`, `Count` — was 6 at report time; the post-analysis sync added `Count { subscription_id, count }` (`message.rs:40`–`46`) | `message.rs:8` |
| `OkResponse` | struct, `Debug + Clone` | `event_id: String`, `accepted: bool`, `message: String` | `message.rs:51`–`57` |
| `WsClientError` | enum, `Debug + thiserror::Error` | 10 variants | `error.rs:5`–`45` |

Trait impls exposed: `Error`/`Display` on `WsClientError` via `#[derive(Error)]`
(`error.rs:4`), `From<tokio_tungstenite::tungstenite::Error>` (`error.rs:8`),
`From<serde_json::Error>` (`error.rs:12`),
`From<nostr::event::builder::Error>` (`error.rs:47`).

---

### 5. Function → wire message mapping

| Public fn | Sends on the wire | Consumes/handles from the wire | Return type |
|---|---|---|---|
| `connect` | WebSocket HTTP upgrade only (`connect_async`, `connection.rs:53`) | nothing | `Result<Self, WsClientError>` |
| `authenticate` | `["AUTH", <signed event>]` (`connection.rs:82`) | waits for `AUTH` challenge (`connection.rs:76`→`:197`), then `OK` matching the AUTH event id (`connection.rs:85`→`:254`); replies `Pong` to `Ping` (`:209`) | `Result<(), WsClientError>` |
| `connect_authenticated` | upgrade + `["AUTH", …]` (`connection.rs:42`–`43`) | as `authenticate` | `Result<Self, WsClientError>` |
| `send_event` | `["EVENT", <event>]` (`connection.rs:98`) | `OK` with matching event id (`connection.rs:99`, `:254`); buffers `EVENT`/`EOSE`/`CLOSED`/`NOTICE`, records `AUTH` (`:255`–`:259`) | `Result<OkResponse, WsClientError>` |
| `next_event` | nothing (may emit `Pong`, `connection.rs:149`) | any one `RelayMessage` (buffer first, `connection.rs:108`) | `Result<RelayMessage, WsClientError>` |
| `send_raw` | any caller-provided JSON array — this is how `REQ`/`CLOSE`/`COUNT` are sent, since the crate has no typed subscribe API (`connection.rs:121`–`124`) | nothing | `Result<(), WsClientError>` |
| `disconnect` | WebSocket Close frame (`close(None)`, `connection.rs:116`) | nothing | `Result<(), WsClientError>` |
| `publish_event` | upgrade + `["AUTH", …]` + `["EVENT", …]` + Close (`connection.rs:285`–`288`) | `AUTH` challenge, both `OK`s | `Result<OkResponse, WsClientError>` |
| `parse_relay_message` | — (pure parser) | `EVENT`, `OK`, `EOSE`, `CLOSED`, `NOTICE`, `AUTH`, and — added after the original analysis — `COUNT` (`message.rs:71`–`162`; the `COUNT` arm is `:147`–`162`) | `Result<RelayMessage, WsClientError>` |
| `build_auth_event` | — (pure builder; produces the event the caller/`authenticate` sends) | — | `Result<Event, WsClientError>` |

WebSocket control frames: inbound `Ping` is answered with `Pong` in all three read
loops (`connection.rs:148`–`150`, `:208`–`:210`, `:262`–`:264`); inbound `Close`
maps to `WsClientError::ConnectionClosed` (`:151`, `:211`, `:265`); all other frame
kinds (Binary, Pong, Frame) are silently ignored via `_ => {}` (`:152`, `:212`,
`:266`).

**Not implemented in this crate:** typed `REQ`/`CLOSE`/`COUNT` builders, a
subscription abstraction, `EOSE`-driven collect helpers, and any reconnect API.
Callers assemble those from `send_raw` + `next_event` (as `buzz-test-client` does at
`crates/buzz-test-client/src/lib.rs:154` and `:160`).

---

### 6. Known consumers (for API-contract impact)

| Consumer | Uses | file:line |
|---|---|---|
| `buzz-cli` | `buzz_ws_client::publish_event(&ws_url, event, &self.keys, self.auth_tag.as_ref(), 75)` | `crates/buzz-cli/src/client.rs:1080`; dep at `crates/buzz-cli/Cargo.toml:77` |
| `buzz-test-client` | wraps `NostrWsConnection` (`connect`, `send_event`, `send_raw`, `next_event`) and re-exports `parse_relay_message`, `OkResponse`, `RelayMessage`, `WsClientError` | `crates/buzz-test-client/src/lib.rs:13`–`14`, `:85`, `:98`, `:123`, `:154`, `:175`; dep at `crates/buzz-test-client/Cargo.toml:13` |
| workspace | path dependency declared once | `Cargo.toml:16`, `Cargo.toml:134` |


## Module: buzz-db (`crates/buzz-db`)

### Aspect: API Surface

Shape of the crate's public API:

| Kind | Count | Where |
|------|-------|-------|
| `Db` inherent methods | 215 | `crates/buzz-db/src/lib.rs:360`–`:3628` |
| Free functions in `lib.rs` | 1 (`insert_mentions`) | `crates/buzz-db/src/lib.rs:97` |
| Module-level free functions | 187 | the 19 sibling modules |
| `ReplicaFence` methods | 5 | `crates/buzz-db/src/replica_fence.rs:88`–`:140` |
| `UsageMetricsLeader::is_live` | 1 | `crates/buzz-db/src/lib.rs:215` |
| Public constants | 12 | see table at end |

Two-layer design: nearly every `Db` method is a thin delegate that passes
`&self.pool` (or `self.read()`) plus a server-resolved `CommunityId` to a
module-level free function. Exceptions — statements written inline on `Db` —
are called out below.

Public module list and doc intent: `crates/buzz-db/src/lib.rs:12-51`.
Re-exports: `pub use error::{DbError, Result}` and
`pub use event::{EventQuery, ReactionEventInsertOutcome}` — `crates/buzz-db/src/lib.rs:53-54`.

---

#### `lib.rs` — `Db` handle, community registry, replaceable-event writes

| Fn | Operation | Tables touched |
|----|-----------|----------------|
| `insert_mentions` `:97` | multi-row `QueryBuilder` `INSERT … ON CONFLICT DO NOTHING`; extracts + validates lowercase-hex 64-char `p` tags | `event_mentions` |
| `Db::new` `:360` / `connect_pool` `:387` | `PgPoolOptions` connect; writer pool sets `SELECT set_config('buzz.created_at_floor', $1, false)` on every connection | — |
| `Db::from_pool` `:403`, `from_pools` `:419` | construct from existing pools (tests) | — |
| `Db::fence` `:429`, `read` `:470`, `has_read_pool` `:475`, `pool_stats` `:494`, `read_pool_stats` `:503` | accessors | — |
| `Db::spawn_fence_probe` `:449` | runs `verify_floor_guard_catalog` + `verify_floor_guard_behavior`, then spawns `replica_fence::run_probe` | `pg_trigger`/`pg_class` catalog; scratch rows in `communities`+`events` (rolled back) |
| `Db::migrate` `:480` | `migration::run_migrations` | `_sqlx_migrations` + all |
| `Db::ping` `:485` | `SELECT 1` | — |
| `Db::try_lock_usage_metrics` `:517` | `SELECT pg_try_advisory_lock($1)` on a detached connection | — (advisory lock) |
| `Db::admin_list_reports` `:537`, `admin_get_report` `:563`, `admin_list_feedback` `:571`, `admin_get_feedback` `:579` | delegate to `admin_moderation` (deployment-global) | `moderation_reports`, `product_feedback`, `communities` |
| `Db::usage_*` (11 methods) `:587`–`:640` | delegate to `usage` | `communities`, `users`, `channels`, `events`, `relay_members`, `workflows`, `git_repo_names` |
| `Db::begin_transaction` `:648` | `pool.begin()` → `Transaction<'static, Postgres>` | — |
| `Db::lookup_community_by_host` `:656` | `SELECT id, host FROM communities WHERE lower(host)=lower($1) AND archived_at IS NULL` | `communities` |
| `Db::is_community_active` `:685` | `SELECT EXISTS(… archived_at IS NULL)` | `communities` |
| `Db::lookup_community_by_host_for_management` `:696` | same lookup **without** the archived filter | `communities` |
| `Db::list_communities_owned_by` `:717` | `communities JOIN relay_members … WHERE rm.pubkey=$1 AND rm.role='owner'` (no community filter — operator plane) | `communities`, `relay_members` |
| `Db::lookup_community_host` `:762` | `SELECT host … WHERE id=$1 AND archived_at IS NULL` | `communities` |
| `Db::get_community_icon` `:786` / `set_community_icon` `:806` | `SELECT icon` / `UPDATE … SET icon` | `communities` |
| `Db::ensure_configured_community` `:830` | `INSERT … ON CONFLICT (lower(host)) DO UPDATE SET host = communities.host RETURNING id, host, (xmax = 0) AS created` | `communities` |
| `Db::create_community_with_owner` `:862` | tx: `pg_advisory_xact_lock(owner_count_advisory_lock_key)` → `INSERT communities … ON CONFLICT DO NOTHING` → owner count check → `INSERT relay_members … 'owner'` | `communities`, `relay_members` |
| `Db::archive_community_owned_by` `:947` | `UPDATE communities … FROM relay_members … SET archived_at = COALESCE(archived_at, now())`, refuses the protected deployment host | `communities`, `relay_members` |
| `Db::unarchive_community_owned_by` `:980` | `UPDATE communities SET archived_at = NULL FROM relay_members …` | `communities`, `relay_members` |
| `Db::community_of_channel` `:1012` | `SELECT community_id FROM channels WHERE id=$1 AND deleted_at IS NULL` (resolves tenancy — deliberately unscoped) | `channels` |
| `Db::communities_of_channels` `:1050` | `SELECT id, community_id … WHERE id = ANY($1) AND deleted_at IS NULL`; missing ids are absent from the map | `channels` |
| `Db::insert_event` `:1079` | `event::insert_event`, then best-effort `insert_mentions` when inserted | `events`, `event_mentions` |
| `Db::query_events` `:1095`, `count_events` `:1100` | delegate to `event` | `events` (+`event_mentions` on `#p`) |
| `Db::huddle_started_link_exists` `:1106` | delegate | `events` |
| `Db::get_latest_global_replaceable` `:1128`, `get_event_by_id` `:1140`, `get_event_by_id_including_deleted` `:1149`, `get_events_by_ids` `:1215` | delegate | `events` |
| `Db::soft_delete_event` `:1158`, `soft_delete_by_coordinate` `:1168`, `soft_delete_event_and_update_thread` `:1179` | delegate | `events`, `thread_metadata` |
| `Db::get_last_message_at` `:1197`, `_bulk` `:1206` | delegate | `events` |
| `Db::claim_due_push_match_batch` `:1224` … `disable_push_endpoint` `:1338`, `accept_push_lease_event` `:1357` (14 methods) | delegate to `push` | `push_match_queue`, `push_leases`, `push_wake_outbox`, `events` |
| `Db::insert_event_with_thread_metadata` `:1379` | delegate + best-effort `insert_mentions` | `events`, `thread_metadata`, `event_mentions` |
| `Db::insert_reaction_event_with_thread_metadata` `:1404` | delegate + best-effort `insert_mentions` | `events`, `reactions`, `thread_metadata`, `event_mentions` |
| `Db::create_channel` `:1438` … `reap_expired_ephemeral_channels` `:1725` (26 methods) | delegate to `channel` | `channels`, `channel_members`, `users`, `communities` |
| `Db::query_due_reminders` `:1732`, `claim_due_reminder` `:1741`, `claim_due_reminder_with_stamp` `:1751`, `release_due_reminder` `:1769` | delegate to `event` | `events`, `communities` |
| `Db::ensure_user` `:1791` … `set_channel_add_policy` `:1877` (9 methods) | delegate to `user` | `users` |
| `Db::find_dm_by_participants` `:1887` … `list_hidden_dms` `:1950` (7 methods) | delegate to `dm` | `channels`, `channel_members`, `users` |
| `Db::insert_thread_metadata` `:1960`, `get_thread_summary` `:2044`, `get_thread_metadata_by_event` `:2079`, `decrement_reply_count` `:2088` | delegate to `thread` | `thread_metadata`, `events` |
| `Db::get_thread_replies` `:2004` | **replica routing**: cursor page tried on `read()` when fence open; re-run on writer unless page is full AND tail ≤ fence | `thread_metadata`, `events` |
| `Db::get_channel_window` `:2063` | **replica routing**: `read()` only when `fence.covers(cursor_ts)` | `events`, `thread_metadata` |
| `Db::add_reaction` `:2099` … `get_reactions_bulk` `:2212` (7 methods) | delegate to `reaction` | `reactions` |
| `Db::query_feed_mentions` `:2221`, `query_feed_needs_action` `:2241`, `query_feed_activity` `:2261` | delegate to `feed` | `events`, `event_mentions` |
| `Db::create_api_token` `:2273`, `create_api_token_if_under_limit` `:2298`, `get_api_token_by_hash_including_revoked` `:2352`, `list_tokens_by_owner` `:2421`, `revoke_token` `:2430`, `revoke_all_tokens` `:2448` | delegate to `api_token` | `api_tokens` |
| `Db::get_api_token_by_hash` `:2327` | **inline** `SELECT … WHERE community_id=$1 AND token_hash=$2 AND revoked_at IS NULL` | `api_tokens` |
| `Db::touch_api_token` `:2366` / `update_token_last_used` `:2378` (alias) | **inline** `UPDATE api_tokens SET last_used_at = NOW() WHERE community_id=$1 AND token_hash=$2` | `api_tokens` |
| `Db::list_active_tokens` `:2387` | **inline** `SELECT … WHERE community_id=$1 AND revoked_at IS NULL ORDER BY created_at DESC LIMIT 1000` | `api_tokens` |
| `Db::create_workflow` `:2464` … `update_approval_by_stored_hash` `:2782` (27 methods) | delegate to `workflow` | `workflows`, `workflow_runs`, `workflow_approvals`, `scheduled_workflow_fires` |
| `Db::ensure_future_partitions` `:2802` | delegate to `partition` | DDL on `events`, `delivery_log` |
| `Db::backfill_d_tags` `:2810` | **inline** `UPDATE events SET d_tag = COALESCE((SELECT elem->>1 FROM jsonb_array_elements(tags) …), '') WHERE kind BETWEEN 30000 AND 39999 AND d_tag IS NULL` — **no community filter** | `events` |
| `Db::is_pubkey_allowed` `:2826`, `has_allowlist_entries` `:2839`, `add_to_allowlist` `:2850`, `remove_from_allowlist` `:2871`, `list_allowlist` `:2886` | **inline** count/insert/delete/select, all `WHERE community_id = $1` | `pubkey_allowlist` |
| `Db::is_relay_member` `:2907` … `backfill_from_allowlist` `:3027` (12 methods) | delegate to `relay_members` | `relay_members`, `join_policy_acceptances`, `pubkey_allowlist` |
| `Db::insert_product_feedback` `:3032`, `list_product_feedback` `:3041` | delegate to `product_feedback` | `product_feedback` |
| `Db::insert_moderation_report` `:3049` … `list_moderation_actions` `:3185` (15 methods) | delegate to `moderation` | `moderation_reports`, `community_bans`, `moderation_actions` |
| `Db::repo_name_owner` `:3195`, `reserve_repo_name` `:3207`, `count_repos_for_owner` `:3217`, `release_repo_name` `:3228` | delegate to `git_repo` | `git_repo_names` |
| `Db::is_archived` `:3238`, `archive` `:3244`, `unarchive` `:3268`, `list_archived` `:3273` | delegate to `archived_identities` | `archived_identities` |
| `Db::soft_delete_discovery_events` `:3281` | **inline** `UPDATE events SET deleted_at=NOW() WHERE community_id=$1 AND channel_id=$2 AND pubkey=$3 AND deleted_at IS NULL AND kind IN (39000,39001,39002)` | `events` |
| `Db::replace_addressable_event` `:3306` | **inline tx**: `pg_advisory_xact_lock(fnv1a(community,kind,pubkey,channel))` → newest-live probe (`ORDER BY created_at DESC, id ASC LIMIT 1`) → stale reject → soft-delete old → `INSERT … ON CONFLICT DO NOTHING` → commit → best-effort mentions | `events`, `event_mentions` |
| `Db::nip43_membership_snapshot_needs_reconciliation` `:3438` | `query_events` for the relay-signed kind:13534 snapshot + `list_relay_members`, compares sorted `(pubkey, role)` sets | `events`, `relay_members` |
| `Db::publish_nip43_membership_locked` `:3488` | **inline tx**: advisory lock → `SELECT pubkey, role FROM relay_members WHERE community_id=$1 ORDER BY created_at ASC` → build+sign kind:13534 → soft-delete prior snapshots → insert | `relay_members`, `events`, `event_mentions` |
| `Db::replace_parameterized_event` `:3628` | **inline tx**: advisory lock → classify NIP-RS / buzz-mesh-status → probe live head + `parameterized_event_watermarks` → stale reject → hard-DELETE (with `set_config('buzz.nip_rs_hard_delete','on',true)` for NIP-RS) or soft-delete → insert → upsert watermark | `events`, `parameterized_event_watermarks`, `event_mentions` |

Private helper: `event_replacement_lock_key` `crates/buzz-db/src/lib.rs:63` —
FNV-1a over `(community_id, kind, pubkey, optional coordinate)` → `i64`
advisory-lock key.

---

#### `event.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `extract_d_tag` `:141` | pure; NIP-33 kinds only, missing `d` ⇒ `Some("")` | — |
| `extract_not_before` `:166` | pure; kind 30300 only | — |
| `huddle_started_link_exists` `:199` | `SELECT content … WHERE community_id/channel_id/kind=48100/pubkey AND octet_length(content)<=512 AND content ILIKE $6 ORDER BY created_at DESC, id ASC LIMIT 32`, then JSON check in Rust | `events` |
| `insert_event` `:240` | rejects `KIND_AUTH`/ephemeral, then single `INSERT … ON CONFLICT DO NOTHING` (12 columns) | `events` |
| `query_events` `:302` | `QueryBuilder` SELECT with 14 optional predicates; `INNER JOIN event_mentions` when `p_tag_hex` set; `ORDER BY created_at DESC, id ASC LIMIT/OFFSET` | `events`, `event_mentions` |
| `count_events` `:557` | same predicate set, `SELECT COUNT(*)` (no cursor/order/limit) | `events`, `event_mentions` |
| `soft_delete_event` `:703` | `UPDATE … SET deleted_at=NOW() WHERE community_id AND id AND deleted_at IS NULL` | `events` |
| `soft_delete_by_coordinate` `:730` | same on `(kind, pubkey, d_tag)` | `events` |
| `soft_delete_event_and_update_thread` `:756` | tx: soft-delete + `reply_count = GREATEST(reply_count-1,0)` + `descendant_count = GREATEST(…-1,0)` | `events`, `thread_metadata` |
| `get_last_message_at` `:806` / `_bulk` `:831` | `MAX(created_at)` / `GROUP BY channel_id` | `events` |
| `get_event_by_id` `:868`, `get_event_by_id_including_deleted` `:924` | scoped id lookup, `ORDER BY created_at DESC LIMIT 1` | `events` |
| `get_latest_global_replaceable` `:894` | `… channel_id IS NULL AND deleted_at IS NULL ORDER BY created_at DESC, id ASC LIMIT 1` | `events` |
| `get_events_by_ids` `:948` | `QueryBuilder` `id IN (…)`; `debug_assert!(ids.len() <= 500)` | `events` |
| `insert_event_with_thread_metadata` `:1180` (+ private `_tx` `:1017`) | tx: event insert → `thread_metadata` insert → parent/root stub inserts → `reply_count+1`, `last_reply_at=NOW()`, `descendant_count+1` | `events`, `thread_metadata` |
| `insert_reaction_event_with_thread_metadata` `:1201` | tx: resolve live target → `reaction::add_reaction_tx` → short-circuit on active duplicate → event+thread insert | `events`, `reactions`, `thread_metadata` |
| `query_due_reminders` `:1293` | `SELECT DISTINCT ON (community_id, pubkey, d_tag) … JOIN communities … WHERE kind=30300 AND not_before<=$2 AND deleted_at IS NULL AND delivered_at IS NULL AND c.archived_at IS NULL ORDER BY …, created_at DESC, id ASC LIMIT $3` | `events`, `communities` |
| `claim_due_reminder` `:1344` / `_with_stamp` `:1370` | `UPDATE events SET delivered_at=$1 WHERE community_id AND created_at AND id AND delivered_at IS NULL` | `events` |
| `release_due_reminder` `:1402` | compare-and-clear `… AND delivered_at = $4` | `events` |
| `row_to_stored_event` (crate-private) `:451` | rebuilds `nostr::Event` from JSON; unparseable rows are skipped with a warn | — |

---

#### `channel.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `create_channel` `:87` | validates 32-byte pubkey + canonical non-empty name; tx: insert channel (`ttl_deadline = NOW() + ($8 || ' seconds')::interval`) → upsert creator as `'owner'` → read back | `channels`, `channel_members` |
| `create_channel_with_id` `:175` | same, client UUID, rejects nil UUID, `ON CONFLICT (community_id, id) DO NOTHING`; returns `(record, was_created)` | `channels`, `channel_members` |
| `get_channel` `:271` | scoped select, `deleted_at IS NULL`, else `ChannelNotFound` | `channels` |
| `get_canvas` `:298` / `set_canvas` `:315` | select / update `canvas` | `channels` |
| `add_member` `:346` | tx: read channel → role enforcement (private needs elevated inviter unless creator bootstrap; elevated roles need owner/admin granter) → `INSERT … ON CONFLICT DO UPDATE SET removed_at=NULL, removed_by=NULL, role=EXCLUDED.role` → read back | `channels`, `channel_members` |
| `remove_member` `:459` | tx: actor role check (or self-remove, or `user::is_agent_owner` on the **pool**) → last-owner guard (`COUNT(*) … role='owner' AND removed_at IS NULL <= 1`) → `UPDATE SET removed_at=NOW(), removed_by=$1` | `channel_members`, `channels`, `users` |
| `is_member` `:531` | `COUNT(*)` with `JOIN channels … deleted_at IS NULL` | `channel_members`, `channels` |
| `membership_pairs` `:554` | one statement, `channel_id = ANY($2) AND pubkey = ANY($3)` | `channel_members`, `channels` |
| `get_members` `:581` (LIMIT 1000) / `get_members_bulk` `:610` (`ANY($2)`) | active members joined to live channels | `channel_members`, `channels` |
| `get_accessible_channel_ids` `:638` | membership `UNION` all `visibility='open'` channels (no LIMIT) | `channel_members`, `channels` |
| `list_channels` `:669` | two static variants (with/without `visibility::text = $2`), `LIMIT 1000` | `channels` |
| `get_accessible_channels` `:828` | `format!`-built SQL (`AssertSqlSafe`) with a fixed membership clause + optional `$3` visibility bind; `LEFT JOIN channel_members`; DM rows hidden when `cm.hidden_at IS NOT NULL`; `LIMIT 1000` | `channels`, `channel_members` |
| `get_bot_members` `:894` | `json_agg(DISTINCT jsonb_build_object('name', c.name, 'id', c.id::text))` grouped per bot; `LIMIT 1000` | `channel_members`, `users`, `channels` |
| `get_users_bulk` `:938` | `format!`-built `$n` placeholder list (`AssertSqlSafe`), all values bound | `users` |
| `update_channel` `:1050` | dynamic SET list (`format!` + `AssertSqlSafe`, positional binds); TTL changes run in a tx that first takes `pg_advisory_xact_lock(hashtextextended('buzz_channel_ttl:<community>:<channel>'))` | `channels` |
| `set_topic` `:1155` / `set_purpose` `:1179` | update + `*_set_by`/`*_set_at` | `channels` |
| `archive_channel` `:1206` | read state → `AccessDenied` if already archived → `UPDATE SET archived_at=NOW()` | `channels` |
| `unarchive_channel` `:1248` | read state → `AccessDenied` if not archived → clear `archived_at` and renew `ttl_deadline` when `ttl_seconds IS NOT NULL` | `channels` |
| `soft_delete_channel` `:1292` | `UPDATE SET deleted_at=NOW() … deleted_at IS NULL` | `channels` |
| `get_member_count` `:1309` / `get_member_counts_bulk` `:1328` | `COUNT(*)` / `QueryBuilder` `GROUP BY channel_id` | `channel_members` |
| `get_member_role` `:1363` | scoped role read joined to live channel | `channel_members`, `channels` |
| `reap_expired_ephemeral_channels` `:1387` | global `UPDATE channels … FROM communities WHERE ttl_deadline < NOW() AND archived_at IS NULL AND deleted_at IS NULL AND c.archived_at IS NULL RETURNING community_id, host, id` | `channels`, `communities` |

Private: `get_active_role_tx` `:697`, `get_channel_tx` `:718`,
`row_to_channel_record` `:983`, `row_to_member_record` `:1027`.

---

#### `thread.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `insert_thread_metadata` `:116` | tx: insert row `ON CONFLICT DO NOTHING` → parent/root stub inserts → `reply_count+1`/`last_reply_at`/`descendant_count+1` (only when the row was actually inserted) | `thread_metadata` |
| `increment_reply_count` `:251` | `#[allow(dead_code)]`, documented as unused; standalone counter bump | `thread_metadata` |
| `decrement_reply_count` `:292` | `GREATEST(x-1, 0)` on parent and root | `thread_metadata` |
| `get_thread_replies` `:345` | `format!`-built SQL (`AssertSqlSafe`) joining `thread_metadata ⋈ events` on `(community_id, created_at, id)`; composite `(event_created_at, event_id) > ($ts,$id)` keyset, legacy 8-byte timestamp-only cursor supported; `ORDER BY event_created_at ASC, event_id ASC LIMIT $n` | `thread_metadata`, `events` |
| `get_thread_summary` `:489` | counters read + top-10 participants (`DISTINCT` pubkey, `MAX(created_at)` order) | `thread_metadata`, `events` |
| `get_channel_window` `:565` | `format!`-built SQL (`AssertSqlSafe`); top-level predicate `tm.depth IS NULL OR tm.depth=0 OR (tm.depth=1 AND tm.broadcast)`; `kind IN (...)` list built from `u32::to_string()`; `LIMIT $n+1` probe drives `has_more`; batched per-root participants via `ROW_NUMBER() OVER (PARTITION BY root_event_id …) WHERE rn <= 10` | `events`, `thread_metadata` |
| `get_thread_metadata_by_event` `:755` | single scoped row | `thread_metadata` |

---

#### `feed.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `query_mentions` `:125` (builder `:87`) | `INNER JOIN event_mentions m ON e.community_id=m.community_id AND e.id=m.event_id`, `m.pubkey_hex = $`, kinds `{KIND_STREAM_MESSAGE, _V2, KIND_FORUM_POST, KIND_FORUM_COMMENT}`, visible-channel filter, `ORDER BY m.event_created_at DESC LIMIT min(limit, 100)` | `events`, `event_mentions` |
| `query_needs_action` `:186` (builder `:148`) | same join, kinds `{KIND_WORKFLOW_APPROVAL_REQUESTED, KIND_STREAM_REMINDER}` | `events`, `event_mentions` |
| `query_activity` `:235` (builder `:207`) | no join; kinds `{stream msg, v2, forum post, job request/progress/result}` | `events` |
| private `push_visible_channel_filter` `:59` | empty accessible list ⇒ `channel_id IS NULL` only | — |

---

#### `reaction.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `add_reaction` `:82` / `add_reaction_tx` (crate-private) `:114` | shared `ADD_REACTION_SQL` `:66`: `INSERT … ON CONFLICT (community_id, event_created_at, event_id, pubkey, emoji) DO UPDATE SET created_at=NOW(), removed_at=NULL, reaction_event_id=COALESCE(EXCLUDED…, reactions…) WHERE reactions.removed_at IS NOT NULL` | `reactions` |
| `remove_reaction` `:140` | `UPDATE SET removed_at=NOW() … removed_at IS NULL` | `reactions` |
| `remove_reaction_by_source_event_id` `:174` | same keyed on `reaction_event_id` | `reactions` |
| `get_active_reaction_record` `:197` | scoped single-row select | `reactions` |
| `set_reaction_event_id` `:238` | backfill `reaction_event_id` on the active row | `reactions` |
| `get_reactions` `:280` | two-step: inner `DISTINCT emoji … LIMIT $4` then all rows for those groups; grouping in Rust; `_cursor` parameter is unused | `reactions` |
| `get_reactions_bulk` `:366` | **one query per input pair** (`GROUP BY emoji`) | `reactions` |

---

#### `dm.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `compute_participant_hash` `:48` | pure: sort + dedup pubkeys, SHA-256, no separator | — |
| `find_dm_by_participants` `:65` | `… participant_hash=$2 AND channel_type='dm' AND deleted_at IS NULL LIMIT 1` | `channels` |
| `create_dm` `:101` | validates 2–9 participants and 32-byte keys; tx: idempotency probe → insert `channel_type='dm', visibility='private'` → insert every participant as `'member'` → read back | `channels`, `channel_members` |
| `list_dms_for_user` `:226` | `limit.min(200)`; cursor resolved to `updated_at`; then **one participants query per DM** | `channels`, `channel_members`, `users` |
| `open_dm` `:356` | merges `created_by`, enforces ≤9, fast-path find (+`unhide_dm`), else `create_dm` | `channels`, `channel_members` |
| `hide_dm` `:397` | `UPDATE channel_members SET hidden_at=NOW() … removed_at IS NULL`; 0 rows ⇒ `NotFound` | `channel_members` |
| `unhide_dm` `:429` | clears `hidden_at` (no-op safe) | `channel_members` |
| `list_hidden_dms` `:454` | `hidden_at IS NOT NULL` + live DM channel | `channel_members`, `channels` |

---

#### `user.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `ensure_user` `:42` | `INSERT INTO users (community_id, pubkey) … ON CONFLICT DO NOTHING`; returns `rows_affected == 1` | `users` |
| `get_user` `:58` | `query_as` tuple select | `users` |
| `update_user_profile` `:103` | dynamic SET list (`format!` + `AssertSqlSafe`, positional binds); empty string ⇒ NULL | `users` |
| `get_user_by_nip05` `:169` | `LOWER(nip05_handle) = LOWER($2) LIMIT 1` | `users` |
| `search_users` `:224` | `escape_like` `:214` escapes `\ % _`, `LIKE … ESCAPE '\'` over display name / handle / `encode(pubkey,'hex')`; 6-tier `CASE` ranking; `limit.clamp(1, 500)` | `users` |
| `set_agent_owner` `:291` | conditional `UPDATE … WHERE agent_owner_pubkey IS NULL` (first-mint-wins), then existence probe to distinguish not-found from already-owned | `users` |
| `get_agent_channel_policy` `:330` | `channel_add_policy::text` + `agent_owner_pubkey` | `users` |
| `is_agent_owner` `:354` | `SELECT agent_owner_pubkey = $3 … AND agent_owner_pubkey IS NOT NULL` | `users` |
| `set_channel_add_policy` `:374` | Rust-side vocabulary check then `$1::channel_add_policy` cast | `users` |

---

#### `api_token.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `create_api_token` `:15` | plain insert of a caller-supplied SHA-256 hash | `api_tokens` |
| `create_api_token_if_under_limit` `:69` | `INSERT … SELECT … WHERE (SELECT COUNT(*) … community_id AND owner_pubkey AND revoked_at IS NULL AND (expires_at IS NULL OR expires_at > NOW())) < 10`; `created_by_self_mint = TRUE` | `api_tokens` |
| `get_api_token_by_hash_including_revoked` `:144` | `WHERE community_id=$1 AND token_hash=$2` | `api_tokens` |
| `list_tokens_by_owner` `:208` | all tokens incl. revoked, `ORDER BY created_at DESC` (**no LIMIT**) | `api_tokens` |
| `revoke_token` `:272` | `UPDATE … WHERE community_id AND id AND owner_pubkey AND revoked_at IS NULL` | `api_tokens` |
| `revoke_all_tokens` `:303` | same without `id` | `api_tokens` |

---

#### `workflow.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `create_workflow` `:276` | insert with `'active'`/`TRUE`; documented "No current callers" | `workflows` |
| `upsert_workflow` `:313` | `ON CONFLICT (community_id, id) DO UPDATE … WHERE workflows.owner_pubkey = EXCLUDED.owner_pubkey AND workflows.channel_id IS NOT DISTINCT FROM EXCLUDED.channel_id RETURNING id`; `None` ⇒ `AccessDenied` | `workflows` |
| `get_workflow` `:363` | scoped select | `workflows` |
| `list_channel_workflows` `:389` | `limit.clamp(1, LIST_MAX_LIMIT)`, `offset.max(0)` | `workflows` |
| `list_enabled_channel_workflows` `:425` | `status='active' AND enabled` , `LIMIT LIST_MAX_LIMIT` | `workflows` |
| `list_all_enabled_workflows` `:457` | global scan `JOIN communities … WHERE definition->'trigger'->>'on' = 'schedule' AND c.archived_at IS NULL LIMIT 1000` | `workflows`, `communities` |
| `claim_scheduled_workflow_fire` `:496` | `INSERT … SELECT w.community_id, w.id, $3 FROM workflows w WHERE community_id AND id ON CONFLICT DO NOTHING RETURNING …` | `scheduled_workflow_fires`, `workflows` |
| `latest_scheduled_workflow_fire` `:537` | `MAX(scheduled_for)` | `scheduled_workflow_fires` |
| `attach_scheduled_workflow_run` `:563` | `UPDATE … WHERE workflow_run_id IS NULL` | `scheduled_workflow_fires` |
| `prune_scheduled_workflow_fires_before` `:597` | global `DELETE … WHERE claimed_at < $1` | `scheduled_workflow_fires` |
| `update_workflow` `:620`, `update_workflow_status` `:654`, `set_workflow_enabled` `:684`, `delete_workflow` `:715` | scoped update/delete; 0 rows ⇒ `NotFound`; all documented "No current callers" | `workflows` |
| `delete_workflow_for_owner` `:738` | `DELETE … AND owner_pubkey=$3 RETURNING channel_id` | `workflows` (cascades) |
| `create_workflow_run` `:767` | insert `'pending'`, `execution_trace='[]'` | `workflow_runs` |
| `get_workflow_run` `:795`, `list_workflow_runs` `:818` (`limit.min(1000)`) | scoped reads | `workflow_runs` |
| `update_workflow_run` `:850` | sets status/step/trace/error, stamps `started_at` when the **bind** = `'running'` and it is NULL, `completed_at` for terminal states | `workflow_runs` |
| `create_approval` `:918` | hashes the raw token with SHA-256 (`hash_approval_token` `:33`) then inserts | `workflow_approvals` |
| `get_approval` `:956` / `get_approval_by_stored_hash` `:973` | scoped `(community_id, token)` lookup | `workflow_approvals` |
| `get_run_approvals` `:996` | `ORDER BY step_index, created_at` | `workflow_approvals` |
| `update_approval` `:1031` / `update_approval_by_stored_hash` `:1059` | `UPDATE … WHERE community_id AND token AND status='pending'`; stamps `granted_at`/`denied_at` | `workflow_approvals` |
| `find_by_owner_and_name` `:1169` | `WHERE community_id AND owner_pubkey AND name LIMIT 1` | `workflows` |

---

#### `push.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `accept_lease_event` `:213` | tx: SHA-256-derived address lock + author lock (+ push-gate lock when activating) → source-event collision probe → `FOR UPDATE` ordering probe → expire stale active leases → quota + endpoint-uniqueness checks → soft-delete prior kind:30350 → insert event → upsert lease → `backfill_push_match_jobs` | `push_leases`, `events`, `push_match_queue` |
| `replace_active_lease` `:419` / `revoke_lease` `:439` / private `replace_lease` `:447` | upsert whose `ON CONFLICT … WHERE` clause **is** the ordering state machine (`source_created_at`/`source_event_id` then `generation`) | `push_leases`, `push_match_queue` |
| `enqueue_wake` `:580` | batch-of-one wrapper over `enqueue_wakes` | as below |
| `enqueue_wakes` `:619` | tx: `FOR UPDATE` lock of distinct lease rows via `UNNEST` in deterministic order → per-request generation match → one multi-row `INSERT … ON CONFLICT (community_id, endpoint_hash, event_id) DO NOTHING RETURNING …` → set-wise duplicate lookup | `push_leases`, `push_wake_outbox` |
| `claim_due_match_batch` `:819` (+ `_with_loader` `:833`) | CTE picks ONE community, `FOR UPDATE OF q SKIP LOCKED LIMIT $4`, `attempts+1`; then loads events; jobs with no live event are deleted | `push_match_queue`, `events` |
| `reap_exhausted_matches` `:933` | global `DELETE … WHERE attempts >= 8 AND (pending OR expired lease)` | `push_match_queue` |
| `active_match_leases` `:945` | active + endpoint-enabled + unexpired leases for one community | `push_leases` |
| `complete_match_batch` `:970` / `retry_match_batch` `:993` | fenced `DELETE` / `UPDATE` by `claim_id` | `push_match_queue` |
| `claim_due_wakes` `:1021` | CTE joins outbox ⋈ lease (community, author, installation, generation, endpoint_hash) ⋈ live event; `FOR UPDATE OF o SKIP LOCKED`; `attempts+1` | `push_wake_outbox`, `push_leases`, `events` |
| `revalidate_wake_for_send` `:1085` | the load-bearing send-time re-join (`state='sending'`, `lease_until >= now()`, unexpired, active, endpoint-enabled) | `push_wake_outbox`, `push_leases`, `events` |
| `complete_wake` `:1132` / `retry_wake` `:1152` / `fail_wake` `:1174` | fenced state transitions requiring `claim_id` match | `push_wake_outbox` |
| `disable_endpoint_generation` `:1197` | `UPDATE … WHERE generation=$4 AND active AND endpoint_enabled` | `push_leases` |
| `prune_wake_outbox` `:1223` | `DELETE … terminal/expired AND NOT EXISTS (matching push_match_queue row)` | `push_wake_outbox`, `push_match_queue` |
| private `acquire_push_gate_lock` `:24`, `backfill_push_match_jobs` `:52`, `constraint_acceptance_outcome` `:392`, `row_to_claimed_wake` `:1245` | — | — |

---

#### `moderation.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `insert_report` `:172` | `ON CONFLICT (community_id, report_event_id) DO UPDATE SET report_event_id = EXCLUDED.report_event_id RETURNING id` (idempotent) | `moderation_reports` |
| `list_reports` `:213` | `($2::text IS NULL OR status = $2)`, `ORDER BY created_at DESC LIMIT $3` | `moderation_reports` |
| `get_report` `:240` / `get_report_by_event` `:263` | scoped single-row reads | `moderation_reports` |
| `resolve_report` `:287` | `UPDATE … AND status='open'` | `moderation_reports` |
| `ban_member` `:314` / `timeout_member` `:371` | upserts keyed `(community_id, pubkey)` | `community_bans` |
| `unban_member` `:347` | `… AND banned = true` | `community_bans` |
| `untimeout_member` `:403` | `… AND muted_until > now()` | `community_bans` |
| `restriction_state` `:441` | computes `banned AND (ban_expires_at IS NULL OR ban_expires_at > now())` and future-only `muted_until`; missing row ⇒ default | `community_bans` |
| `get_ban` `:470` / `list_restricted` `:494` | same expiry-aware projection | `community_bans` |
| `insert_action` `:518` | plain insert `RETURNING id` | `moderation_actions` |
| `list_actions` `:549` | `ORDER BY created_at DESC LIMIT $2` | `moderation_actions` |

---

#### `admin_moderation.rs` — deployment-global reads (documented exception)

| Fn | Operation | Tables |
|----|-----------|--------|
| `list_reports` `:85` | keyset `(created_at, id) < ($7,$8)`, optional community/status/type/target/time filters, `bounded_limit` clamps to `MAX_PAGE_SIZE=200` | `moderation_reports`, `communities` |
| `get_report` `:132` | `WHERE r.id = $1` (no community predicate) | `moderation_reports`, `communities` |
| `list_feedback` `:181` / `get_feedback` `:200` | newest-first / by row id | `product_feedback`, `communities` |

---

#### `product_feedback.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `insert` `:59` | `ON CONFLICT (event_id) DO UPDATE SET event_id = EXCLUDED.event_id RETURNING id` — deployment-wide idempotency, first community keeps provenance | `product_feedback` |
| `list` `:89` | `ORDER BY received_at DESC, id LIMIT $1` — **no community filter** | `product_feedback` |

---

#### `relay_members.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `is_relay_member` `:31`, `get_relay_member` `:41`, `list_relay_members` `:69` | scoped reads | `relay_members` |
| `add_relay_member` `:97` | `ON CONFLICT (community_id, pubkey) DO NOTHING` | `relay_members` |
| `claim_relay_membership` `:122` | tx: member insert + optional `join_policy_acceptances` insert | `relay_members`, `join_policy_acceptances` |
| `has_join_policy_acceptance` `:160` | existence probe | `join_policy_acceptances` |
| `remove_relay_member` `:196` | `DELETE … AND role <> 'owner'`, then one probe to distinguish `IsOwner`/`NotFound` | `relay_members` |
| `remove_relay_member_if_role` `:242` | `DELETE … AND role = $3`, then probe → `IsOwner`/`RoleMismatch`/`NotFound` | `relay_members` |
| `update_relay_member_role` `:287` | `UPDATE … AND role <> 'owner'` | `relay_members` |
| `bootstrap_owner` `:320` | tx: upsert owner → demote other owners to `'admin'`; does **not** enforce the per-owner limit | `relay_members` |
| `owner_count_advisory_lock_key` `:410` | pure FNV-1a over the hex pubkey | — |
| `transfer_ownership` `:437` | tx: advisory lock on transferee → `SELECT … role='owner' FOR UPDATE` → expected-owner check → cross-community owner count vs `MAX_COMMUNITIES_PER_OWNER` → upsert new owner → demote others to `'member'` | `relay_members` |
| `backfill_from_allowlist` `:542` | `information_schema` existence probe → empty-target guard → `INSERT … SELECT encode(pubkey,'hex') FROM pubkey_allowlist WHERE community_id=$1` | `pubkey_allowlist`, `relay_members` |

---

#### `archived_identities.rs`, `git_repo.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `archived_identities::is_archived` `:34` | `SELECT 1 …` | `archived_identities` |
| `archive` `:50` | `ON CONFLICT (community_id, pubkey) DO NOTHING` | `archived_identities` |
| `unarchive` `:83` | scoped `DELETE` | `archived_identities` |
| `list_archived` `:95` | `ORDER BY archived_at ASC` | `archived_identities` |
| `git_repo::repo_name_owner` `:47` | scoped owner read | `git_repo_names` |
| `reserve_repo_name` `:81` | `INSERT … ON CONFLICT (community_id, repo_id) DO NOTHING RETURNING owner_pubkey`, then classification read | `git_repo_names` |
| `count_repos_for_owner` `:142` | `COUNT(*) … owner_pubkey=$2` (quota input) | `git_repo_names` |
| `release_repo_name` `:164` | owner-scoped `DELETE` | `git_repo_names` |

---

#### `usage.rs` — operator rollups (all cross-community by design)

| Fn | Operation | Tables |
|----|-----------|--------|
| `community_count` `:20` | `COUNT(*)` | `communities` |
| `user_counts` `:41` | `COUNT(*) FILTER (…)` split human/agent, `WHERE deactivated_at IS NULL GROUP BY community_id` | `users` |
| `channel_counts` `:79` | `GROUP BY community_id, channel_type WHERE deleted_at IS NULL` | `channels` |
| `message_counts` `:113` | `WHERE kind = 9 AND deleted_at IS NULL GROUP BY community_id` | `events` |
| `relay_member_counts` `:146` | `GROUP BY community_id, role` | `relay_members` |
| `workflow_counts` `:179` | `GROUP BY community_id, status` | `workflows` |
| `git_repo_counts` `:210` | `GROUP BY community_id` | `git_repo_names` |
| `active_user_counts` `:254` | `format!`-interpolated `INTERVAL '{interval_sql}'` (`&'static str`) + `AssertSqlSafe`; 3-way human/agent/unknown classification | `events`, `users` |
| `active_channel_counts` `:308` | same interval interpolation; `COUNT(DISTINCT channel_id) WHERE kind = 9` | `events` |
| `community_hosts` `:347` | `SELECT id, host FROM communities` | `communities` |

---

#### `partition.rs`, `migration.rs`, `replica_fence.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `partition::ensure_future_partitions` `:15` | loops months, computes suffix/date strings, calls private `ensure_partition` `:76` | `pg_catalog.pg_class`/`pg_namespace`; DDL `CREATE TABLE … PARTITION OF` on `events` and `delivery_log` |
| `migration::run_migrations` `:14` | pre-flight `reject_legacy_nip_rs_cardinality_ambiguity` `:34` → `MIGRATOR.run(pool)` → `verify_floor_guard_catalog` | `_sqlx_migrations`, `events`, all |
| `replica_fence::verify_floor_guard_catalog` `:145` | catalog shape check on `events` + every partition (`pg_trigger` tgtype bits) | `pg_inherits`, `pg_class`, `pg_trigger` |
| `replica_fence::verify_floor_guard_behavior` `:199` | one rolled-back tx: `SHOW buzz.created_at_floor`, `SET CONSTRAINTS ALL IMMEDIATE`, 5 adversarial inserts/updates under savepoints | `communities`, `events` (rolled back) |
| `replica_fence::probe_once` `:466` (private `sample_writer` `:404`, `replica_covers` `:449`) | writer: `clock_timestamp()`, then `pg_stat_activity`/`pg_prepared_xacts` scan, then `pg_current_wal_lsn()`; replica: `pg_last_wal_replay_lsn() >= $1` gated on `pg_is_in_recovery()` | system views only |
| `replica_fence::run_probe` `:488` | 5 s loop, closes the fence on any error | — |
| `ReplicaFence::new` `:88`, `close` `:97`, `verified_through` `:114`, `covers` `:129`, `force_open_for_tests` `:136` | in-memory atomics | — |

---

#### Public constants

| Constant | Value | File:line |
|----------|-------|-----------|
| `admin_moderation::MAX_PAGE_SIZE` | 200 | `crates/buzz-db/src/admin_moderation.rs:15` |
| `event::D_TAG_MAX_LEN` | 1024 | `crates/buzz-db/src/event.rs:140` |
| `feed::FEED_MAX_LIMIT` | 100 | `crates/buzz-db/src/feed.rs:29` |
| `moderation::MODERATION_ACTION_CHECK_VOCAB` | 12 strings | `crates/buzz-db/src/moderation.rs:104` |
| `push::MAX_MATCH_ATTEMPTS` | 8 | `crates/buzz-db/src/push.rs:70` |
| `relay_members::MAX_COMMUNITIES_PER_OWNER` | 3 | `crates/buzz-db/src/relay_members.rs:379` |
| `replica_fence::CREATED_AT_FLOOR_SECS` | 960 | `crates/buzz-db/src/replica_fence.rs:51` |
| `replica_fence::FENCE_CLOCK_MARGIN_SECS` | 5 | `crates/buzz-db/src/replica_fence.rs:59` |
| `replica_fence::PROBE_INTERVAL` | 5 s | `crates/buzz-db/src/replica_fence.rs:62` |
| `replica_fence::FENCE_STALENESS` | 30 s | `crates/buzz-db/src/replica_fence.rs:66` |
| `workflow::LIST_DEFAULT_LIMIT` | 100 | `crates/buzz-db/src/workflow.rs:25` |
| `workflow::LIST_MAX_LIMIT` | 1000 | `crates/buzz-db/src/workflow.rs:27` |


## Module: buzz-auth (`crates/buzz-auth`)

### Aspect: API Surface

Crate lints: `#![deny(unsafe_code)]` and `#![warn(missing_docs)]`
(`crates/buzz-auth/src/lib.rs:1-2`).

### Modules (all `pub`)

| Module | Declared | Contents |
|--------|----------|----------|
| `access` | `crates/buzz-auth/src/lib.rs:19` | `ChannelAccessChecker`, scope/access helpers, `MockAccessChecker` |
| `error` | `crates/buzz-auth/src/lib.rs:21` | `AuthError` |
| `nip42` | `crates/buzz-auth/src/lib.rs:23` | challenge gen + AUTH verification |
| `nip98` | `crates/buzz-auth/src/lib.rs:25` | HTTP auth verification |
| `nip98_replay` | `crates/buzz-auth/src/lib.rs:27` | `Nip98ReplayGuard`, keys, TTL constants |
| `rate_limit` | `crates/buzz-auth/src/lib.rs:29` | `RateLimiter`, config, keys |
| `scope` | `crates/buzz-auth/src/lib.rs:31` | `Scope`, `parse_scopes` |

### Root re-exports

| Re-export list | file:line |
|----------------|-----------|
| `access::{check_read_access, check_write_access, require_scope, ChannelAccessChecker}` | `crates/buzz-auth/src/lib.rs:33` |
| `error::AuthError` | `crates/buzz-auth/src/lib.rs:34` |
| `nip42::{generate_challenge, verify_nip42_event}` | `crates/buzz-auth/src/lib.rs:35` |
| `nip98::verify_nip98_event` | `crates/buzz-auth/src/lib.rs:36` |
| `nip98_replay::{nip98_replay_key, nip98_replay_key_for_scope, Nip98ReplayGuard, DEFAULT_REPLAY_TTL_SECS, MAX_REPLAY_TTL_SECS}` | `crates/buzz-auth/src/lib.rs:37-40` |
| `rate_limit::{ip_rate_limit_key, rate_limit_key, LimitType, RateLimitConfig, RateLimitResult, RateLimiter}` | `crates/buzz-auth/src/lib.rs:41-43` |
| `scope::{parse_scopes, Scope}` | `crates/buzz-auth/src/lib.rs:44` |
| `access::MockAccessChecker` — `#[cfg(any(test, feature = "test-utils"))]` | `crates/buzz-auth/src/lib.rs:46-47` |
| `nip98_replay::AlwaysFreshReplayGuard` — same cfg | `crates/buzz-auth/src/lib.rs:48-49` |
| `rate_limit::AlwaysAllowRateLimiter` — same cfg | `crates/buzz-auth/src/lib.rs:50-51` |

Not re-exported at root (module-path access only): `AuthMethod`, `AuthContext`,
`AuthConfig`, `AuthService` are defined directly in `lib.rs` so they are already
root items (`crates/buzz-auth/src/lib.rs:55`, `:64`, `:91`, `:100`).

---

### Public functions

| Signature | Returns | file:line |
|-----------|---------|-----------|
| `pub fn generate_challenge() -> String` | 32 CSPRNG bytes hex-encoded | `crates/buzz-auth/src/nip42.rs:38` |
| `pub fn verify_nip42_event(event: &Event, expected_challenge: &str, relay_url: &str) -> Result<(), AuthError>` | `()` on success | `crates/buzz-auth/src/nip42.rs:47-51` |
| `pub fn verify_nip98_event(event_json: &str, expected_url: &str, expected_method: &str, body: Option<&[u8]>) -> Result<nostr::PublicKey, AuthError>` | authenticated pubkey | `crates/buzz-auth/src/nip98.rs:55-60` |
| `pub fn require_scope(scopes: &[Scope], required: Scope) -> Result<(), AuthError>` | `()` or `InsufficientScope` | `crates/buzz-auth/src/access.rs:60` |
| `pub async fn check_read_access(checker: &impl ChannelAccessChecker, ctx: &TenantContext, pubkey: &PublicKey, channel_id: Uuid, scopes: &[Scope]) -> Result<(), AuthError>` | `()` or error | `crates/buzz-auth/src/access.rs:72-78` |
| `pub async fn check_write_access(checker: &impl ChannelAccessChecker, ctx: &TenantContext, pubkey: &PublicKey, channel_id: Uuid, scopes: &[Scope]) -> Result<(), AuthError>` | `()` or error | `crates/buzz-auth/src/access.rs:88-94` |
| `pub fn parse_scopes(raw: &[impl AsRef<str>]) -> Vec<Scope>` | never fails | `crates/buzz-auth/src/scope.rs:170` |
| `pub fn rate_limit_key(ctx: &TenantContext, pubkey: &PublicKey, limit_type: &LimitType) -> String` | Redis key | `crates/buzz-auth/src/rate_limit.rs:201` |
| `pub fn ip_rate_limit_key(ip: &IpAddr) -> String` | Redis key | `crates/buzz-auth/src/rate_limit.rs:213` |
| `pub fn nip98_replay_key(ctx: &TenantContext, event_id: &EventId) -> String` | Redis key | `crates/buzz-auth/src/nip98_replay.rs:114` |
| `pub fn nip98_replay_key_for_scope(scope: &str, event_id: &EventId) -> String` | Redis key | `crates/buzz-auth/src/nip98_replay.rs:119` |
| `pub fn derive_pubkey_from_username(username: &str) -> Result<nostr::PublicKey, AuthError>` — `#[cfg(any(test, feature = "dev"))]` | derived pubkey | `crates/buzz-auth/src/lib.rs:159-160` |

Private helpers: `normalize_relay_url(raw: &str) -> String`
(`crates/buzz-auth/src/nip42.rs:19`), `normalize_url(raw: &str) -> String`
(`crates/buzz-auth/src/nip98.rs:145`), and the seven `default_*` fns
(`crates/buzz-auth/src/rate_limit.rs:110-130`).

---

### Public inherent methods

| Type | Method | file:line |
|------|--------|-----------|
| `AuthContext` | `pub fn has_scope(&self, scope: &Scope) -> bool` | `crates/buzz-auth/src/lib.rs:84` |
| `AuthService` | `pub fn new(config: AuthConfig) -> Self` | `crates/buzz-auth/src/lib.rs:106` |
| `AuthService` | `pub fn config(&self) -> &AuthConfig` | `crates/buzz-auth/src/lib.rs:111` |
| `AuthService` | `pub async fn verify_auth_event(&self, auth_event: nostr::Event, expected_challenge: &str, relay_url: &str) -> Result<AuthContext, AuthError>` | `crates/buzz-auth/src/lib.rs:118-123` |
| `Scope` | `pub fn all_known() -> Vec<Scope>` (16 items) | `crates/buzz-auth/src/scope.rs:68` |
| `Scope` | `pub fn all_non_admin() -> Vec<Scope>` (14 items) | `crates/buzz-auth/src/scope.rs:94` |
| `Scope` | `pub fn as_str(&self) -> &str` | `crates/buzz-auth/src/scope.rs:114` |
| `LimitType` | `pub fn key_suffix(&self) -> &'static str` | `crates/buzz-auth/src/rate_limit.rs:71` |
| `RateLimitResult` | `pub fn allowed(current: u64, limit: u64, reset_in_secs: u64) -> Self` | `crates/buzz-auth/src/rate_limit.rs:32` |
| `RateLimitResult` | `pub fn denied(current: u64, limit: u64, reset_in_secs: u64) -> Self` | `crates/buzz-auth/src/rate_limit.rs:42` |
| `MockAccessChecker` | `pub fn new() -> Self` (cfg-gated) | `crates/buzz-auth/src/access.rs:115` |
| `MockAccessChecker` | `pub fn allow(&mut self, ctx: &TenantContext, pubkey: &PublicKey, channel_id: Uuid)` (cfg-gated) | `crates/buzz-auth/src/access.rs:122` |

Derived/blanket trait impls: `Display for Scope`
(`crates/buzz-auth/src/scope.rs:137`), `FromStr for Scope` with
`Err = Infallible` (`crates/buzz-auth/src/scope.rs:143-144`), `Default for
RateLimitConfig` (`crates/buzz-auth/src/rate_limit.rs:132`), `Default for
MockAccessChecker` (`crates/buzz-auth/src/access.rs:129`).

---

### Trait: `ChannelAccessChecker` (`crates/buzz-auth/src/access.rs:31-57`)

Supertraits: `Send + Sync`. Uses RPITIT (return-position `impl Future`), so the
trait is **not** dyn-compatible.

```rust
pub trait ChannelAccessChecker: Send + Sync {
    fn accessible_channel_ids(
        &self,
        ctx: &TenantContext,
        pubkey: &PublicKey,
    ) -> impl Future<Output = Result<HashSet<Uuid>, AuthError>> + Send;   // access.rs:35-39

    fn can_access(
        &self,
        ctx: &TenantContext,
        pubkey: &PublicKey,
        channel_id: Uuid,
    ) -> impl Future<Output = Result<bool, AuthError>> + Send { /* default */ } // access.rs:46-56
}
```

`can_access` has a default body that calls `accessible_channel_ids` and does a
`HashSet::contains` (`crates/buzz-auth/src/access.rs:52-55`). Only
`accessible_channel_ids` is required.

Contract in the doc comment: every method takes `&TenantContext` and
"Implementations MUST scope every query by `ctx.community()`"
(`crates/buzz-auth/src/access.rs:22-30`).

**Implementors in this crate:** exactly one, and it is test-only.

| Implementor | Gate | file:line |
|-------------|------|-----------|
| `MockAccessChecker` | `#[cfg(any(test, feature = "test-utils"))]` | `crates/buzz-auth/src/access.rs:135-151` |

Repo-wide search for `impl ... ChannelAccessChecker` and for the identifier
`ChannelAccessChecker` outside this crate returns **no** matches — the trait's
doc claim that it is "Implemented by the database layer (`buzz-db`) in
production" (`crates/buzz-auth/src/access.rs:18-19`) is not backed by code.

---

### Trait: `RateLimiter` (`crates/buzz-auth/src/rate_limit.rs:168-194`)

Supertraits: `Send + Sync`. Also RPITIT — not dyn-compatible (consumers take it
as a generic bound; see `crates/buzz-relay/src/admission.rs:17`).

```rust
pub trait RateLimiter: Send + Sync {
    fn check_and_increment(
        &self,
        ctx: &TenantContext,
        pubkey: &PublicKey,
        limit_type: LimitType,
        window_secs: u64,
        limit: u64,
    ) -> impl std::future::Future<Output = Result<RateLimitResult, AuthError>> + Send; // rate_limit.rs:174-181

    fn check_ip_connection(
        &self,
        ip: &IpAddr,
        window_secs: u64,
        limit: u64,
    ) -> impl std::future::Future<Output = Result<RateLimitResult, AuthError>> + Send; // rate_limit.rs:188-193
}
```

No default bodies — both methods are required.

Documented scoping contract: pubkey-keyed limits are community-prefixed;
IP-keyed limits are deliberately operator-global and take no `TenantContext`
(`crates/buzz-auth/src/rate_limit.rs:151-163`). Documented algorithm caveat:
fixed windows permit up to 2× burst at boundaries
(`crates/buzz-auth/src/rate_limit.rs:165-167`, also `:6-7`).

**Implementors in this crate:** exactly one, test-only.

| Implementor | Gate | Behaviour | file:line |
|-------------|------|-----------|-----------|
| `AlwaysAllowRateLimiter` (unit struct) | `#[cfg(any(test, feature = "test-utils"))]` | both methods return `RateLimitResult::allowed(1, limit, window_secs)` | decl `crates/buzz-auth/src/rate_limit.rs:218-219`, impl `:221-242` |

Implementors elsewhere in the repo (see the security aspect for the full
verdict): `RedisRateLimiter` in `crates/buzz-pubsub/src/rate_limiter.rs:99` and a
test `StubLimiter` in `crates/buzz-relay/src/admission.rs:69`.

---

### Trait: `Nip98ReplayGuard` (`crates/buzz-auth/src/nip98_replay.rs:64-104`)

Supertraits: `Send + Sync`. Unlike the other two traits this one uses
`Pin<Box<dyn Future ...>>` returns, so it **is** dyn-compatible — the relay
stores it as `Arc<dyn Nip98ReplayGuard>`
(`crates/buzz-relay/src/state.rs:582`).

```rust
pub trait Nip98ReplayGuard: Send + Sync {
    fn try_mark_in_scope<'a>(
        &'a self,
        scope: &'a str,
        event_id: &'a EventId,
        ttl_secs: u64,
    ) -> Pin<Box<dyn Future<Output = Result<bool, AuthError>> + Send + 'a>>;  // nip98_replay.rs:66-71

    fn try_mark<'a>(
        &'a self,
        ctx: &'a TenantContext,
        event_id: &'a EventId,
        ttl_secs: u64,
    ) -> Pin<Box<dyn Future<Output = Result<bool, AuthError>> + Send + 'a>> { /* default */ } // nip98_replay.rs:97-103
}
```

`try_mark` default body derives the scope from `ctx.community().to_string()` and
delegates to `try_mark_in_scope` (`crates/buzz-auth/src/nip98_replay.rs:99-102`).

Documented obligations on implementors
(`crates/buzz-auth/src/nip98_replay.rs:73-96`): `Ok(true)` = newly inserted
(proceed), `Ok(false)` = replay (caller MUST reject); on `Err` callers MUST fail
closed; the operation MUST be atomic set-if-absent; `ttl_secs` MUST be clamped
up to `DEFAULT_REPLAY_TTL_SECS` and down to `MAX_REPLAY_TTL_SECS`.

**Implementors in this crate:** exactly one, test-only.

| Implementor | Gate | Behaviour | file:line |
|-------------|------|-----------|-----------|
| `AlwaysFreshReplayGuard` (unit struct) | `#[cfg(any(test, feature = "test-utils"))]` | `try_mark_in_scope` always `Ok(true)` | decl `crates/buzz-auth/src/nip98_replay.rs:126-127`, impl `:129-139` |

Production implementor lives outside the crate: `RedisNip98ReplayGuard`
(`crates/buzz-pubsub/src/nip98_replay.rs:34`).

---

### Error type

`pub enum AuthError` with 10 variants (`crates/buzz-auth/src/error.rs:9-59`) —
full table in the conventions aspect. It is the error type of every fallible
public fn in the crate.


## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: API Surface

`buzz-pubsub` exposes **no HTTP routes, no WebSocket handlers, and no CLI**. Its
entire API surface is a Rust library facade consumed by `buzz-relay`. Public items
were enumerated from all 10 `.rs` files.

Consumers declaring the dependency (`grep buzz-pubsub` across manifests):
`crates/buzz-relay/Cargo.toml`, `crates/buzz-admin/Cargo.toml`,
`crates/buzz-conformance/Cargo.toml`. Only `buzz-relay` exercises it in the code
paths verified below.

---

### 1. Module tree (`lib.rs:24-58`)

| Item | Line | Visibility |
|---|---|---|
| `pub mod cache_invalidation` | `lib.rs:25` | public |
| `pub mod conn_control` | `lib.rs:27` | public |
| `pub mod error` | `lib.rs:29` | public |
| `pub mod nip98_replay` | `lib.rs:31` | public |
| `pub use nip98_replay::RedisNip98ReplayGuard` | `lib.rs:32` | re-export |
| `pub mod presence` | `lib.rs:34` | public |
| `pub mod publisher` | `lib.rs:36` | public |
| `pub mod rate_limiter` | `lib.rs:38` | public |
| `pub mod subscriber` | `lib.rs:40` | public — but every item inside is `pub(crate)` |
| `pub mod topic` | `lib.rs:42` | public |
| `pub use error::PubSubError` | `lib.rs:44` | re-export |
| `pub use crate::topic::{channel_key, global_key, EventTopic, EventTopicKey}` | `lib.rs:58` | re-export |

`pub mod subscriber` (`lib.rs:40`) is declared public yet contains **zero public
items** — `DesiredTopics` (`subscriber.rs:21`), `SubscriptionCommand`
(`subscriber.rs:26`), and `run_subscriber` (`subscriber.rs:41`) are all
`pub(crate)`. The module is therefore an empty public namespace.

`channel_key` / `global_key` are reachable by **three distinct paths**:
`buzz_pubsub::channel_key` (re-export `lib.rs:58`), `buzz_pubsub::topic::channel_key`
(`topic.rs:103`), and `buzz_pubsub::publisher::channel_key` (`publisher.rs:12`) —
the last being a one-line delegate to the first (`publisher.rs:12-14`, `:17-19`).

### 2. `PubSubManager` — the primary facade (`lib.rs:100-368`)

| Method | Line | Signature summary | Verified production caller |
|---|---|---|---|
| `new` | `lib.rs:117` | `(&str, Pool) -> Result<Self, PubSubError>` | `buzz-relay/src/state.rs` construction path |
| `with_config` | `lib.rs:122` | `(PubSubConfig, Pool) -> Result<Self, PubSubError>` | tests only (`lib.rs:513`, `:594`) |
| `run_subscriber` | `lib.rs:148` | `(self: Arc<Self>)` — never returns | `buzz-relay/src/main.rs` (spawned) |
| `run_cache_invalidation_subscriber` | `lib.rs:165` | `(self: Arc<Self>)` — never returns | `buzz-relay/src/main.rs` |
| `run_conn_control_subscriber` | `lib.rs:175` | `(self: Arc<Self>)` — never returns | `buzz-relay/src/main.rs:366` |
| `subscribe_local` | `lib.rs:184` | `-> broadcast::Receiver<ChannelEvent>` | `buzz-relay/src/main.rs:822`, `handlers/event.rs:1644` |
| `retain_topic` | `lib.rs:192` | `(&TenantContext, EventTopic)` | `handlers/req.rs:256`, `handlers/event.rs:1683`, `:1687` |
| `release_topic` | `lib.rs:215` | `(&TenantContext, EventTopic)` | `connection.rs:268`, `handlers/close.rs:21`, `handlers/req.rs:251`, `handlers/side_effects.rs:81` |
| `topic_refcount` | `lib.rs:248` | `-> usize` | **none** — in-crate tests only |
| `subscribe_cache_invalidations` | `lib.rs:259` | `-> broadcast::Receiver<ScopedCacheInvalidation>` | `buzz-relay/src/main.rs` |
| `subscribe_conn_control` | `lib.rs:264` | `-> broadcast::Receiver<ScopedConnControl>` | `buzz-relay/src/main.rs:903` |
| `publish_cache_invalidation` | `lib.rs:272` | `-> Result<i64, PubSubError>` | `buzz-relay/src/state.rs:970` |
| `publish_conn_control` | `lib.rs:292` | `-> Result<i64, PubSubError>` | `buzz-relay/src/state.rs:1044`, `:1066` |
| `publish_event` | `lib.rs:322` | `(&TenantContext, EventTopic, &nostr::Event) -> Result<i64, PubSubError>` | relay event ingest |
| `set_presence` | `lib.rs:332` | `(&TenantContext, &PublicKey, &str)` | `handlers/event.rs:798` |
| `clear_presence` | `lib.rs:342` | — | `connection.rs:280`, `handlers/event.rs:793` |
| `get_presence` | `lib.rs:351` | `-> Result<Option<String>, _>` | — (bulk variant is used instead) |
| `get_presence_bulk` | `lib.rs:360` | `-> Result<HashMap<String,String>, _>` | `api/bridge.rs:1972` |

All 18 methods return either `()`, a `broadcast::Receiver`, `usize`, or
`Result<_, PubSubError>`; the three `publish_*` methods return the Redis
subscriber count as `i64` (`lib.rs:279`, `:299`, `publisher.rs:27`).

`topic_refcount` (`lib.rs:248`) is documented "for tests and metrics" and has
**zero non-test callers** — no metric is exported from it.

### 3. `PubSubConfig` (`lib.rs:73-97`)

`DEFAULT_UNSUBSCRIBE_DEBOUNCE = 500ms` (`lib.rs:82`); `new` (`lib.rs:85`);
builder `with_unsubscribe_debounce` (`lib.rs:93`). The builder has **zero
production callers** — only `lib.rs:514`, `:596`, `:621` (tests). Production
therefore always runs the 500 ms default via `PubSubManager::new` → `with_config`
(`lib.rs:117-119`).

### 4. `topic` module (`topic.rs`)

`BUZZ_PREFIX` (`:13`), `EventTopic` (`:17`), `EventTopicKey` (`:26`),
`EventTopicKey::from_context` (`:35`), `::redis_channel` (`:43`),
`::parse_redis_channel` (`:53`), free fns `channel_key` (`:103`), `global_key`
(`:108`). 8 public items.

### 5. `presence` module (`presence.rs`)

`PRESENCE_TTL_SECS: u64 = 90` (`:16`), `presence_key` (`:19`), `set_presence`
(`:28`), `clear_presence` (`:47`), `get_presence` (`:62`), `get_presence_bulk`
(`:74`). All four operations take `&Pool` directly, so callers may bypass
`PubSubManager` entirely — the relay's own presence integration test does exactly
that (`lib.rs:477-508`).

### 6. `cache_invalidation` module (`cache_invalidation.rs`)

`CACHE_INVALIDATION_SUFFIX` (`:23`), `CACHE_INVALIDATION_PATTERN` (`:27`),
`cache_invalidation_channel` (`:30`), `parse_cache_invalidation_channel` (`:38`),
`CacheInvalidation` (`:58`), `ScopedCacheInvalidation` (`:83`),
`run_cache_invalidation_subscriber` (`:100`). 7 public items.

### 7. `conn_control` module (`conn_control.rs`)

`CONN_CONTROL_SUFFIX` (`:26`), `CONN_CONTROL_PATTERN` (`:30`),
`conn_control_channel` (`:33`), `parse_conn_control_channel` (`:38`), `ConnControl`
(`:56`), `ScopedConnControl` (`:75`), `run_conn_control_subscriber` (`:90`).
7 public items. Both `ConnControl` variants have live producers and a consumer:
`DisconnectPubkey` built at `buzz-relay/src/state.rs:1034`, `DisconnectCommunity`
at `:1066`, both dispatched in `buzz-relay/src/main.rs:908` and `:913`.

### 8. Trait implementations exported to `buzz-auth` seams

| Impl | Line | Trait | Methods |
|---|---|---|---|
| `RedisRateLimiter` | `rate_limiter.rs:99` | `buzz_auth::rate_limit::RateLimiter` | `check_and_increment` (`:100`), `check_ip_connection` (`:112`) |
| `RedisNip98ReplayGuard` | `nip98_replay.rs:34` | `buzz_auth::nip98_replay::Nip98ReplayGuard` | `try_mark_in_scope` (`:35`) — returns a boxed pinned future, i.e. the trait is object-safe rather than `async fn` |

`RedisRateLimiter::new` (`rate_limiter.rs:94`) is constructed once at
`buzz-relay/src/state.rs:712` and stored as
`admission_rate_limiter: Arc<RedisRateLimiter>` (`state.rs:584`, import `state.rs:26`).
`RedisNip98ReplayGuard::new` (`nip98_replay.rs:29`) is constructed at
`buzz-relay/src/state.rs:711` (import `state.rs:27`) and additionally instantiated
in relay tests as two simulated pods (`api/bridge.rs:2275-2276`, `:2304`).

`check_ip_connection` (`rate_limiter.rs:112`) has **no production caller anywhere**.
The only other implementations are the trait declaration
(`buzz-auth/src/rate_limit.rs:188`, default/blanket at `:234`) and a `#[cfg(test)]`
`StubLimiter` inside `buzz-relay/src/admission.rs:85`.

### 9. Aggregate

Roughly 55 public items across 9 public modules. No `#[non_exhaustive]` markers on
any public enum or struct, so adding a `CacheInvalidation` or `ConnControl` variant,
or a `PubSubError` variant, is a semver-breaking change for downstream matchers.


## Module: buzz-search (`crates/buzz-search`)

### Aspect: API Surface

The crate exposes **two callable entry points** (one free fn, one method) plus one
constructor. Everything else public is data.

#### Crate root exports — `crates/buzz-search/src/lib.rs`

| Item | Line | Kind |
|---|---|---|
| `pub mod error` | `lib.rs:25` | module |
| `pub mod query` | `lib.rs:27` | module |
| `pub use buzz_core::CommunityId` | `lib.rs:29` | re-export |
| `pub use error::SearchError` | `lib.rs:30` | re-export |
| `pub use query::{search, ChannelScope, SearchHit, SearchMode, SearchQuery, SearchResult}` | `lib.rs:31` | re-export |
| `pub struct SearchService` | `lib.rs:40` | type |

Crate lints: `#![deny(unsafe_code)]` (`lib.rs:1`), `#![warn(missing_docs)]`
(`lib.rs:2`).

---

#### Functions and methods (complete list)

| Signature | Line | Public? |
|---|---|---|
| `impl SearchService { pub fn new(pool: PgPool) -> Self }` | `lib.rs:46-48` | yes |
| `impl SearchService { pub async fn search(&self, query: &SearchQuery) -> Result<SearchResult, SearchError> }` | `lib.rs:51-53` | yes |
| `pub async fn query::search(pool: &PgPool, query: &SearchQuery) -> Result<SearchResult, SearchError>` | `query.rs:216` | yes |
| `fn push_tsquery(qb: &mut QueryBuilder<sqlx::Postgres>, mode: SearchMode, search_text: &str)` | `query.rs:140` | private |
| `fn normalized_search_text(q: &str) -> Option<String>` | `query.rs:179` | private |

`SearchService::search` is a one-line delegate to the free function:
`query::search(&self.pool, query).await` (`lib.rs:52`). There is no other method
on `SearchService`, no `delete`, no `index`, no `count`.

No trait impls are hand-written; only derives (`Debug`, `Clone` on
`SearchService` at `lib.rs:39`; see data-model doc for the rest) and
`thiserror`'s `Error`/`Display`/`From<sqlx::Error>` on `SearchError`
(`error.rs:4-8`).

---

#### Construction

`SearchService::new(pool: PgPool)` takes ownership of a clone of the caller's
pool (`lib.rs:46-48`). No pool options, timeouts, or config are read here — the
pool is fully configured by the caller. Callers in-tree construct it from the
relay's pool, e.g. `crates/buzz-relay/src/state.rs:1273` and
`crates/buzz-relay/src/api/operator.rs:597`; the relay stores it as
`Arc<SearchService>` in `AppState` (`ARCHITECTURE.md:586`).

---

#### fn → SQL predicate shape → return type

| Fn | SQL emitted | Predicates | Returns |
|---|---|---|---|
| `SearchService::new` | none | — | `SearchService` |
| `SearchService::search` | delegates to `query::search` | see below | `Result<SearchResult, SearchError>` |
| `query::search` (empty/whitespace/NUL-only `q`) | **none — no roundtrip** (`query.rs:217-222`) | — | `Ok(SearchResult { hits: [], page: clamp(page,1,1000) })` |
| `query::search` (non-empty `q`) | single `SELECT` over `events` (`query.rs:233-298`) | `community_id = $1` **always** (`query.rs:240-241`); `deleted_at IS NULL` **always** (`query.rs:242`); `search_tsv @@ search_query.query` **always** (`query.rs:242`); then optional `channel_id` scope (`query.rs:248-264`), `kind = ANY` (`query.rs:267-273`), `pubkey = ANY` (`query.rs:275-281`), `created_at >= to_timestamp` (`query.rs:283-287`), `created_at <= to_timestamp` (`query.rs:289-293`) | `Result<SearchResult, SearchError>` |
| `push_tsquery` (private) | `websearch_to_tsquery('simple', $n)` for `FullText` (`query.rs:143-145`); token-split/`quote_literal`/`:*` aggregation subselect for `Prefix` (`query.rs:154-176`) | — | `()` (mutates the builder) |
| `normalized_search_text` (private) | none | — | `Option<String>` — `None` for empty-after-trim or empty-after-NUL-scrub (`query.rs:180-190`) |

Projection and ordering are fixed, not caller-selectable:
`SELECT id, kind, pubkey, channel_id, EXTRACT(EPOCH FROM created_at)::bigint AS created_at_s, ts_rank_cd(...) AS rank`
(`query.rs:234-236`) and
`ORDER BY rank DESC, created_at DESC, id LIMIT $n OFFSET $n`
(`query.rs:295-298`).

---

#### Error surface

Every failure path returns `SearchError::Db` (`error.rs:8`):

| Failure | Mechanism | Line |
|---|---|---|
| Query execution / connection error | `?` on `fetch_all` → `From<sqlx::Error>` | `query.rs:300` |
| Column type mismatch / missing column | `?` on `row.try_get` | `query.rs:304-305`, `314-318` |
| `id` not exactly 32 bytes | mapped to `sqlx::Error::Decode("event id column is N bytes, expected 32")` | `query.rs:306-308` |
| `pubkey` not exactly 32 bytes | mapped to `sqlx::Error::Decode("pubkey column is N bytes, expected 32")` | `query.rs:309-311` |

No panics in `src/`: no `unwrap()`, `expect()`, or indexing that can panic appears
in `lib.rs`, `query.rs`, or `error.rs` (verified by reading all three files).


## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: API Surface

Crate root declares 5 public modules — `action`, `entry`, `error`, `hash`, `service`
(`crates/buzz-audit/src/lib.rs:21-29`) — with re-exports at `lib.rs:31-35`.
Lints: `#![deny(unsafe_code)]`, `#![warn(missing_docs)]` (`lib.rs:1-2`).

### Public functions and methods

| Symbol | Signature (file:line) | Operation | Tables touched | Return type |
|---|---|---|---|---|
| `AuditService::new` | `pub fn new(pool: PgPool) -> Self` (`service.rs:43-45`) | stores the pool; no I/O | none | `AuditService` |
| `AuditService::log` | `pub async fn log(&self, entry: NewAuditEntry) -> Result<AuditEntry, AuditError>` (`service.rs:53`) | acquire pooled conn → `pg_advisory_lock` → `log_inner` under `catch_unwind` → `pg_advisory_unlock` → resume panic if any | `audit_log` (via `log_inner`); advisory-lock functions are not table reads | `Result<AuditEntry, AuditError>` |
| `AuditService::verify_chain` | `pub async fn verify_chain(&self, community: CommunityId, from_seq: i64, to_seq: i64) -> Result<bool, AuditError>` (`service.rs:160-165`) | one `SELECT` of the range, then per-row link check + hash recompute | `audit_log` (read) | `Result<bool, AuditError>` |
| `AuditService::get_entries` | `pub async fn get_entries(&self, community: CommunityId, from_seq: i64, limit: i64) -> Result<Vec<AuditEntry>, AuditError>` (`service.rs:212-217`) | one `SELECT` with `LIMIT`, decode each row | `audit_log` (read) | `Result<Vec<AuditEntry>, AuditError>` |
| `compute_hash` | `pub fn compute_hash(entry: &AuditEntry) -> Result<[u8; 32], AuditError>` (`hash.rs:42`) | pure SHA-256 over the fixed pre-image | none | `Result<[u8; 32], AuditError>` |
| `to_storage_precision` | `pub fn to_storage_precision(created_at: DateTime<Utc>) -> DateTime<Utc>` (`hash.rs:22-24`) | `trunc_subsecs(6)` | none | `DateTime<Utc>` |
| `GENESIS_HASH` | `pub const GENESIS_HASH: [u8; 32]` (`hash.rs:9`) | constant | none | `[u8; 32]` |
| `AuditAction::as_str` | `pub fn as_str(&self) -> &'static str` (`hash-independent`, `action.rs:35-49`) | variant → stable string | none | `&'static str` |
| `impl Display for AuditAction` | `fmt(&self, f) -> fmt::Result` (`action.rs:66-70`) | writes `as_str()` | none | `fmt::Result` |
| `impl FromStr for AuditAction` | `fn from_str(s: &str) -> Result<Self, String>` (`action.rs:75-81`) | linear search over `ALL` | none | `Result<AuditAction, String>` |

Public **types**: `AuditEntry`, `NewAuditEntry` (`entry.rs:14`, `:52`), `AuditAction`
(`action.rs:8`), `AuditError` (`error.rs:12`), `AuditService` (`service.rs:37`).
`AuditService` has no `Clone`/`Debug` derive (`service.rs:37-39`).

### Private helpers

| Symbol | Signature (file:line) | Purpose |
|---|---|---|
| `AuditService::log_inner` | `async fn log_inner(&self, conn: &mut PoolConnection<Postgres>, entry: NewAuditEntry) -> Result<AuditEntry, AuditError>` (`service.rs:82-86`) | the transactional head-read + insert |
| `row_to_audit_entry` | `fn row_to_audit_entry(row: &sqlx::postgres::PgRow) -> Result<AuditEntry, AuditError>` (`service.rs:238-256`) | row → `AuditEntry`, parsing `action` |
| `canonical_json` | `fn canonical_json(value: &serde_json::Value) -> Result<String, serde_json::Error>` (`hash.rs:80-116`) | recursive sorted-key JSON serialization |
| `log_timestamp` | `fn log_timestamp() -> DateTime<Utc>` (`service.rs:21-23`) | `to_storage_precision(Utc::now())` |
| `AUDIT_LOCK_NAMESPACE` | `const AUDIT_LOCK_NAMESPACE: &str = "buzz_audit:"` (`service.rs:29`) | advisory-lock key prefix (private) |

### Exact SQL issued

| Call site | SQL | Binds |
|---|---|---|
| `service.rs:59-62` | `SELECT pg_advisory_lock(hashtextextended($1, 0))` | `$1` = `"buzz_audit:{community_id}"` |
| `service.rs:71-74` | `SELECT pg_advisory_unlock(hashtextextended($1, 0))` | same key; result discarded via `let _ =` |
| `service.rs:94-101` | `SELECT seq, hash FROM audit_log WHERE community_id = $1 ORDER BY seq DESC LIMIT 1` | `$1` = raw `Uuid`; `fetch_optional` inside the tx |
| `service.rs:130-147` | `INSERT INTO audit_log (community_id, seq, hash, prev_hash, action, actor_pubkey, object_id, detail, created_at) VALUES ($1..$9)` | 9 binds in column order (`service.rs:137-145`) |
| `service.rs:166-179` | `SELECT community_id, seq, hash, prev_hash, action, actor_pubkey, object_id, detail, created_at FROM audit_log WHERE community_id = $1 AND seq BETWEEN $2 AND $3 ORDER BY seq ASC` | community, `from_seq`, `to_seq`; run on `&self.pool` (no tx) |
| `service.rs:218-232` | same projection `... WHERE community_id = $1 AND seq >= $2 ORDER BY seq ASC LIMIT $3` | community, `from_seq`, `limit`; run on `&self.pool` |

All queries are `sqlx::query(...)` with bind parameters (no string interpolation of
values; the only formatted string is the advisory-lock key, which is bound as `$1`
at `service.rs:58-60`). `sqlx::query` (untyped) is used throughout — no
compile-time-checked `query!` macros, so column names are resolved at runtime via
`Row::get` (`service.rs:105-106`, `246-254`).

### Transaction / connection semantics

- `log` takes one pooled connection (`service.rs:54`) and holds it for the lock,
  transaction, and unlock — required because Postgres advisory locks are
  session-scoped (`service.rs:49-51`).
- `log_inner` opens the transaction with `conn.begin()` (`service.rs:87`, via
  `sqlx::Acquire` imported at `service.rs:3`) and commits at `service.rs:149`.
  Head read and insert are both inside that transaction (`service.rs:100`, `:146`).
- `verify_chain` / `get_entries` are non-transactional single statements against the
  pool (`service.rs:178`, `:231`) and take **no** advisory lock.

### Return-value contracts

- `log` returns the fully materialised `AuditEntry` including assigned `seq`,
  `prev_hash`, computed `hash`, and `created_at` (`service.rs:151`).
- `verify_chain` returns `Ok(false)` when the range is empty (`service.rs:181-183`) —
  an empty range is *not* an error; `Ok(true)` when the whole segment is internally
  consistent (`service.rs:205`); `Err(ChainViolation)` / `Err(HashMismatch)` on the
  first offending `seq` (`service.rs:193`, `:199`).
- `get_entries` collects `Result` per row, so a single unparsable `action` fails the
  whole call (`service.rs:234`).

### Observed consumers (outside this crate)

| Consumer | Call | Notes |
|---|---|---|
| `crates/buzz-relay/src/main.rs:321-334` | `AuditService::new(audit_pool)` behind `config.audit_enabled` | dedicated pool, `max_connections(5)`, `min_connections(1)` |
| `crates/buzz-relay/src/state.rs:654`, `:1199-1207` | `audit.log(entry)` from a background worker fed by an `mpsc` channel (capacity 1000) | errors counted (`buzz_audit_log_errors_total`) and logged, never retried |
| `crates/buzz-relay/src/handlers/event.rs:563-600` | builds `NewAuditEntry { action: EventCreated, … }` | `object_id` = event id hex; `detail` = `{event_kind, channel_id}` |
| `crates/buzz-relay/src/api/media.rs:422-442` | builds `NewAuditEntry { action: MediaUploaded, … }` | `object_id` = media sha256 |
| `crates/buzz-admin/Cargo.toml:20` | declares `buzz-audit` dependency | no `audit`/`AuditService` reference found in `crates/buzz-admin/src` (grep for `audit` returned nothing) |

`verify_chain` and `get_entries` have **no production caller** in the repo — the only
call sites are this crate's `#[ignore]` tests (`service.rs:368`, `:417`, `:427`, `:468`,
`:508`, `:523`) and relay tests (`crates/buzz-relay/src/handlers/event.rs:1906-1952`).


## Module: buzz-media (`crates/buzz-media`)

### Aspect: API Surface

Library crate only — "no Axum dependency for handlers. Axum handlers live in `buzz-relay`" (`crates/buzz-media/src/lib.rs:3`). Axum *is* a dependency, but only for `StatusCode`/`IntoResponse` (`crates/buzz-media/src/error.rs:3-4`) and `HeaderName` validation (`crates/buzz-media/src/config.rs:118`).

Modules, all `pub`: `auth`, `bucket_index`, `config`, `error`, `storage`, `thumbnail`, `types`, `upload`, `upload_record`, `validation` (`crates/buzz-media/src/lib.rs:5-15`). Re-exports at `crates/buzz-media/src/lib.rs:17-28`.

---

### 1. Upload pipeline (`upload.rs`)

| Fn | Operation | Storage backend calls | Return |
|---|---|---|---|
| `process_upload(storage, config, ctx, auth_event, body: Bytes, attribution)` (`crates/buzz-media/src/upload.rs:207-236`) | image path: sniff+validate → sha256 → Blossom auth (600 s window) → idempotency → put blob → thumbnail+blurhash → optional record → put sidecar | `head` ×2, `get_sidecar`, `put` (blob), `put` (thumb), `put` (record), `put_sidecar` | `Result<BlobDescriptor, MediaError>` |
| `process_file_upload(... body: Bytes, attribution)` (`crates/buzz-media/src/upload.rs:245-274`) | generic-file path: deny-list validate → sha256 → auth (600 s) → idempotency → put blob → minimal sidecar | same minus thumbnail | `Result<BlobDescriptor, MediaError>` |
| `process_video_upload(... body_stream, content_length, attribution)` (`crates/buzz-media/src/upload.rs:292-511`) | streaming path: stream→tempfile + incremental sha256 → ISO-BMFF check → auth (3600 s) → full MP4 validation → idempotency → `put_file` → sidecar with `duration_secs` | `head` ×2, `get_sidecar`, `put_file`, `put` (record), `put_sidecar` | `Result<BlobDescriptor, MediaError>` |
| `process_buffered_upload` (private, `crates/buzz-media/src/upload.rs:54-192`) | shared skeleton for the two buffered paths, parameterised by a `validate` closure and a `prepare_metadata` future | — | `Result<BlobDescriptor, MediaError>` |
| `prepare_image_metadata` (private, `crates/buzz-media/src/upload.rs:513-537`) | `spawn_blocking` thumbnail/blurhash then thumb PUT | `put` | `Result<BlobMeta, MediaError>` |
| `build_descriptor` (private, `crates/buzz-media/src/upload.rs:539-560`) | pure descriptor assembly; empty strings → `None` | none | `BlobDescriptor` |

`process_video_upload`'s stream parameter type: `impl futures_core::Stream<Item = Result<Bytes, axum::Error>> + Send + 'static` (`crates/buzz-media/src/upload.rs:298`).

---

### 2. Storage backend — `MediaStorage` (`storage.rs`)

Single backend: **S3-compatible object storage via `rust-s3`**, always path-style (`crates/buzz-media/src/storage.rs:66-68`). There is **no trait**, and no local-filesystem or in-memory backend implementation in this crate. Test doubles exist only for the sweep fold, via a closure (`crates/buzz-media/src/bucket_index.rs:377-383`).

| Method | Operation | `rust-s3` call | Return |
|---|---|---|---|
| `new(&MediaConfig)` (`crates/buzz-media/src/storage.rs:34-70`) | build client; static creds vs AWS chain | `Credentials::new` / `Credentials::default`, `Bucket::new(..).with_path_style()` | `Result<Self, MediaError>` |
| `put(key, bytes, content_type)` (`crates/buzz-media/src/storage.rs:73-79`) | write from slice | `put_object_with_content_type` | `Result<(), MediaError>` |
| `put_file(key, path, content_type)` (`crates/buzz-media/src/storage.rs:85-103`) | streamed write from disk, 8 MiB `BufReader` | `put_object_stream_with_content_type` | `Result<(), MediaError>` |
| `get(key)` (`crates/buzz-media/src/storage.rs:105-111`) | full read into `Vec<u8>` | `get_object`; 404 → `NotFound` | `Result<Vec<u8>, MediaError>` |
| `get_range(key, start, end)` (`crates/buzz-media/src/storage.rs:118-124`) | inclusive-range read (HTTP 206 support) | `get_object_range` | `Result<Vec<u8>, MediaError>` |
| `get_stream(key)` (`crates/buzz-media/src/storage.rs:131-146`) | chunked read, never buffered | `get_object_stream`; `status_code == 404` → `NotFound` | `Result<ByteStream, MediaError>` |
| `head(key)` (`crates/buzz-media/src/storage.rs:149-155`) | existence check | `head_object`; 404 → `Ok(false)` | `Result<bool, MediaError>` |
| `delete(key)` (`crates/buzz-media/src/storage.rs:158-164`) | delete object | `delete_object` | `Result<(), MediaError>` |
| `head_with_metadata(key)` (`crates/buzz-media/src/storage.rs:167-175`) | HEAD → size only | `head_object` | `Result<Option<BlobHeadMeta>, MediaError>` |
| `sidecar_key(community, sha256)` (`crates/buzz-media/src/storage.rs:183-185`) | key builder (assoc fn) | none | `String` |
| `ctx_sidecar_key(ctx, sha256)` (`crates/buzz-media/src/storage.rs:188-190`) | key builder from tenant | none | `String` |
| `get_sidecar(ctx, sha256)` (`crates/buzz-media/src/storage.rs:193-202`) | read + deserialize `BlobMeta` | `get_object` | `Result<BlobMeta, MediaError>` |
| `put_sidecar(ctx, sha256, meta)` (`crates/buzz-media/src/storage.rs:210-221`) | serialize + write sidecar | `put` → `put_object_with_content_type` | `Result<(), MediaError>` |
| `read_sidecar_mime(ctx, sha256_ext)` (`crates/buzz-media/src/storage.rs:226-233`) | MIME-only convenience; absence and failure both → `None` | `get_object` | `Option<String>` |
| `list_page(continuation_token, max_keys)` (`crates/buzz-media/src/storage.rs:242-265`) | one listing page, converted to storage-agnostic `Page` | `list_page` (manual pagination, not auto-paginating `list`) | `Result<Page, MediaError>` |

---

### 3. Validation (`validation.rs`)

| Fn | Operation | Return |
|---|---|---|
| `validate_content(bytes, config)` (`crates/buzz-media/src/validation.rs:238-274`) | image path: sniff → allowlist → size cap → metadata-free structural check → pixel-count cap | `Result<String, MediaError>` (MIME) |
| `validate_file_content(bytes, config)` (`crates/buzz-media/src/validation.rs:159-209`) | generic path: size cap → ISO-BMFF reject → sniff → reject image/video/audio → deny-list → ext mapping | `Result<(String, String), MediaError>` (mime, ext) |
| `validate_video_file(path, config)` (`crates/buzz-media/src/validation.rs:289-395`) | on-disk MP4: moov-before-mdat, box allowlist, brand, single H.264 track, ≤1 AAC track, duration, resolution | `Result<VideoMeta, MediaError>` |
| `looks_like_iso_bmff(bytes)` (`crates/buzz-media/src/validation.rs:48-50`) | structural `ftyp` box detection | `bool` |
| `looks_like_mp4_iso_bmff(bytes)` — `pub(crate)` (`crates/buzz-media/src/validation.rs:52-62`) | `ftyp` + MP4 brand match (major or compatible) | `bool` |
| `serve_inline(mime)` (`crates/buzz-media/src/validation.rs:216-218`) | inline-vs-attachment policy: `image/*` or `video/*` | `bool` |
| `mime_to_ext(mime)` (`crates/buzz-media/src/validation.rs:930-939`) | MIME → extension, fallback `"bin"` | `&'static str` |

Private validators: `iso_bmff_ftyp_payload` (`:22`), `file_mime_to_ext` (`:99`), `check_moov_before_mdat` (`:408`), `validate_image_metadata_free` (`:492`), `validate_jpeg_metadata_free` (`:502`), `is_snapshot_text_chunk` (`:584`), `validate_png_metadata_free` (`:592`), `validate_webp_metadata_free` (`:659`), `validate_gif_metadata_free` (`:734`), `validate_mp4_metadata_free` (`:831`).

---

### 4. Auth (`auth.rs`)

| Fn | Operation | Return |
|---|---|---|
| `verify_blossom_auth_event_for_verb(event, verb, server_domain, max_age_secs)` (`crates/buzz-media/src/auth.rs:31-145`) | signature, kind 24242, non-empty content, `t` verb, `expiration` future, `created_at` window, `server` tag match | `Result<(), MediaError>` |
| `verify_blossom_auth_event(event, server_domain, max_age_secs)` (`crates/buzz-media/src/auth.rs:147-152`) | upload-shaped wrapper over the above | `Result<(), MediaError>` |
| `verify_blossom_upload_auth(event, sha256, server_domain, max_age_secs)` (`crates/buzz-media/src/auth.rs:175-198`) | above + at least one `x` tag == `sha256` | `Result<(), MediaError>` |
| `verify_blossom_get_auth(event, sha256, server_domain, max_age_secs)` (`crates/buzz-media/src/auth.rs:207-236`) | verb `get` + (`x` match OR `server` match), else `InsufficientScope` | `Result<(), MediaError>` |
| `normalize_server_host` (private, `crates/buzz-media/src/auth.rs:163-170`) | strip scheme/path → `buzz_core::tenant::normalize_host` | `String` |

---

### 5. Thumbnails (`thumbnail.rs`)

| Fn | Operation | Return |
|---|---|---|
| `generate_image_metadata_sync(config, sha256, bytes, mime, ext)` (`crates/buzz-media/src/thumbnail.rs:15-50`) | sync/CPU-bound: full decode, 320px thumbnail, JPEG encode, 4×3 blurhash; non-`image/*` → `(BlobMeta::default(), None)` | `Result<(BlobMeta, Option<Vec<u8>>), MediaError>` |

---

### 6. Upload records (`upload_record.rs`)

| Fn | Operation | Storage call | Return |
|---|---|---|---|
| `record_upload_event(storage, ctx, uploader, attribution, facts)` (`crates/buzz-media/src/upload_record.rs:139-178`) | build `UploadRecord`, ULID id, write JSON | `put` | `Result<(), MediaError>` |
| `upload_record_key(ctx, sha256, event_id)` (`crates/buzz-media/src/upload_record.rs:181-183`) | key builder | none | `String` |
| `parse_public_ip(raw)` (`crates/buzz-media/src/upload_record.rs:191-194`) | parse + public-range filter (fail-empty) | none | `Option<IpAddr>` |
| `parse_port(raw)` (`crates/buzz-media/src/upload_record.rs:197-199`) | non-zero u16 parse | none | `Option<u16>` |
| `is_public_ip` (private, `crates/buzz-media/src/upload_record.rs:207-256`) | explicit reserved-range enumeration (v4 + v6) | none | `bool` |
| `UPLOAD_RECORD_VERSION` const = 1 (`crates/buzz-media/src/upload_record.rs:52`) | — | — | `u32` |

---

### 7. Bucket sweep (`bucket_index.rs`)

| Fn / method | Operation | Return |
|---|---|---|
| `classify_key(key)` (`crates/buzz-media/src/bucket_index.rs:54-72`) | thumb → blob → sidecar → auxiliary → unknown | `KeyClass` |
| `BucketAggregate::fold(&mut self, key, size)` (`crates/buzz-media/src/bucket_index.rs:250-274`) | incremental per-sha/per-binding accumulation | `()` |
| `BucketAggregate::finish(self)` (`crates/buzz-media/src/bucket_index.rs:277-337`) | orphan/multi-variant/per-community computation | `BucketSnapshot` |
| `fold_bucket_listing(cap, fetch_page)` (`crates/buzz-media/src/bucket_index.rs:377-411`) | paginate with cap check before folding each page | `Result<BucketSnapshot, SweepError>` |

`fetch_page: FnMut(Option<String>) -> Future<Output = Result<Page, MediaError>>` — the injection point production code fills with `MediaStorage::list_page` (`crates/buzz-media/src/bucket_index.rs:11-14`).

---

### 8. Config + error surface

| Item | Signature | file:line |
|---|---|---|
| `MediaConfig::validate(&self)` | `Result<(), String>` | `crates/buzz-media/src/config.rs:66-122` |
| `impl From<image::ImageError> for MediaError` | → `InvalidImage` | `crates/buzz-media/src/error.rs:88-92` |
| `impl From<s3::error::S3Error> for MediaError` | → `StorageError(String)` | `crates/buzz-media/src/error.rs:94-98` |
| `impl From<serde_json::Error> for MediaError` | → `StorageError(String)` | `crates/buzz-media/src/error.rs:100-104` |
| `impl IntoResponse for MediaError` | HTTP status mapping + JSON `{"error": …}` | `crates/buzz-media/src/error.rs:106-160` |

---

### 9. Consumers (boundary, outside this crate)

Relay routes that call into this crate: `PUT /upload`, `PUT /media/upload`, `GET|HEAD /media/{sha256_ext}` (`crates/buzz-relay/src/router.rs:38-45`). The relay selects the pipeline by sniffing the first 4096 bytes and calling `looks_like_iso_bmff` (`crates/buzz-relay/src/api/media.rs:47-51`, `crates/buzz-relay/src/api/media.rs:334-399`), and applies `serve_inline` for `Content-Disposition` (`crates/buzz-relay/src/api/media.rs:663`).


## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: API Surface

Crate lints: `#![deny(unsafe_code)]`, `#![warn(missing_docs)]` (`lib.rs:1-2`). Public modules: `action_sink`, `error`, `executor`, `schema` (`lib.rs:33-36`). Re-exports: `ActionSink`, `ActionSinkError`, `PartialProgress`, `WorkflowError`, `ExecutionResult`, `ActionDef`, `Step`, `TriggerDef`, `WorkflowDef` (`lib.rs:38-41`).

---

### `WorkflowEngine` (crate root)

| Fn | Signature | Purpose | Returns | Line |
|---|---|---|---|---|
| `new` | `fn new(db: Db, config: WorkflowConfig) -> Self` | Construct engine; semaphore permits = `max_concurrent.max(1)`; moka cache 10 000 entries / 10 s TTL | `Self` | `lib.rs:109-127` |
| `invalidate_channel_workflows` | `fn invalidate_channel_workflows(&self, community_id: CommunityId, channel_id: Uuid)` | Drop the cached enabled-workflow list for a channel (same-pod only) | `()` | `lib.rs:131-133` |
| `set_action_sink` | `fn set_action_sink(&self, sink: Arc<dyn ActionSink>)` | Late-init the side-effect sink; **panics if called twice** | `()` | `lib.rs:139-143` |
| `action_sink` | `pub(crate) fn action_sink(&self) -> Result<&dyn ActionSink, WorkflowError>` | Accessor; `InvalidDefinition` error if not wired | `Result<&dyn ActionSink, _>` | `lib.rs:148-156` |
| `parse_yaml` | `fn parse_yaml(yaml: &str) -> Result<(WorkflowDef, String), WorkflowError>` (associated, no `self`) | Parse + validate + canonical JSON | `(WorkflowDef, String)` | `lib.rs:163-165` |
| `finalize_run` | `async fn finalize_run(&self, community_id: CommunityId, run_id: Uuid, result: Result<ExecutionResult, (WorkflowError, PartialProgress)>, existing_trace: Option<Vec<serde_json::Value>>)` | Single mapping point from executor result → DB run status; approval-gated results are written as `Failed` | `()` (logs DB errors) | `lib.rs:175-263` |
| `on_event` | `async fn on_event(self: &Arc<Self>, community_id: CommunityId, event: &buzz_core::StoredEvent) -> Result<(), WorkflowError>` | Post-store hook: channel workflow lookup, trigger match, run creation, spawn execution | `Result<(), WorkflowError>` | `lib.rs:276-383` |
| `interval_prefilter_should_fire` | `fn interval_prefilter_should_fire(&self, community_id, workflow_id: Uuid, dur: &str, last: Option<DateTime<Utc>>, now: DateTime<Utc>) -> bool` (private) | Interval prefilter + cold-start anchor seed | `bool` | `lib.rs:401-403` |
| `run` | `async fn run(self: &Arc<Self>)` | Scheduler entry point — infinite 60 s tick loop for `schedule` triggers | never returns | `lib.rs:428-672` |

Free functions in `lib.rs`:

| Fn | Visibility | Purpose | Returns | Line |
|---|---|---|---|---|
| `build_trigger_context(event: &StoredEvent)` | `pub` | Map a stored event to `TriggerContext` (author prefers `actor` tag, reaction target from last 64-hex `e` tag) | `executor::TriggerContext` | `lib.rs:884-953` |
| `cron_fire_instant(expr, now, window_secs, workflow_id)` | private | Window-based cron match; returns the cron's *scheduled* instant | `Option<DateTime<Utc>>` | `lib.rs:688-706` |
| `interval_fire_instant(dur, now, workflow_id)` | private | Quantize `now` to `floor(now/interval)*interval` | `Option<DateTime<Utc>>` | `lib.rs:719-745` |
| `interval_should_fire(dur, last_fired, now, workflow_id)` | private | Elapsed-interval predicate; `None` ⇒ treated as `now` ⇒ false | `bool` | `lib.rs:753-774` |
| `interval_prefilter_should_fire(last_fired: &DashMap<…>, …)` | private | Testable prefilter + seed | `bool` | `lib.rs:784-797` |
| `should_fire_workflow(def, trigger_ctx, workflow_id)` | private, `async` | Emoji equality + `MessagePosted`/`DiffPosted` filter evaluation | `bool` | `lib.rs:806-882` |
| `trigger_matches_event(trigger, kind_u32)` | private | Kind→trigger-type match | `bool` | `lib.rs:955-964` |

---

### `schema` module

| Fn / method | Signature | Purpose | Returns | Line |
|---|---|---|---|---|
| `parse_yaml` | `pub fn parse_yaml(yaml: &str) -> Result<(WorkflowDef, String), WorkflowError>` | `serde_yaml` deserialize → `validate()` → `serde_json` canonical string | `(WorkflowDef, String)` | `schema.rs:262-268` |
| `WorkflowDef::validate` | `pub fn validate(&self) -> Result<(), WorkflowError>` | Name, steps, step-ID charset/uniqueness, schedule cron/interval invariants | `Result<(), WorkflowError>` | `schema.rs:152-229` |
| `validate_cron` | private `fn validate_cron(expr: &str) -> Result<(), WorkflowError>` | Normalize + parse via `cron::Schedule` | `Result<(), WorkflowError>` | `schema.rs:237-243` |
| `normalize_cron` | `pub(crate) fn normalize_cron(expr: &str) -> String` | 5→7 field / 6→7 field normalization | `String` | `schema.rs:250-257` |
| `default_true` | private | serde default for `enabled` | `bool` | `schema.rs:29-31` |

---

### `executor` module

| Fn | Signature | Purpose | Returns | Line |
|---|---|---|---|---|
| `TriggerContext::get_field` | `pub fn get_field(&self, name: &str) -> Option<&str>` | Named field lookup, falling back to `webhook_fields` | `Option<&str>` | `executor.rs:49-59` |
| `resolve_template` | `pub fn resolve_template(template: &str, trigger_ctx: &TriggerContext, step_outputs: &HashMap<String, JsonValue>) -> Result<String, WorkflowError>` | Single-pass `{{…}}` substitution with `| truncate(N)` / `| npub` filters | `String` | `executor.rs:70-123` |
| `resolve_variable` | private | `trigger.X` / `steps.ID.output.FIELD` lookup | `Option<String>` | `executor.rs:126-151` |
| `json_get_str` / `json_to_string` | private | JSON→string coercion for substitution | `Option<String>` / `String` | `executor.rs:154-174` |
| `apply_filter` | private | `truncate(N)`, `npub`, `truncate_pubkey`; unknown ⇒ `TemplateError` | `Result<String, WorkflowError>` | `executor.rs:176-201` |
| `build_eval_context` | `pub fn build_eval_context(trigger_ctx, step_outputs) -> Result<HashMapContext, WorkflowError>` | Register 4 custom fns + webhook vars + 6 trigger vars + step-output vars | `HashMapContext` | `executor.rs:224-316` |
| `json_value_to_eval` | private | `serde_json::Value` → `evalexpr::Value` | `evalexpr::Value` | `executor.rs:318-335` |
| `evaluate_condition` | `pub async fn evaluate_condition(expr: &str, trigger_ctx, step_outputs) -> Result<bool, WorkflowError>` | 4096-byte length gate, `spawn_blocking` + 100 ms timeout, `eval_boolean_with_context` | `bool` | `executor.rs:350-384` |
| `resolve_step_templates` | `pub fn resolve_step_templates(step: &Step, trigger_ctx, step_outputs) -> Result<ActionDef, WorkflowError>` | Returns a new `ActionDef` with templated fields substituted | `ActionDef` | `executor.rs:390-453` |
| `resolve_send_message_channel` | private | Destination resolution/validation for `send_message` | `Result<String, WorkflowError>` | `executor.rs:468-517` |
| `dispatch_action` | `pub async fn dispatch_action(step_id: &str, action: &ActionDef, engine: &WorkflowEngine, community_id: CommunityId, run_id: Uuid, trigger_ctx: &TriggerContext) -> Result<StepResult, WorkflowError>` | Per-action side effect | `StepResult` | `executor.rs:519-692` |
| `generate_approval_token` | private `fn(_run_id: Uuid, _step_id: &str) -> String` | `Uuid::new_v4().to_string()`; both args unused | `String` | `executor.rs:698-700` |
| `parse_duration_secs` | `pub(crate) fn parse_duration_secs(duration: &str) -> Result<u64, WorkflowError>` | `h`/`m`/`s`/bare-number parsing with checked multiply | `u64` | `executor.rs:705-735` |
| `check_ssrf` | private, `#[cfg(feature = "reqwest")]` `async fn(host: &str, port: u16) -> Result<IpAddr, WorkflowError>` | Resolve + reject private/reserved IPs; returns first IP for DNS pinning | `IpAddr` | `executor.rs:745-776` |
| `call_webhook_impl` | private, `#[cfg(feature = "reqwest")]` | Outbound HTTP with SSRF pinning, no redirects, 1 MiB cap | `JsonValue` | `executor.rs:781-869` |
| `shared_http_client` | private, `#[cfg(feature = "reqwest")]` | `LazyLock<reqwest::Client>`, 10 s timeout | `&'static Client` | `executor.rs:874-885` |
| `add_reaction_impl` | private, `#[cfg(feature = "reqwest")]` | `POST {BUZZ_RELAY_BASE_URL}/api/messages/{id}/reactions` | `JsonValue` | `executor.rs:888-933` |
| `execute_run` | `pub async fn execute_run(engine, community_id, run_id, def, trigger_ctx) -> Result<ExecutionResult, (WorkflowError, PartialProgress)>` | Acquire permit (`try_acquire`), set `Running`, run from step 0 | `ExecutionResult` | `executor.rs:970-1003` |
| `execute_from_step` | `pub async fn execute_from_step(engine, community_id, run_id, def, trigger_ctx, start_index: usize, initial_outputs: Option<HashMap<String, JsonValue>>) -> Result<ExecutionResult, (WorkflowError, PartialProgress)>` | Same but starts at `start_index`, preserves existing trace, seeds outputs | `ExecutionResult` | `executor.rs:1018-1075` |
| `execute_steps` | private `async fn` | Shared step loop (condition → templates → per-step `timeout` → dispatch → trace) | `ExecutionResult` | `executor.rs:1083-1217` |

---

### `action_sink` module

| Item | Signature | Purpose | Line |
|---|---|---|---|
| `ActionSink::send_message` | `fn send_message(&self, community_id: CommunityId, channel_id: &str, text: &str, author_pubkey: &str) -> Pin<Box<dyn Future<Output = Result<String, ActionSinkError>> + Send + '_>>` | Only sink operation; returns the created event id hex | `action_sink.rs:50-70` |
| `From<ActionSinkError> for WorkflowError` | maps to `WorkflowError::WebhookError(e.to_string())` | error bridging | `action_sink.rs:34-38` |

`error` module: `From<buzz_db::error::DbError> for WorkflowError` → `Database(String)` (`error.rs:62-66`).

---

### External call sites (who uses this API)

| API | Caller | Line |
|---|---|---|
| `WorkflowEngine::new` + `WorkflowConfig::default` | relay startup and many relay test fixtures | `crates/buzz-relay/src/main.rs:389-390`, `crates/buzz-relay/src/state.rs:1274-1276` |
| `set_action_sink` | relay startup | `crates/buzz-relay/src/main.rs:595` |
| `run()` (scheduler) | spawned after the sink is wired | `crates/buzz-relay/src/main.rs:597-599` |
| `on_event` | relay post-store fan-out hook | `crates/buzz-relay/src/handlers/event.rs:545` |
| `WorkflowEngine::parse_yaml` | workflow upsert command handler | `crates/buzz-relay/src/handlers/command_executor.rs:684` |
| `invalidate_channel_workflows` | workflow upsert + NIP-09 deletion | `crates/buzz-relay/src/handlers/command_executor.rs:787`, `crates/buzz-relay/src/handlers/side_effects.rs:2018`, `:2039` |
| `executor::execute_from_step` + `finalize_run` | manual trigger, inbound webhook, approval resume | `crates/buzz-relay/src/handlers/command_executor.rs:926`, `:1314`, `crates/buzz-relay/src/api/bridge.rs:1908` |
| `ActionSink` impl | `RelayActionSink` | `crates/buzz-relay/src/workflow_sink.rs:13`, `:159` |

`execute_run` has no callers outside this crate (only `lib.rs:373` and `lib.rs:651`); `build_trigger_context`, `resolve_template`, `resolve_step_templates`, `build_eval_context`, `evaluate_condition`, `dispatch_action` have no callers outside this crate.


## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: API Surface

Three independent listeners are bound at startup (`main.rs:1113-1224`), plus one Prometheus listener bound inside `metrics::install` (`metrics.rs:73-74`):

| Listener | Bind | Router | Bound at | Middleware |
|----------|------|--------|----------|------------|
| App TCP | `config.bind_addr` (default `0.0.0.0:3000`) | `build_router` (`router.rs:32`) | `main.rs:1157` | metrics → trace → CORS |
| App UDS (optional, unix only) | `config.uds_path` | same router, `.into_make_service()` (**no `ConnectInfo`**) | `main.rs:1178-1187` | same |
| Health TCP | hard-coded `0.0.0.0:config.health_port` (default `8080`) | `build_health_router` (`router.rs:225`) | `main.rs:1116` | **none** |
| Metrics TCP | hard-coded `0.0.0.0:config.metrics_port` (default `9102`) | `PrometheusBuilder` internal | `metrics.rs:73-74`, spawned `metrics.rs:146` | none |

`router.rs` contains **34** `.route(...)` registrations: 33 production + 1 in `#[cfg(test)]` (`router.rs:476`).

---

#### 1. Complete route inventory

##### 1a. App router — `api_router` sub-router (`router.rs:56-131`, body limit 1 MB at `router.rs:129`)

| Method | Path | Handler | Reg. line | Auth enforced |
|--------|------|---------|-----------|---------------|
| GET | `/` | `nip11_or_ws_handler` (`router.rs:235`) | `router.rs:63` | **none** for NIP-11 branch; WS branch requires host→community bind (`router.rs:262`) then NIP-42 inside the socket |
| GET | `/info` | `nip11::relay_info_handler` (`nip11.rs:172`) | `router.rs:64` | **none** |
| GET | `/.well-known/nostr.json` | `api::nip05::nostr_nip05` | `router.rs:65` | **none** (NIP-05 is public by spec) |
| GET | `/health` | `health_handler` (`router.rs:295`) | `router.rs:67` | none |
| GET | `/_liveness` | `liveness_handler` (`router.rs:299`) | `router.rs:68` | none |
| GET | `/_readiness` | `readiness_handler` (`router.rs:304`) | `router.rs:69` | none |
| POST | `/events` | `api::bridge::submit_event` | `router.rs:71` | NIP-98 (`api/bridge.rs:111`) |
| POST | `/query` | `api::bridge::query_events` | `router.rs:72` | NIP-98 |
| POST | `/count` | `api::bridge::count_events` | `router.rs:73` | NIP-98 |
| GET | `/operator/communities` | `api::operator::list_owned_communities` | `router.rs:74-77` | NIP-98 + `relay_operator_pubkeys` allowlist (`api/operator.rs:70,91`) |
| POST | `/operator/communities` | `api::operator::provision_community` | `router.rs:74-77` | same |
| POST | `/operator/communities/archive` | `api::operator::archive_community` | `router.rs:78-81` | same |
| POST | `/operator/communities/unarchive` | `api::operator::unarchive_community` | `router.rs:82-85` | same |
| GET | `/operator/communities/availability` | `api::operator::community_availability` | `router.rs:86-89` | same |
| POST | `/operator/communities/transfer` | `api::operator::transfer_community` | `router.rs:90-93` | same |
| POST | `/api/invites` | `api::invites::mint_invite` | `router.rs:95` | NIP-98 + owner/admin |
| GET | `/api/join-policy` | `api::invites::join_policy` | `router.rs:96` | **none** |
| GET | `/api/join-policy/terms` | `api::invites::join_policy_terms` | `router.rs:99-102` | **none** |
| GET | `/api/join-policy/privacy` | `api::invites::join_policy_privacy` | `router.rs:103-106` | **none** |
| POST | `/api/invites/accept-policy` | `api::invites::accept_policy` | `router.rs:107-110` | NIP-98 (`api/invites.rs:193`) |
| POST | `/api/invites/claim` | `api::invites::claim_invite` | `router.rs:111` | NIP-98, **membership-gate exempt** (`router.rs:94` comment) + per-pubkey claim limiter (`api/invites.rs:380`) |
| GET | `/moderation/reports` | `api::bridge::moderation_reports` | `router.rs:113` | NIP-98 + mod-authz |
| GET | `/moderation/audit` | `api::bridge::moderation_audit` | `router.rs:114` | NIP-98 + mod-authz |
| GET | `/moderation/restricted` | `api::bridge::moderation_restricted` | `router.rs:115-118` | NIP-98 + mod-authz |
| POST | `/hooks/{id}` | `api::bridge::workflow_webhook` | `router.rs:120` | webhook secret only, **no NIP-98** (`router.rs:119` comment) |
| POST | `/_mesh/demo/echo` | `api::mesh_demo::demo_echo` | `router.rs:123` | 404 unless `BUZZ_MESH=on` **and** `BUZZ_MESH_DEMO_ECHO=on` |
| GET | `/huddle/{channel_id}/audio` | `audio::handler::ws_audio_handler` | `router.rs:125-128` | in-handler; `huddle_audio_available` gate at `audio/handler.rs:357` |

##### 1b. App router — `media_router` (`router.rs:37-45`, body limit `max(max_image_bytes, max_video_bytes)` at `router.rs:33-36,44`)

| Method | Path | Handler | Reg. line | Auth |
|--------|------|---------|-----------|------|
| PUT | `/upload` | `api::media::upload_blob` | `router.rs:39` | Blossom kind:24242 |
| PUT | `/media/upload` | `api::media::upload_blob` (same fn) | `router.rs:40` | same |
| GET | `/media/{sha256_ext}` | `api::media::get_blob` | `router.rs:41-44` | **off by default** — gated by `require_media_get_auth` (`config.rs:197`, default `false`) |
| HEAD | `/media/{sha256_ext}` | `api::media::head_blob` | `router.rs:43` | same |

##### 1c. App router — `git_router` (`api/git/transport.rs:1756-1765`, body limit `git_max_pack_bytes`)

| Method | Path | Handler | Line |
|--------|------|---------|------|
| GET | `/git/{owner}/{repo}/info/refs` | `info_refs` | `api/git/transport.rs:1760` |
| POST | `/git/{owner}/{repo}/git-upload-pack` | `upload_pack` | `api/git/transport.rs:1761` |
| POST | `/git/{owner}/{repo}/git-receive-pack` | `receive_pack` | `api/git/transport.rs:1762` |

##### 1d. App router — `git_policy_router` (`api/git/mod.rs:60-66`, body limit 1 MB)

| Method | Path | Handler | Line | Auth |
|--------|------|---------|------|------|
| POST | `/internal/git/policy` | `policy::hook_policy_check` | `api/git/mod.rs:62` | HMAC + `require_localhost` middleware (`api/git/mod.rs:38-52`, applied `:64`) |

##### 1e. App router — `admin_router`, conditional (`router.rs:47-54`)

Present only when `config.admin.is_some()` (`router.rs:47`), i.e. `BUZZ_ADMIN_HOST` is set. Nested under `/api/admin/v1` (`router.rs:53`). Body limit 1 KB, `security_headers` middleware (`api/admin/mod.rs:38-39`).

| Method | Path | Handler | Line |
|--------|------|---------|------|
| GET | `/api/admin/v1/reports` | `reports` | `api/admin/mod.rs:30` |
| GET | `/api/admin/v1/reports/{id}` | `report_detail` | `api/admin/mod.rs:31` |
| GET | `/api/admin/v1/feedback` | `feedback` | `api/admin/mod.rs:32` |
| GET | `/api/admin/v1/feedback/{id}` | `feedback_detail` | `api/admin/mod.rs:33` |
| GET | `/api/admin/v1/feedback/{id}/attachments/{sha256}` | `feedback_attachment` | `api/admin/mod.rs:34-37` |

##### 1f. App router — SPA fallback service (`router.rs:145-183`)

Installed only when `admin_web_dir.is_some() || web_dir.is_some()` (`router.rs:147`). Dispatch order (`router.rs:153-180`):
1. Admin host check first (`router.rs:158` → `api::admin::is_admin_host`). If admin host: serve `/assets/*` from admin `ServeDir` (`router.rs:160-162`), serve index for `is_admin_spa_path` (`router.rs:163-165` / predicate `router.rs:194-200`: `/`, `/reports`, `/reports/*`, `/feedback`, `/feedback/*`), else **404** (`router.rs:167`) — admin host can never fall through to the public bundle.
2. Public bundle: `/assets/*` from `ServeDir` (`router.rs:171-173`), index for `should_serve_spa` (`router.rs:174-176` / `router.rs:206-208`) = `is_invite_landing_path` (`/invite/{code}` with exactly one non-empty segment, `router.rs:202-204`) **or** `serve_git_web_gui && is_git_web_gui_path` (`/`, `/repos`, `/repos/*`, `router.rs:210-212`).
3. Otherwise 404 (`router.rs:178`).

##### 1g. Health router (`build_health_router`, `router.rs:225-232`)

| Method | Path | Handler | Line | Auth |
|--------|------|---------|------|------|
| GET | `/_liveness` | `liveness_handler` | `router.rs:227` | **none** |
| GET | `/_readiness` | `readiness_handler` | `router.rs:228` | **none** |
| GET | `/_status` | `status_handler` (`router.rs:387`) | `router.rs:229` | **none** — leaks `CARGO_PKG_VERSION` + uptime |
| GET | `/_mesh` | `mesh_status_handler` (`router.rs:380`) | `router.rs:230` | **none** — leaks the full mesh peer table incl. `endpoint_addrs` (`crates/buzz-relay-mesh/src/status.rs:20`) |

##### 1h. Metrics listener

`GET /metrics` (Prometheus text format) on `0.0.0.0:metrics_port`, served by `metrics_exporter_prometheus` (`metrics.rs:71-146`). Not registered in any axum router; not authenticated.

##### 1i. Unregistered route that production code calls

`crates/buzz-workflow/src/executor.rs:892` builds `POST {BUZZ_RELAY_BASE_URL}/api/messages/{message_id}/reactions` for the `add_reaction` workflow action. **No such route exists** in `router.rs:37-131`, `api/git/*`, or `api/admin/mod.rs`. The request lands on the SPA fallback (`router.rs:178` → 404) or a bare axum 404. Verified: the only two mentions of `api/messages` in the whole workspace are `crates/buzz-workflow/src/executor.rs:886` (doc) and `:892` (URL construction). Confirmed defect.

---

#### 2. Router-level middleware stack

Applied over the **merged** router in this order (`router.rs:185-191`, outermost last):

| Order | Layer | Line | Scope |
|-------|-------|------|-------|
| 1 (innermost) | per-sub-router `RequestBodyLimitLayer` | `router.rs:44` (media), `router.rs:129` (api, 1 MB), `api/git/transport.rs:1763`, `api/git/mod.rs:63`, `api/admin/mod.rs:39` | that sub-router only |
| 2 | `middleware::from_fn(track_metrics)` | `router.rs:188` | whole app router |
| 3 | `TraceLayer::new_for_http()` | `router.rs:189` | whole app router |
| 4 (outermost) | `build_cors_layer(&state.config.cors_origins)` | `router.rs:190`, impl `router.rs:409-432` | whole app router |

There is **no authentication middleware at any layer**. Every route's auth is enforced inside its handler. The two route-scoped middlewares are `require_localhost` (git policy, `api/git/mod.rs:64`) and `security_headers` (admin, `api/admin/mod.rs:38`).

`track_metrics` (`metrics.rs:164-206`) emits `http_requests_total{code,caller,action}` and `http_request_latency_ms{code,caller,action}`, skipping `MatchedPath` starting `/_`, `/health`, `/metrics`, and all unmatched paths (`metrics.rs:169-179`). `caller` comes from `x-envoy-downstream-service-cluster` with a `len<=64` + `[A-Za-z0-9_-]` filter (`metrics.rs:187-196`). The `p == "/metrics"` arm (`metrics.rs:170`) is unreachable — `/metrics` is never an app-router `MatchedPath`.

---

#### 3. `nip11_or_ws_handler` content negotiation (`router.rs:235-292`)

Decision order:

1. Read `ConnectInfo<SocketAddr>` or fall back to `0.0.0.0:0` (`router.rs:236-240`) — the UDS listener always hits the fallback because `main.rs:1179` uses `.into_make_service()`.
2. Read `accept` (`router.rs:242-245`) and raw `Host` (`router.rs:247-250`).
3. **Admin host short-circuit** (`router.rs:255-273`): if `is_admin_host`, require `text/html` else 404; serve admin `index.html` or 404. Admin host never reaches NIP-11 or WS.
4. `accept` contains `application/nostr+json` → `Json(nip11_document(...))` (`router.rs:275-277`). **Served before any tenant binding**, deliberately fail-open (`router.rs:279-286`).
5. Bind community from host (`router.rs:288-300`). Failure → `404 "relay: no community is configured for this host"` — a single generic message for both unmapped and lookup error (`router.rs:292-297`).
6. `WebSocketUpgrade::from_request` (`router.rs:303`):
   - Ok: if `shutting_down` → `503 "relay restarting"` (`router.rs:311-313`); else apply `limit_relay_websocket` (`router.rs:315` → `router.rs:324-332`, sets both `max_message_size` and `max_frame_size` to `config.max_frame_bytes`) and `on_upgrade(handle_connection)`.
   - Err: if `serve_git_web_gui && web_dir` and `accept` has `text/html`, serve `index.html` (`router.rs:320-330`); else fall back to the NIP-11 JSON document (`router.rs:333`).

Note step 4/6-Err: NIP-11 is reachable on `GET /` with **any** `Accept`, including none.

---

#### 4. NIP-11 document fields

`RelayInfo` (`nip11.rs:25-59`) as produced by `RelayInfo::build` (`nip11.rs:136-170`):

| Field | Line | Value | Conditional |
|-------|------|-------|-------------|
| `name` | `nip11.rs:27,152` | `"Buzz Relay"` | always (hard-coded) |
| `description` | `nip11.rs:29,153` | `"Buzz — private team communication relay"` | always (hard-coded) |
| `icon` | `nip11.rs:33,154` | `communities.icon` for the request host | omitted when `None`/empty (`skip_serializing_if`, `nip11.rs:32`); sourced `nip11.rs:277-286` |
| `pubkey` | `nip11.rs:35,155` | **always `None`** | hard-coded `None` |
| `contact` | `nip11.rs:37,156` | **always `None`** | hard-coded `None` |
| `supported_nips` | `nip11.rs:39,157` | `[1,2,10,11,16,17,23,25,29,33,38,42,50,56]` (`nip11.rs:15`) `+ [43]` when `advertise_nip43` (`nip11.rs:148-151`) | |
| `supported_extensions` | `nip11.rs:42,158` | `["nip-er"]`, `+ "nip-pl"` when push configured (`nip11.rs:254-256`) | omitted when `None` |
| `push` | `nip11.rs:45,159` | `push_descriptor(...)` object (`nip11.rs:183-233`) | omitted unless `push_gateway_delivery_url.is_some()` **and** the host binds to a community (`nip11.rs:187-188`) |
| `software` | `nip11.rs:47,160` | `"https://github.com/block/buzz"` | always |
| `version` | `nip11.rs:49,161` | `env!("CARGO_PKG_VERSION")` = `0.2.0` (`Cargo.toml:7`) | always |
| `limitation` | `nip11.rs:51,162` | `relay_limitation(config.max_frame_bytes)` | always `Some` |
| `pairing_relay_url` | `nip11.rs:54,163` | `config.pairing_relay_url` | omitted when `None` |
| `relay_self` (JSON key `self`) | `nip11.rs:58,164` | `relay_keypair.public_key().to_hex()` | only when `relay_private_key.is_some()` (`nip11.rs:302-303`) |

`RelayLimitation` (`nip11.rs:62-92`) from `relay_limitation` (`nip11.rs:96-119`):

| Field | Line | Value | Enforced? |
|-------|------|-------|-----------|
| `max_message_length` | `nip11.rs:103` | `config.max_frame_bytes` (default 524,288) | yes — `router.rs:330-331` (parser) + `recv_loop` |
| `max_subscriptions` | `nip11.rs:104` | `1024` hard-coded | yes — but via a **separate** hard-coded `MAX_SUBSCRIPTIONS = 1024` at `handlers/req.rs:26`, checked `handlers/req.rs:66` |
| `max_filters` | `nip11.rs:105` | `10` hard-coded | yes — separate const `protocol.rs:14`, checked `protocol.rs:93/131` |
| `max_limit` | `nip11.rs:106` | `10_000` hard-coded | **NO** — `buzz-db/src/event.rs:347` clamps to `max_limit.unwrap_or(1000)`; ordinary REQ does not raise it. 10× over-advertised |
| `max_subid_length` | `nip11.rs:107` | `256` | yes — separate const `protocol.rs:11`, checked `protocol.rs:86/125` |
| `min_pow_difficulty` | `nip11.rs:108` | `None` | n/a |
| `auth_required` | `nip11.rs:109` | `true` hard-coded | yes for REQ/EVENT/COUNT (asserted `nip11.rs:441-447`) |
| `payment_required` | `nip11.rs:110` | `false` | n/a |
| `restricted_writes` | `nip11.rs:111` | `true` hard-coded | advertised even on fully open relays (`require_relay_membership=false`, `pubkey_allowlist_enabled=false`) |
| `due_delivery_mode` | `nip11.rs:112` | `Some("push")` hard-coded | advertised **unconditionally**, even when `push_gateway_delivery_url` is `None` and no push worker was spawned (`main.rs:686-691`) |
| `max_not_before_delta` | `nip11.rs:113` | `SPROUT_MAX_NOT_BEFORE_DELTA` env, default `31_536_000` (`nip11.rs:97-100`) | read *per request* from env, not via `Config` |

##### NIP-11 advertisement gaps (verified)
- **NIP-45 (COUNT) is implemented but not advertised.** `ClientMessage::Count` (`protocol.rs:36`), `RelayMessage::count` (`protocol.rs:211`), `handlers/count.rs:285`, and `POST /count` (`router.rs:73`) all exist; `45` is absent from `SUPPORTED_NIPS` (`nip11.rs:15`).
- **NIP-98 (HTTP auth) is required on 12 routes but not advertised** (`api/bridge.rs:111`, `api/invites.rs:193`, `api/operator.rs`); `98` absent from `nip11.rs:15`.
- `due_delivery_mode: "push"` is advertised even with push disabled (see table above).

##### Per-request DB cost of the unauthenticated NIP-11 path
`nip11_document` (`nip11.rs:235-263`) performs, per request, with no caching and no rate limit:
1. `workspace_icon_for_host` → `bind_community` (`nip11.rs:278`) = 1 host lookup.
2. `get_community_icon` (`nip11.rs:283`) = 1 query.
3. A **second** `bind_community` (`nip11.rs:246`) when `push_gateway_delivery_url.is_some()` — which is the **default** (`config.rs:752-758` falls back to `DEFAULT_PUSH_GATEWAY_DELIVERY_URL`, `config.rs:332`).

So `GET /` and `GET /info` each cost up to 3 Postgres round-trips for an unauthenticated caller.

---

#### 5. Public items by file

| File | Public items | Notes |
|------|--------------|-------|
| `lib.rs` | 21 `pub mod` + 3 `pub use` (`lib.rs:53-55`) | all 3 re-exports unused (no external lib consumer; `main.rs:17-24` uses module paths) |
| `config.rs` | `DEFAULT_MAX_FRAME_BYTES:15`, `ConfigError:19`, `AdminConfig:29`, `JoinPolicyConfig:37`, `Config:51`, `Config::from_env:405` | all consumed |
| `error.rs` | `RelayError:8` (10 variants), `Result:49` | 9 of 10 variants dead; `Result` used only by `protocol.rs:6` |
| `protocol.rs` | `ClientMessage:23` (5 variants), `ClientMessage::parse:40`, `RelayMessage:176` + 7 fns | all consumed |
| `tenant.rs` | `HostResolver:31`, `BindError:53`, `bind_community:79`, `bind_deployment_community:130`, `relay_url_authority:139` (re-export), `impl HostResolver for Db:141` | `bind_deployment_community` ← `main.rs:562`; `relay_url_authority` ← `main.rs:239` |
| `telemetry.rs` | `service_resource:47`, `TracerInit:64`, `try_init_tracer:79` | all ← `main.rs:99-100`, `main.rs:1053` |
| `metrics.rs` | `install:71`, `track_metrics:164` | ← `main.rs:138`, `router.rs:188` |
| `nip11.rs` | `RelayInfo:25`, `RelayLimitation:62`, `RelayInfo::build:136`, `relay_info_handler:172`; `pub(crate)`: `SUPPORTED_NIPS:15`, `NIP_RELAY_MEMBERSHIP:21`, `nip11_document:235`, `nip11_facts:301` | `_RELAY_INFO_BUILD_STATIC_INPUT_FENCE` (`nip11.rs:329-335`) pins `build`'s exact signature as a compile-time conformance guard |
| `router.rs` | `build_router:32`, `build_health_router:225` | both ← `main.rs:939-940` |
| `state.rs` | see data-model §1/§5 | `bound_communities:111` has no external caller |
| `admission.rs` | `pub(crate)` only: `AdmissionError:12`, `check_principal:18`, `ws_admission_budget:39` | `check_principal` ← `api/bridge.rs:30`, `connection.rs:615/638`; `ws_admission_budget` ← `connection.rs:614` |

##### Name collision
Two distinct `AdmissionError` types coexist: `crate::admission::AdmissionError` (`admission.rs:12`, variants `Exceeded`/`Unavailable`) and `crate::audio::room::AdmissionError` (`audio/room.rs:83`, variants `Full`/`Ended`/`VersionMismatch`). Both are referenced in `handlers`/`audio` code paths (`connection.rs:657`, `audio/handler.rs:515`).


## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: API Surface

The WS surface is NIP-01 + NIP-42 + NIP-45, with NIP-29 scoping expressed through the `#h` tag rather than distinct message types. Entry point: `router.rs:301-315` (`WebSocketUpgrade::from_request` → `handle_connection`), reached only through `GET /` with a WebSocket upgrade after the host resolves to a community (`router.rs:286-300`).

---

#### 1. Inbound wire surface — exactly 5 `ClientMessage` variants

Parser: `protocol.rs:57-170`. Dispatcher: `connection.rs:489-585`.

| Wire form | `ClientMessage` | Parse | Dispatch | Executed | Metered by admission |
|---|---|---|---|---|---|
| `["EVENT", <event>]` | `Event(Event)` | `protocol.rs:66-76` | `connection.rs:510-537` | spawned task under `handler_semaphore` | **yes** — `WsEvents` + `Messages` |
| `["REQ", <sub_id>, <filter>…]` | `Req { sub_id, filters }` | `protocol.rs:77-110` | `connection.rs:538-559` | spawned task under `handler_semaphore` | **yes** — `WsEvents` |
| `["COUNT", <sub_id>, <filter>…]` | `Count { sub_id, filters }` | `protocol.rs:111-151` | `connection.rs:560-580` | spawned task under `handler_semaphore` | **yes** — `WsEvents` |
| `["CLOSE", <sub_id>]` | `Close(String)` | `protocol.rs:152-162` | `connection.rs:581-583` | **inline, awaited** in the recv loop | no (`connection.rs:599-602`) |
| `["AUTH", <event>]` | `Auth(Event)` | `protocol.rs:163-172` | `connection.rs:503-509` | **inline, awaited** in the recv loop | no (`connection.rs:599-602`) |

Any other first element → `RelayError::InvalidMessage("unknown message type: …")` (`protocol.rs:173-175`) → `["NOTICE","invalid message: …"]` (`connection.rs:493`).

Parse-time rejections (all become one `NOTICE`):

| Condition | Message fragment | Site |
|---|---|---|
| not valid JSON | `JSON parse error: …` | `protocol.rs:59-60` |
| not a JSON array | `expected JSON array` | `:62-64` |
| empty array | `empty array` | `:66-68` |
| element 0 not a string | `first element must be a string` | `:70-72` |
| `EVENT`/`AUTH` with `len < 2` | `EVENT requires event object` / `AUTH requires event object` | `:67-71`, `:164-168` |
| event body not deserializable | `invalid event: …` / `invalid auth event: …` | `:73-74`, `:169-170` |
| `REQ`/`COUNT`/`CLOSE` with `len < 2` | `REQ requires sub_id` etc. | `:78-82`, `:112-116`, `:153-157` |
| sub_id not a string | `REQ sub_id must be a string` etc. | `:84-88`, `:118-122`, `:159-163` |
| empty sub_id (REQ/COUNT only — **CLOSE allows empty**) | `must not be empty` | `:89-93`, `:123-127` |
| sub_id > 256 B | `… sub_id exceeds maximum length of 256 bytes` | `:94-98`, `:128-133` |
| > 10 filters | `… contains N filters, maximum is 10` | `:100-105`, `:135-140` |
| a filter not deserializable | `invalid filter: …` | `:106-112`, `:141-147` |

Frame-level handling in `recv_loop` (`connection.rs:407-487`):

| Frame | Behaviour |
|---|---|
| `Text` > `max_frame_bytes` | `["NOTICE","error: frame too large (N bytes, limit M)"]` then **disconnect** (`:421-434`) — note this is a hand-built JSON string, not `RelayMessage::notice` |
| `Binary` > limit | `["NOTICE","error: binary frame too large …"]` then disconnect (`:440-452`) |
| `Binary` valid UTF-8 | treated as text (`:457-459`) — documented as a non-NIP-01 extension (`:454-456`) |
| `Binary` invalid UTF-8 | **silently dropped**, no NOTICE (`:457`, no `else`) |
| `Ping` | Pong via `ctrl_tx.try_send`; a full ctrl channel is terminal (`:464-472`) |
| `Pong` | resets `missed_pongs` (`:461-463`) |
| `Close` / stream end / stream error | break (`:474-481`) |

---

#### 2. Outbound wire surface — 7 `RelayMessage` formatters

`protocol.rs:178-217`. All are `String`-returning helpers on a unit struct; there is no outbound enum.

| Formatter | Wire form | Site |
|---|---|---|
| `auth_challenge` | `["AUTH", <challenge>]` | `protocol.rs:181` |
| `event` | `["EVENT", <sub_id>, <event>]` | `:185-189` |
| `notice` | `["NOTICE", <msg>]` | `:192` |
| `eose` | `["EOSE", <sub_id>]` | `:196` |
| `ok` | `["OK", <event_id>, <bool>, <msg>]` | `:200` |
| `closed` | `["CLOSED", <sub_id>, <msg>]` | `:204` |
| `count` | `["COUNT", <sub_id>, {"count": N}]` | `:208` |

Fan-out bypasses these helpers for performance: `event_frame_for_sub` hand-formats `["EVENT","<sub>",<json>]` (`event.rs:55-57`), byte-for-byte equality with the legacy form pinned by `event.rs:1178-1189`.

---

#### 3. Complete outbound message table with exact conditions and reason strings

##### 3.1 `AUTH` challenge

| Condition | Site |
|---|---|
| Immediately after a slot is granted, before any frame is read; on send failure the connection is abandoned without registering | `connection.rs:182-192` |

##### 3.2 `OK` (EVENT / AUTH acknowledgements)

| `accepted` | Reason string | Condition | Site |
|---|---|---|---|
| `true` | `""` | AUTH verified and all gates passed | `auth.rs:282` |
| `false` | `auth-required: already authenticated` | AUTH while `Authenticated` | `auth.rs:49-57` |
| `false` | `auth-required: authentication already failed` | AUTH while `Failed` | `auth.rs:58-66` |
| `false` | `auth-required: verification failed` | allowlist denial (`pubkey_allowlist_enabled` + `Nip42`) | `auth.rs:207-211` |
| `false` | `auth-required: verification failed` | `verify_auth_event` error | `auth.rs:284-292` |
| `false` | `restricted: not a relay member` | `enforce_relay_membership` error | `auth.rs:231-235` |
| `false` | `blocked: you are banned from this community` | self or NIP-OA owner banned — **queued on `ctrl_tx`, then `cancel()`** | `auth.rs:167-183` |
| `false` | `error: internal error checking restriction state` | ban-state DB error (fail closed) | `auth.rs:126-129`, delivered `:174-177` |
| `false` | `auth-required: not authenticated` | EVENT before auth | `event.rs:644-651` |
| `false` | `invalid: event pubkey does not match authenticated identity` | `event.pubkey != auth.pubkey` and kind ≠ 1059 | `event.rs:660-668` |
| `false` | `invalid: AUTH events cannot be submitted via EVENT` | kind 22242 over EVENT | `event.rs:670-678` |
| `false` | `restricted: insufficient scope for agent observer frames` | kind 24200 without `MessagesWrite` | `event.rs:681-689` |
| `false` | `restricted: insufficient scope for ephemeral events` | kind 20000–29999 without `MessagesWrite` | `event.rs:699-707` |
| `false` | `invalid: <verify error>` | signature/id verification failure (ephemeral) | `event.rs:753-759`; (observer) `:929-935` |
| `false` | `error: internal error` | `spawn_blocking` join failure | `event.rs:783-790`, `:936-943` |
| `false` | `<check_channel_membership msg>` | ephemeral event `#h` channel not permitted | `event.rs:832-843` |
| `true` | `""` | ephemeral accepted (after publish + local fan-out) | `event.rs:896` |
| `false` | `invalid: observer frame timestamp outside ±5 minute freshness window` | \|ts − now\| > 300 s | `event.rs:975-981` |
| `false` | `invalid: observer content must be NIP-44 encrypted` | plaintext content | `event.rs:1095-1097` → `:970` |
| `false` | `invalid: observer frame missing/multiple <tag> tags` | `p` / `agent` / `frame` tag arity | `event.rs:1117-1130` → `:970` |
| `false` | `invalid: observer <tag> tag must be a hex pubkey` | bad hex | `event.rs:1111-1115` → `:970` |
| `false` | `invalid: observer frame must be agent-to-owner telemetry or owner-to-agent control` | direction cannot be derived | `event.rs:1116-1120` |
| `true` | `""` | **unknown `frame` value → silently accepted and dropped** | `event.rs:1099-1102` → `:963-967` |
| `false` | `restricted: observer frame is not authorized for this agent owner` | not the agent's owner | `event.rs:1044-1052` |
| `false` | `error: internal server error` | `is_agent_owner` DB error | `event.rs:1029-1038` |
| `false` | `rate-limited: observer frame rate exceeded (100/sec per agent)` | telemetry burst | `event.rs:1059-1066` |
| `true` | `""` | observer frame accepted | `event.rs:1091` |
| pass-through | `result.message` | persistent event → `ingest_event` outcome | `event.rs:742-746` |
| `false` | `<IngestError::Rejected msg>` / `<AuthFailed msg>` / `error: internal server error` | ingest failure, internal errors sanitised | `event.rs:748-757` |

##### 3.3 `CLOSED`

| Reason string | Condition | Site |
|---|---|---|
| `""` | successful CLOSE ack — sent **even if the sub_id was never registered** | `close.rs:27` |
| `rate-limited: too many concurrent requests` | `handler_semaphore` exhausted on REQ | `connection.rs:544-547` |
| `rate-limited: quota exceeded; retry in Ns` | admission quota on REQ | `connection.rs:662-668` |
| `rate-limited: shared admission unavailable` | Redis limiter error on REQ | `connection.rs:670-676` |
| `auth-required: not authenticated` | REQ before auth (preceded by a NOTICE) | `req.rs:80-83` |
| `auth-required: not authenticated` | COUNT before auth (**no** preceding NOTICE) | `count.rs:44-48` |
| `restricted: insufficient scope` | REQ without `MessagesRead` (preceded by a NOTICE) | `req.rs:56-59` |
| `error: too many subscriptions` | ≥1024 live subs on this connection | `req.rs:67-70` |
| `error: database error` | accessible-channel lookup failure | `req.rs:101-104`; membership confirm `:158-161`; COUNT `:85`, `:130-132` |
| `restricted: not a channel member` | `#h` channel not accessible | `req.rs:167-171` |
| `restricted: p-gated events require #p matching your pubkey` | REQ p-gate — **global subs only** | `req.rs:184-189` |
| `restricted: p-gated kinds require #p tag matching your pubkey` | COUNT p-gate — **all** COUNTs | `count.rs:56-61` |
| `restricted: agent-engram reads require authors=[self] or #p=[self]` | kind 30174 gate | `req.rs:191-196`, `count.rs:63-68` |
| `restricted: author-only kinds require authors=[self]` | kinds 30300 / push-lease gate | `req.rs:198-203`, `count.rs:70-75` |
| `error: mixed search and non-search filters not supported` | some filters have `search`, some don't | `req.rs:213-218` |
| `restricted: count filter requires narrower constraints` | COUNT fallback candidate set > 5000 | `count.rs:189-195`, `:258-264` |
| `error: <db error>` | COUNT DB error — **raw error text is forwarded to the client** | `count.rs:179`, `:209`, `:249`, `:278` |

##### 3.4 `NOTICE`

| Message | Condition | Site |
|---|---|---|
| `invalid message: <parse error>` | any `ClientMessage::parse` failure | `connection.rs:493` |
| `error: frame too large (N bytes, limit M)` | oversized text frame (then close) | `connection.rs:428-432` |
| `error: binary frame too large (…)` | oversized binary frame (then close) | `connection.rs:447-451` |
| `rate-limited: too many concurrent requests` | `handler_semaphore` exhausted on EVENT or COUNT | `connection.rs:516-518`, `:566-568` |
| `rate-limited: quota exceeded; retry in Ns` | admission quota on EVENT or COUNT | `connection.rs:662-668` with `sub_id = None` (`:624-627`, `:647`) |
| `rate-limited: shared admission unavailable` | limiter error on EVENT or COUNT | `connection.rs:670-676`, same `None` |
| `auth-required: authenticate before subscribing` | REQ before auth, paired with a CLOSED | `req.rs:77-79` |
| `restricted: insufficient scope` | REQ without `MessagesRead`, paired with a CLOSED | `req.rs:55` |

##### 3.5 `EVENT`

| Path | Site |
|---|---|
| historical delivery (non-search REQ) | `req.rs:396-400` via `RelayMessage::event` |
| historical delivery (NIP-50 search) | `req.rs:720-722` |
| live fan-out — relay-local | `event.rs:239-252` (`fan_out_event_to_local_subscribers`) |
| live fan-out — persistent ingest (+ DM-visibility owner fence) | `event.rs:453-472` |
| live fan-out — Redis cross-node | `event.rs:293-306` |

##### 3.6 `EOSE`

| Condition | Site |
|---|---|
| after historical delivery completes | `req.rs:408` |
| **on historical query error** (in place of CLOSED) | `req.rs:320-325` |
| search: no accessible channels and no global access | `req.rs:520-525` |
| search: after all filters/pages | `req.rs:730` |

Never sent when `conn.send` returns `false` mid-delivery: `req.rs:398-400` and `:720-722` both `return` without EOSE.

##### 3.7 `COUNT` response

| Condition | Site |
|---|---|
| all filters processed without an early return | `count.rs:285` |

##### 3.8 Control frames (not NIP-01 payloads)

| Frame | Condition | Site |
|---|---|---|
| `Ping` | every 30 s | `connection.rs:383`, `:394` |
| `Pong` | client Ping | `connection.rs:467` |
| `Close(None)` | any `cancel()` — sent after draining queued ctrl frames | `connection.rs:334-345` |
| `Close(1012 "relay restarting")` | graceful shutdown / late registration | `state.rs:368-373`, queued `:359`, `:236` |

---

#### 4. NIP-29 expression on the wire

There is **no** NIP-29-specific message type. Channel scope is carried entirely by the `#h` generic tag:

- Subscription-level scope: `extract_channel_id_from_filters` (`req.rs:1009-1035`) — returns `Some(id)` only if **every** filter carries an `h` tag and they all agree; any kindless-h filter or divergent id → `None` (global).
- Per-filter scope for query construction: `req.rs:265-283` prefers a filter's own single `#h`, falling back to the subscription scope.
- COUNT scope: `extract_channel_from_filter` (`count.rs:17-26`) — single-valued `#h` only.

Consequences visible on the wire: a REQ mixing two different `#h` values is silently treated as a **global** subscription (`req.rs:1019-1023`) and therefore receives no channel-scoped events at all (`subscription.rs:320-327`). No NOTICE is emitted for this.

---

#### 5. Rust API exported by this group

| Item | Visibility | Consumers outside the group |
|---|---|---|
| `connection::handle_connection` | `pub` (`connection.rs:118`) | `router.rs:313` |
| `connection::{AuthState, ConnectionState}` | `pub` (`:36`, `:53`) | `handlers/*`, `api/*`, `audio/*` |
| `connection::ConnectionSubscriptions` | `pub(crate)` (`:30`) | `state.rs:34` |
| `subscription::SubscriptionRegistry` + `{ConnId, SubId, SubEntry, IndexKey, RemovedSubscription}` | `pub` (`:12`–`:45`) | `state.rs`, `handlers/side_effects.rs` |
| `handlers::event::filter_fanout_by_access` | `pub` (`event.rs:115`) | tests only outside the file; production callers all in-file (`:247`, `:307`, `:432`) |
| `handlers::event::fan_out_pubsub_event` | `pub` (`event.rs:282`) | `main.rs:827` |
| `handlers::event::fan_out_event_to_local_subscribers` | `pub(crate)` (`event.rs:241`) | `audio/handler.rs:1318`, `api/git/transport.rs:1703`, `side_effects.rs:803`/`:889`/`:2759`/`:2911` |
| `handlers::event::dispatch_persistent_event` | `pub(crate)` (`event.rs:349`) | `handlers/ingest.rs` |
| `handlers::event::bounded_kind_label` | `pub(crate)` (`event.rs:35`) | `handlers/ingest.rs` |
| `handlers::req::{p_gated_filters_authorized, engram_filters_authorized, author_only_filters_authorized, filter_can_match_*, result_gated_count_safe_for_pushdown, is_author_only_event, event_visible_to_reader, resolve_request_local_access, apply_access_scope_to_query, build_search_channel_scope_filter, apply_count_fallback_limit, count_fallback_exceeded, FILTER_QUERY_CONCURRENCY, COUNT_FALLBACK_CANDIDATE_LIMIT}` | `pub(crate)` | `count.rs`, `api/bridge.rs` |

`ab3af828` added two members to that `pub(crate)` set: `filter_can_match_persona_shared_kinds` (`req.rs:1148`, covered by the `filter_can_match_*` glob above) and `event_visible_to_reader` (`req.rs:1222`), the single combined per-event result gate now called from `count.rs:202`/`:271`, `api/bridge.rs:1295`/`:1503`/`:1569`/`:1753`, and in-file at `req.rs:388`/`:705`.
| `handlers::req::{build_event_query_from_filter, filter_fully_pushable}` | `pub` (`:737`, `:777`) | `count.rs`, `api/bridge.rs` |
| `handlers::auth::extract_auth_tag_json` | `pub` (`auth.rs:28`) | in-file tests; git-auth path documented at `auth.rs:3-13` |
| `handlers::resolve_ttl` | `pub` (`mod.rs:42`) | `ingest.rs:2125`, `side_effects.rs:1681` |

---

#### 6. Deltas against `ARCHITECTURE.md` §2

| `ARCHITECTURE.md` claim | Code | Verdict |
|---|---|---|
| Wire table lists 4 inbound / 6 outbound (`:150-159`) | 5 inbound (`protocol.rs:23-42`), 7 outbound (`protocol.rs:181-208`) | **incomplete** — `COUNT` is missing in both directions |
| "Max frame size: 65,536 bytes" (`:161`) | `DEFAULT_MAX_FRAME_BYTES = 512 * 1024` (`config.rs:14`) | **wrong by 8×** |
| "Max subscriptions per connection: 1024" (`:161`) | `MAX_SUBSCRIPTIONS = 1024` (`req.rs:26`) | correct |
| "Max historical results per filter: 500" (`:161`) | `MAX_HISTORICAL_LIMIT = 2_000` (`req.rs:25`) | **wrong by 4×** |
| Does not mention the 10-filter or 256-byte sub_id caps | `protocol.rs:11`, `:14` | **omission** (both are NIP-11-advertised) |
| §3 Step 3 describes auth without any deadline | `AUTH_TIMEOUT = 5 s`, `connection.rs:27`, `:228-252` | **omission** |


## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: API Surface

This module group has **no HTTP routes of its own**. Its API is (a) the Rust entry point
`ingest_event`, called by exactly two transports, and (b) the **kind → handler dispatch
table** that entry point implements.

---

### 1. Rust entry points and their callers

| Item | Signature | file:line | Production callers |
|---|---|---|---|
| `ingest_event` | `async fn(&Arc<AppState>, &TenantContext, Event, IngestAuth) -> Result<IngestResult, IngestError>` | `ingest.rs:1367-1425` | WS `EVENT` (`handlers/event.rs`, via `IngestAuth::Nip42`); HTTP `POST /events` (`api/bridge.rs:830`, via `IngestAuth::Http`) |
| `ingest_event_inner` | private, same signature + `&Arc<dyn Tracer>` | `ingest.rs:1453-2531` | `ingest_event` only |
| `reject_with_transport` | `fn(&'static str, &'static str)` | `ingest.rs:156-164` | `api/bridge.rs:783,851,859,867`; `handlers/event.rs:31` |
| `handle_command` | `async fn(&TenantContext, &Arc<AppState>, Event, IngestAuth) -> Result<IngestResult, IngestError>` | `command_executor.rs:36-78` | `ingest.rs:1561` only |
| `handle_side_effects` | `async fn(&TenantContext, u32, &Event, &Arc<AppState>) -> anyhow::Result<()>` | `side_effects.rs:142-177` | `ingest.rs:2462` only |
| `validate_admin_event` | `async fn(&TenantContext, u32, &Event, &Arc<AppState>) -> anyhow::Result<()>` | `side_effects.rs:259-676` | `ingest.rs:1932` only |
| `validate_standard_deletion_event` | `async fn(&TenantContext, &Event, &Arc<AppState>) -> anyhow::Result<()>` | `side_effects.rs:179-238` | `ingest.rs:1948` only |
| `validate_imeta_tags` | `fn(&[Vec<String>], &str) -> Result<(), String>` | `imeta.rs:11-208` | `ingest.rs:2239` (re-exported via `crate::api`) |
| `verify_imeta_blobs` | `async fn(&TenantContext, &[Vec<String>], &MediaStorage) -> Result<(), String>` | `imeta.rs:209-335` | `ingest.rs:2241` |

#### Public types

| Type | Variants / fields | file:line |
|---|---|---|
| `HttpAuthMethod` | `Nip98`, `DevPubkey` | `ingest.rs:54-59` |
| `IngestAuth` | `Nip42 { pubkey, scopes, channel_ids, conn_id }`, `Http { pubkey, scopes, auth_method }` | `ingest.rs:63-86` |
| `IngestResult` | `event_id: String`, `accepted: bool`, `message: String` | `ingest.rs:166-173` |
| `IngestError` | `Rejected(String)` → WS `OK false` / HTTP 400; `AuthFailed(String)` → HTTP 403; `Internal(String)` → HTTP 500 | `ingest.rs:177-184` |
| `ReactionChannelResult` | `Channel(Uuid)`, `NoChannel`, `NotFound`, `NoTarget`, `DbError(String)` | `ingest.rs:322-328` |
| `ThreadMetadataOwned` | 9 fields; `as_params()` → `buzz_db::event::ThreadMetadataParams` | `ingest.rs:535-561` |
| `PersistResult` (private) | `Duplicate`, `Inserted(sqlx::Transaction)` | `command_executor.rs:80-85` |

`IngestAuth` accessors: `pubkey()` `:88`, `principal_pubkey_bytes()` `:95`, `scopes()` `:100`,
`conn_id()` `:107`, `channel_ids()` `:117`, `is_http()` `:128`.

---

### 2. Kind acceptance census (verified)

| Measure | Count | Evidence |
|---|---|---|
| Kind constants in `buzz-core` | 130 | `crates/buzz-core/src/kind.rs` |
| `ALL_KINDS` entries | 127 | `kind.rs:566-693` |
| Distinct kinds accepted by `required_scope_for_kind` | **81** | `ingest.rs:198-306` (enumerated below) |
| …of which are in `ALL_KINDS` | 80 | `KIND_PUSH_LEASE` (30350) is accepted but absent from `ALL_KINDS` |
| `ALL_KINDS` entries **rejected** by ingest | **47** | see §5 |
| Kinds routed away from generic storage entirely | 15 | 7 command kinds + 5 moderation commands + 1984 + 42000 + 30350 |
| Kinds that get a bespoke pre-storage validator | 12 | 0, 5, 9002/9005/9000/9001/9008/9022 (via `validate_admin_event`), 9035/9036, 40003, 40008, 44200, 45002, 30174, 30175, 30300, 9007, 9021 |
| Side-effect dispatch arms in `handle_side_effects` | **13** + `_` | `side_effects.rs:143-176` |
| `validate_admin_event` match arms | **7** + `_` | `side_effects.rs:284-675` |
| `handle_command` match arms | **7** + `_` | `command_executor.rs:66-77` |

---

### 3. THE DISPATCH TABLE — every kind ingest accepts

Legend for **Route**: `store` = generic storage + fan-out; `store+SE` = stored then
`handle_side_effects`; `direct` = handled and returned, never stored; `store+direct` =
handler runs pre-storage *and* the event is stored.
`Scope` is the value returned by `required_scope_for_kind` (`ingest.rs:198`) — but see
security.md BR: in production **every** authenticated principal holds
`Scope::all_known()` (`buzz-auth/src/lib.rs:137`; `api/bridge.rs:827`), so the scope column
documents intent, not enforcement. `h`: `R` = required (`requires_h_channel_scope`
`ingest.rs:455`), `N` = forced-null (`is_global_only_kind` `ingest.rs:379`),
`O` = optional/derived.

| Kind | Constant | Scope | `h` | Extra required tags / pre-storage validator | Route | Distinct reject strings |
|---|---|---|---|---|---|---|
| 0 | `KIND_PROFILE` | UsersWrite | N | content must parse as JSON (`ingest.rs:2260`) | store+SE (`handle_kind0_profile` `side_effects.rs:1113`) | `invalid: kind:0 content must be valid JSON` |
| 1 | `KIND_TEXT_NOTE` | MessagesWrite | N | — | store | — |
| 3 | `KIND_CONTACT_LIST` | UsersWrite | N | — | store (replaceable) | — |
| 5 | `KIND_DELETION` | MessagesWrite | O (derived from target) | exactly one `e`+`a` target (`ingest.rs:1972`); `validate_standard_deletion_event` (`ingest.rs:1948`) | store+SE (`handle_standard_deletion_event` `side_effects.rs:2108`) | `invalid: malformed deletion target id`; `invalid: deletion events must reference exactly one target via e or a tag (got e=…, a=…)`; `invalid: must be event author`; `invalid: target event not found`; `invalid: missing e or a tag for target`; `invalid: invalid a-tag format`; `invalid: invalid pubkey in a-tag` |
| 7 | `KIND_REACTION` | MessagesWrite | O (derived from target's channel, `ingest.rs:1671`) | ≥1 `e` tag w/ 64-hex; emoji ≤64 chars (`ingest.rs:2335`) | **inline transactional** (`insert_reaction_event_with_thread_metadata` `ingest.rs:2298`) — deliberately excluded from `is_side_effect_kind` (`side_effects.rs:31-34`) | `invalid: reaction must reference a target event via e tag`; `invalid: reaction target event not found`; `invalid: malformed reaction target id`; `invalid: reaction emoji exceeds 64 characters (got n)`; OK-but-rejected `duplicate: reaction already exists` |
| 9 | `KIND_STREAM_MESSAGE` | MessagesWrite | **R** | NIP-10 `e root`/`e reply` resolved (`ingest.rs:2246`) | store | `invalid: channel-scoped events must include an h tag`; `invalid: reply parent not found`; `invalid: parent event belongs to a different channel`; `invalid: parent event has no channel association`; `invalid: root tag does not match thread ancestry`; `invalid: thread depth limit exceeded` |
| 1059 | `KIND_GIFT_WRAP` | MessagesWrite | forced `None` (`ingest.rs:1690`) | pubkey-match check waived (`ingest.rs:1525`) | store (**WS only** — `ingest.rs:1475`) | `invalid: kind 1059 is only accepted via WebSocket` |
| 1617 | `KIND_GIT_PATCH` | MessagesWrite | N | — | store | — |
| 1618 | `KIND_GIT_PULL_REQUEST` | MessagesWrite | N | — | store | — |
| 1619 | `KIND_GIT_PR_UPDATE` | MessagesWrite | N | — | store | — |
| 1621 | `KIND_GIT_ISSUE` | MessagesWrite | N | — | store | — |
| 1630–1633 | `KIND_GIT_STATUS_{OPEN,MERGED,CLOSED,DRAFT}` | MessagesWrite | N | — | store | — |
| 1984 | `KIND_REPORT` | MessagesWrite | — | `report::handle_report_event` (`ingest.rs:1588`) | **direct** → `moderation_reports` only; never stored, never fanned out | whatever `handle_report_event` returns, wrapped verbatim as `Rejected` |
| 9000 | `KIND_NIP29_PUT_USER` | AdminChannels | **R** | `p` tag; optional `role`; `validate_admin_event` 9000 arm (`side_effects.rs:284`) | store+SE (`handle_put_user` `side_effects.rs:1203`) | `invalid: missing p tag`; `invalid: invalid role: X`; `invalid: actor not authorized`; `invalid: only owners/admins may grant elevated roles`; `invalid: policy:owner_only — only the agent owner can add this agent`; `invalid: policy:owner_only — agent has no owner set`; `invalid: policy:nobody — this agent has disabled external channel additions` |
| 9001 | `KIND_NIP29_REMOVE_USER` | AdminChannels | **R** | `p` tag; 9001 arm (`side_effects.rs:373`) | store+SE (`handle_remove_user` `side_effects.rs:1265`) | `invalid: missing p tag`; `invalid: actor is not an active member`; `invalid: cannot remove the last owner`; `invalid: actor not authorized` |
| 9002 | `KIND_NIP29_EDIT_METADATA` | AdminChannels if an `archived` tag is present, else ChannelsWrite (`ingest.rs:276-287`) | **R** | ≥1 of `name/about/archived/topic/purpose/visibility/ttl` (`side_effects.rs:410`) | store+SE (`handle_edit_metadata` `side_effects.rs:1335`) | `invalid: kind:9002 must include at least one metadata tag (…)`; `invalid: invalid archived value: X (must be "true" or "false")`; `invalid: archived tag must have a value`; `invalid: channel name is required`; `invalid: invalid visibility value: X`; `invalid: visibility tag must have a value`; `invalid: invalid ttl value: X`; `invalid: ttl tag must have a value (…)`; `invalid: actor not authorized for name/about/archived/visibility/ttl changes`; `invalid: not a member` |
| 9005 | `KIND_NIP29_DELETE_EVENT` | MessagesWrite | **R** | exactly one `e`+`a` (`ingest.rs:1972`); 9005 arm (`side_effects.rs:508`) | store+SE (`handle_delete_event_side_effect` `side_effects.rs:1560`) | `invalid: missing e tag for target event`; `invalid: invalid action_id tag`; `invalid: target event not found`; `invalid: target event belongs to a different channel`; `invalid: target event has no channel`; `invalid: must be event author or channel owner/admin` |
| 9007 | `KIND_NIP29_CREATE_GROUP` | ChannelsWrite | O (client-chosen UUID) | `name` non-empty after canonicalisation; `visibility`∈{open,private} default open; `channel_type` default stream (`ingest.rs:2031-2085`) | pre-create channel (`ingest.rs:2129`) then store+SE (`handle_create_group` `side_effects.rs:1660`) | `invalid: channel name is required`; `invalid visibility: X`; `invalid channel_type: X`; OK-but-rejected `duplicate: channel already exists` |
| 9008 | `KIND_NIP29_DELETE_GROUP` | AdminChannels | **R** | 9008 arm — owner only or owner-of-owner-agent (`side_effects.rs:625`) | store+SE (`handle_delete_group` `side_effects.rs:1783`) | `invalid: only owner can delete group` |
| 9021 | `KIND_NIP29_JOIN_REQUEST` | ChannelsRead | O (but rejected if absent, `ingest.rs:2162`) | channel must exist and be `open` (`ingest.rs:2167`) | store+SE (`handle_join_request` `side_effects.rs:1835`) | `invalid: join request must include an h tag`; `restricted: channel is private`; `invalid: channel not found`; SE-only: `channel is private — request an invitation` |
| 9022 | `KIND_NIP29_LEAVE_REQUEST` | ChannelsRead | **R** | 9022 arm — active member, not last owner (`side_effects.rs:645`) | store+SE (`handle_leave_request` `side_effects.rs:1913`) | `invalid: actor is not an active member`; `invalid: cannot remove the last owner` |
| 9030 | `RELAY_ADMIN_ADD_MEMBER` | AdminUsers | N | delegated to `relay_admin::handle_relay_admin_event` (`ingest.rs:1835`) | **direct** — mutates `relay_members`, not stored | `restricted: relay admin commands require a global token, not a channel-scoped token`; `invalid: {handler error}` |
| 9031 | `RELAY_ADMIN_REMOVE_MEMBER` | AdminUsers | N | ditto | direct | ditto |
| 9032 | `RELAY_ADMIN_CHANGE_ROLE` | AdminUsers | N | ditto | direct | ditto |
| 9033 | `RELAY_ADMIN_SET_WORKSPACE_PROFILE` | AdminUsers | N | ditto | direct | ditto |
| 9035 | `KIND_IA_ARCHIVE_REQUEST` | **UsersWrite** (deliberately not AdminUsers — `ingest.rs:265-273`) | N | `identity_archive::handle_identity_archive_event` (`ingest.rs:1942`) | **store+direct** — handler runs pre-storage, then the request itself is stored so the 8002 delta's `["e", request_id]` resolves | `invalid: {handler error}` |
| 9036 | `KIND_IA_UNARCHIVE_REQUEST` | UsersWrite | N | ditto | store+direct | `invalid: {handler error}` |
| 9040 | `KIND_MODERATION_BAN` | MessagesWrite | N | `moderation_commands::handle_moderation_command` (`ingest.rs:1606`) | **direct** — exempt from the ban/timeout write-block gate (`ingest.rs:1613`) | handler string verbatim (`restricted: moderator access required`, `invalid: event timestamp out of range: …`, …) |
| 9041–9044 | `KIND_MODERATION_{UNBAN,TIMEOUT,UNTIMEOUT,RESOLVE_REPORT}` | MessagesWrite | N | ditto | direct | ditto |
| 10000 | `KIND_MUTE_LIST` | UsersWrite | N | — | store (replaceable) | — |
| 10001 | `KIND_PIN_LIST` | UsersWrite | N | — | store (replaceable) | — |
| 10002 | `KIND_NIP65_RELAY_LIST_METADATA` | UsersWrite | N | — | store (replaceable) | — |
| 10003 | `KIND_BOOKMARK_LIST` | UsersWrite | N | — | store (replaceable) | — |
| 10030 | `KIND_EMOJI_LIST` | UsersWrite | N | — | store (replaceable) | — |
| 10100 | `KIND_AGENT_PROFILE` | UsersWrite | N | — | store+SE (`handle_agent_profile` `side_effects.rs:1078`) | SE-only: `kind:10100 content parse error: …`; `kind:10100 missing channel_add_policy field` |
| 28936 | `KIND_NIP43_LEAVE_REQUEST` | ChannelsRead | N | `require_relay_membership=true`; ±120 s freshness; NIP-70 `["-"]` tag (`ingest.rs:1846-1882`) | **direct** — removes from `relay_members`, not stored | `restricted: leave requests require a global token`; `invalid: relay membership is not enabled`; `invalid: leave request timestamp out of range (delta=Ns, max ±120s)`; `invalid: leave request must include NIP-70 protected event tag ["-"]`; `invalid: you are not a relay member`; `invalid: relay owner cannot leave`; success msg `info: you have left this relay` |
| 30000 | `KIND_FOLLOW_SET` | UsersWrite | N | `d` ≤1024 B | store (NIP-33) | `invalid: d tag too long (n bytes, max 1024)` |
| 30003 | `KIND_BOOKMARK_SET` | UsersWrite | N | ditto | store (NIP-33) | ditto |
| 30023 | `KIND_LONG_FORM` | MessagesWrite | N | ditto | store (NIP-33) | ditto |
| 30030 | `KIND_EMOJI_SET` | UsersWrite | N | ditto | store (NIP-33) | ditto |
| 30078 | `KIND_READ_STATE` | UsersWrite | N | ditto | store (NIP-33) | ditto |
| 30174 | `KIND_AGENT_ENGRAM` | UsersWrite | N | exactly one 64-lc-hex `d`; exactly one 64-lc-hex `p`; content = NIP-44 v2 shape (`ingest.rs:965-1025`) | store (NIP-33) | `invalid: agent-engram event must have exactly one \`d\` tag (got n)`; `…one \`p\` tag (got n)`; `invalid: agent-engram \`d\` tag must be 64 lowercase hex chars`; `invalid: agent-engram \`p\` tag must be 64 lowercase hex chars (pubkey)`; `invalid: agent-engram content must not be empty (NIP-44 ciphertext)`; `invalid: agent-engram content is not valid base64 (length)`; `invalid: agent-engram content is not valid base64`; `invalid: agent-engram content too short for NIP-44 v2`; `invalid: agent-engram content is not NIP-44 v2 (expected 0x02 version prefix)` |
| 30175 | `KIND_PERSONA` | UsersWrite | N | exactly one non-empty `d` matching `^[a-z0-9][a-z0-9_-]{0,63}$`, **plus at most one `shared` tag which must be exactly `["shared","true"]`** (`ingest.rs:1034-1093`; `shared` rules `:1042-1060`) | store (NIP-33) | `invalid: persona event must have exactly one \`d\` tag (got n)`; `invalid: persona event \`d\` tag must not be empty`; `invalid: persona event \`d\` tag too long (n chars, max 64)`; `invalid: persona event \`d\` tag must start with a lowercase letter or digit`; `invalid: persona event \`d\` tag must match [a-z0-9_-] after the first character`; **`invalid: persona event \`shared\` tag must be exactly ["shared","true"] (got […])`**; **`invalid: persona event must have at most one \`shared\` tag (got n)`** |
| 30176 | `KIND_TEAM` | UsersWrite | N | `d` ≤1024 B only | store (NIP-33) | `invalid: d tag too long …` |
| 30177 | `KIND_MANAGED_AGENT` | UsersWrite | N | ditto | store (NIP-33) | ditto |
| 30300 | `KIND_EVENT_REMINDER` | UsersWrite | N | exactly one non-empty `d`; ≤1 `not_before` (canonical decimal ≤ 2^53−1, ≤ now+`SPROUT_MAX_NOT_BEFORE_DELTA`); `expiration > not_before` when both present (`ingest.rs:1252-1326`) | store (NIP-33) | `invalid: malformed not_before`; `invalid: missing d tag`; `invalid: duplicate d tag`; `invalid: empty d tag`; `invalid: not_before too far in future`; `invalid: expiration before not_before` |
| 30315 | `KIND_USER_STATUS` | UsersWrite | N | `d` ≤1024 B | store (NIP-33) | `invalid: d tag too long …` |
| 30350 | `push_lease::KIND_PUSH_LEASE` | UsersWrite | N | `push_lease::accept` (`ingest.rs:2183`) | **direct** — `push_leases` table, not the events table | `invalid: stale replacement`; `invalid: stale generation`; `invalid: endpoint already leased`; `invalid: lease quota exceeded`; `invalid: source event collision`; `invalid: lease constraint violation`; `invalid: {validation}`; `Internal({reason})` |
| 30617 | `KIND_GIT_REPO_ANNOUNCEMENT` | **ReposWrite** | N | `d` = repo id `[a-zA-Z0-9._-]{1,64}`, no leading dot, no `..` (`side_effects.rs:2391`) | store+SE (`handle_git_repo_announcement` `side_effects.rs:2412`) | SE-only (logged, not returned): `kind:30617 missing d tag`; `invalid repo identifier: …`; `repo name 'X' already taken by another owner`; `repo limit exceeded: n >= m`; `failed to ensure manifest pointer: …` |
| 30618 | `KIND_GIT_REPO_STATE` | ReposWrite | N | `d` ≤1024 B | store (NIP-33) | `invalid: d tag too long …` |
| 30620 | `KIND_WORKFLOW_DEF` | MessagesWrite | required by handler (`h`) | `h` UUID; `d` = workflow UUID; caller must be channel member; YAML parses (`command_executor.rs:653`) | **command executor** (own tx) | `invalid: missing h tag (channel_id)`; `invalid: bad channel_id format`; `invalid: missing d tag (workflow_id)`; `invalid: bad workflow_id format`; `forbidden: not a member of this channel`; `invalid: workflow YAML parse error: …`; `forbidden: workflow belongs to a different owner or channel`; `invalid: workflow channel not found`; `invalid: d tag too long …`; `duplicate: already processed` |
| 40002 | `KIND_STREAM_MESSAGE_V2` | MessagesWrite | **R** | NIP-10 resolution | store | same thread errors as kind 9 |
| 40003 | `KIND_STREAM_MESSAGE_EDIT` | MessagesWrite | **R** | `validate_edit_ownership` (`ingest.rs:763`) — target exists, same channel, actor is effective author **or** the agent's owning human; author path re-gates membership | store | `invalid: missing e tag for edit target`; `invalid: invalid target event ID`; `invalid: edit target event not found`; `invalid: target event belongs to a different channel`; `invalid: target event has no channel`; `invalid: must be event author to edit`; `invalid: restricted: not a channel member`; `invalid: db error…` |
| 40004 | `KIND_STREAM_MESSAGE_PINNED` | MessagesWrite | **R** | — | store | h-tag error only |
| 40005 | `KIND_STREAM_MESSAGE_BOOKMARKED` | MessagesWrite | **R** | — | store | h-tag error only |
| 40006 | `KIND_STREAM_MESSAGE_SCHEDULED` | MessagesWrite | **R** | — | store | h-tag error only |
| 40007 | `KIND_STREAM_REMINDER` | MessagesWrite | **R** | — | store | h-tag error only |
| 40008 | `KIND_STREAM_MESSAGE_DIFF` | MessagesWrite | **R** | `validate_diff_event` (`ingest.rs:896`) — content ≤60 KiB, `repo` http(s), `commit` ≥7 hex | store | `invalid: diff content exceeds 60KB limit (got n bytes)`; `invalid: repo URL must be http or https`; `invalid: commit SHA must be at least 7 hex characters`; `invalid: parent-commit SHA must be at least 7 hex characters`; `invalid: branch tag requires both source and target`; `invalid: pr number must be a positive integer`; `invalid: diff event requires a repo tag`; `invalid: diff event requires a commit tag` |
| 40100 | `KIND_CANVAS` | ChannelsWrite | **R** | — | store | h-tag error only |
| 41010 | `KIND_DM_OPEN` | MessagesWrite | — | 1–8 `p` tags (9 total max) (`command_executor.rs:310`) | command executor | `invalid: pubkeys must contain at least 1 other participant`; `invalid: pubkeys may contain at most 8 other participants (9 total)`; `invalid: bad pubkey hex: X`; `invalid: pubkey must be 32 bytes: X`; success `response:{"channel_id":…,"created":bool}`; `duplicate: already processed` |
| 41011 | `KIND_DM_ADD_MEMBER` | MessagesWrite | `h` required by handler | `h` UUID + ≥1 `p`; caller must be a member; channel must be `dm`; total ≤9 (`command_executor.rs:443`) | command executor (creates a **new** DM — sets are immutable) | `invalid: missing h tag (channel_id)`; `invalid: bad channel_id format`; `invalid: must specify at least 1 new participant in p tags`; `forbidden: not a member of this DM`; `invalid: DM not found`; `invalid: channel is not a DM`; `invalid: DM supports at most 9 participants`; success `response:{"channel_id":…}` |
| 41012 | `KIND_DM_HIDE` | MessagesWrite | `h` required by handler | `h` UUID; member; channel is `dm` (`command_executor.rs:580`) | command executor | `invalid: missing h tag (channel_id)`; `invalid: bad channel_id format`; `forbidden: not a member of this DM`; `invalid: DM not found`; `invalid: channel is not a DM`; success message `{}` |
| 42000 | `KIND_PRODUCT_FEEDBACK` | MessagesWrite | — | `product_feedback::handle` (`ingest.rs:1567`) | **direct** → private deployment table; no storage, no fan-out | handler string verbatim |
| 44200 | `KIND_AGENT_TURN_METRIC` | MessagesWrite | N (an explicit `h` tag is a **hard reject**) | exactly one 64-lc-hex `p`; exactly one 64-lc-hex `agent` == `event.pubkey`; no `h`; NIP-44 v2 content; `p` must be the DB-registered owner of `event.pubkey` (`ingest.rs:1177-1247`, `:1981-2016`) | store | ``invalid: agent-turn-metric event must not have an `h` tag (…)``; ``invalid: agent-turn-metric event must have exactly one `p` tag (got n)``; ``invalid: agent-turn-metric `p` tag must be 64 lowercase hex chars``; ``invalid: agent-turn-metric event must have exactly one `agent` tag (got n)``; ``invalid: agent-turn-metric `agent` tag must be 64 lowercase hex chars``; ``invalid: agent-turn-metric `agent` tag must equal event pubkey``; `invalid: agent-turn-metric content …`; ``restricted: agent-turn-metric `p` tag must be the registered owner of this agent`` |
| 45001 | `KIND_FORUM_POST` | MessagesWrite | **R** | NIP-10 resolution | store | h-tag + thread errors |
| 45002 | `KIND_FORUM_VOTE` | MessagesWrite | **R** | `validate_forum_vote_target` (`ingest.rs:844`) — target exists, kind ∈ {45001, 45003}, same channel | store | `invalid: missing e tag for vote target`; `invalid: invalid target event ID`; `invalid: vote target event not found`; `invalid: vote target must be a forum post or comment`; `invalid: target event belongs to a different channel`; `invalid: target event has no channel` |
| 45003 | `KIND_FORUM_COMMENT` | MessagesWrite | **R** | NIP-10 resolution | store | h-tag + thread errors |
| 46020 | `KIND_WORKFLOW_TRIGGER` | MessagesWrite | — | `d` or `e` = workflow UUID; caller must be workflow **owner** (`command_executor.rs:809`) | command executor; persisted under `workflow.channel_id` | `invalid: missing workflow reference (d or e tag)`; `invalid: bad workflow_id format`; `invalid: workflow not found`; `forbidden: not authorized to trigger this workflow`; success `response:{"run_id":…}` |
| 46030 | `KIND_APPROVAL_GRANT` | MessagesWrite | — | `d`/`e` = token hash hex; approval pending + unexpired; `check_approver_spec` (`command_executor.rs:961`) | command executor | `invalid: missing approval reference (d or e tag)`; `invalid: bad approval token hash hex`; `invalid: approval not found`; `invalid: approval already {status}`; `invalid: approval token has expired`; `forbidden: not the designated approver for this request`; `forbidden: approver spec 'X' is not yet supported`; `invalid: approval already acted on (race)`; success `response:{"status":"granted","run_id":…}` |
| 46031 | `KIND_APPROVAL_DENY` | MessagesWrite | — | ditto | command executor | ditto, success `response:{"status":"denied","run_id":…}` |
| 48100–48103 | `KIND_HUDDLE_{STARTED,PARTICIPANT_JOINED,PARTICIPANT_LEFT,ENDED}` | ChannelsWrite | **R** | — | store | h-tag error only |
| 48106 | `KIND_HUDDLE_GUIDELINES` | ChannelsWrite | **R** | — | store | h-tag error only |

---

### 4. Cross-cutting reject strings (apply to every kind)

Emitted before per-kind dispatch, in this order:

| # | Condition | Variant | String | file:line |
|---|---|---|---|---|
| 1 | kind == 22242 | Rejected | `invalid: AUTH events cannot be submitted` | `ingest.rs:1464` |
| 2 | kind ∈ {44100, 44101} | Rejected | `invalid: membership notifications are relay-signed only` | `ingest.rs:1469` |
| 3 | HTTP + kind ∈ {1059, 20001} | Rejected | `invalid: kind {n} is only accepted via WebSocket` | `ingest.rs:1475` |
| 4 | `is_relay_only_kind` (13534, 30622, 39005, 39006, 40901, 40902) | Rejected | `restricted: relay-only kind` | `ingest.rs:1481` |
| 5 | signature/id verify fail | Rejected | `invalid: {VerificationError}` | `ingest.rs:1495` |
| 5b | verify task panic | Internal | `error: internal verification error` | `ingest.rs:1499` |
| 6 | \|created_at − now\| > 900 s | Rejected | `invalid: event timestamp too far from server time` | `ingest.rs:1509` |
| 7 | content > 262 144 B | Rejected | `invalid: content exceeds maximum size of 262144 bytes (got n)` | `ingest.rs:1516` |
| 8 | `event.pubkey != auth.pubkey` and not gift-wrap | **AuthFailed** | `invalid: event pubkey does not match authenticated identity` | `ingest.rs:1525` |
| 9 | kind not in the 81-kind allowlist | Rejected | `restricted: unknown event kind` | `ingest.rs:305`, raised at `:1507` |
| 10 | relay-admin kind + channel-scoped token | AuthFailed | `restricted: relay admin commands require a global token, not a channel-scoped token` | `ingest.rs:1538` |
| 11 | kind 28936 + channel-scoped token | AuthFailed | `restricted: leave requests require a global token` | `ingest.rs:1546` |
| 12 | missing scope | AuthFailed | `restricted: insufficient scope (need {scope})` | `ingest.rs:1551` |
| 13 | actor banned (all kinds except 9040–9044 and 9030–9033) | AuthFailed | `blocked: you are banned from this community` | `ingest.rs:1647` |
| 14 | actor timed out | AuthFailed | `restricted: you are timed out until {unix_ts}` | `ingest.rs:1653` |
| 15 | restriction lookup DB error (fail-closed) | Internal | `error: internal error checking restriction state: {e}` | `ingest.rs:1663` |
| 16 | requires `h`, none present | Rejected | `invalid: channel-scoped events must include an h tag` | `ingest.rs:1739` |
| 17 | channel-scoped token + global event | AuthFailed | `restricted: channel-scoped tokens cannot publish global events` | `ingest.rs:1752` |
| 18 | not a member and channel not `open` | Rejected | `restricted: not a channel member` | `ingest.rs:521` |
| 19 | membership lookup failed | Rejected | `error: database error: {e}` (note: **`Rejected`**, not `Internal`) | `ingest.rs:501` |
| 20 | channel `archived_at IS NOT NULL` (except 9002 `archived=false`) | Rejected | `invalid: channel is archived` | `ingest.rs:1964` |
| 21 | replaceable/NIP-33 write lost the LWW race | **Ok** `accepted=true` | `duplicate:` | `ingest.rs:2453-2457` |

`skip_membership` set (`ingest.rs:1796-1801`): 9021, 9007, 40003, 9002, 9005, 9008 — six
kinds bypass gate #18 and rely on their per-kind validator instead.

---

### 5. `ALL_KINDS` entries ingest **rejects** (47)

| Reject reason | Count | Kinds |
|---|---|---|
| `restricted: relay-only kind` (`ingest.rs:1481`) | 6 | 13534, 30622, 39005, 39006, 40901, 40902 |
| `invalid: membership notifications are relay-signed only` | 2 | 44100, 44101 |
| `restricted: unknown event kind` (no allowlist arm) | 39 | 41, 1063, 8000, 8001, 8002, 8003, 9009, 13535, 20001, 20002, 24134, 24200, 24242, 24810, 27235, 39000, 39001, 39002, 39003, 40099, 41001, 43001–43006, 46001–46007, 46010–46012, 48001, 49001 |

Notes:
- 20001 (`KIND_PRESENCE_UPDATE`) and 20002 are ephemeral and handled in
  `handlers/event.rs` before ingest — they never reach `required_scope_for_kind`.
  The HTTP guard at `ingest.rs:1449` names 20001 explicitly even though the kind can
  never pass the allowlist anyway (belt-and-braces, or vestigial).
- 8000/8001/8002/8003/13535/39000–39002 are relay-signed *outputs* of this very module
  (`side_effects.rs:2881`, `:3008`, `:3142`, `:962`) but are **not** in
  `is_relay_only_kind`, so their rejection message is the generic
  `restricted: unknown event kind` rather than `restricted: relay-only kind`.
- 9009 (`KIND_NIP29_CREATE_INVITE`) has a live `handle_side_effects` arm
  (`side_effects.rs:161-167`) that can never be reached — see debt.md D-08.

---

### 6. Success-response shapes

| Route | `accepted` | `message` |
|---|---|---|
| Generic store, new | `true` | `""` (`ingest.rs:2501`) |
| Generic store, LWW-dominated / dup id | `true` | `"duplicate:"` (`ingest.rs:2456`) |
| Reaction, new | `true` | `""` (`ingest.rs:2362`) |
| Reaction, active duplicate | **`false`** | `"duplicate: reaction already exists"` (`ingest.rs:2344`) |
| 9007 channel already exists | **`false`** | `"duplicate: channel already exists"` (`ingest.rs:2146`) |
| Direct handlers (9030–9033, 9040–9044, 1984, 42000, 30350) | `true` | `""` |
| 28936 leave | `true` | `"info: you have left this relay"` (`ingest.rs:1926`) |
| Command kinds, first time | `true` | `"response:{json}"` or `"{}"` (41012) |
| Command kinds, replay | `true` | `"duplicate: already processed"` |

The `accepted=true` + `message="duplicate:"` shape is the **NIP-33 LWW write-conflict
signal**: `buzz-cli` translates it to `CliError::Conflict` → exit code 5
(`crates/buzz-cli/src/commands/mem.rs:105-108`, `commands/notes.rs:560-563`,
`crates/buzz-cli/src/error.rs:103`).

---

### 7. imeta sub-API (`imeta.rs`)

`validate_imeta_tags` allows 13 keys (`imeta.rs:11-14`), of which 12 are singletons
(`imeta.rs:15-18`); `url`, `m`, `x`, `size` are all mandatory (`imeta.rs:163`).
`verify_imeta_blobs` performs 5 storage checks (`imeta.rs:246`, `:252`, `:262`, `:290`,
`:305`). Private helpers: `is_well_formed_mime` `:340`,
`extract_hash_from_media_url` `:351`, `extract_ext_from_media_url` `:362`,
`is_local_media_url` `:373`.

| Reject string | Line |
|---|---|
| `only imeta tags allowed in media_tags` | `imeta.rs:32` |
| `disallowed imeta key: {k}` | `imeta.rs:52` |
| `duplicate imeta key: {k}` | `imeta.rs:55` |
| `imeta url must be a local /media/ path` | `imeta.rs:61` |
| `imeta url must not be a thumbnail path; use thumb field` | `imeta.rs:64` |
| `imeta m must be a valid MIME type` | `imeta.rs:77` |
| `imeta x must be a 64-char lowercase hex SHA-256` | `imeta.rs:86` |
| `imeta size must be a positive integer` | `imeta.rs:94` |
| `imeta thumb must be a local .thumb.jpg path` | `imeta.rs:103` |
| `imeta duration must be a positive finite number` / `… must be a valid float` | `imeta.rs:110`, `:113` |
| `imeta bitrate must be a positive integer` | `imeta.rs:117` |
| `imeta image must be a local /media/ path` | `imeta.rs:122` |
| `imeta image must reference a standalone poster frame, not a thumbnail` | `imeta.rs:126` |
| `imeta image must reference an image file (jpg, png, gif, webp), not video` | `imeta.rs:133` |
| `imeta filename must be 1–255 chars` | `imeta.rs:144` |
| `imeta filename must not contain path separators or control characters` | `imeta.rs:151` |
| `imeta tag must include url, m, x, and size` | `imeta.rs:164` |
| `imeta {key} is only valid for video/mp4, not {m}` | `imeta.rs:172` |
| `imeta url hash does not match x` | `imeta.rs:181` |
| `imeta url extension does not match m` | `imeta.rs:189` |
| `imeta thumb hash does not match x` | `imeta.rs:198` |
| `imeta references nonexistent blob: {x}` | `imeta.rs:249` |
| `imeta blob object missing in storage: {x}` | `imeta.rs:257` |
| `storage error checking blob {x}: {e}` | `imeta.rs:255` |
| `imeta m ({m}) does not match stored MIME ({stored})` | `imeta.rs:262` |
| `imeta size ({n}) does not match stored size ({stored})` | `imeta.rs:268` |
| `imeta duration ({d}) does not match stored duration ({stored})` | `imeta.rs:275` |
| `imeta thumb references missing thumbnail: {x}` | `imeta.rs:293` |
| `imeta image URL has no extractable hash: {url}` | `imeta.rs:306` |
| `imeta image references nonexistent poster: {hash}` | `imeta.rs:311` |
| `imeta image poster MIME must be image type, got {m}` | `imeta.rs:315` |
| `imeta image extension ({url_ext}) does not match stored extension ({sidecar})` | `imeta.rs:322` |
| `imeta image references missing poster frame: {hash}` | `imeta.rs:332` |

All 33 strings reach the client prefixed with `invalid: ` (`ingest.rs:2240`, `:2217`).

---

### 8. Doc/code delta

| Claim | Location | Reality |
|---|---|---|
| "Both WebSocket `["EVENT", ...]` and HTTP `POST /events` feed into `ingest_event` — two doors, one room." | `ingest.rs:3-4` | **Accurate**, with two documented asymmetries: HTTP rejects kinds 1059 and 20001 (`ingest.rs:1449`), and `IngestAuth::Http` has no `channel_ids` field at all. |
| "Command kinds (41010–41012, 30620, 46020, 46030–46031) are processed transactionally: validate → begin tx → insert event → execute mutations → commit." | `command_executor.rs:3-4` | Kind list **matches** `is_command_kind` (`buzz-core/src/kind.rs:743-755`). The atomicity claim is contradicted five lines later by the module's own note that "Domain mutations … execute on the connection pool, NOT inside this transaction" (`command_executor.rs:92-98`). |
| `ARCHITECTURE.md` mentions ingest exactly once | `ARCHITECTURE.md:620` | No ingest pipeline, kind-dispatch table, or side-effect contract is documented anywhere in `ARCHITECTURE.md`. This table has no doc counterpart. |


## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: API Surface

Scope: the 13 assigned files (9,626 LOC). Route registrations cross-checked against
`crates/buzz-relay/src/router.rs` (530 lines) line by line.

---

#### 1. Complete endpoint inventory

Auth-mechanism legend:
- **NIP-98\*** — `Authorization: Nostr <b64(kind:27235)>` **or** the unsigned `X-Pubkey: <hex>`
  header when `config.require_auth_token == false` (**default false**, `config.rs:475-477`).
  The `*` marks endpoints that are impersonatable by default. See `bridge.rs:117-125`.
- **NIP-98 (strict)** — `verify_bridge_auth_with_options(..., require_auth_token = true, ...)`;
  no `X-Pubkey` path.
- **Blossom** — kind:24242 event in `Authorization: Nostr <b64>` (`media.rs:887-908`).
- **Operator allowlist** — NIP-98 (strict) + pubkey ∈ `config.relay_operator_pubkeys`.
- **Admin host** — `Host` header equals `config.admin.host` exactly, plus an `Origin`
  match *if* `Origin` is present. **No credential of any kind.** (`admin/auth.rs:16-35`)
- **Webhook secret** — `X-Webhook-Secret` header or `?secret=`.
- **none** — no authentication.

##### 1a. Nostr HTTP bridge (`api/bridge.rs`)

| Method | Path | Handler (`file:line`) | Route reg. | Auth | Request | Response | Error codes |
|---|---|---|---|---|---|---|---|
| POST | `/events` | `submit_event` `bridge.rs:613` | `router.rs:71` | NIP-98\* + relay membership + Redis rate limit | raw signed Nostr event JSON (≤1 MiB) | `{event_id, accepted, message}` | 400 parse/`Rejected`, 401 auth+replay, 403 `AuthFailed`/membership, 404 unmapped host, 429 quota, 500 `Internal`, 503 limiter down |
| POST | `/query` | `query_events` `bridge.rs:880` | `router.rs:72` | NIP-98\* + membership + rate limit | JSON array of NIP-01 filters + bridge extensions | JSON array of events | 400 bad filters / bad cursor / mixed search, 401, 403 p-gate/engram/author-only/membership, 404, 429, 500, 503 |
| POST | `/count` | `count_events` `bridge.rs:1314` | `router.rs:73` | NIP-98\* + membership + rate limit | JSON array of NIP-45 filters | `{count: N}` | 400 bad filters / "count filter requires narrower constraints", 401, 403, 404, 429, 500, 503 |
| GET | `/moderation/reports` | `moderation_reports` `bridge.rs:2092` | `router.rs:113` | NIP-98\* + `ModerationAction::ViewQueue` | `?status=&limit=` (clamped ≤500) | JSON array of report rows | 401, 403 `restricted: moderator access required`, 404, 500 |
| GET | `/moderation/audit` | `moderation_audit` `bridge.rs:2118` | `router.rs:114` | NIP-98\* + ViewQueue | `?limit=` (≤500) | JSON array of action rows | 401, 403, 404, 500 |
| GET | `/moderation/restricted` | `moderation_restricted` `bridge.rs:2136` | `router.rs:115-118` | NIP-98\* + ViewQueue | none | JSON array of ban rows | 401, 403, 404, 500 |
| POST | `/hooks/{id}` | `workflow_webhook` `bridge.rs:1780` | `router.rs:120` | **webhook secret only** | UUID path + optional JSON body | 202 `{run_id, workflow_id, status:"pending"}` | 400 bad UUID / not a webhook trigger / bad JSON, 401 bad-or-unconfigured secret, 404 unmapped host or unknown workflow, 500 corrupt definition / DB |

Notes on the bridge:
- The three POST bodies are capped at **1 MiB** by `RequestBodyLimitLayer` (`router.rs:130`).
- Tenant is bound from `Host` before any auth (`bridge.rs:626`, `:894`, `:1327`, `:1783`, `:2018`);
  unmapped host → generic 404 with a fixed string, never echoing the host.
- `bridge.rs:635`, `:903`, `:1336`, `:2031` build the NIP-98 `u`-tag expectation from
  `tenant.host()`, not `config.relay_url`'s host (`nip98_expected_url`, `bridge.rs:195-206`).
- `authorize_moderation_read` (`bridge.rs:2026`) appends the **verbatim** raw query string to the
  expected URL (`:2027-2030`), so `?limit=…&status=…` reads verify.

##### 1b. Blossom media (`api/media.rs`)

| Method | Path | Handler (`file:line`) | Route reg. | Auth | Request | Response | Error codes |
|---|---|---|---|---|---|---|---|
| PUT | `/upload` | `upload_blob` `media.rs:305` (extractor `AuthenticatedUpload::from_request_parts` `media.rs:142`) | `router.rs:39` | Blossom + `X-SHA-256` + membership + per-pod rate/concurrency | binary body ≤ `max(max_image, max_video)` = **500 MB** default; headers `Authorization`, `X-SHA-256`, optional `X-Auth-Tag`, `Content-Length` | `BlobDescriptor` JSON (url/sha256/size/mime/uploaded/dim/blurhash/thumb/duration) | 401 (all auth failures collapse to `authentication failed`), 403 membership/scope, 404 unmapped host or unknown route mode, 413, 415, 422, 429 rate/concurrency, 500 |
| PUT | `/media/upload` | same handler, `UploadRouteMode::LegacyMedia` (`media.rs:57-63`) | `router.rs:40` | same | same | same | same, **plus 415 for any non-image** (`media.rs:386-388`) |
| GET | `/media/{sha256_ext}` | `get_blob` `media.rs:604` → `serve_blob_for_tenant` `media.rs:619` | `router.rs:41-44` | **none by default**; Blossom `get` + membership when `require_media_get_auth=true` | path `{hash}` / `{hash}.{ext}` / `{hash}.thumb.jpg`; optional `Range` | 200 streamed body or 206 slice (≤16 MiB, `media.rs:587`) | 401/403 when auth on, 404 bad path or missing sidecar, 416 unsatisfiable range, 500 |
| HEAD | `/media/{sha256_ext}` | `head_blob` `media.rs:798` | `router.rs:41-44` | same as GET | same path | 200 with `content-type`/`content-length`/`accept-ranges`/`cache-control` | 401/403, 404, 500 |

##### 1c. Invites and join policy (`api/invites.rs`)

| Method | Path | Handler (`file:line`) | Route reg. | Auth | Request | Response | Error codes |
|---|---|---|---|---|---|---|---|
| POST | `/api/invites` | `mint_invite` `invites.rs:230` | `router.rs:95` | NIP-98 (strict) + payload tag + role ∈ {owner, admin} | `{ttl_secs?}` (empty body allowed) | `{code, expires_at, url}` | 400 bad JSON, 401 auth/payload-tag/replay, 403 non-owner/admin, 404 unmapped host, 500 role lookup |
| POST | `/api/invites/claim` | `claim_invite` `invites.rs:291` | `router.rs:111` | NIP-98 (strict) + payload tag; **membership gate deliberately skipped** | `{code, policy_receipt?}` | `{status:"joined"\|"already_member", community_id, host, role}` | 400 bad JSON, 401, 403 `invite_invalid`/`invite_expired`/`join_policy_required`, 404, 429 claim limit, 500 |
| POST | `/api/invites/accept-policy` | `accept_policy` `invites.rs:162` | `router.rs:107-110` | **none** | `{code, policy_version, age_confirmed?}` | `{receipt}` | 400 bad JSON / `join_policy_not_accepted`, 404 `join_policy_not_configured` |
| GET | `/api/join-policy` | `join_policy` `invites.rs:75` | `router.rs:96` | **none** | none | `{policy:{terms_markdown, privacy_markdown, age_attestation_required, version}}` or `{}` | 200 only |
| GET | `/api/join-policy/terms` | `join_policy_terms` `invites.rs:95` | `router.rs:99-102` | **none** | none | `text/html` page | 404 `join_policy_not_configured` |
| GET | `/api/join-policy/privacy` | `join_policy_privacy` `invites.rs:104` | `router.rs:103-106` | **none** | none | `text/html` page | 404 |

##### 1d. Deployment-operator control plane (`api/operator.rs`)

All six authenticate via `authorize_operator_request` (`operator.rs:60`): NIP-98 (strict),
expected URL = `config.relay_operator_api_origin` + path + verbatim query (`:73-77`), payload tag
required iff a body is present (`:84`), replay scope `"operator-management"` (`:55`, `:104`), then
the `relay_operator_pubkeys` allowlist (`:88-98`). **The request `Host` is never bound to a tenant.**

| Method | Path | Handler (`file:line`) | Route reg. | Request | Response | Error codes |
|---|---|---|---|---|---|---|
| POST | `/operator/communities` | `provision_community` `operator.rs:149` | `router.rs:74-77` | `{host, initial_owner_pubkey?, create_only?}` | `{community_id, host, status:"created"\|"existed", owner_pubkey?}` | 400 validation, 401, 403 not-operator, 409 `community already exists`/`limit_reached:`, 500 origin unset or persistence |
| GET | `/operator/communities` | `list_owned_communities` `operator.rs:302` | `router.rs:74-77` | `?owner_pubkey=<hex>` | `{owner_pubkey, communities:[{community_id, host, created_at, archived_at}]}` | 400 bad pubkey, 401, 403, 500 |
| POST | `/operator/communities/archive` | `archive_community` `operator.rs:203` | `router.rs:78-81` | `{host, owner_pubkey}` | `{community_id, host, archived_at, status:"archived"}` | 400, 401, 403, 404 not found, 409 deployment community, **503** `propagation:"pending"` (body carries the archived state), 500 |
| POST | `/operator/communities/unarchive` | `unarchive_community` `operator.rs:265` | `router.rs:82-85` | `{host, owner_pubkey}` | `{community_id, host, archived_at:null, status:"active"}` | 400, 401, 403, 404, 500 |
| GET | `/operator/communities/availability` | `community_availability` `operator.rs:468` | `router.rs:86-89` | `?host=<authority>` | `{host, normalized_host, available, community_id}` | 400 bad host, 401, 403, 500 |
| POST | `/operator/communities/transfer` | `transfer_community` `operator.rs:354` | `router.rs:90-93` | `{community_id, new_owner_pubkey, expected_owner_pubkey}` | `{community_id, new_owner_pubkey, status:"transferred"\|"already_owner", previous_owner?}` | 400 bad UUID/pubkey, 401, 403, 404 `NoOwner`, 409 `owner_conflict:`/`limit_reached:`, 500 |

##### 1e. Deployment-admin read API (`api/admin/mod.rs`)

Sub-router built only when `config.admin.is_some()` (i.e. `BUZZ_ADMIN_HOST` set), nested at
`/api/admin/v1` (`router.rs:57-59`, `admin/mod.rs:28-41`). Body limit **1024 bytes**
(`admin/mod.rs:39`). Only this sub-router carries `security_headers` (`admin/mod.rs:38`, `:43-61`).

| Method | Path | Handler (`file:line`) | Auth | Request | Response | Error codes |
|---|---|---|---|---|---|---|
| GET | `/api/admin/v1/reports` | `reports` `admin/mod.rs:93` | Admin host | `?communityId&status&reportType&targetKind&before&after&limit` (camelCase, `deny_unknown_fields`) | `Vec<AdminReport>` | 400 `invalid_limit`/`invalid_status`/`invalid_target_kind`, 403 `forbidden`, 404 (admin unconfigured), 500 |
| GET | `/api/admin/v1/reports/{id}` | `report_detail` `admin/mod.rs:125` | Admin host | UUID path | `AdminReport` | 403, 404 `not_found`, 500 |
| GET | `/api/admin/v1/feedback` | `feedback` `admin/mod.rs:151` | Admin host | none (**hard-coded limit 100**, `:155`) | `Vec<FeedbackSummary>` | 403, 404, 500 |
| GET | `/api/admin/v1/feedback/{id}` | `feedback_detail` `admin/mod.rs:177` | Admin host | UUID path | `AdminFeedback` | 403, 404, 500 |
| GET | `/api/admin/v1/feedback/{id}/attachments/{sha256}` | `feedback_attachment` `admin/mod.rs:191` | Admin host | UUID + 64-hex path | raw blob (streamed, via `media::serve_blob_for_tenant`) | 403, 404 (all provenance failures collapse here), 405 for write verbs, 500 |

Error envelope differs from the rest of the surface:
`{"error":{"code","message","requestId"}}` (`admin/error.rs:16-28`, `:60-77`).

##### 1f. NIP-05 (`api/nip05.rs`)

| Method | Path | Handler | Route reg. | Auth | Request | Response | Error codes |
|---|---|---|---|---|---|---|---|
| GET | `/.well-known/nostr.json` | `nostr_nip05` `nip05.rs:26` | `router.rs:65` | **none** | `?name=<local>` | `{names:{name:hex}, relays:{hex:[ws url]}}` or `{names:{},relays:{}}` + `Access-Control-Allow-Origin: *` (`nip05.rs:65-69`) | **always 200** — misses, unmapped host, and absent `name` all return the empty shape (`nip05.rs:59-62`) |

##### 1g. Mesh demo (`api/mesh_demo.rs`)

| Method | Path | Handler | Route reg. | Auth | Request | Response | Error codes |
|---|---|---|---|---|---|---|---|
| POST | `/_mesh/demo/echo` | `demo_echo` `mesh_demo.rs:58` | `router.rs:123` | **none** (gated on `BUZZ_MESH=on` **and** `BUZZ_MESH_DEMO_ECHO=on`) | `{community_id, session_id, payload}` — **community from the body, not the Host** | `{outcome:"owned", generation, owner_runtime_id}` or `{outcome:"forwarded", …, echoed_payload}` | 404 when either flag is off, 502 send/recv/join/goodbye/closed, 504 echo timeout (10 s, `:44`), **400/422 from the `Json` extractor before the 404 gate** |

##### 1h. Adjacent routes in the same app router (out of this module group, listed for completeness)

| Method | Path | Route reg. | Auth |
|---|---|---|---|
| GET | `/` (NIP-11 or WS upgrade) | `router.rs:63` | none for NIP-11; tenant bound before upgrade (`:280-296`) |
| GET | `/info` | `router.rs:64` | none |
| GET | `/health`, `/_liveness`, `/_readiness` | `router.rs:67-69` | none |
| GET | `/huddle/{channel_id}/audio` | `router.rs:125-128` | NIP-42-style, `audio/handler.rs:219` uses `bridge::nip42_expected_relay_url` |
| GET | `/git/{owner}/{repo}/info/refs`; POST `…/git-upload-pack`, `…/git-receive-pack` | `git/transport.rs:1760-1762` | git credential / membership (`git/transport.rs:211`) |
| POST | `/internal/git/policy` | `git/mod.rs:62` | HMAC + `require_localhost` middleware (`git/mod.rs:38-50`) |
| — | SPA fallback: `/assets/*`, `/invite/{code}`, `/`, `/repos*` | `router.rs:158-186` | none |
| GET | `/_liveness`, `/_readiness`, `/_status`, `/_mesh` on the **health port** (default 8080) | `router.rs:225-231` | **none** |

**Total app-router endpoints (method × path): 40.** Of those, **26 belong to this module group**
(7 bridge + 4 media + 6 invites/policy + 6 operator + 5 admin + 1 NIP-05 + 1 mesh demo = 30 rows;
minus the 4 non-group rows counted above). Counting only the assigned files: **30 method×path pairs**.

---

#### 2. Handlers never routed / dead entry points

| Item | `file:line` | Finding |
|---|---|---|
| `api/events.rs` (whole file, 5 lines) | `api/events.rs:1-5` | **100% dead.** Re-exports `submit_event`/`query_events`/`count_events` "for backward compatibility with router.rs" — but `router.rs:71-73` calls `api::bridge::*` directly. Repo-wide grep for `api::events`, `events::submit_event`, `events::query_events`, `events::count_events` returns **zero** hits outside the file itself. |
| `webhook_secret::strip_secret` | `webhook_secret.rs:57` | **Zero production callers** (only `webhook_secret.rs:128`, `:136` in its own tests). Its doc says "Use this before returning a definition to API callers — the secret must never be embedded in a response body." Nothing in the HTTP surface calls it. |
| `HttpAuthMethod::DevPubkey` | `handlers/ingest.rs:58` | **Zero constructors repo-wide.** `bridge.rs:830` hardcodes `HttpAuthMethod::Nip98` even when the request authenticated via the unsigned `X-Pubkey` header. |
| `IngestAuth::Http { auth_method }` | `bridge.rs:830` | Write-only: no code reads the field (only 3 construction sites — `bridge.rs:830` plus two in `ingest.rs` tests). |
| `not_found`'s `#[allow(dead_code)]` | `api/mod.rs:28` | Stale attribute — `not_found` **is** used at `bridge.rs:1803` and `:1792`. |
| `relay_members::check_relay_membership` / `MembershipDecision` | `api/mod.rs:61`, `:46` | Only caller is `enforce_relay_membership` in the same module (`api/mod.rs:130`). The transport-neutral enum has one consumer, so the abstraction currently buys nothing. |
| `relay_url_for_tenant_host` | `nip05.rs:105` | Single caller, same file (`nip05.rs:55`). Fine, but `pub(crate)` is wider than needed. |
| `POST /api/messages/{id}/reactions` | — | **Confirmed absent** from `router.rs:32-190` (and from `git`/`admin`/`media` sub-routers). `buzz-workflow`'s `add_reaction_impl` POSTs there (`buzz-workflow/src/executor.rs:889`), so that workflow action cannot succeed against this relay. |
| `/api/presence` | — | Referenced by `ARCHITECTURE.md:824` as an existing endpoint. **No such route exists.** `mobile/test/features/profile/presence_cache_provider_test.dart:13` records that it "has been removed". Presence is now synthesized inside `POST /query` (`bridge.rs:1920-1985`). |

---

#### 3. Routes whose auth differs from expectation

| Route | Expected (docs/name) | Actual | `file:line` |
|---|---|---|---|
| `POST /events`, `/query`, `/count` | "NIP-98 auth" per the module doc header | NIP-98 **or** unsigned `X-Pubkey` when `require_auth_token=false` (default) | `bridge.rs:1-4` vs `:118-127`, `config.rs:475-477` |
| `GET /moderation/*` | "NIP-98 auth + mod-authz gate" (`router.rs:112`) | Same `X-Pubkey` fallback applies — `authorize_moderation_read` passes `state.config.require_auth_token` | `bridge.rs:2051` |
| `GET /api/admin/v1/*` | "deployment-admin API" | **No credential.** Host equality + optional-`Origin` check only; a missing `Origin` passes | `admin/auth.rs:16-35` |
| `POST /api/invites/accept-policy` | sits beside two NIP-98 invite routes | **No auth at all**, and no rate limit — an open HMAC-minting oracle | `invites.rs:162-190` |
| `POST /_mesh/demo/echo` | "404 (not 403) so a non-demo deployment is indistinguishable from one without the route" | `Json<DemoEchoRequest>` is a `FromRequest` extractor evaluated **before** the flag gate, so a malformed body returns 400/422 on a mesh-off relay and reveals the route | `mesh_demo.rs:71-73` vs `:60-62` |
| `GET/HEAD /media/{sha256_ext}` | Blossom read auth implied by `verify_blossom_get_auth` existing | Unauthenticated by default (`require_media_get_auth` default **false**) | `media.rs:489-514`, `config.rs:682-689` |
| `GET /operator/communities` etc. | tenant-scoped like every other route | Deliberately **not** host-bound; authority is `relay_operator_api_origin` + allowlist | `operator.rs:57-60`, `:69-77` |
| `POST /operator/*` when unconfigured | 403/404 | **500** `internal_error("operator API origin is not configured")` | `operator.rs:69-72` |

---

#### 4. Delta vs `AGENTS.md`'s "deliberately narrow" HTTP surface

`AGENTS.md` lists the sanctioned set as: NIP-11/NIP-05, `POST /events|/query|/count`,
workflow webhooks `/hooks/{id}`, Blossom media, git smart HTTP, git policy hooks, health probes.

**14 routed endpoints in this module group fall outside that list:**

| # | Endpoint | `file:line` |
|---|---|---|
| 1 | `GET /operator/communities` | `router.rs:74-77` |
| 2 | `POST /operator/communities` | `router.rs:74-77` |
| 3 | `POST /operator/communities/archive` | `router.rs:78-81` |
| 4 | `POST /operator/communities/unarchive` | `router.rs:82-85` |
| 5 | `GET /operator/communities/availability` | `router.rs:86-89` |
| 6 | `POST /operator/communities/transfer` | `router.rs:90-93` |
| 7 | `POST /api/invites` | `router.rs:95` |
| 8 | `POST /api/invites/claim` | `router.rs:111` |
| 9 | `POST /api/invites/accept-policy` | `router.rs:107-110` |
| 10 | `GET /api/join-policy` (+ `/terms`, `/privacy` = 3 routes) | `router.rs:96`, `:99-106` |
| 11 | `GET /moderation/reports` | `router.rs:113` |
| 12 | `GET /moderation/audit` | `router.rs:114` |
| 13 | `GET /moderation/restricted` | `router.rs:115-118` |
| 14 | `POST /_mesh/demo/echo` | `router.rs:123` |

Plus, outside this group but also outside the documented set: `GET /huddle/{channel_id}/audio`
(`router.rs:125-128`), the five `/api/admin/v1/*` routes (`router.rs:57-59`), and `/_status` +
`/_mesh` on the health port (`router.rs:229-230`).

`bridge.rs:1994-2000` self-documents the moderation exception ("Mod-only structured rows … are not
nostr events, so they are served over dedicated NIP-98-authed GET endpoints"), and `operator.rs:3-6`
and `invites.rs:3-6` likewise argue their cases — but `AGENTS.md` was never updated.

**Delta vs `ARCHITECTURE.md:610-628`** (its own endpoint table): omits `PUT /upload`, all 6
operator routes, all 6 invite/policy routes, all 3 moderation routes, `/_mesh/demo/echo`,
`/huddle/{channel_id}/audio`, all 5 admin routes, and `/_status` + `/_mesh`. It also states
`PUT /media/upload` has a "50 MB limit" — the actual layer is
`max(max_image_bytes, max_video_bytes)` = **500 MB** by default (`router.rs:33-36`,
`config.rs:657-672`).

---

#### 5. Bridge `/query` extension grammar (undocumented outside code)

`nostr::Filter` silently drops unknown fields, so the bridge does a two-pass parse
(`bridge.rs:969-976`) and reads extensions off the raw JSON.

| Field (aliases) | Type | Effect | `file:line` |
|---|---|---|---|
| `before_id` | 64-hex | keyset cursor; requires `until`; malformed ⇒ **400** | `bridge.rs:277-291`, `:1198-1216` |
| `top_level` | literal `true` only | channel-window read model; requires exactly one `#h` | `bridge.rs:295-297`, `:404-581` |
| `include_aux` | bool | 2-hop reaction/deletion/edit closure (kinds at `:379-390`) | `bridge.rs:483-521` |
| `include_summaries` | bool | relay-signed kind:39005 overlays | `bridge.rs:534-556` |
| `depth_limit` | u32 | thread-reply path, needs exactly one `#e` | `bridge.rs:299-303`, `:1112-1183` |
| `thread_cursor` / `threadCursor` + `thread_cursor_id` / `threadCursorId` | i64 + hex | composite thread cursor, encoded BE-i64 ‖ id bytes | `bridge.rs:305-345` |
| `feed_types` | `["mentions","needs_action","activity"\|"agent_activity"]` | server-side feed queries, limit ≤100 | `bridge.rs:347-358`, `:1029-1109` |
| `search_mode` / `searchMode` | `"prefix"` else full-text | NIP-50 mode | `bridge.rs:368-378` |
| `page` / `search_page` / `searchPage` | u32 ≥1 | FTS page **and** (via `extract_page_offset`) offset paging on non-search queries | `bridge.rs:380-388`, `:390-410`, `:1218-1229` |

Response-side synthetic kinds emitted by the bridge, relay-signed with `state.relay_keypair`:
kind:39005 thread summary and exactly one kind:39006 window bounds per window response
(`bridge.rs:523-576`), and kind:20001 presence (`bridge.rs:1966-1977`).

Caps: `BRIDGE_FEED_MAX_LIMIT=100` (`:270`), `BRIDGE_THREAD_MAX_LIMIT=500` (`:271`),
`BRIDGE_WINDOW_DEFAULT_LIMIT=50` / `BRIDGE_WINDOW_MAX_LIMIT=200` (`:374-375`),
FTS `limit.min(500)` (`:1665`), aux hop `limit=1000` (`:492`), `MODERATION_READ_LIMIT=500` (`:2059`).


## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: API Surface

#### 1. HTTP endpoints — exact count: 4

Three git smart-HTTP routes (`transport.rs:1756-1765`) plus one internal policy route (`mod.rs:60-66`). Both routers are merged into the single relay router at `router.rs:48-50`, `:137-138`.

| # | Method + path | Handler | Auth | Body limit | Response CT |
|---|---|---|---|---|---|
| 1 | `GET /git/{owner}/{repo}/info/refs?service=…` | `info_refs` (`transport.rs:539`) | `GitAuth` extractor | `git_max_pack_bytes` | `application/x-git-{service}-advertisement` (`:568-571`, `:717`) |
| 2 | `POST /git/{owner}/{repo}/git-upload-pack` | `upload_pack` (`transport.rs:786`) | `GitAuth` | `git_max_pack_bytes` | `application/x-git-upload-pack-result` (`:825`) |
| 3 | `POST /git/{owner}/{repo}/git-receive-pack` | `receive_pack` (`transport.rs:858`) | `GitAuth` | `git_max_pack_bytes` | `application/x-git-receive-pack-result` (`:1500-1503`) |
| 4 | `POST /internal/git/policy` | `policy::hook_policy_check` (`policy.rs:173`) | HMAC-SHA256 in body + `require_localhost` middleware | **1 MiB** (`mod.rs:63`) | `application/json` |

Both body limits come from `tower_http::limit::RequestBodyLimitLayer`. The git limit is read once at router-construction time and cast: `state.config.git_max_pack_bytes as usize` (`transport.rs:1757`). No layer other than the body limit is applied inside `git_router`; the shared `track_metrics` / `TraceLayer` / CORS layers are applied over the merged router (`router.rs:187-190`).

##### Route-shape notes

- The git routes use axum 0.8 `{param}` capture syntax, so `{repo}` cannot contain `/`. `owner`/`repo` are re-validated by `validate_repo_id` in every handler (`transport.rs:266-297`, called at `:549`, `:788`, `:860`).
- `info/refs` is the only route with a required query parameter; `InfoRefsQuery { service: String }` (`transport.rs:358-361`) is a mandatory field, so a missing `?service=` is a 400 from axum's `Query` rejection before the handler body runs.
- `git_router` does **not** register `HEAD`, `OPTIONS`, or the dumb-HTTP paths (`objects/info/packs`, `objects/<2>/<38>`). Dumb HTTP is entirely absent.

#### 2. Request/response framing

##### `info/refs`

Two code paths produce byte-different-but-protocol-equivalent output.

| Path | Trigger | Body layout | Permit | Subprocess |
|---|---|---|---|---|
| Fast path | `service == git-upload-pack` **and** `fast_path_eligible(manifest)` (`transport.rs:552-577`) | `pkt(# service=git-upload-pack\n)` `0000` `pkt(<oid> HEAD\0<caps>\n)` `pkt(<oid> <ref>\n)`× `0000` (`transport.rs:497-537`) | **none acquired** | none |
| Subprocess | receive-pack advertisement, or upload-pack on a repo with any `refs/tags/*` | `pkt(# service=<svc>\n)` `0000` ‖ raw `git <svc> --stateless-rpc --advertise-refs` stdout (`transport.rs:712-721`) | yes (`:598`) | yes (`:645-653`) |

Fast-path capability offer is a fixed string (`transport.rs:484-494`): `multi_ack thin-pack side-band side-band-64k ofs-delta shallow deepen-since deepen-not deepen-relative no-progress include-tag multi_ack_detailed no-done symref=HEAD:<head> object-format=<sha1|sha256> agent=buzz-git`. `object-format` is derived from the HEAD oid width (`transport.rs:473-479`), which is the only place SHA-256 is inferred rather than rejected.

pkt-line encoding is centralized in `pkt_line` (`transport.rs:437-454`). It refuses payloads > `0xffff - 4` by emitting an empty `0004` frame and logging at `error` — a deliberately non-panicking degradation (pinned `transport.rs:2231-2237`).

Both paths set `Cache-Control: no-cache` (`transport.rs:566`, `:719`).

##### `git-upload-pack`

- Request body: pkt-line want/have negotiation, optionally `Content-Encoding: gzip`. Transparently inflated by `decode_git_request_body` (`transport.rs:745-784`) with a **decoded** cap of `UPLOAD_PACK_MAX_DECODED_BYTES` = 64 MiB (`transport.rs:59`, applied `:789`).
- Response: **streamed**. `stream_git_read` (`transport.rs:1414-1498`) pipes child stdout into `Body::from_stream` through three wrappers: `TimedByteStream` (300 s deadline, byte/duration histograms — `:1282-1390`), `StreamingGit` (owns `Child` + `HydratedRepo` + stdin pump; kills the child on timeout — `:1262-1332`), `GitPermitStream` (holds the semaphore permit to EOF — `:1293-1310`, pinned `:1924-1941`).
- Status is committed to 200 before any pack byte exists, so post-head errors surface in-band per git's smart-HTTP contract (`transport.rs:1405-1412`).
- `extra_args` is `&[]` at the only call site (`transport.rs:824`) — the parameter is unused in production.

##### `git-receive-pack`

- Request body decoded with a cap of `git_max_pack_bytes` (`transport.rs:861`) — i.e. the *same* number bounds compressed (router layer) and decoded (decode seam) bytes.
- Response: **buffered**, capped at `RECEIVE_PACK_MAX_OUTPUT_BYTES` = 1 MiB (`transport.rs:50`, enforced `:1092-1108`). Buffering is the structural fence: `run_git_at` returns an owned `PackOutput` (`transport.rs:971-991`), never a `Response`, so `finalize_push` can sequence the CAS before a 2xx exists (`transport.rs:1540-1547`).

##### `/internal/git/policy`

Request JSON is `HookCallbackRequest` (`policy.rs:53-86`); response is `HookCallbackResponse` (`policy.rs:87-97`) with 200 on allow and 403 on every denial or error. Deny bodies are sometimes JSON (`policy.rs:407-413`) and sometimes bare text (`:176-233`, `:274-281`, …) — the hook only reads the status code and echoes the body to stderr (`hook.rs:141-146`).

#### 3. Status-code matrix

| Status | Condition | Site |
|---|---|---|
| 200 | all success paths | `transport.rs:566`, `:1487`, `:1501` |
| 400 | invalid `service` value | `transport.rs:548` |
| 400 | `owner` not 64 lowercase hex | `transport.rs:273` |
| 400 | repo name empty / >64 / leading `.` / contains `..` / bad alphabet | `transport.rs:288` |
| 400 | path matches no git endpoint shape (NIP-98 `u` derivation) | `transport.rs:140` |
| 400 | `CasError::ManifestInvalid` | `transport.rs:1626` |
| 401 | missing `Authorization` header (+ `WWW-Authenticate: Nostr realm="buzz", method=…`) | `transport.rs:88-97` |
| 401 | header lacks `Nostr ` prefix | `transport.rs:99-108` |
| 401 | bad base64 / bad utf-8 / unparseable event / NIP-98 verify failure | `transport.rs:110-115`, `:185-190`, `:200-202` |
| 403 | relay membership denied | `transport.rs:221` |
| 403 | every policy-endpoint rejection (structural, HMAC, TTL, missing 30617, archived channel, non-member, unknown role, DB error, protection denial) | `policy.rs:176-413` |
| 403 | non-loopback (or `ConnectInfo`-absent) request to `/internal/git/policy` | `mod.rs:49` |
| 404 | Host does not resolve to a community (`bind_community` error) — deliberately reused as "repository not found" | `transport.rs:130` |
| 404 | pointer absent | `transport.rs:579`, `:619`, `:812` |
| 409 | `CasError::Conflict` (CAS 412) | `transport.rs:1609` |
| 413 | `HydrateError::ResourceLimit` | `transport.rs:344-349` |
| 413 | advertisement stdout > 4 MiB | `transport.rs:691-698` |
| 413 | subprocess stdout > `max_output_bytes` | `transport.rs:1098-1106` |
| 413 | `CasError::ResourceLimit` | `transport.rs:1639` |
| 500 | tempfile/spawn/wait/read failures, non-zero `--advertise-refs`, hook install failure, other `CasError` | `transport.rs:630-700`, `:908`, `:1656` |
| 503 | git semaphore exhausted (+ `Retry-After: 5`) | `transport.rs:326-337` |
| 504 | `INFO_REFS_TIMEOUT` 120 s or `PACK_OPS_TIMEOUT` 300 s exceeded | `transport.rs:667`, `:1076` |

Body-limit overflow (compressed bytes past `git_max_pack_bytes`) is produced by `RequestBodyLimitLayer` itself, not by this module.

#### 4. Public Rust items and their callers

"External" = outside `crates/buzz-relay/src/api/git/`. Verified by workspace grep across `crates/**`.

##### `transport.rs`

| Item | Vis | External callers | Notes |
|---|---|---|---|
| `git_router` | `pub` (re-exported `mod.rs:35`) | `router.rs:48` | 1 |
| `info_refs`, `upload_pack`, `receive_pack` | `pub` | 0 direct — reached via routes `transport.rs:1760-1762` | axum handlers |
| `GitAuth { pubkey, tenant }` | `pub` | 0 | extractor, used by the three handlers |
| `InfoRefsQuery`, `GitRepoParams` | `pub` | 0 | fields are private, so unusable externally |
| `harden_git_env` | `pub(crate)` | 0 outside module; 4 in-module (`cas_publish.rs:290`, `:344`, `:412`, `:534`, `:614`, `:717`, and test `:1534`) | crate-visible but effectively module-private |
| `PackOutput`, `PushContext` | `pub(crate)` | 0 | fence types |

##### `store.rs` — entire module carries `#![allow(dead_code)]` (`store.rs:25`)

| Item | Vis | Callers |
|---|---|---|
| `GitStore::new` | `pub` | `state.rs:694-701` (1 prod), tests |
| `GitStore::content_key` | `pub` | in-module only (`store.rs:260`, probe `:726`) — **zero external callers** |
| `GitStore::idx_key_for_pack_digest` | `pub` | in-module only (`:295`, `:328`) |
| `put_pack` | `pub` | `cas_publish.rs:826`, `:1150`; probe `store.rs:594` |
| `put_manifest` | `pub` | `cas_publish.rs:1194`, `handlers/side_effects.rs:2643` |
| `put_idx` | `pub` | `cas_publish.rs:429`, `:830` |
| `get_idx` | `pub` | `hydrate.rs:389` |
| `get` | `pub` | in-module (`:391`, `:428`) |
| `get_verified` | `pub` | `cas_publish.rs:1315`; probe |
| `get_verified_limited` | `pub` | `hydrate.rs:277` |
| `get_limited` | `pub` | in-module (`:329`, `:391`) |
| `get_pointer` | `pub` | `hydrate.rs:251`, `cas_publish.rs:1297`, `handlers/side_effects.rs:2665`, `:2715` |
| `put_pointer` | `pub` | `cas_publish.rs:1235`, `handlers/side_effects.rs:2653`, probe |
| `run_conformance_probe` | `pub` | `main.rs:492` (fatal gate) |
| `ETag`, `Precond`, `CasOutcome`, `StoreError`, `ProbeConfig` | `pub` | `Precond`/`CasOutcome` used by `side_effects.rs:2622`; `ProbeConfig` by `main.rs:481` |
| `ProbeReport`, `ProbeFailure` | `pub` | in-module only; `ProbeReport` fields read at `main.rs:496-499` |

##### `manifest.rs`

| Item | Vis | Callers |
|---|---|---|
| `Manifest` + `validate` / `canonical_bytes` / `from_bytes` | `pub` | `cas_publish.rs`, `hydrate.rs`, `transport.rs`, `handlers/side_effects.rs:2621-2645` |
| `pointer_key` | `pub` | `cas_publish.rs:1023`, `hydrate.rs:247`, `handlers/side_effects.rs:2650`, `:2712` |
| `MANIFEST_VERSION` | `pub` | `handlers/side_effects.rs:2629` |
| `MAX_MANIFEST_PACKS`, `PACK_COMPACTION_THRESHOLD`, `MAX_MANIFEST_REFS` | `pub` | in-module only (`cas_publish.rs`, `manifest.rs`) — **zero external callers** |
| `is_safe_refname`, `is_hex_oid` | `pub` | `hydrate.rs`, `transport.rs:402-419`, `manifest.rs:210-232` — **zero external callers** |
| `is_pack_key` | `pub` | `manifest.rs:234` only |
| `ManifestError` | `pub` | `cas_publish.rs:139`, `hydrate.rs:101` |

##### `hydrate.rs`

| Item | Vis | Callers |
|---|---|---|
| `hydrate_for_read` | `pub` | `transport.rs:600`, `:792`; test `cas_publish.rs:1786` |
| `hydrate_for_write` | `pub` | `transport.rs:878`; test `cas_publish.rs:1712` |
| `load_manifest_for_read` | `pub` | `transport.rs:553` |
| `HydratedRepo` + `path` / `hydrated_bytes` / `hydrated_packs` | `pub` | `transport.rs`, `cas_publish.rs:1587-1591`; `hydrated_packs` used only by the metric at `hydrate.rs:136` |
| `HydrationOptions`, `HydrateError` | `pub` | `transport.rs`, `pack_cache.rs` |
| `get_verified_limited`, `install_or_generate_idx` | `pub(super)` | `pack_cache.rs:288`, `:290` |

##### `cas_publish.rs`

| Item | Vis | Callers |
|---|---|---|
| `cas_publish` | `pub` | `transport.rs:1582` only |
| `CasError` (7 variants) | `pub` | `transport.rs:1601-1656` |
| `PublishLimits` | `pub` | `transport.rs:1588-1592` |
| `CasSuccess` | `pub` | `transport.rs:1582`, `:1685-1690` |
| `ParentState` + `fresh` / `from_loaded` | `pub` | `hydrate.rs:212`, `:241`; `transport.rs:878` |

##### `manifest_event.rs`

| Item | Vis | Callers |
|---|---|---|
| `build_ref_state_event` | `pub` | `transport.rs:1690`, `handlers/side_effects.rs:2749` |
| `RefStateInputs` | `pub` | `transport.rs:1685`, `handlers/side_effects.rs:2743` |
| `BuildError` | `pub` | returned only; matched in tests (`:369-380`) |

##### `hook.rs` / `policy.rs` / `mod.rs`

| Item | Vis | Callers |
|---|---|---|
| `install_hook` | `pub` | `transport.rs:906` only |
| `hook_policy_check` | `pub` | route `mod.rs:62` |
| `HookCallbackRequest`, `HookRefUpdate`, `HookCallbackResponse`, `DenialResponse` | `pub` | deserialized/serialized by axum; `HookRefUpdate` also used by `generate_hook_hmac` |
| `generate_hook_hmac` | `pub` | **ZERO production callers.** Only `policy.rs:581`, `:626`, `:720` — all `#[cfg(test)]`. Test-only public API. |
| `git_policy_router` | `pub` | `router.rs:50` |
| `require_localhost` | private | `mod.rs:64` |

##### Dead-code summary (zero production callers anywhere)

1. `policy::generate_hook_hmac` (`policy.rs:419-437`) — the *only* item with zero non-test callers.
2. `stream_git_read`'s `extra_args` parameter (`transport.rs:1418`) — always `&[]` (`:824`).
3. `GitStore::content_key` / `idx_key_for_pack_digest` / `get` / `get_limited` are `pub` but reachable only in-module; the module-wide `#![allow(dead_code)]` (`store.rs:25`) suppresses any future warning if the last caller disappears.
4. `MAX_MANIFEST_PACKS`, `PACK_COMPACTION_THRESHOLD`, `MAX_MANIFEST_REFS`, `is_safe_refname`, `is_hex_oid`, `is_pack_key`, `HydratedRepo::hydrated_packs`, `ProbeReport`, `ProbeFailure` — `pub` with no external caller (over-exported surface, not dead).

#### 5. Consumers outside the relay

| Consumer | Surface used | Site |
|---|---|---|
`web/` repo browser | isomorphic-git `clone`/`fetch` with `depth: 1, singleBranch, noTags` against `/git/{owner}/{repo}.git`; NIP-98 header from `makeNip98AuthHeader(<repo-root>.git, "GET")` | `web/src/features/repos/git-client.ts:41-113` |
| `web/` repo list / refs | `POST /query` for kinds 30617 and 30618 — not this module's HTTP surface | `web/src/features/repos/use-repos.ts:66`, `use-repo-refs.ts:54-57` |
| `git-credential-nostr` | supplies the NIP-98 `Authorization` header to the git CLI | `crates/buzz-test-client/tests/e2e_git.rs:63-88` |
| `e2e_git.rs` | drives all three routes through real `git`, and reads the S3 pointer directly | `crates/buzz-test-client/tests/e2e_git.rs:195-475` |

#### 6. Delta: the NIP-98 `u` tag is repo-root, `.git`-sensitive

`git_expected_url` (`transport.rs:235-253`) derives the expected `u` by stripping the endpoint suffix, keeping whatever the client sent — **including `.git`**. So a token signed for `…/git/o/r` will not authenticate a request to `…/git/o/r.git` and vice versa, even though both address the same pointer (`manifest.rs:182`). The web client deliberately signs the `.git` form (`web/src/features/repos/git-client.ts:41-50`). The scheme comes from `config.relay_url` (`wss://` ⇒ `https`, else `http`) but the **host always comes from the resolved tenant**, pinned by `transport.rs:2067-2141`.

The `/info/refs` branch uses `split_once("/info/refs")` (`transport.rs:243-244`), so `…/info/refs/anything` also derives a valid prefix; harmless because the route itself would not match.


## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: API Surface

---

#### 1. Event kinds owned by this module

**14 inbound kinds** across 6 handlers. Counts verified against `buzz-core/src/kind.rs` and the ingest dispatch chain.

| Kind | Owner | Routed at | Stored as an event? | Direct DB effect |
|---|---|---|---|---|
| 1984 | `report::handle_report_event` | `ingest.rs:1586-1596` | **no** | `moderation_reports` insert |
| 9030 | `relay_admin::handle_relay_admin_event` | `ingest.rs:1834-1844` | **no** | `relay_members` insert |
| 9031 | same | same | **no** | `relay_members` delete |
| 9032 | same | same | **no** | `relay_members` role update |
| 9033 | same | same | **no** | `communities.icon` update |
| 9035 | `identity_archive::handle_identity_archive_event` | `ingest.rs:1939-1945` | **yes** (falls through) | `archived_identities` insert |
| 9036 | same | same | **yes** (falls through) | `archived_identities` delete |
| 9040 | `moderation_commands::handle_moderation_command` | `ingest.rs:1600-1613` | **no** | `community_bans` + `moderation_actions` |
| 9041 | same | same | **no** | `community_bans` + `moderation_actions` |
| 9042 | same | same | **no** | `community_bans` + `moderation_actions` |
| 9043 | same | same | **no** | `community_bans` + `moderation_actions` |
| 9044 | same | same | **no** | `moderation_reports` + `moderation_actions` |
| 30350 | `push_lease::accept` | `ingest.rs:2155-2199` | **yes** (atomic with lease) | `push_leases` + source event |
| 42000 | `product_feedback::handle` | `ingest.rs:1564-1578` | **no** | `product_feedback` insert |

**Outbound (relay-signed) kinds emitted by this module:**

| Kind | Emitter | Site |
|---|---|---|
| 0 (`Metadata`) | `moderation_notices::publish_moderation_profile` — `"{host} Moderation"` | `moderation_notices.rs:186-213` |
| 9 (`KIND_STREAM_MESSAGE`) | moderation notice DM | `moderation_notices.rs:160-178` |
| 9 (`KIND_STREAM_MESSAGE`) | workflow `send_message` | `workflow_sink.rs:302-352` |
| 39000 | DM discovery via `emit_group_discovery_events` | `moderation_notices.rs:155` |
| 8000-series NIP-43 member add/remove + membership list | `relay_admin.rs:214-220`, `:274-279`, `:334-336` (delegated to `side_effects.rs:2826/2923/2932`) | |
| 8002 / 8003 / 13535 (NIP-IA) | `identity_archive.rs:104-136` (delegated to `side_effects.rs:3206/3234/3008`) | |
| `HttpAuth` (27235, NIP-98) | outbound gateway auth header, never stored | `push_runtime.rs:551-565` |

##### Routing-order facts (verified)

- **9040–9044 are dispatched *before* the ban/timeout write-block gate** (`ingest.rs:1574-1587` vs the gate at `:1613-1650`), so a timed-out admin can lift a timeout. The gate explicitly exempts both moderation-command and relay-admin kinds (`ingest.rs:1613`).
- The handler re-checks the durable ban itself at `moderation_commands.rs:103-108` → `ensure_actor_not_banned` (`:135-142`). **`relay_admin.rs` performs no such re-check** — grep for `restriction`/`banned` in `relay_admin.rs` returns zero hits. A banned owner/admin whose live disconnect was missed can still issue 9030–9033.
- All five moderation kinds are listed in `is_global_only_kind` (`ingest.rs:429-433`), as are 9030–9033 (`:436-439`), 9035/9036 (`:445-446`) and 30350 (`:450`). 1984 and 42000 are **not** in that list.
- 9035/9036 deliberately fall through to normal storage so the 8002/8003 delta's `["e", request_id]` resolves (`ingest.rs:1935-1938`).

##### Required scopes (`required_scope_for_kind`)

| Kinds | Scope | Site |
|---|---|---|
| 1984, 42000 | `MessagesWrite` | `ingest.rs:212` |
| 9040–9044 | `MessagesWrite` | `ingest.rs:216` |
| 9030–9033 | `AdminUsers` | `ingest.rs:251-256` |
| 9035, 9036 | `UsersWrite` | `ingest.rs:266` |
| 30350 | (author-only kind) | `kind.rs:120` |

> **Contract delta:** `moderation_commands.rs:50` pins "reject channel-scoped API tokens" for 9040–9044. Ingest rejects channel-scoped tokens only for relay-admin kinds (`ingest.rs:1512-1516`) and leave requests (`:1520-1523`); the generic global-event token gate (`ingest.rs:1721-1724`) sits *after* the moderation dispatch's early `return` at `:1582-1586` and is therefore unreachable for 9040–9044. Combined with `Scope::MessagesWrite`, a legacy channel-scoped WS API token held by a community admin can issue a community-wide ban.

---

#### 2. Exact accept/reject strings

##### 2.1 `moderation_commands` (9040–9044)

Three prefix helpers: `invalid()` → `"invalid: {msg}"` (`:552-554`), `error()` → `"error: {msg}"` (`:556-558`), `authz_denial()` → `"restricted: {e}"` (`:548-550`). Pinned by a unit test (`:669-680`).

| Reject string | Site |
|---|---|
| `blocked: you are banned from this community` (no prefix) | `:139` |
| `invalid: event timestamp out of range: created_at=…, now=…, delta=…s (max ±120s)` | `:117-120` |
| `invalid: unexpected moderation command kind: {other}` | `:130-132` |
| `invalid: missing or invalid p tag` | `:147`, `:228`, `:264`, `:331` |
| `invalid: timeout requires an expiration tag` | `:271` |
| `invalid: invalid expiration tag: {raw}` / `invalid: expiration out of range: {secs}` | `:598`, `:602` |
| `invalid: member is not banned` | `:252` |
| `invalid: member is not timed out` | `:355` |
| `invalid: missing or invalid report tag (expect 64-hex event id)` | `:368` |
| `invalid: missing status tag` / `invalid: missing action tag` | `:369`, `:370` |
| `invalid: invalid status: {status} (expect resolved\|dismissed)` | `:381-383` |
| `invalid: invalid action: {action} (expect delete\|kick\|ban\|timeout\|dismiss\|escalate)` | `:389-391` |
| `` invalid: action `dismiss` pairs only with status `dismissed` `` | `:394-396` |
| `invalid: report not found in this community` | `:417` |
| `invalid: report is not open (already resolved or dismissed)` | `:427-429`, `:471-473` |
| `restricted: moderator access required` | via `:549` + `moderation_authz.rs:178` |
| `restricted: an admin cannot ban or time out a community owner or fellow admin` | via `:549` + `moderation_authz.rs:167` |
| `error: database error: {e}` | `:174`, `:250`, `:292`, `:353`, `:416`, `:469` |
| `error: database error checking restriction state: {e}` | `:108` |
| `error: failed to write audit row: {e}` | `:544` |

Accept: `Ok(())`; ingest converts to `IngestResult { accepted: true, message: "" }` (`ingest.rs:1609-1612`).

##### 2.2 `relay_admin` (9030–9033)

Returns **unprefixed** strings; ingest wraps every one as `format!("invalid: {e}")` (`ingest.rs:1837`). This means an authorization failure surfaces to clients as `invalid: actor not authorized: …` rather than `restricted: …` — inconsistent with the moderation and REQ paths.

| Reject string | Site |
|---|---|
| `event timestamp out of range: created_at=…, now=…, delta=…s (max ±120s)` | `:126-129` |
| `database error: {e}` | `:137`, `:201`, `:246`, `:253`, `:311`, `:319`, `:321` |
| `actor not authorized: must be admin or owner` | `:148`, `:177`, `:227` |
| `actor not authorized: must be owner` | `:286` |
| `actor not authorized: only owner can grant admin role` | `:188` |
| `actor not authorized: admins can only remove members` | `:264` |
| `icon contains invalid characters` | `:72` |
| `icon data URL too long: {n} bytes (max 98304)` | `:78-81` |
| `icon must be an http(s) URL or data:image/* URL` | `:84` |
| `icon URL too long: {n} bytes (max 2048)` | `:89-92` |
| `failed to store workspace icon: {e}` | `:162` |
| `missing or invalid p tag` | `:168` |
| `invalid role: use kind:9032 to promote to owner` | `:185` |
| `invalid role: {role}` | `:191`, `:304` |
| `cannot remove yourself` | `:232` |
| `cannot remove the relay owner` | `:258` |
| `member not found: {target_hex}` | `:261`, `:323` |
| `cannot change your own role` | `:291` |
| `cannot set role to owner` | `:301` |
| `cannot change the relay owner's role` | `:322` |
| `missing role tag` | `:295` |
| `unexpected relay admin kind: {other}` | `:340` |

##### 2.3 `report` (1984)

Self-prefixes (unlike relay_admin), so ingest's `IngestError::Rejected` passes them through verbatim (`ingest.rs:1589`).

| String | Site |
|---|---|
| `invalid: report must include a p tag` | `:123` |
| `invalid: report must include exactly one p tag` | `:126` |
| `invalid: report must target only one of e or x` | `:135` |
| `invalid: report must include at most one e tag` | `:138` |
| `invalid: report must include at most one x tag` | `:141` |
| `invalid: report target tag missing report type` | `:202` |
| `invalid: unsupported report type: {value}` | `:207` |
| `invalid: malformed {label}` (labels: `p tag pubkey`, `e tag event id`, `x tag sha256`) | `:213`, `:215`, `:217` |
| `invalid: report target event not found` | `:59` |
| `invalid: report target blob not found` | `:72` |
| `error: database error resolving report target: {e}` | `:58` |
| `error: database error inserting report: {e}` | `:88` |

##### 2.4 `product_feedback` (42000)

| String | Site |
|---|---|
| `invalid: unsupported feedback category` | `:88` |
| `invalid: feedback must include at most one category tag` | `:89` |
| `invalid: feedback body must not be empty` | `:96` |
| `invalid: feedback body exceeds maximum size of 32768 bytes` | `:98-100` |
| `invalid: feedback tags exceed maximum size of 65536 bytes` | `:73-75` |
| `invalid: feedback timestamp is out of range` | `:49` |
| `error: failed to serialize feedback tags: {e}` | `:71` |
| `error: failed to deserialize feedback tags: {e}` | `:77` |
| `error: database error inserting product feedback: {e}` | `:70` (via `:66`) |
| imeta errors | delegated to `crate::api::validate_imeta_tags` / `verify_imeta_blobs` (`:32-35`) |

##### 2.5 `identity_archive` (9035/9036)

Unprefixed; ingest wraps as `invalid: {e}` (`ingest.rs:1943`).

| String | Site |
|---|---|
| `unexpected identity archive kind: {kind}` | `:52` |
| `event timestamp out of range: … (max ±120s)` | `:148-151` |
| `request must include exactly one NIP-70 protected event tag ["-"] (got {n})` | `:158-160` |
| `missing or invalid p tag` | `:57` |
| `replaced-by is not valid on unarchive requests` | `:61` |
| `invalid replaced-by tag` / `invalid replaced-by pubkey` | `:209`, `:216` |
| `replaced-by must differ from target` | `:219` |
| `multiple replaced-by tags` | `:222` |
| `database error: {e}` | `:78`, `:88`, `:242`, `:280` |
| `auth tag must have exactly four elements` / `multiple auth tags` / `missing auth tag` | `:305`, `:308`, `:314` |
| `invalid request auth tag: {e}` | `:262` |
| `request auth owner must equal request signer` | `:264` |
| `request auth time bound not satisfied: created_at {n} >= {bound}` / `<= {bound}` | `:339-341`, `:348-350` |
| `invalid created_at< bound: {b}` / `invalid created_at> bound: {b}` | `:336`, `:345` |
| `target has no live kind:0 profile` | `:284` |
| `live kind:0 author did not match target` | `:288` |
| `invalid live kind:0 auth tag: {e}` | `:292` |
| `live kind:0 no longer attests to request signer` | `:294` |
| `invalid target pubkey: {e}` | `:278`, `:322` |
| `failed to encode auth tag: {e}` | `:317` |
| `invalid auth tag json: {e}` / `auth tag missing conditions` | `:330`, `:332` |

##### 2.6 `push_lease` (30350)

Two-channel error type `AcceptError::{Validation(String), Internal(String)}` (`push_lease.rs:457-461`), mapped by `map_push_accept_error` (`ingest.rs:187-195`): `Validation` → `invalid: {reason}` (400/OK-false), `Internal` → `IngestError::Internal` (500). `From<String>` defaults to `Validation` (`:463-467`).

Validation strings: `push not supported` (`:481`), `wrong event kind` (`:88`), `content too long` (`:91`), `empty public tag` (`:99`), `{name} tag must have exactly one value` (`:101`), `duplicate {name} tag` (`:114`), `unexpected public tag: {name}` (`:116`), `missing d tag` (`:129`), `invalid d tag length` (`:131`), `missing expiration tag` (`:133`), `expiration must be integer Unix seconds` (`:136`), `lease already expired` (`:139`), `lease ttl too long` (`:142`), `missing exec tag` (`:144`), `empty exec tag` (`:146`), `unknown executor key` (`:485`), `invalid encrypted content` (`:491`), `plaintext too long` (`:151`), `lease plaintext must be an object` (`:158`), `missing active` / `active must be a boolean` (`:161`, `:163`), `missing {key}` (`:186`), `unknown field: {key}` (`:190`), `invalid lease schema: {e}` (`:192`), `unsupported version` (`:196`), `generation must be a positive safe integer` (`:199`), `origin mismatch` (`:201`), `inactive lease must use minimal schema` (`:215`), `missing app_profile`/`transport`/`endpoint`/`subscriptions` (`:220-223`), `app profile not supported` (`:228`), `transport mismatch` (`:231`), `invalid endpoint length` (`:234`), `invalid string length` (`:378`), `subscription quota exceeded` (`:240`), `class not supported` (`:246`), `ignore quota exceeded` (`:250`), `p_tags_max must be positive` (`:257`), `filter member not permitted: {key}` (`:266`), `kind must be an integer` (`:274`), `kind not push-eligible` (`:279`), `class not permitted for kind` (`:282`), `lease filter not narrowed` (`:294`), `p-tag must be self` (`:302`), `invalid h tag` (`:308`, `:310`), `non-exact match value for {label}` (`:371`), `invalid {key} count` (`:334`, `:352`), `{key} must be an array` (`:331`, `:348`), `{key} values must be strings` (`:357`), `missing {key}` (`:325`), `internal optional-array misuse` (`:327`), `invalid relay URL` / `invalid tenant host` (`:591`, `:594`), `invalid generation` (`:558`), `invalid subscriptions` (`:550`), `duplicate object key: {key}` (`:441`), `non-finite number` (`:419`).

Internal string: exactly one — `lease persistence failed` (`:572`), which deliberately swallows the underlying `DbError`.

Outcome strings produced by **ingest**, not the handler (`ingest.rs:2186-2212`): `invalid: stale replacement`, `invalid: stale generation`, `invalid: endpoint already leased`, `invalid: lease quota exceeded`, `invalid: source event collision`, `invalid: lease constraint violation`.

##### 2.7 `community_provisioning` (HTTP)

| String | HTTP status mapped at `api/operator.rs` | Site |
|---|---|---|
| `actor not authorized: not a relay operator` | 403 (`operator.rs:180`) | `:265` |
| `community already exists` | 409 (`operator.rs:183`) | `:292` |
| `limit_reached: owner already owns the maximum number of communities` | 409 | `:295-298` |
| `initial_owner_pubkey is required when create_only is true` | 400 (`operator.rs:194`) | `:281-283` |
| `invalid initial_owner_pubkey: expected 64-char hex pubkey` | 400 | `:275-277` |
| `host is empty` / `host too long: {n} bytes (max 255)` | 400 | `:85`, `:88-90`, `:183`, `:185-188` |
| `host is not normalized: expected {…}` | 400 | `:93-96` |
| `host contains invalid characters` | 400 | `:107`, `:191` |
| `host must be a bare authority (no scheme, path, query, or userinfo)` | 400 | `:114-116`, `:195-197` |
| `host is not a valid authority` | 400 | `:124`, `:127` |
| `host is not a canonical authority: expected {…}` | 400 | `:141-143` |
| `domain name too long` / `domain contains an empty label` / `domain label too long` / `domain label contains invalid characters` | 400 | `:152`, `:156`, `:159`, `:168` |
| `failed to create community: {e}` | 500 (`operator.rs:186-192`) | `:288`, `:325` |
| `community provisioned but owner bootstrap failed: {e}` | 500 | `:332` |

---

#### 3. Public Rust items and their callers

##### 3.1 `handlers/moderation_commands.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `handle_moderation_command` | `pub` | 1 — `ingest.rs:1606` |

Everything else is private (`handle_ban/unban/timeout/untimeout/resolve`, `resolution_audit_action`, `insert_audit`, `authz_denial`, `invalid`, `error`, `extract_*`, `ensure_actor_not_banned`).

##### 3.2 `handlers/moderation_notices.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `ModerationNotice` (enum) | `pub` | `moderation_commands.rs` only |
| `ModerationNotice::ReportResolved` | — | `moderation_commands.rs:485` |
| `ModerationNotice::Restriction` | — | `moderation_commands.rs:208`, `:313` |
| **`ModerationNotice::ContentActioned`** | — | **ZERO production constructors** (only `moderation_notices.rs:390` in tests) |
| `send_moderation_notice` | `pub` | 3 — `moderation_commands.rs:204`, `:309`, `:481` |

##### 3.3 `handlers/moderation_authz.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `authorize_moderation_action` | `pub` | 6 — `moderation_commands.rs:156/235/274/338/399`, `api/bridge.rs:2055` |
| `ModerationAction` | `pub` | 6 of 8 variants used |
| ├ `Ban`, `Unban`, `Timeout`, `Untimeout`, `ResolveReport` | — | `moderation_commands.rs:162/241/280/344/405` |
| ├ `ViewQueue` | — | `api/bridge.rs:2061` |
| ├ **`DeleteMessage`** | — | **ZERO** — only `moderation_authz.rs:123/175/190/282/298` (own match arms + tests) |
| └ **`Kick`** | — | **ZERO** — same |
| `ModerationTarget<'a>` | `pub` | all 3 variants constructed |
| `ModerationAuthority` | `pub` | returned by every call, **discarded by every caller** |

**Consequence:** because `DeleteMessage`/`Kick` are never requested, and all 6 call sites pass `channel_id: None`, the `channel_role` lookup at `moderation_authz.rs:120-131` always evaluates to `None`, and the `ModerationAuthority::ChannelRole` arm (`:174-178`) is **dead in production**. The whole channel-local-authority feature is exercised by unit tests only (`:277-310`).

##### 3.4 `handlers/relay_admin.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `handle_relay_admin_event` | `pub` | 1 — `ingest.rs:1835` |

##### 3.5 `handlers/community_provisioning.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `ProvisionCommunityRequest` | `pub` | `api/operator.rs:164` |
| `ProvisionCommunityResponse` | `pub` | `api/operator.rs:174` (serialized to JSON) |
| `provision_community` | `pub` | 1 — `api/operator.rs:171` |
| `validate_pubkey_hex` | `pub(crate)` | 5 — `api/operator.rs:225/280/318/383/391` |
| `normalize_candidate_host` | `pub(crate)` | 3 — `api/operator.rs:216/278/484` |

HTTP route: `POST /operator/communities` (`router.rs:76`).

##### 3.6 `handlers/push_lease.rs`
| Item | Visibility | Production callers outside this file |
|---|---|---|
| `accept` | `pub` | 1 — `ingest.rs:2182` |
| `AcceptError` | `pub` | `ingest.rs:187-194` |
| `KIND_PUSH_LEASE` | `pub` | `ingest.rs:204/450/2156`, `side_effects.rs:2004/2130` |
| `PUSH_KINDS` | `pub(crate)` | `nip11.rs:208`, `:353` |
| `URGENT_KINDS` | `pub(crate)` | `nip11.rs:209` |
| `Subscription` | `pub` | `push_runtime.rs:13`, `:250` |
| `Suppress` | `pub` | reachable only through `Subscription.suppress` |
| **`validate_envelope`** | `pub` | **ZERO external** |
| **`parse_plaintext`** | `pub` | **ZERO external** |
| **`validate_plaintext`** | `pub` | **ZERO external** |
| **`LeaseEnvelope`** | `pub` | **ZERO external** |
| **`LeasePlaintext`** | `pub` | **ZERO external** |
| **`LeaseLimits<'a>`** | `pub` | **ZERO external** |
| **`AppProfile<'a>`** | `pub` | **ZERO external** (the `AppProfile` hits in `buzz-push-gateway` are a *different* type, `buzz-push-gateway/src/model.rs:14`) |
| **`MAX_SAFE_JSON_INTEGER`** | `pub` | **ZERO external** — used only at `push_lease.rs:198` |

8 public items with no consumer outside this module. `validate_envelope`/`parse_plaintext`/`validate_plaintext` are documented as a public validation API (`push_lease.rs:1-6`) that nothing calls except `accept` and tests.

##### 3.7 `handlers/identity_archive.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `handle_identity_archive_event` | `pub` | 1 — `ingest.rs:1942` |

##### 3.8 `handlers/report.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `handle_report_event` | `pub` | 1 — `ingest.rs:1588` |
| **`REPORT_TYPES`** | `pub` | **ZERO external** — only `report.rs:202` |

##### 3.9 `handlers/product_feedback.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `handle` | `pub` | 1 — `ingest.rs:1567` |

##### 3.10 `push_runtime.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `run_matcher` | `pub` | 1 — `main.rs:687` |
| `run_delivery_worker` | `pub` | 1 — `main.rs:688-690` |

Both spawned only when `state.config.push_gateway_delivery_url.is_some()` (`main.rs:686`) — which is the **default**, because an unset `BUZZ_PUSH_GATEWAY_DELIVERY_URL` falls back to the hard-coded `https://push.buzz.xyz/v1/deliveries/apns` (`config.rs:339`, `:752-758`). Only an explicitly-empty value disables it (`config.rs:753`).

##### 3.11 `storage_sweep.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `StorageSweepConfig` | `pub` | `main.rs:1447`, `:1453` |
| `StorageSweepConfig::from_env` | `pub` | 1 — `main.rs:1453` (behind a function-local `OnceLock`) |
| `StorageSweepState` | `pub` | `state.rs:561`, `:764` |
| `maybe_spawn_sweep` | `pub` | 1 — `main.rs:1460-1471` |
| `emit_storage_metrics` | `pub` | 1 — `main.rs:1474-1477` |
| `StorageEmittedKey` | `pub(crate)` | **used only inside `storage_sweep.rs`** — could be private |

Both entry points are called only from the leader-only branch of the usage tick (`main.rs:1423-1430`).

##### 3.12 `workflow_sink.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `RelayActionSink` | `pub` | `main.rs:594` |
| `RelayActionSink::new` | `pub` | 1 — `main.rs:594` |
| `impl ActionSink for RelayActionSink::send_message` | trait impl | `buzz-workflow/src/executor.rs:567-569` |
| `resolve_mention_pubkeys` | private | `workflow_sink.rs:291` |

---

#### 4. Workflow-action surface actually implemented by `workflow_sink`

The `ActionSink` trait declares **exactly one** method (`buzz-workflow/src/action_sink.rs:44-64`): `send_message`. `RelayActionSink` implements it (`workflow_sink.rs:172-179`) and nothing else.

Against the 7 workflow action types documented in `ARCHITECTURE.md:533-542`:

| Workflow action | Path | Reaches `workflow_sink`? | Working end-to-end? |
|---|---|---|---|
| `send_message` | `executor.rs:566-569` → `ActionSink::send_message` | **yes** | **yes** |
| `send_dm` | `executor.rs:575-579` | no | **no** — `Err(WorkflowError::NotImplemented("SendDm"))` at `executor.rs:578` |
| `set_channel_topic` | `executor.rs:580-584` | no | **no** — `Err(NotImplemented("SetChannelTopic"))` at `executor.rs:583` |
| `add_reaction` | `executor.rs:585-607` → `add_reaction_impl` HTTP POST | no | **no** — POSTs to `{BUZZ_RELAY_BASE_URL}/api/messages/{id}/reactions` (`executor.rs:886-888`); no such route is registered in `router.rs` (verified: zero `reactions` and zero `api/messages` matches). Returns `WorkflowError::WebhookError("AddReaction: relay returned 404 …")` |
| `call_webhook` | `executor.rs:608+` (own HTTP client) | no | yes (outside this module) |
| `request_approval` | `executor.rs` → `StepResult::Suspended` | no | **no** — ARCHITECTURE.md:826 (WF-08) |
| `delay` | `executor.rs` | no | yes (outside this module) |

**Verified from the sink side:** `workflow_sink` implements 1 of 7 workflow actions. ARCHITECTURE.md items 5 and 6 (`:826-827`) are accurate as written but **understate** the gap: `add_reaction` is a third broken action not listed there, and the relay-side sink surface is a single method, so `send_dm`/`set_channel_topic` cannot be fixed without widening the trait.

---

#### 5. HTTP surface touched by this module

| Route | Method | Handler | Auth |
|---|---|---|---|
| `/operator/communities` | POST | `api::operator::provision_community` → `community_provisioning::provision_community` | NIP-98 against `RELAY_OPERATOR_API_ORIGIN` + replay guard (`operator.rs:104-135`) + `RELAY_OPERATOR_PUBKEYS` allowlist (`community_provisioning.rs:258-266`) |
| `/moderation/reports` | GET | `api::bridge::moderation_reports` | NIP-98 + `ModerationAction::ViewQueue` (`bridge.rs:2054-2067`) |
| `/moderation/audit` | GET | `api::bridge::moderation_audit` | same |

Read cap: `MODERATION_READ_LIMIT = 500` (`bridge.rs:2089`).

No handler in this module registers a route itself; all three are wired in `router.rs:76`, `:113`, `:114`.

---

#### 6. Outbound HTTP surface

One only: the push gateway `POST` (`push_runtime.rs:517-528`).

Request body (`DeliveryRequest`, `push_runtime.rs:31-37`): `{ "v": 1, "endpoint_grant": <string>, "request_id": <uuid>, "expires_at": <i64> }`. `request_id` is the durable wake row id and is **deliberately stable across retries** (`push_runtime.rs:488-490`), pinned by an HTTP-level test (`push_runtime.rs:626-655`).

Headers: `Authorization: Nostr <base64(kind-27235 event)>` (`push_runtime.rs:551-565`) and `Content-Type: application/json`.

Response contract (`DeliveryResponse`, `push_runtime.rs:39-51`), an internally-tagged enum on `status`:

| HTTP | Body `status` | Relay action | Site |
|---|---|---|---|
| 2xx | `accepted` | `complete_push_wake` | `:434-441` |
| 2xx | anything else / unparseable | `fail_push_wake` | `:442-447` |
| 410 | `invalid_endpoint { generation, invalid_at }` | `disable_push_endpoint` **iff** generation matches, then `fail` | `:448-473` |
| 503 | `retry { retry_after_seconds }` | `retry_or_fail`, delay clamped to `>0` else 2 s | `:474-484` |
| 429 | (ignored) | `retry_or_fail(2)` | `:485-487` |
| 404 **and** `attempt > 1` | (ignored) | `complete_push_wake` — treats a replayed timed-out attempt as delivered | `:491-497` |
| timeout / connect error | — | `retry_or_fail(2)` | `:498` |
| anything else | — | `fail_push_wake` | `:499-503` |


## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: API Surface

---

#### 1. HTTP / WebSocket routes owned by this group

| Route | Method | Handler | Registered | Auth |
|---|---|---|---|---|
| `/huddle/{channel_id}/audio` | GET (WS upgrade) | `audio::handler::ws_audio_handler` | `router.rs:125-128` | host→tenant bind, then in-band NIP-42 |
| `/_mesh/demo/echo` | POST | `api::mesh_demo::demo_echo` | `router.rs:123` | **none** — gated only on `BUZZ_MESH=on` + `BUZZ_MESH_DEMO_ECHO=on`, else 404 |
| `/_mesh` | GET | `mesh_status_handler` (`router.rs:399-406`) | `router.rs:230` (health router) | **none** — health router has "no metrics middleware, no auth, no CORS, no body limit" (`router.rs:222-224`) |

`/huddle/…/audio` sits on `api_router`, which carries
`RequestBodyLimitLayer::new(1024*1024)` (`router.rs:130`) — irrelevant to a WS
upgrade — plus the shared metrics/trace/CORS layers (`router.rs:186-189`).

The `{channel_id}` path segment is extracted as `Path<Uuid>` (`handler.rs:67`), so a
non-UUID segment is rejected by axum with 400 before the handler runs.

---

#### 2. `GET /huddle/{channel_id}/audio` — full frame inventory

##### 2.1 Pre-upgrade rejections (HTTP status, no WebSocket)

| Status | Body | Condition | Line |
|---|---|---|---|
| 404 | `relay: no community is configured for this host` | `tenant::bind_community(&state.db, Host)` errs — unmapped host or DB failure. Deliberately generic so an anonymous caller cannot enumerate communities (`handler.rs:69-73`) | `handler.rs:80-88` |
| 503 | `relay: connection limit reached` | `conn_semaphore.try_acquire_owned()` fails — **shared with ordinary relay WebSockets** | `handler.rs:90-99` |
| — | (silent return) | `state.db.is_community_active(community)` false, via `run_registered_community_connection` | `handler.rs:156-164` |

Parser bounds installed **before** upgrade: `max_message_size` and `max_frame_size`
both `MAX_WEBSOCKET_MESSAGE_BYTES = 8192` (`handler.rs:52`, applied
`handler.rs:116-119`, `:105`). Pinned by
`audio_websocket_parser_rejects_oversized_messages_before_handler_reads_them`
(`handler.rs:1417-1427`).

##### 2.2 Inbound frames (client → relay)

| Frame | Shape / condition | Handling | Line |
|---|---|---|---|
| Text `{"type":"auth", …}` | Required first; anything else is ignored while the 5 s `AUTH_TIMEOUT` runs. `>8192` chars → warn + `continue` (still inside the window) | `AuthMsg` deserialize | `handler.rs:126-142`, `:188-214` |
| Text `{"type":"leave"}` | any valid JSON with `type == "leave"` | breaks `recv_loop` → clean teardown | `handler.rs:1030-1036` |
| Text other | valid or invalid JSON, `≤8192` | **silently ignored** (no error frame) | `handler.rs:1030-1036` |
| Text `>8192` (post-auth) | | warn + drop; connection survives | `handler.rs:1026-1029` |
| Binary | `≤ MAX_AUDIO_FRAME_BYTES = 4096` (`handler.rs:44`). If room pin `≥2`: must be `> 8` bytes and `FrameHeader::parse` must yield a non-empty payload | forwarded opaquely (owner path: `room.broadcast_frame`; ingress path: `session.forward_media`) | `handler.rs:960-1023` |
| Binary `>4096` | | warn + drop; connection survives | `handler.rs:961-964` |
| Ping | | `Pong` echoed via the **priority** control channel (`try_send`, dropped if that 8-slot channel is full) | `handler.rs:1041-1044` |
| Pong | | resets `missed_pongs` to 0 | `handler.rs:1038-1040` |
| Close / stream end | | breaks `recv_loop` | `handler.rs:1045` |

`AuthMsg` (`handler.rs:126-139`):

| Field | Type | Required | Default |
|---|---|---|---|
| `type` | String | yes, must equal `"auth"` | — |
| `event` | `nostr::Event` | yes | — |
| `parent_channel_id` | `Option<Uuid>` | only for TTL channels | `None` |
| `protocol_version` | u8 | no | **1** (`handler.rs:141-142`) |

Note the size window: the parser admits 8192-byte binary frames but the handler
drops anything over 4096 — so 4097..8192 binary bytes are accepted by tungstenite,
buffered, then discarded.

##### 2.3 Outbound frames (relay → client)

Every outbound text frame is a JSON object with a `type`. **6 distinct `type`
values**: `challenge`, `error`, `joined`, `left`, `roster`, plus WS control frames.

| `type` | Fields | Emitted when | Line |
|---|---|---|---|
| `challenge` | `challenge: String` (`buzz_auth::generate_challenge`) | first frame after upgrade, unconditionally | `handler.rs:176-186` |
| `joined` | `pubkey`, `peer_index`, `peers: [{pubkey, peer_index}]` | after successful admission. **Owner/local path broadcasts to the whole room** (`handler.rs:641`); ingress path sends **only to the joining socket** (`handler.rs:628-632`) | `handler.rs:620-643` |
| `left` | `pubkey`, `peer_index` | on disconnect, owner/local path only (`remote_session.is_none()`, `handler.rs:818-820`); owner also fans `left` for remote peers on `UnregisterPeer` (`join.rs:1301-1310`) and on control-stream teardown (`join.rs:1354-1366`) | `handler.rs:812-820` |
| `joined` (revision-bearing) | `revision`, `pubkey`, `peer_index`, `peers` | ingress path only, forwarded from an owner `RosterDelta` | `join.rs:1570-1576` |
| `left` (revision-bearing) | `revision`, `pubkey`, `peer_index` | ingress path only, from an owner `RosterDelta` | `join.rs:1577-1581` |
| `roster` | `revision`, `peers: [{pubkey, peer_index}]` | ingress path only, from an owner `RosterSnapshot` (initial or post-resync) | `join.rs:1552-1558` |

##### 2.4 Outbound `error` frames — exact conditions

Eleven distinct error emissions; only 8 carry a machine-readable `code`.

| `code` | `message` | Condition | Line |
|---|---|---|---|
| *(none)* | `auth failed` | `state.auth.verify_auth_event` rejects (bad sig, wrong challenge, wrong relay URL, allowlist/token policy) | `handler.rs:226-236` |
| *(none)* | `restricted: not a relay member` | `api::relay_members::enforce_relay_membership` errs — **no-op when `require_relay_membership=false`, the default** (`api/mod.rs:67`, `:130-131`) | `handler.rs:244-262` |
| *(none)* | `not a member` | `ensure_membership` errs: DB error, archived channel, missing/unlinked `parent_channel_id`, non-member of a non-open channel | `handler.rs:265-286` |
| *(none)* | `huddle has ended` | post-`get_or_create` re-check found `archived_at.is_some()` | `handler.rs:389-403` |
| `huddle_relay_draining` | `relay is draining; reconnect` | mesh on and `HuddleOwnerRegistry::is_draining()` | `handler.rs:308-320` |
| `join_rejected` | `huddle join rejected` | `resolve_join_owner_ready` errs (fence rejection, Redis error, or the 25-attempt owner-ready loop exhausting) | `handler.rs:342-355` |
| `huddle_audio_unavailable` | `huddle audio unavailable in this deployment` | mesh **off** and `huddle_audio_available == false` | `handler.rs:356-375` |
| `unsupported_version` | `huddle audio protocol v{n} not supported; relay max is v2`, plus `current_version` | `requested_version == 0 \|\| > 2` | `handler.rs:417-441` |
| `huddle_owner_unreachable` | `could not reach the huddle owner` | `DialError::Mesh` from `dial_remote_owner` | `handler.rs:487-503` |
| `room_full` | `peer index space exhausted` | `AdmissionError::Full` (soft cap 25 **or** index exhaustion) | `handler.rs:513-522`; cross-pod mirror `handler.rs:917-921` |
| `room_ended` | `huddle has ended` | `AdmissionError::Ended` | `handler.rs:523-533`; mirror `handler.rs:922-924` |
| `upgrade_required` | `this huddle is using audio protocol v{pinned}; your client requested v{requested}`, plus `pinned_version`, `requested_version` | `AdmissionError::VersionMismatch` | `handler.rs:534-548`; mirror `handler.rs:925-932` |
| `join_rejected` + `fence_reason` | `huddle join rejected`, `fence_reason ∈ {stale_generation, no_active_lease, owner_mismatch, future_generation}` | owner replied `RegisterRejected(Fenced(..))` | `handler.rs:933-937` |

`remote_rejection_ws_error` (`handler.rs:915-939`) is the cross-pod mirror: it
reproduces the same `code`s a same-pod join emits, so a client cannot tell which
pod owned the room.

##### 2.5 Silent teardowns (no `error` frame)

| Condition | Line |
|---|---|
| challenge send fails | `handler.rs:180-185` |
| 5 s auth timeout or disconnect before auth | `handler.rs:207-214` |
| `get_channel` pre-join check returns `Err` (fail-closed) | `handler.rs:404-410` |
| `joined` send fails on the ingress path | `handler.rs:628-636` |
| 3 missed pongs (`MAX_MISSED_PONGS`, 30 s `HEARTBEAT_INTERVAL`) → `cancel` → `send_loop` sends bare `Close(None)` | `handler.rs:1127-1151`, `:1066-1069` |
| owner lost/draining (owner path) → `cancel` + `fence.forget` | `handler.rs:727-772` |
| owner `Goodbye`/stream close (ingress path) → `teardown_remote_huddle` | `handler.rs:707-714`, `:897-910` |
| community deactivated mid-connection | `state.rs` connection registry via `handler.rs:156` |

---

#### 3. Huddle event kinds

| Kind | Constant | Producer in this group | Not produced here |
|---|---|---|---|
| 48100 `KIND_HUDDLE_STARTED` | `buzz-core/src/kind.rs:530` | **none** — the relay only *reads* it, via `db.huddle_started_link_exists` (`handler.rs:1176-1186`) to validate a claimed parent. Produced by the desktop client (`desktop/src-tauri/src/huddle/mod.rs:252`) | |
| 48101 `KIND_HUDDLE_PARTICIPANT_JOINED` | `kind.rs:532` | `handler.rs:645-653` — after successful admission, on **every** path (owner and ingress) | |
| 48102 `KIND_HUDDLE_PARTICIPANT_LEFT` | `kind.rs:534` | `handler.rs:822-831` — after peer removal, unconditionally | |
| 48103 `KIND_HUDDLE_ENDED` | `kind.rs:536` | `handler.rs:850-859` — only when `should_auto_end && archive_channel` succeeded | |
| 48106 `KIND_HUDDLE_GUIDELINES` | `kind.rs:538` | **none in this group.** Only relay reference is the kind-label allowlist `handlers/event.rs:49`. Produced client-side (`desktop/src-tauri/src/huddle/agents.rs:31`) | |

All three relay-emitted events are signed with `state.relay_keypair` (`handler.rs:1268`),
so their author is the relay, not the participant. Tags:
`h = parent_channel_id`, `p = participant_pubkey` (`handler.rs:1240-1256`); content
is `{"ephemeral_channel_id": "<uuid>"}` (`handler.rs:1238`). The `parent_channel_id`
is the ephemeral channel's *parent* for TTL channels and the channel itself
otherwise (`handler.rs:1170-1194`).

Emission is a 4-step pipeline (`handler.rs:1274-1332`): persist
(`db.insert_event`) → `mark_local_event` → `fan_out_event_to_local_subscribers` →
`pubsub.publish_event`. On a duplicate insert, fan-out is **skipped entirely**
(`handler.rs:1285-1295`). On an insert error, the event is still fanned out from an
in-memory `StoredEvent::new` (`handler.rs:1296-1307`) — live subscribers see it,
late joiners never will. On a publish error, `local_event_ids` is invalidated
(`handler.rs:1326-1330`).

---

#### 4. `HuddleControl` mesh stream API

Opened by `dial_remote_owner` (`join.rs:1660-1724`) with
`StreamHello{sender: local_runtime_id, role: Session{fenced, profile: HuddleControl}}`.
Accepted by `HuddleControlAcceptor::accept_inbound` (`join.rs:1054-1093`).

##### 4.1 `accept_inbound` structural validation (before any state touch)

| Check | Rejection | Line |
|---|---|---|
| `hello.sender == from` (authenticated peer) | `MeshError::Transport` | `join.rs:1060-1065` |
| role is `Session` | `MeshError::Transport` | `join.rs:1066-1070` |
| `profile == HuddleControl` | `MeshError::Transport` | `join.rs:1071-1075` |
| `fenced.owner_runtime_id == self.local_runtime_id` | `MeshError::OwnerMismatch` | `join.rs:1079-1086` |

Deliberately **not** Redis-fenced here — the fence key needs the community, which
only arrives on the first `RegisterPeer` (`join.rs:1032-1041`).

##### 4.2 `serve_control_loop` frame handling (`join.rs:1115-1370`)

| Inbound | Precondition | Effect |
|---|---|---|
| `Data{fenced: f, payload}` where `f != fenced` | — | break `Err(OwnerMismatch)` (`join.rs:1191-1198`) |
| `RegisterPeer{community_id, pubkey, protocol_version}` | community latched on first frame; a later frame naming a different community → break `Err(Transport)` (`join.rs:1200-1209`) | `is_draining()` → `Goodbye(Draining)` (`:1219-1222`); else `get_or_create` room, `subscribe_roster`, then **`directory.validate(community, fenced)` before `add_peer`** (`:1231-1245`). Fence error classifiable → `RegisterRejected(Fenced(..))`; non-fence error tears the stream down |
| `UnregisterPeer{pubkey}` | pubkey present in this stream's `registered` map | `room.remove_peer` + `left` fan-out to local peers. **No fence** — cannot touch another community's room (`join.rs:1264-1288`) |
| `RosterResync` | community latched | replies `RosterSnapshot` (`join.rs:1290-1300`) |
| `PeerRegistered` / `RosterSnapshot` / `RosterDelta` / `RegisterRejected` | — | break `Err(Transport)` — owner→non-owner replies are a protocol violation on the accept side (`join.rs:1302-1310`) |
| `Goodbye{..}` or clean close | — | break `Ok(())` (`join.rs:1183`) |
| any other frame (`Hello`, `Gossip`) | — | break `Err(Transport)` (`join.rs:1184-1188`) |

Owner-initiated arms: `draining` fires → `Goodbye(Draining)`; `lost` fires →
`Goodbye(StaleGeneration)` (`join.rs:1157-1166`, sent at `join.rs:1315-1321`).
A `roster_rx` `Lagged` recovers by sending a fresh `RosterSnapshot`
(`join.rs:1174-1182`); `Closed` breaks the loop.

Teardown always drops every peer this stream registered, regardless of exit path
(`join.rs:1345-1367`).

##### 4.3 Ingress-side reader `read_owner_control` (`join.rs:1527-1612`)

| Owner frame | Ingress action |
|---|---|
| `Goodbye{reason}` | return `HuddleTeardownCause::from_goodbye` (`join.rs:1497-1504`) |
| `RosterSnapshot{revision, peers}` | set `revision`, emit WS `roster` |
| `RosterDelta` where `next == revision+1` | advance, emit WS `joined`/`left` |
| `RosterDelta` where `next <= revision` | ignore (`join.rs:1583`) |
| `RosterDelta` with a gap | send `RosterResync` upstream, **do not apply** (`join.rs:1584-1597`) |
| any other decoded msg | ignore (`join.rs:1598`) |
| decode error | `debug!` and keep reading (`join.rs:1599`) |
| clean close / transport error | `StreamClosed` |

Pinned end-to-end by `roster_revision_gap_requests_resync_before_forwarding_new_state`
(`join.rs:2113-2183`).

---

#### 5. Public Rust API and its callers

##### 5.1 `audio` module

| Item | Line | Production callers |
|---|---|---|
| `ws_audio_handler` | `handler.rs:64` | `router.rs:127` |
| `AudioRoomManager::{new, get_or_create, get, get_unambiguous_by_channel, cleanup_if_empty}` | `room.rs:496-550` | `state.rs:768`; `handler.rs:380`, `:401`, `:408`, `:484`, `:501`, `:637`, `:849`, `:865`; `join.rs:1176`, `:1229`, `:1266`, `:1291`, `:1347`; `mesh.rs:221` |
| `Room::{add_peer, add_peer_at_index, remove_peer, remove_peer_and_check_ended, broadcast_frame, deliver_prefixed, broadcast_control, subscribe_roster, roster_snapshot, peer_pubkeys, is_empty, clear_ended}` | `room.rs:228-487` | all reached from `handler.rs` / `join.rs` / `mesh.rs` |
| `Room::mark_ended` | `room.rs:192` | **zero production callers** — only `room.rs:660` (test). Production ends rooms via `remove_peer_and_check_ended` |
| `FrameHeader::{parse, is_dtx}`, `V2_HEADER_LEN`, `FLAG_DTX` | `wire.rs:29-88` | `handler.rs:948`, `:983`, `:986`, `:1001`. `FLAG_DTX` itself is referenced only in `wire.rs` (relay side) — the desktop has its own copy (`desktop/src-tauri/src/huddle/wire.rs:48`) |
| `GenerationFloor::{new, check, forget}`, `FenceVerdict` | `mesh.rs:93-146` | `mesh_boot.rs:516`; `mesh.rs:214`; `handler.rs:755`, `:763`, `:909` |
| `MeshAudioRouter::{with_fence, on_media_datagram}` | `mesh.rs:180-250` | `mesh_boot.rs:239-245` |
| `MeshAudioRouter::{new, fence, local_runtime_id}` | `mesh.rs:169`, `:196`, `:201` | **zero production callers** (`new` only in `mesh.rs` tests) |
| `HuddleOwnerDirectory` trait + `mesh::Ownership` | `mesh.rs:67-80` | **zero implementors, zero callers anywhere** — fully dead |
| `spawn_remote_peer_sink` | `mesh.rs:262` | `join.rs:1391` |

##### 5.2 `audio::join` module

| Item | Line | Production callers |
|---|---|---|
| `HuddleDirectory` trait (5 methods) | `join.rs:66-101` | impl for `SessionDirectory` at `join.rs:107-183`; used by `resolve_join`, the renewer, `HuddleControlAcceptor` |
| `resolve_join` | `join.rs:317` | `join.rs:426` only (via `resolve_join_owner_ready`) — no direct production caller |
| `resolve_join_owner_ready` | `join.rs:416` | `handler.rs:322` |
| `dial_remote_owner` | `join.rs:1660` | `handler.rs:459` |
| `send_clean_close` | `join.rs:1770` | `handler.rs:520`, `:530`, `:544`, `:716` |
| `read_owner_control` | `join.rs:1527` | `handler.rs:707` |
| `read_teardown_cause` | `join.rs:1623` | **zero production callers** — 4 test callers only (`join.rs:2948`, `:2981`, `:3007`). Superseded by `read_owner_control` |
| `HuddleOwnerRegistry::{new, is_draining, lost_for, drain_for, attach_signals, release, drain, drain_all}` | `join.rs:636-773` | `mesh_boot.rs:474`; `handler.rs:308`, `:582`, `:588`, `:589`, `:876`; `join.rs:1089`, `:1090`, `:1219`; `mesh_boot.rs:489` |
| `HuddleOwnerRegistry::attach` | `join.rs:659` | **zero production callers** — thin wrapper over `attach_signals`; 5 test callers |
| `HuddleOwnerRegistry::drain` | `join.rs:750` | reached only through `drain_all` (`join.rs:772`) |
| `spawn_observable_huddle_renewer` | `join.rs:482` | `join.rs:674`, `:688`, `:695` (all inside `attach_signals`) — no external caller |
| `HuddleLeaseRenewer` (`.task`, `.lost`) | `join.rs:464-471` | `.lost` cloned at `join.rs:694`; **`.task` is never awaited in production** — the struct is dropped, detaching the task |
| `HuddleControlAcceptor::{new, accept_inbound}` | `join.rs:1013`, `:1054` | `mesh_boot.rs:262-268`, `:277` |
| `encode_control` / `decode_control` | `join.rs:1006`, `:1011` | `join.rs` internally (7 sites) |
| `FenceRejection::{from_mesh_error, code}` | `join.rs:996`, `:1013` | `join.rs:1239`; `handler.rs:936` |
| `JoinOutcome::fenced_header` | `join.rs:272` | `handler.rs:452` |
| `RemoteHuddleSession::{peer_index, roster, fenced, pubkey, forward_media}` | `join.rs:1728-1766` | `handler.rs:508`, `:609`, `:520`, `:694`, `:1019` |
| `HUDDLE_CONTROL_PROFILE` | `join.rs:1027` | `join.rs:143` |
| `HUDDLE_SESSION_ENDED` | `join.rs:1451` | `join.rs:1781` |

##### 5.3 `tunnel` module

`tunnel/mod.rs:1-8` exports exactly two submodules: `directory`, `reliable`.

| Item | Line | Production callers |
|---|---|---|
| `SessionDirectory::{new, with_lease_ttl, acquire, renew, release, lookup, validate_fenced_header}` | `directory.rs:180-439` | `mesh_boot.rs:512`; `join.rs:110-183`; `reliable.rs:87`, `:383` |
| `SessionDirectory::takeover` | `directory.rs:233` | **zero callers anywhere** (delegates to `acquire`) |
| `SessionDirectory::known_generation` | `directory.rs:324` | **zero production callers** — 2 test callers (`directory.rs:695`, `:764`) |
| `SessionLease::fenced_header` | `directory.rs:444` | `reliable.rs:117`; tests |
| `ReliableStreamRouter::{new, directory, local_runtime_id, join, accept_inbound}` | `reliable.rs:50-172` | `mesh_boot.rs:283-287`, `:290`; `api/mesh_demo.rs:73`, `:95`, `:239`+ |
| `ReliableStreamRouter::{spawn_renewer, spawn_observable_renewer}` | `reliable.rs:179`, `:192` | **zero callers.** `mesh_boot.rs:212-215` explicitly documents that renewal is not wired yet |
| `ReliableMeshStream::{new, fenced, send_bytes, send_goodbye, finish, recv_validated, community_id}` | `reliable.rs:243-359` | `mesh_boot.rs:330-357`; `api/mesh_demo.rs:110`, `:116` |
| `ReliableMeshStream::{new_inbound, with_community}` | `reliable.rs:253`, `:263` | `new_inbound` from `reliable.rs:170`; **`with_community` has zero callers** |
| `MAX_RELIABLE_PAYLOAD_BYTES` | `reliable.rs:31` | `reliable.rs:280` |
| `ReliableJoin`, `ReliableInbound`, `ReliableFrame`, `ReliableStreamError` | `reliable.rs:207-568` | `mesh_boot.rs`, `api/mesh_demo.rs` |

##### 5.4 `mesh_boot` module

| Item | Line | Production callers |
|---|---|---|
| `boot_mesh` | `mesh_boot.rs:411` | `main.rs:442` |
| `MeshHandle::wire_consumers` | `mesh_boot.rs:180` | `main.rs:454-458` |
| `MeshHandle::status` | `mesh_boot.rs:172` | `router.rs:401` |
| `MeshHandle` public fields (`directory`, `transport`, `membership`, `local_runtime_id`, `dispatcher`, `audio_fence`, `owners`) | `mesh_boot.rs:137-168` | `handler.rs`, `api/mesh_demo.rs` |
| `MeshHandle.membership` | `mesh_boot.rs:141` | **zero readers** — populated at `mesh_boot.rs:501` but never used by any consumer; `/_mesh` goes through the private `runtime` field instead |
| `wire_mesh_consumers` | `mesh_boot.rs:224` | `mesh_boot.rs:183` + 1 test |
| `MeshInboundDispatcher::register_{huddle_control,reliable_stream,datagrams}` | `mesh_boot.rs:70-88` | `mesh_boot.rs:242`, `:269`, `:288` |
| `run_demo_echo` (`pub(crate)`) | `mesh_boot.rs:307` | `mesh_boot.rs:294`; `api/mesh_demo.rs:294` (test) |
| `SessionStreamHandler`, `DatagramHandler` type aliases | `mesh_boot.rs:39`, `:42` | internal |

`main.rs:455-459` wires consumers **before** `state.mesh.set(handle)`, so inbound
mesh traffic can be served before any local join can resolve ownership.
`main.rs:459` is `unreachable!("mesh handle is set exactly once, right here")`.

##### 5.5 `conformance` module — the `Tracer` surface

`Tracer` (`buzz-conformance/src/lib.rs:314-318`) is a single-method trait:
`fn record(&self, step: TraceStep)`. It is `Send + Sync`, takes `&self`, returns
nothing, and cannot fail — so an emit can never propagate an error into a request
path, and cannot apply backpressure.

| Impl | Line | Behaviour |
|---|---|---|
| `NoopTracer` | `tracers.rs:20-24` | discards; **this is the production binding** (`state.rs:798`) |
| `JsonlTracer` | `tracers.rs:30-72` | truncate-on-create, one JSON line per step, `flush()` per record, `Mutex` poisoning recovered via `into_inner` |
| `CountingTracer` (private) | `conformance/mod.rs:357-372` | increments then delegates |
| `buzz_conformance::NoopTracer` | `buzz-conformance/src/lib.rs:323` | **zero users** |

**`JsonlTracer` has zero callers in the entire workspace** — no production site, no
test, no integration harness. The only references are its own definition and two
doc comments (`state.rs:616`, `:795`) promising that "conformance tests bind" it.

Public helpers and their callers:

| Helper | Line | Callers |
|---|---|---|
| `state_for_request` | `:55` | `ingest.rs:145`, `:1381`, `:1801`, `:2195`, `:2348`, `:2485`, `:2547`; `req.rs:118` |
| `msg_id_label` | `:78` | `ingest.rs:142`, `:2192`, `:2337`, `:2343`, `:2471`, `:2476`, `:2481` |
| `channel_label` | `:89` | `conformance/mod.rs:154`, `:279`, `:314`; tests |
| `claimed_community_from_event` | `:101` | `ingest.rs:143`, `:1788`, `:2193`, `:2334`, `:2468` |
| `step` | `:121` | **zero callers** |
| `emit` | `:127` | `ingest.rs:1440`, `:2348`, `:2485` |
| `record_req_authcheck` | `:148` | `req.rs:144` |
| `project_row_communities` | `:234` | `conformance/mod.rs:281`, `:316`; tests |
| `record_read_message_rows` | `:265` | `req.rs:359` |
| `record_read_by_id_rows` | `:300` | `req.rs:671` |
| `EmitGuard::arm` | `:383` | `ingest.rs:1408`, `:2576` (test) |
| `sanitized_reason_for` | `:422` | `ingest.rs:1437` |
| `RowCommunityProjection` | `:216` | internal + tests |

The `EmitGuard` contract: `arm(tracer, state, kind)` returns
`(guard, counting_tracer)`; the caller **must** thread the returned wrapper
downstream. `ingest_event` does exactly this (`ingest.rs:1408-1413`), passing
`&tracer` into `ingest_event_inner`. On `Drop` with zero records, the guard emits
`ImplBug{kind}` onto the *original* tracer — which in production is `NoopTracer`,
so the breach is discarded.

---

#### 6. What this group deliberately does **not** expose

- No HTTP endpoint returns huddle room state. There is no `GET /huddle/{id}/peers`,
  no participant count, no roster REST surface. Room state is only observable by
  joining the WebSocket or by replaying kinds 48101/48102/48103.
- No admin/operator surface for huddles: no force-end, no kick, no mute, no
  capacity override. `buzz-admin` has no huddle subcommand.
- No `buzz-cli` subcommand touches `/huddle/…/audio` (grep across `crates/buzz-cli`
  finds no huddle route).
- The tunnel lane has **no product-facing API**: the only route reaching
  `ReliableStreamRouter::join` is the demo-gated `POST /_mesh/demo/echo`
  (`api/mesh_demo.rs:73`), documented at `mesh_boot.rs:206-215` as "no product
  session consumer is wired yet".


## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: API Surface

The crate exposes **8 public modules** (`lib.rs:21-28`) with **no `#[doc(hidden)]`
and no feature gates**. Everything is unconditionally public. `Cargo.toml` declares
no `[features]` section.

Public-item census (counting named types, traits, functions, consts, type aliases —
not trait-impl methods): **~120 items**. Breakdown below with verified caller data.

**Sole production consumer: `buzz-relay`** (`crates/buzz-relay/Cargo.toml:26`).
Verified — grep for `buzz_relay_mesh` across `crates/**` and `desktop/**` matches
exactly 8 files, all under `crates/buzz-relay/src/`: `mesh_boot.rs` (9),
`tunnel/reliable.rs` (6), `audio/join.rs` (5), `api/mesh_demo.rs` (4),
`config.rs` (2), `audio/mesh.rs` (2), `tunnel/directory.rs` (1),
`audio/handler.rs` (1).

---

#### 1. The boot-path handle and consumer-wiring API

The crate itself exposes **no boot function**. The handle consumers actually hold is
defined in `buzz-relay`, not here:

| Item | Where | Notes |
|---|---|---|
| `mesh_boot::boot_mesh(config, redis_pool, relay_keypair, shutting_down) -> anyhow::Result<Option<MeshHandle>>` | `crates/buzz-relay/src/mesh_boot.rs:412-420` | called once, `main.rs:442`; `Ok(None)` when `BUZZ_MESH` off (`mesh_boot.rs:417-419`) |
| `mesh_boot::MeshHandle` | `mesh_boot.rs:134-172` | published to `AppState.mesh: Arc<OnceLock<MeshHandle>>` (`state.rs:627`) at `main.rs:457`; `main.rs:460` is `unreachable!("mesh handle is set exactly once, right here")` |
| `MeshHandle::status()` | `mesh_boot.rs:172-174` | delegates to `MeshRuntime::membership().status()`; sole caller `router.rs:381` |
| `MeshHandle::wire_consumers(rooms, demo_echo, shutting_down)` | `mesh_boot.rs:183-199` | sole caller `main.rs:455-459`, i.e. **before** publication to `AppState` |
| `mesh_boot::wire_mesh_consumers(...)` (9 args) | `mesh_boot.rs:229-241` | `#[allow(clippy::too_many_arguments)]`; callers: `MeshHandle::wire_consumers` + one test (`mesh_boot.rs:686`) |
| `mesh_boot::MeshInboundDispatcher` + `register_huddle_control` / `register_reliable_stream` / `register_datagrams` | `mesh_boot.rs:57-59`, `:71,:78,:84` | the fan-out over the transport's single `set_inbound` slot; first registration wins (`OnceLock::set`) |
| `AppState::mesh() -> Option<&MeshHandle>` | `crates/buzz-relay/src/state.rs:812-814` | read only at `router.rs:381` |

`MeshHandle` fields (`mesh_boot.rs:136-171`): `directory` (Redis fenced session
directory), `transport: Arc<dyn RelayPeerTransport>`, `membership: Arc<dyn
RelayMeshMembership>`, `local_runtime_id`, `dispatcher`, `audio_fence`,
`runtime: MeshRuntime` (private), `owners`.

**Zero-reader field:** `MeshHandle.membership` is populated at `mesh_boot.rs:501`
and **never read**. grep for `.membership` field access on a `MeshHandle` across
`crates/buzz-relay/src` returns nothing; the three `.membership()` hits
(`mesh_boot.rs:173,488,501`) are all the `MeshRuntime` *method*, not the field.
Consequently `RelayMeshMembership::peers()` and `::local_runtime_id()` have **zero
production callers** (see §6).

---

#### 2. `MeshConfig` (`lib.rs:53-64`)

```rust
pub struct MeshConfig {
    pub enabled: bool,                          // lib.rs:57
    pub bind_addr: std::net::SocketAddr,        // lib.rs:60
    pub registry_refresh: std::time::Duration,  // lib.rs:63
}
```

`#[derive(Clone, Debug)]` (`lib.rs:52`). No `Default`, no builder, no
`Deserialize` — the relay hand-constructs it.

| Field | Constructed | Read |
|---|---|---|
| `enabled` | `config.rs:509` | `mesh_boot.rs:417` |
| `bind_addr` | `config.rs:510` | `mesh_boot.rs:383`, logged `mesh_boot.rs:396` |
| `registry_refresh` | `config.rs:511` (hardcoded 15 s) | `mesh_boot.rs:447` |

Struct field `Config.mesh` at `crates/buzz-relay/src/config.rs:136`.

---

#### 3. The two seam traits (the crate's documented contract)

`lib.rs:11-19` declares the relay consumes the crate "exclusively through two seams."
Verified: both are used, plus a third (`InboundHandler`) and the concrete
`MeshStream`/half traits, so the "exclusively two" claim is understated.

##### `RelayMeshMembership` (`lib.rs:144-151`)

| Method | Impl | Production callers |
|---|---|---|
| `peers() -> Vec<PeerInfo>` | `membership.rs:359-379` | **0** |
| `local_runtime_id() -> RuntimeId` | `membership.rs:381-383` | **0** |
| `begin_drain()` | `membership.rs:385-388` | 1 — `mesh_boot.rs:488` (drain watcher), reached via `MeshRuntime::membership()`, not via the trait object |

##### `RelayPeerTransport` (`lib.rs:158-175`)

| Method | Impl | Production callers |
|---|---|---|
| `send_datagram(to, dgram)` | `runtime.rs:167-175` | `audio/mesh.rs` media sink path (`audio/mesh.rs:37,56,260`); tests `mesh_boot.rs:660` |
| `open_session_stream(to, hello) -> BoxFuture<Result<MeshStream>>` | `runtime.rs:177-194` | `tunnel/reliable.rs:46,718`; `audio/join.rs:1002,1019,1483,1667`; `api/mesh_demo.rs:40,89` |
| `set_inbound(handler)` | `runtime.rs:196-198` | `mesh_boot.rs:509` (once, with `MeshInboundDispatcher`) |

Doc note at `lib.rs:161-163`: implementations do datagram-size + wire-version checks
but **not** generation fencing. Verified — `runtime.rs:167-194` touches neither
`generation` nor Redis.

##### `InboundHandler` (`lib.rs:177-180`)

`on_datagram(from, dgram)` / `on_session_stream(from, hello, stream)`.
Implemented by `MeshInboundDispatcher` (`mesh_boot.rs:95-133`) and by test doubles
(`runtime.rs:660` in-crate; `tunnel/reliable.rs:733`, `audio/join.rs:2198`,
`api/mesh_demo.rs:199`).

##### `MeshStream` + halves

`MeshStream` (`lib.rs:184-189`) is a concrete struct with `pub(crate)` fields; the
public constructor `MeshStream::new` lives in `peer.rs:199-201` and is documented as
existing so consumers can build in-memory streams in tests (`peer.rs:196-198`).
Verified consumers: `mesh_boot.rs:564` (`stub_stream`), `audio/join.rs:2083-2115`
(in-memory duplex over `tokio::sync::mpsc`).

`StreamSendHalf` (`lib.rs:191-194`: `send_frame`, `finish`) and `StreamRecvHalf`
(`lib.rs:196-198`: `recv_frame`) — production impls `IrohSendHalf`/`IrohRecvHalf`
(`peer.rs:132-192`); test impls at `mesh_boot.rs:566-580`, `audio/join.rs:2086-2109`.

`BoxFuture<'a, T>` (`lib.rs:141`) is public specifically so out-of-crate implementors
can name it (`lib.rs:138-140`); used at `mesh_boot.rs:566`, `audio/join.rs:2083`.

---

#### 4. `wire` — the frozen wire contract (`wire.rs`)

| Item | Location | External callers |
|---|---|---|
| `ALPN: &[u8] = b"buzz/mesh/1"` | `wire.rs:37` | none outside crate |
| `WIRE_VERSION: u8 = 1` | `wire.rs:42` | `mesh_boot.rs:367` (→ `proto_version`) |
| `MAX_STREAM_FRAME: u32 = 16 MiB` | `wire.rs:46` | `tunnel/reliable.rs:28` (doc), `:945` (test assert) |
| `RuntimeId` (+ `to_hex`, `Debug`, `Display`) | `wire.rs:62-81` | pervasive |
| `FencedHeader` | `wire.rs:85-93` | `tunnel/*`, `audio/*`, `api/mesh_demo.rs` |
| `Profile` (3 variants) | `wire.rs:97-109` | dispatcher routing `mesh_boot.rs:112-127` |
| `MeshDatagram` | `wire.rs:111-122` | `audio/mesh.rs`, `audio/join.rs` |
| `MeshStreamFrame` (4 variants) | `wire.rs:126-144` | `tunnel/reliable.rs`, `audio/join.rs` |
| `StreamRole` (2 variants) | `wire.rs:148-158` | `mesh_boot.rs:103`, `tunnel/reliable.rs`, `audio/join.rs` |
| `StreamHello` | `wire.rs:160-163` | as above |
| `GoodbyeReason` (3 variants) | `wire.rs:166-174` | `mesh_boot.rs:28`, `tunnel/reliable.rs`, `audio/join.rs` |
| `encode<T>` / `decode<T>` | `wire.rs:176-188` | in-crate only (`peer.rs:137,169,190`; test `tunnel/reliable.rs` uses the frame types, not the codec) |

**`#[non_exhaustive]` count: 0.** grep across `crates/buzz-relay-mesh/src` returns
nothing. Every public enum — including the "FROZEN" wire enums `MeshStreamFrame`,
`StreamRole`, `Profile`, `GoodbyeReason`, `GossipMessage`, `ConnectionState`, and
the 16-variant `MeshError` — is exhaustive. Adding a variant to any of them is a
breaking change for every downstream `match`, and the wire enums additionally have
no unknown-variant tolerance on decode: postcard rejects an out-of-range enum
discriminant, so a newer pod sending a 5th `MeshStreamFrame` variant produces
`MeshError::Decode` on an older pod, not a skip. The ALPN-bump discipline
(`wire.rs:34-36`) is the only mitigation, and it is a convention, not a mechanism.

---

#### 5. `endpoint` — transport bind/dial (`endpoint.rs`)

| Item | Location | Callers |
|---|---|---|
| `MeshEndpoint` | `endpoint.rs:12-15` | `mesh_boot.rs:25,383,422`; tests `tunnel/reliable.rs:663,782-785`, `api/mesh_demo.rs:152,272-273` |
| `MeshEndpoint::bind(SocketAddr)` | `endpoint.rs:19-21` | `mesh_boot.rs:383` — the only production bind |
| `MeshEndpoint::bind_with_secret_key(SecretKey, SocketAddr)` | `endpoint.rs:26-44` | tests only (in-crate `endpoint.rs:157-162`, `runtime.rs:627`; relay-side `tunnel/reliable.rs`, `api/mesh_demo.rs`) |
| `runtime_id()` | `endpoint.rs:47-49` | `mesh_boot.rs:386`; in-crate `runtime.rs:134,219,296,463` |
| `endpoint() -> iroh::Endpoint` | `endpoint.rs:51-53` | **0 callers anywhere** |
| `addr() -> EndpointAddr` | `endpoint.rs:55-57` | in-crate only (`endpoint.rs:64`, tests) |
| `ip_addrs() -> Vec<SocketAddr>` | `endpoint.rs:61-71` | `mesh_boot.rs:391-401` (`advertise_addrs`) |
| `accept()` / `connect(EndpointAddr)` | `endpoint.rs:73-93` | in-crate `runtime.rs:262,338`; relay tests |
| `runtime_id_from_public_key` | `endpoint.rs:96-98` | in-crate `endpoint.rs:38`, `peer.rs:58` |
| `public_key_from_runtime_id` | `endpoint.rs:100-102` | in-crate `endpoint.rs:105` |
| `direct_addr(RuntimeId, SocketAddr)` | `endpoint.rs:104-109` | in-crate `runtime.rs:333` |

`bind_with_secret_key` is documented "production should use `Self::bind`"
(`endpoint.rs:23-25`) — verified true.

---

#### 6. `membership` (`membership.rs`)

`MeshMembership` — 26 public methods. **Every one has zero callers outside the crate
except `status()` and `begin_drain()`**, both reached through
`MeshRuntime::membership()`:

| Method | Location | External callers |
|---|---|---|
| `new(GossipRecord)` | `:46-59` | `mesh_boot.rs:444` |
| `with_expected_relay_pubkey(String)` | `:61-64` | `mesh_boot.rs:445` |
| `with_phi_suspect_threshold(f64)` | `:66-69` | **0** — the 8.0 default is unoverridable in production |
| `local_record()` | `:71-75` | 0 (in-crate ×4) |
| `apply_ready_records(iter)` | `:85-118` | 0 (in-crate `runtime.rs:311,318`) |
| `apply_gossip_record(rec) -> bool` | `:120-153` | 0 (in-crate `runtime.rs:106,536`) |
| `mark_connection_state` | `:155-164` | 0 (in-crate ×4) |
| `update_local<F>` | `:166-178` | 0 (in-crate `runtime.rs:565`, `membership.rs:387`) |
| `is_draining()` | `:180-182` | 0 (the `is_draining` hits in `buzz-relay` are `HuddleOwnerRegistry`'s, `audio/join.rs:625`) |
| `has_peer(RuntimeId)` | `:187-192` | 0 (in-crate `runtime.rs:306,321`) |
| `records()` | `:195-206` | 0 (in-crate `runtime.rs:299`, `membership.rs:210,232`) |
| `digest()` | `:208-223` | 0 (in-crate `runtime.rs:567`) |
| `delta_for(&[GossipDigestEntry])` | `:225-247` | 0 (in-crate `runtime.rs:539`) |
| `record_stream_opened/received`, `record_datagram_sent/received`, `record_gossip_frame_sent/received` (6) | `:249-283` | 0 (in-crate, one call site each) |
| `record_stale_generation_rejection(Option<RuntimeId>)` | `:285-293` | **0 anywhere** — only the test at `:486` |
| `status() -> MeshStatus` | `:295-313` | `mesh_boot.rs:173` |

##### `PeerInfo` (`lib.rs:129-139`)

Referenced in `buzz-relay` only at `tunnel/reliable.rs:664` (test import) and
`:949` — a `#[allow(dead_code)] fn _peer_info_is_not_an_owner_signal(_peer: PeerInfo)`
that exists purely to document the fencing law. **No production code reads a
`PeerInfo`.**

---

#### 7. `runtime` (`runtime.rs`)

| Item | Location | Callers |
|---|---|---|
| `DEFAULT_RECONCILE_INTERVAL = 5s` | `:42` | in-crate `:97` |
| `DEFAULT_GOSSIP_INTERVAL = 2s` | `:44` | in-crate `:96` |
| `MeshRuntime` | `:77-80` | `mesh_boot.rs:29,161,218,473` |
| `MeshRuntime::start(endpoint, membership, Option<registry>)` | `:88-100` | `mesh_boot.rs:473` |
| `MeshRuntime::start_with_intervals(..., gossip, reconcile)` | `:102-127` | **0 external** (in-crate tests `:632`) |
| `membership() -> &MeshMembership` | `:129-131` | `mesh_boot.rs:173,488,501` |
| `local_runtime_id()` | `:133-135` | **0 external** (tests only) |
| `connected_peers() -> Vec<RuntimeId>` | `:138-146` | **0 anywhere outside in-crate tests** |
| `reconcile_now()` | `:150-152` | `mesh_boot.rs:478` |
| `shutdown()` | `:155-164` | **0 external** — the relay never calls it; only in-crate tests do |
| `spawn_registry_heartbeat(registry, record, ready)` | `:594-608` | `mesh_boot.rs:467` |

`MeshRuntime::shutdown()` being uncalled is a real leak: `MeshRuntime` doc
(`runtime.rs:75-76`) says "dropping all clones does NOT stop the loops — call
`shutdown()` for that." The relay's drain watcher (`mesh_boot.rs:481-497`) calls
`begin_drain()` + `owners.drain_all()` and returns; the accept/reconcile/gossip
loops and the registry heartbeat run until process exit. Registry cleanup relies on
`ReadyHeartbeat::tick(false)` (`registry.rs:299-302`) — and `ReadyHeartbeat::shutdown()`
(`registry.rs:306-312`) also has **zero callers**.

---

#### 8. `registry` (`registry.rs`)

| Item | Location | Callers |
|---|---|---|
| `READY_KEY_PREFIX`, `DEFAULT_REGISTRY_REFRESH`, `REGISTRY_EXPIRY_MULTIPLIER`, `ATTESTATION_CONTEXT` | `:19-22` | **0 external each**; `DEFAULT_REGISTRY_REFRESH` is not even used by the relay, which hardcodes 15 s at `config.rs:511` |
| `RuntimeAttestation` + `new` + `verify` | `:30-51` | `new` in-crate `:121`; **`verify` has 0 callers anywhere** |
| `attestation_preimage` | `:85-91` | in-crate `:94` only |
| `ReadyRecord` + `new` + `key` + `verify_attestation` | `:99-147` | `new` → `mesh_boot.rs:448`; `key`/`verify_attestation` in-crate |
| `ready_key(RuntimeId)` | `:150-152` | in-crate `:136`, `:200` |
| `expiry_for(Duration)` | `:154-156` | in-crate `:175` |
| `ReadyRegistry::new / refresh_interval / expiry / publish_ready / clear_ready / scan_ready / heartbeat` | `:166-262` | `new` → `mesh_boot.rs:447`; `publish_ready` → `mesh_boot.rs:459`; the rest in-crate (`runtime.rs:311,318,601,604`) |
| `ReadyHeartbeat::record / published / tick / shutdown` | `:287-312` | `tick` in-crate `:602`; `record`/`published`/`shutdown` → **0 production callers** |

---

#### 9. `gossip` (`gossip.rs`)

`GOSSIP_PAYLOAD_VERSION` (`:13`), `GossipRecord` + `new` (`:16-42`),
`GossipDigestEntry` (`:45-48`), `GossipMessage` (`:51-60`), `encode_message` /
`decode_message` (`:62-77`), `GossipState` + 7 methods (`:81-166`), `PhiAccrual` +
`new/observe/phi_at/mean_secs` (`:168-220`), `now_millis` (`:223-229`),
`system_time_from_millis` (`:231-233`).

External callers: **`GossipRecord` and `GossipRecord::new` only**
(`mesh_boot.rs:26,439`). Everything else in this module is in-crate or unused:

- `GossipState` and all 7 of its methods — **zero callers outside its own tests**.
- `PhiAccrual::new` (only `Default` is used, `membership.rs:137`),
  `PhiAccrual::mean_secs` (in-crate `:212` only), `GossipDigestEntry`,
  `GossipMessage`, `encode_message`/`decode_message`, `now_millis`,
  `system_time_from_millis` — all zero external callers.

---

#### 10. `status` (`status.rs`) and `lib` root

`MeshStatus`, `MeshPeerStatus`, `ConnectionState`, `MeshCounters`, `MeshPeerCounters`
— all re-exported (`lib.rs:46`). External use: `MeshStatus` is named at
`mesh_boot.rs:29,172` and serialized at `router.rs:381`. The other four have zero
direct external references (they arrive as JSON through `serde_json::to_value`).

`ConnectionState` appears to have 25 hits in `buzz-relay` — **all false positives**:
that is the relay's own WebSocket `ConnectionState` (`connection.rs:53`,
`handlers/auth.rs:17`, etc.), an unrelated type.

Root-level items: `MeshConfig` (§2), `MeshError` (16 variants, `lib.rs:65-127`),
`PeerInfo` (`lib.rs:129-139`), `BoxFuture` (`lib.rs:141`), the 4 traits, `MeshStream`,
and `encode_datagram_checked` (`lib.rs:213-226`) — the last has **zero callers
outside `peer.rs:110`** despite being documented as a "raw bytes helper used by
transport internals."

`peer::PeerCounters` (`peer.rs:10-15`) and `MeshPeer::counters()` (`peer.rs:73-75`)
are public with **zero callers anywhere** — the atomics are incremented
(`peer.rs:84,97,114,121`) and never read.

---

#### 11. Public-surface summary

| Category | Count | Zero-external-caller count |
|---|---|---|
| Public modules | 8 | 0 |
| Public traits | 4 (`RelayMeshMembership`, `RelayPeerTransport`, `InboundHandler`, `StreamSendHalf`+`StreamRecvHalf` = 5) | 0 |
| Public structs | 17 | 8 (`GossipState`, `PhiAccrual`, `GossipDigestEntry`, `RuntimeAttestation`, `ReadyHeartbeat`, `PeerCounters`, `MeshCounters`, `MeshPeerCounters`) |
| Public enums | 7 | 1 (`GossipMessage`) |
| Public consts | 9 | 7 |
| Free functions | 11 | 8 |
| `#[non_exhaustive]` | **0** | — |
| `#[deprecated]` | 0 | — |

Roughly **40% of the public surface has no caller outside the crate**, concentrated
in `gossip::GossipState` (a full duplicate of `MeshMembership`'s logic), the
registry's constant/helper layer, and the two parallel counter models.


## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: API Surface

The crate's entire surface is a Rust library API. It exposes **no HTTP route, no WebSocket
frame, no CLI subcommand, and no Nostr event kind** — the only dependencies are `serde`,
`serde_json`, `thiserror`, `uuid` (`crates/buzz-conformance/Cargo.toml:26-29`), none of which
carry a transport. `crates/buzz-relay/src/conformance/mod.rs` re-exports the types into the
relay but adds no route either; the relay's router is untouched by this module.

Two crate-level lints frame the whole API: `#![deny(unsafe_code)]` and
`#![warn(missing_docs)]` (`src/lib.rs:38-39`). Two public modules:
`checker` (`src/lib.rs:41`) and `transitions` (`src/lib.rs:42`).

---

### Exported value types (`src/lib.rs`)

| Item | Line | Shape | Notes |
|---|---|---|---|
| `CommunityLabel(pub uuid::Uuid)` | `src/lib.rs:66` | `#[serde(transparent)]` newtype | field is `pub`, so construction is unrestricted |
| `CommunityLabel::from_uuid` | `src/lib.rs:74` | `pub const fn (Uuid) -> Self` | the documented conversion seam |
| `impl Display for CommunityLabel` | `src/lib.rs:79-83` | forwards to the inner `Uuid` | the only `Display` impl in the crate |
| `SCHEMA_VERSION: u32 = 1` | `src/lib.rs:86` | `pub const` | compared by `check_trace` |
| `OpaqueId(pub String)` | `src/lib.rs:93` | transparent newtype | |
| `HostLabel(pub String)` | `src/lib.rs:100` | transparent newtype | |
| `ChannelLabel(pub uuid::Uuid)` | `src/lib.rs:106` | transparent newtype | |
| `ActorLabel(pub String)` | `src/lib.rs:112` | transparent newtype | |
| `Verdict { Allow, Deny }` | `src/lib.rs:117-123` | `snake_case` serde | |
| `SanitizedReason { Restricted, Invalid, ServerError }` | `src/lib.rs:132-140` | `snake_case` serde | |
| `AbstractState` | `src/lib.rs:150-175` | 3 pub fields | `resolved_community`, `bound_host`, `actor` |
| `TraceAction` | `src/lib.rs:179-261` | 9 variants, `#[serde(tag = "type")]` | see data-model doc for the variant table |
| `TraceAction::kind()` | `src/lib.rs:266` | `pub fn (&self) -> &'static str` | canonical coverage-set strings |
| `TraceAction::is_critical()` | `src/lib.rs:283` | `pub const fn (&self) -> bool` | returns `true` unconditionally (`src/lib.rs:284`); **zero callers repo-wide** |
| `TraceStep` | `src/lib.rs:290-299` | 3 pub fields | `schema_version`, `action`, `state_after` |
| `TraceStep::new` | `src/lib.rs:303` | `pub fn (TraceAction, AbstractState) -> Self` | stamps `SCHEMA_VERSION`; no version-override constructor exists |

All newtype inner fields are `pub`, so any consumer can mint any label. The only fenced type
is the production `buzz_core::CommunityId` the crate deliberately does **not** reuse
(`Cargo.toml:8-18`, `src/lib.rs:47-63`).

---

### The tracer trait — the sole production-facing surface

```
pub trait Tracer: Send + Sync {
    fn record(&self, step: TraceStep);
}
```
`src/lib.rs:314-318`. One method, no return value, no error channel, `&self` (interior
mutability is the implementor's problem). `Send + Sync` is what lets the relay hold it as
`Arc<dyn buzz_conformance::Tracer>` in `AppState` (`crates/buzz-relay/src/state.rs:620`).

`NoopTracer` (`src/lib.rs:323`, `impl` at `:325-327`) is a unit struct deriving
`Debug, Default, Clone, Copy` — its `record` body is empty (`src/lib.rs:326`).

Note the duplication: the relay declares a **second, independent** `NoopTracer`
(`crates/buzz-relay/src/conformance/tracers.rs:16-20`) with the same name and behavior, and
re-exports *that* one (`crates/buzz-relay/src/conformance/mod.rs:46`). The crate's own
`NoopTracer` therefore has no consumer; `AppState::default` binds the relay's copy
(`crates/buzz-relay/src/state.rs:798`).

---

### `checker` module (`src/checker.rs`)

| Item | Line | Signature |
|---|---|---|
| `Scenario` | `:24-38` | struct with `pub trace: Vec<TraceStep>` (`:29`) and `pub required_critical_actions: HashSet<String>` (`:37`) |
| `Scenario::unstructured` | `:45` | `pub fn (Vec<TraceStep>) -> Self` — empty coverage set |
| `Scenario::require` | `:54` | `pub fn (self, &str) -> Self` builder; **zero callers repo-wide** — every construction site uses `unstructured` or a struct literal |
| `check_trace` | `:74` | `pub fn (&Scenario) -> Result<(), TransitionError>` |

`check_trace` is the crate's whole entry point. Four stages, documented at `:63-73` and
implemented as: empty-trace guard (`:75-82`), first-step schema check (`:84-93`), bootstrap
(`:94`), per-step loop with a redundant per-step schema check (`:96-110`), coverage-set diff
(`:113-129`). Callers observed: only the crate's own tests
(`src/checker.rs`, `tests/proptest_checker.rs`, `tests/replay_fixtures.rs`) — no production
call site exists anywhere in `crates/`.

---

### `transitions` module (`src/transitions.rs`)

| Item | Line | Signature / shape |
|---|---|---|
| `Verdict_ { Ok }` | `:53-56` | `pub enum`, doc says "Reserved — internal placeholder"; zero references |
| `TransitionError` | `:61-102` | `thiserror::Error`, 4 variants: `IllegalTransition{step_index,detail}` (`:65`), `StateMismatch{…}` (`:77`), `NonInterference{…}` (`:87`), `CoverageBreach{detail}` (`:97`) |
| `ModelState` | `:105-118` | `pub` fields `resolved_community` (`:108`), `bound_host` (`:115`), `actor` (`:117`); `Debug + Clone` only — no `PartialEq` |
| `ModelState::bootstrap` | `:123` | `pub fn (&AbstractState) -> Self` |
| `check_step` | `:138` | `pub fn (usize, &ModelState, &TraceStep) -> Result<(), TransitionError>` |
| `action_channel` | `:318` | `pub fn (&TraceAction) -> Option<&ChannelLabel>`; **zero callers repo-wide** |
| `check_row_labels` | `:294` | private `fn (usize, &ModelState, &[CommunityLabel])` |

`TransitionError` carries no machine-readable payload beyond `step_index` — the offending
community/host/channel values are formatted into the `detail: String`
(`:146-151`, `:236-241`, `:304-309`), so a mechanical consumer can match the variant but not
extract the values.

---

### Feature gates

`crates/buzz-conformance/Cargo.toml` has **no `[features]` section** — verified by grep. There
is no `cfg` gate on any public item in `src/`; the emit path is switched at runtime by which
`Tracer` impl is bound, not at compile time. `src/lib.rs:321-322` and
`crates/buzz-relay/src/conformance/tracers.rs:9-13` both describe a hypothetical
feature-flagged elimination ("the build can omit emission entirely behind a feature") that
does not exist.

---

### Relay-side helper API (`crates/buzz-relay/src/conformance/mod.rs`)

Public because `handlers/ingest.rs` and `handlers/req.rs` call across module boundaries;
not part of `buzz-conformance` itself.

| Item | Line | Purpose |
|---|---|---|
| re-export block | `:40-44` | `AbstractState, ActorLabel, ChannelLabel, CommunityLabel, HostLabel, OpaqueId, SanitizedReason, TraceAction, TraceStep, Tracer, Verdict` |
| `state_for_request` | `:55` | `(&TenantContext, &PublicKey) -> AbstractState` |
| `actor_label` (private) | `:70` | pubkey hex prefix, ≤16 chars |
| `msg_id_label` | `:78` | first 8 bytes of the event id, hex |
| `channel_label` | `:89` | direct `Uuid` wrap |
| `claimed_community_from_event` | `:101` | first `h` tag parsed as `Uuid`, else `None` |
| `step` | `:121` | thin `TraceStep::new` wrapper; only caller is… none (grep finds no `conformance::step(` call) |
| `emit` | `:127` | `tracer.record(TraceStep::new(...))` |
| `record_req_authcheck` | `:148` | REQ-path `AuthCheck`, `claimed_community` hard-wired to `None` (`:152`) |
| `project_row_community` (private) | `:186` | per-row label, `None` on lookup miss |
| `RowCommunityProjection { Ok, MissingLookup }` | `:210-224` | discriminated projection outcome |
| `project_row_communities` | `:234` | vectorized projection |
| `record_read_message_rows` | `:265` | `ReadMessageRows` or `ImplBug` |
| `record_read_by_id_rows` | `:300` | `ReadByIdRows` or `ImplBug` |
| `EmitGuard` | `:344-354` | RAII coverage guard |
| `EmitGuard::arm` | `:383` | `(Arc<dyn Tracer>, AbstractState, &'static str) -> (Self, Arc<dyn Tracer>)` |
| `impl Drop for EmitGuard` | `:403-415` | emits `ImplBug` when the counter is 0 |
| `sanitized_reason_for` | `:422` | `&IngestError -> SanitizedReason` |
| `JsonlTracer` / `NoopTracer` re-export | `:46` | from `tracers.rs:16`, `tracers.rs:30` |
| `JsonlTracer::create` | `tracers.rs:37` | `<P: AsRef<Path>>(P) -> io::Result<Self>`; truncating open (`:39-43`) |

`CountingTracer` (`:356-359`) is private; it is the `Arc<dyn Tracer>` `EmitGuard::arm` hands
back, and its `record` bumps a `Relaxed` atomic before delegating (`:368-372`).

There is **no relay helper for `TraceAction::ReadHostFeedRows`** — grep for
`ReadHostFeedRows` across `crates/buzz-relay/` returns nothing. The variant is
constructible only from tests (`tests/proptest_checker.rs:164-166` and `:255-257`).


## Module: buzz-acp — harness core & orchestration (`crates/buzz-acp/src`)
### Aspect: API Surface

#### Public Rust surface: two items

```
pub use usage::TurnUsage;      // lib.rs:15
pub fn run() -> Result<()>     // lib.rs:1233
```

That is the entire `pub` surface of the 6,570-line crate root (`grep -n '^pub ' lib.rs` → lines 15 and 1233). Everything else is private or `pub(crate)`. The one `pub(crate)` item is `is_dm_channel` at `lib.rs:273-286`, exposed only so the in-file `author_gate_tests` module can drive it.

`pub fn run()` at `lib.rs:1233` **has no doc comment**. The block above it (`lib.rs:1226-1231`) is `//`, not `///`, and is separated by a blank line — so the single exported function of the crate violates the repo's "new public API must have doc comments" rule (`AGENTS.md § Quality Gates`).

`TurnUsage` is re-exported but never referenced by `lib.rs` itself; it exists for `sprig`/downstream consumers.

#### Binary surface

`src/main.rs` is three lines and delegates entirely:

```rust
fn main() -> anyhow::Result<()> { buzz_acp::run() }   // main.rs:1-3
```

`run()` does sync legacy-env propagation then enters `#[tokio::main] async fn tokio_main()` (`lib.rs:1233-1239`).

#### CLI subcommands — hand-rolled dispatch, not clap subcommands

`is_subcommand(name)` (`lib.rs:58-60`) reads `std::env::args().nth(1)` and compares strings *before* any clap parsing. Each match then rebuilds argv with index 1 filtered out and re-parses with a dedicated `Parser` (`lib.rs:1245-1279`).

| Subcommand | Dispatch | Handler | Parser | Timeout |
|---|---|---|---|---|
| `models` | `lib.rs:1245-1254` | `run_models` `lib.rs:4005-4139` | `ModelsArgs` | `MODELS_TIMEOUT` 10 s (`lib.rs:63`) |
| `auth-methods` | `lib.rs:1256-1264` | `run_auth_methods` `lib.rs:3899-3945` | `AuthMethodsArgs` | 10 s |
| `authenticate` | `lib.rs:1266-1274` | `run_authenticate` `lib.rs:3947-4003` | `AuthenticateArgs` | `AUTHENTICATE_TIMEOUT` 600 s (`lib.rs:67`) |
| *(default)* | `lib.rs:1276+` | main harness loop | `CliArgs` via `Config::from_cli()` (`lib.rs:1290`) | — |

Documented constraint at `lib.rs:52-56`: the subcommand **must** be argv[1]. `buzz-acp --verbose models` is not supported. None of the three subcommands is documented in `crates/buzz-acp/README.md` — the README's only CLI content is flags for the default path.

All three subcommands use `std::process::exit(1)` on failure rather than returning `Err` (`lib.rs:3904`, `3910`, `3915`, `3952`, `3958`, `3964`, `3974`, `3986`, `3992`, `4020`, `4034`, `4039`), so `anyhow` context is discarded and the exit code is always 1.

`run_models --json` emits a stable envelope (`lib.rs:4084-4098`) with `agent.{name,version}`, `stable.configOptions`, `unstable.{currentModelId,availableModels}`; the comment at `lib.rs:4083` names its consumer as "Phase 3 `get_agent_models`".

#### Nostr event kinds published by `lib.rs`

| Kind | Constant | Publish site | Transport |
|---|---|---|---|
| 20001 | `KIND_PRESENCE_UPDATE` (`buzz-core/src/kind.rs:327`) | `publish_presence` `lib.rs:77-91` | WebSocket only — comment at `lib.rs:71-72` states ephemeral kinds are rejected by the HTTP bridge |
| 20002 | `KIND_TYPING_INDICATOR` (`kind.rs:331`) | built by `relay.build_typing_event` and fired at `lib.rs:2332-2340` on a 3 s tick (`lib.rs:1593-1599`) | WebSocket, `try_publish_event` (non-blocking) |
| 24200 | `KIND_AGENT_OBSERVER_FRAME` (`kind.rs:333`) | `publish_relay_observer_event` `lib.rs:790-833` via `buzz_sdk::build_agent_observer_frame` (`lib.rs:810`) | WebSocket |

Presence content is a bare status string `"online"` / `"offline"` with no tags (`lib.rs:85-87`); the initial `online` is published *after* channel subscriptions so it doubles as a readiness signal (`lib.rs:1511-1515`), and `offline` is best-effort with a 2 s timeout at shutdown (`lib.rs:2679-2692`). A 60 s presence heartbeat re-publishes `online` (`lib.rs:1583-1591`, fired `lib.rs:2301-2320`).

#### Nostr event kinds consumed

Default subscription kinds when `subscribe_mode == Mentions` (`lib.rs:1442-1450`):

| Kind | Constant | Value |
|---|---|---|
| `KIND_STREAM_MESSAGE` | `kind.rs:343` | 9 |
| `KIND_WORKFLOW_APPROVAL_REQUESTED` | `kind.rs:442` | 46010 |
| `KIND_STREAM_REMINDER` | `kind.rs:355` | 40007 |

Overridable wholesale by `config.kinds_override` (`--kinds` / `BUZZ_ACP_KINDS`). `SubscribeMode::All` uses `kinds_override.unwrap_or_default()` — i.e. an **empty** kinds list (`lib.rs:1456`), which per `AGENTS.md § Common Gotchas #2` trips the relay p-gate.

Control-plane kinds handled inline in the main loop:

| Kind | Constant | Value | Handling |
|---|---|---|---|
| member added | `KIND_MEMBER_ADDED_NOTIFICATION` | 44100 (`kind.rs:396`) | dedup → subscribe new channel (`lib.rs:1891-1907`) |
| member removed | `KIND_MEMBER_REMOVED_NOTIFICATION` | 44101 (`kind.rs:400`) | unsubscribe, drain queue, invalidate sessions, remove 👀 (`lib.rs:1908-1946`) |
| observer control | `KIND_AGENT_OBSERVER_FRAME` | 24200 | `handle_relay_observer_control_event` `lib.rs:837-893` |

Every kind integer resolves through `buzz_core::kind` constants (`lib.rs:23-26`, `lib.rs:82`). **No bare kind literal exists in production code** — `grep -n 'Kind::Custom([0-9]'` finds `Kind::Custom(9)` only inside `#[cfg(test)]` fixtures (`lib.rs:5442`, `5548`, `5670`, `5763`, `5829`, `5832`, `6157`, `6242`).

#### In-band owner control commands (text protocol over kind 9)

`is_owner_control_command` (`lib.rs:2718-2727`) requires kind == 9 **and** `content.trim() == command` **and** a `p` tag equal to this agent's pubkey.

| Command | Site | Effect |
|---|---|---|
| `!shutdown` | `lib.rs:2033-2059` | `shutdown_tx.send(())` → graceful exit |
| `!cancel` | `lib.rs:2064-2092` | `ControlSignal::Cancel` to in-flight task; no-op when idle |
| `!rotate` | `lib.rs:2102-2133` | cancel + rotate if in flight, else invalidate idle session |

All three fall through to normal prompt handling when the sender is not the owner (`lib.rs:2056-2058`, `2090-2091`, `2130-2131`) — the string is then delivered to the agent as ordinary content.

#### Encrypted observer control frames (owner → harness)

`handle_relay_observer_control_event` (`lib.rs:837-893`) accepts two `type` values after signature, sender, and freshness checks:

| `type` | Handler | Result statuses emitted |
|---|---|---|
| `cancel_turn` | `lib.rs:895-938` | `sent`, `no_active_turn` |
| `switch_model` | `lib.rs:940-1005` | `sent`, `turn_ending`, `switched`, `unsupported_model`, `no_active_turn` |

Unknown `type` values are logged at debug and dropped (`lib.rs:888-891`). Outcomes are reported back as `control_result` observer events, not as JSON-RPC replies.


## Module: buzz-acp — relay client & observer (`crates/buzz-acp/src`)
### Aspect: API Surface

#### `HarnessRelay` — public methods

| Method | Location | Signature notes |
|---|---|---|
| `connect` | `relay.rs:596-601` | `async (relay_url: &str, keys: &Keys, agent_pubkey_hex: &str, auth_tag: Option<nostr::Tag>) -> Result<Self, RelayError>`; spawns the background task |
| `discover_channels` | `relay.rs:656` | `async (&self) -> Result<HashMap<Uuid, ChannelInfo>, RelayError>` |
| `rest_client` | `relay.rs:719` | `(&self) -> RestClient` |
| `subscribe_channel` | `relay.rs:735-739` | `async (&mut self, Uuid, ChannelFilter)` → delegates to `subscribe_channel_from(.., None)` |
| `subscribe_channel_from` | `relay.rs:749-754` | adds `replay_since: Option<u64>` |
| `subscribe_membership_notifications` | `relay.rs:768` | `async (&mut self)` |
| `subscribe_observer_controls` | `relay.rs:777` | `async (&mut self)` |
| `take_observer_control_rx` | `relay.rs:786` | `(&mut self) -> Option<mpsc::Receiver<Event>>` — one-shot take |
| `event_publisher` | `relay.rs:791` | `(&self) -> RelayEventPublisher` |
| `unsubscribe_channel` | `relay.rs:798` | `async (&mut self, Uuid)` |
| `next_event` | `relay.rs:806` | `async (&mut self) -> Option<BuzzEvent>` |
| `publish_event` | `relay.rs:821` | `async (&self, Event)`; marked `#[allow(dead_code)]` at `relay.rs:820` |
| `try_publish_event` | `relay.rs:834` | `(&self, Event) -> Result<(), RelayError>` — `try_send`, never blocks |
| `build_typing_event` | `relay.rs:843-848` | `(&self, Uuid, root: Option<&str>, parent: Option<&str>) -> Result<Event, RelayError>` |
| `set_startup_watermark` | `relay.rs:877` | `async (&self, ts: u64)` |
| `reconnect` | `relay.rs:886` | `async (&mut self)` |
| `shutdown` | `relay.rs:900` | `async (self)` — consumes, waits 5 s, then aborts |

`Drop for HarnessRelay` (`relay.rs:915-923`) sends `Shutdown` via `try_send`
then `abort()`s the task immediately — no close frame guarantee.

All the mutating subscribe/unsubscribe methods take `&mut self` but only push a
`RelayCommand`; every one returns `Ok(())` on a successful enqueue, not on a
successful REQ. `RelayError::ConnectionClosed` here means "background task
gone", not "socket down".

#### `RelayEventPublisher`

| Method | Location |
|---|---|
| `publish_event` | `relay.rs:563-573` — `async (&self, Event) -> Result<(), RelayError>` |
| `test_pair` | `relay.rs:575-588`, `#[cfg(test)]` only |

#### `RestClient` — the HTTP bridge surface

This module **does** use the relay's HTTP bridge for every read, in addition to
the WebSocket:

| Method | Location | HTTP |
|---|---|---|
| `query` | `relay.rs:399-406` | `POST {base_url}/query`, body = JSON array of `nostr::Filter` |
| `submit_event` | `relay.rs:411-423` | `POST {base_url}/events`, body = signed event JSON |
| `bridge_post` (private) | `relay.rs:368-393` | adds `Authorization: Nostr <b64>` and optional `x-auth-tag` |
| `request_with_retry` (private) | `relay.rs:314-361` | 1 attempt + 3 retries on 429/502/503/504/timeout/connect |
| `sign_nip98` / `nip98_header` (private) | `relay.rs:266-305` | kind:27235 NIP-98 with `u`, `method`, `nonce`, optional `payload` sha256 tags |

`base_url` is derived from the WebSocket URL by string replacement
(`relay_ws_to_http`, `relay.rs:3470-3475`). Auth is NIP-98 (signed kind:27235),
**not** the unauthenticated `X-Pubkey` header path — so the
`BUZZ_REQUIRE_AUTH_TOKEN=false` impersonation weakness on
`POST /events|/query|/count` is not exercised by this module. The optional
`x-auth-tag` header carries the NIP-OA attestation JSON (`relay.rs:389-391`,
built at `relay.rs:724-727`).

#### Event kinds published

| Kind | Constant | Where built | Transport |
|---|---|---|---|
| 22242 (`Kind::Authentication`) | `nostr::Kind::Authentication` | `relay.rs:3452` / `EventBuilder::auth` at `relay.rs:3457` | WS `["AUTH", ev]` |
| 27235 (`Kind::HttpAuth`) | `nostr::Kind::HttpAuth` | `relay.rs:292` | HTTP `Authorization` header |
| 20002 | `KIND_TYPING_INDICATOR` (`buzz-core/src/kind.rs:331`) | `relay.rs:866` | WS `["EVENT", ev]` |
| 24200 | `KIND_AGENT_OBSERVER_FRAME` (`buzz-core/src/kind.rs:333`) | built in `lib.rs` via `buzz_sdk::build_agent_observer_frame`; routed through `RelayCommand::PublishEvent` and detected by kind at `relay.rs:1234`, `:1310`, `:1470`, `:1497` | WS `["EVENT", ev]` |

Anything else reaching `PublishEvent` (kind:20001 presence from `lib.rs:85`,
kind:44200 metrics via `RestClient::submit_event`) is treated as droppable
ephemera by the rate-limit gate — see `relay.rs:1483-1493` for the stated
invariant.

#### REQ filter shapes

| Subscription id | Builder | Filter |
|---|---|---|
| `ch-<uuid>` (`channel_sub_id`, `relay.rs:3478`) | `send_subscribe`, `relay.rs:3160-3222` | `kinds` **only if** `filter.kinds.is_some()` (`relay.rs:3172-3175`); `#h: [channel_uuid]` always (`:3178`); `#p: [agent_pubkey]` only when `require_mention` (`:3181-3183`); `since` always (`:3187-3194`) |
| `membership-notif` (`relay.rs:497`) | `send_membership_subscribe`, `relay.rs:3227-3270` | `kinds: [44100, 44101]` via `KIND_MEMBER_ADDED_NOTIFICATION` / `KIND_MEMBER_REMOVED_NOTIFICATION` (`:3233-3239`); `#p: [agent_pubkey]` (`:3240`); `since` (`:3242-3250`) — no `#h` (global) |
| `agent-observer-control` (`relay.rs:499`) | `send_observer_control_subscribe`, `relay.rs:3273-3305` | `kinds: [24200]` (`:3278`); `#p: [agent_pubkey]` (`:3279`); `since: now` (`:3280-3283`) — no `#h`, no `authors` |

`AGENTS.md` requires every REQ to carry `kinds`. The channel REQ can omit it:
`ChannelFilter.kinds` is `Option<Vec<u32>>` with `None` documented as wildcard
(`config.rs:479-480`), and `--subscribe-mode all` without `--kinds` produces
exactly that (`config.rs:1180`, `config.rs:1272`), as does a `Config`-mode rule
with an empty `kinds` list (`config.rs:1195-1196`, `config.rs:1286-1287`).
`send_subscribe` then emits a REQ with only `#h` + `since`, which trips the
relay p-gate. No guard or warning exists at the send site.

#### HTTP query filters

| Call site | Filter |
|---|---|
| `discover_channels` step 1 | `kind: 39002` (`KIND_NIP29_GROUP_MEMBERS`) + `#p: [agent_pubkey]` — `relay.rs:662-668` |
| `discover_channels` step 2 | `kind: 39000` (`KIND_NIP29_GROUP_METADATA`) + `#d: [uuid…]` — `relay.rs:700-706` |
| `fetch_core_body` | `kind: 30174` (`KIND_AGENT_ENGRAM`) + `author: agent_pubkey` + `#d: [d_tag]` + `#p: [owner_hex]` + `limit(16)` — `engram_fetch.rs:79-89` |

All three specify `kinds`, so none trips the p-gate.

#### `engram_fetch` surface

| Item | Location | Signature |
|---|---|---|
| `ONBOARDING_NUDGE` (`pub const`) | `engram_fetch.rs:29-31` | `&'static str` |
| `build_core_section` (`pub`) | `engram_fetch.rs:39-44` | `async (&RestClient, &Keys, &PublicKey) -> Option<String>` |
| `fetch_core_body` (private) | `engram_fetch.rs:71-76` | `async (...) -> Result<Option<String>, String>` |
| `decode_core_body` (private, pure) | `engram_fetch.rs:110-115` | `(&[Value], &Keys, &PublicKey) -> Result<Option<String>, String>` |

Only `build_core_section` is called outside the module — from
`pool.rs:1387-1391`, wrapped in a 3 s timeout (`pool.rs:1386`, `:1394`).

#### `observer` surface

| Item | Location | Signature |
|---|---|---|
| `ObserverHandle::in_process` | `observer.rs:83-85` | `() -> Self` |
| `ObserverHandle::subscribe` | `observer.rs:88-90` | `(&self) -> broadcast::Receiver<ObserverEvent>` |
| `ObserverHandle::snapshot` | `observer.rs:93-102` | `(&self) -> Vec<ObserverEvent>` |
| `ObserverHandle::emit` | `observer.rs:105-136` | `(&self, kind: impl Into<String>, agent_index: Option<usize>, &ObserverContext, payload: Value)` — infallible, no return value |
| `context_for` | `observer.rs:138-149` | `(Option<Uuid>, Option<String>, Option<String>) -> ObserverContext` |
| `context_for_turn` | `observer.rs:152-164` | `(Option<Uuid>, Option<String>, String, String) -> ObserverContext` |

The module doc explicitly states this is process-local and deliberately exposes
no local HTTP port (`observer.rs:1-6`). All relay publication of these events
happens in `lib.rs` (`spawn_relay_observer_publisher`, `run_relay_observer_publisher`,
`publish_relay_observer_event`), not here.

#### `pub(crate)` items in `relay.rs`

`channel_type_from_tags` (`:138`), `merge_discovered_channels` (`:171`),
`parse_relay_message` (`:3533`, annotated `#[allow(private_interfaces)]` at
`:3532` because it returns the private `RelayMessage`),
`parse_rate_limit_retry_secs` (`:3328`), `is_dns_error` (`:3354`),
`relay_ws_to_http` (`:3470`), `channel_sub_id` (`:3478`).


## Module: buzz-acp — agent pool & lifecycle (`crates/buzz-acp/src`)
### Aspect: API Surface

#### Visibility reality

Both files are private modules: `mod pool;` (`lib.rs:7`) and `mod pool_lifecycle;` (`lib.rs:8`). The crate's only public re-export is `pub use usage::TurnUsage` (`lib.rs:13`). Every `pub` item in `pool.rs` is therefore reachable only inside `buzz_acp` — the `pub` keyword is decorative at the crate boundary, and `pool.rs` mixes `pub`, `pub(crate)`, and private for items with identical reach (e.g. `pub fn try_claim` at `pool.rs:558` vs `pub(crate) fn prepend_base_for_legacy` at `pool.rs:1090`).

`pool_lifecycle.rs` is uniformly `pub(crate)` (`pool_lifecycle.rs:14`, `:28`, `:37`, `:60`, `:70`, `:77`, `:84`, `:91`, `:99`).

#### `AgentPool` inherent methods (`impl` block `pool.rs:534-763`)

| Method | Site | Signature summary |
|---|---|---|
| `from_slots` | `pool.rs:541` | `Vec<Option<OwnedAgent>> -> Self` — only constructor |
| `try_claim` | `pool.rs:558` | `Option<Uuid> -> Option<OwnedAgent>`; two-pass (affinity, then any idle) |
| `return_agent` | `pool.rs:577` | `OwnedAgent -> ()`; logs `BUG:` and overwrites on double-return |
| `any_idle` | `pool.rs:593` | `-> bool` |
| `has_session_for` | `pool.rs:599` | `Uuid -> bool`; used to compute `affinity_hit` before claiming |
| `live_count` | `pool.rs:610` | `-> usize` = idle slots + `task_map.len()` |
| `task_map` / `task_map_mut` | `pool.rs:616`, `:620` | raw `&`/`&mut HashMap<task::Id, TaskMeta>` |
| `send_steer` | `pool.rs:646` | `(Uuid, SteerRequest) -> Result<(), SteerError>` |
| `result_tx` | `pool.rs:664` | clone of the unbounded sender for a new task |
| `rx_and_join_set` | `pool.rs:671` | split-borrow `(&mut Receiver, &mut JoinSet)` for one `select!` |
| `result_rx_try_recv` | `pool.rs:679` | non-blocking drain, shutdown-only |
| `slot_alive` | `pool.rs:686` | `usize -> bool` (idle **or** checked out) |
| `agents_mut` | `pool.rs:695` | `&mut Vec<Option<OwnedAgent>>` |
| `invalidate_channel_sessions` | `pool.rs:707` | `Uuid -> usize`; idle agents only |
| `switch_idle_agent_model` | `pool.rs:732` | `(Uuid, &str) -> IdleSwitchResult` |

`task_map_mut` and `agents_mut` hand the caller unrestricted mutable access to the pool's two invariant-bearing structures, so the slot/index invariant is enforced by convention in `lib.rs`, not by the type.

#### Free functions exported from `pool.rs`

| Item | Site | Consumer |
|---|---|---|
| `pub async fn run_prompt_task` | `pool.rs:1265` | spawned by `dispatch_pending` (`lib.rs:2947-2957`) and `dispatch_heartbeat` (`lib.rs:3537-3580`) |
| `pub(crate) fn prepend_base_for_legacy` | `pool.rs:1090` | `lib.rs` heartbeat path + own tests |
| `pub(crate) fn prepend_canvas_for_legacy` | `pool.rs:1110` | `run_prompt_task` initial-message path |
| `pub(crate) async fn fetch_channel_info` | `pool.rs:2237` | `ChannelInfoResolver::resolve` and `lib.rs` |
| `pub(crate) fn canvas_section_from_query_response` | `pool.rs:2366` | `fetch_canvas_section`, unit tests |
| `pub(crate) fn render_canvas_section` | `pool.rs:2479` | canvas rendering, unit tests |
| `pub(crate) async fn reaction_add` | `pool.rs:3462` | `react_working`, `lib.rs` 👀 path |
| `pub(crate) async fn reaction_remove` | `pool.rs:3540` | `clear_reactions` |
| `pub(crate) async fn post_failure_notice` | `pool.rs:3495` | `spawn_failure_notice` (`lib.rs:3014`) |
| `pub(crate) fn OwnedAgent::has_system_prompt_support` | `pool.rs:198` | prompt framing decisions |

`run_prompt_task` is the module's real entry point: 948 lines (`pool.rs:1265-2212`) with 20 distinct `send_prompt_result` exit sites (`pool.rs:1496`, `1509`, `1538`, `1549`, `1630`, `1656`, `1674`, `1692`, `1708`, `1789`, `1884`, `1920`, `1975`, `2038`, `2060`, `2094`, `2120`, `2145`, `2174`, `2201`).

Twelve private helpers are internal to the turn path and never named outside the file: `create_session_and_apply_model` (`pool.rs:804`), `apply_model_switch` (`:939`), `agent_supports_mode` (`:1015`), `apply_permission_mode` (`:1032`), `framed_system_prompt` (`:1137`), `workspace_section` (`:1165`), `with_team`/`with_core`/`with_canvas` (`:1180`, `:1199`, `:1213`), `send_prompt_result` (`:1235`), `requeue_batch_if_queue` (`:2973`), `requeue_cancelled_batch` (`:2986`), `classify_control_cancel_failure` (`:3029`), `publish_agent_turn_metric` (`:3322`).

#### `PoolLifecycle<P>` surface

| Method | Site | Contract |
|---|---|---|
| `listening()` | `pool_lifecycle.rs:28` | initial state |
| `start_wake_if_due(has_pending_work, now) -> Option<u32>` | `pool_lifecycle.rs:37` | returns the attempt token exactly once per entry into `Waking` |
| `take_ready() -> Option<P>` | `pool_lifecycle.rs:60` | moves the pool out, resets to `Listening` |
| `waking_attempt() -> Option<u32>` | `pool_lifecycle.rs:70` | read-only probe |
| `retry_at() -> Option<Instant>` | `pool_lifecycle.rs:77` | drives the retry sleep arm |
| `failed_error() -> Option<&str>` | `pool_lifecycle.rs:84` | only consumer is a `debug_assert_eq!` (`lib.rs:2565`) |
| `cancel_wake(attempt, error, now) -> bool` | `pool_lifecycle.rs:91` | thin wrapper over `complete_wake(.., Err(..))` |
| `complete_wake(attempt, Result<P,String>, now) -> Result<(), &'static str>` | `pool_lifecycle.rs:99` | rejects stale/duplicate results |

The two rejection strings are part of the contract and asserted verbatim in tests: `"wake result attempt did not match Waking attempt"` (`pool_lifecycle.rs:106`) and `"wake completed while lifecycle was not Waking"` (`pool_lifecycle.rs:107`).

#### How `lib.rs` drives the module

Construction forks on `config.lazy_pool` (`lib.rs:1317-1321`): the lazy path builds an all-`None` pool of `config.agents` slots, the eager path calls `initialize_agent_pool` (`lib.rs:3741-3841`) which spawns and initializes each agent before `from_slots`.

Main-loop wiring (`lib.rs:1698-1880`): a local `enum PoolEvent { Result, Panic, SteerAck, Wake }` (`lib.rs:1703-1708`) is produced by a single `biased` `select!` over `pool.rx_and_join_set()` (`lib.rs:1821`), the steer-ack channel, the wake channel, and the `retry_at` sleep. Dispatch of the resulting event goes to `handle_prompt_result` (`lib.rs:3034`), `recover_panicked_agent` (`lib.rs:3402`), or the `Wake` arm (`lib.rs:2535-2575`).

Turn dispatch (`lib.rs:2889-2981`): `has_session_for` → `try_claim(Some(channel_id))` → clone batch for `recoverable_batch` when `DedupMode::Queue` → install a capacity-1 steer channel on the client (`lib.rs:2937-2938`) → `join_set.spawn(run_prompt_task(..))` → insert `TaskMeta` keyed by the returned `AbortHandle::id()` (`lib.rs:2960-2970`). Heartbeats take the same shape with `try_claim(None)` and `control_rx = None` (`lib.rs:3538-3580`).

Shutdown ordering (`lib.rs:2590-2678`): fire the shutdown watch → drain `wake_tasks` under a 30 s timeout → drain `join_set` + `result_rx` under a 30 s grace, calling `acp.shutdown()` on each returned agent → `join_set.shutdown()` if the grace expires → `result_rx_try_recv` sweep → `acp.shutdown()` on remaining idle slots → `drop(pool)` → `respawn_tasks.shutdown()`.


## Module: buzz-acp — work queue, filtering & usage accounting (`crates/buzz-acp/src`)
### Aspect: API Surface

Nothing in these three modules is reachable from outside the crate except `TurnUsage`, which is re-exported at `lib.rs:15` (`pub use usage::TurnUsage;`) — the crate's only public re-export. `lib.rs` never names `TurnUsage` itself; the only consumers are inside the crate (`acp.rs:783`, `pool.rs:3324`). The sole downstream crate, `sprig`, calls `buzz_acp::run()` only (`sprig/src/main.rs:17`), so the re-export currently has no external reader.

#### `EventQueue` — constructors and lifecycle

| Item | Line | Callers |
|---|---|---|
| `EventQueue::new(DedupMode)` | `queue.rs:179-195` | `lib.rs:1505-1506` (only production site) |
| `.with_in_flight_deadline(max_turn_duration_secs)` | `queue.rs:197-201` | `lib.rs:1506` |
| `impl Default` → `new(DedupMode::Drop)` | `queue.rs:822-824` | tests only |
| `extend_in_flight_deadline(channel_id, max_turn_secs)` | `queue.rs:210-228` | `lib.rs:2509` (`SteerAck::Success` arm) |

#### `EventQueue` — hot path

| Item | Line | Callers |
|---|---|---|
| `push(QueuedEvent) -> bool` | `queue.rs:230-252` | `lib.rs:2198` |
| `flush_next() -> Option<FlushBatch>` | `queue.rs:260-380` | `lib.rs:2896` (`dispatch_pending` loop) |
| `mark_complete(channel_id)` | `queue.rs:392-410` | `lib.rs:2913` (pool exhausted), `lib.rs:3171` (`handle_prompt_result`), `lib.rs:3440` |
| `requeue(FlushBatch) -> Option<FlushBatch>` | `queue.rs:429-498` | `lib.rs:3120` (hard-timeout recently-active), `lib.rs:3145` (generic failure), `lib.rs:3428` (panic recovery) |
| `requeue_preserve_timestamps(FlushBatch)` | `queue.rs:508-529` | `lib.rs:2912` (no agent available) |
| `requeue_as_cancelled(FlushBatch, CancelReason)` | `queue.rs:542-548` | `lib.rs:3090` |
| `has_flushable_work() -> bool` (`&mut self`) | `queue.rs:556-591` | `lib.rs:1715`, `:1777`, `:2288` |
| `is_channel_in_flight(channel_id) -> bool` | `queue.rs:645-647` | `lib.rs:2219` |
| `drain_channel(channel_id) -> Vec<String>` | `queue.rs:625-642` | `lib.rs:1990` (agent removed from channel); returns hex event ids for 👀 reaction cleanup |
| `pending_channels() -> usize` | `queue.rs:594-596` | `lib.rs:2910`, `:2975` (log fields) |
| `compact_expired_state()` | `queue.rs:807-818` | `lib.rs:1746` (30 s maintenance tick, interval at `lib.rs:1608`) |

#### `EventQueue` — goose-native steer side table

| Item | Line | Callers |
|---|---|---|
| `mark_native_steer_pending(channel_id, event_id) -> bool` | `queue.rs:673-691` | `lib.rs:2847` |
| `release_native_steer(channel_id, event_id)` | `queue.rs:703-730` | `lib.rs:2515` (`SteerAck::Err` / `PromptCompletedNeutral`) |
| `remove_event(channel_id, event_id)` | `queue.rs:738-751` | `lib.rs:2512` (`SteerAck::Success`) |
| `recover_withheld_for_expired_channel` (private) | `queue.rs:766-789` | `flush_next` (`queue.rs:286`) and `has_flushable_work` (`queue.rs:580`) expiry blocks |

#### `EventQueue` — test-only accessors

| Item | Line | Callers |
|---|---|---|
| `queued_event_count(&Uuid) -> usize` (`#[cfg(test)]`) | `queue.rs:600-602` | `lib.rs:5507`, `:5612`, `:5808`, `:6231`, `:6316` |
| `set_retry_count_for_test(Uuid, u32)` (`#[cfg(test)]`) | `queue.rs:610-612` | `lib.rs:5733` |
| `MAX_RETRIES` (`pub(crate)`) | `queue.rs:30` | `lib.rs:5733` and doc reference `lib.rs:5722` |

All other queue consts (`MAX_PENDING_PER_CHANNEL`, `MAX_BATCH_EVENTS`, `BASE_RETRY_DELAY_SECS`, `MAX_RETRY_DELAY_SECS`, `IN_FLIGHT_DEADLINE_BUFFER_SECS`, `DEFAULT_IN_FLIGHT_DEADLINE_SECS`) are private to `queue.rs` (`queue.rs:24-42`) and cannot be observed or overridden by callers.

#### Prompt-formatting surface (all in `queue.rs`, all consumed by `pool.rs` / `lib.rs`)

| Item | Visibility | Line | Callers |
|---|---|---|---|
| `format_prompt(&FlushBatch, &FormatPromptArgs) -> Vec<String>` | `pub` | `queue.rs:1406-1564` | `pool.rs:1771` |
| `FormatPromptArgs<'a>` | `pub` | `queue.rs:1353-1375` | `pool.rs:1773` |
| `format_event_block(channel_id, channel_info, &BatchEvent, profile_lookup) -> String` | `pub(crate)` | `queue.rs:1076-1147` | `lib.rs:2831` (native steer); internally by `format_prompt` |
| `base_section(&str) -> String` | `pub(crate)` | `queue.rs:1382-1384` | `pool.rs:1097`, `:1145`, `:1147`; internally by `format_prompt` (`queue.rs:1436`) |
| `native_steer_framing() -> (&'static str, &'static str)` | `pub(crate)` | `queue.rs:1623-1626` | `lib.rs:2824` |
| `parse_thread_tags(&Event) -> ThreadTags` | `pub` | `queue.rs:849-900` | `lib.rs:2904`, `:3023`; `pool.rs:2512`, `:2546`; `setup_mode.rs:605` |
| `extract_slash_command(&str, &[&str]) -> Option<String>` | `pub` | `queue.rs:902-959` | only `slash_command_for_batch` + tests |
| `slash_command_for_batch(&FlushBatch, &[&str]) -> Option<String>` | `pub` | `queue.rs:961-967` | `pool.rs:1761` |
| `ThreadTags`, `ConversationContext`, `ContextMessage`, `PromptChannelInfo`, `PromptProfile`, `PromptProfileLookup`, `CancelReason`, `FlushBatch`, `BatchEvent`, `QueuedEvent` | `pub` | `queue.rs:45-1012` | imported by `pool.rs:37-41`, `lib.rs:44` |

Private helpers with no external surface: `normalize_lookup_key` (`:1017`), `sanitize_prompt_label` (`:1028`), `resolve_prompt_label` (`:1042`), `format_prompt_actor` (`:1060`), `append_reply_instruction` (`:1149`), `append_new_thread_reply_instruction` (`:1164`), `turn_is_human_facing` (`:1182`), `resolve_reply_anchor` (`:1209`), `format_context_hints` (`:1233`), `format_conversation_context` (`:1317`), `MergeFraming::for_reason` (`:1584`).

#### `filter.rs` surface

| Item | Visibility | Line | Callers |
|---|---|---|---|
| `match_event(&Event, Uuid, &[SubscriptionRule], &str) -> Option<MatchedRule>` (async) | `pub` | `filter.rs:368-459` | `lib.rs:2174`, `setup_mode.rs:444` |
| `evaluate_filter(&str, &FilterContext, Option<Arc<Node>>) -> Result<bool, FilterError>` (async) | `pub` | `filter.rs:197-262` | `match_event` (`filter.rs:428`) + tests only |
| `FilterContext::from_event(&Event, Uuid)` | `pub` | `filter.rs:42-51` | `match_event` (`filter.rs:374`) + tests |
| `SubscriptionRule` (+ `Default`, `Clone`, `Deserialize`) | `pub` | `filter.rs:83-148` | `config.rs:16`, `config.rs:1060` (`load_rules`), `lib.rs:1439-1474`, `setup_mode.rs:545-565` |
| `ChannelScope::matches(&Uuid) -> bool` | `pub` | `filter.rs:67-72` | `match_event` (`filter.rs:378`), `config.rs:1118` |
| `MatchedRule`, `FilterError` | `pub` | `filter.rs:150-156`, `:16-24` | `lib.rs:2175`, internal |
| `build_eval_context` | private | `filter.rs:264-338` | `evaluate_filter` |
| `MAX_EXPR_LEN`, `EVAL_TIMEOUT`, `MAX_CONCURRENT_FILTER_EVALS`, `MAX_CONSECUTIVE_TIMEOUTS`, `FILTER_EVAL_SEMAPHORE` | private | `filter.rs:162`, `:165`, `:173`, `:341`, `:182-183` | internal; `MAX_EXPR_LEN`'s 4096 value is duplicated as a literal at `config.rs:1098` |

#### `usage.rs` surface

| Item | Visibility | Line | Callers |
|---|---|---|---|
| `TurnUsage` | `pub` (re-exported `lib.rs:15`) | `usage.rs:117-140` | `acp.rs:783` return type, `pool.rs:3324` parameter, constructed in `pool.rs` tests `:5136`, `:5168`, `:5201`, `:5234` |
| `UsageTracker` | `pub(crate)` | `usage.rs:163-173` | field `AcpClient.goose_usage` (`acp.rs:202`), initialized `acp.rs:498` |
| `UsageTracker::begin_turn(&str)` | `pub(crate)` | `usage.rs:182-185` | `acp.rs:690` (immediately before `session/prompt`) |
| `UsageTracker::record(&str, &UsageUpdatePayload)` | `pub(crate)` | `usage.rs:211-302` | `acp.rs:1659` (via `handle_goose_usage_update`, called at `acp.rs:1141` and `:1488`) |
| `UsageTracker::take() -> Option<TurnUsage>` | `pub(crate)`, `#[cfg_attr(not(test), allow(dead_code))]` | `usage.rs:304-320` | `acp.rs:784` (`AcpClient::take_turn_usage`) |
| `GooseSessionUpdateNotification`, `GooseSessionUpdateVariant`, `UsageUpdatePayload` | `pub(crate)` | `usage.rs:58-93` | `acp.rs:1638` (import inside `handle_goose_usage_update`) |

`UsageTracker::take` carries `#[cfg_attr(not(test), allow(dead_code))]` (`usage.rs:303`) even though `acp.rs:784` calls it in production — the attribute is unnecessary and masks real dead-code signal. `MatchedRule::rule_index` (`filter.rs:152`) and `UsageUpdatePayload::{used, context_limit}` (`usage.rs:80`, `:84`) carry the same allow and genuinely are unread outside tests.


## Module: buzz-acp — ACP protocol, config & setup mode (`crates/buzz-acp/src`)
### Aspect: API Surface

#### `AcpClient` — lifecycle

| Signature | Line |
|---|---|
| `pub async fn spawn(command: &str, args: &[String], extra_env: &[(String,String)], has_generated_codex_config: bool) -> Result<Self, AcpError>` | `acp.rs:408-413` |
| `pub async fn shutdown(&mut self)` | `acp.rs:376` |
| `impl Drop for AcpClient` | `acp.rs:1953-1967` |

#### `AcpClient` — protocol methods

| Signature | JSON-RPC method sent | Line |
|---|---|---|
| `pub async fn initialize(&mut self) -> Result<serde_json::Value, AcpError>` | `initialize` | `acp.rs:539-544` |
| `pub async fn authenticate(&mut self, method_id: &str) -> Result<Value, AcpError>` | `authenticate` (`params.methodId`) | `acp.rs:549-554` |
| `pub async fn session_new_full(&mut self, cwd: &str, mcp_servers: Vec<McpServer>, system_prompt: Option<&str>) -> Result<SessionNewResponse, AcpError>` | `session/new` | `acp.rs:563-588` |
| `pub async fn session_new(...) -> Result<String, AcpError>` | `session/new` (wrapper) | `acp.rs:592-599`, `#[allow(dead_code)]` at `acp.rs:591` |
| `pub async fn session_set_goose_system_prompt(&mut self, session_id: &str, text: &str) -> Result<Value, AcpError>` | `_goose/unstable/session/system-prompt/set` with `mode:"append"`, `key:"buzz"` | `acp.rs:605-619` |
| `pub async fn session_set_config_option(&mut self, session_id, config_id, value: &str) -> Result<Value, AcpError>` | `session/set_config_option` | `acp.rs:623-634` |
| `pub async fn session_set_model(&mut self, session_id, model_id: &str) -> Result<Value, AcpError>` | `session/set_model` (unstable) | `acp.rs:638-650` |
| `pub async fn session_prompt_with_idle_timeout(&mut self, session_id, prompt_text, idle_timeout, max_duration) -> Result<StopReason, AcpError>` | `session/prompt` (single block) | `acp.rs:654-670` |
| `pub async fn session_prompt_blocks_with_idle_timeout(&mut self, session_id, prompt_blocks: &[&str], idle_timeout, max_duration) -> Result<StopReason, AcpError>` | `session/prompt` (N blocks) | `acp.rs:676-745` |
| `pub async fn session_cancel(&mut self, session_id: &str) -> Result<(), AcpError>` | `session/cancel` **notification** | `acp.rs:747-752` |
| `pub async fn cancel_with_cleanup(&mut self, session_id, _idle_timeout) -> Result<StopReason, AcpError>` | permission-cancel + `session/cancel` + drain | `acp.rs:837-875` |
| `pub async fn cancel_with_cleanup_grace(&mut self, session_id, grace: Duration) -> Result<StopReason, AcpError>` | same, bounded window | `acp.rs:881-895` |

`_goose/unstable/session/steer` has no public wrapper — it is written only from inside the read loop's steer arm (`acp.rs:1338-1355`), because `expectedRunId` must be sampled at write time.

#### `AcpClient` — accessors and per-turn plumbing

| Signature | Line | Notes |
|---|---|---|
| `pub fn set_observer(&mut self, observer: Option<ObserverHandle>, agent_index: usize)` | `acp.rs:503` | |
| `pub fn set_observer_context(&mut self, context: ObserverContext)` | `acp.rs:509` | |
| `pub(crate) fn observer_handle(&self) -> Option<ObserverHandle>` | `acp.rs:514` | |
| `pub(crate) fn observer_agent_index(&self) -> Option<usize>` | `acp.rs:519` | |
| `pub fn observe(&self, kind: impl Into<String>, payload: serde_json::Value)` | `acp.rs:524` | |
| `pub fn has_in_flight_prompt(&self) -> bool` | `acp.rs:755` | `last_prompt_id.is_some()` |
| `pub fn active_run_id(&self) -> Option<&str>` | `acp.rs:769` | `#[cfg_attr(not(test), allow(dead_code))]` (`acp.rs:768`) — kept public only for tests |
| `pub fn take_turn_usage(&mut self) -> Option<TurnUsage>` | `acp.rs:783` | at most once per turn |
| `pub fn install_steer_rx(&mut self, rx: mpsc::Receiver<pool::SteerRequest>)` | `acp.rs:800` | **panics** via `assert!` if one is already installed (`acp.rs:801-805`) |
| `pub fn clear_steer_rx(&mut self)` | `acp.rs:815` | idempotent |
| `pub fn steer_rx_is_none(&self) -> bool` | `acp.rs:824` | `#[cfg(test)]` (`acp.rs:823`) |
| `pub async fn drain_stale_responses(&mut self, drain_timeout: Duration)` | `acp.rs:1023` | `#[allow(dead_code)]` — "not yet wired" (`acp.rs:1022`) |

Private internals: `write_ndjson` (`acp.rs:951`), `send_request` (`acp.rs:979`), `send_notification` (`acp.rs:1047`), `read_until_response` (`acp.rs:1074`), `read_until_response_with_idle_timeout` (`acp.rs:1198-1204`), `cancel_with_cleanup_until` (`acp.rs:897`), `handle_session_update` (`acp.rs:1527`), `handle_goose_usage_update` (`acp.rs:1637`), `handle_permission_request` (`acp.rs:1680`), `parse_stop_reason` (`acp.rs:1758`).

#### Module-level free functions in `acp.rs`

| Signature | Visibility | Line |
|---|---|---|
| `pub fn extract_model_config_options(result: &Value) -> Vec<Value>` | pub | `acp.rs:1851` |
| `pub fn extract_model_state(result: &Value) -> Option<Value>` | pub | `acp.rs:1866` |
| `pub fn resolve_model_switch_method(session_new_result: &Value, desired_model: &str) -> Option<ModelSwitchMethod>` | pub | `acp.rs:1876-1879` |
| `pub fn model_in_catalog(config_options: &[Value], available_models: Option<&Value>, desired_model: &str) -> bool` | pub | `acp.rs:1922-1926` |
| `pub(crate) fn build_codex_config_env(extra_env, parent_codex_config, has_generated_codex_config) -> Result<Option<String>, AcpError>` | crate | `acp.rs:257-261` |
| `fn deep_merge(base: &mut Map, overlay: Map)` | private | `acp.rs:208-211` |
| `fn agent_error_from_json(error: &Value) -> AcpError` | private | `acp.rs:115` |
| `fn kill_process_group(pid: u32) -> bool` | private, `#[cfg(unix)]` / `#[cfg(not(unix))]` | `acp.rs:1979`, `acp.rs:1990` |
| `fn configure_no_window(cmd: &mut Command)` | private | `acp.rs:1997` |

#### Notifications and agent-initiated requests handled inbound

Dispatched identically in both read loops (`acp.rs:1136-1163` and `acp.rs:1481-1510`):

| Inbound method | Handling | Line |
|---|---|---|
| `session/update` | `handle_session_update`, returns "tool call started" flag | `acp.rs:1138-1140` / `acp.rs:1483-1490` |
| `_goose/unstable/session/update` | `handle_goose_usage_update` (usage tracking) | `acp.rs:1141-1143` / `acp.rs:1491-1493` |
| `session/request_permission` | auto-answered by `handle_permission_request` | `acp.rs:1144-1146` / `acp.rs:1494-1496` |
| anything else **with** an `id` | replied `-32601 Method not found` | `acp.rs:1147-1156` / `acp.rs:1497-1506` |
| anything else **without** an `id` | debug-logged, dropped | `acp.rs:1160` / `acp.rs:1507` |

`session/update` sub-kinds recognised by `handle_session_update` (discriminator is `sessionUpdate`, `acp.rs:1529-1532`): `agent_message_chunk` (`:1535`), `tool_call` (`:1541`, returns `true`), `tool_call_update` (`:1553`), `plan` (`:1563`), `agent_thought_chunk` (`:1567`), `available_commands_update` (`:1573`), `session_info_update` (`:1590`), `keepalive` (`:1621`), anything else (`:1622-1625`).

#### Clap CLI surface — `CliArgs` (`config.rs:234-474`)

| Flag | Env | Default | Line |
|---|---|---|---|
| `--relay-url` | `BUZZ_RELAY_URL` | `ws://localhost:3000` | `config.rs:240` |
| `--private-key` | `BUZZ_PRIVATE_KEY` | *(required)* | `config.rs:243` |
| `--agent-owner` | `BUZZ_ACP_AGENT_OWNER` | — | `config.rs:247` |
| `--agent-command` | `BUZZ_ACP_AGENT_COMMAND` | `goose` | `config.rs:250` |
| `--agent-args` | `BUZZ_ACP_AGENT_ARGS` | `acp` (comma-delimited) | `config.rs:253-258` |
| `--mcp-command` | `BUZZ_ACP_MCP_COMMAND` | `""` | `config.rs:261` |
| `--idle-timeout` | `BUZZ_ACP_IDLE_TIMEOUT` | none → `900` | `config.rs:266` |
| `--max-turn-duration` | `BUZZ_ACP_MAX_TURN_DURATION` | `7200` | `config.rs:270` |
| `--turn-timeout` | `BUZZ_ACP_TURN_TIMEOUT` | — | `config.rs:274`, **`hide = true`**, deprecated |
| `--system-prompt` | `BUZZ_ACP_SYSTEM_PROMPT` | — | `config.rs:277-281`, conflicts with `--system-prompt-file` |
| `--system-prompt-file` | `BUZZ_ACP_SYSTEM_PROMPT_FILE` | — | `config.rs:284-288` |
| `--agents` | `BUZZ_ACP_AGENTS` | `1`, range `1..=32` | `config.rs:292-293` |
| `--heartbeat-interval` | `BUZZ_ACP_HEARTBEAT_INTERVAL` | `0` | `config.rs:297` |
| `--turn-liveness-secs` | `BUZZ_ACP_TURN_LIVENESS_SECS` | `10` | `config.rs:302` |
| `--heartbeat-prompt` | `BUZZ_ACP_HEARTBEAT_PROMPT` | — | `config.rs:306-310` |
| `--heartbeat-prompt-file` | `BUZZ_ACP_HEARTBEAT_PROMPT_FILE` | — | `config.rs:314-318` |
| `--initial-message` | `BUZZ_ACP_INITIAL_MESSAGE` | — | `config.rs:321` |
| `--subscribe` | `BUZZ_ACP_SUBSCRIBE` | `mentions` | `config.rs:324-329` |
| `--kinds` | `BUZZ_ACP_KINDS` | — (comma) | `config.rs:332` |
| `--channels` | `BUZZ_ACP_CHANNELS` | — (comma) | `config.rs:335` |
| `--no-mention-filter` | `BUZZ_ACP_NO_MENTION_FILTER` | `false` | `config.rs:338` |
| `--config` | `BUZZ_ACP_CONFIG` | `./buzz-acp.toml` | `config.rs:341` |
| `--dedup` | `BUZZ_ACP_DEDUP` | `queue` | `config.rs:344` |
| `--multiple-event-handling` | `BUZZ_ACP_MULTIPLE_EVENT_HANDLING` | `steer` | `config.rs:353-358` |
| `--no-ignore-self` | `BUZZ_ACP_NO_IGNORE_SELF` | `false` | `config.rs:361` |
| `--context-message-limit` | `BUZZ_ACP_CONTEXT_MESSAGE_LIMIT` | `12`, range `0..=100` | `config.rs:366-367` |
| `--max-turns-per-session` | `BUZZ_ACP_MAX_TURNS_PER_SESSION` | `0` | `config.rs:372-373` |
| `--no-presence` | `BUZZ_ACP_NO_PRESENCE` | `false` | `config.rs:377` |
| `--no-typing` | `BUZZ_ACP_NO_TYPING` | `false` | `config.rs:381` |
| `--memory` | `BUZZ_ACP_MEMORY` | `true` | `config.rs:393-398`, conflicts with `no_memory` |
| `--no-memory` | `BUZZ_ACP_NO_MEMORY` | `false` | `config.rs:404` |
| `--no-base-prompt` | `BUZZ_ACP_NO_BASE_PROMPT` | `false` | `config.rs:409` |
| `--base-prompt-file` | `BUZZ_ACP_BASE_PROMPT_FILE` | — | `config.rs:414-418`, conflicts with `no_base_prompt` |
| `--model` | `BUZZ_ACP_MODEL` | — | `config.rs:423` |
| `--permission-mode` | `BUZZ_ACP_PERMISSION_MODE` | `bypass-permissions` | `config.rs:432-437` |
| `--respond-to` | `BUZZ_ACP_RESPOND_TO` | `owner-only` | `config.rs:442-447` |
| `--respond-to-allowlist` | `BUZZ_ACP_RESPOND_TO_ALLOWLIST` | — (comma) | `config.rs:452` |
| `--allowed-respond-to` | `BUZZ_ACP_ALLOWED_RESPOND_TO` | — (comma) | `config.rs:460` |
| `--team-instructions` | `BUZZ_ACP_TEAM_INSTRUCTIONS` | — | `config.rs:464` |
| `--relay-observer` | `BUZZ_ACP_RELAY_OBSERVER` | `false` | `config.rs:468` |
| `--lazy-pool` | `BUZZ_ACP_LAZY_POOL` | `false` | `config.rs:472` |

43 `env = "…"` attributes across the file. `--turn-timeout` is the only hidden flag.

#### Auxiliary clap parsers

Three standalone `Parser` structs, each dispatched by manual argv sniffing in `lib.rs` rather than a clap subcommand:

| Struct | Fields | Line | Dispatched at |
|---|---|---|---|
| `AuthAgentArgs` | `--agent-command` (`BUZZ_ACP_AGENT_COMMAND`, default `goose`), `--agent-args` (`BUZZ_ACP_AGENT_ARGS`, default `acp`) | `config.rs:188-203` | flattened into all three below |
| `ModelsArgs` | `#[command(flatten)] agent`, `--json` | `config.rs:171-186` | `lib.rs:1252-1253` |
| `AuthMethodsArgs` | `agent`, `--json` | `config.rs:205-216` | `lib.rs:1262-1263` |
| `AuthenticateArgs` | `agent`, `--method-id` (required, no env) | `config.rs:218-232` | `lib.rs:1272-1273` |

`ModelsArgs`' doc comment (`config.rs:165-169`) states the `models` path deliberately bypasses `Config::from_cli()` — no relay, no private key.

#### `Config` and free-function surface in `config.rs`

| Signature | Line |
|---|---|
| `pub fn Config::from_cli() -> Result<Self, ConfigError>` | `config.rs:729` |
| `pub fn Config::from_args(mut args: CliArgs) -> Result<Self, ConfigError>` | `config.rs:740` |
| `pub fn Config::summary(&self) -> String` | `config.rs:1012` |
| `pub fn propagate_legacy_env_vars()` | `config.rs:715` |
| `pub fn load_rules(path: &Path) -> Result<Vec<SubscriptionRule>, ConfigError>` | `config.rs:1060` |
| `pub fn resolve_channel_filters(config, discovered_channels: &[Uuid], rules) -> HashMap<Uuid, ChannelFilter>` | `config.rs:1134-1138` |
| `pub fn resolve_dynamic_channel_filter(config, channel_id: Uuid, rules) -> Option<ChannelFilter>` | `config.rs:1236-1240` |
| `pub fn codex_network_env(agent_command: &str, relay_url: &str) -> Option<(String, String)>` | `config.rs:646` |
| `pub fn normalize_agent_args(command: &str, agent_args: Vec<String>) -> Vec<String>` | `config.rs:679` |
| `pub(crate) fn normalize_agent_command_identity(command: &str) -> String` | `config.rs:600` |
| `pub fn PermissionMode::as_wire_str(&self) -> &'static str` / `is_default(&self) -> bool` | `config.rs:144`, `config.rs:156` |
| private: `validate_allowlist` (`:558`), `validate_multiple_event_handling` (`:579`), `default_agent_args` (`:617`), `rule_applies_to_channel` (`:1316`) | |

#### `setup_mode` surface (all `pub(crate)`)

| Item | Line |
|---|---|
| `const SETUP_PAYLOAD_ENV_VAR: &str = "BUZZ_ACP_SETUP_PAYLOAD"` | `setup_mode.rs:83` |
| `SetupPayload::from_env() -> Result<Option<Self>>` | `setup_mode.rs:213` |
| `SetupPayload::from_raw_env_value(raw: Option<String>) -> Result<Option<Self>>` | `setup_mode.rs:226` |
| `async fn run_setup_listener(config: Config, payload: SetupPayload) -> Result<()>` | `setup_mode.rs:309` |
| `#[must_use] fn should_nudge_for_event(event_id, author_allowed, filter_matched, nudged_event_ids) -> bool` | `setup_mode.rs:494-500` |
| private `SetupPayload::nudge_body(&self) -> String` | `setup_mode.rs:243` |
| private `RequirementPayload::instruction(&self) -> String` | `setup_mode.rs:122` |
| private `build_setup_subscription_rules(config) -> Vec<SubscriptionRule>` | `setup_mode.rs:521` |
| private `mentions_rule(kinds: Vec<u32>) -> SubscriptionRule` | `setup_mode.rs:545` |
| private `async fn handle_setup_membership(relay, buzz_event, config, rules, _initial_channel_ids)` | `setup_mode.rs:563-569` |
| private `async fn publish_setup_nudge(publisher, keys, channel_id, triggering_event, payload) -> Result<()>` | `setup_mode.rs:595-601` |

`handle_setup_membership`'s fifth parameter `_initial_channel_ids` is accepted and never used (`setup_mode.rs:568`).


## Module: buzz-agent — core loop, ACP wire types & handoff (`crates/buzz-agent/src`)
### Aspect: API Surface
The real API of this group is a JSON-RPC 2.0 line protocol over stdin/stdout. The Rust surface is deliberately thin: one `pub fn run()` plus re-exports.

#### Rust public items originating in this group
| Item | Kind | Line | Doc comment? |
|---|---|---|---|
| `run()` | `pub fn` | `lib.rs:110` | No |
| `WINDOWS_SHELL_RESOLUTION_ENV` | `pub const &[&str]`, `#[cfg(windows)]` | `lib.rs:22-30` | Yes (`lib.rs:18-20`) |
| `pub mod auth`, `pub mod catalog`, `pub mod config`, `pub mod types` | module re-export | `lib.rs:4-12` | No |
| `pub use catalog::{discover_databricks_models, ModelEntry, DATABRICKS_V2_KNOWN_MODELS}` | re-export | `lib.rs:14` | n/a |
| `pub use config::Provider` | re-export | `lib.rs:15` | n/a |
| `pub use types::AgentError` | re-export | `lib.rs:16` | n/a |

`mod agent`, `mod handoff`, `mod wire`, `mod builtin`, `mod hints`, `mod llm`, `mod mcp` are private (`lib.rs:2-13`). Everything marked `pub` inside `agent.rs` and `wire.rs` is therefore **pub-in-private** — visible crate-wide but not part of the crate's external API: `RunCtx` and its 17 `pub` fields (`agent.rs:24-64`), `RunCtx::run` (`agent.rs:66`), the four JSON-RPC error constants (`wire.rs:8-11`), `WireMsg`/`WireSender`/`Inbound` (`wire.rs:13-37`), all six params structs, and `classify`/`ok`/`err`/`session_update`/`goose_session_update`/`session_update_with_goose_meta`/`send`/`read_bounded_line`/`writer_task` (`wire.rs:93-237`).

Because `pub mod types` is public (`lib.rs:11`), the entire contents of `types.rs` **are** external API: `ToolResultContent`, `HistoryItem`, `ToolCall`, `ToolResult`, `LlmResponse`, `ProviderStop`, `ToolDef`, `StopReason`, `McpServerStdio`, `EnvVar`, `ContentBlock`, `AgentError`, and the free function `clamp` (`types.rs:259`). No external consumer uses them: the only cross-crate consumers of `buzz-agent` are `sprig` (calls `buzz_agent::run()`, `crates/sprig/src/main.rs:18`) and the desktop Tauri backend (uses `WINDOWS_SHELL_RESOLUTION_ENV`, `desktop/src-tauri/src/managed_agents/git_bash.rs:136`).

#### Undocumented public items (AGENTS.md violation)
AGENTS.md:114 states "New public API must have doc comments". Undocumented public items in this group, by file:
- `types.rs`: `ToolResultContent` (`:17`), `as_text_lossy` (`:49`), `HistoryItem` (`:60`), `estimated_bytes` (`:70`), `ToolCall` (`:107`), `ToolResult` (`:114`), `text` (`:121`), `LlmResponse` (struct itself, `:131` — its fields are documented), `ProviderStop` (`:158`), `ToolDef` (`:167`), `StopReason` (`:174`), `as_wire` (`:183`), `McpServerStdio` (`:195`), `EnvVar` (`:205`), `ContentBlock` (`:212`), `AgentError` (`:224`), `json_rpc_code` (`:249`), `clamp` (`:259`). That is 18 of 20 public items in the crate's one public data module.
- `lib.rs`: `run()` (`:110`).
- `wire.rs` (crate-internal but `pub`): `classify` (`:93`), `ok` (`:123`), `err` (`:127`), `session_update` (`:131`), `send` (`:170`), `read_bounded_line` (`:174`), `writer_task` (`:220`), plus the four error constants.

#### Inbound JSON-RPC requests handled
Dispatch is a flat `match` on the method string (`lib.rs:224-265`). There is no state machine: `initialize` is **not** required before `session/new` or `session/prompt` — nothing records that it happened (`negotiated_version` is computed and dropped, `lib.rs:284`).

| Method | Handler | Concurrency | Success result | Error codes |
|---|---|---|---|---|
| `initialize` | `initialize` (`lib.rs:273`) | inline | `{protocolVersion, agentCapabilities{loadSession:false, promptCapabilities{image:false,audio:false,embeddedContext:false}, mcpCapabilities{http:false,sse:false}}, agentInfo{name:"buzz-agent",version:CARGO_PKG_VERSION}}` (`lib.rs:288-297`) | `-32602` on bad params |
| `session/new` | `session_new` (`lib.rs:329`), `tokio::spawn`ed (`lib.rs:227-231`) | concurrent | `{sessionId, models:{currentModelId, availableModels:[{modelId,name}]}}` (`lib.rs:464-474`) | `-32602` (bad params / relative `cwd` / max sessions / oversize system prompt), MCP spawn error code via `e.json_rpc_code()` (`lib.rs:392`), `-32000` on RNG failure (`lib.rs:396`) |
| `session/prompt` | `run_prompt` (`lib.rs:627`), spawned (`lib.rs:623-625`) | one per session | `{stopReason}` (`lib.rs:753-758`) | `-32602` (bad params, unknown session, "prompt already in flight"), else `AgentError::json_rpc_code()` |
| `session/set_model` | `set_model_session` (`lib.rs:503`) | inline | `{sessionId, modelId}` (`lib.rs:535-540`) | `-32602` (bad params, empty `modelId`, unknown session) |
| `session/cancel` | `cancel_session` (`lib.rs:487`) | inline | `null` (`lib.rs:239`) | none — unknown session is silently accepted (`lib.rs:487-494`) |
| `_goose/unstable/session/steer` | `steer_session` (`lib.rs:554`) | inline | `{runId, messageId}` (`lib.rs:607-610`) | `-32602` (empty prompt, empty `expectedRunId`, unknown session, no active run, run-id mismatch) |
| anything else | — | — | — | `-32601` `jsonrpc: method not found: {method}` (`lib.rs:248-258`) |

#### Inbound notifications
Only `session/cancel` is acted on (`lib.rs:267-271`); every other notification is silently discarded. Bare responses (`id` without `method`) map to `Inbound::Ignored` and are dropped without a log line (`wire.rs:110-111`, `lib.rs:215`). Malformed frames: non-object or wrong `jsonrpc` version → `-32600` (`wire.rs:94-101`); missing both method and id → `-32600` (`wire.rs:113-117`); unparseable JSON → `-32700` with id `null` (`lib.rs:196-203`).

#### Outbound notifications emitted
| Method | `update.sessionUpdate` | Emitted at | Payload |
|---|---|---|---|
| `session/update` | `session_info_update` + `_meta.goose.activeRunId` | `lib.rs:661-670` | run id, at prompt start only |
| `session/update` | `session_info_update` + `_meta.goose.queuedSteer` | `lib.rs:612-620` | `{messageId, runId}` after an accepted steer |
| `session/update` | `keepalive` | `agent.rs:134-144` | empty; every 30 s while awaiting the provider |
| `session/update` | `agent_thought_chunk` | `agent.rs:179-192` | `content.text` = provider reasoning, only when non-empty |
| `session/update` | `agent_message_chunk` | `agent.rs:194-206` | `content.text` = assistant text, only when non-empty |
| `session/update` | `tool_call` (`status:"pending"`) | `agent.rs:552-568` | `toolCallId`, `title`=tool name, `kind:"other"`, `rawInput`=arguments |
| `session/update` | `tool_call_update` (`in_progress`) | `agent.rs:570-583` | `toolCallId`, status |
| `session/update` | `tool_call_update` (`completed`) | `agent.rs:585-600` | `content[]` text + `rawOutput.isError` |
| `session/update` | `tool_call_update` (`failed`) | `agent.rs:602-616` | `rawOutput.error` |
| `_goose/unstable/session/update` | `usage_update` | `lib.rs:730-750` | `used`, `contextLimit:0`, `accumulatedInputTokens`, `accumulatedOutputTokens`, `model` |

The `_meta` envelope is nested **inside** `update`, not beside it (`wire.rs:157-169`) — deliberately matching goose's `SessionInfoUpdate` layout. `usage_update` uses a separate top-level method (`wire.rs:143-149`), not `session/update`.

Ordering guarantee: the `usage_update` notification is always sent before the `session/prompt` response (`lib.rs:714-752` precedes `lib.rs:753-759`), because buzz-acp's `UsageTracker` must see it while the turn is still open. Locked down by `usage_notification_emitted_before_prompt_response` (`tests/fake_llm.rs:801`), `no_usage_turn_emits_no_usage_notification` (`tests/fake_llm.rs:888`), and `cancelled_turn_with_usage_emits_notification_before_response` (`tests/fake_llm.rs:926`).

#### Stop reasons returned
`StopReason::as_wire` (`types.rs:183-191`) emits `end_turn`, `cancelled`, `max_tokens`, `max_turn_requests`, `refusal`. `map_stop` (`agent.rs:740-746`) collapses `ProviderStop::{EndTurn,ToolUse,Other}` → `end_turn`. All five strings are accepted by the client side (`crates/buzz-acp/src/acp.rs:66-71`, test `stop_reason_parses_all_known_values` at `acp.rs:2012`).

#### Error-code mapping
| Code | Meaning | Source |
|---|---|---|
| `-32700` | parse error | `wire.rs:8`, used `lib.rs:200` |
| `-32600` | invalid request | `wire.rs:9`, used `wire.rs:98`, `wire.rs:115` |
| `-32601` | method not found | `wire.rs:10`, used `lib.rs:252` |
| `-32602` | invalid params | `wire.rs:11`; `AgentError::InvalidParams` (`types.rs:251`) |
| `-32001` | LLM auth failure | `AgentError::LlmAuth` (`types.rs:252`) |
| `-32002` | model not found | `AgentError::LlmModelNotFound` (`types.rs:253`) |
| `-32000` | everything else (`Llm`, `Mcp`, `Cancelled`) | `types.rs:254`; also literal at `lib.rs:396` |

#### CLI entry points
`main.rs` is 6 lines: call `buzz_agent::run()`, print `Error: {e}` to stderr and `exit(1)` on failure (`main.rs:1-6`). `run()` (`lib.rs:110-121`) inspects `argv[1]`:
- `auth` → `auth_subcommand(&args[2..])` on a fresh multi-thread runtime (`lib.rs:111-116`). Accepts `databricks`, `databricks_v2`, `databricks-v2` (`lib.rs:132`); any other value errors `auth: unknown provider {other:?}` (`lib.rs:150`); missing provider errors with a usage hint (`lib.rs:151`).
- anything else (including `--help`, `--version`, or a typo) → falls through to `async_main()` and the agent begins reading stdio (`lib.rs:117-120`). There is no `--help`/`--version` and unknown arguments are silently ignored.

Exit codes: `0` normal EOF **and** after a fatal reader error (which is only logged, `lib.rs:170-176`, then `Ok(())` at `lib.rs:121`); `1` from `main.rs:4`; `2` from `die()` on config/LLM-construction failure (`lib.rs:105-108`). When invoked through the `sprig` multicall binary, dispatch is by `argv[0]` (`crates/sprig/src/main.rs:10-19`) so `argv[1]` semantics are preserved.

#### Advertised vs actual capability
`initialize` advertises `promptCapabilities.image:false` (`lib.rs:293`) and the code enforces it — only `text` and `resource_link` blocks are accepted (`agent.rs:618-631`), covered by `test_unsupported_content_block` (`tests/golden_transcripts.rs:384`). `loadSession:false` is honored: no `session/load` handler exists. Version negotiation is `min(client, PROTOCOL_VERSION=2)` (`lib.rs:284`, `config.rs:3`), tested by `test_initialize_version_check` (`tests/golden_transcripts.rs:288`, asserts 99→2 and 1→1).


## Module: buzz-agent — LLM providers & configuration (`crates/buzz-agent/src`)
### Aspect: API Surface
Two very different visibility stories sit in this group. `config` is a `pub mod` (`lib.rs:6`) so everything marked `pub` in `config.rs` is part of the crate's external API. `llm` is a **private** module (`lib.rs:9`), so every `pub` item in `llm.rs` is effectively crate-internal despite the keyword.

#### Module visibility
| Declaration | Site | Effect |
|---|---|---|
| `pub mod config;` | `lib.rs:6` | all `pub` items in `config.rs` are externally reachable as `buzz_agent::config::*` |
| `mod llm;` | `lib.rs:9` | `pub struct Llm`, `pub fn new`, `pub async fn complete`, `pub async fn summarize` are **not** externally reachable |
| `pub use config::Provider;` | `lib.rs:15` | the only `config` item re-exported at crate root |

`lib.rs:14` and `lib.rs:16` re-export from `catalog` and `types`; nothing else from `config` or `llm` is re-exported. So `buzz_agent::Provider` is a supported name, while `buzz_agent::config::Config` requires the longer path.

#### Public API from `config.rs`
| Item | Signature / shape | Site | Doc comment? |
|---|---|---|---|
| `PROTOCOL_VERSION` | `pub const u32 = 2` | `config.rs:3` | no |
| `ThinkingEffort` | `pub enum`, 7 variants | `config.rs:19-27` | yes (`config.rs:5-17`) |
| `ThinkingEffort::anthropic_budget_tokens` | `pub fn(self) -> u32` | `config.rs:33` | yes |
| `ThinkingEffort::openai_effort_str` | `pub fn(self) -> &'static str` | `config.rs:45` | yes |
| `ThinkingEffort::anthropic_effort_str` | `pub fn(self) -> &'static str` | `config.rs:60` | yes |
| `anthropic_thinking_config` | `pub fn(&str, ThinkingEffort, u32) -> (Option<Value>, Option<Value>)` | `config.rs:124-128` | yes (`config.rs:100-123`) |
| `clamp_adaptive_effort` | `pub fn(&str, ThinkingEffort) -> ThinkingEffort` | `config.rs:205` | yes (`config.rs:191-204`) |
| `anthropic_efforts_for_model` | `pub fn(&str) -> (&'static [ThinkingEffort], Option<ThinkingEffort>)` | `config.rs:415-417` | yes (`config.rs:403-414`) |
| `normalize_effort_for_openai_route` | `pub fn(ThinkingEffort, &str) -> ThinkingEffort` | `config.rs:538` | yes (`config.rs:522-537`) |
| `normalize_effort_for_anthropic_route` | `pub fn(ThinkingEffort) -> Option<ThinkingEffort>` | `config.rs:563` | yes (`config.rs:552-562`) |
| `parse_thinking_effort` | `pub fn(Option<&str>) -> Result<Option<ThinkingEffort>, String>` | `config.rs:622` | one-liner (`config.rs:621`) |
| `MAX_PROMPT_BYTES` | `pub const usize` = 1 MiB | `config.rs:638` | no |
| `MAX_SYSTEM_PROMPT_BYTES` | `pub const usize` = 512 KiB | `config.rs:639` | no |
| `MAX_TOOL_RESULT_BYTES` | `pub const usize` = 8 MiB | `config.rs:643` | yes (`config.rs:640-642`) |
| `DEFAULT_TOOL_RESULT_TEXT_BYTES` | `pub const usize` = 50 KiB | `config.rs:649` | yes (`config.rs:644-648`) |
| `MAX_TOOL_CALLS_PER_TURN` | `pub const usize` = 64 | `config.rs:650` | no |
| `HANDOFF_MAX_OUTPUT_TOKENS` | `pub const u32` = 8192 | `config.rs:652` | no |
| `HANDOFF_ORIGINAL_TASK_MAX_BYTES` | `pub const usize` = 16 KiB | `config.rs:654` | no |
| `HANDOFF_MAX_TOOL_NAMES` | `pub const usize` = 20 | `config.rs:656` | no |
| `Provider` | `pub enum`, 4 variants | `config.rs:662-678` | variants 3-4 documented (`config.rs:665-677`), `Anthropic`/`OpenAi` not |
| `OpenAiApi` | `pub enum`, 3 variants | `config.rs:680-684` | yes (`config.rs:673-678`) |
| `Config` | `pub struct`, **27** `pub` fields | `config.rs:687-739` | 7 of 27 fields documented (`config.rs:701-704`, `:706-711`, `:716-717`, `:719-721`, `:727`, `:729-733`, `:736-737`); the other 20 carry none |
| `Config::from_env` | `pub fn() -> Result<Self, String>` | `config.rs:742` | **no doc comment** |
| `Config::for_discovery` | `pub fn(Provider, String, String) -> Self` | `config.rs:845` | yes (`config.rs:837-844`) |
| `is_openai_host` | `pub fn(&str) -> bool` | `config.rs:1037` | yes (`config.rs:1033-1036`) |
| `HookServers` | `pub enum`, 3 variants | `config.rs:1063-1067` | yes (`config.rs:1057-1061`) |
| `HookServers::allows` | `pub fn(&self, &str) -> bool` | `config.rs:1071` | yes |
| `HookServers::is_disabled` | `pub fn(&self) -> bool` | `config.rs:1080` | yes |

`Config` grew one field in `16d4ec33`: `pub prefer_mesh_for_auto: bool` (`config.rs:734`). Because every field is `pub` and there is no constructor discipline, this is a **breaking change for any external struct-literal construction** of `Config` — and it broke exactly that inside the crate: the `llm.rs` test fixture had to be extended at `llm.rs:1606`. It is the first field in the struct whose only production consumer is `llm.rs` (read at `llm.rs:412`).

`Config::validate` is **private** (`config.rs:878`), so an external caller who builds a `Config` literal (all fields are `pub`) or uses `for_discovery` has no way to run the invariant checks. That makes `Config` a public struct with un-runnable invariants from outside the crate.

#### Undocumented public items
- `Config::from_env` (`config.rs:742`) — the crate's primary entry point for configuration, with no doc comment at all. Contradicts the `AGENTS.md` rule "New public API must have doc comments".
- `PROTOCOL_VERSION` (`config.rs:3`), `MAX_PROMPT_BYTES` (`config.rs:638`), `MAX_SYSTEM_PROMPT_BYTES` (`config.rs:639`), `MAX_TOOL_CALLS_PER_TURN` (`config.rs:650`), and the three `HANDOFF_*` constants (`config.rs:652`, `config.rs:654`, `config.rs:656`) — all `pub`, none documented.
- **20 of the 27** `Config` fields carry no doc comment (`config.rs:688-700`, `config.rs:713-715`, `config.rs:723-726`, `config.rs:735`). The count is unchanged by `16d4ec33` even though the struct grew: `prefer_mesh_for_auto` arrived *with* a five-line doc comment (`config.rs:729-733`) that names its env var, so the field/doc ratio improved from 6-of-26 to 7-of-27.

#### Public items with no caller outside their own file
| Item | Only production caller | Note |
|---|---|---|
| `clamp_adaptive_effort` (`config.rs:205`) | `config.rs:166`, same file | `pub` with no cross-file use; grep across `crates/**/*.rs` excluding `config.rs` returned zero matches |
| `anthropic_efforts_for_model` (`config.rs:415`) | **none in production** — sole caller is the test helper at `config.rs:2596` | see below |
| `parse_thinking_effort` (`config.rs:622`) | `config.rs:833`, same file | `pub` with no cross-file use |
| `is_openai_host` (`config.rs:1037`) | `llm.rs:546` | crate-internal only, but `pub` |
| `DEFAULT_TOOL_RESULT_TEXT_BYTES` (`config.rs:649`) | `config.rs:824`, same file | `pub` with no cross-file use |
| `ThinkingEffort::anthropic_budget_tokens` (`config.rs:33`) | `config.rs:145`, same file | `pub` |
| `ThinkingEffort::anthropic_effort_str` (`config.rs:60`) | `config.rs:169`, same file | `pub` |

`anthropic_efforts_for_model` is the notable one. Its doc comment asserts (`config.rs:409-413`): "Both `anthropic_thinking_config` (request-time) and the effort-table UI ... must derive their behaviour from this helper so the two stay in sync." But `anthropic_thinking_config` (`config.rs:124-178`) does **not** call it — it calls `is_manual_budget_model` (`config.rs:136`), `is_adaptive_thinking_model` (`config.rs:161`), and `clamp_adaptive_effort` (`config.rs:166`) directly. The two functions duplicate the same three-way family classification independently. The doc is aspirational, not descriptive.

#### `llm.rs` items (nominally `pub`, actually crate-internal)
| Item | Signature | Site |
|---|---|---|
| `Llm` | `pub struct` with **4** private fields | `llm.rs:88-105` |
| `Llm::new` | `pub fn(&Config) -> Result<Self, AgentError>` | `llm.rs:108` |
| `Llm::complete` | `pub async fn(&self, &Config, &str, &[HistoryItem], &[ToolDef], &str) -> Result<LlmResponse, AgentError>` | `llm.rs:123-130` |
| `Llm::summarize` | `pub async fn(&self, &Config, &str, &str, u32, &str) -> Result<String, AgentError>` | `llm.rs:230-237` |
| `build_token_source` | `pub(crate) fn(&Config) -> Result<Arc<dyn TokenSource>, AgentError>` | `llm.rs:1529` |

`build_token_source` is the only correctly-scoped one (`pub(crate)`) and the only one with a cross-file caller: `catalog.rs:117` (imported at `catalog.rs:17`). `Llm::new` is called at `lib.rs:161`; `Llm::complete` at `agent.rs:124`.

The four public signatures are **unchanged** by these commits — the mesh work is entirely additive below `Llm::complete`. `Llm::new` and `Llm::complete` still have **no doc comments** (`llm.rs:108`, `llm.rs:123`); `Llm::summarize` likewise (`llm.rs:230`). Every private helper is documented, including all three new mesh methods (`llm.rs:340-343`, `llm.rs:406-409`, `llm.rs:532-534`).

#### New crate-internal items added by the mesh work
All private to `llm.rs`, all introduced in `16d4ec33` (with `mesh_catalog_supports_collective`'s `@main`-normalisation refined by `8eb6e3eb`):

| Item | Signature / shape | Site | Doc comment? |
|---|---|---|---|
| `MESH_VIRTUAL_MODEL_ID` | `const &str = "mesh"` | `llm.rs:28` | no |
| `MESH_AUTO_MODEL_ID` | `const &str = "auto"` | `llm.rs:29` | no |
| `MESH_AUTO_CATALOG_TTL` | `const Duration` = 5 s | `llm.rs:30` | no |
| `MESH_AUTO_CATALOG_TIMEOUT` | `const Duration` = 2 s | `llm.rs:31` | no |
| `MESH_AUTO_COOLDOWN` | `const Duration` = 30 s | `llm.rs:32` | no |
| `MESH_AUTO_ENABLE_OBSERVATIONS` | `const u8 = 2` | `llm.rs:33` | no |
| `MESH_MOA_UNAVAILABLE_MESSAGE` | `const &str` — the provider's exact 503 text | `llm.rs:34` | no |
| `MeshCatalogObservation` | `enum`, 3 variants | `llm.rs:41-45` | no (the `Unknown` semantics are commented at the use site, `llm.rs:452-456`) |
| `mesh_catalog_supports_collective` | `fn(&Value) -> Option<bool>` | `llm.rs:47-64` | no |
| `looks_like_unstructured_tool_call` | `fn(&str) -> bool` | `llm.rs:66-72` | no |
| `MeshAutoState` | `struct`, 4 fields, `Debug + Default` | `llm.rs:73-79` | no (described on the `Llm` field, `llm.rs:95-97`) |
| `Llm::cool_down_collective` | `async fn(&self)` | `llm.rs:397-404` | no |
| `Llm::resolve_openai_model` | `async fn(&self, &Config, &str) -> String` | `llm.rs:410-469` | yes (`llm.rs:406-409`) |
| `Llm::observe_mesh_virtual_model` | `async fn(&self, &Config) -> MeshCatalogObservation` | `llm.rs:472-530` | no |
| `Llm::openai_request_for_model` | `async fn(&self, &Config, &str, &mut F) -> Result<LlmResponse, PostError>` | `llm.rs:535-574` | yes (`llm.rs:532-534`) |
| `Llm::post_anthropic` | `async fn(&self, &Config, &Value) -> Result<Value, AgentError>` | `llm.rs:330-338` | no |
| `PostError` | `enum`, 2 variants | `llm.rs:1344-1347` | yes (`llm.rs:1337-1342`) |
| `PostError::into_agent` | `fn(self) -> AgentError` | `llm.rs:1350-1356` | no |
| `impl From<AgentError> for PostError` | `fn(AgentError) -> Self` | `llm.rs:1358-1362` | no |
| `is_mesh_moa_unavailable_body` | `fn(&str) -> bool` | `llm.rs:1364-1375` | no |
| `is_mesh_moa_failure_body` | `fn(&str) -> bool` | `llm.rs:1377-1388` | no |

Two existing signatures changed shape rather than arity:
- `post` (`llm.rs:1390-1396`) gained a `detect_mesh_fallback: bool` parameter (`llm.rs:1394`) and now returns `Result<Value, PostError>` instead of `Result<Value, AgentError>` (`llm.rs:1396`).
- `post_openai` (`llm.rs:599-605`) likewise returns `Result<Value, PostError>` (`llm.rs:605`). Its three callers each adapt differently: `post_anthropic` collapses with `map_err(PostError::into_agent)` (`llm.rs:337`), `databricks_v2_request` does the same (`llm.rs:590`), and `openai_request_for_model` propagates the `PostError` so `openai_request` can branch on it (`llm.rs:540-573`).

#### Provider dispatch: enum + match, no trait
There is no provider trait. `Llm::complete` (`llm.rs:123-228`) dispatches on `cfg.provider` in a single `match` (`llm.rs:132-215`) with three arms — `Anthropic` (`llm.rs:133-146`), `OpenAi | Databricks` (shared, `llm.rs:147-176`), and `DatabricksV2` (`llm.rs:177-214`). `Llm::summarize` (`llm.rs:230-328`) repeats the identical three-arm structure. This matches the design claim in `crates/buzz-agent/README.md:181`: "`Provider` is a Rust `enum` with one `match` in `Llm::complete`. There is no trait, no `Box<dyn>`, no async-trait."

That claim is now half-true in two ways: `Llm::summarize` (`llm.rs:230`) contains a *second* full provider `match`, and there *is* a `Box<dyn>`-equivalent — `Arc<dyn TokenSource>` (`llm.rs:104`) with `#[async_trait]` methods, so adding a provider means touching `build_token_source` (`llm.rs:1529-1553`) as well. `16d4ec33` added a third wrinkle: `resolve_openai_model` (`llm.rs:411-415`) now branches on `Provider::OpenAi` *outside* both matches, so provider-specific behaviour is no longer confined to the two `match` sites. The README's "Adding a provider is a `match` arm and one `body`/`parse` pair in `llm.rs`" (`README.md:181`) understates it by four sites.

Route selection helpers, all private to `llm.rs`:
| Helper | Purpose | Site |
|---|---|---|
| `resolve_openai_model` | effective model → request model (mesh `auto` policy) | `llm.rs:410-469` |
| `mesh_catalog_supports_collective` | `/models` catalog → is MoA viable? | `llm.rs:47-64` |
| `databricks_v2_route_for_model` | model name → `DatabricksV2Route` | `llm.rs:970-982` |
| `databricks_v2_path` | route → static URL path | `llm.rs:983-990` |
| `is_responses_required_error` | provider error text → should auto-upgrade | `llm.rs:963-968` |
| `is_mesh_moa_unavailable_body` / `is_mesh_moa_failure_body` | provider error body → mesh fallback signal | `llm.rs:1364-1375`, `llm.rs:1377-1388` |
| `looks_like_unstructured_tool_call` | model text → pseudo-tool-call markup | `llm.rs:66-72` |
| `strip_model` | remove top-level `model` for legacy Databricks | `llm.rs:1560-1570` |
| `map_stop`, `sum_usage`, `str_field`, `make_tool_call` | shared parse helpers | `llm.rs:1092`, `llm.rs:1107`, `llm.rs:1150`, `llm.rs:1248` |
| `read_error_body`, `backoff_with_jitter`, `is_retryable_transport_error`, `terminal_llm_error`, `post` | HTTP layer | `llm.rs:1268`, `llm.rs:1289`, `llm.rs:1311`, `llm.rs:1323`, `llm.rs:1390` |

#### CLI flags
**Neither file parses any CLI argument.** grep for `std::env::args`, `clap`, `argh`, `structopt` in `llm.rs` and `config.rs` returned zero matches. The only argument handling in the crate is `lib.rs:111-117`, which recognises a single `auth` subcommand. `Config::from_env` (`config.rs:742`) reads process environment exclusively — including the new `BUZZ_AGENT_PREFER_MESH_FOR_AUTO` (`config.rs:807`), which has no flag equivalent.

`crates/buzz-agent/README.md:128` states "Everything is environment variables. No flags, no config files." That is accurate for this group but stale for the crate: `buzz-agent auth databricks` exists (`lib.rs:111-113`, handler `lib.rs:129-153`) and is documented nowhere in the README's Configuration section.

#### Test coverage for this aspect
Every public `config.rs` function has direct unit tests: `parse_thinking_effort` (`config.rs:1311`, `config.rs:1330`, `config.rs:1337`, `config.rs:1349`), `anthropic_thinking_config` (24 tests, `config.rs:1408-2191`), `clamp_adaptive_effort` (14 tests, `config.rs:1659-2141`), `normalize_effort_for_openai_route` (`config.rs:1996`, `config.rs:2421-2532`), `normalize_effort_for_anthropic_route` (`config.rs:2026`, `config.rs:2035`, `config.rs:2044`), `is_openai_host` (`config.rs:1275`), `HookServers::allows` (`config.rs:1176`, `config.rs:1184`, `config.rs:1189`), the three `ThinkingEffort` string/budget mappers (`config.rs:1359`, `config.rs:1372`, `config.rs:1383`).

Not covered:
- `Config::from_env` (`config.rs:742`) — zero tests; grep for `from_env()` inside the `config.rs` test module returned only two comment mentions (`config.rs:1256`, `config.rs:1842`). `prefer_mesh_for_auto`'s env parse (`config.rs:807`) inherits that gap.
- `Config::for_discovery` (`config.rs:845`) is exercised only as a fixture builder (`config.rs:1848`), never asserted on.
- `HookServers::is_disabled` (`config.rs:1080`) — grep for `is_disabled` in `config.rs` returned only the definition; no test.
- `Llm::new` (`llm.rs:108`) — **Status: resolved** in `16d4ec33` (all 10 async mesh tests call `Llm::new(&config).unwrap()` — `llm.rs:1837`, `llm.rs:1876`, `llm.rs:1940`, `llm.rs:1979`, `llm.rs:2025`, `llm.rs:2069`, `llm.rs:2113`, `llm.rs:2127`, `llm.rs:2145`, `llm.rs:2161` — so the production `connect_timeout` + `read_timeout` wiring at `llm.rs:109-113` is now constructed and exercised end-to-end against a local stub). The original finding: no test constructed it; the test helper `llm_with` (`llm.rs:3634-3648`) bypassed it entirely and built the struct literal with a *different* `reqwest::Client` configuration (`.timeout(5s)` at `llm.rs:3637` instead of `.connect_timeout(10s).read_timeout(...)`). `llm_with` still does that, so the *auth* tests continue to run against a stricter client than production.
- `Llm::complete` (`llm.rs:123`) — **Status: resolved** in `16d4ec33` (the mesh tests drive it through helpers `complete_model` (`llm.rs:1765-1778`) and `complete_model_with_tool` (`llm.rs:1780-1804`), covering the `OpenAi` arm end-to-end; `explicit_models_are_never_rewritten_or_fallback_retried` (`llm.rs:2136`) asserts on the returned error string, which exercises the error-stamping `map_err` at `llm.rs:222-228` indirectly). Still untested: the `Anthropic` and `DatabricksV2` arms, and any direct assertion on the `(model-name)` prefix the `map_err` adds.
- `Llm::summarize` (`llm.rs:230`) — still zero tests; grep for `summarize` in the `llm.rs` test module returns no matches.


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
| `catalog::ModelEntry` | `catalog.rs:23-27` | `{ id, name }` | yes (`catalog.rs:21-22`) |
| `catalog::DATABRICKS_V2_KNOWN_MODELS` | `catalog.rs:32-33` | `&[&str; 2]` | yes (`catalog.rs:29-31`) |
| `catalog::discovery_failure_fallback` | `catalog.rs:51-80` | `(Provider, &str) -> Vec<ModelEntry>` | yes (`catalog.rs:35-50`) |
| `catalog::discover_databricks_models` | `catalog.rs:116-130` | `async (&Config) -> Result<Vec<ModelEntry>, AgentError>` | yes (`catalog.rs:108-115`) |

`discovery_failure_fallback`'s signature is unchanged by `8eb6e3eb`, but its **return contract is not**: for `Provider::DatabricksV2` it now yields `DATABRICKS_V2_KNOWN_MODELS.len() + 1` entries with the (trimmed) configured model first, rather than just the known list (`catalog.rs:61-76`; asserted `lib.rs:926-941`). Any caller that assumed `len() == DATABRICKS_V2_KNOWN_MODELS.len()` sees one more entry. The count drops back to `DATABRICKS_V2_KNOWN_MODELS.len()` when the configured model is blank after trimming (`catalog.rs:63-65`) or already a member of the known list (`catalog.rs:69`).

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
| `catalog::parse_v1_endpoints` / `parse_v2_endpoints_page` | `catalog.rs:171`, `catalog.rs:353` | `pub(crate)` purely for unit tests. `parse_v2_endpoints_page`'s return type changed in `8eb6e3eb` from `(Vec<ModelEntry>, Option<String>)` to `(Vec<V2Endpoint>, Option<String>)` (`catalog.rs:355`) |
| `catalog::V2Endpoint` + fields | `catalog.rs:306-313` | `pub(crate)`; new in `8eb6e3eb`; documented (`catalog.rs:306`, field doc `catalog.rs:310-311`) |
| `catalog::is_chat_capable_endpoint` | `catalog.rs:97-106` | `pub(crate)` for unit tests; new in `8eb6e3eb`; documented (`catalog.rs:82-96`) |
| `catalog::sort_v2_endpoints_newest_first` | `catalog.rs:337-344` | `pub(crate)` for unit tests; new in `8eb6e3eb`; documented (`catalog.rs:327-336`) |
| `catalog::endpoint_created_ms` | `catalog.rs:320-325` | private (not `pub(crate)`) — reached only through `parse_v2_endpoints_page`; documented (`catalog.rs:315-319`) |

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
| `{host}/api/2.0/serving-endpoints` | GET | `catalog.rs:136-164` | `bearer_auth` |
| `{host}/api/ai-gateway/v2/endpoints?page_size=100[&page_token=…]` | GET | `catalog.rs:235-304` | `bearer_auth` |

Both request shapes are unchanged by `8eb6e3eb` — no query parameter, header, or path was added. The whole of that commit's behaviour change is post-parse: filtering (`catalog.rs:370-373`) and sorting (`catalog.rs:298`).

Local inbound listener: `browser_pkce_flow` binds an axum router with a single `GET /` route on `127.0.0.1:0` (`auth.rs:539-577`) for the redirect callback. It is not part of the crate's declared API and is torn down on every exit path via `AbortOnDrop` (`auth.rs:584-586`, `auth.rs:426-432`).

#### Error variants surfaced

All five files return `crate::types::AgentError`. Usage is uneven:

| Variant | Used by | Sites (verified by `grep -n`) |
|---|---|---|
| `Mcp` | every failure in `mcp.rs` (23 construction sites) | `mcp.rs:136,142,178,198,201,233,239,243,250,496,502,528,535,571,588,613,624,702,738,760,763,770,773` |
| `Cancelled` | `do_call` cancel branches | `mcp.rs:593`, `mcp.rs:602` |
| `Llm` | all OAuth failures (24 sites) and all catalog HTTP failures (8 sites) | `auth.rs:148,166,168,171,176,182,193,197,199,221,224,229,458,486,511,520,573,576,603,604,605,620,623,628`; `catalog.rs:147,152,158,176,261,267,273,360` |
| `LlmAuth` | refresh-token exhaustion and the no-browser path | `auth.rs:341`, `auth.rs:355`, `auth.rs:416` |
| `InvalidParams` | wrong provider passed to discovery | `catalog.rs:126-130` |

Re-verified after `8eb6e3eb`: still 8 catalog `Llm` sites and 1 `InvalidParams` site. The new filter, sort, and timestamp-parse paths add **no** error variants — every failure mode they introduce degrades to a dropped entry or a `None` sort key instead (`catalog.rs:370-373`, `catalog.rs:320-325`).

`AgentError::json_rpc_code()` (`types.rs:249-256`) maps `LlmAuth → -32001` and everything else in this group to `-32000` / `-32602`, so an OAuth failure raised as `Llm` (e.g. a dead discovery endpoint, `auth.rs:166`) is indistinguishable on the wire from a provider outage.

#### Test coverage — API Surface

Covered: the public `TokenSource` surface is exercised through the trait from an integration test (`tests/databricks_oauth.rs:20` imports `PkceOAuthConfig, PkceOAuthTokenSource, TokenSource`; `cache_hit_short_circuits_network` `:105`, `expired_cache_silently_refreshes` `:144`, `refreshed_token_is_persisted_to_disk` `:179`, `refresh_now_runs_grant_on_unexpired_rejected_token` `:212`, `refresh_now_coalesces_when_another_caller_already_refreshed` `:261`, `refresh_now_without_refresh_token_is_terminal` `:305`). `load_skill`'s schema-driven behaviour is covered by `builtin.rs:277-573` (12 `#[tokio::test]`s) plus one end-to-end path (`tests/hints_integration.rs:517 load_skill_tool_returns_body`). `catalog`'s pure functions are covered by fifteen unit tests (`catalog.rs:401-630`) — six before `8eb6e3eb`, nine added by it: `v2_parse_drops_embedding_endpoints` (`:498`), `v2_parse_reads_created_timestamp_in_either_wire_shape` (`:522`), `v2_endpoints_sort_newest_first_then_by_name` (`:542`), `is_chat_capable_endpoint_keeps_unrecognised_names` (`:578`), and four `v2_discovery_failure_fallback_*` (`:590`, `:603`, `:612`, `:621`). That makes `discovery_failure_fallback`, `is_chat_capable_endpoint`, `endpoint_created_ms` and `sort_v2_endpoints_newest_first` all directly exercised.

Not covered: `discover_databricks_models` itself, and both HTTP fetchers — `grep -rn 'discover_databricks_models\|fetch_v1_models\|fetch_v2_models\|percent_encode' crates/buzz-agent/tests` still returns zero matches, as does `grep -rn 'sort_v2_endpoints_newest_first\|endpoint_created_ms\|is_chat_capable_endpoint\|V2Endpoint' crates/buzz-agent/tests` — the new coverage is entirely in-file unit tests, so nothing wires the filter and sort to a real HTTP response. The only test of the discovery *contract* lives in `lib.rs:832-878` and injects a future instead of calling the real function. `interactive_login` (`auth.rs:235`) has no test. `McpRegistry::call_hooks` is covered indirectly through the agent loop (`tests/regressions.rs:787`, `872`, `927`, `979`, `1035`, `1112`, `1514`) but never called directly, so its documented determinism ("results in config order", `mcp.rs:309-310`) is untested with more than one hook server.


## Module: buzz-dev-mcp (`crates/buzz-dev-mcp`)
### Aspect: API Surface

#### Rust public API

Exactly one item is reachable from outside the crate: `pub fn run() -> Result<(),
Box<dyn std::error::Error>>` (`crates/buzz-dev-mcp/src/lib.rs:138`). Every module
is private (`mod paths; mod read_file; … mod view_image;`, `lib.rs:13-21`), so the
`pub` items inside them (`shell::run`, `todo::text_result`, `paths::read_text_file`,
etc.) are crate-internal despite the `pub` keyword. `run()` carries **no doc
comment** (`lib.rs:137-138`).

Two `pub(crate)` helpers exist for Windows console suppression:
`configure_no_window` (`lib.rs:191`) and `configure_no_window_async`
(`lib.rs:205`).

#### Binary / CLI surface — multicall dispatch

`buzz-dev-mcp` is a multicall binary. `run()` reads `argv[0]`'s file stem,
lowercases it, and dispatches (`lib.rs:138-160`):

| `argv[0]` stem | Behaviour | Site |
|---|---|---|
| `rg` | `std::process::exit(rg::run(args[1..]))` — sync, no tokio runtime built | `lib.rs:149` |
| `tree` | `std::process::exit(tree::run(args[1..]))` — sync | `lib.rs:150` |
| `git-credential-nostr` | `std::process::exit(git_credential_nostr::run())` | `lib.rs:151` |
| `git-sign-nostr` | `std::process::exit(git_sign_nostr::run())` | `lib.rs:152` |
| `buzz` | `std::process::exit(buzz_cli::run_from_args(std::env::args()).await)` — needs the runtime | `lib.rs:168-171` |
| anything else | MCP server mode over stdio | `lib.rs:173-186` |

There are **no flags of its own** — no `--help`, no `--version`, no argument
parsing in MCP-server mode. Unknown `argv[0]` names fall through to server mode
rather than erroring. `crates/sprig/src/main.rs:39-41` relies on this: sprig
forwards any unmatched personality to `buzz_dev_mcp::run()`.

#### MCP server registration

Server info: name `"buzz-dev-mcp"`, version `env!("CARGO_PKG_VERSION")`, only the
`tools` capability enabled, plus `instructions` set from
`SharedState.bootstrap_instructions` (`lib.rs:126-136`). Transport is stdio
(`rmcp::transport::stdio`, `lib.rs:7`, `lib.rs:185`). Tracing goes to **stderr**
with ANSI disabled (`lib.rs:174-177`).

#### MCP tools — complete reference

Seven tools are registered via `#[tool_router]` / `#[tool(...)]` on
`impl DevMcp` (`lib.rs:30-125`). No tool is defined-but-unregistered; every
`#[tool]`-annotated method is inside the single `#[tool_router]` block.

| # | Tool name | Handler | Params type | Returns |
|---|---|---|---|---|
| 1 | `shell` | `lib.rs:44-50` | `ShellParams` | `CallToolResult` (JSON text block) |
| 2 | `read_file` | `lib.rs:56-61` | `ReadFileParams` | `String` |
| 3 | `view_image` | `lib.rs:67-72` | `ViewImageParams` | `CallToolResult` (text + image blocks) |
| 4 | `str_replace` | `lib.rs:78-83` | `StrReplaceParams` | `String` |
| 5 | `todo` | `lib.rs:89-98` | `TodoParams` | `CallToolResult` |
| 6 | `_Stop` | `lib.rs:105-110` | `HookParams` | `CallToolResult` |
| 7 | `_PostCompact` | `lib.rs:118-123` | `HookParams` | `CallToolResult` |

**`shell`** (`lib.rs:40-50` → `shell::run`, `shell.rs:130-323`)

- Params: `command: string` (required), `workdir: string?`, `timeout_ms: integer?`.
- Success: `CallToolResult::success` with one text block containing the JSON object
  documented in the Data Model aspect (`shell.rs:309-323`).
- `Err(ErrorData::invalid_params)`: command over 1,000,000 bytes
  (`shell.rs:135-140`); `workdir` missing or not a directory (`shell.rs:151-159`).
- `Ok(CallToolResult::error)` (not a protocol error): no shell could be resolved —
  returns the resolver's diagnostic text (`shell.rs:161-164`); spawn failed —
  `"failed to spawn shell: {e}"` (`shell.rs:183-190`); the request was cancelled —
  the literal string `"cancelled"` (`shell.rs:220-238`).
- The tool description advertises the on-PATH helpers (`rg`, `tree`, `buzz`) and
  the timeout defaults (`lib.rs:42`).

**`read_file`** (`lib.rs:52-61` → `read_file::run`, `read_file.rs:23-63`)

- Output is a header line `"{path} (lines {start}-{end} of {total})"` followed by
  `"{1-based-line-number}:{content}"` per line (`read_file.rs:48-58`), plus a
  continuation footer `"[showing lines … use offset=… to continue]"` when the
  window ends before EOF (`read_file.rs:59-63`).
- Two non-error string returns: `"{path} is empty (0 lines)"` (`read_file.rs:29-31`)
  and `"{path} (no lines in range, file has {n} lines)"` (`read_file.rs:39-44`).
- Errors come from `paths::read_text_file`: `invalid_params` for unresolvable path
  (`paths.rs:112-115`), non-regular file (`paths.rs:127-132`), file over 10 MiB
  (`paths.rs:131-142`), file grown mid-read (`paths.rs:154-161`);
  `internal_error` for stat/open/read failure and invalid UTF-8
  (`paths.rs:117-125`, `paths.rs:145-153`, `paths.rs:163-176`, `paths.rs:170-175`).

**`str_replace`** (`lib.rs:74-83` → `str_replace::run`, `str_replace.rs:25-106`)

- Success string: `"Replaced {1 occurrence|N occurrence(s)} in {abs-path}.\n\n{unified diff}"`
  (`str_replace.rs:97-106`). Diff has `context_radius(3)` and is cut at 64 KiB with
  a `"[diff truncated]"` marker (`str_replace.rs:140-155`).
- `invalid_params` errors: empty `old_str` (`str_replace.rs:26-31`); `old_str`/`new_str`
  over 1 MiB (`str_replace.rs:32-37`); zero matches — includes a truncated echo of
  `old_str` and an optional fuzzy nearest-line hint (`str_replace.rs:46-58`);
  multiple matches without `replace_all` (`str_replace.rs:60-68`); projected result
  over 10 MiB (`str_replace.rs:70-82`).
- `internal_error`: atomic write failure (`str_replace.rs:90-95`).

**`view_image`** (`lib.rs:63-72` → `view_image::run`, `view_image.rs:88-107`)

- Returns `[Content::text(header), Content::image(base64, mime)]` where header is
  `"{WxH}, {size} [(resized from WxH)] ({mime} from {source_label})"`
  (`view_image.rs:96-107`, `view_image.rs:530-611`).
- MIME is always one of `image/png` or `image/jpeg` on the resize path
  (`view_image.rs:640-654`), or the sniffed source MIME on the pass-through path
  (`view_image.rs:396-421`, `view_image.rs:560-567`).
- `invalid_params` errors: unsupported URL scheme (`view_image.rs:130-136`); data
  URL malformed / non-image / non-base64 / oversized (`view_image.rs:190-231`);
  file not a regular file or over 20 MiB (`view_image.rs:146-160`); non-2xx HTTP,
  with a distinct message naming `BUZZ_PRIVATE_KEY` on an unauthenticated
  401/403 (`view_image.rs:347-361`); `Content-Length` over cap
  (`view_image.rs:362-370`); mid-stream cap breach (`view_image.rs:378-384`);
  unsupported magic bytes (`view_image.rs:396-421`); animated GIF/WebP
  (`view_image.rs:534-536`); pixel count over 64 Mpx (`view_image.rs:549-558`);
  still oversized after two resize passes (`view_image.rs:589-595`).
- `internal_error`: HTTP client init, fetch failure, fetch read failure, stat/open/read
  failure (`view_image.rs:334-336`, `view_image.rs:344-346`, `view_image.rs:372-377`,
  `view_image.rs:141-144`).

**`todo`** (`lib.rs:85-98` → `TodoState::handle_todo`, `todo.rs:71-94`)

- Omitting `todos` (or sending `null`) reads; sending an array replaces
  (`todo.rs:72-94`). Output is the rendered checklist (`todo.rs:219-239`), or
  `"(todo list is empty)"`, with a `⚠️` warning block appended when open items
  disappeared (`todo.rs:78-92`, `todo.rs:192-218`).
- Validation failures return `Ok(CallToolResult::error)` with `"Error: {msg}"`
  (`lib.rs:93-97`, `todo.rs:245-248`), **not** a protocol error.

**`_Stop`** (`lib.rs:101-110`) returns `stop_objection()` — non-empty objection text
only when at least one item is not `done`, otherwise the empty string
(`todo.rs:99-112`). **`_PostCompact`** (`lib.rs:114-123`) returns
`post_compact()` — `"# Todo List\n{rendered}"`, or empty when the list is empty
(`todo.rs:113-124`).

#### `shim.rs`'s purpose

`shim.rs` is not an MCP surface; it constructs the execution environment the
`shell` tool hands to children (`Shim::install`, `shim.rs:25-76`):

1. Creates a `0700` tempdir (`shim.rs:26-27`).
2. Symlinks (Unix) or copies with `.exe` (Windows) the running binary under five
   names — `rg`, `tree`, `buzz`, `git-credential-nostr`, `git-sign-nostr` — which
   the multicall dispatch in `lib.rs:144-153` then recognises (`shim.rs:31-40`,
   `shim.rs:231-242`).
3. Prepends that dir to a `path_env` string built with `std::env::join_paths`
   (`shim.rs:42-49`).
4. Reads and unconditionally removes `NOSTR_PRIVATE_KEY` from the process env,
   writes it to a `0600` keyfile, and derives `GIT_CONFIG_COUNT` /
   `GIT_CONFIG_KEY_n` / `GIT_CONFIG_VALUE_n` for ten git settings
   (`shim.rs:51-68`, `shim.rs:178-216`).

`shim::artifact_dir` (`shim.rs:244-248`) creates and returns
`{session_root}/artifacts` and is the only other item `shell.rs` consumes from
this module (`shell.rs:914-915`). It has no doc comment.

#### Internal command-line surfaces (`rg`, `tree` personalities)

`rg::run` (`rg.rs:11-16`) first tries to exec the real system `rg`
(`try_system_rg`, `rg.rs:18-29`); when absent it parses a restricted flag set
itself (`rg.rs:87-135`): `--files`, `-n|--line-number`, `-i|--ignore-case`,
`-l|--files-with-matches`, `-C|--context <n>`, `-g|--glob <pat>`, `--`. Any other
`-`-prefixed token is rejected with `"unsupported flag (fallback rg): {s}"` and
exit 2 (`rg.rs:115-117`). Exit codes: `0` match found, `1` no match, `2` parse
error (`rg.rs:168-227`).

`tree::run` (`tree.rs:18-135`) accepts only `-d|--depth <n>`, a single positional
path, and `--` (`tree.rs:137-168`). Unknown flags → `"unknown flag: {s}"`, exit 2.
Output is an indented listing with per-file and per-directory line counts
(`tree.rs:100-134`); exit code is always `0` on a valid invocation.


## Module: buzz-cli — dispatch, relay client & validation (`crates/buzz-cli/src`)
### Aspect: API Surface

#### Rust library surface

`buzz-cli` ships both a lib (`buzz_cli`, `Cargo.toml:11-13`) and a bin (`buzz`,
`Cargo.toml:15-17`). `main.rs` is a 4-line shim:
`std::process::exit(buzz_cli::run_from_args(std::env::args()).await)`
(`main.rs:1-4`).

Module visibility (`lib.rs:1-5`): only `agent_management` is `pub`; `client`,
`commands`, `error`, `validate` are private. Everything `pub` inside
`client.rs`/`validate.rs`/`error.rs` is therefore crate-internal despite the
keyword.

| Public item | Kind | Site |
|---|---|---|
| `run_from_args<I, S>(args) -> i32` | async fn — the only real entry point | `lib.rs:23-60` |
| `ChannelType`, `ChannelVisibility`, `PresenceStatus`, `EmojiScope`, `OutputFormat`, `RespondToArg`, `RepoPushRole` | value enums | `lib.rs:101,118,135,145,164,242,1183` |
| `AgentsCmd`, `MessagesCmd`, `ChannelsCmd`, `CanvasCmd`, `ReactionsCmd`, `EmojiCmd`, `DmsCmd`, `UsersCmd`, `WorkflowsCmd`, `FeedCmd`, `SocialCmd`, `NotesCmd`, `ReposCmd`, `ReposProtectCmd`, `PatchesCmd`, `PrCmd`, `IssuesCmd`, `UploadCmd`, `MediaCmd`, `MemCmd`, `PackCmd`, `ModerationCmd` | subcommand enums | `lib.rs:260`–`lib.rs:1644` |
| `agent_management::{CreateAgentDraft, UpdateAgentDraft, BuiltDraftRequest, build_create, build_update}` | structs + fns | `agent_management.rs:15,23,64,128,144` |

**Exported but unreachable:** the 22 `*Cmd` enums are `pub`, yet `Cli`
(`lib.rs:79`) and `Cmd` (`lib.rs:175`) are private and no public function takes
or returns a `*Cmd`. An external crate can name `buzz_cli::MessagesCmd` but has
no way to feed one into the CLI. The single downstream consumer uses only the
entry point: `crates/buzz-dev-mcp/src/lib.rs:170` calls
`buzz_cli::run_from_args(std::env::args())` when invoked as `buzz`
(`grep -rn 'buzz_cli::' crates/ | grep -v '^crates/buzz-cli/'` → that one hit).
`crates/git-sign-nostr/Cargo.toml:23` mentions buzz-cli only in a comment.

Undocumented public items (AGENTS.md: "New public API must have doc comments"):
`ChannelType` (`lib.rs:101`), `ChannelVisibility` (`lib.rs:118`),
`PresenceStatus` (`lib.rs:135`), `EmojiScope` (`lib.rs:145`), and all of
`AgentsCmd`/`MessagesCmd`/`ChannelsCmd`/`CanvasCmd`/`ReactionsCmd`/`EmojiCmd`/
`DmsCmd`/`UsersCmd`/`WorkflowsCmd`/`FeedCmd`/`SocialCmd` carry no `///`
comment; `NotesCmd` (`lib.rs:1016`), `MemCmd` (`lib.rs:1540`), `PackCmd`
(`lib.rs:1623`), `ModerationCmd` (`lib.rs:1636-1642`), `ReposProtectCmd`
(`lib.rs:1141`) and `RepoPushRole` (`lib.rs:1182`) do. In
`agent_management.rs` the module has a `//!` header (`:1`) but none of its five
public items are documented.

#### Crate-internal client surface (`client.rs`)

| Item | Signature | Site |
|---|---|---|
| `BuzzClient::new` | `(String, Keys, Option<Tag>, Option<String>) -> Result<Self, CliError>` | `client.rs:541-560` |
| `keys` | `-> &Keys` | `client.rs:562-564` |
| `relay_url` | `-> &str` — `#[allow(dead_code)]`, zero callers in `commands/` | `client.rs:567-570` |
| `auth_tag_owner_hex` | `-> Option<String>` (index 1 of the auth tag) | `client.rs:576-580` |
| `sign_event` | injects NIP-OA tag, then *rejects* events with an unexpected `auth` tag count | `client.rs:588-614` |
| `sign_event_unchecked` | verbatim signing, NIP-IA only | `client.rs:743-747` |
| `get_public` | unauthenticated GET, `Accept: application/nostr+json` | `client.rs:753-765` |
| `get_authed` | NIP-98 GET of an arbitrary root-relative path | `client.rs:836-856` |
| `query` / `query_multi` | `POST /query` with one/many filters | `client.rs:767-801` |
| `query_paginated` / `query_all` | cursor-following wrappers over `query_pages` | `client.rs:715-727` |
| `count` | `POST /count` — `#[allow(dead_code)]`, zero callers in `commands/` | `client.rs:802-834` |
| `submit_event` | kind-aware dispatch to moderation vs stored policy | `client.rs:863-870` |
| `publish_ephemeral_event` | WS publish via `buzz_ws_client` | `client.rs:1073-1098` |
| `upload_file` | Blossom PUT with legacy fallback | `client.rs:1100-1227` |
| `download_media` | Blossom GET with origin pinning | `client.rs:1229-1256` |
| free fns | `normalize_relay_url` (`:1291`), `normalize_events` (`:1307`), `extract_d_tag` (`:1325`), `extract_tag_value` (`:1346`), `extract_p_tags` (`:1366`), `create_response_with_id` (`:1391`), `print_create_response` (`:1401`), `extract_relay_response_field` (`:1407`), `normalize_write_response` (`:1420`), `build_imeta_tag` (`:40`) | |

`validate.rs` exposes `parse_event_id` (`:11`), `parse_uuid` (`:19`),
`validate_uuid` (`:23`), `validate_hex64` (`:29`), `validate_repo_id` (`:39`),
`validate_content_size` (`:64`), `truncate_diff` (`:103`), `infer_language`
(`:124`), `sdk_err` (`:155`), `read_or_stdin` (`:162`), `read_file_or_stdin`
(`:180`) — plus `percent_encode` (`:76`), which is gated `#[cfg(test)]`
(`validate.rs:75`) and so is not part of the production surface at all.

`error.rs` exposes `CliError` (`:4`), `is_retryable_error` (`:74`), `exit_code`
(`:92`), `print_error` (`:127`).

#### CLI surface

Global flags (must precede the subcommand — see below):

| Flag | Value | Default |
|---|---|---|
| `--relay <RELAY>` | URL | `http://localhost:3000` (`lib.rs:81`) |
| `--private-key <PRIVATE_KEY>` | hex or nsec | none (`lib.rs:85`) |
| `--auth-tag <AUTH_TAG>` | NIP-OA JSON | none (`lib.rs:89`) |
| `--format <json\|compact>` | enum | `json` (`lib.rs:93`) |
| `-h`, `--help` | — | exits 0 (`lib.rs:48-52`) |

`--version` is **not** implemented: no `version` attribute on
`#[command(...)]` (`lib.rs:62-78`). Verified by running the built binary —
`buzz --version` prints
`{"error":"user_error","message":"error: unexpected argument '--version' found…}`
and exits **1**, even though `run_from_args` has a branch commented
"`--help` and `--version`: print normally" (`lib.rs:49`).

Subcommand inventory, as pinned by the crate's own tests
(`lib.rs:1806-1853`, `1855-1994`, `1996-2034`):

| Group | Subcommands | Count |
|---|---|---|
| `agents` | archive, archived, draft-create, draft-update, unarchive | 5 |
| `messages` | delete, edit, get, search, send, send-diff, thread, vote | 8 |
| `channels` | add-member, archive, create, delete, get, join, leave, list, members, purpose, remove-member, search, set-add-policy, topic, unarchive, update | 16 |
| `canvas` | get, set | 2 |
| `reactions` | add, get, remove | 3 |
| `emoji` | export, import, list, rm, set | 5 |
| `dms` | add-member, hide, list, open | 4 |
| `users` | get, presence, set-presence, set-profile | 4 |
| `workflows` | approve, create, delete, get, list, runs, trigger, update | 8 |
| `feed` | get | 1 |
| `social` | contacts, event, list, notes, publish, set-contacts, set-list | 7 |
| `notes` | get, ls, rm, set | 4 (untested inventory) |
| `repos` | create, get, list, protect{list,remove,set} | 4 (+3) |
| `patches` | get, list, send, status | 4 |
| `issues` | create, get, list, status | 4 |
| `pr` | get, list, open, status, update | 5 |
| `media` | get | 1 |
| `upload` | file | 1 |
| `mem` | get, hash, ls, patch, rm, set | 6 (untested inventory) |
| `moderation` | audit, ban, reports, resolve, restricted, timeout, unban, untimeout | 8 |
| `pack` | inspect, validate | 2 |

`--format` position is a hard contract, not a convention: because the arg lives
on the top-level `Cli` without `global = true`, clap rejects it after the
subcommand. Verified against the built binary:
`buzz messages get --channel <uuid> --format compact` →
`error: unexpected argument '--format' found`, exit 1; while
`buzz --format compact channels list` parses and proceeds to the relay call.
This confirms `AGENTS.md:192-193` and **contradicts the example at
`AGENTS.md:181`** (`buzz messages thread --channel … --event … --format compact`),
which cannot parse.

`messages search` has **no `--kinds` flag** (`lib.rs:472-489` defines only
`--query`, `--author`, `--since`, `--limit`). Verified:
`buzz messages search --query x --kinds 9` → `error: unexpected argument
'--kinds' found`, exit 1. `AGENTS.md` gotcha 3 ("`messages search` must include
`--kinds` … Pass at least `--kinds 9,45001,45003`") is therefore unfollowable;
the kinds are hardcoded in the sibling-owned handler
(`commands/messages.rs:361`). `--kinds` exists only on `messages get`
(`lib.rs:453-455`), where it is optional and defaults are supplied downstream
(`commands/messages.rs:276`).

#### Exit-code contract

`exit_code` (`error.rs:92-107`) plus the clap-parse path (`lib.rs:44-47`):

| Class | Code | Site | Verified |
|---|---|---|---|
| success | 0 | `lib.rs:55` | — |
| clap usage error | 1 | `lib.rs:44-47` | yes: `--version`, stray `--format`, stray `--kinds` all exit 1 |
| `Usage` | 1 | `error.rs:94` | yes: `channels get --channel not-a-uuid` → 1, `{"error":"user_error"}` |
| `NotFound` | 1 | `error.rs:104` | not exercised |
| `Relay{401,403}` | 3 | `error.rs:96-100` | test `query_403_is_not_retried` (`client.rs:1949-1977`) covers the error value, not the code |
| `Relay{other}` | 2 | `error.rs:99` | — |
| `Network` | 2 | `error.rs:102` | yes: dead port → 2, `{"error":"network_error","retryable":true}` |
| `Auth` | 3 | `error.rs:103` | yes: unset key → 3; malformed `BUZZ_AUTH_TAG` → 3 |
| `Key` | 3 | `error.rs:103` | yes: `BUZZ_PRIVATE_KEY=zzz` → 3, `{"error":"key_error"}` |
| `Conflict` | 5 | `error.rs:103` | not exercised |
| `DeliveryUnknown` | 2 | `error.rs:105` | not exercised at process level |
| `Other` | 4 | `error.rs:106` | not exercised |
| `--help` / `-h` | 0 | `lib.rs:48-52` | yes |

`exit_code` itself has **no unit test**:
`grep -c 'exit_code' <(awk 'NR>=137' error.rs)` returns 0. The documented
contract in `AGENTS.md:189-190` and `lib.rs:76` matches the implementation for
the six classes it names, but silently omits that `NotFound` shares code 1 with
input errors and `DeliveryUnknown` shares code 2 with network errors — so an
agent cannot distinguish "the write may have landed" from "the connection
failed" by exit code alone, only by the `error` field on stderr
(`delivery_unknown` vs `network_error`, `error.rs:117-125`).

A schemeless `--relay` is mis-classified: `buzz --relay notaurl channels list`
exits **2** with `{"error":"network_error","message":"network error: builder
error: relative URL without a base","retryable":false}` — a pure input error
reported as a network error, because nothing validates the scheme
(`normalize_relay_url`, `client.rs:1291-1297`, only does substring replacement).

#### Outbound relay surface

| Method + path | Auth | Caller |
|---|---|---|
| `POST /query` | NIP-98 kind 27235 + optional `x-auth-tag` | `query_multi` (`client.rs:773-801`) |
| `POST /count` | same | `count` (`client.rs:803-834`) — dead |
| `POST /events` | same | `submit_moderation_event` (`client.rs:873`), `submit_stored_event` (`client.rs:1024`) |
| `PUT /upload` | Blossom kind 24242 `t=upload` | `upload_file` (`client.rs:1152-1178`) |
| `PUT /media/upload` | same | legacy fallback on 404/405 (`client.rs:1195-1226`) |
| `GET /media/<sha256[.ext]>` | Blossom kind 24242 `t=get` | `download_media` (`client.rs:1229-1256`) |
| `GET <arbitrary path>` (authed) | NIP-98 GET | `get_authed` (`client.rs:836-856`); used for `/moderation/reports?…`, `/moderation/restricted`, `/moderation/audit?limit=…` (`commands/moderation.rs:114,120,127`) |
| `GET /` (NIP-11) | none | `get_public` (`client.rs:753-765`); called with `"/"` from `commands/agents.rs:273` |
| WebSocket `EVENT` after NIP-42 `AUTH` | Nostr AUTH | `publish_ephemeral_event` → `buzz_ws_client::publish_event` (`client.rs:1084`) |

The `/moderation/*` GET endpoints are outside the HTTP surface AGENTS.md
documents: `grep -c '/moderation/' AGENTS.md` → 0. Likewise the `x-auth-tag`
request header (`client.rs:616-621`) appears in no markdown doc
(`grep -rln 'x-auth-tag' --include='*.md' .` outside `.aidlc/` → no hits).

WebSocket message flow is fully delegated, not reimplemented: connect →
NIP-42 challenge wait → AUTH → EVENT → OK → close, all inside
`buzz_ws_client::publish_event` (`crates/buzz-ws-client/src/connection.rs:277-293`),
with a 75 s outer budget chosen to exceed the 20+20+30 s inner ceilings
(`client.rs:1075-1085`).


## Module: buzz-cli — channel, message & social commands (`crates/buzz-cli/src/commands`)
### Aspect: API Surface

Eight command groups live in these files: `messages`, `channels`, `canvas`,
`reactions`, `emoji`, `dms`, `feed`, `social`, `notes` (nine enums; `channels.rs`
serves both `channels` and `canvas`). Each group has a module-level `dispatch`
called from `lib.rs:1772-1782`. `mod.rs` is a bare re-export list of 20 command
modules (`mod.rs:1-20`) with no logic and no doc comment.

#### Dispatch entry points

| Group | Dispatcher | Called from | Receives `&OutputFormat`? |
|-------|-----------|-------------|---------------------------|
| `messages` | `messages.rs:754` | `lib.rs:1772` | yes |
| `channels` | `channels.rs:1066` | `lib.rs:1773` | yes |
| `canvas` | `channels.rs:1168` `dispatch_canvas` | `lib.rs:1774` | no |
| `reactions` | `reactions.rs:127` | `lib.rs:1775` | no |
| `emoji` | `emoji.rs:311` | `lib.rs:1776` | no |
| `dms` | `dms.rs:128` | `lib.rs:1777` | no |
| `feed` | `feed.rs:67` | `lib.rs:1780` | yes |
| `social` | `social.rs:211` | `lib.rs:1781` | no |
| `notes` | `notes.rs:752` | `lib.rs:1782` | no |

`--format` is a global flag (`lib.rs:93-94`, default `json`). Only three of the
nine dispatchers accept it, and inside those only the multi-event read paths
honour it: `messages get`/`thread`/`search` via `format_events`
(`messages.rs:242-261`), `channels list` via its own inline match
(`channels.rs:96-109`), and `feed get` via a third copy of the same logic
(`feed.rs:47-60`). `--format compact` is therefore silently ignored by
`channels search`, `channels get`, `channels members`, `canvas get`,
`reactions get`, `emoji list`/`export`, `dms list`, all `social` reads, and all
`notes` reads.

#### `messages` (8 subcommands)

| Subcommand | Handler | Relay calls | Output |
|-----------|---------|-------------|--------|
| `send` | `cmd_send_message` `messages.rs:483` | 0..N `upload_file` (`:509`), 1 `/query` per reply (`resolve_thread_ref` `:64`), 2 `/query` for mentions (`:145`,`:157`), 1 `POST /events` (`:576`) | `{event_id,accepted,message}` |
| `send-diff` | `cmd_send_diff_message` `messages.rs:596` | 1 `/query` if `--reply-to` (`:637`), 1 `POST /events` (`:664`) | same |
| `edit` | `cmd_edit_message` `messages.rs:701` | 1 `/query` (`resolve_channel_id` `:93`), 1 `POST /events` (`:718`) | same |
| `delete` | `cmd_delete_message` `messages.rs:669` | 1 `/query`, 1 `POST /events` (`:695`) | same |
| `get` | `cmd_get_messages` `messages.rs:263` | 1 `/query` (`:296`) | event array |
| `thread` | `cmd_get_thread` `messages.rs:304` | 1 `/query` with **two** ORed filters via `query_multi` (`:332`) | event array |
| `search` | `cmd_search` `messages.rs:340` | 0..1 `/query` for author resolution (`:411`), 1 `/query` (`:373`) | event array |
| `vote` | `cmd_vote_on_post` `messages.rs:724` | 1 `/query`, 1 `POST /events` (`:749`) | same |

Every write in this group is preceded by at least one read when it needs the
channel: `resolve_channel_id` (`messages.rs:89-115`) re-derives the `h` tag from
the target event rather than taking `--channel`, so `edit`, `delete` and `vote`
cost one extra round trip and fail with `CliError::Other` (exit 4, not 1) when
the event is missing (`messages.rs:98`) or has no `h` tag (`messages.rs:112-114`).

#### `channels` (16 subcommands) and `canvas` (2)

| Subcommand | Handler | Relay calls |
|-----------|---------|-------------|
| `list` | `cmd_list_channels` `channels.rs:25` | 1 paginated `/query` (kind 39000) or 2 when `--member` (`:41`,`:65`) |
| `get` | `cmd_get_channel` `channels.rs:224` | 1 `/query` |
| `search` | `cmd_search_channels` `channels.rs:119` | 1 paginated `/query` (`:133`) |
| `create` | `cmd_create_channel` `channels.rs:282` | 1 `POST /events` (`:326`) |
| `create --template` | `cmd_create_channel_from_template` `channels.rs:655` | 1 `/query` per team (`:410`), 1 `query_all` for 30177 (`:449`), 1 `GET /` + 1 `/query` for the archive snapshot (`agents.rs:273`,`:287`), then 1 `POST /events` for the channel (`:721`), 1 for canvas (`:732`), 1 per resolved agent (`:754`) |
| `update` | `cmd_update_channel` `channels.rs:832` | 1 `POST /events` |
| `topic` / `purpose` | `channels.rs:864` / `:880` | 1 `POST /events` each |
| `join` / `leave` | `channels.rs:896` / `:908` | 1 each |
| `archive` / `unarchive` / `delete` | `channels.rs:920` / `:932` / `:944` | 1 each |
| `members` | `cmd_list_channel_members` `channels.rs:244` | 1 `/query` |
| `add-member` / `remove-member` | `channels.rs:956` / `:987` | 1 each |
| `set-add-policy` | `cmd_set_add_policy` `channels.rs:1005` | 1 `POST /events` (kind 10100) |
| `canvas get` | `cmd_get_canvas` `channels.rs:262` | 1 `/query` |
| `canvas set` | `cmd_set_canvas` `channels.rs:1049` | 1 `POST /events` |

`channels create --template` is the only handler in the module with a multi-write
transaction shape, and it is explicitly non-atomic: roster resolution runs first
so an ambiguous roster aborts with zero writes (`channels.rs:659-661` doc,
`:715`), but the channel-create response is **discarded** (`channels.rs:721`
`client.submit_event(event).await?;`), so a relay `accepted:false` is invisible
and the printed report still claims `status:"ok"` plus a `channel_id`. Canvas and
per-member failures are collected, not fatal (`channels.rs:737`, `:763-767`).

#### `reactions` (3), `emoji` (5), `dms` (4), `feed` (1)

| Subcommand | Handler | Relay calls | Output |
|-----------|---------|-------------|--------|
| `reactions add` | `reactions.rs:9` | 1 `POST /events` | write response |
| `reactions remove` | `reactions.rs:34` | 1 `/query` (own kind 7 on the target `:49`), 1 `POST /events` (`:75`) | write response |
| `reactions get` | `reactions.rs:80` | 1 `/query` (`:86`) | `{"reactions":[…]}` |
| `emoji list` | `emoji.rs:77` | 1 `/query` (`:82`) | `{"emojis":[…]}` |
| `emoji set` | `emoji.rs:128` | 1 `/query` (read-own), 1 `POST /events` | write response |
| `emoji rm` | `emoji.rs:141` | 1 `/query`, then 1 `POST /events` **or** short-circuit print (`:148-153`) | write response, or `{"accepted":true,"message":"not present"}` |
| `emoji export` | `emoji.rs:197` | 1 `/query` (own or workspace) | `{"emojis":[…]}` to stdout or `--file` |
| `emoji import` | `emoji.rs:234` | 0..1 `/query` (merge mode `:290`), 1 `POST /events` unless `--dry-run` | write response, or dry-run set + stderr note |
| `dms list` | `dms.rs:8` | 1 `/query` | `[{dm_id,participants,created_at}]` |
| `dms open` | `dms.rs:51` | 1 `POST /events` | write response + `dm_id` |
| `dms add-member` | `dms.rs:112` | 1 `POST /events` | write response |
| `dms hide` | `dms.rs:96` | 1 `POST /events` | write response |
| `feed get` | `feed.rs:9` | 1 `/query` | event array |

#### `social` (7)

| Subcommand | Handler | Relay call | Output |
|-----------|---------|-----------|--------|
| `publish` | `cmd_publish_note` `social.rs:22` | `POST /events` (kind 1) | normalized write response |
| `set-contacts` | `cmd_set_contact_list` `social.rs:43` | `POST /events` (kind 3) | normalized write response |
| `event` | `cmd_get_event` `social.rs:72` | `/query` `{ids:[…]}` | **raw** relay JSON |
| `notes` | `cmd_get_user_notes` `social.rs:83` | `/query` kind 1 | **raw** relay JSON |
| `contacts` | `cmd_get_contact_list` `social.rs:115` | `/query` kind 3 | **raw** relay JSON |
| `set-list` | `cmd_set_list` `social.rs:162` | `POST /events` | **raw** relay JSON (not normalized — `social.rs:180`) |
| `list` | `cmd_get_list` `social.rs:184` | `/query` | **raw** relay JSON |

#### `notes` (4)

| Subcommand | Handler | Relay calls | Output |
|-----------|---------|-------------|--------|
| `set` | `cmd_set` `notes.rs:487` | 1 `/query` read-before-write (`:530`), 1 `POST /events` (`:545`) | 5 plain-text lines (`:571-580`) |
| `get` | `cmd_get` `notes.rs:612` | 1 `/query` by coordinate or slug; +1 when `--author` is a name (`:216`) | pretty JSON, or raw markdown with `--content-only` |
| `ls` | `cmd_ls` `notes.rs:671` | 0..1 `/query` for author resolution, 1 `/query` (`:697`) | pretty JSON array |
| `rm` | `cmd_rm` `notes.rs:717` | 1 `/query` read-before-write (`:721`), 1 `POST /events` (`:733`) | 2 plain-text lines (`:747-748`) |

`notes set` and `notes rm` are the only writes in the module that inspect the
relay's `accepted` flag and message (`notes.rs:546-567`, `notes.rs:734-745`).

#### Underlying transport

Everything goes through three `BuzzClient` methods, all NIP-98-signed with the
`x-auth-tag` header when configured (`client.rs:767`, `:773`, `:863`,
`:120-127`):

| Client method | HTTP | Used by |
|---------------|------|---------|
| `query` (`client.rs:767`) | `POST /query`, single filter | all reads except the four below |
| `query_multi` (`client.rs:773`) | `POST /query`, ORed filters | `messages thread` only (`messages.rs:332`) |
| `query_paginated` (`client.rs:715`) | repeated `POST /query` with `(until,before_id)` cursor | `channels list` (`:42`,`:65`), `channels search` (`:133`) |
| `query_all` (`client.rs:724`) | same, unbounded | template roster scan (`channels.rs:449`) |
| `submit_event` (`client.rs:863`) | `POST /events` | every write |
| `upload_file` (`client.rs:1100`) | Blossom upload | `messages send --file` (`messages.rs:510`) |

No file in scope opens a WebSocket, and none calls `count`, `get_authed` or
`get_public` directly (`channels.rs` reaches `get_public` transitively through
`agents::fetch_archived_snapshot`).

#### Exit-code classes produced

Mapping is `error.rs:96-111`. Which classes each group can produce:

| Class | Variant | Raised in scope by |
|-------|---------|-------------------|
| 1 | `Usage` | flag validation everywhere, e.g. `channels.rs:288`, `:296`, `:824`, `:844`, `:975`, `:1015`, `:1028`; `messages.rs:345`, `:547`, `:568`, `:603`, `:733`; `dms.rs:53`; `feed.rs:34`; `social.rs:135`, `:147`, `:170`, `:189`; `notes.rs:52`, `:58`, `:64`, `:258`, `:262`, `:435`, `:496`, `:501`, `:590`, `:648`, `:687`, `:800`; `emoji.rs:178`, `:249`, `:256`, `:262`, `:266` |
| 1 | `NotFound` | `channel_templates.rs:90`, `:113`; `channels.rs:415`; `notes.rs:618`, `:637`, `:643`, `:722` |
| 2 / 3 | `Relay`, `Network` | propagated from `client.rs`, never constructed here |
| 4 | `Other` | build/parse failures, e.g. `messages.rs:66`, `:98`, `:112`, `:543`, `:571`; `channels.rs:311`, `:412`; `reactions.rs:21`, `:52`, `:58`, `:63`; `emoji.rs:84`, `:120`, `:131`, `:167`, `:171` |
| 5 | `Conflict` | **only** `notes.rs:562` (NIP-33 LWW `duplicate:`) |
| 3 | `Auth`, `Key` | not produced in scope (all in `lib.rs:1745-1766`) |

Because only `notes` inspects `accepted`, a relay rejection on any other write
prints `{"event_id":"…","accepted":false,"message":"…"}` and exits **0**
(`client.rs:1425-1431` then `Ok(())` at e.g. `messages.rs:577`,
`channels.rs:860`, `emoji.rs:123`, `dms.rs:107`, `reactions.rs:30`). Scripts that
branch on exit status alone will read a rejected write as success.

#### Public items and doc discipline

`pub` items with no doc comment (undocumented public API, against the AGENTS.md
rule "New public API must have doc comments"):

| Item | Site |
|------|------|
| `cmd_list_channels` | `channels.rs:25` |
| `cmd_get_channel` | `channels.rs:224` |
| `cmd_list_channel_members` | `channels.rs:244` |
| `cmd_create_channel` | `channels.rs:282` |
| `cmd_update_channel` | `channels.rs:832` |
| `cmd_set_channel_topic` / `cmd_set_channel_purpose` | `channels.rs:864` / `:880` |
| `cmd_join_channel` / `cmd_leave_channel` / `cmd_archive_channel` / `cmd_unarchive_channel` / `cmd_delete_channel` | `channels.rs:896`,`:908`,`:920`,`:932`,`:944` |
| `cmd_add_channel_member` / `cmd_remove_channel_member` | `channels.rs:956` / `:987` |
| `cmd_set_canvas` | `channels.rs:1049` |
| `dispatch` (all nine) | `channels.rs:1066`, `:1168`; `messages.rs:754`; `notes.rs:752`; `emoji.rs:311`; `social.rs:211`; `dms.rs:128`; `reactions.rs:127`; `feed.rs:67` |
| `SendMessageParams` / `SendDiffParams` and all their fields | `messages.rs:474-482`, `:581-595` |
| `cmd_get_messages` / `cmd_get_thread` / `cmd_search` | `messages.rs:263`, `:304`, `:340` |
| `cmd_add_reaction` / `cmd_remove_reaction` / `cmd_get_reactions` | `reactions.rs:9`, `:34`, `:80` |
| `cmd_publish_note` / `cmd_set_contact_list` / `cmd_set_list` | `social.rs:22`, `:43`, `:162` |
| `ContactEntry` fields | `social.rs:16-20` |
| `ChannelTemplateRecord` and nested structs' fields | `channel_templates.rs:22-52` |

`notes.rs` is the outlier in the other direction: a 27-line module doc
(`notes.rs:1-31`) plus doc comments on every public function, including a
ratified-semantics block for `build_set_event` (`notes.rs:390-417`).

Also note that `cmd_get` (`notes.rs:612`) and `cmd_set`/`cmd_rm` are `pub` but
depend on validation performed by the dispatcher: `cmd_get` calls
`name.expect("dispatch enforces --name xor --naddr")` (`notes.rs:632`), so any
caller that bypasses `dispatch` panics instead of getting an error. The
mutual-exclusion rules themselves live in `validate_get_args`
(`notes.rs:588-610`), not in clap — `grep -n 'conflicts_with' lib.rs` returns no
match inside the `NotesCmd` block (`lib.rs:1016-1093`), so `buzz notes get
--help` does not advertise them.

#### Test coverage — API surface

Command inventory is pinned in `lib.rs` unit tests: `command_inventory_is_stable`
(`lib.rs:1805`) lists all 21 groups; `subcommand_names_are_stable`
(`lib.rs:1849`) asserts exact subcommand name sets for `messages`, `channels`,
`canvas`, `reactions`, `emoji`, `dms`, `feed`, `social`;
`subcommand_counts_are_stable` (`lib.rs:1978`) pins counts. `notes` appears in
the group list (`lib.rs:1816`) but has **no** name assertion and **no** count
entry — `grep -n '"notes"' lib.rs` shows it only in the group inventory and as
the `social notes` alias, so `NotesCmd`'s four subcommand names are the one
uncovered inventory in scope.

No handler in scope has an end-to-end test: there is no `crates/buzz-cli/tests/`
directory (`ls crates/buzz-cli` shows only `Cargo.toml`, `README.md`,
`TESTING.md`, `src`), and the single `#[tokio::test]` in scope
(`channels.rs:1362`) exercises `cmd_set_add_policy`'s env gate only, which
returns before any network call.


## Module: buzz-cli — repo, agent, memory & moderation commands (`crates/buzz-cli/src/commands`)
### Aspect: API Surface

#### Dispatch entry points

Eleven modules, nine `dispatch` functions plus two directly-called free functions. All are
reached from the single `match cli.command` in `lib.rs:1769-1793`.

| Entry | Signature site | Called from | Accepts `&OutputFormat`? |
|---|---|---|---|
| `agents::dispatch` | `agents.rs:12` | `lib.rs:1771` | no |
| `users::dispatch` | `users.rs:307-311` | `lib.rs:1778` | **yes**, and honors it |
| `workflows::dispatch` | `workflows.rs:214` | `lib.rs:1779` | no |
| `repos::dispatch` | `repos.rs:349` | `lib.rs:1783` | no |
| `patches::dispatch` | `patches.rs:206` | `lib.rs:1784` | no |
| `issues::dispatch` | `issues.rs:147` | `lib.rs:1785` | no |
| `pr::dispatch` | `pr.rs:216` | `lib.rs:1786` | no |
| `upload::dispatch_media` | `upload.rs:17` | `lib.rs:1787` | no |
| `upload::dispatch` | `upload.rs:4` | `lib.rs:1788` | no |
| `mem::dispatch` | `mem.rs:737` | `lib.rs:1789` | no (has its own `--json` on `ls`) |
| `moderation::dispatch` | `moderation.rs:133-137` | `lib.rs:1790` | **yes, but ignores it** — parameter is bound as `_format` at `moderation.rs:136` |
| `pack::cmd_validate` / `cmd_inspect` | `pack.rs:15`, `pack.rs:52` | `lib.rs:1737-1740`, short-circuited before any client is constructed | no |

`--format` is a global flag (`lib.rs:90-92`, default `json`). Of the eleven modules here,
exactly one (`users.rs`) reads it. `moderation.rs` takes it and discards it. The other nine
never see it — so `buzz --format compact repos list`, `… pr list`, `… mem ls`,
`… workflows list` and `… moderation reports` all silently produce full JSON. This
contradicts nothing in `AGENTS.md` (which only documents flag *position*), but the CLI's
own `--format` help text ("Output format… 'compact' (reduced fields)", `lib.rs:90`) promises
behavior these commands do not implement.

#### Subcommand handler table

| Command | Handler | Relay call | Kind / path | Output |
|---|---|---|---|---|
| `agents draft-create` | inline, `agents.rs:16-48` | `publish_ephemeral_event` (WSS) | 24200 observer frame | relay OK object + `request_id`/`action`/`saved:false`/`message` |
| `agents draft-update` | inline, `agents.rs:50-91` | `publish_ephemeral_event` (WSS) | 24200 | same |
| `agents archive` | inline, `agents.rs:93-127` | `query` (kind:0 probe) then `submit_event` | 9035 | `{ok, event_id, action, target}` |
| `agents unarchive` | inline, `agents.rs:129-156` | `query` then `submit_event` | 9036 | `{ok, event_id, action, target}` |
| `agents archived` | `cmd_archived`, `agents.rs:310` | `get_public("/")` + `query` | NIP-11 doc, then 13535 | `{archived:[…]}` |
| `users get` | `cmd_get_users`, `users.rs:12` | `query` | kind:0 | profile array (format-aware) |
| `users set-profile` | `cmd_set_profile`, `users.rs:150` | `query` then `submit_event` | kind:0 | write response |
| `users presence` | `cmd_get_presence`, `users.rs:247` | `query` | 40902 | `[{pubkey,status,updated_at}]` |
| `users set-presence` | `cmd_set_presence`, `users.rs:298` | `publish_ephemeral_event` (WSS) | 20001 | write response |
| `workflows list` | `cmd_list_workflows`, `workflows.rs:13` | `query` | 30620 by `#h` | normalized array |
| `workflows get` | `cmd_get_workflow`, `workflows.rs:38` | `query` | 30620 by `#d` | object or `null` |
| `workflows create` | `cmd_create_workflow`, `workflows.rs:98` | `submit_event` | 30620 | write response + `workflow_id` |
| `workflows update` | `cmd_update_workflow`, `workflows.rs:119` | `submit_event` | 30620 | write response |
| `workflows delete` | `cmd_delete_workflow`, `workflows.rs:139` | `submit_event` | kind:5 with `a` tag | write response |
| `workflows trigger` | `cmd_trigger_workflow`, `workflows.rs:156` | `submit_event` | 46020 | write response |
| `workflows runs` | `cmd_get_workflow_runs`, `workflows.rs:66` | `query` | 46001/46002/46003 | normalized array (see below) |
| `workflows approve` | `cmd_approve_step`, `workflows.rs:193` | `submit_event` | 46030 / 46031 | write response |
| `repos create` | `cmd_create_repo`, `repos.rs:202` | `submit_event` | 30617 | raw relay body |
| `repos get` | `cmd_get_repo`, `repos.rs:232` | `query` | 30617 | raw relay array |
| `repos list` | `cmd_list_repos`, `repos.rs:256` | `query` | 30617 | raw relay array |
| `repos protect list` | `cmd_protect_list`, `repos.rs:295` | `query` | 30617 (own) | derived protection JSON |
| `repos protect set` | `cmd_protect_set`, `repos.rs:301` | `query` then `submit_event` | 30617 | normalized write response |
| `repos protect remove` | `cmd_protect_remove`, `repos.rs:327` | `query` then `submit_event` | 30617 | normalized write response |
| `patches send` | `cmd_send_patch`, `patches.rs:9` | `submit_event` | 1617 | raw relay body |
| `patches get` | `cmd_get_patch`, `patches.rs:73` | `query` | 1617 by `ids` | raw relay array |
| `patches list` | `cmd_list_patches`, `patches.rs:84` | `query` | 1617 by `#a` | raw relay array |
| `patches status` | `cmd_patch_status`, `patches.rs:114` | `submit_event` | 1630-1633 | raw relay body |
| `pr open` | `cmd_open_pr`, `pr.rs:20` | `submit_event` | 1618 | raw relay body |
| `pr update` | `cmd_update_pr`, `pr.rs:66` | `submit_event` | 1619 | raw relay body |
| `pr get` | `cmd_get_pr`, `pr.rs:107` | `query` | 1618 by `ids` | raw relay array |
| `pr list` | `cmd_list_prs`, `pr.rs:118` | `query` | 1618 by `#a` | raw relay array |
| `pr status` | `cmd_pr_status`, `pr.rs:152` | `submit_event` | 1630-1633 | raw relay body |
| `issues create` | `cmd_create_issue`, `issues.rs:6` | `submit_event` | 1621 | raw relay body |
| `issues get` | `cmd_get_issue`, `issues.rs:36` | `query` | 1621 by `ids` | raw relay array |
| `issues list` | `cmd_list_issues`, `issues.rs:47` | `query` | 1621 by `#a` | raw relay array |
| `issues status` | `cmd_issue_status`, `issues.rs:81` | `submit_event` | 1630-1633 | raw relay body |
| `moderation ban` | `cmd_ban`, `moderation.rs:34` | `submit_event` (moderation retry policy) | 9040 | normalized write response |
| `moderation unban` | `cmd_unban`, `moderation.rs:51` | `submit_event` | 9041 | normalized write response |
| `moderation timeout` | `cmd_timeout`, `moderation.rs:61` | `submit_event` | 9042 | normalized write response |
| `moderation untimeout` | `cmd_untimeout`, `moderation.rs:79` | `submit_event` | 9043 | normalized write response |
| `moderation resolve` | `cmd_resolve`, `moderation.rs:89` | `submit_event` | 9044 | normalized write response |
| `moderation reports` | `cmd_reports`, `moderation.rs:105` | `get_authed` | `GET /moderation/reports?limit=…[&status=…]` | raw relay array |
| `moderation restricted` | `cmd_restricted`, `moderation.rs:119` | `get_authed` | `GET /moderation/restricted` | raw relay array |
| `moderation audit` | `cmd_audit`, `moderation.rs:125` | `get_authed` | `GET /moderation/audit?limit=…` | raw relay array |
| `mem ls` | `cmd_ls`, `mem.rs:189` | `query` | 30174 | TSV or `--json` array |
| `mem get` | `cmd_get`, `mem.rs:277` | `query` | 30174 | raw value bytes |
| `mem hash` | `cmd_hash`, `mem.rs:508` | `query` | 30174 | 64 hex + `\n` |
| `mem set` | `cmd_set`, `mem.rs:314` | `query` (head) then `submit_event` | 30174 | stderr line only |
| `mem patch` | `cmd_patch`, `mem.rs:538` | `query` (head) then `submit_event` | 30174 | stderr diff + digest |
| `mem rm` | `cmd_rm`, `mem.rs:706` | `query` (head) then `submit_event` | 30174 tombstone | stderr line only |
| `upload file` | inline, `upload.rs:6-13` | `upload_file` | `PUT /upload`, falling back to `PUT /media/upload` | pretty-printed `BlobDescriptor` |
| `media get` | inline, `upload.rs:19-33` | `download_media` | `GET /media/<sha256[.ext]>` | raw bytes |
| `pack validate` | `cmd_validate`, `pack.rs:15` | none — local | — | plain text |
| `pack inspect` | `cmd_inspect`, `pack.rs:52` | none — local | — | plain text |

#### HTTP surface reached

| Path | Method | Reached by | Auth |
|---|---|---|---|
| `/query` | POST | every read command in this group, via `BuzzClient::query` (`client.rs:767-771`) | NIP-98 + optional `x-auth-tag` |
| `/events` | POST | every stored-event write, via `submit_event` (`client.rs:863-870`) | NIP-98 + optional `x-auth-tag` |
| `/` (NIP-11 relay info) | GET | `agents archived`, via `get_public` (`client.rs:753-765`) | **none** — deliberately unauthenticated |
| `/moderation/reports` | GET | `moderation reports` (`moderation.rs:107-112`) | NIP-98 via `get_authed`, plus relay-side mod authz |
| `/moderation/restricted` | GET | `moderation restricted` (`moderation.rs:120`) | same |
| `/moderation/audit` | GET | `moderation audit` (`moderation.rs:126-127`) | same |
| `/upload` then `/media/upload` | PUT | `upload file` (`client.rs:1139`, `client.rs:1195`) | Blossom BUD-02 auth |
| `/media/<sha256[.ext]>` | GET | `media get` (`client.rs:1230`) | Blossom BUD-01 `t=get` |
| WSS `/` | EVENT | `agents draft-*`, `users set-presence` (`client.rs:1073-1096`) | NIP-42 |

`AGENTS.md` names `POST /events`, `POST /query`, `POST /count`, `/hooks/{id}`, Blossom
media, git smart HTTP, git policy hooks, NIP-11/NIP-05, and health probes as the whole HTTP
surface. The `/moderation/*` triple is **not** in that list, yet three commands here depend
on it. The `moderation.rs` module doc (`moderation.rs:9-15`) explains and defends the
choice; `AGENTS.md` has not been updated to match. Routes confirmed at
`crates/buzz-relay/src/router.rs:113-116`. Conversely, `POST /count` is defined
(`client.rs:803`) but marked `#[allow(dead_code)]` and is called by no command in this
group — `grep -n 'client.count\|\.count(' ` across the eleven files returns zero matches.

#### Undocumented / semi-public items

- `agents::fetch_archived_snapshot` is `pub(crate)` (`agents.rs:270`) and is the one item in
  this group consumed by a *sibling* module: `channels.rs:11` imports it for
  `--template` roster resolution. Its doc comment (`agents.rs:255-269`) is the only
  description of the fail-open-vs-fail-closed split between the two callers.
- `patches::parse_status` is `pub(crate)` (`patches.rs:194`) and is called cross-module by
  `pr.rs:169` and `issues.rs:83`. It is the single source of the status-word vocabulary.
- `repos.rs` exposes `cmd_create_repo`, `cmd_get_repo`, `cmd_list_repos` as `pub`
  (`repos.rs:202`, `:232`, `:256`) while the three `protect` handlers are private
  (`repos.rs:295`, `:301`, `:327`) — inconsistent visibility with no external caller for
  either set beyond the local `dispatch`.
- `users.rs` and `mem.rs` mark all their command functions `pub` even though only
  `dispatch` calls them; `mem.rs`'s are documented, `users.rs`'s partly so
  (`cmd_get_users` at `users.rs:7-11` yes, `cmd_set_profile` at `users.rs:150` no doc).

#### Output contract deviations within the group

Three commands break the "JSON out" contract stated in `crates/buzz-cli/README.md:3`
("JSON in, JSON out") and `README.md:159`:

1. `mem get` writes raw value bytes with no newline (`mem.rs:295-303`) — documented as
   intentional at `mem.rs:296` so it round-trips with `mem set <slug> -`.
2. `mem ls` without `--json` emits TSV (`mem.rs:268-270`) and an empty listing prints
   `(no memories besides core)` to **stderr** (`mem.rs:266`) while stdout stays empty.
3. `pack validate` / `pack inspect` emit only human-readable text (`pack.rs:26-45`,
   `pack.rs:66-149`) with no JSON mode at all.

And one formatting outlier: `upload file` is the only command in the whole group that
pretty-prints (`upload.rs:9-10`); everything else emits single-line JSON.

#### Exit-code classes produced

Mapping is `error::exit_code` (`error.rs:89-107`); the categories printed on stderr come
from `error::print_error` (`error.rs:112-137`).

| Code | `CliError` variant | Produced in this group by |
|---|---|---|
| 0 | — | every success path |
| 1 | `Usage` | all `validate_hex64` / `validate_repo_id` / `validate_uuid` failures; mutual-exclusion errors (`users.rs:17-19`, `mem.rs:57-60`, `mem.rs:551-555`, `pr.rs:9-11`); `parse_status` (`patches.rs:202-204`); `parse_committer` (`patches.rs:66-68`); `--inputs` not-an-object (`workflows.rs:165-169`); empty-stdin guards (`mem.rs:331-338`, `mem.rs:586-591`); `pack validate` failure (`pack.rs:40`); `pack` path errors (`pack.rs:17-22`, `pack.rs:54-59`); patch-application failures (`mem.rs:611-628`); size-cap breaches (`mem.rs:326-329`, `mem.rs:630-636`) |
| 1 | `NotFound` | `mem get`/`hash`/`patch` on absent or tombstoned slug (`mem.rs:294`, `mem.rs:296`, `mem.rs:487-491`); `repos protect *` when the repo is not the caller's (`repos.rs:290-292`); `repos protect remove` for an absent rule (`repos.rs:334-338`) |
| 2 | `Relay` (non-401/403), `Network`, `DeliveryUnknown` | any `/query`, `/events`, `/moderation/*` failure surfaced by `client.rs` |
| 3 | `Auth`, `Key`, `Relay{401,403}` | `require_owner` when `BUZZ_AUTH_TAG` is absent (`agents.rs:161-165`); relay 403 on any command |
| 4 | `Other` | build/sign failures (`mem.rs:356`, `mem.rs:681`, `mem.rs:725`); relay-response parse failures (`mem.rs:95-96`, `agents.rs:36`, `repos.rs:174-175`); all 13535 trust failures (`agents.rs:322-372`); `repos` invalid-existing-rules refusal (`repos.rs:126-131`); `pack inspect` resolve failure (`pack.rs:62-63`) |
| 5 | `Conflict` | **`mem set`/`patch`/`rm`** via `submit_engram` (`mem.rs:100-104`); **`mem patch`** base-hash mismatch (`mem.rs:594-598`); **`repos protect set`/`remove`** via `validate_write_response` (`repos.rs:154-158`) |

`AGENTS.md` names `mem set`/`mem rm` as "the documented producers" of exit 5. That is
incomplete in two directions: `mem patch` produces it on two distinct paths (base-hash
mismatch and relay duplicate), and `repos protect set`/`remove` produce it too — the only
non-`mem` producers in the codebase, added at `repos.rs:154-158` with a test
(`duplicate_write_response_is_a_conflict`, `repos.rs:628`). No other command in this group
maps anything to exit 5: `grep -n 'CliError::Conflict' ` across the eleven files matches
only `mem.rs:101`, `mem.rs:595`, and `repos.rs:155`.

#### Test coverage of the API surface

Covered: `read_optional_body`'s mutual exclusion and empty default (`pr.rs:334`, `pr.rs:339`);
`parse_status` word set and rejection (`patches.rs:304`, `patches.rs:319`); `parse_committer`
arity (`patches.rs:284`, `patches.rs:298`); the write-response normalization/conflict split
(`repos.rs:628`, `repos.rs:640`); `resolve_reader`'s three-way perspective resolution and
both mutual-exclusion rules (`mem.rs:793`-`mem.rs:845`).

Not covered anywhere: every `dispatch` arm (no test constructs a `*Cmd` and dispatches);
`--format` propagation; the group's exit-code mapping end to end; the `/moderation/*` URL
construction; `resolve_expiry`; `cmd_get_workflow`'s `null` branch. There is no
`crates/buzz-cli/tests/` directory (`ls crates/buzz-cli/tests` → "No such file or
directory") and no `#[tokio::test]` in any of the eleven files
(`grep -rn 'tokio::test' crates/buzz-cli/src/commands/` returns zero matches).

