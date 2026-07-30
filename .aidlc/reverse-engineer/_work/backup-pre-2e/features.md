<!-- Analyzed: 2026-07-25T01:12:08Z | Scope: full project -->
# Feature Inventory

> Status: initialized in Phase 1. Feature inventory, user journeys, and completeness
> assessment are populated per-module during Phase 2 and consolidated in Phase 3.

## Summary

Scan-time signal: 28 desktop feature packages, 11 mobile feature packages, and 5 declared
preview features (`workflows`, `projects`, `pulse`, `forum`, `agentManagedProfiles`).

Batch 2a covers the foundation layer, which **enables** capabilities rather than exposing
user-facing surfaces: 17 capabilities in `buzz-core` (13 full, 4 partial), 25 capability
areas in `buzz-sdk`, 19 in `buzz-persona`, and a narrow connect→auth→publish→read flow in
`buzz-ws-client`.

**Zero TODO/FIXME/HACK/XXX markers exist across all four crates.** Deferred work is
expressed as prose doc comments instead, which makes it invisible to marker-based debt
tooling. The notable stubs found this way:

| Stubbed / absent | Where | Statement in code |
|---|---|---|
| Persona lifecycle hooks | `crates/buzz-persona/src/resolve.rs:57` | "parsed, not executed — reserved for future use, not yet wired" |
| Persona skill scoping | `crates/buzz-persona/src/pack.rs:249` | `resolve_skills` fully implemented + tested but **never called** by any production path |
| MCP `${VAR}` interpolation | `crates/buzz-persona/src/resolve.rs:274` | env values pass through as literals |
| `engines.buzz` version gate | `crates/buzz-persona/src/manifest.rs:39-40` | parsed, never evaluated |
| Full NIP-10 threading for kind 1 | `crates/buzz-sdk/src/builders.rs:732-737` | "Full NIP-10 threading … is deferred" |
| Typed `REQ`/`CLOSE`/`COUNT`, reconnect/retry | `crates/buzz-ws-client` | absent — callers hand-roll via `send_raw` |
| Filter `limit` + NIP-50 `search` | `crates/buzz-core/src/filter.rs:35-104` | not implemented in the matcher |

Batch 2b covers the service layer. The same pattern repeats and intensifies: **zero
TODO/FIXME/HACK markers in six of seven crates** (only `buzz-workflow` has one, WF-08),
while a substantial amount of functionality is present-but-unreachable. Marker-based
tooling sees none of it.

Capabilities that exist in code but cannot be reached in production:

| Non-functional / unreachable | Where | Nature |
|---|---|---|
| Workflow `send_dm`, `set_channel_topic` | `crates/buzz-workflow/src/executor.rs` | Return `NotImplemented` (documented) |
| Workflow `add_reaction` | `crates/buzz-workflow/src/executor.rs` | **Undocumented failure** — POSTs to `/api/messages/{id}/reactions`, which the relay never registers (`crates/buzz-relay/src/router.rs:39-125`) |
| Workflow approval gates | `crates/buzz-workflow/src/executor.rs:663` | Token generated, never persisted; runs marked `Failed`; **nothing ever writes `WaitingApproval`**, so the relay's resume path is unreachable and `Db::create_approval` has no caller |
| `buzz-auth::ChannelAccessChecker` | `crates/buzz-auth` | **Zero implementors repo-wide**; the trait doc's claim that `buzz-db` implements it is false |
| `Scope::all_non_admin()` (14 scopes) | `crates/buzz-auth` | Never called — NIP-42 grants `all_known()` (16) |
| `check_ip_connection` / `LimitType::IpConnections` | `crates/buzz-pubsub/src/rate_limiter.rs:112-120` | Fully implemented, no production caller — and it is the one control that would cover currently-unmetered pre-auth traffic |
| `verify_chain` / `get_entries` (audit) | `crates/buzz-audit` | No production caller — nothing verifies the hash chain in normal operation |
| `verify_blossom_get_auth` | `crates/buzz-media` | Never called from the crate; the relay gate defaults to off |
| `PubSubConfig::with_unsubscribe_debounce`, `topic_refcount` | `crates/buzz-pubsub/src/lib.rs:93`, `:248` | Knob and metric hook, both unwired |
| Media GC / orphan cleanup / storage quota | `crates/buzz-media` | Absent entirely |
| **Typing indicators** | `crates/buzz-pubsub` | **Advertised in four places, implemented nowhere.** `Cargo.toml:8`, an orphaned doc comment at `src/lib.rs:43`, `AGENTS.md`, and `ARCHITECTURE.md:82`, `:432`, `:434`, `:777`, `:801` — including a concrete ZADD/ZREMRANGEBYSCORE/EXPIRE design at `:452-456`. Repo-wide: zero sorted-set calls, zero `buzz:typing` keys. Real behaviour is kind-20002 ephemeral events (`crates/buzz-core/src/kind.rs:407`) produced by `buzz-acp` and fanned out generically |

Capabilities confirmed **working** in 2b, contrary to documentation that says otherwise:

| Capability | Doc claim | Reality |
|---|---|---|
| Rate limiting | `ARCHITECTURE.md:823` (§9 #2), `:390`, `:460` — "not enforced, test stub only" | Enforced before work on WS `EVENT`/`REQ`/`COUNT` and the HTTP bridge (`crates/buzz-pubsub/src/rate_limiter.rs:99`, wired `crates/buzz-relay/src/state.rs:712`) |
| Workflow `execute_from_step` | "exists for future use" | **3 live callers**; `execute_run` is the one with none |
| Media hash verification | implied absent | Verified server-side on upload (not on read) |

Also undocumented-but-present: a **5th** workflow trigger, `diff_posted`, and an **11th**
audit action, `MediaUploaded`.

User journeys and completeness against the README's "Works today / Being wired up"
table are assembled in Phase 3.

### Batch 2c features (relay, mesh, conformance)

Three feature families land in this batch, and in each one a headline capability is present in
code but not reachable in a deployed configuration.

- **Git hosting is real and broad** — smart HTTP, policy hooks, CAS publish, ref protection,
  and a hydrate path — but has no read authorization
  (`crates/buzz-relay/src/api/git/transport.rs:539-594`, `:786-827`), and pointer creation is
  ungated (a zero-ref-command push creates pointers, bypassing `git_repo_names` and quota).
- **Huddle audio and the tunnel** are hosted directly by the relay rather than a separate
  service (`crates/buzz-relay/src/` audio group).
- **`workflow_sink` implements 1 of the 7 workflow actions** the engine defines.
- **The mesh subsystem is complete enough to run and never runs.** `buzz-relay-mesh` is absent
  from `ARCHITECTURE.md` and `AGENTS.md`, and absent from `just test-unit`, so all 32 of its
  tests compile but never execute. `MeshRuntime::shutdown()` is never called.
- **The 5 `crates/buzz-relay/examples/mesh_*.rs` do not exercise `buzz-relay-mesh`** — they are
  MeshLLM (`mesh_llm_sdk`) smoke tests pulled in via git dev-dependencies
  (`crates/buzz-relay/Cargo.toml:84-85`).
- **The conformance gate is built but unassembled.** Emission is live at the ingest and REQ
  seams; `EmitGuard` is armed at exactly one seam
  (`crates/buzz-relay/src/handlers/ingest.rs:1408-1412`); row-community projection runs on both
  read lanes. But `check_trace` (`crates/buzz-conformance/src/checker.rs:74`) has zero
  production callers and `JsonlTracer` — the only persisting tracer
  (`crates/buzz-relay/src/conformance/tracers.rs:30-72`) — is never instantiated. Production
  binds `NoopTracer` (`crates/buzz-relay/src/state.rs:798`), the sole assignment to that field.
  So every `TraceStep` is constructed and discarded, and the relay→JSONL→checker pipeline the
  crate exists to run is not wired anywhere.

### Batch 2d features (agent surface: buzz-acp, buzz-agent, buzz-dev-mcp, buzz-cli)

2d is the batch that makes Buzz an agent platform rather than a chat relay. Four capability
clusters: the **ACP harness** (`buzz-acp`) that turns Buzz events into agent turns, a **minimal
reference agent** (`buzz-agent`) with four LLM providers, an **unrestricted developer toolbelt**
(`buzz-dev-mcp`), and the **agent-first CLI** (`buzz-cli`) that is how agents actually talk to the
relay.

| ID | Capability | Limits and notable gaps |
|---|---|---|
| FEAT-2d-1 | **Agent pooling with idle reuse, lazy start, and mid-turn model switching** (`buzz-acp` `pool.rs`) | The pooling logic itself has **zero tests** — 11 functions including `try_claim`, `return_agent`, `any_idle` and `switch_idle_agent_model` have no test references, and `run_prompt_task` (948 lines, 20 exit points) is untested. `tests/pool_lifecycle_state.rs` is a 4-line `#[path]` shim adding no coverage. |
| FEAT-2d-2 | **Event batching, dedupe and mid-turn steering** — multiple Buzz events arriving during a turn are merged into one prompt, or queued, or dropped (`queue.rs`) | `DedupMode::Drop` loses batches on three paths including steers (`pool.rs:2971-2977`, `lib.rs:2196-2199`); `queue.rs:11-14` documents "Drop (default)" while the CLI default is `queue` (`config.rs:344`). `cancelled_batches` is unbounded (`queue.rs:543-546`). |
| FEAT-2d-3 | **Observer feed** — every ACP frame republished to the owner as a NIP-44-encrypted kind-24200 event (`lib.rs:790-833`) | The only transform is length elision at 3,000 bytes (`lib.rs:632`), which preserves a 63-char nsec — see SEC-43. `observer.rs` has zero tests. |
| FEAT-2d-4 | **Typing indicators** driven from a 3 s tick (`lib.rs:2332-2341` → `relay.rs:842-870`) | Implemented as kind-20002 ephemeral events. The ZADD design at `ARCHITECTURE.md:452-458` exists nowhere, and `KIND_TYPING_INDICATOR` (`kind.rs:331`) appears in neither `buzz-relay/src` nor `buzz-pubsub/src`. |
| FEAT-2d-5 | **Setup mode** — a first-run flow that diverts startup instead of beginning the pool (`setup_mode.rs`, gated by `BUZZ_ACP_SETUP_PAYLOAD`) | Silently ignores reminder events: uses only `[STREAM_MESSAGE, WORKFLOW_APPROVAL_REQUESTED]` (`setup_mode.rs:526`) where `config.rs:1161-1165` uses a third kind. When the payload is set the pool never starts (`lib.rs:1298-1303`). |
| FEAT-2d-6 | **Remote shutdown** via a `!shutdown` chat message (`lib.rs:2033-2059`) | Plaintext, replayable, no freshness window, no signature re-verification. The ±300 s guard exists only on the encrypted observer control path (`lib.rs:860-869`). |
| FEAT-2d-7 | **ACP JSON-RPC 2.0 agent over stdio** with an agentic tool loop (`buzz-agent`) | Non-streaming by design — one request per round, text arrives as a single `agent_message_chunk`. One frame ≤ `max_line_bytes` (4 MiB default); an oversize frame terminates the read loop. No session load/resume, no image/audio prompts, no HTTP/SSE MCP transport — all three advertised `false` and all three actually enforced. |
| FEAT-2d-8 | **Four LLM providers across three wire dialects and five endpoint shapes**: Anthropic Messages, OpenAI Chat Completions, OpenAI Responses, legacy Databricks serving, and DatabricksV2's three-way gateway routing (`llm.rs:67-160`, `:685-704`) | No streaming, no pre-send token counting, no provider failover, no multi-key rotation, and **no total wall-clock request timeout** (see DEBT-69). Cache-token accounting is summed for Anthropic and Chat but not for Responses (`llm.rs:797`), with no comment explaining the asymmetry. |
| FEAT-2d-9 | **A per-model reasoning-effort capability system** — seven levels, doc-verified capability tables per model family, boundary-safe matchers so `gpt-5.1` cannot match `gpt-5.10` (`config.rs:239-311`, `:333-401`) | Two contradictory classifiers coexist in the same request path (DEBT-66). `BUZZ_AGENT_THINKING_EFFORT`, the only knob that reaches it, is documented nowhere (CFG-2d-6). |
| FEAT-2d-10 | **Automatic context handoff** — the agent summarizes its own history and starts a fresh context when the window fills (`handoff.rs:31-107`) | ≤ `max_handoffs` per session (default 10); the summary is capped at 8,192 output tokens; after the cap, plain truncation takes over. After a handoff **only** the summary and the current prompt survive. |
| FEAT-2d-11 | **Interruptible turns with mid-turn steering that does not cancel** (`lib.rs:554-621`, `agent.rs:265-280`) | Applied at round boundaries only; a steer arriving after the final round is silently never delivered. The steer channel is unbounded and steers bypass the 1 MiB prompt cap. |
| FEAT-2d-12 | **MCP lifecycle hooks** (`_Stop`, `_PostCompact`) letting an external server object to a premature end-of-turn (`agent.rs:224-236`, `handoff.rs:70-77`) | Off unless `MCP_HOOK_SERVERS` is set — a variable absent from the crate README's own config table. Advisory and fail-open with a per-prompt objection budget (default 3). `_PostCompact` output is **not** JSON-encoded, contrary to the spec doc (DEBT-65). |
| FEAT-2d-13 | **Hints and skills discovery** — `AGENTS.md` files from `$HOME` and every directory between the git root and `cwd`, plus `SKILL.md` packs, appended to the system prompt (`hints.rs:41-66`, `:233-247`) | Entirely undocumented (zero matches for `load_skill`/`skills` in the crate README). Skill metadata enters at **system-prompt** trust with only `trim()` applied and no length cap (SEC-53). Discovery follows symlinks deliberately, with a test locking that in. No logging at all in `hints.rs`, so a shadowed or skipped skill is invisible. |
| FEAT-2d-14 | **Interactive Databricks OAuth PKCE login** — `buzz-agent auth databricks`, browser flow with a loopback callback and a token cache (`lib.rs:129-152`, `auth.rs:527-630`) | PKCE itself is correct (48-byte verifier, S256, `state` compared, loopback-only bind, 60 s window). The cache is world-readable (SEC-55) and endpoint URLs are unvalidated (SEC-56). `browser_pkce_flow` has no test. |
| FEAT-2d-15 | **Databricks model-catalog discovery** advertised through `session/new` (`catalog.rs`, `lib.rs:443-474`) | Real discovery only for the two Databricks providers; other providers get a one-entry list echoing `cfg.model`. Paginates to 20 pages × 100 then silently truncates with no log. No HTTP timeout, and it is awaited **inline inside `session/new`**, so a hung host stalls session creation. The whole HTTP layer is untested. |
| FEAT-2d-16 | **A developer toolbelt exposed to the model as MCP tools**: shell, file read/edit, `str_replace`, `view_image`, `rg`, `tree` (`buzz-dev-mcp` `lib.rs:40-125`) | Unrestricted by design and self-documented as such — absolute paths accepted, `..` never checked, symlinks followed, no shell allow/deny list (SEC-46). `rg` silently degrades to literal substring matching with a hand-rolled glob, and is permanently degraded on Windows (DEBT-71). |
| FEAT-2d-17 | **A shim that gives the model `buzz`, `git-sign-nostr` and `git-credential-nostr` on `PATH`** via `argv[0]` multicall dispatch into a `0700` tempdir (`shim.rs:31-49`, `lib.rs:144-171`) | This is how relay access reaches tools without any relay operation being an MCP tool — the `AGENTS.md:147` boundary holds. `shim.rs` has zero tests, so its nsec-removal invariant is unverified. |
| FEAT-2d-18 | **`view_image` with relay-media credential scoping, magic-byte sniffing and a decompression-bomb guard** (`view_image.rs:236-247`, `:394-421`, `:549-558`) | The best-hardened code in the batch — but `fetch_url` itself is untested. |
| FEAT-2d-19 | **21 CLI command groups / ~100 subcommands from one binary** (`buzz-cli` `lib.rs:175-240`) | No `--version`, no shell completions, no man page, no config file, no verbosity/quiet/colour controls, no logging framework, no interactive confirmations on destructive commands, and no proxy/CA/client-cert/TLS-pinning options. One relay, one identity, one command per process. |
| FEAT-2d-20 | **Agent-friendly reduced output** via `--format compact` | Reaches 5 of 21 groups and is silently ignored by the other 16, with three mutually inconsistent compact shapes among the commands that do honour it (CFG-2d-13). |
| FEAT-2d-21 | **Retry with failure classification an agent can act on** — 3 attempts with jittered backoff, `retry in Ns` hint honoured to a 30 s cap, body-transfer failures inside the retry boundary, and a `retryable` flag plus a `DeliveryUnknown` category so a caller knows whether a failed write is safe to re-run (`client.rs:638-681`, `error.rs:74-88`) | The best-engineered feature in the batch, with 12 behavioural integration tests over live servers. Attempts, jitter bases and the hint cap are compile-time constants; only the two timeouts are env-tunable. |
| FEAT-2d-22 | **Automatic pagination** — `(until, before_id)` composite cursor following at 500/page with no flags required (`client.rs:683-727`) | `query_all` has no page or memory cap; a broad filter accumulates every matching event in RAM. |
| FEAT-2d-23 | **Delegated action on an owner's behalf** via a locally-verified NIP-OA `--auth-tag`, injected into every event and sent as `x-auth-tag` (`lib.rs:1752-1767`, `client.rs:588-621`) | The `conditions` slot is never inspected anywhere in the CLI, so an agent cannot tell locally whether an action is within its delegation (SEC-57). |
| FEAT-2d-24 | **Owner-reviewed agent creation** — `agents draft-create`/`draft-update` publish an encrypted proposal and change nothing until the owner saves it (`agents.rs:26-88`) | The only human-in-the-loop gate in the CLI, and it is structural rather than advisory. |
| FEAT-2d-25 | **Encrypted agent memory (`buzz mem`)** — get/set/patch/rm/ls/hash over kind-30174 engrams with HMAC-derived addressing and strict-position patch application (`mem.rs`) | The only CLI family that bounds stdin during the read, and the only one with a genuinely discriminating patch test. `mem set/patch/rm` print **nothing** on stdout. `mem rm core` is refused. `--patch-file` skips the size bound (SEC-54). |
| FEAT-2d-26 | **NIP-34 git collaboration from the CLI** — repo announcements with branch-protection rules, patches, pull requests, issues and status transitions (`repos.rs`, `patches.rs`, `pr.rs`, `issues.rs`) | The write end of a contract the relay's git policy hook enforces; the two agree only because both call into `buzz_core::git_perms`. `git-sign-nostr` and `git-credential-nostr` are peer tools invoked by `git`, not by the CLI. Eight of these read paths print the relay body unnormalized. |
| FEAT-2d-27 | **Persona-pack validation offline** — `buzz pack validate` / `inspect` are the only commands that need no key and no relay (`lib.rs:1737-1743`, `pack.rs`) | Prints human-readable text only. Deliberately prints the MCP-server **count** rather than the servers, which keeps declared credentials out of the output — the right call. Neither command has a test. |
| FEAT-2d-28 | **Workflow definition, trigger and approval from the CLI** (`workflows.rs`) | `workflows create`/`update` publish YAML with **no client-side validation** — `buzz-workflow` is not even a dependency — so a malformed `if:` expression fails only relay-side. `workflows runs` ships knowingly returning `[]` (BR-175). `workflows approve --approved` defaults to granting. |
| FEAT-2d-29 | **Moderation queue, audit log and restricted-member reads** (`moderation.rs`) | Reached over three HTTP endpoints the house docs do not describe (API-166). Prints private ban reasons verbatim to stdout, which in an agent harness means into the model's context (SEC-58). |

## Features

| Feature | Surfaces | Status | Modules | Source |
|---|---|---|---|---|
| _pending_ | | | | |

## Desktop Feature Packages (scan-level)

`agent-memory`, `agents`, `channel-templates`, `channels`, `chat`, `communities`,
`community-members`, `custom-emoji`, `forum`, `home`, `huddle`, `identity-archive`,
`local-archive`, `mesh-compute`, `messages`, `moderation`, `notifications`, `onboarding`,
`presence`, `profile`, `projects`, `pulse`, `reminders`, `search`, `settings`, `sidebar`,
`user-status`, `workflows`

## Mobile Feature Packages (scan-level)

`activity`, `channels`, `custom_emoji`, `forum`, `home`, `invites`, `pairing`, `profile`,
`pulse`, `search`, `settings`

## User Journeys

_Pending Phase 2._

## Completeness Assessment

_Pending Phase 2/3 — cross-check against README's "Works today / Being wired up /
Pending" table and ARCHITECTURE.md §9 Known Limitations._

---

# Phase 2 — Module Findings

## Module: buzz-core (`crates/buzz-core`)

### Aspect: Features

`buzz-core` is a foundation library — it enables capabilities in other crates rather than exposing user-facing surfaces itself. Completeness below is judged only against what this crate's own code does; a capability marked "partial" means part of the contract described in its own doc comments is not implemented here.

---

### Capability inventory

| # | Capability | What this crate provides | Completeness | Evidence |
|---|-----------|--------------------------|--------------|----------|
| F-1 | Nostr event verification | ID-hash recomputation + Schnorr signature check, with a typed error carrying computed vs claimed ID | **full** | `src/verification.rs:11-32`, `src/error.rs:2-20` |
| F-2 | Relay event metadata wrapper | `StoredEvent` binding an event to receive time, channel scope, and a private verified flag | **full** | `src/event.rs:11-48` |
| F-3 | Event-kind registry | 130 kind constants, 4 range bounds, 4 kind-set slices, `ALL_KINDS` (127), 9 classification predicates, 2 kind extractors, 25 compile-time range assertions | **full** | `src/kind.rs:9-820` |
| F-4 | NIP-01 subscription filter matching | kinds/authors/since/until/ids(prefix)/generic-tag matching with OR across filters, AND within a filter, plus the `#h` channel fallback | **full** for the fields it implements; **partial** vs NIP-01 overall — `limit` and NIP-50 `search` are not handled here (no reference to either in `filter.rs`) | `src/filter.rs:10-104` |
| F-5 | Per-viewer result-level read authorization | `reader_authorized_for_event` gate for DM-visibility and agent-turn-metric kinds | **full** (as a predicate; enforcement at delivery sites lives in `buzz-relay`, per doc `filter.rs:19-22`) | `src/filter.rs:23-33` |
| F-6 | SSRF address classification | `is_private_ip` covering 7 IPv4 and 9 IPv6 categories plus 3 recursive embedded-IPv4 paths (was "6 IPv6 categories plus IPv4-mapped recursion" — `c26bf594` added NAT64 local-use, Teredo and 6to4 to the direct checks, and NAT64 well-known + IPv4-translated + IPv4-compatible to the recursive paths) | **full** for the enumerated ranges; see security doc for ranges *not* covered | `src/network.rs:46-95` |
| F-7 | Multi-tenant community identity | `CommunityId` newtype, `TenantContext`, host normalization, relay-URL authority extraction | **partial by design** — the type system removes accidental client-controlled tenants; the deliberate path is closed by lint + review, stated explicitly at `tenant.rs:23-30` | `src/tenant.rs:37-172` |
| F-8 | Relay runtime identity canonicalization | `normalize_relay_url` folding loopback spellings, case, default ports, root path | **full** | `src/relay.rs:37-78` |
| F-9 | Channel + role vocabulary | visibility/type/role enums with string round-trip and a numeric permission ladder | **full** | `src/channel.rs:22-181` |
| F-10 | Channel-name canonicalization | strips leading `#`/whitespace, trims trailing space | **full** | `src/channel.rs:15-19` |
| F-11 | Presence vocabulary | curated `online`/`away`/`offline` enum for structured APIs | **full** | `src/presence.rs:11-35` |
| F-12 | Agent observer frame crypto | NIP-44 v2 encrypt/decrypt of arbitrary serde payloads with size envelopes and zeroization; frame/tag name constants | **full** | `src/observer.rs:13-110` |
| F-13 | NIP-AM agent turn metrics | payload type (`TokenCounts`, `StopReason`, `AgentTurnMetricPayload`), numeric validation, symmetric encrypt/decrypt | **partial** — `session_id`/`turn_seq` requirements and monotonicity documented at `agent_turn_metric.rs:97-108` are not enforced in code; only `cost_usd` is validated (`:147-169`) | `src/agent_turn_metric.rs:22-191` |
| F-14 | NIP-AE agent engrams | slug grammar, shorthand normalization, conversation key, d-tag HMAC, byte-exact body codec, strict JSON parsing, `[[ref]]` extraction, event build, envelope validation + decrypt, LWW head selection, monotonic clock rule, `Listing` wire type | **full** for the primitives; signature verification is delegated to the caller by contract (`engram.rs:478-482`) | `src/engram.rs:20-603` |
| F-15 | Git push permission engine | ref-pattern grammar + matcher, update classification, `buzz-protect` tag parsing with forward-compat warnings, effective-rule union, default role table, per-ref and whole-push evaluation with denial reasons | **full** for evaluation inputs; the `is_ancestor` fact and the Bot→Member promotion are supplied by callers (`git_perms.rs:208-210`, test note `:910-924`) | `src/git_perms.rs:19-597` |
| F-16 | NIP-AB device pairing | HKDF derivations (session id, SAS, transcript hash), 6-digit SAS formatting, constant-time compare, `nostrpair://` QR encode/decode, full 7-state session machine for both roles with dedup, expiry, abort handling, and secret zeroization | **full** as a pure protocol engine; relay I/O and user interaction are the caller's job (`session.rs:6-8`) | `src/pairing/` (5 files, 2,638 lines) |
| F-17 | Test fixtures for dependents | `make_event`, `make_event_with_keys`, `make_stored_event` behind the `test-utils` feature | **full** | `src/lib.rs:47-74`; consumed by `crates/buzz-relay/Cargo.toml:89` |

---

### TODO / FIXME / HACK / XXX comments

**Zero.** A recursive search of `crates/buzz-core/src` for `TODO`, `FIXME`, `HACK`, and `XXX` returns no matches. The crate carries no in-code deferral markers.

Related in-code forward-looking notes (not TODO-tagged, quoted verbatim):

| Note | file:line |
|------|-----------|
| "Currently a tiny linear set. If this grows past ~4 kinds, convert to a / compile-time bitset or sorted array with binary search for hot-path use." | `src/kind.rs:118-119` |
| "Forward-compatibility: unknown rules are skipped but reported." | `src/git_perms.rs:345` |
| "We still verify rather than `.expect()` so a future change to the serializer can't silently introduce a panic on the hot path." | `src/engram.rs:445-448` |
| "Residual transient copies that cannot be zeroized: 1. `serde_json::to_string` may create intermediate buffers during serialization 2. `nip44::encrypt` reads the plaintext but does not zero its internal copy" | `src/pairing/session.rs:556-558` |

One truncated/orphaned comment fragment sits in the engram test module with no preceding context line:

```
    //    vectors". Pinning these as CI invariants is the single best
```
— `src/engram.rs:615`. (Recorded in the debt doc as well.)

---

### Capabilities explicitly *not* in this crate

| Absent capability | Evidence |
|---|---|
| Any I/O, async runtime, DB, cache, or HTTP | `Cargo.toml:28` comment "NO tokio, NO sqlx, NO redis, NO axum — zero I/O dependencies"; no `tokio`/`sqlx`/`redis`/`axum` entries in `[dependencies]` (`Cargo.toml:13-27`) |
| Environment/config reading | no `env::var` or `std::env` occurrences anywhere in `src/` |
| Filter `limit` handling and NIP-50 `search` routing | `filter.rs:35-104` implements neither |
| NIP-42 AUTH URL equivalence | delegated to `buzz-auth` by explicit doc statement (`relay.rs:28-32`) |
| Signature verification inside engram decrypt | delegated to the caller (`engram.rs:478-482`) |
| Enforcement of the p-gate / author-only / result-gated read rules | this crate only *declares* the kind sets (`kind.rs:112-156`); enforcement is documented as living in the relay and search crates |


## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: Features

The crate's stated scope: build validated, unsigned Nostr events for Buzz
operations; the caller signs and transports them
(`crates/buzz-sdk/src/lib.rs:5-13`). Every capability below is therefore
"produce the correct wire form", never "perform the operation".

---

### 1. Capabilities driveable through the SDK

| Capability area | Builders / functions | Kinds | Completeness | Evidence |
|---|---|---|---|---|
| Stream chat messaging | `build_message` | 9 | Full — threading, mentions, broadcast flag, imeta media | `builders.rs:219-241` |
| Forum discussions | `build_forum_post`, `build_forum_comment`, `build_vote` | 45001, 45003, 45002 | Full for post/comment/vote | `builders.rs:278-305`, `446-460` |
| Message editing | `build_edit` | 40003 | Full | `builders.rs:378-389` |
| Message deletion | `build_delete_message`, `build_delete_message_with_options`, `build_delete_compat` | 9005, 5 | Full, incl. moderation tombstone metadata | `builders.rs:403-443` |
| Reactions | `build_reaction`, `build_custom_emoji_reaction`, `build_remove_reaction` | 7, 5 | Full (NIP-25 + NIP-30) | `builders.rs:463-498` |
| Custom emoji palette | `build_custom_emoji_set`, `normalize_custom_emoji_shortcode`, `CUSTOM_EMOJI_SET_D_TAG` | 30030 | Partial — write-side only; the doc states add/remove is "read-own-set → mutate → rebuild", and the read/union step is left to callers | `builders.rs:503-527` |
| Code-diff sharing | `build_diff_message` + `DiffMeta` | 40008 | Full metadata surface (repo, commit, file, branch pair, PR number, language, truncation, alt text) | `builders.rs:308-375` |
| Canvas documents | `build_set_canvas` | 40100 | Partial — no content-length validation, unlike every other content builder | `builders.rs:529-532` |
| Profiles (NIP-01) | `build_profile` | 0 | Partial — only 5 fields (`display_name`, `name`, `picture`, `about`, `nip05`); other kind-0 fields cannot be set | `builders.rs:537-562` |
| Channel lifecycle | `build_create_channel`, `build_update_channel`, `build_set_topic`, `build_set_purpose`, `build_archive`, `build_unarchive`, `build_delete_channel` | 9007, 9002, 9008 | Full for name/about/visibility/type/ttl/topic/purpose/archive | `builders.rs:604-730` |
| Membership | `build_add_member`, `build_remove_member`, `build_join`, `build_leave` | 9000, 9001, 9021, 9022 | Full | `builders.rs:565-598`, `703-706` |
| Direct messages | `build_dm_open`, `build_dm_add_member` | 41010, 41011 | Partial — conversation setup only; no DM message-body or gift-wrap builder in this crate | `builders.rs:1544-1566` |
| Presence | `build_presence_update` | 20001 | Full for the three-state vocabulary | `builders.rs:1570-1585` |
| Global social notes | `build_note` | 1 | Partial by design — flat reply model; "Full NIP-10 threading (root + reply + p-tags) is deferred" | `builders.rs:732-748` |
| Contact lists (NIP-02) | `build_contact_list` | 3 | Full replacement semantics; deltas require caller read-before-write | `builders.rs:753-813` |
| Git repo announcement (NIP-34) | `build_repo_announcement`, `build_repo_announcement_with_tags` | 30617 | Full, incl. read-modify-write path preserving unknown tags | `builders.rs:828-963` |
| Git patches | `build_git_patch` + `GitPatchMeta` | 1617 | Full tag surface (euc, series markers, commit/parent, PGP sig, committer identity) | `builders.rs:1007-1069` |
| Git issues | `build_git_issue` + `GitIssueMeta` | 1621 | Full (subject, labels, recipients) | `builders.rs:1081-1111` |
| Git status transitions | `build_git_status`, `GitStatus`, `GitStatusMeta`, `GitAppliedPatchRef::parse` | 1630–1633 | Full, incl. merge-only fields and `q`-tag relay/pubkey hints | `builders.rs:1114-1299` |
| Git pull requests | `build_git_pull_request`, `build_git_pr_update` | 1618, 1619 | Full event shape; explicitly does **not** verify commit reachability or perform network work | `builders.rs:1301-1460` |
| Workflows | `build_workflow_def`, `build_workflow_update`, `build_workflow_delete`, `build_workflow_trigger`, `build_workflow_approval` | 30620, 5, 46020, 46030/46031 | Full CRUD + trigger + approve/deny | `builders.rs:1462-1541` |
| Community moderation | `build_moderation_ban`/`unban`/`timeout`/`untimeout`/`resolve_report` | 9040–9044 | Full command surface; status↔action pairing deliberately delegated to the relay | `builders.rs:1587-1690` |
| Identity archival (NIP-IA) | `build_archive_identity_request`, `build_unarchive_identity_request` | 9035, 9036 | Full request shape; consent-path selection is relay-side | `builders.rs:1692-1823` |
| Agent observer streaming | `build_agent_observer_frame` | 24200 | Partial — validates that content *looks like* NIP-44 ciphertext but does not encrypt; encryption lives in `buzz_core::observer` | `builders.rs:243-274`; `crates/buzz-core/src/observer.rs:53-55` |
| Mention resolution | `extract_at_names`, `extract_at_mentions_with_known`, `match_names_to_profiles`, `merge_mentions`, `normalize_mention_pubkeys`, `strip_code_regions`, `extract_nostr_uris` | n/a | Full pure-function pipeline; membership/profile querying is the caller's job | `mentions.rs:1-30`, `64-387` |
| NIP-OA owner attestation | `compute_auth_tag`, `verify_auth_tag`, `parse_auth_tag` | n/a (tag) | Full sign/verify/parse with spec test vector pinned | `nip_oa.rs:146-299`; vector `nip_oa.rs:313-333`, `nip_oa.rs:487-494` |
| Event reading | `extract_channel_id` | n/a | Minimal — the crate is write-oriented; this is the only inbound-event helper | `builders.rs:816-826` |

Kind coverage: **35 distinct event kinds** across 51 public builder functions
(kinds 0, 1, 3, 5, 7, 9, 1617, 1618, 1619, 1621, 1630, 1631, 1632, 1633, 9000,
9001, 9002, 9005, 9007, 9008, 9021, 9022, 9035, 9036, 9040, 9041, 9042, 9043,
9044, 20001, 24200, 30030, 30617, 30620, 40003, 40008, 40100, 41010, 41011,
45001, 45002, 45003, 46020, 46030, 46031 — 45 kind integers, 35 unique
kind *families* if the four Git status kinds and the two approval kinds are each
counted once).

---

### 2. Explicitly stubbed / deferred behavior

| Item | Statement in code | File:line |
|---|---|---|
| Full NIP-10 threading for kind 1 | "This is intentionally simpler than the full `ThreadRef` mechanism used for channel messages — social notes use a flat reply model for now. Full NIP-10 threading (root + reply + p-tags) is deferred." | `builders.rs:732-737` |
| PR tip reachability | "this builder does no network work and does not verify reachability" | `builders.rs:1311-1315` |
| NIP-OA verification depth in builders | "Structural check only — the relay performs full NIP-OA verification." | `builders.rs:1722-1723` |
| NIP-IA consent path | "The relay verifies; this builder's job is to produce a well-formed, signed request — the relay selects the consent path (self / admin / owner)." | `builders.rs:1697-1699` |
| Moderation status/action pairing | "(`dismiss` pairs with `dismissed`, everything else with `resolved` — the relay enforces the pairing)" | `builders.rs:1648-1651` |
| Emoji palette union | "The workspace palette shown in clients is the union of every member's set, deduped by `(shortcode, url)` on read." (read side not implemented here) | `builders.rs:508-510` |
| `parse_auth_tag` skips crypto | "This is the fast path used at MCP startup — no crypto is performed." | `nip_oa.rs:249-250` |
| `GitStatusMeta.recipients` defaulting | "GitStatusMeta.recipients is the caller's responsibility (the CLI defaults it)" | `builders.rs:3157-3159` (test comment) |

---

### 3. TODO / FIXME / HACK / XXX comments

A recursive search across `crates/buzz-sdk/` (all files, including tests and the
example) for `TODO`, `FIXME`, `HACK`, and `XXX` returned **zero matches**. There
are no such markers in this crate.

The deferred-work statements listed in section 2 are ordinary doc comments, not
tagged markers.

---

### 4. Feature-flag-gated capabilities

None. There is no `[features]` table in `crates/buzz-sdk/Cargo.toml` and no
`#[cfg(feature = ...)]` anywhere in the crate (verified by search across
`src/` and `examples/`). The only conditional compilation is `#[cfg(test)]` on
the three test modules (`builders.rs:1825`, `mentions.rs:389`,
`nip_oa.rs:301`).


## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: Features

### Capability inventory

| # | Capability | Entry point | Completeness | Evidence |
|---|---|---|---|---|
| F1 | Parse a `.persona.md` (YAML frontmatter + markdown body) from a string | `persona::parse_persona_md` (`crates/buzz-persona/src/persona.rs:208`) | Full | 29 unit tests in `crates/buzz-persona/src/persona.rs:331-644` covering minimal, full-field, missing/empty fields, malformed YAML, size limits, delimiter edge cases |
| F2 | Parse a `.persona.md` from disk with a pre-read size guard | `persona::parse_persona_file` (`crates/buzz-persona/src/persona.rs:262`) | Full | size check at `:263-267`; no dedicated test for the `TooLarge` path (see debt doc) |
| F3 | Frontmatter splitting with strict own-line closing delimiter | `persona::split_frontmatter` (`crates/buzz-persona/src/persona.rs:277`) | Full | scanner loop `:287-306`; tests `:482-501` |
| F4 | `provider:model-id` splitting on first colon | `persona::split_model` (`crates/buzz-persona/src/persona.rs:324`) | Full | tests `:545-563` |
| F5 | Parse `plugin.json` (OPS-superset manifest) | `manifest::parse_manifest` / `parse_manifest_file` (`crates/buzz-persona/src/manifest.rs:152`, `:190`) | Full | 14 tests `crates/buzz-persona/src/manifest.rs:195-364` |
| F6 | Load a whole pack directory (manifest + personas + instructions + `.mcp.json` + skills-dir detection) | `pack::load_pack` (`crates/buzz-persona/src/pack.rs:125`) | Full for the paths it reads; **partial** in coverage — pack-level `hooks/hooks.json` is deliberately never loaded (`crates/buzz-persona/src/pack.rs:111-112`) | 14 tests `crates/buzz-persona/src/pack.rs:447-734` |
| F7 | Path-traversal / symlink-escape defense for manifest-declared paths | `safe_resolve` (`crates/buzz-persona/src/pack.rs:323`) | Full for `personas`, `pack_instructions`, `mcp_config`. **Not applied** to hook paths (`crates/buzz-persona/src/resolve.rs:339-346`) or persona `skills` paths | tests `crates/buzz-persona/src/pack.rs:561-607` |
| F8 | Behavioral-config precedence merge (levels 3–5) | `merge::merge_behavioral_config`, `merge::resolve_persona_config` (`crates/buzz-persona/src/merge.rs:47`, `:85`) | Full | 22 tests `crates/buzz-persona/src/merge.rs:200-464` including null/empty-container semantics and shallow-replacement |
| F9 | Full pack resolution to ACP-ready output | `resolve::resolve_pack`, `resolve_loaded_pack` (`crates/buzz-persona/src/resolve.rs:108`, `:118`) | Full | 26 tests `crates/buzz-persona/src/resolve.rs:400-892` |
| F10 | Single-persona lookup by name | `resolve::resolve_persona_by_name` (`crates/buzz-persona/src/resolve.rs:186`) | Full — note it re-reads and re-resolves the entire pack (`:187`) | tests `:806-861` |
| F11 | MCP server merge (pack shared + per-persona, deterministic order) | `merge_mcp_servers` (`crates/buzz-persona/src/resolve.rs:277`) | Partial — merge works; `${VAR}` interpolation is explicitly out of scope | doc `:274`; test `:558-572` asserts literals survive |
| F12 | Env-var projection per agent runtime (`goose` vs `buzz-agent`) | `runtime_env_vars` (`crates/buzz-persona/src/resolve.rs:365`) | Full for the two known runtimes; every other runtime silently falls into the GOOSE_* branch (`:380`) | tests `:602-737`, `crates/buzz-persona/tests/e2e_env_flow.rs:35-356` |
| F13 | Lifecycle hooks | `resolve_hooks` (`crates/buzz-persona/src/resolve.rs:347`) | **Stubbed** — parsed and carried only. Field comment: "Hooks (parsed, not executed — reserved for future use, not yet wired)" (`crates/buzz-persona/src/resolve.rs:57`) | no process-spawn code anywhere in the crate |
| F14 | Skill scoping (claimed vs shared) | `pack::resolve_skills` (`crates/buzz-persona/src/pack.rs:249`) | **Stubbed / orphaned** — implemented and tested, but never called by `load_pack` or `resolve_pack`; `ResolvedPersona.skills` carries raw declared paths instead (`crates/buzz-persona/src/resolve.rs:249`), and the field is marked "reserved for future use, not yet wired" (`:60`) | grep for `resolve_skills` finds only the definition, its own tests, and no production caller |
| F15 | Pack validation with error/warning severity and exit codes | `validate::validate_pack` (`crates/buzz-persona/src/validate.rs:143`) | Partial — see gaps table below | 22 tests `crates/buzz-persona/src/validate.rs:436-963` |
| F16 | Human-readable validation output | `Display for ValidationReport` (`crates/buzz-persona/src/validate.rs:74-96`) | Full | rendered by `crates/buzz-cli/src/commands/pack.rs:26-35` (which iterates diagnostics itself rather than using `Display`) |
| F17 | `engines.buzz` version negotiation | field only (`crates/buzz-persona/src/manifest.rs:39-40`) | **Not implemented** — parsed, never compared; no semver dependency in `crates/buzz-persona/Cargo.toml:10-14` | — |
| F18 | Pack integrity (sha256 / `pack.lock`), zip/git install, pack packaging | — | **Absent** — no such code in the crate; `PERSONA_PACK_SPEC.md` §11/§13 describe it as buzz-acp/CLI responsibility | no `sha2`, `zip`, or HTTP dependency in `crates/buzz-persona/Cargo.toml` |
| F19 | Prompt templating / variable substitution | — | **Absent** — `system_prompt` is a verbatim copy (`crates/buzz-persona/src/resolve.rs:200`) | no template engine dependency |

---

### Validation feature gaps (F15 detail)

Measured against the checks the crate itself declares or that
`PERSONA_PACK_SPEC.md` §16 (PF-2) lists as remaining:

| Gap | Evidence |
|---|---|
| `skills:` paths are never checked for existence | `advisory_check_skill_names` only inspects directories that already exist (`crates/buzz-persona/src/validate.rs:365-370`); no "declared skill missing" diagnostic exists |
| `hooks:` paths are never checked for existence | no hook handling in `validate.rs` (grep for `hook` in that file matches only the `hooks_config` entry of `KNOWN_MANIFEST_KEYS` at `crates/buzz-persona/src/validate.rs:115`) |
| `SKILL.md` missing `name:`/`description:` is not an error | only `name` is read (`crates/buzz-persona/src/validate.rs:420-422`); `description` never inspected |
| Only the **first** structural fault is reported | early `return` at `crates/buzz-persona/src/validate.rs:148-151`; test comment acknowledges this: "load_pack fails on the first missing required field ... a single error is emitted" (`crates/buzz-persona/tests/integration.rs:296-299`) |
| `temperature` range is not validated | no range check in `merge.rs` or `validate.rs`; matches spec §10 which states type-only checking |
| Unknown persona-frontmatter keys produce a serde error message, not a targeted diagnostic | K1 path — `Error("pack failed to load: invalid file ...: failed to parse YAML frontmatter: unknown field ...")`; test asserts only that the key name appears (`crates/buzz-persona/src/validate.rs:707-733`) |

---

### TODO / FIXME / HACK / XXX comments

**Zero.** A grep for `TODO`, `FIXME`, `HACK`, and `XXX` across the entire
`crates/buzz-persona` directory (all `.rs` files, `Cargo.toml`, and
`PERSONA_PACK_SPEC.md`) returns no matches.

Deferred work is instead expressed as prose in doc comments. Verbatim, with locations:

| Location | Comment (verbatim) |
|---|---|
| `crates/buzz-persona/src/resolve.rs:57` | `// Hooks (parsed, not executed — reserved for future use, not yet wired)` |
| `crates/buzz-persona/src/resolve.rs:60` | `// Skills (bare names — reserved for future use, not yet wired)` |
| `crates/buzz-persona/src/resolve.rs:274` | `/// Env values are passed through as literals — no `${VAR}` interpolation.` |
| `crates/buzz-persona/src/resolve.rs:341-346` | `/// Security: we intentionally do NOT resolve these to absolute paths.` … `/// Hook paths come from untrusted persona frontmatter and could contain` / `/// `../` traversal. Since hooks are not executed in this PR, we store` / `/// them as-is. The PR that wires execution MUST validate through` / `/// `safe_resolve()` before use.` |
| `crates/buzz-persona/src/resolve.rs:225-231` | `// Version: LoadedPersona has no per-persona version field — persona files` / `// don't declare a version in frontmatter. The pack version is used as-is.` / `// If per-persona versioning is added in the future, LoadedPersona should` / `// gain `version: Option<String>` and this line should become:` / `//   lp.version.clone().unwrap_or_else(|| pack_version.to_owned())` |
| `crates/buzz-persona/src/resolve.rs:178-179` | `// Pack-level description not yet wired through PackManifestData.` |
| `crates/buzz-persona/src/pack.rs:111-112` | `// hooks_config is intentionally omitted: hooks are a runtime concern loaded` / `// separately by buzz-acp, not a pack-parsing concern.` |
| `crates/buzz-persona/src/persona.rs:230` | `// Fix #1: enforce non-empty required strings` |
| `crates/buzz-persona/src/persona.rs:263` | `// Fix #4: check file size before reading to avoid large allocations` |
| `crates/buzz-persona/src/merge.rs:9` | `/// Levels 1–2 (operator env vars, desktop UI) are resolved at runtime.` |

Note on `crates/buzz-persona/src/resolve.rs:178-179`: the comment says the pack
description is "not yet wired", but the code on the next line does read it
(`loaded.manifest.description.clone().unwrap_or_default()` at `:180`) and
`PackManifestData.description` exists (`crates/buzz-persona/src/pack.rs:106`) and is
populated (`:143`). The comment appears stale.

Two `// Fix #N:` markers (`persona.rs:230`, `persona.rs:263`) reference an external
review-item numbering that is not defined anywhere in the repo.

---

### Spec-vs-implementation deltas

`PERSONA_PACK_SPEC.md` ships inside this crate and describes behavior that lives partly
outside it. Items the spec marks as implemented-elsewhere or planned, cross-checked
against this crate's code:

| Spec item | Spec location | State in this crate |
|---|---|---|
| Skill copying to `$AGENT_CWD/.agents/skills/` | `PERSONA_PACK_SPEC.md:359-361` ("planned for a future release") | absent; `resolve_skills` computes the mapping but nothing copies |
| MCP `${VAR}` interpolation | `PERSONA_PACK_SPEC.md:512-514` ("planned but not yet implemented") | confirmed absent (`crates/buzz-persona/src/resolve.rs:274`) |
| Hook execution | `PERSONA_PACK_SPEC.md:551-552` ("parsed and validated at pack load time but not yet executed") | parsed only; pack-level hooks not even parsed (`crates/buzz-persona/src/pack.rs:111-112`) |
| `engines.buzz` rejection of too-new packs | `PERSONA_PACK_SPEC.md:113-114` | not implemented (F17) |
| `buzz pack validate` — "Remaining: verify `skills:` and `hooks:` paths exist; error on `SKILL.md` missing `name:` or `description:`" | `PERSONA_PACK_SPEC.md:1139` (PF-2) | confirmed still missing (gap table above) |
| Unknown keys in `defaults` are validation warnings; unknown persona frontmatter keys are hard errors | `PERSONA_PACK_SPEC.md:803-807` | matches implementation (K6, A8) |
| V6 namespaced `buzz:` block is unsupported | `PERSONA_PACK_SPEC.md:1108` | consistent — `buzz` is not a `Frontmatter` field, so such a file fails `deny_unknown_fields` |


## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: Features

Library crate, no binary targets (`crates/buzz-ws-client/Cargo.toml:1`–`7`, no
`[[bin]]`/`[features]` sections). Purpose per the module docs: connect, NIP-42
authenticate, publish, and read relay messages.

---

### 1. Capability matrix

| Capability | State | Evidence |
|---|---|---|
| Connect to a relay over WebSocket (ws/wss via `MaybeTlsStream`) | Full | `connection.rs:48`–`65`, `:8`, `:14` |
| NIP-42 authentication (await challenge → sign → send AUTH → await OK) | Full | `connection.rs:70`–`93`; `message.rs:174`–`190` |
| Connect + authenticate in one call | Full | `connection.rs:37`–`45` |
| Attach an extra authorization tag (documented as "NIP-OA") to the AUTH event | Full (single tag only) | `connection.rs:36`, `message.rs:169`–`186` |
| Publish a signed event and await its `OK` | Full | `connection.rs:96`–`101` |
| One-shot publish helper with overall timeout | Full | `connection.rs:277`–`294` |
| Read the next relay message with a caller-set timeout | Full | `connection.rs:104`–`112`, `:128`–`155` |
| Out-of-order message buffering while awaiting a specific frame | Full | `connection.rs:28`, `:205`, `:257`, `:259` |
| Parse the seven NIP-01/NIP-42/NIP-45 relay message types | Full for `EVENT`, `OK`, `EOSE`, `CLOSED`, `NOTICE`, `AUTH`, and `COUNT` (the seventh was added by the post-analysis sync; six at report time) | `message.rs:70`–`162` |
| WebSocket keepalive (Ping → Pong) | Full | `connection.rs:148`–`150`, `:208`–`:210`, `:262`–`:264` |
| Graceful close | Full | `connection.rs:115`–`118` |
| Send arbitrary JSON frames (escape hatch used for `REQ`/`CLOSE`/`COUNT`) | Full, but untyped | `connection.rs:121`–`126` |
| Subscription management (typed `REQ`/`CLOSE`, filter types, EOSE-driven collection) | **Absent** — no such API; callers hand-roll it (e.g. `crates/buzz-test-client/src/lib.rs:154`, `:160`) | no `REQ`/`CLOSE` literal anywhere in the crate |
| `COUNT` message support | **Status: resolved** by the post-analysis sync (was: absent — no `RelayMessage` variant, so a `COUNT` reply failed as `UnexpectedMessage`). A `Count { subscription_id, count }` variant (`message.rs:40`–`46`) and a `"COUNT"` parse arm reading `arr[2]["count"]` as `u64` (`:147`–`162`) now handle it; a malformed payload still falls back to `UnexpectedMessage`. | `message.rs:40`–`46`, `:147`–`162` |
| Reconnect / retry / backoff | **Absent** | no retry or backoff code in `connection.rs` (verified over the whole file) |
| Automatic re-auth on mid-session challenge | **Partial** — challenge is captured and buffered, but re-auth is not triggered; caller must call `authenticate` again | `connection.rs:255`–`258` vs `:161` |
| Rejection surfacing for published events | **Partial** — `accepted: false` is returned to the caller rather than raised as an error, and the `EventRejected` variant exists unused | `connection.rs:96`–`101`; `error.rs:40` |
| Binary frame handling | **Ignored by design** (`_ => {}`) | `connection.rs:152`, `:212`, `:266` |
| Metrics / instrumentation | **Minimal** — three `debug!` lines, no spans, no counters | `connection.rs:57`, `:91`, `:123` |
| Configuration surface (env vars, cargo features) | **Absent** — see the configuration aspect | no `env::var`, no `cfg(feature = …)` in the crate |

---

### 2. Completeness notes

- The public API is closed over a narrow flow: connect → auth → publish → read →
  close. Everything a NIP-01 subscriber needs beyond that is delegated to
  `send_raw` + `next_event` (`connection.rs:121`, `:104`).
- `next_event` re-checks the buffer immediately before delegating to `recv_one`,
  which checks it again (`connection.rs:108`–`111` then `:129`–`131`) — behaviourally
  redundant but harmless.
- The `Auth` challenge cap (1024 bytes) exists on only one of the three challenge
  intake paths (`connection.rs:198`; the paths at `:161` and `:170` are uncapped).

---

### 3. TODO / FIXME / HACK / XXX markers

**Zero.** A case-insensitive search for `TODO`, `FIXME`, `HACK`, `XXX`,
`unimplemented`, and `todo` across `crates/buzz-ws-client/` returned no matches
(exit code 1, no output). There are also no `#[deprecated]`, `#[allow(dead_code)]`,
or `unimplemented!()`/`todo!()` markers. The only lint attributes present are
`#![deny(unsafe_code)]` (`lib.rs:1`) and `#[allow(clippy::result_large_err)]`
(`message.rs:61`).

---

### 4. Tests shipped with the crate

Three unit tests, all compile-time assertions on the timeout floors — no I/O, no
mock relay, no parser tests:

| Test | Asserts | file:line |
|---|---|---|
| `auth_challenge_timeout_meets_floor` | `AUTH_CHALLENGE_TIMEOUT_SECS >= 20` | `connection.rs:300`–`303` |
| `auth_ok_timeout_meets_floor` | `AUTH_OK_TIMEOUT_SECS >= 20` | `connection.rs:305`–`308` |
| `publish_ok_timeout_meets_floor` | `PUBLISH_OK_TIMEOUT_SECS >= 30` | `connection.rs:310`–`313` |

Module gate at `connection.rs:296`–`298`. Each body uses an inline `const { assert!(…) }`
block, so the check is evaluated at compile time rather than at run time.


## Module: buzz-db (`crates/buzz-db`)

### Aspect: Features

Capabilities the crate provides, with completeness read from the code (not from
docs). "Complete" means the module has both read and write paths and callers in
the workspace surface; "read-only", "partial", and "schema-only" are noted.

| # | Capability | State | Evidence |
|---|-----------|-------|----------|
| 1 | **Nostr event store** — insert (dedup), scoped query with 14 pushdown filters + the kind-30175 visibility clause added by `ab3af828` (one more optional predicate block: 11 → 12 `if let Some(…) = q.…` blocks in `query_events`), COUNT, scoped id lookup, batch id lookup, soft delete, coordinate delete | complete | `insert_event` `crates/buzz-db/src/event.rs:256`, `query_events` `:318`, `count_events` `:598`, `get_event_by_id` `:909`, `get_events_by_ids` `:989`, `soft_delete_event` `:744`, `soft_delete_by_coordinate` `:771` |
| 2 | **Monthly range partitioning** of `events` and `delivery_log` with startup/cron partition creation | complete for creation; **no drop/detach/rotation-out path exists** | `crates/buzz-db/src/partition.rs:15-73`; partitions declared `migrations/0001_initial_schema.sql:237-252`, `:343-354` |
| 3 | **Replaceable-event replacement** — NIP-16 addressable (`kind, pubkey, channel_id`) and NIP-33 parameterized (`kind, pubkey, d_tag`), each under an advisory lock with deterministic tie-breaks | complete | `crates/buzz-db/src/lib.rs:3306`, `:3628` |
| 4 | **NIP-RS read-state retention** — exact-cardinality classification, payload hard delete, durable ordering watermark, mixed-version DB triggers, hard-delete opt-in fence | complete | `crates/buzz-db/src/lib.rs:3672-3796`; `migrations/0007`, `0009`, `0010`, `0011` |
| 5 | **Mesh-status heartbeat retention** (kind 30003 `buzz-mesh-member-status:*`) — one physical row per coordinate | complete | `crates/buzz-db/src/lib.rs:3688-3695`; `migrations/0019_mesh_status_retention.sql` |
| 6 | **Channel CRUD + membership** with transactional role enforcement, soft delete, last-owner protection, archive/unarchive, topic/purpose, canvas | complete | `crates/buzz-db/src/channel.rs` (26 public fns) |
| 7 | **Ephemeral channels with TTL** — deadline init, in-commit refresh, advisory-lock-ordered TTL transitions, idempotent reaper | complete | `crates/buzz-db/src/channel.rs:1387`; `migrations/0022`, `0024` |
| 8 | **Threading** — parent/root/depth metadata, materialized `reply_count`/`descendant_count`/`last_reply_at`, keyset-paginated replies, channel window with server `has_more`, batched participants | complete | `crates/buzz-db/src/thread.rs` |
| 9 | **Reactions** — TOCTOU-free upsert with new/reactivate/duplicate semantics, soft delete, grouped and bulk reads, single-transaction kind:7 coupling | complete | `crates/buzz-db/src/reaction.rs`; `crates/buzz-db/src/event.rs:1242` |
| 10 | **DMs** — participant-hash identity, 2–9 participants, idempotent open, per-user hide/unhide, hidden-DM listing | complete | `crates/buzz-db/src/dm.rs` |
| 11 | **Home feed** — mentions / needs-action / activity over the normalized `event_mentions` index with a hard 100-row cap | complete | `crates/buzz-db/src/feed.rs` |
| 12 | **Mention index** (`event_mentions`) — multi-row upsert built from validated `p` tags, best-effort (failures logged, never block the event) | complete | `crates/buzz-db/src/lib.rs:97-165`, call sites `:1085-1090`, `:1391-1396`, `:1419-1428` |
| 13 | **User profiles** — upsert, profile update with empty→NULL semantics, NIP-05 lookup, LIKE-escaped search with ranking, agent ownership, channel-add policy | complete | `crates/buzz-db/src/user.rs` |
| 14 | **API tokens** — hashed storage, atomic 10-token quota, community-scoped lookup (incl. revoked), listing, single/bulk revoke, `last_used_at` touch | complete | `crates/buzz-db/src/api_token.rs`, `crates/buzz-db/src/lib.rs:2327-2461` |
| 15 | **Workflows / runs / approvals** — owner-and-channel-guarded upsert, bounded lists, run lifecycle with `started_at`/`completed_at` stamping, SHA-256 approval tokens, single-decision approval updates | complete; 5 functions are documented as having **no current callers** (`create_workflow`, `update_workflow`, `update_workflow_status`, `set_workflow_enabled`, `delete_workflow`) | `crates/buzz-db/src/workflow.rs:275`, `:619`, `:653`, `:683`, `:714` |
| 16 | **Cron fire claims** — at-most-once `(community, workflow, scheduled_for)` claim, DB-authoritative interval anchor, run-id attach, retention prune | complete | `crates/buzz-db/src/workflow.rs:496-611` |
| 17 | **NIP-ER reminders** — due-set query with per-tenant host provenance, exactly-once claim, stamped release | complete | `crates/buzz-db/src/event.rs:1334-1472` |
| 18 | **Relay membership (NIP-43)** — CRUD, invite claim with policy-acceptance evidence, owner-protected removal/role change, owner bootstrap, atomic ownership transfer with a 3-community cap, allowlist backfill, locked snapshot publication | complete | `crates/buzz-db/src/relay_members.rs`; snapshot at `crates/buzz-db/src/lib.rs:3438-3626` |
| 19 | **Pubkey allowlist** — membership check, enforcement-active check, add/remove/list | complete (inline on `Db`) | `crates/buzz-db/src/lib.rs:2826-2905` |
| 20 | **Archived identities (NIP-IA)** — community-scoped archive/unarchive/list, idempotent | complete | `crates/buzz-db/src/archived_identities.rs` |
| 21 | **Community moderation** — NIP-56 report ingest/queue/resolve, ban + timeout state with expiry-aware reads, audit action log | complete | `crates/buzz-db/src/moderation.rs` |
| 22 | **Deployment-admin moderation plane** — keyset-paginated global report list, global feedback list, single-row fetches | read-only by construction | `crates/buzz-db/src/admin_moderation.rs` |
| 23 | **Product feedback sidecar** — deployment-wide idempotent insert by event id, newest-first list | complete | `crates/buzz-db/src/product_feedback.rs` |
| 24 | **Git repo-name registry (NIP-34)** — atomic per-community reservation, owner classification, quota count, owner-scoped release | complete | `crates/buzz-db/src/git_repo.rs` |
| 25 | **NIP-PL push leases + wake outbox + match queue** — signed-lease acceptance with ordering/quota/endpoint rules, gate-guarded matcher enqueue, batched wake enqueue, fenced claim/complete/retry/fail, send-time revalidation, generation-scoped endpoint disable, pruning | complete for the relay-owned side | `crates/buzz-db/src/push.rs` |
| 26 | **Read-replica routing with a freshness fence** — commit-time floor guard, ordered LSN handshake, staleness expiry, two-part startup verification, per-query routing decisions | complete | `crates/buzz-db/src/replica_fence.rs`; `crates/buzz-db/src/lib.rs:2004`, `:2063` |
| 27 | **Per-community usage rollups** for Prometheus gauges (11 queries) plus a detached-session advisory leader lock | complete | `crates/buzz-db/src/usage.rs`; `crates/buzz-db/src/lib.rs:517-535` |
| 28 | **Embedded migrations + tenant-isolation lint harness** — 24 migrations, pre-flight ambiguity guard, post-migration floor-guard verification, SQL-parsing lint tests | complete | `crates/buzz-db/src/migration.rs` |
| 29 | **Pool observability** — `ping`, writer/replica pool stats | complete | `crates/buzz-db/src/lib.rs:485-511` |
| 30 | **Community lifecycle** — host resolve (active / management), icon get/set, ensure-configured, atomic create-with-owner, archive/unarchive, channel→community reverse resolution (single + batch) | complete | `crates/buzz-db/src/lib.rs:656-1077` |
| 31 | **d_tag backfill** maintenance for legacy NIP-33 rows | complete, idempotent, **unscoped by community** | `crates/buzz-db/src/lib.rs:2810-2824` |
| 32 | **Subscriptions / delivery_log / audit_log / rate_limit_violations** | **schema-only in this crate** — tables exist, no Rust read/write path | `migrations/0001_initial_schema.sql:304`, `:329`, `:498`, `:606`; no matching module (verified by search) |
| 33 | **push_gateway_* authority tables** (6) | **schema-only in this crate** — created by `migrations/0015`, referenced only by the lint allowlist in tests | `migrations/0015_push_gateway_authority.sql`; `crates/buzz-db/src/migration.rs:343-348` |

---

#### TODO / FIXME / HACK / XXX inventory

**Zero occurrences.** A case-insensitive search for `TODO`, `FIXME`, `HACK`, and
`XXX` across `crates/buzz-db/**/*.rs`, all 24 files in `migrations/`, and
`schema/schema.sql` returns no matches. Deferred work is instead expressed as
prose in doc comments; the concrete instances are:

| Marker text (verbatim) | File:line |
|------------------------|-----------|
| `/// creation path is [`upsert_workflow`] via event ingest. (No current callers.)` | `crates/buzz-db/src/workflow.rs:275` |
| `/// matching lags the change by up to the cache TTL. (No current callers.)` | `crates/buzz-db/src/workflow.rs:619` |
| `/// [`update_workflow`]. (No current callers.)` | `crates/buzz-db/src/workflow.rs:653` |
| `/// on [`update_workflow`]. (No current callers.)` | `crates/buzz-db/src/workflow.rs:683` |
| `/// `channel_id` needed for invalidation. (No current callers.)` | `crates/buzz-db/src/workflow.rs:714` |
| `/// `cursor` is reserved for future keyset pagination (currently unused).` | `crates/buzz-db/src/reaction.rs:279` (parameter `_cursor` at `:286`) |
| `/// NOTE: The primary increment path is inlined inside [`insert_thread_metadata`]'s` … `/// This standalone version exists for future use cases` (function carries `#[allow(dead_code)]`) | `crates/buzz-db/src/thread.rs:243-250` |
| `// Run one query per event. For typical message-list sizes (<=100 events)` … `// a single-query approach with dynamic IN clauses over` / `// composite keys can be added later if needed.` | `crates/buzz-db/src/reaction.rs:373-375` |
| `//  If query performance` / `// degrades, add a composite index like `(pubkey, kind, created_at DESC, id ASC)`.` | `crates/buzz-db/src/event.rs:532-534` |
| `//! At scale these can become recurring partition scans; if that becomes a` / `//! problem, move them to a maintained rollup table and drop the interval.` | `crates/buzz-db/src/usage.rs:8-10` |
| `/// For large channel sets, consider pagination.` | `crates/buzz-db/src/channel.rs:602` |
| `-- the search lane can` / `-- revisit the config behind evidence.` (FTS `'simple'` config) | `migrations/0001_initial_schema.sql:200-201` |


## Module: buzz-auth (`crates/buzz-auth`)

### Aspect: Features

Completeness legend: **full** = implemented and exercised by tests; **partial** =
implemented but with a documented or observable gap; **interface-only** = trait or
type defined here, behaviour lives elsewhere; **stubbed** = test-only
implementation.

| # | Capability | Completeness | Evidence | Notes |
|---|-----------|--------------|----------|-------|
| 1 | NIP-42 challenge generation | full | `crates/buzz-auth/src/nip42.rs:38-41` | 32 CSPRNG bytes via `rand::random`, hex-encoded; test pins 64 chars + uniqueness (`:102-109`) |
| 2 | NIP-42 AUTH event verification (kind, sig, challenge, relay, timestamp) | full | `crates/buzz-auth/src/nip42.rs:47-86` | 5 rejection tests + 2 normalisation tests |
| 3 | Relay-URL normalisation for AUTH | full | `crates/buzz-auth/src/nip42.rs:19-33` | loopback aliasing + trailing-slash stripping; `ws` vs `wss` not aliased |
| 4 | `AuthService` async wrapper with `spawn_blocking` offload | full | `crates/buzz-auth/src/lib.rs:118-143` | 3 async tests (`:199-242`) |
| 5 | NIP-98 HTTP auth verification (8-step) | full | `crates/buzz-auth/src/nip98.rs:55-131` | 11 tests including payload-hash and loopback-distinctness |
| 6 | NIP-98 body-hash binding (`payload` tag) | partial | `crates/buzz-auth/src/nip98.rs:117-127` | only enforced when the tag **and** a body are both present; presence is the caller's responsibility (`crates/buzz-relay/src/api/bridge.rs:99-112`) |
| 7 | NIP-98 replay protection | interface-only | trait `crates/buzz-auth/src/nip98_replay.rs:64-104`; keys `:114-121`; constants `:46`, `:59` | production Redis impl is `crates/buzz-pubsub/src/nip98_replay.rs:34`; this crate ships only `AlwaysFreshReplayGuard` (`:126-139`) |
| 8 | Scope model + wire-format round-trip | full | `crates/buzz-auth/src/scope.rs:16-178` | 16 known variants + `Unknown` passthrough; 5 tests |
| 9 | Scope granting on NIP-42 | full (but coarse) | `crates/buzz-auth/src/lib.rs:136-142` | grants all 16 scopes unconditionally; no per-connection differentiation |
| 10 | Scope checking (`require_scope`, `has_scope`) | full | `crates/buzz-auth/src/access.rs:60-69`, `crates/buzz-auth/src/lib.rs:84-86` | exact-match only; no hierarchy/wildcards. `require_scope` has no external caller |
| 11 | Channel access checking | interface-only | trait `crates/buzz-auth/src/access.rs:31-57` | **no production implementor anywhere in the repo** (see debt aspect); only `MockAccessChecker` (`:135-151`) |
| 12 | Combined scope+membership helpers | full but unused | `crates/buzz-auth/src/access.rs:72-101` | 4 tests; zero callers outside this crate |
| 13 | Rate limiting | interface-only | trait `crates/buzz-auth/src/rate_limit.rs:168-194` | production Redis impl is `crates/buzz-pubsub/src/rate_limiter.rs:99-121`, wired in `buzz-relay` — see security aspect verdict |
| 14 | Rate-limit tier configuration | full (parse) / partial (consumption) | `crates/buzz-auth/src/rate_limit.rs:86-144` | all 7 fields deserialise with defaults; only 4 are read by any consumer (see configuration aspect) |
| 15 | Rate-limit / replay Redis key construction | full | `crates/buzz-auth/src/rate_limit.rs:201-215`, `crates/buzz-auth/src/nip98_replay.rs:114-121` | community-prefixed for pubkey/event keys, global for IP; lowercase invariant tested |
| 16 | Test doubles (`MockAccessChecker`, `AlwaysAllowRateLimiter`, `AlwaysFreshReplayGuard`) | stubbed by design | `crates/buzz-auth/src/access.rs:108`, `crates/buzz-auth/src/rate_limit.rs:219`, `crates/buzz-auth/src/nip98_replay.rs:127` | all `#[cfg(any(test, feature = "test-utils"))]`; none referenced outside this crate |
| 17 | Dev-only username→pubkey derivation | full but orphaned | `crates/buzz-auth/src/lib.rs:159-167` | `#[cfg(any(test, feature = "dev"))]`; no caller in the repo, and no test either |
| 18 | Error taxonomy | full | `crates/buzz-auth/src/error.rs:9-59` | 10 variants; 2 (`Nip98Replay`, `PubkeyMismatch`) are never constructed in this crate |

---

### Capabilities explicitly absent

| Absent capability | Evidence |
|-------------------|----------|
| JWT / OAuth token validation, token minting, refresh | crate doc "No JWT validation, no token management, no IdP runtime dependency" (`crates/buzz-auth/src/lib.rs:16`, `:98`); no such types in any of the 8 source files |
| Session or credential persistence | no `sqlx`, no DB dependency in `crates/buzz-auth/Cargo.toml:14-26` |
| Any network I/O | no HTTP/WS client dependency; see integrations aspect |
| Per-channel scope narrowing | `AuthContext.channel_ids` is documented "reserved for future per-channel access control" and always constructed as `None` (`crates/buzz-auth/src/lib.rs:69-72`, `:139`) |
| Constant-time comparison of challenge / payload hash | plain `!=` on `&str` (`crates/buzz-auth/src/nip42.rs:64`, `crates/buzz-auth/src/nip98.rs:122`); no `subtle` dependency |
| NIP-98 → `AuthContext` construction | `verify_nip98_event` returns a bare `PublicKey` (`crates/buzz-auth/src/nip98.rs:60`), contradicting the crate-doc invariant at `crates/buzz-auth/src/lib.rs:15` |

---

### TODO / FIXME / HACK / XXX comments

A case-insensitive search of the entire `crates/buzz-auth/` directory for
`TODO`, `FIXME`, `HACK`, and `XXX` returns **zero matches**. There are no
in-code work markers in this crate.

Deferred-work statements are instead written as prose in doc comments. The ones
that carry an explicit "not yet / future / v2 / deferred" claim, verbatim:

| Marker text (verbatim) | file:line |
|------------------------|-----------|
| `/// Channel restriction (reserved for future per-channel access control).` | `crates/buzz-auth/src/lib.rs:69` |
| `/// Reserved for future use. Not currently enforced by git HTTP routes —` / `/// those use NIP-98 auth directly. Will be enforced when collaborator` / `/// access (read-only, maintainer) is added in v2.` | `crates/buzz-auth/src/scope.rs:47-49` |
| `/// Enforced for kind:30617/30618 events via WebSocket ingest, but NOT` / `/// enforced by git HTTP push routes (which use NIP-98 + owner check).` / `/// Full enforcement deferred to v2 collaborator model.` | `crates/buzz-auth/src/scope.rs:53-55` |
| `//! ⚠️ Fixed windows allow up to 2× burst at boundaries. Upgrade to sliding` / `//! window or token bucket for strict limiting.` | `crates/buzz-auth/src/rate_limit.rs:6-7` (repeated at `:165-167`) |
| `/// ⚠️ The fixed-window algorithm used by the Redis implementation allows up to 2×` / `/// burst at window boundaries. Upgrade to a sliding window or token bucket if strict` / `/// per-second limiting is required.` | `crates/buzz-auth/src/rate_limit.rs:165-167` |
| `/// If per-(community, IP) caps are ever needed as a tenant-fairness signal, that` / `/// belongs in an additive \`LimitType\` keyed on \`(community, ip)\`, not in this trait.` | `crates/buzz-auth/src/rate_limit.rs:161-163` |
| `/// # ⚠️ SECURITY — Dev/test only` … `/// and **must never be compiled into a production release build**.` | `crates/buzz-auth/src/lib.rs:152-155` |
| `// Set later by relay membership gate if NIP-OA` | `crates/buzz-auth/src/lib.rs:141` |


## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Features

`buzz-pubsub` is a horizontal-scaling substrate, not a user-facing feature area.
It supplies six capabilities that make a multi-pod relay behave like one relay.

---

### F-PS-1 — Community-scoped realtime event fan-out

Cross-pod delivery of signed Nostr events. A pod publishes to
`buzz:{community}:channel:{id}` or `buzz:{community}:global` (`topic.rs:45-50`),
and every pod holding local interest re-broadcasts to its WebSocket sessions
through a `broadcast::channel(4096)` (`lib.rs:126`, forward at `subscriber.rs:165`).

Architecture as documented in the crate header (`lib.rs:8-16`): a
`deadpool-redis` pool serves all imperative commands, while a **dedicated**
`redis::aio::PubSub` connection — explicitly not from the pool because it is
stateful (`lib.rs:19-20`) — handles subscriptions. Obtained at
`subscriber.rs:85-87` and split into sink/stream so commands and messages can be
serviced from one `tokio::select!` (`subscriber.rs:109-171`).

Relay integration: `subscribe_local` consumers at `buzz-relay/src/main.rs:822` and
`handlers/event.rs:1644`; publishes flow through `PubSubManager::publish_event`
(`lib.rs:322-330`).

### F-PS-2 — Dynamic, refcounted subscription scoping

Rather than subscribing to a firehose, each pod subscribes only to topics with
live local interest. `retain_topic` / `release_topic` (`lib.rs:192`, `:215`)
maintain a refcount map (`subscriber.rs:21`) and emit `Subscribe` /
`UnsubscribeIfIdle` commands (`subscriber.rs:26-31`) to the pub/sub task.

Two properties make this safe:
- **Debounced teardown** (default 500 ms, `lib.rs:82`) with a re-check of the
  refcount at execution time (`subscriber.rs:123-130`), so subscription churn from
  a client re-issuing `REQ` doesn't thrash Redis.
- **Reconnect reconciliation** — the refcount map, not the connection, is the
  source of truth; a fresh connection re-subscribes from a snapshot before
  processing messages (`subscriber.rs:90-101`).

Wired into the relay's subscription lifecycle at `handlers/req.rs:251`, `:256`,
`handlers/close.rs:21`, `handlers/side_effects.rs:81`, `connection.rs:268`, and the
global-topic retains at `handlers/event.rs:1683`, `:1687`.

### F-PS-3 — Presence (online/away) with TTL

`SET buzz:{community}:presence:{pubkey_hex} <status> EX 90` (`presence.rs:36-43`),
TTL const `presence.rs:16`. Designed around a 30 s client heartbeat with 3× TTL
headroom so a single missed beat doesn't flap (`presence.rs:4-6`). Clean disconnect
deletes eagerly (`presence.rs:52-56`).

Reads: single (`presence.rs:62-72`) and bulk `MGET` returning `pubkey_hex → status`
for present keys only (`presence.rs:74-95`). The bulk path is what the relay
actually uses, at `buzz-relay/src/api/bridge.rs:1954`; multi-tenant isolation of
this exact call is pinned by conformance tests
(`crates/buzz-test-client/tests/conformance_multitenant.rs:2371`, `:2484`).
Writes come from the relay event handler (`handlers/event.rs:798`) and clears from
`connection.rs:280` / `handlers/event.rs:793`.

Capability limit: there is **no per-community presence index** (no `SET`/`ZSET` of
online members). `get_presence_bulk` requires the caller to already know the
candidate pubkey list, so "list everyone online in this community" is not
answerable from this crate.

### F-PS-4 — Cross-pod cache invalidation

Each pod keeps in-memory (moka) membership / accessible-channels / visibility
caches with a 10 s TTL; without this feature a membership change would stay stale on
every pod except the writer for up to that TTL (`cache_invalidation.rs:4-9`).

Four drop operations, each mirroring one relay-local `invalidate_*` call
(`cache_invalidation.rs:58-80`): `Membership { channel_id, pubkey }`,
`AccessibleAll`, `Visibility { channel_id }`, `ChannelDeleted`.

Delivery is a wildcard `psubscribe "buzz:*:cache-invalidate"`
(`cache_invalidation.rs:27`, `:139`) with per-message community demux
(`cache_invalidation.rs:38-51`, `:144-148`). Publisher: `buzz-relay/src/state.rs:970`.

The design invariant is that the message is a *pure key drop*, never an
authorization decision — the per-event access gate remains the enforcement point
and the next read re-fetches from the DB (`cache_invalidation.rs:11-14`).

### F-PS-5 — Cross-pod connection control (live ban enforcement)

Solves the "banned member's socket is on another pod" problem
(`conn_control.rs:3-6`). Two commands (`conn_control.rs:56-73`):
- `DisconnectPubkey { pubkey, event_id, reason }` — carries enough context to
  reproduce the same NIP-01 `OK` frame the origin pod sent, so the member learns
  why regardless of which pod held the socket (`conn_control.rs:62-65`).
- `DisconnectCommunity` — drops every socket bound to the carrying community.

Producers: `buzz-relay/src/state.rs:1034` (`DisconnectPubkey`), `:1066`
(`DisconnectCommunity`), both published via `publish_conn_control`
(`state.rs:1044`, `:1066`). Consumer: the subscriber spawned at
`buzz-relay/src/main.rs:366`, receiving on `subscribe_conn_control`
(`main.rs:903`) and dispatching both variants at `main.rs:908` and `:913`.

Deliberately a separate channel from F-PS-4 so that the cache channel's
idempotent-hint invariant is preserved (`conn_control.rs:12-18`). The DB ban row is
the durable backstop: a dropped message still results in refusal at the next auth
attempt (`conn_control.rs:18-21`).

### F-PS-6 — HA primitives borrowed by `buzz-auth` seams

Two trait implementations that let single-pod auth logic work across pods:

- **`RedisRateLimiter`** (`rate_limiter.rs:88-120`) — fixed-window `INCR`+`EXPIRE`
  via one atomic Lua script (`rate_limiter.rs:24-31`), with self-repair for
  TTL-less keys left by an older non-atomic implementation (`rate_limiter.rs:57-70`).
  Constructed at `buzz-relay/src/state.rs:712`, held as
  `admission_rate_limiter` (`state.rs:584`), and enforced for WS `EVENT`/`REQ`/`COUNT`
  at `buzz-relay/src/connection.rs:593-648`.
- **`RedisNip98ReplayGuard`** (`nip98_replay.rs:23-101`) — atomic set-if-absent
  seen-set for NIP-98 HTTP auth, described as the "§5 pre-build gate for
  multi-tenant HA replay protection" (`nip98_replay.rs:4-6`). Constructed at
  `buzz-relay/src/state.rs:711`; two-pod behaviour simulated in relay tests at
  `api/bridge.rs:2275-2276`.

### Documented-but-absent capability: typing indicators

The crate description advertises typing indicators —
`description = "Redis pub/sub fan-out, presence, and typing indicators for Buzz"`
(`Cargo.toml:8`) — and a doc comment reading "Typing indicator tracking in Redis."
sits at `lib.rs:43`. **No typing module, key, or function exists.** The module list
ends at `pub mod topic` (`lib.rs:42`), and the doc comment at `:43` is
mis-attached to `pub use error::PubSubError;` (`lib.rs:44`). A repo-wide grep for
typing-related Redis operations (`ZADD`, `ZREMRANGE`, `typing_key`, `mod typing`)
across `crates/**/*.rs` returns nothing. Feature F-PS-7 does not exist; see the
debt file.


## Module: buzz-search (`crates/buzz-search`)

### Aspect: Features

#### Capability inventory

| Capability | Present | Where |
|---|---|---|
| Community-scoped full-text search over `events.content` (via `search_tsv`) | yes | `query.rs:216-323` |
| Word/lexeme search with Postgres websearch grammar (`FullText`) | yes | `query.rs:142-146` |
| Typeahead prefix search on the trailing token (`Prefix`) | yes | `query.rs:147-176` |
| Relevance ranking (`ts_rank_cd`) surfaced to callers as `SearchHit.rank` | yes | `query.rs:236`, `query.rs:117` |
| Channel scoping with four explicit variants | yes | `query.rs:41-53`, `query.rs:248-264` |
| Kind pushdown | yes | `query.rs:267-273` |
| Author pushdown | yes | `query.rs:275-281` |
| `since` / `until` time-window pushdown | yes | `query.rs:283-293` |
| Soft-delete exclusion | yes | `query.rs:242` |
| Offset pagination with clamped page/per-page | yes | `query.rs:224-231`, `query.rs:295-298` |
| Empty-query short-circuit (no DB roundtrip) | yes | `query.rs:217-222` |
| Input hardening (NUL scrub, 4096-char cap) | yes | `query.rs:179-194` |
| Row-decode length validation for `id`/`pubkey` | yes | `query.rs:306-311` |

#### Deliberately absent

| Not present | Evidence |
|---|---|
| Any write/index/delete path | Only SQL verb in the crate is `SELECT` (`query.rs:234`); tests do their own `INSERT`/`UPDATE` directly (`tests/fts_integration.rs:107, 130, 690`). No `INSERT`/`UPDATE`/`DELETE` in `src/`. |
| Total result count / `found` | `SearchResult` has only `hits` and `page` (`query.rs:122-127`) |
| Keyset (cursor) pagination | only `LIMIT`/`OFFSET` (`query.rs:295-298`) |
| Highlighting / snippets (`ts_headline`) | not referenced anywhere in the crate |
| Faceting, aggregation, suggestions, synonyms, stemming | `'simple'` config = no stemming/stopwords (`query.rs:143`, `173`); no other features present |
| Tag (`#p`, `#e`, `#h`, `#d`) or `ids` pushdown | `SearchQuery` has no such fields (`query.rs:73-99`); caller re-filters (`crates/buzz-relay/src/api/bridge.rs:1582-1592`) |
| Authorization / membership enforcement | none in crate; see security doc |
| Retry, timeout, circuit-breaker, tracing/metrics | no `tracing`, `tokio::time`, or retry code in `src/` |
| Typesense client | no dependency (`Cargo.toml:10-15`); only two historical mentions in doc comments (`query.rs:20`, `query.rs:46`) |

#### Completeness assessment

The query side is functionally complete for the two callers in the repo (WS NIP-50
`REQ` at `crates/buzz-relay/src/handlers/req.rs:504-611` and the HTTP `/query`
bridge at `crates/buzz-relay/src/api/bridge.rs:1628-1698`). Both use the crate as
a candidate generator and then re-fetch + re-authorize, which is the documented
contract (`lib.rs:15-18`, `query.rs:3-9`).

Two rough edges that are visible in code rather than declared as gaps:

1. `per_page` handling has a redundant intermediate binding: `per_page` is clamped
   to `1..=500` at `query.rs:224`, then `per_page_actual` re-derives the value and
   substitutes `PER_PAGE_DEFAULT` when the *raw* input was `0`
   (`query.rs:225-229`). The clamp at `224` already maps `0 → 1`, so the `0` branch
   exists only because it inspects `query.per_page` again. Net behavior: `0 → 100`,
   `1..=500 → as-is`, `>500 → 500`.
2. The `search()` doc comment sketches the SQL as `FROM events, <tsquery> AS query`
   with `ts_rank_cd(search_tsv, query)` (`query.rs:196-207`), while the emitted SQL is
   `FROM events CROSS JOIN LATERAL (SELECT <tsquery> AS query) AS search_query` with
   `ts_rank_cd(search_tsv, search_query.query)` (`query.rs:233-242`). Same semantics,
   drifted text.

#### TODO / FIXME / HACK / XXX comments

A case-insensitive grep for `TODO`, `FIXME`, `HACK`, `XXX` across the whole crate
(`Cargo.toml`, `src/**`, `tests/**`) returns **zero matches**. The only
near-miss vocabulary is descriptive, not a marker:

| Text | File:line | Note |
|---|---|---|
| `the kind:0 content-flattening hack` | `crates/buzz-search/tests/fts_integration.rs:264-265` | prose in a test doc comment describing removed legacy behavior; not a `HACK:` marker |
| `xyzzy` / `qwerty` inside test token literals | `tests/fts_integration.rs:1109`, `1270`, `1366` | test fixtures, not markers |


## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Features

### Capability inventory

| Capability | Where | Completeness |
|---|---|---|
| Append an entry to a per-community hash chain | `AuditService::log` (`crates/buzz-audit/src/service.rs:53-80`) + `log_inner` (`:82-152`) | Complete: assigns `seq`, links `prev_hash`, stamps `created_at`, computes hash, inserts in one transaction |
| Per-community write serialization across processes | `pg_advisory_lock(hashtextextended(...))` (`service.rs:56-62`, key at `:29`, `:58`) | Complete for the happy and error paths; unlock errors are discarded (`service.rs:71`) |
| Panic-safe lock release | `catch_unwind` + unconditional unlock + `resume_unwind` (`service.rs:67-79`) | Implemented; **no test exercises it** (no panic-inducing test in `service.rs:258-527`) |
| Deterministic hashing of an entry | `compute_hash` (`hash.rs:42-73`) | Complete; 9-field fixed order, presence tags, canonical JSON |
| Timestamp precision normalization | `to_storage_precision` (`hash.rs:22-24`), `log_timestamp` (`service.rs:21-23`), re-normalization in `compute_hash` (`hash.rs:47-51`) | Complete and double-enforced |
| Canonical JSON serialization | `canonical_json` (`hash.rs:80-116`) | Complete for object/array/scalar; recursion has no depth guard |
| Chain verification over a `seq` range | `verify_chain` (`service.rs:160-206`) | Partial by design: checks links + recomputed hashes inside the range; does not check range start against genesis, does not check `seq` contiguity, does not detect tail truncation |
| Paginated chain read | `get_entries` (`service.rs:212-235`) | Complete: `seq >= from_seq`, `ORDER BY seq ASC`, `LIMIT` |
| Action enum ↔ string round-trip | `as_str` (`action.rs:35-49`), `Display` (`:66-70`), `FromStr` (`:72-82`) | Complete for all 11 variants; unknown strings error |
| Error taxonomy | `AuditError` (`error.rs:12-41`) | 5 variants; see Conventions |
| Genesis sentinel | `GENESIS_HASH` (`hash.rs:9`) | Complete |

### Not implemented / absent (verified by reading, not inferred)

- **No DDL / migrations in the crate.** Stated at `lib.rs:17-18`; table lives in
  `migrations/0001_initial_schema.sql:606-619`.
- **No head/tip accessor.** There is no public `head()`/`latest_seq()`; head is read
  privately inside `log_inner` (`service.rs:94-101`).
- **No count/aggregate, no filter-by-action, no time-range query.** The only reads are
  `verify_chain` and `get_entries`, both keyed on `seq` (`service.rs:160`, `:212`).
- **No external anchoring, signing, or notarization.** No signature, HMAC, Merkle
  root, or export path exists in `crates/buzz-audit/src` (the crate's only crypto
  dependency use is `sha2` at `hash.rs:2`).
- **No retention, archival, or pruning.** No `DELETE`/`TRUNCATE` statement in the
  crate.
- **No batch append.** `log` takes a single `NewAuditEntry` (`service.rs:53`).
- **No retry/backoff on DB failure.** Errors are returned to the caller; the relay
  worker logs and drops them (`crates/buzz-relay/src/state.rs:1199-1207`).
- **No cargo features.** `crates/buzz-audit/Cargo.toml` has no `[features]` section.
- **No integration/`tests/` directory.** All tests are inline `#[cfg(test)]` modules.

### Actions defined vs. actions actually produced

11 variants are defined (`action.rs:8-31`), but repo-wide grep for `AuditAction::`
outside this crate finds only two producers:

| Action | Produced at |
|---|---|
| `EventCreated` | `crates/buzz-relay/src/handlers/event.rs:583` |
| `MediaUploaded` | `crates/buzz-relay/src/api/media.rs:428` |

The remaining 9 (`EventDeleted`, `ChannelCreated`, `ChannelUpdated`, `ChannelDeleted`,
`MemberAdded`, `MemberRemoved`, `AuthSuccess`, `AuthFailure`, `RateLimitExceeded`) have
no production call site in the repo; they appear only in this crate's own tests
(e.g. `service.rs:352`, `:356`, `:452`, `:455`).

### TODO / FIXME / HACK / XXX comments

A case-insensitive search for `todo`, `fixme`, `hack`, `xxx`, `unimplemented!`,
`todo!` across `crates/buzz-audit/src` returns **zero matches**. There are no
deferred-work markers in the crate.

### Documentation-vs-code drift found while reading

| ARCHITECTURE.md claim (line) | Code reality |
|---|---|
| "10 audit actions" (`ARCHITECTURE.md:503`) | 11 variants (`action.rs:8-31`); `MediaUploaded` is missing from the doc list |
| Hash covers `event_id`, `event_kind`, `channel_id` (`ARCHITECTURE.md:501`) | Those fields do not exist on `AuditEntry` (`entry.rs:14-37`); pre-image has `community_id`, `seq`, `created_at`, `action`, `actor_pubkey`, `object_id`, `detail`, `prev_hash` (`hash.rs:45-71`) |
| `AuditError::AuthEventForbidden` returned for `KIND_AUTH` (`ARCHITECTURE.md:505`) | No such variant (`error.rs:12-41`); the refusal is in the relay ingest path (`crates/buzz-relay/src/handlers/ingest.rs:1464-1468`) |
| `GENESIS_HASH` is "64 zeros" (`ARCHITECTURE.md:497`) | `[u8; 32]` of zeros, i.e. 32 bytes = 64 zero hex chars (`hash.rs:9`) |
| "`pg_advisory_lock` before each transaction" implying one lock (`ARCHITECTURE.md:501`) | Lock key is per-community (`service.rs:29`, `:58`) |


## Module: buzz-media (`crates/buzz-media`)

### Aspect: Features

Completeness is judged against what the code does, not against the Blossom spec's full endpoint set.

| Feature | Completeness | Evidence |
|---|---|---|
| Content-addressed blob upload (image path) — sniff, validate, hash, store, thumbnail, blurhash, sidecar | Full | `crates/buzz-media/src/upload.rs:207-236`, `crates/buzz-media/src/thumbnail.rs:15-50` |
| Generic-file upload path (documents, archives, text, data) | Full | `crates/buzz-media/src/upload.rs:245-274`, `crates/buzz-media/src/validation.rs:159-209` |
| Streaming video upload (never fully buffered in RAM) | Full | `crates/buzz-media/src/upload.rs:292-511` |
| Server-side SHA-256 + `x`-tag hash verification | Full | `crates/buzz-media/src/upload.rs:84-86`, `crates/buzz-media/src/auth.rs:190-196` |
| Blossom kind-24242 auth verification (upload + get verbs) | Full for the two verbs implemented; `delete`/`list`/`mirror` verbs absent | `crates/buzz-media/src/auth.rs:6-10`, `crates/buzz-media/src/auth.rs:31-236` |
| Per-community sidecar metadata as tenant read gate | Full | `crates/buzz-media/src/storage.rs:177-221` |
| Idempotent re-upload short-circuit | Full | `crates/buzz-media/src/upload.rs:97-128`, `crates/buzz-media/src/upload.rs:424-453` |
| Thumbnail generation (320px JPEG) + 4×3 blurhash | Full for images; **none for video** ("no thumbnail — desktop handles that") | `crates/buzz-media/src/thumbnail.rs:29-38`, `crates/buzz-media/src/upload.rs:437-448` |
| Metadata/EXIF policy enforcement for JPEG/PNG/WebP/GIF/MP4 | Full as a *rejection* policy; no stripping/sanitizing is performed server-side | `crates/buzz-media/src/validation.rs:487-928` |
| Image-bomb protection (25 MP pre-decode cap, fail-closed on unparseable geometry) | Full | `crates/buzz-media/src/validation.rs:264-273` |
| MP4 structural validation (fast-start, codec, duration, resolution, box allowlist) | Full | `crates/buzz-media/src/validation.rs:289-395`, `crates/buzz-media/src/validation.rs:408-484`, `crates/buzz-media/src/validation.rs:831-928` |
| S3 range reads and streaming reads (HTTP 206 / large video serving support) | Full | `crates/buzz-media/src/storage.rs:118-146` |
| HEAD with size metadata | Partial — only `content_length` is surfaced; no ETag, last-modified, or content-type | `crates/buzz-media/src/storage.rs:167-175`, `crates/buzz-media/src/storage.rs:379-381` |
| Per-upload moderation records (`_uploads/` side channel) with opt-in IP/port capture | Full, off by default | `crates/buzz-media/src/upload_record.rs:139-199`, `crates/buzz-media/src/config.rs:45-61` |
| Bucket key taxonomy + storage-accounting sweep (physical/logical/orphan/anomaly gauges) | Full as a pure fold; the S3 driver is one page-listing method and the scheduling lives in the relay | `crates/buzz-media/src/bucket_index.rs:54-411`, `crates/buzz-media/src/storage.rs:242-265` |
| AWS credential-chain support (IRSA / Pod Identity) alongside static keys | Full | `crates/buzz-media/src/storage.rs:25-65` |
| Blob deletion / GC / orphan sweeping (actual deletes) | **Stubbed** — only a raw `delete(key)` primitive; no GC job, no cascade, no retention | `crates/buzz-media/src/storage.rs:158-164`, `crates/buzz-media/src/upload.rs:122-127` |
| Blossom BUD-02 `/list`, `DELETE /{sha256}`, BUD-04 mirror | **Absent** in this crate | no such fn in `crates/buzz-media/src/lib.rs:17-28`; relay routes only upload/get/head (`crates/buzz-relay/src/router.rs:38-45`) |
| Signed/presigned URL generation | **Absent** — no `presign_*` call anywhere | `crates/buzz-media/src/storage.rs:73-265` |
| Multipart upload | **Absent** — `put_object_stream_with_content_type` is used; no explicit multipart API | `crates/buzz-media/src/storage.rs:96-101` |
| Audio support | Deliberately **not supported**: sniffed `audio/*` is rejected on the generic path "until Buzz has an explicit sanitizer and location-metadata validator for its container" | `crates/buzz-media/src/validation.rs:186-196` |
| PDF inline preview | Deliberately deferred — "inline PDF preview is a planned fast-follow" | `crates/buzz-media/src/validation.rs:211-215` |
| Transcoding / re-encoding of stored media | **Absent** by design (validation rejects non-canonical inputs instead) | `crates/buzz-media/src/validation.rs:487-491` |

---

### TODO / FIXME / HACK / XXX comments

**None.** A repo-relative scan of all 12 `.rs` files in the crate returns zero matches for `TODO`, `FIXME`, `HACK`, or `XXX` (verified across `crates/buzz-media/src/*.rs` and `crates/buzz-media/tests/static_creds_minio.rs`).

Deferred work is instead recorded in prose comments. The forward-looking ones, quoted verbatim:

- "A V2 background GC job can sweep blobs with no matching sidecar after a grace period." — `crates/buzz-media/src/upload.rs:126-127`
- "PDF is intentionally *not* inline yet — inline PDF preview is a planned fast-follow; until the renderer handles it, force download like any other file." — `crates/buzz-media/src/validation.rs:213-215`
- "audio is rejected until Buzz has an explicit sanitizer and location-metadata validator for its container." — `crates/buzz-media/src/validation.rs:188-190`
- "Revert to crates.io once #449 lands upstream." (workspace-level, about the `aws-creds` fork this crate depends on) — `Cargo.toml:169`

For contrast, the adjacent relay handler *does* carry a TODO for the quota gap this crate does not cover: "TODO(v2): Add persistent per-pubkey storage quotas. Admission limits below bound active parser/storage work, but they do not cap durable bytes stored." — `crates/buzz-relay/src/api/media.rs:302-304`.


## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Features

---

### 1. Capabilities

| Capability | Where | Status |
|---|---|---|
| YAML → validated `WorkflowDef` → canonical JSON | `schema.rs:262-268`, `lib.rs:163-165` | full |
| Definition validation (name, steps, step-id charset/uniqueness, cron/interval invariants) | `schema.rs:152-229` | full |
| Event-driven trigger matching with cache | `lib.rs:276-383` | full |
| Cron + interval scheduling with cross-pod at-most-once claim | `lib.rs:428-672` | full |
| Sequential step execution with per-step timeout | `executor.rs:1083-1217` | full |
| `if:` conditions via evalexpr with 4 custom string helpers | `executor.rs:224-384` | full |
| Template substitution with 2 filters | `executor.rs:70-201` | full (single-pass only) |
| Execution trace persistence + partial-progress on failure | `executor.rs:1108-1204`, `error.rs:9-15`, `lib.rs:175-263` | full |
| Concurrency admission control | `executor.rs:978-983`, `lib.rs:110-111` | full |
| Approval-gate suspension + resume plumbing | `executor.rs:650-668`, `executor.rs:1018-1075` | partial — token is generated but never persisted; runs end `Failed` |
| Side-effect abstraction (`ActionSink`) | `action_sink.rs:48-70` | partial — only `send_message` exists on the trait |
| Outbound HTTP with SSRF protection | `executor.rs:745-871` | full, gated behind the `reqwest` feature |

---

### 2. Trigger completeness

| Trigger | Status | Evidence |
|---|---|---|
| `message_posted` | **full** — kind-9 gate + optional evalexpr filter | `lib.rs:958`, `lib.rs:824-846` |
| `reaction_added` | **full** — kind-7 gate + exact emoji equality (no shortcode/unicode normalization) | `lib.rs:959`, `lib.rs:807-822` |
| `diff_posted` | **full** — kind-40008 gate + optional filter | `lib.rs:960`, `lib.rs:848-880` |
| `schedule` (`cron`) | **full** — window-based match, deterministic claim anchor | `lib.rs:526-534`, `lib.rs:688-706` |
| `schedule` (`interval`) | **full** — bucket-quantized anchor, restart-safe prefilter, ≥ 60 s enforced at parse | `lib.rs:535-538`, `lib.rs:719-797`, `schema.rs:212-225` |
| `webhook` | **schema + relay-side only** — this crate never fires it; the entry point is the relay's `POST /hooks/{id}` handler, which builds the `TriggerContext` and calls `execute_from_step` directly | `lib.rs:962`, `crates/buzz-relay/src/router.rs:120`, `crates/buzz-relay/src/api/bridge.rs:1777-1911` |

---

### 3. Action completeness (precise, as of the code read)

| Action | Status | Detail |
|---|---|---|
| `send_message` | **full** (requires a wired `ActionSink`) | Community-scoped run/workflow lookup, destination validation, owner attribution, real event emission through `RelayActionSink` (`executor.rs:530-578`; `crates/buzz-relay/src/workflow_sink.rs:159`). Without `set_action_sink` it fails with `InvalidDefinition` (`lib.rs:148-156`). |
| `send_dm` | **stubbed** | Returns `WorkflowError::NotImplemented("SendDm")` — the step and therefore the run fails (`executor.rs:580-584`). |
| `set_channel_topic` | **stubbed** | Returns `WorkflowError::NotImplemented("SetChannelTopic")` (`executor.rs:586-590`). |
| `add_reaction` | **partial / non-functional against the current relay** | Code path is complete under `feature = "reqwest"` (which the relay enables, `crates/buzz-relay/Cargo.toml:63`) but targets `POST {BUZZ_RELAY_BASE_URL}/api/messages/{message_id}/reactions` (`executor.rs:888-894`). No such route is registered in the relay router (`crates/buzz-relay/src/router.rs:39-125` — verified by grep for `reactions` and `/api/messages`), so the call returns a non-success status and the step fails with `WebhookError` (`executor.rs:914-922`). Without the `reqwest` feature it silently returns `{added:false, skipped:true}` (`executor.rs:606-616`). |
| `call_webhook` | **full** under `feature = "reqwest"` | SSRF guard, DNS pinning, redirects disabled, 10 s timeout, 1 MiB cap (`executor.rs:781-869`). Without the feature it is a no-op returning `{status:0, body:null, skipped:true}` (`executor.rs:636-647`). |
| `request_approval` | **partial / effectively non-functional** | Generates a token and suspends, but nothing persists an approval record or emits kind 46010 (`executor.rs:650-668`); `finalize_run` converts the suspension into `RunStatus::Failed` (`lib.rs:192-215`). |
| `delay` | **full, bounded** | Max 270 s; longer delays fail the step (`executor.rs:671-690`). |

Header note: the module doc comment still claims "Action dispatch uses placeholder implementations that log intent. Real event emission is wired in WF-07/08" (`executor.rs:9-10`) and `dispatch_action`'s doc says "For MVP, most actions log their intent and return a success output" (`executor.rs:521-522`). Both are stale relative to the `send_message`/`call_webhook` implementations.

---

### 4. Feature-flag effects

`feature = "reqwest"` (`Cargo.toml:28-29`) toggles `add_reaction` and `call_webhook` between real HTTP and skip-stubs. `buzz-relay` enables it (`crates/buzz-relay/Cargo.toml:63`); `buzz-admin` depends on the crate without it (`crates/buzz-admin/Cargo.toml:21`), so an admin-side build compiles the skip-stubs.

---

### 5. All TODO/FIXME/HACK/XXX comments (verbatim)

Repo-grep over `crates/buzz-workflow/` found exactly three markers, all `TODO`:

| Marker | Verbatim text | Location |
|---|---|---|
| TODO | `// TODO (WF-07): emit DM event.` | `crates/buzz-workflow/src/executor.rs:582` |
| TODO | `// TODO (WF-07): update channel topic via DB.` | `crates/buzz-workflow/src/executor.rs:588` |
| TODO | `// TODO (WF-08): create approval record in DB, emit kind:46010.` | `crates/buzz-workflow/src/executor.rs:663` |

No `FIXME`, `HACK`, or `XXX` comments exist in the crate.

Adjacent in-code work markers that are not TODO-tagged but state incomplete behaviour:
- `"// Approval gates are not yet implemented (WF-08). / Fail explicitly rather than creating unreachable WaitingApproval rows."` (`lib.rs:192-193`).
- `"Long delays (hours/days) should use the scheduled resume pattern (future work: WF-09)."` (`executor.rs:674-675`).
- `"In-memory only — lost on restart. Missed fires during downtime are not replayed (acceptable for MVP)."` (`lib.rs:85-86`).
- Numbered "Fix N" comments referencing prior review rounds: Fix 4 (`schema.rs:218`), Fix 2 (`lib.rs:465`), Fix 5 (`lib.rs:572`), Fix 6 (`lib.rs:638`), Fix 1 (`lib.rs:664`).


## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Features

---

#### 1. What bootstrap enables

`main()` (`main.rs:83-1060`) is the single composition root. Everything the relay can do is switched on here.

| Capability | Enabled by | Gate |
|-----------|-----------|------|
| NIP-01 WebSocket relay | `build_router` (`router.rs:32`) + `handle_connection` via `router.rs:316` | always |
| NIP-11 relay info (content-negotiated + `/info`) | `nip11.rs:235`, `router.rs:63-64` | always, unauthenticated |
| NIP-05 discovery | `router.rs:65` | always |
| Nostr-over-HTTP bridge (`/events`, `/query`, `/count`) | `router.rs:71-73` | always, NIP-98 |
| Blossom media upload/download | `media_router` (`router.rs:37-45`) | always; GET auth off by default (`config.rs:197`) |
| Git smart HTTP | `git_router` (`api/git/transport.rs:1756`) | always, but gated behind the fatal A3 probe (`main.rs:466-503`) |
| Git policy hook callback | `git_policy_router` (`api/git/mod.rs:60`) | always, localhost-only |
| Huddle audio WebSocket | `router.rs:125-128` | `huddle_audio_available` (default `true`, `config.rs:489-491`) checked at `audio/handler.rs:357` |
| Relay invites + join policy | `router.rs:95-111` | join-policy routes return content only when `config.join_policy.is_some()` (`config.rs:793-810`) |
| Operator community provisioning | `router.rs:74-93` | `relay_operator_pubkeys` non-empty (fail-closed default: disabled, `config.rs:161-169`) |
| Deployment-admin API + SPA | `router.rs:47-54` | `BUZZ_ADMIN_HOST` set (`config.rs:813-838`) |
| Public web bundle (invite landing / git browser) | SPA fallback `router.rs:145-183` | `BUZZ_WEB_DIR` set; git-browser routes additionally need `BUZZ_SERVE_GIT_WEB_GUI` |
| Moderation queue reads | `router.rs:113-118` | always, NIP-98 + mod-authz |
| Workflow webhooks | `router.rs:120` | always, secret-authenticated |
| Prometheus `/metrics` | `metrics::install` (`main.rs:138`) | always |
| Distributed tracing (OTLP) | `telemetry::try_init_tracer` (`main.rs:100`) | `OTEL_EXPORTER_OTLP_ENDPOINT` set (`telemetry.rs:80-83`) |
| Tamper-evident audit log | separate PG pool + worker (`main.rs:322-334`, `state.rs:653-690`) | `BUZZ_AUDIT_ENABLED` (default `true`, `config.rs:793`) |
| Inter-relay QUIC mesh | `mesh_boot::boot_mesh` (`main.rs:442`) | `BUZZ_MESH=on` (default off, `config.rs:492-509`) |
| Mesh reliable-stream echo probe | `handle.wire_consumers(..., mesh_demo_echo, ...)` (`main.rs:455`) + `router.rs:123` | `BUZZ_MESH_DEMO_ECHO=on` (`config.rs:514-518`) |
| NIP-PL push (matcher + delivery worker) | `main.rs:686-691` | `push_gateway_delivery_url.is_some()` — **on by default** (`config.rs:752-758`) |
| NIP-ER reminder scheduler | `main.rs:693-802` | always |
| NIP-43 relay membership enforcement + snapshot reconciliation | `main.rs:505-546` | `BUZZ_REQUIRE_RELAY_MEMBERSHIP` (default false) |
| NIP-OA owner-attested agent auth | `api/mod.rs:81` | `BUZZ_ALLOW_NIP_OA_AUTH` (default false, `config.rs:520-523`) |
| Ephemeral channel reaping | `main.rs:602-684` | always |
| Multi-node fan-out / cache invalidation / conn-control | `main.rs:350-367` (subscribers) + `main.rs:804-936` (consumers) | always |
| Community lifecycle durable revalidation | `main.rs:869-890` | always |
| Usage + storage metrics polling | `main.rs:987-1041` | always; per-community series gated by `BUZZ_USAGE_METRICS_PER_COMMUNITY`, storage sweep by `BUZZ_STORAGE_METRICS` (`storage_sweep.rs:69`) |
| Postgres FTS search | `main.rs:369-386` | always |
| Replica read routing behind a freshness fence | `main.rs:177-198` | `READ_DATABASE_URL` set + probe verifies the floor guard |
| Runtime conformance tracing | `state.rs:797` | production always binds `NoopTracer` (zero cost); only tests swap in `JsonlTracer` |
| UDS listener (service mesh sidecar) | `main.rs:1163-1206` | `BUZZ_UDS_PATH` set, unix only |
| Auto migrations | `main.rs:161-172` | `BUZZ_AUTO_MIGRATE` (default off) |
| Channel-event reconciliation (dev/CI) | `main.rs:547-590` | `BUZZ_RECONCILE_CHANNELS` present |
| Dev relay identity | `main.rs:396-408` | `BUZZ_RELAY_PRIVATE_KEY` unset **and** `!require_auth_token` |

#### 2. Background tasks — every `tokio::spawn`

**23 production spawn sites** in this file group (19 in `main.rs`, 3 in `state.rs`, 1 in `metrics.rs`). One additional spawn at `main.rs:1834` is inside `#[cfg(test)]`. `mesh_boot::boot_mesh` / `wire_consumers` spawn further tasks outside this group.

| # | Task | Spawn | Cadence / trigger | On error | Cancellable? |
|---|------|-------|-------------------|----------|--------------|
| 1 | Prometheus exporter HTTP server | `metrics.rs:146` | event-driven | none (future returns) | no |
| 2 | Audit worker | `state.rs:658` | event-driven on `audit_tx` | per-entry error → `buzz_audit_log_errors_total` + `error!` (`state.rs:1200-1206`); worker keeps going | **yes** — `AuditShutdownHandle` (`state.rs:1182-1196`), 5 s drain from `main.rs:1049` |
| 3 | Cache-invalidation publisher (fire-and-forget, per invalidation) | `state.rs:968` | per `invalidate_*` call | `warn!` and swallow; backstopped by ≤10 s TTL (`state.rs:960-963`) | no |
| 4 | Conn-control publisher (ban fan-out, per ban) | `state.rs:1043` | per ban | `warn!` and swallow; DB ban row is the durable backstop (`state.rs:1046-1050`) | no |
| 5 | Redis pub/sub event subscriber | `main.rs:354` | continuous | inside `run_subscriber` | no |
| 6 | Redis cache-invalidation subscriber | `main.rs:360` | continuous | inside | no |
| 7 | Redis conn-control subscriber | `main.rs:366` | continuous | inside | no |
| 8 | NIP-43 membership snapshot reconciler | `main.rs:522` | `BUZZ_NIP43_RECONCILE_INTERVAL_SECS`, default 60 s, `.max(1)`; first tick consumed immediately (`main.rs:524`) | `warn!`, loop continues | no |
| 9 | Channel-event reconciler (dev/CI) | `main.rs:551` | 24 attempts, 5 s apart (≈2 min) then exits | `warn!` per attempt | no |
| 10 | Workflow cron loop | `main.rs:599` | inside `WorkflowEngine::run` | inside | no |
| 11 | Ephemeral channel reaper | `main.rs:613` | `BUZZ_REAPER_INTERVAL_SECS`, default 60 s | tick error → `error!` + `continue`; per-channel side-effect errors logged individually (`main.rs:657/667`) | no |
| 12 | NIP-PL matcher | `main.rs:687` | inside `push_runtime::run_matcher` | inside | no |
| 13 | NIP-PL delivery worker | `main.rs:688` | inside `push_runtime::run_delivery_worker` | inside | no |
| 14 | NIP-ER reminder scheduler | `main.rs:709` | `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS`, default 10 s; batch `SPROUT_REMINDER_SCHEDULER_BATCH_LIMIT`, default 100 | query error → `error!` + `continue`; claim error → `warn!` + skip; publish error → release the claim (compare-and-clear on a per-attempt stamp), release failure → `warn!` **and the reminder stays claimed forever** (`main.rs:781-798`) | no |
| 15 | Multi-node fan-out consumer | `main.rs:823` | continuous `broadcast::recv` | `Lagged(n)` → `buzz_multinode_fanout_lag_total += n` + `warn!`; `Closed` → `error!` + **break (task dies, fan-out silently stops)** | no |
| 16 | Cache-invalidation consumer | `main.rs:853` | continuous | `Lagged` → counter + `warn!`; `Closed` → `error!` + break | no |
| 17 | Community lifecycle revalidator | `main.rs:888` | `BUZZ_COMMUNITY_REVALIDATE_INTERVAL_SECS`, default 30 s, `.clamp(1,300)` | per-community failure retains sockets, logged `warn!` (`state.rs:1082-1084`) | **yes** — `community_revalidator_cancel`, `main.rs:1045`; exits without waiting for the next tick (test `main.rs:1827-1849`) |
| 18 | Conn-control consumer | `main.rs:904` | continuous | `Lagged` → counter + `warn!`; `Closed` → `error!` + break | no |
| 19 | Pool metrics poller | `main.rs:950` | `BUZZ_POOL_METRICS_INTERVAL_SECS`, default 10 s, `.max(1)` | none — no fallible call in the loop | no |
| 20 | Usage metrics poller (+ storage sweep trigger) | `main.rs:1008` | `BUZZ_USAGE_METRICS_INTERVAL_SECS`, default 300 s, `.max(5)`; first tick jittered `rand % interval` | tick error → `error!` "skipping"; leader demotes | no |
| 21 | Health listener server | `main.rs:1122` | event-driven | `.ok()` — errors swallowed | no (deliberate, BR-RC-72) |
| 22 | Shutdown watchdog | `main.rs:1132` | one-shot on signal | ends in `std::process::exit(1)` after 30 s | no |
| 23 | UDS server (unix + `BUZZ_UDS_PATH`) | `main.rs:1181` | event-driven | `.ok()` | via `shutdown_tx` watch, then `abort()` (`main.rs:1197`) |
| — | Storage sweep body | `storage_sweep::maybe_spawn_sweep` called `main.rs:1456` | single-flight, `BUZZ_STORAGE_SWEEP_INTERVAL_SECS` | inside `storage_sweep` | no |

##### Failure-mode observations
- **Only 2 of 23 tasks are cancellable** (#2 audit, #17 revalidator). The other 21 are abandoned at process exit.
- Tasks #15, #16, #18 `break` out of their loop on `RecvError::Closed`. After that the pod keeps serving traffic but silently loses multi-node fan-out, cross-pod cache invalidation, or ban propagation respectively. There is no restart, no health-check coupling, and no gauge for "consumer alive" — only an `error!` line (`main.rs:838-841`, `main.rs:868-871`, `main.rs:929-932`).
- Task #14's release-failure branch (`main.rs:789-797`) leaves a reminder permanently claimed and never retried; the log explicitly says so.
- No task uses `JoinHandle` supervision or panic capture except #2 (`state.rs:1188` reports `Audit worker panicked`).

#### 3. Concurrency budget features

| Semaphore | Field | Bound | Default | Acquisition |
|-----------|-------|-------|---------|-------------|
| Connections | `conn_semaphore` (`state.rs:514`) | `max_connections` | 10,000 (`config.rs:449-452`) | `try_acquire_owned` — instant refusal |
| Handlers | `handler_semaphore` (`state.rs:516`) | `max_concurrent_handlers` | 1,024 (`config.rs:454-457`) | `try_acquire_owned` |
| Git subprocesses | `git_semaphore` (`state.rs:521`) | `git_max_concurrent_ops` | 20 (`config.rs:735-738`) | `api/git/transport.rs:322` |
| Media uploads | `media_upload_semaphore` (`state.rs:523`) | `media_max_concurrent_uploads` | 8 (`config.rs:663-668`) | `api/media.rs:119` |

`git_semaphore`'s doc is explicit that it bounds resources and is **not** writer serialization — that is the manifest-pointer CAS (`state.rs:517-520`).

#### 4. Observability features surfaced by this module

- **Framework HTTP metrics**: `http_requests_total`, `http_request_latency_ms` with `{code, caller, action}` (`metrics.rs:200-205`). `action` is the axum `MatchedPath` — never the raw URI, deliberately, to avoid unbounded cardinality from scanners (`metrics.rs:166-179`).
- **Per-metric histogram buckets**: 5 bucket families registered at `metrics.rs:76-141` — HTTP latency ms (11 buckets, `metrics.rs:37-39`), generic `_seconds` suffix (10, `metrics.rs:42`), git durations (13, `metrics.rs:45-47`), git bytes (9, `metrics.rs:50-60`), git pack counts (9, `metrics.rs:63`), fan-out recipients (9, `metrics.rs:66`).
- **Gauge idle-timeout**: `MetricKindMask::GAUGE` with the BR-RC-06 timeout (`metrics.rs:75-78`) so intentionally-stopped series disappear.
- **Pool gauges** (`main.rs:955-985`): `buzz_db_pool_{size,idle,active,max}`, `buzz_db_read_pool_{size,idle,active,max}`, `buzz_db_replica_fence_{open,lag_seconds}`, `buzz_redis_pool_{available,size,max,waiting}`. **The audit pool (`main.rs:325-329`) and the search pool (`main.rs:378-382`) are not instrumented at all.**
- **Usage gauges** (`main.rs:1481-1806`): fleet totals `buzz_total_{ws_connections,users_online_pod,subscriptions,users,channels,messages,relay_members,workflows,git_repos,active_users,active_channels}` + `buzz_communities_total`; per-community `buzz_community_{ws_connections,users_online_pod,subscriptions,users,channels,messages,relay_members,workflows,git_repos,active_users,active_channels}`.
- **Leader gauge**: `buzz_usage_poller_is_leader` (`main.rs:1032-1036`).
- **Lag counters**: `buzz_multinode_fanout_lag_total` (`main.rs:836`), `buzz_cache_invalidation_lag_total` (`main.rs:866`), `buzz_conn_control_lag_total` (`main.rs:927`).
- **Audit metrics**: `buzz_audit_enabled` gauge (`main.rs:139`), `buzz_audit_log_errors_total`, `buzz_audit_log_seconds` (`state.rs:1201-1205`).
- **Cache-hit metrics**: `buzz_membership_cache_{hits,misses}_total` (`state.rs:833/836`), `buzz_accessible_channels_cache_{hits,misses}_total` (`state.rs:1095/1098`). The other five caches have **no** hit/miss instrumentation.
- **Backpressure**: `buzz_ws_backpressure_disconnects_total` (`state.rs:466`).
- **Tracing**: JSON-flattened stdout always (`main.rs:109`); OTLP gRPC layer attached only when `OTEL_EXPORTER_OTLP_ENDPOINT` is set (`main.rs:101-107`, `telemetry.rs:79-90`), tracer name `"buzz-relay"` (`main.rs:104`).

#### 5. Feature flags / gates summary

Cargo features (`Cargo.toml:80-81`): exactly one — `dev = ["buzz-auth/dev"]`. No `#[cfg(feature = ...)]` in this file group; the flag only forwards dev key derivation into `buzz-auth`.

Compile-time cfg used: `#[cfg(unix)]` / `#[cfg(not(unix))]` for UDS (`main.rs:1162`, `main.rs:1207`) and the SIGTERM handler (`main.rs:1227`, `main.rs:1236`); `#[cfg(unix)]` on one config test (`config.rs:1329`).

Runtime kill switches that produce **byte-identical off behaviour** (explicit design intent, verified):
- `BUZZ_MESH` off ⇒ no UDP bind, no Redis registry write, no spawn (`config.rs:492-497`, `main.rs:437-441`).
- `BUZZ_MESH_DEMO_ECHO` off ⇒ inbound reliable streams accepted, logged, closed (`config.rs:139-143`).
- `EmissionScope::Off` ⇒ no per-community series; totals unchanged (`main.rs:76-78`).
- Storage sweep disabled ⇒ zero storage-family gauges including health (`main.rs:1436-1453`).
- Audit disabled ⇒ `audit_tx = None`, worker parked on cancellation (`state.rs:659-663`, `state.rs:763`).


## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Features

---

#### 1. Capabilities delivered over WebSocket

| Capability | NIP | Implemented at | Notes |
|---|---|---|---|
| Relay-initiated challenge/response auth | NIP-42 | `connection.rs:182-186` + `handlers/auth.rs:44-296` | mandatory within 5 s (`connection.rs:27`) |
| Owner-attestation agent auth (agent→owner delegation) | NIP-OA | `auth.rs:28-42`, `:224-281` | tag extracted from the signed AUTH event, so it is signature-protected |
| Signed event submission | NIP-01 | `event.rs:585-736` | delegates persistent kinds to the shared ingest seam |
| Ephemeral event relay (20000–29999), never stored | NIP-16 range | `event.rs:762-897` | WS-only; HTTP explicitly rejects 1059 and 20001 (`ingest.rs:1448-1452`) |
| Presence set/clear | Buzz kind 20001 | `event.rs:772-800` | Redis-backed; falls through to global ephemeral fan-out |
| Encrypted agent observer frames (bidirectional) | Buzz kind 24200 | `event.rs:920-1069` | NIP-44 content, structural direction inference, own rate limiter |
| Filter subscriptions with historical backfill + EOSE | NIP-01 | `req.rs:44-417` | one DB query per filter, bounded concurrency 4 |
| Full-text search subscriptions | NIP-50 | `req.rs:505-727` | one-shot (no fan-out registration), paginated |
| Aggregate counts | NIP-45 | `count.rs:30-286` | exact-or-refuse (5000-candidate budget); since `ab3af828` any 30175-matching filter is forced off the exact SQL path (`count.rs:110`, `:174`, `:243`) |
| Subscription cancellation | NIP-01 | `close.rs:12-30` | idempotent |
| Channel (group) scoping via `#h` | NIP-29 | `req.rs:1009-1036`, `count.rs:17-26`, `subscription.rs:265-330` | no dedicated message types |
| Gift-wrapped DMs | NIP-17/59 | pubkey-mismatch exemption `event.rs:636-645`; read gate `req.rs:1038-1071` | WS-only ingest |
| Author-only-unless-shared persona catalog | NIP-AP (kind 30175) | gate `req.rs:388`, `:705`, `count.rs:202`/`:271`, fan-out `event.rs:154-175`; helper `req.rs:1222-1234` | **added by `ab3af828`**; SQL pre-filter `crates/buzz-db/src/event.rs:504-527`; ingest shape rule `ingest.rs:1042-1060` |
| Live cross-node delivery | — | `event.rs:282-332` ← `main.rs:818-845` | Redis pub/sub fan-in |
| Heartbeat / dead-peer detection | RFC 6455 | `connection.rs:378-405` | 30 s ping, 3 misses |
| Slow-client shedding | — | `connection.rs:88-113`, `state.rs:436-475` | 15-strike grace, shared counter |
| Live ban enforcement on open sockets | — | `state.rs:290-325` (called from moderation) + `auth.rs:105-183` | frame-then-close idiom |
| Graceful-restart signalling | RFC 6455 1012 | `state.rs:352-374` | sticky drain flag |
| Community archival closing live sockets | — | `state.rs:99-176`, `connection.rs:132-140` | plus a periodic durable revalidation backstop |
| Runtime conformance tracing on the read seam | — | `req.rs:110-116`, `:141-148`, `:332-366`, `:626-668` | `NoopTracer` in production (`state.rs:798`) |

---

#### 2. Realtime delivery path, end to end

##### 2.1 Ingest → fan-out (persistent event, same pod)

```
client ──["EVENT",e]──▶ recv_loop                     connection.rs:419-436
                        ├─ frame size check           connection.rs:421-434
                        ├─ ClientMessage::parse       protocol.rs:66-76
                        ├─ enforce_ws_admission       connection.rs:498-500 → :594-653
                        │    · WsEvents  (5 s window)
                        │    · Messages  (60 s window)
                        ├─ handler_semaphore permit   connection.rs:513-521
                        └─ tokio::spawn(handle_event) connection.rs:530-536
handle_event                                          event.rs:608
  ├─ auth read                                        event.rs:611-631
  ├─ pubkey match (1059 exempt)                       event.rs:636-645
  ├─ kind 22242 reject                                event.rs:647-655
  ├─ kind 24200 → observer branch                     event.rs:657-669
  ├─ ephemeral → ephemeral branch                     event.rs:675-696
  └─ ingest_event(state, tenant, event, IngestAuth)   event.rs:728
       └─ (verify, membership, DB insert, …)          handlers/ingest.rs:1393+
       └─ dispatch_persistent_event                   event.rs:349
            ├─ AWAITED: enqueue_event_created_audit   event.rs:358-366  (bounded chan, cap 1000)
            └─ spawn dispatch_persistent_event_inner  event.rs:374-393
                 ├─ mark_local_event                  event.rs:417
                 ├─ pubsub.publish_event(topic)       event.rs:418-427
                 ├─ sub_registry.fan_out_scoped       event.rs:429-431
                 ├─ filter_fanout_by_access(threaded) event.rs:432-439
                 │    └─ incl. kind-30175 shared gate event.rs:154-175
                 ├─ DM-visibility owner fence         event.rs:457-491
                 ├─ frame cache + send_to_text_bytes  event.rs:492-493
                 └─ spawn workflow_engine.on_event    event.rs:533-557
◀──["OK",id,true,""]── conn.send                      event.rs:719-723
◀──["EVENT",sub,e]──── per matching subscription      event.rs:76-98
```

The `OK` and the `EVENT` frames are on **independent** timelines: the `OK` is emitted as soon as `dispatch_persistent_event` returns (which is after the audit enqueue only), while fan-out happens in a spawned task (`event.rs:374`). A client can therefore observe its own event arriving on a subscription **before or after** its `OK`.

##### 2.2 Ingest → fan-out (ephemeral / observer, same pod)

Shorter: verify on `spawn_blocking` → optional membership check → `mark_local_event` → `publish_event` → `fan_out_event_to_local_subscribers` → `OK true`.

- channel-scoped ephemeral: `event.rs:831-865` (topic `Channel(ch)`)
- channel-less ephemeral: `event.rs:866-894` (topic `Global`)
- observer frame: `event.rs:1069-1089` (topic `Global`, always)

`fan_out_event_to_local_subscribers` (`event.rs:241-279`) is the canonical guarded local send: `fan_out_scoped` → `filter_fanout_by_access(…, None)` → serialise once → frame cache → `send_to_text_bytes`. It has six other production callers outside this group (`audio/handler.rs:1318`, `api/git/transport.rs:1703`, `side_effects.rs:803`/`:889`/`:2759`/`:2911`) — i.e. huddle audio events, git push notifications, and NIP-29/NIP-25 side effects all reach WS subscribers through this same gate.

##### 2.3 Where cross-node Redis events re-enter

Single re-entry point:

```
another pod ──▶ Redis PSUBSCRIBE buzz:{community}:{topic}
                  └─ buzz-pubsub subscriber task
                       └─ broadcast → pubsub.subscribe_local()
main.rs:818-845   loop { rx.recv() → fan_out_pubsub_event(&state, ev) }
event.rs:282-332  fan_out_pubsub_event
                    ├─ topic → Option<Uuid>              event.rs:287-290
                    ├─ StoredEvent::new(event, ch)       event.rs:292
                    ├─ local-echo dedup (community,id)   event.rs:295-304   ← consume-on-read
                    ├─ sub_registry.fan_out_scoped       event.rs:306
                    ├─ filter_fanout_by_access(…, None)  event.rs:307
                    ├─ buzz_multinode_fanout_total       event.rs:308
                    └─ frame cache + send_to_text_bytes  event.rs:311-330
```

Notes verified against the code:
- The **community label comes from the parsed Redis channel** (`channel_event.community_id`, `event.rs:291`), not from the event body — so a forged event body cannot relabel a delivery.
- The **`threaded` visibility argument is `None`** on this path (`event.rs:307`), so a cross-node private-channel delivery always pays a fresh visibility read (or a cached `private`).
- The **DM-visibility owner fence (BR-WS-110) does not exist on this path.** It lives only in `dispatch_persistent_event_inner` (`event.rs:457-491`). A kind 30622 / 44200 event that reaches a second pod via Redis is fanned out with only the `filter_fanout_by_access` fences applied — neither of those kinds is in `AUTHOR_ONLY_KINDS` (`buzz-core/src/kind.rs:120`), so the owner check is absent. The kind-30175 gate added by `ab3af828` **is** inside `filter_fanout_by_access` (`event.rs:154-175`) and therefore does reach this path — the same placement would fix the 30622/44200 case. See the security and debt aspects.
- Broadcast lag is counted (`buzz_multinode_fanout_lag_total`, `main.rs:834`) but **not** repaired — lagged events are lost, not re-fetched.

##### 2.4 Redis subscription demand is driven only by REQ

`retain_topic` is called from exactly **one** production site: `req.rs:255`, after a successful non-search registration. The three `release_topic` sites are `req.rs:250` (replaced subscription), `close.rs:21` (explicit CLOSE), `connection.rs:268` (per removed subscription at disconnect).

Consequence: a pod that only *publishes* never PSUBSCRIBEs. `handlers/event.rs:1706` and `:1687` do call `retain_topic(&tenant, EventTopic::Global)` — but both are inside `#[cfg(test)]` (the Redis round-trip test, module begins `event.rs:1158`), precisely because the test needed to force the demand-driven subscription that production only creates via REQ (see the comment at `event.rs:1715-1722`). This is correct behaviour, not a gap, but it means **local echo suppression on the publishing pod only matters when that pod also has a subscriber for the topic**.

---

#### 3. Fan-out index selection (the performance feature)

`fan_out_scoped` (`subscription.rs:265-330`) is sub-linear in total subscriptions:

| Event shape | Indexes consulted | Site |
|---|---|---|
| `channel_id = Some(ch)` | `channel_kind_index[(community,{ch,kind})]`, then `channel_wildcard_index[(community,ch)]` | `:271-289` |
| `channel_id = None` | `global_p_kind_index[{community,kind,p}]` for **each** distinct `p` tag, then `global_kind_index[(community,kind)]`, then `global_wildcard_index[community]` | `:290-318` |

Every candidate still runs the full `filters_match` predicate (`:369-387`), with a `seen` set suppressing duplicates when a subscription appears in two indexes. The `(kind, #p)` index exists specifically so an observer-frame or membership-notification broadcast does not scan every subscription of that kind — pinned by `subscription.rs:1218-1250`.

---

#### 4. Frame-serialisation optimisation

One `serde_json::to_string` per event per fan-out cycle (`event.rs:252-259`, `:288-295`, `:420-431`), then one `Arc<Bytes>` per distinct `sub_id` (`fanout_frame_cache`, `event.rs:63-74`), cloned per recipient without copying the body (`state.rs:444-448`). Byte-for-byte compatibility with the legacy `format!` output is pinned at `event.rs:1178-1189`, and cross-cycle `Arc` reuse is explicitly forbidden and tested at `event.rs:1168-1188`.

Outbound writes are batched: up to 64 data frames per `flush()` (`connection.rs:347-369`), with control frames always drained first (`connection.rs:322-325`).

---

#### 5. Observability emitted by this group

| Metric | Type | Site |
|---|---|---|
| `buzz_ws_connections_total{community}` | counter | `connection.rs:184-188` |
| `buzz_ws_connections_active` | gauge | `connection.rs:196`, `:286` |
| `buzz_ws_auth_timeouts_total` | counter | `connection.rs:243` |
| `buzz_ws_backpressure_disconnects_total` | counter | `connection.rs:101`, `state.rs:466` |
| `buzz_ws_send_batch_size` | histogram | `connection.rs:368` |
| `buzz_admission_rejections_total{transport,reason}` | counter | `connection.rs:663`, `:671` |
| `buzz_auth_attempts_total{method}` | counter | `auth.rs:84` |
| `buzz_auth_failures_total{reason}` | counter | `auth.rs:169-171`, `:210`, `:235`, `:288` — reasons `banned`, `ban_check_error`, `allowlist_denied`, `not_relay_member`, `nip42_invalid` |
| `buzz_subscriptions_active` | gauge | `subscription.rs:83`, `:170`, `:192` |
| `buzz_events_received_total{kind}` | counter | `event.rs:624` (kind label bounded by `bounded_kind_label`, `:35-53`) |
| `buzz_community_events_received_total{community}` | counter | `event.rs:628-632` |
| `buzz_event_processing_seconds` | histogram | `event.rs:740` |
| `buzz_fanout_recipients` | histogram | `event.rs:248`, `:418` |
| `buzz_multinode_fanout_total` | counter | `event.rs:308` |
| `buzz_post_commit_dispatch_scheduled_total` | counter | `event.rs:374` |
| `buzz_post_commit_dispatch_errors_total{stage}` | counter | `event.rs:449` |
| `buzz_audit_send_errors_total` | counter | `event.rs:599` |
| `buzz_req_global_access_resolution_skips_total{kind}` | counter | `req.rs:90` |
| `buzz_count_fallback_rejections_total` | counter | `count.rs:190`, `:259` |
| `buzz_workflow_runs_total{trigger,community}` | counter | `event.rs:545-550` |

Cardinality is deliberately managed: kind is fleet-wide only and community is kind-free, because `bounded_kind_label` passes through all 10 000 client-controlled values in 20000–29999 (rationale `event.rs:620-623`).

Tracing spans: `ws.auth` (`connection.rs:505`), `ws.event` (`:524-529`), `ws.req` (`:551`), `ws.count` (`:572`) — each captured *before* the `tokio::spawn` so context propagates (comment `:522-523`).

---

#### 6. Capabilities notably **absent** from the WS surface

| Missing | Evidence |
|---|---|
| NIP-01 `["CLOSED", sub, "…"]` on relay-side subscription eviction by the reaper | `side_effects.rs:62` removes registry entries; no `CLOSED` is emitted from this group's code for that path |
| Any per-IP connection cap | `check_ip_connection` never called (`buzz-pubsub/src/rate_limiter.rs:112` has no relay caller) |
| Subscription-level result limits beyond per-filter `limit` | only `MAX_HISTORICAL_LIMIT` per filter (`req.rs:881`) |
| NIP-45 `HLL` / approximate counts | `RelayMessage::count` emits only `{"count": N}` (`protocol.rs:208`) |
| Compression (permessage-deflate) | no negotiation anywhere in `router.rs:334-342` |
| Backfill resumption / cursors | EOSE is terminal; no `since`-cursor handshake |


## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: Features

Every product capability that writes state passes through `ingest_event`
(`ingest.rs:1367`). This aspect maps features to the branches that realize them, then
lists the mismatches: handled kinds with no reachable feature, and declared kinds ingest
refuses.

---

### 1. Feature → ingest branch map

| Feature | Kinds | Ingest branch | Side effects |
|---|---|---|---|
| **Channel messaging** | 9, 40002 | generic store (`ingest.rs:2417-2427`) + NIP-10 thread resolution (`:2220-2231`) | none; kind:39005 summary emitted for replies (`:2448-2455`) |
| **Message editing** | 40003 | `validate_edit_ownership` (`ingest.rs:1986-1990`) → generic store | none |
| **Message deletion (self / owner)** | 5 | `validate_standard_deletion_event` (`ingest.rs:1947-1951`) → store → `handle_standard_deletion_event` (`side_effects.rs:2108`) | soft-delete + counter decrement + reaction-row removal + kind:39005 |
| **Moderated deletion (channel admin)** | 9005 | `validate_admin_event` 9005 arm (`side_effects.rs:508`) → store → `handle_delete_event_side_effect` (`side_effects.rs:1560`) | soft-delete, counter decrement, kind:39005, kind:40099 tombstone carrying `action_id`/`reason_code`/`public_reason` |
| **Reactions** | 7 | inline transactional path (`ingest.rs:2271-2391`) | none (dedup + `reactions` row happen pre-storage) |
| **Threads (NIP-10) + live badge counts** | any `requires_h` kind with `e root`/`e reply` | `resolve_nip10_thread_meta` (`ingest.rs:564-717`) | `emit_live_thread_summary` → kind:39005 (`side_effects.rs:724`) |
| **Message pinning / bookmarking / scheduling / reminders** | 40004, 40005, 40006, 40007 | generic store, `h` required — **no validator, no side effect** | none |
| **Code-diff messages** | 40008 | `validate_diff_event` (`ingest.rs:896-963`) → generic store | none |
| **Canvas** | 40100 | generic store, `h` required | none |
| **Forums** | 45001, 45002, 45003 | generic store; 45002 gated by `validate_forum_vote_target` (`ingest.rs:844`) | none |
| **Channel lifecycle (create)** | 9007 | eager `create_channel_with_id` (`ingest.rs:2129`) → store → `handle_create_group` (`side_effects.rs:1660`) | kind:40099 `channel_created`, 39000/39001/39002 discovery, kind:44100 to the creator, `buzz_channels_created_total` metric |
| **Channel metadata (name/about/topic/purpose/visibility/ttl/archive)** | 9002 | `validate_admin_event` 9002 arm (`side_effects.rs:410`) → store → `handle_edit_metadata` (`side_effects.rs:1335`) | per-tag DB update, kind:40099 per change type, cache invalidation, subscription eviction on open→private, kind:44100 resubscribe fan-out on unarchive, discovery re-emit |
| **Channel deletion** | 9008 | 9008 arm (`side_effects.rs:625`) → store → `handle_delete_group` (`side_effects.rs:1783`) | soft-delete channel, soft-delete discovery events, cache invalidation, kind:40099 |
| **Channel membership (invite / remove)** | 9000, 9001 | 9000/9001 arms → store → `handle_put_user` / `handle_remove_user` | `channel_members` write, cache invalidation, subscription eviction (remove only), kind:40099, discovery re-emit, kind:44100/44101 |
| **Self-join / self-leave** | 9021, 9022 | `ingest.rs:2134-2154` / 9022 arm → `handle_join_request` / `handle_leave_request` | same shape as 9000/9001 |
| **Agent channel-add policy** | 10100 | store → `handle_agent_profile` (`side_effects.rs:1078`) | `ensure_user`, `set_channel_add_policy`; consumed by the 9000 third-party-add gate (`side_effects.rs:340-365`) |
| **Profiles + NIP-05** | 0 | JSON pre-check (`ingest.rs:2234`) → store → `handle_kind0_profile` (`side_effects.rs:1113`) | absolute-state sync to `users`; NIP-05 canonicalised against the tenant host and silently cleared if off-domain; UNIQUE collision retried without the handle |
| **Direct messages (NIP-59 payload)** | 1059 | store with `channel_id = None`; pubkey-match waived; **WS only** | none |
| **DM channel lifecycle** | 41010, 41011, 41012 | `command_executor.rs:310` / `:443` / `:580` | `open_dm`/`hide_dm`, kind:40099 `dm_created`, 39000/39002 discovery, kind:44100 per participant, kind:30622 NIP-DV snapshot |
| **Workflows (definition)** | 30620 | `command_executor.rs:653` | `upsert_workflow`, webhook-secret generation/preservation, workflow cache invalidation |
| **Workflows (manual trigger)** | 46020 | `command_executor.rs:809` | `create_workflow_run` + spawned `execute_from_step` |
| **Workflow approvals** | 46030, 46031 | `command_executor.rs:986` / `:1098` | approval status update; grant spawns `resume_workflow_after_approval` (`:1236`), deny cancels the run |
| **Workflow deletion** | 5 with an `a` tag `30620:…` | `handle_a_tag_deletion` (`side_effects.rs:1990-2049`) | `delete_workflow_for_owner` by UUID or owner+name, cache invalidation |
| **Community moderation (ban/timeout/report resolution)** | 9040–9044 | direct route (`ingest.rs:1598-1614`) | out of module (`moderation_commands.rs`) |
| **Abuse reporting** | 1984 | direct route (`ingest.rs:1560-1570`) | `moderation_reports` row only |
| **Ban/timeout write enforcement** | all except 9030–9033, 9040–9044 | `moderation_restriction_state` gate (`ingest.rs:1613-1642`) | — |
| **Relay membership admin (NIP-43)** | 9030–9033 | direct route (`ingest.rs:1834-1844`) | 8000/8001 delta + 13534 snapshot from `relay_admin.rs` |
| **Self-service relay leave** | 28936 | direct route (`ingest.rs:1846-1928`) | `publish_nip43_member_removed` + `publish_nip43_membership_list` (both fire-and-forget) |
| **Identity archival (NIP-IA)** | 9035, 9036 | pre-storage handler then normal storage (`ingest.rs:1941-1945`) | 8002/8003 delta + 13535 snapshot |
| **Agent memory / engrams (NIP-AE)** | 30174 | `validate_engram_envelope` (`ingest.rs:2002-2005`) → NIP-33 store | none |
| **Agent telemetry (NIP-AM)** | 44200 | envelope + owner check (`ingest.rs:1981-2016`) → store | none |
| **Personas / teams / managed agents (NIP-AP)** | 30175, 30176, 30177 | 30175 gets `validate_persona_envelope`; 30176/30177 only the `d`-length check | none |
| **Event reminders (NIP-ER)** | 30300 | `validate_event_reminder` (`ingest.rs:2044-2047`) → NIP-33 store | none |
| **Web push leases (NIP-PL)** | 30350 | direct route (`ingest.rs:2156-2204`) | `push_leases` table |
| **Media attachments** | any kind with `imeta` tags | `validate_imeta_tags` + `verify_imeta_blobs` (`ingest.rs:2232-2244`) | none — the blob was uploaded out-of-band via Blossom |
| **Git repos (NIP-34)** | 30617, 30618, 1617–1621, 1630–1633 | 30617 → store → `handle_git_repo_announcement` (`side_effects.rs:2412`); the rest are plain stores | name reservation, empty-manifest pointer seed/repair, initial kind:30618 |
| **Huddles (audio)** | 48100–48103, 48106 | generic store, `h` required — **no validator, no side effect** | none |
| **User status / read state / NIP-51 lists / NIP-65 / NIP-30 emoji** | 30315, 30078, 10000, 10001, 10002, 10003, 10030, 30000, 30003, 30030 | replaceable / NIP-33 store, forced global | none |
| **Long-form posts** | 30023 | NIP-33 store, forced global | deletion supported via the generic coordinate soft-delete (`side_effects.rs:2051-2088`, referencing block/sprout#714) |
| **Product feedback** | 42000 | direct route (`ingest.rs:1538-1558`) | private deployment table |
| **Text notes** | 1 | generic store, forced global | none |
| **Contact list** | 3 | replaceable store, forced global | none |
| **Operational reconciliation** | — | not ingest; `main.rs` startup/periodic tasks | `reconcile_nip43_membership_snapshots` (`side_effects.rs:2776`, called `main.rs:508`, `:527`), `reconcile_channel_events` (`:2948`, called `main.rs:577`), `evict_all_channel_subscriptions` (`:128`, called `main.rs:672` by the ephemeral-channel reaper) |

---

### 2. Handled kinds whose feature has **no other reachable surface**

| Kind | Situation |
|---|---|
| 40004 `STREAM_MESSAGE_PINNED`, 40005 `…_BOOKMARKED`, 40006 `…_SCHEDULED`, 40007 `STREAM_REMINDER` | Accepted and stored with only the `h`-tag gate (`ingest.rs:455-491`). No pre-storage validator, no side effect, no `is_side_effect_kind` membership. Semantics are entirely client-side convention; the relay cannot tell a pin from a scheduled message. Nothing enforces that the `e` target exists or is in the same channel — unlike 40003 (`ingest.rs:763`) and 45002 (`:844`), which do check. |
| 48100–48103, 48106 huddles | Same: stored, `h` required, zero relay-side validation. A client can post `HUDDLE_ENDED` for a huddle that never started. |
| 40100 `CANVAS` | Same. |
| 30176 `TEAM`, 30177 `MANAGED_AGENT` | Accepted with only the 1024-byte `d`-length check. 30175 `PERSONA` gets a strict slug grammar (`ingest.rs:1027`) precisely because an empty `d` causes LWW data loss — 30176/30177 share that exact addressing shape and have **no** such guard, so an empty `d` collapses every team/managed-agent into one slot. Asymmetric by omission, not by design. |
| 30618 `GIT_REPO_STATE` | Client-submittable with `ReposWrite` scope, but the relay also mints it itself (`side_effects.rs:2733`, and `api/git/manifest_event.rs` on push). A client can publish a competing 30618 for a repo it does not own — the `d`-tag coordinate includes the *submitter's* pubkey, so it cannot overwrite the relay-signed head, but it can pollute the kind. |
| 1617–1621, 1630–1633 git patches/PRs/issues/statuses | Stored with `MessagesWrite` and forced global. No validation that the referenced repo exists or that the author has any relationship to it. |

---

### 3. Declared kinds ingest **rejects** — feature reachability

47 of the 127 `ALL_KINDS` entries never pass `required_scope_for_kind`
(`ingest.rs:198-306`). Grouped by why:

| Group | Kinds | Assessment |
|---|---|---|
| **Relay-signed outputs of this very module** | 8000, 8001, 8002, 8003, 13534, 13535, 39000, 39001, 39002 | Correct to reject. But note only 13534 is in `is_relay_only_kind` (`buzz-core/src/kind.rs:758-769`); the other eight fall through to the generic `restricted: unknown event kind`. Same outcome, worse diagnostic. |
| **Relay-signed, correctly classified** | 30622, 39005, 39006, 40901, 40902 | `is_relay_only_kind` → `restricted: relay-only kind` (`ingest.rs:1481`). |
| **Relay-signed, special-cased** | 44100, 44101 | Own message: `invalid: membership notifications are relay-signed only` (`ingest.rs:1469`). |
| **Relay-signed, no guard at all** | 40099 `SYSTEM_MESSAGE` | Minted by `emit_system_message` (`side_effects.rs:677`) but absent from `is_relay_only_kind`. It appears in `is_side_effect_kind` (`side_effects.rs:36`), which is dead code since ingest rejects it. |
| **Ephemeral (handled upstream)** | 20001, 20002, 24134, 24200, 24810 | Dispatched in `handlers/event.rs` before ingest; `ephemeral_kinds_not_in_scope_allowlist` (`ingest.rs:2790-2793`) asserts the exclusion. |
| **Auth artefacts** | 24242 `BLOSSOM_AUTH`, 27235 `HTTP_AUTH` | Consumed by the media and NIP-98 paths, never ingested. Correct. |
| **Unimplemented features** | 9009 `NIP29_CREATE_INVITE`, 39003 `NIP29_GROUP_ROLES`, 41 `CHANNEL_METADATA`, 1063 `FILE_METADATA`, 41001 `DM_CREATED`, 43001–43006 job kinds (6), 46001–46012 workflow-execution kinds (10), 48001 `AUDIT_ENTRY`, 49001 `MEDIA_UPLOAD` | **26 declared kinds with no ingest path.** The job kinds (43001–43006) and workflow-execution kinds (46001–46012) are declared, documented in `buzz-core/src/kind.rs:456-522`, and unreachable: nothing can publish them. 48001 `AUDIT_ENTRY` is particularly notable — the audit log is Postgres-only (`buzz-audit/src/service.rs`), so this kind exists purely as a reservation. |

The single most concrete mismatch: **9009** has a live `handle_side_effects` arm
(`side_effects.rs:161-167`) that logs `"NIP-29 kind 9009 handler deferred to future
phase"` and returns `Ok`. It cannot be reached — 9009 is not in the allowlist. The arm is
provably dead.

---

### 4. Cross-crate feature mismatches found

| Mismatch | Evidence |
|---|---|
| **Workflow `add_reaction` action is broken.** `buzz-workflow` POSTs to `{base_url}/api/messages/{message_id}/reactions` (`buzz-workflow/src/executor.rs:883-889`). No such route exists — the relay registers exactly three `/api/*` paths (`router.rs:95`, `:96`, `:111`) and no `/api/messages` prefix anywhere. Meanwhile ingest **does** have a full native reaction path (kind:7, `ingest.rs:2271-2391`) that the workflow action does not use. Any workflow with an `add_reaction` step will 404 and surface `AddReaction: relay returned {status}` (`executor.rs:917`). | `buzz-workflow/src/executor.rs:883-917`; `crates/buzz-relay/src/router.rs` |
| **Reaction-triggered workflows can fire but cannot react back.** `reaction_added` is a valid workflow trigger (`buzz-workflow/src/schema.rs:290`), and kind:7 does reach `dispatch_persistent_event` → `workflow_engine.on_event` (`handlers/event.rs:525-554`), so the trigger works. Only the response action is dead. | as above |
| **9 of 11 audit actions have no producer.** `AuditAction` declares 11 variants (`buzz-audit/src/action.rs:8-31`). Production emits exactly two: `EventCreated` (`handlers/event.rs:583`) and `MediaUploaded` (`api/media.rs`). `EventDeleted`, `ChannelCreated`, `ChannelUpdated`, `ChannelDeleted`, `MemberAdded`, `MemberRemoved`, `AuthSuccess`, `AuthFailure`, `RateLimitExceeded` are never written — despite this module performing every one of those actions. | grep for `AuditAction::*` outside `crates/buzz-audit/` and `tests/` |
| **`is_side_effect_kind` claims two ranges that no kind can reach.** `41001..=41003` and `40099` (`side_effects.rs:36`) — 41001 is `DM_CREATED` (not in the allowlist), 41002/41003 are undefined, 40099 is relay-signed. The DM *command* kinds are 41010–41012 and are routed to `command_executor` long before this predicate runs (`ingest.rs:1560`). | `side_effects.rs:35-37` vs `ingest.rs:198-306` |


## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Features

---

#### 1. Product capability → endpoint map

| Capability | Endpoints | Handler `file:line` | Consumers (verified by grep) |
|---|---|---|---|
| **Send a message / publish any event over HTTP** | `POST /events` | `bridge.rs:613` | `buzz-cli` (`client.rs:874`, `:1025`), desktop Tauri (`huddle/pipeline.rs:321`, `commands/personas/snapshot/import.rs:550`), desktop E2E bridge (`e2eBridge.ts:5189`) |
| **Read messages / channels / profiles over HTTP** | `POST /query` | `bridge.rs:880` | `buzz-cli` (`client.rs:774`), mobile (`relay_session.dart:136`), desktop timeline via the bridge (`channelWindowResponse.ts:84`, `e2eBridge.ts:4219`, `:4368`, `:4527`) |
| **Unread / badge counts** | `POST /count` | `bridge.rs:1314` | `buzz-cli` (`client.rs:804`) |
| **Server-assembled channel timeline (read model)** | `POST /query` with `top_level` / `include_aux` / `include_summaries` | `bridge.rs:404-581` | desktop (`channelWindowResponse.ts`, `e2eBridge.ts:4527`) |
| **Threaded replies with keyset pagination** | `POST /query` with `depth_limit` + `thread_cursor` | `bridge.rs:1112-1183`, `:305-345` | desktop `get_thread_replies` paging (doc at `bridge.rs:307-315`); `buzz messages thread` |
| **Activity / mentions / needs-action feeds** | `POST /query` with `feed_types` | `bridge.rs:1029-1109` | desktop feed views |
| **Full-text search (NIP-50), prefix + paged** | `POST /query` with `search` | `bridge.rs:1616-1749` | desktop search; `buzz messages search` |
| **Presence (online/away)** | `POST /query` for kind:20001/40902 | `bridge.rs:1920-1985` | desktop/mobile presence; replaced the removed `GET /api/presence` (`mobile/test/features/profile/presence_cache_provider_test.dart:13`) |
| **Moderation queue, audit log, active restrictions** | `GET /moderation/reports\|audit\|restricted` | `bridge.rs:2092`, `:2100`, `:2118` | desktop (`shared/api/moderation.ts:340-373`, `features/settings/lib/moderationQueue.ts`), `buzz-cli` (`commands/moderation.rs:110`, `:120`, `:127`) |
| **Workflow automation triggered by external systems** | `POST /hooks/{id}` | `bridge.rs:1780` | external callers only — no in-repo client |
| **Media upload (images, video, generic files)** | `PUT /upload`, `PUT /media/upload` | `media.rs:305` | desktop Tauri (`commands/media.rs:487`, `:498` with legacy fallback), mobile (`relay_client.dart:23`, `media_upload.dart:19`), `buzz-cli` (`client.rs:1144`, `:1197`) |
| **Media download, thumbnails, video seeking** | `GET/HEAD /media/{sha256_ext}` | `media.rs:604`, `:798` | desktop markdown/video (`mediaEntry.ts:75`, `MarkdownVideoPlayer.tsx:36`, `VideoPlayer.tsx:96`), `buzz-cli` (`client.rs` `/media/{hash}.jpg`) |
| **Invite a new member to a closed relay** | `POST /api/invites` | `invites.rs:230` | desktop (`shared/api/invites.ts:183`, `InviteLinkSection.tsx:44`) |
| **Redeem an invite (self-service join)** | `POST /api/invites/claim` | `invites.rs:291` | desktop (`invites.ts:203`, `useClaimInvite.ts`), web (`invite-api` via `InvitePage.tsx:110`), mobile (`invite_join_provider.dart:248`) |
| **Terms/privacy consent gate before join** | `GET /api/join-policy`, `/terms`, `/privacy`, `POST /api/invites/accept-policy` | `invites.rs:75`, `:95`, `:104`, `:162` | desktop (`invites.ts:106`, `:129`, `:160`, `JoinPolicyNotice.tsx`), web (`InvitePage.tsx:69`, `:80`) |
| **Invite landing page (browser)** | SPA fallback `/invite/{code}` | `router.rs:191-194` (`is_invite_landing_path`) | `web/src/features/invite/ui/InvitePage.tsx` |
| **NIP-05 identity resolution** | `GET /.well-known/nostr.json` | `nip05.rs:26` | **no product client** — only E2E/conformance tests (`e2e_relay.rs:1061`, `:1106`; `conformance_multitenant.rs:1201`, `:1242`); external Nostr clients are the real audience |
| **Community provisioning / lifecycle (control plane)** | 6 × `/operator/communities*` | `operator.rs:149`, `:203`, `:265`, `:302`, `:354`, `:468` | **no in-repo client at all** — see §2 |
| **Deployment-wide moderation + product-feedback dashboard** | 5 × `/api/admin/v1/*` | `admin/mod.rs:93`, `:125`, `:151`, `:177`, `:191` | `admin-web/` SPA (`admin-web/src/api.ts:1`, `App.tsx:520`, `tests/routes.spec.ts`) |
| **Mesh reliable-stream smoke test** | `POST /_mesh/demo/echo` | `mesh_demo.rs:58` | **none** — self-described as "Not a product flow" (`mesh_demo.rs:21-23`) |

#### 2. Endpoints with no in-repo client

Grepped `desktop/src`, `desktop/src-tauri`, `web/src`, `mobile/lib`, `crates/buzz-cli`,
`crates/buzz-ws-client`, `crates/buzz-sdk`, `crates/buzz-admin`, `crates/buzz-pairing-cli`,
plus a repo-wide sweep.

| Endpoint | Status | Notes |
|---|---|---|
| `GET /operator/communities` | zero clients | Only relay code + its own `#[ignore]`d tests reference the path (`operator.rs:312`, `:769`). Intended for an out-of-repo control plane (likely the buzz.xyz console). |
| `POST /operator/communities` | zero clients | same |
| `POST /operator/communities/archive` | zero clients | same |
| `POST /operator/communities/unarchive` | zero clients | same |
| `GET /operator/communities/availability` | zero clients | same |
| `POST /operator/communities/transfer` | zero clients | same |
| `POST /_mesh/demo/echo` | zero clients | testbed-only, double-flag-gated |
| `GET /.well-known/nostr.json` | zero product clients | tests only, in-repo |
| `POST /hooks/{id}` | zero in-repo clients | by design (external webhook senders) |
| `GET /api/admin/v1/*` | in-repo client is `admin-web/`, a separate SPA bundle | not shipped inside desktop/mobile |

**Consequence:** 6 of the 30 method×path pairs in this module group (20%) are an unexercised
control-plane surface from this repository's perspective; their only coverage is 11 `#[ignore]`d
Postgres-backed tests in `operator.rs`.

#### 3. Features that exist in code but are unreachable

| Item | `file:line` | Why |
|---|---|---|
| `api/events.rs` re-export shim | `api/events.rs:1-5` | Zero references repo-wide; `router.rs:71-73` binds `api::bridge::*` directly. Its stated reason ("backward compatibility with router.rs") is false. |
| `webhook_secret::strip_secret` | `webhook_secret.rs:57` | Zero production callers. The documented invariant "the secret must never be embedded in a response body" therefore has no enforcement point in the HTTP surface. |
| `POST /api/messages/{id}/reactions` | absent from `router.rs:32-190` | `buzz-workflow`'s `add_reaction` action POSTs here (`buzz-workflow/src/executor.rs:889`), so that workflow action always fails against this relay. |
| `GET /api/presence` | absent | `ARCHITECTURE.md:824` describes it as existing; it was removed (`mobile/test/features/profile/presence_cache_provider_test.dart:13`). |
| `HttpAuthMethod::DevPubkey` | `handlers/ingest.rs:58` | Never constructed; `bridge.rs:830` always reports `Nip98`. |
| `IngestAuth::Http.auth_method` | `bridge.rs:830` | Never read by any consumer. |
| `relay_members::check_relay_membership` / `MembershipDecision` | `api/mod.rs:61`, `:46` | Single caller (`api/mod.rs:130`) inside the same module; the transport-neutral indirection has no second consumer. |
| `MediaError::DisallowedContentType` on `/upload` | `media.rs:384-388` | Only reachable via the **legacy** `/media/upload` alias; the modern `/upload` route accepts generic files. |

#### 4. Feature flags / staged rollouts visible here

| Flag | Default | Gated feature | `file:line` |
|---|---|---|---|
| `require_auth_token` | **false** | real NIP-98 vs `X-Pubkey` header on `/events`, `/query`, `/count`, `/moderation/*` | `bridge.rs:118-127`; `config.rs:475-477` |
| `require_relay_membership` | **false** | closed-relay admission on `/events`, `/query`, `/count`, uploads, git | `api/mod.rs:67-69`; `config.rs:483-485` |
| `allow_nip_oa_auth` | **false** | NIP-OA owner delegation *for membership admission* (the owner **backfill** path is unflagged because the signature is self-proving) | `api/mod.rs:81`, `:151-156`; `config.rs:520-522` |
| `require_media_get_auth` | **false** | Blossom read auth on `GET/HEAD /media/*`; also flips `Cache-Control` public→private | `media.rs:491-514`, `:517-523`; `config.rs:682-689` |
| `join_policy` (derived from 3 env vars) | `None` | the consent gate on `claim` and the three policy endpoints | `config.rs:794-811`; `invites.rs:314-322` |
| `admin` (`BUZZ_ADMIN_HOST`) | `None` | the entire `/api/admin/v1` sub-router **and** its security-header middleware | `router.rs:56-59`; `config.rs:813-841` |
| `relay_operator_pubkeys` + `relay_operator_api_origin` | empty / `None` | effective usability of the 6 operator routes (routes always registered; requests 403 or 500 when unconfigured) | `operator.rs:69-98`; `config.rs:549-581` |
| `mesh` + `mesh_demo_echo` | false / false | `/_mesh/demo/echo` (404 unless both on) | `mesh_demo.rs:64-70`; `config.rs:509-518` |
| `upload_records_enabled` | **false** | per-upload `_uploads/` attribution records incl. uploader display name and edge IP | `media.rs:246-249`; `config.rs:651-653` |
| `serve_git_web_gui` | false | SPA fallback for `/`, `/repos*` (invite landing is always served) | `router.rs:196-198`; `config.rs:848-851` |

#### 5. Explicitly deferred / stubbed

| Item | `file:line` | Statement |
|---|---|---|
| Persistent per-pubkey storage quotas for media | `media.rs:302-304` | `TODO(v2)` — admission limits bound *active* work but do not cap durable bytes stored. The only TODO marker in all 13 files. |
| Per-invite-code revocation | `invite_token.rs:43-46` | "Revocation is coarse: rotate the relay keypair… Per-code revocation requires the future `relay_invites` table increment." |
| `/media/upload` legacy alias | `media.rs:5-7`, `:57-63`, `:313-317` | "temporary media-only legacy alias"; usage tracked via `buzz_media_legacy_upload_route_total`. Still called by desktop as a 404/405 fallback (`desktop/src-tauri/src/commands/media.rs:498`) and `buzz-cli` (`client.rs:1183-1197`). |
| `/_mesh/demo/echo` | `mesh_demo.rs:21-23` | "The real join-side consumer (goose/berd session wiring) replaces this; the route stays demo-gated until it is deleted." |
| Timestamp-only thread cursor | `bridge.rs:316-320` | "still accepted and paginates on time alone (unsafe across same-second ties)" |


## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: Features

#### 1. What this module is

A git server with **no persistent per-repo filesystem**. Every request materializes an ephemeral bare repo from an object-store manifest, shells out to stock `git`, and drops the tree (`hydrate.rs:1-27`, doc §v1 deployment architecture). Refs commit by compare-and-swap on a single manifest pointer (`cas_publish.rs:1208-1226`).

| Capability | Status | Evidence |
|---|---|---|
| Smart HTTP clone / fetch | supported | `transport.rs:786-827` |
| Smart HTTP push | supported | `transport.rs:858-962` |
| Ref advertisement | supported, two implementations | fast path `transport.rs:471-537`; subprocess `:596-724` |
| Dumb HTTP (`objects/info/packs`, loose-object URLs) | **absent** — no routes registered | `transport.rs:1758-1763` |
| `git://` daemon, SSH transport | **absent** | no listener anywhere in the crate |
| Protocol v2 (`Git-Protocol: version=2`) | **not implemented server-side**; the request header is never read, and the fast-path advertisement is a v0 `# service=` stream. The subprocess path inherits whatever the installed `git upload-pack --advertise-refs` does without the env var set, i.e. v0. | `transport.rs:471-537`, `:645-653` |

#### 2. Git operations

| Operation | Supported | Mechanism / limit |
|---|---|---|
| Full clone | yes | subprocess `upload-pack`, streamed (`transport.rs:1414-1498`) |
| Fetch / incremental | yes | same path |
| **Shallow clone** (`--depth`) | yes, via subprocess. The fast-path advertisement offers `shallow deepen-since deepen-not deepen-relative` as a fixed string (`transport.rs:484-491`); the real negotiation happens against `git upload-pack` in the follow-up POST. Exercised in production by the web repo browser (`web/src/features/repos/git-client.ts:86-113`, `depth: 1`). |
| `--single-branch`, `--no-tags` | yes (client-side negotiation, server-transparent) | `web/src/features/repos/git-client.ts:100-112` |
| **Partial clone** (`filter=blob:none`, promisor) | **not offered**. `filter` is absent from the fast-path capability string (`transport.rs:484-494`); the subprocess path may advertise it if the installed git does, but no promisor remote/fetch-object protocol exists on the relay. | |
| **Thin pack** | offered (`thin-pack` in caps, `transport.rs:485`) | |
| `side-band` / `side-band-64k` | offered; report-status parsing explicitly handles the nested band-1 framing | `transport.rs:485`, `:1181-1208` |
| Branch create / fast-forward | yes | `git_perms.rs:403-428` defaults Member |
| **Force push** (non-fast-forward) | yes, Admin by default, deniable per-pattern via `no-force-push` | `git_perms.rs:419-427`, `:545-551`; e2e `crates/buzz-test-client/tests/e2e_git.rs:197` |
| **Branch deletion** | yes, Admin by default, deniable via `no-delete`. `capture_pack` handles delete-all (no positive tips ⇒ `None`) and refs-only deletes. | `git_perms.rs:428`, `:553-559`; `cas_publish.rs:507-511` |
| **Tags** | stored and served, but any `refs/tags/*` **disables the advertisement fast path** because the manifest cannot reproduce an annotated tag's `^{}` peel line. Tag *creation* defaults to Member; tag *move* to Admin. | `transport.rs:388-403`; `git_perms.rs:410-424` |
| Atomic push (`atomic` capability) | not offered by the fast path (receive-pack advertisement always goes through the subprocess, so whatever git offers applies). Policy evaluation is nonetheless all-or-nothing. | `transport.rs:549-551`; `git_perms.rs:584-600` |
| **Git LFS** | **absent** — no `objects/batch` endpoint, no `lfs` string anywhere in the module. | |
| **Submodules** | no special handling; gitlinks are ordinary objects inside packs. No recursive fetch support, no `.gitmodules` awareness. | |
| **Signed pushes** (`push-cert`) | not offered; `receive-pack` runs without `--signed`. Object-level signing is a separate concern handled by the `git-sign-nostr` crate. | `transport.rs:1019-1027` |
| **Server-side hooks other than pre-receive** | absent. Only `hooks/pre-receive` is installed; `update`, `post-receive`, `post-update` are never written. | `hook.rs:152-178` |
| **Repository deletion / rename** | absent. No protocol path deletes a pointer or a repo; the doc's "no deletion under the protocol" rule (doc §Axioms A1) is upheld by omission. | |
| Garbage collection / repack of stored objects | only proactive **pack compaction** during an accepted push; old packs are never deleted. Physical pruning is an out-of-band retention concern. | `cas_publish.rs:666-875`, doc §Theorem 2 Remark |
| SHA-256 repositories | advertised as derivable (`object-format=sha256` from oid width, `transport.rs:473-479`) and accepted by `is_hex_oid`, but **not usable**: `init_bare_repo` always creates SHA-1 (`hydrate.rs:181-184`) and `snapshot_workspace_state` drops non-40-hex oids (`cas_publish.rs:325-328`). |

#### 3. Performance / operability features

| Feature | Description | Site |
|---|---|---|
| Manifest advertisement fast path | Serves `info/refs` for `git-upload-pack` straight from the verified manifest — no hydrate, no subprocess, **no semaphore permit**. Gated by `fast_path_eligible`, which doubles as a safety re-check. Byte-compatible with `git upload-pack --advertise-refs` for the branches-only case. | `transport.rs:380-537`, `:552-577` |
| Streamed fetch | `upload-pack` stdout is streamed into the HTTP body instead of buffered; the child, the tempdir, the stdin pump, and the semaphore permit are all kept alive until EOF or client disconnect. | `transport.rs:1262-1332`, `:1414-1498` |
| Streaming deadline + metrics | 300 s in-band deadline that kills the subprocess; `buzz_git_upload_pack_{timeouts_total, stream_seconds, stream_bytes}`. | `transport.rs:1282-1391` |
| Bounded pack/index cache | Process-local, digest-keyed, byte-bounded, LRU-evicted, single-flight populated, hard-linked into workspaces. | `pack_cache.rs:66-456` |
| Cross-process cache adoption | A lost publication rename is resolved by adopting the winner's directory — designed for several relay processes sharing a scratch volume. | `pack_cache.rs:314-330` |
| Stale-session GC | Abandoned `session-*` cache dirs older than 10 min are removed at startup. | `pack_cache.rs:482-509` |
| Proactive pack compaction | At ≥ 96 parent packs an accepted push captures the full post-push closure into ≤ 128 bounded packs and replaces the pack list. | `cas_publish.rs:1046-1103`, `:666-875` |
| gzip request-body inflation | Git gzips large negotiation bodies; the relay inflates transparently with an independent decoded-byte cap. Without it the subprocess dies with `bad line length character`. | `transport.rs:726-784` |
| Startup conformance gate | Fatal A1/A3 probe against the configured bucket before the listener opens. | `store.rs:571-877`, `main.rs:466-503` |
| Observability | 8 counters, 11 histograms, 3 gauges under `buzz_git_*`. | see integrations aspect |

#### 4. Authorization features

| Feature | Description | Site |
|---|---|---|
| NIP-98 transport auth | Repo-root-scoped, tenant-host-bound, ±60 s window. Method and body-hash binding intentionally waived. | `transport.rs:76-231` |
| NIP-OA managed-agent delegation | An agent's attestation may travel in the signed NIP-98 event's `auth` tag (git cannot carry `x-auth-tag` through the credential helper). | `transport.rs:204-210` |
| Managed-agent owner authority | A verified NIP-OA *owner* of the repo-owner key gets `MemberRole::Owner` on that repo. | `policy.rs:343-368` |
| Channel-bound roles | Non-owners derive their role from the channel named by kind:30617's `buzz-channel` tag. | `policy.rs:308-395` |
| Bot promotion | `MemberRole::Bot` → `Member` for git pushes only. | `policy.rs:397-402` |
| Ref protection via `buzz-protect` tags | Glob patterns (≤ 3 wildcards, ≤ 256 chars, ≤ 50 rules) with `push:<role>`, `no-force-push`, `no-delete`, `require-patch`; unioned strictest-wins; explicit rules can never weaken defaults. | `git_perms.rs:244-262`, `:445-486`, `:534-550` |
| Archived-channel read-only | An archived bound channel denies **all** pushes, owner included. | `policy.rs:308-330` |
| Per-pubkey repo quota | Default 100, enforced at announce. | `handlers/side_effects.rs:2486-2492` |
| Repo-name reservation | `(community_id, repo_id)` unique in Postgres via `INSERT … ON CONFLICT DO NOTHING`. | `handlers/side_effects.rs:2441-2513` |
| **Read authorization** | **absent.** No per-repo, per-channel, or visibility check on `info/refs`/`upload-pack`. `transport.rs:60-66` claims "No public repos for v1", but in practice every repo is readable by any caller that clears the transport gate. | absence in `transport.rs:539-827` |

#### 5. Repo browser surface

The relay does not render repository content itself. Two cooperating surfaces:

| Surface | Mechanism | Site |
|---|---|---|
| Repo/branch discovery | `POST /query` for kinds 30617 (repos) and 30618 (ref state) — plain Nostr, not this module. The web client explicitly notes it must not trust arbitrary authors' 30618. | `web/src/features/repos/use-repos.ts:66`, `use-repo-refs.ts:54-57` |
| File/tree/blob browsing | **client-side**: `isomorphic-git` over LightningFS/IndexedDB does a `depth:1, singleBranch, noTags` clone of `/git/{owner}/{repo}.git` with a NIP-98 header, then reads trees/blobs locally. | `web/src/features/repos/git-client.ts:17-140` |
| SPA route gating | The relay serves `/`, `/repos`, `/repos/*` from the web bundle only when `BUZZ_SERVE_GIT_WEB_GUI` is truthy. | `router.rs:206-213`; config `config.rs:848-850` |

Consequences: no server-side tree/blob/commit API exists, browsing costs a real clone per repo per browser, and the browser's IndexedDB holds full repo content. There is no server-side diff, blame, or search over repository contents.

#### 6. Not implemented (explicit non-features)

- No repo delete/rename/transfer protocol.
- No `receive-pack` `--signed` / push certificates.
- No `update`/`post-receive` hooks, so no server-side CI trigger from git itself (workflows are driven by kind:30618 events instead).
- No mirror/replication, no cross-relay fetch.
- No pack-content policy: received packs are never inspected for object types, sizes, or path names beyond what `git index-pack` validates.
- No per-repo read ACL, no anonymous/public read mode (both directions of that missing knob: no anonymous access, and no way to *restrict* read below community membership).
- No liveness/latency guarantee — explicitly out of scope in the design (doc §Scope and Non-Goals).


## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: Features

---

#### 1. Feature inventory

| # | Subsystem | File(s) | Status | Entry point |
|---|---|---|---|---|
| F1 | Community moderation commands (ban/unban/timeout/untimeout/resolve) | `handlers/moderation_commands.rs` (768 LOC) | working | `ingest.rs:1606` |
| F2 | Moderation authorization capability seam | `handlers/moderation_authz.rs` (335) | working, **partly dead** | `moderation_commands.rs` ×5, `api/bridge.rs:2055` |
| F3 | Relay-signed moderation notice DMs | `handlers/moderation_notices.rs` (398) | working, **1 of 3 templates unused** | `moderation_commands.rs` ×3 |
| F4 | NIP-56 report intake | `handlers/report.rs` (337) | working | `ingest.rs:1588` |
| F5 | NIP-43 relay-membership admin + workspace icon | `handlers/relay_admin.rs` (468) | working | `ingest.rs:1835` |
| F6 | Operator community provisioning | `handlers/community_provisioning.rs` (445) | working | `api/operator.rs:171` |
| F7 | NIP-PL push-lease validation + acceptance | `handlers/push_lease.rs` (771) | working, **`urgent` class dead** | `ingest.rs:2182` |
| F8 | NIP-PL durable matcher + APNs delivery worker | `push_runtime.rs` (656) | working, **no leader election** | `main.rs:687-690` |
| F9 | NIP-IA identity archive/unarchive | `handlers/identity_archive.rs` (580) | working | `ingest.rs:1942` |
| F10 | Product feedback sidecar | `handlers/product_feedback.rs` (161) | working | `ingest.rs:1567` |
| F11 | Hourly S3 storage sweep + gauges | `storage_sweep.rs` (1090) | working | `main.rs:1460`, `:1474` |
| F12 | Workflow action sink | `workflow_sink.rs` (711) | **1 of 7 workflow actions implemented** | `main.rs:594` |

---

#### F1 — Community moderation commands

Five direct commands, never stored as events, each writing durable state plus a `moderation_actions` audit row.

| Kind | Delivers | Side effects actually implemented |
|---|---|---|
| 9040 ban | permanent or expiring community ban | `community_bans` upsert (`:169`), audit row (`:180`), cluster-wide live disconnect (`:195-200`), best-effort notice DM (`:204-220`) |
| 9041 unban | ban lift, refuses if not banned | `community_bans` clear (`:248`), audit row (`:256`) — **no notice DM** |
| 9042 timeout | mandatory-expiry write-block | `muted_until` upsert (`:287`), audit row (`:298`), best-effort notice DM (`:309-321`) — **no disconnect** |
| 9043 untimeout | mute clear, refuses if not timed out | `community_bans` clear (`:351`), audit row (`:358`) — **no notice DM** |
| 9044 resolve report | report status + decision audit + reporter DM | audit row (`:453`), `resolve_moderation_report` (`:461`), reporter notice (`:481-495`) |

**Declared but not delivered.** The module docs claim 9044's `delete`/`kick`/`ban` resolutions "fan out through the existing 9005/9001 + 9040 paths" (`moderation_commands.rs:20`, `:47-50`). Verified: `handle_resolve` performs **no** fan-out — it writes a `resolve:*` decision row and returns (`:453-499`). The docs do say the *client* composes the paired 9040-9043 (`:48-50`), so the relay-side behaviour is by design, but the summary table at `:14-21` reads as if the relay chains the enforcement. Nothing in the relay guarantees the enforcement ever happens.

**`escalated` report status has no producer.** 9044 accepts `action=escalate` but always stores `status=resolved` (`:380-385`); the `ReportRecord.status` doc advertises `escalated` (`buzz-db/src/moderation.rs:71`) and `ModerationNotice::body` has an `"escalated"` arm (`moderation_notices.rs:281`) that is unreachable.

---

#### F2 — Moderation authorization seam

Delivers an exhaustively unit-tested pure policy function (`decide_authority`, `moderation_authz.rs:146-181`) separated from its I/O, plus a 3-role capability grid.

**Non-functional in production:** the entire channel-local authority branch.

| Capability | Production requester | Status |
|---|---|---|
| `Ban`, `Unban`, `Timeout`, `Untimeout` | 9040–9043 | live |
| `ResolveReport` | 9044 | live |
| `ViewQueue` | `GET /moderation/reports`, `/moderation/audit` | live |
| **`DeleteMessage`** | **none** | dead — zero call sites |
| **`Kick`** | **none** | dead — zero call sites |

Because no caller passes a `channel_id` (all 6 sites pass `None`) and no caller requests `DeleteMessage`/`Kick`, the `get_member_role` lookup (`:120-131`) is unreachable and `ModerationAuthority::ChannelRole` (`:174-178`) can never be returned. The 9005-delete and 9001-kick paths use a **separate, unrelated** authorization function, `side_effects::validate_admin_event` (`ingest.rs:1929-1933`) — the very function the module doc says the seam is "the bridge … missing today" for (`moderation_authz.rs:73-75`). The bridge exists but is not connected.

The declared-and-recorded `ModerationAuthority` is also non-functional: computed on every call, discarded by every caller, never written to the audit row despite the doc at `:61`.

---

#### F3 — Relay-signed moderation notice DMs

Delivers a closed-loop notification primitive: one DM thread per (community, user), relay-authored, with a named `"{host} Moderation"` kind:0 profile, kind:39000 discovery, and per-source idempotency.

| Template | Production producer |
|---|---|
| `Restriction { kind: "ban" }` | 9040 (`moderation_commands.rs:208`) |
| `Restriction { kind: "timeout" }` | 9042 (`moderation_commands.rs:313`) |
| `ReportResolved` | 9044 (`moderation_commands.rs:485`) |
| **`ContentActioned`** | **none** — declared (`moderation_notices.rs:52-57`), rendered (`:288-292`), tested (`:388-397`), never constructed outside tests |

`ContentActioned` is documented as the "actioned-author" notice for the same primitive (`moderation_notices.rs:25-26`) and its body mirrors the delete tombstone's `public_reason` — but nothing in the 9005 delete path calls `send_moderation_notice`. Verified: the only 3 call sites are in `moderation_commands.rs`.

**Also non-delivered:** unban (9041) and untimeout (9043) send no notice at all, so a user learns they were restricted but never that the restriction was lifted. The `Restriction` body promises to render the expiry ("with expiry rendered into the message", `moderation_notices.rs:66`) — the actual body does **not** include any expiry (`:296-305`), only the action verb and reason.

Replies are non-replyable in v1 by intent (`moderation_notices.rs:27-28`); the profile `about` string tells the user replies are not monitored (`:191-192`).

---

#### F4 — NIP-56 report intake

Delivers tenant-fenced report intake for three target classes with idempotency on the signed event id, and remains available during a write-block so users can signal abuse while timed out (`ingest.rs:1551-1559`).

Working: `e` (event, in-tenant only), `x` (media blob via tenant-scoped sidecar), `p`-only (community-local pubkey report). 7 report types.

**Known Phase-1 limitation, documented in-code:** blob lookup cannot distinguish not-found from transient storage failure, so both surface as `invalid: report target blob not found` (`report.rs:66-70`).

**Dead field:** `ParsedReportTarget::{Event,Blob}.author_pubkey` is parsed and validated but never persisted (`report.rs:106-107`, `:112-113`; discarded at `:54`, `:65`).

**Dead public constant:** `REPORT_TYPES` is `pub` with zero external consumers (`report.rs:29`).

---

#### F5 — NIP-43 relay-membership admin

Delivers four operations with a documented permission matrix and TOCTOU-safe removal.

| Kind | Delivers |
|---|---|
| 9030 | idempotent member add (admin/owner); admins can only add `member` |
| 9031 | member remove; admin path is a conditional delete restricted to `member` targets; owner path refuses other owners |
| 9032 | role change (owner only); `owner` role unreachable by design |
| 9033 | workspace icon set/clear (admin/owner), http(s) URL or `data:image/*` |

Every mutation best-effort publishes NIP-43 announcement events (`relay_admin.rs:214-220`, `:274-279`, `:334-336`).

**Misleading behaviour:** 9030 with `role=owner` says "use kind:9032 to promote to owner" (`:185`) but 9032 refuses `owner` too (`:301`). There is no event-based path to owner promotion at all; it requires `RELAY_OWNER_PUBKEY` config or the operator endpoint.

**Missing relative to the moderation path:** no ban re-check. `relay_admin.rs` contains zero references to `moderation_restriction_state`, and `ingest.rs:1639` exempts relay-admin kinds from the write gate — so a banned owner/admin with a surviving socket can still mutate membership.

---

#### F6 — Operator community provisioning

Delivers deployment-root community creation over `POST /operator/communities`, gated by the `RELAY_OPERATOR_PUBKEYS` allowlist and NIP-98 with a replay guard, plus strict canonical-authority host validation (13 unit tests, `community_provisioning.rs:354-443`).

Two modes:
- `create_only=true` — atomic host+owner creation, refuses existing hosts, enforces a per-owner community limit.
- default convergence mode — idempotent on the host row and **can rotate an existing community's owner** (`:236-247`, `:321-334`). Documented as the price of retry convergence; clients acting for end users are *told* to use `create_only` but nothing enforces it.

Also exports two validators reused by 5 other operator endpoints: `validate_pubkey_hex` (`:71`) and `normalize_candidate_host` (`:180`).

---

#### F7 — NIP-PL push-lease validation

Delivers a genuinely strict, closed-world envelope + encrypted-plaintext validator: 4 permitted public tags, duplicate-key rejection at every JSON depth via a custom `DeserializeSeed`, active/inactive schema bifurcation, server-resolved origin binding, 5-member filter allowlist, self-`#p` confinement, v4-UUID `#h` validation, and 12 hard-coded quotas.

Strongest invariant in the module: a unit test asserts the SQL trigger allowlist in `migrations/0018_push_match_queue.sql` uses `PUSH_KINDS` **exactly** (`push_lease.rs:696-710`), so the Rust and DB views of push-eligible kinds cannot drift.

**Declared but non-functional: the `urgent` delivery class.** `URGENT_KINDS = &[]` (`:16`) and `supported_classes` omits `"urgent"` (`:509`), so a lease requesting `class: "urgent"` is rejected earlier by `class not supported` (`:246`) and the urgent-kind confinement check at `:281-283` is unreachable. `class_rank` still ranks it in both copies (`:582`, `push_runtime.rs:574`) and NIP-11 publicly advertises `urgent_kinds: []` (`nip11.rs:209`).

**Declared but unused public API:** 8 public items with zero external consumers — `validate_envelope`, `parse_plaintext`, `validate_plaintext`, `LeaseEnvelope`, `LeasePlaintext`, `LeaseLimits`, `AppProfile`, `MAX_SAFE_JSON_INTEGER`. The module presents itself as a reusable validation library (`:1-6`) but only `accept` is called.

**Naming vs behaviour:** `ActiveLease.endpoint_grant` is documented as an "opaque endpoint grant issued by the stateless gateway" (`buzz-db/src/push.rs:94`), but the relay stores the client-supplied `endpoint` string verbatim (`push_lease.rs:544`, `:555`) and forwards it as `endpoint_grant` in the delivery body (`push_runtime.rs:507-515`). The relay never mints a grant.

---

#### F8 — NIP-PL matcher + delivery worker

Delivers two always-on background loops (spawned as one unit, `main.rs:686-692`):

**Matcher** — batch-claims up to 64 accepted events per community, loads leases + membership pairs once per batch, evaluates matches purely (no DB in `match_job`), flushes all wakes in one transaction, and reaps poison jobs on a separate 30 s cadence. Per-batch cost is documented as constant regardless of match count (`push_runtime.rs:52-56`). Idle backoff 250 ms → 2 s.

**Delivery worker** — enumerates communities via `usage_community_hosts()`, claims 16 wakes each, and for every wake performs revalidate → membership → revalidate before a single NIP-98-signed HTTPS POST. Handles 6 distinct response cases with generation-fenced endpoint invalidation and exponential retry to 8 attempts. Idle backoff 500 ms → 2 s.

**Notable design guarantee delivered:** the gateway request id is the durable wake row id and is byte-stable across retries, pinned by an HTTP-level test that runs a real axum server (`push_runtime.rs:626-655`).

**Not delivered: leader election.** Unlike the storage sweep, both loops run on **every** pod with no advisory lock (`main.rs:686-692`). Correctness rests entirely on DB claim/fence tokens; the cost is N× the claim-scan load at N pods. The delivery worker's per-community scan is also unbounded — it iterates every community returned by `usage_community_hosts()` on every sweep (`push_runtime.rs:320-338`).

**Not delivered: SSRF exposure surface.** By construction the destination URL is operator config, validated to an exact HTTPS `/v1/deliveries/apns` path (`config.rs:341-361`) — user data travels only in the body. This is a positive: there is no user-controlled URL anywhere in the delivery path.

---

#### F9 — NIP-IA identity archive

Delivers three consent paths (self / community owner-or-admin / cryptographic NIP-OA key owner), a NIP-70 protected-event requirement, optional successor-key (`replaced-by`) recording, and relay-signed 8002/8003 deltas plus a 13535 snapshot.

The distinguishing feature is **live revocation**: an owner-signed archive request is re-verified against the target's *current* kind:0 auth tag, so replacing your kind:0 immediately invalidates outstanding owner requests (`identity_archive.rs:270-296`). This is the module's only real integration test (`:515-578`).

Requests deliberately fall through to normal storage so the delta's `["e", request_id]` audit reference resolves (`ingest.rs:1935-1938`) — meaning 9035/9036 are the **only** kinds in this module that produce an `events` row and therefore the only ones that reach the `buzz-audit` hash chain.

Delta and snapshot publication failures only warn (`:130-136`), so `archived_identities` can silently diverge from the event-backed view. Unarchive is a hard `DELETE` (`buzz-db/src/archived_identities.rs:83-91`) — the consent/actor/reason provenance is destroyed, not tombstoned.

---

#### F10 — Product feedback sidecar

Delivers private, operator-only feedback capture that bypasses event storage and fan-out entirely: optional 3-value category, 32 KiB body, 64 KiB tag array, and full `imeta` attachment validation + blob verification against tenant media (`product_feedback.rs:26-36`). Smallest file in the group (161 LOC) with 4 focused unit tests.

---

#### F11 — S3 storage sweep

Delivers a cadence-decoupled bucket sweep whose results are cached and re-published on every usage tick, so a transient S3 blip never blanks dashboards:

| Capability | Site |
|---|---|
| Single-flight, harvest-then-spawn under one lock | `storage_sweep.rs:151-256` |
| Warm-cache retention across failures | `:175-184`, test `:531-591` |
| Immediate respawn after failure (self-healing) | `:105-127`, `:89-103` |
| Whole-attempt timeout, converted to `SweepError::Timeout` | `:249-252` |
| Object cap enforced before folding each page | `buzz-media/src/bucket_index.rs:392-395` |
| Hard kill switch suppressing even health gauges | `main.rs:1454-1456` |
| Zeroing of per-community series on disappearance / rename / scope exclusion | `:333-345`, test `:857-1088` |
| Unmapped-community rollup gauge | `:318-323`, `:330` |
| Demoted-leader safety (unharvested snapshot never published) | test `:643-670` |

18 gauges emitted: 3 health (`sweep_ok`, `sweep_failures`, `sweep_duration_seconds`), 1 freshness (`sweep_age_seconds`), 4 totals (physical/logical × bytes/objects), 7 anomaly/orphan, 1 unmapped rollup, 2 per-community series.

By far the best-tested file in the group: 15 tests including paused-time (`#[tokio::test(start_paused = true)]`) coverage of stall/timeout and a 3-scenario state-machine regression for stale series.

---

#### F12 — Workflow action sink

**This is the module's largest gap.** `ActionSink` declares exactly **one** method (`buzz-workflow/src/action_sink.rs:44-64`) and `RelayActionSink` implements exactly that one (`workflow_sink.rs:172-179`).

Against the 7 action types ARCHITECTURE.md:533-542 advertises:

| Action | Where it executes | Wired end-to-end? | Evidence |
|---|---|---|---|
| `send_message` | `workflow_sink.rs:180-357` | **YES** | full validate → build → persist → dispatch path |
| `send_dm` | `buzz-workflow/src/executor.rs:575-579` | **NO** | `Err(NotImplemented("SendDm"))` at `:578`; `// TODO (WF-07)` at `:577` |
| `set_channel_topic` | `executor.rs:580-584` | **NO** | `Err(NotImplemented("SetChannelTopic"))` at `:583`; `// TODO (WF-07)` at `:582` |
| `add_reaction` | `executor.rs:585-607` → `add_reaction_impl` `:885-919` | **NO** | POSTs `{BUZZ_RELAY_BASE_URL}/api/messages/{id}/reactions` (`:886-888`). Verified: `router.rs` registers **zero** `reactions` routes and zero `/api/messages` routes. Every attempt returns `WebhookError("AddReaction: relay returned 404 …")` (`:903-908`). Without the `reqwest` feature it silently returns `{"added": false, "skipped": true}` (`:597-606`) |
| `call_webhook` | `executor.rs:608+` | yes (own HTTP client, outside this module) | — |
| `request_approval` | `executor.rs` → `StepResult::Suspended` | **NO** | ARCHITECTURE.md:826 (WF-08) |
| `delay` | `executor.rs` | yes (outside this module) | — |

**Verified answer to the ARCHITECTURE.md §9 question:** `workflow_sink` implements **1 of 7** workflow actions. ARCHITECTURE.md items 5 and 6 (`:826-827`) are accurate but incomplete — `add_reaction` is a third broken action not listed as a limitation, and ARCHITECTURE.md:541 presents it as working. Because the trait has a single method, `send_dm` and `set_channel_topic` cannot be implemented without widening `ActionSink`.

**What `send_message` does deliver, well:**
- Community-correct tenancy (never re-derives from `config.relay_url`, fails closed on an unmapped community, `:190-210`).
- Access control: owner must be a channel member unless the channel is open (`:243-251`).
- `@Name` → `p` tag mention resolution that *defines* the agent-wake contract, deliberately conservative (members-only, exact name, greedy-longest, ambiguity wakes nobody) with genuinely hard Unicode correctness — the case-folding walk works in original-char coordinates because `İ`(U+0130) lowercases to two code points (`:78-96`), regression-tested at `:490-503`, `:544-560`, `:590-604`.
- 18 unit tests for mention resolution; `resolve_mention_pubkeys` is the most thoroughly tested pure function in the group.

**What it does not deliver:** threading (always `depth: 0`, `:333`), so workflows cannot reply in-thread; and `dispatch_persistent_event`'s result is discarded (`:351`), so fan-out/search/audit failures are silent.

---

#### 2. Cross-feature summary of declared-but-non-functional items

| Item | Declared at | Production producers/consumers |
|---|---|---|
`ModerationAction::DeleteMessage` | `moderation_authz.rs:32` | 0 |
`ModerationAction::Kick` | `moderation_authz.rs:34` | 0 |
`ModerationAuthority::ChannelRole` | `moderation_authz.rs:68` | unreachable |
`ModerationAuthority` return value | `moderation_authz.rs:61` | computed, discarded by all |
`ModerationNotice::ContentActioned` | `moderation_notices.rs:53` | 0 |
`ReportRecord.status = "escalated"` | `buzz-db/src/moderation.rs:71` | 0 |
`ModerationNotice::body` `"escalated"` arm | `moderation_notices.rs:281` | unreachable |
`NewAction.matched_principal` | `buzz-db/src/moderation.rs:139` | always `None` (`moderation_commands.rs:540`) |
`NewAction.reason_code` / `.private_reason` / `.channel_id` | `:130-136` | always `None` (`moderation_commands.rs:536-539`) |
`MODERATION_ACTION_CHECK_VOCAB` `"delete_message"`, `"kick"` | `:105-106` | 0 writers |
`resolution_audit_action` `"resolve:unknown"` | `moderation_commands.rs:511` | unreachable, and not in the CHECK vocab |
`ParsedReportTarget::{Event,Blob}.author_pubkey` | `report.rs:106`, `:112` | parsed, never persisted |
`REPORT_TYPES` (pub) | `report.rs:29` | 0 external |
`URGENT_KINDS` / `class: "urgent"` | `push_lease.rs:16` | unreachable |
`push_lease` 8 public validation items | `push_lease.rs:19-81`, `:83`, `:149`, `:194` | 0 external |
`StorageEmittedKey` (`pub(crate)`) | `storage_sweep.rs:121` | 0 outside its own file |
`send_dm`, `set_channel_topic` workflow actions | `buzz-workflow/src/executor.rs:575`, `:580` | `NotImplemented` |
`add_reaction` workflow action | `buzz-workflow/src/executor.rs:585` | 404 — route not registered |


## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Features

---

#### 1. Huddle audio — capability inventory

| Capability | Where | Completeness |
|---|---|---|
| WebSocket audio room per (community, channel) | `audio/room.rs:490-550`, route `router.rs:125-128` | Complete for single-pod |
| NIP-42 challenge/response auth on the audio socket | `handler.rs:176-238` | Complete — reuses `state.auth.verify_auth_event`, the same verifier as the main relay door |
| Channel-membership gate with ephemeral auto-add | `handler.rs:1153-1235` | Complete; auto-add restricted to TTL channels whose caller is a parent member |
| Opaque Opus fan-out with 1-byte `peer_index` prefix | `room.rs:391-411` | Complete; never decodes, never re-encodes |
| v2 frame header parse + telemetry clamp | `audio/wire.rs:29-88` | Complete; 4 tests pin byte order, short input, clamp-keeps-frame, reserved-bit passthrough |
| Room protocol-version pinning | `room.rs:110-127`, `:239-247` | Complete; 5 tests |
| Peer-index allocation with recycling | `room.rs:143-157` | Complete; 0..=254 |
| Separate audio / control queues with priority drain | `room.rs:23-27`, `handler.rs:1060-1086` | Complete |
| Heartbeat + missed-pong disconnect | `handler.rs:1127-1151` | Complete (30 s / 3 misses) |
| Auto-end + channel archive when the room empties | `room.rs:358-388`, `handler.rs:833-866` | Complete, with rollback on archive failure |
| Lifecycle events 48101 / 48102 / 48103 (relay-signed, persisted, fanned out locally, published to Redis) | `handler.rs:645-653`, `:822-831`, `:850-859`, `:1237-1332` | Complete |
| Ordered roster snapshot/delta model | `room.rs:55-81`, `:452-476` | Complete; broadcast cap 64 with snapshot recovery |
| Cross-pod huddle: owner-authoritative fan-out over the mesh | `audio/join.rs`, `audio/mesh.rs` | Implemented end-to-end; **only active when `BUZZ_MESH=on`** (default off) |
| Redis fenced-CAS ownership with generation monotonicity | `tunnel/directory.rs:20-79`, `:348-439` | Complete |
| Per-room owner-lease renewer with observable loss | `join.rs:452-562`, `:600-773` | Complete |
| Graceful drain of owned huddles on SIGTERM | `join.rs:750-773`, `mesh_boot.rs:481-496` | Complete |
| Ambiguity-safe media routing (fail closed on channel-UUID collision) | `room.rs:526-541`, `mesh.rs:221-227` | Complete |

---

#### 2. Absent from huddle audio (verified by reading, not inferred)

| Feature | Evidence of absence |
|---|---|
| **Recording** | No writer, no storage call, no media handle in `crates/buzz-relay/src/audio/`. `ARCHITECTURE.md:825` lists "Huddle recording/tracks not built" as known-absent; confirmed |
| **Transcription / STT** | Not in the relay. STT is client-side: `desktop/src-tauri/src/huddle/stt.rs` |
| **Video** | No video kind, no video track, no SDP. `buzz-core/src/kind.rs:530` calls 48100 "audio/video session" but nothing in the relay handles video |
| **Screen share** | No reference anywhere in `crates/buzz-relay/src` |
| **SFU mixing / transcoding** | The relay never decodes: `broadcast_frame` copies bytes with a 1-byte prefix (`room.rs:398-411`). Fan-out is N×(N−1) mesh, not a mixer. No Opus, WebRTC, or codec crate in `crates/buzz-relay/Cargo.toml` |
| **Echo cancellation / noise suppression / AGC** | None; no DSP dependency |
| **TURN / STUN / ICE** | None. Clients connect straight to the relay over WSS. The default `BUZZ_MESH_BIND_ADDR` port `3478` (`config.rs:507`) is the *STUN* port number but the protocol spoken there is iroh/QUIC, not STUN — a naming coincidence, not a TURN server |
| **Codec negotiation** | Nothing is negotiated except the integer `protocol_version` (`handler.rs:134-138`). Sample rate, channel count, frame duration, and bitrate are never exchanged. The 48 kHz assumption lives only in a field name (`wire.rs:44`) and the "20 ms/frame" comment behind the queue sizing (`room.rs:40`) |
| **Simulcast / layered audio / redundancy (RED/FEC)** | None |
| **Jitter buffer / packet reordering / loss concealment** | None relay-side; `seq`/`ts_48k` are parsed for `tracing::trace!` only (`handler.rs:996-1003`). Playout is client-side (`desktop/src-tauri/src/huddle/playout.rs`) |
| **Server-side mute / kick / moderation** | No mute state, no kick message, no moderator role on `Room` (`room.rs:157-170`) |
| **Active-speaker / dominant-talker election** | Explicitly forbidden from consuming `level_dbov` (`wire.rs:16-20`); done client-side (`playout.rs:220`) |
| **Per-room bitrate/frame-rate limits or admission-time quality control** | Only a byte cap (4096/frame) and a queue depth (8) |
| **Push-to-talk, hand-raise, reactions in-huddle** | None |
| **Huddle metrics** | Zero `metrics::` calls in `audio/` — no gauge for active rooms/peers, no counter for dropped frames, no join-failure counter. The only metric in the whole group is `mesh_fence_rejections_total` (`tunnel/directory.rs:483`) |
| **Bandwidth accounting / per-peer rate limiting** | None |
| **A `speakers` control message** | Promised by the doc comment at `room.rs:33`; never produced |
| **`PeerCtrl::Close`** | Variant exists (`room.rs:35`) and is handled (`handler.rs:1112`) but has **zero producers** — the graceful per-peer shutdown path it implies is not wired |
| **Session resumption / reconnect continuity** | A rejoin is a fresh handshake with a fresh peer UUID and (usually) a different index |
| **Kind 48106 (huddle guidelines)** | Defined `kind.rs:538`, allowlisted for labelling `handlers/event.rs:49`, but never produced or consumed by the relay. Client-side only (`desktop/src-tauri/src/huddle/agents.rs:31`) |
| **Kind 48100 (huddle started)** | The relay only *reads* it, to validate a claimed parent (`handler.rs:1176-1186`). The producer is the desktop client (`desktop/src-tauri/src/huddle/mod.rs:252`) |

---

#### 3. What the tunnel lane delivers

| Capability | Where | Completeness |
|---|---|---|
| Redis fenced session directory (acquire / renew / release / lookup / validate) with a non-expiring generation counter | `tunnel/directory.rs:180-439` | Complete and well-tested (7 tests, 5 of which need live Redis) |
| Typed fence-rejection taxonomy shared with `/_mesh` and the huddle wire | `directory.rs:348-439`; `join.rs:982-1017` | Complete |
| Reliable-stream join routing (own locally vs. open a fenced bi-stream to the owner) | `reliable.rs:78-126` | Complete |
| Owner-side inbound accept with structural validation | `reliable.rs:135-172` | Complete |
| 1 MiB-chunked ordered framing with a community-bearing inner header | `reliable.rs:274-294`, `:412-492` | Complete |
| Per-frame Redis fence validation that **fails the session** on mismatch | `reliable.rs:326-389` | Complete |
| Observable lease renewer | `reliable.rs:571-657` | Implemented; **never spawned** |

##### 3.1 What the tunnel lane does **not** deliver

- **No product consumer.** `mesh_boot.rs:299-303` accepts an inbound reliable stream,
  logs `"reliable stream accepted; no session consumer wired — closing"`, and closes
  it. `mesh_boot.rs:206-215` states plainly that renewal attaches on the join path
  "which lands with the first product session consumer".
- **No join-side product caller.** The only route reaching
  `ReliableStreamRouter::join` is the demo-gated `POST /_mesh/demo/echo`
  (`api/mesh_demo.rs:73`, `:95`), documented as "Not a product flow"
  (`api/mesh_demo.rs:21-22`).
- **No renewal in the demo path either** — `api/mesh_demo.rs:11-15` documents that
  the `Owned` arm deliberately lets the lease die with its 30 s TTL, and instructs
  operators to drive the second pod inside that window.
- **No retransmission, ACK, flow control, or backpressure signalling** beyond what
  QUIC provides. There is no seq/ack in `ReliableWireFrame` (`reliable.rs:412-422`).
- **No goose/berd bridge.** The module doc says it is "for berd ↔ goose-server
  sessions" (`reliable.rs:1`) but no such wiring exists in the repo.
- `SessionDirectory::takeover` (`directory.rs:233`) and
  `ReliableMeshStream::with_community` (`reliable.rs:263`) have **zero callers**;
  `SessionDirectory::known_generation` (`directory.rs:324`) has test-only callers.

---

#### 4. What the conformance seam delivers

| Capability | Where | Completeness |
|---|---|---|
| `Tracer` trait + two impls (`NoopTracer`, `JsonlTracer`) | `conformance/tracers.rs:14-72` | `NoopTracer` in use; `JsonlTracer` **never instantiated anywhere** |
| Label projection (community, host, actor, msg-id, channel) | `conformance/mod.rs:47-91` | Complete, with one documented delta (§4.2) |
| Claimed-vs-resolved community separation | `conformance/mod.rs:94-117` | Complete |
| Write-path emits (`WriteInsert`/`WriteInsertGlobal`/`WriteDuplicate`/`SanitizedError`/`AuthCheck`) | consumed from `handlers/ingest.rs:2218`, `:2334-2348`, `:2468-2485` | Wired on the ingest path |
| Read-path emits (`ReadMessageRows`, `ReadByIdRows`) with row-community projection | `conformance/mod.rs:265-330`, called `req.rs:355`, `:671` | Wired |
| `ReadHostFeedRows` | schema `buzz-conformance/src/lib.rs:250` | **No emitter in the relay** — only the conformance crate's own proptest references it |
| Row-lookup coverage guard (`MissingLookup` → `ImplBug`) | `conformance/mod.rs:216-255` | Complete; 4 tests |
| `EmitGuard` RAII coverage-breach guard | `conformance/mod.rs:334-414` | Complete; 2 self-tests (`:454-528`) |
| `IngestError → SanitizedReason` closed-alphabet mapping | `conformance/mod.rs:422-429` | Complete, exhaustive match |

##### 4.1 Is it active in production? No.

`state.rs:794-798` binds `Arc::new(crate::conformance::NoopTracer)` unconditionally
at `AppState` construction. There is no config flag, env var, or feature gate that
swaps it. `NoopTracer::record` is `fn record(&self, _step: TraceStep) {}`
(`tracers.rs:22-24`). Consequences:

- Every emit site still **constructs** its `TraceAction` (allocating `String`s for
  labels, cloning `AbstractState`) and then discards it. The claimed "zero cost"
  (`tracers.rs:11-13`, `state.rs:615`) is not achieved — the work is done, only the
  write is elided. There is no `#[cfg(feature = ...)]` anywhere in the module.
- The `EmitGuard`'s `ImplBug` on a coverage breach is recorded into `NoopTracer` and
  therefore **silently discarded** in production. The guard is a test-time
  mechanism that is armed in production for no observable benefit.
- `JsonlTracer` — the only impl that would make the seam observable — has **zero
  callers in the workspace**. `state.rs:795-797` promises "Conformance tests
  overwrite this with a JsonlTracer after construction (see test helpers in
  `crates/buzz-test-client` once those land)". Those helpers have not landed: grep
  for `JsonlTracer` finds only the definition and two doc comments.
- `conformance/mod.rs:47-48` documents wire points in `req.rs`/`event.rs` as
  "held back as additive patch for Eva to apply onto Max's req.rs writes — see
  thread `c882c9b1…`". The `req.rs` sites **have** landed (`req.rs:144`, `:355`,
  `:671`); the doc comment is stale.

##### 4.2 Documented deltas in the conformance module

| Doc claim | Line | Actual code |
|---|---|---|
| "`actor` is the lower 16 bytes of `blake3(pubkey_bytes)` as a hex string" | `conformance/mod.rs:51-53` | `actor_label` takes the **first 16 hex chars of the pubkey hex** — no hashing (`:70-74`). The rationale at `:63-69` explains why this is acceptable, but the doc comment 10 lines earlier still says blake3 |
| "The trace carries opaque labels … never … wall-clock timestamps" | `conformance/mod.rs:12-15` | Honoured: `TraceStep` has no timestamp field (`buzz-conformance/src/lib.rs:290-299`). `bound_host` is a full host string, not opaque (`:58`), and `channel_label` is a direct UUID wrap — both deliberate (`:87-88`) |
| "the build can have the compiler eliminate them entirely behind a feature flag" | `tracers.rs:11-13` | No such feature flag exists in `crates/buzz-relay/Cargo.toml` (`[features] dev = ["buzz-auth/dev"]` only) |

---

#### 5. Mesh boot seam — what it delivers

| Capability | Where |
|---|---|
| Single construction point for all mesh machinery, `None` when off | `mesh_boot.rs:411-520` |
| iroh endpoint bind with a boot-unique keypair → boot-unique `RuntimeId` | `mesh_boot.rs:423-431` |
| Relay-key-attested ready record published to Redis + readiness-gated heartbeat | `mesh_boot.rs:445-472` |
| Advertise-address preference chain (`BUZZ_MESH_ADVERTISE_ADDR` → `POD_IP`+bound port → all endpoint IPs) | `mesh_boot.rs:379-402` |
| Static capability advertisement (all three profiles in one binary) | `mesh_boot.rs:369-377` |
| Per-profile inbound dispatcher over a single transport slot | `mesh_boot.rs:44-130` |
| Immediate reconcile pass so seed peers are dialed at boot | `mesh_boot.rs:478` |
| Drain watcher: gossip `draining=true` then drain owned huddles | `mesh_boot.rs:481-496` |
| Shared single `GenerationFloor` between the datagram hot path and huddle teardown | `mesh_boot.rs:149-166`, `:236-245`; pinned `mesh_boot.rs:665-737` |
| `BUZZ_MESH_DEMO_ECHO` reliable-stream echo consumer | `mesh_boot.rs:307-364` |

##### 5.1 Absent from the boot seam

- **No mesh readiness gate on the huddle path.** `boot_mesh` returns a handle as
  soon as the endpoint binds and the ready record publishes; there is no wait for
  peer connectivity. A join that resolves `RemoteOwner` before the owner peer is
  dialed fails with `huddle_owner_unreachable` (`handler.rs:487-503`).
- **`MeshHandle.membership` is written but never read.** Populated at
  `mesh_boot.rs:501`, exposed at `:141`, with zero consumers. `/_mesh` reads the
  private `runtime` field instead (`mesh_boot.rs:172-174`).
- **No mesh peer count / session gauge.** `/_mesh` returns whatever
  `MeshStatus` carries, unauthenticated (`router.rs:399-406`).


## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Features

Crate description: "Inter-relay QUIC mesh: transport, membership, and the fenced
wire contract" (`Cargo.toml:3`). 3,169 LOC across 10 files; 32 unit tests, all
passing, none `#[ignore]`d (verified by running `cargo test -p buzz-relay-mesh --lib`:
`32 passed; 0 failed; 0 ignored`).

---

#### 1. Opt-in status: **off by default, hard no-op when off**

`BUZZ_MESH` must be an explicit `on`/`true`/`1`; absent, `off`, or any typo means
disabled (`config.rs:498-500`). When disabled, `boot_mesh` returns `Ok(None)` before
touching anything (`mesh_boot.rs:417-419`) — no UDP bind, no Redis write, no spawned
task. `AppState.mesh` stays an empty `OnceLock` (`state.rs:627`), so
`AppState::mesh()` returns `None` (`state.rs:812`) and every consumer takes its
single-instance path.

Two tests pin this: `mesh_off_boots_nothing` points the Redis pool at an unroutable
`redis://127.0.0.1:1` so any accidental Redis touch would fail
(`mesh_boot.rs:526-541`), and `mesh_defaults_off_when_env_absent`
(`mesh_boot.rs:544-555`) asserts the fail-safe reading.

Nothing in `.env.example`, `deploy/`, or the helm chart sets `BUZZ_MESH` — verified
by grep. **No deployment in this repo turns the mesh on.**

---

#### 2. What ships and works today

| Capability | Where | Evidence |
|---|---|---|
| iroh/QUIC endpoint bind with boot-unique ed25519 identity | `endpoint.rs:19-44` | test `two_endpoints_connect_with_alpn_and_authenticated_identity` (`endpoint.rs:180-187`) |
| ALPN-pinned, mutually-authenticated peer connections, iroh relays disabled | `endpoint.rs:35`, `peer.rs:50-55` | same test |
| Length-delimited reliable bi-streams carrying `MeshStreamFrame` | `peer.rs:135-192` | `reliable_stream_roundtrip_carries_mesh_stream_frame` (`endpoint.rs:189-227`) |
| QUIC datagrams carrying `MeshDatagram`, size-checked pre-send | `peer.rs:105-129`, `lib.rs:213-226` | `datagram_roundtrip_carries_mesh_datagram` (`endpoint.rs:229-237`), `oversized_datagram_is_rejected_before_send` (`endpoint.rs:239-254`) |
| Opus-sized datagram loss/order gate (64 frames, loopback) | — | `opus_sized_datagrams_clear_empirical_local_loss_gate` (`endpoint.rs:256-291`) |
| Datagram header-overhead budget (≤64 B) | — | `datagram_header_overhead_within_budget` (`wire.rs:271-284`) |
| Redis ready registry: publish / clear / scan, TTL 3× refresh | `registry.rs:182-257` | `ready_key_is_stable_and_namespaced` (`registry.rs:317-322`), `expiry_is_three_refreshes` (`:324-327`), `ready_record_roundtrips_json` (`:336-346`) |
| Relay-key schnorr attestation of the boot-unique endpoint key | `registry.rs:29-96` | `ready_record_attestation_verifies_and_binds_runtime_pubkey` (`:348-356`), `attestation_rejects_signature_for_other_runtime` (`:358-364`) |
| Deployment-anchored seed admission (foreign relay identity rejected, unanchored = fail-closed) | `membership.rs:85-118` | 4 tests, `membership.rs:425-471` |
| Scuttlebutt digest/delta membership gossip on a per-connection control stream | `membership.rs:208-247`, `runtime.rs:501-551` | `digest_delta_only_sends_newer_records` (`gossip.rs:238-251`), `warm_pair_connects_and_gossips_membership` (`runtime.rs:721-...`) |
| Last-version-wins record merge | `membership.rs:120-153` | `stale_gossip_record_is_ignored` (`membership.rs:474-479`), `apply_delta_ignores_stale_versions` (`gossip.rs:253-266`) |
| Accept-loop admission gate with one registry rescan | `runtime.rs:262-323` | exercised indirectly by `connected_pair` (`runtime.rs:645-670`) |
| Warm full-mesh reconcile/dial loop | `runtime.rs:285-355` | `warm_pair_connects_and_gossips_membership` |
| Deterministic simultaneous-dial tie-break | `runtime.rs:204-213` | `tie_break_is_symmetric` (`runtime.rs:846-856`), `simultaneous_dial_converges_to_one_connection` (`runtime.rs:858-...`) |
| Inbound fan-out to a single `InboundHandler` slot | `runtime.rs:196-198`, `:358-372`, `:461-470` | `transport_datagram_reaches_inbound_handler` (`runtime.rs:745-...`), `transport_session_stream_reaches_inbound_handler` (`runtime.rs:775-...`) |
| Typed error for sending to an unconnected peer | `runtime.rs:167-175` | `send_to_unconnected_peer_is_typed_error` (`runtime.rs:823-843`) |
| Drain: gossip `draining=true`, stop being dialed | `membership.rs:385-388`, `runtime.rs:302` | `begin_drain_updates_local_record` (`membership.rs:482-489`) |
| Readiness-gated registry heartbeat with clear-on-not-ready | `registry.rs:295-312`, `runtime.rs:594-608` | `heartbeat_starts_unpublished` (`registry.rs:329-334`) |
| Per-peer counters surfaced at `/_mesh` | `membership.rs:249-283`, `status.rs` | `counters_are_reflected_in_status` (`membership.rs:481-...`) |
| Phi-accrual-style suspicion (exponential approximation) | `gossip.rs:168-220` | `phi_rises_as_heartbeats_age` (`gossip.rs:268-278`) |
| Wire-version and gossip-payload-version rejection | `wire.rs:182-188`, `gossip.rs:66-77` | `unknown_version_rejected` (`wire.rs:246-257`), `gossip_payload_roundtrips` (`gossip.rs:...`) |

---

#### 3. What is stubbed, inert, or dead

| Item | Status | Location |
|---|---|---|
| `gossip::GossipState` (+ 7 methods) | complete duplicate of `MeshMembership`'s scuttlebutt logic; **zero callers** outside its own tests | `gossip.rs:81-166` |
| `PeerInfo.load` | structurally always `0.0` — no writer anywhere | `lib.rs:137-138`, `gossip.rs:35`, `runtime.rs:566` |
| `MeshCounters.stale_generation_rejections` | always 0 in production; sole writer's only caller is a test | `membership.rs:285-293`, test `:486` |
| `mesh_fence_rejections_total{reason=…}` metric | documented, does not exist | `lib.rs:102-109` |
| `MeshError::Disabled`, `MeshError::PeerDraining` | dead variants — zero constructors repo-wide | `lib.rs:94`, `lib.rs:119` |
| `RuntimeAttestation::verify` | zero callers | `registry.rs:48-50` |
| `ReadyHeartbeat::shutdown` / `record` / `published` | zero production callers | `registry.rs:287-312` |
| `MeshRuntime::shutdown` | zero production callers — loops leak to process exit | `runtime.rs:155-164` |
| `MeshRuntime::connected_peers` | zero callers outside in-crate tests | `runtime.rs:138-146` |
| `MeshEndpoint::endpoint()` | zero callers anywhere | `endpoint.rs:51-53` |
| `MeshPeer::counters()` / `PeerCounters` | atomics incremented, never read | `peer.rs:10-15`, `:73-75` |
| `MeshMembership::with_phi_suspect_threshold` | zero callers — 8.0 is unoverridable | `membership.rs:66-69` |
| `RelayMeshMembership::peers()` / `local_runtime_id()` | zero production callers; `MeshHandle.membership` is written (`mesh_boot.rs:501`) and never read | `membership.rs:359-383` |
| `proto_version` / `capabilities` negotiation | advertised, never compared | `registry.rs:110`, `mesh_boot.rs:371-377` |
| Peer eviction | none — membership table only grows | `membership.rs` (no `remove`/`retain`) |
| Reconnect backoff | none | `runtime.rs:328-355` |
| `hmac`, `futures-util` deps | declared, **zero `use` sites** | `Cargo.toml:20,25` |
| `proptest` dev-dep | declared, zero property tests | `Cargo.toml:29` |
| Peer selection / gossip fan-out sampling | none — gossip is 1:N to all connected peers | `runtime.rs:571-585` |

Also inert at the consumer boundary: **no product session consumer is wired**.
`wire_mesh_consumers` accepts a reliable stream, logs
`"reliable stream accepted; no session consumer wired — closing"`
(`mesh_boot.rs:288-292`), unless `BUZZ_MESH_DEMO_ECHO` is on. And
`mesh_boot.rs:216-220` notes owner-side lease renewal "lands with the first product
session consumer" — i.e. not yet.

---

#### 4. Delivered end-to-end paths (via `buzz-relay`)

Three inbound profiles are wired (`mesh_boot.rs:229-299`):

1. **`RealtimeMedia` datagrams → huddle audio fan-in.** `MeshAudioRouter` over a
   shared `GenerationFloor` (`mesh_boot.rs:243-254`). This is the one path with a
   real product consumer (cross-pod huddle audio, `audio/mesh.rs`, `audio/join.rs`).
2. **`HuddleControl` streams → `HuddleControlAcceptor::accept_inbound`**
   (`mesh_boot.rs:256-278`) — owner-side peer register/unregister.
3. **`ReliableStream` streams → `ReliableStreamRouter::accept_inbound`**
   (`mesh_boot.rs:280-299`) — accepted and fence-validated, then closed (or echoed
   under the demo flag). Goose/berd tunnels are the intended consumer and are absent.

Operator surface: `GET /_mesh` on the health listener (`router.rs:230`, handler
`router.rs:396-406`) and the double-gated `POST /_mesh/demo/echo`
(`router.rs:123`, `api/mesh_demo.rs`).

---

#### 5. The five `mesh_*` examples — **they do not exercise this crate**

The brief describes `crates/buzz-relay/examples/mesh_*.rs` as exercising
`buzz-relay-mesh`. **That is not the case.** All five import `mesh_llm_sdk` /
`mesh_llm_host_runtime` — the external **MeshLLM** peer-to-peer LLM-serving project
(git dev-deps, `crates/buzz-relay/Cargo.toml:84-85`, tag `v0.73.1`). Verified: grep
for `buzz_relay_mesh` across `crates/buzz-relay/examples/` returns **zero matches**.

Two unrelated things named "mesh" live in one crate, both spelled `mesh_*`:

| Example | LOC | What it actually proves | Subject |
|---|---|---|---|
| `mesh_admission_smoke.rs` | 452 | MeshLLM **owner-allowlist** admission: 3 subprocesses (serve + trusted client + non-member); possession of an invite token admits nobody, only allowlisted `OwnerKeypair` ids route inference (`mesh_admission_smoke.rs:1-30`, verdict matching `:262-283`) | mesh-llm |
| `mesh_agent_e2e.rs` | 421 | share-compute → agent-env preset → real `buzz-agent` over ACP stdio → inference; 4 permutations incl. a context-fit rejection (P3, `:106-134`) and `buzz-dev-mcp` tool use (P4, `:136-166`) | mesh-llm |
| `mesh_serve_client_smoke.rs` | 171 | MeshLLM serve node + client node joined by invite token, one completion routed client→serve (`:1-24`) | mesh-llm |
| `mesh_stack_smoke.rs` | 121 | tokio worker **stack-size** repro/fix: 2 MiB dies on the stack guard, 8 MiB completes a model download (`:1-30`); mirrors `buzz_lib::mesh_llm::MESH_WORKER_STACK_SIZE` (`:35-37`) | mesh-llm |
| `mesh_serve_smoke.rs` | 96 | single-node MeshLLM serve-and-self-consume; documents that `console_ui(true)` is load-bearing for `serve::start` readiness polling at mesh rev `bd16da4` (`:22-31`) | mesh-llm |

They are all explicitly **hardware-gated and not CI** — stated in each header
(`mesh_admission_smoke.rs:26-27`, `mesh_agent_e2e.rs:22-24`,
`mesh_serve_client_smoke.rs:25-26`, `mesh_stack_smoke.rs:26-27`) and reachable only
via `just mesh-e2e-hardware` (`justfile:327-331`), `just mesh-e2e-admission`
(`justfile:335-339`), `just mesh-e2e-confidence` (`justfile:342-349`).

##### CI-gate check

**No CI workflow gates any of the five examples, and none gates this crate's
tests.** Verified:

- Every `mesh` hit in `.github/workflows/{ci,release,signed-macos-canary,windows-canary}.yml`
  is MeshLLM build plumbing (resolving the `mesh-llm-sdk` rev from `Cargo.lock`,
  caching llama native libs, `--features mesh-llm` desktop builds) — e.g.
  `ci.yml:1015-1052`, `release.yml:141-181`.
- CI's Rust lint step is `just clippy` (`ci.yml:94`) = `cargo clippy --workspace
  --all-targets -- -D warnings` (`justfile:106-107`). `--all-targets` **compiles**
  the five examples (and therefore forces the git dev-deps to build) but runs
  nothing.
- CI's unit step is `just test-unit` (`ci.yml:116`), which runs only
  `-p buzz-core -p buzz-auth --lib`, `-p buzz-db --lib`, `-p buzz-conformance`,
  `-p buzz-push-gateway` (`justfile:275-295`). **`buzz-relay-mesh` is not in that
  list.**
- The backend-integration job archives `-p buzz-relay -p buzz-test-client --lib`
  (`ci.yml:336-342`) — also not this crate.

**Net: all 32 `buzz-relay-mesh` unit tests compile in CI and never execute.** They
pass locally (verified), including the four that stand up real loopback iroh
endpoints and the two multi-runtime gossip/tie-break tests.

---

#### 6. Test inventory (32 total, 0 `#[ignore]`d)

| File | Tests | Kind |
|---|---|---|
| `endpoint.rs` | 5 | `#[tokio::test]`, real loopback iroh endpoints |
| `membership.rs` | 7 | pure `#[test]`, real `nostr::Keys::generate()` signing |
| `registry.rs` | 6 | pure `#[test]`; `heartbeat_starts_unpublished` builds a pool but never dials |
| `runtime.rs` | 6 | 5 `#[tokio::test]` (2-runtime loopback meshes) + 1 pure |
| `gossip.rs` | 4 | pure |
| `wire.rs` | 4 | pure roundtrip/negative/budget |
| `lib.rs`, `peer.rs`, `status.rs` | 0 | — |

Untested behaviour of note: no test covers `is_known_peer`'s registry-rescan branch
(`runtime.rs:309-320`), the `remove_peer` path (`runtime.rs:267-281`), `dial_peer`'s
bad-addr/bad-id branches (`runtime.rs:328-348`), `ReadyRegistry::publish_ready`/
`clear_ready`/`scan_ready` against a live Redis, or `spawn_registry_heartbeat`
(`runtime.rs:594-608`). All Redis paths are effectively untested.


## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: Features

Five capabilities, split cleanly by whether the relay actually drives them.

| # | Feature | Where it lives | Wired into the relay? |
|---|---|---|---|
| F1 | Trace emission (the seam) | `crates/buzz-relay/src/conformance/mod.rs`, `handlers/ingest.rs`, `handlers/req.rs` | **Yes** — but always into `NoopTracer` (`crates/buzz-relay/src/state.rs:798`) |
| F2 | Coverage-breach guard (`EmitGuard`) | `crates/buzz-relay/src/conformance/mod.rs:344-415` | **Yes** — armed at one seam only (`handlers/ingest.rs:1408-1412`) |
| F3 | Row-community projection | `crates/buzz-relay/src/conformance/mod.rs:186-333` | **Yes** — both read lanes in `req.rs` |
| F4 | Replay checking (`check_trace`) | `crates/buzz-conformance/src/checker.rs:74` | **No** — zero production callers |
| F5 | Test lanes: property tests + golden fixtures | `crates/buzz-conformance/tests/` | Test-only by construction |

---

### F1 — Trace emission

Nine actions in the vocabulary (`src/lib.rs:179-261`); the relay emits **six** of them.

| Action | Relay emit site(s) | Producer |
|---|---|---|
| `SanitizedError` | `handlers/ingest.rs:1436-1443` | terminal-error wrapper, one place for all `Err` returns |
| `AuthCheck` (write path) | `handlers/ingest.rs:1794-1802` | after `check_channel_membership` (`:1785-1787`) |
| `AuthCheck` (read path) | `handlers/req.rs:143-151` → `conformance/mod.rs:148-168` | after `db.is_member` (`req.rs:137-142`), only on the cache-miss branch (`:134-136`) |
| `WriteInsertGlobal` | `handlers/ingest.rs:139-146` (product feedback, called at `:1573`), `:2215-2222` (push lease), `:2506-2509` (trailing dispatch, channel-less arm) | |
| `WriteInsert` | `handlers/ingest.rs:2362-2366` (reaction lane), `:2470-2474` (trailing dispatch) | |
| `WriteDuplicate` | `handlers/ingest.rs:2368-2372` (reaction lane), `:2501-2505` (trailing dispatch) | |
| `ReadMessageRows` | `handlers/req.rs:355-361` → `conformance/mod.rs:265-298` | non-search REQ lane, per filter |
| `ReadByIdRows` | `handlers/req.rs:671-677` → `conformance/mod.rs:300-333` | NIP-50 search refetch lane |
| `ImplBug` | `conformance/mod.rs:404-414` (guard drop), `:284-296` / `:319-331` (projection miss) | |
| `ReadHostFeedRows` | **none** — grep across `crates/buzz-relay/` finds no producer | |

Emission is per-decision, synchronous, and untimed. There is no batching, no sampling, no
async queue; `emit` is `tracer.record(TraceStep::new(action, state))`
(`conformance/mod.rs:127-129`).

**One emit arm is unreachable.** `handlers/ingest.rs:2452-2458` returns early on
`!was_inserted`, so the trailing dispatch's `(Some(ch), false) => WriteDuplicate` arm
(`:2501-2505`) can never execute. Only the reaction lane's `WriteDuplicate` (`:2368`) is
reachable, and only when `buzz_db::ReactionEventInsertOutcome::Inserted` carries
`was_inserted: false` (`:2348-2351`) — the explicit `Duplicate` outcome returns earlier at
`:2341-2347` without emitting.

**Fan-out is deliberately not a feature.** `handlers/event.rs:405-410` documents the decision:
the spec has no fan-out action, and acceptance is already recorded at the ingest seam.

---

### F2 — Coverage-breach guard

`EmitGuard::arm` (`conformance/mod.rs:383-400`) returns a pair: the guard, and a
`CountingTracer`-wrapped `Arc<dyn Tracer>` the caller must use in place of the original. The
wrapper bumps a `Relaxed` `AtomicUsize` on each `record` (`:367-373`); `Drop` emits
`ImplBug { kind }` on the **inner** tracer when the count is still zero (`:403-415`).

Callers never disarm — the design note at `:337-343` is explicit that production paths just
call `record` as before.

**Armed at exactly one seam:** `ingest_event` (`handlers/ingest.rs:1408-1412`, `kind =
"ingest_event_exited_without_trace"`). The REQ path arms nothing; `handle_req` builds a
`trace_state` (`handlers/req.rs:116-118`) but no guard, so a REQ that returns before any read
emits produces no `ImplBug`. Six paths through `ingest_event_inner` return `Ok` without any
emit, so the guard fires `ImplBug` on each:

| Path | Line | Kind(s) |
|---|---|---|
| Command routing | `ingest.rs:1560-1562` | 7 kinds: 30620, 41010, 41011, 41012, 46020, 46030, 46031 (`buzz-core/src/kind.rs:743-754`) |
| NIP-56 report | `ingest.rs:1587-1596` | 1984 (`buzz-core/src/kind.rs:267`) |
| Moderation commands | `ingest.rs:1605-1614` | 5 kinds (`buzz-core/src/kind.rs:316-325`) |
| Relay-admin kinds | `ingest.rs:1834-1842` | 4 kinds (`buzz-core/src/kind.rs:723-731`) |
| NIP-43 leave request | `ingest.rs:1846-1928` | 28936 (`buzz-core/src/kind.rs:344`) |
| Reaction duplicate | `ingest.rs:2341-2347` | 7 (`buzz-core/src/kind.rs:58`) |
| Non-reaction duplicate | `ingest.rs:2452-2458` | all stored kinds |

That is seven, not six. Each returns `accepted: true` or `accepted: false` with no trace step,
so the guard's `ImplBug` is the *only* record for the request — which the checker classifies as
`CoverageBreach` (`src/transitions.rs:277-279`). The guard therefore works as designed; the
gap is that a legitimate accepted write is indistinguishable from a forgotten emit site.

---

### F3 — Row-community projection ("(B) strategy")

The read seam's non-tautology mechanism, documented at `conformance/mod.rs:170-185`:

- Channel-less rows (`row.channel_id == None`) → project as `resolved_community`
  (`:191`).
- Channel-scoped rows → look up the row's **own** `channel_id` in a map from
  `buzz_db::Buzz::communities_of_channels` (`:192-194`). A hit yields the looked-up label; a
  miss yields `None`.
- `project_row_communities` (`:234-262`) turns the first `None` into
  `RowCommunityProjection::MissingLookup { kind: "row_community_lookup_missing",
  first_missing_channel }` (`:246-253`), which the record helpers convert into `ImplBug`
  (`:284-296`, `:319-331`).

The key property is that the label comes from the row, not from the query's `WHERE` clause —
so a channel-scoped row cannot masquerade as channel-less to dodge the lookup
(`:182-185`). Two unit tests pin it: `project_row_communities_channel_scoped_uses_lookup_label`
asserts the foreign label survives and is *not* replaced by resolved
(`conformance/mod.rs:621-648`), and `project_row_communities_channel_scoped_missing_is_breach`
asserts a miss is a breach, not a substitution (`:650-673`).

**Cost:** the projection requires an extra `db.communities_of_channels` round-trip per REQ
filter (`handlers/req.rs:345`) and per search page (`:661`), gated on
`trace_state.is_some()` (`:334`, `:649`) — a condition independent of which tracer is bound.
See the debt doc.

---

### F4 — Replay checking

`check_trace` (`src/checker.rs:74-131`) walks a `Scenario` and returns the first
`TransitionError`. Stages, per the doc at `:63-73`: empty guard → schema check → bootstrap →
per-step `check_step` → coverage-set diff.

**No production wiring exists.** Repo-wide grep for `check_trace` returns hits only in the
crate's own source, its two test files, and its two markdown docs. Nothing constructs a
`Scenario` from a live relay, and `JsonlTracer` — the only tracer that persists anything —
has zero instantiation sites outside its own definition (`conformance/tracers.rs:30-45`;
grep for `JsonlTracer` returns only the definition, the re-export at `mod.rs:46`, two doc
comments at `state.rs:616`/`:795`, and two markdown mentions). So the end-to-end pipeline the
crate is built for (relay → JSONL → checker) is not assembled anywhere in the repo.
`crates/buzz-conformance/LIMITS.md:120-125` describes it as the "next ratchet".

---

### F5 — Test lanes

22 tests, all passing (`cargo test -p buzz-conformance`: 9 + 7 + 6).

**Unit lane — 9 tests in `src/checker.rs::tests` (`:135-337`).** One passing case
(`write_insert_then_read_with_only_resolved_rows_passes`, `:172-207`) plus one bite case per
failure mode. Helpers `cid`/`ch`/`state`/`step` at `:144-162`.

**Property lane — 7 proptest cases, 128 cases each
(`ProptestConfig::with_cases(128)`, `tests/proptest_checker.rs:193`).**

| ID | Test | Line | Asserts |
|---|---|---|---|
| P2 | `clean_trace_is_accepted` | `:199` | no false rejects on 1..=12 clean actions |
| P1 | `foreign_row_label_is_rejected` | `:213` | `NonInterference` across all three read variants |
| P3a | `auth_allow_foreign_claim_bites` | `:272` | `IllegalTransition` |
| P3b | `auth_deny_any_claim_is_ok` | `:304` | `Deny` never bites on the claim axis |
| P4 | `impl_bug_bites_coverage_breach` | `:328` | `CoverageBreach` |
| P5 | `state_flip_bites_state_mismatch` | `:354` | `StateMismatch` on any one flipped field |
| P6 | `check_trace_is_deterministic_and_total` | `:406` | two runs stringify identically; no panic |

The lane explicitly refuses a parallel oracle — rationale at `:9-25`: a reference
implementation "would just be a copy of the checker". Community/channel pools are 3 wide
(`POOL`, `:51`) so foreign-vs-resolved collisions are frequent (`:44-49`).

**Fixture lane — 6 tests, 4 committed JSONL files.** `assert_fixture_matches`
(`tests/replay_fixtures.rs:206-235`) serializes the typed builder, byte-compares against the
committed file, then re-parses and compares structurally (`:233-234`).
`BUZZ_CONFORMANCE_UPDATE=1` regenerates instead of comparing (`:210-214`).

| Fixture | Builder | Test | Verdict |
|---|---|---|---|
| `good.jsonl` | `:75-101` | `good_trace_passes_check` `:238-250` | `Ok(())`, with 3 required actions (`:241-244`) |
| `bad_host_channel_mismatch.jsonl` | `:107-129` | `:253-264` | `IllegalTransition` |
| `bad_coverage_breach.jsonl` | `:134-141` | `:267-278` | `CoverageBreach` |
| `bad_foreign_row_leak.jsonl` | `:154-168` | `:281-292` | `NonInterference` |

Two fixture-free tests round out the lane: `empty_trace_is_coverage_breach` (`:295-304`) and
`missing_required_action_is_coverage_breach` (`:307-320`).

**Relay-side lane — 9 tests in `crates/buzz-relay/src/conformance/mod.rs::tests`
(`:431-726`)**, covering the guard (`:467-495`, `:498-527`), the REQ AuthCheck verdict table
(`:530-570`, `:572-593`), and the four projection cases (`:596-618`, `:621-648`, `:650-673`,
`:675-696`, `:698-725`). Plus one in `handlers/ingest.rs:2530-2565`
(`feedback_success_action_satisfies_ingest_emit_guard`) proving the feedback path satisfies
the guard.

These relay-side tests are **not** in the unit gate: `just test-unit` runs `-p buzz-core
-p buzz-auth --lib`, `-p buzz-db --lib`, `-p buzz-conformance`, `-p buzz-push-gateway`
(`justfile:278-293`) and the shell fallback matches (`scripts/run-tests.sh:81-102`). Neither
list includes `buzz-relay`, so the emitter-side proofs run only via
`run_integration_tests`' catch-all `cargo test --test '*'` (`scripts/run-tests.sh:118-120`),
which matches integration targets, not `--lib` unit tests. `crates/buzz-conformance/LIMITS.md:109`
instructs `cargo test -p buzz-relay --lib conformance::` as a required CI surface; no justfile
recipe or workflow runs it.


## Module: buzz-acp — harness core & orchestration (`crates/buzz-acp/src`)
### Aspect: Features

#### End-to-end startup sequence (`tokio_main`, `lib.rs:1239-1700`)

| Step | Line | Notes |
|---|---|---|
| install rustls ring provider | `lib.rs:1241-1243` | `.expect(...)` — panics if it fails |
| subcommand short-circuit | `lib.rs:1245-1274` | `models` / `auth-methods` / `authenticate` exit before tracing init |
| tracing init | `lib.rs:1276-1281` | `EnvFilter` from env, default `buzz_acp=info` |
| `Config::from_cli()` | `lib.rs:1290` | |
| setup-mode branch | `lib.rs:1298-1303` | `BUZZ_ACP_SETUP_PAYLOAD` present ⇒ `setup_mode::run_setup_listener`, agent pool never starts |
| observer handle | `lib.rs:1307-1316` | in-process only when `config.relay_observer` |
| agent pool | `lib.rs:1317-1321` | eager `initialize_agent_pool` unless `lazy_pool` |
| startup watermark | `lib.rs:1330-1333` | captured **before** relay connect |
| relay connect (NIP-42) | `lib.rs:1343-1346` | `HarnessRelay::connect` with optional NIP-OA tag |
| `set_startup_watermark` | `lib.rs:1352-1354` | best-effort; failure only loses the startup-window guard |
| membership subscription | `lib.rs:1358-1361` | fatal on error |
| owner resolution | `lib.rs:1370-1393` | |
| observer control subscription | `lib.rs:1395-1435` | only if owner resolved |
| channel discovery | `lib.rs:1437-1443` | fatal on error |
| rule construction | `lib.rs:1445-1474` | `mentions` / `all` / `config` |
| per-channel subscribe | `lib.rs:1480-1489` | per-channel failure is a warning, not fatal |
| presence `online` | `lib.rs:1511-1515` | published after subscriptions as a readiness boundary |
| build `PromptContext` | `lib.rs:1530-1573` | |

#### Wired capabilities

| Capability | Evidence | State |
|---|---|---|
| Mention-triggered turns | rules built `lib.rs:1445-1454`, matched `lib.rs:2172-2181` | wired |
| Per-channel batching (drain backlog → one prompt) | `queue.flush_next()` `lib.rs:2892` | wired |
| N parallel agent subprocesses (1–32) | `initialize_agent_pool` `lib.rs:3741-3846` | wired |
| Session affinity per channel | `pool.try_claim(Some(channel_id))` `lib.rs:2902`, `affinity_hit` logged `lib.rs:2911` | wired |
| Partial pool startup tolerated | `lib.rs:3826-3840` — zero live agents is fatal, fewer than requested is a warning | wired |
| Lazy pool (subscribe first, spawn on first accepted event) | `lib.rs:1317-1321`, wake path `lib.rs:1709-1748`, `PoolEvent::Wake` `lib.rs:2502-2545` | wired |
| Circuit-broken respawn with backoff | `SlotCircuit` `lib.rs:1048-1134`, `spawn_respawn_task` `lib.rs:3635-3684` | wired |
| 30 s maintenance tick (queue compaction + slot refill + retry flush) | `lib.rs:1750-1794` | wired |
| Panic recovery (requeue + unwedge channel + respawn) | `recover_panicked_agent` `lib.rs:3402-3499` | wired |
| Heartbeat prompts | `dispatch_heartbeat` `lib.rs:3537-3586` | wired, off by default (interval 0) |
| Presence 20001 + 60 s heartbeat + offline on exit | `lib.rs:77-91`, `1586-1592`, `2300-2320`, `2673-2685` | wired |
| Typing indicators 20002 on a 3 s tick | `lib.rs:1593-1599`, `2321-2341` | wired |
| 👀 reaction on accept, cleaned on drain | `lib.rs:2204-2213`, `1934-1944` | wired, fire-and-forget |
| Owner text commands `!shutdown` / `!cancel` / `!rotate` | `lib.rs:2033-2133` | wired |
| Encrypted observer telemetry (kind 24200, NIP-44 to owner) | `lib.rs:790-833` | wired, requires `--relay-observer` **and** a resolved owner |
| Encrypted observer control (`cancel_turn`, `switch_model`) | `lib.rs:837-1005` | wired |
| Non-cancelling ACP steer with cancel+merge fallback | `try_native_steer` `lib.rs:2803-2887`, ack arm `lib.rs:2417-2500` | wired |
| Auth-error fast dead-letter | `is_auth_error` `lib.rs:3003-3011`, used `lib.rs:3118` | wired |
| User-visible failure notices in-channel | `spawn_failure_notice` `lib.rs:3014-3031` | wired for hard timeout, retries-exhausted, and auth errors |
| Dynamic channel join/leave via membership notifications | `lib.rs:1861-1949` | wired |
| Graceful shutdown (SIGINT + SIGTERM, 30 s grace, child reaping) | `lib.rs:1635-1650`, `2588-2669` | wired |
| MCP server handed to the agent | `build_mcp_servers` `lib.rs:4141-4184` | wired, but only when `BUZZ_ACP_MCP_COMMAND` is non-empty — **default is empty** (`config.rs:261`), so a stock run gives the agent zero MCP servers |
| `models` / `auth-methods` / `authenticate` probe subcommands | `lib.rs:4005`, `3899`, `3947` | wired, undocumented in README |
| Setup-listener mode | `setup_mode::run_setup_listener` `lib.rs:1298-1303` | wired |

#### Declared but not reachable / not exercised

| Item | Evidence |
|---|---|
| `buzz-persona` dependency | declared `Cargo.toml:22`; `grep -rn 'buzz_persona' crates/buzz-acp/` returns only the manifest line. The `persona_env_vars` plumbing (`lib.rs:1762`, `3488`, `3666`, `3733`) is a plain `Vec<(String,String)>` built in `config.rs:945-999` — it does not call into the crate. |
| `pub use usage::TurnUsage` | `lib.rs:15`; no reference to `TurnUsage` anywhere else in `lib.rs`. Exported for external consumers only. |
| Owner re-resolution after startup | `OwnerCache.pubkey` has no setter (`lib.rs:161-163`). With `respond-to=owner-only` and no owner at boot, the harness drops every event for its whole life; the warning at `lib.rs:1379-1384` is the only signal. |
| Relay observer without an owner | `lib.rs:1421-1425` logs a warning and leaves the feature silently off, even though `--relay-observer` was explicitly requested. |
| `RespawnResult` third tuple element (`supports_goose_steer`) | Documented at `lib.rs:1142-1145` as "always `true`" because the supervisor uses try-and-tolerate. The field it describes no longer exists in the tuple, which is `(AcpClient, u32, String)` — the doc comment is stale. |
| `SubscribeMode::All` default kinds | `lib.rs:1456` yields an **empty** kinds vector when `--kinds` is absent, which per `AGENTS.md § Common Gotchas #2` trips the relay p-gate (403). Nothing in `lib.rs` warns about this. |

#### Recovery / resilience behaviours

- Relay stream end triggers `relay.reconnect()`; only a dead background task exits the loop (`lib.rs:2295-2303`).
- Requeued batches are re-flushed on three triggers so quiet channels don't stall: the maintenance tick (`lib.rs:1788-1793`), a completed respawn collection (`lib.rs:1795-1801`), and every pool result (`lib.rs:2390`). The comments at `lib.rs:1783-1787` and `1795-1798` state the failure mode this fixes.
- Shutdown reaps in four stages so `AcpClient::Drop` (start_kill + try_wait, best-effort) is never the primary path: drain `join_set` + `result_rx` under a 30 s grace (`lib.rs:2608-2636`), drain late results (`lib.rs:2643-2648`), reap idle slots (`lib.rs:2649-2661`), then abort and drain respawn tasks (`lib.rs:2663-2672`).
- Wake tasks are *drained*, not aborted, on shutdown — the comment at `lib.rs:2573-2589` explains that aborting mid-init would drop partially spawned `AcpClient`s and create exactly the zombies the eager path avoids. A 30 s timeout is the backstop before falling back to `shutdown()`.


## Module: buzz-acp — relay client & observer (`crates/buzz-acp/src`)
### Aspect: Features

#### Supported

| Feature | Location | Notes |
|---|---|---|
| WebSocket connect + NIP-42 AUTH | `relay.rs:3825-3862` | 30 s connect timeout, 20 s per auth step |
| NIP-OA `auth_tag` in the AUTH event | `relay.rs:3444-3456` | hand-built tag vec; `EventBuilder::auth` used when absent |
| Bounded initial-connect retry | `relay.rs:3786-3822` | 6 attempts, terminal errors short-circuit |
| Autonomous reconnect on socket loss | `relay.rs:2893-3018` | 5 attempts + separate flat DNS retries |
| Caller-driven reconnect with persisted backoff | `relay.rs:3022-3151` | unbounded loop, 60 s tail |
| Channel REQ subscribe / CLOSE | `relay.rs:3160-3222`, `:1419-1432` | sub id `ch-<uuid>` |
| Membership-notification subscribe (44100/44101) | `relay.rs:3227-3270` | global, `#p`-scoped |
| Observer-control subscribe (24200) | `relay.rs:3273-3305` | `#p`-scoped, `since: now` |
| Publish signed events over WS | `relay.rs:2611-2624` | via `RelayCommand::PublishEvent` |
| Typing-indicator construction (kind 20002) | `relay.rs:843-867` | `h` tag + NIP-10 `root`/`reply` `e` tags |
| Non-blocking fire-and-forget publish | `relay.rs:834-840` | `try_send`, drops on full command channel |
| Cloneable publisher handle | `relay.rs:556-591` | lets spawned tasks publish without socket access |
| Client-initiated ping/pong liveness | `relay.rs:1937-1999` | `PING_INTERVAL` 30 s, `PONG_TIMEOUT` 10 s (`relay.rs:48-52`) |
| Server-initiated Ping → Pong | `relay.rs:2370-2376`, `:3903-3907`, `:3963-3967` | all three intake loops |
| Two-generation bounded dedup | `relay.rs:935-972` | 6k–12k ids |
| `since`-based replay with skew | `relay.rs:3185-3194` | −5 s on reconnect |
| Backpressure-driven proactive resubscribe | `relay.rs:2118-2181`, `:1584-1596` | runs on the existing socket |
| Rate-limit gate with jitter + max-extend | `relay.rs:1182-1207` | armed from NOTICE and CLOSED |
| Paced REQ drain (≈8 frames/s) | `relay.rs:1706-1779`, `:2667-2775` | 1 frame per select tick |
| Durable observer-frame parking + in-flight tracking | `relay.rs:1213-1263`, `:2629-2665` | bounded 256, drop-oldest with counter |
| Per-channel access-denial handling without reconnect | `relay.rs:3498-3529` | exact string match only |
| Graceful shutdown with close frame | `relay.rs:900-912`, `:1782-1791` | 5 s wait then abort |
| Channel discovery over HTTP bridge | `relay.rs:657-714` | two `POST /query` calls |
| Archived-channel exclusion | `relay.rs:171-232` | skips `archived=true` |
| NIP-98-authenticated HTTP `POST /query` | `relay.rs:399-406` | retried on 429/5xx |
| NIP-98-authenticated HTTP `POST /events` | `relay.rs:411-423` | used by metrics, reactions, notices |
| NIP-AE core-engram fetch + prompt rendering | `engram_fetch.rs:39-165` | fail-closed on undecryptable candidates |
| In-process observer bus with replay buffer | `observer.rs:83-136` | 1,000-event ring, monotonic `seq` |

#### Stubbed, absent, or partial

| Gap | Evidence |
|---|---|
| **No `COUNT` support** | no `"COUNT"` arm in `parse_relay_message` (`relay.rs:3535-3620`); nothing builds a COUNT frame anywhere in the module |
| **No NIP-50 `search`** | no `search` key in any filter; `RestClient::query` takes raw `nostr::Filter` and no call site sets it (`relay.rs:663`, `:698`, `engram_fetch.rs:79`) |
| **EOSE is inert** | logged only (`relay.rs:2190-2192`); no "initial replay complete" signal to the harness |
| **AUTH OK is not matched to the AUTH event id** | `wait_for_any_ok` accepts the first OK of any id; the comment concedes it (`relay.rs:3846-3854`) |
| **Mid-session AUTH from the handshake buffer is discarded** | `relay.rs:2432-2433` returns `None` for `RelayMessage::Auth` on replay |
| **No `limit` on any WS REQ** | `send_subscribe` / `send_membership_subscribe` / `send_observer_control_subscribe` build no `limit` key (`relay.rs:3170-3194`, `:3232-3250`, `:3274-3285`); only the HTTP engram query sets one (`engram_fetch.rs:88`) |
| **Wildcard REQ possible** | `kinds` omitted when `ChannelFilter.kinds` is `None` (`relay.rs:3172-3175`); reachable via `--subscribe-mode all` without `--kinds` (`config.rs:1180`, `:1272`) and via an empty-`kinds` config rule (`config.rs:1195-1196`) |
| **No `authors` filter on the observer-control REQ** | `relay.rs:3274-3285` — any pubkey's kind:24200 addressed to this agent is forwarded; owner enforcement lives in `lib.rs` |
| **No signature verification of forwarded events** | `relay.rs:2069-2076` and `:2154-2181` forward relay-supplied events verbatim; `engram_fetch.rs:126` is the only place this module calls `event.verify()` |
| **`publish_event` on `HarnessRelay` is unused** | `#[allow(dead_code)]` at `relay.rs:820`; callers use `RelayEventPublisher::publish_event` (`lib.rs:89`, `:829`, `setup_mode.rs:641`) or `try_publish_event` (`lib.rs:2338`) |
| **`_state` parameter unused in `send_subscribe`** | `relay.rs:3162` — takes `&BgState` and never reads it |
| **No `x-auth-tag` on the WS path** | the NIP-OA tag goes into the AUTH event only; there is no per-EVENT delegation tag |
| **Observer bus has no relay transport of its own** | `observer.rs:1-6` states this explicitly; publication is entirely in `lib.rs` |
| **`ObserverHandle::snapshot` silently returns empty on a poisoned mutex** | `observer.rs:96-100` |

#### Typing indicators: kind 20002 published from this crate

`build_typing_event` constructs a kind:20002 (`KIND_TYPING_INDICATOR`,
`buzz-core/src/kind.rs:331`) event with an `h` tag and, when threaded, NIP-10
`["e", root, "", "root"]` + `["e", parent, "", "reply"]` tags
(`relay.rs:849-867`). The publisher is the 3-second `typing_refresh` tick in the
main loop: `lib.rs:2333-2341` calls `relay.build_typing_event(...)` then
`relay.try_publish_event(event)` for every channel in `typing_channels`. The
interval is set at `lib.rs:1593-1596`, gated on `config.typing_enabled`.

This is a different mechanism from the one `ARCHITECTURE.md:452-458` documents.
That section describes a Redis sorted-set protocol
(`ZADD buzz:typing:{channel_id}` / `ZREMRANGEBYSCORE` / `EXPIRE`, 5 s activity
window, 60 s key TTL). No `KIND_TYPING_INDICATOR` reference exists anywhere in
`crates/buzz-relay/src` or `crates/buzz-pubsub/src`; kind 20002 falls in the
generic ephemeral range (`EPHEMERAL_KIND_MIN = 20000`,
`buzz-core/src/kind.rs:321`, `is_ephemeral` at `:621`) and is fanned out via
plain Redis pub/sub without being stored. The documented sorted-set design is
not what this module's typing events exercise.

#### Kind-registry compliance

Every event-kind integer used in production code in this module resolves through
`buzz_core::kind`: `KIND_AGENT_OBSERVER_FRAME`, `KIND_MEMBER_ADDED_NOTIFICATION`,
`KIND_MEMBER_REMOVED_NOTIFICATION`, `KIND_TYPING_INDICATOR` (imported at
`relay.rs:118-122`), `KIND_NIP29_GROUP_MEMBERS` (`relay.rs:665`),
`KIND_NIP29_GROUP_METADATA` (`relay.rs:700`), `KIND_AGENT_ENGRAM`
(`engram_fetch.rs:17`, used `:81`). NIP-42 and NIP-98 use the `nostr` crate's
own `Kind::Authentication` (`relay.rs:3452`) and `Kind::HttpAuth`
(`relay.rs:292`). No bare kind literal appears in the production half of
`relay.rs` (lines 1–3,994) — the only numeric kinds in the file are `Kind::Custom(9)`
inside tests. Kind numbers do appear in doc comments (`relay.rs:162`, `:262`,
`:653-654`, `:842`, `:1036`, `:1469`, `:3225`), which is drift risk but not a
registry bypass.


## Module: buzz-acp — agent pool & lifecycle (`crates/buzz-acp/src`)
### Aspect: Features

#### Take-and-return pooling

The pool holds `1..=32` agent subprocesses (`config.rs:292-295`) in a fixed-length slot vector and moves ownership out on claim, back on return (`pool.rs:206-219`). `AcpClient` is not `Clone`, so the compiler enforces single-ownership of each child's stdio (`pool.rs:22`). Reachable and exercised on every dispatch (`lib.rs:2907`).

#### Channel affinity (session reuse)

`try_claim(Some(channel_id))` prefers an idle agent that already holds an ACP session for that channel, avoiding a `session/new` round-trip and re-fetch of core/canvas (`pool.rs:558-568`). `has_session_for` (`pool.rs:599-607`) computes the `affinity_hit` flag logged alongside `agent_claimed` (`lib.rs:2918`). Reachable. With the default `--agents 1` affinity is trivially satisfied by the single slot; the feature only has an effect at `agents > 1`.

#### Warm reuse of sessions across turns

A session survives between turns: `run_prompt_task` reuses `state.sessions[cid]` when present (`pool.rs:1468-1470`) and only creates a new one when absent. Cached alongside it are the rendered NIP-AE core block and the `[Channel Canvas]` block, both fetched "once per new channel session" and cleared on invalidation (`pool.rs:1379-1416`, `:1428-1449`). This is the module's only warm-start mechanism.

#### Deferred (lazy) pool start

`--lazy-pool` / `BUZZ_ACP_LAZY_POOL` (`config.rs:471-473`) constructs an all-`None` pool (`lib.rs:1318`) and defers subprocess spawning until the first accepted event is buffered; `PoolLifecycle` then runs exactly one initialization task with bounded exponential backoff on failure (`pool_lifecycle.rs:37-58`, `lib.rs:1714-1743`). Reachable, off by default. Runtime state is surfaced to the desktop as `managed_agent_runtime_lifecycle` frames with `waking`/`ready`/`failed` (`lib.rs:1721-1728`, `:2550-2557`, `:2566-2573`).

There is no partial or per-slot warm start: the wake initializes the whole pool in one task (`initialize_agent_pool`, `lib.rs:3741`), sequentially per slot, each spawn+init bounded at 60 s (`lib.rs:3766`). With `--agents 32` and slow-starting agents, first-response latency is up to 32 × 60 s.

#### Partial-pool tolerance

`initialize_agent_pool` pushes `None` for any slot that fails to spawn or initialize and continues (`lib.rs:3812-3823`), erroring only when zero agents came up (`lib.rs:3825-3830`) and warning on a reduced pool (`lib.rs:3832-3838`). `from_slots` preserves positions so index-to-slot alignment survives the gaps (`pool.rs:536-556`). Empty slots are later refilled by the maintenance pass when the slot's circuit allows (`lib.rs:1748-1768`).

#### Per-agent isolation

Isolation is per **process**, not per channel or per user:

- Each `OwnedAgent` owns its own child, stdio pipes, and `SessionState` (`pool.rs:150-171`).
- All agents in the pool run the same command, args, and env — one `PoolStartup` snapshot from `Config` (`lib.rs:3717-3737`) — so there is no per-agent policy, model, cwd, or credential differentiation.
- All N agents authenticate as the **same** Nostr identity (`README.md:197`); `ctx.agent_keys` is a single `nostr::Keys` clone shared by every task (`lib.rs:1557`).
- `ctx.cwd` is one process-wide value from `std::env::current_dir()` (`lib.rs:1547-1550`), so every agent and every channel shares one working directory.
- Cross-channel state does leak within one agent: `state.sessions`, `turn_counts`, `core_sections`, and `canvas_sections` are per-agent maps keyed by channel (`pool.rs:86-104`), and `agent_name`/`goose_system_prompt_supported`/`model_capabilities` are process-wide latches.

#### Mid-turn message delivery (steer)

Two paths. The native path sends a Goose `session/steer`-style request into the running read loop over a capacity-1 channel installed per turn (`pool.rs:646-662`, install at `lib.rs:2937-2938`), with the read loop filling `expectedRunId` at write time because it is a moving target (`pool.rs:296-318`). The universal fallback is `ControlSignal::Steer` — cancel the turn, requeue the batch stamped `CancelReason::Steer`, and re-prompt with merge framing (`pool.rs:2986-3004`). Both reachable; the steer channel is installed for **every** prompt task regardless of agent, using try-and-tolerate on `-32601` (`lib.rs:2930-2939`, `pool.rs:339-346`).

#### Runtime model switching

Two entry points. Busy path: `ControlSignal::SwitchModel(id)` sets `desired_model` + `model_overridden` before cancelling, so the requeued turn re-creates the session under the new model (`pool.rs:1855-1858`). Idle path: `switch_idle_agent_model` validates against the agent's cached catalog *before* invalidating, returning `UnsupportedModel` without disturbing the session (`pool.rs:732-762`). Both reachable from the observer control channel (`lib.rs:981`). Runtime-only — never persisted; reset on respawn because every `OwnedAgent` construction passes `desired_model: config.model.clone()` and `model_overridden: false` (`lib.rs:1793-1795`, `:3799-3801`).

Acknowledged gap in the code's own doc: an idle-path switch does not re-emit `session_config_captured`, so the desktop panel does not reflect it until the agent next runs a turn (`pool.rs:723-731`).

#### Proactive session rotation

Rotate on `MaxTokens`/`MaxTurnRequests`, or after `max_turns_per_session` successful turns when that knob is non-zero (`pool.rs:1994-2022`). Default is `0` = rotate only on stop-reason (`config.rs:372-375`), so the turn-count path is off unless configured.

#### Per-turn observability

`turn_started` (`pool.rs:1295`), `session_resolved` (`pool.rs:1573`), `session_config_captured` (`pool.rs:908`), `control_result` on an unsupported model (`pool.rs:888`), periodic `turn_liveness` (`pool.rs:3194`), and `turn_completed` from a drop guard covering every exit path including panic (`pool.rs:3291-3301`). All gated on `--relay-observer` producing an `ObserverHandle` (`lib.rs:1299-1301`); with the observer absent, `run_turn_liveness` parks forever and emits nothing (`pool.rs:3169-3171`).

Per-turn cost/token metrics are published as encrypted NIP-AM kind `44200` events, but only when the agent emitted usage **and** an owner pubkey is configured — otherwise `publish_agent_turn_metric` returns immediately (`pool.rs:3336-3339`).

#### Two-phase user-visible reaction lifecycle

👀 "seen" is added by `lib.rs` at queue-push time; 💬 "working" is added by the pool just before the prompt fires (`pool.rs:1802-1809`); both are removed by `ReactionGuard` on any exit path (`pool.rs:3125-3141`). Fan-out is chunked at `REACTION_CONCURRENCY = 10` (`pool.rs:3618-3648`). The code documents the accepted cosmetic race where a fast failure clears before the 👀 add lands (`pool.rs:3104-3110`).

#### Not present

- No idle-timeout eviction or scale-to-zero of individual agents; the only way a live agent's process ends is crash, poisoning, or harness shutdown (`pool.rs:534-763` contains no reaper).
- No dynamic resize: `agents` is allocated once and there is no add/remove-slot API.
- No persona-driven spawning. `buzz-persona` is a declared dependency (`Cargo.toml:22`) with zero references anywhere under `crates/buzz-acp/src`; the pool receives only `config.persona_env_vars` as opaque `(String, String)` pairs by way of `PoolStartup` (`lib.rs:3721`, `:3733`). Persona-defined `mcp_servers` never reach this module — `PromptContext::mcp_servers` is built solely from `config.mcp_command` (`lib.rs:4141-4184`).
- No per-agent MCP server sets: `ctx.mcp_servers` is one shared `Vec` cloned into every `session/new` (`pool.rs:832`).


## Module: buzz-acp — work queue, filtering & usage accounting (`crates/buzz-acp/src`)
### Aspect: Features

#### Reachability summary

| Feature | Entry point | Reachable by default? |
|---|---|---|
| Per-channel FIFO queueing with cross-channel fairness | `EventQueue::push` / `flush_next` (`queue.rs:230`, `:260`) | yes — always on |
| Batching up to 50 events per prompt | `queue.rs:336-345` | yes; only observable when >1 event queues for one channel |
| Drop-mode dedup | `queue.rs:231-241` | **no** — CLI default is `--dedup queue` (`config.rs:344`) |
| Queue-mode accumulation | `queue.rs:231` (guard is Drop-only) | yes (default) |
| Exponential-backoff retry with jitter | `queue.rs:454-464` | yes, on any transient prompt failure |
| Dead-lettering after 10 attempts | `queue.rs:437-451` | yes |
| Immediate dead-letter on hard timeout with no recent activity | `lib.rs:3091-3109` | yes |
| Immediate dead-letter on auth error | `lib.rs:3130-3144` | yes |
| Cancel + merged re-prompt (steer / interrupt framing) | `requeue_as_cancelled` (`queue.rs:542`) + `MergeFraming` (`queue.rs:1584`) | yes — `multiple_event_handling` defaults to `steer` (`config.rs:352-359`) |
| Goose-native non-cancelling steer (withhold side table) | `mark_native_steer_pending` / `release_native_steer` / `remove_event` (`queue.rs:673`, `:703`, `:738`) | attempted for every agent; falls back to cancel+merge on `-32601` (`lib.rs:2476-2483`) |
| In-flight deadline backstop with monotonic extension | `queue.rs:263-287`, `:210-228` | yes |
| Periodic map compaction | `compact_expired_state` (`queue.rs:807`) | yes — 30 s tick (`lib.rs:1608`, `:1746`) |
| Channel teardown on agent removal | `drain_channel` (`queue.rs:625`) | yes (`lib.rs:1990`) |
| Structured prompt framing (9 section types) | `format_prompt` (`queue.rs:1406`) | yes |
| Legacy-agent prompt injection (`[Base]`/`[System]`/`[Team Instructions]`/core/canvas in the user message) | `queue.rs:1432-1462` | only when `has_system_prompt_support == false` (protocol < 2) |
| Human-aware reply anchoring | `resolve_reply_anchor` (`queue.rs:1209`) | yes, but degrades to "everyone is human" without a `profile_lookup` |
| Slash-command pass-through | `slash_command_for_batch` (`queue.rs:961`) | yes (`pool.rs:1761`) |
| Profile-aware display labels | `resolve_prompt_label` (`queue.rs:1042`) | only when `profile_lookup` is supplied — **not** on the native-steer path (`lib.rs:2831` passes `None`) |
| Rule-based subscription matching | `filter::match_event` (`filter.rs:368`) | yes |
| evalexpr content filters | `evaluate_filter` (`filter.rs:197`) | only in `--subscribe config` mode with a `filter =` key; `Mentions`/`All` rules set `filter: None` (`lib.rs:1446`, `:1464`) |
| Filter timeout circuit breaker | `filter.rs:405-415` | yes, but only exercisable via config-mode filters |
| Per-turn token/cost accounting | `UsageTracker` (`usage.rs:163`) | only when the harness emits `_goose/unstable/session/update` with `sessionUpdate: "usage_update"` |
| NIP-AM kind 44200 metric publishing | `pool.rs:3322-3420` | only when usage is present **and** `agent_owner_pubkey` is configured (`pool.rs:3333-3336`) |

#### Batching

`flush_next` drains the whole channel queue up to `MAX_BATCH_EVENTS = 50` into one `FlushBatch` (`queue.rs:27`, `:336-345`), then re-sorts ascending by `created_at` because relay replay arrives newest-first (`queue.rs:346-350`). `format_prompt` renders 1 event as `[Buzz event: {tag}]` and N events as `[Buzz events — N events]` with `--- Event i (tag) ---` separators (`queue.rs:1528-1552`). Tests: `test_batch_drain_all_events` (`queue.rs:1759`), `test_format_prompt_batch` (`:2173`), `test_flush_orders_replayed_events_chronologically` (`:1782`).

Cross-channel interleaving is by oldest head event, so a busy channel cannot starve a quiet one — `test_multi_channel_interleave` (`queue.rs:1826`), `test_fifo_fairness_picks_oldest_channel` (`:1810`).

#### Dedup

Two modes only (`config.rs:57-61`). `Drop` discards in-flight-channel events at admission; `Queue` accumulates. Beyond that there is **no content-level dedup** — the same event delivered twice by the relay queues twice. The only id-based suppression in this area is `remove_event` on a successful native steer (`queue.rs:738-751`) and `should_nudge_for_event`'s id set in setup mode (`setup_mode.rs:389`, `:495`). Tests: `test_drop_mode_discards_in_flight_events` (`queue.rs:2504`), `test_drop_mode_queues_other_channels` (`:2522`), `test_drop_mode_drops_for_any_in_flight_channel` (`:2593`).

#### Retry and dead-lettering

Per-channel attempt counter, 5 s base doubling to a 300 s cap, ±20 % jitter, 10 attempts (≈25 min of wall clock) before the batch is handed back to the caller for a user-visible failure notice (`queue.rs:429-498`; notice construction `lib.rs:3122-3127`, `:3150-3153`). `requeue_preserve_timestamps` is a deliberately budget-free variant used when no agent is free (`queue.rs:508-529`) — it can loop forever without dead-lettering.

Three distinct dead-letter triggers exist, only one of which consults the retry budget:

1. budget exhausted inside `requeue` (`queue.rs:437`),
2. hard timeout with `recently_active: false` (`lib.rs:3091-3109`) — bypasses the queue entirely,
3. auth error (`lib.rs:3130-3144`) — bypasses the queue entirely.

`hard_timeout_fate_suffix` (`lib.rs:3059`, set at `:3108`, `:3125`, `:3128`, `:3162`) exists solely to make the death message describe the batch's actual fate.

#### Prompt framing

`format_prompt` returns a `Vec<String>` of independent sections rather than one string so the observer frame trimmer can elide one section's body while keeping every `[Header]` line countable in the desktop "Prompt context" panel (`queue.rs:1394-1400`; trimmer `lib.rs:659`). Section inventory: `[Base]`, `[System]`, `[Team Instructions]`, agent-core memory block (rendered upstream by `engram_fetch::build_core_section` (`engram_fetch.rs:39`)), `[Channel Canvas]`, `[Context]`, `[Thread Context]`/`[Conversation Context]`, the cancelled/new event blocks, and a closing note.

`[Context]` has three shapes — `Scope: dm`, `Scope: thread`, `Scope: channel` — each with a different `buzz` CLI hint and, for human-facing turns, a `--reply-to` instruction (`queue.rs:1233-1315`). DM is checked first so a DM reply reports `dm`, not `thread` (`queue.rs:1246-1248`).

Merge framing is a two-variant table selected by `CancelReason` (`queue.rs:1584-1609`). Native steer reuses `new_header_single` + `closing_note` from the same table via `native_steer_framing()` (`queue.rs:1623-1626`) and the same `format_event_block`, so the two transports are structurally prevented from drifting (`lib.rs:2812-2823` states this as the requirement).

#### Slash-command extraction

Mention-stripping loop handles NIP-27 `nostr:npub1…`/`nostr:nprofile1…` tokens, `@`-prefixed multi-word known display names (longest-first, case-insensitive), and `@`-prefixed single tokens of `[A-Za-z0-9._-]` (`queue.rs:906-947`). Pass-through is gated to single-event, no-carryover batches (`queue.rs:961-967`). Known names come from the caller at `pool.rs:1761`.

#### Usage / cost accounting

Cumulative-to-delta normalization with three explicit unreliability cases (first turn, token decrease, cost decrease), multi-notification-per-turn tolerance, session-restart handling, and cross-session isolation (`usage.rs:211-302`). `turn_seq` counts **published** metrics, not notifications (`usage.rs:99-102`, `:231`). Downstream, `publish_agent_turn_metric` builds an `AgentTurnMetricPayload`, encrypts it to the owner pubkey, and publishes kind `KIND_AGENT_TURN_METRIC` (44200) with `p` and `agent` tags (`pool.rs:3363-3400`). Cache-token and total-token fields exist in the payload type but are always `None` from this path (`pool.rs:3341-3342`, `:3358-3360`) — `used` / `context_limit` from the wire payload are parsed and never read (`usage.rs:78-85`), so context-window utilization is not reported anywhere.


## Module: buzz-acp — ACP protocol, config & setup mode (`crates/buzz-acp/src`)
### Aspect: Features

#### ACP client capabilities — implemented

| Feature | Evidence |
|---|---|
| Spawn an agent binary and speak NDJSON JSON-RPC 2.0 over its stdio | `acp.rs:408-497` |
| Protocol handshake with capability advertisement | `initialize` (`acp.rs:539-544`), `build_client_capabilities` (`acp.rs:347-368`) |
| Adapter-owned login flows | `authenticate` (`acp.rs:549-554`), driven by `run_authenticate` (`lib.rs:3947`) and `run_auth_methods` (`lib.rs:3899`) |
| Session creation with MCP server declarations and an optional system prompt | `session_new_full` (`acp.rs:563-588`) |
| Multi-block prompts (slash-command pass-through) | `session_prompt_blocks_with_idle_timeout` (`acp.rs:676-745`); rationale at `acp.rs:672-675` — connectors detect commands on the **first** block, so the harness sends `["/cmd args", "<buzz context>"]` |
| Dual-deadline turn supervision (idle + absolute) | `acp.rs:1198-1523` |
| Cooperative cancellation with permission cleanup and response draining | `cancel_with_cleanup` (`acp.rs:837`), `cancel_with_cleanup_until` (`acp.rs:897`) |
| Bounded Stop-button cancellation with a distinct error | `cancel_with_cleanup_grace` (`acp.rs:881-895`) |
| Automatic tool-permission approval | `handle_permission_request` (`acp.rs:1680-1755`) |
| Stable + unstable model switching with precedence | `resolve_model_switch_method` (`acp.rs:1876-1920`), `session_set_config_option` (`acp.rs:623`), `session_set_model` (`acp.rs:638`) |
| Goose-native non-cancelling mid-turn steer | read-loop steer arm (`acp.rs:1279-1358`), `build_steer_params` (`acp.rs:1791-1805`) |
| Goose token-usage ingestion | `handle_goose_usage_update` (`acp.rs:1637-1678`), `take_turn_usage` (`acp.rs:783`) |
| Raw-wire observer feed for the desktop app | `observe` (`acp.rs:524-533`), emitted at `acp.rs:963` (write), `acp.rs:1120`/`acp.rs:1414` (read), `acp.rs:1105`/`acp.rs:1399` (parse errors) |
| Process-group teardown | `kill_process_group` (`acp.rs:1979-1987`) used by `shutdown` (`acp.rs:384-388`) and `Drop` (`acp.rs:1956-1962`) |
| Windows console-window suppression | `configure_no_window` (`acp.rs:1997-2006`) |
| Codex Seatbelt network widening | `codex_network_env` (`config.rs:646-677`) + `build_codex_config_env` (`acp.rs:257-345`) |

#### Stubs, dead scaffolding, and not-implemented paths

| Item | State | Evidence |
|---|---|---|
| `drain_stale_responses` | Fully implemented, never called — "Scaffolding for future model-switch timeout cleanup; not yet wired" | `acp.rs:1022-1023` |
| `session_new` (id-only wrapper) | `#[allow(dead_code)]`, "Public API — callers outside the harness may use this" | `acp.rs:591-599` |
| `active_run_id()` accessor | `#[cfg_attr(not(test), allow(dead_code))]`; production reads the field directly inside the read loop | `acp.rs:768-771` |
| `available_commands_update` | Logged only; the doc comment says "UI surfacing is a follow-up" | `acp.rs:1573-1588` |
| `plan` updates | Logged as "plan update received"; payload discarded | `acp.rs:1563-1566` |
| `keepalive` updates | Consumed with no side effect beyond the generic idle reset | `acp.rs:1621` |
| `authenticate` | No retry, no method discovery inside `acp.rs` — the caller enumerates methods from `initialize`'s result | `acp.rs:549-554` |
| Non-Unix process-group kill | Returns `false`, caller falls back to `child.start_kill()` | `acp.rs:1990-1992` |
| `allowed_respond_to` enforcement | Validated at startup, then display-only | `config.rs:919-937`, `config.rs:1019-1025` |
| `BUZZ_API_TOKEN` | Propagated from the legacy alias, then never read | `config.rs:718`; no other read site in the crate |
| `persona_env_vars` as a persona mechanism | Declared as a general injection vector but only ever populated with the one generated `CODEX_CONFIG` entry | `config.rs:945-955` |
| `handle_setup_membership`'s `_initial_channel_ids` | Accepted, unused | `setup_mode.rs:568` |

`buzz-persona` is declared as a dependency (`crates/buzz-acp/Cargo.toml:22`, notably by `path` rather than `workspace = true` like every other internal dep) with **zero** code references anywhere under `crates/buzz-acp/src`. What `config.rs` actually does instead: `from_args` initialises `persona_env_vars` as an empty `Vec` (`config.rs:945`) and the only thing ever pushed into it is the `codex_network_env` pair (`config.rs:951-957`). The comment at `config.rs:943-944` explains the design shift — spawned desktop agents now carry a complete instance snapshot, and team instructions arrive independently so they can be layered at runtime. Persona resolution has moved out of this crate; the dependency and the field's doc comment ("Populated from persona pack resolution", `config.rs:534`) are both stale.

#### Setup mode

Entered only when `BUZZ_ACP_SETUP_PAYLOAD` is present and parses (`setup_mode.rs:83`, read at `setup_mode.rs:214`, branch at `lib.rs:1290-1295`). The module header states the contract as non-negotiable (`setup_mode.rs:16-25`): desktop is the only readiness source, buzz-acp does not re-derive readiness, and normal startup gains no second readiness path.

What it does:

- Connects to the relay, sets a startup watermark, subscribes to membership notifications and to resolved channel filters (`setup_mode.rs:329-380`).
- Runs an event loop that, for @mentions passing the author + filter + dedup gates, publishes a single "needs configuration" reply naming exactly what is missing (`setup_mode.rs:388-478`).
- Reacts to membership add/remove by subscribing/unsubscribing channels, with no queue or session teardown because there is no pool (`setup_mode.rs:563-592`).
- Reconnects when the relay stream ends, relying on the `nudged_event_ids` set to avoid double-nudging on replay (`setup_mode.rs:390-397`, `setup_mode.rs:385-386`).

What it does not do: spawn any agent subprocess, create ACP sessions, or run any prompt. The agent pool is never reached because `run_setup_listener` is a `return` from the startup path (`lib.rs:1294`).

Nudge output is dual-format: human-readable markdown plus an appended fenced `buzz:config-nudge` block carrying the payload as JSON, so the desktop renders a `ConfigNudgeCard` and other clients see a code block (`setup_mode.rs:236-242`, `setup_mode.rs:296-302`).

Five requirement surfaces produce distinct copy (`setup_mode.rs:122-193`): a missing dropdown field, a missing env-backed credential, a CLI login step (with four sub-cases keyed on `AcpAvailabilityStatus`), an unparseable external CLI config (with a stderr diagnostic and a synthesised `~/.<cli>/config.toml` path at `setup_mode.rs:184`), and missing Git for Windows.

#### The compiled-in base prompt

`base_prompt.md` (136 lines) is embedded with `include_str!("base_prompt.md")` at `lib.rs:1544`, and again at `lib.rs:3592` and `lib.rs:3602`.

Selection ladder (`lib.rs:1539-1545`): `--no-base-prompt` → `None`; else `--base-prompt-file` content if `Config::base_prompt_content` is `Some`; else the compiled-in default. `base_prompt_content` is resolved and size-checked (1 MB cap) during `Config::from_args` (`config.rs:778-791`), so a missing or oversized file fails at startup rather than at prompt time. Clap marks `--base-prompt-file` and `--no-base-prompt` mutually exclusive (`config.rs:414-418`).

Injection differs by negotiated protocol version: v2 receives the base prompt through `session/new`'s `systemPrompt`, while v1 gets it prefixed onto the prompt text — the heartbeat path documents both branches at `lib.rs:4197-4210`, and `session_new_system_prompt` gates on `agent.protocol_version` at `pool.rs:829-833`. Layering order for the combined prompt is base → `[System]` persona → team instructions → agent core memory → canvas (`pool.rs:823-830`).

Content of the prompt, by section:

| Section | What it establishes |
|---|---|
| Preamble | The agent is running inside Buzz, a Nostr-based messaging platform; the buzz-acp harness routes channel events to its session |
| Buzz CLI | `buzz` is the primary interface; names the three auth env vars (`BUZZ_RELAY_URL`, `BUZZ_PRIVATE_KEY`, `BUZZ_AUTH_TAG`), the exit-code contract (0/1/2/3/4), a 13-row command-group table, stdin usage for multiline content, and the `--channel` requirement when opening PRs |
| Conversational Agent Creation | Restricts agent-creation questions to name + purpose; gives the `buzz agents draft-create` invocation; forbids asking about runtime/provider/model/credentials; states drafts require owner review |
| Communication Patterns | Mention rules (exact full display name, no formatting, no narrative mentions), mandatory callback mentions on completed delegated work, threading rules keyed off the `[Context]` block, mandatory publishing of results, prohibition on bare acknowledgements, todo discipline, GFM formatting, polling instead of push |
| Startup Recovery | A four-step catch-up procedure using `buzz feed get`, `buzz messages get`, `AGENTS.md`, and the knowledge directories |
| Workspace Layout | Names eight working directories (`RESEARCH/`, `PLANS/`, `GUIDES/`, `WORK_LOGS/`, `OUTBOX/`, `REPOS/`, `.scratch/`) and forbids recursive searches over `$HOME` or `/` |
| Agent Memory | `core` memory is auto-injected each turn; a 65,535-byte hard limit with a ~10 KB target; eviction and cold-slug rules |
| Engineering Discipline | Work-in-the-open, candour, read-before-change, commit/verification discipline, second opinions on risky changes |
| Working in the Repo | Worktree-not-default-branch; read repo-local git identity before committing |
| Autonomy | Resolve questions independently; escalate only for product intent |

The file is plain text with no templating, placeholders, or per-agent substitution — every managed agent receives byte-identical content unless the operator overrides the whole file.


## Module: buzz-agent — core loop, ACP wire types & handoff (`crates/buzz-agent/src`)
### Aspect: Features
#### Operator-visible capabilities
| Capability | Where | Limits |
|---|---|---|
| ACP JSON-RPC 2.0 agent over stdio | `read_loop` (`lib.rs:187-206`), `writer_task` (`wire.rs:220-237`) | Newline-delimited frames only; one frame ≤ `max_line_bytes` (default 4 MiB, `config.rs:813`); oversize frame terminates the read loop (`wire.rs:194-198`) |
| Agentic tool loop (LLM → tool calls → results → repeat) | `RunCtx::run` (`agent.rs:66-257`) | Non-streaming: one request per round (`agent.rs:121-146`); rounds capped by `max_rounds` (0 = unlimited, `agent.rs:88`) |
| Parallel tool execution | `execute_parallel` (`agent.rs:370-490`) | ≤ `max_parallel_tools` concurrent (default 8, `config.rs:821`); ≤ 64 calls per round (`agent.rs:242-252`) |
| Interruptible turns | biased `select!` at every await (`agent.rs:121-125`, `agent.rs:424-441`, `handoff.rs:45-48`) | Cancel is per session, not per tool; drain bounded to 5 s then tasks are aborted (`agent.rs:455-478`) |
| Live model switching | `session/set_model` (`lib.rs:503-542`) | Takes effect on the *next* prompt (`lib.rs:671-673`); no validation against the advertised catalog; no notification to the client that a switch happened |
| Model catalog advertisement | `session/new` response (`lib.rs:443-474`) | Real discovery only for Databricks/DatabricksV2; other providers get a one-entry list echoing `cfg.model` (`lib.rs:459`). When discovery fails the response falls back to `discovery_failure_fallback` (`lib.rs:324`), which for `DatabricksV2` now advertises the configured model **plus** `DATABRICKS_V2_KNOWN_MODELS` (`catalog.rs:61-76`) so the picker can always represent the model actually running — asserted at `lib.rs:919-942` |
| Mid-turn steering without cancel | `steer_session` (`lib.rs:554-621`) + `drain_steers` (`agent.rs:265-280`) | Applied at round boundaries only; requires a matching `expectedRunId`; a steer queued after the last round is silently never delivered |
| Long-provider-call keepalive | `agent.rs:126-145` | Fixed 30 s interval, not configurable; consumed by the harness idle clock (`crates/buzz-acp/src/acp.rs:1623`) |
| Automatic context handoff (self-summarizing compaction) | `maybe_handoff` (`handoff.rs:31-107`) | ≤ `max_handoffs` per session (default 10); summary capped at 8192 output tokens (`config.rs:652`); after the cap, plain truncation takes over (`handoff.rs:34-41`) |
| Graceful history truncation | `truncate_history` (`agent.rs:711-738`) | Drops oldest whole turns; can return over budget when no later user turn exists (`agent.rs:723-725`) |
| MCP lifecycle hooks (`_Stop`, `_PostCompact`) | `agent.rs:224-236`, `handoff.rs:70-77` | Off unless `MCP_HOOK_SERVERS` is set (`config.rs:824`); advisory, fail-open, per-prompt objection budget (default 3) |
| Built-in `load_skill` tool | injected `agent.rs:115-119`, executed inline `agent.rs:317-327` | Only when the session discovered skills; no MCP round trip |
| Reasoning/thinking passthrough | `agent.rs:179-192` | Emitted as `agent_thought_chunk` before the message chunk; empty for providers that don't expose it (`types.rs:141-155`) |
| Per-turn + cumulative token reporting | `lib.rs:714-752` | Emitted only when the provider reported at least one count; `contextLimit` is hardcoded `0` (`lib.rs:742`) |
| Interactive provider auth | `buzz-agent auth databricks` (`lib.rs:129-152`) | Databricks OAuth PKCE only; needs a browser; writes under `~/.config/buzz-agent/oauth/databricks/` (`lib.rs:143`, path built at `auth.rs:454-461`) |

#### Deliberate non-features (verified in code, not just claimed)
- **No session load/resume**: `loadSession:false` advertised (`lib.rs:292`); no `session/load` arm exists (`lib.rs:224-265`).
- **No image/audio/embedded-context prompts**: advertised false (`lib.rs:293`) and enforced — `ContentBlock` accepts only `text` and `resource_link` (`types.rs:210-221`), anything else errors (`agent.rs:623-628`).
- **No HTTP/SSE MCP transport**: advertised false (`lib.rs:294`); `session/new` only accepts stdio server specs (`types.rs:195-202`).
- **No persistence**: zero filesystem writes in this group (`grep -c 'fs::write'` and `grep -c 'File::create'` → 0 in all six files).
- **No streaming**: a single `llm.complete` per round (`agent.rs:124`); text arrives as one `agent_message_chunk` (`agent.rs:194-206`).
- **No session teardown**: `grep -n 'sessions.remove' lib.rs` → 0 matches. Sessions and their MCP children persist until the process exits; shutdown only broadcasts cancel (`lib.rs:178-180`).
- **No CLI surface beyond `auth`**: unknown arguments are ignored and the agent starts reading stdio (`lib.rs:110-121`). No `--help`, no `--version`.

#### Feature limits a user will actually hit
- A prompt over 1 MiB is rejected outright (`agent.rs:69-73`) — but a *steer* has no equivalent per-message cap, only the frame cap.
- Concurrent prompts on one session are refused rather than queued (`lib.rs:786-788`); the client must serialize.
- `session/cancel` for an unknown session succeeds silently (`lib.rs:487-494`), so a client cannot distinguish "cancelled" from "no such session".
- After a handoff, the summary replaces all detail: only `[Context Handoff]\n{summary}` and the current prompt survive (`handoff.rs:84-95`).
- Tool results are middle-elided at `max_tool_result_text_bytes` (default 50 KiB) before entering history — enforced in the MCP layer via the `ResultBudget` this group constructs (`agent.rs:385-388`, `config.rs:649`, `config.rs:815-818`).


## Module: buzz-agent — LLM providers & configuration (`crates/buzz-agent/src`)
### Aspect: Features
This group is the provider abstraction layer: four providers, three wire dialects, five HTTP endpoint shapes, a per-model reasoning-effort capability system, and — since `16d4ec33` — adaptive collective (Mixture-of-Agents) routing for a mesh relay.

#### Providers supported
| `BUZZ_AGENT_PROVIDER` | `Provider` variant | Wire dialect(s) | Auth | Site |
|---|---|---|---|---|
| `anthropic` | `Anthropic` (`config.rs:663`) | Anthropic Messages | `x-api-key` header | `llm.rs:133-146`, `llm.rs:330-338` |
| `openai`, `openai-compat` | `OpenAi` (`config.rs:664`) | Chat Completions or Responses | `Authorization: Bearer` | `llm.rs:147-176`, `llm.rs:344-574` |
| `databricks` | `Databricks` (`config.rs:668`) | OpenAI-Chat-compatible, model in URL path | Bearer, static token or OAuth PKCE | `llm.rs:608-616`, `llm.rs:1534-1553` |
| `databricks_v2`, `databricks-v2` | `DatabricksV2` (`config.rs:677`) | routed per model family across three gateway dialects | Bearer, static token or OAuth PKCE | `llm.rs:177-214`, `llm.rs:576-592` |

Provider aliases are accepted at `config.rs:1003` (`openai-compat`) and `config.rs:1008` (`databricks-v2`); neither alias appears in `crates/buzz-agent/README.md:132`, which lists only `anthropic`, `openai`, `databricks`, `databricks_v2`.

#### Endpoints reached
| Provider | Method + URL | Site |
|---|---|---|
| Anthropic | `POST {base_url}/v1/messages` | `llm.rs:331` |
| OpenAI (Responses) | `POST {base_url}/responses` | `llm.rs:551`, `llm.rs:567` |
| OpenAI (Chat) | `POST {base_url}/chat/completions` | `llm.rs:558` |
| OpenAI (mesh catalog, **GET**) | `GET {base_url}/models` | `llm.rs:473`, issued `llm.rs:485-491` |
| Databricks (legacy) | `POST {base_url}/serving-endpoints/{effective_model}/invocations` | `llm.rs:609-613` |
| DatabricksV2 (GPT-5 family) | `POST {base_url}/ai-gateway/openai/v1/responses` | `llm.rs:985` |
| DatabricksV2 (Claude family) | `POST {base_url}/ai-gateway/anthropic/v1/messages` | `llm.rs:986` |
| DatabricksV2 (everything else) | `POST {base_url}/ai-gateway/mlflow/v1/chat/completions` | `llm.rs:987` |

`GET /models` (`llm.rs:473`) is the first non-POST request this group has ever made, and the first request that is not an inference call. Trailing slashes on `base_url` are stripped at every construction site (`llm.rs:331`, `llm.rs:473`, `llm.rs:611`, `llm.rs:618`, `llm.rs:1540`).

#### Adaptive collective routing ("mesh Auto")
The capability: when the agent is pointed at Buzz's relay-mesh router and configured with the model `auto`, it will dynamically send inference to the router's virtual Mixture-of-Agents model (`mesh`) whenever the live catalog shows enough peers to make MoA meaningful, and fall back to the router's ordinary single-model `auto` otherwise — **without a restart**. The whole feature is opt-in behind `BUZZ_AGENT_PREFER_MESH_FOR_AUTO` (`config.rs:807`), read as `Config.prefer_mesh_for_auto` (`config.rs:734`) and gated at `llm.rs:411-415`.

| Behaviour | Rule | Site |
|---|---|---|
| Discovery | `GET {base_url}/models`, 2 s timeout, bearer-authenticated | `llm.rs:472-530` |
| Viability test | catalog must advertise a virtual `mesh` entry **and** ≥2 distinct physical models (`@main` spellings deduplicated, `auto` excluded) | `llm.rs:47-64` |
| Adopt | two consecutive qualifying observations | `llm.rs:441-446`, `MESH_AUTO_ENABLE_OBSERVATIONS` `llm.rs:33` |
| Abandon | one non-qualifying observation | `llm.rs:447-451` |
| Re-probe interval | 5 s TTL on the last observation | `llm.rs:426-435`, `MESH_AUTO_CATALOG_TTL` `llm.rs:30` |
| Fallback | one retry through `auto` on a mesh-specific 5xx or on pseudo tool-call markup | `llm.rs:361-390` |
| Cooldown | 30 s of `auto` with no probing after any fallback | `llm.rs:397-404`, `MESH_AUTO_COOLDOWN` `llm.rs:32` |
| Fail-open | an unreachable or unparseable catalog preserves the last confirmed route; a cold start means `auto` | `llm.rs:452-457`, tested `llm.rs:2104` |

Explicit limits of the capability, all asserted by tests:
- **It only applies to the literal model string `auto`.** An explicit model is never rewritten and never triggers a probe (`llm.rs:2155`).
- **It only applies to `Provider::OpenAi`.** `Anthropic`, `Databricks`, and `DatabricksV2` ignore the flag entirely (`llm.rs:411`).
- **An explicit `mesh` request gets no safety net.** The fallback requires `effective_model == "auto"` (`llm.rs:355-356`), so asking for `mesh` directly surfaces the mesh's own error (`llm.rs:2136`).
- **The 2 s probe bound does not extend to inference.** The `.timeout(...)` is on the catalog `GET` only (`llm.rs:488`); the POST that follows still has no wall-clock bound.
- **Fallback is once per call, not a loop.** The retry result is returned as-is (`llm.rs:371-372`, `llm.rs:388-389`).
- **No fan-out or model selection happens agent-side.** The agent picks between exactly two model *names*; all ensembling, worker dispatch, and reduction live in the mesh router. `MESH_MOA_UNAVAILABLE_MESSAGE` (`llm.rs:34`) and the `moa_failure` error type (`llm.rs:1386`) are the only two things the agent knows about MoA internals.
- **The tuning constants are not configurable.** TTL, timeout, cooldown, and the confirmation count are `const`s (`llm.rs:30-33`) with no env override — grep for `MESH_AUTO` in `config.rs` returns nothing.
- **Only the Chat/Responses dialects participate.** `Llm::summarize` calls `openai_request` too (`llm.rs:253-284`), so a summarization on a mesh-enabled config is *also* subject to the policy; nothing special-cases it, and no test covers it.

#### Capabilities exposed per dialect
| Capability | Anthropic | OpenAI Chat | Responses |
|---|---|---|---|
| Tool calling | yes (`llm.rs:721-729`) | yes + `tool_choice:"auto"` (`llm.rs:822-838`) | yes + `tool_choice:"auto"` (`llm.rs:931-955`) |
| Parallel tool calls | yes, batched tool_results (`llm.rs:714-722`) | yes, contiguous `role:"tool"` run (`llm.rs:778-818`) | yes, one `function_call_output` each (`llm.rs:906-910`) |
| Image tool results | native `image` block (`llm.rs:751-755`) | data-URI `image_url` on a trailing user message (`llm.rs:854-865`) | `input_image` data URI (`llm.rs:912-925`) |
| Reasoning/thinking request | `thinking` + `output_config` (`llm.rs:733-740`) | `reasoning_effort` string (`llm.rs:831`) | `reasoning: {effort}` (`llm.rs:950`) |
| Reasoning content in response | `thinking` blocks (`llm.rs:1167-1174`) | `reasoning_content` / `reasoning` on the message (`llm.rs:1208-1218`) | `summary[].text` on `reasoning` items (`llm.rs:1037-1058`) |
| Cache-token accounting | yes, summed (`llm.rs:1123-1132`) | yes, summed (`llm.rs:1139-1148`) | **no** — `input_tokens` only (`llm.rs:1080`) |
| Collective (`mesh`) routing | no | yes, via `resolve_openai_model` (`llm.rs:410-469`) | yes, same path — endpoint choice is independent of model choice (`llm.rs:544-546`) |
| Streaming | no | no (`"stream": false`, `llm.rs:828`) | no (no `stream` key emitted) |

#### Reasoning-effort levels
Seven levels, parsed from `BUZZ_AGENT_THINKING_EFFORT` (`config.rs:622-637`), case-insensitive and trimmed:

| Level | OpenAI string | Anthropic effort string | Anthropic manual budget |
|---|---|---|---|
| `none` | `none` | `low` (defensive only) | 0 |
| `minimal` | `minimal` | `low` (defensive only) | 0 |
| `low` | `low` | `low` | 1 024 |
| `medium` | `medium` | `medium` | 8 192 |
| `high` | `high` | `high` | 32 768 |
| `xhigh` | `xhigh` | `xhigh` | 32 768 (clamped) |
| `max` | `max` | `max` | 32 768 (clamped) |

Sites: string mapper `config.rs:45-57`, Anthropic mapper `config.rs:60-71`, budget mapper `config.rs:33-43`. All three are exhaustively tested (`config.rs:1359`, `config.rs:1372`, `config.rs:1383`).

#### OpenAI model families and their effort limits
`openai_efforts_for_model` (`config.rs:333-401`) is the doc-verified capability table:

| Family | Supported efforts | Table const | Match tokens |
|---|---|---|---|
| `gpt-5-pro` / `gpt5-pro` | `high` only | `GPT5_PRO` (`config.rs:335`) | `config.rs:367` |
| `gpt-5.6` / `gpt5.6` / `gpt-5-6` / `gpt5-6` | `none, low, medium, high, xhigh, max` | `GPT5_6` (`config.rs:336-343`) | `config.rs:369-373` |
| `gpt-5.5`, `gpt-5.4` (+ `gpt5.5`, `gpt-5-5`, `gpt5-5`, `gpt5.4`, `gpt-5-4`, `gpt5-4`) | `none, low, medium, high, xhigh` | `GPT5_5_AND_5_4` (`config.rs:344-350`) | `config.rs:375-384` |
| `gpt-5.1` / `gpt5.1` / `gpt-5-1` / `gpt5-1` | `none, low, medium, high` | `GPT5_1` (`config.rs:351-356`) | `config.rs:386-390` |
| `gpt-5` base / `gpt5` base | `minimal, low, medium, high` | `GPT5_BASE` (`config.rs:357-362`) | `config.rs:392` |
| anything else — including `mesh` and `auto` | not doc-verified → `None`; `max` clamps to `xhigh` | — | `config.rs:394-397`, clamp `config.rs:541-548` |

The `none` vs `minimal` split is the notable asymmetry: base `gpt-5` supports `minimal` but not `none`; the versioned families support `none` but not `minimal` (`config.rs:322-324`). Both directions are tested (`config.rs:2263`, `config.rs:2448`, `config.rs:2478`).

Match ordering matters and is documented: `-pro` is checked before the versioned strings so `gpt-5-pro` cannot fall into the base bucket (`config.rs:326-327`, `config.rs:365-367`), tested by `openai_efforts_for_model_pro_before_base_gpt5` (`config.rs:2409`).

Boundary safety is a first-class feature here, with two distinct matchers:
- `gpt5_token_matches` (`config.rs:239-254`) requires end-of-string or `-` after the token, so `gpt-5.1` does not match `gpt-5.10` and `gpt-5-4` does not match `gpt-5-4o`.
- `gpt5_base_matches` (`config.rs:267-311`) additionally rejects short 1-3-digit numeric suffixes (`-10`, `-10-preview`) as version-like while accepting 4+-digit date/build segments (`-1106`, `-0514`) and non-digit capability suffixes (`-pro`).

Eight boundary tests cover this (`config.rs:2286`, `config.rs:2304`, `config.rs:2323`, `config.rs:2363`, `config.rs:2387`, `config.rs:2398`, `config.rs:2409`, plus `config.rs:2516`).

#### Anthropic model families and their thinking shapes
| Family | Shape | Predicate site |
|---|---|---|
| `claude-3*` (all 3.x) | manual `budget_tokens` | `config.rs:587` |
| `claude-opus-4-5` (exact) | manual `budget_tokens` | `config.rs:587` |
| `claude-opus-4-6`, `-4-7`, `-4-8` | adaptive + `output_config.effort` | `config.rs:606-608` |
| `claude-sonnet-5*` | adaptive | `config.rs:610` |
| `claude-sonnet-4-6*` | adaptive | `config.rs:612` |
| `claude-fable-5*` | adaptive (always-on) | `config.rs:614` |
| `claude-mythos-5*` | adaptive (always-on) | `config.rs:615` |
| `claude-mythos-preview*` | adaptive (default-on) | `config.rs:618` |
| any other `claude-*` or non-Anthropic name | both fields omitted | `config.rs:171-176` |

`xhigh` support is a narrower set than adaptive membership (`anthropic_model_supports_xhigh`, `config.rs:184-190`): Opus 4.7, Opus 4.8, Sonnet 5, Fable 5, Mythos 5 — explicitly **not** Opus 4.6, Sonnet 4.6, or Mythos Preview (`config.rs:196-198`). `max` is supported by all adaptive families (`config.rs:428-433`).

Every family is individually tested: Claude 3 (`config.rs:1408`), Opus 4.5 (`config.rs:1497`), Opus 4.6 (`config.rs:1790`, `config.rs:1804`), Opus 4.7 (`config.rs:1464`), Opus 4.8 (`config.rs:1453`), Sonnet 5 (`config.rs:1475`), Sonnet 4.6 (`config.rs:1486`), Fable 5 (`config.rs:2063`), Mythos 5 (`config.rs:2074`), Mythos Preview (`config.rs:2085`), unknown Claude (`config.rs:1542`), non-Claude (`config.rs:1567`).

#### Catalog-prefix tolerance
`strip_catalog_prefix` (`config.rs:89-97`) makes the Anthropic classifiers work regardless of how a gateway names its endpoints: it finds the first occurrence of `claude-` or `gpt-` and drops everything before it, rather than maintaining an allowlist of prefixes. So `databricks-claude-opus-4-7`, `goose-claude-fable-5`, and `team-x-claude-opus-4-7` all resolve correctly. Tested for `databricks-` (`config.rs:1582`, `config.rs:1592`, `config.rs:1605`), `goose-` (`config.rs:1618`, `config.rs:1631`), and an arbitrary `team-x-` prefix (`config.rs:1643`).

Note: `openai_efforts_for_model` does **not** call `strip_catalog_prefix` — it relies on `gpt5_token_matches`/`gpt5_base_matches` tolerating a prefix because `-` is a legal boundary character (`config.rs:250-252`, `config.rs:265`). Both approaches work but the crate has two different prefix-tolerance mechanisms. The mesh catalog adds a third normalisation idiom, narrower than either: `id.replace("@main", "")` (`llm.rs:60`), which strips a revision *suffix* rather than a vendor prefix.

#### Auto-upgrade from Chat Completions to Responses
When `OPENAI_COMPAT_API=auto` and a Chat Completions call comes back with a "use `/v1/responses`" provider error, the process latches into Responses mode for its remaining lifetime (`llm.rs:90-94`, `llm.rs:561-570`, latch at `llm.rs:665-672`). Matcher covers the literal path and two prose phrasings (`llm.rs:963-968`). This is orthogonal to mesh routing: the model is resolved first (`llm.rs:354`) and the endpoint second (`llm.rs:544-546`), so a `mesh` request can be upgraded to `/responses` like any other.

#### Summarization
`Llm::summarize` (`llm.rs:230-328`) is a separate, tool-free single-turn completion used for context handoff. It takes an explicit `max_output_tokens` argument (callers pass `HANDOFF_MAX_OUTPUT_TOKENS` = 8192, `config.rs:652`, from `handoff.rs:51` and `handoff.rs:197`) rather than `cfg.max_output_tokens`, and it never sends tools, never sends thinking/reasoning config, and discards `LlmResponse.reasoning` — it returns only `.text` (`llm.rs:249`, `llm.rs:285`, `llm.rs:325`).

Its Anthropic body (`llm.rs:240-248`) is hand-built rather than routed through `anthropic_body`, and its Responses body passes `input` as a bare string (`llm.rs:264`) rather than the item array `responses_body` builds (`llm.rs:946`). Both forms are accepted by the respective APIs, but the duplication means a future body-shape fix must be applied in two places. There are still no tests for `summarize` at any level — grep for `summarize` in the `llm.rs` test module returned zero matches. Since `16d4ec33` it also silently inherits the mesh policy (it calls `openai_request` at `llm.rs:252-284` with `tools_supplied: false` passed at `llm.rs:256`), which means a handoff summary can be routed to the collective and can trigger the 5xx fallback — but never the pseudo-tool-call fallback, since that arm requires `tools_supplied` (`llm.rs:375`).

#### Hook-server allowlist
`HookServers` (`config.rs:1063-1112`) supports three modes from a comma-separated `MCP_HOOK_SERVERS`: unset/empty/whitespace-only → `None` (hooks off, the default), a lone `*` → `All` (`config.rs:1108-1109`), otherwise `Only([names])` with entries trimmed and empties dropped (`config.rs:1096-1100`). A mixed `*,foo` deliberately degrades to a literal `Only(["*","foo"])` to avoid silently widening scope on a typo (`config.rs:1104-1110`). Ten tests cover the parser and `allows()` (`config.rs:1119-1203`).

#### Limits and known non-features
| Limit | Value | Site |
|---|---|---|
| LLM response body | 16 MiB, checked twice (Content-Length + streaming) | `llm.rs:25`, `llm.rs:1484-1513` |
| LLM error body captured | 4 KiB | `llm.rs:26`, `llm.rs:1268-1284` |
| Attempts per LLM POST | 3 | `llm.rs:1285` |
| Backoff ceiling | 8 s (never reached at 3 attempts) | `llm.rs:1287` |
| Auth refresh attempts per call | 1 | `llm.rs:631-649` |
| Mesh fallback retries per call | 1 | `llm.rs:361-390` |
| Mesh catalog probes per 5 s window | 1 | `llm.rs:426-435` |
| Connect timeout | 10 s | `llm.rs:110` |
| Read (inactivity) timeout | `BUZZ_AGENT_LLM_TIMEOUT_SECS`, default 240 s | `llm.rs:111`, `config.rs:810` |
| Mesh catalog probe timeout | 2 s | `llm.rs:488` |
| Total wall-clock timeout on an inference POST | **absent** | `Llm::new` sets only connect + read (`llm.rs:109-113`) |

Not features of this group: streaming, token counting before send, prompt caching control, cross-*provider* failover (a failed provider is not retried on a different provider — `Llm::complete`'s `match` has one arm per provider and no fallback), and multi-key rotation. The mesh work adds a narrow, *intra*-provider failover — `mesh` → `auto` within `Provider::OpenAi` (`llm.rs:361-390`) — which is the first fallback of any kind in this file; it does not generalise, and there is no configuration to extend it to other model pairs.


## Module: buzz-agent — MCP registry, OAuth, hints & catalog (`crates/buzz-agent/src`)
### Aspect: Features

#### Feature inventory

| Feature | Entry point | Limits / caveats |
|---|---|---|
| Host stdio MCP servers per session | `McpRegistry::spawn_all` (`mcp.rs:172`) | 16 servers, 128 tools, stdio only; spawn is serial, any single failure fails `session/new` (`mcp.rs:217`) |
| Merge all servers into one namespaced tool catalog | `mcp.rs:230-261`, `tools()` (`mcp.rs:286`) | `server__tool`, `__` banned in both halves, qname ≤64 bytes |
| Fan tool calls out to the owning child | `McpRegistry::call` (`mcp.rs:485`) | exact-match routing; tool-set drift and non-object arguments rejected locally (`mcp.rs:500-506`, `mcp.rs:566-575`) |
| Bounded, lossy-but-marked tool results | `tool_result_content` (`mcp.rs:913`) | 8 MiB total / 50 KiB text by default; text middle-elided, images kept whole or marker-replaced |
| Process-group kill of child + grandchildren | `process_group(0)` (`mcp.rs:733`) + `killpg` (`mcp.rs:845`) | unix only; on non-unix the "kill" is a log line and reliance on `Drop` (`mcp.rs:855-857`) |
| Lazy restart of a dead server with backoff | `maybe_restart` (`mcp.rs:646`) | 3 attempts, 500 ms→30 s ±20%; only *consecutive spawn failures* are budgeted |
| Lifecycle hooks (`_Stop`, `_PostCompact`) | `call_hooks` (`mcp.rs:315`) | off by default, fail-open, 2.5 s per hook, hidden from the LLM, killed after 2 consecutive timeouts |
| MCP cancellation propagation | `fire_and_forget_cancel` (`mcp.rs:788`) | best-effort `notifications/cancelled`, never blocks the agent |
| Interactive browser OAuth 2.0 PKCE login | `interactive_login` (`auth.rs:235`), CLI at `lib.rs:126-153` | Databricks only; needs a browser and a 60 s window |
| Silent token cache + refresh | `PkceOAuthTokenSource::bearer` (`auth.rs:246`) | on-disk JSON keyed by `sha256(discovery_url\|client_id\|scopes)`; cross-process re-read |
| Headless bearer resolution (no browser) | `try_bearer_no_browser` (`auth.rs:367`) | returns `LlmAuth` with a "run `buzz-agent auth databricks`" hint instead of prompting |
| 401-driven force refresh | `refresh_now` (`auth.rs:317`) | coalesces on token identity; terminal on refresh failure, never falls to the browser |
| `AGENTS.md` hint layering into the system prompt | `build_hints_section` (`hints.rs:219`) | `$HOME` + git-root→cwd chain, 128 KiB total |
| Multi-vendor skill discovery | `discover_skills_impl` (`hints.rs:204`) | `.agents/skills`, `.goose/skills`, `.claude/skills`, `~/.agents/skills`; first name wins |
| Lazy skill loading (`load_skill`) | `builtin.rs:16`, `builtin.rs:41` | in-process, no MCP round-trip; 32 KiB per call; only offered when ≥1 skill exists (`agent.rs:117-119`) |
| Supporting-file loading within a skill | `load_supporting_file` (`builtin.rs:118`) | pre-enumerated allowlist + canonicalise containment check |
| Databricks model-catalog discovery | `discover_databricks_models` (`catalog.rs:116`) | v1 filtered by READY+chat/completions (`catalog.rs:187-206`); v2 paginated to 20 pages (`catalog.rs:244-245`), embedding endpoints dropped by name heuristic (`catalog.rs:97-106`, `catalog.rs:370-373`) and results ordered newest-first then by name (`catalog.rs:337-344`); known-model fallback when v2 yields nothing (`catalog.rs:287-296`) |

#### stdio MCP hosting

Each `session/new` receives an `mcpServers` array (`types.rs:McpServerStdio`) and gets its own registry — sessions do not share children (`lib.rs:390-393`). The child is launched with the session's `cwd` as its working directory (`mcp.rs:729`) and inherits the agent's stderr (`mcp.rs:730`), so child diagnostics land in the agent's log stream unfiltered. Windows child processes get `CREATE_NO_WINDOW` so a GUI-launched agent does not flash console windows (`mcp.rs:992-1003`).

Limits worth stating plainly: only the child-process transport is wired (`rmcp` is pulled in with `features = ["client", "transport-child-process"]`, `Cargo.toml`), and the agent advertises `mcpCapabilities: { http: false, sse: false }` at `lib.rs:294`. There is no support for server-initiated MCP traffic because the client is built from the unit type (`mcp.rs:83`) — including no handling of `notifications/tools/list_changed` (`grep -rn 'list_changed\|sampling\|roots' crates/buzz-agent/src` → zero matches).

#### Tool fan-out and result shaping

The registry is the single tool namespace the LLM sees; `agent.rs:116-119` appends the built-in `load_skill` on top. Results are shaped for a context window rather than for fidelity: text is middle-elided so both "what ran" and "how it ended" survive (`mcp.rs:880-885`), images pass whole because providers bill them as tiles (`mcp.rs:28-32`), and everything else (audio, embedded resources) degrades to a one-line marker (`mcp.rs:974-985`). Every elision leaves an inline marker, so the model can tell it is looking at a truncated result — except for the two `load_skill` caps, which cut silently (`builtin.rs:102-105`, `builtin.rs:206-209`).

#### Process-group kill

`process_group(0)` is set on the `tokio::process::Command` before spawn (`mcp.rs:732-733`), and every teardown path calls `killpg(SIGKILL)`: `Drop for Server` (`mcp.rs:116-122`), spawn abandonment (`PgidGuard`, `mcp.rs:741-756`), explicit kill (`mcp.rs:445-447`), and transport failure (`mcp.rs:464-466`). The crate README claims this is done "via `setpgid(0,0)` in `pre_exec`" — the code uses the safe `Command::process_group` API instead, which matters because `lib.rs:1` is `#![forbid(unsafe_code)]` and a `pre_exec` implementation would not compile.

On non-unix targets the "kill" is a no-op that logs "relying on Drop to kill MCP …" (`mcp.rs:854-857`), so the grandchild guarantee is unix-only. The fake MCP server supports `FAKE_MCP_SPAWN_GRANDCHILD` (`tests/bin/fake_mcp.rs:17`, `:228`) but no test uses it — `grep -rn 'FAKE_MCP_SPAWN_GRANDCHILD' --include='*.rs' .` matches only those two lines in the fake itself.

#### Lifecycle hooks

Delivered capability: any MCP server can expose `_`-prefixed tools; the agent calls them at two lifecycle points (`_Stop` before honouring `end_turn`, `agent.rs:224-236`; `_PostCompact` after a context handoff, `handoff.rs:73-92`) and injects non-empty responses back into the conversation. The registry contributes the discovery, allowlisting, parallel dispatch, per-hook timeout, deterministic ordering, and the kill-on-second-timeout escalation (`mcp.rs:315-419`).

Limits: hooks are advisory (dropped on any error), invisible to the model, opt-in per server via `MCP_HOOK_SERVERS`, and non-cancellable by design (`mcp.rs:342-347`). Hook results are capped at 16 KiB each but the *number* of hook results is not capped (see Business Rules).

#### Browser OAuth login with token caching

`buzz-agent auth databricks` (`lib.rs:126-153`) runs the flow once and prints where the token landed (`lib.rs:147`). The runtime path is silent: `Llm` asks for a bearer per request (`llm.rs:49`), the source serves it from memory, then disk, then a refresh grant, and only then opens a browser (`auth.rs:246-297`). Discovery-only paths use the no-browser variant so a headless harness degrades instead of hanging (`auth.rs:299-301`, `auth.rs:367-423`).

Limits: single provider shape (RFC 8414 discovery + public-client PKCE), no device-code or client-credentials flow (`grep -n 'device_code\|client_credentials' auth.rs` → zero matches), no logout/revocation (`grep -n 'revoke' auth.rs` → zero matches), no way to point the cache at a different directory in production (`cache_dir_override` is documented as test-only, `auth.rs:102-106`), and a hard `$HOME` dependency — no `$HOME`, no OAuth (`auth.rs:457-458`).

#### Hints and skills injection

Two independent capabilities behind one entry point (`hints.rs:219-221`), both gated by `BUZZ_AGENT_NO_HINTS` (`config.rs:825`, applied at `lib.rs:355-360`):

- project hints: the `AGENTS.md` of `$HOME` plus every directory from the git root down to `cwd`, concatenated most-general-first so the closest file has the last word (`hints.rs:40-84`);
- skills: name + description only, with an instruction to call `load_skill` for the body (`hints.rs:239-247`).

The lazy-loading design is deliberate and asserted: bodies must not appear in the system prompt (`hints.rs:479-495`). Limits: discovery happens once per session with no invalidation; skill descriptions are unbounded and land in the system prompt verbatim; the combined prompt is validated against a 512 KiB ceiling by the caller, and exceeding it fails `session/new` outright rather than truncating (`lib.rs:375-388`, constant at `config.rs:639`).

#### `load_skill`

Two request forms, one tool (`builtin.rs:16-39`). Both read from the blocking pool so a large file cannot stall a Tokio worker (`builtin.rs:68-71`, `builtin.rs:192`, `builtin.rs:197`), and error results are model-readable: they enumerate available skill names or available relative paths (`builtin.rs:60-65`, `builtin.rs:151-172`). Supporting files are advertised inside the loaded body with a copy-pasteable call form (`builtin.rs:88-98`).

Limits: 32 KiB per call with a silent head cut; no directory listing tool (the model can only see what discovery enumerated); a skill whose name contains `/` is unreachable in the plain form (`builtin.rs:52`); and symlinked supporting files are listed but refused at load time (interaction of `builtin.rs:143-149` with `builtin.rs:196`).

#### Databricks model discovery

Used in two places: `session/new` advertises `availableModels` so a UI model picker and `session/set_model` can work (`lib.rs:440-472`), and the desktop resolves a model list without spawning the agent (`desktop/src-tauri/src/commands/agent_models.rs:709-758`). Successful results are cached process-wide in a `OnceCell`; failures are not cached, so the next session retries (`lib.rs:312-329`).

What the v2 path returns, as of `8eb6e3eb`: every named endpoint from up to 20 pages, **minus** anything the name heuristic calls an embedding endpoint (`is_chat_capable_endpoint`, `catalog.rs:97-106`, applied at `catalog.rs:370-373`), ordered newest-`created_timestamp`-first with a name tiebreak (`sort_v2_endpoints_newest_first`, `catalog.rs:337-344`, called once after pagination at `catalog.rs:298`). The stated motivation is that the gateway pages Databricks-managed endpoints before workspace-created ones, each alphabetically, which buries a newly released frontier model mid-list (`catalog.rs:329-333`). Before this the v2 path applied no filtering and preserved gateway order.

Limits: Databricks only (`catalog.rs:123-130`); no timeout on the HTTP calls (`Client::new()` at `catalog.rs:120` versus the configured client at `llm.rs:53-57`); v2 stops silently at 2 000 endpoints (`catalog.rs:244-245`); v1 can legitimately return an empty list despite the doc promising non-empty (`catalog.rs:110` vs `catalog.rs:163`); and for non-Databricks providers the caller does not call this at all — it reports only the configured model (`lib.rs:455-457`).

Limits specific to the new filter and sort: the chat-capability test is a name heuristic with no wire signal behind it — the v2 payload carries neither `task` nor a readiness field (`catalog.rs:85-88`) — so it drops on `embedding` as a substring plus `bge`/`gte` as whole `-`-delimited segments (`catalog.rs:99-105`) and keeps everything else, including image endpoints, by design (`catalog.rs:93-96`). A future embedding family named outside that vocabulary is still offered and still fails at send time; conversely a chat endpoint with `embedding` in its name is dropped with no log (`catalog.rs` has zero `tracing::` calls). The sort reads `created_timestamp` in either wire shape — JSON string or bare number (`endpoint_created_ms`, `catalog.rs:320-325`) — and endpoints with an absent or unparseable timestamp sort last rather than first (`catalog.rs:340-342`), so a gateway that stops sending the field degrades to pure alphabetical order rather than to an error.

#### Documented-but-absent / undocumented features

| Claim | Where | Reality |
|---|---|---|
| MCP child env whitelist is "`PATH`, `HOME`, `TERM`, `LANG`, `LC_ALL`, `TMPDIR`" | crate README, Security Model table | the allowlist has 17 entries including `SSH_AUTH_SOCK`, `GIT_*`, `NOSTR_PRIVATE_KEY`, `BUZZ_PRIVATE_KEY`, `BUZZ_AUTH_TAG` (`mcp.rs:39-63`) |
| process group established "via `setpgid(0,0)` in `pre_exec`" | crate README, Security Model table | `Command::process_group(0)` (`mcp.rs:733`); no `unsafe`, none possible under `#![forbid(unsafe_code)]` (`lib.rs:1`) |
| hints / skills / `load_skill` | — | not mentioned anywhere in `crates/buzz-agent/README.md` (`grep -cn 'load_skill\|skills\|AGENTS.md' crates/buzz-agent/README.md` → 0); the only repo-level trace is `CHANGELOG.md:738` |
| OAuth / `buzz-agent auth` subcommand | — | the README mentions OAuth twice — a quick-start comment (README:53) and the `DATABRICKS_TOKEN` row (README:143, "If unset, Databricks uses browser OAuth + refresh cache") — but never documents the `auth` subcommand, the 60 s browser window, or the cache path |
| model discovery / `availableModels` | — | undocumented in the crate README; the ACP transcript there still shows a `session/new` result with only `sessionId` (README "ACP Transcript" section) |

#### Test coverage — Features

End-to-end coverage exists for hosting (`tests/regressions.rs:241 init_session`, `:746 init_session_with_fake_mcp`), caps (`:354`, `:420`, `:606`, `:681`), init timeout (`:307`), hooks (`:787`-`:1112`, `:1514`), cancellation propagation (`:1573`, `:1710`), hints/skills/`load_skill` (`tests/hints_integration.rs:193`, `223`, `254`, `300`, `343`, `385`, `469`, `517`), and the OAuth cache/refresh feature set (`tests/databricks_oauth.rs:105`-`:305`).

No test exercises: process-group kill of grandchildren, restart-after-death as a user-visible feature, `interactive_login`, the model-discovery feature end to end (only the pure fallback helper is tested, `lib.rs:886`-`:959`), or the Windows-specific behaviours — both Windows tests are `#[cfg(windows)]` (`mcp.rs:1016-1033`, `mcp.rs:1127-1140`) and cannot run on the macOS/Linux CI paths this repo builds; the non-Windows counterpart asserts only "didn't crash" (`mcp.rs:1118-1125`).


## Module: buzz-dev-mcp (`crates/buzz-dev-mcp`)
### Aspect: Features

#### Tool inventory — registered vs defined

Seven tools are annotated `#[tool(...)]` inside the single `#[tool_router] impl
DevMcp` block (`crates/buzz-dev-mcp/src/lib.rs:30-125`), and `Self::tool_router()`
is installed into `DevMcp` at construction (`lib.rs:32-38`). There are **no
defined-but-unregistered tools** — every `#[tool]` method sits inside the router
block, and there is no second router or conditional registration anywhere in the
crate.

| Tool | Registered | Implementation | Purpose |
|---|---|---|---|
| `shell` | yes (`lib.rs:40-50`) | `shell.rs:130-323` | run a command string through an OS shell, ephemeral process per call |
| `read_file` | yes (`lib.rs:52-61`) | `read_file.rs:23-63` | line-numbered windowed text read |
| `view_image` | yes (`lib.rs:63-72`) | `view_image.rs:88-107` | load an image from path/URL/data-URL as an MCP image block |
| `str_replace` | yes (`lib.rs:74-83`) | `str_replace.rs:25-106` | atomic find-and-replace returning a unified diff |
| `todo` | yes (`lib.rs:85-98`) | `todo.rs:71-94` | read or replace the in-process session task list |
| `_Stop` | yes (`lib.rs:101-110`) | `todo.rs:99-112` | lifecycle hook returning an objection when items are open |
| `_PostCompact` | yes (`lib.rs:114-123`) | `todo.rs:113-124` | lifecycle hook re-emitting todo state after history compaction |

Tool count is bounded downstream, not here: `buzz-agent` caps a session at
`MAX_TOOLS_PER_SESSION = 128` and `MAX_MCP_SERVERS = 16`
(`crates/buzz-agent/src/mcp.rs:23`, `crates/buzz-agent/src/mcp.rs:26`).

#### Multicall personalities

The same binary serves six roles selected by `argv[0]` (`lib.rs:138-171`). Five
are made reachable to shell children through symlinks in the shim tempdir
(`shim.rs:31-40`).

| Personality | Provided by | Nature |
|---|---|---|
| `buzz-dev-mcp` (or any unmatched name) | `lib.rs:173-186` | MCP server over stdio |
| `rg` | `rg.rs:11-16` | delegate to system ripgrep, else a built-in substring-search fallback |
| `tree` | `tree.rs:18-135` | gitignore-aware directory listing with line counts |
| `buzz` | `buzz_cli::run_from_args` (`lib.rs:168-171`) | the full Buzz relay CLI, bundled in-process |
| `git-credential-nostr` | `git_credential_nostr::run` (`lib.rs:151`) | git credential helper for Nostr-authed push/fetch |
| `git-sign-nostr` | `git_sign_nostr::run` (`lib.rs:152`) | git object signing with a Nostr key |

The dispatch is ordered so that `rg`, `tree`, and the two git helpers exit before
any tokio runtime, tracing subscriber, or allocation beyond argv parsing is set up
(`lib.rs:146-153`).

#### Session bootstrap

`SharedState::new` builds an `instructions` string once and serves it as the MCP
server's `instructions` field (`shell.rs:40-63`, `lib.rs:128-135`). It contains
the working directory, a detected stack fingerprint, the resolved shell name with
the `BUZZ_SHELL` override hint, and an instruction to pass `workdir` per call
instead of using `cd` (`shell.rs:75-92`).

Stack detection is a flat existence check for nine marker files in the cwd —
`Cargo.toml`, `package.json`, `go.mod`, `pyproject.toml`, `requirements.txt`,
`Gemfile`, `pom.xml`, `build.gradle`, `build.gradle.kts` — sorted and
comma-joined, or `"unknown"` (`shell.rs:94-117`). It does not recurse.

A conditional extra line, `"Buzz relay configured. Run `buzz --help` …"`, appears
only when **both** `BUZZ_RELAY_URL` and `BUZZ_PRIVATE_KEY` are present in the
environment (`shell.rs:77-83`) — making the instructions string an observable
signal of whether a private key was injected.

#### Ephemeral git identity for shell children

When `NOSTR_PRIVATE_KEY` is present at startup, `Shim::install` provisions a
complete git identity for every subsequent `shell` call without writing to any
user config file (`shim.rs:51-68`, `shim.rs:178-216`). Ten settings are passed as
`GIT_CONFIG_KEY_n`/`GIT_CONFIG_VALUE_n` pairs:

| Setting | Value |
|---|---|
| `user.name` | the derived `npub` (`shim.rs:181`) |
| `user.email` | `<pubkey_hex>@<relay_host>`, falling back to `<pubkey_hex>@buzz` for localhost/127.* or an unset relay (`shim.rs:154-172`) |
| `credential.helper` | `nostr` (`shim.rs:187`) |
| `credential.useHttpPath` | `true` — required for NIP-98 against the full repo-root URL (`shim.rs:188-191`) |
| `nostr.keyfile` | the `0600` keyfile path (`shim.rs:191`) |
| `gpg.format` | `x509` (`shim.rs:192`) |
| `gpg.x509.program` | `git-sign-nostr` (`shim.rs:193`) |
| `commit.gpgSign` / `tag.gpgSign` | `true` (`shim.rs:194-195`) |
| `user.signingkey` | pubkey hex (`shim.rs:196`) |

The index base composes with any pre-existing `GIT_CONFIG_COUNT` rather than
clobbering it (`shim.rs:199-215`).

#### Output-artifact escape hatch

Truncated `shell` output is not lost: the captured bytes (up to 10 MiB) are
written to `{session}/artifacts/{callid:06}.{stdout|stderr}.txt` and the absolute
path is returned in the JSON body, so the agent can page through the full log with
`read_file` (`shell.rs:914-928`, `shell.rs:309-320`). The last 8 files are kept
(`shell.rs:971-981`).

#### Cancellation support

`shell` receives the `rmcp` `RequestContext` and threads `context.ct` into
`shell::run` (`lib.rs:44-50`), which races the child against the cancellation
token in a `biased` `tokio::select!` (`shell.rs:218-239`). This is exercised
end-to-end from the agent side by
`cancel_kills_inflight_tool_via_mcp_notification`
(`crates/buzz-agent/tests/regressions.rs:1572-1600`), which asserts a `sleep 60`
tool call completes in under 5 s and the process is actually dead. That test
skips itself when the `buzz-dev-mcp` binary has not been built
(`crates/buzz-agent/tests/regressions.rs:1592-1600`).


## Module: buzz-cli — dispatch, relay client & validation (`crates/buzz-cli/src`)
### Aspect: Features

This group is the CLI's front door and transport: what it *offers* is a single
non-interactive binary that turns argv into signed Nostr events and relay JSON,
with machine-readable success and failure envelopes. The per-command behavior
lives in `commands/` (sibling scope); everything below is what this layer itself
makes possible or prevents.

#### What an operator or agent can do through this layer

| Capability | How | Site |
|---|---|---|
| Run 21 command groups / ~100 subcommands from one binary | `enum Cmd` + dispatch match | `lib.rs:175-240`, `lib.rs:1771-1792` |
| Point at any relay per invocation | `--relay` / `BUZZ_RELAY_URL`, normalized from ws/wss | `lib.rs:81-82`, `client.rs:1291-1297` |
| Act as any identity per invocation | `--private-key` / `BUZZ_PRIVATE_KEY` | `lib.rs:84-85`, `lib.rs:1749-1750` |
| Act on behalf of an owner (NIP-OA delegation) | `--auth-tag` / `BUZZ_AUTH_TAG`, verified locally then injected into every event and sent as `x-auth-tag` | `lib.rs:1752-1767`, `client.rs:588-621` |
| Get agent-friendly reduced output | `--format compact` | `lib.rs:92-93` — but only for `messages`, `channels`, `users`, `feed`, `moderation` (`lib.rs:1772-1790`) |
| Work offline | `pack validate` / `pack inspect` short-circuit before any auth or network use | `lib.rs:1736-1743` |
| Pipe content in | `-` means stdin for content-style flags; `read_file_or_stdin` for path-style flags | `validate.rs:162-193` |
| Survive flaky links | 3-attempt retry with full jitter, `retry in Ns` honoring, body-transfer failures inside the retry boundary | `client.rs:638-681` |
| Know whether a failed write is safe to re-run | `retryable` field + `DeliveryUnknown` category | `error.rs:74-88`, `error.rs:117-125` |
| Page through large histories without flags | automatic `(until, before_id)` cursor following, 500/page | `client.rs:683-727` |
| Publish ephemeral kinds the HTTP bridge rejects | WS publish with NIP-42 AUTH | `client.rs:1073-1098` |
| Upload/download relay media with Blossom auth | `upload_file`, `download_media` | `client.rs:1100-1256` |
| Read structured moderation queue/audit rows | NIP-98-authed GET of arbitrary relay paths | `client.rs:836-856`, used by `commands/moderation.rs:114-128` |
| Request owner-reviewed agent creation/edits | encrypted observer frames, kind 24200 | `agent_management.rs:87-181` |
| Script against stable command names | inventory-drift tests fail the build on rename | `lib.rs:1806-2034` |

#### Limits and non-features

- **No `--version`.** Not declared on `#[command(...)]` (`lib.rs:62-78`);
  `buzz --version` exits 1 as an unknown argument (verified against the built
  binary), despite `lib.rs:48-52` handling a `--version` branch that can never
  fire.
- **`--format` is positional-sensitive and mostly inert.** It must precede the
  subcommand (verified: after-subcommand use exits 1), and 16 of 21 groups
  ignore it with no diagnostic.
- **No shell completions, no man page, no config file.** `grep -rn 'clap_complete\|generate(' lib.rs`
  → no matches; nothing in this group reads a dotfile or app-data path
  (`grep -rn 'dirs::' lib.rs client.rs validate.rs error.rs agent_management.rs`
  → zero matches).
- **No verbosity, quiet, colour or logging controls.** The crate has no logging
  framework at all (`grep -rn 'tracing\|log::' lib.rs client.rs validate.rs error.rs agent_management.rs main.rs`
  → zero matches). Diagnostics are limited to the single-line error envelope.
- **No proxy, CA-bundle, client-cert or TLS-pinning options**, and no
  `--insecure`. Transport config is whatever `reqwest` defaults to
  (`client.rs:547-551`).
- **Retry/backoff is not tunable.** Attempts, jitter bases and the 30 s hint cap
  are `const` (`client.rs:122-130`); only the two timeout values are env-tunable
  (`client.rs:140-150`).
- **One relay, one identity, one command per process.** No batch/multi-relay
  mode, no fan-out; `BuzzClient` is constructed once per run (`lib.rs:1768`).
- **No interactive prompts or confirmations** anywhere in this layer — every
  destructive subcommand (`channels delete`, `moderation ban`, `mem rm`) is
  immediate once parsed.
- **Content ceilings are hard.** 64 KiB per content field
  (`validate.rs:4`, `:64-72`), 60 KiB diffs before truncation
  (`validate.rs:7`), 50 MiB image / 500 MiB video uploads
  (`client.rs:73-76`), 5 allowed MIME types (`client.rs:64-71`) — so PDFs, SVGs,
  archives and non-mp4 video cannot be uploaded through `buzz upload file`.
- **Media downloads are relay-origin-only.** A URL for any other host is
  refused rather than fetched (`client.rs:305-315`), so the CLI cannot be used
  as a general fetcher — deliberate, since it would otherwise sign a Blossom
  capability for a foreign host.
- **`count` is unreachable.** `POST /count` is implemented but has zero callers
  and is marked `#[allow(dead_code)]` (`client.rs:802-834`;
  `grep -rn 'client.count(' commands/` → no matches), so COUNT filters are not
  exposed to users.
- **Unbounded reads are possible.** `query_all` (`client.rs:724-727`) keeps
  paging with no cap, and `read_or_stdin` (`validate.rs:162-175`) slurps stdin
  into memory with no size check, so a broad filter or a huge pipe is bounded
  only by RAM.
- **No `--kinds` on `messages search`** (`lib.rs:472-489`), so a caller cannot
  narrow a search the way `AGENTS.md` gotcha 3 instructs; verified as an exit-1
  usage error.


## Module: buzz-cli — channel, message & social commands (`crates/buzz-cli/src/commands`)
### Aspect: Features

This module is the agent-facing surface for everything a Buzz participant does in
a community: talk in channels, thread, react, run forums, manage channel
lifecycle and membership, keep a long-form knowledge base, curate custom emoji,
open DMs, and read a personal activity feed. 45 subcommands across nine groups.

#### Messaging

| Capability | Command | Limits |
|-----------|---------|--------|
| Send a channel message | `messages send --channel --content` (`messages.rs:483`) | body ≤ 64 KiB (`validate.rs:5`, checked `messages.rs:492`); `-` reads stdin unbounded before the check (`validate.rs:186-197`) |
| Send from stdin to dodge shell quoting | `--content -` (`messages.rs:488`, rationale `:484-487`) | no byte cap on the read itself |
| Reply / thread | `--reply-to <event-id>` (`messages.rs:534-538`) | resolves the parent's root via one relay fetch; **ignored** for `--kind 45001` |
| Auto-`@mention` | any `@name` in the body (`messages.rs:521`) | members of that channel only; capped at `MENTION_CAP`; silently empty on any lookup failure |
| NIP-27 inline references | `nostr:npub1…` in the body (`messages.rs:527-529`) | code regions stripped first |
| Broadcast to public Nostr | `--broadcast` (`lib.rs:369-371`) | adds `["broadcast","1"]` (`builders.rs:233`) |
| Attach media | `--file` (repeatable, `lib.rs:373-375`) | each upload is a separate Blossom call (`messages.rs:509-521`); markdown link appended per file, `![video]` vs `![image]` by MIME prefix (`messages.rs:513-517`) |
| Forum post / comment | `--kind 45001` / `45003` (`messages.rs:541-557`) | 45003 requires `--reply-to` |
| Post a diff with metadata | `messages send-diff` (`messages.rs:596`) | diff truncated at a hunk boundary at 60 KiB (`validate.rs:6`, `messages.rs:614`); language inferred from the file extension across 25 extension groups (`validate.rs:126-158`); NIP-31 `alt` text auto-composed (`messages.rs:621-626`) |
| Edit your message | `messages edit` (`messages.rs:701`) | channel re-derived from the event's `h` tag |
| Delete a message, optionally with public tombstone metadata | `messages delete [--action-id --reason-code --public-reason]` (`messages.rs:669`) | kind 9005; metadata free-text, unvalidated |
| Read a channel | `messages get [--limit --before --since --kinds]` (`messages.rs:263`) | default 50, cap 200, ascending time |
| Read a thread | `messages thread [--limit --depth-limit]` (`messages.rs:304`) | default 100, cap 500; root event included |
| Full-text / author search | `messages search [--query --author --since --limit]` (`messages.rs:340`) | community-wide only, no channel scoping; `--author` accepts hex, npub or a display name |
| Vote on forum content | `messages vote --direction up\|down` (`messages.rs:724`) | kind 45002, content `+`/`-` |

#### Channels

| Capability | Command | Notes |
|-----------|---------|-------|
| List channels | `channels list [--visibility --member --limit]` (`channels.rs:25`) | `--member` costs two queries; visibility filtering is client-side after fetch |
| Search by name | `channels search --query [--exact --include-archived --limit]` (`channels.rs:119`) | substring or exact, case-insensitive; archived excluded by default |
| Inspect one channel | `channels get` (`channels.rs:224`) | prints `null` when absent |
| Create | `channels create --name --type --visibility [--description --ttl]` (`channels.rs:282`) | `--ttl` seconds, positive and ≤ `i32::MAX` (`channels.rs:822-830`); relay archives on idle |
| Create from a desktop template | `channels create --name --template [--templates-file …]` (`channels.rs:655`) | applies type/visibility/description/canvas, resolves and adds the agent roster |
| Update metadata | `channels update [--name --description --ttl\|--no-ttl]` (`channels.rs:832`) | `--no-ttl` makes the channel permanent |
| Topic / purpose | `channels topic`, `channels purpose` (`channels.rs:864`, `:880`) | free text, no length cap client-side |
| Lifecycle | `join`, `leave`, `archive`, `unarchive`, `delete` (`channels.rs:896`–`:952`) | all single kind-90xx writes |
| Membership | `members`, `add-member [--role]`, `remove-member` (`channels.rs:244`, `:956`, `:987`) | roles owner/admin/member/guest/bot |
| Personal add policy | `channels set-add-policy --policy` (`channels.rs:1005`) | who may add you to channels; optionally restricted per deployment by env var |
| Channel canvas | `canvas get`, `canvas set --content [-]` (`channels.rs:262`, `:1049`) | markdown document per channel; `get` prints the raw body, not JSON |

The template flow is the richest single feature here. From one command it: loads a
JSON store the desktop app owns (`channel_templates.rs:60-84`), expands team
entries into persona slugs via kind 30176 (`channels.rs:399-431`), scans the
owner's kind-30177 managed agents (`channels.rs:440-464`), filters archived
identities (`channels.rs:526-587`), enforces one live instance per persona,
creates the channel, applies the canvas with `{channel.name}` /
`{template.name}` substitution (`channels.rs:725-727`), adds each resolved agent
as a `bot` member, and prints a structured report distinguishing
`members_added` / `skipped` (with reasons) / `archived_excluded` /
`member_failures` / `archive_state_warning` (`channels.rs:795-820`).

Limits worth knowing: the store must already exist on this machine — there is no
`channels template create`, and a missing store is `NotFound` with a hint to use
Buzz Desktop or `--templates-file` (`channel_templates.rs:88-93`). Cold-start
provisioning of a persona that has no live instance is explicitly out of scope
(`channels.rs:353-355`, `:472-476`), so those slugs are skipped. The channel is
created even if canvas and every member-add fail; only the roster stage can abort
before any write.

#### Notes (NIP-23 knowledge base)

| Capability | Command | Limits |
|-----------|---------|--------|
| Idempotent upsert keyed by `(you, slug)` | `notes set --name --content -` (`notes.rs:487`) | stdin capped at 1 MiB (`notes.rs:485`, checked `:496-500`); empty stdin refused without `--allow-empty` (`:501-507`) |
| Preserve publication date across edits | automatic (`notes.rs:453`) | `published_at` stamped once |
| Carry or clear title/summary/tags | `--title`/`--summary` omitted vs `""`; `--tag` vs `--clear-tags` (`notes.rs:420-450`) | `--tag` replaces, never merges |
| Read by coordinate | `notes get --naddr <naddr1…\|30023:pk:slug>` (`notes.rs:614-627`) | kind must be 30023 |
| Read by slug across authors | `notes get --name [--author\|--latest]` (`notes.rs:628-656`) | candidate scan capped at 50 |
| Body-only output for piping | `--content-only` (`notes.rs:659-665`) | appends a newline if missing |
| List | `notes ls [--author me\|all\|<ref> --tag --limit]` (`notes.rs:671`) | default own notes, 50, cap 200 |
| Delete your own note | `notes rm --name` (`notes.rs:717`) | a-tag-only kind 5; `NotFound` if you never published it |

`notes set` prints paste-ready durable references (`event_id`, `naddr`,
`coordinate`, `slug`, `title`) as plain text (`notes.rs:571-580`), and surfaces
NIP-33 LWW domination as exit 5 rather than a false success (`notes.rs:556-566`).

#### Reactions and custom emoji

| Capability | Command | Limits |
|-----------|---------|--------|
| React with a unicode emoji | `reactions add --event --emoji` (`reactions.rs:9`) | ≤64 chars (`builders.rs:466-468`); duplicates not prevented |
| React with a custom emoji | `reactions add --emoji <shortcode> --emoji-url <url>` (`reactions.rs:20-22`) | content becomes `:shortcode:` |
| Un-react | `reactions remove --event --emoji` (`reactions.rs:34`) | matches `content` exactly, so custom emoji need `:shortcode:` |
| Tally reactions | `reactions get --event` (`reactions.rs:80`) | counts are per-event, not per-user-deduped; no `limit` sent |
| See the workspace palette | `emoji list` (`emoji.rs:77`) | union of every member's set |
| Curate your own set | `emoji set --shortcode --url`, `emoji rm --shortcode` (`emoji.rs:128`, `:141`) | read-modify-write; shortcode `[A-Za-z0-9_-]` ≤64 bytes, url http(s) ≤2048 bytes |
| Back up / migrate | `emoji export [--file --scope own\|workspace]` (`emoji.rs:197`) | stable sort so `export \| import --replace` round-trips |
| Bulk load | `emoji import [--file --replace --dry-run]` (`emoji.rs:234`) | stdin capped at 10 MB (`emoji.rs:160`); merge keeps existing on conflict |

#### DMs, feed, and the social graph

| Capability | Command | Limits |
|-----------|---------|--------|
| Open a 1:1 or small-group DM | `dms open --pubkey … (1–8)` (`dms.rs:51`) | returns the relay-assigned `dm_id`; messages are then sent with `messages send --channel <dm_id>` in **plaintext** |
| Grow a DM group | `dms add-member` (`dms.rs:112`) | kind 41011 |
| Hide a conversation | `dms hide` (`dms.rs:96`) | kind 41012 |
| List conversations | `dms list [--limit]` (`dms.rs:8`) | default 50, cap 200; returns `dm_id` + participants only, no last-message preview |
| Personal activity feed | `feed get [--since --limit --types]` (`feed.rs:9`) | default 20, cap 50; **without `--types` this is not a feed query at all** — the relay's feed path only engages when `feed_types` is present (`bridge.rs:1054-1057`), so a bare `feed get` degrades to "any event p-tagging me" |
| Publish a public note | `social publish --content [--reply-to]` (`social.rs:22`) | kind 1, flat reply model only (`builders.rs:733-737`) |
| Replace your follow list | `social set-contacts --contacts <json>` (`social.rs:43`) | whole-list replace, ≤10 000 contacts (`builders.rs:750`) |
| Fetch any event / a user's notes / a user's contacts | `social event`, `social notes`, `social contacts` (`social.rs:72`, `:83`, `:115`) | raw relay JSON passthrough, signatures included |
| Publish and read NIP-51/NIP-65 lists | `social set-list --kind --tags`, `social list --kind [--d-tag]` (`social.rs:162`, `:184`) | six kinds only; parameterized kinds require a `d` tag |

#### Cross-cutting features

- **Agent-friendly output**: every read path except `canvas get`, `notes
  get --content-only` and the `notes set`/`rm` receipts emits one line of JSON on
  stdout; errors are JSON on stderr with a `retryable` hint
  (`error.rs:113-140`).
- **`--format compact`** reduces events to `{id,content,created_at}` for cheaper
  agent scanning, but only for `messages get`/`thread`/`search`,
  `channels list` and `feed get` (`messages.rs:242-261`, `channels.rs:96-109`,
  `feed.rs:47-60`).
- **Stdin escape hatch** (`-`) exists for `messages send --content`, `messages
  send-diff --diff`, `canvas set --content`, `notes set --content`, and
  `emoji import` (implicitly, when `--file` is omitted) — but with four different
  size policies: none, none, none, 1 MiB, 10 MB.
- **NIP-OA delegation** is transparent: the auth tag is injected into every event
  and sent as a header without any per-command handling (`client.rs:588-596`,
  `:120-127`); template resolution is the only feature that reads the owner out of
  it (`channels.rs:645-651`).

#### Capabilities notably absent

Verified by grepping the ten files for the relevant identifiers with zero
matches: no pin/bookmark of a message (kinds 40004/40005 exist at `kind.rs:425`,
`:427`), no scheduled message (40006), no typing indicator or presence from this
module (20001/20002 live in `users.rs`), no read-state write (30078), no huddle
commands (48100-48106), no channel invite (9009), no NIP-17 encrypted DM (1059),
no watch/tail/subscribe mode (nothing here opens a WebSocket), no `--channel`
scoping for `messages search`, no delete-my-own-note-by-naddr (only `--name`),
and no local template authoring.


## Module: buzz-cli — repo, agent, memory & moderation commands (`crates/buzz-cli/src/commands`)
### Aspect: Features

Eleven modules, 46 operator- or agent-invocable subcommands. Grouped by what they let you do,
with the limits stated alongside.

#### Agent persistent memory (NIP-AE) — `mem`, 6 subcommands

An agent can keep encrypted, owner-scoped notes on the relay and edit them safely under
concurrency.

| Capability | Command | Limits |
|---|---|---|
| List every live memory slug | `mem ls` (`mem.rs:189`) | excludes `core` and tombstones (`mem.rs:243-248`); asks for 5000 events but the relay caps around 1000 (`mem.rs:201` vs `crates/buzz-db/src/event.rs:346-347`) with no truncation signal |
| Read a value verbatim | `mem get <slug>` (`mem.rs:277`) | raw bytes, no trailing newline, so it round-trips with `mem set <slug> -` (`mem.rs:296-303`); absent or tombstoned → exit 1 |
| Capture a pre-edit digest | `mem hash <slug>` (`mem.rs:508`) | 64 hex chars, matches `printf '%s' "$v" \| sha256sum` (`mem.rs:373-381`) |
| Write a whole value | `mem set <slug> <value\|->` (`mem.rs:314`) | ≤ 65 535 bytes (`mem.rs:326-329`); an empty **stdin** read is refused unless `--allow-empty` (`mem.rs:331-338`) — a literal `""` argument is still accepted |
| Edit surgically with a unified diff | `mem patch <slug>` (`mem.rs:538`) | `--base-hash` required unless `--no-base-hash`; hunks must match verbatim **at their declared line numbers** — no fuzz, no sliding (`mem.rs:611-618`); single virtual file only (`mem.rs:600-608`); no-context insertion into a non-empty value refused (`mem.rs:435-443`); `--dry-run` echoes the input diff plus the would-be digest and writes nothing (`mem.rs:699-702`) |
| Tombstone a slug | `mem rm <slug>` (`mem.rs:706`) | `core` cannot be tombstoned (`mem.rs:711-716`) |
| Owner-side recovery | `--agent <pubkey>` on `ls`/`get`/`hash` (`mem.rs:63-74`) | read-only: `set`/`patch`/`rm` have no `--agent` flag at all (`lib.rs:1571-1622`) |

What you cannot do: enumerate slugs without decrypting (the `d` tag is an HMAC, `mem.rs:148`);
patch across multiple slugs in one call; recover from a lost write without re-reading the
head; get a paginated listing.

#### Git collaboration over Nostr (NIP-34) — `repos`, `patches`, `pr`, `issues`, 18 subcommands

| Capability | Command | Limits |
|---|---|---|
| Announce a repository | `repos create --id …` (`repos.rs:202`) | repo id must be 1-64 chars of `[A-Za-z0-9._-]`, no leading dot, no `..` (`validate.rs:40-61`) |
| Look up a repo announcement | `repos get --id [--owner]` (`repos.rs:232`) | without `--owner`, multiple same-named repos from different owners come back together — noted in the comment at `repos.rs:237-238` |
| List a pubkey's repos | `repos list [--owner] [--limit]` (`repos.rs:256`) | defaults to your own pubkey (`repos.rs:262-268`) |
| Inspect branch/tag protection | `repos protect list --id` (`repos.rs:295`) | shows unknown future rules and a `validation_error` string so a broken rule is repairable rather than invisible (`repos.rs:141-171`) |
| Set a protection rule | `repos protect set --id --ref … [--push owner\|admin\|member] [--no-force-push] [--no-delete] [--require-patch]` (`repos.rs:301`) | your own repos only (`repos.rs:22`); at least one constraint required (`repos.rs:521` test); **full replacement** for that exact ref pattern; refuses to write if the stored announcement already has a malformed rule (`repos.rs:118-131`); ≤ 50 rules per repo (`git_perms.rs:19`) |
| Remove a protection rule | `repos protect remove --id --ref` (`repos.rs:327`) | exact pattern match required; absent rule → exit 1 (`repos.rs:334-338`) |
| Send a patch series | `patches send --patch-file <path\|->` (`patches.rs:9`) | reads a `git format-patch` file or stdin; optional `--root`/`--root-revision`, `--reply-to`, `--commit`, `--parent-commit`, `--commit-pgp-sig`, `--committer 'name\|email\|ts\|tz'` |
| Read / list patches | `patches get --event`, `patches list --repo-owner --repo-id [--author] [--limit]` (`patches.rs:73`, `:84`) | raw relay array, unfiltered |
| Set patch status | `patches status --root --status open\|merged\|closed\|draft` (`patches.rs:114`) | `--revision`, `--q`, `--merge-commit`, `--applied-as-commit` are documented "merged only" but not enforced (`patches.rs:180-188`) |
| Open a pull request | `pr open --repo-owner --repo-id --subject --commit --clone …` (`pr.rs:20`) | `--clone` is required (`lib.rs:1327`); `--body` and `--body-file` are mutually exclusive (`pr.rs:9-11`); optional `--channel` binds it to a NIP-29 channel, `--revision-of` to a prior root |
| Update a PR tip | `pr update --pr --pr-author --commit --clone …` (`pr.rs:66`) | requires both PR event id and PR author pubkey |
| Read / list PRs | `pr get --event`, `pr list … [--author] [--label] [--limit]` (`pr.rs:107`, `:118`) | label filter is a `#t` tag match |
| Set PR status | `pr status --pr --status …` (`pr.rs:152`) | narrower than `patches status`: no `--revision`, no `--q`, no `--applied-as-commit` (all hardcoded empty at `pr.rs:199-207`) |
| File / read / list issues | `issues create`, `get`, `list` (`issues.rs:6`, `:36`, `:47`) | `--title` aliases `--subject` (`lib.rs:1451`) |
| Set issue status | `issues status --issue --status open\|resolved\|closed\|draft` (`issues.rs:81`) | narrowest of the three: no merge/revision affordances at all (`issues.rs:136-143`) |

Across all four modules, status events auto-`p`-tag the repo owner when `--repo-owner` is
supplied and let you add reviewers with repeated `--to` (`patches.rs:170-178`,
`pr.rs:186-194`, `issues.rs:127-135`).

What you cannot do: browse a repo's git tree, push, fetch, or clone — those are the relay's
git smart-HTTP surface, not this CLI; edit another owner's protection rules; list statuses
(there is no `patches statuses` / `pr statuses` read command anywhere in the group).

#### Agent identity lifecycle (NIP-IA / NIP-OA) — `agents`, 5 subcommands

| Capability | Command | Limits |
|---|---|---|
| Propose a new agent for owner review | `agents draft-create --channel --display-name --system-prompt` (`agents.rs:16`) | requires `BUZZ_AUTH_TAG` (`agents.rs:161-165`); nothing changes until the human saves it in Desktop (`"saved": false`, `agents.rs:43`); name ≤ 120 chars, prompt ≤ 20 000 chars (`agent_management.rs:11-12`) |
| Propose an edit to a personal agent | `agents draft-update --channel --agent-name [fields…]` (`agents.rs:50`) | at least one field must change (`agent_management.rs:170-179`); `--respond-to owner-only\|anyone` |
| Request an identity be archived | `agents archive <pubkey> [--reason] [--replaced-by]` (`agents.rs:93`) | the relay picks the consent path; an agent signing as itself can only ever satisfy the self path — stated in the help text at `lib.rs:275-279`; `--reason` ≤ 64 bytes and control-char free (`builders.rs:1706-1719`); `--replaced-by` must differ from the target (`builders.rs:1757-1761`) |
| Request an un-archive | `agents unarchive <pubkey> [--reason]` (`agents.rs:129`) | no `--replaced-by` — undefined on unarchive (`builders.rs:1806-1807`) |
| Verify the relay's archive snapshot | `agents archived` (`agents.rs:310`) | any trust failure is a nonzero exit, never a false-empty success (`agents.rs:306-309`); checks NIP-11 `self` authorship, exactly one NIP-70 `-` tag, and the signature before trusting a single pubkey (`agents.rs:320-372`) |

What you cannot do: create or update an agent directly (drafts are advisory by design); list
agents (no read command); archive a third party as an agent (the relay's consent model
blocks it).

#### Community moderation — `moderation`, 8 subcommands

| Capability | Command | Limits |
|---|---|---|
| Read the report queue | `moderation reports [--status] [--limit]` (`moderation.rs:105`) | mod-only relay endpoint; limit defaults to 50 and is clamped server-side to 500 (`crates/buzz-relay/src/api/bridge.rs:2084-2088`) |
| Resolve or dismiss a report | `moderation resolve --report --status --action [--reason]` (`moderation.rs:89`) | status ∈ `resolved\|dismissed`, action ∈ `delete\|kick\|ban\|timeout\|dismiss\|escalate` — validated in the SDK, not here (`builders.rs:1661-1676`); `--reason` is relayed to the reporter (`lib.rs:1685`) |
| Ban a member | `moderation ban --pubkey [--expires-in\|--expires-at] [--reason]` (`moderation.rs:34`) | omitting an expiry means permanent (`moderation.rs:39`); community-wide, no channel scope (`moderation.rs:16-18`) |
| Lift a ban | `moderation unban --pubkey` (`moderation.rs:51`) | — |
| Time out a member (write block) | `moderation timeout --pubkey --expires-in\|--expires-at [--reason]` (`moderation.rs:61`) | an expiry is **mandatory** (`moderation.rs:69-71`) — the one duration rule the CLI itself enforces |
| Clear a timeout early | `moderation untimeout --pubkey` (`moderation.rs:79`) | — |
| See who is restricted | `moderation restricted` (`moderation.rs:119`) | — |
| Read the audit trail | `moderation audit [--limit]` (`moderation.rs:125`) | newest first per help text (`lib.rs:1765`) |

The CLI performs **no local authorization check** — every mutation is a signed 9040-9044
command event the relay validates, authorizes, and executes without storing
(`moderation.rs:3-8`). Because those kinds execute before any dedup, `client.rs` gives them a
non-idempotent retry policy: an ambiguous outcome surfaces as `DeliveryUnknown` rather than
being re-sent (`client.rs:851-861`, `is_moderation_kind` at `client.rs:211-213`).

What you cannot do: read a single report by id; kick or delete content (those actions exist
only as `resolve --action` values the relay executes); scope any of it to one channel;
get `--format compact` output (the flag is accepted and discarded at `moderation.rs:136`).

#### Workflow authoring and operation — `workflows`, 8 subcommands

| Capability | Command | Limits |
|---|---|---|
| List / read definitions | `workflows list --channel`, `workflows get --workflow` (`workflows.rs:13`, `:38`) | `get` filters on `#d` with **no author constraint** and takes the first hit (`workflows.rs:40-47`), so a same-`d` definition from another author can win |
| Create from YAML | `workflows create --channel --yaml <text\|->` (`workflows.rs:98`) | YAML ≤ 64 KiB (`builders.rs:1468`); a client-side UUID is generated but a relay-supplied `workflow_id` wins (`workflows.rs:107-112`) |
| Update in place | `workflows update --channel --workflow --yaml` (`workflows.rs:119`) | `--channel` is required because the `h` tag is re-emitted (`builders.rs:1487-1490`) |
| Delete | `workflows delete --workflow` (`workflows.rs:139`) | kind:5 with an `a` coordinate bound to your own pubkey (`builders.rs:1502-1506`) |
| Trigger a run | `workflows trigger --workflow [--inputs '{…}']` (`workflows.rs:156`) | `--inputs` must be a JSON **object** (`workflows.rs:165-169`) and is **not** size-capped on that branch (`workflows.rs:170-181`) |
| Approve or deny a gated step | `workflows approve --token [--approved false] [--note]` (`workflows.rs:193`) | bare invocation grants (`lib.rs:909`); the relay is sent `hex(SHA256(token))`, never the token (`workflows.rs:205`) |
| Read run history | `workflows runs --workflow [--limit]` (`workflows.rs:66`) | **always returns `[]`** — the relay stores run history in the `workflow_runs` table and emits no 46001-46003 events; documented at `workflows.rs:62-65` and verified: no producer of those kinds exists anywhere in `crates/` |

`buzz-workflow`'s evalexpr conditions and the `/hooks/{id}` webhook trigger surface are not
reachable from this CLI at all — conditions live inside the YAML text this group only
transports, and no command in the group posts to `/hooks/`
(`grep -n 'hooks' ` across the eleven files returns zero matches).

#### Profiles and presence — `users`, 4 subcommands

| Capability | Command | Limits |
|---|---|---|
| Look up profiles | `users get [--pubkey …]` (`users.rs:12`) | ≤ 200 pubkeys (`users.rs:35-37`); no args = your own profile |
| Search by name | `users get --name <q>` (`users.rs:82`) | mutually exclusive with `--pubkey` (`users.rs:16-20`); NIP-50 server search narrowed by a client-side case-insensitive substring match (`users.rs:120-134`); returns `[]` on a relay without NIP-50 |
| Update your profile | `users set-profile [--name] [--avatar] [--about] [--nip05]` (`users.rs:150`) | at least one field (`users.rs:157-161`); read-merge-write preserves untouched fields (`users.rs:165-195`); the kind:0 `name` (username) field is not exposed and is written as `None` (`users.rs:200`) |
| Read presence | `users presence --pubkeys a,b,c` (`users.rs:247`) | **no count cap**, unlike `users get`; blank CSV yields `[]` |
| Set presence | `users set-presence --status online\|away\|offline` (`users.rs:298`) | kind 20001 is ephemeral, so this is the one `users` command that goes over WebSocket (`users.rs:292-301`) |

`users get` is one of only two commands in the group that honor `--format compact`
(`users.rs:62-74`, `users.rs:135-145`), reducing each profile to `{pubkey, display_name}`.

#### Media — `upload`, `media`, 2 subcommands

| Capability | Command | Limits |
|---|---|---|
| Upload a blob | `upload file --file <path>` (`upload.rs:6`) | MIME sniffed from magic bytes and checked against an allowlist (`client.rs:64-71`, `client.rs:1112-1119`); 50 MiB images, 500 MiB video (`client.rs:73-76`, `client.rs:1121-1132`); falls back from `PUT /upload` to `PUT /media/upload` on 404/405 (`client.rs:1180-1195`) |
| Download a blob | `media get <url\|sha256[.ext]> [-o path\|-]` (`upload.rs:19`) | input must resolve to a `/media/` path on the **same** relay origin (`client.rs:270-325`); `sha256`, `sha256.ext`, and `sha256.thumb.jpg` are the only accepted shapes (`client.rs:250-268`); writes to a file or raw to stdout (`upload.rs:21-33`) |

`upload file` is the only command in the group whose output is pretty-printed
(`upload.rs:9-10`). Note `AGENTS.md`'s PR-screenshot rule explicitly forbids using
`buzz upload` for PR images.

#### Local persona packs — `pack`, 2 subcommands

| Capability | Command | Limits |
|---|---|---|
| Validate a pack directory | `pack validate <path>` (`pack.rs:15`) | no relay connection — short-circuited before the client is built (`lib.rs:1735-1741`); warnings exit 0, errors exit 1; diagnostics to stderr |
| Show effective persona config | `pack inspect <path>` (`pack.rs:52`) | plain text only, no JSON mode; prints resolved model/provider, temperature, context limit, subscriptions, triggers, reply flags, MCP-server count, skills, avatar, a 77-char system-prompt preview, and derived `runtime_env_vars` (`pack.rs:66-149`) |

These are the only two commands in the entire CLI that work with no key and no relay —
`lib.rs:1744-1746` requires `BUZZ_PRIVATE_KEY` for everything else.

#### Cross-cutting limits worth knowing

- No command in this group paginates. Every read is one `POST /query`; result sets larger than
  the relay clamp (~1000 rows) are silently truncated. `grep -n 'query_paginated\|query_all'`
  across the eleven files returns zero matches.
- `--format compact` works on `users get` only. `moderation` accepts and drops it
  (`moderation.rs:136`); the other nine modules never receive it.
- `mem` and `pack` deliberately emit non-JSON output; everything else emits single-line JSON
  except `upload file`.
- Exit code 5 (write conflict) is reachable from exactly three commands in this group:
  `mem set`, `mem patch`, `mem rm` (`mem.rs:101`, `mem.rs:595`) and `repos protect
  set`/`remove` (`repos.rs:155`).

