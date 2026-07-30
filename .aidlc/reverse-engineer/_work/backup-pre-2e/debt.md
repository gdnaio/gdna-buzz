<!-- Analyzed: 2026-07-25T01:12:08Z | Scope: full project -->
# Technical Debt

> Status: initialized in Phase 1. Complexity hotspots, coverage gaps, dependency issues,
> and TODO/FIXME inventory are populated per-module during Phase 2 and prioritized in
> Phase 3.

## Summary

Scan-time signals are recorded below, followed by per-module findings.

Batch 2a debt signal counts: `buzz-core` 20 structural findings + 14 coverage gaps,
`buzz-sdk` 13 categories + 13 coverage gaps, `buzz-persona` 63 findings across 7 categories,
`buzz-ws-client` 8 hotspots + 11 untested behaviours. **Zero TODO/FIXME/HACK/XXX markers in
any of the four crates** — deferred work lives in prose comments, so marker-based tooling
sees a clean tree.

Highest-signal items:

| ID | Finding | Location |
|---|---|---|
| DEBT-1 | **Doc drift on the kind registry is large.** `ARCHITECTURE.md` says 81 kinds / 80 `ALL_KINDS` entries; actual is **130 / 127**. Two of the three `ALL_KINDS` exclusions are undocumented, and no code comment explains them. **Still open at `07d0265c`** — recounted: 134 `u32` consts − 4 range bounds = **130**, `ALL_KINDS` body = **127** entries, exclusions unchanged (`KIND_AUTH`, `KIND_NOSTR_IDENTITY_BINDING`, `KIND_PUSH_LEASE`); that commit added three persona helper fns but **no new kind integers**, so it opened no new `ALL_KINDS` gap. | `ARCHITECTURE.md:142`, `:346` vs `crates/buzz-core/src/kind.rs:566-693` |
| DEBT-2 | **`buzz-persona::resolve_skills` is orphaned** — 73 production lines + 4 tests, fully implemented, **never called** by any production path. `ResolvedPersona.skills` carries raw declared paths instead, contradicting its own "bare names" doc comment. | `crates/buzz-persona/src/pack.rs:249`; `crates/buzz-persona/src/resolve.rs:60-61` vs `:249` |
| DEBT-3 | **Desktop maintains a parallel event-builder library.** `desktop/src-tauri/src/events.rs` has 36 `EventBuilder::new` sites reimplementing ~26 SDK builders with a different error type, while declaring `buzz-sdk` as a dependency it never references. Wire-form parity is held by comments and mirrored tests, not shared code. | `desktop/src-tauri/src/events.rs:143-842` vs `crates/buzz-sdk/src/builders.rs` |
| DEBT-4 | **`buzz-acp` reimplements `buzz-ws-client` entirely** — its own `connect_async`, `RelayMessage`, AUTH parse, handshake, and timeout, without depending on the shared crate. It also **diverges functionally**: it re-authenticates on a mid-session challenge, whereas `buzz-ws-client` only records it. **The divergence widened at `07d0265c`**: `buzz-ws-client::RelayMessage` gained a 7th variant `Count` plus a `"COUNT"` parse arm (`crates/buzz-ws-client/src/message.rs:40-46`, `:147-162`), while the `buzz-acp` copy still declares 6 variants (`crates/buzz-acp/src/relay.rs:471-494`) and has no `"COUNT"` arm (zero matches for `"COUNT"` in that file), so a NIP-45 COUNT frame is still an `UnexpectedMessage` on the ACP path. The two copies now differ in *wire coverage*, not just re-auth behaviour. | `crates/buzz-acp/src/relay.rs:2344-2350`, `:3435-3461`, `:3843-3845`, `:471-494` |
| DEBT-5 | **`buzz-sdk/src/builders.rs` is 3,824 lines** holding 51 of 61 public functions. No Rust file-size guard exists to catch this. | `crates/buzz-sdk/src/builders.rs` |
| DEBT-6 | **20 of `buzz-sdk`'s builders pass bare kind literals** instead of `buzz_core::kind::KIND_*`, bypassing the registry `AGENTS.md` designates as the single source of truth — and the tests assert the same literals, so divergence would not be caught. | `crates/buzz-sdk/src/builders.rs:241`, `:288`, `:304`, … |
| DEBT-7 | **Untyped JSON merge layer in `buzz-persona` has already caused a regression.** Field lookups are string literals with no compile-time link to the structs; the regression test documents the `respond_to` vs `triggers` break. | `crates/buzz-persona/src/merge.rs:95-168`, regression note `:400-403` |
| DEBT-8 | **`serde_yaml` 0.9 is unmaintained upstream** and is the YAML backend for all persona parsing. | `crates/buzz-persona/Cargo.toml:12` |
| DEBT-9 | **`buzz-ws-client` ships 3 tests, all compile-time constant assertions** — the NIP-42 handshake, parser, buffering, and timeout logic have no behavioural tests and are exercised only through a live-relay integration suite that `just test-unit` does not run. | `crates/buzz-ws-client/src/connection.rs:300-313` |
| DEBT-10 | **Duplicated validation across crates** — `repo_id` validated in both `buzz-sdk` and `buzz-cli`; pubkey-hex validated by 3+ independent helpers; persona-name rules implemented twice (const vs inline literal). | `crates/buzz-sdk/src/builders.rs:92`, `crates/buzz-cli/src/validate.rs:39`, `crates/buzz-persona/src/resolve.rs:146` vs `crates/buzz-persona/src/validate.rs:168` |
| DEBT-11 | **Constants duplicated rather than shared** — the NIP-44 plaintext cap (65,535) is defined three times and the ciphertext window twice, once as inline literals in the pairing path. | `crates/buzz-core/src/observer.rs:25`, `engram.rs:28`, `pairing/session.rs:595`, `:609` |
| DEBT-12 | **Three undocumented crates confirmed absent from `ARCHITECTURE.md`** — `buzz-persona` has zero mentions in `ARCHITECTURE.md` *and* `CONTRIBUTING.md`; `buzz-ws-client` appears only in `AGENTS.md:63`. **Still open at `07d0265c`** — re-checked after that commit rewrote 57 lines of `CONTRIBUTING.md`: `buzz-persona`, `buzz-ws-client` and `buzz-dev-mcp` are all still zero-match in both `CONTRIBUTING.md` and `ARCHITECTURE.md`; `buzz-ws-client` remains a single `AGENTS.md:63` row. | verified by grep; `AGENTS.md:63` |
| DEBT-13 | **Stale `sprout` metadata** — `buzz-persona` and `buzz-core` both declare `repository = "https://github.com/block/sprout"`; `buzz-persona` also hardcodes its own version/edition/license instead of inheriting from the workspace. | `crates/buzz-persona/Cargo.toml:1-8` |
| DEBT-14 | **PR-relative comments baked into source** — "in this PR", `// Fix #1:`, `// Fix #4:`, and version labels ("V7 spec", "V3 contract") that match nothing in the repo. | `crates/buzz-persona/src/resolve.rs:68`, `:344`; `crates/buzz-persona/src/persona.rs:230`, `:263`, `:95` |
| DEBT-15 | **`buzz-persona` validation reports only the first structural error**, so a pack with five broken personas needs five fix cycles. Its `exit_code()` and `Display` impls are both unused by the only consumer, which re-derives its own. | `crates/buzz-persona/src/validate.rs:148-151`; `crates/buzz-cli/src/commands/pack.rs:26-44` |

### Batch 2b debt (service crates)

Same clean-marker pattern as 2a: **zero TODO/FIXME/HACK markers** in six of the seven
crates (`buzz-workflow` carries the one exception, TODO WF-08). Zero `unsafe` throughout.
The debt here is different in character from 2a — it is dominated by **features that
appear implemented but are not reachable**, and by **documentation that describes controls
incorrectly**. Marker-based tooling finds none of it.

| ID | Finding | Location |
|---|---|---|
| DEBT-16 | **The rate-limiting "contradiction" is resolved and it is a doc bug, not a code gap.** `ARCHITECTURE.md` §9 #2 (`:823`), plus `:390` and `:460`, claim only a test stub exists. `RedisRateLimiter` is real, ungated, and enforced before work. Three of seven `BUZZ_RATE_LIMIT_*` vars (`AGENT_STANDARD_API_CALLS`, `AGENT_ELEVATED_MESSAGES`, `AGENT_PLATFORM_MESSAGES`) are parsed and never read. | `crates/buzz-pubsub/src/rate_limiter.rs:99`; wiring `crates/buzz-relay/src/state.rs:712`; dead vars `crates/buzz-relay/src/config.rs:303-314` |
| DEBT-17 | **`ChannelAccessChecker` has zero implementors repo-wide**, and its own doc comment falsely claims `buzz-db` implements it. `check_read_access`/`check_write_access`/`require_scope` have no production callers — a trait-shaped hole where an authorization seam appears to be. | `crates/buzz-auth` access-checker module |
| DEBT-18 | **`Scope::all_non_admin()` (14 scopes) is never called**; NIP-42 grants `all_known()` (16). The 14-scope figure in the docs describes a code path nothing uses. `derive_pubkey_from_username` also has zero callers. | `crates/buzz-auth` scope module |
| DEBT-19 | **`ARCHITECTURE.md` is wrong on five separate points about the audit hash chain**: 11 actions not 10 (`MediaUploaded` omitted); the hash pre-image contains no `event_id`/`event_kind`/`channel_id`; `AuditError::AuthEventForbidden` does not exist; `GENESIS_HASH` is 32 zero *bytes* not a string; the advisory lock is per-community, not global. | `crates/buzz-audit/src/{action,hash,error}.rs` vs `ARCHITECTURE.md` audit section |
| DEBT-20 | **Nothing verifies the audit chain in production.** `verify_chain` and `get_entries` have no production caller; tail truncation and a missing genesis entry are undetectable by design; the chain is unsigned and unanchored, so DB write access is sufficient to rewrite it wholesale. | `crates/buzz-audit/src/{hash,service}.rs` |
| DEBT-21 | **Three workflow actions are non-functional, not two.** `send_dm` and `set_channel_topic` return `NotImplemented` (as documented), but **`add_reaction` is also dead** — it POSTs to `/api/messages/{id}/reactions`, a route the relay never registers. | `crates/buzz-workflow/src/executor.rs`; router `crates/buzz-relay/src/router.rs:39-125` |
| DEBT-22 | **Workflow approval gates are structurally unreachable.** The token is generated but never persisted (TODO WF-08), runs are marked `Failed`, and **no code anywhere writes `WaitingApproval`** — so the relay's resume path can never execute and `Db::create_approval` has no caller. | `crates/buzz-workflow/src/executor.rs:663` |
| DEBT-23 | **Workflow doc claims inverted on two points:** `execute_from_step` is documented as existing "for future use" but has **3 live callers**, while `execute_run` has **none**. `MAX_DELAY_SECS` is 270, not the documented 300. | `crates/buzz-workflow/src/executor.rs` |
| DEBT-24 | **`schema/schema.sql` claims to be the source of truth but is missing 12 objects** present in the migration end-state, and its `search_tsv` definition matches no migration end-state. Anyone diffing against it gets false positives. | `crates/buzz-db/schema/schema.sql` |
| DEBT-25 | **Migration 0008 makes fresh vs brownfield installs permanently divergent** — a positive FTS allowlist is granted only to empty databases, so two deployments of the same version run different search policies indefinitely. | `migrations/0008_fresh_install_search_allowlist.sql` |
| DEBT-26 | **`buzz-db/src/lib.rs` is 6,106 lines** — the largest single Rust file found so far, exceeding even `buzz-sdk/src/builders.rs` (DEBT-5). Still no Rust file-size guard exists. | `crates/buzz-db/src/lib.rs` |
| DEBT-27 | **`buzz-db` test coverage is effectively zero in the default gate: 121 of 122 async tests are `#[ignore]`d.** Combined with no sqlx offline cache (no compile-time SQL validation), the data layer has neither static nor default-run dynamic verification. | `crates/buzz-db/src/` |
| DEBT-28 | **`ARCHITECTURE.md`'s media size limit is wrong for the crate.** "50 MB" is a relay default for **images only**; real caps are video 500 MB and file 100 MB. The metadata policy is also **reject**, not strip as implied. | `crates/buzz-media/src/validation.rs` |
| DEBT-29 | **No media garbage collection, orphan cleanup, or storage quota exists.** Uploaded blobs accumulate indefinitely with no reclamation path. | `crates/buzz-media/src/` |
| DEBT-30 | **`verify_blossom_get_auth` is implemented but never called from `buzz-media`**; the real gate is a relay flag (`BUZZ_REQUIRE_MEDIA_GET_AUTH`) that defaults to false. Content hashes are verified on write but never on read. | `crates/buzz-media/src/` |
| DEBT-31 | **`ARCHITECTURE.md`'s excluded-kind list for search is stale** — documented as `(1059, 30300, 30622)`; actual is `(1059, 30300, 30350, 30622, 44100, 44101, 44200)`. | `crates/buzz-db/schema/schema.sql:212` |
| DEBT-32 | **`buzz-pubsub` advertises typing indicators that do not exist.** Claimed in `Cargo.toml:8`, an orphaned doc comment at `lib.rs:43` (mis-attached to a `pub use`, which is why `missing_docs` never fired), `AGENTS.md`, and `ARCHITECTURE.md:82`, `:432`, `:434`, `:777`, `:801` — the last including a concrete ZADD/ZREMRANGEBYSCORE/EXPIRE design at `:452-456`. Repo-wide there are **zero** sorted-set calls and **zero** occurrences of `buzz:typing`. Typing is really a kind-20002 ephemeral event fanned out through the generic path. | `crates/buzz-pubsub/Cargo.toml:8`, `src/lib.rs:43`; `ARCHITECTURE.md:452-456` |
| DEBT-33 | **`buzz-pubsub` rate limiter has zero tests** despite being a live security control; relay tests stub Redis out entirely, so the Lua script, the `count <= limit` boundary, and the TTL-repair branch are unverified anywhere. | `crates/buzz-pubsub/src/rate_limiter.rs`; stub `crates/buzz-relay/src/admission.rs:65-90` |
| DEBT-34 | **`retain_topic` can block indefinitely or silently lose a subscription** — the send result is discarded, so a closed channel leaves the refcount claiming "subscribed" while no Redis `SUBSCRIBE` exists; a never-started subscriber task makes the 4097th retain await forever. | `crates/buzz-pubsub/src/lib.rs:203-207` |
| DEBT-35 | **Unreachable knobs and dead controls across 2b:** `PubSubConfig::with_unsubscribe_debounce` has no production caller (500 ms is effectively hardcoded); `topic_refcount` is documented for metrics and unwired; `check_ip_connection` is fully implemented with no caller — and it is exactly the control that would cover the unmetered pre-auth traffic gap. | `crates/buzz-pubsub/src/lib.rs:93`, `:248`, `src/rate_limiter.rs:112-120` |
| DEBT-36 | **Unchecked arithmetic in the MP4 atom parser** — `offset += atom_size` at one site while a sibling site 400 lines away correctly uses `checked_add`. | `crates/buzz-media/src/validation.rs:481` vs `:891-893` |
| DEBT-37 | **Triplicated reconnect machinery in `buzz-pubsub`** — backoff constants declared three times, three near-identical `run_*_subscriber` loops, and two nearly identical pattern-subscribe bodies (~50 duplicated lines). A clean-disconnect backoff reset also permits an indefinite 1 s reconnect loop with no escalation. | `crates/buzz-pubsub/src/{subscriber,cache_invalidation,conn_control}.rs` |
| DEBT-38 | **Unused dependency** — `buzz-pubsub` declares `chrono` with no `chrono::` path in any source file. | `crates/buzz-pubsub/Cargo.toml:18` |

### Batch 2c debt (relay, mesh, conformance)

The dominant pattern from 2b intensifies here: **subsystems that are complete in code but not
reachable in a deployed configuration**, plus documentation that describes a different system
than the one that exists. Marker-based tooling finds almost none of it.

| ID | Finding | Location |
|---|---|---|
| DEBT-39 | **`crates/buzz-relay/src/api/events.rs` is 100% dead** — no route references it. `webhook_secret::strip_secret` likewise has zero callers. | `crates/buzz-relay/src/api/events.rs` |
| DEBT-40 | **`ARCHITECTURE.md` is wrong on four more points**: the rate-limiting claims at `:823`, `:390`, and `:459` are false (resolved in 2b as DEBT-16); `:824` documents `/api/presence`, which does not exist; `:623` says 50 MB where the router configures 500 MB. | `ARCHITECTURE.md:823`, `:390`, `:459`, `:824`, `:623` vs `crates/buzz-relay/src/router.rs:33-36` |
| DEBT-41 | **14 routed endpoints fall outside the "narrow HTTP surface"** `AGENTS.md` describes, so the repo guide understates the attack surface a contributor must reason about. | `crates/buzz-relay/src/router.rs` vs `AGENTS.md` |
| DEBT-42 | **`workflow_sink` implements 1 of the 7 workflow actions** the engine defines — compounding DEBT-21/22 from 2b. | `crates/buzz-relay/src/` workflow sink |
| DEBT-43 | **`buzz-relay-mesh` is absent from `ARCHITECTURE.md`, `AGENTS.md`, and `just test-unit`** — all 32 of its tests compile but never run. `MeshRuntime::shutdown()` is never called, and there is no peer eviction path. | `crates/buzz-relay-mesh/`; `justfile` unit lists |
| DEBT-44 | **The 5 `crates/buzz-relay/examples/mesh_*.rs` do not exercise `buzz-relay-mesh`** — they are MeshLLM (`mesh_llm_sdk`) smoke tests pulled in through git dev-dependencies, so the name collision misleads anyone looking for mesh coverage. | `crates/buzz-relay/Cargo.toml:84-85` |
| DEBT-45 | **`iroh` version drift** — manifest declares `1.0.0-rc.0`; the lockfile resolves 1.0.2. | `Cargo.lock:3902-3905` |
| DEBT-46 | **The conformance gate is unassembled end-to-end.** `check_trace` has zero production callers; `JsonlTracer` — the only persisting tracer — is never instantiated; production binds `NoopTracer`, the sole assignment to that field. The crate's own docs call the replay "the **next** ratchet", but the read-seam patch it was waiting on has landed. | `crates/buzz-conformance/src/checker.rs:74`; `crates/buzz-relay/src/conformance/tracers.rs:30-72`; `state.rs:798`; `LIMITS.md:120-121` vs landed `handlers/req.rs:334-361`, `:649-677` |
| DEBT-47 | **An extra Postgres round-trip is paid per REQ filter and per search page for a discarded result.** The gate is `trace_state.is_some()`, not the tracer type, and `trace_state` is `Some` whenever the pubkey parses. The `Tracer` trait exposes no `is_recording()` to short-circuit it. | `handlers/req.rs:349`, `:661`, gates `:338`, `:649`, `:116-118`; trait `crates/buzz-conformance/src/lib.rs:314-318` |
| DEBT-48 | **Seven `Ok`-returning ingest paths emit no trace step**, so a legitimate accepted write is indistinguishable from a forgotten emit site. The pattern for fixing it exists — product feedback and push lease were instrumented for exactly this reason — and was applied to 2 of 9 early-return success paths. | `handlers/ingest.rs:1560-1562`, `:1587-1596`, `:1605-1614`, `:1834-1842`, `:1846-1928`, `:2341-2347`, `:2452-2458`; instrumented counterexamples `:1567-1580`, `:2215-2222` |
| DEBT-49 | **`WriteDuplicate` is unreachable on the main write path** — an early return on `!was_inserted` makes the trailing match arm dead, while the comment above it reasons about the case as if live. | dead arm `handlers/ingest.rs:2501-2505`, early return `:2452-2458`, comment `:2484-2492` |
| DEBT-50 | **Spec coverage is a small fraction of what the spec asserts** — 8 of 23 `Next` actions and ~1.5 of 13 `Safety` invariants. Eleven invariants are structurally uncheckable because `AbstractState` carries three fields and none of the spec's state variables. `ReadHostFeedRows` has no emitter at all. | `docs/spec/MultiTenantRelay.tla:933-956`, `:1129-1142`; `crates/buzz-conformance/src/lib.rs:150-175` |
| DEBT-51 | **The write-path conformance rule is mis-keyed against Buzz's own tag convention** — `h` carries a channel UUID, the emitter parses it as a community UUID, and the committed fixtures encode the community reading, so the fixtures would not catch it. | `crates/buzz-relay/src/conformance/mod.rs:101-119`; fixture `crates/buzz-conformance/tests/replay_fixtures.rs:78-86` |
| DEBT-52 | **Twelve stale claims inside the conformance crate's own docs and comments**, including "the read-seam half of the gate is not yet armed" (armed), "held back as additive patch" (landed), a test count of 16 (actual 22 in-crate plus 10 relay-side), documented wire keys that do not match serde output, and three drifted spec line numbers. | `LIMITS.md:56-57`, `:112`, `:88-118`; `TRACE_SCHEMA.md:37-46`, `:57`, `:69`, `:137`; `conformance/mod.rs:37-38`; `src/lib.rs:193`, `:321-322` |
| DEBT-53 | **Zero repo-level documentation of the formal-methods lane.** `buzz-conformance`, `conformance`, `MultiTenantRelay.tla`, `TLA`, and `formal` appear nowhere in `AGENTS.md`, `ARCHITECTURE.md`, or `CONTRIBUTING.md`. The rule "new endpoints touching the tenant boundary MUST arm a guard" is stated only inside the crate's own `LIMITS.md:36-41`, which nothing links to — so it is invisible to any contributor following the house guide. **Still open at `07d0265c`** — the `CONTRIBUTING.md` rewrite in that range added a "Before You Open a PR" section but no formal-methods guidance; all five search terms remain zero-match in the three house documents. | verified by grep; `crates/buzz-conformance/LIMITS.md:36-41` |
| DEBT-54 | **Relay-side conformance emitter tests are outside the unit gate.** `just test-unit` does not include `buzz-relay`, so the 9 tests proving `EmitGuard` fires, that `member → Allow` / `!member → Deny`, and that a missing projection lookup becomes `ImplBug` never run pre-PR — despite `LIMITS.md:105-110` naming that command mandatory. No GitHub workflow mentions conformance. | `justfile:278-293`; `scripts/run-tests.sh:81-102`; tests `crates/buzz-relay/src/conformance/mod.rs:466-726`, `handlers/ingest.rs:2556-2591` |
| DEBT-55 | **Six unreferenced public items in the conformance surface**, none marked `#[allow(dead_code)]` or TODO'd: `Verdict_`, `action_channel` (whose doc claims the checker uses it), `TraceAction::is_critical` (returns `true` unconditionally; the mechanism it documents is unimplemented), `Scenario::require`, `conformance::step`, and the crate's own `NoopTracer` (shadowed by the relay's identically-named copy). | `src/transitions.rs:53-56`, `:318-330`; `src/lib.rs:283-285`, `:323-327`; `src/checker.rs:54-57`; `crates/buzz-relay/src/conformance/mod.rs:121-123` |
| DEBT-56 | **Coverage-set membership is stringly-typed and opt-in** — required action names are `String`s compared against `kind()` output with no validation, so a typo becomes a permanently-missing requirement; 18 of 22 scenario constructions declare no requirements at all. | `crates/buzz-conformance/src/checker.rs:37`, `:113-118`, `:45-50` |
| DEBT-57 | **`M1`…`M8` mutation IDs are used throughout the conformance code with no legend anywhere in the repo**, and several comments cite review threads and named reviewers that are not in the repo. | `src/lib.rs:127`, `:190`, `:238`; `src/transitions.rs:218-221`; `conformance/mod.rs:18-19`, `:37-38`; `handlers/ingest.rs:1779` |

### Batch 2d debt (agent surface: buzz-acp, buzz-agent, buzz-dev-mcp, buzz-cli)

Two patterns dominate 2d and neither is visible to tooling. First, **incomplete work is marked
with `#[allow(dead_code)]` and prose rather than a marker** — `TODO`/`FIXME`/`HACK`/`XXX` return
**zero matches** across `buzz-acp`, `buzz-dev-mcp`, `buzz-agent`'s core and LLM groups, and all
21 `buzz-cli` command modules, so no grep-based audit surfaces any of it. Second, **the agent
surface's documentation describes a different system than the one that exists**: the crate READMEs
are wrong on defaults, method counts, protocol versions and architecture, and `ARCHITECTURE.md`'s
LOC table for `buzz-acp` is wrong on all seven rows.

Also structural: two large crates reimplement code they could depend on, and several tests assert
against a *copy* of the production rule, so those rules can drift green.

| ID | Finding | Location |
|---|---|---|
| DEBT-58 | **`ARCHITECTURE.md:658-667`'s LOC table for `buzz-acp` is wrong on all seven rows**, and wrong in a way that misdirects a reader about where the code lives. `main.rs` is listed at 2,457 LOC doing "Event loop, pool orchestration" — it is **3 lines**; the loop is in `lib.rs` (6,570 lines), which the table omits entirely. `relay.rs` 3,143→6,233, `queue.rs` 2,565→4,759, `pool.rs` 2,253→5,620, `config.rs` 1,903→2,709, `acp.rs` 1,785→3,717. `usage.rs`, `observer.rs`, `setup_mode.rs`, `engram_fetch.rs` and `pool_lifecycle.rs` are absent. | `ARCHITECTURE.md:658-667` vs `wc -l crates/buzz-acp/src/*.rs` |
| DEBT-59 | **`buzz-acp` fully reimplements `buzz-ws-client` with no dependency on it** — including a byte-identical debug string. `WsStream` (`relay.rs:525` vs `connection.rs:14`), the debug literal (`relay.rs:3838` vs `:57`), `RelayMessage` (`relay.rs:471-495` vs `message.rs:8-41`), `OkResponse` (`relay.rs:3917-3921` vs `message.rs:44-52`), the parser (`relay.rs:3533-3620` vs `message.rs:55-144`), the AUTH wait (`relay.rs:3865-3913` vs `connection.rs:157-215`), and the 20 s timeouts (`relay.rs:64` vs `connection.rs:17`, `:20`). The two copies have now **diverged in wire coverage**, not just in re-auth behaviour: `buzz-ws-client` handles 7 inbound messages including `COUNT`, the acp copy declares 6 variants and has zero `"COUNT"` matches. `buzz-cli`, by contrast, depends on the shared crate properly (`client.rs:1084`). | `crates/buzz-acp/src/relay.rs:471-495`, `:525`, `:3533-3620`, `:3838`, `:3865-3913` vs `crates/buzz-ws-client/src/message.rs`, `connection.rs` |
| DEBT-60 | **The `buzz-acp` pool's actual pooling logic has zero tests.** Eleven functions — `try_claim`, `return_agent`, `live_count`, `slot_alive`, `any_idle`, `has_session_for`, `send_steer`, `switch_idle_agent_model`, `invalidate_channel_sessions`, `from_slots`, `IdleSwitchResult` — have **0 matches** in the file's 1,970-line test region, and `run_prompt_task` (948 lines, `pool.rs:1265-2212`, 20 exit points) is untested. `tests/pool_lifecycle_state.rs:1-4` is a 4-line `#[path]` shim that re-runs the same 9 in-source tests in a second binary, adding zero coverage. Also untested: `EventQueue::remove_event` (`queue.rs:738-751`, the `SteerAck::Success` handler), `is_channel_in_flight` (`:645`), `pending_channels` (`:594`), `native_steer_framing()` (`:1623`), `RestClient` (`relay.rs:261-424`), `discover_channels`, the `do_connect` happy path, `wait_for_reconnect`, `process_handshake_buffer`, the mid-session AUTH arm; `observer.rs` and `shim.rs` have **zero** tests each — so the nsec-removal/zeroize invariant in `shim.rs` is entirely unverified. | `crates/buzz-acp/src/pool.rs:1265-2212`; `queue.rs:738-751`; `relay.rs:261-424`; `tests/pool_lifecycle_state.rs:1-4` |
| DEBT-61 | **Ordering contracts enforced only by comment, with silent failure modes.** `requeue` MUST precede `mark_complete`: `mark_complete` reads `retry_after` (`queue.rs:397-409`) and only `requeue` writes it (`:496`), so inverting the two silently resets every retry to attempt 1. There is no `debug_assert!`; the contract lives at `queue.rs:426-427` and `lib.rs:3061-3065`. Likewise `cancelled_batches` is unbounded (`queue.rs:543-546`) with no `MAX_BATCH_EVENTS` analogue and is untouched by `compact_expired_state` (`:809-817`), so it grows the merged prompt without limit — same for `withheld_native_steer` and `in_flight_batch_sizes`. The cancelled-batch fallback also bypasses the retry throttle and is non-deterministic: `queue.rs:308-312` and `:586-590` check only `in_flight_channels`, never `retry_after`, and selection is `keys().find()` over a `HashMap` with no tie-break (`:299`). Untested. | `crates/buzz-acp/src/queue.rs:397-409`, `:426-427`, `:496`, `:543-546`, `:308-312`, `:586-590`, `:809-817` |
| DEBT-62 | **`--subscribe-mode all` without `--kinds` produces a REQ with no `kinds`**, which the relay p-gate rejects with 403 (`AGENTS.md § Common Gotchas #2`). Worse, `filter.rs:383` treats the empty set as a local wildcard, so the agent matches everything and receives nothing — with no warning. `test_all_mode_wildcard` (`config.rs:1630`) asserts the wildcard as correct behaviour. | `crates/buzz-acp/src/config.rs:1180`, `:1195-1196`, `:1272`, `:1286-1287`, `:1630`; emit gate `relay.rs:3172-3175`; `crates/buzz-core/src/filter.rs:383` |
| DEBT-63 | **`ARCHITECTURE.md` contradicts itself on typing indicators, and the design it documents exists nowhere.** Typing indicators are kind-20002 ephemeral events published from the 3 s tick (`lib.rs:2332-2341` → `relay.rs:842-870`); `KIND_TYPING_INDICATOR` (`buzz-core/src/kind.rs:331`) appears nowhere in `buzz-relay/src` or `buzz-pubsub/src`. The ZADD design at `ARCHITECTURE.md:452-458` is not implemented anywhere, while `ARCHITECTURE.md:824` correctly describes 20002 — so the document disagrees with itself. | `ARCHITECTURE.md:452-458` vs `:824`; `crates/buzz-acp/src/lib.rs:2332-2341`, `relay.rs:842-870` |
| DEBT-64 | **Nine crate-README claims in `buzz-agent` are materially wrong.** "The full server is hand-rolled in `main.rs`" (`README.md:124`) — `main.rs` is 6 lines, the server is `lib.rs:187-825`. Three request methods documented; **six** are handled, including `session/set_model` (`lib.rs:234`) and `_goose/unstable/session/steer` (`:245`), plus ten outbound update variants against three documented. The handshake transcript shows `protocolVersion: 1` (`README.md:68-70`) against `PROTOCOL_VERSION = 2` (`config.rs:3`). `BUZZ_AGENT_MAX_HISTORY_BYTES` is documented as 1 MiB (`README.md:155`, repeated `:236`) against `16 * 1024 * 1024` in code (`config.rs:821`, was `:814` before `16d4ec33` shifted it by 7) — a **16× sizing error**. "No flags, no config files" (`:128`) — `buzz-agent auth <provider>` exists (`lib.rs:111-116`). "There is no trait, no `Box<dyn>`, no async-trait" (`:181`) — `Arc<dyn TokenSource>` with `#[async_trait]` at `auth.rs:43-44` (the field itself, `llm.rs:104`, shifted from the pre-sync `:50` by the mesh-routing additions), and a second full provider `match` in `Llm::summarize`. | `crates/buzz-agent/README.md:68-70`, `:124`, `:128`, `:155`, `:181`, `:236` |
| DEBT-65 | **Cross-crate contract drift on steer.** `crates/buzz-acp/src/acp.rs:185-186` documents that buzz-agent emits `activeRunId: null` at end of turn, and the client relies on it to clear its cached run id. buzz-agent emits `activeRunId` exactly once, at prompt start (`lib.rs:661-670`, the sole emission site), clearing the field only internally (`:707`). The client's cache therefore goes stale, and the only thing preventing a misdirected steer is buzz-agent's own mismatch rejection (`lib.rs:588-598`). Separately, `docs/MCP_DRIVEN_HOOKS.md:14-17` states hook output is JSON-encoded for prompt-injection safety — true for `_Stop` (`agent.rs:641-651`), false for `_PostCompact`, which is concatenated as plain `[{server}]\n{text}` (`handoff.rs:247-253`) into a synthetic **user** message (`:85-92`). | `crates/buzz-acp/src/acp.rs:185-186` vs `crates/buzz-agent/src/lib.rs:661-670`; `docs/MCP_DRIVEN_HOOKS.md:14-17` vs `handoff.rs:247-253` |
| DEBT-66 | **Two model-family classifiers disagree inside the same request.** `config.rs` deliberately built boundary-safe matchers — `gpt5_token_matches` (`:239-254`), `gpt5_base_matches` (`:267-311`) — with eight tests (`:2278-2399`, unaffected by the sync) so `gpt-5.1` cannot match `gpt-5.10`. `databricks_v2_route_for_model` (`llm.rs:970-980`, moved from the pre-sync `:689-694` by the mesh-routing insertions earlier in the file) then uses naked `contains("gpt-5")` / `contains("claude")` for routing, so `databricks-gpt-5-10` routes to the OpenAI Responses gateway while `openai_efforts_for_model` classifies it as unknown (asserted `config.rs:2346`, was `:2338`). `contains("claude")` is likewise unanchored. No test pairs the two. | `crates/buzz-agent/src/llm.rs:970-980` vs `config.rs:239-254`, `:267-311`, `:2346` |
| DEBT-67 | **The effort-table drift guard validates a test-only re-implementation, not production.** `effort_table_fixture_matches_rust_implementation` (`config.rs:2674`) compares `effortTable.fixture.json` against `valid_effort_values_for_provider_model` (`config.rs:2567-2660`), a 94-line `#[cfg(test)]` function (unaffected by the `16d4ec33`/`8eb6e3eb` sync — untouched by either commit). Three rules live only there: prefix stripping copied from `strip_catalog_prefix` (`:2578-2590` vs `:89-97`), a default-effort-per-family table that exists nowhere in production (`:2607-2615`), and a DBv2 gpt-5 predicate (`:2631-2644`) that omits the `gpt-5-6` variants production checks at `:371-372`. That omission is undetectable because both branches at `:2645-2651` return the same value, making the 14-line `is_gpt5` computation dead. Also a compile-time coupling: `include_str!` from a Rust crate into `desktop/src/features/agents/ui/effortTable.fixture.json` (`:2676`), so moving that desktop file breaks `cargo test -p buzz-agent`. Related: `anthropic_efforts_for_model`'s doc comment claims to be "the single production source of truth" and that `anthropic_thinking_config` derives from it (`:407-413`) — `anthropic_thinking_config` never calls it (`:124-178`), and the function's only caller in the repo is a test (`:2596`). | `crates/buzz-agent/src/config.rs:2567-2660`, `:2674-2708`, `:407-413` |
| DEBT-68 | **`Config::from_env` has zero tests, no doc comment, and its 13 numeric invariants are unexercised.** 36 env vars and ~10 error paths (`config.rs:742-837`); the `BUZZ_AGENT_SYSTEM_PROMPT`/`_FILE` mutual exclusion (`:792-794`) and the `BUZZ_AGENT_NO_HINTS` inversion (`:832`) are untested — and neither is the new `BUZZ_AGENT_PREFER_MESH_FOR_AUTO` `!= 0` inversion (`:807`, added `16d4ec33`). The only `validate()` tests vary `thinking_effort` alone, and `make_config_for_validation` has to *undo* `for_discovery`'s invalid values (`:1857-1864`) to reach that check — direct evidence the numeric branches at `:884-939` are unreached. `Config::for_discovery` (`:845-874`) itself produces a struct that would **fail** `validate` (`max_output_tokens: 1` `:856`, `mcp_max_restart_attempts: 0` `:860`, `mcp_restart_base_ms: 0` `:861`) and never calls it; all **27** fields are `pub` (was 26 before `16d4ec33` added `prefer_mesh_for_auto`, `:734`) while `validate` is private (`:878`), so the invariants are `from_env` post-conditions rather than type invariants. | `crates/buzz-agent/src/config.rs:742-837`, `:845-874`, `:884-939`, `:1857-1864` |
| DEBT-69 | **No wall-clock request timeout on LLM inference calls, and the tests run against a stricter client than production.** `Llm::new` sets only `connect_timeout(10s)` and `read_timeout` (`llm.rs:109-113`) — no `.timeout(...)` on the client builder, so a provider trickling one byte every 239 s holds an inference call open indefinitely; `STALL_NOTICE_THRESHOLD` (`llm.rs:27`) only logs (`:1324-1330`). **Status: partially resolved** in `16d4ec33` — that commit added the crate's first *production* `.timeout(` call, a 2 s bound on the new mesh-catalog-probe `GET /models` request (`llm.rs:488`, constant `MESH_AUTO_CATALOG_TIMEOUT` at `:31`), but it covers only that probe; every POST that actually runs inference (Anthropic, OpenAI, Databricks, DatabricksV2) still has no wall-clock cap. The rest of the original finding still holds: the other four `.timeout(` matches in the file are all in the test module (`llm.rs:3171`, `:3235`, `:3286`, `:3637`), because `llm_with` (`:3634-3641`) builds `Llm` by struct literal with `.timeout(5s)`, bypassing `Llm::new` entirely — so the production timeout wiring for inference is still never exercised. `buzz-agent`'s OAuth and catalog clients have no timeout at all (`auth.rs:153`, `catalog.rs:80`). | `crates/buzz-agent/src/llm.rs:109-113`, `:27`, `:1324-1330`, `:488`, `:31`, `:3634-3641`; `auth.rs:153`; `catalog.rs:80` |
| DEBT-70 | **`kill_server`'s documented restart budget does not exist.** The comment at `mcp.rs:414-419` claims a server that "starts fine, deadlocks on every call" eventually exhausts its budget; `attempts` is hardcoded to 1 on every kill (`:432`) and a successful restart stores `Healthy` with no counter (`:672-677`), so kill→restart cycles are unbounded. Only consecutive *spawn* failures are budgeted (`:685-699`), and the exhausted-state 86,400 s retry (`:687-690`) is unreachable because `check_restart_state` errors first (`:135-140`). Other vestigial items in the same crate: `MAX_NAME_LEN = 128` (`mcp.rs:20`) is unreachable behind the 64-byte qname check (`:242-248`); `PkceOAuthConfig::cache_dir_override` (`auth.rs:102-107`) is test-only on a public struct; `configure_no_window_compiles_and_applies_flag_on_windows` (`mcp.rs:1128-1140`) is a test with **no assertion**. | `crates/buzz-agent/src/mcp.rs:414-419`, `:432`, `:672-699`, `:135-140` |
| DEBT-71 | **`rg.rs` is broken on Windows and silently degrades everywhere.** PATH is split on a hardcoded `':'` (`:34`, `:50`) and the probe name is a bare `"rg"` with no `.exe` (`:39`, `:54`), so the fallback is permanent there. The fallback does **literal substring matching, not regex** (`:289-296`) with a hand-rolled `*`/`?` glob (`:401-434`), while `lib.rs:42` advertises real `rg` to the model; it caps output with no in-band marker (`:152-166`), unlike every other tool in the crate. | `crates/buzz-dev-mcp/src/rg.rs:34`, `:39`, `:50`, `:54`, `:152-166`, `:289-296`, `:401-434` vs `lib.rs:42` |
| DEBT-72 | **`buzz-dev-mcp` is undocumented everywhere it should appear.** Zero matches in `ARCHITECTURE.md` and `CONTRIBUTING.md`, no crate README, no `//!` module doc, and a missing doc comment on its only public item `pub fn run()` (`lib.rs:138`). `BUZZ_SHELL`, `GIT_BASH` and `NOSTR_PRIVATE_KEY` are absent from `.env.example`. `shell.rs` (1,503 lines) and `view_image.rs` (1,136) exceed the 1,000-line ceiling `AGENTS.md` documents. Of 97 tests, 23 are Windows-gated → 74 run on Unix CI; `lib.rs`, `shim.rs` and `tree.rs` have zero tests. `Shim::install` symlinks the running exe to `rg`, `tree`, `buzz`, `git-credential-nostr` and `git-sign-nostr` in a `0700` tempdir prepended to the child `PATH` (`shim.rs:31-49`, dispatch `lib.rs:144-171`) — which explains the `buzz-cli` dependency; no relay operation is exposed as an MCP tool (`lib.rs:40-125`), so the `AGENTS.md:147` boundary does hold. | `crates/buzz-dev-mcp/src/lib.rs:40-125`, `:138`; `shim.rs:31-49`; verified by grep against `ARCHITECTURE.md`, `CONTRIBUTING.md`, `.env.example` |
| DEBT-73 | **`AGENTS.md` § Agent CLI documents a flag that does not exist and an output contract that describes roughly half the CLI.** Gotcha #3 instructs "`messages search` must include `--kinds` … Pass at least `--kinds 9,45001,45003`", but `MessagesCmd::Search` (`lib.rs:472-489`) declares no such flag — verified `--kinds 9` → exit 1 `unexpected argument`; the kinds are hardcoded at `commands/messages.rs:361`. The same stale block is duplicated in `CLAUDE.md:172`. The example at `AGENTS.md:181` puts `--format compact` *after* the subcommand, which cannot parse (verified exit 1) and contradicts `AGENTS.md:192-193`. On the output contract: at least 12 read paths print the relay body verbatim without sig-stripping (`commands/social.rs:78`, `:111`, `:122`, `:205`; `repos.rs:251`, `:280`; `patches.rs:79`, `:109`; `pr.rs:113`, `:145`; `issues.rs:42`, `:75`), 7 reads are not arrays, 12 writes deviate from `{event_id, accepted, message}` — `mem set/patch/rm` print **nothing** on stdout (`mem.rs:370`, `:696`, `:733`) — and 2 of 3 creates omit the entity ID. | `AGENTS.md:181`, `:188-193`; `CLAUDE.md:172` vs `crates/buzz-cli/src/lib.rs:472-489` and the command modules |
| DEBT-74 | **`--format` is parsed for every invocation and silently ignored by 16 of 21 command groups.** Declared on the top-level `Cli` without `global = true` (`lib.rs:92-93`) and forwarded to only five dispatchers (`lib.rs:1772`, `:1773`, `:1778`, `:1780`, `:1790`); within those, only 5 commands honour it. `moderation::dispatch` is the sharpest case — it *accepts* the argument and binds it as `_format` (`moderation.rs:133-137`). `mem ls` has a redundant private `--json` instead (`lib.rs:1551-1552`). No warning, no test (zero `OutputFormat` matches in any command test module). For an agent-first CLI whose documented calling convention centres on `--format compact`, a silently-ignored output mode is a correctness trap. | `crates/buzz-cli/src/lib.rs:92-93`, `:1772-1790`; `commands/moderation.rs:133-137` |
| DEBT-75 | **The CLI's drift guards skip the three largest command groups.** `subcommand_counts_are_stable` (`lib.rs:1996-2033`) enumerates 18 of 21 groups, omitting `mem`, `moderation` and `notes`; `subcommand_names_are_stable` (`lib.rs:1855-1993`) covers `moderation` but not `mem` or `notes`. So adding or renaming a `mem` or `notes` verb breaks no test. The counts list is a hand-maintained literal with no cross-check against `command_inventory_is_stable`'s 21-name list, so any new group inherits the same blind spot silently. | `crates/buzz-cli/src/lib.rs:1855-1993`, `:1996-2033` |
| DEBT-76 | **One rule, two implementations, two output contracts: NIP-33 LWW conflict detection.** `mem`'s `submit_engram` hand-rolls the `duplicate:` ladder (`mem.rs:93-111`) and never calls `client::normalize_write_response` (`client.rs:1420`), which is why `mem set/patch/rm` emit nothing on stdout. `repos::validate_write_response` (`repos.rs:173-193`) re-derives the identical rule and *does* normalize (`:192`). Only the `repos` side is tested (`repos.rs:619`, `:629`). Broader duplication in the same crate: `parse_events` exists three times (`repos.rs:11-14` and `notes.rs:156-159` byte-identical strict; `mem.rs:114-136` deliberately lenient); the NIP-34 `#a` coordinate is hand-formatted as `format!("30617:{owner}:{id}")` in three files (`pr.rs:129`, `patches.rs:94`, `issues.rs:58`) despite `GitRepoCoord::to_a_tag_value` (`builders.rs:976`); the `(repo_owner, repo_id)` pairing match and the recipient-dedup loop are each triplicated verbatim; `channels.rs` contains **three** independent kind-39000 parsers whose outputs disagree (`:16-24`, `:166-211`, `:68-89`), so `channels get` returns strictly less than `channels search` for the same event; and `resolve_author` is implemented twice with different accepted inputs (`messages.rs:394-440` vs `notes.rs:204-252`). | `crates/buzz-cli/src/commands/mem.rs:93-111`; `repos.rs:173-193`; `channels.rs:16-24`, `:68-89`, `:166-211` |
| DEBT-77 | **`config.rs` in `buzz-acp` re-implements four validation rules as test-local copies** (`:1959`, `:2001`, `:2214`, `:2490`) rather than calling `from_args`, so those rules can drift green. `buzz-cli` has the same pattern in three places: 3 of the 4 `BUZZ_ACP_ALLOWED_CHANNEL_ADD_POLICIES` tests assert against a verbatim copy of the production gate (`channels.rs:1296-1307` duplicating `:1022-1033`) — deleting the gate leaves them green, which the file's own comment at `:1345-1350` admits; `multi_file_header_count` (`mem.rs:1037-1046`) re-implements the multi-file predicate instead of calling `mem.rs:618`; and `error.rs`'s two JSON-shape tests rebuild `print_error`'s object inline (`error.rs:197-221`) so the production serializer and its category strings are never executed. `exit_code` (`error.rs:92-107`) — the contract `AGENTS.md` publishes — has **no unit test** at all. | `crates/buzz-acp/src/config.rs:1959`, `:2001`, `:2214`, `:2490`; `crates/buzz-cli/src/commands/channels.rs:1296-1307`; `error.rs:92-107`, `:197-221` |
| DEBT-78 | **Parsed-but-never-read and undocumented configuration across the agent surface.** `BUZZ_API_TOKEN` is written by `propagate_legacy_env_vars` (`config.rs:718`) and has **zero read sites**, while `README.md:107` still calls it required and `crates/buzz-cli/Cargo.toml:19` still claims it is "auto-wired". Nineteen `buzz-acp` env vars are missing from `.env.example`, including `BUZZ_ACP_RESPOND_TO`, `BUZZ_ACP_RESPOND_TO_ALLOWLIST`, `BUZZ_ACP_PERMISSION_MODE`, `BUZZ_AUTH_TAG`, `BUZZ_ACP_LAZY_POOL`, `BUZZ_ACP_SETUP_PAYLOAD`, `BUZZ_ACP_MAX_TURN_DURATION` and `BUZZ_ACP_MULTIPLE_EVENT_HANDLING`; `.env.example:152` documents the hidden deprecated `BUZZ_ACP_TURN_TIMEOUT=320` instead of `BUZZ_ACP_IDLE_TIMEOUT` (default 900, `config.rs:27`; README says 620). Eleven of `buzz-agent`'s 36 env vars are undocumented anywhere (was 10 of 32 before `16d4ec33` added the equally-undocumented `BUZZ_AGENT_PREFER_MESH_FOR_AUTO`, `crates/buzz-agent/src/config.rs:807` — see CFG-2d-6), and `BUZZ_TIMEOUT_SECS`/`BUZZ_CONNECT_TIMEOUT_SECS` exist only in a Rust doc comment (`client.rs:535-540`). `BUZZ_ACP_EVENT_BUFFER` (`relay.rs:35-42`, default 256, commented out at `.env.example:221`) silently sizes **two** channels (`relay.rs:610-612`) though the docs describe one. `BUZZ_ACP_ALLOWED_CHANNEL_ADD_POLICIES` (`channels.rs:1021`) is wholly undocumented and its absence means permit-all. | `crates/buzz-acp/src/config.rs:718`, `:27`; `relay.rs:35-42`, `:610-612`; `.env.example:152`, `:221`; `crates/buzz-cli/src/client.rs:535-540`; `crates/buzz-agent/src/config.rs:807` |
| DEBT-79 | **`buzz-persona` is a dead dependency of `buzz-acp`** — declared at `Cargo.toml:22`, the only internal dep declared by `path` rather than `workspace = true`, with zero references anywhere under `crates/buzz-acp/src`. The `persona_env_vars` doc comment (`config.rs:534`) is stale; the only writer is `codex_network_env` (`config.rs:951-957`). Related dead or misplaced items: `pub use usage::TurnUsage` (`lib.rs:15`) is `buzz-acp`'s only public re-export and has no external consumer (`sprig` calls `buzz_acp::run()` only), and `run()` itself (`lib.rs:1233`) has no doc comment; `buzz-cli`'s `POST /count` integration is fully implemented with retry and has zero callers, kept alive by `#[allow(dead_code)]` (`client.rs:802-834`), as is `relay_url()` (`:567`); and `parse_retry_in_secs` (`client.rs:172-186`) plus `percent_encode` (`validate.rs:75-99`) are `#[cfg(test)]`-gated functions in production files with 6 and 5 tests each, covering code that never runs — while `README.md:216` still advertises percent-encode as live. | `crates/buzz-acp/Cargo.toml:22`; `config.rs:534`; `lib.rs:15`, `:1233`; `crates/buzz-cli/src/client.rs:172-186`, `:567`, `:802-834`; `validate.rs:75-99` |
| DEBT-80 | **`AGENTS.md § Common Gotchas #2` states a flat rule the implementation contradicts.** The doc says omitting `kinds` triggers the p-gate 403; the actual predicate `p_gated_filters_authorized` (`crates/buzz-relay/src/handlers/req.rs:1038-1071`) exempts any filter carrying non-empty `ids` or a `#p` whose values all equal the authenticated pubkey. Five kindless filters in the CLI's messaging commands depend on exactly that undocumented nuance (`messages.rs:63`, `:91`, `:328`; `social.rs:74`; `feed.rs:19`), and all five pass only because an exemption happens to apply. | `AGENTS.md § Common Gotchas #2` vs `crates/buzz-relay/src/handlers/req.rs:1038-1071` |
| DEBT-81 | **Twelve stale in-code comments and cross-references across the agent surface.** Five stale `file:line` pointers in `buzz-acp` (`queue.rs:163`, `:654`, `:702`, `:765`; `lib.rs:2846`); three doc-comment splices leaving public items undocumented (`pool.rs:426-430`/`:482`, `:1028`, `:2824-2826`); `drain_stale_responses` marked "not yet wired" with `#[allow(dead_code)]` instead of a marker (`acp.rs:1022`). In `buzz-agent`: eleven lines describing byte-cap clamping sit above a one-line `None => current_tokens` arm whose logic lives elsewhere (`handoff.rs:152-163`); `execute_calls`' doc describes a "(degenerate, max_parallel=1) path" as though two code paths existed (`agent.rs:284-293`); the `[Base]` gating note (`lib.rs:255-259`) describes a gate that lives in `buzz-acp` (`pool.rs:181`). In `buzz-cli`: `users.rs:5`'s `TODO(phase-4)` is complete (the work is at `users.rs:299`, and the file's only `EventBuilder` match is the comment itself); `messages.rs:316-318` says "no kind restriction" immediately above a kind restriction (`:320`); `dms.rs:66-67` claims to use the SDK builder while `:69` hand-builds an `EventBuilder`; `emoji.rs:104` says "be defensive" above an `events.last()` that is not (`:105`); `notes.rs:24-25` is a merged-PR note left behind. | `crates/buzz-acp/src/queue.rs:163`, `:654`, `:702`, `:765`; `crates/buzz-agent/src/handoff.rs:152-163`; `crates/buzz-cli/src/commands/users.rs:5`, `messages.rs:316-320`, `dms.rs:66-69` |
| DEBT-82 | **File-size discipline: seven of thirteen `buzz-acp` files and four others exceed the 1,000-line ceiling `AGENTS.md` documents, and no Rust gate enforces it.** `buzz-acp/src/lib.rs` is 6,570 lines; `relay.rs` 6,233; `pool.rs` 5,620; `queue.rs` 4,759; `acp.rs` 3,717; `config.rs` 2,709. In `buzz-agent`, `llm.rs` 3,846 (was 2,894 — `16d4ec33` added the mesh-routing subsystem) and `config.rs` 2,709 (was 2,701). In `buzz-cli`, `client.rs` 2,477, `lib.rs` 2,035, `channels.rs` 1,713, `notes.rs` 1,330, `messages.rs` 1,167, `mem.rs` 1,045. In `buzz-dev-mcp`, `shell.rs` 1,503 and `view_image.rs` 1,136. The three `check-file-sizes.mjs` scripts cover desktop, web and mobile only; `just check` has no Rust size step. `AGENTS.md` states the ceiling inside the **Mobile App § Rules** section while also saying "mirroring desktop/web", so whether it was ever intended to bind Rust is genuinely ambiguous — either the doc should scope it explicitly or a Rust gate should exist. Function-level: `run_prompt_task` 948 lines (`pool.rs:1265-2212`), `cmd_patch` 168 (`mem.rs:538-705`), `cmd_create_channel_from_template` 133 (`channels.rs:655-787`), and seven `#[allow(clippy::too_many_arguments)]` suppressions in `buzz-cli` on functions of 8-16 parameters that each immediately construct the SDK meta struct they could have taken as a parameter. | `AGENTS.md` Mobile App § Rules; `justfile:123`, `:585`, `:617`; `wc -l` across the four crates |

Positive controls confirmed in 2d: **zero `unsafe`** in `buzz-acp`, `buzz-agent` and `buzz-cli`
(`#![deny(unsafe_code)]` / `#![forbid(unsafe_code)]` at each crate root); **zero
`#[allow(dead_code)]`** in `buzz-agent`'s core, LLM and tools groups; `unwrap()`/`expect()` on
production paths still reduced to four sites across ~34,000 lines (`config.rs:510`, `llm.rs:1516`
— re-verified as an `unreachable!` in the retry loop's final-iteration arm, not an `unwrap`/`expect`,
same as before the sync; moved from the pre-sync `:1162` by the mesh-routing insertions earlier in
the file — `agents.rs:299`, `client.rs:506`/`:1379`) — though nothing enforces the rule mechanically, since
`clippy::unwrap_used` is allow-by-default with no `[lints]` table and no `clippy.toml`. The
`buzz-cli` retry and idempotency layer is the best-engineered code in the batch: a kind-aware
`DeliveryUnknown` policy that deliberately declines to retry ambiguous moderation writes
(`client.rs:863-870`), 12 behavioural integration tests over live axum/TCP servers
(`client.rs:1582-2295`), and an asserted invariant tying `RETRY_BASE_SECS.len()` to
`RETRY_MAX_ATTEMPTS` so a constant change cannot silently panic (`client.rs:1547-1552`).
`buzz-agent`'s `config.rs` carries dated vendor-doc citations on every capability table
(`:11`, `:101`, `:194`, `:315`, `:578`, `:591`), which makes staleness auditable rather than
invisible — the opposite of the pattern in DEBT-58 and DEBT-64.

## Complexity Hotspots (scan-level, by size)

| Module / File | LOC | Signal |
|---|---|---|
| `crates/buzz-relay` | 60,056 (76 files) | Largest crate; sole integrator of all subsystems |
| `desktop/src` (frontend) | 281,922 (1,249 files) | Largest module overall; 28 feature packages |
| `desktop/src-tauri` | 97,897 (219 files) | Large native shell; excluded from root cargo workspace |
| `crates/buzz-acp` | 33,155 (14 files) | High LOC-per-file ratio (~2,368 avg) — `relay.rs` 3,143, `queue.rs` 2,565 |
| `crates/buzz-db` | 25,580 (22 files) | Data-access surface breadth |
| `crates/buzz-agent` | 17,864 (20 files) | — |
| `crates/buzz-cli` | 15,211 (27 files) | — |

## Dependency Issues (scan-level)

| Issue | Location | Impact |
|---|---|---|
| `aws-creds` pinned to a git fork pending upstream `durch/rust-s3#449` | `Cargo.toml` `[patch.crates-io]` | Blocks clean crates.io-only builds; needed for EKS Pod Identity |
| `iroh` at `1.0.0-rc.0` | workspace deps (mesh feature) | Pre-release dependency in opt-in feature |
| Patched npm deps (`isomorphic-git`, `virtua@0.49.3`) | `pnpm-workspace.yaml` `patchedDependencies` | Maintenance burden on upgrade |
| Radix `react-dismissable-layer` pinned via override | `pnpm-workspace.yaml` | Workaround for a stuck `pointer-events` bug (#1482) |
| No sqlx offline query cache (`sqlx::query()` not `query!()`) | `buzz-db` | No compile-time SQL validation |

## Documentation Drift (scan-level)

| Drift | Detail |
|---|---|
| `.env.example` documents Typesense | Search moved to Postgres FTS; no Typesense service in compose |
| Keycloak in compose is undocumented | No relay integration surface described |
| `ARCHITECTURE.md` crate reference incomplete | Documents ~15 crates; workspace has 26 (missing: agent, dev-mcp, persona, push-gateway, relay-mesh, conformance, pair-relay, pairing-cli, git-*, sprig, ws-client) |
| Rate limiting: docs contradict config | ✅ **RESOLVED in 2b — the docs are wrong.** `ARCHITECTURE.md` §9 says not enforced; the Redis limiter is real, ungated, and enforced before work. See DEBT-16 |
| Legacy `sprout` naming | `repository` field in `Cargo.toml` points to `block/sprout`; `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS` in CI |

## Coverage Gaps

_Pending Phase 2._

## TODO / FIXME Inventory

_Pending Phase 2._

## Prioritized Remediation

_Pending Phase 3._

---

# Phase 2 — Module Findings

## Module: buzz-core (`crates/buzz-core`)

### Aspect: Debt

Findings are factual observations from the source. No severity ranking is implied by ordering.

---

### 1. Size and complexity hotspots

#### File sizes (`wc -l`, 20 `.rs` files, 7,577 lines total)

| File | Lines | Test module starts | Non-test lines (approx.) |
|---|---|---|---|
| `src/pairing/session.rs` | 1,315 | `:756` | ~755 |
| `src/engram.rs` | 1,049 | `:607` | ~606 |
| `src/git_perms.rs` | 1,003 | `:601` | ~600 |
| `src/kind.rs` | 955 (was 784) | `:823` (was `:747`) | ~822 |
| `src/pairing/qr.rs` | 588 | `:245` | ~244 |
| `src/agent_turn_metric.rs` | 508 | `:194` | ~193 |
| `src/pairing/crypto.rs` | 413 | `:131` | ~130 |
| `src/network.rs` | 362 (was 200) | `:97` (was `:56`) | ~96 |
| `src/filter.rs` | 300 | `:106` | ~105 |
| `src/tenant.rs` | 275 | `:175` | ~174 |
| `src/pairing/types.rs` | 242 | `:98` | ~97 |
| `src/channel.rs` | 198 | `:181` | ~180 |
| `src/observer.rs` | 159 | `:113` | ~112 |
| `src/relay.rs` | 121 | `:80` | ~79 |
| `src/pairing/mod.rs` | 80 | — | 80 |
| `src/lib.rs` | 75 | — | 75 |
| `src/event.rs` | 74 | `:53` | ~52 |
| `src/presence.rs` | 73 | `:37` | ~36 |
| `src/verification.rs` | 71 | `:34` | ~33 |
| `src/error.rs` | 20 | — | 20 |

Three files exceed 1,000 lines. The repo's per-file size guard (`mobile/scripts/check-file-sizes.mjs`, 1,000-line ceiling) applies to `mobile/`, `desktop/`, and `web/`; no equivalent guard was found for Rust crates, so these are not gate failures — just the largest units in the crate.

#### Longest non-test functions

| Lines | Function | Location | Notes |
|---|---|---|---|
| 117 | `decode_qr` | `src/pairing/qr.rs:104` | one function performs 9 distinct validations plus query parsing; each step is a separate guard clause |
| 85 | `parse_strict_json` | `src/engram.rs:283` | contains a nested `struct StrictValue` and two full trait impls (`DeserializeSeed`, `Visitor` with 11 visit methods) inside the function body |
| 71 | `evaluate_ref_update` | `src/git_perms.rs:508` | 4 sequential policy checks with distinct denial messages |
| 70 | `filter_match_one` | `src/filter.rs:35` | 6 field checks plus the nested `#h` fallback with 4 levels of `if` nesting (`:83-100`) |
| 70 | `validate_and_decrypt` | `src/engram.rs:488` | 8 envelope checks + decrypt + re-derivation |
| 59 | `parse_protection_tag_with_warnings` | `src/git_perms.rs:303` | |
| 58 | `RefPattern::parse` | `src/git_perms.rs:83` | one 60-line `for` loop with a 5-branch `if/else if` chain (`:97-144`) |
| 48 | `handle_offer` | `src/pairing/session.rs:149` | |
| 45 | `extract_refs` | `src/engram.rs:384` | hand-rolled byte scanner with two nested `while` loops and manual index arithmetic |

`kind.rs` is long but flat: 130 constants, 4 slices, 11 functions, and a 38-line compile-time assertion block — no control flow deeper than a `matches!`.

---

### 2. Panic-capable code in non-test paths

The repo rule is "do not introduce new `unwrap()`/`expect()` in production paths" (AGENTS.md). Current state:

| Construct | Location | Justification present in code? |
|---|---|---|
| `.expect("valid keys produce conversation key")` | `src/engram.rs:137` | no inline justification; relies on caller supplying valid keys |
| `.expect("HMAC-SHA256 is keyed-prefix MAC; new_from_slice cannot fail")` | `src/engram.rs:149` | yes — `:145-147` explains why it is infallible and why the error is not propagated |
| `unreachable!()` in `RefPattern::matches` | `src/git_perms.rs:163` and `:177` | no comment; relies on the invariant that `RecursiveWildcard` only ever occupies the last segment (enforced in `parse` at `:100-107`). A `RefPattern` constructed by any future path that violates that invariant would panic |
| `.expect("sign")` ×2 | `src/lib.rs:59`, `:68` | inside the `test-utils`-gated helper module, not compiled into a normal build |

Everything else in the crate returns `Result`. No `panic!`, `todo!`, or `unimplemented!` appears in `src/`.

---

### 3. Duplicated constants and hard-coded literals

| Duplication | Locations |
|---|---|
| NIP-44 plaintext cap `65_535` defined **three times** | `observer::OBSERVER_MAX_PLAINTEXT_LEN` (`src/observer.rs:25`), `engram::NIP44_PLAINTEXT_MAX` (`src/engram.rs:28`), inline literal in the pairing session (`src/pairing/session.rs:609`) |
| NIP-44 ciphertext window `132..=87472` defined twice | named constants `observer::NIP44_MIN_CONTENT_LEN` / `NIP44_MAX_CONTENT_LEN` (`src/observer.rs:21`, `:23`) vs an inline range literal in `PairingSession::decrypt_message` (`src/pairing/session.rs:595`) |
| Result-gated kind set defined twice | `kind::RESULT_GATED_KINDS` (`src/kind.rs:129`) vs two hard-coded `!=` comparisons in `reader_authorized_for_event` (`src/filter.rs:25`) — adding a kind to the constant will not extend the runtime gate |
| Pairing kind | `kind::KIND_PAIRING` (`src/kind.rs:405`) re-derived into a module-private `PAIRING_KIND: u16` (`src/pairing/session.rs:46`) |
| `KIND_*` → `Kind::Custom(x as u16)` casts | repeated at `src/engram.rs:468`, `:860`, `src/pairing/session.rs:46`, and in tests (`src/filter.rs:244`, `:276`, `src/observer.rs:131`, `:147`, plus bare numeric `Kind::Custom(44200)` at `src/agent_turn_metric.rs:239`, `:262`, `:493`) — no shared conversion helper despite `event_kind_u32`/`event_kind_i32` existing for the reverse direction (`src/kind.rs:772-780`) |
| Tag-kind string comparison idiom | `t.kind().to_string() == "d"` / `"p"` / `"h"` appears at `src/engram.rs:506`, `:525`, `:851`, `src/filter.rs:74`, `:84` — string allocation per tag per comparison, and no shared helper |

---

### 4. Registry consistency issues (`kind.rs`)

| Finding | Evidence |
|---|---|
| 3 of 130 kind constants are absent from `ALL_KINDS` with **no code comment explaining the omission**: `KIND_AUTH` (22242), `KIND_NOSTR_IDENTITY_BINDING` (24243), `KIND_PUSH_LEASE` (30350) | `src/kind.rs:77`, `:81`, `:109` vs the list at `:566-693` |
| The implied "never stored ⇒ excluded" rule does not hold: `KIND_BLOSSOM_AUTH` (`:78-79`) and `KIND_HTTP_AUTH` (`:82-83`) carry "not stored" doc comments yet appear in `ALL_KINDS` (`:627`, `:630`) | same |
| `no_duplicate_kind_values` only covers the 127 listed kinds; the 3 excluded constants are outside the duplicate check | `src/kind.rs:828-834` |
| Declaration order does not follow numeric order (e.g. `KIND_CHANNEL_METADATA` 41 at `:54` sits between 30030 (`:52`) and 5 (`:56`); `KIND_STREAM_MESSAGE` 9 at `:343` sits inside the 40000-series block), so a reader cannot scan for range collisions visually | `src/kind.rs:9-563` |
| Naming inconsistency: 4 admin command kinds use a `RELAY_ADMIN_` prefix while the other 126 use `KIND_` | `src/kind.rs:329-335` |
| `KIND_CHANNEL_METADATA` (41) is documented "Not used by Buzz today" yet is a live member of `is_replaceable` and `ALL_KINDS` | `src/kind.rs:53-54`, `:705`, `:578` |
| Legacy-migration notes remain in doc comments as historical record: "V1 used kind:10001 (replaceable range — wrong), then 40001" (`:414`), "V1 used kind:10002 (replaceable range — wrong)" (`:420`), "V1 used kind:10004 (replaceable range + NIP-51 collision — wrong)" (`:422`), "V1 used addressable range (30001–30003) — wrong" (`:488`) | |
| Self-identified scaling note: `AUTHOR_ONLY_KINDS` is "a tiny linear set… If this grows past ~4 kinds, convert to a compile-time bitset or sorted array" | `src/kind.rs:118-119` |
| An empty section header with no members: `// User groups (47000–47999)` | `src/kind.rs:524` |

---

### 5. Documentation contradictions

| Repo doc claim | Reality in code | Delta |
|---|---|---|
| `ARCHITECTURE.md:142`: "`buzz-core` defines all 81 kinds as `pub const KIND_*: u32`" | 130 kind constants (`src/kind.rs`, 134 `u32` consts minus 4 range bounds) | undercounts by 49 |
| `ARCHITECTURE.md:346`: "`pub const ALL_KINDS: &[u32]  // 80 entries (KIND_AUTH excluded — never stored)`" | `ALL_KINDS` has 127 entries (`src/kind.rs:566-693`) | undercounts by 47 |
| `ARCHITECTURE.md:346`: only `KIND_AUTH` named as excluded | 3 constants are excluded (`KIND_AUTH`, `KIND_NOSTR_IDENTITY_BINDING`, `KIND_PUSH_LEASE`) | 2 omissions; the `KIND_AUTH` exclusion itself is accurate |
| Task brief / module description: "~81 kinds" | 130 | same drift as ARCHITECTURE.md |
| Task brief / module description: "SSRF helpers" (plural) | exactly one *public* function, `is_private_ip` (`src/network.rs:46`); `c26bf594` added a private helper `embedded_ipv4` (`:15-20`), so the public surface is still singular | minor |
| `AGENTS.md` gotcha #1: "Kind `39000` for channel metadata, not `41` — kind 41 is NIP-01 (unused)" | matches the code (`src/kind.rs:53-54`, `:362`) | consistent |
| `AGENTS.md`: "All event kind integers are defined in `buzz-core/src/kind.rs`" | consistent for Rust; parallel mirrors exist outside this crate (`desktop/src/shared/constants/kinds.ts`, `mobile/.../nostr_models.dart` per AGENTS.md) with no automated cross-language drift check found in this crate | partial |
| `crates/buzz-core/Cargo.toml:29`: "NO tokio, NO sqlx, NO redis, NO axum" | true today, but nothing enforces it: root `deny.toml:90-92` `[bans]` has no per-crate bans, and no `[workspace.lints]`/`[lints]` table exists | convention only |

---

### 6. Test-coverage gaps

229 `#[test]` functions exist (static count; was 213 at `b8510ede` — `c26bf594` added 6 in `network.rs`, `07d0265c` added 10 in `kind.rs`), but coverage is uneven. Gaps identified by reading the test modules:

| Area | Gap | Evidence |
|---|---|---|
| `error.rs` | no tests at all; `VerificationError` `Display` strings are unverified | `src/error.rs` has no `#[cfg(test)]` block |
| `event.rs` | its single test (`tampered_signature_fails_verify`, `:56-66`) exercises `nostr::Event`, not `StoredEvent`. `StoredEvent::new`, `with_received_at`, and `is_verified()` have no direct test | `src/event.rs:53-67` |
| `channel.rs` | the only test covers `canonical_channel_name`. `MemberRole::permission_level`, `has_at_least`, `is_elevated`, all three `FromStr` impls, `as_str`, and `Display` have no direct tests in this module (the role ladder is exercised indirectly through `git_perms` tests) | `src/channel.rs:181-198` |
| `filter.rs` | 6 tests. No test for: `ids` **prefix** matching (only exact ids are used), `since`/`until` **boundary equality** (`created_at == since`), multiple generic-tag keys AND-ed together, a filter with no constraints matching everything, or a present-but-empty `kinds` list | `src/filter.rs:106-300` |
| `kind.rs` | 14 tests (was 4 at report time; `07d0265c` added 10 covering `persona_event_is_shared` / `is_unshared_persona_event`, `kind.rs:861-954`). Still open: no test asserts the `ALL_KINDS` **count**, no test asserts that the 3 excluded constants are intentionally absent, and `is_command_kind` / `is_relay_admin_kind` / `is_workflow_execution_kind` / `is_identity_archive_request_kind` / `is_ephemeral` / `event_kind_u32` / `event_kind_i32` have no unit tests (some are covered only by the compile-time assertion block) | `src/kind.rs:823-955` |
| `observer.rs` | 2 tests. `encrypt_observer_payload`'s `PlaintextTooLarge` path and the upper ciphertext bound (87,472) are untested; only the too-short case is covered | `src/observer.rs:113-158` |
| `relay.rs` | 3 tests. `MissingHost` and `InvalidUrl` variants are never asserted; only `InvalidScheme`, `Credentials`, `Fragment` are | `src/relay.rs:80-121` |
| `tenant.rs` | `TenantContext` equality/clone semantics and `CommunityId` ordering (`PartialOrd`/`Ord` are derived) are untested | `src/tenant.rs:175-275` |
| `engram.rs` | `select_head`'s **id tiebreak** branch is not actually exercised: the test named `select_head_lww_with_id_tiebreak` (`src/engram.rs:789`) builds two events with *different* `created_at` values (`1_700_000_000` and `1_700_000_001` at `:800-801`), so only the timestamp branch runs; its own comment claims three events and a tie check (`:790-791`). `Body::Core` round-trip through `build_event`/`validate_and_decrypt` and the `BodyTooLarge` boundary (exactly 65,535 vs 65,536) are also untested | `src/engram.rs:789-803` |
| `agent_turn_metric.rs` | `validate()` is well covered, but the documented `session_id`/`turn_seq`-with-`cumulative` requirement has no test because it has no implementation | `src/agent_turn_metric.rs:147-169` |
| `git_perms.rs` | `MAX_PROTECTION_RULES` (51-tag) path, `MAX_PATTERN_LENGTH` (257-char) path, and `parse_protection_tags` (the multi-tag entry point) have no tests — all 34 tests call `parse_protection_tag` on a single tag | `src/git_perms.rs:601-1003` |
| `pairing/session.rs` | no test for `sign_event`, `relay_urls`, or the ciphertext-length rejection path in `decrypt_message`; the 0–30 s jitter is untested | `src/pairing/session.rs:756-1315` |
| whole crate | no property-based tests (no `proptest`/`quickcheck` dependency or usage), despite several pure functions with clear algebraic properties (`normalize_host` idempotence, `Body` JSON round-trip, `is_private_ip` range partitioning, `select_head` total order) | `Cargo.toml:13-27` |
| whole crate | no integration test directory (`tests/`); all tests are unit tests inside the crate | crate root contains only `Cargo.toml` and `src/` |

Note on measurement: the crate was **not compiled** during this analysis (`cargo` is unavailable without the repo's Hermit activation in this environment), so all test counts are static reads of `#[test]` attributes and no pass/fail state is claimed.

---

### 7. Possibly-unused public surface

Measured by searching all `*.rs` files under `crates/` and `desktop/src-tauri/` (excluding `crates/buzz-core/` itself) for each symbol name. Caveats: constants may also be referenced by numeric literal, by the TypeScript/Dart kind mirrors, or via type inference (so a type can be *used* without its name appearing).

**24 of 130 kind constants have no by-name reference outside `buzz-core`:**

`KIND_CHANNEL_METADATA`, `KIND_FILE_METADATA`, `KIND_BLOSSOM_AUTH`, `KIND_HTTP_AUTH`, `KIND_NIP29_CREATE_INVITE`, `KIND_NIP43_MEMBER_ADDED`, `KIND_NIP43_MEMBER_REMOVED`, `KIND_NIP29_GROUP_ROLES`, `KIND_HUDDLE_REACTION`, `KIND_SYSTEM_MESSAGE`, `KIND_CHANNEL_SUMMARY`, `KIND_DM_CREATED`, `KIND_JOB_ACCEPTED`, `KIND_JOB_CANCEL`, `KIND_JOB_ERROR`, `KIND_WORKFLOW_STEP_STARTED`, `KIND_WORKFLOW_STEP_COMPLETED`, `KIND_WORKFLOW_STEP_FAILED`, `KIND_WORKFLOW_COMPLETED`, `KIND_WORKFLOW_FAILED`, `KIND_WORKFLOW_CANCELLED`, `KIND_WORKFLOW_APPROVAL_GRANTED`, `KIND_AUDIT_ENTRY`, `KIND_MEDIA_UPLOAD`.

Several of these are documented as relay-emitted or client-side kinds (e.g. `KIND_CHANNEL_SUMMARY` is relay-signed, `KIND_MEDIA_UPLOAD` is labelled "Internal kind for media upload audit entries. Not a relay event kind." at `src/kind.rs:541`), so absence of a Rust reference does not by itself mean the kind is dead.

**Other public items with no by-name reference outside `buzz-core`:**

| Item | Location | Note |
|---|---|---|
| `parse_protection_tag_with_warnings` | `src/git_perms.rs:303` | the `unknown_rules` reporting path exists so callers "can log warnings" (doc at `:376-377`) but no caller outside this crate reads it |
| `default_min_role` | `src/git_perms.rs:403` | used internally by the evaluator |
| `evaluate_ref_update` | `src/git_perms.rs:508` | consumers appear to use `evaluate_push` |
| `EffectiveRules` (and `for_ref`) | `src/git_perms.rs:432`, `:447` | used internally |
| `MAX_PROTECTION_RULES`, `MAX_PATTERN_LENGTH`, `MAX_WILDCARDS_PER_PATTERN` | `src/git_perms.rs:19-23` | limits not surfaced to callers for error messages |
| `NIP44_MIN_CONTENT_LEN`, `NIP44_MAX_CONTENT_LEN` | `src/observer.rs:21`, `:23` | see the duplication finding in §3 |
| `D_TAG_DOMAIN` | `src/engram.rs:24` | exported for spec traceability |
| `MemberRole::permission_level`, `MemberRole::has_at_least` | `src/channel.rs:142`, `:155` | used internally by `git_perms` |
| `Body::is_tombstone` | `src/engram.rs:183` | callers appear to match on `Body::Memory { value: None, .. }` directly |
| `validate_slug` | `src/engram.rs:67` | callers go through `normalize_slug` |
| `QrPayload` (type name) | `src/pairing/qr.rs:34` | the type **is** exercised by `crates/buzz-pairing-cli/src/main.rs:119-227` through `encode_qr`/`decode_qr` and inference — a naming-search artifact, not dead code |

---

### 8. Structural / consistency observations

| # | Observation | Evidence |
|---|---|---|
| D-1 | Orphaned comment fragment with no antecedent line, inside the engram test module: `//    vectors". Pinning these as CI invariants is the single best` — reads as a partially deleted doc block | `src/engram.rs:615` |
| D-2 | A `// SAFETY:` comment annotates a **safe** slice operation; the convention normally marks `unsafe` blocks, of which the crate has none | `src/engram.rs:410-413` |
| D-3 | `filter.rs` re-implements the result-gated kind check instead of consuming `kind::RESULT_GATED_KINDS`, creating two sources of truth for a security-relevant set | `src/filter.rs:25` vs `src/kind.rs:129` |
| D-4 | `git_perms::parse_protection_tags` checks the rule cap **before** parsing, so a malformed 51st tag surfaces `TooManyRules` instead of its parse error | `src/git_perms.rs:383-394` |
| D-5 | `PatternError::InvalidSegment(String)` is used for two unrelated conditions — a bad segment and "`**` must be the last segment" — the latter passing a *sentence* as the segment value | `src/git_perms.rs:101-106` vs `:123-137` |
| D-6 | `MAX_WILDCARDS_PER_PATTERN = 3` counts `*` and `**` together; the doc comment says "wildcard segments per pattern" without stating that `**` counts as one, so the effective limit for `refs/*/*/*` (3) vs `refs/*/*/**` (3) is only discoverable from code | `src/git_perms.rs:23`, `:100-116` |
| D-7 | `agent_turn_metric.rs` re-exports `ObserverPayloadError` under a second name (`AgentTurnMetricError`) while its own functions return the original type in their signatures, so both names appear in the public API for the same errors | `src/agent_turn_metric.rs:15` vs `:169`, `:185` |
| D-8 | `AgentTurnMetricPayload.timestamp` is an unvalidated `String` and `channel_id` is a `String` rather than `Uuid`, despite the crate already depending on `uuid` and `chrono` | `src/agent_turn_metric.rs:97` (`channel_id`), `:112` (`timestamp`) |
| D-9 | `TokenCounts` mixes two serde strategies: four fields serialize `null` when absent while `cache_read_tokens`/`cache_write_tokens` are skipped entirely — intentional per the "not reported vs zero" contract, but it means consumers must handle both shapes | `src/agent_turn_metric.rs:24-42`, test `:301-319` |
| D-10 | `StopReason` has a derived `Serialize` but a hand-written `Deserialize`, so the two are maintained independently; a new variant added to the enum will silently deserialize as `Unknown` until the manual match is updated | `src/agent_turn_metric.rs:49-77` |
| D-11 | `parse_strict_json` embeds a full serde `Visitor` implementation (11 methods) inside a function body, which keeps it private but makes it untestable in isolation and unavailable for reuse by other strict-JSON needs | `src/engram.rs:283-380` |
| D-12 | `extract_refs` is a hand-rolled byte scanner with manual index arithmetic and documented surprising cases (`[[[mem/x]]]` yields nothing) rather than a parser or regex; correctness rests entirely on its 15 tests | `src/engram.rs:384-430`, tests `:891-1047` |
| D-13 | Engram `d`-tag and `p`-tag comparisons are non-constant-time (`!=`, `eq_ignore_ascii_case`) while the pairing module uses `ct_eq` for analogous 32-byte comparisons — an internal inconsistency in comparison policy | `src/engram.rs:533`, `:551` vs `src/pairing/crypto.rs:126-129` |
| D-14 | `PairingSession` is split across four `impl` blocks (`:109`, `:282`, `:424`, `:546`) plus a `#[cfg(test)]` block (`:530-531`); role-specific methods are separated by block but nothing in the type system prevents calling a Target method on a Source session — enforcement is the runtime `expect_role` check | impl blocks at `src/pairing/session.rs:109`, `:282`, `:424`, `:531`, `:546`; `expect_role` at `:717-726` |
| D-15 | `SessionState` has 7 variants and transitions are asserted by string-formatted `UnexpectedMessage` errors rather than a typed transition table, so an illegal transition is only discoverable at runtime | `src/pairing/session.rs:59-78`, `:706-726` |
| D-16 | `PairingError::UnexpectedMessage` is overloaded for at least five distinct failure classes (wrong message type, wrong state, wrong role, bad content length, oversized plaintext), so callers cannot distinguish protocol-shape errors from size-limit errors | 11 construction sites: `src/pairing/session.rs:166` (version), `:272` (complete=false), `:432`/`:451` (terminal state), `:596` (content length), `:611` (plaintext size), `:639` (duplicate id), `:646` (kind), `:708` (state), `:719` (role), `:750` (message type) |
| D-17 | `QrPayload` derives `Clone` **and** implements `Drop` to zeroize its secret — a clone extends the lifetime of the secret beyond the original's drop, and a pairing test relies on that clone (`session.rs:990`) | `src/pairing/qr.rs:37`, `:56-60` |
| D-18 | `is_private_ip` mixes std helpers with hand-written bit masks; the std helpers' exact semantics (e.g. what `is_private()` covers) are only documented in the doc comment, not asserted by tests for each RFC1918 block boundary. **Still open at `07d0265c`** — the IPv4 arm is untouched, and `c26bf594` added more hand-written masks on the IPv6 side (`:83-92`) alongside the new octet-prefix helper `embedded_ipv4` (`:15-20`) | `src/network.rs:48-60` |
| D-19 | The crate exposes both `normalize_host` (`tenant.rs:121`) and `normalize_relay_url` (`relay.rs:37`) and `relay_url_authority` (`tenant.rs:156`) — three overlapping normalizers with deliberately different rules; the differences are documented but a caller must read all three to pick correctly | `src/tenant.rs:106-172`, `src/relay.rs:20-78` |
| D-20 | `nostr` types are re-exported from the crate root (`lib.rs:42`), so consumers can depend on `buzz_core::Event` and be coupled to the `nostr` 0.44 major version through this crate without declaring it | `src/lib.rs:42`, `Cargo.toml:14` |

---

### 9. Deprecated API usage

No `#[deprecated]` attribute is declared in this crate, and no call site is annotated as using a deprecated API. Nothing in the crate references a `deprecated` item by name. The closest artefacts are the historical "V1 used kind:X — wrong" doc notes in `src/kind.rs:414-422`, `:488`, which record superseded kind numbers rather than deprecated Rust items.


## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: Debt

---

### 1. File-size distribution

| File | Total LOC | Production LOC | Test LOC | Tests start |
|---|---|---|---|---|
| `crates/buzz-sdk/src/builders.rs` | 3 824 | 1 824 | 1 999 | `builders.rs:1825` |
| `crates/buzz-sdk/src/mentions.rs` | 820 | 388 | 432 | `mentions.rs:389` |
| `crates/buzz-sdk/src/nip_oa.rs` | 595 | 300 | 295 | `nip_oa.rs:301` |
| `crates/buzz-sdk/src/lib.rs` | 112 | 112 | 0 | — |
| `crates/buzz-sdk/examples/compute_auth_tag.rs` | 29 | 29 | 0 | — |
| **Total** | **5 380** | **2 653** | **2 726** | — |

`builders.rs` is 71 % of the crate and holds 51 of the 61 public functions in
one file. Slightly more than half the crate's lines are tests.

For reference, the repo enforces a 1 000-line ceiling on mobile files
(`AGENTS.md` § Mobile Rules, `mobile/scripts/check-file-sizes.mjs`) and the same
guard exists for desktop/web; no equivalent guard applies to Rust crates, so
`builders.rs` at 3 824 lines is unconstrained.

---

### 2. Complexity hotspots (largest production functions)

| Function | Body lines | File:line | Why it is large |
|---|---|---|---|
| `build_repo_announcement` | 112 | `builders.rs:834-945` | six sequential validation blocks (repo_id, name, description, clone_urls loop, web_url, relays loop) then tag assembly; each block is an inline `if len > N` rather than a shared helper |
| `build_git_status` | 70 | `builders.rs:1222-1291` | validates root/revision/recipients/repo/euc, enforces the merged-only rule, then a four-arm match over `(relay, pubkey)` for `q` tags plus three commit-tag paths |
| `build_diff_message` | 68 | `builders.rs:308-375` | validates 6 `DiffMeta` fields and conditionally emits 10 optional tags |
| `build_git_pull_request` | 62 | `builders.rs:1330-1391` | 5 validations + 11 conditional tag emissions |
| `build_git_patch` | 57 | `builders.rs:1013-1069` | 9 conditional tag emissions including the root/root-revision exclusivity check |
| `build_contact_list` | 48 | `builders.rs:764-811` | per-contact validation loop with three inline bound checks |
| `build_update_channel` | 46 | `builders.rs:604-649` | at-least-one-field check + tri-state TTL handling |
| `strip_code_regions` | 96 | `mentions.rs:244-339` | hand-rolled markdown scanner: nested `loop` inside `while let` with manual byte-offset arithmetic, four separate "is this a line start" predicates, and manual iterator advancement. The highest-complexity function in the crate |
| `verify_auth_tag` | 58 | `nip_oa.rs:179-236` | 8 sequential `ok_or_else` element extractions duplicating `parse_auth_tag`'s first half |
| `parse_auth_tag` | 48 | `nip_oa.rs:252-299` | same extraction sequence as `verify_auth_tag` with different length rules |

---

### 3. Duplication

**Within the crate**

| Duplication | Detail | File:line |
|---|---|---|
| `build_workflow_def` vs `build_workflow_update` | Identical bodies — same kind, same two tags, same content cap, same 64 KiB check. Only the doc comment differs | `builders.rs:1463-1478` vs `1481-1494` |
| `verify_auth_tag` / `parse_auth_tag` element extraction | Lines `nip_oa.rs:181-206` and `254-286` repeat the arity check, label check, and four `as_str().ok_or_else(...)` extractions verbatim; no shared "destructure" helper | `nip_oa.rs:179-299` |
| Five overlapping hex validators | `check_hex_len`, `check_commit_hex`, `check_pubkey_hex`, `check_hex_exact`, plus an inline 64-hex check in `build_contact_list` (`builders.rs:779-784`) and another in `build_workflow_approval` (`builders.rs:1528-1532`) — six variants of "is this hex of length N" | `builders.rs:44-89`, `779-784`, `1528-1532` |
| `check_hex_len` misuse for pubkeys | `build_add_member`/`build_remove_member` validate pubkeys with `check_hex_len(pubkey, 64, …)`, which is a **minimum**-length check and produces an `InvalidDiffMeta` error variant for a membership problem — while every other pubkey site uses `check_pubkey_hex` (exact length, `InvalidInput`). A 65-hex pubkey passes here but is rejected in the moderation builders | `builders.rs:569`, `586` vs `69-77` |
| `build_archive` / `build_unarchive` | Same body modulo the boolean literal | `builders.rs:709-724` |
| `build_set_topic` / `build_set_purpose` | Same body modulo the tag name | `builders.rs:652-667` |
| Inline size literals | 13 sites pass `64 * 1024`, 2 pass `60 * 1024`, plus 2048/1024/512/256/128 scattered inline instead of named constants (contrast with `MAX_CONTACTS`/`MAX_REASON_BYTES` which are named) | `builders.rs:227`–`1816` (see configuration doc) |

**Across crates**

| Duplication | Detail | Evidence |
|---|---|---|
| Desktop maintains a parallel builder library | `desktop/src-tauri/src/events.rs` defines its own `build_create_channel`, `build_join`, `build_leave`, `build_update_channel`, `build_set_topic`, `build_set_purpose`, `build_archive`, `build_unarchive`, `build_delete_channel`, `build_add_member`, `build_remove_member`, `build_message`, `build_forum_post`, `build_forum_comment`, `build_delete_compat`, `build_reaction`, `build_remove_reaction`, `build_set_canvas`, `build_profile`, `build_note`, `build_contact_list`, `build_dm_open`, `build_workflow_delete`, `build_workflow_trigger`, `build_archive_identity_request`, `build_unarchive_identity_request` — 36 `EventBuilder::new` sites, returning `Result<_, String>` instead of `SdkError`. `desktop/src-tauri` declares `buzz-sdk` as a dependency but references no `buzz_sdk::` path in `src/` | `desktop/src-tauri/src/events.rs:143-842`; SDK equivalents `builders.rs:219-1823` |
| Wire-form drift is managed by comment + mirrored tests, not shared code | The SDK's NIP-IA section states it mirrors `desktop/src-tauri/src/events.rs:624-743` "so both clients emit the same wire form", and duplicates the desktop's byte-based reason check with a comment explaining the desktop's misleading error text | `builders.rs:1697-1704`, test comment `builders.rs:3696-3699` |
| `repo_id` validation exists twice | `buzz-cli`'s `validate_repo_id` (`crates/buzz-cli/src/validate.rs:39`) has its own copy with its own 7 tests; the SDK's `check_repo_id` was added specifically because a coordinate could bypass it ("bypassing CLI-side `validate_repo_id`") | `builders.rs:87-91`; `crates/buzz-cli/src/validate.rs:39`, `423-461` |
| `validate_pubkey_hex` in desktop huddle path | third independent pubkey-hex validator | `desktop/src-tauri/src/huddle/relay_api.rs:29` |

---

### 4. Dead code / unused API

- No `#[allow(dead_code)]` and no `#[deprecated]` attributes anywhere in the crate.
- Every one of the 51 public builders plus all 10 public helper functions is
  referenced from at least one `.rs` file outside `crates/buzz-sdk/` (verified by
  per-symbol search across `crates/` and `desktop/`). There is no unreachable
  public API by that measure.
- Reference concentration is uneven: `compute_auth_tag` (12 files),
  `verify_auth_tag` (10), `parse_auth_tag` (8), `build_message` (6),
  `build_archive_identity_request` (6) at the top; 22 builders are referenced from
  exactly one external file, all of them `buzz-cli` command modules (spot-checked:
  `build_custom_emoji_set` → `crates/buzz-cli/src/commands/emoji.rs:119`,
  `build_diff_message` → `crates/buzz-cli/src/commands/messages.rs:659`,
  `build_vote` → `crates/buzz-cli/src/commands/messages.rs:744`).
- `serde` is declared as a dependency (`crates/buzz-sdk/Cargo.toml:14`) but no
  `use serde` or serde derive appears in `src/` — it is only needed transitively
  by `serde_json`, so the direct declaration is unused.

---

### 5. Test coverage gaps

235 tests exist (162 builders / 51 mentions / 22 nip_oa). Gaps observed by
comparing the test module against the production surface:

| Untested or thinly tested | Detail | File:line |
|---|---|---|
| `build_git_pr_update` merge-base / euc validation | happy path, bad `pr_event`, and missing clone URL are covered; invalid `merge_base`/`euc` hex are not | `builders.rs:1416-1460`; tests `builders.rs:3533-3577` |
| `build_repo_announcement` bound checks | repo_id rules have 5 tests, but the `name` > 128, `description` > 1024, `clone_urls` > 5, `clone_url` > 512, `web_url` scheme/length, `relays` > 10, and relay-scheme rejections have no tests | validations `builders.rs:840-919`; tests `builders.rs:2788-2925` |
| `build_repo_announcement_with_tags` | one test (`builders.rs:2837-2866`); no test that an invalid `repo_id` is rejected on this path | `builders.rs:952-963` |
| `build_custom_emoji_set` / `build_custom_emoji_reaction` negatives | happy paths only; no test for duplicate shortcode, empty shortcode, > 64-byte shortcode, non-`http(s)` URL, or > 2048-byte URL | validations `builders.rs:127-170`, `517-521`; tests `builders.rs:2276-2301` |
| `normalize_custom_emoji_shortcode` | no direct unit test; exercised only indirectly through the reaction happy path | `builders.rs:127-150` |
| `build_set_canvas` | one happy-path test; nothing pins the (absent) content bound | `builders.rs:529-532`; test `builders.rs:2310-2317` |
| `build_workflow_approval` note length | no bound exists and no test asserts either way | `builders.rs:1522-1541` |
| `build_dm_add_member` / `build_dm_open` boundary | 8-pubkey upper bound tested at 9 (reject); exactly-8 accept case not tested | `builders.rs:1546`; tests `builders.rs:3324-3346` |
| `extract_channel_id` on multiple `h` tags | first-match behavior is not pinned by a test | `builders.rs:816-826` |
| `strip_code_regions` edge cases | 6 tests; no coverage for indented fences, nested/triple-backtick-inside-fence, unclosed fenced block at EOF, or the `before.chars().all(is_ascii_whitespace)` branch | `mentions.rs:244-339`; tests `mentions.rs:662-700` |
| `merge_mentions` with a custom `cap` | all three tests pass `MENTION_CAP`; no test uses a different cap value | `mentions.rs:208-220`; tests `mentions.rs:616-638` |
| `compute_auth_tag` byte-exactness | round-trip and spec-vector verification are tested, but no test asserts `compute_auth_tag` reproduces the spec signature byte-for-byte (the test comment calls this out: "round-trip without byte comparison") | `nip_oa.rs:335-350` |
| Kind literals vs `buzz-core` constants | 20 builders pass bare integer literals to `Kind::Custom` (9, 45001, 45003, 40008, 40003, 9005, 5, 45002, 7, 40100, 0, 9000, 9001, 9002, 9007, 9021, 9008, 9022, 1, 3) and the tests assert the same literals, so a divergence from `buzz-core/src/kind.rs` would not be caught | `builders.rs:241`, `288`, `304`, …; registry `crates/buzz-core/src/kind.rs` |

No property-based tests (no `proptest`/`quickcheck` anywhere in the crate), and
no integration-test directory (`crates/buzz-sdk/tests/` does not exist) despite
the crate producing wire forms consumed by the relay.

---

### 6. Other debt signals

| Signal | Detail | File:line |
|---|---|---|
| Kind sourcing is inconsistent | 26 builders use `buzz_core::kind::KIND_*` constants (`builders.rs:6-19`), 20 use bare literals. `AGENTS.md` states all kind integers are defined in `buzz-core/src/kind.rs`; the literal sites bypass that registry | `builders.rs:241` (literal) vs `builders.rs:270-271` (constant) |
| Error-variant semantics leak | `InvalidDiffMeta` is used for non-diff failures because `check_hex_len` hardcodes that variant — e.g. an invalid membership pubkey reports "invalid diff metadata" | `builders.rs:44-52` used at `builders.rs:569`, `586` |
| `check_hex_len` is a minimum-length check used where an exact length is meant | see duplication table above — permits over-long pubkeys on kinds 9000/9001 | `builders.rs:44-52`, `569`, `586` |
| Stringly-typed parameters where enums exist | `build_update_channel` takes `visibility: Option<&str>` and re-validates the vocabulary by hand, while `build_create_channel` takes the typed `Option<Visibility>`; `build_presence_update` and `build_moderation_resolve_report` also take raw `&str` vocabularies | `builders.rs:604-621` vs `674-696`; `1570-1584`; `1654-1680` |
| Positional-parameter builders with 5–6 arguments | `build_message` (6), `build_profile` (5), `build_update_channel` (5), `build_repo_announcement` (6), `build_create_channel` (6) — no options struct, so call sites are order-sensitive; the crate already has the `…Meta`/`Options` pattern available | `builders.rs:219`, `537`, `604`, `674`, `834` |
| Cross-repo coupling encoded in comments | correctness depends on line-referenced comments pointing at `desktop/src-tauri/src/events.rs:624-743` / `:635-647`, `moderation_commands.rs`, relay helpers `extract_p_tag_bytes` and `extract_report_tag` — these references will silently rot as those files change | `builders.rs:1697-1704`, `1593-1594`; test comments `builders.rs:3613-3615`, `3688-3690` |
| Manual byte-offset string scanning | `strip_code_regions` (`mentions.rs:244-339`) and `GitAppliedPatchRef::parse` (`builders.rs:1143-1185`) do their own index arithmetic instead of using a parser; both are correct today and guarded by tests, but they are the crate's most fragile code |
| Repeated `as u16` narrowing casts | `buzz_core` kind constants are `u32` and every use is cast `as u16`; a kind ≥ 65536 would wrap silently | 27 sites: `builders.rs:271`, `525`, `944`, `961`, `1068`, `1110`, `1190-1193`, `1390`, `1455`, `1473`, `1491`, `1507`, `1513`, `1538`, `1555`, `1562`, `1580`, `1610`, `1617`, `1636`, `1643`, `1685`, `1798`, `1819` |
| No deprecated API usage | no `#[deprecated]` items and no calls into deprecated `nostr` APIs observed; the crate is current with nostr 0.44 semantics (including the `allow_self_tagging` change at `builders.rs:1801`) | — |


## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: Debt

### 1. Size and complexity profile

Total: 9 `.rs` files, 5,197 LOC (7 in `src/`, 2 in `tests/`).

| File | LOC | Production LOC (before `#[cfg(test)]`) | Test LOC | Test share |
|---|---|---|---|---|
| `crates/buzz-persona/src/validate.rs` | 1,070 | 435 | 635 | 59% |
| `crates/buzz-persona/src/resolve.rs` | 892 | 399 | 493 | 55% |
| `crates/buzz-persona/src/pack.rs` | 734 | 446 | 288 | 39% |
| `crates/buzz-persona/src/persona.rs` | 645 | 330 | 315 | 49% |
| `crates/buzz-persona/src/merge.rs` | 464 | 199 | 265 | 57% |
| `crates/buzz-persona/src/manifest.rs` | 365 | 194 | 171 | 47% |
| `crates/buzz-persona/src/lib.rs` | 6 | 6 | 0 | — |
| `crates/buzz-persona/tests/integration.rs` | 650 | — | 650 | — |
| `crates/buzz-persona/tests/e2e_env_flow.rs` | 371 | — | 371 | — |

Production code is ~2,009 LOC; tests are ~3,188 LOC (1.59:1 test-to-code). No file exceeds
the repo's 1,000-line desktop/mobile guard, though `validate.rs` at 1,070 would trip it if
that guard applied to Rust crates (it does not — `mobile/scripts/check-file-sizes.mjs` per
`AGENTS.md` covers mobile, desktop, and web only).

#### Largest functions (production only)

| Function | Approx. body length | Location |
|---|---|---|
| `load_pack` | ~123 lines | `crates/buzz-persona/src/pack.rs:125` |
| `resolve_persona_config` | ~87 lines | `crates/buzz-persona/src/merge.rs:85` |
| `advisory_check_skill_names` | ~81 lines | `crates/buzz-persona/src/validate.rs:354` |
| `resolve_skills` | ~73 lines | `crates/buzz-persona/src/pack.rs:249` |
| `resolve_loaded_pack` | ~67 lines | `crates/buzz-persona/src/resolve.rs:118` |
| `resolve_one_persona` | ~60 lines | `crates/buzz-persona/src/resolve.rs:194` |
| `parse_persona_file` (private) | ~54 lines | `crates/buzz-persona/src/pack.rs:392` |
| `parse_persona_md` | ~53 lines | `crates/buzz-persona/src/persona.rs:208` |
| `check_respond_to_value` | ~52 lines | `crates/buzz-persona/src/validate.rs:236` |
| `advisory_check_manifest_keys` | ~51 lines | `crates/buzz-persona/src/validate.rs:302` |

Hotspot notes:

- **`load_pack`** (`crates/buzz-persona/src/pack.rs:125-247`) does six distinct jobs in one
  body (manifest, personas, instructions, MCP, skills-dir, plus size-limit computation).
  Two nearly identical explicit-path-else-implicit-fallback blocks appear back to back
  (`:167-188` for instructions, `:191-220` for MCP) — the same four-step shape duplicated.
- **`resolve_persona_config`** (`crates/buzz-persona/src/merge.rs:85-171`) contains two
  hand-written `match (persona_value, default_value)` tuples with 4 and 6 arms
  (`:104-121` for `subscribe`, `:131-152` for `triggers`) that encode the same
  null-vs-empty-container rule twice with different arm structures. The `triggers` match
  ends with an unreachable-in-practice `_ => None` catch-all (`:151`).
- **`advisory_check_skill_names`** (`crates/buzz-persona/src/validate.rs:354-434`) has five
  chained early-`continue` guards and four nested `if let` levels (`:420-431`), so a
  `SKILL.md` that fails any step yields no diagnostic at all.

---

### 2. Dead / orphaned code

| # | Item | Evidence | Status |
|---|---|---|---|
| D1 | **`pack::resolve_skills` has no production caller.** It is fully implemented (73 lines) and covered by 4 tests, but neither `load_pack` nor `resolve_pack` nor `validate_pack` invokes it, and no consumer in the workspace calls it. | grep for `resolve_skills` across the repo yields only `crates/buzz-persona/src/pack.rs:249` (definition) and its own tests (`:611-682`) | orphan — 73 production lines + 4 tests exercising nothing in the live path |
| D2 | **`ResolvedPersona.skills` is documented as "bare names" but carries raw declared paths.** The field comment says "Skills (bare names — reserved for future use, not yet wired)" (`crates/buzz-persona/src/resolve.rs:60-61`) while the assignment is `skills: lp.skills.clone()` (`:249`), i.e. the un-normalized frontmatter strings such as `"./skills/code-review/"`. | `crates/buzz-persona/src/resolve.rs:60-61` vs `:249`; `crates/buzz-persona/tests/integration.rs:73-74` shows the raw form a persona declares | comment/code mismatch; `buzz-cli` prints these raw paths as-is (`crates/buzz-cli/src/commands/pack.rs:124-126`) |
| D3 | **`manifest::parse_manifest_file` is never called and never tested.** | only occurrence is the definition at `crates/buzz-persona/src/manifest.rs:190`; `load_pack` reads + parses manually instead (`crates/buzz-persona/src/pack.rs:136-137`) | untested public API |
| D4 | **`persona::parse_persona_file` is never called inside the crate and never tested.** `load_pack` uses its own private `parse_persona_file` with a different signature (`crates/buzz-persona/src/pack.rs:392`). No workspace consumer calls the public one either. | grep results: definition `crates/buzz-persona/src/persona.rs:262`, private homonym `crates/buzz-persona/src/pack.rs:392`, call site `:165` resolves to the private one | untested public API; the name collision is itself a readability hazard |
| D5 | **`PackManifest.hooks_config` is parsed then discarded.** Pack-level hooks can never take effect through this crate. | field `crates/buzz-persona/src/manifest.rs:116`; drop comment `crates/buzz-persona/src/pack.rs:111-112` | intentional, documented — but leaves a manifest field that silently does nothing |
| D6 | **`PersonaConfig.version` and `PersonaConfig.author` are parsed and then dropped.** Neither reaches `LoadedPersona` (field list `crates/buzz-persona/src/pack.rs:77-98`) nor `ResolvedPersona`; `ResolvedPersona.version` is always the pack version. | `crates/buzz-persona/src/persona.rs:116`, `:119`; `crates/buzz-persona/src/resolve.rs:225-231` | frontmatter `version:`/`author:` are accepted but inert |
| D7 | **`Engines`/`engines.buzz` is parsed and never evaluated.** | `crates/buzz-persona/src/manifest.rs:37-41`; no semver dependency in `crates/buzz-persona/Cargo.toml:9-15` | version negotiation described in `crates/buzz-persona/PERSONA_PACK_SPEC.md:113-114` is unimplemented |
| D8 | **`resolve::resolve_loaded_pack` has no direct test** — it is exercised only transitively through `resolve_pack` (`crates/buzz-persona/src/resolve.rs:110`). | grep: only `:110` and `:118` | untested-as-entry-point public API |
| D9 | **`"repository"` is warning-exempt in the validator but absent from `PackManifest`.** | `crates/buzz-persona/src/validate.rs:108` vs field list `crates/buzz-persona/src/manifest.rs:79-121` | schema drift between the known-keys list and the type |
| D10 | **`buzz-acp` declares `buzz-persona` as a dependency but never uses it.** | `crates/buzz-acp/Cargo.toml:22`; grep for `buzz_persona` across all of `crates/buzz-acp` returns zero matches | unused dependency — adds compile time; also means the crate's primary stated consumer is not actually wired |
| D11 | **Stale comment: "Pack-level description not yet wired through PackManifestData."** The very next line reads it successfully, and the field does exist and is populated. | comment `crates/buzz-persona/src/resolve.rs:178-179`; read `:180`; field `crates/buzz-persona/src/pack.rs:106`; populated `:142` | stale |

---

### 3. Duplication

| # | Duplicated logic | Sites | Risk |
|---|---|---|---|
| U1 | **Persona-name validation implemented twice, independently.** Charset + 64-char rules exist in `resolve_loaded_pack` and again in `validate_persona_name`, with different error plumbing and different constant handling (`MAX_NAME_LEN` const vs. an inline `64`). | `crates/buzz-persona/src/resolve.rs:134-155` vs `crates/buzz-persona/src/validate.rs:167-185` | the two can drift; a change to the rule must be made in both places |
| U2 | **Zero-persona and duplicate-name checks implemented twice.** | `crates/buzz-persona/src/resolve.rs:119-133` vs `crates/buzz-persona/src/validate.rs:189-203` | same as U1; the error strings are already near-identical but not shared ("pack contains zero personas" appears at both `resolve.rs:121` and `validate.rs:190`) |
| U3 | **Built-in trigger defaults declared twice.** `mentions=true / keywords=[] / all_messages=false` appear as literals in `parse_triggers` and again in `resolve_triggers`. | `crates/buzz-persona/src/merge.rs:180-198` vs `crates/buzz-persona/src/resolve.rs:255-268` | a default change in one place silently diverges |
| U4 | **Skill-path normalization implemented twice.** Identical `trim_end_matches('/')` → `file_name()` → fallback chain. | `crates/buzz-persona/src/pack.rs:253-260` (nested fn) vs `crates/buzz-persona/src/validate.rs:365-369` (inline) | |
| U5 | **`plugin.json` read + JSON-parsed three times per validation run.** Once in `load_pack` (`crates/buzz-persona/src/pack.rs:136-137`), then again in `advisory_check_respond_to_types` (`crates/buzz-persona/src/validate.rs:212-219`) and `advisory_check_manifest_keys` (`:304-312`). | 3 reads | wasted I/O plus a TOCTOU window where the validator could report on different bytes than were loaded |
| U6 | **Size-limit arithmetic repeated with inconsistent slack.** `MAX_FRONTMATTER + MAX_BODY + 100` in `persona.rs`, `+ 200` in `pack.rs` — and computed twice within `load_pack` under two different variable names for the same value. | `crates/buzz-persona/src/persona.rs:266`; `crates/buzz-persona/src/pack.rs:153-154` (`persona_size_limit`) and `:168-169` (`text_size_limit`) | the two `pack.rs` expressions are byte-identical yet duplicated; the 100-vs-200 divergence is unexplained |
| U7 | **`crate::persona::` fully-qualified path used where `persona::` is already imported.** `:168-169` writes `crate::persona::MAX_FRONTMATTER_BYTES` while `:153-154` writes `persona::MAX_FRONTMATTER_BYTES` for the same constant. | `crates/buzz-persona/src/pack.rs:153-154` vs `:168-169` | cosmetic inconsistency |
| U8 | **Consumer import-filter logic reimplemented in this crate's tests.** `DERIVED_PROVIDER_MODEL_ENV_KEYS` + `filter_derived` are described as mirroring "desktop import_persona_pack logic". | `crates/buzz-persona/tests/e2e_env_flow.rs:15-32`, comment at `:200` | a test that asserts a *copy* of behavior owned by another crate — will not catch drift in the real implementation |

---

### 4. Test coverage gaps

145 test functions total (per-module: `persona.rs` 29, `resolve.rs` 26, `merge.rs` 22,
`validate.rs` 22, `manifest.rs` 14, `pack.rs` 14; `tests/integration.rs` 13,
`tests/e2e_env_flow.rs` 5). Coverage is dense; the gaps are specific.

| # | Gap | Evidence |
|---|---|---|
| T1 | `persona::parse_persona_file` — no test at all; its `PersonaError::TooLarge` branch is therefore never exercised (only `FrontmatterTooLarge` and `BodyTooLarge` are asserted) | `crates/buzz-persona/src/persona.rs:262-270`; assertions exist only for `:551` and `:559` |
| T2 | `manifest::parse_manifest_file` — no test | `crates/buzz-persona/src/manifest.rs:190-193` |
| T3 | `resolve::resolve_loaded_pack` — no direct test | `crates/buzz-persona/src/resolve.rs:118` |
| T4 | Windows absolute-path rejection (`X:` drive prefix) is `#[cfg(windows)]` and has no test on any platform | `crates/buzz-persona/src/pack.rs:333-337` |
| T5 | `PackError::McpConfigParse` — the malformed-`.mcp.json` path has no test (the only `.mcp.json` test writes valid JSON at `crates/buzz-persona/src/pack.rs:513`) | error defined `crates/buzz-persona/src/pack.rs:52`, raised `:193-196` |
| T6 | Explicit-but-missing `pack_instructions` / `mcp_config` paths (the hard-error branches) are untested; only the implicit fallbacks are covered | branches at `crates/buzz-persona/src/pack.rs:173-178` and `:201-206` |
| T7 | `read_bounded_file`'s oversize rejection is untested (no test writes a >1.25 MiB pack file) | `crates/buzz-persona/src/pack.rs:378-384` |
| T8 | `merge_behavioral_config` non-object short-circuits (scalar `persona_config` or scalar `pack_defaults`) untested | `crates/buzz-persona/src/merge.rs:53-60` |
| T9 | MCP entries with a missing/non-string `command`, or a persona entry with no `name`, are silently dropped — untested | `crates/buzz-persona/src/resolve.rs:294`, `:312` |
| T10 | `.mcp.json` with a shape other than `{"mcpServers": {...}}` (silently contributes nothing) untested | `crates/buzz-persona/src/resolve.rs:285` |
| T11 | `check_respond_to_value` is effectively unreachable in practice — the typed `BehavioralDefaults` parse fails first (test at `crates/buzz-persona/src/validate.rs:900-936` acknowledges this: "Serde catches the type mismatch during manifest parsing"), so its 52 lines of type-error messages are never surfaced. No test asserts the function's own error strings. | `crates/buzz-persona/src/validate.rs:236-287` |
| T12 | Runtime values other than `"buzz-agent"` and `"goose"` (e.g. `"claude"`, `"codex"`, garbage) falling into the GOOSE_* branch — untested | wildcard arm `crates/buzz-persona/src/resolve.rs:380` |
| T13 | `resolve_skills` with a persona claiming a nonexistent skill name — untested | `crates/buzz-persona/src/pack.rs:305` emits it unconditionally |
| T14 | Duplicate entries in `manifest.personas[]` on the bare `load_pack` path (no dedup, no error) — untested | `crates/buzz-persona/src/pack.rs:155-164` |
| T15 | No test covers a persona `name` validated on the `parse_persona_md` path — name charset/length is only checked in `resolve`/`validate`, so a direct-parse consumer (like the desktop importer) is unguarded and no test documents that asymmetry | `crates/buzz-persona/src/persona.rs:208-260` vs `crates/buzz-persona/src/resolve.rs:134-155` |
| T16 | No fuzz/property tests on `split_frontmatter`, which is the most intricate parser in the crate (a hand-rolled delimiter scanner) | `crates/buzz-persona/src/persona.rs:277-319` |
| T17 | No test for CRLF (`\r\n`) input end to end — the code has three separate `\r` handling branches (`crates/buzz-persona/src/persona.rs:283`, `:299`, `:314`) but every fixture uses `\n` | fixtures throughout `crates/buzz-persona/src/persona.rs:334-644` |

---

### 5. Deprecated / legacy API usage

| # | Item | Evidence |
|---|---|---|
L1 | **`serde_yaml` 0.9** is the YAML backend. The upstream crate was archived/deprecated by its author; no maintained successor is pinned here. | `crates/buzz-persona/Cargo.toml:12` |
| L2 | **`respond_to` legacy alias is carried in three places** with no deprecation timeline: persona frontmatter (`crates/buzz-persona/src/persona.rs:186`), manifest defaults (`crates/buzz-persona/src/manifest.rs:62`), and the validator's known-keys list with the comment `// legacy alias — still accepted` (`crates/buzz-persona/src/validate.rs:124`). | as cited |
| L3 | **`Engines.buzz` carries `alias = "buzz"`, which aliases the field to its own name** — a no-op serde attribute. | `crates/buzz-persona/src/manifest.rs:39` |
| L4 | **Doc comment references "V7 spec" and "(V3 contract)" version labels** that do not correspond to anything in the repo (`PERSONA_PACK_SPEC.md` carries no version number; it documents migration *from* V6). | `crates/buzz-persona/src/persona.rs:95` ("V7 spec"), `crates/buzz-persona/src/resolve.rs:208` and `crates/buzz-persona/tests/integration.rs:389` ("V3 contract") |
| L5 | **PR-relative comments baked into source**: "no interpolation in this PR" (`crates/buzz-persona/src/resolve.rs:68`), "Since hooks are not executed in this PR" (`crates/buzz-persona/src/resolve.rs:344`), "// Fix #1:" / "// Fix #4:" (`crates/buzz-persona/src/persona.rs:230`, `:263`), and a test module titled "env var flow introduced in PRs #783 and #794" (`crates/buzz-persona/tests/e2e_env_flow.rs:1-2`). These lose meaning once the PR is merged. | as cited |
| L6 | **`Cargo.toml` metadata is stale and inconsistent with the workspace.** `repository = "https://github.com/block/sprout"` (pre-rename) and hard-coded `version`/`edition`/`license` instead of `workspace = true` inheritance used by sibling crates (compare `crates/buzz-acp/Cargo.toml:3-7`). | `crates/buzz-persona/Cargo.toml:1-8` |
| L7 | **`buzz-agent`/`sprout-agent` rename residue**: this crate hard-codes `"buzz-agent"` as the runtime discriminant (`crates/buzz-persona/src/resolve.rs:374`) while the desktop layer still migrates legacy `"sprout-agent"` values (`desktop/src-tauri/src/migration.rs:1102`, `:1128`) using this crate's `split_frontmatter`. The legacy value is not recognized here, so a legacy persona silently gets GOOSE_* env vars. | as cited |

---

### 6. Design-level debt

| # | Item | Evidence |
|---|---|---|
| A1 | **The merge layer is untyped.** `PersonaConfig` is serialized to `serde_json::Value` and merged as loose JSON (`crates/buzz-persona/src/pack.rs:406-409`, `crates/buzz-persona/src/merge.rs:47-83`). Consequences: (a) the merge silently tolerates wrong types via `and_then(as_f64/as_bool/as_u64)` (`crates/buzz-persona/src/merge.rs:96-97`, `:159-168`); (b) the merge depends on serde field *names* matching, which has already caused a real bug — the regression test at `crates/buzz-persona/src/merge.rs:400-403` documents "This was broken when BehavioralDefaults serialized the field as \"respond_to\" but merge looked for \"triggers\"". | as cited |
| A2 | **Serde-name coupling is load-bearing and unenforced.** Nothing prevents a future rename of a `PersonaConfig` or `BehavioralDefaults` field from silently breaking merge lookups again — the string keys in `merge.rs` (`"model"`, `"temperature"`, `"subscribe"`, `"triggers"`, `"thread_replies"`, `"broadcast_replies"`) are literals with no compile-time link to the structs. | `crates/buzz-persona/src/merge.rs:95-168` |
| A3 | **`resolve_persona_by_name` re-reads and fully re-resolves the entire pack to return one persona.** | `crates/buzz-persona/src/resolve.rs:186-192` |
| A4 | **Tri-state `subscribe` semantics are carefully built then thrown away.** `ResolvedConfig.subscribe: Option<Vec<String>>` distinguishes absent from "subscribe to nothing" (`crates/buzz-persona/src/merge.rs:29-31`), but `LoadedPersona.subscribe` collapses it with `unwrap_or_default()` (`crates/buzz-persona/src/pack.rs:432`), so downstream consumers cannot tell the two apart. | as cited |
| A5 | **Validation reports only the first structural error.** `validate_pack` returns immediately when `load_pack` fails (`crates/buzz-persona/src/validate.rs:148-151`), so a pack with five broken personas requires five validate-fix cycles. The test suite acknowledges this (`crates/buzz-persona/tests/integration.rs:296-299`). | as cited |
| A6 | **`ValidationReport::exit_code()` is never used by the only consumer.** `buzz-cli` re-derives its own exit behavior from `has_errors()`/`has_warnings()` (`crates/buzz-cli/src/commands/pack.rs:37-44`), and its mapping differs from the documented contract — the crate documents `2 = warnings only` (`crates/buzz-persona/src/validate.rs:62`) but the CLI returns success for warnings. | as cited |
| A7 | **`Display for ValidationReport` is never used either** — `buzz-cli` iterates `report.diagnostics` and formats its own output (`crates/buzz-cli/src/commands/pack.rs:26-35`), duplicating the `ERROR:`/`WARN:` prefixes already implemented at `crates/buzz-persona/src/validate.rs:26-31`. | as cited |
| A8 | **The crate's own 1,190-line spec (`PERSONA_PACK_SPEC.md`) documents behavior that mostly lives in other crates**, and marks six features as planned (PF-1…PF-6 at `crates/buzz-persona/PERSONA_PACK_SPEC.md:1136-1145`). It is not linked from any repo-level doc, so it will drift silently. | as cited |
| A9 | **Error-source chains are flattened at module boundaries**, so a caller sees a string, not a typed cause: `PackError::ManifestParse(e.to_string())` (`crates/buzz-persona/src/pack.rs:58`) and `FileParse { reason: e.to_string() }` (`crates/buzz-persona/src/pack.rs:395`). | as cited |

---

### 7. Missing documentation

| # | Gap | Evidence |
|---|---|---|
| M1 | **`ARCHITECTURE.md` does not mention this crate at all.** A grep for both `buzz-persona` and `persona` (case-insensitive) across `ARCHITECTURE.md` returns **zero** matches. The crate is absent from the dependency diagram at `ARCHITECTURE.md:78-95`, absent from the "Agent surface" grouping alongside `buzz-acp`/`buzz-cli`, and has no per-crate section (unlike `buzz-core` at `ARCHITECTURE.md:332`, `buzz-acp` at `:644`, `buzz-admin` at `:678`, etc.). | verified by grep |
| M2 | **`CONTRIBUTING.md` does not mention it either** (zero matches for `persona`). So there is no documented guidance on adding a persona field, adding a validation rule, or the pack-schema change process — despite `CONTRIBUTING.md` covering the equivalent for event kinds, CLI subcommands, and HTTP endpoints per `AGENTS.md`. | verified by grep |
| M3 | Existing coverage is one line each: `AGENTS.md:52` (`buzz-persona        # Agent persona packs`) and `README.md:209` (`· \`buzz-persona\` (agent persona packs)`). Neither describes the pack schema, the precedence model, or the resolve pipeline. | as cited |
| M4 | **`crates/buzz-persona/src/lib.rs` has no crate-level doc comment** — no `//!` header, so `cargo doc` renders a bare module list with no orientation. | `crates/buzz-persona/src/lib.rs:1-6` |
| M5 | **`pack.rs` and `merge.rs` use `///` where `//!` was intended.** The file-opening comments (`crates/buzz-persona/src/pack.rs:1-19`, `crates/buzz-persona/src/merge.rs:1-9`) are outer doc comments that attach to the following item instead of documenting the module — so `pack`'s directory-layout diagram documents the `use` statement, and `merge`'s precedence explanation documents `struct TriggersData`. | as cited |
| M6 | **No `TESTING.md` coverage.** `TESTING.md` has zero `persona` matches, so the pack-fixture testing approach (`tempfile` + hand-built pack dirs) is undocumented for contributors. | verified by grep |
| M7 | **The `PERSONA_PACK_SPEC.md` ↔ code relationship is undocumented.** The spec describes buzz-acp and desktop behavior extensively but never states which sections this crate actually implements; readers must diff it against the code (as done in the features doc for this module). | `crates/buzz-persona/PERSONA_PACK_SPEC.md` |
| M8 | **No CHANGELOG entries reference the crate.** `CHANGELOG.md` has 32 `persona` matches, all desktop/UI or test-stability entries (e.g. `CHANGELOG.md:228`, `:313`, `:361`) — none describe pack-schema or `buzz-persona` library changes, so schema evolution is untracked. | verified by grep |

---

### 8. Debt signal summary

| Category | Count | Highest-signal item |
|---|---|---|
| Complexity hotspots | 3 functions >70 lines | `load_pack` ~123 lines with 6 responsibilities (`crates/buzz-persona/src/pack.rs:125`) |
| Dead / orphaned code | 11 (D1–D11) | `resolve_skills` fully implemented and tested but never called (`crates/buzz-persona/src/pack.rs:249`) |
| Duplication | 8 (U1–U8) | persona-name validation implemented twice with a const vs. an inline literal (`crates/buzz-persona/src/resolve.rs:146` vs `crates/buzz-persona/src/validate.rs:168`) |
| Test coverage gaps | 17 (T1–T17) | three public functions with zero direct tests (`parse_persona_file`, `parse_manifest_file`, `resolve_loaded_pack`) |
| Deprecated / legacy usage | 7 (L1–L7) | `serde_yaml` 0.9 (unmaintained upstream) at `crates/buzz-persona/Cargo.toml:12` |
| Design-level debt | 9 (A1–A9) | untyped JSON merge layer that has already produced one field-name regression (`crates/buzz-persona/src/merge.rs:400-403`) |
| Missing documentation | 8 (M1–M8) | crate entirely absent from `ARCHITECTURE.md` and `CONTRIBUTING.md` |
| TODO/FIXME/HACK/XXX markers | **0** | deferred work is expressed as prose comments instead (see features doc) |
| `unsafe` blocks | **0** | — |
| `unwrap()`/`expect()` in production paths | **0** | — |


## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: Debt

Recorded factually, without severity judgement. Total surface: 541 LOC across 4
`.rs` files (`connection.rs` 314, `message.rs` 167, `error.rs` 51, `lib.rs` 9) plus a
17-line manifest.

---

### 1. Complexity hotspots

| Hotspot | Detail | file:line |
|---|---|---|
| Three near-identical frame-dispatch loops | `recv_one`, `wait_for_auth_challenge`, and `wait_for_ok` each contain the same `timeout(... self.ws.next())` + `match raw { Text / Ping / Close / _ }` skeleton, differing only in the `Text` arm and the timeout error | `connection.rs:133`–`154`, `:178`–`214`, `:235`–`268` |
| Duplicated Ping/Pong and Close handling | Same two arms written three times | `connection.rs:148`–`151`, `:208`–`:211`, `:262`–`:265` |
| Duplicated buffer-scan-then-remove pattern | `position(...)` followed by `remove(idx).unwrap()` and an `unreachable!()` arm, written twice with different predicates | `connection.rs:165`–`174`, `:224`–`:233` |
| Three challenge-intake paths with divergent validation | `pending_challenge.take()`, buffered `Auth`, and socket-read `Auth`; only the third applies the 1024-byte cap | `connection.rs:161`, `:170`, `:198` |
| Two timeout semantics in one file | `recv_one` applies `timeout_dur` **per read** inside its loop, while the two waiters use an absolute deadline; both are described to callers as "waiting up to" a duration | `connection.rs:133`–`134` vs `:176`–`185`, `:222`–`:242` |
| Redundant buffer drain | `next_event` pops the buffer, then `recv_one` pops it again as its first act | `connection.rs:108`–`111` and `:129`–`131` |
| Dual bookkeeping of a mid-flight `AUTH` | `wait_for_ok` stores the challenge in `pending_challenge` **and** pushes the same message onto `buffer`, so the same challenge is observable through two channels and can be consumed twice (once by `authenticate`, once by `next_event`) | `connection.rs:255`–`258` |
| Raw-vs-normalized URL split | The dialed URL is normalized (`parsed.as_str()`), the AUTH-event relay tag uses the original string | `connection.rs:53` vs `:63`, `:79` |

---

### 2. Dead / unused code

| Item | Status | file:line |
|---|---|---|
| `WsClientError::EventRejected(String)` | Never constructed anywhere in the repo. `send_event` returns `Ok(OkResponse { accepted: false, .. })` instead. Only `buzz-test-client` maps the variant across to its own error type (`crates/buzz-test-client/src/lib.rs:71`) | decl `error.rs:38`–`40`; non-use `connection.rs:96`–`101` |
| `From<nostr::event::builder::Error> for WsClientError` | No call site inside the crate — `build_auth_event` performs the same conversion inline with `map_err` rather than `?`, so the impl is only reachable by external callers | impl `error.rs:47`–`51`; inline duplicate `message.rs:189` |
| `WsClientError::UnexpectedMessage` for a *state* violation | The doc comment says "a message that was not expected at this point" (`error.rs:30`), but every construction site is a *parse* failure in `message.rs`; no state-machine violation ever produces it | `error.rs:30`–`32` vs `message.rs:68`, `:75`, `:80`, `:91`, `:109`, `:119`, `:143`, `:163` |
| `#[allow(clippy::result_large_err)]` | Suppression on `parse_relay_message` indicates the error enum is large (it wraps `tungstenite::Error`), left unaddressed rather than boxed | `message.rs:61` |
| Two `unwrap()` + `unreachable!()` pairs | Contradicts the repo rule against new `unwrap()` in production paths (`AGENTS.md`, Quality Gates); provably unreachable given the preceding `position` check, but still panic sites in a library | `connection.rs:170`, `:172`, `:229`, `:231` |

---

### 3. Test coverage gaps

Shipped tests: three compile-time constant-floor assertions
(`connection.rs:300`–`313`). No `tests/` directory, no `[dev-dependencies]`
(`Cargo.toml` has no such section), no `#[tokio::test]`, no mock relay.

| Untested behaviour | Where it lives |
|---|---|
| `parse_relay_message` — all **seven** message types (six at report time; the `"COUNT"` arm added by the sync is likewise untested), missing/short arrays, wrong JSON types, unknown message type | `message.rs:62`–`167`; `COUNT` arm `:147`–`162` |
| `OK` lenient defaults (`accepted` → `false`, `message` → `""`) | `message.rs:93`–`98` |
| `build_auth_event` — tag injection vs. no tag, invalid relay URL | `message.rs:174`–`190` |
| The whole NIP-42 handshake, including the `accepted == false` rejection path | `connection.rs:70`–`93` |
| 1024-byte challenge cap (and the two paths that bypass it) | `connection.rs:198`; bypasses at `:161`, `:170` |
| Buffering / out-of-order delivery, `OK` correlation by event id | `connection.rs:205`, `:227`, `:254`, `:259` |
| Timeout expiry → `NoAuthChallenge` / `Timeout`, deadline recomputation | `connection.rs:183`–`189`, `:240`–`:246` |
| Ping→Pong keepalive, `Close`/stream-end → `ConnectionClosed` | `connection.rs:148`–`151` (×3) |
| `publish_event` outer-budget behaviour and the discarded `disconnect` result | `connection.rs:284`–`293` |
| URL parsing/normalization and plaintext-`ws://` acceptance | `connection.rs:48`–`55` |

The crate's behaviour is exercised only indirectly, through `buzz-test-client`'s
integration suite (`crates/buzz-test-client/src/lib.rs:85`, `:98`, `:123`, `:154`,
`:175`), which requires a live relay and therefore does not run under `just test-unit`.

---

### 4. Duplication with other implementations in the repo

| Duplicate | Detail |
|---|---|
| `buzz-acp` reimplements this crate | `crates/buzz-acp/src/relay.rs` has its own `connect_async`, its own `RelayMessage` with an `Auth { challenge }` variant (`:492`, `:2344`), its own AUTH-message parse arm (`:3610`–`3616`), its own `send_auth_response` (`:3435`–`3461`), its own `wait_for_auth_challenge` (`:3864`) and handshake (`:3843`–`3845`), its own `"No auth challenge received"` error (`:447`), and its own `AUTH_TIMEOUT` — and it does **not** depend on `buzz-ws-client` |
| `buzz-acp` diverges functionally | It handles **mid-session AUTH challenges by re-authenticating** (`crates/buzz-acp/src/relay.rs:2344`–`2350`) and notes that `EventBuilder::auth()` cannot carry extra tags so it builds the event manually (`:3444`–`3456`) — whereas `buzz-ws-client` uses `EventBuilder::auth` plus `builder.tags([...])` (`message.rs:181`–`186`) and only records mid-session challenges (`connection.rs:255`–`258`) |
| `buzz-test-client` re-wraps rather than re-exports the connection | It defines its own wrapper type around `NostrWsConnection` and mirrors every `WsClientError` variant into a parallel `TestClientError` enum (`crates/buzz-test-client/src/lib.rs:65`–`72`, `:85`–`98`), so error variants must be kept in sync by hand |
| Other independent WebSocket clients | `crates/buzz-relay/src/router.rs`, `crates/buzz-relay/src/audio/handler.rs`, `crates/buzz-pairing-cli/src/main.rs`, `crates/buzz-pair-relay/tests/integration.rs`, `crates/buzz-test-client/tests/nip42_host_binding_live.rs`, `crates/buzz-test-client/tests/conformance_multitenant.rs`, `desktop/src-tauri/src/native_websocket.rs`, `desktop/src-tauri/src/huddle/relay_api.rs`, `desktop/src-tauri/src/commands/pairing.rs` all call `connect_async` directly rather than going through this crate (relay-side and test-side cases are inbound/other-protocol, so not all are candidates for consolidation) |
| Timeout knowledge duplicated in a caller | `buzz-cli` hardcodes `75` for the outer budget with a comment pointing at the three constants, so the relationship is documented in prose rather than derived in code | `crates/buzz-cli/src/client.rs:1077`, `:1080` |

---

### 5. Documentation gaps

| Gap | Evidence |
|---|---|
| **`buzz-ws-client` is absent from `ARCHITECTURE.md`'s crate reference.** The crate map lists `buzz-core`, `buzz-db`, `buzz-auth`, `buzz-pubsub`, `buzz-search`, `buzz-audit`, `buzz-workflow`, `buzz-relay`, `buzz-acp`, `buzz-sdk`, `buzz-media`, `buzz-cli`, `buzz-admin`, `buzz-test-client` — and no entry for `buzz-ws-client`; a repo-wide grep for the name in `*.md` matches only `AGENTS.md:63` | `ARCHITECTURE.md:78`–`94`; `AGENTS.md:63` |
| No module-level (`//!`) docs on any file, so `cargo doc` for the crate root shows only the re-export list | `lib.rs:1`–`9` |
| No `README.md` in the crate directory | `crates/buzz-ws-client/` contains only `Cargo.toml` and `src/` |
| No `CHANGELOG` entry or migration note for the mid-session-AUTH limitation, which callers must know about to stay authenticated | `connection.rs:255`–`258` |
| Doc/behaviour mismatch: `next_event` is documented as "waiting up to `timeout_dur`" but the underlying `recv_one` restarts the timer per frame | doc `connection.rs:103`; impl `connection.rs:133`–`134` |
| Doc/behaviour mismatch: `UnexpectedMessage`'s doc describes a protocol-state error; all uses are parse errors | `error.rs:30` vs `message.rs` (8 sites) |

---

### 6. Missing capabilities that callers must re-implement

| Missing | Consequence | Evidence |
|---|---|---|
| No reconnect/retry/backoff | Every `ConnectionClosed` is terminal for the caller; `publish_event` does a full fresh connect per call | `connection.rs:277`–`294`; no retry code in the file |
| No typed `REQ`/`CLOSE`/`COUNT` | Subscription framing is hand-built by each caller through `send_raw` | `connection.rs:121`; example `crates/buzz-test-client/src/lib.rs:154`, `:160` |
| No `COUNT` response variant | A `COUNT` reply surfaces as `UnexpectedMessage` | `message.rs:70`–`165` |
| No authenticated-state tracking | Nothing distinguishes an authenticated from an unauthenticated connection | `connection.rs:70`–`93` (no flag written) |
| No buffer capacity bound | `buffer` can grow for the duration of a wait | `connection.rs:28`, `:205`, `:257`, `:259` |
| No `Debug` impl on `NostrWsConnection` | Callers embedding it cannot derive `Debug` | `connection.rs:26` |

---

### 7. Deprecated API usage

None found. No `#[deprecated]` items are defined or referenced, and no compiler
deprecation shims appear in the source. Dependency versions in use are recent
workspace pins: `nostr` 0.44 (root `Cargo.toml:61`), `tokio-tungstenite` 0.29 (root
`Cargo.toml:113`), `thiserror` 2 (root `Cargo.toml:85`). The
`Message::Text(text.into())` conversion at `connection.rs:124` is the current
tungstenite 0.29 API shape (it requires an owned UTF-8 payload type rather than a
`String`), not a deprecated form.


## Module: buzz-db (`crates/buzz-db`)

### Aspect: Debt

#### 1. Schema drift: `migrations/` vs `schema/schema.sql`

`schema/schema.sql` (1016 lines) is described in its own header as the "source of
truth for fresh database setup" (`schema/schema.sql:3`), but nothing in the crate
reads it — the runner embeds `migrations/` only
(`crates/buzz-db/src/migration.rs:11`). It has drifted substantially. Verified
absences (grep count 0 in `schema/schema.sql`):

| Object | Created by | Missing from `schema/schema.sql` |
|--------|-----------|----------------------------------|
| table `git_repo_names` + `idx_git_repo_names_owner` | `migrations/0002_git_repo_names.sql:20`, `:29` | yes |
| column `communities.icon` | `migrations/0003_community_icon.sql:12` | yes |
| index `idx_events_tags_gin` (GIN `jsonb_path_ops`) | `migrations/0004_events_tags_gin.sql:21` | yes |
| table `parameterized_event_watermarks` | `migrations/0007_nip_rs_retention.sql:14` | yes |
| index `idx_event_mentions_community_event` | `migrations/0007_nip_rs_retention.sql:26` | yes |
| function+trigger `guard_nip_rs_watermark` / `trg_events_nip_rs_watermark` | `migrations/0009:11,70`, replaced `0010:4`, `0011:62` | yes |
| function+trigger `purge_soft_deleted_nip_rs` | `migrations/0009:77,104`, replaced `0011:123` | yes |
| function+trigger `guard_event_mention_live` | `migrations/0009:111,133` | yes |
| function+trigger `guard_nip_rs_hard_delete` | `migrations/0011:45,56` | yes |
| function+trigger `purge_soft_deleted_buzz_mesh_status` | `migrations/0019:21,41` | yes |
| table `product_feedback` (+2 indexes, + allowlist row) | `migrations/0017_product_feedback.sql:5` | yes |
| table `join_policy_acceptances` | `migrations/0020_join_policy_acceptances.sql:4` | yes |

`schema/schema.sql` also encodes an `events.search_tsv` expression that matches
**neither** end state produced by the migrations:

```sql
    search_tsv  TSVECTOR GENERATED ALWAYS AS (
        CASE WHEN kind IN (1059, 30300, 30350, 30622, 44100, 44101, 44200) THEN NULL::tsvector
             ELSE to_tsvector('simple', content)
        END
    ) STORED,
```
(`schema/schema.sql:196-200`, with the comment "Keep in sync with migrations
(final state: 0001 + 0005 + 0009)" at `:194` — the relevant migration is `0014`,
not `0009`). Migrations produce either the brownfield negative list
(`0001`+`0005`+`0014` ⇒ `{1059, 30300, 30622, 44100, 44101, 44200}` ∪ `{30350}`)
or, for an empty database, the positive allowlist from `0008`
(`{0, 9, 40002, 45001, 45003}` minus `30350`). The divergence between fresh and
brownfield installs is itself latent debt: the same relay binary can be running
two different search policies (asserted, not fixed, by
`crates/buzz-db/src/migration.rs:1075-1109`), and the remediation is an
out-of-band operator script named at
`migrations/0008_fresh_install_search_allowlist.sql:3-4`
(`scripts/maintenance/nip_rs_search_allowlist.sql`).

`schema/schema.sql` does contain a few things migrations also have (moderation
tables, push lease/queue tables, the `0021`/`0023`/`0024` functions), so it is a
partially-updated snapshot rather than a stale copy — which makes it more likely
to be trusted by mistake. There is no test asserting the two files agree.

#### 2. Complexity hotspots

File sizes (measured; prod = lines before `#[cfg(test)]`):

| File | Total | Prod | Test | Test share |
|------|------:|-----:|-----:|-----------:|
| `lib.rs` | 6106 | 3916 | 2190 | 36% |
| `event.rs` | 2465 | 1428 | 1037 | 42% |
| `push.rs` | 2388 | 1261 | 1127 | 47% |
| `workflow.rs` | 2276 | 1197 | 1079 | 47% |
| `channel.rs` | 1896 | 1415 | 481 | 25% |
| `thread.rs` | 1730 | 809 | 921 | 53% |
| `migration.rs` | 1248 | 97 | 1151 | 92% |
| `relay_members.rs` | 953 | 555 | 398 | 42% |
| `moderation.rs` | 895 | 629 | 266 | 30% |
| `feed.rs` | 825 | 248 | 577 | 70% |
| `usage.rs` | 723 | 356 | 367 | 51% |
| `user.rs` | 674 | 400 | 274 | 41% |
| `replica_fence.rs` | 662 | 507 | 155 | 23% |
| `dm.rs` | 558 | 518 | 40 | 7% |
| `api_token.rs` | 523 | 326 | 197 | 38% |
| `reaction.rs` | 419 | 419 | 0 | 0% |
| `git_repo.rs` | 381 | 181 | 200 | 52% |
| `admin_moderation.rs` | 231 | 231 | 0 | 0% |
| `archived_identities.rs` | 222 | 126 | 96 | 43% |
| `product_feedback.rs` | 189 | 119 | 70 | 37% |
| `partition.rs` | 183 | 151 | 32 | 17% |
| `error.rs` | 55 | 55 | 0 | 0% |
| **total** | **25602** | **14944** | **10658** | **42%** |

`lib.rs` at 6106 lines is the single biggest structural problem: it is
simultaneously the crate root, the `Db` façade with **215** delegating methods,
the community-registry repository, the replaceable-event write engine
(`replace_addressable_event`, `replace_parameterized_event`,
`publish_nip43_membership_locked`), the allowlist repository, the api-token
partial repository, and a 2190-line test module. Nothing in the crate's own
module convention explains why community/allowlist/token/replacement SQL lives
in the root rather than in `community.rs` / `allowlist.rs` / `api_token.rs` /
`replace.rs`.

Longest functions (brace-matched):

| Lines | Location | Notes |
|------:|----------|-------|
| 214 | `crates/buzz-db/src/lib.rs:3628` `replace_parameterized_event` | classification + two ordering sources + conditional hard/soft delete + watermark upsert in one body |
| 208 | `crates/buzz-db/src/event.rs:318` `query_events` | 14 optional predicates — **now 15**: `ab3af828` appended the kind-30175 persona visibility clause (`:504-527`), taking the `if let Some(…) = q.…` block count from 11 to 12; `col_prefix` string threaded through 20 `format!` calls |
| 185 | `crates/buzz-db/src/thread.rs:565` `get_channel_window` | SQL assembly + `has_more` probe + cursor capture + batched participant hydration |
| 183 | `crates/buzz-db/src/push.rs:213` `accept_lease_event` | three locks + five rejection paths + two inserts |
| 170 | `crates/buzz-db/src/push.rs:619` `enqueue_wakes` | four-phase set-wise protocol with three intermediate hash maps |
| 168 | `crates/buzz-db/src/event.rs:1045` `insert_event_with_thread_metadata_tx` | 5-deep nesting (`if was_inserted` → `if let Some(meta)` → `if rows_affected` → `if let Some(pid)` → `if let Some(root_id)` → `if root_id != pid`) |
| 142 | `crates/buzz-db/src/thread.rs:345` `get_thread_replies` | manual positional-parameter bookkeeping (`param_idx`) mirrored in two places |
| 140 | `crates/buzz-db/src/event.rs:598` `count_events` | **near-duplicate** of `query_events`' predicate block |
| 125 | `crates/buzz-db/src/lib.rs:3306` `replace_addressable_event` | |
| 124 | `crates/buzz-db/src/thread.rs:116` `insert_thread_metadata` | **near-duplicate** of the `_tx` variant in `event.rs` |

Duplication that has to be maintained in lockstep:

1. `query_events` (`crates/buzz-db/src/event.rs:318`) and `count_events`
   (`:598`) repeat the same ~10 predicate blocks with the same `col_prefix`
   trick; `count_events` silently omits the composite-cursor branch and the
   `LIMIT/OFFSET`, so the two can drift. **The drift widened at `07d0265c`**:
   the kind-30175 persona-visibility clause added by `ab3af828` exists only in
   `query_events` (`crates/buzz-db/src/event.rs:517-527`, driven by the new
   `EventQuery::persona_reader` field at `:88`) — `persona_reader` is never read
   inside `count_events` (`:598-737`; grep for `persona_reader` in the file
   returns only `:88`, `:117`, `:517`). Not currently a leak, because the relay
   forces any 30175-matching filter off the `count_events` fast path and counts
   through the `query_events`-backed fallback instead (BR-WS-124 / BR-WS-126a in
   `business-rules.md`), but it is exactly the lockstep-maintenance hazard this
   item describes: the two functions now differ by a *security* predicate, not
   just a pagination one.
2. `thread::insert_thread_metadata` (`crates/buzz-db/src/thread.rs:116-247`) and
   `event::insert_event_with_thread_metadata_tx`'s inner block
   (`crates/buzz-db/src/event.rs:1138-1214`) contain the same stub-insert +
   counter-bump logic, written twice.
3. `thread::increment_reply_count` (`crates/buzz-db/src/thread.rs:251-289`) is a
   third copy of just the counter half — and is dead (see §3).
4. `Db` wrappers repeat every module signature; 20 of them carry
   `#[allow(clippy::too_many_arguments)]` on both sides of the boundary
   (e.g. `crates/buzz-db/src/lib.rs:1437` and `crates/buzz-db/src/channel.rs:86`).
5. The push-eligible kind allowlist `{7, 9, 1059, 40007, 46010}` exists in three
   places: `migrations/0018_push_match_queue.sql:25`,
   `migrations/0023_push_match_gate.sql:26`, and
   `crates/buzz-db/src/push.rs:58` — with only a comment
   ("Keep this allowlist identical to the relay's validated NIP-PL descriptor")
   holding them together.
6. The FTS kind lists are inlined in four migrations plus `schema/schema.sql`,
   with the sync obligation stated in prose
   (`migrations/0001_initial_schema.sql:214-221`).
7. The NIP-RS exact-cardinality predicate is implemented once in Rust
   (`crates/buzz-db/src/lib.rs:3672-3687`) and three times in plpgsql
   (`migrations/0011_nip_rs_exact_tag_cardinality.sql:66-88`, `:126-148`, and the
   `DELETE` classifier at `:12-40`).
8. `ChannelRecord` is mapped by two separate row mappers with slightly different
   column expectations: `crates/buzz-db/src/channel.rs:983` (full projection) and
   `crates/buzz-db/src/dm.rs:481` (DM projection that never selects
   `ttl_seconds`/`ttl_deadline` and relies on `unwrap_or(None)`).

#### 3. Dead / unreachable code

| Item | Evidence |
|------|----------|
| `thread::increment_reply_count` | carries `#[allow(dead_code)]` and a doc comment saying the real path is inlined elsewhere — `crates/buzz-db/src/thread.rs:243-289` |
| `workflow::create_workflow` | "(No current callers.)" — `crates/buzz-db/src/workflow.rs:275` |
| `workflow::update_workflow` | "(No current callers.)" — `crates/buzz-db/src/workflow.rs:619` |
| `workflow::update_workflow_status` | "(No current callers.)" — `crates/buzz-db/src/workflow.rs:653` |
| `workflow::set_workflow_enabled` | "(No current callers.)" — `crates/buzz-db/src/workflow.rs:683` |
| `workflow::delete_workflow` | "(No current callers.)" — `crates/buzz-db/src/workflow.rs:714` |
| `reaction::get_reactions`' `_cursor` parameter | "reserved for future keyset pagination (currently unused)" — `crates/buzz-db/src/reaction.rs:279-286`. The public signature advertises pagination it does not implement. |
| `event::D_TAG_MAX_LEN` | declared but never compared against anything — `crates/buzz-db/src/event.rs:138-140` |
| `Db::update_token_last_used` | pure alias for `touch_api_token` — `crates/buzz-db/src/lib.rs:2377-2384` |
| `communities.signing_key` | column exists, no accessor in this crate — `migrations/0001_initial_schema.sql:56` |
| `channels.max_members`, `channels.nip29_group_id`, `users.okta_user_id`, `users.metadata_event_id`, `users.deactivated_at`, `users.agent_type`, `users.capabilities` | surfaced in records but never written by any function in this crate (only read) |
| `thread_metadata.parent_event_created_at` / `root_event_created_at` | written on insert but never used in a WHERE/JOIN by any read in this crate |

#### 4. Tables with schema but no code

`subscriptions`, `delivery_log`, `audit_log`, `rate_limit_violations`, and all six
`push_gateway_*` tables have DDL in `migrations/` but **no** read or write path in
buzz-db. For `delivery_log` this is doubly odd: the crate manages its partitions
(`crates/buzz-db/src/partition.rs:12`) and its PK is the single hand-maintained
exception in the tenant-isolation lint
(`crates/buzz-db/src/migration.rs:497-501`), yet nothing in the crate ever inserts
a row. `ARCHITECTURE.md:793` describes it as "(partitioned; Rust module pending)".
`subscriptions` carries a full filter/delivery model that nothing populates.

#### 5. Test coverage gaps

Overall the crate is heavily tested (203 test functions, 42% of its lines), but
the distribution is uneven and **121 of 122** async tests are `#[ignore]`-gated,
so `cargo test` with no database exercises only the 81 pure `#[test]`s plus one
lazy-pool wiring test.

| Gap | Evidence |
|-----|----------|
| `reaction.rs` has **no** `mod tests` at all — the three-way upsert semantics are covered only indirectly, from `event.rs` and `lib.rs` | `crates/buzz-db/src/reaction.rs` (419 lines, 0 tests) |
| `admin_moderation.rs` has **no** tests — including the keyset-cursor SQL and the `bounded_limit` clamp | `crates/buzz-db/src/admin_moderation.rs` (231 lines, 0 tests) |
| `error.rs` has no tests (low risk: pure `Display` impls) | `crates/buzz-db/src/error.rs` |
| `dm.rs` has 4 pure tests and **0** database tests; participant-hash collision behaviour, hide/unhide, and the 9-participant ceiling are unexercised against Postgres | `crates/buzz-db/src/dm.rs:520+` |
| `channel.rs` has the lowest test share of the big modules (25%); `add_member`'s role matrix has no test for the private-channel elevated-grant refusal, and `update_channel`'s dynamic-SET builder has no test | `crates/buzz-db/src/channel.rs:1417+` (7 tests) |
| `partition.rs`: validators are unit-tested but `ensure_future_partitions`/`ensure_partition` have **no** database test, including the `42P17` overlap-tolerance branch | `crates/buzz-db/src/partition.rs:153-182` |
| `usage.rs`'s `active_user_counts`/`active_channel_counts` interval interpolation has no test asserting the produced SQL | `crates/buzz-db/src/usage.rs:254-341` |
| `api_token.rs`: only 2 tests; the 10-token quota boundary and expiry-aware "active" definition are not directly asserted | `crates/buzz-db/src/api_token.rs:328+` |
| No test asserts `schema/schema.sql` matches the migration end state | — |
| No test asserts the three copies of the push kind allowlist agree (unlike the moderation vocabulary, which *is* asserted at `crates/buzz-db/src/migration.rs:640-645`) | — |
| Tests share one database by default and rely on unique hosts for isolation; one test explicitly documents that a shared fixed pubkey picks up rows leaked by sibling ignored tests | `crates/buzz-db/src/lib.rs:4879-4884` |

#### 6. Migration-history debt

- **24 checksum-frozen files.** Because `sqlx::migrate!` pins checksums, four of
  the 24 migrations exist purely to patch earlier ones:
  `0010` and `0011` rewrite `0009`'s function bodies; `0023` rewrites `0018`'s;
  `0024` rewrites `0022`'s. Reading the current behaviour of
  `guard_nip_rs_watermark`, `purge_soft_deleted_nip_rs`,
  `enqueue_push_match_job`, or `refresh_channel_ttl_after_event_insert` requires
  reading up to three files and knowing which wins.
- `events.search_tsv` is dropped and re-added **three** times after `0001`
  (`0005`, `0008` conditionally, `0014`), each time rebuilding the GIN index. On a
  populated database `0005` rewrites the whole heap column.
- `0007` performs irreversible `DELETE`s of user payloads during startup
  migration (`migrations/0007_nip_rs_retention.sql:88-140`), guarded only by the
  pre-flight ambiguity check in `crates/buzz-db/src/migration.rs:34-96`.
- `0008` behaves differently depending on whether the table is empty, permanently
  bifurcating the deployment population.
- `0011` deletes rows from `parameterized_event_watermarks`
  (`migrations/0011_nip_rs_exact_tag_cardinality.sql:7-43`).
- `0004` builds a GIN index on a partitioned parent without `CONCURRENTLY`, which
  the file itself flags as requiring a deploy window
  (`migrations/0004_events_tags_gin.sql:13-17`).
- The lint harness that guarantees tenant isolation lives in `#[cfg(test)]`
  (`crates/buzz-db/src/migration.rs:99-830`) — it is a test, not a runtime or
  CI-independent gate, and its "operator-global" allowlist is a **hard-coded Rust
  array** (`crates/buzz-db/src/migration.rs:334-352`) that must be updated
  alongside the `_operator_global_tables` INSERTs; a new allowlisted table that
  someone forgets to add to the array will make the lint fail closed (safe), but
  a table added to the array without the DB row will silently exempt itself.

#### 7. Performance debt (self-documented)

| Issue | Evidence |
|-------|----------|
| `get_reactions_bulk` runs **one query per event** | "For typical message-list sizes (<=100 events) this is acceptable; a single-query approach … can be added later if needed" — `crates/buzz-db/src/reaction.rs:371-375` |
| `list_dms_for_user` runs **one participants query per DM** | `crates/buzz-db/src/dm.rs:308-334` |
| `query_events`' `ORDER BY created_at DESC, id ASC` has no covering index for the trailing column; Postgres sorts in memory | "No existing index covers this trailing column… If query performance degrades, add a composite index" — `crates/buzz-db/src/event.rs:530-534` |
| `usage.rs` event-derived aggregates are recurring partition scans | "At scale these can become recurring partition scans; if that becomes a problem, move them to a maintained rollup table" — `crates/buzz-db/src/usage.rs:6-10` |
| `get_members_bulk` has no pagination | "For large channel sets, consider pagination." — `crates/buzz-db/src/channel.rs:600-604` |
| `get_accessible_channel_ids` has **no LIMIT** (deliberately — a test asserts 1001 rows are returned) so the result set grows with the community | `crates/buzz-db/src/channel.rs:638-667`; test `crates/buzz-db/src/channel.rs:1751-1783` |
| `api_token::list_tokens_by_owner` has no `LIMIT` | `crates/buzz-db/src/api_token.rs:208-266` |
| FTS uses the `'simple'` configuration (no stemming/stopwords) as a compatibility choice, flagged for revisit | `migrations/0001_initial_schema.sql:198-201` |
| Two prior perf regressions are documented as fixed but leave the shape fragile: unindexed `tags @>` containment cost ~1.7 s/page (`migrations/0004_events_tags_gin.sql:5-12`) and the `0022` TTL row lock took commit latency from 0.07 ms to ~15 ms at 200 QPS (`migrations/0024_event_ttl_refresh_shared_lock.sql:1-9`) |

#### 8. API-shape debt

- **215 delegating methods on one `Db` struct.** Every new module function needs a
  matching wrapper; several wrappers are already inconsistent — `api_token`
  functions take `Uuid` while their `Db` wrappers take `CommunityId` and
  dereference (`crates/buzz-db/src/lib.rs:2283-2292`), whereas every other module
  takes `CommunityId` directly.
- Naming collisions between generic `Db` methods and specific ones:
  `Db::archive`/`Db::unarchive`/`Db::is_archived`/`Db::list_archived` are about
  *identities* (`crates/buzz-db/src/lib.rs:3238-3279`), while
  `Db::archive_channel` and `Db::archive_community_owned_by` are about other
  entities — the unqualified names are the least specific.
- Return-type inconsistency for "not found": `get_channel` returns
  `Err(ChannelNotFound)` (`crates/buzz-db/src/channel.rs:294`) while
  `get_user`/`get_relay_member`/`get_ban` return `Ok(None)`; `get_workflow`
  returns `Err(NotFound)` while `find_by_owner_and_name` returns `Ok(None)`.
- `ApiTokenRecord` exposes `token_hash` to every caller and relies on a doc
  comment to prevent leaking it (`crates/buzz-db/src/api_token.rs:201-206`).
- `EventQuery` has 18 fields with several mutually exclusive combinations
  validated at runtime rather than by type
  (`crates/buzz-db/src/event.rs:21-121`, checks at `:304-317`).

#### 9. Deprecated API usage

None found. No `#[deprecated]` items are defined or consumed; no
`#[allow(deprecated)]` appears in the crate. Dependency versions are current for
the workspace (sqlx 0.9, nostr 0.44, thiserror 2, sha2 0.11 — `Cargo.toml:52`,
`:61`, `:85`, `:96`), and the code uses the post-0.8 sqlx `AssertSqlSafe` API
rather than the older implicit-`&str` path, which is itself evidence of a recent
upgrade having been done properly.

#### 10. Risk summary

| Risk | Severity | Why |
|------|----------|-----|
| `schema/schema.sql` mistaken for authoritative | high | it claims to be the source of truth, is missing 12 objects, and encodes a `search_tsv` expression no migration produces |
| Fresh-vs-brownfield FTS policy divergence | medium | same binary, two search behaviours, remediation is a manual script |
| Kind-allowlist triplication (push) and FTS lists in frozen SQL | medium | a new privacy-sensitive kind silently becomes searchable / a new push kind silently stops waking devices |
| `lib.rs` size and mixed responsibilities | medium | 3916 production lines in the crate root, including the most intricate write paths |
| `query_events`/`count_events` duplication | medium | a filter fixed in one and not the other changes COUNT vs REQ semantics |
| 121/122 async tests gated behind `#[ignore]` | medium | the correctness argument for locking, triggers, and tenancy only holds if CI actually runs the gated suite |
| `delivery_log`/`subscriptions` schema without code | low | dead weight, plus a hand-maintained lint exception for a table nothing writes |
| `reaction.rs` / `admin_moderation.rs` with zero tests | low–medium | `admin_moderation` is the one module allowed to skip tenant scoping |
| `BUZZ_MAX_COMMUNITIES_PER_OWNER` absent from both `.env.example` files | low | added by `2a051a40` (`crates/buzz-db/src/relay_members.rs:391`) but never added to the root `.env.example` or `deploy/compose/.env.example` (grep count 0 in both), so the only description is the Rust doc comment at `:381-386` — operators cannot discover the knob from the canonical template |
| Owner cap cached for the process lifetime | low | `OnceLock` resolves on first use and is never invalidated (`crates/buzz-db/src/relay_members.rs:388-395`); raising the cap needs a relay restart, and there is no admin event or endpoint that re-reads it |
| **NEW (introduced by `2a051a40`)** — the two owner-cap tests do not read the effective limit | low | production now compares against `relay_members::max_communities_per_owner()` (`crates/buzz-db/src/lib.rs:901`, `relay_members.rs:501`), but `create_community_with_owner_enforces_per_owner_limit` still loops on the literal `3` (`crates/buzz-db/src/lib.rs:4863`) and `fresh_host_at_owner_limit_returns_limit_reached_conflict` loops on the *constant* `MAX_COMMUNITIES_PER_OWNER` rather than the function (`crates/buzz-relay/src/api/operator.rs:1086`). With `BUZZ_MAX_COMMUNITIES_PER_OWNER` set in the environment both tests fail spuriously — the env knob is untested end-to-end, and only the pure `effective_owner_limit` parse rules have coverage (`relay_members.rs:584-606`) |


## Module: buzz-auth (`crates/buzz-auth`)

### Aspect: Debt

### Dead / orphaned public API

Determined by repo-wide grep for each identifier, excluding `target/` and this
crate's own `#[cfg(test)]` modules.

| Item | Declared | Callers outside this crate | Notes |
|------|----------|---------------------------|-------|
| `ChannelAccessChecker` (trait) | `crates/buzz-auth/src/access.rs:31` | **zero** — the identifier does not appear anywhere else in the repo | The doc claims "Implemented by the database layer (`buzz-db`) in production" (`crates/buzz-auth/src/access.rs:18-19`). No `buzz-db` type implements it. The trait's entire tenant-scoping contract (`:22-30`) is unenforced because nothing implements it |
| `check_read_access` | `crates/buzz-auth/src/access.rs:72` | zero | tested (`:208-223`) but unreachable in production |
| `check_write_access` | `crates/buzz-auth/src/access.rs:88` | zero | untested directly; only its scope constant differs from the read variant |
| `require_scope` | `crates/buzz-auth/src/access.rs:60` | zero | the only scope-enforcement function in the crate has no production caller |
| `parse_scopes` | `crates/buzz-auth/src/scope.rs:170` | zero | tested (`:199-203`); contains the crate's only production-path `expect` |
| `Scope::all_non_admin()` | `crates/buzz-auth/src/scope.rs:94` | zero | the admin-excluding grant set is dead; only `all_known()` is used (`crates/buzz-relay/src/api/bridge.rs:829`) |
| `derive_pubkey_from_username` | `crates/buzz-auth/src/lib.rs:160` | zero — no caller anywhere, not even a test | Doc says it exists so "the relay can resolve usernames to Nostr pubkeys in dev mode" (`:149-151`); nothing does |
| `ip_rate_limit_key` | `crates/buzz-auth/src/rate_limit.rs:213` | one, in a code path with no caller: `crates/buzz-pubsub/src/rate_limiter.rs:118` inside `check_ip_connection`, which is never invoked in production |
| `LimitType::IpConnections` | `crates/buzz-auth/src/rate_limit.rs:66` | zero constructions repo-wide | the variant, its key suffix (`:75`), and the whole `check_ip_connection` method are effectively dead |
| `AuthMethod::Nip98` | `crates/buzz-auth/src/lib.rs:59` | zero constructions | the relay's HTTP path uses its own `HttpAuthMethod` enum instead (`crates/buzz-relay/src/handlers/ingest.rs:2570`, `crates/buzz-relay/src/api/bridge.rs:830`), so this variant is unreachable |
| `AuthError::Nip98Replay` | `crates/buzz-auth/src/error.rs:37` | never constructed in this crate; intended for callers per the usage example at `crates/buzz-auth/src/nip98_replay.rs:24` |
| `AuthError::PubkeyMismatch` | `crates/buzz-auth/src/error.rs:41` | never constructed anywhere in the crate |
| `AuthContext.channel_ids` | `crates/buzz-auth/src/lib.rs:72` | always `None` at every construction site (`crates/buzz-auth/src/lib.rs:139`, `crates/buzz-relay/src/handlers/event.rs:1393`); no code reads it | self-documented as "reserved for future" (`:69`) |
| `MockAccessChecker`, `AlwaysAllowRateLimiter`, `AlwaysFreshReplayGuard` | `access.rs:108`, `rate_limit.rs:219`, `nip98_replay.rs:127` | zero external users | The `test-utils` feature that exposes them is never enabled by any crate. `buzz-relay` and `buzz-pubsub` hand-roll their own doubles instead (`crates/buzz-relay/src/admission.rs:64-96`, `crates/buzz-relay/src/api/bridge.rs:3284`, `crates/buzz-relay/src/api/invites.rs:419`, `crates/buzz-relay/src/api/operator.rs:521`) — including a locally-redefined `AlwaysFreshReplayGuard` in four separate files, duplicating what this crate already exports |

Roughly a third of the crate's public surface (the whole `access` module plus
`parse_scopes`, `all_non_admin`, the dev derivation, and the IP-limit path) has no
production consumer.

### Unused dependency

`tracing` is declared (`crates/buzz-auth/Cargo.toml:20`) but never used — no
`use tracing`, no `tracing::` macro call, no `#[instrument]` anywhere in
`crates/buzz-auth/src/`. `buzz-admin` also declares `buzz-auth`
(`crates/buzz-admin/Cargo.toml:17`) while referencing zero `buzz_auth` items.

---

### Documentation drift inside the crate

| Doc claim | Reality | file:line |
|-----------|---------|-----------|
| "All paths produce an [`AuthContext`] bound to the connection" | `verify_nip98_event` returns a bare `PublicKey`; no `AuthContext` is ever produced on the NIP-98 path | claim `crates/buzz-auth/src/lib.rs:15`; code `crates/buzz-auth/src/nip98.rs:60`, `:130` |
| `ChannelAccessChecker` "Implemented by the database layer (`buzz-db`) in production" | no implementor exists outside this crate | claim `crates/buzz-auth/src/access.rs:18-19` |
| `all_known()` doc: "Used in dev mode (`require_auth_token=false`) where `X-Pubkey` header auth grants unrestricted access" | it is the production NIP-42 grant set (`crates/buzz-auth/src/lib.rs:138`); the doc text is copy-pasted from `all_non_admin()` and describes the wrong caller | claim `crates/buzz-auth/src/scope.rs:66-67` vs `:91-93` |
| `RateLimiter` doc: "The Redis-backed production implementation lives in `buzz-relay` / `buzz-pubsub`" | it is specifically `buzz-pubsub`; `buzz-relay` holds the wiring, not the impl. Minor, but the ambiguity is what `ARCHITECTURE.md` got wrong | claim `crates/buzz-auth/src/rate_limit.rs:3-4`, `:148`; impl `crates/buzz-pubsub/src/rate_limiter.rs:99` |
| `Scope` doc: "Scopes are stored as `TEXT[]` in the database" | unverifiable from this crate (no DB dependency); no code here reads or writes scopes | claim `crates/buzz-auth/src/scope.rs:12-14` |
| `Scope::ReposRead` / `ReposWrite` enforcement notes | claims about `buzz-relay` git routes and WS ingest that cannot be verified from here; they date the code to a "v2 collaborator model" that does not exist yet | `crates/buzz-auth/src/scope.rs:46-56` |

Related external drift (documented in the security aspect): `ARCHITECTURE.md:390`,
`:460`, and `:823` all state that no Redis-backed rate limiter exists and that
rate limiting is unenforced. Both statements are contradicted by
`crates/buzz-pubsub/src/rate_limiter.rs:99` and the enforcement call sites at
`crates/buzz-relay/src/connection.rs:498-500` and
`crates/buzz-relay/src/api/bridge.rs:760`, `:955`, `:1386`.

---

### Correctness / robustness hotspots

| # | Issue | Detail | file:line |
|---|-------|--------|-----------|
| 1 | Duplicated tolerance constant | `TIMESTAMP_TOLERANCE_SECS = 60` is defined twice, privately, in two modules. Nothing links them, and nothing links either to `DEFAULT_REPLAY_TTL_SECS = 120`, which is documented as 2× the NIP-98 value. Changing one silently breaks the derived relationship; the tripwire test asserts against the literal `120`, not against the tolerance | `crates/buzz-auth/src/nip42.rs:35`, `crates/buzz-auth/src/nip98.rs:32`, `crates/buzz-auth/src/nip98_replay.rs:42-46`, test `:210-218` |
| 2 | Two divergent URL normalisers | `normalize_relay_url` collapses loopback aliases and returns unparseable input verbatim; `normalize_url` does neither and lowercases unparseable input. The divergence is justified for NIP-98 (`crates/buzz-auth/src/nip98.rs:138-144`) but **not documented on the NIP-42 side**, so the weaker host binding there reads as an oversight rather than a decision | `crates/buzz-auth/src/nip42.rs:19-33` vs `crates/buzz-auth/src/nip98.rs:145-153` |
| 3 | Duplicated `make_auth_event` test helper | byte-identical helper in two test modules | `crates/buzz-auth/src/nip42.rs:95-100` and `crates/buzz-auth/src/lib.rs:174-179` |
| 4 | `expect` on a production path | `parse_scopes` calls `.expect("infallible: Scope::from_str cannot fail")`. Statically unreachable given `Err = Infallible`, but it is the crate's only non-test `expect`/`unwrap` and violates the repo rule against new `unwrap`/`expect` in production paths. `Result::unwrap_or_else(\|e\| match e {})` or `Infallible`-aware destructuring would remove it | `crates/buzz-auth/src/scope.rs:170-178` |
| 5 | NIP-98 verification is not offloaded | `verify_nip98_event` performs a Schnorr verify synchronously and, unlike the NIP-42 path, this crate never wraps it in `spawn_blocking`, despite `buzz-core` documenting that requirement. The obligation is implicit on every HTTP caller | `crates/buzz-auth/src/nip98.rs:55`; NIP-42 comparison `crates/buzz-auth/src/lib.rs:128-132`; upstream note `crates/buzz-core/src/verification.rs:1-2` |
| 6 | Error coarsening loses diagnostics | NIP-42 maps both "wrong kind" and "bad signature/id" to `AuthError::InvalidSignature`, discarding the underlying `VerificationError`. Good for oracle resistance, poor for operator debugging, and there is no logging to compensate (the `tracing` dep is unused) | `crates/buzz-auth/src/nip42.rs:52-56` |
| 7 | NIP-98 error strings echo attacker input | `URL mismatch: event has \`{u_tag}\`, expected \`{expected_url}\`` interpolates the untrusted `u` tag. The doc says not to forward to clients, but nothing enforces it, and `buzz-relay` does forward it: `api_error(UNAUTHORIZED, &format!("NIP-98: {e}"))` | message `crates/buzz-auth/src/nip98.rs:98-100`; forwarded at `crates/buzz-relay/src/api/bridge.rs:112` |
| 8 | First-tag-wins on duplicate tags | `tags.find(...)` returns the first match for `challenge`, `relay`, `u`, `method`, and `payload`. A second conflicting tag is ignored rather than rejected. Contrast `buzz-relay`'s NIP-OA handling, which explicitly treats >1 `auth` tag as none | `crates/buzz-auth/src/nip42.rs:58-70`, `crates/buzz-auth/src/nip98.rs:89-117`; contrast `crates/buzz-relay/src/handlers/auth.rs:31-34` |
| 9 | Two async-trait styles in one crate | `ChannelAccessChecker` and `RateLimiter` use RPITIT (not dyn-compatible); `Nip98ReplayGuard` uses `Pin<Box<dyn Future>>`. The reason is real (the relay needs `Arc<dyn Nip98ReplayGuard>`, `crates/buzz-relay/src/state.rs:582`) but is not documented at the trait, so the inconsistency looks accidental | `crates/buzz-auth/src/access.rs:35-51`, `crates/buzz-auth/src/rate_limit.rs:174-193`, `crates/buzz-auth/src/nip98_replay.rs:66-103` |
| 10 | `AuthContext` has no invariants | all fields `pub`, no constructor, no validation. Any consumer can widen `scopes` after the fact — and the relay does hold it as `mut` (`crates/buzz-relay/src/handlers/auth.rs:91`). A private-field + builder shape would make the grant decision auditable in one place | `crates/buzz-auth/src/lib.rs:64-80` |
| 11 | `AuthService` is a near-empty wrapper | holds only `AuthConfig` and forwards to a free function; adds `spawn_blocking` and nothing else. `verify_auth_event` also ignores `self.config` entirely — the config is only there for callers to read back via `config()` | `crates/buzz-auth/src/lib.rs:100-143` |
| 12 | Fixed-window burst | documented, not fixed: up to 2× the configured rate at window boundaries. The relay compounds it deliberately with a 5s burst window for WS | `crates/buzz-auth/src/rate_limit.rs:6-7`, `:165-167`; `crates/buzz-relay/src/admission.rs:6-9` |

---

### Complexity hotspots

The crate is small and flat; no function is deeply nested.

| Function | Lines | Shape |
|----------|-------|-------|
| `verify_nip98_event` | 77 (`crates/buzz-auth/src/nip98.rs:55-131`) | 8 sequential guard clauses, one nesting level. Longest fn in the crate; readable but each step is untestable in isolation |
| `verify_nip42_event` | 40 (`crates/buzz-auth/src/nip42.rs:47-86`) | 7 guard clauses, one nesting level |
| Everything else | < 25 lines | mostly `match` tables and struct literals |

The real complexity is not in control flow but in the **cross-crate contracts
expressed as doc comments**: `Nip98ReplayGuard` carries ~30 lines of MUST/MUST NOT
prose (`crates/buzz-auth/src/nip98_replay.rs:73-100`) and `RateLimiter` ~25
(`crates/buzz-auth/src/rate_limit.rs:151-167`). None of it is machine-checked in
this crate — correctness depends on `buzz-pubsub` and `buzz-relay` honouring prose.

---

### Test coverage gaps

45 tests total (35 `#[test]`, 10 `#[tokio::test]`) across 7 of 8 files; no
integration-test directory.

| Gap | Detail |
|-----|--------|
| `crates/buzz-auth/src/error.rs` | zero tests — no `Display` output assertions, so the user-facing error strings (including the "±60s" text at `:23`) can drift from the constants unnoticed |
| `check_write_access` | never tested. Only `check_read_access` has coverage (`crates/buzz-auth/src/access.rs:178-223`); the write path's `Scope::MessagesWrite` requirement is unverified |
| `derive_pubkey_from_username` | no test at all, despite `#[cfg(any(test, ...))]` compiling it during tests. Nothing pins the `"buzz-test-key:{username}"` prefix, so the desktop↔relay derivation compatibility the doc claims (`crates/buzz-auth/src/lib.rs:149-151`) is unverified on this side |
| `parse_scopes` with `Unknown` inputs | tested only with two known scopes (`crates/buzz-auth/src/scope.rs:199-203`) |
| `Scope` round-trip | only 3 of 16 variants are round-tripped (`crates/buzz-auth/src/scope.rs:185-190`); the `as_str`/`FromStr` tables are exact inverses today but nothing enforces that for the other 13 |
| NIP-42 boundary timestamps | rejection tested at 120s (`crates/buzz-auth/src/nip42.rs:143-157`); the exact boundary (delta 60 passes, 61 fails) is untested on both paths |
| NIP-42 missing-tag paths | no test for an AUTH event with no `challenge` tag or no `relay` tag (`crates/buzz-auth/src/nip42.rs:62`, `:72`) |
| NIP-42 tampered-signature path | wrong-kind and wrong-challenge are tested; a valid kind:22242 with a mutated signature (exercising `buzz_core::verify_event` failure at `:56`) is not |
| NIP-98 missing-tag paths | no test for a missing `u` tag or missing `method` tag (`crates/buzz-auth/src/nip98.rs:95`, `:108`) |
| NIP-98 unparseable-URL fallback | the `to_lowercase()` branch (`crates/buzz-auth/src/nip98.rs:148`) is untested; likewise NIP-42's verbatim fallback (`crates/buzz-auth/src/nip42.rs:22`) |
| `payload` tag present + `body == None` | the silent-skip case (`crates/buzz-auth/src/nip98.rs:119`) is untested — the inverse case is (`:269-276`) |
| `RateLimiter` / `Nip98ReplayGuard` contracts | only the always-allow / always-fresh happy paths are tested (`crates/buzz-auth/src/rate_limit.rs:315-325`, `crates/buzz-auth/src/nip98_replay.rs:239-248`). No test asserts a denied result, an `Err` propagating, or TTL clamping — the documented MUSTs are unverified in this crate |
| `ChannelAccessChecker::can_access` override path | only the default body is exercised, via `MockAccessChecker` which does not override it |
| `spawn_blocking` panic path | `AuthError::Internal("spawn_blocking panicked")` (`crates/buzz-auth/src/lib.rs:132`) is unreachable in tests |

---

### Unimplemented / stubbed traits

| Trait | Production implementor | Assessment |
|-------|----------------------|------------|
| `ChannelAccessChecker` | **none anywhere in the repo** | genuinely unimplemented. The crate ships the enforcement helpers (`check_read_access`/`check_write_access`) with nothing to plug into them |
| `RateLimiter` | `RedisRateLimiter` (`crates/buzz-pubsub/src/rate_limiter.rs:99`), wired at `crates/buzz-relay/src/state.rs:712` | implemented and enforced (see security aspect) |
| `Nip98ReplayGuard` | `RedisNip98ReplayGuard` (`crates/buzz-pubsub/src/nip98_replay.rs:34`), wired at `crates/buzz-relay/src/state.rs:582` | implemented and enforced |

Partially-implemented tier model: `RateLimitConfig` defines 4 tiers, but only
`human` and `agent-standard` reach an enforcement point.
`agent_standard_api_calls_per_min`, `agent_elevated_messages_per_min`, and
`agent_platform_messages_per_min` are parsed from env
(`crates/buzz-relay/src/config.rs:303-314`) and never read — three documented,
CI-settable knobs that do nothing.

---

### Deprecated API usage

None found. No `#[deprecated]` items are declared in the crate, and no
`#[allow(deprecated)]` appears in `crates/buzz-auth/src/`. Dependencies are all
`workspace = true` with no pinned-back versions
(`crates/buzz-auth/Cargo.toml:15-26`).

The only lint suppressions in the crate are two
`#[allow(clippy::assertions_on_constants)]` in the const-drift tripwire tests,
each with a comment explaining why the assertion is intentionally over a constant
(`crates/buzz-auth/src/nip98_replay.rs:214`, `:226`).


## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Debt

Baseline health is good: 2,118 lines of Rust, zero `unsafe`, zero `TODO`/`FIXME`/
`HACK` markers, zero `unwrap`/`expect` outside `#[cfg(test)]`, 34 tests. The debt is
concentrated in (a) a capability the crate advertises but never implemented, (b) an
untested security control, and (c) unreachable knobs.

---

### Summary

| # | Item | Severity |
|---|---|---|
| D-PS-1 | Typing indicators advertised in the manifest and ARCHITECTURE.md; **no implementation exists** | High (doc integrity) |
| D-PS-2 | `rate_limiter.rs` has **zero tests** despite being a security control | High |
| D-PS-3 | `retain_topic` can block forever / silently lose subscriptions if the subscriber task isn't running | High |
| D-PS-4 | Control messages are unversioned and unauthenticated; a rolling deploy can silently skip a ban | Medium-High |
| D-PS-5 | ARCHITECTURE.md presence key omits community scope | Medium |
| D-PS-6 | `lib.rs:331` says 60 s presence TTL; actual is 90 s | Medium |
| D-PS-7 | `check_ip_connection` is dead code — and pre-auth traffic is unmetered | Medium |
| D-PS-8 | Fixed-window limiter permits 2× burst (known, self-documented) | Medium |
| D-PS-9 | Clean-disconnect backoff reset enables a 1 s hot reconnect loop | Medium |
| D-PS-10 | Redis is a hard availability dependency for authenticated reads, with no degraded mode | Medium |
| D-PS-11 | No metrics — reconnects, drops, lag, and refcounts are invisible | Medium |
| D-PS-12 | Hardcoded capacities/TTLs; unsubscribe-debounce knob unreachable in production | Low-Medium |
| D-PS-13 | Triplicated reconnect loops and backoff constants (~50 duplicated lines) | Low-Medium |
| D-PS-14 | Broken intra-doc link `[crate::ConnectionManager]` | Low |
| D-PS-15 | No presence index — "who is online" is unanswerable from this crate | Low-Medium |
| D-PS-16 | `pub mod subscriber` is an empty public namespace | Low |
| D-PS-17 | `channel_key`/`global_key` reachable by three paths | Low |
| D-PS-18 | Unused `chrono` dependency | Low |
| D-PS-19 | `topic_refcount` documented for metrics but unwired | Low |
| D-PS-20 | Test-fixture duplication and inconsistent test Redis endpoint | Low |
| D-PS-21 | 11 of 34 tests `#[ignore]`d behind live Redis | Low |
| D-PS-22 | `missing_docs` is `warn`, not `deny` | Low |

---

### D-PS-1 — Typing indicators: advertised everywhere, implemented nowhere (High)

Four independent places claim this crate implements typing indicators:

| Claim | Location |
|---|---|
| `description = "Redis pub/sub fan-out, presence, and typing indicators for Buzz"` | `crates/buzz-pubsub/Cargo.toml:8` |
| Doc comment `/// Typing indicator tracking in Redis.` | `crates/buzz-pubsub/src/lib.rs:43` |
| `buzz-pubsub (Redis pub/sub, presence, typing indicators)` in the crate map; section heading `### buzz-pubsub — Redis Pub/Sub, Presence, Typing`; "Manages … typing indicators" | `ARCHITECTURE.md:82`, `:432`, `:434` |
| `buzz-pubsub  # Redis pub/sub fan-out, presence, typing indicators` | `AGENTS.md` repo-structure table |

ARCHITECTURE.md goes further and specifies a concrete Redis design:

```
ZADD buzz:typing:{channel_id} {now_unix} {pubkey_hex}
ZREMRANGEBYSCORE buzz:typing:{channel_id} -inf {now - 5.0}
EXPIRE buzz:typing:{channel_id} 60
```
(`ARCHITECTURE.md:452-456`), lists `buzz:typing:{channel_uuid}` as a live 60 s
sorted-set key in the Redis keyspace table (`ARCHITECTURE.md:801`), and describes
Redis as providing "typing (sorted sets)" in the infrastructure table
(`ARCHITECTURE.md:777`).

**None of it exists.** Evidence of absence:
- No `typing` module — the module list ends at `pub mod topic` (`lib.rs:42`). The doc
  comment at `lib.rs:43` is **mis-attached to `pub use error::PubSubError;`**
  (`lib.rs:44`), which is how the claim survived: rustdoc renders it as the
  re-export's description, so `missing_docs` never fired.
- `ZADD`/`ZREMRANGEBYSCORE` appear **once** in all of `crates/**/*.rs`, and only as
  words inside the crate-header comment listing example commands (`lib.rs:10`). There
  is no sorted-set call anywhere in the workspace.
- The string `buzz:typing` appears **zero** times in the repository.

What actually happens: typing is a plain ephemeral Nostr event,
`KIND_TYPING_INDICATOR = 20002` (`crates/buzz-core/src/kind.rs:407`, registered in
`ALL_KINDS` at `:625`), produced by the agent harness
(`crates/buzz-acp/src/config.rs:380-382`, `:517`, refresh loop
`crates/buzz-acp/src/lib.rs:1593-1602`) and fanned out through the generic
`publish_event` path — consistent with `ARCHITECTURE.md:258-263`, which contradicts
`:452-456`. There is no server-side typing *state*, which is also why
`ARCHITECTURE.md:824` correctly notes there is no way to query current typers.

Fix is a choice, not a code change: either implement the ZSET design or delete the
claim from `Cargo.toml:8`, `lib.rs:43`, `AGENTS.md`, and `ARCHITECTURE.md:82`,
`:432`, `:434`, `:452-456`, `:777`, `:801`. Deleting is the honest option — the
event-based approach works and needs no Redis state.

### D-PS-2 — The Redis rate limiter has no tests (High)

`rate_limiter.rs` contains **zero test functions** — the only file in the crate with
public logic and no tests (alongside `error.rs` and `publisher.rs`, which are
trivial). Untested behaviours that matter:

- The Lua script itself (`rate_limiter.rs:24-31`) — that `EXPIRE` fires only when
  `count == 1`, and that the returned `{count, ttl}` tuple decodes as `(u64, i64)`.
- The allow boundary `count <= limit` (`rate_limiter.rs:74-78`) — an off-by-one here
  changes every limit in the product by one request.
- The TTL-repair branch (`rate_limiter.rs:57-70`), including that repair resets the
  window to full duration (i.e. a key in the broken state gets a fresh allowance).
- Key construction for both pubkey and IP paths (`rate_limiter.rs:110`, `:118`).

`buzz-relay`'s admission tests do not close this gap: they substitute a
`#[cfg(test)] StubLimiter` (`crates/buzz-relay/src/admission.rs:65-90`) that returns
canned `denied`/`Err` values and never touches Redis. So the enforcement *decision*
is tested while the *counting* is not, anywhere in the repo.

### D-PS-3 — `retain_topic` can block forever, or silently lose a subscription (High)

`retain_topic` increments the refcount, then awaits
`subscription_tx.send(Subscribe(topic))` and **discards the result**
(`lib.rs:203-207`). Two failure modes:

1. **Silent loss.** If the send fails (receiver dropped), the refcount already says
   "retained" but no Redis `SUBSCRIBE` will ever be issued. The connection believes
   it is subscribed and receives nothing — a silent correctness failure with no log.
2. **Unbounded await.** The channel is `mpsc::channel(4096)` (`lib.rs:129`), so
   `send` awaits when full. If `run_subscriber` was never spawned, nothing drains the
   receiver; after 4096 retains, every subsequent `retain_topic` awaits indefinitely,
   stalling whatever relay task called it.

The take-once guard (`lib.rs:149-152`) protects against a *double* start but does
nothing about a *missing* start. The same discard pattern is at `lib.rs:238-243`
(release path), where the consequence is milder — a stale Redis subscription.

Minimal fix: log on send failure, and make the "subscriber not started" state
observable rather than silent.

### D-PS-4 — Unversioned, unauthenticated control messages (Medium-High)

`CacheInvalidation` (`cache_invalidation.rs:57-80`) and `ConnControl`
(`conn_control.rs:55-73`) are internally-tagged enums with no version field, no
origin-pod id, no timestamp, no signature, and no `#[non_exhaustive]`.

Rolling-deploy consequence: a pod running an older binary receives a new variant,
fails deserialization, and skips it with a `warn`
(`cache_invalidation.rs:161-165`, `conn_control.rs:152-156`). For
`ConnControl::DisconnectPubkey` that means a ban is **not live-enforced** on old pods
until they restart. The DB row is the durable backstop
(`conn_control.rs:18-21`) so the member cannot re-authenticate, but an
already-connected socket survives. A forward-compat test pins the skip behaviour
(`conn_control.rs:209-217`) — the behaviour is intentional; the deploy-ordering
consequence is undocumented.

Also: `pubkey` is documented as "32 raw bytes" (`conn_control.rs:60`) but typed
`Vec<u8>` and length-validated on neither publish nor receive.

Injection risk from Redis write access is covered in the security aspect
(attacker-controlled `reason` echoed to the victim, arbitrary disconnects).

### D-PS-5 / D-PS-6 — Presence documentation drift (Medium)

| Claim | Reality |
|---|---|
| `ARCHITECTURE.md:450`: `SET buzz:presence:{pubkey_hex} {status} EX 90` | Actual key is `buzz:{community}:presence:{pubkey_hex}` (`presence.rs:19-25`) — the community segment is unconditional |
| `ARCHITECTURE.md:800`: lists `buzz:presence:{pubkey_hex}`, hedged as "single-community form; shared multi-community Redis **must** scope by community" | The code has no unscoped form; the hedge describes a requirement that is already met, which reads as though it isn't |
| `lib.rs:331`: "Set presence with **60s** TTL" | `PRESENCE_TTL_SECS = 90` (`presence.rs:16`); the module doc correctly says `EX 90` (`presence.rs:5`) and explains the 3× rationale (`presence.rs:4-6`, `:15`) |

D-PS-6 is an internal contradiction inside one crate, and it is the doc a caller sees
first (it is on the `PubSubManager` method, not the private helper). ARCHITECTURE.md
happens to be right about the 90 s value while the code comment is wrong.

### D-PS-7 — Dead IP limiter, live pre-auth gap (Medium)

`check_ip_connection` (`rate_limiter.rs:112-120`) has no production caller. Across
`crates/**/*.rs`, `LimitType::IpConnections` and `ip_rate_limit_key` appear only in
the `buzz-auth` definitions (`crates/buzz-auth/src/rate_limit.rs:66`, `:76`, `:213`),
this implementation, and a `#[cfg(test)]` stub
(`crates/buzz-relay/src/admission.rs:85`).

That matters because it is precisely the limiter that would cover the gap it leaves:
`enforce_ws_admission` returns `true` for any non-authenticated connection
(`crates/buzz-relay/src/connection.rs:602-609`) because the live limiter is
pubkey-keyed. Pre-AUTH WebSocket traffic is therefore unmetered by this subsystem —
functioning code exists to fix it and is simply not wired up.

### D-PS-8 — Fixed-window 2× burst (Medium, accepted)

Self-documented with a `⚠️` and a stated upgrade path: "Fixed windows allow up to 2×
burst at boundaries. Upgrade to sliding window or token bucket for strict limiting"
(`rate_limiter.rs:9-10`). Independently flagged at the call site:
"this is still a fixed-window limiter, so a Redis-backed token bucket would be a
better long-term fit for smoother refill behavior"
(`crates/buzz-relay/src/admission.rs:4-7`). Well-managed debt; recorded for
completeness.

### D-PS-9 — Clean-disconnect backoff reset permits a 1 s hot loop (Medium)

All three subscriber loops reset the backoff to `BACKOFF_INITIAL_SECS` whenever the
stream ends *cleanly* (`subscriber.rs:57-63`, `cache_invalidation.rs:105-111`,
`conn_control.rs:95-101`). The intent is stated and reasonable — "a brief Redis
restart should reconnect quickly" (`subscriber.rs:58-60`).

Failure mode: an endpoint that accepts a connection and immediately closes the stream
(misconfigured TCP proxy, `maxclients` eviction, TLS terminator that drops pub/sub)
returns `Ok(())` every cycle, so the backoff never escalates past 1 s. Three loops ×
one reconnect per second, indefinitely, each opening a fresh `redis::Client`
(`subscriber.rs:85`, `cache_invalidation.rs:135`, `conn_control.rs:126`). A
consecutive-clean-end counter, or a minimum-uptime requirement before resetting,
would bound this.

### D-PS-10 — Redis on the critical read path with no degraded mode (Medium)

`AuthError::Internal` from the limiter (`rate_limiter.rs:44-46`, `:52`, `:66`) becomes
`AdmissionError::Unavailable` (`crates/buzz-relay/src/admission.rs:29-33`), which the
relay treats as denial (`crates/buzz-relay/src/connection.rs:612-621`). A Redis
outage therefore rejects every authenticated `EVENT`, `REQ`, and `COUNT` — reads
included. Fail-closed is the right default for a limiter, but there is no circuit
breaker, no in-process fallback limiter, and no documented operator override. Worth an
explicit decision record rather than an emergent property.

The replay guard has the same shape by contract (`nip98_replay.rs:53-57`, `:74-79`),
which is correct for a replay control but compounds the blast radius of one Redis.

Related unstated operational requirement: the replay seen-set assumes keys survive
their TTL. Under `maxmemory` with an eviction policy, Redis may evict
`buzz:{community}:nip98:*` markers early, silently shortening the replay window.
Nothing in the crate or the deployment docs asserts `noeviction`.

### D-PS-11 — No metrics (Medium)

Observability is `tracing` logs only. Nothing counts events published or received,
messages dropped for parse failure (`subscriber.rs:143`, `:154`), broadcast lag
(`error.rs:22`), reconnect attempts, or active subscription counts. Prometheus is
part of the deployment, so the collection path exists. `topic_refcount`
(`lib.rs:248`) was written for exactly this and is unwired (D-PS-19).

### D-PS-12 — Hardcoded tunables; one knob unreachable (Low-Medium)

Four channel capacities at 4096 (`lib.rs:126-129`), `PRESENCE_TTL_SECS = 90`
(`presence.rs:16`), and the backoff envelope are all compile-time constants with no
operator override.

Worse, a knob that *was* built is unreachable: `PubSubConfig::with_unsubscribe_debounce`
(`lib.rs:93`) has zero callers outside this crate's tests, because every production
construction goes through `PubSubManager::new` (`lib.rs:117-119`), which hardcodes
`PubSubConfig::new`. All 11 relay call sites use `new`; the production one is
`crates/buzz-relay/src/main.rs:344` and the other ten are test fixtures. The 500 ms
default (`lib.rs:82`) is effectively a constant.

### D-PS-13 — Triplicated reconnect machinery (Low-Medium)

`BACKOFF_INITIAL_SECS` / `BACKOFF_MAX_SECS` are declared three times with identical
values (`subscriber.rs:16-19`, `cache_invalidation.rs:91-94`, `conn_control.rs:81-84`).
The three `run_*_subscriber` loops (`subscriber.rs:41-76`,
`cache_invalidation.rs:100-127`, `conn_control.rs:90-117`) are structurally identical,
and the two pattern-subscribe bodies (`cache_invalidation.rs:129-179` vs
`conn_control.rs:120-168`) differ only in payload type, pattern constant, and log
strings — roughly 50 duplicated lines. A generic
`run_pattern_subscriber<T: DeserializeOwned>(pattern, parse_scope, tx)` would collapse
both, and the shared backoff would then have one definition. `conn_control.rs:91-92`
already documents itself as "mirrors" the cache-invalidation loop.

### D-PS-14 — Broken intra-doc link (Low)

`conn_control.rs:7` links `[crate::ConnectionManager]`. No `ConnectionManager` exists
in `buzz-pubsub` — it is a `buzz-relay` type. The crate sets
`#![warn(missing_docs)]` (`lib.rs:2`) but not `#![deny(rustdoc::broken_intra_doc_links)]`,
so this produces a rustdoc warning and renders as literal text.

### D-PS-15 — No presence index (Low-Medium)

Presence is individual keys with no per-community roster set, so `get_presence_bulk`
(`presence.rs:74-95`) requires the caller to already know which pubkeys to ask about.
"List everyone online in this community" cannot be answered from Redis; the relay must
source candidates from the DB first (`crates/buzz-relay/src/api/bridge.rs:1954`).
Acceptable for member-list hydration, but it rules out cheap online counts and makes
presence cost scale with roster size rather than with online population.

### D-PS-16 to D-PS-19 — API-surface tidiness (Low)

- **D-PS-16:** `pub mod subscriber` (`lib.rs:40`) exports nothing — `DesiredTopics`
  (`subscriber.rs:21`), `SubscriptionCommand` (`:26`), and `run_subscriber` (`:41`) are
  all `pub(crate)`. An empty public namespace in the docs.
- **D-PS-17:** `channel_key`/`global_key` are reachable three ways —
  `buzz_pubsub::` (re-export `lib.rs:58`), `buzz_pubsub::topic::` (`topic.rs:103`,
  `:108`), and `buzz_pubsub::publisher::` (`publisher.rs:12`, `:17`, pure delegates).
  `publisher.rs` is 37 lines, two of which are redundant wrappers.
- **D-PS-18:** `chrono` is declared (`Cargo.toml:18`) but no `chrono::` path appears in
  any source file — unused dependency carrying build time and audit surface.
- **D-PS-19:** `topic_refcount` (`lib.rs:248`) is documented "for tests and metrics"
  and has zero non-test callers; no metric consumes it.

### D-PS-20 to D-PS-22 — Test hygiene (Low)

- **D-PS-20:** `test_presence_set_and_get` is defined twice with overlapping
  assertions (`lib.rs:477` and `presence.rs:138`). The `ctx(id, host)` fixture is
  copied six times (`lib.rs:388`, `topic.rs:119`, `presence.rs:119`,
  `cache_invalidation.rs:185`, `conn_control.rs:171`, `subscriber.rs:191`). Redis
  endpoint resolution is inconsistent: `nip98_replay.rs:110` honours `REDIS_URL`
  while `test_util::make_test_pool` (`lib.rs:371-376`) hardcodes `127.0.0.1:6379` —
  and the relay's own default is spelled `localhost:6379`
  (`crates/buzz-relay/src/config.rs:419`), which resolves differently on
  IPv6-preferring hosts.
- **D-PS-21:** 11 of 34 tests are `#[ignore = "requires Redis"]`, so the default
  unit-test gate runs 23 and covers none of the presence round-trip, replay-guard, or
  pub/sub round-trip behaviour. The two most valuable tests in the crate — the
  cross-community topic-isolation regression (`lib.rs:510-590`) and the replay-guard
  clamp tests (`nip98_replay.rs:164-200`) — only run when Redis is present.
- **D-PS-22:** `missing_docs` is `warn` (`lib.rs:2`), so doc coverage can regress
  without failing CI. D-PS-1's orphaned doc comment is a concrete example of what
  slips through.

### Inherited debt

`Cargo.toml:7` inherits `repository.workspace = true`, which resolves to the stale
`github.com/block/sprout` URL flagged in the Phase 2a findings — the same legacy
naming affects this crate's published metadata.


## Module: buzz-search (`crates/buzz-search`)

### Aspect: Debt

#### Markers

Zero `TODO` / `FIXME` / `HACK` / `XXX` markers exist in the crate (grep over
`Cargo.toml`, `src/**`, `tests/**`). Everything below is inferred from reading code,
not from declared debt.

#### Complexity hotspots

| Hotspot | Location | Nature |
|---|---|---|
| Prefix-mode tsquery construction | `src/query.rs:154-176` (~23 lines of SQL inside two `push` calls) | A nested subselect combining `regexp_split_to_table ... WITH ORDINALITY`, a window `max(token_ord) OVER ()`, `CROSS JOIN LATERAL unnest(tsvector_to_array(to_tsvector(...))) WITH ORDINALITY`, `quote_literal`, `string_agg` with `ORDER BY`, `COALESCE`, and a `::tsquery` cast — expressed as a Rust string literal with `\` continuations. Not unit-testable in isolation; correctness depends entirely on 3 ignored integration tests (`tests/fts_integration.rs:306-539`). |
| `search()` length | `src/query.rs:216-323` (108 lines) | Single function does normalization dispatch, clamping, SQL assembly across 9 optional fragments, execution, and row decoding. No sub-functions besides `push_tsquery`. |
| Manual row decoding | `src/query.rs:302-320` | Six `try_get` calls plus two `try_into` length checks with hand-written error strings; no `FromRow` derive, so column-name drift is a runtime failure. |

#### Redundant / effectively dead logic

| Item | Location | Detail |
|---|---|---|
| `per_page` intermediate binding | `src/query.rs:224-229` | `let per_page = query.per_page.clamp(1, PER_PAGE_MAX)` already maps `0 → 1`; the following `if query.per_page == 0 { PER_PAGE_DEFAULT }` re-reads the raw field to override that. The `per_page` binding is used only in the `else` arm, so the clamp's `0 → 1` result is unreachable. Behavior is fine (`0 → 100`), the code path is redundant. |
| `SearchHit.rank` / `kind` / `pubkey` / `channel_id` for in-tree callers | `src/query.rs:107-117` vs `crates/buzz-relay/src/handlers/req.rs:625-626`, `crates/buzz-relay/src/api/bridge.rs:1725` | Both callers immediately map hits to `event_id` only; the other four fields are selected, decoded, and discarded. `SELECT` of `kind`/`pubkey` and the `ts_rank_cd` computation are still needed for ordering (`rank` is in `ORDER BY`), but the decode work is unused. |
| `SearchResult.page` | `src/query.rs:126` | Returned but not read by either in-tree caller (WS drives pages in its own loop, `crates/buzz-relay/src/handlers/req.rs:588-604`). Its only consumer is a test assertion (`tests/fts_integration.rs:1044`). |
| `pub use query::search` free function | `src/lib.rs:31`, `src/query.rs:216` | Both the free fn and the `SearchService` method are public; all in-tree callers use the service. Two public entry points for one behavior. |

#### Documentation drift

| Drift | Location | Detail |
|---|---|---|
| SQL sketch vs emitted SQL | `src/query.rs:196-207` vs `src/query.rs:233-242` | Doc says `FROM events, <mode-specific tsquery> AS query` and `ts_rank_cd(search_tsv, query)`; code emits `FROM events CROSS JOIN LATERAL (SELECT ... AS query) AS search_query` and `ts_rank_cd(search_tsv, search_query.query)`. |
| Crate doc quotes an outdated `search_tsv` definition | `src/lib.rs:6-8` | States the column is `to_tsvector('simple', content)`; the actual expression is a `CASE` with a kind exclusion list (`migrations/0001_initial_schema.sql:222-226`) or, on fresh installs, a positive allowlist (`migrations/0008_fresh_install_search_allowlist.sql:15-20`). The privacy behavior the crate relies on is not described in its own doc. |
| ARCHITECTURE.md exclusion list is stale | `ARCHITECTURE.md:808-810` | Lists `CASE WHEN kind IN (1059, 30300, 30622)`; current schema is `(1059, 30300, 30350, 30622, 44100, 44101, 44200)` (`schema/schema.sql:212`). |
| `SearchQuery.per_page` doc | `src/query.rs:95` | Says "Clamped at 500 internally" without mentioning the `0 → 100` substitution. |

#### Test coverage gaps

| Gap | Detail |
|---|---|
| 18 of 21 tests require infrastructure | Every behavior test is `#[ignore = "requires Postgres"]` (`tests/fts_integration.rs`, 18 occurrences). `cargo test -p buzz-search` in a plain CI run executes only the 3 `normalized_search_text` unit tests (`src/query.rs:329-351`). |
| No SQL-string assertions | `QueryBuilder` output is never inspected in a unit test, so predicate presence/absence (`kinds = Some(vec![])`, `authors = Some(vec![])`, `since` only, `until` only, `ChannelScope::Any`) has no infra-free coverage. |
| `authors` filter untested | No integration test sets `authors: Some(...)` — every call site in `tests/fts_integration.rs` passes `authors: None`. The `pubkey = ANY` branch (`src/query.rs:275-281`) is uncovered. |
| `per_page` clamping untested | No test passes `per_page: 0` or `per_page > 500`; only `page: u32::MAX` is exercised (`tests/fts_integration.rs:1014-1055`). |
| `mode: Prefix` combined with `authors`/`since`/`until` untested | Prefix tests only vary `kinds` and content. |
| Rank ordering untested | No test asserts that a higher-`ts_rank_cd` document precedes a lower one; ordering coverage is limited to counts and `created_at` (`tests/fts_integration.rs:748-810`). |
| Decode-error branches untested | The "N bytes, expected 32" paths (`src/query.rs:306-311`) have no test — they require a malformed row. |
| `SearchError` surface untested | No test exercises a DB failure. |
| Migration list is manually maintained | `tests/fts_integration.rs:22-32`, `57-84` enumerate migrations by hand; a future FTS-affecting migration that is not added silently makes the privacy tripwires test a stale schema. The file notes the obligation at `tests/fts_integration.rs:55-56`. |
| Allowlist vs exclusion-list divergence is untested | Tests replay `0001…0008 + 0014` against an *empty* `events` table, so migration 0008's emptiness branch fires and the tests actually validate the **positive allowlist** path (`migrations/0008_fresh_install_search_allowlist.sql:13-23`). The brownfield exclusion-list expression (`migrations/0005_agent_turn_metric_fts.sql:26-31`) is therefore not the expression under test, even though the test doc comments describe the exclusion CASE (`tests/fts_integration.rs:1085-1104`). |

#### Coupling and structural debt

| Item | Detail |
|---|---|
| Privacy policy split across three languages/locations | Rust constants (`buzz_core::kind::AUTHOR_ONLY_KINDS`, `P_GATED_KINDS`), inlined SQL literals in each migration, and `schema/schema.sql` must be kept in sync by hand; the migrations say so explicitly (`migrations/0001_initial_schema.sql:216-221`, `migrations/0005_agent_turn_metric_fts.sql:19-22`). Tripwire tests exist but are ignored by default. |
| Divergent effective search policy per deployment | Fresh installs get an allowlist; populated installs keep an exclusion list until an operator runs `scripts/maintenance/nip_rs_search_allowlist.sql`. Two databases can answer the same query differently, and nothing in this crate can detect which policy is active. |
| Tests compile-time-depend on `migrations/` paths | `include_str!("../../../migrations/...")` (`tests/fts_integration.rs:22-32`) breaks the crate's test build if a migration file is renamed. |
| Test file is 1448 LOC vs 415 LOC of source | 3.5:1 test-to-source ratio concentrated in one file with a repeated 12-field `SearchQuery` literal (18 occurrences); no builder/helper for query construction, so adding a `SearchQuery` field requires touching every test. |
| No offline sqlx verification | Runtime `QueryBuilder` only (`src/query.rs:233`); column names and types are unverified at compile time. Matches the repo-wide limitation recorded at `ARCHITECTURE.md:823`. |
| Unqualified table name | `FROM events` relies on `search_path` (`src/query.rs:237`), which is what lets the test harness redirect to a per-test schema — but it also means the crate cannot be pointed at a schema-qualified table without changing the pool's `search_path`. |

#### Deprecated API usage

None found. `sqlx 0.9` `QueryBuilder`/`Row::try_get` are current; `thiserror 2`
attribute style is current; `sqlx::AssertSqlSafe` (`tests/fts_integration.rs:44`,
`100`) is the sqlx 0.9-era explicit opt-in for non-literal SQL, not a deprecated
path. No `#[deprecated]` items are referenced.

#### Observability gap

No `tracing`/`log` dependency and no instrumentation in `src/` — slow searches,
clamped pages, truncated search text, and empty-query short-circuits are all
invisible to operators from inside this crate. Callers log only failures
(`crates/buzz-relay/src/handlers/req.rs:614`, `crates/buzz-relay/src/api/bridge.rs:1719-1722`).


## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Debt

### Explicit debt markers

None. A case-insensitive search for `TODO`, `FIXME`, `HACK`, `XXX`, `todo!`,
`unimplemented!` across `crates/buzz-audit/src` returns **zero matches**. All debt below
is inferred from reading the code, not from author annotations.

### Complexity hotspots

| Item | Size / shape | Note |
|---|---|---|
| `AuditService::log` + `log_inner` | `service.rs:53-152` (~95 lines across two fns) | The most intricate control flow in the crate: pinned connection + advisory lock + `catch_unwind` + transaction + `resume_unwind`. Four interacting concerns in one place, no test for the panic branch |
| `compute_hash` | `hash.rs:42-73` (32 lines, 8 sequential `update` groups) | Straight-line but order-critical: any reordering silently invalidates every existing chain (warned at `hash.rs:28`). No golden/known-answer test pins the digest — the tests only assert determinism and per-field sensitivity (`hash.rs:152-254`), so an accidental field reorder would still pass CI |
| `canonical_json` | `hash.rs:80-116` | Hand-rolled recursive serializer with manual string building and `first` flags for both objects and arrays; duplicated comma logic in two branches. No recursion-depth guard |
| `verify_chain` | `service.rs:160-206` | Loads the whole `[from_seq, to_seq]` range into memory via `fetch_all` (`service.rs:178`) before walking it — memory grows linearly with the requested range; no streaming/cursor variant |
| `get_entries` | `service.rs:212-235` | Same `fetch_all` pattern; `limit` is caller-supplied with no cap (`service.rs:216`, `:230`) |

### Correctness/robustness gaps (see Security aspect for the threat framing)

1. **No golden-hash regression test.** Nothing asserts a fixed expected digest for a
   fixed entry, so the "field order is frozen" invariant (`hash.rs:28`) is unenforced by
   tests. `deterministic` (`hash.rs:152-157`) only compares the function to itself.
2. **`verify_chain` cannot detect tail truncation or a missing genesis.** No stored head
   pointer; the first row of a range is not validated against `seq = 1`/`GENESIS_HASH`,
   and `seq` contiguity is never asserted (`service.rs:185-203`).
3. **Empty range returns `Ok(false)`, not a distinct outcome.** `Ok(false)` conflates
   "nothing to verify" with a caller-visible boolean result (`service.rs:181-183`), so a
   caller cannot distinguish "no entries" from "verified nothing meaningful" without a
   second query. No enum/`Option` return.
4. **Advisory unlock errors swallowed.** `let _ = sqlx::query(...unlock...)`
   (`service.rs:71`) — a failed unlock leaves the lock held on a pooled connection with
   no log line and no metric.
5. **Cancellation leaks the lock.** `catch_unwind` covers unwinding only; dropping the
   `log` future skips the unlock (`service.rs:67-74`).
6. **Advisory-lock key collisions unhandled.** `hashtextextended` maps text → `bigint`
   (`service.rs:59`); two communities can collide and serialize each other, defeating
   the per-tenant design goal stated at `service.rs:25-28`. No detection or mitigation.
7. **`prev_hash` length never validated** (`hash.rs:69`; column is untyped `BYTEA`,
   `migrations/0001_initial_schema.sql:610`).
8. **Unframed hash pre-image.** Adjacent variable-length fields (`action`, `object_id`,
   canonical `detail`) have no length prefixes (`hash.rs:52-67`); safety currently rests
   on the restricted value domains rather than on the encoding.
9. **One unparsable `action` row fails an entire read.** `get_entries` collects
   `Result` (`service.rs:234`) and `verify_chain` decodes before checking
   (`service.rs:188`), so a single legacy/foreign row makes the whole call error with
   `UnknownAction` — there is no skip-and-report path. The DB column has no CHECK
   constraint (`migrations/0001_initial_schema.sql:611`), so such rows are insertable.
10. **`detail` "no secrets" rule is documentation only** (`entry.rs:64-71`) — no
    redaction, key denylist, or size limit.
11. **Untyped sqlx queries throughout.** `sqlx::query` + runtime `Row::get`
    (`service.rs:94`, `:130`, `:166`, `:218`, `:238-255`) instead of the compile-time
    checked macros, so a column rename in a future migration fails at runtime rather
    than at build time.

### Dead / unused code

| Item | Evidence |
|---|---|
| `hex` dependency | Declared at `crates/buzz-audit/Cargo.toml:21`; grep for `hex::` in `crates/buzz-audit/src` returns nothing |
| `tokio` as a normal dependency | `crates/buzz-audit/Cargo.toml:13`, but used only in `#[cfg(test)]` code (`service.rs:265`, `#[tokio::test]` at `:318`–`:512`) — belongs in `[dev-dependencies]` |
| `verify_chain` / `get_entries` | No production caller in the repo; only this crate's `#[ignore]` tests (`service.rs:368`, `:417`, `:427`, `:468`, `:508`, `:523`) and relay tests (`crates/buzz-relay/src/handlers/event.rs:1906-1952`). `buzz-admin` declares the dependency (`crates/buzz-admin/Cargo.toml:20`) but `crates/buzz-admin/src` contains no `audit` reference — the operator verification surface described in the docs does not exist yet |
| 9 of 11 `AuditAction` variants | Only `EventCreated` (`crates/buzz-relay/src/handlers/event.rs:583`) and `MediaUploaded` (`crates/buzz-relay/src/api/media.rs:428`) are produced in production. `EventDeleted`, `ChannelCreated`, `ChannelUpdated`, `ChannelDeleted`, `MemberAdded`, `MemberRemoved`, `AuthSuccess`, `AuthFailure`, `RateLimitExceeded` are defined (`action.rs:11-28`) but never logged |
| `to_storage_precision` public but unexported | `pub` at `hash.rs:22` yet absent from the root re-export list (`lib.rs:34`); reachable only as `buzz_audit::hash::to_storage_precision` |
| `AuditEntry`'s `Serialize`/`Deserialize` derives | `entry.rs:13` — no code in this crate or its consumers serializes an `AuditEntry` (relay never sends one on the wire); unverified whether any downstream needs it |

### Test coverage gaps

19 tests total: 13 `#[test]` + 6 `#[tokio::test]`, all 6 async ones
`#[ignore = "requires Postgres"]` (`service.rs:319`, `:339`, `:377`, `:438`, `:476`,
`:513`). Uncovered behaviour:

| Gap | Why it matters |
|---|---|
| Panic path through `catch_unwind`/`resume_unwind` | The documented single-writer safety property (`service.rs:64-79`) is never exercised |
| Advisory lock behaviour (contention, per-community independence at the lock level) | Tests serialize themselves with an in-process `Mutex` (`service.rs:263-267`), so the Postgres lock is never actually contended |
| Concurrent `log` calls / duplicate-`seq` race | No test attempts two simultaneous appends to one community |
| Tail truncation and genesis validation | No test — consistent with the code not implementing the check |
| `ChainViolation` variant | Constructed at `service.rs:193` but no test asserts it; the tampering test hits `HashMismatch` instead (`service.rs:472`) |
| `UnknownAction` on read | Constructed at `service.rs:242`; no test inserts a bogus `action` string |
| `Database` / `Serialization` error variants | Never asserted; the `Database` variant is also excluded from the error-sanitization test (`error.rs:79-83`) |
| `get_entries` pagination edges (`limit = 0`, negative `from_seq`, `limit` boundary) | Only the happy path is tested (`service.rs:425-433`) |
| Non-trivial `detail` payloads (nested objects, arrays, floats, unicode keys) | `canonical_json_key_order_is_stable` uses a flat 3-key object (`hash.rs:266-271`); the array branch (`hash.rs:101-113`) is untested |
| Whole-crate CI signal without Postgres | Only 13 tests run in `just test-unit`; every chain-behaviour assertion sits behind `--ignored` |

### Deprecated API usage

None found. `chrono` `trunc_subsecs`/`to_rfc3339` (`hash.rs:23`, `:49`), `sha2` 0.11
`Digest` (`hash.rs:2`), `sqlx` 0.9 `Acquire`/`Row` (`service.rs:3`), and `thiserror` 2
(`error.rs:1`) are all current for the pinned workspace versions
(`Cargo.toml:52-54`, `:85`, `:90`, `:96`). No `#[deprecated]` items and no
`#[allow(deprecated)]` in the crate.

### Documentation drift (repo docs vs. this crate)

`ARCHITECTURE.md:493-505` describes an older shape of this crate and is now wrong on
five points: the action count (10 vs 11), the hash pre-image field set (`event_id`,
`event_kind`, `channel_id` do not exist on `AuditEntry`, `entry.rs:14-37`), the
non-existent `AuditError::AuthEventForbidden` variant (`error.rs:12-41`), "64 zeros"
for a 32-byte constant (`hash.rs:9`), and an implied single global advisory lock
(actually per-community, `service.rs:29`, `:58`). Details in the Features aspect.


## Module: buzz-media (`crates/buzz-media`)

### Aspect: Debt

### 1. Complexity hotspots

| File | LOC | Non-test share | Note |
|---|---|---|---|
| `src/validation.rs` | 2594 | ~940 lines of logic; the `#[cfg(test)] mod tests` starts at `crates/buzz-media/src/validation.rs:941` and runs to EOF (~1650 lines, 64% of the file) | Single file holds five hand-written binary-format parsers plus three public validators |
| `src/bucket_index.rs` | 755 | logic ends ~`crates/buzz-media/src/bucket_index.rs:411`; tests `:413-661` | Well-separated pure logic |
| `src/upload.rs` | 732 | logic ends at `crates/buzz-media/src/upload.rs:560`; tests `:562-731` | Contains the longest function in the crate |
| `src/auth.rs` | 552 | logic `:1-236`; tests `:241-551` | |
| `src/upload_record.rs` | 419 | logic `:1-256`; tests `:258-418` | |
| `src/storage.rs` | 404 | logic + tests interleaved (tests at `:269-377`, then two public structs at `:378-403`) | Type definitions placed *after* the test module — unusual layout |

Largest functions:

| Function | Approx. lines | file:line |
|---|---|---|
| `process_video_upload` | ~220 | `crates/buzz-media/src/upload.rs:292-511` |
| `validate_video_file` | ~107 | `crates/buzz-media/src/validation.rs:289-395` |
| `validate_mp4_metadata_free` (incl. nested `walk`) | ~98 | `crates/buzz-media/src/validation.rs:831-928` |
| `verify_blossom_auth_event_for_verb` | ~115 | `crates/buzz-media/src/auth.rs:31-145` |
| `check_moov_before_mdat` | ~77 | `crates/buzz-media/src/validation.rs:408-484` |
| `validate_jpeg_metadata_free` | ~77 | `crates/buzz-media/src/validation.rs:502-578` |
| `validate_webp_metadata_free` (incl. nested fn) | ~74 | `crates/buzz-media/src/validation.rs:659-732` |
| `validate_gif_metadata_free` (incl. nested fn) | ~96 | `crates/buzz-media/src/validation.rs:734-829` |
| `process_buffered_upload` | ~139 | `crates/buzz-media/src/upload.rs:54-192` |
| `BucketAggregate::finish` | ~61 | `crates/buzz-media/src/bucket_index.rs:277-337` |

`process_video_upload` is a 7-step inline pipeline (stream+hash, container check, auth, validation, idempotency, store, sidecar) with a large nested block for step 1 (`crates/buzz-media/src/upload.rs:322-398`) — the natural extraction candidate, and the one path *not* covered by the shared `process_buffered_upload` refactor that unified the other two.

---

### 2. Dead / unused code

| Item | Evidence |
|---|---|
| 9 `MediaError` variants are never constructed in this crate: `MissingAuth`, `InvalidAuthScheme`, `InvalidBase64`, `Unauthorized`, `RelayMembershipRequired`, `TokenRevoked`, `PubkeyMismatch`, `UploadRateLimitExceeded`, `UploadConcurrencyLimitReached` (`crates/buzz-media/src/error.rs:44-63`, `:67-74`) — they exist for relay handlers that reuse the type (e.g. `crates/buzz-relay/src/api/media.rs:88-140`, `:887`). Legitimate shared-error-type coupling, but it means the crate's error enum is not self-contained. |
| `MediaError::TokenRevoked` in particular has **no revocation mechanism anywhere in this crate** — no nonce/jti store exists (`crates/buzz-media/src/auth.rs:31-236`). |
| `MediaStorage::delete` has no in-crate caller — no upload path, GC, or sweep uses it (`crates/buzz-media/src/storage.rs:158-164`); only the ignored MinIO test exercises it (`crates/buzz-media/tests/static_creds_minio.rs:71-74`). |
| `MediaStorage::get` has no in-crate caller either; `get_sidecar` uses `bucket.get_object` directly rather than going through it (`crates/buzz-media/src/storage.rs:197-199`) — a small inconsistency that duplicates the 404-mapping logic omission. |
| `MediaStorage::head_with_metadata`, `get_range`, `get_stream`, `list_page` are all relay-only consumers — fine, but nothing in-crate tests them (see coverage gaps). |
| `BlossomVerb::Get` / `verify_blossom_get_auth` are unused within the crate (`crates/buzz-media/src/auth.rs:207-236`); only the relay calls them (`crates/buzz-relay/src/api/media.rs:502`). |
| `MediaConfig::validate` is never called inside the crate (`crates/buzz-media/src/config.rs:66`) — startup enforcement depends on the relay invoking it. |
| `looks_like_iso_bmff` is public and used by the relay (`crates/buzz-relay/src/api/media.rs:50`), while `looks_like_mp4_iso_bmff` is `pub(crate)` — the two near-identical names with different visibility is a readability trap (`crates/buzz-media/src/validation.rs:48-62`). |

---

### 3. Test coverage gaps

104 tests total (98 `#[test]`, 6 `#[tokio::test]`, 1 `#[ignore]`). Distribution is heavily skewed toward validation.

| Gap | Evidence |
|---|---|
| **No test executes any upload pipeline.** `process_upload`, `process_file_upload`, `process_video_upload` have zero direct tests — `upload.rs`'s 4 tests cover only `build_descriptor` (×3) and the body-limit substring heuristic (`crates/buzz-media/src/upload.rs:565-730`). The 7-step ordering contract (blob → thumb → record → sidecar), the idempotent short-circuit, and the orphan-blob-on-failure behaviour are therefore untested in-crate. |
| **No test exercises `MediaStorage` against a store** except the `#[ignore]`d MinIO round-trip (`crates/buzz-media/tests/static_creds_minio.rs:44-75`). The 4 in-crate `storage.rs` tests cover credential selection and key formatting only (`crates/buzz-media/src/storage.rs:302-377`). |
| **404/error mapping untested**: the `HttpFailWithBody(404, _)` branches in `get`, `get_range`, `head`, `head_with_metadata`, and the `status_code == 404` check in `get_stream` have no coverage (`crates/buzz-media/src/storage.rs:105-175`). |
| `get_range`, `get_stream`, `put_file`, `list_page` have **no tests at all** (`crates/buzz-media/src/storage.rs:85-265`); `list_page`'s conversion into `Page` is the one place where the otherwise-pure sweep meets S3 and it is unverified. |
| `thumbnail.rs` has **zero tests** (`crates/buzz-media/src/thumbnail.rs:1-51`) — the 320px/blurhash/thumb-URL contract and the non-image early return are unverified. |
| `types.rs` and `lib.rs` have zero tests (serde renames like `type`/skip-if-none are covered indirectly via `upload.rs`'s descriptor tests). |
| `record_upload_event` itself is untested (`crates/buzz-media/src/upload_record.rs:139-178`); only the record's serialization shape, key format, and IP/port parsing are (`crates/buzz-media/src/upload_record.rs:258-417`). |
| `MediaError::into_response` covers only the 415 and 422 groups (`crates/buzz-media/src/error.rs:163-198`) — the collapsed-401 behaviour, the 403 split, 413, 429, and the 5xx body flattening are untested despite being explicit security decisions. |
| Video-path negative tests exist for `validate_video_file` but not for the streaming wrapper: the `MIN_SNIFF_BYTES` accumulation, the Content-Length fast-fail, and the mid-stream running-total cap are untested (`crates/buzz-media/src/upload.rs:311-395`). |
| The ignored MinIO test is the only integration coverage; the AWS-credential-chain branch (`Credentials::default()`) is intentionally never exercised (`crates/buzz-media/src/storage.rs:51-56`). |

---

### 4. Fragile / risky constructs

| # | Item | file:line |
|---|---|---|
| D-1 | Body-limit detection by matching three Display substrings, because "axum wraps LengthLimitError in its error chain but doesn't expose the inner type for downcasting" — a dependency-wording dependency. The guard test only proves the current strings, not that axum still produces one of them | `crates/buzz-media/src/upload.rs:328-341`, test `crates/buzz-media/src/upload.rs:685-712` |
| D-2 | `EMPTY_FFMPEG_UDTA` hardcodes 53 exact bytes of a specific ffmpeg output; any encoder change breaks uploads with a 422 | `crates/buzz-media/src/validation.rs:834-839`, `:895-903` |
| D-3 | Metadata policy is a strict allowlist over five binary formats, hand-implemented. Any new legitimate chunk/box from a client encoder version bump becomes a hard rejection (already visible in the fixture pairs: unsanitized iOS/Android output is rejected, sanitized accepted — `crates/buzz-media/src/validation.rs:1203-1280`) | `crates/buzz-media/src/validation.rs:487-928` |
| D-4 | `check_moov_before_mdat` uses unchecked `offset += atom_size` on an attacker-supplied 64-bit size, while the sibling `validate_mp4_metadata_free` walker uses `checked_add` — inconsistent overflow discipline | `crates/buzz-media/src/validation.rs:481` vs `:891-893` |
| D-5 | Two independent MP4 traversals (`check_moov_before_mdat` and `validate_mp4_metadata_free::walk`) each re-open and re-scan the file, plus a third parse by the `mp4` crate — three passes over the same temp file per video upload | `crates/buzz-media/src/validation.rs:291-307`, `:408`, `:831` |
| D-6 | Duration (`600.0`) and resolution (`3840`/`2160`) limits are inline literals rather than named constants or config, unlike every byte cap | `crates/buzz-media/src/validation.rs:357`, `:364` |
| D-7 | `max_image_bytes`/`max_gif_bytes` lack `#[serde(default)]` while `max_video_bytes`/`max_file_bytes`/`s3_region` have one — inconsistent defaulting policy inside one struct | `crates/buzz-media/src/config.rs:38-50` |
| D-8 | Auth windows (600 s / 3600 s) are magic numbers passed at call sites rather than config or named constants | `crates/buzz-media/src/upload.rs:85`, `:412` |
| D-9 | `BlobHeadMeta`/`BlobMeta` are declared *after* the `#[cfg(test)] mod tests` block in `storage.rs`, splitting the type definitions from the impl | `crates/buzz-media/src/storage.rs:268-403` |
| D-10 | 9 non-test `.unwrap()` calls on `try_into()` of fixed-size slices. All are provably infallible given the preceding bounds checks, but they violate the repo's "no new unwrap in production paths" guideline and would be free to convert | `crates/buzz-media/src/validation.rs:604`, `:605`, `:673`, `:674`, `:697`, `:706`, `:707`, `:885`, `:886` |
| D-11 | `blurhash::encode(...).unwrap_or_default()` silently substitutes an empty blurhash on failure | `crates/buzz-media/src/thumbnail.rs:36-37` |
| D-12 | `content_length.unwrap_or(0)` in `head_with_metadata` conflates "absent header" with "zero bytes" | `crates/buzz-media/src/storage.rs:170-172` |
| D-13 | Unchecked `+=` accumulation of `u64` byte totals in the sweep aggregate | `crates/buzz-media/src/bucket_index.rs:251-252` |
| D-14 | `SweepError::Timeout` is defined here but only ever constructed by the relay — an error variant whose producer lives in another crate (documented as deliberate, still a coupling smell) | `crates/buzz-media/src/bucket_index.rs:349-357` |
| D-15 | `get_sidecar` bypasses `MediaStorage::get`, so it does not get the 404→`NotFound` mapping and surfaces a generic `StorageError` for a missing sidecar | `crates/buzz-media/src/storage.rs:193-202` vs `:105-111` |
| D-16 | `read_sidecar_mime` splits on `'.'` and falls back to the whole string (`split('.').next().unwrap_or(sha256_ext)`) — tolerant parsing at a security-relevant seam (the tenant gate), relying on the relay's `validate_media_path` upstream | `crates/buzz-media/src/storage.rs:227-228` |

---

### 5. Functional gaps carried as debt

| Gap | Evidence |
|---|---|
| No GC / orphan cleanup — orphan blobs are knowingly accumulated ("A V2 background GC job can sweep blobs with no matching sidecar after a grace period") | `crates/buzz-media/src/upload.rs:122-127` |
| No retention/TTL logic of any kind | no lifecycle code in `crates/buzz-media/src/storage.rs:34-265` |
| No durable storage quota per pubkey or community; only in-flight admission limits exist, and they live in the relay | `crates/buzz-relay/src/api/media.rs:302-304` (TODO) |
| No Blossom `/list`, `DELETE /{sha256}`, or mirror support | `crates/buzz-media/src/lib.rs:17-28` (no such exports) |
| Audio uploads unsupported pending a sanitizer | `crates/buzz-media/src/validation.rs:186-196` |
| Inline PDF preview deferred | `crates/buzz-media/src/validation.rs:211-215` |
| No S3 retry or timeout policy | `crates/buzz-media/src/storage.rs:34-265` |
| No integrity re-verification on read | `crates/buzz-media/src/storage.rs:105-146` |
| Video thumbnails delegated to the desktop client rather than generated server-side | `crates/buzz-media/src/upload.rs:437-448` |

---

### 6. Deprecated API usage

None found. `rust-s3` 0.37's manual `list_page` is used deliberately over the auto-paginating `list` ("NOT the auto-paginating `list`, which has no cap" — `crates/buzz-media/src/storage.rs:235-241`). No `#[deprecated]` items, no `#[allow(deprecated)]`, no compiler-deprecation suppressions appear in the crate. The one non-standard dependency situation is the temporary workspace `[patch.crates-io]` pin of `aws-creds` to a fork, explicitly marked "Revert to crates.io once #449 lands upstream" (`Cargo.toml:163-171`).

---

### 7. Inconsistencies worth flagging

| # | Inconsistency |
|---|---|
| I-1 | Three different 404 detection mechanisms: `HttpFailWithBody(404, _)` matching (`crates/buzz-media/src/storage.rs:107`), `response.status_code == 404` (`crates/buzz-media/src/storage.rs:136-139`), and no check at all in `get_sidecar` (`crates/buzz-media/src/storage.rs:197-199`) |
| I-2 | Byte caps are configurable; pixel, duration, resolution, atom, box, and depth caps are hardcoded |
| I-3 | Overflow discipline differs between the two MP4 walkers (D-4) |
| I-4 | The buffered image and file paths were unified behind `process_buffered_upload`, but the video path duplicates all shared steps inline (`crates/buzz-media/src/upload.rs:54-192` vs `:292-511`) |
| I-5 | `ext` derivation has three separate implementations: `mime_to_ext` (`crates/buzz-media/src/validation.rs:930-939`), `file_mime_to_ext` + `infer` fallback (`crates/buzz-media/src/validation.rs:99-156`, `:203-206`), and a hardcoded `"mp4"` (`crates/buzz-media/src/upload.rs:420`) — and a fourth, independent notion of a valid ext in the sweep classifier (`crates/buzz-media/src/bucket_index.rs:84-87`) |
| I-6 | `_meta/` and `_uploads/` key builders live in different modules (`storage.rs` vs `upload_record.rs`) while their parsers both live in `bucket_index.rs` — the writer/reader pair for the same layout is split across three files, so a layout change must be made in three places (the sweep tests are the only cross-check) |


## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Debt

---

### 1. Incomplete features carrying explicit ticket tags

| Item | State | Ticket | Line |
|---|---|---|---|
| `send_dm` | returns `NotImplemented`, fails the run | WF-07 | `executor.rs:580-584` |
| `set_channel_topic` | returns `NotImplemented`, fails the run | WF-07 | `executor.rs:586-590` |
| `request_approval` | token generated, nothing persisted, no kind-46010 emission; `finalize_run` turns the suspension into `Failed` | WF-08 | `executor.rs:650-668`, `lib.rs:192-215` |
| Long delays | capped at 270 s; "scheduled resume pattern" deferred | WF-09 | `executor.rs:673-684` |

---

### 2. Complexity hotspots

| Location | Size / shape | Notes |
|---|---|---|
| `WorkflowEngine::run` | `lib.rs:428-672` — ~245 lines, single `loop` with a nested per-workflow body, 5 sequential DB calls, two `continue`-heavy match arms, and inline `tokio::spawn` | The largest function in the crate. Cron vs interval branching, claim handling, trigger-context building, run creation, claim attachment, anchor updates, and map pruning are all inlined; only the two instant-computation helpers and the interval prefilter are extracted |
| `dispatch_action` | `executor.rs:519-692` — ~175 lines, 7 arms, two of which contain `#[cfg]`-split bodies | The `send_message` arm performs two DB round-trips inline (`executor.rs:535-556`); `#[cfg(feature = "reqwest")]` / `#[cfg(not(...))]` duplication in `add_reaction` and `call_webhook` doubles the surface to review |
| `on_event` | `lib.rs:276-383` — ~108 lines mixing cache management, per-workflow deserialization, gating, run creation and task spawning | |
| `should_fire_workflow` | `lib.rs:806-882` — the `MessagePosted` filter block (`:824-846`) and `DiffPosted` filter block (`:848-880`) are near-identical copies differing only in the matched variant | Straight duplication; a shared `filter_of(&TriggerDef) -> Option<&str>` would collapse both |
| `execute_steps` | `executor.rs:1083-1217` — five distinct early-return error paths, each rebuilding `PartialProgress { step_index: i, trace }` by moving `trace` | Repeated construct at `:1116-1120`, `:1129-1133`, `:1161-1165`, `:1169-1176` |
| `resolve_send_message_channel` | `executor.rs:468-517` — four-branch precedence ladder with two separate UUID parses producing the same error text | |
| `parse_duration_secs` | `executor.rs:705-735` — three near-identical suffix blocks | |
| `finalize_run` | `lib.rs:175-263` — three DB-write branches each with its own error log; approval branch is dead-weight until WF-08 lands | |
| File sizes | `executor.rs` 1834 LOC (≈1215 non-test, test module starts `executor.rs:1219`), `lib.rs` 1564 LOC (≈965 non-test, `lib.rs:966`), `schema.rs` 878 LOC (≈269 non-test, `schema.rs:270`) | Test code is roughly 40% of `lib.rs` and 70% of `schema.rs` |

---

### 3. Dead / unreachable code

| Item | Why it is dead | Line |
|---|---|---|
| `StepResult::Skipped` | `dispatch_action` never constructs it (it returns only `Completed` or `Suspended`), so the handling arm in `execute_steps` is unreachable. Condition-based skipping is handled earlier by `continue`, not by this variant | `executor.rs:463-464`, `executor.rs:1200-1206`, `executor.rs:1096-1112` |
| `finalize_run` approval branch | Reachable only if a run suspends, and its sole purpose is to convert that into `Failed` — i.e. the approval feature's "success" path is a failure path | `lib.rs:192-215` |
| `ExecutionResult.step_outputs` | Populated on every return but never read by any caller in the repo (relay call sites pass the whole `Result` to `finalize_run`, which reads only `trace`, `step_index`, `approval_token`) | `executor.rs:949`, `lib.rs:186-238` |
| `execute_run` | No caller outside this crate; both internal callers are in `lib.rs`. Meanwhile all three relay entry points use `execute_from_step(..., 0, None)` instead, duplicating `execute_run`'s behaviour | `executor.rs:970`, `lib.rs:373`, `lib.rs:651`; relay: `crates/buzz-relay/src/handlers/command_executor.rs:926`, `crates/buzz-relay/src/api/bridge.rs:1908` |
| `build_trigger_context` (`pub`) | No caller outside this crate | `lib.rs:884` |
| `resolve_template`, `resolve_step_templates`, `build_eval_context`, `evaluate_condition`, `dispatch_action` (`pub`) | All public, none called outside this crate | `executor.rs:70`, `:224`, `:350`, `:390`, `:519` |
| `generate_approval_token` parameters | `_run_id` and `_step_id` are accepted and deliberately unused | `executor.rs:698-700` |
| Approval-resume machinery end-to-end | `execute_from_step`'s `start_index`/`initial_outputs` path is exercised only with `(0, None)`. The relay's `resume_workflow` gates on `run.status == WaitingApproval` (`crates/buzz-relay/src/handlers/command_executor.rs:1253`), and **no code in the repository ever writes `WaitingApproval`** — grep finds it only in that guard, a sibling guard at `:1197`, and the comment at `lib.rs:193`. `Db::create_approval` likewise has no caller outside `buzz-db` | `executor.rs:1018-1075` |
| `add_reaction_impl` target route | Posts to `/api/messages/{id}/reactions`, which the relay router does not register (`crates/buzz-relay/src/router.rs:39-125`) — the code is live but cannot succeed | `executor.rs:888-894` |

---

### 4. Stale documentation inside the crate

| Claim | Reality | Line |
|---|---|---|
| "Action dispatch uses placeholder implementations that log intent. Real event emission is wired in WF-07/08 (relay integration)." | `send_message` emits real events through `ActionSink`; `call_webhook` performs real HTTP | `executor.rs:9-10` |
| "For MVP, most actions log their intent and return a success output." | Only `send_dm`/`set_channel_topic` are log-and-fail stubs | `executor.rs:521-522` |
| `CallWebhook.url` — "must be a public HTTPS endpoint" | No scheme validation; `http://` accepted | `schema.rs:120` vs `executor.rs:786-798` |
| `RequestApproval.timeout` — "Defaults to 24h" | Never parsed; used only in a log line | `schema.rs:138-140` vs `executor.rs:653-658` |
| Crate-root usage example shows `engine.on_event(...)` then `engine.run()` | Accurate, but omits the mandatory `set_action_sink` call without which `send_message` fails | `lib.rs:19-31` vs `lib.rs:148-156` |
| `Db::create_workflow` doc "(No current callers.)" | Confirms the workflow creation path is `upsert_workflow` only | `crates/buzz-db/src/workflow.rs:272-275` |

---

### 5. Test coverage gaps

149 tests total (127 `#[test]`, 22 `#[tokio::test]`) split `schema.rs` 50, `lib.rs` 38, `executor.rs` 61. All are pure-function tests. Untested:

| Untested surface | Reason | Line |
|---|---|---|
| `WorkflowEngine::on_event` | needs a `Db` (Postgres) | `lib.rs:276` |
| `WorkflowEngine::run` (scheduler loop) | needs `Db` + a 60 s tick; only the extracted helpers `cron_fire_instant`, `interval_fire_instant`, `interval_should_fire`, `interval_prefilter_should_fire` are covered | `lib.rs:428` |
| `WorkflowEngine::finalize_run` | needs `Db`; the approval→`Failed` mapping — the single most surprising behaviour in the crate — has no test | `lib.rs:175` |
| `execute_run` / `execute_from_step` / `execute_steps` | need `Db`; therefore step sequencing, per-step timeout, `StepTimeout`, partial-progress trace accumulation, resume-from-index, and semaphore admission (`CapacityExceeded`) are all untested | `executor.rs:970`, `:1018`, `:1083` |
| `dispatch_action` (all 7 arms) | needs `WorkflowEngine`; only the extracted `resolve_send_message_channel` has 3 tests (`executor.rs:1804-1836`) | `executor.rs:519` |
| `check_ssrf`, `call_webhook_impl`, `add_reaction_impl` | no HTTP test server, no `is_private_ip` integration test in this crate | `executor.rs:745`, `:781`, `:888` |
| `WorkflowEngine::new` / `set_action_sink` / `invalidate_channel_workflows` | no test constructs an engine, so the double-init panic and cache invalidation are unverified here | `lib.rs:109`, `:139`, `:131` |
| `MAX_DELAY_SECS` boundary | no test asserts that a 271 s delay is rejected or that 270 s is accepted | `executor.rs:676` |
| `WorkflowError` display strings / `From<DbError>` | `error.rs` has zero tests | `error.rs` |
| `ActionSink` / `ActionSinkError` | `action_sink.rs` has zero tests | `action_sink.rs` |
| Trigger-context `webhook_fields` shadowing defence | the "webhook keys cannot spoof `trigger_*`" skip logic (`executor.rs:287-291`) has no negative test; only the positive case is tested (`executor.rs:1703-1711`) | `executor.rs:285-296` |
| `evaluate_condition` timeout path | `MAX_EXPR_LEN` rejection is tested (`executor.rs:1395-1412`), but the 100 ms timeout branch is not | `executor.rs:370-380` |

Relay-side integration coverage exists but explicitly documents the approval gap — see `crates/buzz-test-client/tests/conformance_multitenant.rs:1863-1946`, which states the approval-request half is unreachable ("see WF-08") and that `create_approval` is reached only from unit tests.

---

### 6. Design debt / operational risk

| Item | Detail | Line |
|---|---|---|
| Non-cancellable blocking evaluation | The code documents that `spawn_blocking` work survives its own timeout; the mitigation is a length cap, not cancellation. Enough concurrent pathological expressions could saturate the blocking pool | `executor.rs:355-380` |
| Permit held across sleeps | A `delay` step holds a concurrency permit for up to 270 s, so ~100 delaying runs exhaust admission and new runs are rejected with `CapacityExceeded` (which finalizes as `Failed`, not retried) | `executor.rs:978`, `executor.rs:685`, `lib.rs:242-261` |
| No retry / backoff anywhere | Any step error terminates the run permanently; there is no dead-letter, no re-drive, and no idempotency key for side effects | `executor.rs:1160-1176` |
| Fire-and-forget spawns | Runs are spawned detached with no `JoinHandle` tracking, so a relay shutdown mid-run leaves the row in `Running` forever — there is no reaper for stuck `Running` rows in this crate | `lib.rs:371-381`, `lib.rs:649-661` |
| Interval anchors are in-memory | `last_fired` is a `DashMap`, documented as "lost on restart. Missed fires during downtime are not replayed (acceptable for MVP)"; the durable claim row is the only cross-restart guard | `lib.rs:84-86`, `lib.rs:494-520` |
| Cron scan is unbounded | `list_all_enabled_workflows()` loads every enabled workflow in every community each tick, then filters to schedule triggers in-process | `lib.rs:436` |
| Cache staleness accepted by design | No cross-pod invalidation; the documented worst case is a just-deleted workflow firing (or a new one missing events) for up to 10 s | `lib.rs:92-103` |
| Trace can grow to megabytes | Each `call_webhook` step can append up to 1 MiB of response body into `execution_trace`, and the full array is rewritten on every status update | `executor.rs:865-868`, `executor.rs:1179-1183` |
| Two HTTP client strategies | Per-request client for webhooks (needed for DNS pinning) vs a `LazyLock` shared client for reactions — inconsistent, and the shared one lacks the SSRF/redirect/cap hardening | `executor.rs:800-815` vs `executor.rs:874-885` |
| Duplicated entry points | `execute_run` and `execute_from_step(…, 0, None)` are behaviourally equivalent for fresh runs but differ in trace handling (the latter reads the existing trace first, `executor.rs:1037-1045`), and the relay uses only the latter | `executor.rs:970`, `:1018` |
| Direct (non-workspace) dependency versions | `evalexpr = "11"` and `cron = "0.16"` are pinned in the crate rather than the workspace table, unlike every other dependency | `Cargo.toml:20-21` |
| No deprecated API usage found | Grep found no `#[deprecated]`, no `#[allow(deprecated)]`, and no deprecation warnings referenced in the crate | — |


## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Debt

Severity scale: **S1** = correctness/security bug reachable today; **S2** = latent bug, operational hazard, or misleading documentation that would cause a wrong decision; **S3** = maintainability / consistency.

---

#### S1 — Live defects

##### D-01 (S1) Workflow `add_reaction` POSTs to a route that does not exist
`crates/buzz-workflow/src/executor.rs:892` constructs `POST {BUZZ_RELAY_BASE_URL}/api/messages/{message_id}/reactions`. No such route is registered in `router.rs:37-131`, `api/git/transport.rs:1760-1762`, `api/git/mod.rs:62`, or `api/admin/mod.rs:30-37`. The request falls through to the SPA fallback's 404 (`router.rs:178`) or a bare axum 404, so every workflow that adds a reaction fails silently at the HTTP layer. The only two mentions of `api/messages` in the workspace are the doc comment at `crates/buzz-workflow/src/executor.rs:886` and the URL at `:892`. Confirmed.

Also worth noting: this whole path violates the repo's own "prefer Nostr events over new HTTP endpoints" rule (AGENTS.md § Key Patterns) — the fix is a kind, not a route.

##### D-02 (S1) Three rate-limit env vars are parsed, validated, stored, and never read
`config.rs:303-306`, `:307-310`, `:311-314` parse `BUZZ_RATE_LIMIT_AGENT_STANDARD_API_CALLS_PER_MIN`, `..._AGENT_ELEVATED_MESSAGES_PER_MIN`, `..._AGENT_PLATFORM_MESSAGES_PER_MIN` and hard-error on invalid values, giving an operator every signal that they take effect. Verified across `crates/**` and `desktop/src-tauri/**`: the field names appear only in their declarations (`crates/buzz-auth/src/rate_limit.rs:101,104,107`) and default assignments (`:139,140,141`). Zero readers. Two of the three name enforcement tiers ("elevated", "platform") that have no code path at all. The four live siblings are consumed at `api/bridge.rs:29`, `connection.rs:614`, `connection.rs:634`, `connection.rs:636`.

##### D-03 (S1) Two background intervals have no lower bound → busy-loop on `0`
`BUZZ_REAPER_INTERVAL_SECS` (`main.rs:609-612`) and `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS` (`main.rs:701-704`) parse straight to `u64` with no floor. Set to `0`, `tokio::time::sleep(Duration::ZERO)` turns each task into a hot loop issuing `reap_expired_ephemeral_channels` / `query_due_reminders` continuously against Postgres. The two neighbouring pollers were explicitly floored for exactly this hazard: `main.rs:949` (`.max(1)` with the comment "tokio::time::interval panics on Duration::ZERO") and `main.rs:1261` (`.max(5)` "with a floor that prevents a busy loop"). `BUZZ_NIP43_RECONCILE_INTERVAL_SECS` has `.max(1)` (`main.rs:521`) and `BUZZ_COMMUNITY_REVALIDATE_INTERVAL_SECS` has `.clamp(1,300)` (`main.rs:886`). Two of six lack the guard.

##### D-04 (S1) Redis broadcast consumers die permanently on `RecvError::Closed`
`main.rs:838-841` (multi-node fan-out), `main.rs:868-871` (cache invalidation), `main.rs:929-932` (conn-control) all `break` out of their loop on `Closed`, log one `error!`, and end the task. The pod keeps serving traffic and keeps reporting `/_readiness: ready` (readiness only pings Postgres + Redis, `router.rs:352-373`) while:
- events from other pods are no longer fanned out to local subscribers,
- membership/visibility changes on other pods no longer drop local caches,
- bans recorded on other pods no longer close local sockets.

No restart, no supervision, no gauge for "consumer alive". The cache case degrades to a ≤10 s TTL wait (tolerable, `state.rs:960-963`); the fan-out and ban cases degrade to silent cross-pod incoherence.

##### D-05 (S1) Reminder scheduler can permanently orphan a reminder
`main.rs:781-798`: after winning the claim, if the Redis publish fails **and** the compensating `release_due_reminder` also fails, the code logs `warn!("… reminder stays claimed, will not retry")` and moves on. The `delivered_at` sentinel stays set, the partial index excludes the row, and no later tick or pod will ever pick it up. The comment states the outcome plainly; there is no dead-letter path, no metric, and no reconciliation sweep for stuck claims.

##### D-06 (S1) `expect` in the fan-out hot path
`state.rs:444-447`: `send_to_text_bytes` does `WsUtf8Bytes::try_from(...).expect("relay fan-out frames are serialized UTF-8 JSON")`. The doc says "Callers must only pass valid UTF-8 bytes" — an unenforceable contract on a `pub fn` taking `Arc<Bytes>`. A single non-UTF-8 payload panics the fan-out task. A `Result` or a `debug_assert` + drop would be equivalent in the happy path and non-fatal in the bad one.

Similarly `main.rs:1253` `serde_json::from_value(event_json).expect("valid event JSON from DB row")` panics the whole reminder-scheduler task on one malformed row, taking down reminder delivery for the pod.

##### D-07 (S1) Hard `process::exit(1)` bypasses the audit drain and span flush
`main.rs:1153` exits with code 1 after a 30 s drain timeout. The audit drain (`main.rs:1049`) and OTEL shutdown (`main.rs:1053`) are downstream of `serve()` returning, so an overrunning drain loses every buffered audit entry (up to 1,000, `state.rs:654`) and every pending span. It also reports a *failed* termination to Kubernetes for what is a graceful-shutdown overrun. A 30 s drain that is still holding connections is exactly the case where the audit trail matters most.

##### D-08 (S1) `/_mesh` and `/metrics` are unauthenticated on hard-coded `0.0.0.0`
`main.rs:1116` binds the health router to `("0.0.0.0", config.health_port)` and `metrics.rs:74` binds the exporter to `([0,0,0,0], port)` — both ignore `BUZZ_BIND_ADDR`'s address. `GET /_mesh` (`router.rs:230` → `router.rs:380-386`) serialises the full `MeshStatus`, including every peer's `endpoint_addrs` (`crates/buzz-relay-mesh/src/status.rs:20`) — internal socket addresses of the relay fleet. `GET /metrics` publishes per-community gauges labelled with the tenant **host** (`main.rs:1341-1359`, `:1546-1552`) — precisely the enumeration the `404` at `router.rs:292-297` is designed to prevent. `GET /_readiness` additionally discloses which backing store is down (`router.rs:369-372`). No auth, no CORS restriction, no rate limit, no way to bind them to loopback.

##### D-09 (S1) Two unbounded per-pubkey `DashMap`s
`observer_rate_limiter` (`state.rs:589`) and `media_upload_rate_limiter` (`state.rs:592`) are `DashMap<(CommunityId,[u8;32]), (u32, Instant)>` with **no capacity cap, no TTL, and no eviction anywhere in `state.rs`**. Keys are attacker-chosen pubkeys. The sibling field three lines below spells out the exact threat model and defends against it: `invite_claim_rate_limiter`'s doc says "the cache has a hard capacity because pre-membership callers can cheaply generate fresh Nostr keys" (`state.rs:593-596`) and uses a bounded moka cache (`state.rs:775-780`). The same reasoning applies verbatim to the two `DashMap`s; the defence was not applied. `media_uploads_in_flight` (`state.rs:600`) has the same shape (decremented by `api/media.rs`, so bounded in the happy path only).

##### D-10 (S1) Per-IP connection limiting is implemented and never called
`RateLimiter::check_ip_connection` is declared at `crates/buzz-auth/src/rate_limit.rs:188` and implemented by `RedisRateLimiter` at `crates/buzz-pubsub/src/rate_limiter.rs:112`. The only other reference in the workspace is the test stub at `admission.rs:85`. **Zero production callers.** The relay therefore has no IP-level connection-flood control; the only bound is `conn_semaphore` at 10,000 (`config.rs:449-452`).

Compounding it: the client IP *is* derived (`router.rs:236-240`), passed to `handle_connection` (`router.rs:316`), and stored on `ConnectionState::remote_addr` (`connection.rs:61`, populated `:170`) — and then **never read in production**. The only reads are test fixtures (`state.rs:1358`, `handlers/event.rs:1388`). The IP is captured and discarded: not logged, not audited, not limited on.

##### D-11 (S1) `/metrics` per-community host labels leak tenant identity by default
Same code path as D-08, but the *default* matters independently: `EmissionScope::from_env` defaults to `All` (`main.rs:65`), so a fresh deployment publishes one labelled series per community per metric. `BUZZ_USAGE_METRICS_PER_COMMUNITY=off` disables it, but nothing in `.env.example` mentions the variable exists.

---

#### S2 — Latent bugs, operational hazards, doc drift

##### D-12 (S2) `AppState::audit` is a completely unread field
`state.rs:496` declares `pub audit: Option<Arc<AuditService>>`, written at `state.rs:718`. Grep across `crates/**` and `desktop/src-tauri/**`: **no read anywhere**. All audit writes go through the separate `audit_tx` channel (`state.rs:555`, `:763`), and the `is_some()` gate uses the local `audit_enabled` (`state.rs:716`), not the field. The `AuditService` stays alive only because the worker closure captured `audit_for_worker` (`state.rs:656`). Pure retained state on a struct that is cloned into every request.

##### D-13 (S2) `RelayError` — 9 of 10 variants dead, and it is unusable by HTTP handlers
`error.rs:8-48` declares 10 variants. Only `InvalidMessage` (`error.rs:44`) is ever constructed, exclusively in `protocol.rs:43-171`. Dead: `WebSocket:11`, `Json:15`, `Database:19`, `Auth:23`, `PubSub:27`, `ConnectionLimitReached:31`, `RateLimitExceeded:35`, `NotAuthenticated:39`, `Internal:47`. The four `#[from]` conversions (`error.rs:15,19,23,27`) are therefore unreachable. `crate::error::Result` has exactly one importer (`protocol.rs:6`). There is no `impl IntoResponse for RelayError`, which is why HTTP handlers invented a third error style (raw `Response` tuples). The type is 50 lines of scaffolding for one parse-error variant.

##### D-14 (S2) All three `lib.rs` re-exports are unused, and nothing depends on the crate as a library
`lib.rs:53-55` re-exports `Config`, `RelayError`, `Result`, `AppState`. No workspace crate declares `buzz-relay` as a dependency (`buzz-admin`, `buzz-conformance`, `buzz-relay-mesh`, `git-sign-nostr` mention it only in comments), and `main.rs:17-24` imports through full module paths (`buzz_relay::config::Config`, `buzz_relay::state::AppState`). The entire 21-module `pub` surface exists to serve one in-crate binary, which means `#![warn(missing_docs)]` is doing documentation work for an audience of zero and every `pub` should probably be `pub(crate)`.

##### D-15 (S2) Postgres pool accounting in `DbConfig`'s doc is wrong
`crates/buzz-db/src/lib.rs:244-246` states "At 20 main + 5 audit = 25/pod, four relay pods fit within the PG limit" (PG `max_connections=100`). `main.rs` opens a **third** pool for search (`main.rs:378-382`) with **no sizing knobs set at all** — sqlx defaults apply — and a fourth (replica) at the same 20 when `READ_DATABASE_URL` is set (`crates/buzz-db/src/lib.rs:363` → `:381-386`). Real ceiling per pod: 20 + 5 + search-default (+20 with a replica). The four-pod arithmetic does not hold, and the two extra pools are **not instrumented** — `main.rs:955-985` reports only `db.pool_stats()`, `db.read_pool_stats()`, and Redis. A search-pool exhaustion is invisible.

##### D-16 (S2) ARCHITECTURE.md claims rate limiting does not exist — three places
- `ARCHITECTURE.md:390`: "No Redis-backed rate limiter exists anywhere in the codebase — rate limiting is not currently enforced."
- `ARCHITECTURE.md:459`: buzz-pubsub "**Does NOT:** implement the rate limiter."
- `ARCHITECTURE.md:823` (§9 Known Limitations #2, header at `:816`): "Only implementation is `AlwaysAllowRateLimiter` (test stub) … none are enforced." §9 is prefaced by "These are verified gaps in the current implementation — not design aspirations."

All three are false. `RedisRateLimiter` lives at `crates/buzz-pubsub/src/rate_limiter.rs:88-99` (i.e. in the very crate `:459` says does not implement it), is constructed at `state.rs:712`, held at `state.rs:584`, and enforced at `api/bridge.rs:31`, `connection.rs:616`, and `connection.rs:639` — fail-closed on limiter error (`admission.rs:29-33`). A reader trusting §9 would conclude rate limiting needs building from scratch and would likely build a second, competing limiter.

The *accurate* residual gap is narrower and is D-02 + D-10: 3 of 7 configured tiers have no reader, and per-IP limiting is implemented but never called.

##### D-17 (S2) NIP-11 `max_limit` over-advertises by 10×
`nip11.rs:106` advertises `max_limit: Some(10_000)`. `crates/buzz-db/src/event.rs:347-348`: `let clamp = q.max_limit.unwrap_or(1000); let limit_val = q.limit.unwrap_or(100).min(clamp);`. Ordinary REQ handling does not raise `max_limit` (the only setter is the COUNT-fallback path, `handlers/req.rs:755`). A client honouring the advertised limit and requesting 10,000 events silently receives 1,000.

##### D-18 (S2) NIP-45 (COUNT) and NIP-98 are implemented but not advertised
`SUPPORTED_NIPS` (`nip11.rs:15`) omits 45 despite `ClientMessage::Count` (`protocol.rs:36`), `RelayMessage::count` (`protocol.rs:211`), `handlers/count.rs:285`, and `POST /count` (`router.rs:73`) all existing. It also omits 98 despite NIP-98 auth being mandatory on 12 routes (`api/bridge.rs:111`, `api/invites.rs:193`, `api/operator.rs`). Nostr clients use `supported_nips` for capability negotiation; both omissions cause clients to avoid features the relay supports.

##### D-19 (S2) `due_delivery_mode: "push"` advertised unconditionally
`nip11.rs:112` hard-codes `due_delivery_mode: Some("push")`. When `push_gateway_delivery_url` is `None` the matcher and delivery worker are never spawned (`main.rs:684-691`), so the relay advertises push delivery it cannot perform. `restricted_writes: true` (`nip11.rs:111`) is likewise unconditional even on a fully open relay (`require_relay_membership=false`, `pubkey_allowlist_enabled=false`).

##### D-20 (S2) The NIP-43/`self` consistency check is compiled out of release builds
`nip11.rs:143-146` uses `debug_assert!` to catch `advertise_nip43=true` without `relay_self`. A release build ships the inconsistent document — clients would be told the relay speaks NIP-43 with no key to verify NIP-43 events against. The test `nip11.rs:513-517` proves the assert fires in debug only. Given `nip11.rs:139-142` calls this "a programmer error", a runtime `if` that drops NIP-43 from the list would be strictly safer.

##### D-21 (S2) Stale `#[ignore]` instructions in `tenant.rs`
`tenant.rs:225-236` and `:238-243` instruct the reader to "Delete this `#[ignore]` when the fix lands" and give a `cargo test --include-ignored` invocation. The fix landed (`tenant.rs:81-88`) and the attributes were removed — there are **zero** `#[ignore]` attributes anywhere in this file group. The instructions now describe a state that does not exist and imply the tests are still red gates.

##### D-22 (S2) `disconnect_pubkey_clusterwide`'s "unrepresentable" claim is false
`state.rs:1026-1030`: "Callers must not invoke the pod-local `conn_manager.disconnect_pubkey` directly … Pairing both halves here makes that mistake unrepresentable." `ConnectionManager::disconnect_pubkey` is `pub` (`state.rs:310`) and is called directly at `main.rs:918`. That call is *correct* (it is the receiver side of the cross-pod command, which must not re-publish), but the doc neither carves out the exception nor makes the mistake unrepresentable. `pub(crate)` plus a named receiver-side wrapper would make the claim true.

##### D-23 (S2) Weak-by-default secrets that silently activate in production
| Item | Default | Cite |
|------|---------|------|
| relay signing key | hard-coded `0000…0001` whenever `BUZZ_RELAY_PRIVATE_KEY` is unset **and** `require_auth_token=false` (itself the default) | `main.rs:396-408`, `config.rs:475-477` |
| `BUZZ_GIT_HOOK_HMAC_SECRET` | fresh random per **process** ⇒ different secret on every pod in a multi-pod deployment | `config.rs:739-744` |
| `BUZZ_S3_ACCESS_KEY` / `BUZZ_S3_SECRET_KEY` | `buzz_dev` / `buzz_dev_secret` | `config.rs:622-625` |
| `DATABASE_URL` | `postgres://buzz:buzz_dev@localhost:5432/buzz` | `config.rs:410-411` |

Each is individually reasonable for dev; composed with the five permissive auth defaults (`require_auth_token=false`, `pubkey_allowlist_enabled=false`, `require_relay_membership=false`, `require_media_get_auth=false`, permissive CORS) a bare deployment is fully open with a well-known signing key. Only one of the five emits a warning (`config.rs:590-594`).

##### D-24 (S2) `Config` derives `Debug` over six secret-bearing fields with no redaction
`config.rs:50` `#[derive(Debug, Clone)]` covers `database_url:55`, `read_database_url:58`, `redis_url:60`, `relay_private_key:92`, `git_hook_hmac_secret:239`, and `media.s3_secret_key`/`s3_access_key` (populated `config.rs:622-625`). No `SecretString`, no `Zeroize`, no manual `Debug`. Verified that no current call site prints it (`main.rs:128-136` selects 6 safe fields; `AppState`'s manual `Debug` prints 2, `state.rs:1209-1215`) — so this is a loaded gun, not a live leak. One `tracing::debug!(?config)` added during future debugging exfiltrates the PG password, the relay private key, the S3 secret, and the git hook HMAC into the log pipeline.

Related, and live: `config.rs:538-542` echoes the raw `RELAY_OWNER_PUBKEY` value into a `warn!` when it fails validation — an nsec pasted into the wrong variable would be logged verbatim.

##### D-25 (S2) `.env.example` documents 5 of 93 vars and 2 that no longer exist
`.env.example` (233 lines) names 12 variables, only 5 of which this module reads. Missing: every security switch (`BUZZ_REQUIRE_AUTH_TOKEN`, `BUZZ_REQUIRE_RELAY_MEMBERSHIP`, `BUZZ_PUBKEY_ALLOWLIST`, `BUZZ_REQUIRE_MEDIA_GET_AUTH`, `BUZZ_CORS_ORIGINS`, `BUZZ_RELAY_PRIVATE_KEY`, `RELAY_OWNER_PUBKEY`, `RELAY_OPERATOR_PUBKEYS`, `RELAY_OPERATOR_API_ORIGIN`, `BUZZ_AUDIT_ENABLED`, `BUZZ_AUTO_MIGRATE`, `BUZZ_ADMIN_HOST`). Present but dead: `TYPESENSE_API_KEY` (`.env.example:40`), `TYPESENSE_URL` (`.env.example:41`) — no Rust code reads either; search is Postgres FTS (`main.rs:369-372`, `handlers/event.rs:482-483`). AGENTS.md's onboarding step is `cp .env.example .env`, so the documented path produces a fully permissive relay with no hint the switches exist.

##### D-26 (S2) No security headers outside the admin router
`api/admin/mod.rs:43-56` sets `Cache-Control: no-store`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, and a referrer policy — on that router only. The public app router (`router.rs:185-191`), the SPA fallback that serves `text/html` from `web_dir` (`router.rs:214-219`), the health router (`router.rs:225-232`), and `/metrics` have no CSP, no HSTS, no `nosniff`, no frame protection.

##### D-27 (S2) Unauthenticated NIP-11 costs up to 3 Postgres queries per request
`nip11.rs:235-263`: `workspace_icon_for_host` does `bind_community` (`:278`) + `get_community_icon` (`:283`), then `nip11_document` does a **second** `bind_community` (`:246`) whenever push is configured — which is the default (`config.rs:332`, `:752-757`). No cache, no rate limit, on `GET /` and `GET /info`. The two `bind_community` calls resolve the identical host and could trivially share one result.

##### D-28 (S2) `SPROUT_MAX_NOT_BEFORE_DELTA` is read from the environment on every NIP-11 request
`nip11.rs:96-100` calls `std::env::var` + `parse` inside `relay_limitation`, which runs per request. Every other advertised limit comes from `Config`. Inconsistent, and a per-request syscall on an unauthenticated path.

##### D-29 (S2) Legacy `SPROUT_` prefix survives on three env vars
`SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS` (`main.rs:701`), `SPROUT_REMINDER_SCHEDULER_BATCH_LIMIT` (`main.rs:705`), `SPROUT_MAX_NOT_BEFORE_DELTA` (`nip11.rs:97`). Everything else in the repo has been renamed to `buzz`/`BUZZ_`. Renaming is a breaking change for deployed manifests, so this needs an alias-with-deprecation, not a silent rename.

##### D-30 (S2) `AppState::community_disconnect_publish_attempts` is test-only telemetry on a production struct
`state.rs:512` is incremented on every community-archive publish (`state.rs:1063`) but read only at `api/operator.rs:1010,1020`, both inside `#[cfg(test)]` (module opens `api/operator.rs:500`). Its own doc admits "Test/telemetry counter". A production `AtomicU64` field and a fetch_add on a live code path to serve assertions.

##### D-31 (S2) Eight distinct boolean grammars for env vars
Within one config surface: strict `parse_bool` (`true/1/on` vs `false/0/off/""`, error otherwise — `config.rs:363-377`); bare `=="true"||=="1"` silently false otherwise (`config.rs:475,479,483,520,651,848`); **inverted** `!(=="false"||=="0")` (`config.rs:489`); case-insensitive `on` plus `true`/`1` (`config.rs:498,516`); `true|1|yes|on` (`config.rs:682-689`); trimmed case-insensitive `true|1|yes|on` (`main.rs:25-33`); `!= "false"` (`main.rs:469-471`); mere presence via `is_ok()` (`main.rs:549`). An operator cannot predict whether `BUZZ_X=yes` works without reading the source for that specific variable.

Matching split on numerics: hard error for `BUZZ_RATE_LIMIT_*` and `BUZZ_PUSH_GATEWAY_TIMEOUT_MS`; silent default for ~25 others, one of which has a test *asserting* the silent fallback (`config.rs:988-1011`).

---

#### S3 — Maintainability

##### D-32 (S3) File and function sizes
`main.rs` is 1,940 lines with a **977-line `main()`** (`main.rs:83-1060`) that inlines 19 `tokio::spawn` bodies, three multi-branch DB bootstrap sequences, and the full startup narrative. `state.rs` is 1,932 lines. `config.rs` is 1,346 lines with a **~470-line `Config::from_env()`** (`config.rs:405-874`). Extractable units are obvious and self-labelling: the metrics poller (`main.rs:1279-1806`, ~530 lines) is already a coherent module; the NIP-43 bootstrap block (`main.rs:199-320`), the pub/sub consumer trio (`main.rs:804-936`), and the reminder scheduler (`main.rs:693-802`) each stand alone. The mobile app enforces a 1,000-line ceiling (`mobile/scripts/check-file-sizes.mjs`, AGENTS.md); the relay has no equivalent guard.

##### D-33 (S3) Duplicated methods and duplicated constants
- `ConnectionManager::pubkey_for` (`state.rs:425-430`) and `ConnectionManager::pubkey_for_conn` (`state.rs:286-291`) are **byte-identical** in signature, body, and intent. Both live: `pubkey_for` ← `handlers/event.rs:483`; `pubkey_for_conn` ← `handlers/event.rs:146/184`, `handlers/side_effects.rs:108`.
- `health_handler` (`router.rs:295-297`) and `liveness_handler` (`router.rs:299-301`) return identical `(StatusCode::OK, "ok")`.
- `/_liveness` and `/_readiness` are registered on **both** routers (`router.rs:68-69` and `:227-228`) with no shared helper.
- Three NIP-11 limits are hard-coded twice with no shared const: `1024` at `nip11.rs:104` and `handlers/req.rs:26`; `10` at `nip11.rs:105` and `protocol.rs:14`; `256` at `nip11.rs:107` and `protocol.rs:11`. D-17 is exactly what happens when one copy drifts.

##### D-34 (S3) Two distinct `AdmissionError` types
`crate::admission::AdmissionError` (`admission.rs:12`, `Exceeded`/`Unavailable`) and `crate::audio::room::AdmissionError` (`audio/room.rs:83`, `Full`/`Ended`/`VersionMismatch`). Both are referenced from sibling call sites (`connection.rs:657`, `audio/handler.rs:515`), so every use must be path-qualified to stay unambiguous.

##### D-35 (S3) 26 production `unwrap`/`expect` against the repo's own rule
AGENTS.md: "Do not introduce new `unwrap()` or `expect()` in production paths." Count outside `#[cfg(test)]` in this group: **26** — `metrics.rs` 17 (`:79,84,89,94,99,104,109,114,119,124,129,134,136,141,143,145,180`), `main.rs` 4 (`:90,401,1230,1253`), `state.rs` 3 (`:446,701,708`), `config.rs` 1 (`:507`), `protocol.rs` 1 (`:189`). Plus `panic!` (`main.rs:409`), `unreachable!` (`main.rs:460`), `debug_assert!` (`nip11.rs:143`), `process::exit` (`main.rs:1153`).

Most are boot-time over compile-time literals and defensible; two are request/task-path (D-06). `main.rs:409`'s `panic!` is also stylistically inconsistent with the two adjacent preconditions that `return Err(anyhow!(…))` (`main.rs:206-211`, `:216-219`).

##### D-36 (S3) `metrics.rs` has zero tests
207 lines, 17 `expect`s, the `install` function that panics the process if a recorder already exists or the port is taken, and `track_metrics` with a hand-rolled `caller`-header validator (`metrics.rs:187-196`) — no test coverage at all. Every other file in the group has 4–22 tests (92 total). The header filter (length ≤64, `[A-Za-z0-9_-]`) is the one piece of untrusted-input handling in the file and is untested.

##### D-37 (S3) `admission.rs` is the only module with no doc comment
`lib.rs:5` `mod admission;` is the crate's only private module and `admission.rs:1` starts with a `use`, not `//!`. Every other file in the group opens with module docs (`state.rs:1`, `config.rs:1`, `router.rs:1`, `nip11.rs:1`, `protocol.rs:1`, `tenant.rs:1-15`, `telemetry.rs:1-32`, `metrics.rs:1-19`, `error.rs:1`). Relatedly, `lib.rs:43` `pub mod storage_sweep;` is the only `pub mod` in `lib.rs` without a `///` line.

##### D-38 (S3) Cache instrumentation covers 2 of 7 caches
Hit/miss counters exist for `membership_cache` (`state.rs:833/836`) and `accessible_channels_cache` (`state.rs:1095/1098`). `local_event_ids`, `channel_visibility_cache`, `observer_owner_cache`, `author_type_cache`, and `invite_claim_rate_limiter` have none — so their capacity (10,000 / 10,000 / 1,000 / 10,000) cannot be tuned from telemetry. `channel_visibility_cached` (`state.rs:1124`) is on the fan-out access-gate path.

##### D-39 (S3) Only 2 of 23 background tasks are cancellable
Cancellable: the audit worker (`AuditShutdownHandle`, `state.rs:1177-1196`) and the community revalidator (`community_revalidator_cancel`, `state.rs:510` → `main.rs:1045`). The other 21 (`main.rs:354,360,366,522,551,599,613,687,688,709,823,853,904,950,1008,1122,1132,1181`; `state.rs:968,1043`; `metrics.rs:146`) are abandoned at process exit with in-flight work. The reaper (#11) and reminder scheduler (#14) both perform DB writes.

##### D-40 (S3) `EmissionScope::allows` ignores its argument
`main.rs:76-78`: `fn allows(&self, _community_id: &Uuid) -> bool { matches!(self, Self::All) }`. The parameter exists solely for the planned `top:<k>` mode documented at `main.rs:44-47`. Honest and documented, but the signature promises per-community discrimination that does not exist, and three call sites pass a UUID that is discarded (`main.rs:1335`, `:1467/1509`, `main.rs:1475`).

##### D-41 (S3) `parse_optional_bool` is a single-caller one-liner
`config.rs:379-381` wraps `parse_bool(name, false)` and has exactly one caller (`config.rs:792`). Two names for one behaviour in a file that already has three overlapping parse helpers.

##### D-42 (S3) `bound_communities` is `pub` with no external caller
`state.rs:111` is `pub` but is only used by `revalidate_registered_communities` in the same file (`state.rs:174`) and by tests. Given D-14 (nothing consumes the crate as a library), `pub(crate)` or private is correct.

##### D-43 (S3) `AppState` derives `Clone` that is never used as a clone
`state.rs:487`. Every consumer takes `&AppState` or `Arc<AppState>`. The derive forces `git_store` (`state.rs:565`, the only non-`Arc` non-pool field) to stay `Clone` and invites accidental deep copies of a 40-field struct.

##### D-44 (S3) Test-only Postgres/Redis coupling in unit tests
`state.rs:1257-1290` (`test_state`) and the five tests using it, plus `config.rs`'s 22 env-mutating tests, need live infrastructure or a specific env. `test_state` points Redis at `redis://127.0.0.1:1` deliberately (`state.rs:1259`) but connects Postgres lazily to the real `database_url` (`state.rs:1260`). `just test-unit` will therefore behave differently depending on the developer's environment. There is no `crates/buzz-relay/tests/` directory to hold these.

##### D-45 (S3) Dev-dependencies fetch two crates from a third-party GitHub repo
`Cargo.toml:84-85` pins `mesh-llm-sdk` and `mesh-llm-host-runtime` to `git = "https://github.com/Mesh-LLM/mesh-llm.git", tag = "v0.73.1"`. `cargo test -p buzz-relay` therefore requires network access to an external org, and the supply-chain surface of the test build is outside the workspace lockfile's registry provenance.

##### D-46 (S3) Unreachable branch in `track_metrics`
`metrics.rs:170` skips `p == "/metrics"`. `/metrics` is served by the Prometheus exporter's own listener (`metrics.rs:73-74`), never registered on the app router, so it can never be a `MatchedPath`. Harmless, but it implies a routing arrangement that does not exist.

---

#### Positives worth preserving

Recorded so a future refactor does not remove them:

- `#![deny(unsafe_code)]` (`lib.rs:1`), **0 unsafe blocks**, **0 TODO/FIXME/HACK/XXX markers** across all 12 files, **0 `#[ignore]`d tests**, 92 tests.
- The compile-time conformance fence at `nip11.rs:329-335` — a documented invariant turned into a build break. The pattern deserves reuse (e.g. for the tenancy seam).
- The row-zero tenancy seam (`tenant.rs`) is trait-based, database-free at the call site, fail-closed on both `None` and `Err`, and has a red-team test module with a negative control (`tenant.rs:249-332`).
- The drain race is closed by an explicit insert-then-check / store-then-iterate pair with a sticky one-way flag and a test that names the interleaving (`state.rs:227-235`, `:353-356`, `:1875-1930`).
- `channel_visibility_cached` caches only the restrictive value so the worst stale entry is over-restrictive, never a leak (`state.rs:1106-1122`).
- Cache-invalidation-closure failure over-invalidates rather than serving stale access state (`state.rs:886-894`, `:932-974`).
- "Collect all DB results before emitting any gauge" prevents mixed fresh/stale metric snapshots (`main.rs:1481-1516`).
- Per-community gauge zero-fill from the host map, with a final `0.0` for disappeared label keys that preserves the host label so renames can be zeroed (`main.rs:1316-1382`).
- Usage-poll jitter uses true per-process randomness with a written rationale about PID 1 in containers (`main.rs:1009-1017`).
- Background sweeps build per-row `TenantContext` from the DB `RETURNING` rather than a default tenant (`main.rs:637-644`, `:741-746`).
- Claim-before-publish with a per-attempt stamp and compare-and-clear release in the reminder scheduler (`main.rs:748-798`) — correct except for D-05's terminal branch.
- Six startup steps become fatal precisely when membership enforcement is on, so a membership relay cannot boot into an unadministrable state (`main.rs:201,214,242,250,276,296`).


## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Debt

---

#### 0. Hard counts

| Metric | Value |
|---|---|
| Files / LOC in scope | 8 / 7,590 |
| Files over 1000 lines | 3 — `handlers/event.rs` 2461, `handlers/req.rs` 1946, `subscription.rs` 1562 |
| Inbound wire message types handled | **5** (`protocol.rs:23-42`) |
| Outbound formatters | **7** (`protocol.rs:181-208`) |
| Tests | **106** — `connection.rs` 5, `subscription.rs` 29, `event.rs` 24, `req.rs` 45, `auth.rs` 3, `count.rs` **0**, `close.rs` **0**, `mod.rs` **0** |
| `#[ignore]`d tests | **0** |
| `unsafe` | **0** |
| `unwrap()` / `expect()` outside `#[cfg(test)]` | **1** — `event.rs:88` |
| `TODO` / `FIXME` / `XXX` / `HACK` | **0** |
| Distinct subscription-related limits | 13 (see the data-model aspect); **11 of 13 are hard-coded** |
| Items with zero production callers | **3** (§3) |
| Duplicated helper implementations | 4 sets (§4) |

---

#### 1. CRITICAL

##### D-01 — The WS p-gate does not run for channel-scoped REQ
`req.rs:182` gates the p-gate, engram gate, and author-only gate behind `if channel_id.is_none()`. The in-code justification (`req.rs:179-181`) covers only *live fan-out*, but **historical delivery runs unconditionally** at `req.rs:261-406`.

Verified chain:
- `KIND_GIFT_WRAP` (1059) is p-gated (`buzz-core/src/kind.rs:150`) but is **not** in `is_global_only_kind` (`ingest.rs:379-436`), so a gift wrap with an `h` tag stores with `channel_id = Some(ch)`.
- `reader_authorized_for_event` gates only 30622 / 44200 (`buzz-core/src/filter.rs:25-27`) — no backstop for 1059, 24200, or member notifications.
- `is_author_only_event` covers only reminders + push leases (`buzz-core/src/kind.rs:120`) — no backstop.

So `["REQ","s",{"#h":["<a channel I'm in>"],"kinds":[1059]}]` reaches historical delivery with no `#p` predicate, returning every member's channel-scoped gift wraps. The HTTP bridge applies the gate unconditionally (`api/bridge.rs:981`), so WS is the weaker surface.

**Fix:** move `req.rs:182-205` out of the `channel_id.is_none()` branch. `count.rs:53-76` already does exactly this and is the reference implementation. Note `ab3af828` added a fourth read class (kind 30175) as a purely *result-level* gate applied on both transports (`req.rs:388`, `count.rs:202`/`:271`, `api/bridge.rs:1295`/`:1503`/`:1569`) — it therefore side-steps this asymmetry entirely, but it also did not repair it.
*Not verified:* whether shipped clients attach `h` tags to gift wraps — that determines exploitability today, not whether the path is open.

---

#### 2. HIGH

##### D-02 — Result-gated kind list is hardcoded in three more places
`RESULT_GATED_KINDS` (`buzz-core/src/kind.rs:129`) is honoured at `req.rs:1165-1170` and open-coded at:
- `buzz-core/src/filter.rs:25-27` — the actual result-level read gate
- `event.rs:461-462` — the live fan-out owner fence
- `req.rs:1058-1063` — the ids-exemption carve-out

`ab3af828` shows the shape this wants: the new kind-30175 class puts its kind test in one place (`crates/buzz-core/src/kind.rs:192-194`) and every read surface calls one combined predicate (`req.rs:1222-1234`). The result-gated copies were left untouched.

The relay therefore **compounds** the known `filter.rs:25` drift. Adding a third result-gated kind updates only COUNT's pushdown decision and leaves the enforcement points open — a **fail-open** divergence. Fix: make all four sites read `RESULT_GATED_KINDS`, and add a test asserting each site's coverage equals the constant.

##### D-03 — DM-visibility owner fence is missing on the cross-node fan-out path
`event.rs:457-491` implements the owner fence for kinds 30622 / 44200, but only inside `dispatch_persistent_event_inner` (`event.rs:396`). `fan_out_pubsub_event` (`event.rs:282`) and `fan_out_event_to_local_subscribers` (`event.rs:241`) apply only the `filter_fanout_by_access` fences, and neither kind is in `AUTHOR_ONLY_KINDS` so `F2` does not substitute. On a multi-pod deployment a kindless `ids:[…]` subscription on pod B receives another user's 30622/44200 event — exactly the case the fence's own comment (`event.rs:480-483`) says it exists to close.

Aggravating: the doc comment at `event.rs:235-240` asserts both exception paths are "equivalent to this helper plus their own extra step". For `fan_out_pubsub_event` that is false — it has one fewer step. Fix: move the owner fence into `filter_fanout_by_access` (where `AUTHOR_ONLY_KINDS` already lives) so all three paths inherit it.

**`ab3af828` proved the fix.** The kind-30175 delivery gate it added was placed inside `filter_fanout_by_access` (`event.rs:154-175`) rather than in `dispatch_persistent_event_inner`, so it is inherited by the Redis cross-node path (`event.rs:307`) and the local path (`:241`) alike. The same one-block move applied to the 30622/44200 fence closes D-03.

##### D-04 — No per-IP connection limit; the machinery exists and is unwired
`LimitType::IpConnections` (`buzz-auth/src/rate_limit.rs:66`) and `RateLimiter::check_ip_connection` (trait `:188`, Redis impl `buzz-pubsub/src/rate_limiter.rs:112`) have **zero** callers in the relay. Combined with:
- `max_connections` default 10 000 (`config.rs:449-452`),
- admission bypassed pre-auth (`connection.rs:604-609`),
- `remote_addr` captured but used only for logging (`connection.rs:183`, `:289`),

one host can hold every connection slot with unauthenticated sockets, each costing a 1000-slot mpsc plus three tasks, recycled every 5 s (`connection.rs:27`).

Note a prerequisite: `router.rs:238-243` reads `ConnectInfo` with a `0.0.0.0:0` fallback and does **no** `X-Forwarded-For` handling, so behind a proxy every connection shares one apparent address. A per-IP cap needs trusted proxy-header config first.

##### D-05 — No bound on filter internal cardinality; index amplification
`MAX_SUBSCRIPTIONS` = 1024 (`req.rs:26`) and `MAX_FILTERS_PER_REQ` = 10 (`protocol.rs:14`) bound the *counts*, but nothing bounds a single filter's `ids`, `authors`, `kinds`, or generic-tag arrays.

Two amplification vectors, both reachable by one ordinary authenticated member inside every advertised limit:
1. **Match cost** — `filter_match_one` (`buzz-core/src/filter.rs:41-46`) linear-scans `authors` / `ids` on **every** candidate event, for every one of up to 10 240 registered filters.
2. **Index cost** — `register_scoped` inserts one index entry **per kind in the union** (`subscription.rs:104-112`), each carrying a cloned `sub_id` of up to 256 B (`protocol.rs:11`). 10 filters × 1000 kinds × 1024 subs ≈ 10 M entries ≈ 2.6 GB of `String` on one connection. Removal is `Vec::retain` per bucket (`subscription.rs:407`, `:429`), so disconnect cleanup is O(that).

No test covers filter cardinality. Fix: cap per-filter array lengths at parse time in `protocol.rs`, and cap the kind-union size in `register_scoped`.

##### D-06 — `ARCHITECTURE.md` §9 #2 is factually wrong about rate limiting
`ARCHITECTURE.md:823` states "**No rate limiting implementation** … `RateLimitConfig` defines 4 tiers … but none are enforced." Three limits **are** enforced on this path: `human_ws_events_per_sec`, `human_messages_per_min`, `agent_standard_messages_per_min` (`connection.rs:612-650`) against a Redis-backed limiter (`state.rs:584`, ctor `:712`), failing closed (`admission.rs:34-40`).

This is the highest-impact doc defect in the group: an operator reading it would conclude the relay has no throttling and add an external one, or would not investigate a `rate-limited:` NOTICE. Accurate replacement: *"IP-connection limiting is not wired (`check_ip_connection` has no caller); the elevated and platform agent tiers are unreferenced."*

---

#### 3. MEDIUM

##### D-07 — Dead code: three `SubscriptionRegistry` methods with zero production callers
| Item | Site | Only referenced by |
|---|---|---|
| `get_filters` | `subscription.rs:338-343` | `subscription.rs:739`, `:743` (tests) |
| `total_subscriptions` | `subscription.rs:345-348` | `subscription.rs:651`, `:655` (tests) |
| `total_connections` | `subscription.rs:350-353` | `subscription.rs:656` (test) |

Verified by workspace-wide grep. `main.rs:1328-1331` computes its own totals from `per_community_subscriptions()` and `per_community_ws_connections()` — the local variables there are coincidentally named `total_subscriptions` / `total_connections`, which is why a shallow grep looks like a hit. All three are `pub` on a `pub` type, so the compiler does not flag them.

##### D-08 — COUNT skips the read-scope check
`handle_req` refuses a session lacking `MessagesRead` (`req.rs:54-61`); `handle_count` has no equivalent (`count.rs:37-51`). A scoped session with only write scopes can use COUNT as an existence oracle over every channel it belongs to.

##### D-09 — COUNT leaks raw database error text to clients
`count.rs:179`, `:209`, `:249`, `:278` all emit `CLOSED "error: {e}"` with the `buzz_db` error rendered verbatim. `handle_event` sanitises the identical case with an explicit comment (`event.rs:749-750`). Three different postures exist for a DB failure across the group: sanitised (`event.rs`), verbatim (`count.rs`), and silently swallowed into an `EOSE` (`req.rs:323-329`).

##### D-10 — A failed historical query is indistinguishable from an empty result
`req.rs:318-326` sends `EOSE` on a `query_events` error and returns, leaving the subscription registered. The client cannot tell "no matches" from "DB down", and there is no metric for the case. `handle_search_req` behaves the same way for FTS and batch-fetch errors (`req.rs:611-616`, `:646-650`) — both `break` out of pagination and fall through to `EOSE`. Fix: emit `CLOSED "error: …"` (or at minimum a counter) and consider deregistering.

##### D-11 — Delivery abort skips EOSE
`req.rs:398-400` and `:720-722` `return` on a failed `conn.send`, so the client never receives `EOSE` for a subscription that is registered and live. Combined with the 15-strike grace (`connection.rs:100`) the connection survives, so the client waits indefinitely on a promise the relay silently abandoned.

##### D-12 — Search REQ does not replace a same-`sub_id` live subscription (NIP-01 violation)
The search branch returns at `req.rs:232`, **before** `subs.insert` (`:238`) and `register_scoped` (`:241`). NIP-01 requires a REQ with an existing `sub_id` to overwrite it. Here the previous subscription keeps fanning out under an id the client believes was replaced by a one-shot query. Same-shaped consequence: a search REQ never counts against `MAX_SUBSCRIPTIONS` (`req.rs:66` reads `conn.subscriptions`, which search never writes).

##### D-13 — Rate-limited EVENT/COUNT get no protocol-level acknowledgement
`enforce_ws_admission` derives `sub_id` only from `ClientMessage::Req` (`connection.rs:624-627`), and the EVENT `Messages` check passes `None` explicitly (`:647`). So a throttled EVENT gets a `NOTICE` with **no `OK`** — a NIP-01 client keyed on `OK` hangs on that event id. Same for COUNT: no `COUNT` response, only a `NOTICE`. Fix: thread the event id into the admission result so an EVENT rejection is `OK false "rate-limited: …"`.

##### D-14 — `observer_rate_limiter` is unbounded and never evicted
`state.rs:589` is a plain `DashMap`; `event.rs:920-924` only ever `entry().or_insert()`. One permanent entry per distinct `(community, agent pubkey)` for the process lifetime. Every sibling structure on `AppState` is a moka cache with an explicit cap and TTL (`state.rs:734-787`). Fix: convert to `moka` with a TTL matching the 1 s window.

##### D-15 — Conformance side-queries run in production
`req.rs:355` and `:648` issue `db.communities_of_channels(...)` whenever `trace_state.is_some()` — i.e. on every REQ filter and every search page — even though production binds `NoopTracer` (`state.rs:798`) and discards the result. That is one extra Postgres round-trip per filter and per search page on the hot read path, purchasing nothing outside conformance runs. Fix: gate on a tracer-enabled predicate.

##### D-16 — Cross-node fan-out loop is not restartable
`main.rs:837-840`: a `RecvError::Closed` logs an error and `break`s out of the loop, permanently ending cross-node delivery for the process while readiness continues to report ready (`router.rs:353-390` checks only Postgres + Redis reachability, not the broadcast channel). Lag is counted (`main.rs:833-836`) but never repaired — lagged events are silently lost.

##### D-17 — Three identical `topic_for_subscription` copies
Byte-identical private functions at `req.rs:1270-1275`, `close.rs:30-35`, `connection.rs:681-686`. All three must move in lockstep with the `EventTopic` enum. The `retain`/`release` refcount correctness depends on all three agreeing.

##### D-18 — Duplicate `ConnectionManager` pubkey accessors
`pubkey_for_conn` (`state.rs:286-290`) and `pubkey_for` (`state.rs:425-429`) are line-for-line identical. `pubkey_for_conn` is used by `event.rs:146`, `:184` and `side_effects.rs:108`; `pubkey_for` only by `event.rs:483` — inside `dispatch_persistent_event_inner`, i.e. the *same file* uses both names for the same operation.

##### D-19 — Filters are stored twice per subscription
`conn.subscriptions` (`connection.rs:65`, written `req.rs:236-239`) duplicates the filter vectors already in `sub_registry.subs` (`subscription.rs:79-82`). The duplicate's only read is `len()`/`contains_key` for the cap check (`req.rs:66`); `subscriptions_for` (`state.rs:383`) hands the `Arc` to `side_effects.rs:71`, which also does not read the filters. Two `Vec<Filter>` clones per REQ (`req.rs:238`, `:245`) plus one inside `register_scoped` (`subscription.rs:81`) for a value never used as filters. Fix: store a count, or read the cap from the registry.

##### D-20 — `count.rs` and `close.rs` have zero tests
`handlers/count.rs` is 281 LOC of authorization-and-pushdown logic — the p-gate, the token narrowing, the pushdown safety predicate, the 5000-row budget, and four error paths — with **no** in-file test. Its helpers are tested in `req.rs`, but the composition (which is where D-08 and D-09 live) is not. `handlers/close.rs` (35 LOC) has none either, including the release-topic ordering it exists to guarantee.

Related: **no test drives `handle_req`, `handle_count`, or `handle_close` end-to-end.** The only message handler with an end-to-end unit test is `handle_agent_observer_event` (`event.rs:1318-1404`). Everything else in this group is tested at the helper level or in `crates/buzz-test-client`.

---

#### 4. LOW

##### D-21 — `ARCHITECTURE.md` numeric and structural drift
| Line | Claim | Actual |
|---|---|---|
| `:161` | frame size 65 536 B | 524 288 (`config.rs:14`) |
| `:161` | max historical results/filter 500 | 2 000 (`req.rs:25`) |
| `:208` | `SLOW_CLIENT_GRACE_LIMIT (3)` | config field, default 15 (`config.rs:470-473`) |
| `:150-159` | 4 inbound / 6 outbound messages | 5 / 7 — COUNT missing both ways |
| `:161` | no mention of 10-filter or 256-B sub_id caps | `protocol.rs:11`, `:14` (both NIP-11-advertised) |
| `:181-187` | auth described with no deadline | 5 s (`connection.rs:27`) |
| `:211-217` | cleanup = 5 steps | 7 — adds per-subscription `release_topic` (`connection.rs:265-270`) and last-connection presence clear (`:274-285`) |
| `:235` | step 10 "SEARCH INDEX — `search_index_tx.send` (bounded worker queue)" | removed; the code documents the removal at `event.rs:502-508` |
| `:225-237` | pipeline omits the 24200 observer branch and the pubkey-match gate ordering | `event.rs:636-669` |

Downstream: `.aidlc/reverse-engineer/configuration.md:89` and `:91` inherit the frame-size and grace-limit errors.

##### D-22 — Stale in-code comments naming line numbers and struct shapes
- `event.rs:2368` — "`filter_fanout_by_access` (this file, line 62)"; it is at `:115`.
- `event.rs:2361` — cites `state.rs:30-44` for `ConnEntry`; it is `:41-58`.
- `event.rs:2361-2366` — asserts `ConnEntry` "records `authenticated_pubkey` but NO community/tenant binding" and that `filter_fanout_by_access` "never compares" the receiver's tenant. Both were true at the cited revision `fb0d6a4ea` and are **false now**: `ConnEntry.community_id` exists (`state.rs:51`) and the comparison is the first fence (`event.rs:126-133`). The header also references a companion "documents the current (broken) behavior" test that it says MUST be deleted with the fix — that test is **absent**, so the deletion happened but the header did not.
- `event.rs:868-874` — describes a `Uuid::nil()` "global channel" Redis routing sentinel and a `is_nil()` check in the `main.rs` subscriber loop. Neither exists: `EventTopic::Global` is an enum variant (`event.rs:880`) and `main.rs:818-845` has no nil handling.
- `buzz-auth/src/lib.rs:69` — `channel_ids` documented as "reserved for future per-channel access control"; it is load-bearing at `req.rs:87-88`, `:137-139`, `count.rs:95-97`, `:142-149`.

##### D-23 — Production `expect()` on the fan-out hot path
`event.rs:88` `.expect("fan-out frame cache covers every recipient subscription id")`. The invariant holds at all three call sites (the cache is built from the same iterator that drives the send), but this is the group's only non-test `expect()`, it sits in delivery, a panic poisons a spawned dispatch task, and `AGENTS.md` prohibits new `unwrap()`/`expect()` in production paths. Fix: `if let Some(frame) = …` and count the miss.

##### D-24 — Unvalidated numeric config can panic or self-disable
| Var | Loader | Failure mode |
|---|---|---|
| `BUZZ_SEND_BUFFER=0` | `config.rs:459-462`, no `>0` filter | `mpsc::channel(0)` **panics** at `connection.rs:159` on the first connection |
| `BUZZ_MAX_CONNECTIONS=0` | `config.rs:449-452` | `Semaphore::new(0)` → every connection silently rejected with no frame |
| `BUZZ_MAX_CONCURRENT_HANDLERS=0` | `config.rs:454-457` | every EVENT/REQ/COUNT rejected as rate-limited |
| `BUZZ_SLOW_CLIENT_GRACE_LIMIT=0` | `config.rs:470-473` | disconnect on the **first** full buffer (`connection.rs:100`) |

`BUZZ_MAX_FRAME_BYTES` has a `>0` filter (`config.rs:467`) and the `BUZZ_RATE_LIMIT_*` family hard-errors on zero (`config.rs:270-283`), so the pattern exists — it just was not applied here. The `defaults_are_valid` test (`config.rs:938`) asserts exactly the invariants the loader fails to enforce, and only for the default path.

##### D-25 — `AuthState::Failed` is permanently sticky, including for transient causes
`auth.rs:161-165` writes `Failed` for a ban-state **DB error**, which is terminal (`auth.rs:58-66`). The surrounding comment (`auth.rs:98-112`) goes to visible lengths to avoid mislabelling a blip as a ban, but still pins the socket. Fail-closed is right; the residual cost is a forced reconnect on any Postgres hiccup, with no retry and no metric distinguishing it from a real auth failure (both increment `buzz_auth_failures_total`, differing only by the `reason` label — `auth.rs:169-171`).

##### D-26 — Silent drops with no signal
| Case | Site |
|---|---|
| Binary frame with invalid UTF-8 | `connection.rs:457-459` — no `else`, no NOTICE, no counter |
| Observer frame with an unrecognised `frame` tag value | `event.rs:1099-1102` → `OK true` (deliberate, documented) |
| REQ with two different `#h` values silently downgraded to a global subscription | `req.rs:1019-1023` — and a global subscription receives **no** channel events (`subscription.rs:320-327`), so the client gets a permanently silent subscription |
| Presence-set / presence-clear Redis errors | `event.rs:814-822` — `let _ =` |
| `#h` values all inaccessible in a search filter → filter skipped | `req.rs:580-582` |
| COUNT filter on an inaccessible channel → contributes 0 | `count.rs:150-151` |

##### D-27 — Case-sensitivity inconsistency across the filter gates
The p-gate compares `#p` with `==` (`req.rs:1066-1069`) and `result_gated_count_safe_for_pushdown` likewise (`:1188-1191`), while the engram gate (`:1119-1122`) and author-only gate (`:1261-1264`) use `eq_ignore_ascii_case`. Uppercase-hex `#p` therefore fails the p-gate but would clear an equivalent engram check. Fails closed, so it is a usability wart — but four sibling predicates should not disagree.

##### D-28 — `remote_addr` stored but unused for anything but logging
`connection.rs:61`, read only at `:183` and `:289`. No abuse correlation, no per-IP accounting, no IP in any rejection metric. The field's existence implies a capability the relay does not have (see D-04).

##### D-29 — File sizes
`event.rs` at 2484 lines mixes six concerns: metric-label bounding, frame serialisation, the access-filter chokepoint, the Redis fan-in path, post-commit dispatch, and three message handlers. `req.rs` at 1991 mixes the REQ handler, the search handler, filter→SQL translation, and **eight** authorization predicates that `count.rs` and `api/bridge.rs` import (`ab3af828` added `filter_can_match_persona_shared_kinds` and `event_visible_to_reader`). There is no line-count guard for Rust (the `check-file-sizes.mjs` 1000-line ceiling in `AGENTS.md` applies to `mobile/`), so these will keep growing — both files grew again in the 2c sync.

##### DEBT-58 (NEW, introduced by `ab3af828`) — kind-30175 COUNT loses the exact SQL path even for the author's own personas
The fast `count_events()` pushdown is now suppressed for **any** filter that can match kind 30175 (`count.rs:110`, guards `:174` and `:243`; HTTP twin `api/bridge.rs:1447-1448`, guards `:1477`, `:1543`). Unlike its two sibling gates, this one has no escape hatch:

| Gate | Pushdown-safe escape |
|---|---|
| author-only kinds | `\|\| author_is_self` — `count.rs:172` |
| result-gated kinds | `result_gated_count_safe_for_pushdown(filter, self)` folded in at `count.rs:117-118` |
| **persona shared-gate** | **none** — `&& !needs_persona_filtering` is unconditional (`count.rs:174`, `:243`) |

So `COUNT {kinds:[30175], authors:[self]}` — where the reader is trivially authorized for every row — is forced onto the bounded 5001-row candidate fallback and, past 5000 *visible* rows, is answered with `CLOSED "restricted: count filter requires narrower constraints"` (`count.rs:189-195`, `:258-264`) instead of an exact count. The SQL pre-filter (`crates/buzz-db/src/event.rs:504-527`) does reduce the candidate page, so this only bites large persona catalogs, but the asymmetry is unnecessary: an `author_is_self`-shaped escape is sound here for exactly the same reason it is sound for author-only kinds.

Note this is *not* a regression for kindless filters — `filter_can_match_author_only_kinds` (`req.rs:1133-1138`) already returned `true` for them, so a kindless COUNT never had the fast path.

---

#### 5. Untested paths (production code with no test coverage in this group)

| Path | Site | Nearest coverage |
|---|---|---|
| `handle_req` end-to-end | `req.rs:44-417` | helper-level only |
| `handle_count` end-to-end | `count.rs:30-286` | **none** in-file |
| `handle_close` | `close.rs:12-30` | **none** |
| `handle_auth` (all four gates: ban cascade, allowlist, membership, NIP-OA materialisation) | `auth.rs:44-296` | only `extract_auth_tag_json` (3 tests, `auth.rs:297-349`) |
| `handle_event` dispatch gates (auth, pubkey match, kind 22242, scope) | `event.rs:611-696` | **none** |
| `handle_ephemeral_event` incl. presence parsing + 128-B truncation | `event.rs:762-897` | **none** |
| `handle_connection` / `handle_active_connection` lifecycle | `connection.rs:118-291` | send-loop only (4 tests) |
| `recv_loop` frame handling (oversize, binary, Ping/Pong) | `connection.rs:407-487` | **none** |
| `heartbeat_loop` 3-strike logic | `connection.rs:378-405` | **none** |
| `enforce_ws_admission` / `send_admission_result` | `connection.rs:594-679` | only `request_rejection_message` (`connection.rs:776-785`) and `admission.rs:46-157` |
| Auth-timeout task | `connection.rs:228-252` | **none** |
| `handle_search_req` | `req.rs:504-732` | only `build_search_channel_scope_filter` (2 tests) |
| `filter_to_query_params` beyond the `#d` matrix | `req.rs:856-987` | `#d` pushdown only (`req.rs:1594-1648`) |
| `filter_fully_pushable` | `req.rs:773-823` | **none** |
| Filter-cardinality limits | — | **no such limits exist** |
| Per-IP admission | — | **no such control exists** |

---

#### 6. Prioritised remediation order

1. **D-01** — move the sensitive-kind gates out of the `channel_id.is_none()` branch (`req.rs:182`). One-line structural change; `count.rs:53-76` is the model.
2. **D-03** — hoist the 30622/44200 owner fence into `filter_fanout_by_access` so all three fan-out paths inherit it.
3. **D-02** — collapse the four result-gated-kind lists onto `RESULT_GATED_KINDS`, with a coverage test.
4. **D-06 / D-21** — correct `ARCHITECTURE.md` §9 #2, the §2 limits line, §3 step 3/step 5, and §4 step 10.
5. **D-05** — cap per-filter array cardinality in `protocol.rs` and the kind-union size in `register_scoped`.
6. **D-08 / D-09** — add the `MessagesRead` check and sanitise DB errors in `count.rs`.
7. **D-13 / D-11 / D-10 / D-12** — protocol-correctness cluster: rate-limited `OK`, missing `EOSE`, `EOSE`-on-error, search-REQ sub_id replacement.
8. **D-20** — tests for `count.rs` and `close.rs`, then end-to-end tests for `handle_req` / `handle_count`.
9. **D-04** — wire `check_ip_connection` (after deciding proxy-header trust).
10. **D-14 / D-15 / D-16** — resource-hygiene cluster.
11. **D-07 / D-17 / D-18 / D-19 / D-22 / D-23 / D-24** — cleanup: remove dead methods, dedupe helpers and accessors, drop the duplicate filter store, fix stale comments, replace the production `expect()`, validate numeric config.


## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: Debt

---

### 0. Complexity quantified

| File | Lines | Prod lines | Test lines | Longest fn | Top-level `if`/`match` (prod) | `.await` (prod) | Tests |
|---|---|---|---|---|---|---|---|
| `ingest.rs` | 3 686 | 2 505 | 1 181 | **`ingest_event_inner` — 1 079 lines** (`:1427-2505`) | 169 `if` / 41 `match` | 46 | 95 |
| `side_effects.rs` | 3 347 | 3 265 | 82 | **`validate_admin_event` — 418 lines** (`:259-676`) | 156 `if` / 34 `match` | 164 | 5 |
| `command_executor.rs` | 1 327 | 1 327 | **0** | `handle_workflow_def` — 155 lines (`:653-807`) | 52 `if` / 17 `match` | 63 | **0** |
| `imeta.rs` | 551 | 418 | 133 | `validate_imeta_tags` — 199 lines (`:10-208`) | 31 `if` / 14 `match` | 5 | 11 |
| **Total** | **8 911** | 7 515 | 1 396 | — | 408 `if` / 106 `match` | 278 | **111** |

`ingest_event_inner` alone: **1 079 lines, 46 `return Err`, 10 `return Ok`, 27 `?`
operators, 47 top-level branch statements, 33 `.await` points** — i.e. **56 distinct exit
points** in one function, each with its own error string.

`#[ignore]`d tests: **0**. `#[tokio::test]`: **0**. `unsafe`: **0**.
`TODO`/`FIXME`/`XXX`/`HACK` markers: **0**.
`unwrap()` outside `#[cfg(test)]`: **2**. `expect()` outside `#[cfg(test)]`: **4**.

The repo enforces a 1 000-line/file ceiling for JS (`desktop/scripts/`) and Dart
(`mobile/scripts/check-file-sizes.mjs`, per AGENTS.md) with the explicit instruction "split
the file — never bump the limit". There is **no equivalent Rust guard**. `ingest.rs` is
3.7× that ceiling, `side_effects.rs` 3.3×, and a single function in `ingest.rs` is itself
over it.

---

### 1. Critical

#### D-01 — `ingest_event_inner` is a 1 079-line linear gauntlet with 56 exits
`ingest.rs:1453-2531`

Every kind's special case is inlined into one function. Consequences that are already
visible in the code:

- **Ordering is invisible and load-bearing.** Six separate comments exist purely to explain
  why a block sits where it does (`:1532-1533`, `:1572-1578`, `:1589-1601`, `:2294-2297`,
  `:2404-2405`, `:2443-2447`). Nothing enforces any of it.
- **Long-range invariants.** The four `expect()` calls (`:1998`, `:2000`, `:2338`, `:2344`)
  each depend on a check 340–570 lines earlier in the same function.
- **No per-stage testability.** All 95 tests in the file target extracted helpers; the
  pipeline itself has zero in-file coverage because it needs `AppState` (and there are no
  `#[tokio::test]`s anywhere in the group).
- **Kind dispatch is scattered across 47 sequential `if kind_u32 == …` blocks** rather than
  one match, so adding a kind means finding the right insertion point among them.

Fix shape: extract phases into named functions taking a small context struct
(`EnvelopeGates`, `RestrictionGate`, `ChannelResolution`, `PerKindValidation`,
`Persist`, `PostCommit`), and replace the 47 sequential kind comparisons with a single
dispatch table keyed by kind. This is mechanical — every block already has a clean
boundary.

#### D-02 — Side-effect failure is silent and reported as success
`ingest.rs:2460-2467`

```rust
if crate::handlers::side_effects::is_side_effect_kind(kind_u32) {
    if let Err(e) = handle_side_effects(tenant, kind_u32, &event, state).await {
        warn!(event_id = %event_id_hex, kind = kind_u32, "Side effect failed: {e}");
    }
}
```

The event is already committed; `dispatch_persistent_event` runs unconditionally 48 lines
later (`:2513`). So a rejected `add_member`, a failed `update_channel`, or a failed
`remove_member` all produce `accepted: true` with an empty message, the command event is
fanned out, and every client renders a state change that never happened. There is **no
metric** for this (32 `warn!` sites in `side_effects.rs`, zero counters), so it is invisible
in monitoring too.

Worst case within this: `handle_edit_metadata` (`side_effects.rs:1335-1558`) iterates tags
with `?`, so tag *n* failing leaves tags 1..n−1 applied — a genuinely partial channel
update reported as success.

Fix shape: either (a) split side effects into "must succeed" (membership, metadata) vs
"best effort" (notifications, discovery) and roll back / reject on the former, or (b) at
minimum add `buzz_side_effect_failures_total{kind}` and return a non-empty `message` so
clients can distinguish.

#### D-03 — Privileged mutations bypass the audit chain
`ingest.rs:1834-1844` (9030–9033), `:1605-1614` (9040–9044), `command_executor.rs:36-78`
(7 command kinds)

`buzz-audit` is a hash-chained tamper-evident log with 11 declared actions
(`buzz-audit/src/action.rs:8-31`). Production has **2** producers: `EventCreated`
(`handlers/event.rs:560`) and `MediaUploaded` (`api/media.rs`). Verified by grepping
`AuditAction::*` outside `crates/buzz-audit/` and `tests/`:

| Action | Production producers |
|---|---|
| `EventCreated` | 1 |
| `MediaUploaded` | 1 |
| `EventDeleted`, `ChannelCreated`, `ChannelUpdated`, `ChannelDeleted`, `MemberAdded`, `MemberRemoved`, `AuthSuccess`, `AuthFailure`, `RateLimitExceeded` | **0 each** |

And because relay-admin and moderation commands are never stored, they never reach
`dispatch_persistent_event`, so they get **no audit row at all** — not even the generic
`EventCreated`. Relay membership/role changes are the highest-privilege operation in the
system. Command kinds are stored (`command_executor.rs:196`) but `handle_command` never
calls `dispatch_persistent_event`, so DM creation, workflow definition, and approval
decisions are also unaudited.

Contrast with NIP-IA, which deliberately falls through to storage precisely so the audit
reference resolves (`ingest.rs:1914-1919`) — the pattern exists and was applied to one
feature only.

#### D-04 — `command_executor.rs` has zero tests
`command_executor.rs` (1 327 lines, no `#[cfg(test)]` module)

It is simultaneously: the only raw-SQL module in the group (`:157-232`), the only
hand-rolled NIP-33 LWW implementation outside `buzz-db` (`:135-224`), the only module
holding an open `sqlx::Transaction` across `await` points (`PersistResult::Inserted`,
`:83`), and the owner of 7 authorization decisions. `check_approver_spec` (`:961-984`) is a
pure, 24-line, fail-closed authorization function with no test.

The hand-rolled LWW is worth restating: an FNV-1a hash folded into an `i64`
`pg_advisory_xact_lock` key (`:158-169`), a coordinate SELECT, a domination comparison
(`created_at < existing || (created_at == existing && incoming_id >= existing_id)`,
`:186-188`), and a manual supersede. Every one of those four steps is a correctness-critical
reimplementation of logic that `buzz-db` already owns, and none is tested.

---

### 2. High

#### D-05 — The scope system is inert; the code reads as if it enforces
`ingest.rs:198-306`, `:1551-1556`

109 lines mapping 81 kinds to 7 scopes, plus a gate that can never fail because both
transports pass `Scope::all_known()` (`buzz-auth/src/lib.rs:137`, `api/bridge.rs:829`).
`AdminUsers` on 9030–9033 and `AdminChannels` on 9000/9001/9008 look like access control
and are not. Nine tests assert scope values (`ingest.rs:2850`, `:2887`, `:2923`, …),
reinforcing the impression.

Either wire real scope derivation, or delete the mapping and rename the function to what it
is: `is_kind_accepted`. The current state is the worst of both — maintenance cost of a
control with none of its benefit.

#### D-06 — Channel-scoped-token logic is entirely dead
`ingest.rs:117-126`, `:525-532`, `:1509-1524`, `:1719-1728`

`AuthContext.channel_ids` is hard-coded `None` (`buzz-auth/src/lib.rs:138`) and
`IngestAuth::Http` has no such field. Four gates, one accessor, and one helper are
unreachable. The doc comment already says so ("In pure Nostr mode this always returns
None", `ingest.rs:113-115`) — the code was left in place rather than removed.

`IngestAuth::conn_id()` (`ingest.rs:107-112`) has **zero callers** anywhere in the
workspace. `HttpAuthMethod::DevPubkey` (`ingest.rs:57`) is never constructed — only
`Nip98` (`api/bridge.rs:828`).

#### D-07 — `side_effects.rs` `validate_admin_event` is a 418-line authorization function
`side_effects.rs:259-676`

20 `return Err` + 35 `anyhow!` constructions + 17 `.await` points in one function covering
7 kinds. It contains the module's only two production `unwrap()`s (`:311`, `:314`), both in
the 9000 private-channel branch. Zero tests — the 5 tests in the file cover pure helpers
(`:3266-3345`).

Three different answers to "may a non-member NIP-OA owner act?" live inside it: 9001 says no
(`:365-367`), 9002 and 9008 say yes (`:598-608`, `:632-640`), and 9005 says yes for the
agent-author case (`:611-621`). Each is individually justified in a comment; none is stated
as a rule.

#### D-08 — Dead branches in the side-effect predicate and dispatcher
`side_effects.rs:35-37`, `:143-176`

`is_side_effect_kind` matches `0 | 5 | 9000..=9022 | 30617 | 10100 | 41001..=41003 | 40099`.
Cross-referenced against the 81-kind allowlist:

| Claimed range | Reachable kinds | Dead |
|---|---|---|
| `9000..=9022` | 9000, 9001, 9002, 9005, 9007, 9008, 9021, 9022 | 9003, 9004, 9006, **9009**, 9010–9020 |
| `41001..=41003` | **none** | 41001 (`DM_CREATED`, not in the allowlist), 41002, 41003 (undefined) |
| `40099` | **none** | relay-signed only |

And `handle_side_effects` has a live-looking arm for **9009** (`:157-163`) that logs
`"NIP-29 kind 9009 handler deferred to future phase"` — provably unreachable, since 9009 is
not in `required_scope_for_kind`.

#### D-09 — Three dead thread-counter mutation entry points
`buzz-db/src/thread.rs:251-287`, `buzz-db/src/lib.rs:1973`, `:2088-2095`

- `thread::increment_reply_count` — **zero callers workspace-wide**.
- `Db::decrement_reply_count` — **zero callers**.
- `Db::insert_thread_metadata` → `thread::insert_thread_metadata` (`thread.rs:116-238`) —
  only reached from `#[cfg(test)]` code (`thread.rs:1315`, inside the test module starting
  at `:810`).

These are the non-transactional variants of the counter mutations. The transactional
versions exist specifically because "a crash between them cannot leave reply_count /
descendant_count inconsistent" (`thread.rs:111-114`). Leaving the unsafe variants publicly
callable invites a future caller to reintroduce exactly the inconsistency they were
replaced to prevent.

(The positive finding: every *live* reply-insert path does update the counters — see
data-model.md §7.)

---

### 3. Medium

#### D-10 — `thread_meta` is silently dropped on the replaceable branches
`ingest.rs:2246-2257` computes `thread_meta` for any `requires_h_channel_scope` kind, but
`replace_addressable_event` (`:2371`) and `replace_parameterized_event` (`:2385`) have **no
thread-metadata parameter**. Today no replaceable kind is in `requires_h_channel_scope`, so
this is latent. Adding one would lose thread ancestry with no compile error, no warning, and
no failing test — the only exhaustive predicate test asserts `is_global_only_kind` vs
`requires_h_channel_scope` disjointness (`:2753-2762`), not disjointness from the
replaceable ranges.

#### D-11 — Error-variant / prefix mapping is inconsistent
- `forbidden: ` exists only in `command_executor.rs` (6 sites) and always as `Rejected`, so
  command authorization failures return HTTP **400** while every other authorization failure
  returns **403**.
- `restricted: ` maps to `Rejected` at `ingest.rs:1482`, `:1507`, `:521` and to `AuthFailed`
  at `:1513`, `:1521`, `:1526`, `:1726`, `:2012`.
- `check_channel_membership`'s DB error becomes `Rejected("error: database error: …")`
  (`ingest.rs:501`, mapped `:1802`) → HTTP 400 for a server fault, contradicting the
  fail-closed-as-`Internal` convention used 200 lines earlier at `:1637`.
- Double prefixing: `validate_edit_ownership` returns `restricted: not a channel member`
  (`ingest.rs:838`) and the call site prepends `invalid: ` (`:1963`), producing
  `invalid: restricted: not a channel member`.

#### D-12 — `e`-tag selection direction is undocumented and load-bearing
Reactions take the **last** `e` tag (`ingest.rs:334`, `:2251`; `side_effects.rs:2192`);
edits (`ingest.rs:766`), votes (`:847`), kind:5 channel derivation (`:1670`), and 9005
(`side_effects.rs:531`) take the **first**. No comment names the rule. A future refactor
that unifies the extractors would silently change NIP-25 target resolution.

#### D-13 — Four duplicated helpers
| Helper | Copies |
|---|---|
| `effective_message_author` | `ingest.rs:729-761` (`pub(crate)`) and `side_effects.rs:2271-2298` (private) — identical semantics; `side_effects.rs:2195` even calls the `ingest.rs` copy, so both are live in one call graph |
| channel-from-`h`-tag | `extract_channel_id` `ingest.rs:308-319`, `extract_h_tag_channel` `side_effects.rs:2237-2249` (byte-equivalent), `extract_h_tag` `command_executor.rs:250-259` (returns `String`) |
| `extract_p_tag` / `extract_p_tags` | `side_effects.rs:2251-2269` vs `command_executor.rs:235-248` |
| tombstone content builder | production `side_effects.rs:1647-1656` vs `#[cfg(test)]`-only `delete_tombstone_content` `:2363-2391` — the tests exercise the **copy**, not the production path |

The last one is the concerning case: two of the five `side_effects.rs` tests assert against
a function that production never calls.

#### D-14 — 30176 / 30177 lack the `d`-tag guard that 30175 has
`ingest.rs:1027-1082` gives 30175 (`PERSONA`) a strict slug grammar with the rationale
"an empty d-tag collapses every persona into the `(pubkey, 30175, "")` slot — last-write-wins
data loss" (`:1022-1024`). 30176 (`TEAM`) and 30177 (`MANAGED_AGENT`) share that exact
addressing shape (asserted at `buzz-core/src/kind.rs:785-787`), are validated in the same
scope arm (`ingest.rs:224`), and get only the generic 1024-byte length check
(`ingest.rs:2378`). An empty `d` collapses every team into one slot.

#### D-15 — `SPROUT_MAX_NOT_BEFORE_DELTA` is read per-request and duplicated
`ingest.rs:1325-1328` does `std::env::var(...).parse()` on **every** kind:30300 ingest — the
only request-time env read in the group. `nip11.rs:97` independently reads the same var to
advertise the limit, with an independently-written `31_536_000` default. No shared constant,
no `Config` field, absent from `.env.example`. The advertised horizon can silently diverge
from the enforced one.

#### D-16 — `KIND_PUSH_LEASE` has two definitions
`buzz-core/src/kind.rs:109` and `handlers/push_lease.rs:19` both define `30350`. `ingest.rs`
imports the `push_lease` copy (`:216`, `:451`, `:2156`), not the `buzz-core` one. AGENTS.md
says "All event kind integers are defined in `buzz-core/src/kind.rs`". It is also the one
accepted kind absent from `ALL_KINDS` (verified: 127 entries, 3 constants missing —
`KIND_AUTH`, `KIND_NOSTR_IDENTITY_BINDING`, `KIND_PUSH_LEASE`).

#### D-17 — Six success exit paths bypass the conformance emit seam
`ingest.rs:1381-1386` arms an `EmitGuard` whose `Drop` records an `ImplBug` if nothing was
emitted. These paths return `Ok` without emitting, and (being global/channel-less) never
emit a prior `AuthCheck` either: kind 1984 (`:1565-1569`), moderation commands
(`:1583-1587`), relay-admin (`:1812-1816`), 28936 (`:1898-1902`), the 9007
duplicate-channel branch (`:2118-2123`), and all 7 command kinds via `handle_command`.

Under a recording tracer this is a CoverageBreach — exactly what the guard exists to catch.
kind 42000 shows the intended pattern (`emit_product_feedback_success` `:133-154`, called
`:1546`) and is the only direct-route kind that follows it. Production impact is nil
(`AppState::tracer` is `NoopTracer`, `state.rs:798`), so severity is test-harness only —
but the seam's fail-closed claim (`:1355-1365`) is not currently true.

#### D-18 — `command_executor.rs` bypasses `buzz-db`'s event-insert API
`command_executor.rs:196-232` writes 11 `events` columns by hand. Consequences: command
events get no `thread_metadata` row and no `mentions` row (both handled inside
`Db::insert_event_with_thread_metadata`, `buzz-db/src/lib.rs:1379-1401`), and any future
`events` column with a `buzz-db`-side default is not applied here. Harmless today; a
schema-drift trap.

#### D-19 — `author_type_cache` is never invalidated
`ingest.rs:1328-1361` populates `state.author_type_cache` (`state.rs:613`) from
`get_agent_channel_policy` and never invalidates it. An agent registered after its first
event is labelled `"human"` for the cache's lifetime, skewing
`buzz_events_stored_total{author_type}`. Explicitly metric-only and never used for
authorization (`ingest.rs:1319-1321`), so impact is observability accuracy.

#### D-20 — `resolve_nip10_thread_meta` issues up to 4 sequential DB round-trips
`ingest.rs:564-717`. Two are parallelised via `tokio::join!` (`:606-611`), but the root
lookup (`:634` or `:665`) is sequential and only reached to compute a timestamp. On the
reply hot path this is 3 round-trips before the write, on top of the channel prefetch
(`:1739`), visibility (`:1752`), membership (`:1785`), and restriction (`:1616`) reads —
7+ round-trips before a single reply is stored.

---

### 4. Low

| ID | Finding | file:line |
|---|---|---|
| D-21 | 6 panic sites in production paths, against AGENTS.md's "do not introduce new `unwrap()`/`expect()` in production paths": `ingest.rs:2024`, `:2000`, `:2338`, `:2344`, `side_effects.rs:311`, `:314` | as listed |
| D-22 | `IngestResult.message` is a stringly-typed tagged union (`""` / `duplicate:` / `info:` / `response:{json}` / `{}`). `buzz-cli` string-matches it to derive exit code 5 (`buzz-cli/src/commands/mem.rs:105`, `commands/notes.rs:560`). A typo in either side silently changes CLI exit-code behaviour. | `ingest.rs:166-173` |
| D-23 | `ReactionEventInsertOutcome::Duplicate` returns `accepted: false` (`ingest.rs:2342-2347`) while the generic LWW duplicate returns `accepted: true` (`:2427-2431`). Two duplicate semantics, two response shapes, no documentation of the difference. | as listed |
| D-24 | 8 relay-minted kinds are absent from `is_relay_only_kind` (8000, 8001, 8002, 8003, 13535, 39000, 39001, 39002) plus 40099, so client submission yields the vague `restricted: unknown event kind` rather than `restricted: relay-only kind`. | `buzz-core/src/kind.rs:758-769` |
| D-25 | 26 declared kinds have no ingest path at all: 43001–43006 (jobs), 46001–46012 (workflow execution), 9009, 39003, 41, 1063, 41001, 48001, 49001. Declared, documented, unreachable. | `buzz-core/src/kind.rs:456-522`, `:528`, `:542` |
| D-26 | Workflow `add_reaction` POSTs to `/api/messages/{id}/reactions`, a route the relay never registers (`router.rs` registers 3 `/api/*` paths, none matching). Ingest has a native kind:7 path the action does not use. Any workflow with an `add_reaction` step 404s. | `buzz-workflow/src/executor.rs:883-917` |
| D-27 | `buzz_channels_created_total` is emitted from 5 sites; the argument that they do not double-count is prose, not structure (`side_effects.rs:1670-1679`). | `ingest.rs:2150`; `side_effects.rs:1704`, `:1731`; `command_executor.rs:373`, `:527` |
| D-28 | 40004–40007 (pin/bookmark/schedule/reminder), 48100–48106 (huddles), 40100 (canvas) are stored with only the `h`-tag gate — no target validation, unlike 40003 and 45002 which do check. Nothing prevents a `HUDDLE_ENDED` for a huddle that never started, or a pin of an event in another channel. | `ingest.rs:455-491` |
| D-29 | 30618 (`GIT_REPO_STATE`) is client-submittable with no repo-ownership check, and is also relay-minted (`side_effects.rs:2733`). A client cannot overwrite the relay head (different `d` coordinate owner) but can pollute the kind. | `ingest.rs:294` |
| D-30 | The 9002 `ttl` update path does not apply `ephemeral_ttl_override`, so a client can raise a channel's TTL post-creation to escape the deployment override that 9007 enforces. | `side_effects.rs:1449-1481` vs `ingest.rs:2125` |
| D-31 | Zero `#[tokio::test]` in the group. Every handler with a DB dependency — i.e. all 13 side-effect handlers, all 7 command handlers, `validate_admin_event`, `validate_edit_ownership`, `validate_forum_vote_target`, `verify_imeta_blobs` — is untested in-file. Coverage exists only in `crates/buzz-test-client/tests/`, which requires Postgres + Redis. | all four files |
| D-32 | `per_kind_scope_allowlist_covers_all_migrated_kinds` (`ingest.rs:2769-2822`) lists 44 kinds out of 81 accepted. It reads like a completeness check and is not one. | as listed |
| D-33 | `BUZZ_REQUIRE_RELAY_MEMBERSHIP` (defaults `false`, gates all of kind 28936) and `BUZZ_GIT_MAX_REPOS_PER_PUBKEY` are absent from `.env.example` (verified: only `RELAY_URL:49` and `BUZZ_EPHEMERAL_TTL_OVERRIDE:100` appear). | `.env.example` |
| D-34 | Relay-admin kinds are exempt from the ban/timeout gate (`ingest.rs:1613`) with only "retain their separate authorization policy" as justification. Moderation commands get the same exemption but with a documented in-handler ban re-check (`moderation_commands.rs:103-108`); the relay-admin path has no such stated re-check. | `ingest.rs:1600-1601` |
| D-35 | `is_global_only_kind` (44 kinds) nulls `channel_id` on write but the signed stray `h` tag still matches `#h` read filters — self-documented as a known limitation to be fixed "in the filter layer as a follow-up". | `ingest.rs:369-377` |
| D-36 | No documentation of this pipeline exists. `ARCHITECTURE.md` mentions ingest exactly once, in a route table (`ARCHITECTURE.md:620`). The 81-kind dispatch table, the validation order, and the side-effect partial-failure contract exist only in source. | `ARCHITECTURE.md:620` |

---

### 5. Prioritised remediation order

1. **D-02** add `buzz_side_effect_failures_total{kind}` (one-line change, immediately makes
   the largest silent-failure class visible).
2. **D-03** emit audit entries for 9030–9033, 9040–9044, and the 7 command kinds.
3. **D-04** add tests for `check_approver_spec` and `persist_command_event`'s domination
   logic — pure functions, no infrastructure needed.
4. **D-05 / D-06** decide: wire real scopes and channel tokens, or delete both and rename
   `required_scope_for_kind` to `is_kind_accepted`.
5. **D-09 / D-16 / D-13** delete the three dead counter entry points, unify
   `KIND_PUSH_LEASE`, and de-duplicate the four helper families. Pure subtraction.
6. **D-01 / D-07** split `ingest_event_inner` and `validate_admin_event`; add a Rust
   file-size guard mirroring `mobile/scripts/check-file-sizes.mjs`.
7. **D-10 / D-14** add the missing predicate-disjointness test and the 30176/30177 `d`-tag
   guard.
8. **D-11** centralise the prefix → `IngestError` mapping in one function.
9. **D-36** write the dispatch table into `ARCHITECTURE.md` (the api-surface aspect of this
   report is a drop-in starting point).


## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Debt

---

#### 1. Hard counts

| Metric | Value |
|---|---|
| Files in scope | 13 |
| Total LOC | 9,626 |
| Production LOC (outside `#[cfg(test)]`) | **5,168** (54%) |
| Test LOC | **4,309** (45%) |
| Routed endpoints owned by this group (method × path) | **30** |
| Endpoints with zero in-repo client | **6** (all `/operator/*`) + `/_mesh/demo/echo` + `/.well-known/nostr.json` (tests only) |
| Test functions | **159** |
| `#[ignore]`d tests | **28** (18%) |
| `unsafe` blocks | **0** (the word appears once, in a doc comment at `bridge.rs:303`) |
| `unwrap()` outside `#[cfg(test)]` | **0** |
| `expect()` outside `#[cfg(test)]` | **5** — all in `invite_token.rs` (`:119`, `:139`, `:172`, `:349`, `:374`), all infallible-by-construction HMAC/serialize calls |
| TODO / FIXME / XXX / HACK markers | **1** — `media.rs:303` `TODO(v2)` persistent storage quotas |
| Production functions in `bridge.rs` | 38 |
| Longest production function | `query_events_authed`, **362 lines** (`bridge.rs:947-1308`), 53 branch keywords |
| Dead code items | **6** (see D-01…D-06) |

##### `bridge.rs` complexity profile (production region = lines 1–2183)

| Function | Lines | Range | Branch keywords (`if`/`match`/`while`/`for`) |
|---|---|---|---|
| `query_events_authed` | **362** | `bridge.rs:947-1308` | **53** |
| `count_events_authed` | 199 | `bridge.rs:1378-1576` | 25 |
| `handle_channel_window_filter` | 178 | `bridge.rs:404-581` | 14 |
| `workflow_webhook` | 152 | `bridge.rs:1780-1931` | 11 |
| `handle_bridge_search` | 134 | `bridge.rs:1616-1749` | 17 |
| `submit_event_authed` | 126 | `bridge.rs:750-875` | 8 |
| `submit_event` | 90 | `bridge.rs:613-702` | 3 |
| `synthesize_presence` | 66 | `bridge.rs:1920-1985` | 8 |
| `verify_bridge_auth_with_options` | 57 | `bridge.rs:72-128` | 5 |
| `authorize_moderation_read` | 47 | `bridge.rs:2008-2054` | 3 |

`query_events_authed` alone contains **five** sequential dispatch stages over the same
`raw_filters.iter().zip(filters.iter()).enumerate()` (window `:1017`, feed `:1029`, thread `:1112`,
catch-all build `:1188`, catch-all post-process `:1266`), coordinated by a shared
`handled: HashSet<usize>`. Each stage is independently comprehensible; the composition is not.

---

#### 2. Findings, prioritized

##### CRITICAL

**D-C1 — `require_auth_token` defaults to `false`, enabling unsigned pubkey impersonation on 6 routes**
`bridge.rs:118-127` accepts a bare `X-Pubkey: <hex>` header in place of a NIP-98 signature when the
flag is off; `config.rs:475-477` defaults it off; `bridge.rs:640`, `:908`, `:1341`, `:2033` pass it
through. Replay protection is bypassed on that path too (`bridge.rs:122-125` → `:150-153`). The
variable is absent from `.env.example`, and the startup warning
("REST API requests bypass token auth", `config.rs:588-593`) understates the effect.
*Fix:* default to `true`; make the dev path require an explicit `BUZZ_DEV_UNSAFE_HEADER_AUTH=1`; add
the var to `.env.example` with the real consequence spelled out.

##### HIGH

**D-H1 — the admin API's only credential is a spoofable `Host` header**
`admin/auth.rs:16-33`: `admin` configured → `Host == config.admin.host` → `Origin` checked **only if
present**. A non-browser client sending no `Origin` passes. Behind it sit 5 cross-tenant read
endpoints, 3 of which take no `CommunityId` at all (`admin/mod.rs:132`, `:155`, `:184`), plus raw
blob streaming (`:226`). `docs/admin/README.md:64-70` accepts this ("the human trust boundary remains
the private admin ingress") and explicitly disclaims per-operator attribution, so this is a knowing
trade — but there is no defence in depth and the access log records no principal
(`admin/mod.rs:228-233`).
*Fix:* require a shared secret or mTLS-derived header in addition to `Host`; reject requests with no
`Origin` from a browser-shaped UA; log a principal.

**D-H2 — `POST /_mesh/demo/echo` derives the tenant from the request body**
`mesh_demo.rs:50-51` + `:99-101`: `community_id` is client-supplied and used to acquire a Redis fenced
session lease. This is the only route in the module that bypasses the row-zero `Host` boundary
(`bridge.rs:621` etc.), and it has no authentication. Double-flag-gated (both default off,
`config.rs:509-518`) and self-described as "Not a product flow… stays demo-gated until it is deleted"
(`mesh_demo.rs:21-23`).
*Secondary:* the claimed 404-indistinguishability (`mesh_demo.rs:71-73`) is false — `Json<T>` is a
`FromRequest` extractor evaluated **before** the flag gate, so a malformed body returns 400/422 and
reveals the route on a mesh-off relay.
*Fix:* delete it, or gate it behind the operator allowlist and bind the tenant from `Host`; move the
body parse inside the handler so the 404 fires first.

**D-H3 — `api/events.rs` is 100% dead, and its doc comment asserts a false reason**
`api/events.rs:1-5` re-exports three bridge handlers "for backward compatibility with router.rs", but
`router.rs:71-73` binds `api::bridge::*` directly. A repo-wide grep for `api::events`,
`events::submit_event`, `events::query_events`, `events::count_events` returns **zero** hits.
*Fix:* delete the file and the `pub mod events;` line (`api/mod.rs:6`).

**D-H4 — `webhook_secret::strip_secret` has zero production callers**
`webhook_secret.rs:52-68`. Its doc says "Use this before returning a definition to API callers — the
secret must never be embedded in a response body (it is returned once, at creation time, via a
dedicated `webhook_secret` field)." Nothing in the HTTP surface calls it. The secret lives inside the
workflow definition JSON (`webhook_secret.rs:3-5`, `:34-41`), so any code path that serializes a
definition back to a client leaks it, and there is no enforcement point.
*Fix:* either wire it into every definition-returning path, or delete it and replace with a
serialization-level redaction that cannot be forgotten.

##### MEDIUM

**D-M1 — `query_events_authed` is a 362-line, 53-branch function**
`bridge.rs:947-1308`. Five dispatch stages coordinated by a mutable `handled: HashSet<usize>` and a
mutable `events: Vec<Value>` accumulator. Adding a sixth filter extension means auditing every
earlier stage's `continue`/`handled.insert` bookkeeping to be sure the new one is reachable and
mutually exclusive.
*Fix:* extract each stage into a `handle_*_filters(…) -> Result<HandledSet, …>` and drive them from a
list, so precedence is data rather than control flow.

**D-M2 — `/moderation/*` reads have no rate limit**
`authorize_moderation_read` (`bridge.rs:2008-2054`) does bind → NIP-98 → replay → capability, but never
calls `enforce_http_admission`. They are the only NIP-98 bridge routes without a limiter and return up
to 500 rows including `private_reason` (`bridge.rs:2183`). Combined with D-C1, an unmetered bulk
export of the moderation record on a default-config relay.
*Fix:* add `enforce_http_admission` to the prelude.

**D-M3 — `media_upload_rate_limiter` is an unbounded, never-swept `DashMap`**
`state.rs:38-39`, `:592`, `:774`; the only read is `media.rs:97`. No capacity, no TTL, no sweep task.
`invites.rs:36-43` documents the exact hazard for its own limiter — "a pre-membership caller can
cheaply create fresh Nostr keypairs; retaining one immortal entry per key would make the limiter
itself an unbounded-memory denial-of-service vector" — and solves it with moka (capacity 10 000 + 60 s
TTL, `state.rs:775-780`). On an open relay (default) media upload authorization is a valid Blossom
signature from any key (`media.rs:196-206`), so the same argument applies verbatim and was not applied.
*Fix:* switch to the same moka pattern; the `(u32, Instant)` value already encodes the window.

**D-M4 — `POST /api/invites/accept-policy` is an unauthenticated MAC-minting oracle**
`invites.rs:162-190`: no NIP-98, no pubkey binding, no rate limit. It signs an attacker-chosen `code`
string with `derive_invite_key(relay_keypair)` — the same key that signs invite codes
(`invite_token.rs:112-117`). Consequences: (a) the consent gate proves "someone POSTed the right
version", not "this pubkey accepted", so the acceptance persisted at `invites.rs:324-338` is not
attributable; (b) no in-payload domain-separation label distinguishes `InvitePayload` from
`PolicyAcceptancePayload` — cross-type confusion is prevented only by serde's missing-field
strictness (`invite_token.rs:64-74` vs `:335-343`), so adding an optional field to either struct
opens it.
*Fix:* require NIP-98 on `accept-policy` and bind the receipt to the signer's pubkey; add an explicit
purpose byte/label to both signed payloads.

**D-M5 — two limiters in this module are per-pod while the bridge's is cluster-wide**
`media_upload_rate_limiter` + `media_uploads_in_flight` (DashMap/`Semaphore`, `state.rs:592`, `:600`,
`:730`) and `invite_claim_rate_limiter` (moka, `state.rs:597-598`) are process-local, whereas
`admission_rate_limiter` is Redis-backed (`state.rs:712`). On N pods the effective media-upload and
invite-claim budgets are N× the configured value, and the per-pubkey upload concurrency ceiling is
N×2. No comment acknowledges the multiplication.
*Fix:* document the per-pod semantics in the config field docs, or move both to Redis.

**D-M6 — operator archive / unarchive / transfer are unattributed**
`operator.rs:355` binds the authenticated operator to `let _pubkey` and discards it;
`operator.rs:209` and `:271` call `authorize_operator_request(...).await?;` with no binding. The
resulting log lines (`operator.rs:281-282`, `:299`, `:429`) carry community + host but no actor, and
no audit entry is written. These are the two highest-impact operations on the surface (community
archival and ownership transfer). `provision_community` is the only one that threads the pubkey
through (`operator.rs:157`, `:179`).
*Fix:* include the operator pubkey in every log line and write a `buzz-audit` entry.

**D-M7 — reflected request content in 4xx bodies contradicts the log-truncation reasoning**
`bridge.rs:735-742` reasons at length that `serde_json`'s `Display` "embeds the offending input
verbatim… would otherwise reflect attacker-controlled text into a log line at full size" and fixes the
log path — then returns that same string to the caller at `bridge.rs:745`. Same pattern at `:970`,
`:1381`; `invites.rs:172-176`, `:252-257`, `:302`; `operator.rs:163-167`. And `submit_event`
truncates `IngestError::Rejected` to 256 bytes for logs (`bridge.rs:850`) but returns it untruncated
(`:855`). Impact is bounded (self-reflection only), but the defence has no response-side twin.
*Fix:* return a bounded, structured 400 (`{error, category, line, column}`) instead of the raw
`Display`.

**D-M8 — `security_headers` covers only the admin sub-router**
`admin/mod.rs:38`, `:43-61`, mounted only when `BUZZ_ADMIN_HOST` is set (`router.rs:56-59`). Every
other route — including the two unauthenticated `text/html` policy pages at `invites.rs:95-124` —
gets no `X-Frame-Options`, `Referrer-Policy`, or CSP. The policy pages do escape raw HTML in operator
Markdown (`invites.rs:126-155`, test `:1254-1271`), so stored XSS is closed, but the pages are
framable and there is no header-level defence in depth.
*Fix:* move `security_headers` (minus `no-store`) onto the merged router in `router.rs:187-190`.

**D-M9 — six routed endpoints have no in-repo client and only `#[ignore]`d coverage**
All 6 `/operator/*` routes (`router.rs:74-93`). Repo-wide grep for `operator/communities` returns only
relay source and its own tests. All **11** tests in `operator.rs` are `#[ignore = "requires Postgres"]`,
so `just test-unit` exercises **zero** operator code. The control plane that consumes them lives
outside this repository.
*Fix:* add non-ignored unit coverage for `authorize_operator_request`'s allowlist and payload-tag
logic (both are pure given a `HeaderMap`), and record the external consumer in `AGENTS.md`.

**D-M10 — media read auth is dead code in the default configuration**
`verify_blossom_get_auth` is defined in `buzz-media` (`auth.rs:207`) and its **only** repo-wide call
site is `media.rs:502`, behind `require_media_get_auth` which defaults to `false`
(`config.rs:682-689`). `authenticate_media_read` returns early at `media.rs:491-494` when off.
Additionally, the admin attachment route calls `serve_blob_for_tenant` directly
(`admin/mod.rs:226`) and so never applies the flag, while `docs/admin/README.md:44-46` says
"Existing community `/media/*` authorization is unchanged, including `BUZZ_REQUIRE_MEDIA_GET_AUTH`" —
technically true of `/media/*` but easy to misread as covering the admin route.
*Fix:* flip the default once clients are rolled out (the "staged client rollout" justification at
`config.rs:973-976` implies a planned flip that has no tracking marker in the code).

##### LOW

**D-L1 — three incompatible handler error dialects and two error-envelope shapes**
Tuple `(StatusCode, Json<Value>)` (bridge/invites/operator), `MediaError` (media), `ApiError`
(admin). Envelopes: `{"error": "<string>"}` (`api/mod.rs:19-21`) vs
`{"error":{"code","message","requestId"}}` (`admin/error.rs:16-28`). Clients cannot share an error
parser across the surface.

**D-L2 — `admin/error.rs` mints a fresh `requestId` per response**
`admin/error.rs:26`, `:71`: `uuid::Uuid::new_v4()` at serialization time, not a correlated trace id.
The field's name promises correlation it cannot deliver.

**D-L3 — operator status codes are classified by error-string prefix**
`operator.rs:180-199` matches `msg.starts_with("actor not authorized")`, `== "community already exists"`,
`starts_with("limit_reached:")`, etc. A wording change in
`handlers/community_provisioning.rs` silently reclassifies HTTP statuses.
*Fix:* return a typed error enum from the provisioning module.

**D-L4 — `/hooks/{id}` is a four-state workflow oracle**
`bridge.rs:1787-1826` distinguishes unknown workflow (404), non-webhook trigger (400), webhook with no
secret (401 with a descriptive `"re-save the workflow to generate one"`), and wrong secret (401
generic). Also, the `?secret=` fallback (`:1752-1757`, `:1809`) puts a bearer credential into access
logs and `Referer` with no warning beyond the doc comment.

**D-L5 — `/count` sums across filters without dedup**
`bridge.rs:1432`, `:1575`: an event matching two filters in one request is counted twice. `/query`
deduplicates in the feed and search paths (`bridge.rs:1069`, `:1727`) but `/count` never does.

**D-L6 — tenant-binding preamble copy-pasted nine times**
`bridge.rs:621-633`, `:888-901`, `:1321-1334`, `:1777-1786`, `:2013-2025`; `invites.rs:198-207`;
`media.rs:154-166`; `nip05.rs:31-40`. Only `media.rs` factors it
(`bind_media_read_tenant`, `media.rs:478-488`). A future change to the fail-closed behaviour must be
applied nine times.

**D-L7 — stale `#[allow(dead_code)]` on a live function**
`api/mod.rs:28` on `not_found`, which is used at `bridge.rs:1803` and `:1792`.

**D-L8 — `check_relay_membership` / `MembershipDecision` abstraction has one consumer**
`api/mod.rs:46`, `:61`; only caller is `enforce_relay_membership` at `:130`. The transport-neutral
enum buys nothing today. Its doc at `api/mod.rs:35-38` also **omits `handlers/auth.rs:217`** — the
WebSocket door — from the list of callers.

**D-L9 — `HttpAuthMethod` is write-only dead data**
`bridge.rs:830` hardcodes `Nip98` even on the `X-Pubkey` path; `HttpAuthMethod::DevPubkey`
(`handlers/ingest.rs:58`) has zero constructors repo-wide; no code reads `IngestAuth::Http.auth_method`.
So downstream logic and logs cannot distinguish a signed request from an unsigned one — which makes
D-C1 harder to detect operationally.

**D-L10 — worst-case upload memory is ~800 MB per pod**
`media.rs:362-367` buffers `max(max_image_bytes, max_file_bytes)` = 100 MB by default for every
non-video upload, and `media_max_concurrent_uploads` defaults to 8 (`config.rs:663-668`,
`state.rs:730`). Video streams to disk, so the exposure is the generic-file path only.

**D-L11 — the `TODO(v2)` storage-quota gap is real and unbounded**
`media.rs:302-304`: "Admission limits below bound active parser/storage work, but they do not cap
durable bytes stored." With `media_uploads_per_minute = 30` and `max_file_bytes = 100 MB`, one key can
durably store ~180 GB/hour per pod. The only TODO marker in all 13 files.

**D-L12 — three duplicated NIP-98 test-header builders**
`invites.rs:505-519` (`nip98_auth_header`), `operator.rs:596-616` (`nip98_auth_header` +
`nip98_auth_header_without_payload`), `bridge.rs:2380-2404` (`build_nip98_event_json` +
`nip98_auth_headers`). Near-identical bodies; a change to NIP-98 tag shape needs three edits.

**D-L13 — `#[ignore]`d tests skip vacuously in two files but panic in others**
`invites.rs:664-666`, `:772-774`, `:930-932`, `:986-988` and `operator.rs:686-688`, `:713-715`, …
use `let Some(state) = … else { return; }`, so the test **passes** when Postgres is unreachable.
`bridge.rs:3423-3425` and `invites.rs:1074-1076` correctly `panic!`/`expect` with an actionable
message. The former hides infrastructure regressions in CI.

**D-L14 — boolean env parsing is inconsistent across four spellings**
`require_media_get_auth` accepts `true`/`1`/`yes`/`on` (`config.rs:682-689`);
`require_auth_token`/`require_relay_membership`/`allow_nip_oa_auth` accept only `true`/`1`
(`config.rs:475-477`, `:483-485`, `:520-522`); mesh vars accept `on`/`true`/`1`
(`config.rs:509-518`). `BUZZ_REQUIRE_AUTH_TOKEN=yes` silently means `false`.

**D-L15 — `serve_blob_for_tenant` re-validates a path its callers already validated**
`media.rs:604`/`:619`/`:630`: `get_blob` calls `validate_media_path` then `serve_blob_for_tenant`
calls it again. Harmless, but it signals uncertainty about which layer owns the invariant — and
`admin/mod.rs:226` relies on the inner call for its own safety.

**D-L16 — `#[allow(private_interfaces)]` splits a doc comment**
`media.rs:296-304`: the attribute sits between two `///` blocks on `upload_blob`, so rustdoc renders
only the second half. Cosmetic but confusing.

**D-L17 — `nostr_nip05` folds DB errors into a 200**
`nip05.rs:64` uses a catch-all `_ =>` arm, so a Postgres failure is indistinguishable from a miss.
Deliberate for privacy (see the doc at `:31-35`) but means NIP-05 outages are invisible to callers
and unmetered.

**D-L18 — `buzz_media_uploads_total` carries an unbounded `community` label**
`media.rs:419-424` labels by tenant host, so series count grows with tenant count. Fine today,
a Prometheus-cardinality problem on a large multi-tenant deployment. Contrast the deliberate
6-value `mime` allowlist immediately above it (`:410-417`).

---

#### 3. Documentation deltas (verified against code)

| Doc claim | Reality | Evidence |
|---|---|---|
| `ARCHITECTURE.md:823` (§9 #2): "No rate limiting implementation… none are enforced" | Redis rate limiting **is** enforced on `/events`, `/query`, `/count` | `bridge.rs:24-56`, called `:760`, `:955`, `:1386`; limiter `state.rs:712` |
| `ARCHITECTURE.md:390`: "No Redis-backed rate limiter exists anywhere in the codebase" | `RedisRateLimiter` is imported from `buzz_pubsub::rate_limiter` | `state.rs:26`, `:712` |
| `ARCHITECTURE.md:459`: `buzz-pubsub` "Does NOT implement the rate limiter" | it does | `state.rs:26` |
| `ARCHITECTURE.md:824` (§9 #3): "`/api/presence` returns online/away status only" | no such route exists; presence is synthesized inside `POST /query` | `router.rs:32-190`; `bridge.rs:1920-1985`; removal noted at `mobile/test/features/profile/presence_cache_provider_test.dart:13` |
| `ARCHITECTURE.md:623`: `PUT /media/upload` "50 MB limit" | route body limit is `max(max_image, max_video)` = **500 MB** default | `router.rs:33-36`; `config.rs:657-672` |
| `ARCHITECTURE.md:610-628` endpoint table | omits `PUT /upload`, 6 operator routes, 6 invite/policy routes, 3 moderation routes, `/_mesh/demo/echo`, `/huddle/{channel_id}/audio`, 5 admin routes, `/_status`, `/_mesh` | `router.rs:39`, `:74-128`, `:57-59`, `:229-230` |
| `AGENTS.md` "Nostr-first HTTP surface" / "deliberately narrow" list | **14 routed endpoints in this group** sit outside the documented set (all operator, all invite/policy, all moderation, mesh demo) | enumerated in the api-surface aspect §4 |
| `api/events.rs:3`: "re-exports bridge handlers for backward compatibility with router.rs" | `router.rs:71-73` uses `api::bridge::*`; zero references to `api::events` repo-wide | `api/events.rs:1-5` |
| `api/mod.rs:35-38`: membership gate "Called by `media.rs`, `bridge.rs`, `git/transport.rs`, and `audio/handler.rs`" | omits `handlers/auth.rs:217`, the WebSocket door | `handlers/auth.rs:217` |
| `webhook_secret.rs:52-56`: `strip_secret` "Use this before returning a definition to API callers" | zero production callers | `webhook_secret.rs:57` |
| `mesh_demo.rs:71-73`: "404 (not 403) so a non-demo deployment is indistinguishable from one without the route" | the `Json` extractor rejects malformed bodies with 400/422 before the gate | `mesh_demo.rs:60-62` |
| `bridge.rs:1-4`: "authenticated via NIP-98 signed events" | also accepts the unsigned `X-Pubkey` header by default | `bridge.rs:118-127`; `config.rs:475-477` |
| p-gate parity comment "same enforcement as WS REQ handler" (`bridge.rs:979-980`, `:1403`) | **HTTP is stricter**: unconditional, vs WS which gates only `channel_id.is_none()` | `bridge.rs:981-998`, `:1404-1421` vs `handlers/req.rs:183-205` |
| `handlers/ingest.rs:57-58`: `DevPubkey` "X-Pubkey dev-mode header (backward compat during transition)" | never constructed; `bridge.rs:830` always reports `Nip98` | `handlers/ingest.rs:58` |
| `docs/admin/README.md:44-46`: "Existing community `/media/*` authorization is unchanged, including `BUZZ_REQUIRE_MEDIA_GET_AUTH`" | true of `/media/*`, but the admin attachment route bypasses `authenticate_media_read` entirely | `admin/mod.rs:226`; `media.rs:619` |
| `.env.example` | omits 11 vars this module consumes, including `BUZZ_REQUIRE_AUTH_TOKEN`, `BUZZ_ADMIN_HOST`, and both `RELAY_OPERATOR_*` | see the configuration aspect §4 |

---

#### 4. Suggested ordering

1. **D-C1** — flip `require_auth_token` to default `true` and document it. Single highest-value change.
2. **D-H1**, **D-H2** — add a real credential to the admin surface; delete or gate the mesh demo.
3. **D-H3**, **D-H4** — delete `api/events.rs`; resolve `strip_secret` (wire it or replace it).
4. **D-M2**, **D-M3**, **D-M4** — add the moderation rate limit; bound the media limiter map;
   authenticate `accept-policy` and add a domain-separation label.
5. **D-M1** — decompose `query_events_authed` before the next filter extension lands.
6. **D-M6**, **D-M8**, **D-M10** — attribute operator mutations; widen `security_headers`; plan the
   `require_media_get_auth` flip.
7. Correct the `ARCHITECTURE.md` / `AGENTS.md` / `.env.example` deltas in §3 — five of them
   (rate limiting ×3, `/api/presence`, the 50 MB claim) actively mislead anyone reasoning about
   the relay's security posture.


## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: Debt

#### 1. Quantified baseline

| Metric | Value |
|---|---|
| Files | 10 |
| Total LOC | 8 935 |
| Production LOC (excluding `#[cfg(test)]` regions) | ≈ 5 830 |
| Test LOC | ≈ 3 105 (35 % of the module) |
| Production functions | 124 |
| Unit tests | 99 (76 `#[test]`, 23 `#[tokio::test]`) |
| `#[ignore]`d tests in-module | **0** |
| `#[ignore]`d E2E tests covering this module | 2 (`crates/buzz-test-client/tests/e2e_git.rs:195`, `:333`) |
| Env-gated tests that silently pass when the gate is off | 10 (`store.rs:1018`, `:1046`, `:1127`, `:1146`; `hydrate.rs:642`, `:766`, `:797`; `cas_publish.rs:1618`) |
| Production `unwrap()`/`expect()` | **17** (`transport.rs` 10, `store.rs` 5, `cas_publish.rs` 1, `policy.rs` 1) |
| `unsafe` | **0** |
| `TODO`/`FIXME`/`XXX`/`HACK` markers | **0** |
| Production `Command::new("git")` sites | **10** |
| HTTP endpoints | **4** |
| Public items with zero production callers | **1** (`policy::generate_hook_hmac`) |
| Public items with zero *external* callers (over-export) | 12 |
| `#![allow(dead_code)]` blanket suppressions | 1 (`store.rs:25`) |
| Stale doc comments identified | 4 |
| Metrics emitted | 8 counters, 11 histograms, 3 gauges |

##### File sizes

| File | LOC | Assessment |
|---|---|---|
| `transport.rs` | 2 288 | over-scoped: auth extractor + pkt-line codec + 3 handlers + 2 subprocess runners + 4 stream adapters + the fence |
| `cas_publish.rs` | 1 891 | 560 lines of that is the compaction subsystem |
| `store.rs` | 1 164 | 308 of them are the conformance probe |
| `hydrate.rs` | 893 | 412 production / 412 test |
| `policy.rs` | 775 | 334 production / 332 test (two of which embed bash scripts) |
| `pack_cache.rs` | 686 | cohesive |
| `manifest.rs` | 570 | cohesive |
| `manifest_event.rs` | 395 | cohesive, pure |
| `hook.rs` | 207 | 118 of them are an embedded bash script |
| `mod.rs` | 66 | cohesive |

##### Longest production functions

| Lines | Location | Function |
|---|---|---|
| 308 | `store.rs:571-878` | `run_conformance_probe` |
| 265 | `cas_publish.rs:1021-1285` | `cas_publish_inner` |
| 242 | `policy.rs:173-414` | `hook_policy_check` |
| 204 | `transport.rs:1548-1751` | `finalize_push` |
| 147 | `transport.rs:79-225` | `GitAuth::from_request_parts` |
| 146 | `transport.rs:994-1139` | `run_git_at` |
| 145 | `cas_publish.rs:666-810` | `capture_compacted_packs` |
| 127 | `transport.rs:596-722` | `info_refs_subprocess` |
| 105 | `transport.rs:858-962` | `receive_pack` |
| 87 | `hydrate.rs:293-379` | `materialize_manifest` |

Four functions exceed 200 lines. `run_conformance_probe` is four independent phases in one body with three near-duplicate error-construction blocks; `cas_publish_inner` carries five mutable locals (`compaction_failure`, `compaction_observation`, `compacted_manifest`, plus the pack/manifest keys) threaded through a compaction branch and six early returns; `hook_policy_check` is nine numbered steps with a `return 403` in each; `finalize_push` interleaves the fence, a five-arm error mapping, and the 30618 emission.

---

#### 2. Prioritized findings

##### D-GIT-01 — Critical: pointer creation is ungated by announcement, reservation, or quota

`receive_pack` performs no kind:30617 or `git_repo_names` lookup (`transport.rs:858-962`). Because git skips `execute_commands` — and therefore the pre-receive hook — when a client sends zero ref commands, `PackOutput.ok` is true (`transport.rs:1136-1139`) and `finalize_push` proceeds to `cas_publish`, which will `put_pointer(IfNoneMatchStar)` a pointer into existence for any `(owner, repo)` (`cas_publish.rs:1208-1237`). Two effects: pointer-namespace squatting that bypasses the reservation and per-pubkey quota (`handlers/side_effects.rs:2441-2513`), and an authenticated ETag-bump that turns concurrent legitimate pushes into 409s.

Also falsifies the design's stated invariant "pointer-absence means never announced" (`docs/git-on-object-storage.md` §Implementation Correspondence), on which `info_refs`'s unambiguous 404 depends.

**Fix:** gate `receive_pack` on an existing pointer *or* a live `git_repo_names` reservation before hydrating, and/or reject a receive-pack request whose report-status shows zero ref commands before reaching `cas_publish`.

*Basis: code structure plus `git receive-pack` command-dispatch semantics; no behavioral test exists to confirm.*

##### D-GIT-02 — Critical: no read authorization on any git path

Neither `info_refs` (`transport.rs:539-594`) nor `upload_pack` (`:786-827`) consults kind:30617, the `buzz-channel` binding, channel membership, or channel visibility. Since `require_relay_membership` defaults to **false** (`config.rs:483-485`, pinned `config.rs:954-955`), the effective read policy on a default relay is "anyone with a keypair." The module doc says "No public repos for v1" (`transport.rs:60-66`), which does not describe the code.

**Fix:** mirror the push path's kind:30617 + channel-role resolution on the read path, or state the model honestly in the doc and require membership for git.

##### D-GIT-03 — High: silent ref-wipe path for non-40-hex OIDs

`snapshot_workspace_state` `warn!`s and drops any ref whose oid is not exactly 40 hex (`cas_publish.rs:325-328`), while `is_hex_oid` (`manifest.rs:156`), `manifest_event::is_valid_oid` (`:129`), and the advertisement's `object-format` derivation (`transport.rs:473-479`) all accept 64. A SHA-256 repository would snapshot to `refs: {}` and the CAS would publish a manifest that deletes every ref — `validate()` accepts it (`manifest.rs:204-245`). Unreachable today only because `init_bare_repo` hardcodes SHA-1 (`hydrate.rs:181-184`).

**Fix:** make the snapshot fail closed (`CasError::PackCapture`) on any unparseable oid line, and either commit to SHA-1 everywhere or support SHA-256 end to end.

##### D-GIT-04 — High: `hydrate::run_git` has no timeout and runs `index-pack` on attacker-influenced bytes

`run_git` (`hydrate.rs:451-470`) awaits `.output()` with no `tokio::time::timeout`, and one of its four call sites is `git index-pack <pack>` (`hydrate.rs:420`) over pack bytes fetched from the object store. Every other subprocess in the module is bounded (120/300/600 s). Combined with `index-pack` running without `--strict` (deliberately, `hydrate.rs:410-416`), a pathological pack can occupy a `git_semaphore` permit indefinitely; 20 such requests stop git relay-wide.

**Fix:** wrap `run_git` in a timeout; consider a separate, smaller bound for `verify-pack`/`index-pack`.

##### D-GIT-05 — High: the publish fence's real invariant is undocumented

`transport.rs:1128-1135` presents `receive_pack_report_rejected` as "the primary fence for a denied push." It is not: a client that never negotiates `report-status` produces no status stream, so a hook decline yields `ok == true` and the CAS runs. It is safe anyway because a pre-receive decline rejects all commands and never migrates quarantined objects, so `refs_after == parent.refs` and the CAS installs an identical digest. **The safety comes from the workspace snapshot, not the status parse** — and that argument appears nowhere in the code or in `docs/git-on-object-storage.md`. A future refactor that reorders the snapshot or trusts `ok` for something other than "skip a redundant CAS" would silently lose the property.

**Fix:** write the invariant down at `finalize_push`, and add a test that a hook-declined push publishes nothing *with* report-status disabled.

##### D-GIT-06 — High: `git_semaphore` is global, and the fast path bypasses it entirely

One `Semaphore::new(20)` per process, not per tenant and not per pubkey (`state.rs:729`). Any authenticated caller can hold all 20 permits with slow pushes and starve git for every community on the pod. Separately, the `info/refs` fast path takes **no** permit (`transport.rs:552-577`), so unbounded concurrent requests each issue two S3 round-trips with no local backpressure. No git route is rate-limited (no `enforce_http_admission`, no `admission_rate_limiter` — verified by absence in `transport.rs`).

**Fix:** per-tenant (or per-pubkey) sub-limits, and an admission limiter on the fast path.

##### D-GIT-07 — High: no metrics for push outcome or authorization decisions

There is no counter for push success / 409 conflict / 400 invalid-manifest / 413, and none for policy allow/deny. The design explicitly defers the "add a best-effort local lock" decision to metrics ("if contention ever shows up in metrics" — `docs/git-on-object-storage.md` §Scope; echoed at `transport.rs:872-875`), but the CAS-conflict signal that decision depends on **does not exist**. `buzz_git_pack_compactions_total{outcome="cas_conflict"}` fires only on the compaction path (`cas_publish.rs:1265-1272`), so a normal-path conflict is invisible.

**Fix:** `buzz_git_push_total{outcome}` and `buzz_git_policy_decisions_total{decision}`.

##### D-GIT-08 — Medium: the `idx/` object is the only non-content-addressed object

`idx/<pack-digest>` is keyed by the *pack's* digest, not by its own bytes (`store.rs:227-241`), so `get_idx` cannot digest-verify it (`hydrate.rs:388-397`). The only defense is `git verify-pack` with regeneration on failure (`:398-406`). Every other object in the design is digest-verified, and `store.rs:19-24` advertises exactly that property. An attacker with bucket write who wins the create race for a pack lacking a sidecar controls bytes git will parse.

**Fix:** key the sidecar on its own digest and record it in the manifest, or drop the sidecar tier and always regenerate (measuring the cost first).

##### D-GIT-09 — Medium: four stale doc comments, one of which masks dead code

| Site | Claim | Reality |
|---|---|---|
| `store.rs:25` | `#![allow(dead_code)] // wired in by the push path in a follow-up commit` | the push path is wired in (`cas_publish.rs:826`, `:1194`, `:1235`); the blanket allow now hides genuinely dead items module-wide |
| `hydrate.rs:24-30` | "We narrow `#[allow(dead_code)]` to those specific items" | there is no `#[allow(dead_code)]` in `hydrate.rs` |
| `hydrate.rs:456-457` | "Match transport.rs's `harden_git_env` semantics" | omits `GIT_CONFIG_GLOBAL`; relies on `HOME=<cwd>` instead |
| `cas_publish.rs:490-497` | "Deduplicate against the same-oid case — no point feeding `X ^X`" | no dedup; every ref value is pushed unconditionally (`:498-517`). `capture_compacted_packs` **does** dedup (`:696-702`) |

##### D-GIT-10 — Medium: design-doc drift

| Doc claim | Site | Reality |
|---|---|---|
| `build_git_response` "has two call sites … shared with the read paths — info_refs, upload_pack" | doc §Implementation Correspondence | both call sites are inside `finalize_push` (`transport.rs:1574`, `:1748`); read paths use `stream_git_read` (`:1414`) and an inline builder (`:715-721`). The doc's caveat about the discriminator is now moot. |
| `ParentState` at `cas_publish.rs:154`, `cas_publish` at `:410`, `CasError::Conflict` at `:92`, `PushContext` at `transport.rs:643`, `finalize_push` at `:674`, `build_git_response` at `:627` | doc §Implementation Correspondence table | actual: `:215`, `:997`, `:105`, `transport.rs:1514`, `:1548`, `:1500`. The doc warns line numbers are pinned at landing time, but the table is the only index a reviewer has. |
| "pointer-absence means never announced" | doc §Implementation Correspondence | see D-GIT-01 |
| Spec §Push step 4 "validate Δ against `m_before.refs`" | doc §Protocol | no counterpart inside `cas_publish`; the check lives entirely in the pre-receive hook against the workspace (`hook.rs:32-150` + `policy.rs:173-417`). Equivalent in effect, but the spec-to-code map is misleading. |
| "A behavioral integration test for runtime ordering (publish-before-response) … is the belt-and-suspenders item to add once a mockable-CAS seam exists" | doc §Implementation Correspondence | still absent; no mockable-CAS seam exists (`GitStore` is a concrete struct with no trait, `store.rs:168-172`) |

##### D-GIT-11 — Medium: `GitStore` is not mockable, so the CAS protocol has no unit-test coverage

`GitStore` is a concrete struct (`store.rs:168-172`) with no trait abstraction. Consequence: `cas_publish` — the 265-line heart of the module — has **zero** tests that exercise the CAS itself. Its 18 tests cover only pure helpers (`compose_after`, `digest_from_*_key`, `resolve_published_head`, `should_compact`, `compacted_pack_set_is_usable`) plus two that need a real `git` binary. The lost-race path, the winner re-read, the compaction/CAS interaction, and the conflict-to-409 mapping are exercised only by an env-gated live test (`cas_publish.rs:1618-1832`) and an `#[ignore]`d E2E (`e2e_git.rs:333`). This is the largest coverage gap in the module and the direct blocker for the fence test the doc still owes.

##### D-GIT-12 — Medium: `require_localhost` is untested and silently incompatible with the UDS listener

`mod.rs` has no test module. `require_localhost` returns 403 whenever `ConnectInfo` is absent (`mod.rs:41-49`), and the UDS listener serves the same router with `.into_make_service()` (`main.rs:1182`) while the TCP listeners use `.into_make_service_with_connect_info` (`:1193`, `:1214`). Harmless today because the hook always dials `127.0.0.1` (`transport.rs:911-915`), but it is an undocumented coupling: a UDS-only deployment mode would silently break every push, and nothing tests either the fail-closed path or the loopback-accept path.

##### D-GIT-13 — Medium: 17 production `unwrap()`/`expect()` against an absolute repo rule

AGENTS.md forbids new `unwrap()`/`expect()` in production paths. All 17 are structurally infallible (`Response::builder().body(..)`, `Stdio::piped()` handle takes, a const header parse, a guarded `winners == 1`), but the rule admits no exceptions and the count will drift. `cas_publish.rs:410` (`pack_path.to_str().unwrap()`) is the one worth fixing on its merits — the sibling compaction path handles the identical case with a typed error 290 lines later (`:702-705`).

##### D-GIT-14 — Medium: pack-cache accounting can exceed its bound and drift

`prune` only evicts entries with `Arc::strong_count == 1`; if every entry is in flight the loop breaks and the cache stays over `max_bytes` (`pack_cache.rs:365-390`). Bypassed over-capacity entries are never counted at all (`:299-313`). And a `lookup` that finds an entry whose files have vanished returns `None` without removing the record, so its bytes keep counting until the next `insert` replaces it (`:241-251`). Self-healing but unbounded in the worst case, on a volume shared with ephemeral workspaces by default (`config.rs:707-712`).

##### D-GIT-15 — Medium: four subprocess sites have no timeout, and five inherit the relay's stdin

No timeout: `for-each-ref` (`cas_publish.rs:284-296`), `symbolic-ref` (`:337-345`), `index-pack` for the idx sidecar (`:409-415`), and all four `hydrate::run_git` arg sets (`hydrate.rs:451-470` — see D-GIT-04). No `stdin` configured (so the child inherits the relay's): `transport.rs:645-653`, `cas_publish.rs:284`, `:337`, `:409`, `hydrate.rs:452`.

##### D-GIT-16 — Low: kind:30618 bypasses the cross-pod publication path

Both emission sites call `fan_out_event_to_local_subscribers` directly (`transport.rs:1701-1710`, `handlers/side_effects.rs:2755-2761`) rather than `dispatch_persistent_event`. The comments justify the bypass only in terms of the access gate being a no-op for `channel_id = None`, and say nothing about Redis fan-out. Net effect: a subscriber on a different pod does not receive the ref-state event in real time. Given the design's own instruction that subscribers must "treat [30618] only as a signal to re-read `pointer(R)`" (doc §System Model), this is a UX gap rather than a correctness one — but it is undocumented.

##### D-GIT-17 — Low: kind:30618 silently omits refs and oids

`is_emittable_ref` drops everything outside `refs/heads/*` and `refs/tags/*` (`manifest_event.rs:117-127`), and malformed oids/names are skipped rather than failing (`:82-93`). A repo with `refs/notes/*`, `refs/stash`, or `refs/pull/*` publishes an incomplete ref set, and any consumer treating 30618 as authoritative (the web repo browser's branch list, `web/src/features/repos/use-repo-refs.ts:54-57`) sees a partial view with no signal that truncation occurred.

##### D-GIT-18 — Low: the conformance probe writes permanently to the production media bucket

`git_store` is built from the **media** S3 config (`state.rs:694-701`), and the probe writes immutable objects it deliberately never deletes ("immutable probe writes accumulate by design", `store.rs:864-866`): at default settings 3 `packs/<digest>` + 3 `probe/inm-race/<digest>` objects per boot, alongside user media. Only `probe/pointer-<uuid>` is cleaned up (`:618`, `:871`).

##### D-GIT-19 — Low: numeric config parses fail silently

Every `BUZZ_GIT_*` numeric var uses `.ok().and_then(parse)` with no warning on failure and no range check (`config.rs:713-738`, `main.rs:474-481`). `BUZZ_GIT_MAX_CONCURRENT_OPS=20x` silently yields 20. Contrast `BUZZ_PUSH_GATEWAY_TIMEOUT_MS`, which hard-errors with a range message (`config.rs:759-771`).

##### D-GIT-20 — Low: `.env.example` documents 6 of 11 vars

Missing: `BUZZ_GIT_MAX_REPOS_PER_PUBKEY`, `BUZZ_GIT_MAX_CONCURRENT_OPS`, `BUZZ_GIT_HOOK_HMAC_SECRET`, `BUZZ_GIT_CONFORMANCE_PROBE`, `BUZZ_GIT_PROBE_WRITERS`, `BUZZ_GIT_PROBE_ROUNDS` (`.env.example:71-79`). The Helm chart also never sets `BUZZ_GIT_MAX_REPO_BYTES`, so raising `maxPackBytes` silently multiplies both the repo bound (×2) and the cache bound (×10) while `packCacheVolumeSize` stays put.

##### D-GIT-21 — Low: dead and over-exported API

- `policy::generate_hook_hmac` (`policy.rs:419-437`) — **zero** production callers; used only at `:581`, `:626`, `:720`, all `#[cfg(test)]`. Either move it into the test module or `#[cfg(test)]`-gate it.
- `stream_git_read`'s `extra_args` parameter (`transport.rs:1418`) — always `&[]` (`:824`).
- Twelve `pub` items with no caller outside the module: `GitStore::{content_key, idx_key_for_pack_digest, get, get_limited}`, `MAX_MANIFEST_PACKS`, `PACK_COMPACTION_THRESHOLD`, `MAX_MANIFEST_REFS`, `is_safe_refname`, `is_hex_oid`, `is_pack_key`, `HydratedRepo::hydrated_packs`, `ProbeFailure`. `InfoRefsQuery`/`GitRepoParams` are `pub` with private fields, so they are unconstructible externally anyway.

##### D-GIT-22 — Low: duplicated code and triplicated test helpers

`read_log_prefix` (`transport.rs:1141-1155`) and `read_prefix` (`cas_publish.rs:877-891`) are the same function. `probe_enabled()` appears three times (`store.rs:996`, `hydrate.rs:579`, `cas_publish.rs:1578`); `tenant()` three times (`hydrate.rs:536`, `cas_publish.rs:1603`, `transport.rs:1902`); the `NamedTempFile::new_in` + `.reopen()` + `Stdio::from` idiom eight times. `harden_git_env` lives in the HTTP transport module (`transport.rs:302`) but is consumed by two storage layers.

##### D-GIT-23 — Low: env-gated live tests are indistinguishable from passing tests

Ten tests return early when `BUZZ_GIT_S3_PROBE != "1"` and report success. Only one prints a skip notice (`store.rs:1020-1022`). CI therefore shows 99/99 passing while the CAS, hydrate-roundtrip, empty-repo, and compaction paths were never executed. Prefer `#[ignore]` (which CI reports as skipped) or an explicit `eprintln!` in all ten.

##### D-GIT-24 — Low: named-reviewer attribution in production comments

"Max's blocker" (`transport.rs:530`), "Eva's call, on record in #proj-git-on-s3" (`:875-877`), "Sami #2 / Max / Dawn" (`cas_publish.rs:1171`), "Dawn's `canonical_bytes`" (`transport.rs:1670`). These encode decisions in people's names and a Slack channel rather than in the argument, and they will outlive both.

##### D-GIT-25 — Low: refname directory/file conflict bricks a repo permanently

`is_safe_refname` allows a manifest holding both `refs/heads/a` and `refs/heads/a/b`. Hydration writes `a` as a file first (BTreeMap order) and then `create_dir_all(.../a)` fails, so every subsequent read returns 500 (`hydrate.rs:351-364`). Unreachable via push (`git receive-pack` rejects D/F conflicts), but there is no repair path and no validation that would catch such a manifest before it is published.

##### D-GIT-26 — Low: `cas_publish_inner`'s compaction branch is the module's densest control flow

265 lines, five mutable locals, six early returns, and five `record_compaction` call sites that must each be kept in sync with a new error path (`cas_publish.rs:1021-1285`, plus `:961-979`). The `record_compaction` calls are copy-pasted into four separate error arms (`:1179-1189`, `:1195-1204`, `:1215-1225`, `:1252-1260`), which is exactly the shape where a new arm silently loses its metric.

---

#### 3. Recommended order of work

1. D-GIT-01, D-GIT-02 — authorization holes; both are small changes with large blast radius.
2. D-GIT-11 — introduce a `GitStore` trait. This unblocks the fence test the design doc still owes, plus tests for the lost-race path, D-GIT-05's invariant, and D-GIT-01's regression fence.
3. D-GIT-04, D-GIT-15 — bound the unbounded subprocesses.
4. D-GIT-03 — make the ref snapshot fail closed.
5. D-GIT-07 — add push-outcome and policy-decision metrics; the "do we need a local lock?" decision is currently unmeasurable.
6. D-GIT-05, D-GIT-09, D-GIT-10 — correct the comments and the doc's correspondence table while the reasoning is fresh.
7. D-GIT-06, D-GIT-08, D-GIT-14 — resource isolation and cache correctness.
8. Everything else as opportunistic cleanup; split `transport.rs` when next touching it (D-GIT-26 and the file-size row in §1).


## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: Debt

Scope: 12 files, 6,720 LOC, 110 tests, 1 `#[ignore]`d test, 0 `unsafe`, 0 production `unwrap()`, 7 production `.expect()`, 0 `TODO`/`FIXME`/`HACK` markers.

Severity: **CRITICAL** = security or data-integrity hole reachable today · **HIGH** = correctness/operability gap with real blast radius · **MEDIUM** = latent risk or meaningful maintenance drag · **LOW** = hygiene · **INFO** = observation.

---

#### Prioritized findings

##### D-01 · HIGH · Relay-admin commands (9030–9033) never re-check the durable ban

`ingest.rs:1639` exempts `is_relay_admin_kind` from the ban/timeout write gate on the stated grounds that relay-admin "retains its separate authorization policy" (`ingest.rs:1646-1648`). That policy is `relay_admin.rs:133-142`, which reads only `relay_members.role`. Grep for `restriction` / `banned` / `moderation_restriction_state` in `relay_admin.rs`: **zero hits**.

The moderation handler *does* re-check, with rationale that HTTP NIP-98 requests and already-authenticated sockets bypass the fresh NIP-42 challenge (`moderation_commands.rs:97-101`). That reasoning applies verbatim to 9030–9033 and is not implemented. A banned owner/admin with a surviving socket — or over NIP-98, which never touches the auth seam — can add members, remove members, change roles, and rewrite the workspace icon.

Fix: mirror `moderation_commands.rs:103-108` at the top of `handle_relay_admin_event`.

---

##### D-02 · HIGH · 12 of 14 privileged kinds produce no hash-chain audit entry

`buzz-audit` declares 11 actions (`buzz-audit/src/action.rs:5-29`); production emits exactly two — `EventCreated` (`handlers/event.rs:583`) and `MediaUploaded` (`api/media.rs:428`). None of the 12 assigned files references `buzz_audit`. Confirmed.

| Unaudited in the hash chain | Any durable trail? |
|---|---|
| 9040 ban, 9041 unban, 9042 timeout, 9043 untimeout, 9044 resolve | `moderation_actions` only — a plain, non-hash-chained table |
| **9030 add member, 9031 remove member, 9032 change role, 9033 set icon** | **none** — `tracing::info!` only (`relay_admin.rs:164`, `:203-209`, `:268-272`, `:327-332`) |
| **operator community provisioning / owner rotation** | **none** — `tracing::info!` only (`community_provisioning.rs:302-308`, `:336-343`) |
| 1984 report, 42000 feedback | own sidecar tables |
| 30350 push lease | event row, but ingest returns at `:2199` before the dispatch that audits |

Only 9035/9036 reach the chain, and only because they deliberately fall through to storage (`ingest.rs:1909-1912`).

The tamper-evident audit surface does not cover the relay's own privilege model. Membership and ownership changes — "who made X an admin", "who rotated community Y's owner" — exist only in ephemeral process logs.

Also: ARCHITECTURE.md:497 says "**10 audit actions**" and omits `MediaUploaded`; the enum has 11.

---

##### D-03 · MEDIUM · Community moderation has no self-action prevention

`moderation_commands.rs` never compares actor to target. An owner issuing 9040 against their own pubkey passes `decide_authority` (`moderation_authz.rs:149-150`), commits the ban (`:169`), then closes their own sockets cluster-wide (`:195-200`). Self-unban is blocked by the handler's own ban re-check (`:103-108`), so recovery needs a second owner/admin or direct DB access.

`relay_admin.rs` implements exactly this check for its own operations — `cannot remove yourself` (`:231-233`), `cannot change your own role` (`:290-292`). The moderation path does not. An admin is protected only incidentally, because the peer guard trips on their own `admin` role (`moderation_authz.rs:165-167`).

---

##### D-04 · MEDIUM · The "reject channel-scoped API tokens" contract for 9040–9044 is not enforced

Pinned at `moderation_commands.rs:50` as part of the routing contract. Ingest rejects channel-scoped tokens only for relay-admin kinds (`ingest.rs:1512-1516`) and leave requests (`:1520-1523`); the generic global-event token gate at `ingest.rs:1721-1724` is **unreachable** for 9040–9044 because the moderation dispatch returns at `:1582-1586`. Combined with `Scope::MessagesWrite` (`ingest.rs:216`), a legacy channel-scoped WS API token held by a community admin can issue a community-wide ban.

---

##### D-05 · MEDIUM · `push_runtime` has 2 tests for 656 LOC and zero metrics

Coverage: `gift_wrap_match_requires_self_p_filter_and_recipient` (`:580-620`) and `gateway_retries_send_the_same_request_id_over_http` (`:626-655`). Untested: `deliver_one`'s 10-branch response state machine (`:349-505`), `retry_or_fail`'s exponential backoff (`:531-550`), `process_match_batch`'s completed/retry/pending partitioning (`:125-214`), `load_match_context` (`:98-123`), and `match_job`'s class/suppress/ignore logic (`:216-290`).

Observability is equally thin: **zero** `metrics::` calls in the file. No wake-enqueued counter, no delivery-outcome counter by HTTP status, no gateway-latency histogram, no matcher-backlog gauge. Push delivery health is observable only through `warn!`/`error!` lines. Compare `storage_sweep.rs`, which emits 18 gauges for a strictly less critical subsystem.

---

##### D-06 · MEDIUM · Every DB call in the push delivery path discards its result

8 sites use `let _ = state.db.…` : `push_runtime.rs:361-364`, `:380-383`, `:410-413`, `:436-440`, `:442-446`, `:454-462`, `:469-472`, `:492-495`, `:533-536`, `:539-546`. A failed `complete_push_wake` leaves the row claimed; recovery depends on the 30 s claim lease expiring and the wake being re-claimed. That is safe (the stable request id makes redelivery idempotent) but completely invisible — combined with D-05 there is no way to detect a persistently failing completion path.

---

##### D-07 · MEDIUM · The delivery worker dies permanently if the HTTP client fails to build

`reqwest::Client::builder().timeout(…).build().expect("push HTTP client")` at `push_runtime.rs:313-316`, inside a `tokio::spawn`ed loop (`main.rs:688-690`). A panic here kills the worker while the matcher keeps enqueuing wakes — an outbox with no consumer, no restart, and (per D-05) no metric to notice.

---

##### D-08 · MEDIUM · The push wire contract is duplicated, not shared

`DeliveryRequest` / `DeliveryResponse` (`push_runtime.rs:31-51`) and the gateway's own model (`buzz-push-gateway/src/model.rs`, `grant.rs:162`, `http.rs:151`) are independent definitions with no shared crate and no cross-artifact test. Contrast the excellent `include_str!` migration test that pins `PUSH_KINDS` to the SQL trigger (`push_lease.rs:696-710`) — that pattern is applied once and not here.

---

##### D-09 · MEDIUM · Push lease DB errors are undiagnosable

`push_lease.rs:571-572`: `.map_err(|_| AcceptError::Internal("lease persistence failed"))`. The underlying `DbError` — constraint name, deadlock, connection failure — is discarded entirely and never logged (the file has zero `tracing` calls). Operators see a 500 with a fixed string and nothing else.

---

##### D-10 · MEDIUM · `LeasePlaintext` models mandatory fields as `Option`, forcing 5 production `.expect()`

`push_lease.rs:32-41` uses `Option<T>` for `app_profile`/`transport`/`endpoint`/`subscriptions`, which are mandatory when `active == true`. `accept` then re-unwraps them 5× with justification comments (`:534`, `:539`, `:543`, `:548`, `:552`). Five of the module's seven production `.expect()` calls come from this one modelling choice. An `enum LeasePlaintext { Active { … }, Inactive { … } }` would make them unrepresentable.

Note the workaround already in place: because `serde(deny_unknown_fields)` cannot express a conditional required-set, the code adds a *second*, hand-rolled key-presence check (`:159-190`) on top of serde. Two validation layers for one schema.

---

##### D-11 · MEDIUM · `push_runtime` runs on every pod with no leader election

`main.rs:686-692` spawns both loops unconditionally when the gateway URL is set (which is the default). `storage_sweep` is protected by a real Postgres advisory lock (`main.rs:1414-1421`); `push_runtime` is not. Correctness rests entirely on DB claim/fence tokens, which is sound, but the cost is N× claim-scan load at N pods. The delivery worker additionally iterates **every** community from `usage_community_hosts()` on every sweep (`push_runtime.rs:320-338`) with no bounding — that scan grows linearly with tenant count per pod per 500 ms at minimum.

---

##### D-12 · MEDIUM · Dead capability surface in `moderation_authz`

| Item | Production consumers |
|---|---|
| `ModerationAction::DeleteMessage` (`:32`) | **0** |
| `ModerationAction::Kick` (`:34`) | **0** |
| `ModerationAuthority::ChannelRole` (`:68`) | unreachable — all 6 call sites pass `channel_id: None` |
| `get_member_role` lookup (`:120-131`) | unreachable |
| `ModerationAuthority` return value | computed on every call, discarded by every caller |

The module doc says the seam is "the bridge `validate_admin_event` is missing today" (`:73-75`). The 9005-delete and 9001-kick paths still use `side_effects::validate_admin_event` (`ingest.rs:1929-1933`). The bridge was built and never connected — ~25% of the file (the channel-role branch plus its 2 tests) is production-dead.

The doc also claims the matched authority is "recorded in the audit row" (`:61`). `insert_audit` takes no authority parameter (`moderation_commands.rs:518-527`). **Documentation delta.**

---

##### D-13 · MEDIUM · 5 of 9 `moderation_actions` columns are always NULL

The single production writer hard-codes `channel_id: None`, `reason_code: None`, `private_reason: None`, `matched_principal: None` (`moderation_commands.rs:536-540`). `matched_principal` is documented as recording the NIP-OA principal an enforcement check matched (`buzz-db/src/moderation.rs:139`) and is surfaced verbatim by the mod-audit read API (`api/bridge.rs:2184`) — so that API field is **always `null`**.

Additionally, `MODERATION_ACTION_CHECK_VOCAB` includes `"delete_message"` and `"kick"` (`buzz-db/src/moderation.rs:105-106`) with zero production writers, and `resolution_audit_action` has an unreachable `"resolve:unknown"` fallback (`moderation_commands.rs:511`) that is **not** in the CHECK vocabulary — if reached it would raise a constraint violation.

---

##### D-14 · MEDIUM · No atomicity across the moderation write set

`handle_ban` performs restriction read (`:105`), ban write (`:169`), audit write (`:180`), disconnect (`:195`), notice (`:204`) with **no enclosing transaction**. An audit-write failure returns `error: failed to write audit row: …` (`:544`) for a ban that is already durable — the client sees failure for an action that took effect. Same shape for timeout (`:287` → `:298`).

`handle_resolve` inverts the order deliberately (audit first at `:453`, resolve at `:461`) and documents the residual race as tolerated (`:419-425`) — but that also means a failed resolve leaves an orphan audit row.

---

##### D-15 · MEDIUM · Unbounded report flooding, with no note size cap

Any authenticated principal may submit arbitrarily many distinct signed 1984 events. Idempotency is per event id (`buzz-db/src/moderation.rs:39`), so fresh signatures always land. There is no per-pubkey rate limit (ARCHITECTURE.md:821 confirms the only repo-wide `RateLimiter` impl is a test stub), no per-target cap, no `(reporter, target)` dedup, and the note is `event.content` stored **uncapped** (`report.rs:222-228`).

Contrast `product_feedback.rs:12-13`, which caps body at 32 KiB and tags at 64 KiB. Same threat model, opposite treatment. The mod queue read caps at 500 rows (`api/bridge.rs:2053`), so a flood degrades moderator usability.

---

##### D-16 · MEDIUM · Legacy provisioning mode is a community-takeover capability

Without `create_only`, an operator-signed request runs `bootstrap_owner` on an **existing** community, demoting the previous owner to admin (`community_provisioning.rs:321-334`). Documented and deliberate (`:236-247`), but the mitigation — "clients provisioning on behalf of end users must use create_only" (`:317-320`) — is prose, not code. Combined with D-02 (no audit trail) an owner rotation is both possible and untraceable.

---

##### D-17 · MEDIUM · `add_reaction` workflow action targets an unregistered route

`buzz-workflow/src/executor.rs:886-888` POSTs to `{BUZZ_RELAY_BASE_URL}/api/messages/{message_id}/reactions`. Verified: `router.rs` registers **zero** `reactions` routes and zero `/api/messages` routes. Every attempt returns `WorkflowError::WebhookError("AddReaction: relay returned 404 …")` (`:903-908`). Without the `reqwest` feature it silently reports success-with-skip (`:597-606`).

ARCHITECTURE.md:541 lists `add_reaction` as a working action; §9 (`:826-827`) lists only `send_dm`/`set_channel_topic` and approval gates. **`add_reaction` is an undocumented third broken action.**

---

##### D-18 · MEDIUM · `ActionSink` has one method, so 3 of 7 workflow actions are structurally unimplementable

`buzz-workflow/src/action_sink.rs:44-64` declares exactly `send_message`; `RelayActionSink` implements exactly that (`workflow_sink.rs:172-179`). `send_dm` and `set_channel_topic` return `NotImplemented` (`executor.rs:578`, `:583`) and cannot be fixed without widening the trait. `add_reaction` bypasses the sink entirely via HTTP (D-17) rather than using it.

All 6 `ActionSinkError` variants also collapse into a single `WorkflowError::WebhookError` (`action_sink.rs:31-34`), so channel-not-found, archived-channel, access-denied, and a DB outage are indistinguishable in run output.

---

##### D-19 · MEDIUM · The `identity_archive` integration test passes green without Postgres

`identity_archive.rs:515-527` is a `#[tokio::test]` with three silent `return` bailouts: `test_pool()` → `None` (`:517-519`), a probe `SELECT` failing (`:520-526`), `test_state()` → `None` (`:527-529`). `test_state` itself is `Option`-returning with `.ok()?` chains (`:442-472`).

This is the module's **only** coverage of the live-kind:0 revocation rule (BR-MOD-87) — arguably its most security-relevant behaviour — and it silently no-ops in any environment without Postgres. `workflow_sink.rs:613` handles the same situation correctly with `#[ignore = "requires Postgres"]`.

---

##### D-20 · MEDIUM · `BUZZ_PUSH_GATEWAY_DELIVERY_URL` defaults to a hard-coded third-party host

Unset ⇒ `Some("https://push.buzz.xyz/v1/deliveries/apns")` (`config.rs:339`, `:755-758`), which enables lease acceptance (`push_lease.rs:480-482`), spawns both workers (`main.rs:686-692`), advertises push in NIP-11 (`nip11.rs:208`), and attempts outbound HTTPS carrying the device token. Disabling requires an explicitly **empty** value (`config.rs:753`) — unsetting does the opposite of the intuitive thing. A self-hosted relay that never configures push still does all of the above.

---

##### D-21 · LOW · Push endpoint ownership is first-claim-wins with no proof of possession

The relay accepts any 1..=4096-byte `endpoint` string, hashes it, and lets a DB unique constraint arbitrate (`push_lease.rs:533-535`, `:563-570`, outcome `EndpointAlreadyLeased` at `ingest.rs:2196-2200`). Registering a token you learned denies push to the real device. The wake body is content-free (`push_runtime.rs:31-37`), so the leak is timing/existence only, scoped by the lease author's own read authorization.

Related: `ActiveLease.endpoint_grant` is documented as "opaque endpoint grant issued by the stateless gateway" (`buzz-db/src/push.rs:94`) but stores the client-supplied token verbatim (`push_lease.rs:544`, `:555`). Misleading name for a credential-adjacent value persisted in plaintext.

---

##### D-22 · LOW · Redirects are not disabled on the push gateway client

`push_runtime.rs:313-316` sets only `.timeout(…)`; `redirect::Policy` defaults to following up to 10 hops. A compromised or misconfigured gateway host could redirect the POST — including the NIP-98 `Authorization` header and the device token — elsewhere. `buzz-workflow`'s `call_webhook` explicitly disables redirects (ARCHITECTURE.md:539); this path does not.

---

##### D-23 · LOW · `moderation_notices` idempotency is not concurrency-safe

`notice_already_sent` is query-then-insert (`moderation_notices.rs:227-252`); the comment states two simultaneous deliveries for the same source can both miss the pre-query and names hard per-source serialization as a follow-up (`:132-138`). The scan limit is hard-coded to 1000 with the reasoning that 1000 notices to one user is a practical ceiling (`:222-226`) — beyond it, crash-retry can duplicate.

---

##### D-24 · LOW · Notice-DM failures are logged at `info!`, not `warn!`

`moderation_commands.rs:217`, `:319`, `:493`. A user who was banned and never told is an `info`; a failed NIP-43 announcement is a `warn` (`relay_admin.rs:215`). Inverted severity for the more user-visible failure.

---

##### D-25 · LOW · Unban and untimeout send no notice at all

9041 (`moderation_commands.rs:227-261`) and 9043 (`:330-364`) have no `send_moderation_notice` call. A user learns they were restricted but never that the restriction was lifted. `ModerationNotice::Restriction`'s doc also promises "with expiry rendered into the message" (`moderation_notices.rs:66`) — the body renders only the verb and reason (`:296-305`), never the expiry.

---

##### D-26 · LOW · `ModerationNotice::ContentActioned` has zero production constructors

Declared (`moderation_notices.rs:52-57`), rendered (`:288-292`), tested (`:388-397`), documented as the "actioned-author" notice for the delete path (`:25-26`) — and never constructed outside tests. The 9005 delete path does not call `send_moderation_notice`.

---

##### D-27 · LOW · `escalated` report status has no producer

9044 always stores `resolved` or `dismissed` (`moderation_commands.rs:380-385`), yet `ReportRecord.status` advertises `escalated` (`buzz-db/src/moderation.rs:71`) and `ModerationNotice::body` has an unreachable `"escalated"` arm (`moderation_notices.rs:281`). An `action=escalate` resolution stores `status=resolved`.

Related: the module doc's summary table implies the relay fans `delete`/`kick`/`ban` resolutions out through 9005/9001/9040 (`moderation_commands.rs:20`). `handle_resolve` performs no fan-out (`:453-499`); the body text does say the *client* composes the paired command (`:48-50`), but nothing guarantees the enforcement ever happens.

---

##### D-28 · LOW · The `urgent` push class is dead

`URGENT_KINDS = &[]` (`push_lease.rs:16`) and `"urgent"` absent from `supported_classes` (`:509`) means `class not supported` (`:246`) fires before the urgent-kind confinement check at `:281-283` can run. `class_rank` still ranks it in both copies (`:582`, `push_runtime.rs:574`); NIP-11 advertises `urgent_kinds: []` (`nip11.rs:209`).

---

##### D-29 · LOW · 8 public items in `push_lease.rs` have no external consumer

`validate_envelope` (`:83`), `parse_plaintext` (`:149`), `validate_plaintext` (`:194`), `LeaseEnvelope` (`:24`), `LeasePlaintext` (`:32`), `LeaseLimits` (`:65`), `AppProfile` (`:60`), `MAX_SAFE_JSON_INTEGER` (`:21`). The module presents itself as a reusable validation library (`:1-6`) but only `accept` is called. Also dead-public: `REPORT_TYPES` (`report.rs:29`); over-scoped: `StorageEmittedKey` is `pub(crate)` but used only inside its own file (`storage_sweep.rs:121`).

---

##### D-30 · LOW · Two competing definitions of `KIND_PUSH_LEASE`

`buzz-core/src/kind.rs:109` (`30350`) and `handlers/push_lease.rs:19` (`30_350`). Ingest imports the `push_lease` copy (`ingest.rs:204`, `:450`, `:2156`) and so does `side_effects.rs:2004`, `:2130`; `req.rs:1734-1755` imports the `buzz-core` one. AGENTS.md requires all kind integers live in `buzz-core/src/kind.rs`. Two sources of truth for one wire constant.

---

##### D-31 · LOW · Three competing error-prefix strategies across six handlers

| Strategy | Handlers | Effect |
|---|---|---|
| self-prefixing | `moderation_commands` (helpers `:548-558`), `report`, `product_feedback` (inline literals) | correct |
| unprefixed, ingest wraps `invalid: {e}` | `relay_admin` (`ingest.rs:1837`), `identity_archive` (`ingest.rs:1943`) | authorization failures surface as `invalid: actor not authorized: …` — semantically wrong |
| typed error enum | `push_lease` → `AcceptError` (`ingest.rs:187-195`) | correct 400/500 split |

Only `moderation_commands` has a test pinning its prefixes (`:669-680`). `report.rs`/`product_feedback.rs` inline the same prefixes at 20+ sites with no helper.

---

##### D-32 · LOW · Duplicated helpers across the handler set

| Duplicate | Copies |
|---|---|
| `extract_tag_value` (identical body) | 3 — `moderation_commands.rs:608`, `relay_admin.rs:49`, `identity_archive.rs:189` |
| p-tag extraction with 3 different contracts | `moderation_commands.rs:561` (bytes, first-valid), `relay_admin.rs:33` (hex, first-valid), `identity_archive.rs:170` (hex, **exactly-one**) |
| 64-hex validation | 4 inline + 2 typed (`report.rs:211`, `push_lease.rs:365` — the latter alone rejects uppercase) |
| ±120 s freshness (3 sites) + `ALLOWED_SKEW` | `moderation_commands.rs:81`, `relay_admin.rs:125`, `identity_archive.rs:147`, `push_lease.rs:476` |
| Freshness error string, verbatim | `moderation_commands.rs:117-120`, `relay_admin.rs:126-129`, `identity_archive.rs:148-151` |
| `blocked: you are banned from this community`, verbatim | `moderation_commands.rs:139`, `:199`, `ingest.rs:1648`, `handlers/auth.rs:171` |
| `class_rank` | `push_lease.rs:575`, `push_runtime.rs:567` |
| DB-state test-harness construction (~30 LOC each) | `identity_archive.rs:442-472`, `workflow_sink.rs:574-599` |

---

##### D-33 · LOW · Six handler signatures, three argument orders, three naming schemes

`(tenant, state, event)` for `moderation_commands`/`relay_admin`/`identity_archive`; `(tenant, event, state)` for `report`/`product_feedback`; `(tenant, state, event, now)` for `push_lease`. Names: `handle_*_event` ×4, `handle_*_command` ×1, bare `handle` ×1. Only `push_lease` injects a clock, so the other five freshness checks are untestable without wall-clock manipulation.

---

##### D-34 · LOW · Pubkey representation split across the same request

`relay_members` and `archived_identities` key on 64-hex `String` (`relay_members.rs:41`, `archived_identities.rs:34`); `community_bans`, `moderation_actions`, `moderation_reports` use raw `Vec<u8>` (`moderation.rs:85`, `:122`). A single 9040 hex-encodes the actor for `get_relay_member` (`moderation_authz.rs:98`) and passes the same actor as bytes to `insert_moderation_action` (`moderation_commands.rs:532`). Role is an untyped `String` compared against literals at 10+ sites with no enum, and `relay_admin.rs:142` collapses "no member row" to `""`.

---

##### D-35 · LOW · Best-effort side effects can silently desync durable state from the event view

| Path | Failure handling |
|---|---|
| NIP-43 member add/remove/list (5 calls) | `warn!` and continue (`relay_admin.rs:214-220`, `:274-279`, `:334-336`) |
| NIP-IA delta + archival list | `warn!` and continue (`identity_archive.rs:130-136`) |
| Provisioning membership snapshot | `warn!` and continue (`community_provisioning.rs:218-227`) |
| Moderation kind:0 profile | `warn!` and continue (`moderation_notices.rs:152-154`) |
| Workflow `dispatch_persistent_event` | result discarded with `let _ =` (`workflow_sink.rs:351`) |
| Ban disconnect Redis publish | `tokio::spawn`ed, `warn!` on failure (`state.rs:1043-1047`) |

Each is individually reasoned, but the aggregate is that `relay_members` / `archived_identities` / `communities` can diverge from their event-backed views with only a log line, and no reconciliation job for the membership/archive views.

---

##### D-36 · LOW · A missed ban-disconnect publish leaves read access open indefinitely

The ingest gate (`ingest.rs:1613-1650`) covers writes; nothing gates an established subscription's reads. On a pod that missed the Redis command, a banned user's socket keeps receiving fan-out until reconnect. `disconnect_pubkey_clusterwide` returns the local close count (`state.rs:1049`) and the caller discards it (`moderation_commands.rs:195`), so zero-sockets-closed is not even observable.

---

##### D-37 · LOW · An owner timeout does not cascade to that owner's NIP-OA agents

Bans cascade at the auth seam (`handlers/auth.rs:134-155`); the ingest write gate checks the authoring pubkey only (`ingest.rs:1598-1611`). Because timeout has no auth-seam presence, timing out a human leaves every agent they own free to post. Documented as a deliberate Phase-1 asymmetry with the follow-up named as a restriction-state cache (`ingest.rs:1607-1611`).

---

##### D-38 · LOW · Storage-sweep config silently absorbs invalid values and is read once, late

The four `BUZZ_STORAGE_*` vars use `.ok().and_then(parse).unwrap_or(default)` (`storage_sweep.rs:56-73`) — a typo is invisible. Every `Config::from_env` field by contrast hard-fails at boot (`config.rs:747-751`, `:764-771`). They are also read only on the **first leader tick** behind a function-local `OnceLock` (`main.rs:1447-1453`), so a non-leader pod never validates them.

`TIMEOUT_SECS` and `MAX_OBJECTS` have no floor: `TIMEOUT_SECS=0` yields a sweep that times out immediately and respawns every usage tick forever, because `should_spawn` returns `true` unconditionally on `!ok` (`storage_sweep.rs:110`). Only `INTERVAL_SECS` is floored (`.max(60)`, `:59-60`).

`BUZZ_STORAGE_METRICS` uses bespoke `off`-only parsing rather than the crate's `parse_bool` (`config.rs:364-378`), so `false`/`0`/`disabled` all **enable** the feature (pinned by a test that documents the asymmetry, `storage_sweep.rs:379-397`).

---

##### D-39 · LOW · Storage sweep requires `s3:ListBucket` on the read-write media credential, whole-bucket

The sweep shares `MediaStorage` and therefore `BUZZ_S3_ACCESS_KEY`/`BUZZ_S3_SECRET_KEY` (`config.rs:622-625`), calling `list_page` with an **empty prefix** (`buzz-media/src/storage.rs:249-256`). No list-only credential, no per-tenant prefix. Enabling metrics widens the blast radius of a relay compromise to a full cross-tenant object inventory. The kill switch is the only mitigation offered (`storage_sweep.rs:42-45`).

---

##### D-40 · LOW · S3 outage is reported to reporters as "blob not found"

`report.rs:66-70` maps every `get_sidecar` failure — including transient storage errors — to `invalid: report target blob not found`. Documented as a known Phase-1 limitation (`:66-69`), but a MinIO/S3 outage tells reporters their evidence does not exist.

---

##### D-41 · LOW · DB error text reaches clients on multiple paths

`error: database error: {e}` (`moderation_commands.rs:174` + 5), `database error: {e}` (`relay_admin.rs:137` + 6), and `restricted: {e}` wrapping a possible `sqlx` error from the authorization seam (`moderation_commands.rs:549` + `moderation_authz.rs:99`). The HTTP moderation read path does this correctly, normalizing every denial to a fixed 403 (`api/bridge.rs:2062-2067`); the event paths do not. `identity_archive.rs:324` similarly surfaces `buzz-sdk` internal error text.

---

##### D-42 · LOW · Workflow messages can never be threaded

`workflow_sink.rs:325-335` hard-codes `depth: 0`, `parent_event_id: None`, `root_event_id: None`, documented as "Workflow messages are always top-level" (`:322`). A workflow triggered by a message cannot reply in that message's thread. `event_created_at` also silently falls back to `Utc::now()` on an out-of-range timestamp (`:311-314`), where `product_feedback.rs:47-49` rejects the same condition.

---

##### D-43 · LOW · Moderation success logs omit the actor

`moderation_commands.rs:223`, `:258`, `:325`, `:362` log `target` but not `actor`. `relay_admin.rs:203-209` logs both `sender` and `target`. Combined with D-02 (no hash-chain entry), a ban is not attributable from logs alone — only from `moderation_actions`.

---

##### D-44 · INFO · Unused validation fields kept for shape-checking

`ParsedReportTarget::{Event,Blob}.author_pubkey` is parsed and annotated "Validation-shape only" (`report.rs:106-107`, `:112-113`) and discarded with `..` at `:54`, `:65`. Deliberate, documented, but a reader must trace two hops to learn the reported `p` tag is not authority for authorship.

---

##### D-45 · INFO · `storage_sweep.rs` is 1090 LOC with 730 of them tests

Production code is `:1-357` (357 LOC); tests are `:359-1090` (731 LOC), a 2:1 test-to-code ratio — the healthiest in the group and the right model for the rest of it. Note also that `emit_storage_metrics` publishes only gauges (even sweep duration, `:300`), so cross-sweep percentile analysis is impossible.

---

#### Debt summary by file

| File | LOC | Tests | Findings | Weight |
|---|---|---|---|---|
| `push_runtime.rs` | 656 | 2 | D-05, D-06, D-07, D-08, D-11, D-22, D-28, D-30, D-32 | **heaviest** — lowest coverage + zero metrics on the most operationally exposed loop |
| `handlers/relay_admin.rs` | 468 | 15 | D-01, D-02, D-31, D-32, D-33, D-35, D-43 | **highest severity** — D-01 + D-02 |
| `handlers/moderation_commands.rs` | 768 | 10 | D-02, D-03, D-04, D-13, D-14, D-24, D-25, D-27, D-31, D-32, D-33, D-36, D-41, D-43 | most findings, but most are policy gaps, not code defects |
| `handlers/push_lease.rs` | 771 | 10 | D-09, D-10, D-21, D-28, D-29, D-30 | typed-modelling debt |
| `handlers/moderation_authz.rs` | 335 | 7 | D-12, D-41 | ~25% production-dead |
| `handlers/identity_archive.rs` | 580 | 6 | D-19, D-32, D-33, D-35, D-41 | D-19 is the important one |
| `workflow_sink.rs` | 711 | 18 | D-17, D-18, D-35, D-42 | gaps are in `buzz-workflow`, not the sink |
| `handlers/report.rs` | 337 | 6 | D-15, D-31, D-33, D-40, D-44 | D-15 is the important one |
| `handlers/moderation_notices.rs` | 398 | 4 | D-23, D-25, D-26, D-35 | low coverage for a user-facing path |
| `handlers/community_provisioning.rs` | 445 | 13 | D-02, D-16, D-35 | best-validated file; the gap is policy |
| `storage_sweep.rs` | 1090 | 15 | D-38, D-39, D-45 | exemplary engineering; config hygiene only |
| `handlers/product_feedback.rs` | 161 | 4 | D-31 | cleanest file in the group |

#### Verified counts

| Metric | Value |
|---|---|
| Inbound event kinds owned | 14 (1984, 9030–9033, 9035, 9036, 9040–9044, 30350, 42000) |
| Tests (`#[test]` + `#[tokio::test]`) | 110 |
| `#[ignore]`d tests | 1 (`workflow_sink.rs:613`) |
| Tests that silently pass without Postgres | 1 (`identity_archive.rs:515`) + 1 env-gated skip (`storage_sweep.rs:381`) |
| `unsafe` | 0 |
| `unwrap()` outside `#[cfg(test)]` | 0 |
| `.expect()` outside `#[cfg(test)]` | 7 — `push_lease.rs:534/539/543/548/552`, `push_runtime.rs:316/514` |
| `panic!` outside `#[cfg(test)]` | 0 |
| `todo!` / `unimplemented!` | 0 |
| `TODO` / `FIXME` / `HACK` / `XXX` markers | 0 (the 2 `TODO (WF-07)` live in `buzz-workflow/src/executor.rs:577`, `:582`) |
| `#[allow(…)]` attributes | 0 |
| Env vars read directly | 4 (all `BUZZ_STORAGE_*` in `storage_sweep.rs`) |
| `SPROUT_*` vars read | 0 |
| Privileged mutations with no hash-chain entry | 12 of 14 kinds |
| Privileged mutations with no durable audit trail at all | 5 (9030, 9031, 9032, 9033, operator provisioning) |
| Public items with zero external callers | 10 (8 in `push_lease.rs`, `REPORT_TYPES`, `StorageEmittedKey`) |
| Declared-but-unconstructed enum variants | 3 (`DeleteMessage`, `Kick`, `ContentActioned`) |
| Workflow actions implemented by `workflow_sink` | 1 of 7 |


## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Debt

---

#### 0. Baseline metrics

| Metric | Value | Source |
|---|---|---|
| Files | 12 | assignment |
| Total lines | 9,266 | `wc -l` |
| Production lines (before each file's `#[cfg(test)]`) | 5,565 | computed |
| Test lines | 3,701 (**40 %**) | computed |
| Tests | **82** | see §6 |
| `#[ignore]`d tests | **0** | grep |
| Tests that silently no-op without Redis | **11** | §6.2 |
| `unsafe` blocks | **0** | grep |
| `unwrap()` outside `#[cfg(test)]` | **0** | §5 |
| `expect(` outside `#[cfg(test)]` | **10** | §5 |
| `panic!`/`unreachable!` outside `#[cfg(test)]` | **1** | §5 |
| TODO/FIXME/XXX/HACK markers | **0** | grep |
| `tokio::spawn` sites | 22 total; **11 production** | §2.3 |
| `metrics::` calls | **1** (`mesh_fence_rejections_total`, `directory.rs:483`) | grep |
| Public items with zero production callers | **13** | §3 |
| Duplicated type names inside the crate | **5** | §4 |

---

#### 1. `audio/join.rs` and `audio/handler.rs` — complexity

##### 1.1 File sizes

| File | Total | Production | Test | Test share |
|---|---|---|---|---|
| `audio/join.rs` | 3,036 | 1,806 | 1,230 | 41 % |
| `audio/handler.rs` | 1,430 | 1,337 | 93 | 7 % |
| `tunnel/reliable.rs` | 950 | 658 | 292 | 31 % |
| `tunnel/directory.rs` | 922 | 576 | 346 | 38 % |
| `audio/room.rs` | 790 | 556 | 234 | 30 % |
| `mesh_boot.rs` | 750 | 521 | 229 | 31 % |
| `conformance/mod.rs` | 727 | 430 | 297 | 41 % |
| `audio/mesh.rs` | 393 | 284 | 109 | 28 % |
| `audio/wire.rs` | 168 | 88 | 80 | 48 % |
| `conformance/tracers.rs` | 73 | 73 | **0** | 0 % |
| `audio/mod.rs` | 19 | 19 | 0 | — |
| `tunnel/mod.rs` | 8 | 8 | 0 | — |

`join.rs` at 3,036 lines is the largest file, but **41 % of it is tests** — 1,806
production lines across 48 functions is dense, not bloated. The real complexity
problem is `handler.rs`, which is 93 % production code with 7 % tests.

##### 1.2 Longest functions (production code only)

| Function | Lines | Location | `if`/`match`/`while`/`for` | match arms |
|---|---|---|---|---|
| **`handle_active_audio_connection`** | **719** | `handler.rs:167-885` | **49** | **36** |
| `serve_control_loop` | 256 | `join.rs:1115-1370` | 25 | 30 |
| `boot_mesh` | 110 | `mesh_boot.rs:411-520` | 5 | 0 |
| `recv_loop` | 107 | `handler.rs:947-1053` | 12 | 12 |
| `emit_participant_event` | 100 | `handler.rs:1237-1336` | 6 | 9 |
| `validate_fenced_header` | 92 | `directory.rs:348-439` | 5 | 0 |
| `read_owner_control` | 86 | `join.rs:1527-1612` | 10 | 13 |
| `ensure_membership` | 83 | `handler.rs:1153-1235` | 8 | 0 |
| `spawn_lease_renewer_with_interval` | 78 | `reliable.rs:580-657` | 3 | 8 |
| `wire_mesh_consumers` | 76 | `mesh_boot.rs:224-299` | 4 | 2 |
| `spawn_huddle_renewer_with_interval` | 73 | `join.rs:490-562` | 3 | 8 |
| `attach_signals` | 60 | `join.rs:665-724` | 6 | 0 |
| `dial_remote_owner` | 59 | `join.rs:1666-1724` | 2 | 6 |
| `resolve_join` | 57 | `join.rs:323-379` | 3 | 4 |
| `add_peer_at_index` | 54 | `room.rs:281-334` | 7 | 0 |

`handle_active_audio_connection` is **2.8× the next-longest function** and carries
49 branch points in one linear scope. It holds 12 mutable locals threaded across the
whole body (`pending_remote`, `acquired_lease`, `remote_session`, `remote_stream`,
`remote_fence`, `owner_lost`, `owner_draining`, `owner_generation`, plus four task
handles), and interleaves at least 9 distinct responsibilities:

1. challenge/auth handshake (`:176-238`)
2. relay + channel membership (`:244-286`)
3. mesh ownership resolution / availability gate (`:300-378`)
4. room acquisition + archived re-check (`:379-413`)
5. protocol-version admission (`:415-441`)
6. cross-pod dial + registration (`:443-503`)
7. local admission + owner-lease install (`:505-604`)
8. task spawning + supervision (`:657-800`)
9. teardown: peer removal, `left`, three lifecycle-event emits, archive, lease release (`:803-884`)

Each of stages 3–7 has its own early-return path that must independently remember to
`send_clean_close` the mesh stream and `cleanup_if_empty` the room — **7
`cleanup_if_empty` sites (`:401`, `:408`, `:484`, `:501`, `:637`, `:849`, `:865`, of
which 5 are on early-return paths) and 3 `send_clean_close` sites (`:520`, `:530`,
`:544`) in one function**, with no RAII guard for either obligation. `:389-403` and
`:404-410` are two arms of one `match` that both call `cleanup_if_empty`; `:637` is a
third for the `joined`-send-failure path. A new early return that forgets either is a
silent room leak or a leaked remote peer on the owner pod.

`serve_control_loop` (256 lines, 25 branches) is the second offender: a `select!`
over four arms inside a `loop` whose `break` value is a `Result`, with a
`teardown_reason` latch mutated from three places (`:1160`, `:1164`, `:1220`) and a
`stream_community` latch read from four (`:1176`, `:1266`, `:1291`, `:1346`).

---

#### 2. Prioritized findings

##### P1 — Correctness / security risk

| # | Severity | Finding | Evidence |
|---|---|---|---|
| D-01 | **High** | **Media datagrams are not Redis-fenced and never check `owner_runtime_id`.** `on_media_datagram` gates on a purely local monotone counter, then delivers. A mesh peer can inject audio into any room on any peer pod, attribute it to any `peer_index`, or poison the floor with one high-generation frame and silence the legitimate owner | `mesh.rs:204-250`, `:102-128`; `wire.rs:90-92` calls the owner field "advisory" |
| D-02 | **High** | **The media datagram envelope carries no community**, so the host-derived tenant boundary does not hold on the media path. `get_unambiguous_by_channel` fails closed only when two *active* communities share the channel UUID — with one active community per UUID (the normal case) a peer addresses that community's room without naming it. Acknowledged in-code as a known limitation | `room.rs:519-541`, `mesh.rs:36-42`, `:221-227` |
| D-03 | **High** (when enabled) | **`POST /_mesh/demo/echo` is unauthenticated and takes `community_id` from the request body** — the only route in the relay that does not derive the tenant from the host. Grants arbitrary lease creation and unbounded generation-counter inflation in any community's Redis key space, including on a channel UUID that is a live huddle's `session_id` | `api/mesh_demo.rs:50-95`; `router.rs:123` |
| D-04 | **Medium** | **The huddle path ignores `lease.profile`.** `HuddleDirectory::owner_of` discards it (`join.rs:110-116`), so a `reliable-stream` lease is honoured as huddle ownership. `reliable.rs:99-105` does check — the asymmetry is what makes D-03 reach huddles | `join.rs:107-127` vs `reliable.rs:99-105` |
| D-05 | **Medium** | **Audio shares the global `conn_semaphore` and holds a permit for the whole 5 s pre-auth window.** An unauthenticated client can exhaust the relay's entire WebSocket budget through `/huddle/…/audio`. No per-IP rate limit, no separate audio budget | `handler.rs:90-99`, `:190`; `state.rs:727` |
| D-06 | **Medium** | **`GenerationFloor.seen` is an unbounded `DashMap` with no TTL sweep.** Entries are added by any inbound datagram and removed only by explicit `forget`. Reachable from within the mesh at will | `mesh.rs:90`, `:131-133` |
| D-07 | **Medium** | **No frame-rate or bitrate limit.** Only a 4096-byte per-frame cap. One authenticated peer produces 24× amplification in-pod and 24× on the network cross-pod | `handler.rs:961-964`; `room.rs:398-411`; `mesh.rs:262-283` |
| D-08 | **Medium** | **`remove_peer` silently does nothing on a poisoned lock** (`let Ok(mut g) = … else { return }`), leaking the peer and its index. Six other sites in the same file handle poisoning six different ways, including one that recovers correctly | `room.rs:337-340`; cf. `:193`, `:202`, `:229`, `:282`, `:363`, `:462` |
| D-09 | **Medium** | **Post-admission lease blind window.** Local WS peers have no per-frame fence (stated in-code), so a >30 s partition can leave two pods fanning out locally for up to ~10 s until the next renew tick | `handler.rs:568-575`; `join.rs:452`; `directory.rs:17` |
| D-10 | **Medium** | **`audio_forward_loop` re-introduces a control drop point** that `room.rs`'s 32-slot queue exists to avoid: it `try_send`s into the 8-slot WS control channel, and unlike `broadcast_control` it does not warn. A `joined`/`left` lost here silently desyncs the client's index→pubkey map | `handler.rs:1104-1108` vs `room.rs:441-446` |

##### P2 — Maintainability

| # | Severity | Finding | Evidence |
|---|---|---|---|
| D-11 | **Medium** | **`handle_active_audio_connection` is 719 lines / 49 branches** with 12 threaded mutable locals, 5 `cleanup_if_empty` sites and 3 `send_clean_close` sites and no RAII guard for either. See §1.2 | `handler.rs:167-885` |
| D-12 | **Medium** | **`handler.rs` has 1,337 production lines and 2 tests**, both peripheral (semaphore budget, parser size cap). The entire join sequence, all 13 WS error codes, teardown ordering, and the 4-step lifecycle-event pipeline are untested | `handler.rs:1341-1358`, `:1417-1427` |
| D-13 | **Medium** | **Two `AdmissionError` types in one crate.** `audio/room.rs:83-95` (`Ended`/`Full`/`VersionMismatch`) and `admission.rs:12` (`Exceeded`/`Unavailable`) are unrelated types with the same name, both live, neither renamed on import. A reader at `handler.rs:513` must resolve the path to know which one | `room.rs:83`; `admission.rs:12` |
| D-14 | **Medium** | **Three parallel `Option`s that are always `Some` together** (`remote_session`, `remote_stream`, `remote_fence`) force **4 of the 6 production `expect`s** in the group. One `Option<struct{…}>` removes all four | `handler.rs:445-503`, `:689-702` |
| D-15 | **Medium** | **Duplicated roster types with no conversion coverage.** `RosterSnapshot`/`RosterDelta` exist in both `room.rs` and `join.rs` with hand-written conversions at `join.rs:1414-1433`. Nothing catches field drift | `room.rs:64-81`; `join.rs:895-945` |
| D-16 | **Low-Medium** | **Duplicated renewer implementation.** `spawn_huddle_renewer_with_interval` (`join.rs:494-562`) and `spawn_lease_renewer_with_interval` (`reliable.rs:580-657`) are structurally identical (73 vs 78 lines, same select, same loss contract, same release-error nuance) over two different lease newtypes. The huddle version's doc even says "A mirror of `crate::tunnel::reliable::spawn_observable_renewer`" (`join.rs:476-481`) | both |
| D-17 | **Low-Medium** | **Two `Ownership` types** with identical fields in sibling modules — `join.rs:186-192` (live) and `mesh.rs:74-80` (attached to a dead trait) | both |
| D-18 | **Low-Medium** | **`ensure_membership` returns `Result<Uuid, String>`.** Four distinct denial reasons (archived, unlinked parent, non-member, DB error) collapse into one client-visible `not a member`, and the caller cannot distinguish them for logging or metrics | `handler.rs:1153-1158`, `:274-285` |
| D-19 | **Low** | **`ReliableStreamError` is `#[allow(missing_docs)]`** — the only doc escape hatch in the group, against the `AGENTS.md` rule "New public API must have doc comments" | `reliable.rs:531` |
| D-20 | **Low** | **Double `get_channel` per join.** The same row is fetched at `handler.rs:1164` (inside `ensure_membership`) and again at `:389`, uncached. Deliberate for the race, but two Postgres round trips on every join | `handler.rs:389`, `:1164` |
| D-21 | **Low** | **`heartbeat_loop` does not set `MissedTickBehavior`**, unlike both renewers. A runtime stall can fire several ticks in a row and trip `MAX_MISSED_PONGS` spuriously | `handler.rs:1131` vs `join.rs:502`, `reliable.rs:591` |
| D-22 | **Low** | **`peer_pubkeys()` is unsorted while `roster_snapshot()` sorts by index**, so a same-pod client's `joined.peers` array is in `DashMap` order and a cross-pod client's is sorted — an avoidable client-visible inconsistency | `room.rs:471` vs `:479-484` |
| D-23 | **Low** | **`V2_HEADER_LEN`/`FLAG_DTX` are duplicated** between `audio/wire.rs:29,33` and `desktop/src-tauri/src/huddle/wire.rs:48`. One protocol constant, two copies, no shared crate | both |
| D-24 | **Low** | **`mesh_stack_smoke.rs:31` requires manual sync** with `buzz_lib::mesh_llm::MESH_WORKER_STACK_SIZE` in the desktop crate — a cross-crate constant coupled by comment | `examples/mesh_stack_smoke.rs:31` |

##### P3 — Observability

| # | Severity | Finding | Evidence |
|---|---|---|---|
| D-25 | **Medium** | **Zero metrics on the entire audio path.** No gauge for active rooms or peers, no counter for dropped frames, join failures, admission rejections by reason, version mismatches, heartbeat deaths, or owner-loss events. The group's only metric is `mesh_fence_rejections_total` | grep: 1 `metrics::` call in 9,266 lines, `directory.rs:483` |
| D-26 | **Medium** | **Audio frame drops are completely invisible.** Three `try_send` sites drop silently with no counter and no log (`room.rs:409`, `:427`, `handler.rs:1115`). A room at capacity degrading to unusable audio produces no signal at all | those three lines |
| D-27 | **Low** | **`seq` is written and never read.** Two independent per-leg counters (`join.rs:1508`/`:1759`, `mesh.rs:264`/`:270`) exist for "loss/reorder observability" (`wire.rs:113-115`) and no consumer computes loss or reorder from either | both |
| D-28 | **Low** | **`GET /_mesh` is unauthenticated** on a router documented as having no auth, and returns peer runtime ids, addresses and drain state | `router.rs:222-224`, `:230`, `:399-406` |

##### P4 — Dead code (zero production callers)

Verified by grep across `crates/**` and `desktop/src-tauri/**`.

| # | Item | Line | Status |
|---|---|---|---|
| D-29 | `HuddleOwnerDirectory` trait + `mesh::Ownership` | `mesh.rs:67-80` | **Zero implementors, zero callers, zero tests.** Superseded by `join.rs`'s `HuddleDirectory`. `mesh.rs:14` still points readers at it |
| D-30 | `read_teardown_cause` | `join.rs:1623-1642` | Zero production callers; 4 test callers (`join.rs:2948`, `:2981`, `:3007`) plus 5 dedicated tests (`:2955-3016`). Superseded by `read_owner_control` (`join.rs:1527`), the only one `handler.rs:707` uses. ~20 production + ~65 test lines |
| D-31 | `Room::mark_ended` | `room.rs:192-199` | Zero production callers; one test (`room.rs:660`). Production ends rooms via `remove_peer_and_check_ended` |
| D-32 | `PeerCtrl::Close` | `room.rs:35` | **Zero producers.** Handled at `handler.rs:1112` but nothing ever sends it — the graceful per-peer shutdown path it implies is unwired |
| D-33 | `SessionDirectory::takeover` | `directory.rs:233-242` | Zero callers anywhere. A documented "distinct operation" that is a verbatim delegate to `acquire` |
| D-34 | `SessionDirectory::known_generation` | `directory.rs:324-339` | Zero production callers; 2 test callers |
| D-35 | `ReliableStreamRouter::spawn_renewer` / `spawn_observable_renewer` | `reliable.rs:179`, `:192` | Zero callers. `mesh_boot.rs:206-215` documents that renewal "lands with the first product session consumer" — which has not landed |
| D-36 | `ReliableMeshStream::with_community` | `reliable.rs:263-266` | Zero callers |
| D-37 | `HuddleOwnerRegistry::attach` | `join.rs:659-661` | Zero production callers (thin wrapper over `attach_signals`); 5 test callers |
| D-38 | `MeshAudioRouter::{new, fence, local_runtime_id}` | `mesh.rs:169`, `:196`, `:201` | Zero production callers. Production uses `with_fence` (`mesh_boot.rs:239`) |
| D-39 | `MeshHandle.membership` | `mesh_boot.rs:141`, populated `:501` | **Written, never read.** `/_mesh` reads the private `runtime` field instead (`mesh_boot.rs:172-174`) |
| D-40 | `conformance::step` | `conformance/mod.rs:121-123` | Zero callers |
| D-41 | **`JsonlTracer`** | `tracers.rs:30-72` | **Zero callers in the entire workspace.** The only impl that would make the conformance seam observable. `state.rs:795-797` promises "test helpers in `crates/buzz-test-client` once those land" — they have not. 43 production lines, 0 tests |
| D-42 | `buzz_conformance::NoopTracer` | `buzz-conformance/src/lib.rs:323` | Zero users — the relay defines its own (`tracers.rs:20`) |
| D-43 | `TraceAction::ReadHostFeedRows` | `buzz-conformance/src/lib.rs:250` | **No emitter in the relay.** Only referenced by the conformance crate's own proptest (`tests/proptest_checker.rs:164`, `:255`) and its transition function |
| D-44 | `HuddleLeaseRenewer.task` / `ReliableLeaseRenewer.task` | `join.rs:467`, `reliable.rs:202` | Public `JoinHandle` fields, never awaited in production — the renewer is detached at `join.rs:694` |
| D-45 | `SessionStreamHandler` inbound reliable consumer | `mesh_boot.rs:299-303` | Accepts the stream, logs "no session consumer wired — closing", closes. Fence validation runs, then the work is discarded |

**Whole-lane dead weight:** the reliable-stream tunnel (`tunnel/reliable.rs`, 950
lines) has **no product consumer at all**. Its only join-side caller is the
demo-gated `POST /_mesh/demo/echo`, and its owner side either echoes (demo) or
closes. `reliable.rs:1` describes it as "for berd ↔ goose-server sessions"; no such
wiring exists in the repo.

##### P5 — Documentation drift

| # | Doc claim | Line | Reality |
|---|---|---|---|
| D-46 | "Operators running multiple relay pods MUST set `BUZZ_HUDDLE_AUDIO_AVAILABLE=false`" | `config.rs:120-128` | **Stale and misleading.** With `BUZZ_MESH=on` the flag is never read — `handler.rs:306-378` only consults it on the mesh-`None` arm. An operator following this doc on a mesh deployment sets a flag that does nothing |
| D-47 | "`actor` is the lower 16 bytes of `blake3(pubkey_bytes)` as a hex string" | `conformance/mod.rs:51-53` | `actor_label` takes the first 16 hex chars of the pubkey; no hashing (`:70-74`). The rationale 10 lines later (`:63-69`) explains the real design, contradicting the doc above it |
| D-48 | "req.rs / event.rs: (held back as additive patch for Eva to apply onto Max's req.rs writes)" | `conformance/mod.rs:46-48` | The `req.rs` wire points **have** landed: `req.rs:144`, `:355`, `:671` |
| D-49 | "Zero-cost tracer used in production builds… the build can have the compiler eliminate them entirely behind a feature flag" | `tracers.rs:11-13`, `state.rs:615` | No such feature flag exists (`Cargo.toml` `[features]` = `dev` only). Every emit site constructs its `TraceAction` (allocating `String`s, cloning `AbstractState`) and discards it |
| D-50 | "JSON control message (joined/left/**speakers**)" | `room.rs:33` | No `speakers` message is ever produced anywhere in the relay |
| D-51 | "Today a huddle's audio only fans out within a single pod… This module removes that wall" | `mesh.rs:1-10` | Present tense on both sides of a change that has shipped; the "today" is the pre-mesh state |
| D-52 | "The `HuddleControl` stream path… is wired in a following change" | `mesh.rs:150-153` | Already wired (`mesh_boot.rs:255-275`) |
| D-53 | "the client frame's encrypted content is NIP-44 between client endpoints, so server-side plaintext of the media itself never exists" | `buzz-relay-mesh/src/wire.rs:118-121` | **No such encryption exists.** `desktop/src-tauri/src/huddle/relay_api.rs:303` builds a plain v2 frame; the relay and every mesh peer hold plaintext Opus |
| D-54 | "older versions stay supported indefinitely for staged rollouts" | `handler.rs:120-122` | True for v1 acceptance, but there is no way to *pin* a deployment below `CURRENT_PROTOCOL_VERSION` — no env var, no config field |
| D-55 | "48100 — A huddle (audio/**video** session) was started" | `buzz-core/src/kind.rs:529-530` | Nothing in the relay handles video |
| D-56 | ARCHITECTURE.md: "**Room state:** an admission guard synchronizes joins…; soft cap 25 peers (hard cap 255 via `u8` peer index)" | `ARCHITECTURE.md:566` | Accurate, except the hard cap is **255 peers using indices 0..=254** — `alloc()` refuses at `next_fresh == 255`, so index 255 is never issued (`room.rs:146-152`) |
| D-57 | ARCHITECTURE.md § Huddle Audio (`:560-568`) makes no mention of the mesh, cross-pod ownership, the Redis session directory, or the `roster` control message | `ARCHITECTURE.md:560-568` | Describes only the single-pod design. The 4,400-line cross-pod subsystem in `join.rs`+`mesh.rs`+`directory.rs` is undocumented at architecture level |
| D-58 | Zero env vars from this group appear in `.env.example` or `deploy/compose/.env.example` | verified by grep | `BUZZ_HUDDLE_AUDIO_AVAILABLE`, `BUZZ_MESH`, `BUZZ_MESH_BIND_ADDR`, `BUZZ_MESH_DEMO_ECHO`, `BUZZ_MESH_ADVERTISE_ADDR`, `POD_IP` are discoverable only from source |

---

#### 3. Dead-code line-count summary

| Category | Approx. production lines |
|---|---|
| `HuddleOwnerDirectory` + `mesh::Ownership` (D-29) | 20 |
| `read_teardown_cause` + its 5 tests (D-30) | 20 prod / 65 test |
| `JsonlTracer` (D-41) | 43 |
| `takeover`, `known_generation`, `with_community`, `spawn_renewer`, `spawn_observable_renewer`, `attach`, `mark_ended`, `step`, `MeshAudioRouter::{new,fence,local_runtime_id}` (D-31…D-40) | ~90 |
| Reliable-stream lane with no product consumer (D-45 + whole-lane note) | 658 |
| **Total unreachable-from-production** | **~830 production lines (15 % of the group)** |

---

#### 4. `expect` / `unreachable` inventory (production paths)

| Site | Rationale quality | Fix |
|---|---|---|
| `handler.rs:451` `pending_remote.expect("RemoteOwner matched above")` | Fragile — the `if let` at `:448-450` already matched, then the value is re-taken | restructure the match to bind the payload once |
| `handler.rs:457` `unreachable!("matched RemoteOwner above")` | Same pattern | as above |
| `handler.rs:689`, `:692`, `:696`, `:701` (×4) | Invariant across three parallel `Option`s, not type-enforced | D-14: collapse into one `Option<struct>` |
| `handler.rs:731` `state.mesh().expect(...)` | Sound — guarded at `:727` | acceptable |
| `reliable.rs:471` `bytes[2..18].try_into().expect(...)` | Infallible given the `len < 18` check at `:462` | use a fixed-size array pattern |
| `directory.rs:261`, `:291` (×2) | Trust the Lua contract (`:47`, `:65`) | restructure the script return so the type carries it |
| `conformance/mod.rs:246` `row.expect(...)` | Sound but avoidable | a `match` removes it |

Against the `AGENTS.md` rule "Do not introduce new `unwrap()` or `expect()` in
production paths": compliant on `unwrap()`, non-compliant on `expect()` in four
files. Five of the ten are removable by one refactor each.

---

#### 5. Recommended remediation order

1. **D-01 / D-02** — Redis-fence media datagrams (or, at minimum, check
   `fenced.owner_runtime_id` against the expected owner and cap floor advancement),
   and add a community label to the media envelope so `get_unambiguous_by_channel`
   can be replaced with the community-scoped `get`. These two are the load-bearing
   security gaps and they share a fix surface.
2. **D-03 / D-04** — Either delete `POST /_mesh/demo/echo` (its own doc says "the
   route stays demo-gated until it is deleted") or move it behind the admin host with
   auth and derive `community_id` from the `Host`. Independently, make
   `HuddleDirectory::owner_of` reject a lease whose `profile != HuddleControl`, as
   `reliable.rs:99-105` already does.
3. **D-25 / D-26** — Add metrics: active rooms/peers gauges, dropped-frame counter,
   admission-rejection counter by reason, owner-loss counter. Zero observability on
   a realtime path is the fastest-compounding operational debt here.
4. **D-11 / D-12** — Extract stages 1–7 of `handle_active_audio_connection` into
   named async functions returning a small state struct, wrap the
   `cleanup_if_empty`/`send_clean_close` obligations in a `Drop` guard, and add
   tests over the resulting seams. This unblocks D-14 and D-18.
5. **D-05 / D-06 / D-07** — Give the audio route its own connection budget or a
   per-IP limiter; add a TTL/LRU bound to `GenerationFloor.seen`; add a per-peer
   frame-rate cap.
6. **D-29 … D-45** — Delete the dead code. ~830 production lines, no behavioural
   risk, and it removes two of the five name collisions (`mesh::Ownership`, the
   unused `NoopTracer`).
7. **D-46 … D-58** — Correct the documentation drift. D-46 (the
   `BUZZ_HUDDLE_AUDIO_AVAILABLE` guidance) and D-53 (the false NIP-44 encryption
   claim) are the two that could cause an operator or auditor to make a wrong
   decision; fix those first.
8. **D-41 + configuration** — Either wire `JsonlTracer` behind an env var
   (`BUZZ_CONFORMANCE_TRACE=<path>`) so the conformance seam becomes operable, or
   delete it and the `EmitGuard` arming in `ingest_event` so the crate stops paying
   for a mechanism that is inert. The current middle state — armed in production,
   discarding every result — is the worst of both.


## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Debt

Prioritized. Severity reflects blast radius **if the mesh is enabled** — today
`BUZZ_MESH` defaults off (`config.rs:498-500`) and nothing in this repo turns it on,
so most items are latent. D-04 is the exception: it is live regardless.

Baseline counts (all verified):

| Metric | Value |
|---|---|
| LOC | 3,169 across 10 files |
| Unit tests | **32**, all passing, **0 `#[ignore]`d** |
| Tests executed in CI | **0** |
| `unsafe` | **0** |
| `TODO`/`FIXME`/`XXX`/`HACK`/`todo!`/`unimplemented!` | **0** |
| `unwrap()` outside `#[cfg(test)]` | **0** |
| `expect()` outside `#[cfg(test)]` | **22** (all lock-poison) |
| `tokio::spawn` in production paths | **9** (all in `runtime.rs`) |
| Wire payload types | **5** (2 envelopes, 14 variants total across 6 enums) |
| `#[non_exhaustive]` public enums | **0 of 7** |
| Public items with zero external callers | ~40% of the surface |
| Unused declared dependencies | **3** (`hmac`, `futures-util`, `proptest`) |

---

#### CRITICAL

##### D-01 — Gossip records are applied without any authentication · CRITICAL (security)

`apply_gossip_record` (`membership.rs:120-153`) performs a version comparison and
nothing else, while its sibling `apply_ready_records` (`membership.rs:85-118`) does a
relay-pubkey anchor match plus a schnorr verify. `MeshStreamFrame::Gossip`
(`wire.rs:144-146`) carries no signature and no `FencedHeader`, and
`control_stream_exchange` applies every record in a `Delta` verbatim
(`runtime.rs:534-538`).

Any single admitted peer can therefore (a) insert arbitrary `runtime_id`s that then
pass `has_peer` admission on every other pod (`membership.rs:187-192` →
`runtime.rs:305-307`), bypassing the attestation the design exists to enforce
(`wire.rs:56-60`); (b) redirect every pod's 5-second dial loop at arbitrary
`host:port` (`runtime.rs:328-348`); (c) permanently pin a legitimate peer's addresses
or `draining` flag with a high `version`, which the version-1 registry seed can never
correct (BR-MESH-24).

**Fix:** embed `{relay_pubkey, relay_sig}` in `GossipRecord` and run the same
anchor+verify in `apply_gossip_record`. The preimage
(`registry.rs:85-91`) already binds the right tuple; this is a wire change, so it
needs an ALPN bump (`wire.rs:34-36`).

---

#### HIGH

##### D-02 — Membership table has no eviction, no expiry, and no revocation · HIGH

Verified: no `remove`/`retain`/`clear` anywhere in `membership.rs`. Entries persist
for the process lifetime. Consequences compound:

- `has_peer` admission is **monotonically permissive** — a retired or compromised
  runtime key stays admissible until every peer pod restarts. Registry TTL (45 s,
  `registry.rs:187`) has no effect on the in-memory allowlist.
- `reconcile_once` (`runtime.rs:296-326`) dials every non-draining record forever, so
  a scaled-down fleet leaves permanent dial churn.
- Unbounded memory growth: each entry holds a `Vec<String>` of arbitrary length plus a
  100-sample `PhiAccrual`, and per D-01 entries are attacker-insertable.
- Suspicion (`phi >= 8.0`) filters `peers()` (`membership.rs:366-369`) but not
  `has_peer` — and `peers()` has **zero production readers** (D-06), so suspicion has
  no effect at all today.

**Fix:** evict on `phi` beyond a hard threshold or on registry-record absence for
N scans; re-check attestation freshness periodically.

##### D-03 — Zero CI coverage: 32 tests compile and never run · HIGH

Verified chain:

- `just clippy` (`ci.yml:94`) = `cargo clippy --workspace --all-targets -- -D warnings`
  (`justfile:106-107`) — compiles the crate and its tests, runs nothing.
- `just test-unit` (`ci.yml:116`) runs only `-p buzz-core -p buzz-auth --lib`,
  `-p buzz-db --lib`, `-p buzz-conformance`, `-p buzz-push-gateway`
  (`justfile:275-295`). `buzz-relay-mesh` is absent.
- The backend-integration job archives `-p buzz-relay -p buzz-test-client --lib`
  (`ci.yml:336-342`). Also absent.
- `scripts/run-tests.sh` (the non-nextest fallback) covers the same four crates
  (`run-tests.sh:82-102`).

So the tie-break invariant, the attestation/anchor rejection tests, the wire
roundtrips, the 64-byte datagram header budget, and the Opus loss/order gate are all
regression-unprotected. They pass locally in 0.25 s — this is a one-line fix in
`justfile:278`.

##### D-04 — `GET /_mesh` is unauthenticated on an all-interfaces listener · HIGH (live today)

Route registered unconditionally at `crates/buzz-relay/src/router.rs:230`; handler
`router.rs:396-406` serialises `MeshStatus` whole (`router.rs:381`). The health
router (`router.rs:225-232`) has no auth layer, and its listener is bound to
**hard-coded `0.0.0.0:health_port`** (`main.rs:1119-1121`) — `BUZZ_BIND_ADDR` cannot
restrict it.

Leaks `peers[].endpoint_addrs` (`status.rs:20`) — the exact IP:port of every mesh UDP
socket in the fleet — plus all runtime ids, replica count, drain state, per-peer phi,
and 7 traffic counters per peer pair (`status.rs:17-61`).

Unlike the rest of this list, the *route* exists whether or not the mesh is enabled
(it returns `{"enabled": false}`, `router.rs:404`), so the surface is live now and
becomes an information leak the moment the mesh is switched on.

**Fix:** move it behind operator NIP-98 auth (the relay already has
`RELAY_OPERATOR_PUBKEYS` machinery), or drop `endpoint_addrs` from the default
projection.

##### D-05 — `iroh` dependency is declared with a pre-release requirement string · HIGH (supply chain)

`Cargo.toml:68`: `iroh = { version = "1.0.0-rc.0", default-features = false,
features = ["tls-ring"] }`.

**Nuance the brief should record:** `Cargo.lock` resolves iroh to **`1.0.2` from
crates.io** (`Cargo.lock:3902-3905`), because `^1.0.0-rc.0` admits stable 1.0.x. So
the shipped artifact is *not* on a release candidate. The manifest string is
nevertheless wrong in two ways: it advertises an rc dependency that is not what
builds, and pre-release requirements have surprising semver semantics (they opt the
requirement into matching other `1.0.0-*` pre-releases). It should read `"1.0"`.

Residual risk that is real: iroh 1.0.x is very new, `buzz-relay-mesh` is its only
consumer in the workspace, and the consumed surface is broad —
`endpoint::presets::Minimal`, `EndpointAddr::from_parts`, `TransportAddr`,
`Connection::remote_id/alpn/max_datagram_size`, `ReadExactError::FinishedEarly`
(`endpoint.rs:3,33,65-68,105-108`; `peer.rs:50,58,69,172`). A 1.1 minor will be
picked up by `cargo update` with no signal.

**Fix:** change the requirement to `"1.0"` (or pin `"=1.0.2"`), and add a
`cargo-deny`/`cargo-audit` gate — there is none in `ci.yml` today.

##### D-06 — The membership seam is wired and never read · HIGH (dead architecture)

`MeshHandle.membership: Arc<dyn RelayMeshMembership>` is populated at
`mesh_boot.rs:501` and **never read** — verified, no `.membership` field access on a
`MeshHandle` exists in `crates/buzz-relay/src`. Therefore:

- `RelayMeshMembership::peers()` (`membership.rs:359-379`) — 0 production callers.
- `RelayMeshMembership::local_runtime_id()` (`membership.rs:381-383`) — 0 callers.
- `PeerInfo` (`lib.rs:129-139`) is read nowhere; its only appearance in the consumer
  is `#[allow(dead_code)] fn _peer_info_is_not_an_owner_signal(_peer: PeerInfo)`
  (`tunnel/reliable.rs:949`), a comment-in-code.

Consequence: the entire "who is alive / draining / dialable?" seam — one of the two
seams `lib.rs:11-19` calls the crate's contract — has no consumer. No routing
decision anywhere in the relay consults liveness, load, or drain state of a peer.
Peer selection for a session is entirely the Redis directory's business. Combined
with D-02 this means phi accrual, the suspect threshold, and `load` are all
computation with no effect.

---

#### MEDIUM

##### D-07 — `MeshRuntime::shutdown()` is never called; loops and the heartbeat leak to process exit · MEDIUM

`runtime.rs:75-76` documents "dropping all clones does NOT stop the loops — call
`shutdown()`." Zero production callers (`runtime.rs:155-164`) — only in-crate tests.
The relay's drain watcher (`mesh_boot.rs:481-497`) calls `begin_drain()` +
`owners.drain_all()` then `return`s.

So after SIGTERM the accept loop keeps admitting connections, the reconcile loop keeps
dialing every 5 s, the gossip loop keeps bumping the local record every 2 s, and
`spawn_registry_heartbeat`'s task (`runtime.rs:598-607`) keeps running — its
`JoinHandle` is discarded at the call site (`mesh_boot.rs:467`) so it cannot be
stopped at all. Registry cleanup depends on that task getting one post-flag tick
inside its 15 s interval before the process exits (30 s graceful window,
`main.rs:1108`) — usually true, not guaranteed. `ReadyHeartbeat::shutdown()`
(`registry.rs:306-312`), which exists for exactly this, has zero callers.

##### D-08 — No reconnect backoff, and dials are serialized inside the reconcile loop · MEDIUM

`dial_peer` (`runtime.rs:328-355`) has no jitter, no exponential delay, and no failure
counter. `reconcile_once` awaits each `dial_peer` sequentially (`runtime.rs:317-326`),
so N unreachable peers serialize N connect timeouts into one pass, delaying dials to
healthy peers. With D-02 (records never evicted) a scaled-down or crashed fleet
produces permanent 5-second dial storms plus an unrate-limited `warn!` per attempt
(`runtime.rs:342-346`). All loops also lack jitter, so a fleet started together
gossips and rescans in lockstep (`runtime.rs:285-293`, `:563-587`, `:600-606`).

##### D-09 — Unauthenticated inbound connections trigger a full Redis scan + N signature verifies · MEDIUM

`is_known_peer` (`runtime.rs:305-323`) rescans the whole registry for any unknown
runtime id: serial `SCAN`+`GET`-per-key (`registry.rs:217-231`, no `MGET`, no
pipelining) with one secp256k1 schnorr verify per record (`registry.rs:233-238`). The
attacker cost is one generated ed25519 keypair plus a QUIC handshake; the defender
cost is a Redis round-trip per key plus N verifies — on the relay's **main**
`deadpool_redis` pool (`mesh_boot.rs:447`). The accept loop is single-tasked
(`runtime.rs:259-283`), so a slow scan head-of-line-blocks legitimate inbound
connections.

**Fix:** single-flight the rescan with a minimum interval, and negatively cache
rejected ids.

##### D-10 — `gossip::GossipState` is a complete unused duplicate of `MeshMembership`'s logic · MEDIUM

`gossip.rs:81-166`: `new/records/get/update_local/digest/delta_for/apply_delta`.
`MeshMembership` reimplements all of it (`membership.rs:120-153` ≈ `apply_delta`,
`:166-178` ≈ `update_local`, `:208-223` ≈ `digest`, `:225-247` ≈ `delta_for`).
`GossipState` has **zero callers outside its own tests** (verified across
`crates/**`), yet it carries 3 of the 4 gossip tests (`gossip.rs:238-266`) — so those
tests validate code that never runs, while the code that *does* run
(`MeshMembership`) is covered by only one merge test (`membership.rs:474-479`).

The two implementations have already drifted: `MeshMembership::apply_gossip_record`
sets `connection_state = Connected` on any accepted record (`membership.rs:133`),
which `GossipState` does not model — see D-13.

**Fix:** delete `GossipState` and move its two merge tests onto `MeshMembership`, or
make `MeshMembership` wrap it.

##### D-11 — 16 MiB frame length is honoured before the body arrives · MEDIUM

`peer.rs:178-186`: read a 4-byte attacker-controlled length, bounds-check against
`MAX_STREAM_FRAME` (16 MiB, `wire.rs:46`), then `vec![0u8; len as usize]` and
`read_exact`. Four bytes buy a 16 MiB allocation, and the peer can then stall.
`stream_accept_loop` (`runtime.rs:386-472`) accepts streams in an unbounded loop with
no per-peer cap or semaphore; `datagram_recv_loop` (`runtime.rs:358-372`) dispatches
inline with no rate limit or quota. The relay's own sender self-limits to 1 MiB
(`tunnel/reliable.rs:26-31`), but that is a convention, not an enforced receive
bound. Gossip is the only bounded path (depth 64, drop-on-full, `runtime.rs:46,556`).

##### D-12 — `ReadyRecord` attestation covers only the two keys · MEDIUM

Preimage (`registry.rs:85-91`) = `ATTESTATION_CONTEXT` + `runtime_pubkey` +
`relay_pubkey`. **Not signed:** `endpoint_addrs`, `proto_version`, `capabilities`.
**No nonce, no issued-at, no expiry.** So anyone with Redis write access can rewrite
a validly-attested record's dial addresses (a redirection primitive on the registry
path, mirroring D-01 on the gossip path), and a captured record is replayable
forever. Redis ACLs are the only defence, and the crate asserts nothing about the
pool it is handed.

##### D-13 — `apply_gossip_record` lies about connection state · MEDIUM

`membership.rs:133` and `:146` set `connection_state = ConnectionState::Connected`
for **any** accepted record, including records about *third parties* learned second-hand
through a `Delta`. A pod therefore reports peers it has never connected to as
`"connected"` on `/_mesh` (`status.rs:23`), and a genuinely disconnected peer flips
back to `Connected` the moment any neighbour gossips a newer record about it —
overwriting the `Disconnected` that `remove_peer` just set (`runtime.rs:277-279`).
`/_mesh`'s connection_state field is not trustworthy.

##### D-14 — Documented observability does not exist · MEDIUM

| Claim | Reality |
|---|---|
| `mesh_fence_rejections_total{reason=stale_generation\|no_active_lease\|owner_mismatch\|future_generation}` (`lib.rs:102-109`, attributed to a chaos-gate ruling) | **no such metric anywhere in the repo**; the crate has no `metrics` dependency at all |
| `MeshCounters.stale_generation_rejections` (`status.rs:43`) | structurally always 0 — the only writer, `record_stale_generation_rejection` (`membership.rs:285-293`), has no caller but the test at `:486`. The relay's real fence rejects are raised in `tunnel/directory.rs:378,395,413,430` and never routed back |
| "reject unknown versions loudly (count it, log it)" (`wire.rs:39-41`) | logged (`runtime.rs:406,549`), never counted |

Net: a cross-pod fencing incident produces log lines and a `/_mesh` page reading all
zeros. No spans either — `tracing` is used only for events, no `#[instrument]`, so a
cross-pod session cannot be traced end to end.

##### D-15 — `MeshError::Transport(String)` swallows 8 distinct failure classes · MEDIUM

22 construction sites. iroh bind/connect/stream/datagram errors are flattened with
`err.to_string()` at 12 of them (`endpoint.rs:38,39,79,91,101`;
`peer.rs:82,95,113,123,151,155,163,174,187`), discarding iroh's structured error;
five attestation failure causes collapse into the same variant
(`registry.rs:56-83`), so "malformed hex" and "signature forgery" are
indistinguishable to a caller and cannot be alerted on separately. The unknown
*gossip* payload version is a `Transport` string (`gossip.rs:72`) while the
unknown *frame* version gets a typed variant (`wire.rs:185`) — asymmetric.

This directly contradicts the crate's own stated policy at `lib.rs:102-105`: "every
fence-visible reject is a typed variant, never a generic `Transport`."

---

#### LOW

##### D-16 — Six stale or incorrect doc comments · LOW

| Location | Claim | Reality |
|---|---|---|
| `lib.rs:55-56` | `BUZZ_MESH` is "`on` (default when replicas can exist)" | defaults **off** (`config.rs:498-500`), tested `mesh_boot.rs:544-555` |
| `lib.rs:186-188` | `MeshStream` halves are "placeholder … pre-transport" | they are the real iroh halves (`peer.rs:132-192`) |
| `lib.rs:11-19` | relay consumes the crate "exclusively through two seams" | also uses `InboundHandler`, `MeshStream`, both half traits, `MeshEndpoint`, `MeshPeer`, `GossipRecord`, `ReadyRecord`/`ReadyRegistry`, `MeshRuntime`, `spawn_registry_heartbeat` |
| `lib.rs:137-138` | `PeerInfo.load` is "gossiped by the peer (0.0..)" | structurally always `0.0` — no writer exists (`gossip.rs:35`, `runtime.rs:566` is a no-op closure) |
| `wire.rs:31` | non-`Hello` first frame ⇒ "the stream is reset" | dropped, not reset (`runtime.rs:404-406`); no `reset()` call exists in the crate |
| `gossip.rs:168` (type name `PhiAccrual`) | implies the Hayashibara phi-accrual detector | variance is never used; `mean_secs` (`:217-220`) is the only statistic — exponential approximation only (`:213-214`) |

##### D-17 — `StreamHello.sender` is never validated against the authenticated peer · LOW

`wire.rs:161` declares `sender: RuntimeId`; nothing compares it to `conn.remote_id()`.
The accept loop passes `hello` straight through (`runtime.rs:461-464`) and consumers
use the authenticated `from` argument, so this is latent — but the field is
attacker-controlled and its presence invites a future consumer to trust it. Also,
`wire.rs:29-31` says the `Hello` contract holds "in both directions," yet nothing
reads the peer's `Hello` on a stream we opened (`runtime.rs:190-191`).

##### D-18 — Three declared dependencies are unused · LOW

`hmac` (`Cargo.toml:20`) and `futures-util` (`:25`) have **zero `use` sites** in
`src/`; `proptest` (`:29`) is a dev-dependency with **zero property tests** despite
the workspace convention of using it (`Cargo.toml:117`). `tokio`'s `test-util`
feature is enabled (`:29`) but `tokio::time::pause` is never used. Removing the three
cuts compile time and audit surface; a proptest for the postcard wire roundtrip and
the scuttlebutt merge would be a natural fit.

##### D-19 — Two parallel counter models, one of them write-only · LOW

`peer::PeerCounters` (`peer.rs:10-15`, atomics at `:19-24`, incremented at
`:84,97,114,121`) and `status::MeshPeerCounters` (`status.rs:51-61`, incremented via
`membership.rs:249-283`) track overlapping quantities with different field names
(`streams_accepted` vs `streams_received`) and are never reconciled.
`MeshPeer::counters()` (`peer.rs:73-75`) has **zero callers** — the per-connection
atomics are pure overhead. Additionally `MeshCounters.peers` duplicates every
`MeshPeerStatus.counters` verbatim in the `/_mesh` JSON (`membership.rs:302`).

##### D-20 — 22 lock-poison `expect()`s escalate a single panic to fleet-wide mesh failure · LOW

`membership.rs:74,126,159,173,190,199,322,332,363`;
`runtime.rs:142,156,159,168,183,197,202,222,270,349,444,553,573`. AGENTS.md forbids
new `expect()` in production paths; these are the conventional lock-poison case, but
one panic inside any critical section makes every subsequent mesh call panic —
including `MeshHandle::status()` (`mesh_boot.rs:173`), i.e. the axum health handler.
`parking_lot` or `unwrap_or_else(|e| e.into_inner())` removes the escalation. No
panic path was found inside those sections (all arithmetic is `saturating_*`, no
indexing), so this is robustness, not an exploit.

##### D-21 — One detached task and one discarded `JoinHandle` break the abort discipline · LOW

The crate's pattern is "track every `JoinHandle`, abort on removal"
(`PeerEntry::abort`, `runtime.rs:56-62`). Two of nine spawns escape it:
`control_stream_exchange` on the accept side (`runtime.rs:449`) is spawned without
being pushed onto `PeerEntry.tasks`, so `remove_peer` aborts the recv loops and
leaves it running until its stream errors; `spawn_registry_heartbeat`'s handle
(`runtime.rs:598`) is dropped by the caller (`mesh_boot.rs:467`).

##### D-22 — Registry seeds are permanently version 1, so a stale gossiped address can never be corrected · LOW

`apply_ready_records` builds `GossipRecord::new(...)` (version 1, `gossip.rs:38`) and
routes it through the version-greater-wins merge (`membership.rs:110-116`). Once any
gossip has been received for a peer, its version exceeds 1 forever, so a *corrected*
Redis record — e.g. after a pod IP change — can never be applied. Combined with D-01
this makes a forged high-version record permanently authoritative.

##### D-23 — No capability or protocol-version negotiation · LOW

`proto_version` (`registry.rs:110`, `gossip.rs:19`) and `capabilities`
(`mesh_boot.rs:371-377`) are advertised and **never compared** — verified: every
occurrence is an assignment or a `/_mesh` echo. `open_session_stream` will open a
`HuddleControl` stream to a peer that never advertised it; the only backstop is the
receiver's dispatcher (`mesh_boot.rs:112-134`). Combined with zero
`#[non_exhaustive]` on any of the 7 public enums, forward compatibility within one
ALPN is nil; the design intent (bump ALPN with the wire version, `wire.rs:34-36`) is
sound but unenforced.

##### D-24 — `git`-sourced dev-dependencies on a third-party project · LOW→MEDIUM (build/supply chain)

`crates/buzz-relay/Cargo.toml:84-85`:

```toml
mesh-llm-sdk          = { git = "https://github.com/Mesh-LLM/mesh-llm.git", tag = "v0.73.1", … }
mesh-llm-host-runtime = { git = "https://github.com/Mesh-LLM/mesh-llm.git", tag = "v0.73.1", … }
```

Not a dependency of `buzz-relay-mesh`, but it lands squarely in this module's blast
radius: these dev-deps exist **only** to build the five `mesh_*` examples, all of
which are MeshLLM smoke tests unrelated to the inter-relay mesh (see
`-features.md` §5). Because CI runs `cargo clippy --workspace --all-targets`
(`ci.yml:94`, `justfile:106-107`), **every CI run must fetch and build a
git-sourced third-party LLM runtime** — which is why `ci.yml:1015-1052`,
`release.yml:141-181`, and `signed-macos-canary.yml:100-139` all carry llama
native-library build/cache plumbing.

Specific risks: tag references are mutable (a retagged `v0.73.1` changes the build,
though `Cargo.lock` pins the rev, resolved at `ci.yml:1019`); the upstream repo is
a hard availability dependency of CI; and the C++/Metal runtime it pulls in is the
source of the teardown crashes the examples work around
(`mesh_serve_client_smoke.rs:129-136`, `mesh_agent_e2e.rs:180-193`).

**Fix:** move the five examples behind a cargo feature so `--all-targets` skips them
by default, or relocate them to a separate crate excluded from the root workspace
(the pattern already used for `desktop/src-tauri`).

##### D-25 — Naming collision: two unrelated "mesh" subsystems in one crate · LOW

`crates/buzz-relay` contains both `mesh_boot.rs`/`api/mesh_demo.rs` (this
inter-relay QUIC mesh) and `examples/mesh_*.rs` (MeshLLM shared compute). The
desktop app adds a third spelling (`BUZZ_MESH_API_PORT`, `BUZZ_MESH_CONSOLE_PORT`,
`BUZZ_MESH_IROH_RELAYS`, `desktop/src-tauri/src/mesh_llm/mod.rs:37-47`), and
`justfile` targets `mesh-e2e`, `mesh-e2e-hardware`, `mesh-e2e-admission`,
`mesh-e2e-confidence` (`justfile:304-349`) all mean MeshLLM, not this crate. Both
subsystems even use iroh. This has already produced at least one mis-scoped
assumption in this very analysis brief. Rename one side (`relay-mesh` vs
`shared-compute`) before adding more surface.

---

#### Documentation debt

##### D-26 — The crate is absent from every repo-level document · MEDIUM

Verified by grep:

| Document | `buzz-relay-mesh` | `mesh` / `iroh` / `quic` |
|---|---|---|
| `ARCHITECTURE.md` (827 lines, §6 "Crate Reference" at `:330`) | absent | **zero hits anywhere in the file** |
| `AGENTS.md` repo-structure table | absent | only mesh-llm-adjacent text |
| `CONTRIBUTING.md` | absent | — |
| `README.md` | absent | — |
| `.env.example`, `deploy/`, helm charts | absent | no `BUZZ_MESH`, no `3478`, no `POD_IP` |
| `.github/CODEOWNERS` | absent | — |

So a 3,169-LOC crate that binds a new UDP port, writes a new Redis key namespace,
introduces a second serialization codec (postcard), a second identity system
(ed25519 node keys alongside secp256k1 Nostr keys), and an unauthenticated HTTP
status route is documented **only in its own source comments**. Those comments are
good — they record rationale and named review blockers (`wire.rs:52-60`,
`lib.rs:102-109`, `mesh_boot.rs:544-546`) — but they are the sole record, and six of
them are already stale (D-16).

The `lib.rs:30-36` per-file human-owner map ("Mari", "Max", "Perci", "Dawn") has no
CODEOWNERS backing, so it will silently rot.

---

#### Prioritized remediation order

1. **D-03** — add `-p buzz-relay-mesh` to `justfile:278`. One line; unlocks
   regression protection for everything below.
2. **D-04** — authenticate or narrow `GET /_mesh`. Live today, independent of
   `BUZZ_MESH`.
3. **D-01** — attest gossip records (needs an ALPN bump; do it before any other wire
   change).
4. **D-02 / D-09** — evict membership entries and single-flight the admission rescan.
5. **D-05 / D-24** — fix the `iroh` requirement string, add `cargo-deny`, feature-gate
   the MeshLLM examples out of `--all-targets`.
6. **D-06 / D-10 / D-14** — decide the membership seam's fate, delete `GossipState`,
   and either implement the documented metrics or delete the claims.
7. **D-07 / D-08 / D-11 / D-21** — lifecycle: call `shutdown()`, add backoff, cap
   per-peer streams, track the detached task.
8. **D-16 / D-26** — correct the six stale doc comments and add the crate to
   `ARCHITECTURE.md` §6 and `AGENTS.md`.


## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: Debt

Items are ordered by how much they subtract from the gate's stated purpose. Data-model-level
drift (label doc mismatches, `SanitizedReason` 9→3 collapse, `Verdict_`, fixture key names) is
catalogued in `buzz-conformance-data-model.md` and referenced rather than repeated.

---

### D-CONF-01 — The checker has no production consumer; the gate is unassembled

`check_trace` (`src/checker.rs:74`) is called only from the crate's own tests — repo-wide grep
for `check_trace` outside `crates/buzz-conformance/{src,tests}` and its two markdown docs
returns nothing. `JsonlTracer` (`crates/buzz-relay/src/conformance/tracers.rs:30-72`), the only
tracer that persists anything, is never instantiated: grep for `JsonlTracer` finds the
definition, the re-export (`conformance/mod.rs:46`), two doc comments
(`state.rs:616`, `:795`), and two markdown mentions.

So the relay→JSONL→checker pipeline the crate exists to run is not assembled anywhere. The
crate's docs acknowledge it as pending: "The integration replay is the **next** ratchet"
(`LIMITS.md:120-121`), "lands with the read-seam patch onto Max's req.rs work"
(`LIMITS.md:116-118`) — but the read-seam patch has landed (`handlers/req.rs:338-365`,
`:649-677`) and the replay did not follow.

**Evidence the emit half is nonetheless live:** `handlers/ingest.rs:1408`, `:1436-1444`,
`:1438`, `:1820`, `:2215`, `:2374`, `:2511`; `handlers/req.rs:144`, `:359`, `:671`.

---

### D-CONF-02 — `NoopTracer` in production, with real cost still paid

`AppState` binds `Arc::new(crate::conformance::NoopTracer)` (`crates/buzz-relay/src/state.rs:798`)
and nothing overwrites it. `NoopTracer::record` is empty (`tracers.rs:18-20`).

The relay nonetheless performs, per request, work whose only consumer is that empty body:

| Work | Site | Frequency |
|---|---|---|
| `state_for_request` — host `to_string()`, pubkey hex slice, `AbstractState` | `conformance/mod.rs:55-61` | once per ingest at `ingest.rs:1407` + once per emit at `:1827`, `:2221`, `:2374`, `:2511`, `:145` |
| `EmitGuard::arm` — 2 `Arc`s + `AtomicUsize` | `conformance/mod.rs:383-400` | every ingest (`ingest.rs:1408`) |
| `claimed_community_from_event` — tag scan + `Uuid::parse_str` | `conformance/mod.rs:101-119` | `ingest.rs:143`, `:1814`, `:2219`, `:2360`, `:2494` |
| **extra `db.communities_of_channels` query** | `handlers/req.rs:349`, `:661` | **every REQ filter, every search page** |
| `BTreeSet` of distinct row channels + per-row label `Vec` | `handlers/req.rs:339-348`, `:648-660`; `conformance/mod.rs:238-260` | same |

The DB round-trip is the material item. Its gate is `trace_state.is_some()`
(`handlers/req.rs:338`, `:649`), and `trace_state` is derived solely from whether the
authenticated pubkey bytes parse (`:116-118`) — the code comment says "on the hot read path
this is always `Some`" (`:114-115`). It is **not** gated on the tracer type, so the query runs
under `NoopTracer`. Neither `req.rs` nor `conformance/mod.rs` exposes a
`tracer.is_recording()`-style predicate to short-circuit it, and the `Tracer` trait has no such
method (`src/lib.rs:314-318`).

Secondary effect: a `communities_of_channels` failure logs `warn!` on a purely observational
path (`req.rs:347-353`, `:663-669`) and substitutes an empty map, which turns the whole page
into a single discarded `ImplBug`.

---

### D-CONF-03 — Seven `Ok` paths through ingest emit nothing

Every one returns success with no trace step, so the `EmitGuard` fires `ImplBug` and the
checker would classify a legitimate accepted write as `CoverageBreach` — indistinguishable from
a genuinely forgotten emit site.

| # | Path | Line | Kinds |
|---|---|---|---|
| 1 | `handle_command` routing | `ingest.rs:1560-1562` | 30620, 41010, 41011, 41012, 46020, 46030, 46031 (`buzz-core/src/kind.rs:743-754`) |
| 2 | NIP-56 report | `ingest.rs:1587-1596` | 1984 (`buzz-core/src/kind.rs:267`) |
| 3 | moderation commands | `ingest.rs:1605-1614` | 5 kinds (`buzz-core/src/kind.rs:316-325`) |
| 4 | relay-admin | `ingest.rs:1834-1842` | 4 kinds (`buzz-core/src/kind.rs:723-731`) |
| 5 | NIP-43 leave | `ingest.rs:1846-1928` | 28936 (`buzz-core/src/kind.rs:344`) |
| 6 | reaction duplicate | `ingest.rs:2341-2347` | 7 (`buzz-core/src/kind.rs:58`) |
| 7 | stored-event duplicate | `ingest.rs:2452-2458` | all stored kinds |

Paths 1 and 3 are the security-relevant ones: both perform community-scoped authorization and
durable mutation inside their own handlers, so a tenant-boundary bug there produces no
`AuthCheck` and no `Write*` observation.

Contrast with the paths that *were* instrumented for exactly this reason: product feedback
(`ingest.rs:1567-1580`, with a dedicated regression test at `:2556-2591` asserting the guard is
satisfied) and push lease (`:2215-2222`). The pattern exists; it was applied to two of nine
early-return success paths.

---

### D-CONF-04 — `WriteDuplicate` is unreachable on the main write path

`handlers/ingest.rs:2452-2458` returns early when `!was_inserted`. The trailing emit block's
match therefore never sees `was_inserted == false`:

```
(Some(ch), false) => TraceAction::WriteDuplicate { … }   // ingest.rs:2501-2505 — dead
(None, _)         => TraceAction::WriteInsertGlobal { … } // ingest.rs:2506-2509 — `_` only ever true
```

The comment above the block (`:2484-2492`) reasons carefully about the duplicate case as if it
were live. The only reachable `WriteDuplicate` producer is the reaction lane
(`:2368-2372`), which requires `buzz_db::ReactionEventInsertOutcome::Inserted` to carry
`was_inserted: false` (`:2348-2351`) — the explicit `Duplicate` outcome returns at `:2341-2347`
without emitting. Net: `WriteDuplicate`, the spec action for the duplicate-probe oracle
(`docs/spec/MultiTenantRelay.tla:606`), is effectively never emitted, while duplicate writes
themselves *are* accepted and silent (D-CONF-03 item 7).

---

### D-CONF-05 — Spec coverage is a small fraction of what the spec asserts

`Next` has 23 disjuncts (`docs/spec/MultiTenantRelay.tla:933-956`); the schema models 8.
`Safety` conjoins 13 invariants (`:1129-1142`); the Rust predicates enforce
`Inv_NonInterference` for read observations (`src/transitions.rs:294-312`) and a fragment of
`AuthCheck` (`:228-250`). The other eleven are structurally uncheckable because
`AbstractState` (`src/lib.rs:150-175`) carries three fields and none of the spec's state
variables:

| Invariant | Line | Missing state |
|---|---|---|
| `Inv_LabelPropagation` | `:990` | `o.kind`, `o.rows`, `o.audit` |
| `Inv_ResolutionFence` | `:1011` | `ChannelCommunity(_)` |
| `Inv_HostBindingFence` | `:1038` | `HostCommunity[_]` — conceded at `src/transitions.rs:110-114` |
| `Inv_ChannelCommunityImmutable` | `:1057` | `createdChannels` |
| `Inv_AdmissionFence` | `:1071` | `admittedMembers` — conceded at `src/transitions.rs:222-224`, `:253-256` |
| `Inv_AcceptedWritesPersist` | `:1104` | `messages` |
| `Inv_MessageKeyUnique` | `:1110` | `messages` |
| `Inv_NoTenantContextFailsClosed` | `:1116` | row count vs. label-set emptiness |
| `Inv_ProjectionDerived` | `:1121` | `projections` |
| `Inv_SanitizedErrors` (9-wide) | `:1124` | 6 of the 9 `.cfg:26` reasons |

The write arms return `Ok(())` unconditionally (`src/transitions.rs:187`, `:191`, `:198`), so
`Inv_ResolutionFence` and `Inv_HostBindingFence` — the two invariants that state the
host/channel fence — contribute nothing at runtime. The rationale given
(`src/transitions.rs:183-186`: "the gate that bites it is the next read's row labels") is only
valid for requests that also read.

Two more gaps in the same family:
- **`ReadHostFeedRows` has no emitter.** Grep across `crates/buzz-relay/` finds none; the
  variant is only constructible from `tests/proptest_checker.rs:164-166` and `:255-257`.
- **The M2/M8 rule is inert on the read path.** `record_req_authcheck` hard-wires
  `claimed_community: None` (`conformance/mod.rs:152`) and the checker's guard requires
  `Some(c)` (`src/transitions.rs:233`). The `None` choice is deliberate and documented
  (`mod.rs:135-145`), but the consequence — one of the two named mutation targets cannot fire
  on half the seam — is not.

---

### D-CONF-06 — The `h`-tag semantic makes the write-path M2 rule a likely false positive

`claimed_community_from_event` (`conformance/mod.rs:101-119`) reads the first `h` tag and parses
it as a UUID. In Buzz, `h` carries a **channel** UUID, not a community UUID — the repo
convention is stated in `AGENTS.md` ("Channels use `h` tags (NIP-29 group tag)") and the
emitter's own comment concedes the ambiguity (`mod.rs:103-105`: "or channel uuid, ambiguous").

The one consumer of the field is the `Allow` + `claimed != resolved` bite
(`src/transitions.rs:233-242`). Since a channel UUID essentially never equals a community
UUID, arming a recording tracer today would produce `IllegalTransition` on nearly every
authorized channel write (`ingest.rs:1788-1802` emits `AuthCheck` with `verdict: Allow` and
`claimed_community: Some(channel_uuid)`).

Two committed artifacts encode the community reading and would therefore not catch this:
`tests/replay_fixtures.rs:78-86` builds `AuthCheck { claimed_community: Some(community_a()) }`
and `good.jsonl` records it. Full detail in the data-model doc's D-CONF-02.

---

### D-CONF-07 — Stale in-crate documentation

| Claim | Location | Reality |
|---|---|---|
| "req.rs / event.rs: (held back as additive patch for Eva to apply onto Max's req.rs writes — see thread `c882c9b1…`)" | `conformance/mod.rs:37-38` | landed — `handlers/req.rs:334-361`, `:649-677` |
| `req.rs` row of the emitter table reads "**held back**" | `TRACE_SCHEMA.md:137` | same |
| "the read-seam half of the gate is **not yet armed**" | `LIMITS.md:56-57` | armed |
| "How the emitter computes that label is the design question for the held-back req.rs patch … The choice is Eva's review call before fixtures land" | `LIMITS.md:47-57` | decided (the (B) strategy, `conformance/mod.rs:170-208`) and fixtures landed |
| "9 + 5 + 2 = 16 tests" | `LIMITS.md:112` | 9 lib + 6 replay + 7 proptest = 22 in-crate, plus 9 in `conformance/mod.rs` and 1 in `ingest.rs` |
| CI contract lists three commands, omits the proptest lane | `LIMITS.md:88-118` | `tests/proptest_checker.rs` (7 properties) is not mentioned |
| "conformance tests bind `JsonlTracer`" | `state.rs:616`, `tracers.rs:1-2` | no test binds it |
| "see test helpers in `crates/buzz-test-client` once those land" | `state.rs:796-797` | `crates/buzz-test-client/tests/conformance_multitenant.rs` never references `buzz_conformance` |
| "the build can omit emission entirely behind a feature" | `src/lib.rs:321-322`, `tracers.rs:9-13` | no `[features]` table exists in `crates/buzz-conformance/Cargo.toml` |
| record shape `{"schema":…, "state":…}` | `TRACE_SCHEMA.md:37-46` | wire keys are `schema_version` / `state_after` (`src/lib.rs:292`, `:298`) |
| `WriteInsertGlobal` "line 562", `WriteDuplicate` "line 612" | `TRACE_SCHEMA.md:57`, `:69` | `docs/spec/MultiTenantRelay.tla:559`, `:606` |
| `ReadHostFeedRows` "line ~720" | `src/lib.rs:193` | `docs/spec/MultiTenantRelay.tla:703` |

Cross-crate comment references to conversations not in the repo compound this:
`conformance/mod.rs:37-38` (thread `c882c9b1…`), `tests/replay_fixtures.rs:19-20` (thread
`06aaf3f7…`), `mod.rs:170-172` and `tests/replay_fixtures.rs:145-152` ("Eva specified" /
"Eva requested"). The `M1`…`M8` mutation IDs used throughout (`src/lib.rs:127`, `:190`, `:238`;
`src/transitions.rs:218-221`, `:239-240`; `conformance/mod.rs:18-19`; `ingest.rs:1779`) have no
legend anywhere in the repo.

---

### D-CONF-08 — Repo-level documentation omits the crate entirely

Grep for `buzz-conformance`, `conformance`, `MultiTenantRelay.tla`, `TLA`, and `formal` across
`AGENTS.md`, `ARCHITECTURE.md`, and `CONTRIBUTING.md` returns **no hits**.

- `AGENTS.md`'s repo-structure crate table lists `buzz-audit` at `:46` and its neighbours but
  has no `buzz-conformance` row, so an agent following that table does not know the crate
  exists — or that new tenant-boundary endpoints are supposed to arm an `EmitGuard`
  (`LIMITS.md:36-41`: "New endpoints touching the tenant boundary MUST arm a guard at entry —
  that's enforced by code review, not by the harness"). That rule is stated only inside the
  crate's own `LIMITS.md`, which nothing links to.
- `ARCHITECTURE.md` has no formal-methods section, so `docs/spec/MultiTenantRelay.tla` (1,142
  lines) and `docs/spec/GitOnObjectStore.tla` are undocumented from the top level.
- No GitHub workflow references conformance (grep `conformance` in `.github/workflows/`).

---

### D-CONF-09 — Unreferenced public surface

Six public items with zero callers anywhere in `crates/`, none marked
`#[allow(dead_code)]` or TODO'd:

| Item | Line | Note |
|---|---|---|
| `Verdict_ { Ok }` | `src/transitions.rs:53-56` | "Reserved — internal placeholder"; see data-model doc |
| `action_channel` | `src/transitions.rs:318-330` | doc claims it is "Used by the checker" (`:315-317`) — it is not |
| `TraceAction::is_critical` | `src/lib.rs:283-285` | returns `true` unconditionally; the "every emit site marked critical" mechanism it documents (`src/lib.rs:279-282`) is not implemented — `check_trace`'s coverage logic uses `kind()` strings instead (`src/checker.rs:113-118`) |
| `Scenario::require` | `src/checker.rs:54-57` | all 22 scenario constructions use `unstructured` or a struct literal |
| `conformance::step` | `crates/buzz-relay/src/conformance/mod.rs:121-123` | "keep the call sites in ingest.rs short" — no call site uses it |
| `buzz_conformance::NoopTracer` | `src/lib.rs:323-327` | shadowed by the relay's identically-named copy (`tracers.rs:16-20`), which is what `mod.rs:46` re-exports and `state.rs:798` binds |

---

### D-CONF-10 — Coverage-set membership is stringly-typed and opt-in

`required_critical_actions: HashSet<String>` (`src/checker.rs:37`) is compared against
`TraceAction::kind()` outputs (`src/checker.rs:113-118`). Nothing validates the required strings
against the enum, so a typo becomes a permanently-missing requirement that reports as a
`CoverageBreach` naming a nonexistent action. And 18 of the 22 scenario constructions in the
crate declare no requirements at all (`Scenario::unstructured`, listed in the business-rules
doc under BR-CONF-10), which means the "scenario-required action never appeared" mode —
described in `src/lib.rs:35-36` as what keeps the gate from being "decorative logging" — is
exercised by four sites: `src/checker.rs:200-205`, `:315-317`,
`tests/replay_fixtures.rs:240-247`, `:308-315`.

---

### D-CONF-11 — Relay-side emitter tests are outside the unit gate

`just test-unit` runs `-p buzz-core -p buzz-auth --lib`, `-p buzz-db --lib`,
`-p buzz-conformance`, `-p buzz-push-gateway` (`justfile:278-293`); the shell fallback matches
(`scripts/run-tests.sh:81-102`). `buzz-relay` is in neither list, so the 9 emitter tests in
`crates/buzz-relay/src/conformance/mod.rs:466-726` and the guard-satisfaction test at
`handlers/ingest.rs:2556-2591` — the ones that prove `EmitGuard` fires, that
`member → Allow` / `!member → Deny`, and that a missing projection lookup becomes `ImplBug`
rather than a substituted label — do not run in the pre-PR gate.
`run_integration_tests`' catch-all is `cargo test --test '*'` (`scripts/run-tests.sh:118-120`),
which selects integration targets, not `--lib` unit tests. `LIMITS.md:105-110` names
`cargo test -p buzz-relay --lib conformance::` a required CI surface; nothing runs it.

The same omission applies to `buzz-relay-mesh` (`Cargo.toml:27`), which appears in no unit
step either — so this is a general gap in the unit gate's crate list rather than a
conformance-specific oversight.


## Module: buzz-acp — harness core & orchestration (`crates/buzz-acp/src`)
### Aspect: Debt

#### File size

`lib.rs` is 6,570 lines / 266,642 bytes in a single file. Five of the crate's twelve source files exceed 3,700 lines:

| File | Lines | vs. the 1,000-line ceiling enforced on desktop/web/mobile |
|---|---|---|
| `lib.rs` | 6,570 | 6.6× |
| `relay.rs` | 6,233 | 6.2× |
| `pool.rs` | 5,620 | 5.6× |
| `queue.rs` | 4,759 | 4.8× |
| `acp.rs` | 3,717 | 3.7× |

`AGENTS.md § Mobile App` mandates a hard 1,000-line ceiling enforced by `mobile/scripts/check-file-sizes.mjs` via `just mobile-check`, mirroring desktop and web guards, and instructs "split the file — never bump the limit". No equivalent guard exists for Rust crates, and `buzz-acp` is where that gap is most visible.

Within `lib.rs`, `tokio_main` alone spans **1,462 lines** (`lib.rs:1239-2700`) as one function containing a `select!` with twelve arms, a nested enum declaration (`PoolEvent`, `lib.rs:1700-1705`), and the entire shutdown sequence. `handle_prompt_result` is 366 lines (`lib.rs:3034-3399`) with 11 parameters behind `#[allow(clippy::too_many_arguments)]` (`lib.rs:3033`) — the same `allow` appears on `recover_panicked_agent` (`lib.rs:3401`) and `drain_ready_join_results` (`lib.rs:3500`).

#### Duplication with `buzz-ws-client`

`buzz-acp` does not depend on `buzz-ws-client` (`Cargo.toml:19-22` lists only `buzz-core`, `buzz-sdk`, `buzz-persona`). It pulls `tokio-tungstenite` directly (`Cargo.toml:31`) and reimplements the full client in `relay.rs` — 6,233 lines including its own `RelayMessage` enum (`relay.rs:471`), `do_connect` (`relay.rs:3825`), `wait_for_auth_challenge` (`relay.rs:3865`), `send_auth_response` (`relay.rs:3433`), reconnect state machine (`relay.rs:2893`, `3022`), terminal-error classification (`relay.rs:3657-3785`), and two-generation dedup (`relay.rs:935`). It also diverges functionally: `relay.rs` re-authenticates on a mid-session AUTH challenge where `buzz-ws-client` only records it, so the two clients now have different NIP-42 semantics.

`lib.rs`'s share of that divergence is the routing knowledge encoded in comments rather than types: `publish_presence`'s doc comment (`lib.rs:69-75`) has to explain that ephemeral kinds 20000–29999 are rejected by the HTTP bridge and must go over WS, and the typing arm has to use `try_publish_event` because a blocking publish would stall the main loop during reconnect (`lib.rs:2329-2331`, citing issue #35). None of this is enforced by the type system — a future contributor can call `publish_event` with an ephemeral kind and it will compile.

#### Zero-usage dependency

`buzz-persona = { path = "../buzz-persona" }` at `Cargo.toml:22`. `grep -rn 'buzz_persona\|buzz-persona' crates/buzz-acp/` returns exactly one line: the manifest. It is also the only internal dep declared by path rather than `workspace = true`.

`Config::persona_env_vars` (`config.rs:533-535`) is a plain `Vec<(String,String)>` built in `config.rs:945-999` and threaded through `lib.rs:1762`, `3488`, `3666`, `3733` into `AcpClient::spawn` — no call into the crate. The dependency adds compile time and a version-coupling edge for nothing.

#### Dead / unreachable code

| Item | Evidence |
|---|---|
| `BUZZ_API_TOKEN` | Propagated from its legacy alias at `config.rs:718` but never read anywhere in `crates/buzz-acp/src/`. Documented as a real setting in `README.md § Core`. |
| `Config::allowed_respond_to` | `config.rs:460`; referenced only by `config.summary()` (`config.rs:1019-1025`). No gate reads it. |
| `pub use usage::TurnUsage` | `lib.rs:15`; `TurnUsage` appears nowhere else in `lib.rs`. |
| `RespawnResult` doc comment | `lib.rs:1142-1145` documents a third tuple element `supports_goose_steer` that is "always `true`". The tuple is `(AcpClient, u32, String)` — the described field does not exist. Stale doc. |
| `SubscribeMode::All` with no `--kinds` | `lib.rs:1456` produces an empty kinds vector, which per `AGENTS.md § Common Gotchas #2` trips the relay p-gate and 403s. `--subscribe all` is documented in `README.md § Forum Channels` only ever paired with an explicit `--kinds`. Nothing warns. |
| Owner re-resolution | `OwnerCache.pubkey` is set once at construction (`lib.rs:158-163`) with no setter. `respond-to=owner-only` + startup resolution failure = permanently deaf agent, and `--relay-observer` silently off (`lib.rs:1421-1425`). |

#### TODO / FIXME inventory

`grep -cn -E 'TODO|FIXME|XXX|HACK' crates/buzz-acp/src/lib.rs` → **0**.

No inline debt markers at all. The debt is instead recorded as prose in long comment blocks, which is harder to inventory mechanically. Examples of accepted-limitation comments that would conventionally be TODOs:

| Accepted limitation | Site |
|---|---|
| Membership remove→re-add race can requeue a stale batch; fix needs per-channel epochs on `TaskMeta` + `PromptResult`, judged not worth it | `lib.rs:1670-1680` |
| Stale 👀 reactions after membership removal because the relay revokes access before emitting the notification; "fix belongs in the relay" | `lib.rs:2000-2006` |
| Fire-and-forget 👀 add can race `ReactionGuard` cleanup, leaving a cosmetic stale reaction | `lib.rs:2204-2208` |
| Native steer accepted but event already drained ⇒ possible duplicate delivery, logged as a warning that should never fire | `lib.rs:2840-2852` |
| `fit_observer_event_to_budget` accepts a redundant serialize on the common path because fixing it would require changing `buzz-core`'s `encrypt_observer_payload` signature | `lib.rs:651-658` |
| `desired_model` does not track `session/set_model`, so the harness's model view can be stale | `lib.rs:3188-3195` |
| Subcommands must be argv[1]; `buzz-acp --verbose models` unsupported | `lib.rs:52-56` |

#### Fragile coupling with no enforcement

| Coupling | Sites | Failure mode if broken |
|---|---|---|
| `OBSERVER_CHUNK_MAX_TEXT_BYTES` (60,000) must stay under `OBSERVER_MAX_PLAINTEXT_LEN` in `buzz-core/src/observer.rs:25` | `lib.rs:539-544` | frames silently dropped by `encrypt_observer_payload` instead of trimmed |
| Requeue must precede `mark_complete` or `retry_counts` is cleared and backoff/dead-lettering is defeated | `lib.rs:3048-3051`, mirrored `lib.rs:3427` | every retry restarts at attempt 1; infinite retry |
| Native steer framing must match cancel+merge framing | `lib.rs:2812-2819` ↔ `queue::native_steer_framing` | agent gets inconsistent orientation depending on transport |
| `crash_history.len()` must equal `config.agents`, not `pool.live_count()` | `lib.rs:1684-1695`, indexed `lib.rs:1774`, `3266`, `3465` | index-out-of-bounds panic |
| `respawn_tx` capacity must equal slot count for `RespawnGuard::send`'s `try_send` to be infallible | `lib.rs:1613`, `lib.rs:1183-1187` | slot lost forever (guard logs `error!` and falls through to `Drop`) |
| `is_auth_error` matches substrings of vendor CLI error text | `lib.rs:3003-3011` | an upstream wording change silently converts non-retryable auth failures back into 10 futile retries |
| `protocolVersion` missing ⇒ silently treated as legacy v1 | `lib.rs:3785`, `lib.rs:3864` | wrong prompt composition (`prepend_base_for_legacy`) |
| `agentInfo`/`serverInfo` precedence differs between `normalized_agent_name` (`lib.rs:3688`) and `run_models` (`lib.rs:4062`) | | agent that sets both reports two different names |

None of these couplings has a dedicated regression test. The closest is `hard_timeout_recently_active_budget_exhausted_reports_dead_lettered` (`lib.rs:5727`), which exercises dead-lettering *through* the requeue path but would still pass if the requeue/`mark_complete` order were inverted, since it pre-seeds `retry_counts` via `set_retry_count_for_test`. The constant coupling, the `crash_history` sizing invariant, and the framing-drift coupling rely on comments alone.

#### Doc drift

| Claim | Doc | Code |
|---|---|---|
| `BUZZ_ACP_IDLE_TIMEOUT` default `620` | `README.md § Core` | `DEFAULT_IDLE_TIMEOUT_SECS = 900` (`config.rs:27`) |
| `BUZZ_API_TOKEN` is a working setting | `README.md § Core` | never read (`config.rs:718` only) |
| Heartbeat calls `get_feed_actions()` / `get_feed_mentions()` | `README.md § Heartbeat Semantics` | prompt says `buzz feed get --types needs_action` / `--types mentions` (`lib.rs:3620-3628`) |
| Default kinds are "9, 46010, 40007" | `README.md § Forum Channels` | matches `lib.rs:1447-1449` ✓ |
| Typing indicators are Redis sorted sets (`ZADD buzz:typing:{channel_id}` + `ZREMRANGEBYSCORE` + `EXPIRE`) | `ARCHITECTURE.md:452-457` | typing is a kind 20002 ephemeral Nostr event published by the harness (`lib.rs:2332-2340` → `relay.rs:842-870`, `KIND_TYPING_INDICATOR` = 20002 at `buzz-core/src/kind.rs:331`). The ZADD/ZREM/EXPIRE design at `ARCHITECTURE.md:454-456` does not exist in `buzz-acp`; `ARCHITECTURE.md:824` separately and correctly describes 20002 delivery via fan-out + pub/sub, so the document contradicts itself. |
| Harness auto-injects `BUZZ_RELAY_URL` / `BUZZ_PRIVATE_KEY` / `BUZZ_AUTH_TAG` into managed agent subprocesses | `AGENTS.md § Agent CLI` | true but imprecise. The explicit injection at `lib.rs:4141-4184` targets the **MCP server** declaration inside `session/new`, delivered over the agent's stdin pipe — not the ACP child's environment. The ACP child receives them only by ordinary environment inheritance (`acp.rs:456-461`), and only if the harness itself was launched with them set. |
| `models` / `auth-methods` / `authenticate` subcommands | not documented anywhere | `lib.rs:1245-1274`, handlers `lib.rs:3899`, `3947`, `4005` |
| 19 env vars including `RESPOND_TO`, `PERMISSION_MODE`, `LAZY_POOL`, `AUTH_TAG` | absent from `.env.example` | declared in `config.rs` / read in `lib.rs` — see the Configuration aspect for the full list |
| `.env.example:152` documents `BUZZ_ACP_TURN_TIMEOUT` | | that flag is `hide = true` and deprecated (`config.rs:274`) |

#### Test coverage

`lib.rs` carries ~2,406 lines of tests across 11 in-file `#[cfg(test)]` modules (lines 3588–3608 and 4186–6570), against ~4,164 lines of production code — roughly 37 % test.

`crates/buzz-acp/tests/` holds a single 159-byte file, `pool_lifecycle_state.rs`. There is effectively **no integration test surface** for a crate that spawns subprocesses, speaks WebSocket + HTTP to a relay, and runs a twelve-arm event loop.

Covered well:

| Area | Tests |
|---|---|
| Author gate incl. DM hardening | `author_gate_tests` `lib.rs:4370-4740` — 14 cases including `test_dm_rejects_stranger_under_anyone`, `test_dm_nobody_rejects_even_owner`, `test_discovery_without_metadata_stays_fail_closed_at_author_gate` |
| Outcome → retry/dead-letter matrix | `error_outcome_emission_tests` `lib.rs:5085-6321` — one `turn_error` per error outcome, `hard_timeout_recently_active_requeue_success_reports_requeued_for_retry`, `hard_timeout_recently_active_budget_exhausted_reports_dead_lettered`, `auth_error_dead_letters_immediately_without_requeueing`, `cancel_drain_timeout_requeues_batch_and_does_not_return_agent` |
| Observer trimming | `observer_payload_trim_tests` `lib.rs:6324-6570` — 8 cases including UTF-8 boundary, stub termination, multi-block header survival |
| Observer pacing + snapshot/live race | `lib.rs:4742-4838` with `start_paused` virtual time |
| Chunk coalescing | `lib.rs:4841-4936` |
| MCP env construction | `build_mcp_servers_tests` `lib.rs:4939-5082` |
| Owner control command parsing + mode gate | `lib.rs:4219-4344` |

Untested in `lib.rs`:

| Gap | Why it matters |
|---|---|
| `SlotCircuit` state machine | No test for `record_crash`, `can_refill`, half-open pre-seeding, backoff math, or `mark_spawn_failed`. The breaker is the sole guard against respawn storms and its half-open pre-seed logic is duplicated in two methods (`lib.rs:1064-1074`, `1113-1131`). |
| `RespawnGuard::Drop` | The documented failure mode is "silently losing the slot forever" (`lib.rs:1168-1171`); nothing verifies `Drop` actually emits the fallback. |
| Steer ack decision table | The six-way `(release, drop, fallback)` matrix at `lib.rs:2474-2486` — including the `-32601` special case — is inline in the `select!` arm, not extracted into a testable function. `try_native_steer` has no test. |
| Membership dedup | Two-generation rotation, the strict-`<` watermark, and the drain/invalidate/un-react sequence (`lib.rs:1861-1949`) are all inline in the loop with no test. |
| Lazy-pool wake path | `PoolEvent::Wake`, stale-attempt discard (`lib.rs:2505-2509`), the busy-spin guard on `retry_at` (`lib.rs:1735-1742`), and the drain-not-abort shutdown (`lib.rs:2566-2603`) are untested. The single file in `tests/` is named for this area but is 159 bytes. |
| Shutdown sequence | Four-stage reaping (`lib.rs:2605-2672`) has no test; the 30 s grace and abort fallback are unverified. |
| `run_models` / `run_auth_methods` / `run_authenticate` | No tests; all three `std::process::exit(1)` on failure, which is itself untestable in-process. |
| `is_dm_channel` under lazy REST failure | Covered (`lib.rs:4720-4739`), but via a raw `TcpListener` stub — no coverage of a 5xx or a slow-loris response against the 2 s gate timeout. |
| `Config` literals | Two hand-maintained full-struct literals in tests (`lib.rs:4946-4990`, `lib.rs:5097-5151`) must be edited for every new field — a recurring merge-conflict and drift source. |

#### Other observations

- `Box::leak` on a user-supplied base prompt (`lib.rs:1545`) intentionally leaks the string for `'static` lifetime. Bounded (one per process) but a deliberate leak.
- Jitter for respawn backoff is derived from `SystemTime` subsec nanos (`lib.rs:1091-1097`), not a PRNG. Slots crashing within the same nanosecond window receive correlated jitter, weakening the thundering-herd protection the jitter exists to provide.
- `handle_prompt_result` recomputes `agent_index` twice (`lib.rs:3048` and `lib.rs:3191`) and `let mut hard_timeout_fate_suffix` threads a `&'static str` through six branches to reconstruct a message string 150 lines later (`lib.rs:3060`, consumed `lib.rs:3254`) — a sign the function wants splitting.
- Three `debug_assert`s (`lib.rs:2531`, `lib.rs:3043`, and the `try_send` contract) encode invariants that vanish in release builds.


## Module: buzz-acp — relay client & observer (`crates/buzz-acp/src`)
### Aspect: Debt

#### File size

| File | Lines | Production | Tests |
|---|---|---|---|
| `relay.rs` | 6,233 | 1–3,994 (3,994) | 3,995–6,233 (2,239, 36 %) |
| `engram_fetch.rs` | 248 | 1–166 | 167–247 (81) |
| `observer.rs` | 166 | all | none |

`relay.rs` at 6,233 lines is a single module holding: the harness handle, the
HTTP bridge client, the wire-format parser, the NIP-42 handshake, the background
task, `BgState`, four drain loops, three reconnect loops, the terminal-error
classifier, and 70 tests. The mobile guard enforces a 1,000-line ceiling
(`mobile/scripts/check-file-sizes.mjs`, per `AGENTS.md`); no equivalent guard
covers `crates/`, so this file has grown past 6× that without tripping anything.

Natural seams already exist and are unused: `RestClient` (`relay.rs:231-424`,
~194 lines) has no dependency on `WsStream` and is doc-commented as
"Extracted from `HarnessRelay` fields so it can be shared" (`relay.rs:227-229`)
— it is a self-contained HTTP client living in a WebSocket file.
`parse_relay_message` + `RelayMessage` + `OkResponse` (`relay.rs:471-495`,
`:3531-3620`, `:3917-3921`) are a self-contained codec.
`is_terminal_connect_error` and its three helpers (`relay.rs:3625-3775`,
~150 lines) are a pure classifier with 40 lines of doc comment.

#### Reimplementation of `buzz-ws-client`

`crates/buzz-acp/Cargo.toml` does not depend on `buzz-ws-client`. The full
side-by-side duplication table is in `acp-relay-integrations.md`; the summary:

| Duplicated behaviour | `relay.rs` | `buzz-ws-client/src/connection.rs` |
|---|---|---|
| `WsStream` alias | `:525` | `:14` |
| URL parse + `connect_async` | `:3830-3837` | `:46-52` |
| `debug!("connected to relay at {url}")` | `:3838` | `:57` |
| `RelayMessage` enum | `:470-495` | `message.rs:8-41` |
| `OkResponse` | `:3917-3921` | `message.rs:44-52` |
| Frame parser | `:3531-3620` | `message.rs:55-144` |
| AUTH-challenge wait | `:3865-3913` | `:156-206` |
| AUTH OK wait | `:3924-3982` | `:208-263` |
| Build + send AUTH | `:3433-3461` | `:70-93` + `message.rs:151-166` |
| Ping → Pong in read loop | `:2370-2376`, `:3903-3907`, `:3963-3967` | `:148-150`, `:208-210`, `:262-264` |
| Frame buffering `VecDeque` | `:3833`, `:3902`, `:3960` | `:28`, `:205`, `:257` |
| 20 s AUTH timeouts | `:64` | `:17`, `:20` |

Roughly 300 lines of duplicated protocol code with three behavioural
divergences: mid-session AUTH (re-auth vs. record-only), AUTH-OK matching
(any-OK vs. id-matched), and the 1024-byte challenge cap (absent vs. present).
Two of the three make this copy weaker. The one place `relay.rs` is stronger —
not logging the AUTH frame — means neither copy can simply adopt the other.

`buzz-ws-client` pins its timeout floors with compile-time tests
(`buzz-ws-client/src/connection.rs:297-313`). `relay.rs` has no equivalent for
`AUTH_TIMEOUT` or `CONNECT_TIMEOUT`, so the two can drift silently.

#### Structural duplication inside the file

The membership `since` computation `match (dropped_since, last_seen)` is written
out four times, identically:

| Site | Location |
|---|---|
| `execute_connected_command` | `relay.rs:1421-1428` |
| main-loop drain | `relay.rs:1721-1728` |
| CLOSED handler | `relay.rs:2295-2301` |
| `resubscribe_after_reconnect` | `relay.rs:2569-2574` |

`channel_since` (`relay.rs:1115-1131`) exists as the channel-side equivalent; no
`membership_since` was extracted.

The reconnect-then-drain-then-fallback block is duplicated five times in
`run_background_task`, each ~40 lines of nested `match` on `ReconnectOutcome`
followed by four state resets (`ping_sent`, `last_pong`, `connected_since`,
`stable_logged`):

| Trigger | Location |
|---|---|
| Handshake-buffer drop signal | `relay.rs:1697-1750` |
| Proactive resubscribe failure | `relay.rs:1600-1652` |
| Socket lost on read | `relay.rs:1766-1830` |
| Command send failure | `relay.rs:1804-1852` |
| Ping/pong death | `relay.rs:1861-1927` |

The `select!` body in that function shows the resulting formatting damage:
indentation is visibly inconsistent from `relay.rs:1702` onward, with argument
lists broken across columns (`relay.rs:1776-1780`, `:1836-1840`) — the block is
past what `rustfmt` can lay out cleanly.

Two shutdown-aware sleeps differ only in what they do with a non-`Shutdown`
command: `pacing_sleep` defers it (`relay.rs:3369-3392`), `dns_flat_sleep`
applies it to state (`relay.rs:3395-3411`). The two inline backoff sleeps
(`relay.rs:2996-3008`, `:3133-3145`) re-implement `dns_flat_sleep`'s body
without the jitter.

Three near-identical REQ builders: `send_subscribe` (`relay.rs:3160-3222`),
`send_membership_subscribe` (`:3227-3270`), `send_observer_control_subscribe`
(`:3273-3305`). Each hand-builds a `serde_json::Map` or inline `json!`, each
repeats the same `serde_json::to_string` → `ws_send_timeout` → log-on-error
tail. `nostr::Filter` is used for HTTP queries but not for WS REQs, so filter
construction is done two different ways in the same file.

#### Known correctness gaps left in place

| Gap | Evidence |
|---|---|
| AUTH OK is not matched to the AUTH event id | `relay.rs:3846-3854` — the comment describes re-deriving the id as "wasteful" and settles for the first OK. An unrelated `OK` arriving first is accepted as the auth result. |
| No challenge length cap on any of three intake paths | `relay.rs:3901`, `:3610-3616`, `:2344` |
| No `ws://` scheme rejection | `relay.rs:3830-3832` |
| EOSE is inert | `relay.rs:2190-2192` — no initial-replay-complete signal |
| Wildcard REQ omits `kinds`, tripping the relay p-gate | `relay.rs:3172-3175`; reachable via `config.rs:1180`, `:1272` (`--subscribe-mode all` with no `--kinds`) and `config.rs:1195-1196`, `:1286-1287` (empty-`kinds` rule). `AGENTS.md` states omitting `kinds` returns 403. No guard, no warning. |
| Dedup amnesia window | `relay.rs:925-933` documents that an id seen 6,001+ inserts ago may be replayed as new |
| `send_subscribe` takes an unused `&BgState` | `relay.rs:3162`, threaded through six call sites |
| `#[allow(dead_code)] pub async fn publish_event` | `relay.rs:820-821` — no in-repo caller |
| Full unparsed frame logged on parse failure | `relay.rs:2059` |
| In-flight-batch / re-add race | documented at `relay.rs`'s caller (`lib.rs`), acknowledged as accepted rather than fixed |
| `RelayError::Http` used as a catch-all | URL parse (`relay.rs:3831`, `:3440`), tag parse (`:3446`, `:3448`), reqwest build (`:594`), NIP-98 signing (`:272`, `:294`, `:296`) all collapse into one variant |
| `nip98_header(...).unwrap_or_default()` sends an empty `Authorization` on signing failure | `relay.rs:379`, justified as "infallible in practice" (`:377-378`) |
| `wait_for_reconnect` ladder is an inline array | `relay.rs:3055-3062` — cannot be asserted against `STARTUP_CONNECT_BACKOFFS` |

#### Repo-rule compliance

| Rule | Status |
|---|---|
| No `unsafe` | Clean. `#![deny(unsafe_code)]` at `lib.rs:1`; no `unsafe` in any of the three files. |
| No new `unwrap()`/`expect()` in production paths | Clean. `relay.rs` production half (1–3,994): **0** occurrences — first is `relay.rs:4040`, inside `mod tests`. `observer.rs`: **0** anywhere. `engram_fetch.rs`: **9**, all in `mod tests` (`:178`, `:190`, `:191`, `:192`, `:212`, `:213`, `:232`, `:233`, `:234`). |
| New public API must have doc comments | Clean across all three files. |

`unwrap_or_default()` on `SystemTime` arithmetic appears five times
(`relay.rs:257`, `:3190`, `:3245`, `:3282`, `:3341`) — not a rule violation, but
it silently maps a pre-epoch clock to `since = 0`, which would request the
relay's full history.

#### TODO / FIXME / HACK / XXX

**Zero** across all three files. The known gaps are instead written as prose in
doc comments — e.g. `relay.rs:3846-3854` (AUTH-OK matching), `:925-933` (dedup
amnesia), `:1487-1493` (the unenforced "only ephemeral kinds on this path"
invariant), `:2481-2485` (gate survives socket replacement). This makes them
invisible to any tooling that greps for markers, and several read as
justifications rather than open items.

#### Test coverage

`relay.rs` `mod tests` (`:3995-6233`) contains **80 test functions**: 62
`#[test]` and 18 `#[tokio::test]` (10 of those with `start_paused = true`), plus
7 helpers (`meta_event` `:4070`, `make_test_event` `:4331`, `test_ws_pair`
`:4340`, `next_test_frame` `:4357`, `test_channel_filter` `:4369`,
`seed_test_subscription` `:4376`, and the nested `ws` shim at `:5079`).

Covered well:

| Area | Representative tests |
|---|---|
| URL / sub-id helpers | `:3999-4068` (10 tests) |
| `merge_discovered_channels` incl. archived | `:4087-4149` (5 tests) |
| `parse_relay_message` all variants + malformed | `:4151-4311` (14 tests) |
| Replay pacing with live socket | `:4388-4536` (4 async tests) |
| `TwoGenDedup` + `last_seen` | `:4566-4753` (9 tests) |
| Backpressure `dropped_since` bookkeeping | `:4755-4925` (5 tests) |
| CLOSED access-denial exact matching | `:4941-5071` (6 tests) |
| Terminal-error classification (exhaustive, `:5073`) | `:5073-5561` |
| `retry_initial_connect` incl. sleep assertions | `:5563-5702` (5 async tests) |
| Rate-limit gate + hint parsing | `:5704-5786` (6 tests) |
| Gated observer queue, ordering, overflow | `:5788-5996` (5 tests) |
| DNS classification | `:5998-6034` |
| Drain / resub retry paths | `:6036-6232` (7 tests) |

Not covered:

| Gap |
|---|
| `do_connect`'s happy path — no test drives a full AUTH challenge → AUTH event → OK handshake against the `test_ws_pair` fixture. The only `do_connect` test is the wrong-scheme terminal case (`:5549`). |
| Mid-session AUTH re-authentication (`relay.rs:2344-2353`) — the divergent behaviour has no test. |
| `wait_for_any_ok`'s accept-any-OK semantics — untested, so the known gap is unpinned. |
| `wait_for_reconnect` (`relay.rs:3022-3151`) — the unbounded loop with the 60 s tail and persisted `backoff_step`; `:2029-2038`'s stability reset is also untested. |
| `process_handshake_buffer` (`relay.rs:2393-2450`) — including the deliberate AUTH-skip at `:2433`. |
| EOSE handling. |
| `send_subscribe` with `kinds: None` — the p-gate-tripping wildcard REQ has no test asserting the emitted frame shape. |
| `RestClient` — `sign_nip98`, `nip98_header`, `request_with_retry`, `bridge_post`, `query`, `submit_event` (`relay.rs:261-424`, ~164 lines) have **no tests in this module**. `discover_channels` (`:657-714`) is likewise untested; only its pure tail `merge_discovered_channels` is. |
| `observer.rs` — **zero tests in the file**. `emit`/`snapshot`/`subscribe` are exercised indirectly by `lib.rs`'s `observer_snapshot_race_tests` and `observer_publish_pacer_tests`, but the 1,000-entry drop-front eviction and the poisoned-mutex degradation paths (`observer.rs:96-100`, `:129-131`) are not asserted anywhere. |

`engram_fetch.rs` has 5 tests (`:167-247`) and they cover the decision table
well — confirmed absence (`:177`), happy path (`:186`), the undecryptable →
`Err` regression (`:206`), non-`Core` body → absent (`:229`), unparseable →
`Err` (`:241`). `fetch_core_body`'s filter construction and `build_core_section`'s
rendering are untested (no HTTP fixture), so the `[Agent Memory — core]` header
format and the `limit(16)` are unpinned.

#### Documentation drift

| Claim | Reality |
|---|---|
| `relay.rs:5` "discovers channels via REST API" | Uses the Nostr HTTP bridge `POST /query` (`relay.rs:399-406`, `:670`, `:705`), not a REST resource API. Same wording at `relay.rs:230` ("Lightweight HTTP client … via the Nostr HTTP bridge") is accurate — the header is stale. |
| `relay.rs:27-28` / `.env.example:219-220` — `BUZZ_ACP_EVENT_BUFFER` sizes "the event channel" | Sizes two channels: `event_tx` and `observer_control_tx` (`relay.rs:610-612`) |
| `ARCHITECTURE.md:452-458` — typing indicators use a Redis sorted set with a 5 s window and 60 s TTL | The kind:20002 event built at `relay.rs:866-868` and published from `lib.rs:2333-2341` is handled by the generic ephemeral (20000–29999) fan-out; `grep KIND_TYPING_INDICATOR crates/buzz-relay/src crates/buzz-pubsub/src` returns nothing |
| `relay.rs:1341-1343` — `execute_connected_command` "handles the five data commands" | It handles six: `Subscribe`, `Unsubscribe`, `SubscribeMembership`, `SubscribeObserverControls`, `PublishEvent`, `SetStartupWatermark` (`relay.rs:1352-1527`) |
| Kind numbers repeated in prose | `relay.rs:162` (39000), `:262` (27235), `:653-654` (39002/39000), `:842` (20002), `:1036`/`:1469` (24200), `:3225` (44100+44101) — each is a drift surface independent of `buzz-core/src/kind.rs` |
| `relay.rs:3489-3496` names `req.rs:153` and `side_effects.rs:71` as the only CLOSED senders | A cross-crate coupling asserted in a comment with no test or compile-time link |
| `relay.rs:3762-3763` — the rustls downcast "relies on a single rustls version in the dep tree (0.23.40)" | `crates/buzz-acp/Cargo.toml` pins `rustls = { version = "0.23", … }` — a caret range, so a patch bump satisfying `0.23` could break the downcast without a version-conflict error |


## Module: buzz-acp — agent pool & lifecycle (`crates/buzz-acp/src`)
### Aspect: Debt

#### File and function size

`pool.rs` is 5,620 lines: 3,649 lines of production code and 1,970 lines of in-file tests (`#[cfg(test)] mod tests` opens at `pool.rs:3650`). Within it, `run_prompt_task` is a single 948-line function (`pool.rs:1265-2212`) with 20 `send_prompt_result` exit sites and roughly ten nesting levels at the deepest point (the `Err(error)` arm of the control-cancel branch nested inside a `select!` inside a `match` inside `run_prompt_task`, `pool.rs:1894-1930`). The invariant "every path must send exactly one `PromptResult` or the main loop deadlocks" is stated in a comment (`pool.rs:1945`) and enforced by review, not by types — a `#[must_use]` result token or a consuming builder would make it structural.

The module's responsibilities are not separable as written. Beyond pooling, `pool.rs` owns: prompt-section composition (`pool.rs:1090-1231`), five relay REST fetchers (`pool.rs:2237-2784`), Nostr event parsing and signature verification for canvas (`pool.rs:2366-2477`), kind-0 profile parsing (`pool.rs:2594-2630`), NIP-AM metric encryption and publishing (`pool.rs:3322-3430`), NIP-25/NIP-09 reaction management (`pool.rs:3462-3648`), and a kind-9 dead-letter notice publisher (`pool.rs:3495-3535`). Nothing named `pool` is required for most of that.

#### The 4-line test file

`crates/buzz-acp/tests/pool_lifecycle_state.rs` contains no assertions. It is a re-inclusion shim:

```
#[allow(dead_code)]
#[path = "../src/pool_lifecycle.rs"]
mod pool_lifecycle;
```

(`tests/pool_lifecycle_state.rs:1-4`). Because integration targets compile with `cfg(test)`, this makes the nine tests already inside `pool_lifecycle.rs` (`:139`, `:150`, `:174`, `:203`, `:226`, `:244`, `:269`, `:283`, `:293`) compile and run a **second** time in a second binary. It adds zero coverage over `cargo test -p buzz-acp --lib`, doubles their runtime, and requires a blanket `#[allow(dead_code)]` because most of the state machine is unused from an integration target's perspective. It is also the crate's **only** file under `tests/` — `crates/buzz-acp/tests/` contains nothing else.

#### Untested surface

The pool's actual pooling logic has **no tests in `pool.rs`**. Searching the test region (`pool.rs:3650-5620`) for each `AgentPool` method returns zero matches for: `try_claim`, `return_agent`, `live_count`, `slot_alive`, `any_idle`, `has_session_for`, `send_steer`, `switch_idle_agent_model`, `invalidate_channel_sessions`, `from_slots`, and `IdleSwitchResult`. Specifically untested behaviours:

| Untested | Site |
|---|---|
| Affinity pass wins over first-idle | `pool.rs:560-568` |
| `try_claim` returns `None` when all checked out | `pool.rs:571-574` |
| Double-return `BUG:` branch and slot overwrite | `pool.rs:579-589` |
| `live_count` = idle + checked-out | `pool.rs:610-614` |
| `slot_alive` true for a checked-out agent | `pool.rs:686-693` |
| `send_steer` → `PromptCompleted` when no task in flight | `pool.rs:650-652` |
| `send_steer` → `Transport` on `Full`/`Closed` | `pool.rs:657-659` |
| All three `IdleSwitchResult` outcomes | `pool.rs:732-762` |
| `invalidate_channel_sessions` return count | `pool.rs:707-720` |
| `from_slots` index-preservation invariant | `pool.rs:541-556` |

`from_slots` is used 14 times in `lib.rs` tests, so it is exercised incidentally as a fixture; the invariant it exists to protect is never asserted. `run_prompt_task` itself has no test — every test in `pool.rs` targets a pure helper, a guard, or `SessionState`.

Lifecycle-state coverage is better but incomplete. Covered transitions: `Listening → Waking`, `Waking → Ready`, `Waking → Failed`, `Failed → Waking` (due and not-due), `Ready → Listening` via `take_ready`, stale-attempt rejection, non-`Waking` `complete_wake` rejection, `cancel_wake`. Not covered:

- `cancel_wake` with a **mismatched** attempt returning `false` (`pool_lifecycle.rs:91-93`) — only the matching case is tested (`:271`).
- `Ready` → `start_wake_if_due` returning `None` is asserted only after a retry cycle (`:218-221`), never directly from a first-attempt success.
- `take_ready` on `Waking` or `Failed` returning `None` without mutating state (`pool_lifecycle.rs:63-66`) — only the `Ready`-then-empty sequence is tested (`:288-289`).
- `waking_attempt()` / `retry_at()` / `failed_error()` returning `None` in non-matching states (`pool_lifecycle.rs:70-89`).
- The `Failed` → `start_wake_if_due(true, now)` path when `now` is exactly `retry_at - 1ns`.
- Interaction with the caller's `pool_ready` flag: nothing tests that a `Listening` state after `take_ready` cannot start a second wake, because that suppression lives in `lib.rs:1714`, not in the machine.

#### Dead code and stale annotations

Three `#[allow(dead_code)]` attributes in `pool.rs`:

| Site | Target | Status |
|---|---|---|
| `pool.rs:73` | `AgentModelCapabilities` | **Stale-ish and mis-documented.** Both fields *are* read in-process at `pool.rs:750-751`. The comment says "Scaffolding for desktop integration — fields read via serde", and the type doc (`pool.rs:70-72`) says they are "read by the desktop's `get_agent_models` Tauri command (Phase 3)". That command is an out-of-process Tauri handler (`desktop/src-tauri/src/commands/agent_models.rs:29`) that cannot read this struct, and neither field derives `Serialize`. |
| `pool.rs:335` | `SteerError` | Explained: variants are `Debug`-rendered through `?ack` in `lib.rs` and the lint cannot follow that (`pool.rs:329-334`). |
| `pool.rs:405` | `PromptOutcome` | Unexplained. All six variants are constructed and matched; the payload of `Ok(_)` is never read at either `lib.rs` match arm (`lib.rs:3183`, `:3231`), which is the likely reason. |

Three `#[cfg(test)]` helpers sit in the production region rather than in the test module: `parse_thread_response` (`pool.rs:2787`), `parse_dm_response` (`pool.rs:2828`), `pct_encode` (`pool.rs:3441`). The first two parse a **legacy REST shape** that production no longer requests — production uses `parse_nostr_thread_response`/`parse_nostr_dm_response` (`pool.rs:2892`, `:2938`) — yet nine tests still assert against the retired format (`pool.rs:3879`, `:3916`, `:3951`, `:3961`, `:3968`, `:4007`, `:4034`, `:4063`, `:4072`). `pct_encode` has six tests (`pool.rs:4213`–`:4241`) and zero production callers; the URL-path encoding it existed for is gone now that reactions go through signed events.

`PromptContext::heartbeat_prompt` (`pool.rs:501`) is carried on the shared context but never read in `pool.rs`. `PoolLifecycle::failed_error()` (`pool_lifecycle.rs:84`) has one caller, a `debug_assert_eq!` (`lib.rs:2565`), so it is dead in release builds.

#### TODO / FIXME / unsafe / unwrap

- `TODO`, `FIXME`, `XXX`, `HACK`: **0** occurrences across both files.
- `unsafe`: **0**; `#![deny(unsafe_code)]` is crate-wide (`lib.rs:1`).
- `unwrap()`/`expect()` in production paths: **3** total. `pool.rs:573` (`take().unwrap()` guarded by a preceding `position(..is_some())`) and `pool.rs:3399-3400` (two `Tag::parse(..).expect(..)` on hex strings). The remaining 93 of the file's 96 occurrences are inside `#[cfg(test)]`.

#### Doc-comment corruption

Three confirmed doc-block splices, each of which mis-attributes documentation and leaves a public item undocumented:

1. `PromptContext`'s doc block is glued to the front of `ChannelInfoResolver`'s: `pool.rs:426-429` reads "Immutable config subset shared (via `Arc`) by all spawned prompt tasks… Avoids cloning the full config into every task." immediately followed at `pool.rs:430` by "Shared channel-metadata resolver…", and the actual `pub struct PromptContext` at `pool.rs:482` has **no doc comment at all**.
2. `apply_permission_mode`'s doc is split across two items: the sentence begins at `pool.rs:1008-1011` ("…falls back to its default permission mode (`"default"`), which still works via") and is interrupted by `agent_supports_mode`'s doc + body (`pool.rs:1012-1026`); the continuation "per-tool auto-approval in `handle_permission_request`." opens the function's own doc block mid-sentence at `pool.rs:1028`.
3. `parse_dm_response` carries two contradictory summary lines: "Parse the DM messages REST response into a `ConversationContext::Dm`." followed by "Parse the legacy REST DM response (used in tests only)." (`pool.rs:2824-2826`).

Other public-item doc gaps under the repo's "new public API must have doc comments" rule: `ChannelInfoResolver::new` (`pool.rs:442`), `ChannelInfoResolver::resolve` (`pool.rs:464`), `AgentPool::task_map` (`pool.rs:616`), `task_map_mut` (`pool.rs:620`), `result_tx` (`pool.rs:664`), `agents_mut` (`pool.rs:695`).

#### Doc drift vs `ARCHITECTURE.md` and `README.md`

`ARCHITECTURE.md:658-667` lists a per-module LOC table for `buzz-acp` that no longer matches any file:

| `ARCHITECTURE.md` claim | Actual | Delta |
|---|---|---|
| `pool.rs` — 2,253 | 5,620 | +150 % |
| `main.rs` — 2,457, "Event loop, pool orchestration, heartbeat" | 3 lines; the event loop is in `lib.rs` (6,570) | wrong file entirely |
| `relay.rs` — 3,143 | 6,233 | +98 % |
| `queue.rs` — 2,565 | 4,759 | +85 % |
| `config.rs` — 1,903 | 2,709 | +42 % |
| `acp.rs` — 1,785 | 3,717 | +108 % |
| `filter.rs` — 814 | 787 | −3 % |

`pool_lifecycle.rs`, `setup_mode.rs`, `usage.rs`, `engram_fetch.rs`, and `observer.rs` are absent from that table. `ARCHITECTURE.md:668-672` also omits the deferred/lazy pool entirely — "Pool of 1–32 agent subprocesses with claim/return lifecycle" and "Crash recovery: agent subprocess crashes are detected and the agent is respawned" describe only the eager path, with no mention of `--lazy-pool`, the wake state machine, the per-slot circuit breaker, or the `CancelDrainTimeout` class.

`crates/buzz-acp/README.md` is accurate on the pool: `--agents` range and default (`README.md:117`), `--lazy-pool` including the retry-with-backoff behaviour (`README.md:118`), single shared bot identity for all N agents (`README.md:197`), heartbeat dropped when all agents are busy (`README.md:204`), one prompt in flight per channel (`README.md:251`). It does not document the steer/interrupt/rotate/switch-model control signals or the circuit breaker.

`.env.example` drift is covered in the Configuration aspect: eight pool-relevant `BUZZ_ACP_*` vars are missing and the deprecated `BUZZ_ACP_TURN_TIMEOUT` is documented in place of `BUZZ_ACP_IDLE_TIMEOUT` (`.env.example:152`).

#### Structural risks carried in comments rather than code

- `return_agent` knowingly overwrites an occupied slot and leaks the resident `AcpClient` rather than leaking the slot (`pool.rs:579-589`). Both outcomes are bad; the choice is documented but not measured — nothing counts how often the `BUG:` branch fires.
- `task_map_mut` and `agents_mut` expose the invariant-bearing structures for unrestricted mutation (`pool.rs:620`, `:695`), so the index invariant is `lib.rs`'s responsibility.
- `result_tx` is unbounded (`pool.rs:542`); safe only because sends are gated by pool width. A future non-agent sender would remove that bound silently.
- `agent_name == "goose"` is a string comparison used as a capability switch in three places (`pool.rs:176`, `:826`, and via `has_system_prompt_support`), rather than a negotiated capability.
- `ctx.cwd` defaults to `/` when `current_dir()` fails (`lib.rs:1547-1550`); the only mitigation is that `workspace_section` declines to name `/` in the prompt (`pool.rs:1165-1178`). The `session/new` `cwd` parameter is still sent as `/`.
- `buzz-persona` is declared as a dependency (`Cargo.toml:22`) with zero references in `crates/buzz-acp/src` — a dependency edge with no code behind it. If persona-driven spawning is intended to land here, the seam does not exist yet: `PoolStartup` carries one command/args/env triple for the whole pool (`lib.rs:3717-3725`) and `ctx.mcp_servers` is a single shared list built only from `config.mcp_command` (`lib.rs:4141-4184`).


## Module: buzz-acp — work queue, filtering & usage accounting (`crates/buzz-acp/src`)
### Aspect: Debt

Zero `TODO` / `FIXME` / `HACK` / `XXX` markers across all three files — the debt below is structural, not annotated.

#### File and function size

| File | Lines | Non-test lines | Test lines |
|---|---|---|---|
| `queue.rs` | 4,759 | 1,627 (`:1-1627`) | 3,132 (`:1628-4759`, 66 %) |
| `filter.rs` | 787 | 461 (`:1-461`) | 326 (`:462-787`) |
| `usage.rs` | 892 | 321 (`:1-321`) | 571 (`:322-892`, 64 %) |

`AGENTS.md` enforces a 1,000-line-per-file ceiling for desktop, web, and mobile (`mobile/scripts/check-file-sizes.mjs` via `just mobile-check`) with **no Rust equivalent**. `queue.rs` is 4.8× that ceiling; even its non-test half exceeds it by 60 %.

Longest functions:

| Function | Lines | Span |
|---|---|---|
| `format_prompt` | 159 | `queue.rs:1406-1564` |
| `flush_next` | 121 | `queue.rs:260-380` |
| `record` (UsageTracker) | 92 | `usage.rs:211-302` |
| `match_event` | 92 | `filter.rs:368-459` |
| `format_context_hints` | 83 | `queue.rs:1233-1315` |
| `build_eval_context` | 75 | `filter.rs:264-338` |
| `format_event_block` | 72 | `queue.rs:1076-1147` |
| `requeue` | 70 | `queue.rs:429-498` |

`flush_next` does four distinct jobs in one body — expire stuck in-flight channels, pick a fair candidate, handle the cancelled-only fallback with an early `return`, then drain/sort/merge — with a shadowed `channel_id` binding across the fallback `match` (`queue.rs:291`, `:305`).

#### Duplicated logic

- The in-flight expiry block is copy-pasted between `flush_next` (`queue.rs:263-287`) and `has_flushable_work` (`queue.rs:558-581`): same filter, same `ERROR` message text, same three removals, same `recover_withheld_for_expired_channel` call. A change to one must be mirrored by hand; the comment at `queue.rs:559` ("same logic as flush_next") acknowledges this.
- The candidate predicate is also duplicated: `queue.rs:294-298` vs `:583-586`.
- The per-channel cap-enforcement `while` loop appears four times with four different log messages (`queue.rs:242-249`, `:486-495`, `:722-729`, `:775-782`).
- `MAX_EXPR_LEN = 4096` (`filter.rs:162`) is re-encoded as a bare literal in `load_rules` (`config.rs:1098`).
- `DEFAULT_IN_FLIGHT_DEADLINE_SECS = 7300` (`queue.rs:42`) hard-codes `DEFAULT_MAX_TURN_DURATION_SECS (7200) + IN_FLIGHT_DEADLINE_BUFFER_SECS (100)` rather than computing it.

#### Untested surface

| Item | Line | Status |
|---|---|---|
| `EventQueue::remove_event` | `queue.rs:738-751` | **zero** calls in the `queue.rs` test module — referenced only in a section comment at `queue.rs:4268`. The `SteerAck::Success` path (`lib.rs:2512`) has no unit test for its queue effect |
| `EventQueue::is_channel_in_flight` | `queue.rs:645-647` | never called in tests; the test helper `any_in_flight` reads the private field instead (`queue.rs:1687-1689`) |
| `native_steer_framing()` | `queue.rs:1623-1626` | no test; the "native and fallback must not drift" requirement (`queue.rs:1618-1621`, `lib.rs:2812-2814`) is unasserted |
| `EventQueue::pending_channels` | `queue.rs:594-596` | never asserted; tests use the private-field helper `pending_count` (`queue.rs:1683-1685`) |
| `format_context_hints` (direct) | `queue.rs:1233-1315` | only exercised transitively through `format_prompt` |
| `FilterError::Timeout` real path | `filter.rs:426-436` | no test drives an actual timeout; `test_consecutive_timeouts_disables_rule` pre-seeds the counter (`filter.rs:766-786`) |
| `MAX_CONCURRENT_FILTER_EVALS` saturation | `filter.rs:173-183` | no test; the permit-held-until-thread-finishes property documented at `filter.rs:167-172` is unverified |
| `ChannelScope` / `SubscriptionRule` deserialization | `filter.rs:55-61`, `:82-114` | no `serde` round-trip test in `filter.rs`; `#[serde(untagged)]` behaviour is only covered indirectly by `config.rs` tests |
| Jitter distribution in `requeue` | `queue.rs:457-464` | no test; `SystemTime`-nanos entropy is untestable as written |
| `queues` map growth in channel count | `queue.rs:138` | no test asserts a bound because there isn't one |

Test **count** is high (112 / 15 / 20) but skewed: 46 of the 112 `queue.rs` tests are `format_prompt`/framing assertions, while the concurrency-sensitive native-steer and expiry paths have 7 between them (`queue.rs:4278-4451`, `:4633-4759`).

#### Unbounded state (no eviction path)

| Structure | Line | Note |
|---|---|---|
| `cancelled_batches[ch]` | `queue.rs:151` | `extend`ed on every cancel (`queue.rs:543-546`) with no `MAX_BATCH_EVENTS`-style cap; flows straight into prompt size |
| `withheld_native_steer[ch]` | `queue.rs:166` | vec has no cap of its own |
| `in_flight_batch_sizes` | `queue.rs:143` | not touched by `compact_expired_state` (`queue.rs:807-818`); leaks for any channel that never completes or expires |
| `queues` (channel count) | `queue.rs:138` | drain paths prune emptied deques (`queue.rs:352-355`, `:683-685`, `:747-749`) but `compact_expired_state` never touches `queues`, so a zero-length entry created by `entry().or_default()` (`queue.rs:475`, `:510`, `:720`, `:771`) persists |
| `UsageTracker::sessions` | `usage.rs:166` | "one entry per goose `sessionId` ever seen in this process" (`usage.rs:165`) — stated as a fact, not flagged |

`compact_expired_state`'s doc comment claims it exists "to prevent unbounded map growth" (`queue.rs:791`) but only touches `retry_after` and `retry_counts` (`queue.rs:809-817`), leaving four other maps uncompacted.

#### Invariants carried only in comments

| Invariant | Where it lives | Blast radius if violated |
|---|---|---|
| `requeue()` must run **before** `mark_complete()` | comment `queue.rs:426-427`, `:386-389` and `lib.rs:3061-3065` | every retry silently resets to attempt 1; exponential backoff and dead-lettering both stop working, with no error and no log |
| `mark_native_steer_pending` must be called synchronously after `send_steer` returns `Ok` and **before** the watcher spawns | comment `queue.rs:667-671`, `pool.rs:636-640` | reopens the `mark_complete` → `flush_next` double-delivery race the side table exists to close |
| `in_flight_deadline` must be strictly greater than `max_turn_duration` | comment `queue.rs:167-169` | a turn running to the hard cap gets its channel released early and re-dispatched while still in flight. Guarded only by `default_in_flight_deadline_exceeds_default_max_turn_duration` (`queue.rs:4530`), which checks the **defaults**, not a configured value |
| Native-steer and cancel+merge framing must not diverge | comment `queue.rs:1618-1621`, `lib.rs:2812-2814` | shared via `native_steer_framing()` + `format_event_block`, but nothing asserts it |
| `drain_channel` must preserve **both** `in_flight_channels` and `in_flight_deadlines` | comment `queue.rs:636-641` | removing deadlines alone disables auto-expiry and permanently wedges the channel |

None of these is enforced by a type, a `debug_assert!`, or a builder that makes the wrong order unrepresentable.

#### Dead / vestigial code

| Item | Line | Note |
|---|---|---|
| `UsageUpdatePayload::used` | `usage.rs:78-81` | `#[allow(dead_code)]`; parsed, tested for wire compat, never read |
| `UsageUpdatePayload::context_limit` | `usage.rs:82-85` | same |
| `MatchedRule::rule_index` | `filter.rs:152-153` | `#[cfg_attr(not(test), allow(dead_code))]`; both call sites discard it (`lib.rs:2175`, `setup_mode.rs:444-451`) |
| `UsageTracker::take` | `usage.rs:303-304` | carries `#[cfg_attr(not(test), allow(dead_code))]` **despite** a live production caller at `acp.rs:784` — a stale attribute that suppresses genuine dead-code signal |
| `pub use usage::TurnUsage` | `lib.rs:15` | the crate's only public re-export, with no external consumer: `sprig` calls `buzz_acp::run()` only (`sprig/src/main.rs:17`) |
| `EventQueue::pending_channels` | `queue.rs:594-596` | production callers are log-field-only (`lib.rs:2910`, `:2975`) |

#### `unwrap` / `expect` in production paths

`AGENTS.md` forbids introducing new `unwrap()`/`expect()` in production paths. Current count in these files outside `#[cfg(test)]`:

| Site | Line | Justification present? |
|---|---|---|
| `q.front().unwrap()` | `queue.rs:299` | no comment; safe only because the enclosing `filter` excludes empty queues (`queue.rs:295`) |
| `.expect("position came from iter so remove must succeed")` | `queue.rs:681-682` | yes, in the expect message |

`filter.rs` and `usage.rs` have zero. The remaining 13 occurrences in `queue.rs`'s non-test half are `unwrap_or*` combinators, which are not in scope of the rule.

#### Documentation drift

| Claim | Reality | Evidence |
|---|---|---|
| `ARCHITECTURE.md`: `queue.rs` = 2,565 LOC | 4,759 | `ARCHITECTURE.md:661` |
| `ARCHITECTURE.md`: `filter.rs` = 814 LOC | 787 | `ARCHITECTURE.md:666` |
| `ARCHITECTURE.md`: `main.rs` = 2,457 LOC, "Event loop, pool orchestration, heartbeat" | `main.rs` is **3 lines** (`fn main() { buzz_acp::run() }`); the event loop lives in `lib.rs` (6,570) | `ARCHITECTURE.md:662`, `crates/buzz-acp/src/main.rs` |
| `ARCHITECTURE.md`: `relay.rs` = 3,143, `pool.rs` = 2,253, `config.rs` = 1,903, `acp.rs` = 1,785 | 6,233 / 5,620 / 2,709 / 3,717 | `ARCHITECTURE.md:660`, `:663-665` |
| `ARCHITECTURE.md` module table | omits `usage.rs`, `observer.rs`, `setup_mode.rs`, `engram_fetch.rs`, `pool_lifecycle.rs` entirely | `ARCHITECTURE.md:656-666` |
| `queue.rs` module header: "**Drop** (default)" | CLI default is `queue` (`config.rs:344`); only `impl Default for EventQueue` uses `Drop`, and that impl is test-only | `queue.rs:11-14` vs `config.rs:344`, `queue.rs:822-824` |
| `queue.rs:163`, `:702`, `:765`: "`requeue_preserve_timestamps` at line 453" | it is at line 508 | `queue.rs:508` |
| `queue.rs:654`: "the contiguous drain at line 285" | the drain is at 336-345 | `queue.rs:336` |
| `lib.rs:2846`: "`EventQueue::mark_native_steer_pending` docs at queue.rs:606" | it is at 673 (line 606 is inside `set_retry_count_for_test`'s doc comment) | `queue.rs:673`, `:604-612` |
| `queue.rs:848`: "see relay messages.rs:762-783" | cross-crate line pointer into `buzz-relay` with no verification mechanism | `queue.rs:846-848` |
| `EventQueue` state-machine doc block | describes `push`/`flush_next`/`mark_complete`/`requeue` but omits `cancelled_batches`, `cancel_reasons`, `withheld_native_steer`, `in_flight_batch_sizes` and the cancelled-only fallback path — the doc is a snapshot of an earlier design | `queue.rs:94-136` vs fields `:151-166` |
| `crates/buzz-acp/README.md` | documents subscription modes, `--kinds`, and the mention filter but never mentions dedup modes, retry/dead-letter behaviour, `MAX_RETRIES`, or the cancel/steer merge framing | `README.md:118-249` |

Five stale in-code line-number references (`queue.rs:163`, `:654`, `:702`, `:765`, `lib.rs:2846`) is a systemic pattern: the codebase uses `file:line` in comments as a cross-reference idiom with no tooling to keep them accurate.

#### Design-level debt

- **Five parallel maps as implicit state.** A channel's state is the conjunction of memberships across `queues` / `in_flight_channels` / `in_flight_deadlines` / `retry_after` / `retry_counts` / `cancelled_batches` / `withheld_native_steer`, so illegal combinations (throttled *and* in flight *and* withheld) are representable and only prevented by call discipline.
- **Cancelled-batch fallback bypasses the retry throttle.** `flush_next`'s fallback (`queue.rs:308-312`) and `has_flushable_work`'s final clause (`queue.rs:586-590`) check only `in_flight_channels`, not `retry_after` — a backed-off channel's cancelled batch can flush during its own backoff window. Nothing tests this interaction.
- **Non-deterministic tie-breaking.** Both the fair-pick `min_by_key` (`queue.rs:299`) and the cancelled fallback `keys().find` (`queue.rs:308-312`) iterate a `HashMap` with no secondary sort key, so behaviour under equal `received_at` (or multiple cancelled channels) is unspecified.
- **`requeue_preserve_timestamps` has no retry budget.** It never increments `retry_counts` and never sets `retry_after` (`queue.rs:508-529`), so a persistently exhausted pool re-queues the same batch indefinitely with no dead-letter and no user-visible failure.
- **No prompt size budget.** `format_prompt` can emit an arbitrarily large prompt (unbounded `content`, unbounded tags, unbounded `cancelled_events`, unbounded `mentioned_pubkeys`); the only trimmer in the crate operates on the observer telemetry copy, not the outbound prompt (`lib.rs:659`).


## Module: buzz-acp — ACP protocol, config & setup mode (`crates/buzz-acp/src`)
### Aspect: Debt

#### File sizes

| File | Lines | Test lines | Non-test |
|---|---|---|---|
| `acp.rs` | 3,717 | from `acp.rs:2008` | ~2,007 |
| `config.rs` | 2,709 | from `config.rs:1328` | ~1,327 |
| `setup_mode.rs` | 1,135 | from `setup_mode.rs:651` | ~650 |
| `base_prompt.md` | 136 | — | — |

For context within the crate: `lib.rs` 6,570, `relay.rs` 6,233, `pool.rs` 5,620, `queue.rs` 4,759. The repo enforces a 1,000-line ceiling on mobile (`mobile/scripts/check-file-sizes.mjs`, per `AGENTS.md`) and the desktop/web equivalents; **no such guard exists for Rust crates**, and seven of thirteen files in this crate exceed 1,000 lines.

#### Zero in-code debt markers

`grep -rn 'TODO\|FIXME\|HACK\|XXX' crates/buzz-acp/src/` returns **0** matches across the entire crate. Known-incomplete work is instead marked with `#[allow(dead_code)]` plus a prose comment, which no lint or grep surfaces:

| Item | Marker | Line |
|---|---|---|
| `drain_stale_responses` | `#[allow(dead_code)] // Scaffolding for future model-switch timeout cleanup; not yet wired.` | `acp.rs:1022` |
| `session_new` (id-only wrapper) | `#[allow(dead_code)] // Public API — callers outside the harness may use this.` | `acp.rs:591` |
| `active_run_id()` | `#[cfg_attr(not(test), allow(dead_code))]` — production reads the field directly | `acp.rs:768` |
| `available_commands_update` | "Logged for observability; UI surfacing is a follow-up." | `acp.rs:1574-1575` |
| `handle_setup_membership`'s 5th param | `_initial_channel_ids` — accepted, unused | `setup_mode.rs:568` |

#### Dead and inert code

| Item | Status |
|---|---|
| `buzz-persona` dependency | Declared at `Cargo.toml:22` — and uniquely by `path = "../buzz-persona"` rather than `workspace = true` like every other internal dep — with **zero** references anywhere under `crates/buzz-acp/src`. Persona resolution moved to the desktop instance snapshot (`config.rs:943-944`); the dep was never removed |
| `Config::persona_env_vars` doc comment | Says "Populated from persona pack resolution" (`config.rs:534`) but the only writer is `codex_network_env` (`config.rs:951-957`) |
| `Config::allowed_respond_to` | Validated at startup (`config.rs:919-937`), then read only by `summary()` (`config.rs:1019-1025`). Operators reading the flag's doc comment ("the harness rejects startup if `--respond-to` is not in this list", `config.rs:456-458`) get exactly that and nothing more — there is no runtime re-check |
| `BUZZ_API_TOKEN` | Written by `propagate_legacy_env_vars` (`config.rs:718`), never read. The README still lists it as "required if relay enforces token auth" (`crates/buzz-acp/README.md:107`) |
| `codex_network_env`'s parsed host | Extracted and logged (`config.rs:654-672`) but not used in the emitted value, which is a hardcoded constant (`config.rs:674-677`). The URL parse survives purely as a fail-closed guard |

#### Duplicated logic

| Duplication | Sites | Risk |
|---|---|---|
| The two read loops | `read_until_response` (`acp.rs:1074-1166`) vs `read_until_response_with_idle_timeout` (`acp.rs:1198-1523`) | ~90 lines of shared frame-parse / observe / id-match / method-dispatch. A fix to one (e.g. the `-32601` reply, `acp.rs:1147-1156` vs `acp.rs:1497-1506`) must be mirrored by hand |
| Channel-filter resolution | `resolve_channel_filters` Config branch (`config.rs:1186-1231`) vs `resolve_dynamic_channel_filter` Config branch (`config.rs:1275-1313`) | The second carries the comment "Same merge logic as `resolve_channel_filters()` Config branch" (`config.rs:1276`) — the invariant is a comment, not a shared function |
| Validation re-implemented in tests | `validate_heartbeat_interval` (`config.rs:1959`), `validate_turn_liveness` (`config.rs:2001`), `resolve_idle_timeout` (`config.rs:2214`), `parse_allowed_respond_to` / `check_allowed_respond_to` (`config.rs:2476`, `:2490`) | These are **copies** of logic inlined in `from_args`, not calls into it. Changing `from_args` leaves the tests green. Only the three `allowed_respond_to_full_path_*` tests (`config.rs:2588`, `:2618`, `:2639`) exercise the real path via `CliArgs::try_parse_from` |
| `AcpAvailabilityStatus` | `setup_mode.rs:58-71` mirrors the desktop enum and `api/types.ts` by hand, justified at `setup_mode.rs:52-57`. Divergence is caught only by the four round-trip tests (`setup_mode.rs:1081-1136` region), which assert the four literals the desktop currently sends |
| Default kind lists | `[STREAM_MESSAGE, WORKFLOW_APPROVAL_REQUESTED, STREAM_REMINDER]` in `config.rs:1161-1165` and `config.rs:1262-1267`, but only the **first two** in `setup_mode.rs:526`. Setup mode silently ignores reminder events |

#### Untested surface

`acp.rs` has 76 tests (27 async), but coverage is concentrated on pure helpers — `StopReason::from_str`, permission-option lookup by kind, `agent_error_from_json` (`acp.rs:3458`, `:3475`), and 13 `build_codex_config_env` cases (`acp.rs:3498-3712`). Not covered:

| Surface | Why it is hard to reach |
|---|---|
| `AcpClient::spawn`'s env-injection precedence | Requires mutating the process environment; `build_codex_config_env` is tested in isolation but the `if std::env::var(key).is_err()` loop at `acp.rs:455-457` is not |
| `stderr` inheritance, `process_group(0)`, `configure_no_window` | Platform-gated, no test asserts them |
| `shutdown()`'s 5 s expiry path | Would need a SIGKILL-resistant child; the "abandoning" branch (`acp.rs:396`) is untested |
| `Drop`'s `try_wait` reap | Untested |
| `MAX_LINE_SIZE` enforcement | No test drives a >10 MB line through the codec |
| Non-matching-id skip behaviour | The silent-skip rule (`acp.rs:1123-1130`) is documented (`acp.rs:1010-1015`) but has no test |
| `-32601` reply to unknown agent requests | No test asserts the harness answers rather than hangs |
| Steer ack drain on every early return | Seven drain sites (`acp.rs:1263`, `:1362`, `:1369`, `:1374`, `:1382`, `:1449`, `:1454`); the invariant "callers are never left hanging" (`acp.rs:1213-1215`) is carried only by the comment |
| `run_setup_listener` | 22 setup-mode tests all target `nudge_body`, `from_raw_env_value`, or `should_nudge_for_event`. The 170-line event loop (`setup_mode.rs:309-480`), `handle_setup_membership`, `publish_setup_nudge`, and `build_setup_subscription_rules` have **no tests** — including the reply-threading choice at `setup_mode.rs:608-624` |
| Wildcard-`kinds` REQ | Nothing asserts that `--subscribe all` without `--kinds` yields `kinds: None`, or warns about it. `test_all_mode_wildcard` (`config.rs:1630`) asserts the wildcard is *produced*, treating it as correct rather than as the p-gate hazard `AGENTS.md § Common Gotchas #2` describes |

#### Invariants carried only in comments

| Invariant | Recorded at | Enforcement |
|---|---|---|
| `initialize` pins `protocolVersion: 2` as a deliberate squat "ahead of the upstream ACP RFD. Revisit when that RFD merges" | `acp.rs:536-537` | None — no tracking issue reference, no version gate |
| Permission write-then-flag ordering prevents an unbounded deadlock | `acp.rs:1735-1748` | Ordering only; a future refactor could reintroduce it |
| The pre-select deadline check is what keeps `biased` from defeating the hard cap | `acp.rs:1189-1194`, `acp.rs:1257-1262` | Comment only; deleting the check compiles and passes tests |
| `expectedRunId` must be sampled at write time, not at dispatch | `acp.rs:1300-1310`, `acp.rs:1786-1789` | Comment + code placement |
| Callers must `await shutdown()`; `kill_on_drop` is best-effort | `acp.rs:373-375`, `acp.rs:422-424`, `acp.rs:1955-1956` | Comment only |
| `take_turn_usage` must be called at most once per turn | `acp.rs:779-781` | Comment; the second call silently returns `None` |
| `install_steer_rx`'s one-receiver rule | `acp.rs:794-799` | The only comment-invariant with a runtime guard — an `assert!` that **panics** in production (`acp.rs:801-805`) |
| Config mode ignores `--channels` "per CLI contract" | `config.rs:1229-1232`, `config.rs:1248-1251` | Warning at startup, then divergent behaviour between two functions |

#### Error-handling gaps

- `--system-prompt-file`, `--heartbeat-prompt`, and `--heartbeat-prompt-file` have **no size cap**, while `--base-prompt-file` is capped at 1 MB (`config.rs:780-790`). All three go into every prompt.
- `--agent-owner` is trimmed and lowercased but not validated as 64-char hex (`config.rs:1003`), unlike `--respond-to-allowlist` entries which are strictly validated (`config.rs:558-572`). An owner typo silently produces an owner that never matches, and with the default `respond_to = owner-only` the agent answers nobody.
- `--max-turns-per-session` uses `value_parser!(u32)` with no range (`config.rs:372-373`), unlike its neighbours which both carry `.range(...)`.
- Invalid `--channels` entries warn and are dropped (`config.rs:816-824`) rather than erroring, so a single typo silently narrows the agent's scope.
- `parse_stop_reason` treats an unknown `stopReason` as a hard `Protocol` error (`acp.rs:1762-1763`), so any future ACP stop reason breaks the turn rather than degrading.

#### Documentation drift

`ARCHITECTURE.md:658-667`'s LOC table is wrong for every row that touches this crate:

| Row | Claimed | Actual |
|---|---|---|
| `relay.rs` (`:660`) | 3,143 | 6,233 |
| `queue.rs` (`:661`) | 2,565 | 4,759 |
| `main.rs` (`:662`) "Event loop, pool orchestration, heartbeat" | 2,457 | **3** — `main.rs` is now just `fn main() { buzz_acp::run() }`; the event loop moved to `lib.rs` |
| `pool.rs` (`:663`) | 2,253 | 5,620 |
| `config.rs` (`:664`) | 1,903 | 2,709 |
| `acp.rs` (`:665`) | 1,785 | 3,717 |
| `filter.rs` (`:666`) | 814 | 787 |

Further drift beyond the table:

- The table omits `lib.rs` (6,570 — the largest file and the actual event loop), `setup_mode.rs`, `usage.rs`, `observer.rs`, `engram_fetch.rs`, and `pool_lifecycle.rs`. Setup mode, the observer feed, and usage metering are absent from the architecture description entirely.
- `ARCHITECTURE.md:656` describes the harness as queueing `@mention` events and says "At most one prompt is in-flight per channel", with no mention of the steer/interrupt cancel modes that are now the **default** (`config.rs:356`).
- `crates/buzz-acp/README.md:105` gives the idle-timeout default as `620`; the code says `900` (`config.rs:27`). The constant's own doc comment (`config.rs:17-26`) explains the 900 choice in detail, so the README is the stale side.
- `crates/buzz-acp/README.md:107` documents `BUZZ_API_TOKEN` as a live setting; it is never read.
- `.env.example:152` steers operators to the hidden, deprecated `BUZZ_ACP_TURN_TIMEOUT=320` and never mentions `BUZZ_ACP_IDLE_TIMEOUT`. `.env.example` covers ~20 of 43 env vars, omitting the whole author-gate group (`BUZZ_ACP_RESPOND_TO`, `..._ALLOWLIST`, `ALLOWED_RESPOND_TO`), `BUZZ_ACP_PERMISSION_MODE`, the base-prompt controls, and `BUZZ_ACP_SETUP_PAYLOAD`.
- `.env.example:221` documents `BUZZ_ACP_EVENT_BUFFER=256`, which is real but read in `relay.rs:36` — the only documented env var with no `config.rs` flag behind it.

#### Structural observations

- `send_request` wraps write and read in **two sequential 60 s timeouts** rather than one budget, so a slow-but-progressing agent can consume ~90 s per non-prompt RPC (`acp.rs:974-976`, `:996-1003`). With `initialize` + `session/new` + optional system-prompt + model set, session establishment has no single bound.
- `SessionNewResponse.raw` (`acp.rs:1831`) and both model extractors (`acp.rs:1851`, `:1866`) return untyped `serde_json::Value`, pushing schema knowledge into string literals spread across `acp.rs` and `pool.rs` (`"configOptions"`, `"category"`, `"options"`, `"value"`, `"models"`, `"availableModels"`, `"modelId"`). A schema change fails silently as a `None` rather than a deserialization error.
- `protocolVersion` is read as `.as_u64().unwrap_or(1)` at `lib.rs:3776` and `lib.rs:3864`, so a missing or non-numeric field silently selects legacy v1 prompt composition. `acp.rs` contributes nothing here — it neither validates the field nor stores the negotiated version on `AcpClient`, so the two call sites are the only guard and they duplicate the same lenient parse.
- `Config` has 40 fields and is constructed literally in three places: `from_args` (`config.rs:961-1007`), `config.rs:1334-1376`, and `lib.rs:4979`/`lib.rs:5145`. Every new field requires four edits, and the test fixture uses non-default values for `respond_to` (`Anyone`) and `multiple_event_handling` (`Queue`), so fixture-based tests do not exercise the shipped posture.


## Module: buzz-agent — core loop, ACP wire types & handoff (`crates/buzz-agent/src`)
### Aspect: Debt
#### Documentation drift (crate README vs code)
The crate README is the only real reference for this group, and seven of its statements no longer match the code:

| README claim | Line | Reality |
|---|---|---|
| "The full server is hand-rolled in `main.rs`" | `README.md:124` | `main.rs` is 6 lines and only calls `buzz_agent::run()` (`main.rs:1-6`); the server is `lib.rs:187-825` |
| "Three request methods (`initialize`, `session/new`, `session/prompt`), one inbound notification (`session/cancel`), and three outbound update variants" | `README.md:124` | Six request methods are handled — plus `session/set_model` (`lib.rs:234`), `session/cancel` as a *request* (`lib.rs:237`), and `_goose/unstable/session/steer` (`lib.rs:245`) — and ten outbound update variants exist (`agent.rs:552-616`, `agent.rs:134-206`, `lib.rs:612-620`, `lib.rs:661-670`, `lib.rs:730-750`) |
| Handshake transcript shows `protocolVersion: 1` in both directions | `README.md:68`, `README.md:70` | `PROTOCOL_VERSION` is 2 (`config.rs:3`); a client asking for 2+ gets 2 (`lib.rs:284`), asserted by `test_initialize_version_check` (`tests/golden_transcripts.rs:288`) |
| `session/new` result is `{"sessionId":"ses_…"}` | `README.md:84` | Result also carries `models.currentModelId` and `models.availableModels` (`lib.rs:464-474`) |
| "Everything is environment variables. No flags, no config files." | `README.md:128` | There is a CLI subcommand: `buzz-agent auth <provider>` (`lib.rs:111-116`, `lib.rs:129-152`) |
| `BUZZ_AGENT_MAX_HISTORY_BYTES` default `1048576` / "History window 1 MiB" | `README.md:158`, `README.md:236` | Code default is 16 MiB (`config.rs:814`) — a 16× discrepancy |
| "One reader, one writer, up to 8 concurrent prompt tasks (one per session)" | `README.md:286` | Sessions default to unlimited (`config.rs:812`) and each gets its own prompt task (`lib.rs:623-625`); `8` is the default `max_parallel_tools` (`config.rs:821`), a different limit |

The transcript section also predates `agent_thought_chunk`, `keepalive`, `usage_update`, `session_info_update`, and `session/set_model`, so a reader implementing a client from `README.md:60-122` will not know to handle them.

#### Documentation drift (repo-level)
- `ARCHITECTURE.md` never mentions this crate: `grep -rn 'buzz-agent' ARCHITECTURE.md` returns zero matches, and the crate inventory at `ARCHITECTURE.md:88-93` omits it while listing `buzz-acp`, `buzz-sdk`, `buzz-cli`, `buzz-admin`, `buzz-test-client`. The root `README.md:209` and `AGENTS.md:50` both do list it, so the architecture doc is the outlier.
- `CONTRIBUTING.md` has zero mentions (`grep -rn 'buzz-agent' CONTRIBUTING.md` → 0 matches), so no contributor guidance exists for the crate that owns the agent loop.
- `.env.example` documents none of the ~20 `BUZZ_AGENT_*` variables (`grep -c 'BUZZ_AGENT' .env.example` → 0) while documenting the `BUZZ_ACP_*` harness set in detail (`.env.example:114-170`).

#### Cross-crate contract drift
`crates/buzz-acp/src/acp.rs:185-186` documents that "goose/buzz-agent emit `activeRunId: null` at end of turn", and the client relies on that to clear its cached run id. buzz-agent emits `activeRunId` exactly once per turn, at prompt start (`lib.rs:661-670`; `grep -n 'activeRunId' src/*.rs` shows one emission site), and clears the value only internally (`lib.rs:707`). The client's cache therefore stays stale until the next turn starts; the only thing preventing a misdirected steer is buzz-agent's own mismatch rejection (`lib.rs:588-598`). Either the comment or the emission is wrong — worth resolving in whichever direction the steer protocol intends.

`docs/MCP_DRIVEN_HOOKS.md:14-17` states hook responses are injected as tool-result messages and are JSON-encoded for prompt-injection safety. True for `_Stop` (`agent.rs:641-651`, `agent.rs:666-695`); false for `_PostCompact`, which is concatenated as plain `[{server}]\n{text}` (`handoff.rs:247-253`) into a synthetic **user** message (`handoff.rs:85-92`). The code has a documented reason (`handoff.rs:78-83`) but the spec doc does not record the exception.

#### Stale / misplaced in-code comments
- `handoff.rs:152-162` — the comment on the `None` arm of `projected_handoff_input_tokens` describes byte-to-token mapping, "capped conservatively", and "Never raise the cap above the configured byte budget". That arm is a one-liner returning `current_tokens` (`handoff.rs:163`); the capping logic it describes lives in `should_handoff` (`handoff.rs:115-126`) and `byte_fallback_threshold` (`handoff.rs:359-368`). Eleven lines of explanation attached to the wrong site.
- `lib.rs:255-259` — a deferred-work note ("Revisit when that RFD merges") about `[Base]` gating that this crate does not implement; the gate is in `crates/buzz-acp/src/pool.rs:181`.
- `agent.rs:284-293` — `execute_calls`' doc describes a "(degenerate, max_parallel=1) path" as if two code paths existed; there is only one (`agent.rs:339`).

There are **no `TODO`/`FIXME`/`XXX`/`HACK` markers anywhere in this group** (`grep -n 'TODO\|FIXME\|XXX\|HACK' lib.rs agent.rs types.rs wire.rs handoff.rs main.rs` → 0 matches). Deferred work is instead recorded in prose comments like the one above, which means no tooling or grep-based audit will ever surface it.

#### Structural debt
- **13-element tuple as an interface.** `acquire_session` returns a 13-field anonymous tuple (`lib.rs:764-783`) destructured into 13 named locals at the call site (`lib.rs:632-652`). Adding one piece of session state means touching the signature, the tuple literal (`lib.rs:806-819`), the destructuring pattern, and the write-back block (`lib.rs:704-712`) — four coupled edits with no compiler help on ordering, since several fields share the type `Option<String>` / `Option<u64>`.
- **`Session::id` duplicates the map key.** Both are set from the same `session_id` (`lib.rs:411-412`) and the field is read once (`lib.rs:809`); they can never legitimately diverge but nothing enforces that.
- **No session teardown.** `grep -n 'sessions.remove' lib.rs` → 0 matches. Sessions and their MCP child processes accumulate for the process lifetime, with `max_sessions` defaulting to `usize::MAX` (`config.rs:812`).
- **Panic in a prompt task wedges the session permanently.** `busy` is cleared only on the normal path (`lib.rs:704-705`), history has already been moved out (`lib.rs:807`), and there is no `catch_unwind` (`grep -c 'catch_unwind'` → 0 in all six files). A panicked turn leaves the session unusable and its context lost, with no diagnostic beyond the tokio task-panic message.
- **Mutex held across an await.** The second session-cap re-check takes the `sessions` lock at `lib.rs:399` and awaits `reject(...)` at `lib.rs:401-408` while still holding it; if the 64-slot wire channel (`lib.rs:164`) is full, that blocks every other session operation including `session/cancel` (`lib.rs:489`).
- **Unbounded steer channel.** `mpsc::unbounded_channel()` (`lib.rs:802`) with no per-steer size check in `drain_steers` (`agent.rs:266-270`), while the initial prompt *is* capped at 1 MiB (`agent.rs:69-73`). The asymmetry is unexplained.
- **Three truncation helpers.** `types::clamp` (`types.rs:259-274`, marker `\n[truncated]`), `handoff::clamp_bytes` (`handoff.rs:300-315`, marker `…`), `mcp::truncate_middle`/`truncate_at_boundary` (`mcp.rs:866`, `mcp.rs:886`). `clamp` additionally lives in `types.rs` but has zero callers there — both users are in `mcp.rs` (`:254`, `:630`), so it is a misplaced helper exported as public API.
- **Inconsistent exit codes.** `die()` → 2 (`lib.rs:105-108`); `main.rs` → 1 (`main.rs:2-5`); a fatal reader error (oversize frame, invalid UTF-8) is only logged and the process exits **0** (`lib.rs:170-176` then `Ok(())` at `lib.rs:121`). A supervisor cannot distinguish clean shutdown from protocol abort.
- **Inconsistent RNG failure policy.** `session_new` fails closed on RNG error (`lib.rs:394-397`) but run ids and steer message ids fail open to the literals `run_x` / `steer_x` (`lib.rs:799`, `lib.rs:577`), which are the values steer authorization compares against (`lib.rs:588-598`).
- **Threshold operator asymmetry.** `should_handoff` uses `>=` on the token path and `>` on the byte path (`handoff.rs:111-113` vs `handoff.rs:115-126`), with no comment explaining the difference.
- **`Session` field write-back is manual and partial.** `run_prompt` restores seven fields (`lib.rs:704-712`) but not `accumulated_*`, which are updated separately under a second lock acquisition (`lib.rs:717-727`) — two lock round-trips per turn and two places that must agree about what "end of turn" means.

#### Hardcoded values that should probably be config
| Value | Site | Why it matters |
|---|---|---|
| keepalive interval 30 s | `agent.rs:129` | Exists to satisfy the harness idle clock, which *is* configurable (`.env.example:169-170`); the two can drift out of alignment |
| post-cancel drain 5 s | `agent.rs:455` | After this, tasks are aborted mid-flight (`agent.rs:466-471`) |
| wire channel depth 64 | `lib.rs:164` | Backpressure point for every notification, including the lock-holding path above |
| `usage_update.contextLimit = 0` | `lib.rs:742` | Reported to the harness as "no limit" even though `max_context_tokens` is known (`handoff.rs:113`) |
| token width 8 bytes | `lib.rs:822` | 64-bit session/run id entropy |

#### Test-coverage gaps
`agent.rs` — 746 lines, the busiest file in the group — has **no unit tests at all** (`grep -n '#\[cfg(test)\]' agent.rs` → 0 matches), while `lib.rs`, `types.rs`, `wire.rs` and `handoff.rs` each carry a test module. Directly untested logic with non-obvious behavior:

| Untested path | Site | Risk |
|---|---|---|
| `truncate_history` `break` when no later `User` item exists | `agent.rs:723-725` | Returns silently over budget; only the happy path is exercised, and only via integration (`tests/regressions.rs:555`) |
| `prompt_to_text` joining and `resource_link` rendering | `agent.rs:618-631` | Only the error branch is covered (`tests/golden_transcripts.rs:384`) |
| `map_stop` collapse of `ToolUse`/`Other` → `end_turn` | `agent.rs:740-746` | Silent semantic mapping |
| `format_hook_output_body` escaping (the prompt-injection defense) | `agent.rs:641-651` | The security property has no direct assertion |
| `synthetic_hook_id` / `unique_nonce` uniqueness | `agent.rs:654-657`, `agent.rs:697-701` | Duplicate ids would corrupt the provider wire format |
| Preflight-cancel slot filling and `j < idx` `failed` emission | `agent.rs:300-313` | Ordering-sensitive; covered only indirectly |
| `max_rounds` cap | `agent.rs:88-90` | `grep -n 'MAX_ROUNDS\|max_turn_requests' tests/` → 0 matches |
| Both `max_sessions` rejections | `lib.rs:346-355`, `lib.rs:399-409` | `grep -n 'max sessions reached' tests/` → 0 matches |
| Non-UTF-8 frame rejection | `wire.rs:207-214` | No test |
| Steer `expectedRunId` empty-string rejection | `lib.rs:567-575` | No test (the other three steer rejections are covered) |
| `session/cancel` for an unknown session returning success | `lib.rs:487-494` | Behavior is indistinguishable from a real cancel |

#### Things the LOC table gets right
The stated sizes were exact at the time and are re-measured here after `16d4ec33`: `wc -l` now gives `lib.rs` **960** (was 954 — `8eb6e3eb` added 6 lines, all inside the test module), `main.rs` 6, `agent.rs` 746, `types.rs` 353, `wire.rs` 293, `handoff.rs` 430 — **2,788** total. Note, though, that 134 of `lib.rs`'s lines are its test module (`lib.rs:827-960`), so the production body is ~2,654 lines. The four in-file test modules (`lib.rs:827`, `types.rs:276`, `wire.rs:239`, `handoff.rs:370`) total 326 lines, not the ~180 previously stated — that figure was wrong before the sync and is corrected here; `agent.rs` and `main.rs` have no test module at all.


## Module: buzz-agent — LLM providers & configuration (`crates/buzz-agent/src`)
### Aspect: Debt
The debt here is not rot — both files are actively maintained, densely tested, and free of the usual markers. Re-verified after `16d4ec33`/`8eb6e3eb` added ~950 lines to `llm.rs`: `grep -rn 'TODO\|FIXME\|HACK\|XXX' llm.rs config.rs` returned **zero matches**, `grep -n '#\[allow' llm.rs config.rs` returned zero matches, and `grep -c 'unsafe' llm.rs config.rs` returned 0 for both. So there is still no `#[allow(dead_code)]` standing in for a `TODO`, and the mesh work did not introduce one. What is present instead is structural: duplicated classification logic, a test-only re-implementation that can mask drift, one wrong default in the docs, an undocumented new switch, and a set of critical paths with no test.

#### Doc drift
| Claim | Doc site | Code | Severity |
|---|---|---|---|
| `BUZZ_AGENT_MAX_HISTORY_BYTES` default is `1048576` / "1 MiB" | `crates/buzz-agent/README.md:155`, repeated `README.md:236` | `16 * 1024 * 1024` at `config.rs:821` | 16× wrong; operator-visible |
| The env table enumerates the complete config surface | `README.md:130-156` | 11 of 36 vars missing: `BUZZ_AGENT_MODEL` (`config.rs:755`), **`BUZZ_AGENT_PREFER_MESH_FOR_AUTO` (`config.rs:807`)**, `BUZZ_AGENT_MCP_INIT_TIMEOUT_SECS` (`config.rs:812`), `BUZZ_AGENT_MCP_RESTART_MAX_ATTEMPTS` (`config.rs:816`), `BUZZ_AGENT_MCP_RESTART_BASE_MS` (`config.rs:817`), `BUZZ_AGENT_MCP_RESTART_MAX_MS` (`config.rs:818`), `BUZZ_AGENT_HOOK_TIMEOUT_MS` (`config.rs:829`), `BUZZ_AGENT_STOP_MAX_REJECTIONS` (`config.rs:830`), `MCP_HOOK_SERVERS` (`config.rs:831`), `BUZZ_AGENT_NO_HINTS` (`config.rs:832`), `BUZZ_AGENT_THINKING_EFFORT` (`config.rs:833`) | 11 undocumented vars, one of which (`MCP_HOOK_SERVERS`) gates a feature the README links to at `README.md:254`, and one of which (`PREFER_MESH_FOR_AUTO`) changes which model actually serves a turn |
| `OPENAI_COMPAT_API=auto` "picks Responses for `*.openai.com`, Chat Completions everywhere else" — the README's only description of any `auto` behaviour | `README.md:140` | there is now a *second* `auto` concept in the crate: the **model** string `auto` plus `prefer_mesh_for_auto` can silently rewrite the request model to `mesh` (`llm.rs:410-469`). The README mentions neither | new drift from `16d4ec33`; the two `auto`s are unrelated and undistinguished in docs |
| "`Provider` is a Rust `enum` with one `match` in `Llm::complete`. There is no trait, no `Box<dyn>`, no async-trait." | `README.md:181` | a second provider `match` in `Llm::summarize` (`llm.rs:238-327`); `Arc<dyn TokenSource>` on `Llm` (`llm.rs:104`) with `#[async_trait]` methods; and now a third provider check outside both matches (`llm.rs:411`) | stale since the `TokenSource` refactor, worse since `16d4ec33` |
| "Adding a provider is a `match` arm and one `body`/`parse` pair in `llm.rs`." | `README.md:181` | requires touching `Llm::complete` (`llm.rs:132`), `Llm::summarize` (`llm.rs:238`), `build_token_source` (`llm.rs:1530`), `resolve_openai_model`'s provider guard (`llm.rs:411`), and `from_env`'s provider arm (`config.rs:763`) | understates by four sites |
| "Everything is environment variables. No flags, no config files." | `README.md:128` | `buzz-agent auth <provider>` subcommand exists (`lib.rs:111-117`, handler `lib.rs:129-153`) | stale |
| Provider list: `anthropic`, `openai`, `databricks`, `databricks_v2` | `README.md:132` | also accepts `openai-compat` (`config.rs:1003`) and `databricks-v2` (`config.rs:1008`) | undocumented aliases |
| `OPENAI_COMPAT_API` values `auto \| chat \| responses` | `README.md:140` | also accepts `chat-completions`, `chat_completions` (`config.rs:1023`) | undocumented aliases |
| MCP child env whitelist: `PATH, HOME, TERM, LANG, LC_ALL, TMPDIR` | `README.md:221` | also passes `XDG_CONFIG_HOME` (`mcp.rs:47`) | minor; sibling module |
| "Not streaming. One non-streaming HTTP POST per round." | `README.md:255` | accurate on streaming, but "one POST per round" is no longer exact: an adaptive `auto` round can issue a catalog `GET` plus **two** POSTs (`llm.rs:361-390`) | new, minor |
| `ARCHITECTURE.md` describes system design | — | `grep -rni 'buzz.agent' ARCHITECTURE.md` returned zero matches — the agent surface is entirely absent from the top-level architecture doc | documentation gap, not a contradiction |

#### Stale in-code cross-references
`anthropic_efforts_for_model`'s doc comment asserts (`config.rs:407-413`):

> "This is the single production source of truth for Anthropic family routing. Both `anthropic_thinking_config` (request-time) and the effort-table UI (`valid_effort_values_for_provider_model`, via its Anthropic branch) must derive their behaviour from this helper so the two stay in sync."

Both halves of that are wrong (re-verified; unchanged by these commits):
1. `anthropic_thinking_config` (`config.rs:124-178`) never calls it. It classifies independently via `is_manual_budget_model` (`config.rs:136`), `is_adaptive_thinking_model` (`config.rs:161`), and `clamp_adaptive_effort` (`config.rs:166`).
2. `valid_effort_values_for_provider_model` is not "the effort-table UI" — it is a `#[cfg(test)]` function defined at `config.rs:2567`, inside `mod tests`.

The net effect: `anthropic_efforts_for_model` (`config.rs:415`) is a `pub fn` whose **only caller in the entire repo is a test** (`config.rs:2596`). `grep -rn 'anthropic_efforts_for_model' --include='*.rs' crates/` returns three hits: the doc mention (`config.rs:180`), the definition (`config.rs:415`), and the test-helper call (`config.rs:2596`).

Second stale reference: the comment at `config.rs:441-442` says "Reuse `anthropic_model_supports_xhigh` (the single source of truth shared with `clamp_adaptive_effort`) — no side-effects, no duplication." That is true for the xhigh predicate specifically, but sits three lines below a manual/adaptive classification (`config.rs:437-440`) that *is* duplicated with `anthropic_thinking_config`.

Third, new: `PostError`'s doc comment says "It is consumed inside `openai_request`" (`llm.rs:1341`). That is accurate for the `MeshFallback` variant but understates the type's reach — `PostError::Agent` is constructed at nine sites in `post` (`llm.rs:1424`-`llm.rs:1514`) and collapsed in three other functions (`llm.rs:337`, `llm.rs:590`, `llm.rs:393`). The doc reads as if the type were local to one function.

#### Duplicated classification logic
Four independent places classify a model *name*, with three different matching strategies:

| Site | Purpose | Strategy |
|---|---|---|
| `config.rs:136`, `config.rs:161` (inside `anthropic_thinking_config`) | pick thinking shape | prefix match after `strip_catalog_prefix` |
| `config.rs:437`, `config.rs:440` (inside `anthropic_efforts_for_model`) | pick capability set | same predicates, same order, independent call |
| `config.rs:367-392` (inside `openai_efforts_for_model`) | pick OpenAI effort table | boundary-safe `gpt5_token_matches` / `gpt5_base_matches` |
| `llm.rs:974-976` (inside `databricks_v2_route_for_model`) | pick gateway route | plain `contains("gpt-5")` / `contains("gpt5")` / `contains("claude")` |

The last one is still the substantive defect. `config.rs` invested two purpose-built matchers (`config.rs:239-254`, `config.rs:267-311`) and eight boundary tests (`config.rs:2286-2409`) specifically to stop `gpt-5.1` matching `gpt-5.10`, `gpt-5-1` matching `gpt-5-1106`, and `gpt-5-4` matching `gpt-5-4o`. `llm.rs:974-975` uses naked `contains` for the routing decision in the same request path. Concretely: `databricks-gpt-5-10` routes to `/ai-gateway/openai/v1/responses` (`llm.rs:985`) while `openai_efforts_for_model` classifies it as unknown (asserted at `config.rs:2346`) and its effort passes through unverified. And `contains("claude")` is unanchored, so `my-claude-killer-llama` routes to the Anthropic Messages path. No test pairs the two classifiers — `databricks_v2_routes_by_model_family` (`llm.rs:2464-2487`) uses three unambiguous names only.

**`16d4ec33` did not make this worse, and is worth noting as a counter-example.** The mesh policy needed to recognise two model names and chose exact equality every time — `llm.rs:413`, `llm.rs:57`, `llm.rs:59`, `llm.rs:637` — plus set-based deduplication for peers (`llm.rs:60`). So the new code follows `config.rs`'s discipline rather than `llm.rs`'s local precedent, and `databricks_v2_route_for_model` remains the file's only naked-`contains` name classifier.

Third duplication: `strip_catalog_prefix` (`config.rs:89-97`) is re-implemented verbatim inside the test helper at `config.rs:2578-2590` rather than being called.

Fourth: `Llm::summarize` hand-builds an Anthropic body (`llm.rs:240-248`) duplicating `anthropic_body`'s top-level shape (`llm.rs:730-731`), and a Chat body (`llm.rs:270-278`) duplicating `openai_body`'s (`llm.rs:828-829`). A fix to either shape must be applied twice.

Fifth: the Databricks PKCE configuration is declared twice — canonically at `llm.rs:1538-1551` using `DATABRICKS_CLIENT_ID` (`llm.rs:22`) and `DATABRICKS_OAUTH_SCOPES` (`llm.rs:23`), and again as bare string literals in `lib.rs:135-143`. Because both constants are private to a private module, `lib.rs` structurally cannot reference them. A scope added to `DATABRICKS_OAUTH_SCOPES` would not be requested by `buzz-agent auth databricks`, producing a cached token the runtime then rejects. No test asserts the two agree.

Sixth, new: the OpenAI-compatible `/models` catalog is fetched and parsed in two places with no shared code — `observe_mesh_virtual_model` + `mesh_catalog_supports_collective` (`llm.rs:472-530`, `llm.rs:47-64`) for the viability probe, and `catalog.rs` for model discovery (which shares only `build_token_source`, `catalog.rs:117`). The probe reimplements URL construction (`llm.rs:473`), bearer application (`llm.rs:487`), and `data[].id` extraction (`llm.rs:48-52`) rather than reusing the existing catalog client, and unlike that client it enforces no response-size cap (`llm.rs:508`).

#### Test that re-implements production rules (drift can go green)
`valid_effort_values_for_provider_model` (`config.rs:2567-2658`) is 92 lines of test-only logic mirroring the TypeScript `getProviderEffortConfig`. The fixture guard `effort_table_fixture_matches_rust_implementation` (`config.rs:2674-2708`) compares `effortTable.fixture.json` against *this function*, not against production code. The comment block at `config.rs:2550-2559` describes it as a drift guard that "fails CI here before it can silently diverge in production" — but three rules live only in the test:

1. **Prefix stripping** (`config.rs:2578-2590`) — a copy of `strip_catalog_prefix` (`config.rs:89-97`). If production's stripping changed, the fixture would still pass.
2. **Default-effort-per-family derivation** (`config.rs:2607-2615`): `GPT5_PRO → high`, `GPT5_1 → none`, else `medium`. This mapping exists nowhere in production Rust. The fixture's `defaultValue` column is therefore validated against a test-local table, not against anything the agent does.
3. **The DBv2 gpt-5 family predicate** (`config.rs:2631-2644`) — a copy of the chain at `config.rs:367-392` that **omits** the `gpt-5-6` / `gpt5-6` dash variants production checks at `config.rs:371-372`.

Item 3 is currently harmless only by accident: both the `is_gpt5` branch (`config.rs:2645-2647`) and the fall-through (`config.rs:2648-2651`) `return openai_result(&m)`, so the 14-line `is_gpt5` computation is **dead for every non-empty model name** — it only affects the blank-model case (`config.rs:2652-2653`). So the guard's most complex logic does nothing, and the omission it contains cannot be detected by any test.

There is also a hard compile-time coupling: `include_str!("../../../desktop/src/features/agents/ui/effortTable.fixture.json")` at `config.rs:2675-2676`. Moving or renaming that desktop file breaks `cargo test -p buzz-agent`, not just a test assertion.

#### Untested critical paths
Each verified by grepping the relevant test module and finding zero matches, unless marked resolved:

| Path | Site | Why it matters |
|---|---|---|
| `Config::from_env` | `config.rs:742-837` | 36 env vars, 10 error paths, the `BUZZ_AGENT_SYSTEM_PROMPT`/`_FILE` mutual exclusion (`config.rs:792-794`), the file-read error (`config.rs:796`), the `BUZZ_AGENT_NO_HINTS` `== 0` inversion (`config.rs:832`), and the new `BUZZ_AGENT_PREFER_MESH_FOR_AUTO` `!= 0` inversion (`config.rs:807`) — none exercised |
| The 13 numeric invariants in `validate` | `config.rs:884-939` | The only `validate()` tests (`config.rs:1867-1991`) vary `thinking_effort` alone. `make_config_for_validation` has to *undo* `for_discovery`'s invalid values (`config.rs:1855-1862`) to reach the effort check — direct evidence the numeric branches are unreached |
| `Llm::new` | `llm.rs:108-121` | **Status: resolved** in `16d4ec33` (the ten async mesh tests construct `Llm` through `Llm::new`, e.g. `llm.rs:1837`, so the production `connect_timeout` + `read_timeout` wiring at `llm.rs:109-113` is now built and used against a real socket). Original finding: the test helper `llm_with` (`llm.rs:3634-3648`) built `Llm` by struct literal with `.timeout(5s)`, so production timeout wiring was never exercised. `llm_with` still does this, so the auth suite remains on a stricter client than production, and no test *asserts* on the client's configuration |
| `Llm::complete` | `llm.rs:123-228` | **Status: resolved (partially)** in `16d4ec33` (the mesh tests drive the `OpenAi` arm end-to-end via `complete_model` / `complete_model_with_tool`, `llm.rs:1765-1804`). Still untested: the `Anthropic` and `DatabricksV2` arms, and the model-name error-stamping `map_err` at `llm.rs:222-228` — `llm.rs:2136` asserts on an error *body*, not on the `(model-name)` prefix |
| `Llm::summarize` | `llm.rs:230-328` | Zero tests; 99 lines, three provider arms, three hand-built bodies. Now also silently subject to the mesh policy (`llm.rs:252-284`) with no coverage of that interaction |
| `try_upgrade` / the sticky latch | `llm.rs:657-673` | The *matcher* is well tested (`llm.rs:2448`), the *latch* is not — nothing asserts a second call skips Chat Completions or that the warn fires once. Unchanged |
| Both response-size caps | `llm.rs:1484-1491`, `llm.rs:1492-1502` | No test for `response too large` or `response exceeded` |
| The catalog probe's **missing** size cap | `llm.rs:508` | `response.json::<Value>()` with no `content_length` precheck and no chunk accounting — the only unbounded read in the file. No test feeds it an oversized document |
| `read_error_body` truncation | `llm.rs:1268-1284` | No test for the 4 KiB cap or the partial-chunk break at `llm.rs:1272-1276`. Now doubly relevant: both mesh classifiers parse this truncated string (`llm.rs:1448-1449`), so a body straddling the 4 KiB boundary could fail to classify |
| `strip_model` | `llm.rs:1560-1570` | Only caller is `llm.rs:614`; no test. The legacy-Databricks body rewrite is unverified |
| Chat-Completions malformed `arguments` | `llm.rs:1226-1228` | The Responses equivalent is tested (`llm.rs:2489`); this one is not |
| `make_tool_call` rejections | `llm.rs:1249-1251`, `llm.rs:1255-1259` | Empty id/name and non-object arguments — no direct test |
| Reasoning extraction, all three dialects | `llm.rs:1037-1058`, `llm.rs:1167-1174`, `llm.rs:1208-1218` | No test asserts a non-empty `LlmResponse.reasoning` |
| `ProviderStop::Refusal` and `::Other` | `llm.rs:1097-1099` | No test; `Refusal` is unreachable on the Responses path (`llm.rs:1064-1079` doesn't use `map_stop`) with no comment noting it |
| `HookServers::is_disabled` | `config.rs:1080` | No test |
| Body-read error non-retry | `llm.rs:1499-1512` | The request was already sent; the decision not to retry is uncommented and untested |
| `anthropic_api_version` on the DBv2 Anthropic route | `llm.rs:986` vs `llm.rs:638` | `post_openai` sends only `bearer_auth`, so no `anthropic-version` header reaches the gateway's Anthropic endpoint. Whether that is correct is unverified |
| `MESH_AUTO_CATALOG_TIMEOUT` | `llm.rs:31`, applied `llm.rs:488` | The stub (`llm.rs:1662-1745`) always answers promptly, so the 2 s bound is never exercised |
| Mesh TTL and cooldown *durations* | `llm.rs:30`, `llm.rs:32` | The tests backdate the state by hand (`expire_mesh_catalog_check`, `llm.rs:1806-1809`; `expire_mesh_auto_cooldown`, `llm.rs:1811-1815`) instead of using `tokio::time::pause`/`advance`, which the crate already enables (`Cargo.toml:51`). Only the branches the durations guard are covered, never the durations |
| `looks_like_unstructured_tool_call` | `llm.rs:66-72` | No direct unit test. Covered only indirectly and only for the `<\|tool_call` prefix (`llm.rs:2054`, fixture at `llm.rs:2059`); the `<tool_call` and `[tool_call]` branches (`llm.rs:70-71`), the `trim_start` and the `to_ascii_lowercase` are all unasserted. This is the loosest classifier in the file and the least tested |
| A 401/403 on the catalog probe | `llm.rs:500-506` | `spawn_sequence_stub` only emits 200/500/502/503 (`llm.rs:1724-1730`), so the "auth failure is silently `Unknown`, no refresh, `debug`-only" branch has no test |
| The composed request budget of a mesh fallback | `llm.rs:361-390` × `llm.rs:631-649` × `llm.rs:1403` | Worst case is 1 GET + 2 POSTs × 2 auth attempts × 3 retries. Nothing caps or tests the composition |
| `Llm::complete`'s mesh path for `Anthropic`/`DatabricksV2` with the flag set | `llm.rs:411` | The guard is asserted only for `Provider::OpenAi` with the flag off (`llm.rs:2121`); no test sets the flag on a non-OpenAI provider |

#### Production-path panics and rule violations
| Site | Issue |
|---|---|
| `config.rs:510` | `.expect("supported is non-empty")` in `resolve_openai_effort` — violates the `AGENTS.md` rule "Do not introduce new `unwrap()` or `expect()` in production paths". Safe only because every table at `config.rs:334-362` is a non-empty `const` |
| `llm.rs:1516` | `unreachable!()` at the end of `post` — reachable if `MAX_RETRIES` (`llm.rs:1285`) were ever set to 0. The invariant is asserted only in the panic message. Re-verified against the `PostError` control flow: the two `continue`s (`llm.rs:1422`, `llm.rs:1461`) are still the only ways to reach a next iteration and both are guarded by `attempt + 1 < MAX_RETRIES`, so the refactor neither fixed nor worsened this |

Both are low-probability, but both are the kind of implicit coupling that a refactor breaks silently. The ~950 new lines added neither a third panic nor a new `unwrap`/`expect` on a production path.

#### Oversized files and functions
| Unit | Size | Ceiling context |
|---|---|---|
| `llm.rs` | 3,846 lines (~59% test), up from 2,894 | `AGENTS.md` states a 1,000-line hard ceiling, but the enforcement (`justfile:123` desktop, `justfile:585` web, `justfile:617` mobile) is JS/TS/Dart only — there is no Rust file-size gate. So `llm.rs` is now ~3.8× the stated ceiling with no gate to trip. It is not the repo's worst (`crates/buzz-acp/src/lib.rs` is 6,570 lines) |
| `config.rs` | 2,709 lines (~59% test) | same; ~2.7× |
| `Config::from_env` | 96 lines (`config.rs:742-837`) | one struct literal with 27 initializers |
| `Llm::complete` | 106 lines (`llm.rs:123-228`), up from 94 | single expression, three-arm `match`, nested closures |
| `Llm::summarize` | 99 lines (`llm.rs:230-328`) | near-mirror of `complete`, untested |
| `post` | 128 lines (`llm.rs:1390-1517`), up from 112 | retry + status mapping + mesh classification + streaming read in one function |
| `openai_efforts_for_model` | 69 lines (`config.rs:333-401`) | mostly a 26-term `else if` chain (`config.rs:367-392`) that would be better as a table |
| `resolve_openai_model` | 60 lines (`llm.rs:410-469`) | the mesh state machine; single linear flow, four state transitions plus a TTL and a cooldown gate in one function |
| `observe_mesh_virtual_model` | 59 lines (`llm.rs:472-530`) | five near-identical failure arms, each with its own `debug!` |
| `spawn_sequence_stub` | 84 lines (`llm.rs:1662-1745`) | test-only; hand-rolled HTTP server, the largest helper in the file |
| `valid_effort_values_for_provider_model` | 92 lines (`config.rs:2567-2658`) | test-only, includes dead logic |

On the credit side, `16d4ec33` reduced one hot spot by extraction rather than growing it: the old monolithic `openai_request` split into `openai_request` (52 lines, `llm.rs:344-395`) for policy and `openai_request_for_model` (40 lines, `llm.rs:535-574`) for dispatch, and `post_anthropic` (9 lines, `llm.rs:330-338`) moved the Anthropic POST out of the inline path.

#### `pub` items with no cross-file caller
Not dead code (all are reachable), but `pub` beyond need, which enlarges the crate's committed API surface:

| Item | Site | Only caller |
|---|---|---|
| `anthropic_efforts_for_model` | `config.rs:415` | a test (`config.rs:2596`) |
| `clamp_adaptive_effort` | `config.rs:205` | `config.rs:166`, same file |
| `parse_thinking_effort` | `config.rs:622` | `config.rs:833`, same file |
| `DEFAULT_TOOL_RESULT_TEXT_BYTES` | `config.rs:649` | `config.rs:824`, same file |
| `ThinkingEffort::anthropic_budget_tokens` | `config.rs:33` | `config.rs:145`, same file |
| `ThinkingEffort::anthropic_effort_str` | `config.rs:60` | `config.rs:169`, same file |
| `is_openai_host` | `config.rs:1037` | `llm.rs:546` — crate-internal only |

`Llm`, `Llm::new`, `Llm::complete`, `Llm::summarize` are declared `pub` (`llm.rs:88`, `llm.rs:108`, `llm.rs:123`, `llm.rs:230`) inside a **private** module (`lib.rs:9`), so the `pub` keyword there is inert. `build_token_source` is the only correctly-scoped item (`pub(crate)`, `llm.rs:1529`). Every item the mesh work added is correctly private — no new `pub` in `llm.rs`, and the one new `Config` field (`config.rs:734`) had to be `pub` to match its 26 siblings.

#### Missing documentation on public API
`AGENTS.md` requires "New public API must have doc comments". Missing:
- `Config::from_env` (`config.rs:742`) — no doc comment at all, on the crate's primary configuration entry point.
- **20 of 27** `Config` fields (`config.rs:688-700`, `config.rs:713-715`, `config.rs:723-726`, `config.rs:735`). Recomputed after `16d4ec33`: the struct grew by one but the undocumented count did not, because `prefer_mesh_for_auto` arrived with a five-line doc comment (`config.rs:729-733`). The previous revision of this document said "19 of 26", which undercounted the documented set by one — the actual pre-change figures were 6 documented / 20 undocumented, now 7 / 20.
- `PROTOCOL_VERSION` (`config.rs:3`), `MAX_PROMPT_BYTES` (`config.rs:638`), `MAX_SYSTEM_PROMPT_BYTES` (`config.rs:639`), `MAX_TOOL_CALLS_PER_TURN` (`config.rs:650`), `HANDOFF_MAX_OUTPUT_TOKENS` (`config.rs:652`), `HANDOFF_ORIGINAL_TASK_MAX_BYTES` (`config.rs:654`), `HANDOFF_MAX_TOOL_NAMES` (`config.rs:656`).
- `Provider::Anthropic` and `Provider::OpenAi` variants (`config.rs:663-664`) — the two Databricks variants are documented (`config.rs:665-677`), the two originals are not.

The discipline remains inverted: private helpers in `llm.rs` (`openai_request` `llm.rs:340-343`, `resolve_openai_model` `llm.rs:406-409`, `openai_request_for_model` `llm.rs:532-534`, `post_openai` `llm.rs:594-598`, `try_upgrade` `llm.rs:655-656`, `PostError` `llm.rs:1337-1342`, `build_token_source` `llm.rs:1519-1528`) are documented while the public methods are not — and `16d4ec33` widened the gap by documenting three more private items and one new private type while leaving `Llm::new`, `Llm::complete` and `Llm::summarize` bare.

Undocumented *new* internals, which matters here because their semantics are non-obvious: all seven `MESH_*` constants (`llm.rs:28-34`), `MeshCatalogObservation` (`llm.rs:41`), `mesh_catalog_supports_collective` (`llm.rs:47`), `looks_like_unstructured_tool_call` (`llm.rs:66`), `MeshAutoState` (`llm.rs:74`), `cool_down_collective` (`llm.rs:397`), `observe_mesh_virtual_model` (`llm.rs:472`), `is_mesh_moa_unavailable_body` (`llm.rs:1364`) and `is_mesh_moa_failure_body` (`llm.rs:1377`). The state machine's rules are recoverable only by reading `resolve_openai_model` (`llm.rs:410-469`) together with the `Llm` field comment (`llm.rs:95-97`) and the `Unknown`-arm comment (`llm.rs:452-456`).

#### Design fragility worth recording
1. **Implicit discriminant dependency.** `resolve_openai_effort` computes distance via `(e as i32 - requested as i32)` (`config.rs:495`), which requires `ThinkingEffort`'s variants to be contiguous 0..=6 in declaration order (`config.rs:19-27`). `thinking_effort_ord_ordering` (`config.rs:1395`) tests the `Ord` relation but not the cast values, so inserting a variant mid-enum would change every fallback distance while the test stayed green.
2. **A documented rule that is only emergent.** `config.rs:459` states "`xhigh` falls back to `high` when not supported (no model skips from `high` to `xhigh`)". Nothing implements that — it falls out of the sort at `config.rs:486-491`. A future table with `max` but not `xhigh` would resolve `xhigh` upward to `max`, contradicting the documented rule with all existing tests passing. Anchors re-confirmed; unchanged by these commits.
3. **Inert defaults that are not inert.** `openai_api` is hard-coded to `Chat` for both Databricks providers (`config.rs:789`) with the comment "only read by OpenAI/legacy Databricks dispatch". It *is* read for legacy Databricks (`llm.rs:545-546`, `llm.rs:563`) and the value permanently disables the chat→responses auto-upgrade there. Anthropic gets `Auto` instead (`config.rs:772`) for no stated reason.
4. **`for_discovery` produces a `Config` that would fail `validate`.** `max_output_tokens: 1` (`config.rs:856`), `mcp_max_restart_attempts: 0` (`config.rs:860`), `mcp_restart_base_ms: 0` (`config.rs:861`) are all rejected by `config.rs:925-933`, and `validate` is never called (`config.rs:845-876`). The odd `max_context_tokens: 200_001` (`config.rs:867`) exists only to satisfy a check that never runs. Nothing prevents this struct from being handed to code that assumes a validated config. (`prefer_mesh_for_auto: false` at `config.rs:854` is at least correctly inert.)
5. **`post_openai` discards `path` for legacy Databricks.** `llm.rs:608-616` rewrites the URL regardless of whether the caller asked for `/chat/completions` or `/responses`. Unreachable today (the forced `Chat` above blocks the upgrade), but the shape allows a Responses-body-to-Chat-URL mismatch if the latch were ever set.
6. **No wall-clock request timeout on inference.** `Llm::new` sets only `connect_timeout` and `read_timeout` (`llm.rs:109-113`). Re-verified: no `.timeout(...)` on the client builder. A slow-drip response is unbounded; `STALL_NOTICE_THRESHOLD` (`llm.rs:27`) only logs (`llm.rs:1324-1330`). The old evidence formulation is now wrong: `grep -n '\.timeout(' llm.rs` finds one **production** match (`llm.rs:488`, the 2 s catalog probe) plus four test matches (`llm.rs:3171`, `llm.rs:3235`, `llm.rs:3286`, `llm.rs:3637`). The production bound is per-request on the `GET /models` builder and does not reach any POST.
7. **No outbound body-size cap.** `serde_json::to_vec` at `llm.rs:1400` serializes any history, and the bytes are cloned per attempt (`llm.rs:1407`). All the relevant bounds are defined in `config.rs` but enforced in `agent.rs`/`handoff.rs`; `grep -n 'max_history_bytes' llm.rs` returns only the test fixture at `llm.rs:1593`. A mesh fallback doubles the exposure by rebuilding and re-sending the same history (`llm.rs:371`).
8. **Unvalidated `mime_type` interpolation.** Tool-supplied `mime_type` (`types.rs:136`) goes straight into `data:{mime_type};base64,{data}` at `llm.rs:860` and `llm.rs:919` and into Anthropic's `source.media_type` at `llm.rs:755`, with no allowlist and no rejection of `;` or `,`.
9. **Test/production HTTP-client divergence.** Narrowed but not gone. `llm_with` (`llm.rs:3634-3648`) still bypasses `Llm::new` with `.timeout(5s)` (`llm.rs:3637`), so the auth suite runs against a client production never uses; and it had to be hand-edited for the new `mesh_auto_state` field (`llm.rs:3641`), which is the recurring cost of a struct-literal fixture. The mesh tests do use `Llm::new` (`llm.rs:1837`), so a regression in its timeout wiring would now at least be *constructed* by the suite — though still never asserted on.
10. **New: a verbatim cross-repository string contract.** `MESH_MOA_UNAVAILABLE_MESSAGE` (`llm.rs:34`) is the mesh router's 503 message copied character-for-character, including the `≥`. If upstream rewords it, `is_mesh_moa_unavailable_body` (`llm.rs:1364-1375`) silently stops matching and the 503 fallback dies quietly — the agent would keep sending `mesh` requests that fail, retried three times each by the ordinary loop (`llm.rs:1453`) instead of falling back once. The tests cannot catch this because they build their fixture from the same constant (`llm.rs:1971`, `llm.rs:2139`, `llm.rs:2148`, `llm.rs:2173`). The `moa_failure` classifier (`llm.rs:1377-1388`) is structurally safer, keying on an error *type* rather than prose.
11. **New: two virtual model names hard-coded on both sides of a process boundary.** `MESH_VIRTUAL_MODEL_ID`/`MESH_AUTO_MODEL_ID` (`llm.rs:28-29`) must agree with the router's catalog and with the desktop launcher's `RELAY_MESH_AUTO_MODEL_ID` (`desktop/src-tauri/src/managed_agents/relay_mesh.rs:4`). Nothing links them, and `BUZZ_AGENT_PREFER_MESH_FOR_AUTO` is likewise a constant on one side (`desktop/src-tauri/src/managed_agents/relay_mesh.rs:6`) and a literal on the other (`config.rs:807`). Renaming any of the three compiles cleanly and silently disables the feature.
12. **New: the model substitution is invisible above `Llm::complete`.** The ACP session, the desktop UI, and any audit consumer see the configured model (`auto`) because `agent.rs:124` never learns the resolved one; the error-stamping `map_err` also stamps the configured name (`llm.rs:223`). The only record is a `debug!` (`llm.rs:464`), which the default subscriber (`lib.rs:155-158`) does not emit. So in normal operation there is no way to tell from logs or from the transcript which of two models answered a turn.
13. **New: unbounded read on the catalog probe.** `response.json::<Value>()` (`llm.rs:508`) is the only response read in this file with neither a `Content-Length` precheck nor a streaming cap, in a file that otherwise applies both (`llm.rs:1484-1502`). Bounded only by the 2 s timeout (`llm.rs:488`) and available memory.
14. **New: the mesh tuning constants are unreachable from configuration.** TTL, probe timeout, cooldown, and the confirmation count are `const` (`llm.rs:30-33`) with no env override, so an operator whose mesh flaps on a longer period than 5 s, or whose router is slower than 2 s, has no remedy short of a rebuild. That is a defensible v1 choice, but it is the only tuning in this group that is not an env var.


## Module: buzz-agent — MCP registry, OAuth, hints & catalog (`crates/buzz-agent/src`)
### Aspect: Debt

#### Correctness debt: comments that describe behaviour the code does not have

| Claim | Claim site | Code reality |
|---|---|---|
| `kill_server` "Counts as one attempt toward the restart budget so that a pathological server (starts fine, deadlocks on every call) eventually exhausts" | `mcp.rs:414-419` | `attempts` is hardcoded to `1` on every kill (`mcp.rs:432`) and a successful restart replaces the state with `Healthy`, which stores no counter (`mcp.rs:672-677`). Kill→restart cycles are unbounded; only consecutive *spawn* failures are budgeted (`mcp.rs:685-699`) |
| discovery "Returns a non-empty `Vec<ModelEntry>` on success" | `catalog.rs:110` | **Still open.** The v1 path returns `parse_v1_endpoints` verbatim, which can be empty (`catalog.rs:163`; `v1_parse_empty_endpoints_returns_empty_vec`, `catalog.rs:435`). Only v2 has an empty-list fallback (`catalog.rs:287-296`). Re-verified against `8eb6e3eb`, which touched neither the doc comment nor the v1 return: the wording at `catalog.rs:110` is byte-identical to the pre-sync version. `8eb6e3eb` did add a way for v2 to produce an empty *parsed* list — a workspace serving only embedding endpoints now filters to nothing (`catalog.rs:370-373`) — but that lands upstream of the `is_empty` check (`catalog.rs:288`), so v2 still cannot return empty and v1 remains the sole counterexample |
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
| `discovery_failure_fallback`'s wildcard arm | `catalog.rs:78` | **Still open, and now more literal.** The `_` arm is byte-identical to the `Provider::Databricks` arm one line above it (`catalog.rs:77` vs `catalog.rs:78` — both `_ => vec![configured]` / `Provider::Databricks => vec![configured]`); before `8eb6e3eb` the two arms each built their own `ModelEntry` literal, so the duplication was four lines rather than one. It is still only reachable by calling the `pub fn` directly with a non-Databricks provider, which `discover_databricks_models` rejects first (`catalog.rs:126-130`) |
| `configure_no_window_compiles_and_applies_flag_on_windows` | `mcp.rs:1128-1140` | a `#[cfg(windows)]` test with **no assertion**; its own comment concedes "The real protection is the cfg-gated production path" |

Notably, none of this is masked with `#[allow(dead_code)]` — the group's only `#[allow]` is `clippy::too_many_arguments` on `do_call` (`mcp.rs:555`). `grep -n '#\[allow' mcp.rs auth.rs hints.rs builtin.rs catalog.rs` returns exactly one match. There are also zero `TODO`/`FIXME`/`XXX`/`HACK` markers in all five files (`grep -n 'TODO\|FIXME\|XXX\|HACK' mcp.rs auth.rs hints.rs builtin.rs catalog.rs` → exit 1, no matches), so known gaps are recorded in prose comments rather than in greppable markers — which is why several of the contradictions above went unnoticed. All three greps re-run after `8eb6e3eb`: unchanged — one `#[allow]`, no markers, and `grep -n 'unsafe\|SAFETY' mcp.rs auth.rs hints.rs builtin.rs catalog.rs` still returns zero matches despite `catalog.rs` gaining 229 lines.

#### Duplication

| Duplicated logic | Sites | Failure mode if they drift |
|---|---|---|
| Databricks OAuth `client_id`, scopes, discovery-URL template | `lib.rs:135-144` (hand-copied) vs `llm.rs:19-20`, `llm.rs:1183-1195` | the cache filename hashes all three (`auth.rs:446-454`), so `buzz-agent auth databricks` would write a token the runtime never reads — and both sides would report success |
| percent-encoding | `catalog.rs:222-233` (hand-rolled) vs the already-declared `urlencoding` crate used at `auth.rs:589-594` | two encoders with different reserved sets; the justifying comment (`catalog.rs:246-247`) cites a reqwest feature but the crate already ships a URL encoder. Still open — `8eb6e3eb` left `percent_encode` and its call site untouched |
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
| catalog HTTP layer (`fetch_v1_models`, `fetch_v2_models`, `percent_encode`, pagination, 20-page cap, v2 empty fallback) | `grep -rn 'discover_databricks_models\|fetch_v1_models\|fetch_v2_models\|percent_encode' crates/buzz-agent/tests` → zero matches. **Partially resolved** in `8eb6e3eb`: nine new *in-file* unit tests (`catalog.rs:498`, `:522`, `:542`, `:578`, `:590`, `:603`, `:612`, `:621`) took `catalog.rs` from 6 to 15 tests and now cover the pure logic that used to be reachable only through the HTTP layer — the v2 chat-capability filter, `endpoint_created_ms` for both wire shapes, `sort_v2_endpoints_newest_first`, and `discovery_failure_fallback` including its trim and dedupe rules. Still uncovered, and unchanged: `discover_databricks_models` itself, `fetch_v1_models`, `fetch_v2_models`, `percent_encode`, the page-token loop and its repeat-token break (`catalog.rs:281-284`), the 20-page cap (`catalog.rs:245`), and the v2 empty-list fallback (`catalog.rs:287-296`). The four re-run greps confirm it: the original grep still returns zero matches, and `grep -rn 'sort_v2_endpoints_newest_first\|endpoint_created_ms\|is_chat_capable_endpoint\|V2Endpoint' crates/buzz-agent/tests` also returns zero — the new coverage is entirely inside `catalog.rs`, so nothing exercises the filter or sort against a real paginated response |
| `MAX_HINTS_BYTES` truncation | `grep -rn 'MAX_HINTS_BYTES' crates/buzz-agent/src crates/buzz-agent/tests` → 2 matches, both in `hints.rs` production code |
| `MAX_HOOK_RESULT_BYTES` truncation | `grep -rn 'MAX_HOOK_RESULT_BYTES' …` → 3 matches, all in `mcp.rs` production code |
| `auth` CLI argument handling | `grep -rn '"auth"' crates/buzz-agent/tests` → zero matches |
| `https`/scheme rejection | nothing to test — no such check exists. Re-run after `8eb6e3eb`: `grep -n 'https\|scheme' auth.rs catalog.rs` still matches only `auth.rs` test fixtures (`auth.rs:689`, `:725`, `:766`, `:818`) and zero lines in `catalog.rs` |
| catalog HTTP timeout | nothing to test — no timeout is configured. Re-run after `8eb6e3eb`: `grep -n 'timeout' catalog.rs` → zero matches; the client is still a bare `Client::new()` (`catalog.rs:120`) |

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

`grep -n '\.rs:[0-9]' mcp.rs auth.rs hints.rs builtin.rs catalog.rs` returns zero matches (re-run after `8eb6e3eb`), so there are no hardcoded `file:line` pointers to go stale. Cross-references are by identifier instead (`[\`build_token_source\`](crate::llm::build_token_source)` at `catalog.rs:6`, `[\`RunCtx::should_handoff\`]` style links elsewhere in the crate), which rustdoc will fail on if the target disappears — a better pattern; `8eb6e3eb` followed it, adding four intra-file rustdoc links (`[\`parse_v1_endpoints\`]` at `catalog.rs:85` and `:96`, `[\`is_chat_capable_endpoint\`]` and `[\`sort_v2_endpoints_newest_first\`]` at `catalog.rs:350` and `:352`). Two references point at other repos' behaviour and cannot be verified from here: "Mirrors goose's `DATABRICKS_V2_KNOWN_MODELS`" (`catalog.rs:31`) and "the same shape goose uses for Databricks" (`auth.rs:9-11`); the two model IDs in that list (`catalog.rs:32-33`) are compile-time constants with no freshness mechanism.

#### Missing observability

`hints.rs`, `builtin.rs`, and `catalog.rs` contain no logging at all (`grep -n 'tracing::' hints.rs builtin.rs catalog.rs` → zero matches). Silent-by-design outcomes with no diagnostic trail: a `SKILL.md` skipped for missing `name` (`hints.rs:124-126`), a skill name shadowed by an earlier directory (`hints.rs:127-131`), a hint chain truncated at 128 KiB (`hints.rs:77-81`), a `load_skill` result cut at 32 KiB (`builtin.rs:102-105`, `:206-209`), and a catalog walk stopped at 20 pages (`catalog.rs:245`). The first two are the ones most likely to be reported as "my skill isn't showing up".

`8eb6e3eb` widened this gap rather than closing it — the grep still returns zero matches for `catalog.rs`, and the commit added two more silent outcomes. First, an endpoint dropped by the chat-capability name heuristic leaves no trace (`catalog.rs:370-373`): because the rule is a name pattern with no wire signal behind it (`catalog.rs:85-88`), a chat-capable endpoint whose name happens to contain `embedding` disappears from the picker with nothing in the log to explain it — the exact "my model isn't showing up" shape as the skill-shadowing case above. Second, a `created_timestamp` that fails to parse silently becomes `None` and sorts the endpoint last (`catalog.rs:320-325`, ordering at `catalog.rs:340-342`), so a gateway-side wire-format change degrades the whole catalog to alphabetical order with no warning. A single `tracing::debug!` at each of those two points would make both diagnosable.

#### Suggested order of repair

1. Set `0o600` on the OAuth token cache and use a unique temp filename (`auth.rs:146-200`) — smallest change, largest security return.
2. Correct or implement the `kill_server` restart-budget comment (`mcp.rs:414-419` vs `mcp.rs:432`, `mcp.rs:672-677`).
3. Add timeouts to the OAuth and catalog HTTP clients (`auth.rs:153`, `catalog.rs:120`), matching `llm.rs:53-57`; a hung host currently stalls `session/new`.
4. Reconcile the README's env-allowlist and `pre_exec` claims with `mcp.rs:39-63` and `mcp.rs:733`, and document `NOSTR_PRIVATE_KEY` / `BUZZ_PRIVATE_KEY` passthrough explicitly.
5. Bound or escape skill `name`/`description` before they enter the system prompt (`hints.rs:239-243`), bringing them in line with the trust rules the hooks doc already states.
6. Reject non-`https` OAuth and catalog URLs, or document the decision to allow `http` (`auth.rs:161-190`, `catalog.rs:121`).
7. Escape the reflected error in the loopback callback page (`auth.rs:565`).
8. Give the four untested critical paths a test each: applied child env, grandchild kill, restart/backoff, and the catalog HTTP layer — the last one narrowed by `8eb6e3eb`, which unit-tested the pure parse/filter/sort logic (`catalog.rs:498-630`) but left `fetch_v2_models`'s pagination loop, 20-page cap, and empty-list fallback (`catalog.rs:235-304`) with no test at all.
9. Log the two new silent drops in the v2 path — a name-filtered endpoint (`catalog.rs:370-373`) and an unparseable `created_timestamp` (`catalog.rs:320-325`) — since `catalog.rs` still has zero `tracing::` calls.


## Module: buzz-dev-mcp (`crates/buzz-dev-mcp`)
### Aspect: Debt

#### Repo-rule compliance

| Rule (`AGENTS.md`) | Status |
|---|---|
| No `unsafe` code | Violated on Windows only: 5 × `#[allow(unsafe_code)]` against the crate's own `deny` (`shell.rs:474`, `:750`, `:753`, `:757`, `:826`), covering the registry probe, Job Object lifecycle, and two hand-written `unsafe impl Send/Sync for KillGroup`. Non-Windows builds are `forbid(unsafe_code)` (`lib.rs:1-2`). All 13 `unsafe` occurrences carry `SAFETY:` comments |
| No new `unwrap()`/`expect()` in production paths | Clean. Every `expect(` and bare `unwrap()` in the crate is inside `#[cfg(test)]`. Production code uses only infallible `unwrap_or*` forms (`lib.rs:143`, `shell.rs:143`, `shell.rs:301`, `rg.rs:28`, `rg.rs:67`, `rg.rs:396`, `shim.rs:164`, `shim.rs:167`, `shim.rs:203`, `tree.rs:111`, `tree.rs:183`, `view_image.rs:91`, `view_image.rs:199`) |
| New public API must have doc comments | Violated. The crate's single externally-reachable item, `pub fn run()` (`lib.rs:138`), has no doc comment. `pub fn artifact_dir` (`shim.rs:244`) and `pub fn Shim::install` (`shim.rs:25`) also lack their own doc comments, though the `Shim` struct doc (`shim.rs:7-17`) covers `install`'s behaviour |
| TODO/FIXME/HACK/XXX count | **0.** The single `grep` hit across the whole crate is the string literal `"GIF89aXXX"` in a test fixture (`view_image.rs:1048`) |

#### File sizes against the 1,000-line ceiling

`AGENTS.md` enforces a hard 1,000-line-per-file ceiling for mobile
(`mobile/scripts/check-file-sizes.mjs` via `just mobile-check`) and states it
mirrors desktop and web. No equivalent guard exists for Rust crates, and two files
here exceed that number:

| File | Lines | Over ceiling |
|---|---|---|
| `shell.rs` | 1,503 | +503 |
| `view_image.rs` | 1,136 | +136 |
| `todo.rs` | 558 | — |
| `rg.rs` | 491 | — |
| `str_replace.rs` | 356 | — |
| `paths.rs` | 279 | — |
| `shim.rs` | 248 | — |
| `read_file.rs` | 234 | — |
| `lib.rs` | 213 | — |
| `tree.rs` | 184 | — |
| `main.rs` | 3 | — |

`shell.rs` mixes five concerns in one file: `SharedState`/bootstrap
(`shell.rs:26-117`), the tool handler (`shell.rs:130-323`), a ~150-line
cross-platform shell resolver (`shell.rs:326-672`), three `KillGroup`
implementations (`shell.rs:674-857`), and output capture/truncation
(`shell.rs:859-981`) — plus 520 lines of tests. `view_image.rs` similarly carries
image handling, HTTP fetching, and a Nostr/Blossom auth signer
(`view_image.rs:236-318`) in one module.

#### Untested surface

Three source files have zero tests: `lib.rs` (213 lines), `shim.rs` (248 lines),
`tree.rs` (184 lines). 23 of the crate's 97 tests are `#[cfg(windows)]`-gated
(`shell.rs:1139-1503`, `paths.rs:215-278`), so 74 run on a Unix CI host.

Specific untested behaviour:

| Untested item | Site |
|---|---|
| `Shim::install` end-to-end — 0700 dir mode, five symlinks, `path_env` prepending | `shim.rs:25-76` |
| `NOSTR_PRIVATE_KEY` removal from the process env — the crate's main secret-handling invariant | `shim.rs:51-56` |
| `write_keyfile` / `write_keyfile_atomic` 0600 mode and `create_new` collision behaviour | `shim.rs:87-132`, `shim.rs:134-149` |
| `build_git_env` — the ten `GIT_CONFIG_*` pairs and `GIT_CONFIG_COUNT` composition | `shim.rs:178-216` |
| `derive_git_email` — localhost/`127.*`/empty-host fallback to `@buzz` | `shim.rs:154-172` |
| `tree::run` and `tree::parse` — every rule, including depth clamping, `[truncated]` emission, leaf-directory annotation suppression, and the 10 MiB line-count skip | `tree.rs:18-184` |
| Multicall dispatch in `run()` — no test exercises `argv[0]`-based personality selection | `lib.rs:138-160` |
| `detect_stack` / `build_bootstrap` — the marker table and the conditional `BUZZ_RELAY_URL`+`BUZZ_PRIVATE_KEY` line | `shell.rs:75-117` |
| `finalize_stream` truncation and artifact writing; `rotate_artifacts` 8-file eviction; `align_to_char_boundary` | `shell.rs:894-981` |
| `read_capped` `CAPTURE_CAP` behaviour and `total_bytes` accounting | `shell.rs:867-892` |
| `shell` cancellation path — covered only from the agent side, in a test that skips when the binary is unbuilt | `shell.rs:218-239`; `crates/buzz-agent/tests/regressions.rs:1572-1600` |
| `rg::try_system_rg`, `clean_path`, `which_rg`, `walk`, `emit_line`, `read_bounded_line`, `CappedSink` line-limit branch | `rg.rs:18-73`, `rg.rs:228-261`, `rg.rs:336-386` |
| `str_replace`'s 10 MiB projected-size preflight and the 64 KiB diff cap | `str_replace.rs:70-82`, `str_replace.rs:140-155` |
| `read_file`'s lack of an output byte cap | `read_file.rs:46-58` |
| `view_image::fetch_url` — no HTTP test server; `Content-Length` rejection, mid-stream cap, redirect-disable, and 401/403 messaging are all unexercised | `view_image.rs:321-390` |
| `relay_media_get_auth` composition — only `is_relay_media_url` and `sign_media_get_auth` are unit-tested in isolation | `view_image.rs:286-318` vs `view_image.rs:1054-1136` |

#### Cross-platform defects

`rg.rs` hardcodes the Unix PATH separator and the Unix executable name:
`.split(':')` at `rg.rs:34` and `rg.rs:50`, and `dir.join("rg")` at `rg.rs:39` and
`rg.rs:54` — no `.exe` suffix, no `PATHEXT`, no `std::env::split_paths`. On Windows
`try_system_rg` therefore effectively never resolves a real ripgrep, so the
substring-only fallback silently becomes the permanent behaviour on that platform.
This contrasts with the rest of the crate, which is scrupulous about
`std::env::split_paths` (`shim.rs:44-48`, `shell.rs:378`, `shell.rs:627`,
`shell.rs:652`) and about `.exe` suffixing (`shell.rs:645-670`,
`shim.rs:236-242`).

`tree.rs` returns exit code `0` when `writeln!` fails (`tree.rs:106-108`,
`tree.rs:126-128`), so a broken pipe or full disk is reported as success.

#### Behavioural divergence from the advertised contract

The `shell` tool description tells the model that `rg` is on PATH with flags
`-n -i -l -g <glob> -C <n> --files` (`lib.rs:42`). When the fallback is in use the
match is a literal substring, not a regex (`rg.rs:289-296`), and the glob engine
is a hand-rolled `*`/`?` matcher with no `**`, no character classes, and no brace
expansion (`rg.rs:401-434`). A model that writes a regex pattern gets silently
wrong results with exit code `1` rather than an error. Nothing in the description
or the output distinguishes the delegated path from the fallback.

The `rg` fallback also caps output silently: `CappedSink` latches, logs a
`tracing::warn!` to stderr, and stops printing with **no marker on stdout**
(`rg.rs:152-166`) — unlike `shell`, `tree`, `read_file`, and `str_replace`, which
all emit an in-band truncation marker.

#### Inconsistency in error channels

`todo` returns validation failures as `Ok(CallToolResult::error)` in-band text
(`lib.rs:93-97`, `todo.rs:245-248`), while `str_replace`, `read_file`, and
`view_image` return the same class of user error as protocol-level
`Err(ErrorData::invalid_params)`. `shell` uses both: argument validation is a
protocol error (`shell.rs:136`, `shell.rs:152`) but shell-resolution failure,
spawn failure, and cancellation are in-band (`shell.rs:163`, `shell.rs:186`,
`shell.rs:237`).

Similarly, only `todo.rs` sets `#[serde(deny_unknown_fields)]` (`todo.rs:24`,
`:33`, `:45`). `ShellParams`, `ReadFileParams`, `StrReplaceParams`, and
`ViewImageParams` silently accept and discard unknown keys, so a model
misspelling `timeout_ms` gets the default with no signal.

#### Documentation drift

| Document | Status |
|---|---|
| `ARCHITECTURE.md` | `buzz-dev-mcp` is **absent** — zero matches for `dev-mcp` or `dev_mcp`. This crate is one of the ~11 not covered by that document |
| `AGENTS.md` | Present in the crate table (`AGENTS.md:51`) and referenced as "separate" from `buzz-cli` (`AGENTS.md:147`). Neither entry mentions `BUZZ_SHELL`, the absence of path containment, or the `buzz`/`rg`/`tree` multicall behaviour |
| `CONTRIBUTING.md` | **No mention at all** — zero matches for `dev-mcp`, despite `AGENTS.md` pointing there for "how to add event kinds / CLI subcommands / HTTP endpoints" |
| `README.md` | One clause: "`buzz-dev-mcp` (shell + file-edit tools)" (`README.md:207`) |
| `VISION_AGENT.md` | The most substantive prose (`VISION_AGENT.md:1`, `:15`, `:45`). Its claim that "File edits resolve against the working directory" (`:15`) is misleading — relative paths resolve against `workdir`, but absolute paths and `..` are accepted without restriction (`paths.rs:3-6`, `paths.rs:32-33`) |
| Crate-level docs | There is **no `crates/buzz-dev-mcp/README.md`**, and `lib.rs` has no `//!` module doc. Four modules do have `//!` headers (`paths.rs:1-9`, `shim.rs:7-17` as a struct doc, `todo.rs:1-18`, `view_image.rs:1-12`); `shell.rs`, `rg.rs`, `tree.rs`, `read_file.rs`, `str_replace.rs`, and `main.rs` have none |

#### Duplication with `buzz-cli` and elsewhere

There is no tool-level duplication of `buzz-cli`: the router exposes no relay
operation (`lib.rs:40-125`), and `buzz-cli` is instead linked in and re-exposed
verbatim as the `buzz` multicall personality (`lib.rs:168-171`). The
`AGENTS.md:147` boundary holds.

Real duplication exists elsewhere:

- The Blossom `t=get` signer (`view_image.rs:252-274`) reimplements the desktop
  client's media-read auth, with the constant `MEDIA_GET_AUTH_EXPIRY_SECS = 600`
  copied by reference rather than shared (`view_image.rs:51-53`). Divergence
  between the two is possible and unguarded.
- `resolve_path` (`paths.rs:20-45`) is a fourth path-resolution implementation
  alongside `buzz-persona::safe_resolve` (`crates/buzz-persona/src/pack.rs:323-361`),
  with different (weaker) semantics and no shared helper.
- An identical `fn make_state(cwd)` test fixture is duplicated in four modules
  (`shell.rs:991-994`, `read_file.rs:74-77`, `str_replace.rs:231-234`,
  `view_image.rs:684-687`).
- Numeric caps are re-declared per module rather than shared: `50 KiB` output cap
  appears as `MAX_BYTES` (`shell.rs:20`), `MAX_OUTPUT_BYTES` (`rg.rs:6`), and
  `MAX_OUTPUT_BYTES` (`tree.rs:6`); `2000` lines as `MAX_LINES` (`shell.rs:21`),
  `MAX_OUTPUT_LINES` (`rg.rs:7`), `MAX_OUTPUT_LINES` (`tree.rs:7`), and
  `DEFAULT_LIMIT` (`read_file.rs:6`); `10 MiB` as `MAX_FILE_BYTES` in both
  `paths.rs:15` and `tree.rs:9`, and as `CAPTURE_CAP` in `shell.rs:19`.

#### Dead or unreachable code

- `GIT_CONFIG_COUNT` composition (`shim.rs:199-215`) is documented as reachable
  only when dev-mcp is run standalone, because `buzz-agent` calls `env_clear()`
  before spawning (`shim.rs:174-177`, `crates/buzz-agent/src/mcp.rs:714`).
- The `#[cfg(not(any(unix, windows)))] KillGroup` stub (`shell.rs:846-857`) exists
  only to keep the crate compiling; no such target is built.
- `Encoded.summary` is always constructed as `String::new()` by `encode_resized`
  and filled by the caller instead (`view_image.rs:617`, `view_image.rs:657-663`,
  `view_image.rs:602-610`) — a field that is never meaningfully written where it
  is defined.
- `truncate` (`str_replace.rs:157-164`) and `truncate_str`
  (`str_replace.rs:166-175`) are two near-identical truncation helpers differing
  only in char-vs-byte accounting.


## Module: buzz-cli — dispatch, relay client & validation (`crates/buzz-cli/src`)
### Aspect: Debt

#### Dead code kept alive by `#[allow(dead_code)]`

| Item | Site | Callers |
|---|---|---|
| `BuzzClient::relay_url()` | `client.rs:567-570` | none (`grep -rn '\.relay_url()' commands/` → zero) |
| `BuzzClient::count()` — the whole `POST /count` integration | `client.rs:802-834` | none (`grep -rn 'client.count(' commands/` → zero) |

Both use `#[allow(dead_code)]` rather than a `TODO` explaining the intent or a
deletion. `POST /count` is a documented relay capability (`AGENTS.md:145`) with a
finished, retry-wrapped client method that no subcommand exposes — either wire it
to a `buzz count` subcommand or delete it. These are the only two `#[allow]`
attributes in the group (`grep -rn '#\[allow' lib.rs client.rs validate.rs error.rs agent_management.rs main.rs`).

There are **zero** `TODO`/`FIXME`/`HACK`/`XXX` markers in the group
(`grep -rn 'TODO\|FIXME\|HACK\|XXX' lib.rs client.rs validate.rs error.rs agent_management.rs main.rs`
returns nothing) — so all debt below is undeclared.

#### Test-only functions living in production files

`parse_retry_in_secs` (`client.rs:172-186`) is `#[cfg(test)]`-gated and has six
tests devoted to it (`client.rs:1444-1477`). Production never calls it: the real
code path calls `parse_retry_hint_text` directly on the (already field-extracted)
body (`client.rs:663`, `client.rs:948`). So six of the group's 95 tests exercise a
JSON-extraction wrapper that does not exist at runtime, while the production
composition — `handle_response` extracts the field, *then*
`parse_retry_hint_text` runs — is covered only indirectly by the two integration
tests that assert elapsed time (`client.rs:1688`, `client.rs:1909`).

`percent_encode` (`validate.rs:75-99`) is the same pattern: `#[cfg(test)]`-gated,
five tests (`validate.rs:277-306`), zero production callers
(`grep -rn 'percent_encode' commands/` → zero). `crates/buzz-cli/README.md:216`
still advertises `validate.rs (UUID, hex, content size, percent-encode)` as if it
were live behavior.

#### Unreachable branch plus stale comment

`run_from_args` handles clap's non-stderr errors with the comment
"`--help` and `--version`: print normally (intentional human output)"
(`lib.rs:48-52`). `--version` is not declared — `#[command(...)]`
(`lib.rs:62-78`) has no `version` attribute — so that half of the comment
describes an impossible input. Verified: `buzz --version` exits **1** with
`unexpected argument '--version' found`.

#### Duplication

- The relay-error-body extraction rule exists three times: the named helper
  `extract_relay_message_field` (`client.rs:190-198`), an inline copy in
  `submit_moderation_event` under the comment "Map the body through
  `handle_response`'s error logic inline" (`client.rs:985-996`), and a third copy
  inside `handle_response` itself (`client.rs:1261-1270`) — which does not call
  the helper either.
- The 403 `BUZZ_AUTH_TAG` hint string is duplicated verbatim
  (`client.rs:991-997`, `client.rs:1271-1279`).
- `resp_was_success(u16)` (`client.rs:217-219`) re-implements
  `StatusCode::is_success` because `submit_moderation_event` consumes the
  response before it checks the status.
- `upload_file` repeats its entire request-building block for the legacy endpoint
  (`client.rs:1152-1178` vs `client.rs:1195-1226`) — ~30 lines differing only in
  the URL.
- `validate_hex64` (`validate.rs:28-36`) and `is_lower_hex_sha256`
  (`client.rs:260-262`) are two hex validators in one crate with different
  case-sensitivity rules.

#### Oversized files and functions

`client.rs` 2,477 lines (production ends at 1,433; 42% is tests) and `lib.rs`
2,035 lines — both above the 1,000-line ceiling the repo enforces for Dart via
`justfile:617`, with no Rust equivalent (`grep -rn 'check-file-sizes' justfile`
matches only the mobile recipe). `lib.rs` is one flat clap declaration: 22
subcommand enums plus the dispatch match, with no submodule split (a
`cli/` module tree mirroring `commands/` would be the obvious cut).
`submit_moderation_event` (`client.rs:873-1022`, ~150 lines, 5 retry branches,
one `unreachable!`) and `upload_file` (`client.rs:1100-1227`) are the two
functions most in need of decomposition.

#### Untested critical paths

| Behavior | Status |
|---|---|
| `exit_code` — the contract `AGENTS.md:189-190` publishes | **no test**: `grep -c 'exit_code' <(awk 'NR>=137' error.rs)` → 0. I verified 6 of the 12 mappings by running the built binary; `Conflict`(5), `NotFound`(1), `DeliveryUnknown`(2), `Other`(4) and both `Relay` branches remain unexercised anywhere |
| `print_error` | never invoked by a test; the two JSON-shape tests rebuild its object inline (`error.rs:197-210`, `:213-221`), so the production category strings (`error.rs:109-126`) are untested |
| `normalize_relay_url` / `to_ws_url` | no tests (`grep -c 'normalize_relay_url' <(awk 'NR>=1434' client.rs)` → 0). The ws↔http mapping every command depends on, and the `str::replace` non-prefix-anchored behavior, are unverified |
| `normalize_events` | no tests — the sig-stripping read envelope `AGENTS.md:188-189` documents |
| `normalize_write_response` | no tests, despite 47 call sites in `commands/` |
| `build_imeta_tag` | no tests |
| `sign_nip98` | no direct test; only observed indirectly through integration tests that assert an `Authorization` header exists (`client.rs:2193-2198`) |
| `extract_d_tag` / `extract_tag_value` / `extract_p_tags` | no tests; `extract_p_tags` contains the group's one bare `unwrap()` on relay data (`client.rs:1379`) |
| `mem` and `notes` command inventories | absent from `subcommand_names_are_stable` — grepping lines 1855-2034 of `lib.rs` for `"mem"` returns zero matches — and from `subcommand_counts_are_stable`, whose list covers 18 of 21 groups (`lib.rs:1997-2016`); the two largest command modules by size have no drift guard |

#### Panic-capable production paths

`advance_query_cursor`'s `.expect("a full query page always has a last event")`
(`client.rs:504-506`) and `extract_p_tags`'s `t.as_array().unwrap()`
(`client.rs:1379`) both violate the AGENTS.md rule against `unwrap`/`expect` in
production paths and both trigger on relay-shaped data rather than programmer
error. `jitter_delay` (`client.rs:133-135`) indexes a fixed 2-element array by
attempt number and is safe only because `retry_constants_are_sensible`
(`client.rs:1547-1552`) pins `RETRY_BASE_SECS.len() == RETRY_MAX_ATTEMPTS - 1`;
raising `RETRY_MAX_ATTEMPTS` alone would panic at runtime.

#### Silent no-op configuration

`--format` reaches only 5 of 21 dispatchers (`lib.rs:1772-1790`), so
`buzz --format compact notes ls` accepts the flag and ignores it. Nothing warns.
For an agent-first CLI whose calling convention is documented around
`--format compact` (`AGENTS.md:179-193`), a silently-ignored output mode is a
correctness trap rather than a cosmetic gap.

#### Documentation drift

| Claim | Reality |
|---|---|
| `AGENTS.md:181` example `buzz messages thread … --format compact` | Cannot parse — `--format` is not `global`; verified exit 1. Directly contradicts `AGENTS.md:192-193`, which states the correct position |
| `AGENTS.md` gotcha 3: "`messages search` must include `--kinds` … Pass at least `--kinds 9,45001,45003`" | `messages search` has no `--kinds` flag (`lib.rs:472-489`); verified `--kinds 9` → exit 1 `unexpected argument`. Kinds are hardcoded downstream (`commands/messages.rs:361`) |
| `AGENTS.md:188-189` "All reads return sig-stripped JSON arrays" | Only `normalize_events` strips `sig` (`client.rs:1307-1323`) and just two command modules call it (`commands/feed.rs`, `commands/messages.rs`); other reads print the relay body verbatim (`commands/social.rs:78`, `commands/issues.rs:32`). Whether the relay's serializer omits `sig` is outside this group and unverified |
| `AGENTS.md:189-190` exit-code table | Matches the six named classes, but omits that `NotFound` also maps to 1 (`error.rs:104`) and `DeliveryUnknown` to 2 (`error.rs:105`), so exit code alone cannot distinguish "may have been written" from "network failed" |
| `AGENTS.md:145-147` documented HTTP surface | The CLI also GETs `/moderation/reports`, `/moderation/restricted`, `/moderation/audit` (`commands/moderation.rs:114-128` via `client.rs:836-856`); `grep -c '/moderation/' AGENTS.md` → 0 |
| `README.md:216-221` architecture diagram (`main.rs ──▶ commands/*.rs`) | `main.rs` is 4 lines; the clap tree and dispatch live in `lib.rs:62-1792`. The diagram also labels the target "Buzz Relay REST API" when the surface is the Nostr HTTP bridge plus Blossom |
| `README.md` command table | Missing 8 of 21 groups: `agents`, `canvas` is present but `emoji`, `notes`, `patches`, `pr`, `issues`, `media`, `moderation` are absent, and `pack`/`mem` are listed. Compare `lib.rs:1808-1829` |
| `README.md:14-20` auth table | Lists only `BUZZ_PRIVATE_KEY`; omits `BUZZ_AUTH_TAG`, which this layer verifies and transmits (`lib.rs:1752-1767`) |
| `TESTING.md:37-73` token/scope model | Contradicted by `lib.rs:1745-1746` ("no tokens, no other auth"); also `TESTING.md:26` points at a stale `REPOS/buzz-nostr` path, and `TESTING.md:82` says "see cargo test for current count" instead of a number |
| `Cargo.toml:19` "(BUZZ_API_TOKEN auto-wired)" | Never read here (`grep -rn 'BUZZ_API_TOKEN' crates/buzz-cli/src` → 0) |
| `Cargo.toml:44` "used in `buzz auth`, auto-mint" | No `auth` subcommand exists (`lib.rs:1808-1829`) |
| `validate.rs:28` "64-character lowercase hex" | Accepts uppercase (`validate.rs:30`) |
| `BUZZ_TIMEOUT_SECS`, `BUZZ_CONNECT_TIMEOUT_SECS` | Documented only in a doc comment (`client.rs:535-540`); absent from `.env.example`, `AGENTS.md` and README |

In-code `file:line` cross-references were checked and are **accurate**:
`client.rs:1077-1080` cites `buzz_ws_client::{AUTH_CHALLENGE_TIMEOUT_SECS,
AUTH_OK_TIMEOUT_SECS, PUBLISH_OK_TIMEOUT_SECS}`, which are 20/20/30 s at
`crates/buzz-ws-client/src/connection.rs:17-23`, summing to the 70 s the comment
claims and justifying the 75 s budget. `Cargo.toml:78-83` correctly points at
`crates/buzz-acp/Cargo.toml` for the matching rustls dependency.

#### Structural debt worth naming

- **Untyped filters.** Every relay query is a hand-built `serde_json::Value`
  (`client.rs:683-801`), so filter-shape errors (a missing `kinds`, a misspelled
  tag key) are relay-side 4xx rather than compile errors. This is the root cause
  of the AGENTS.md p-gate gotchas.
- **Stringly-typed responses.** `submit_event` and friends return `String`
  (`client.rs:863`), pushing JSON re-parsing into 47 `normalize_write_response`
  call sites instead of returning a typed `WriteResult`.
- **`sign_event_unchecked` has no guard rail.** Its "NIP-IA kinds 9035/9036 only"
  restriction is a doc comment (`client.rs:729-742`); a kind assertion would make
  the invariant enforceable rather than reviewable.
- **WS errors lose their class.** Every `buzz_ws_client` failure — timeout, NIP-42
  rejection, connect refusal — becomes `CliError::Other` → exit 4
  (`client.rs:1084`), while the HTTP equivalents are 2/3. Agents branching on
  exit codes see ephemeral publishes as a different failure category from every
  other write.


## Module: buzz-cli — channel, message & social commands (`crates/buzz-cli/src/commands`)
### Aspect: Debt
The dominant debt in these ten files is **duplication of read-path logic** —
three different kind:39000 parsers in one file, two independent `resolve_author`
implementations, two byte-identical compact-output projections — plus a small
number of tests that assert against a copy of a production rule rather than the
rule itself. There is no `TODO`/`FIXME` backlog and no `unsafe`; the problems are
structural.
#### File size against the documented 1,000-line ceiling
| File | Total | Production (pre-`#[cfg(test)]`) | Test region | vs 1,000 |
|---|---|---|---|---|
| `channels.rs` | 1,713 | 1,175 (`#[cfg(test)]` at `:1176`) | 538 (31%) | 1.7× total, **1.2× production alone** |
| `notes.rs` | 1,330 | 820 (`:821`) | 510 (38%) | 1.3× total, production under |
| `messages.rs` | 1,167 | 876 (`:877`) | 291 (25%) | 1.2× total, production under |
| `emoji.rs` | 389 | 325 (`:326`) | 64 | under |
| `social.rs` | 284 | 238 (`:239`) | 46 | under |
| `channel_templates.rs` | 195 | 125 (`:126`) | 70 | under |
| `reactions.rs` | 138 | 138 | 0 | under |
| `dms.rs` | 136 | 136 | 0 | under |
| `feed.rs` | 80 | 80 | 0 | under |
| `mod.rs` | 20 | 20 | 0 | under |
`AGENTS.md § Mobile App` states a "**Hard ceiling: 1000 lines/file**, enforced by
`mobile/scripts/check-file-sizes.mjs` via `just mobile-check` (runs in `just
check` + pre-push, mirroring desktop/web)" and instructs "**split the file —
never bump the limit**". **No Rust gate enforces it.** Proof: the only file-size
checkers in the repo are `desktop/scripts/check-file-sizes.mjs`,
`web/scripts/check-file-sizes.mjs` and `mobile/scripts/check-file-sizes.mjs`;
`grep -rln 'crates/'` across all three returns zero, and the `justfile` wires
them only at `justfile:123` (desktop), `:585` (web) and `:617` (mobile). So
`channels.rs` at 1,713 lines is 71% over a ceiling that is documented as
repo-wide policy and implemented for three of four language surfaces.
#### Oversized functions
| Function | Lines | Range | Note |
|---|---|---|---|
| `channels.rs::cmd_create_channel_from_template` | 133 | `:655-787` | 8 params behind `#[allow(clippy::too_many_arguments)]` (`:654`); does flag validation, template load, roster resolution, channel create, canvas apply, and a sequential member-add loop |
| `messages.rs::dispatch` | 123 | `:754-876` | pure clap-enum-to-struct plumbing |
| `messages.rs::cmd_send_message` | 113 | `:483-595` | stdin, upload loop, media markdown, thread resolution, two mention pipelines, four-arm builder match |
| `channels.rs::dispatch` | 102 | `:1066-1167` | |
| `notes.rs::cmd_set` | 96 | `:487-582` | stdin cap, read-before-write, submit, LWW conflict detection, five-line receipt |
| `emoji.rs::cmd_import` | 75 | `:234-308` | numbered 8-step comment block is doing the work a decomposition should |
| `messages.rs::resolve_content_mentions` | 75 | `:128-202` | 5 numbered stages, none extracted |
Inconsistent remedy for the same problem inside one module: `messages.rs`
introduces parameter structs (`SendMessageParams` at `:474-481`, `SendDiffParams`
at `:577-590`) while `channels.rs` reaches for
`#[allow(clippy::too_many_arguments)]` twice (`:654`, `:794`).
#### Duplication inside the ten files
**Three parsers for kind:39000, all in `channels.rs`.**
| Parser | Site | Fields extracted |
|---|---|---|
| `extract_channel_metadata` | `:16-24` | `channel_id`, `name`, `description` (from `about`), `created_at` |
| `ChannelSummary::from_event` | `:166-211` | `d`, `name`, `t`, `private`/`public`, `about`, `topic`, `purpose`, `archived` |
| inline `--visibility` post-filter | `:68-89` | single-element `public`/`private` tag detection only |
Consequences, not just redundancy: `channels get` (`:224`, via
`extract_channel_metadata`) returns strictly less than `channels search`
(`:119`, via `ChannelSummary`) for the *same event* — no `topic`, `purpose`,
`archived`, `channel_type` or `visibility`. And `channels list` has no archived
filter at all while `channels search` excludes archived by default
(`:136`), so the two read commands over kind:39000 disagree on defaults. The
inline filter at `:68-89` re-derives the single-element-tag rule that
`ChannelSummary::from_event:181-184` already implements. The vocabulary also
splits: `--visibility open` is mapped to the wire value `public` at `:71-73`,
but `ChannelSummary.visibility` reports `"public"` verbatim (`:183`), so input
and output speak different words for the same state.
**Two `resolve_author` implementations.**
| | `messages.rs:394-440` | `notes.rs:204-252` |
|---|---|---|
| Returns | `String` (hex) | `PublicKey` |
| Accepts `"me"` | no | yes (`notes.rs:205-207`) |
| Accepts `npub1…` | yes (`:400-404`) | no |
| Profile query | `{kinds:[0], search, limit:100}` (`:406-410`) | `{kinds:[0], search, limit:100}` (`:213-217`) — identical filter |
| Name match | `match_profiles_by_name` (`:441-476`), extracted + tested | inline closure (`:220-234`), untested |
| Ambiguity error | caps the candidate list at 5 with "… and N more" (`:425-436`) | prints only the count (`:249-251`) |
The exact-case-insensitive `display_name`-or-`name` comparison is written twice
(`messages.rs:456-461`, `notes.rs:224-233`) with the same semantics. `notes.rs`
also has a *third* author-facing formatter, `format_note_candidates`
(`:279-296`), doing the same "list the ambiguous candidates" job in a different
shape.
**Two byte-identical compact projections.** `messages.rs::format_events`
(`:242-262`) and the inline block in `feed.rs:45-64` both build
`{id, content, created_at}` from a normalized array and both fall back to
`unwrap_or_default()`. Neither is shared; `channels.rs:95-109` adds a third,
differently-shaped compact branch. All three live downstream of the same
`--format` flag.
**Two `p`-tag extractors.** `messages.rs::parse_member_pubkeys` (`:226-241`,
canonicalizing through `PublicKey::from_hex`) and the inline closure in
`dms.rs:20-38` (raw strings, no validation) — while `client.rs:1366`
already exports `extract_p_tags`, which `channels.rs:250` does use.
**Duplicated 1–8 pubkey rule.** `dms.rs:52-54` rejects `pubkeys.is_empty() ||
pubkeys.len() > 8`; `buzz_sdk::build_dm_open` enforces exactly the same bound at
`builders.rs:1544-1548`. The CLI never calls `build_dm_open` — it hand-builds
`EventBuilder::new(Kind::Custom(41010), "")` at `dms.rs:69` — so the SDK builder
is dead from this module and its validation is reimplemented.
**Duplicated relay-response parsing.** `notes.rs:544-563` and `notes.rs:723-736`
each hand-parse `accepted`/`message` out of the submit response, and
`notes.rs:557-562` reimplements the `duplicate:` LWW-conflict detection that
`client::normalize_write_response` (`client.rs:1420`) exists to centralize —
the function every other write path in scope calls.
#### Dead and vestigial code
| Item | Evidence |
|---|---|
| `notes.rs::KIND_LONG_FORM` | `notes.rs:38` redeclares `30023`, already `buzz_core::kind::KIND_LONG_FORM` at `kind.rs:66` |
| `emoji.rs::CUSTOM_EMOJI_SET_D_TAG` | `emoji.rs:9` is `const … = buzz_sdk::CUSTOM_EMOJI_SET_D_TAG;` — a pure alias |
| `notes.rs::snapshot_from_event` | `:338-340` is a one-line wrapper around `NoteSnapshot::from_event` with no added behavior |
| `notes.rs:678` `author.unwrap_or("me")` | unreachable — clap supplies `default_value = "me"` (`lib.rs:1069-1071`), so the `Option` is always `Some` |
| `channels.rs` `member: Option<bool>` | `dispatch` always passes `Some(member)` (`:1079`); `cmd_list_channels` tests `member == Some(true)` (`:34`). The `Option` layer can never be `None` |
| `buzz_sdk::build_dm_open` | never called from `buzz-cli`; `dms.rs:69` hand-builds the same event |
No `#[allow(dead_code)]` standing in for a `TODO`: `grep -rn '#\[allow'` over the
ten files returns exactly two lines, both `#[allow(clippy::too_many_arguments)]`
(`channels.rs:654`, `:794`). `grep -rnE 'TODO|FIXME|XXX|HACK'` over the ten files
returns **zero matches**.
#### Stale comments and cross-references
| Comment | Site | Why it is wrong |
|---|---|---|
| "1. Replies referencing this event via e-tag (**no kind restriction**)" | `messages.rs:316-318` | The very next statement restricts kinds to `[9, 40002, 40003, 40008, 45003]` (`:320`) |
| "`build_dm_open` doesn't accept a d-tag, so we build the event manually **using the SDK builder** and add the d-tag ourselves" | `dms.rs:66-67` | No SDK builder is used — `dms.rs:69` calls `EventBuilder::new(Kind::Custom(41010), "")` directly. The "add a d-tag" motive is real; the mechanism described is not |
| "The relay keeps only the latest per (pubkey, d_tag), **but be defensive**" | `emoji.rs:104` | `events.last()` (`:105`) is not defensive — the filter already carries `limit: 1` (`:102`), and if several events did come back, `.last()` picks by array position, not `created_at`. Compare `notes.rs:180-182` and `:361-362`, which sort `Reverse(created_at)` for exactly this case |
| "the relay's #714 coordinate soft-delete **lands in this same change**" | `notes.rs:24-25` | An in-flight-PR note left behind after merge. The relay code exists: `handle_a_tag_deletion` at `buzz-relay/src/handlers/side_effects.rs:1979`, `handle_standard_deletion_event` at `:2108` |
In-code cross-references were checked individually and all **resolve**:
`desktop/src-tauri/src/templates/types.rs` (`channel_templates.rs:5`) exists;
`useApplyTemplate.ts` (`channels.rs:736`) resolves to
`desktop/src/features/channel-templates/useApplyTemplate.ts`; `channelAgents.ts`
(`channels.rs:741`) to `desktop/src/features/agents/channelAgents.ts`; and the
`req.rs` `#d`-pushdown claim (`notes.rs:185-187`) is accurate —
`buzz-relay/src/handlers/req.rs:936-977` pushes a single-value `#d` into the
`d_tag` column, gated on `filter_is_nip33_only` (`:951`), which
`notes.rs::fetch_by_slug` satisfies. The two desktop references cite bare
filenames with no path, which will not survive a file move.
#### Tests that assert against a copy of the rule
This is the module's most consequential test debt.
1. **`BUZZ_ACP_ALLOWED_CHANNEL_ADD_POLICIES` gate.** `channels.rs:1296-1307`
   defines a test-local `check_allowed_channel_add_policy` whose body is a
   verbatim copy of the production gate at `channels.rs:1022-1033` — same split,
   same trim, same filter, same message template. Three tests exercise the copy:
   `set_add_policy_rejects_disallowed_policy` (`:1312`),
   `set_add_policy_accepts_allowed_policy` (`:1330`),
   `set_add_policy_no_restriction_allows_all` (`:1336`). Deleting the gate from
   `cmd_set_add_policy` leaves all three green. The file's own comment at
   `:1345-1350` admits this and adds
   `set_add_policy_env_gate_rejects_disallowed_via_full_path` (`:1362`) as the
   one test that actually calls production. So the honest coverage is 1 real test
   plus 3 tautologies.
2. **Mention pipeline.** `cli_pipeline_resolves_multiword_display_names`
   (`messages.rs:1013`) rebuilds the `name_to_pubkeys` / `display_names` loop
   inline — its own comment says "Simulate the single-parse pipeline from
   `resolve_content_mentions`" (`:1025`). Production's loop lives at
   `messages.rs:161-190` and is never invoked by the test.
3. **The stated regression guard isn't one.**
   `cli_pipeline_resolves_body_at_names_to_member_pubkeys` (`messages.rs:973`)
   documents itself as "the regression guard for the previous stub that always
   returned `vec![]`" (`:969-971`), but it calls `extract_at_names` +
   `match_names_to_profiles` from the SDK, whereas production calls
   `extract_at_mentions_with_known` + a local map (`messages.rs:192-201`). It
   would stay green if `resolve_content_mentions` reverted to a stub.
#### Untested critical paths, grep-proven
| Path | Proof |
|---|---|
| All of `reactions.rs`, `dms.rs`, `feed.rs` | `grep -c '#\[cfg(test)\]'` → 0 for each; `grep -cE '#\[(tokio::)?test\]'` → 0 for each. Includes `reactions remove`'s emoji-match-and-delete (`reactions.rs:42-80`), `dms open`'s relay-id extraction (`dms.rs:72-92`), and `feed get`'s type allowlist (`feed.rs:29-40`) |
| `--format compact` output shaping | `grep -n 'format_events' messages.rs` → definition `:242` and production call sites `:300`, `:336`, `:383`, all above the test boundary at `:877`. `feed.rs` and `channels.rs`' compact branches have no test module / no test importing `cmd_list_channels` (`channels.rs:1178-1184` import list) |
| `emoji import` merge / replace / dedup / dry-run | `grep -n 'cmd_import\|read_source\|write_output' emoji.rs` → only definitions and production call sites (`:164`, `:185`, `:230`, `:234`, `:241`, `:322`); the two tests in the file (`:330`, `:369`) both target `union_custom_emoji` |
| `messages.rs` relay-touching helpers | `grep -n 'resolve_thread_ref\|resolve_channel_id\|resolve_content_mentions'` → all hits are definitions or production call sites (`:58`, `:89`, `:128`, `:524`, `:531`, `:635`, `:679`, `:710`, `:741`); none inside the test module |
| `channel_templates.rs::resolve_templates_path` dev-bundle case | the two path tests (`:139`, `:145`) cover only the override and the prod default; nothing covers `xyz.block.buzz.app.dev` |
| `social.rs` cursor pairing | the five tests (`:242-283`) cover only `validate_social_list_kind`, `parse_tags_json`, `has_d_tag`, `is_parameterized_social_list_kind`. `cmd_get_user_notes` (`:83-113`) is untested |
The `notes` command group is additionally unguarded by the CLI drift tests.
`grep -n 'names(&cmd, ' crates/buzz-cli/src/lib.rs` returns 19 calls covering
`agents, messages, channels, canvas, reactions, emoji, dms, users, workflows,
feed, social, repos, pr, patches, issues, media, upload, pack, moderation` —
`notes` is **not** among them (the `"notes"` string at `lib.rs:1940` is the
`social notes` subcommand inside the `social` list). And
`subcommand_counts_are_stable` (`lib.rs:1996-2033`) enumerates 18 groups,
omitting `mem`, `moderation` and `notes`. `command_inventory_is_stable`
(`lib.rs:1807`) does list `"notes"` at `:1820`, so the *group's existence* is
guarded but none of its four subcommands is. Adding or renaming a `notes` verb
breaks no test.
#### `unwrap()` / `expect()` on production paths
`AGENTS.md § Quality Gates` says "Do not introduce new `unwrap()` or `expect()`
in production paths — use `?` and proper error types". Scanning only the region
above each file's `#[cfg(test)]`:
| Site | Call | Assessment |
|---|---|---|
| `channels.rs:148` | `serde_json::to_string(&matches).expect("serializing ChannelSummary")` | Violates the letter of the rule. Practically unreachable (`ChannelSummary` is all `String`/`Option<String>`/`bool`), but the same file uses `unwrap_or_default()` for the identical operation at `:105` and `:108`, so the module contradicts itself on the idiom |
| `notes.rs:632` | `name.expect("dispatch enforces --name xor --naddr")` | The invariant is real — `validate_get_args` runs at `notes.rs:795` before `cmd_get` — but it is enforced by a *sibling call site*, not by the type. Any second caller of `cmd_get` panics |
| `channels.rs:314`, `:319`, `:703`, `:708` | `unreachable!()` in `match visibility` / `match channel_type` | Guarded by an immediately preceding validating `match` (`:288-305`, `:673-687`). Correct today; a panic-shaped assertion that a `TryFrom` or an enum-typed parameter would make impossible |
Everything else propagates with `?`. `messages.rs`, `emoji.rs`, `social.rs`,
`channel_templates.rs`, `reactions.rs`, `dms.rs`, `feed.rs` and `mod.rs` have
**zero** `unwrap`/`expect`/`panic!`/`unreachable!` in their production regions.
#### Output-contract drift against `AGENTS.md`
`AGENTS.md § Agent CLI` states: "All reads return sig-stripped JSON arrays; all
writes return `{event_id, accepted, message}`". Both halves are violated in
scope.
| Command | Actual output | Site |
|---|---|---|
| `social event` | raw relay response, **`sig` included** — `normalize_events` is never imported by `social.rs` (`social.rs:8`) | `social.rs:78` |
| `social notes` | raw relay response, `sig` included | `social.rs:111` |
| `social contacts` | raw relay response, `sig` included | `social.rs:122` |
| `social list` | raw relay response, `sig` included | `social.rs:205` |
| `social set-list` | raw relay response, not `normalize_write_response` | `social.rs:180` |
| `emoji list` / `export` | `{"emojis": [...]}` — an object, not an array | `emoji.rs:88`, `:228` |
| `reactions get` | `{"reactions": [...]}` — an object, not an array | `reactions.rs:135` |
| `emoji rm` (no-op case) | `{"accepted": true, "message": "not present"}` — **no `event_id`** | `emoji.rs:145-149` |
| `dms open` | hand-assembled object; injects `accepted: true` when the relay omitted it (`dms.rs:90-92`) | `dms.rs:88-93` |
| `notes set` / `rm` | five and two lines of plain-text `key value` receipts | `notes.rs:566-581`, `:738-739` |
| `canvas get` | bare content string or the literal `null` | `channels.rs:270-277` |
| `channels get` | bare object or `null`, not an array | `channels.rs:234-240` |
`normalize_events` (`client.rs:1307-1322`) is the sig-stripping helper; only
`messages.rs` (`:299`, `:335`, `:382`) and `feed.rs` (`:44`) call it.
Also drifting: `social notes --before-id` is accepted without `--before`
(`social.rs:91-93` validates the hex but not the pairing) even though
`buzz-relay/src/api/bridge.rs:1240-1245` returns `400 before_id requires until
to be set`. The caller gets exit **2** (relay/network) for what the exit-code
table at `lib.rs:74` classifies as exit **1** (bad input). Contrast
`messages send-diff`, which *does* validate its own flag pairing locally
(`messages.rs:589-596`).
#### The four `AGENTS.md` leads, verified
**Filters without `kinds`.** Five exist in scope:
| Site | Filter | Trips the p-gate? |
|---|---|---|
| `messages.rs:63` | `{ids, limit}` | No — `ids` exemption, `req.rs:1063-1065` |
| `messages.rs:91-93` | `{ids}` | No — same |
| `messages.rs:328-331` | `{ids, limit}` | No — same |
| `social.rs:74-76` | `{ids}` | No — same |
| `feed.rs:19-22` | `{#p:[self], limit}` | No — `#p` equals the authed pubkey, `req.rs:1067-1069` |
So none of them 403s, but only because both documented exemptions happen to
apply. `AGENTS.md § Common Gotchas #2` states the rule flatly ("omitting `kinds`
triggers the p-gate (403). Always include explicit kind filters"), which
contradicts the actual predicate in
`req.rs::p_gated_filters_authorized:1038-1071`: a kindless filter is only
rejected when it has neither non-empty `ids` nor a self-matching `#p`. Five
in-scope call sites depend on that nuance, undocumented.
**Gotcha #3 documents a flag that does not exist.** `AGENTS.md` says
"`messages search` must include `--kinds` … Pass at least `--kinds 9,45001,45003`".
`MessagesCmd::Search` (`lib.rs:476-489`) declares `query`, `author`, `since`,
`limit` — no `kinds`. Confirmed empirically: `./target/debug/buzz messages search
--kinds 9` → `{"error":"user_error","message":"error: unexpected argument
'--kinds' found …"}`. `cmd_search` hardcodes `kinds: [9, 40002, 45001, 45003]`
at `messages.rs:361`, so the failure mode the gotcha warns about cannot occur.
The same stale text is duplicated in `CLAUDE.md:172` (both docs carry the
identical Common Gotchas block).
**`reply_count` / `descendant_count`.** `AGENTS.md § Key Patterns` requires that
"any code that inserts replies must update these counters". Not applicable here
and correctly so: `grep -rn 'reply_count\|descendant_count' crates/buzz-cli/`
returns **zero matches**. The CLI publishes signed events over
`POST /events`; materialization is the relay's job. No debt — but also no
in-code note pointing a future contributor at that division of labor.
**Kind `39000`, not `41`.** Honored. Channel metadata reads use `39000` at
`channels.rs:54`, `:62`, `:131`, `:227`; there is no bare `41` kind anywhere in
scope. The `41xxx` literals in `dms.rs` (`41001` at `:12`, `41010` at `:69`) are
the distinct DM kinds `KIND_DM_CREATED` / `KIND_DM_OPEN` (`kind.rs:453`, `:447`),
not NIP-01 kind 41. The residual debt is that they are written as literals when
the constants exist — see the Configuration aspect.
**`h`-tag scoping.** Applied inconsistently, some of it deliberately:
| Read | `#h` present? | Site |
|---|---|---|
| `messages get` | yes | `messages.rs:263-268` |
| `messages thread` — reply half | yes | `messages.rs:319-324` |
| `messages thread` — root half | **no** (`ids` only) | `messages.rs:328-331` |
| `messages search` | **no** — cross-channel by design | `messages.rs:360-363` |
| `canvas get` | yes | `channels.rs:265-268` |
| `reactions get` / `remove` | **no** — `#e` (+ `authors`) only | `reactions.rs:83-86`, `:44-48` |
| `feed get` | **no** — `#p` only | `feed.rs:19-22` |
| `dms list` | **no** — `#p` only | `dms.rs:11-15` |
| `notes` (all) | **no** — kind:30023 is not channel-scoped | `notes.rs:171`, `:189`, `:354`, `:681` |
| `channels list` / `get` / `members` | `#d`, not `#h` | `channels.rs:38`, `:55`, `:228`, `:250` |
The `messages thread` root half is the one real gap: the reply half is scoped to
`channel_id` but the root is fetched by bare id, so passing a `--channel` that
does not contain `--event` still returns the root event, silently, in a payload
the caller will read as belonging to that channel. Only relay-side access
control stops a cross-channel read; `cmd_get_thread` itself never checks that the
root's `h` tag matches the `channel_id` it validated at `:312`.
#### Other correctness-adjacent debt
- `reactions.rs:44-48` queries kind:7 with no `limit`, then takes the **first**
  array element whose content matches the emoji (`:56-64`) with no `created_at`
  ordering. If a user reacted, removed, and re-reacted, which reaction gets
  deleted depends on relay row order.
- `notes.rs:191` caps the cross-author `#d` fan-out at `limit: 50`. With more
  than 50 authors on a slug, `notes get --name <slug> --latest` picks the newest
  of a *truncated* set and reports success. The `--latest` doc
  (`lib.rs:1057-1060`) promises "the most recently updated note" unqualified.
- `channel_templates.rs` reads and parses the whole store **twice** on the
  not-found path (`:102` inside `find_template`, `:114` inside
  `available_names`), so a corrupt store produces a parse error from the
  *error-message* code path rather than the lookup.
- `channels.rs:12` imports `fetch_archived_snapshot` from
  `crate::commands::agents` — an in-scope file depending on an out-of-scope
  sibling module for the NIP-IA trust check that
  `resolve_roster_with_archive_filter` (`:526`) is built around.
#### Stated uncertainties
- I did not run `cargo test -p buzz-cli`; every coverage claim above is derived
  from reading the `#[cfg(test)]` regions and from the greps quoted inline.
- The `--format` and `messages search --kinds` empirical checks used the
  prebuilt `target/debug/buzz` (dated 2025-07-26). Both results also follow
  directly from `lib.rs:92-94` and `lib.rs:476-489` as read, so I am confident,
  but they are not from a fresh build.
- Whether the `messages thread` root-half gap is *exploitable* depends on relay
  access control for `POST /query` by bare id, which I traced only as far as the
  p-gate and the accessible-channel scoping at `bridge.rs:997-1001`. I did not
  confirm whether a bare-`ids` lookup is intersected with
  `accessible_channels`, so I am flagging the CLI-side omission, not asserting a
  cross-channel read is possible.


## Module: buzz-cli — repo, agent, memory & moderation commands (`crates/buzz-cli/src/commands`)
### Aspect: Debt

Scope: the eleven files `mem.rs`, `agents.rs`, `repos.rs`, `users.rs`, `pr.rs`,
`patches.rs`, `workflows.rs`, `issues.rs`, `moderation.rs`, `pack.rs`, `upload.rs`.
`lib.rs`/`client.rs`/`validate.rs` are cited only to resolve a call or a rule these
files depend on.

#### Duplication

**`parse_events` exists three times in the crate, two byte-identical.**

| Site | Body | Behaviour |
|---|---|---|
| `repos.rs:11-14` | `serde_json::from_str(json).map_err(…"failed to parse relay response: {error}")` | strict — one bad event fails the whole response |
| `notes.rs:156-159` | byte-identical to the above (same message string, same shape) | strict |
| `mem.rs:114-136` | per-element `serde_json::from_value`, skipping failures | lenient, deliberately (comment at `mem.rs:118-120`, `:126-128`) |

The strict pair is copy-paste; the lenient one is a genuinely different policy that
happens to share a name. Nothing in `client.rs` offers either
(`grep -n 'fn parse_events' crates/buzz-cli/src/client.rs` → zero matches), so the
duplication has no shared home to move to.

**Duplicate-write detection is re-derived instead of shared.** See the LWW lead
below — `submit_engram` (`mem.rs:93-111`) and `repos::validate_write_response`
(`repos.rs:173-193`) implement the same three-rule ladder independently, and only
the latter finishes by calling `client::normalize_write_response`
(`client.rs:1420`).

**The NIP-34 `#a` coordinate is hand-formatted in three files.**
`format!("30617:{repo_owner}:{repo_id}")` at `patches.rs:94`, `pr.rs:129`,
`issues.rs:58`. `buzz-sdk` already models this as `GitRepoCoord`
(`patches.rs:8`, `pr.rs:6`, `issues.rs:4` all import it and use it for the *write*
side), so the read side is the only place the string form is rebuilt by hand — and it
embeds a bare kind integer while doing so.

**The `(repo_owner, repo_id)` pairing match is triplicated verbatim.**
`patches.rs:135-150`, `pr.rs:168-183`, `issues.rs:98-113` — same three arms, same
`"--repo-owner and --repo-id must be given together"` message. Redundant besides:
clap already declares `requires = "repo_id"` / `requires = "repo_owner"`
(`lib.rs:1269-1273`, `lib.rs:1423-1427`, `lib.rs:1501-1505`), so the runtime arm is
unreachable through the CLI.

**The recipient-dedup loop is triplicated verbatim.** `patches.rs:156-163`,
`pr.rs:187-194`, `issues.rs:118-125` — identical `Vec` + `validate_hex64` +
`contains` push. `O(n²)` in `--to` count, which is fine at these sizes but is
copy-paste all the same. Each copy carries its own comment explaining the same rule
(`patches.rs:152-155`, `pr.rs:185-186`, `issues.rs:115-117`).

**`users.rs`'s compact projection is written twice.** The
`OutputFormat::Compact => { … "display_name": p.get("display_name").or_else(|| p.get("name")) … }`
block at `users.rs:63-74` and `users.rs:134-145` are identical apart from the source
variable. Both then repeat the `serde_json::to_string(...).unwrap_or_default()` tail.

**`agents.rs` duplicates the draft-response augmentation.** `agents.rs:26-52`
(draft-create) and `agents.rs:62-88` (draft-update) differ only in the builder call;
the parse, the four `obj.insert` calls, and the identical
`"Draft sent to Buzz Desktop for owner review…"` literal are copied. That literal
appearing twice means a wording change needs two edits.

**64-hex / 128-hex validation is re-implemented rather than calling
`validate::validate_hex64`.** `validate_hex64` exists at `validate.rs:29-36` and is
imported by `agents.rs:9`, yet:

- `agents.rs:231-236` re-checks owner (64) and sig (128) shape inline.
- `agents.rs:251-256` re-checks 64-hex in `normalize_relay_self_hex`.
- `mem.rs:568-574` re-checks 64-hex for `--base-hash`.
- `agents.rs:696` (test) and `verify_archived_event`'s `p`-tag filter
  (`agents.rs:361-365`) each carry a fourth copy of the same predicate.

The `sig`-length-128 check has no shared helper at all, so that one is unavoidable
today; the 64-hex ones are not.

**Event-JSON projection duplicates `client::normalize_events`.**
`workflows.rs:82-89` builds `{event_id, kind, content, created_at, tags}`;
`client::normalize_events` (`client.rs:1307-1322`) builds
`{id, pubkey, kind, content, created_at, tags}`. The shapes differ only in the id key
name and `pubkey`, and no in-scope file calls the shared helper
(`grep -c 'normalize_events'` across the eleven → `0` for every file). `workflows.rs`
also repeats its own projection three times with small variations
(`workflows.rs:23-30`, `:47-53`, `:82-89`).

**`d`-tag extraction is inconsistent.** `workflows.rs` uses the shared
`client::extract_d_tag` (`workflows.rs:24`, `:49`), while `repos.rs:33-45` hand-rolls
`repo_id_from_event` and `mem.rs:212-220` hand-rolls a third variant using
`t.kind().to_string() == "d"` — a string comparison on a tag-kind enum, which is the
most fragile of the three.

#### Stale comments and cross-references

**`users.rs:5` is a stale TODO.**
`// TODO(phase-4): Replace raw nostr::EventBuilder usage in cmd_set_presence with buzz-sdk builder`
— but `cmd_set_presence` already uses `buzz_sdk::build_presence_update`
(`users.rs:299`). `grep -c 'EventBuilder' crates/buzz-cli/src/commands/users.rs`
returns `1`, and that one match is the TODO comment itself. The work is done; the
note is not.

**`workflows.rs:10` is a live TODO, correctly stated.** Same wording, and
`cmd_trigger_workflow` does still construct a raw `EventBuilder`
(`workflows.rs:172-177`) on the `--inputs`-provided branch only, while the
no-inputs branch uses `buzz_sdk::build_workflow_trigger` (`workflows.rs:181`). So
one command has two divergent build paths for the same kind.

**In-code cross-references — both resolve.**

| Reference | Resolves? |
|---|---|
| `agents.rs:263` → `channels::resolve_roster_with_archive_filter`'s doc comment | yes — `channels.rs:526`, and `channels.rs:11` does import `fetch_archived_snapshot` as claimed |
| `mem.rs:428` → "see PR #627 review" | not verifiable from the working tree (no in-repo artifact); the *technical* claim next to it — that a no-context insertion into a non-empty value is rejected rather than mis-applied — matches `mem.rs:421-437` |

`grep -n '\.rs:' ` across the eleven files returns zero matches, so there are no
in-code `file:line` cross-references to go stale. The only other prose pointer is
`workflows.rs:60-66`, whose claim that the relay does not emit kinds 46001-46003 and
that run history lives in the `workflow_runs` table is a statement about another
crate that I did not verify.

**`agents.rs:255-269`'s doc comment describes `verify_archived_event` via
`[`verify_archived_event`]` intra-doc link on a private function** — the link target
is `fn verify_archived_event` at `agents.rs:320`, which is private, so the link
resolves in-crate but not in published docs for the `pub(crate)` caller.

#### `unwrap()` / `expect()` on production paths

`AGENTS.md § Quality Gates` says "Do not introduce new `unwrap()` or `expect()` in
production paths." Filtering out `unwrap_or*` and `#[cfg(test)]` modules, the group
has exactly **one** violation:

- `agents.rs:299` — `let raw_event = events.into_iter().next().unwrap();`

It cannot panic today: `agents.rs:294-296` returns early on `events.is_empty()`
immediately above. It is still a latent panic if that guard is ever moved, and
`.next().ok_or_else(…)` would cost nothing. Every other match is inside
`mod tests` (`mem.rs:789`+, `agents.rs:407`+, `repos.rs:415`+, `patches.rs:285`+,
`pr.rs:340`).

No `unreachable!()` or `panic!()` in the eleven files
(`grep -n 'unreachable!\|panic!'` → zero matches). Note the group's one `unreachable!`
lives in its dispatcher rather than in the module: `lib.rs:1791`
(`Cmd::Pack(_) => unreachable!("handled above")`), guarded by the early return at
`lib.rs:1737-1742`.

**Silent-failure `unwrap_or_default()` on parse paths is the bigger risk than the
unwraps.** `serde_json::from_str(&resp).unwrap_or_default()` turns a malformed relay
response into an empty array with no error and exit code 0:

| Site | Command |
|---|---|
| `users.rs:47` | `users get` |
| `users.rs:263` | `users presence` |
| `workflows.rs:20` | `workflows list` |
| `workflows.rs:45` | `workflows get` (then prints `null`, `workflows.rs:55`) |
| `workflows.rs:79` | `workflows runs` |
| `users.rs:242-243` | `users set-profile`'s read-merge-write — a malformed current profile silently becomes `{}`, so **a merge can drop fields it was supposed to preserve** |

Contrast `mem.rs:95-96`, `agents.rs:36`, `repos.rs:174-175`, which all surface a
parse failure as `CliError::Other`. The inconsistency is the debt: same operation,
two opposite failure policies, no comment explaining either.

`users.rs:242-243` is the one with a correctness consequence rather than just a
misleading exit code, and it is untested — there is no `#[test]` for
`fetch_current_profile` or `cmd_set_profile`
(`grep -n '#\[test\]' crates/buzz-cli/src/commands/users.rs` returns three tests, all
for `presence_subject`: `users.rs:343`, `:349`, `:355`).

#### `#[allow(…)]` attributes

Seven `#[allow(clippy::too_many_arguments)]`, no `#[allow(dead_code)]`
(`grep -n '#\[allow' ` across the eleven returns exactly the seven below; `dead_code`
→ zero matches):

| Site | Function | Arity |
|---|---|---|
| `mem.rs:537` | `cmd_patch` | 8 |
| `pr.rs:19` | `cmd_open_pr` | 16 |
| `pr.rs:65` | `cmd_update_pr` | 12 |
| `pr.rs:151` | `cmd_pr_status` | 10 |
| `patches.rs:8` | `cmd_send_patch` | 13 |
| `patches.rs:113` | `cmd_patch_status` | 12 |
| `issues.rs:80` | `cmd_issue_status` | 8 |

These are suppressions standing in for a refactor that the codebase already knows how
to do: `buzz-sdk` provides exactly the right parameter-object types
(`GitPatchMeta`, `GitPullRequestMeta`, `GitStatusMeta`, `GitIssueMeta`,
`GitPrUpdateMeta`) and each of these functions *constructs one immediately*
(`patches.rs:31-40`, `pr.rs:43-56`, `pr.rs:95-102`, `patches.rs:167-176`,
`issues.rs:21-24`). Taking the meta struct as the parameter instead of 12 loose
arguments would delete the `allow` and the `dispatch` arm's positional-argument
hazard in one move. There is no `TODO` next to any of the seven.

The one clippy-adjacent gap with no `allow`: `dispatch` in `agents.rs:12-154` is a
143-line match arm block, and `mem.rs:737-778`'s dispatch re-lists `cmd_patch`'s
eight arguments positionally (`mem.rs:764-775`) — a silent-reorder hazard that no
test covers.

#### Dead code and unused capability

- No unused items in the eleven files: `cargo`'s own dead-code lint is on (no
  `#[allow(dead_code)]` anywhere here) and every function has at least the local
  `dispatch` as a caller.
- `validate::MAX_CONTENT_BYTES`, `MAX_DIFF_BYTES`, `validate_content_size`,
  `truncate_diff`, `infer_language`, `parse_event_id` are all unreferenced by this
  group (`grep -n 'validate_content_size\|MAX_CONTENT_BYTES\|MAX_DIFF_BYTES\|truncate_diff\|infer_language\|parse_event_id'`
  across the eleven → zero matches). They are live for `messages.rs`, so not dead
  crate-wide, but this group enforces no content ceiling of its own outside `mem`.
- `client::extract_tag_value` (`client.rs:1346`), `extract_p_tags`
  (`client.rs:1366`), `normalize_events` (`client.rs:1307`), `create_response_with_id`
  (`client.rs:1391`), `query_paginated`/`query_all` (`client.rs:715`, `:724`) are all
  unused here — the projection and pagination duplication above is the direct cost.
- `validate::percent_encode` is `#[cfg(test)]`-gated (`validate.rs:76-77`), so the
  crate's only URL-escaper is structurally unavailable to production code. That is
  why `moderation.rs:110-113` interpolates `--status` into a query string raw.

#### File and function sizes vs the documented ceiling

`AGENTS.md:533` documents a **1000-line hard ceiling per file**, enforced by
`mobile/scripts/check-file-sizes.mjs`, "mirroring desktop/web".

| In-scope file | Lines | Over 1000? |
|---|---|---|
| `mem.rs` | 1045 | **yes** |
| `agents.rs` | 718 | no |
| `repos.rs` | 644 | no |
| `users.rs` | 359 | no |
| `pr.rs` | 342 | no |
| `patches.rs` | 323 | no |
| `workflows.rs` | 243 | no |
| `issues.rs` | 198 | no |
| `moderation.rs` | 165 | no |
| `pack.rs` | 151 | no |
| `upload.rs` | 36 | no |

**No Rust gate enforces the ceiling.** The three checkers are
`desktop/scripts/check-file-sizes.mjs`, `web/scripts/check-file-sizes.mjs`,
`mobile/scripts/check-file-sizes.mjs`, wired only into the JS/Dart lanes
(`desktop/package.json:11`, `:15`; `web/package.json:10`, `:13`;
`justfile:617`). `just check` (`justfile:95`) is
`fmt-check clippy desktop-check desktop-tauri-fmt-check desktop-tauri-clippy web-check mobile-check`
— no Rust size step, and `grep -rn 'check-file-sizes' justfile` matches only line 617.
So `mem.rs` at 1045 lines breaches a documented ceiling that nothing checks, and the
sibling out-of-scope modules are further over (`channels.rs` 1713, `notes.rs` 1330,
`messages.rs` 1167 — measured with `wc -l crates/buzz-cli/src/commands/*.rs`). Whether
the ceiling was ever *intended* to bind Rust is genuinely ambiguous: `AGENTS.md:533`
states it inside the **Mobile App § Rules** section, not as a repo-wide rule. Either
the doc should scope it explicitly or a Rust gate should exist — right now it reads
repo-wide and behaves front-end-only.

Oversized functions in `mem.rs` (`grep -n '^pub async fn \|^async fn \|^fn '`):

| Function | Span | Lines |
|---|---|---|
| `cmd_patch` | `mem.rs:538-705` | 168 |
| `cmd_ls` | `mem.rs:189-276` | 88 |
| `verify_hunks_at_declared_position` | `mem.rs:400-483` | 84 |
| `fetch_head` | `mem.rs:136-188` | 53 |

`cmd_patch` does flag validation, base-hash gating, IO, multi-file rejection, parsing,
positional verification, application, size/empty checks, echo, and the write — nine
concerns in one function, and the only one of them with a unit test is the positional
verification (`mem.rs:930`+). `agents.rs`'s `dispatch` (`agents.rs:12-154`, 143 lines)
is the other outlier; the four other in-scope `dispatch` functions are pure routing.

#### Untested critical paths

There is no integration-test directory (`ls crates/buzz-cli/tests` → "No such file or
directory") and no async test anywhere in the group
(`grep -rn 'tokio::test' crates/buzz-cli/src/commands/` → zero matches). Files with
**zero** `#[test]`: `workflows.rs`, `issues.rs`, `moderation.rs`, `pack.rs`,
`upload.rs` (`grep -c '#\[test\]'` → `0` for each).

Untested and consequential:

| Path | Site | Why it matters |
|---|---|---|
| `submit_engram`'s duplicate ladder | `mem.rs:93-111` | the exit-5 producer for `mem set/patch/rm`; only the `repos.rs` twin is tested |
| `resolve_expiry` precedence | `moderation.rs:26-35` | decides whether a ban is permanent or timed |
| `--approved` default-true | `lib.rs:914`, `workflows.rs:193-211` | an omitted flag grants a workflow approval |
| approval `d`-tag = `hex(SHA256(token))` | `workflows.rs:204` | a wrong derivation silently fails to match any pending approval |
| `cmd_trigger_workflow`'s two build paths | `workflows.rs:156-189` | raw `EventBuilder` vs SDK builder for the same kind |
| `users set-profile` read-merge-write | `users.rs:150-214`, `:227-244` | field loss on a malformed current profile |
| `cmd_reports` URL construction | `moderation.rs:105-116` | unescaped `--status` in a NIP-98-signed URL |
| every `dispatch` arm | all eleven | no test constructs a `*Cmd` |
| `pack.rs` path branches | `pack.rs:16-22`, `:53-59`, `:62-63` | |

**Tests that assert against a copy of the rule rather than calling it.**
`mem.rs:1037-1046` (`multi_file_header_count`) re-implements the production predicate
in the test body —
`single.lines().filter(|l| l.starts_with("--- ")).count()` — instead of calling the
code under test at `mem.rs:618`. The production line could change to
`starts_with("---")` or `--- a/` and the test would still pass. It is the clearest
copy-of-the-rule test in the group; the comment above it (`mem.rs:1030-1036`) even
explains the rule it is re-deriving.

Two neighbouring tests assert third-party behaviour rather than this crate's:
`diffy_apply_refuses_mismatched_context` (`mem.rs:885`),
`diffy_apply_succeeds_on_exact_context` (`mem.rs:901`) and
`diffy_roundtrip_preserves_content` (`mem.rs:914`) pin `diffy`, not `mem`. That is
defensible as a dependency-drift canary and the comment at `mem.rs:880-882` says so
— worth noting only because three of the file's fifteen tests exercise no
first-party code.

#### Lead: kindless relay filters and the p-gate

`AGENTS.md § Common Gotchas #2` says a filter without `kinds` triggers a 403. The
actual predicate is `p_gated_filters_authorized`
(`crates/buzz-relay/src/handlers/req.rs:1038-1071`): a kindless filter is treated as
*possibly* p-gated (`is_none_or` at `req.rs:1041-1044`), then exempted if it carries
non-empty `ids` (`req.rs:1064-1066`) or a `#p` all of whose values equal the
authenticated pubkey (`req.rs:1068-1070`).

**Zero in-scope filters are kindless.** Enumerated exhaustively —
`grep -rn 'kinds' ` across the eight files that build filters returns 20 filter
literals plus one comment, and every filter literal carries `kinds`:

| Site | `kinds` |
|---|---|
| `mem.rs:151`, `mem.rs:198` | `[KIND_AGENT_ENGRAM]` |
| `agents.rs:180` | `[0]` |
| `agents.rs:285` | `[KIND_IA_ARCHIVED_LIST]` |
| `repos.rs:21`, `:240`, `:271` | repo announcement (constant, then two literals) |
| `users.rs:42`, `:91`, `:223` | `[0]` |
| `users.rs:258` | `[40902]` |
| `pr.rs:110`, `:131` | `[1618]` |
| `patches.rs:76`, `:96` | `[1617]` |
| `issues.rs:39`, `:60` | `[1621]` |
| `workflows.rs:16`, `:41` | `[30620]` |
| `workflows.rs:74` | `[46001, 46002, 46003]` |

`moderation.rs`, `pack.rs` and `upload.rs` build no filters at all. So the p-gate
exemption analysis is moot for this group — nothing here needs it. Separately, none
of these kinds is in `P_GATED_KINDS` (`crates/buzz-core/src/kind.rs:146-157`), so
the `p_gated_filters_authorized` path returns `true` early for all of them
(`req.rs:1045-1047`). `KIND_AGENT_ENGRAM` is gated by a *different* predicate
documented at `req.rs:1073-1081`, which the `mem` filters satisfy structurally by
always sending both `authors` and `#p` (`mem.rs:151-155`, `mem.rs:198-201`).

The debt here is documentation, not code: `AGENTS.md § Common Gotchas #2` states a
flat rule ("omitting `kinds` triggers the p-gate (403)") that the implementation
contradicts with two exemptions, and `#3` gives `--kinds 9,45001,45003` as the fix
for `messages search` without mentioning that an `ids` lookup needs no kinds at all.

#### Lead: NIP-33 LWW → exit 5, and the `normalize_write_response` split

`mem.rs` **hand-rolls** the detection. `submit_engram` (`mem.rs:93-111`) parses the
`POST /events` reply itself and applies:

1. `accepted` false/absent → `CliError::Other` (`mem.rs:102-104`)
2. `message.starts_with("duplicate:") || message == "duplicate"` →
   `CliError::Conflict` (`mem.rs:105-109`)
3. else `Ok(())`

It does **not** call `client::normalize_write_response` (`client.rs:1420-1435`) —
`grep -n 'normalize_write_response' crates/buzz-cli/src/commands/mem.rs` returns zero
matches, and `mem.rs`'s `use crate::client::BuzzClient` (`mem.rs:29`) imports only the
client type. Consistent with that, `mem set/patch/rm` print human-readable stderr
lines and emit **nothing on stdout** (`mem.rs:370`, `mem.rs:696`, `mem.rs:733`).

`repos::validate_write_response` (`repos.rs:173-193`) implements the same ladder with
the same two message forms, then *does* return
`normalize_write_response(raw)` (`repos.rs:192`) so `repos protect set/remove` emit
the canonical `{event_id, accepted, message}` (`repos.rs:195-199`).

Net: one rule, two implementations, two different output contracts, and the pair is
covered by tests only on the `repos` side (`duplicate_write_response_is_a_conflict`
`repos.rs:619`, `successful_write_response_is_normalized` `repos.rs:629`). Extracting
the ladder into `client.rs` next to `normalize_write_response` would fix the
duplication and let `mem` adopt the standard write shape at the same time. (The
*semantics* of what the relay's `duplicate:` means, including the false-conflict
retry interaction, are covered in Business Rules § NIP-33 LWW.)

#### Lead: drift-guard coverage for `mem` and `moderation`

Three guards in `lib.rs`'s test module:

| Guard | Site | Covers `mem`? | Covers `moderation`? |
|---|---|---|---|
| `command_inventory_is_stable` | `lib.rs:1807-1851` | yes — both appear in the 21-name group list (`lib.rs:1817`, `:1819`) | yes |
| `subcommand_names_are_stable` | `lib.rs:1855-1993` | **no** — `grep 'names(&cmd, "' ` over its body yields 19 group assertions; `mem` and `notes` are absent | yes (`lib.rs:1979-1990`, all 8 names) |
| `subcommand_counts_are_stable` | `lib.rs:1996-2033` | **no** | **no** |

`subcommand_counts_are_stable`'s `expected` list (`lib.rs:1998-2015`) enumerates 18
entries — `agents, canvas, channels, dms, emoji, feed, issues, media, messages, pack,
patches, pr, reactions, repos, social, upload, users, workflows` — omitting `mem`,
`moderation` and `notes`. So the reported "18 of 21" is confirmed, and the omissions
are exactly those three.

Consequence for this group: adding or removing a `mem` subcommand trips **no** guard
beyond the group-name list, because `mem` is missing from both the names guard and the
counts guard. `moderation` is half-covered — the names guard would catch a change, the
counts guard would not, so the two guards disagree about which groups are load-bearing.
And because the counts list is a hand-maintained literal with no cross-check against
`command_inventory_is_stable`'s 21-name list, a new group added tomorrow inherits the
same blind spot silently.

#### Lead: output-contract drift vs `AGENTS.md § Agent CLI`

The stated contract: "All reads return sig-stripped JSON arrays; all writes return
`{event_id, accepted, message}`; creates add the entity ID."

**Reads that are not sig-stripped.** Six commands print the relay's `POST /query`
body verbatim:

| Command | Site |
|---|---|
| `repos get` | `repos.rs:251-253` |
| `repos list` | `repos.rs:279-281` |
| `patches get` | `patches.rs:79-80` |
| `patches list` | `patches.rs:109-110` |
| `pr get` | `pr.rs:113-114` |
| `pr list` | `pr.rs:145-146` |
| `issues get` | `issues.rs:42-43` |
| `issues list` | `issues.rs:75-76` |

The relay serializes the whole event with `serde_json::to_value(&se.event)`
(`crates/buzz-relay/src/api/bridge.rs:1126`), and there is no stripping in that
handler (`grep -c '"sig"' crates/buzz-relay/src/api/bridge.rs` → `0`), nor in any of
the eleven files (`grep -c '"sig"'` → `0` for each). Since a NIP-01 event includes
`sig` by definition, these eight read paths emit `sig`. I did not read the `nostr`
crate's `Serialize` impl, so I am inferring the field's presence from NIP-01 rather
than observing it; the *absence of stripping* on both sides is directly verified.

Commands that comply by accident — they build their own projection, which drops
`sig`: `users get` (`users.rs:50-61`), `users presence` (`users.rs:264-273`),
`workflows list/get/runs` (`workflows.rs:21-31`, `:46-55`, `:80-91`),
`repos protect list` (`repos.rs:147-166`), `mem ls --json` (`mem.rs:263`),
`agents archived` (`agents.rs:312`).

**Reads that are not arrays.**

| Command | Emits | Site |
|---|---|---|
| `workflows get` | a bare object, or the literal `null` when absent | `workflows.rs:46-58` |
| `repos protect list` | a single object | `repos.rs:296-297` |
| `agents archived` | `{"archived": [...]}` — array wrapped in an object | `agents.rs:312` |
| `mem get` | raw value bytes, no newline, not JSON | `mem.rs:293-303` |
| `mem hash` | a bare hex line | `mem.rs:518` |
| `mem ls` (no `--json`) | TSV to stdout, `(no memories besides core)` to stderr | `mem.rs:263-271` |
| `pack validate` / `pack inspect` | human-readable text only | `pack.rs:24-46`, `pack.rs:65-149` |

**Writes that are not `{event_id, accepted, message}`.**

| Command | Actual shape | Site |
|---|---|---|
| `agents archive` | `{ok, event_id, action, target}` — adds `ok`, drops `accepted`/`message` | `agents.rs:111-119` |
| `agents unarchive` | same four-key shape | `agents.rs:140-148` |
| `agents draft-create` / `draft-update` | relay object **plus** `request_id`, `action`, `saved`, `message` | `agents.rs:34-51`, `agents.rs:70-87` |
| `mem set` / `mem patch` / `mem rm` | **nothing on stdout**; a prose line on stderr | `mem.rs:370`, `mem.rs:696`, `mem.rs:733` |
| `repos create` | raw relay body, unnormalized | `repos.rs:227-228` |
| `patches send`, `patches status`, `pr open`, `pr update`, `pr status`, `issues create`, `issues status` | raw relay body (happens to match, but is passthrough not normalization) | `patches.rs:56-57`, `patches.rs:183-185`, `pr.rs:59-60`, `pr.rs:105-106`, `pr.rs:216-218`, `issues.rs:32-33`, `issues.rs:140-142` |
| `upload file` | pretty-printed `BlobDescriptor`, the group's only multi-line output | `upload.rs:8-11` |
| `media get` | raw bytes to stdout or a file | `upload.rs:22-36` |

Compliant writes: `users set-profile` (`users.rs:212`), `users set-presence`
(`users.rs:303`), `workflows update/delete/trigger/approve` (`workflows.rs:134`,
`:148`, `:182`, `:187`, `:210`), all six `moderation` mutations
(`moderation.rs:47`, `:57`, `:75`, `:85`, `:101`), `repos protect set/remove`
(`repos.rs:198`).

**Creates that add the entity ID:** only `workflows create`
(`print_create_response(&resp, "workflow_id", …)`, `workflows.rs:114`). `repos create`
and `issues create` do not — `repos create` prints the raw body with no `repo_id`
(`repos.rs:227-228`) even though the caller supplied the `d`-tag, and `issues create`
returns no issue identifier beyond whatever the relay echoes (`issues.rs:32-33`).

Summary: of the ~54 leaf commands in this group, at least 8 reads leak `sig`, 7 reads are
not arrays, 12 writes deviate from the write shape, and 2 of 3 creates omit the entity
ID. The contract as written in `AGENTS.md § Agent CLI` describes roughly half of this
group's actual behaviour. Whether the fix is code or documentation is a product call I
can't make from the source; what is unambiguous is that agents told to rely on the
documented shapes will mis-parse `mem`, `pack`, `agents` and the eight NIP-34 read
commands.

#### Documentation drift, `crates/buzz-cli/README.md`

- **Six in-scope command groups are entirely missing from the command table**
  (`README.md:81-160`): `agents`, `patches`, `pr`, `issues`, `media`, `moderation`.
  Also missing crate-wide: `emoji`, `notes`. The table documents 13 of 21 groups.
- **`BUZZ_AUTH_TAG` is absent from the auth table** (`README.md:13-15`), which lists
  only `BUZZ_PRIVATE_KEY` — yet `agents draft-create`/`draft-update` hard-require it
  (`agents.rs:156-162`) and `mem` needs it or `--owner` (`mem.rs:37-42`).
- `README.md:3` promises "JSON in, JSON out" and `README.md:25` "All output is JSON on
  stdout"; see the output-contract table above for the seven commands where that is
  false.
- `README.md:1` names `main.rs` as the clap entry point in the architecture diagram
  (`README.md:164-168`). Accurate — `crates/buzz-cli/Cargo.toml` declares
  `[[bin]] path = "src/main.rs"` — but the `Cli` derive and dispatch actually live in
  `lib.rs:63-99` / `lib.rs:1733-1792`, so the diagram under-describes the split.
- `crates/buzz-cli/Cargo.toml:19`'s clap dependency comment reads
  `"derive macros + env var support (BUZZ_API_TOKEN auto-wired)"`. Nothing in this
  crate reads that variable — `grep -rn 'BUZZ_API_TOKEN' crates/buzz-cli/` matches
  only the comment itself, and the crate's three `env =` attributes are
  `BUZZ_RELAY_URL`, `BUZZ_PRIVATE_KEY`, `BUZZ_AUTH_TAG` (`lib.rs:81`, `:85`, `:89`).
  The variable is still live in `buzz-acp` (`crates/buzz-acp/src/config.rs:718`),
  `buzz-workflow` (`crates/buzz-workflow/src/executor.rs:901`) and the desktop agent
  runtime (`desktop/src-tauri/src/managed_agents/env_vars.rs:63`), so the comment is
  stale for `buzz-cli` specifically, not repo-wide.
- `.env.example` documents `BUZZ_RELAY_URL` as `ws://localhost:3000`
  (`.env.example:130`) against the CLI's `http://localhost:3000` default
  (`lib.rs:81`), and never mentions `BUZZ_AUTH_TAG`, `BUZZ_TIMEOUT_SECS` or
  `BUZZ_CONNECT_TIMEOUT_SECS` — see Configuration for the full matrix.

#### Smaller items

- `agents.rs:180` queries kind:0 with `limit: 1` and takes the first event
  (`agents.rs:184-187`) with no `created_at` ordering, unlike `repos.rs:28-30` which
  explicitly sorts by `Reverse(created_at)` before taking one. Whether the relay
  orders `POST /query` results is not asserted at either call site; `repos.rs` defends
  itself, `agents.rs` does not.
- `patches send` accepts `--root` and `--root-revision` simultaneously with no check
  in either clap (`lib.rs:1218`, `:1221` — no `conflicts_with`) or `patches.rs:36-37`.
  Contrast `mem patch`, which enforces its flag pair by hand
  (`mem.rs:551-567`), and `pr open`, which uses `conflicts_with` (`lib.rs:1315-1319`).
  Three different strategies for the same class of problem inside one crate.
- `pr::read_optional_body`'s `(Some, Some)` arm (`pr.rs:11-13`) is dead through the
  CLI because clap already declares the conflict (`lib.rs:1315-1319`,
  `lib.rs:1369-1373`, `lib.rs:1417-1421`); its test (`pr.rs:333-336`) calls the
  function directly, so it passes without proving anything about CLI behaviour.
- `workflows.rs:60-66`'s note that `workflows runs` "will return an empty array"
  documents a command that ships knowingly non-functional, with no `#[test]` and no
  tracking marker beyond the prose.
- `moderation::dispatch` takes `_format` (`moderation.rs:133-137`) — an
  accepted-and-discarded parameter that reads as an unfinished feature; see
  Configuration § `--format` matrix.

