<!-- Analyzed: 2026-07-25T01:12:08Z | Scope: full project -->
# Business Rules

> Status: initialized in Phase 1. Validations, state machines, calculations, and
> authorization rules are populated per-module during Phase 2 and consolidated in Phase 3.

## Summary

Batch 2a documents **344 rules** with `file:line` citations, concentrated in the
foundation layer:

| Module | Rules | Concentration |
|---|---|---|
| `buzz-core` | 152 (BR-1…BR-152) | event verification, NIP-01 filter matching (incl. the `#h` channel fallback and its anti-leak strictness), kind classification, role ladder, tenant host normalization, engram grammar, git push policy, NIP-AB pairing state machine, SSRF ranges |
| `buzz-sdk` | 97 | pre-signing validation (11 shared primitives), tag construction (NIP-29 `h`-tag scoping, NIP-10 threading), 45 per-builder rules, mention resolution, NIP-OA attestation |
| `buzz-persona` | 78 (12 groups) | pack parsing, path-traversal defense, precedence merge (levels 3–5), prompt composition, MCP merge, env projection |
| `buzz-ws-client` | 16 + a 13-step NIP-42 handshake | auth handshake, timeouts, OK correlation, message buffering |

Notable invariants verified in code:
- **Filter `#h` strictness** — an explicit `h` tag is authoritative; the `channel_id`
  fallback applies only when the event carries no `h` tags at all, preventing
  cross-channel leaks (`crates/buzz-core/src/filter.rs:78-100`).
- **Git protection binds everyone including Owner** — an explicit `push:<role>` can never
  weaken a built-in destructive default; the effective minimum is
  `max(explicit, default)` (`crates/buzz-core/src/git_perms.rs:535-552`).
- **Turn-metric owner-only reads** — even the authoring agent cannot read its own metric
  back; only the `p`-tagged owner can (`crates/buzz-core/src/filter.rs:28-32`).
- **Symmetric validation** — `cost_usd` is validated on both encrypt *and* decrypt, so a
  payload written through the lower-level path is still rejected on read
  (`crates/buzz-core/src/agent_turn_metric.rs:174`, `:189-190`).

Batch 2b adds the service layer. `buzz-db` alone documents **136 rules**; `buzz-pubsub`
adds **57**. Concentrations by crate:

| Module | Rules | Concentration |
|---|---|---|
| `buzz-db` | 136 | transactional role enforcement, tenant predicates on every write (one exception: `Db::backfill_d_tags`, `lib.rs:2810`), thread counter materialization, retention/TTL |
| `buzz-pubsub` | 57 | tenant key scoping (8 rules), topic-key grammar (5), subscription lifecycle (8), fan-out (8), presence (6), rate limiting (9), replay protection (5), cross-pod control (5), reconnect (3) |
| `buzz-auth` | scope/session rules | NIP-42 grants 16 scopes; the 14-scope path is dead code |
| `buzz-audit` | hash-chain rules | pre-image field order is community_id (16B) → seq (i64 BE) → created_at (RFC3339 µs) → action → actor_pubkey (presence-tagged) → object_id (presence-tagged) → canonical_json(detail) → prev_hash/GENESIS_HASH |
| `buzz-workflow` | condition + step rules | `evalexpr` conditions; `MAX_DELAY_SECS = 270` (not the documented 300) |
| `buzz-search` | query-shape rules | `community_id` first predicate always; **zero** authorization rules by design |
| `buzz-media` | validation rules | caps are image 50 MB (relay default), video 500 MB, file 100 MB; metadata policy is **reject**, not strip; hash verified on write, not on read |

Notable invariants verified in 2b:
- **Rate limiting is real and fails closed.** Redis error → `AdmissionError::Unavailable`
  → request denied (`crates/buzz-relay/src/admission.rs:29-33`). Correct posture, but it
  puts Redis on the critical path for authenticated *reads*, not just writes.
- **Fixed-window limiter permits 2× burst at boundaries** — self-documented, with a stated
  upgrade path to sliding window / token bucket
  (`crates/buzz-pubsub/src/rate_limiter.rs:9-10`).
- **Pub/sub topics are explicitly *not* an authorization boundary** — the crate says so and
  names the real enforcement point, `filter_fanout_by_access`, which runs on both the
  in-process and cross-node paths (`crates/buzz-pubsub/src/topic.rs:3-6`,
  `src/lib.rs:305-320`).
- **Cross-community subscription isolation is refcounted by `(community, topic)`**, with a
  regression test written specifically to catch the channel-id-only keying bug
  (`crates/buzz-pubsub/src/lib.rs:510-590`).
- **NIP-98 replay TTL is clamped in both directions**, so neither a sub-floor nor a
  `u64::MAX` caller can break the control (`crates/buzz-pubsub/src/nip98_replay.rs:47`).
- **Approval-gate rules exist but are unreachable** — nothing in the repo ever writes the
  `WaitingApproval` state, so the relay's resume path can never run
  (`crates/buzz-workflow/src/executor.rs:663`).

Relay pipeline rules and the full authorization model arrive in batch 2c.

### Batch 2c business rules (relay decisions, git policy, conformance invariants)

- **The per-kind scope gate is inert.** Both transports pass `Scope::all_known()`
  (`crates/buzz-auth/src/lib.rs:137`, `crates/buzz-relay/src/api/bridge.rs:829`), so the
  per-kind denial at `crates/buzz-relay/src/handlers/ingest.rs:1551` can never fire. The
  81-kind scope map is documentation, and all channel-scoped-token logic is dead
  (`channel_ids` hard-coded `None`, `crates/buzz-auth/src/lib.rs:138`).
- **The WS p-gate is skipped for channel-scoped REQ** (`handlers/req.rs:182`) while historical
  delivery runs unconditionally (`req.rs:260-409`). Kind 1059 is absent from
  `is_global_only_kind` (`ingest.rs:379-452`) and from `reader_authorized_for_event`
  (`crates/buzz-core/src/filter.rs:25-27`).
- **The DM-visibility owner fence (kinds 30622 / 44200) is enforced on only one of two
  dispatch paths** — present in `dispatch_persistent_event_inner`
  (`crates/buzz-relay/src/handlers/event.rs:457-491`), absent from `fan_out_pubsub_event`
  (`event.rs:307`), so the cross-node path is unfenced. Contrast the kind-30175 read gate
  added by `ab3af828`, which sits in the shared `filter_fanout_by_access` chokepoint
  (`event.rs:154-175`) and so covers both paths.
- **Side-effect failure is reported as success** — `ingest.rs:2460-2467` only warns and
  `:2513` dispatches anyway, with no metric.
- **A fourth per-event read class exists: kind 30175 is author-only unless it carries
  `["shared","true"]`** (`ab3af828`). Enforced at every read surface through one helper
  (`handlers/req.rs:1222-1234`), with the kind and tag tests in
  `crates/buzz-core/src/kind.rs:192-194`, `:205-216`, `:226-241`, an ingest exact-shape rule
  (`ingest.rs:1034-1093`), and a SQL pre-filter applied before `ORDER … LIMIT`
  (`crates/buzz-db/src/event.rs:504-527`). Unlike the p-gate above it is a result-level
  gate on both transports, so it carries no WS/HTTP asymmetry.
- **Conformance encodes the tenancy rules formally but enforces a small fraction.** The Rust
  predicates cover `Inv_NonInterference` for read observations
  (`crates/buzz-conformance/src/transitions.rs:294-312`) and a fragment of `AuthCheck`
  (`:228-250`) — roughly 1.5 of the 13 invariants in `Safety`
  (`docs/spec/MultiTenantRelay.tla:1129-1142`). All three write arms return `Ok(())`
  unconditionally (`transitions.rs:187`, `:191`, `:198`), so `Inv_ResolutionFence`
  (`tla:1011`) and `Inv_HostBindingFence` (`tla:1038`) — the two invariants stating the
  host/channel fence — contribute nothing at runtime.
- **The one conformance rule that would bite writes is mis-keyed.** `claimed_community_from_event`
  reads the first `h` tag and parses it as a community UUID
  (`crates/buzz-relay/src/conformance/mod.rs:101-119`), but in Buzz `h` carries a **channel**
  UUID. Arming a recording tracer today would fire `IllegalTransition` on nearly every
  authorized channel write.
- **Invariant-name attribution correction:** `Inv_RowZero` is the relay's host-binding seam
  (`crates/buzz-relay/src/tenant.rs:76`, `:291`, `:312`), and
  `Inv_NoFork`/`Inv_Closed`/`Inv_RefEffectApplied`/`Inv_RefDerivedFromParent` belong to
  `docs/spec/GitOnObjectStore.tla:211-253`, consumed by
  `crates/buzz-relay/src/api/git/cas_publish.rs`. Only `Inv_NonInterference`,
  `Inv_ReadConfinement`, `Inv_SanitizedErrors`, and `Inv_AdmissionFence` are referenced by
  `buzz-conformance`. `Inv_LabelPropagation` (`tla:990`) has no Rust predicate.

### Batch 2d business rules (agent turn loop, work queue, CLI dispatch)

The rules in 2d divide into three clusters: the **turn loop and cancellation** semantics in
`buzz-agent`, the **queue/pool scheduling** rules in `buzz-acp`, and the **retry/idempotency**
rules in `buzz-cli`. The last of these is the best-specified logic in the batch; the first two
contain several ordering contracts enforced only by comment.

| ID | Finding | Location |
|---|---|---|
| BR-153 | **The turn loop's step order is load-bearing and fixed.** Each round: round cap → cancel poll → steer drain → handoff gate → tool catalog assembly → `round += 1` → provider call (`agent.rs:88-120`). The steer drain deliberately precedes the handoff decision so a queued steer is folded into the summarized context rather than lost to it. A `Performed` handoff clears **both** `last_request_input_tokens` and `last_request_history_bytes` (`agent.rs:104-106`) so a stale over-threshold reading cannot immediately re-fire; `Skipped` falls back to `truncate_history` (`:111`). | `crates/buzz-agent/src/agent.rs:88-120` |
| BR-154 | **Cancel always wins, and the implementation is unusually careful.** Every await uses `tokio::select! { biased; _ = cancel.changed() => … }` — the provider call (`agent.rs:121-146`), the handoff summarizer (`handoff.rs:45-48`), the tool `JoinSet` (`agent.rs:424-441`). On cancel the semaphore is **closed rather than tasks aborted** (`agent.rs:430-435`) so each in-flight MCP call can send its own `notifications/cancelled`; the drain is bounded to 5 s then `abort_all()` (`:455-478`); every unfilled slot becomes a synthetic `cancelled` result plus a terminal `failed` update so no client is left with a permanently `pending` tool (`:483-489`); and results are appended to history even on the cancel path (`:311`, `:341`) which is what keeps history valid for the next prompt (test `tests/regressions.rs:456`). | `crates/buzz-agent/src/agent.rs:300-313`, `:424-489` |
| BR-155 | **Requeue MUST precede `mark_complete`, enforced only by a comment.** `mark_complete` reads `retry_after` (`queue.rs:397-409`) and only `requeue` writes it (`:496`); inverting the order silently resets every retry to attempt 1. No `debug_assert!`. The contract is stated at `queue.rs:426-427` and `lib.rs:3061-3065`. | `crates/buzz-acp/src/queue.rs:397-409`, `:426-427`, `:496` |
| BR-156 | **The cancelled-batch fallback bypasses the retry throttle and is non-deterministic.** `queue.rs:308-312` and `:586-590` consult only `in_flight_channels`, never `retry_after`, and channel selection is `keys().find()` over a `HashMap` with no tie-break (`:299`), so which channel is served next is unspecified. Untested. | `crates/buzz-acp/src/queue.rs:299`, `:308-312`, `:586-590` |
| BR-157 | **`--subscribe-mode all` without `--kinds` is accepted and produces a filter that matches everything locally and nothing remotely.** The REQ goes out with no `kinds`, which the relay p-gate rejects (403), while `filter.rs:383` treats the empty set as a wildcard for local matching. No warning is emitted, and `test_all_mode_wildcard` (`config.rs:1630`) asserts the wildcard as correct. | `crates/buzz-acp/src/config.rs:1180`, `:1195-1196`, `:1630`; `crates/buzz-core/src/filter.rs:383` |
| BR-158 | **Steering is queue-only and applied at round boundaries.** A steer is accepted only when all of: non-empty prompt, non-empty `expectedRunId`, known session, `active_run_id.is_some()`, id match, and channel send succeeds (`lib.rs:554-605`). It is folded into history at the next round boundary and never mid-round; the turn is not restarted (`agent.rs:94-98`, `:265-280`). An unrenderable steer is dropped with `warn!` rather than aborting the turn (`:274-278`). **Consequence:** a steer arriving after the final round is queued and never drained, because the loop returns before the next boundary (`:207-240`) — and nothing reports the drop to the client. | `crates/buzz-agent/src/lib.rs:554-605`; `agent.rs:207-240`, `:265-280` |
| BR-159 | **`_Stop` hooks can override only `end_turn`, with a per-prompt budget.** Consulted only when the mapped stop reason is exactly `EndTurn` (`agent.rs:217-219`), so `max_tokens` and `refusal` are never overridden. `stop_rejections` is a loop-local counter (`:86`) checked before calling hooks (`:220-222`), so the budget resets per prompt; `cfg.stop_max_rejections = 0` disables hooks entirely. Objections are injected as synthetic assistant-tool-call + tool-result pairs, never as user or system messages (`:660-695`). The path is **fail-open**: `call_hooks` drops errors, timeouts and blank output (`mcp.rs:309-313`), so a broken hook cannot trap the agent. Hook tools are unreachable from the model by two independent rules — filtered out of `mcp.tools()` and answered `unknown tool` on direct invocation (`agent.rs:330-337`). | `crates/buzz-agent/src/agent.rs:217-236`, `:330-337`, `:660-695` |
| BR-160 | **The handoff gate uses `>=` on the token path and `>` on the byte path**, an off-by-one asymmetry with no stated rationale (`handoff.rs:111-113` vs `:115-126`). Everything else about the gate is well specified: cap at `max_handoffs` falls back to truncation rather than erroring (`:34-41`); an empty or failed summary yields `Skipped` rather than a hard error (`:54-63`); history is cleared **before** `_PostCompact` runs so the hook injects into the fresh context (`:66-77`); and the fresh context is exactly one `User` item holding `[Context Handoff]\n{summary}` plus the current prompt re-appended verbatim (`:84-95`), because orphaned tool results would violate the OpenAI Chat/Responses ordering requirement (`:78-83`). `CONSERVATIVE_BYTES_PER_TOKEN = 1` (`:326`) biases the gate toward handing off early, deliberately. | `crates/buzz-agent/src/handoff.rs:34-129`, `:326` |
| BR-161 | **`truncate_history` can silently return over budget.** It drops whole leading segments up to the next `User` item so the window always begins at a user turn and never orphans a tool result — but if no later `User` item exists it `break`s and returns over budget (`agent.rs:723-725`), with no warning. It also uses `estimated_bytes` (real wire size) rather than `context_pressure_bytes`, deliberately per `types.rs:24-28`. The over-budget branch has no test. | `crates/buzz-agent/src/agent.rs:711-738` |
| BR-162 | **Provider selection has no fallback, and the auto-upgrade latch is one-way and process-wide.** `Llm::complete` dispatches on `cfg.provider` in a single three-arm `match` (`llm.rs:132-215`) with no cross-provider retry — the arms are unchanged in shape by `16d4ec33`; mesh routing resolves the request model *before* this match runs, so it swaps which string lands in `effective_model` without touching the match itself. The Chat→Responses upgrade fires only from `AgentError::Llm` (never auth or transport), matches on a lowercase substring test against the provider's error body (`is_responses_required_error`, `llm.rs:963-968`), and latches via a one-way `AtomicBool::swap(true)` in `try_upgrade` (`llm.rs:657-673`) that is never reset (field declared `llm.rs:94`, initialized `:117`). The matcher is well tested (`llm.rs:2448-2462`); the latch itself is not. | `crates/buzz-agent/src/llm.rs:132-215`, `:544`, `:657-673`, `:963-968` |
| BR-163 | **Retry and backoff in `buzz-agent`: 3 attempts, `min(500 << attempt, 8000)` ms with uniform jitter subtracted, and a documented set of retryable conditions.** All of this now lives in `post` (`llm.rs:1390-1517`), which since `16d4ec33` returns `Result<Value, PostError>` rather than a bare `AgentError` — the wrapper is new, the retry semantics it wraps are not. Transport errors where `is_timeout() || is_connect() || is_request()` (`llm.rs:1311-1313`) — the `is_request()` term catches a socket accepted then dropped before any response bytes, regression-tested at `llm.rs:3122`. Any 5xx, plus 429, plus the non-standard 499 (`llm.rs:1444`) — **unless** the new mesh-fallback classifier claims the body first (`:1446-1451`). Non-retryable: 401/403 (routed to a refresh-once path instead), 404, any other non-2xx, JSON decode failure, and — new — a classified mesh failure, which returns `PostError::MeshFallback` for the caller to handle instead of retrying in place. A **body-read error mid-stream is explicitly not retried** (`llm.rs:1499-1512`) even though the request was already sent, with no stated reason and no test. On RNG failure the full backoff base is used with no jitter (`llm.rs:1295-1300`). The retry loop still ends in `unreachable!()` (`llm.rs:1516`), guarded only by `MAX_RETRIES >= 1` (`llm.rs:1285`). | `crates/buzz-agent/src/llm.rs:1285-1517` |
| BR-164 | **Auth recovery is refresh-once-per-call, and 403 is deliberately treated as refreshable.** A bearer is fetched once before the loop (`llm.rs:630`); on the first `LlmAuth` the source is force-refreshed with the *rejected* bearer and the request retried once (`llm.rs:642-649`). The `refreshed` guard is a local, so an earlier turn cannot suppress a later turn's retry — rationale at `llm.rs:622-629`. Treating 403 as refreshable costs one wasted refresh on a pure authorization denial, an explicit trade documented at `llm.rs:1433-1438`. All four 401/403 × recoverable/persistent combinations are tested with exact refresh counts (`llm.rs:3650`, `:3686`, `:3717`, `:3748`). The mesh catalog probe (new in `16d4ec33`) fetches its own bearer per probe (`llm.rs:474`) but has none of this machinery — a 401 there is just logged at `debug` and treated as `Unknown` (`:500-506`), with no refresh attempt. | `crates/buzz-agent/src/llm.rs:474`, `:622-649`, `:1433-1438` |
| BR-179 | **New in `16d4ec33`/`8eb6e3eb`: mesh "Auto" collective routing and live Databricks discovery are both governed by exact-match, not substring, classification — consistent with `config.rs`'s discipline, not `databricks_v2_route_for_model`'s naked `contains`.** Eligibility for mesh routing requires `Provider::OpenAi` **and** `cfg.prefer_mesh_for_auto` **and** `effective_model == "auto"` exactly (`llm.rs:411-415`); the collective is adopted only after two consecutive catalog probes confirm a virtual `mesh` entry plus ≥2 distinct physical models (`llm.rs:441-446`), abandoned on the first non-qualifying probe (`:447-451`), and a mesh-specific 5xx or unstructured-tool-call response triggers exactly one fallback retry through `auto` plus a 30 s cooldown (`:361-390`, `:397-404`). Separately, `catalog.rs`'s live-discovery path (`8eb6e3eb`) now filters out endpoints whose name contains "embedding" or a `-`-delimited `bge`/`gte` segment (`is_chat_capable_endpoint`, `catalog.rs:97-106`), sorts survivors newest-first then by name (`catalog.rs:337-344`), and — for `DatabricksV2` — falls back to the configured model first, trimmed and deduplicated, followed by `DATABRICKS_V2_KNOWN_MODELS` with that model filtered out (`catalog.rs:51-80`), rather than the known-list-only fallback this document previously described. | `crates/buzz-agent/src/llm.rs:361-469`; `catalog.rs:51-106`, `:337-344` |
| BR-165 | **Thinking-effort validation is asymmetric by design, and one documented rule is only emergent.** Only `Provider::Anthropic` rejects `none`/`minimal` at startup (`config.rs:941-948`); the other three defer to request time because `session/set_model` can change the model later — rationale at `:932-938`. All four providers × 7 levels are tested. But `config.rs:459` states as a rule that "`xhigh` falls back to `high` when not supported"; nothing implements it — it is an emergent property of the nearest-ordinal-distance sort at `:486-491`, so a future table supporting `max` but not `xhigh` would resolve *upward* to `max`, contradicting the documented rule with every existing test still green. The distance metric additionally depends on `ThinkingEffort`'s variants being contiguous 0..=6 by declaration order (`:19-27`), guaranteed by nothing — `thinking_effort_ord_ordering` (`:1387`) tests the `Ord` relation but not the cast values. | `crates/buzz-agent/src/config.rs:19-27`, `:459`, `:486-491`, `:932-949` |
| BR-166 | **Anthropic thinking config chooses among three shapes and prefers omission over guessing.** After stripping catalog prefixes (`config.rs:134`), a model is classified manual-budget, adaptive, or unknown (`:136`, `:161`, `:171-176`); unknown emits neither field. On the manual path `budget = min(level_budget, max_output_tokens - 1024)`, and if the result is `< 1024` thinking is **omitted entirely** with a `warn!` rather than sending an answer-starving budget (`:141-158`). Both sides of the 2048 boundary are tested, and five hypothetical `claude-*` names are asserted to omit rather than guess (`:1534-1557`). | `crates/buzz-agent/src/config.rs:124-178` |
| BR-167 | **`buzz-cli` runs two retry policies chosen by event kind, and the ambiguity classifier is the real rule.** `is_moderation_kind` = 9040..=9044 (`client.rs:211-213`). For those kinds a timeout/request/body/decode failure is **not** retried and becomes `DeliveryUnknown` immediately (`:911-923`), and a non-canonical 429 or a 502-504 likewise (`:958-972`) — so a ban cannot be applied three times. For all other kinds those failures *are* retried and exhaustion maps through `is_stored_event_exhaustion_ambiguous` (`:229-248`): connect errors and canonical `rate-limited:` 429s are **not** ambiguous (stay retryable), everything else is. A fresh UUIDv4 NIP-98 `nonce` is minted per attempt specifically so retries survive the relay's replay guard (`:886-887`, `:1041-1042`). Twelve behavioural integration tests cover this over live servers (`:1582-2295`). | `crates/buzz-cli/src/client.rs:211-248`, `:638-681`, `:873-1071` |
| BR-168 | **Pagination is an automatic composite cursor with no page or memory cap.** `query_pages` requests `min(remaining, QUERY_PAGE_SIZE=500)` (`client.rs:687-691`); a short page ends the loop, a full page advances `(until, before_id)` from the last event (`:692-700`), and `advance_query_cursor` validates the id as 64 ASCII hex before reuse (`:511-514`). `query_all` (`:724-727`) loops until a short page with no cap, accumulating every event in a `Vec`. | `crates/buzz-cli/src/client.rs:498-519`, `:683-727` |
| BR-169 | **`sign_event` enforces the "no manual auth tags" rule at runtime**, not by comment: it injects the ambient NIP-OA tag then counts `auth` tags on the signed event and errors if the count differs from the expected 0-or-1 (`client.rs:588-614`). `sign_event_unchecked` (`:743-747`) bypasses both injection and the count check and is reserved for NIP-IA kinds 9035/9036 **by doc comment only** (`:729-742`) — nothing restricts it to those kinds. Its two real callers are `agents archive`/`unarchive` (`agents.rs:105`, `:135`). | `crates/buzz-cli/src/client.rs:588-614`, `:729-747` |
| BR-170 | **Media rules are the CLI's most tightly specified validation chain.** `media_url_from_input` (`client.rs:270-323`) enforces in order: absolute URLs must be `http(s)`, the path must start `/media/`, the segment must be `sha256`, `sha256.ext` or `sha256.thumb.jpg` with lowercase hex and an `[a-z0-9]{1,8}` extension (`:250-268`), and the origin (scheme + host + effective port) must equal the configured relay — otherwise "refusing to sign media GET for a non-relay origin" (`:305-315`). Upload requires a regular file, sniffs MIME from magic bytes rather than the extension (`:1109-1112`), enforces a 5-entry allowlist, and applies 50 MiB image / 500 MiB video caps. Endpoint fallback is narrow by rule: only 404 and 405 switch to the legacy path, and the switch is not itself retried (`:200-207`), with a test pinning the narrowness (`:483-496`). | `crates/buzz-cli/src/client.rs:250-385`, `:1100-1227` |
| BR-171 | **`h`-tag scoping is applied inconsistently across CLI reads, and one gap is real.** Scoped: `messages get` (`messages.rs:263-268`), the reply half of `messages thread` (`:319-324`), `canvas get` (`channels.rs:265-268`). Not scoped, deliberately: `messages search` (cross-channel by design), `notes` (kind 30023 is not channel-scoped), `reactions get`/`remove` (`#e` + authors), `feed get` and `dms list` (`#p`). The one real gap is the **root half of `messages thread`**, which fetches by bare `ids` (`messages.rs:328-331`) while the reply half is scoped to `channel_id` — so passing a `--channel` that does not contain `--event` still returns the root event, silently, in a payload the caller will read as belonging to that channel. `cmd_get_thread` never checks that the root's `h` tag matches the validated `channel_id` (`:312`). | `crates/buzz-cli/src/commands/messages.rs:263-268`, `:312`, `:319-331` |
| BR-172 | **`reply_count` / `descendant_count` maintenance is correctly *not* the CLI's job**, and there is no code here that could violate the `AGENTS.md` rule — zero matches for either identifier across `crates/buzz-cli/`. The CLI publishes signed events over `POST /events`; materialization is relay-side. Worth recording because the rule as written in `AGENTS.md § Key Patterns` reads as though it binds any reply-inserting code. | verified by grep across `crates/buzz-cli/` |
| BR-173 | **Two ordering rules in `buzz-cli` are non-deterministic under realistic data.** `reactions remove` queries kind 7 with **no `limit`**, then deletes the *first* array element whose content matches the emoji, with no `created_at` ordering (`reactions.rs:44-64`) — so after react/remove/re-react, which reaction is deleted depends on relay row order. And `notes get --name <slug> --latest` caps the cross-author fan-out at `limit: 50` (`notes.rs:191`), so with more than 50 authors on a slug it picks the newest of a *truncated* set and reports success, while the flag's help promises "the most recently updated note" unqualified (`lib.rs:1057-1060`). Contrast `repos.rs:28-30`, which explicitly sorts by `Reverse(created_at)` before taking one. | `crates/buzz-cli/src/commands/reactions.rs:44-64`; `notes.rs:191` |
| BR-174 | **Agent-draft creation is the only human-in-the-loop gate in the CLI**, and it is structural rather than advisory: `agents draft-create`/`draft-update` publish a NIP-44-encrypted observer frame to the owner and print `"saved": false` with an explicit "Nothing changes until the owner saves it" message (`agents.rs:38-41`, `:80-83`) — the CLI never writes an agent record itself. An update must change at least one field (`agent_management.rs:169-179`), `respond_to` is checked twice (value enum plus a string-level allowlist at `:148-155`), and every field is trimmed, non-empty and char-capped (`:70-85`). The frame is kind 24200 with `p`, observer-agent and observer-frame tags and deliberately **no `h` tag**, asserted at `:225-227`. | `crates/buzz-cli/src/agent_management.rs:70-85`, `:148-179`, `:225-227` |
| BR-175 | **`workflows runs` is wired to a surface that does not exist.** It queries kinds 46001/46002/46003, and its own doc comment (`workflows.rs:60-64`) states those events are never emitted because run history lives in the `workflow_runs` table — so the command ships knowingly returning `[]`, with no test and no marker beyond the prose. Related: `workflows create`/`update` publish the YAML verbatim with **no client-side parse or validation** (`:104`, `:127`); the only gate is the SDK's 64 KiB byte cap (`builders.rs:1468`, `:1486`). `buzz-workflow` is not a declared dependency of `buzz-cli`, so a malformed `if:` expression, an unknown step type, or an invalid document is accepted locally and fails only relay-side. | `crates/buzz-cli/src/commands/workflows.rs:60-64`, `:104`, `:127` |
| BR-176 | **`workflows approve` defaults to granting.** `--approved` is declared `default_value_t = true` with `ArgAction::Set` (`lib.rs:914`), so omitting the flag approves. The approval `d` tag is `hex(SHA256(approval_token))` rather than the raw UUID (`workflows.rs:204-206`) — a convention documented in a single line comment, with no test, where a wrong derivation silently fails to match any pending approval. | `crates/buzz-cli/src/lib.rs:914`; `commands/workflows.rs:204-206` |
| BR-177 | **`mem patch` applies a strict-position rule that `diffy` alone would not enforce**, and this is the best-tested first-party logic in the CLI. `verify_hunks_at_declared_position` (`mem.rs:400-483`) rejects a hunk that `diffy::apply` would happily slide to a different offset; `strict_position_rejects_offset_slide` (`:928`) first demonstrates the slide, then asserts the refusal — a genuinely discriminating test. A no-context insertion into a non-empty value is rejected rather than mis-applied (`:421-437`). `mem patch` also requires exactly one of `--base-hash` or `--no-base-hash` (`:551-567`), enforced in code rather than by clap `conflicts_with`, and `mem rm core` is refused outright (`:713-716`). | `crates/buzz-cli/src/commands/mem.rs:400-483`, `:551-567`, `:713-716` |
| BR-178 | **Repo protection rules are the write end of a contract the relay enforces**, and the two agree only because both call into `buzz-core`. `repos protect set/remove` writes `buzz-protect` tags via `build_protection_tag` (`repos.rs:64-90`); the relay's git policy hook reads them back through `parse_protection_tags` + `evaluate_push` (`crates/buzz-relay/src/api/git/policy.rs:45`, `:285`) served at `POST /internal/git/policy`. `buzz_core::git_perms` is referenced by exactly three files repo-wide. At least one rule (including `--push`) is required, enforced by the builder rather than clap (`repos.rs:79-81`, test `:521`), and there is a 50-rule-per-repo ceiling enforced in `buzz-core` and surfaced at `repos.rs:120-125`. | `crates/buzz-cli/src/commands/repos.rs:64-90`, `:120-125` |

Rules enforced only by comment or convention in 2d, collected: requeue-before-`mark_complete`
(BR-155); `sign_event_unchecked` restricted to NIP-IA kinds (BR-169); the Responses replay ordering
invariant `function_call` before `function_call_output` (`llm.rs:867-872`, relying on `HistoryItem`
insertion order); `MAX_RETRIES >= 1`, asserted only inside the `unreachable!()` message it protects
(`llm.rs:1516`); `for_discovery`'s exemption from `validate` (`config.rs:839-843`, enforced by
nothing); the `*` wildcard in `MCP_HOOK_SERVERS` being honoured only as a sole entry, resting on a
cross-module assumption that `*` cannot pass the MCP name validator (`config.rs:1104-1110`);
`unique_nonce`'s stated uniqueness target, satisfied by a process-wide `AtomicU64` with no wrap
handling (`agent.rs:693-696`); "a live run always has a `steer_tx`" (`lib.rs:599-601`), which
the code degrades around rather than checking; and, new in `16d4ec33`, `MESH_MOA_UNAVAILABLE_MESSAGE`
(`llm.rs:34`) matching the mesh gateway's 503 text exactly with nothing linking the two sides of
that contract, and `prefer_mesh_for_auto` only being meaningful when the configured base URL
actually points at a mesh router — the eligibility guard checks only `Provider::OpenAi`
(`llm.rs:411`), so pointing an unrelated OpenAI-compatible gateway at `auto` with the flag on
makes the agent probe `{base_url}/models` on every 5 s window for no operational reason.

## Validation Rules

| Rule | Scope | Enforcement Point | Source |
|---|---|---|---|
| _pending_ | | | |

## State Machines

_Pending Phase 2 (connection auth states, workflow run statuses, huddle room lifecycle,
channel member states)._

## Calculations & Derivations

_Pending Phase 2 (thread `reply_count`/`descendant_count` materialization, audit hash
chain, presence/typing windows)._

## Authorization Rules

_Pending Phase 2 (channel membership gating, scopes, roles, owner gates, community
fencing)._

---

# Phase 2 — Module Findings

## Module: buzz-core (`crates/buzz-core`)

### Aspect: Business Rules

Every rule below is read from the implementation. "Trigger" = the call path that evaluates the rule.

---

### 1. Event verification (`src/verification.rs`)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| BR-1 | Event ID must equal the canonical NIP-01 hash of `(pubkey, created_at, kind, tags, content)`. On mismatch the recomputed ID and the claimed ID are both reported. | `verification.rs:12-25` | `verify_event()` — called by relay ingest (doc says call via `spawn_blocking`, `verification.rs:1-2`, `:9`) |
| BR-2 | Schnorr signature over the event ID must validate; checked **only after** the ID check passes. | `verification.rs:27-29` | same |
| BR-3 | Verification order is ID-then-signature, never the reverse. | sequential structure `verification.rs:12-31` | same |

Test evidence: `rejects_tampered_id` (`verification.rs:44-58`) and `rejects_tampered_signature` (`:60-68`); `event.rs:57-66` additionally asserts that tampering only `sig` leaves `verify_id()` true while `verify_signature()` is false.

---

### 2. NIP-01 filter matching (`src/filter.rs`)

Top-level semantics: multiple filters are OR-ed (`filters_match` uses `.any()`, `filter.rs:10-13`); fields inside one filter are AND-ed (sequential early-`return false` in `filter_match_one`, `filter.rs:35-104`).

| # | Rule | Enforced at | Notes / edge cases |
|---|------|-------------|--------------------|
| BR-4 | `kinds`: **absent** (`None`) → no kind constraint; **present** → event kind must be a member. | `filter.rs:36-40` | An explicitly **empty** `kinds` vector, if constructible, would make `contains` false and reject everything. The code does not special-case empty-vs-absent; only `Option` presence is checked. |
| BR-5 | `authors`: absent → unconstrained; present → `event.pubkey` must be a member. | `filter.rs:42-46` | Exact match on `PublicKey`; **no prefix matching for authors** (contrast BR-7). |
| BR-6 | `since`: reject when `event.created_at < since` (i.e. `since` is inclusive). | `filter.rs:48-52` | |
| BR-7 | `until`: reject when `event.created_at > until` (inclusive). | `filter.rs:54-58` | |
| BR-8 | `ids`: **prefix matching** — event id hex must `starts_with` at least one filter id hex. | `filter.rs:60-66` | Comment cites NIP-01 prefix allowance (`filter.rs:60`). Since `EventId::to_hex()` is always 64 chars, the practical effect is full-equality unless a shorter id type is supplied. |
| BR-9 | Generic tag filters (`#e`, `#p`, `#h`, …): for each tag key in the filter, at least one of the filter's values must equal the content of at least one event tag of that key. Different tag keys are AND-ed; values within one key are OR-ed. | `filter.rs:68-77` | Matching compares `tag.kind().to_string()` against the filter key string and `tag.content()` against the value. |
| BR-10 | `#h` fallback: if no h-tag matched **and** the event carries **no** h-tags at all, fall back to `StoredEvent.channel_id`; the filter's `#h` values must contain that UUID string, otherwise reject. If `channel_id` is `None`, reject. | `filter.rs:78-93` | Rationale in code: reactions (kind 7) and deletions (kind 5) derive channel from their target and carry no h-tag (`filter.rs:78-82`). |
| BR-11 | `#h` strictness: if the event **does** have h-tags but none matched, reject — the tag is authoritative and `channel_id` must not override it. | `filter.rs:94-100` | Test `h_tag_fallback_uses_stored_channel_id` (`filter.rs:188`) asserts this anti-leak property at `filter.rs:229-235`. |
| BR-12 | Empty filter list matches nothing (`[].iter().any(..) == false`). | `filter.rs:10-13` | Asserted in `or_semantics` at `filter.rs:173`. |
| BR-13 | A filter with no constraints at all matches every event (all `Option`s `None`, no generic tags → falls through to `true`). | `filter.rs:103` | Implicit; no test pins this. |

#### Result-level read gate

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| BR-14 | For `KIND_DM_VISIBILITY` (30622) and `KIND_AGENT_TURN_METRIC` (44200), the reader's pubkey hex must appear as the content of a `p` tag on the event; all other kinds are unconditionally authorized. | `filter.rs:23-33` | `reader_authorized_for_event()` — doc lists the three call sites it guards (WS historical pull, HTTP bridge, live fan-out) at `filter.rs:19-22` |
| BR-15 | The authoring agent of a turn metric is **not** authorized to read it back (only the `p`-tagged owner is). | consequence of `filter.rs:28-32`; asserted at `filter.rs:294-298` | same |
| BR-15a | **Added by `ab3af828`.** For `KIND_PERSONA` (30175) the event is withheld from a reader when all three hold: the kind is 30175, the reader is not the author, and the event carries no `["shared","true"]` tag. The author always reads their own; a shared persona is readable community-wide. | `crates/buzz-core/src/kind.rs:205-216` (`is_unshared_persona_event`); kind test `:192-194`; tag test `persona_event_is_shared` `:226-241` | Called from `handlers/req.rs:1222-1234` (`event_visible_to_reader`), which every relay read surface uses |
| BR-15b | The `shared` marker is a **tag**, not a content field, so toggling share state leaves the content bytes (and therefore the `source_version` / `persona_content_hash` drift basis) unchanged. | `crates/buzz-core/src/kind.rs:176-182` | design note in the `KIND_PERSONA` doc comment |
| BR-15c | `persona_event_is_shared` requires the tag to be **exactly two elements**, so `["shared","true","extra"]` is not shared; ingest rejects that shape so no such row can exist. | `crates/buzz-core/src/kind.rs:226-241`; ingest rule `crates/buzz-relay/src/handlers/ingest.rs:1047-1052` | keeps the SQL containment check `tags @> '[["shared","true"]]'` sound (`crates/buzz-db/src/event.rs:504-527`, bind at `:524-525`; the soundness argument is spelled out in the field doc at `:82-87`) |

Note: the gated kind set in `reader_authorized_for_event` is hard-coded as two `!=` comparisons (`filter.rs:25`), duplicating the membership of `kind::RESULT_GATED_KINDS` (`kind.rs:129`) rather than reading it. BR-15a took the opposite approach — one shared predicate, one kind test — and is the model the older gate should follow.

---

### 3. Kind classification rules (`src/kind.rs`)

| # | Rule | Enforced at |
|---|------|-------------|
| BR-16 | Ephemeral = `20000 ≤ kind ≤ 29999`; documented as never stored. | `kind.rs:697-699`, range constants `:397-399` |
| BR-17 | Replaceable = kinds `0`, `3`, `41`, or `10000..=19999` (NIP-01). Parameterized-replaceable kinds are explicitly *not* in this set. | `kind.rs:704-706` and doc `:701-703` |
| BR-18 | Parameterized replaceable = `30000 ≤ kind ≤ 39999`, keyed by `(pubkey, kind, d_tag)`, latest `created_at` wins. | `kind.rs:711-713` and doc `:708-710` |
| BR-19 | Replaceable and parameterized-replaceable sets must be disjoint over the whole u16 space. | test `replaceable_and_parameterized_are_disjoint` iterates `0..=65535` (`kind.rs:852-859`) |
| BR-20 | Workflow execution kinds `46001..=46012` must not trigger workflows (loop prevention, per doc). | `kind.rs:717-719`, doc `:715-716` |
| BR-21 | Relay-admin command kinds are exactly 9030–9033. | `kind.rs:723-732` |
| BR-22 | Identity-archive **requests** are only 9035/9036; relay-signed 8002/8003/13535 are deliberately excluded from the request classifier. | `kind.rs:738-740`, doc `:734-737` |
| BR-23 | Community moderation commands are exactly 9040–9044; `KIND_REPORT` (1984) is **not** a command. | `kind.rs:316-325`; compile-time assertions `:818-820` |
| BR-24 | Transactional "command kinds" requiring atomic execution: 30620, 41010, 41011, 41012, 46020, 46030, 46031. | `kind.rs:743-755` |
| BR-25 | Relay-only kinds (client submission must be rejected): 13534, 40901, 40902, 30622, 39005, 39006. | `kind.rs:758-769`; test `nip43_membership_snapshot_is_relay_only` (`kind.rs:836-839`) |
| BR-26 | No two entries in `ALL_KINDS` may share an integer value. | test `no_duplicate_kind_values` (`kind.rs:828-834`) — note: covers only the 127 listed kinds, not the 3 excluded ones |
| BR-27 | Compile-time invariants: new addressable kinds must be in the 30000–39999 range; all kind constants must fit `u16`; `KIND_AGENT_TURN_METRIC` must be neither ephemeral, replaceable, nor parameterized. | `const _: () = assert!(...)` block, `kind.rs:783-820` |
| BR-28 | Author-only read kinds: `KIND_EVENT_REMINDER` (30300) and `KIND_PUSH_LEASE` (30350) — relay must not reveal existence, count, tags, content, schedule, or search matches to non-authors (doc, enforced elsewhere). | `kind.rs:112-120` |
| BR-29 | `#p`-bound read kinds (`P_GATED_KINDS`): 24200, 44100, 44101, 1059, 30622, 44200 — a REQ that can match any of these is closed unless the filter's `#p` equals the reader's pubkey; stored kinds in the set also get a NULL `search_tsv`. | `kind.rs:122-156` (declaration + contract doc) |
| BR-30 | Result-gated kinds (30622, 44200) force the per-event fallback path in COUNT instead of fast SQL counting. | `kind.rs:122-129` |

---

### 4. Channel + role rules (`src/channel.rs`)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| BR-31 | Canonical channel name strips **all** leading `#` characters and leading/trailing whitespace (interleaved), then trims the end. | `channel.rs:15-19` | `canonical_channel_name()`; test `channel_names_trim_whitespace_and_drop_all_leading_hashes` (`channel.rs:186-197`) pins `"### ###" → ""` and `"channel#topic"` unchanged |
| BR-32 | Role hierarchy is numeric: Owner 4 > Admin 3 > Member 2 > Guest 1 > Bot 0. | `channel.rs:142-150` | `permission_level()` |
| BR-33 | Bot never satisfies any role requirement at this layer (level 0 < every other level). | consequence of `channel.rs:142-157` | `has_at_least()`; the `git_perms` test `evaluate_bot_cannot_push_without_explicit_grant` documents that the policy layer promotes Bot→Member before evaluation (`git_perms.rs:910-924`) |
| BR-34 | Elevated roles (only grantable by owners/admins per doc) are Owner and Admin. | `channel.rs:134-136` | `is_elevated()` |
| BR-35 | `FromStr` for all three channel enums is strict: unknown strings error with a formatted message; there is no default variant. | `channel.rs:44-53`, `:88-99`, `:163-179` | parsing DB/Nostr tag values |

---

### 5. Presence rules (`src/presence.rs`)

| # | Rule | Enforced at |
|---|------|-------------|
| BR-36 | Structured (REST/MCP) presence accepts exactly `online`, `away`, `offline`; unknown values fail deserialization. | serde derive with `rename_all = "lowercase"` (`presence.rs:10-18`); test `serde_rejects_unknown_variant` (`presence.rs:55-59`) |
| BR-37 | `Offline` clears the presence entry (documented semantic). | doc comment `presence.rs:17` |
| BR-38 | `as_str()`, `Display`, and the serde form must agree. | `presence.rs:22-35`; tests `as_str_matches_serde` (`presence.rs:61-66`), `display_matches_as_str` (`presence.rs:68-73`) |

---

### 6. Tenant resolution rules (`src/tenant.rs`)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| BR-39 | A request's community is resolved by the server from the connection host — never supplied or influenced by the client. Expressed in types by having no `Default`, no `Deserialize`, and no parse-from-input constructor. | `tenant.rs:10-30` (invariant doc), `:43-44` (no serde derives), `:74-78` | construction of `TenantContext` |
| BR-40 | Host normalization: trim whitespace → ASCII-lowercase → strip a `:443` or `:80` suffix → strip one trailing `.`. Non-default ports are preserved as distinct selectors. | `tenant.rs:121-139` | `normalize_host()` on both the inbound `Host` header and the stored `communities.host` |
| BR-41 | Order matters: the default-port strip happens **before** the trailing-dot strip, so `"Relay.Example.:443"` normalizes to `"relay.example"`. | `tenant.rs:128-138`; test `normalize_host_collapses_tenant_split_variants` (`tenant.rs:196-213`) | |
| BR-42 | Empty/whitespace-only host normalizes to `""`; resolution must fail closed rather than defaulting to a tenant. | `tenant.rs:117-119` (doc), test `normalize_host_empty_stays_empty` (`tenant.rs:229-234`) | |
| BR-43 | IPv6 literals are left intact (`[::1]` keeps brackets) because only an exact `:80`/`:443` suffix is stripped. | `tenant.rs:126-134`; test `normalize_host_leaves_ipv6_literal_intact` (`tenant.rs:222-227`) | |
| BR-44 | `relay_url_authority` must produce byte-identical output to `normalize_host` for the same deployment: host + explicit non-default port, IPv6 in brackets, empty string when unparseable (caller fails closed). | `tenant.rs:156-172`; tests `tenant.rs:236-274` | startup community seeding, `bind_deployment_community`, `buzz-admin` tenant resolution (per doc `:146-151`) |

---

### 7. Relay URL identity rules (`src/relay.rs`)

| # | Rule | Enforced at |
|---|------|-------------|
| BR-45 | Scheme must be `ws` or `wss`; anything else → `InvalidScheme`. | `relay.rs:40-42` |
| BR-46 | URLs carrying a username or password are rejected (`Credentials`). | `relay.rs:43-45` |
| BR-47 | URLs carrying a fragment are rejected (`Fragment`). | `relay.rs:46-48` |
| BR-48 | A host is required (`MissingHost`). | `relay.rs:50` |
| BR-49 | All loopback spellings (`localhost` case-insensitive, loopback IPv4, loopback IPv6) fold to `127.0.0.1`; other DNS hosts are ASCII-lowercased. | `relay.rs:51-64` |
| BR-50 | Default ports are dropped (`ws`→80, `wss`→443); a root path `/` is removed; trailing `/` is trimmed from the final string. Non-root paths and queries are preserved. | `relay.rs:66-77` |
| BR-51 | This normalizer is explicitly **not** the NIP-42 AUTH comparison helper; AUTH equivalence must stay narrower. | doc `relay.rs:28-32` |

---

### 8. NIP-AE engram rules (`src/engram.rs`)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| BR-52 | Slug grammar: either the reserved `"core"`, or `mem/` + one-or-more `/`-separated segments; each segment 1–64 bytes, first byte `[a-z0-9]`, remaining bytes `[a-z0-9_-]`; whole slug ≤ 255 bytes. | `engram.rs:67-92` + `validate_segment` `:94-112` | `validate_slug()` from parse, build, extract-refs paths |
| BR-53 | Shorthand normalization: `core` passes through; anything not already prefixed gets `mem/` prepended; the result is then validated (so `"Foo"` still fails). | `engram.rs:123-131`; test `normalize_slug_adds_mem_prefix` (`engram.rs:773-778`) | `normalize_slug()` |
| BR-54 | Conversation key `K_c` is NIP-44 v2 ECDH-derived and symmetric: `derive(sk_a, pk_o) == derive(sk_o, pk_a)`. | `engram.rs:136-138`; test `conversation_key_matches_spec` pins both directions to one hex vector (`engram.rs:636-644`) | |
| BR-55 | `d` tag = lowercase hex of `HMAC-SHA256(K_c, "agent-memory/v1/d-tag" ‖ 0x00 ‖ slug)`. | `engram.rs:144-155` | `d_tag()` |
| BR-56 | Body JSON is byte-exact: slug first, whitespace-free, `{"slug":…,"value":…}` or `{"slug":"core","profile":…}`; `null` value = tombstone. | `engram.rs:189-217`; test `body_round_trips_byte_exact` (`engram.rs:656-693`) | `to_json_bytes()` |
| BR-57 | Body parsing rejects duplicate object member names **at any nesting depth**; repeats inside arrays are allowed. | `parse_strict_json` `engram.rs:283-380` (`visit_map` dup check `:353-370`); tests `body_parse_rejects_duplicate_keys` (`engram.rs:695`), `body_parse_accepts_arrays_of_repeated_strings` (`engram.rs:710`), `body_parse_rejects_duplicates_at_any_depth` (`engram.rs:739`) | `Body::from_json_bytes()` |
| BR-58 | Trailing data after the JSON document is rejected. | `engram.rs:377-378` | same |
| BR-59 | Body shape: top level must be an object; `slug` must be a present string and pass BR-52; `core` requires a string `profile`; non-core requires `value` as string or null; unknown fields are ignored. | `engram.rs:228-258`; test `body_parse_ignores_unknown_fields` (`engram.rs:703-708`) | same |
| BR-60 | Reference extraction: only literal `[[slug]]` substrings count; the first `]]` closes, no nesting; an inner `[[` restarts the scan; candidates failing `validate_slug` are silently dropped; results are first-occurrence-ordered and deduplicated. | `engram.rs:384-430` | `extract_refs()`; 15 tests spanning `engram.rs:891-1047` |
| BR-61 | Serialized body must not exceed 65,535 bytes (NIP-44 plaintext cap) — rejected at build time with `BodyTooLarge`. | `engram.rs:436-440`; test `body_too_large_rejected_at_build_time` (`engram.rs:877-889`) | `build_event()` |
| BR-62 | Built engram events carry exactly two tags — `["d", <hex>]` and `["p", <owner hex>]` — kind 30174, with a caller-supplied `created_at`. | `engram.rs:456-476` | `build_event()` |
| BR-63 | Envelope validation (head-selection rules 1 and 5): kind must be 30174; `event.pubkey` must equal the expected agent; exactly one `d` tag; `d` must be 64 lowercase hex chars; exactly one `p` tag; `p` must equal the expected owner (case-insensitive compare). | `engram.rs:489-536` | `validate_and_decrypt()` |
| BR-64 | After decryption, the body's slug must re-derive to the event's `d` tag (rule 4), else `InvalidEnvelope`. | `engram.rs:545-557` | same |
| BR-65 | Signature verification is the **caller's** responsibility before calling `validate_and_decrypt` (NIP-44 requires outer-signature-before-decrypt). | doc `engram.rs:478-482` | contract note, not enforced in this function |
| BR-66 | Head selection (LWW): greatest `created_at` wins; ties broken by **lowest** event id hex. | `engram.rs:564-583` | `select_head()` |
| BR-67 | Write monotonicity: `created_at = max(now, prior_head + 1)` with saturating add. | `engram.rs:588-593`; test `monotonic_clock_rule` (`engram.rs:781-786`) | `monotonic_created_at()` |

---

### 9. NIP-AM turn metric rules (`src/agent_turn_metric.rs`)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| BR-68 | `cost_usd` (in `turn` and in `cumulative`) must be finite and ≥ 0 when present; NaN, ±inf, and negatives are rejected with `InvalidPayload`. | `agent_turn_metric.rs:147-169` | `validate()` |
| BR-69 | Token counts cannot be negative by construction (`Option<u64>`); `None` means "not reported", never zero. | type choice `:24-33` + doc `:18-20` | serde round-trip |
| BR-70 | `total_tokens` is provider-reported and must not be derived by summing input+output. | doc `:31-32` | consumer contract |
| BR-71 | Validation is symmetric: both `encrypt_agent_turn_metric` and `decrypt_agent_turn_metric` call `validate()`, so a payload written through the lower-level `encrypt_observer_payload` path is still rejected on read. | `:174` and `:189-190`; regression test `decrypt_agent_turn_metric_rejects_negative_cost_bypassing_encrypt` (`agent_turn_metric.rs:476-507`) | encrypt / decrypt |
| BR-72 | Unrecognized `stopReason` strings deserialize to `Unknown` instead of failing the payload; token counts survive. | hand-written `Deserialize` `:64-77`; test `unknown_stop_reason_maps_to_unknown_not_error` (`agent_turn_metric.rs:321-345`) | payload parse |
| BR-73 | `deltaReliable` defaults to `true` when absent on the wire. | `#[serde(default)]` `:126-127` + `:142-144`; test `delta_reliable_defaults_to_true_when_absent` (`agent_turn_metric.rs:275-283`) | payload parse |
| BR-74 | `session_id` and `turn_seq` are REQUIRED whenever `cumulative` is present; `turn_seq` must strictly increase within a `session_id`, and a publisher restart that loses the counter must start a new `session_id`. | doc only, `:97-108` — **not enforced in code** | documented publisher contract |

---

### 10. Observer payload rules (`src/observer.rs`)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| BR-75 | Ciphertext must be 132–87,472 chars to be treated as NIP-44 v2; out-of-range content is rejected **before** any decryption attempt. | `observer.rs:53-55`, `:85-90` | `decrypt_observer_payload()` |
| BR-76 | Plaintext JSON must be ≤ 65,535 bytes on both encrypt and decrypt; oversize plaintext is zeroized before the error returns. | `observer.rs:64-72` (encrypt), `:96-104` (decrypt) | both helpers |
| BR-77 | Plaintext is zeroized after use on every path — success, size failure, and parse failure. | `observer.rs:67`, `:74`, `:100`, `:107` | both helpers |
| BR-78 | Encryption always uses NIP-44 `Version::V2`. | `observer.rs:70` | `encrypt_observer_payload()` |
| BR-79 | Decryption uses `event.pubkey` as the peer key, so only the recipient of the event's ECDH pair can read it. | `observer.rs:92-95` | `decrypt_observer_payload()` |

---

### 11. Git permission rules (`src/git_perms.rs`)

#### Pattern grammar

| # | Rule | Enforced at |
|---|------|-------------|
| BR-80 | Pattern must be non-empty. | `git_perms.rs:84-86` |
| BR-81 | Pattern length ≤ 256 chars (`MAX_PATTERN_LENGTH`). | `:87-89` |
| BR-82 | Pattern must start with `refs/`. | `:90-92` |
| BR-83 | `**` is allowed only as the final segment. | `:100-107` |
| BR-84 | At most 3 wildcard segments (`*` and `**` both count). | `:108-111`, `:113-116` |
| BR-85 | Empty segments (e.g. `refs//x`) are rejected. | `:121-122` |
| BR-86 | Partial globs (`v*`, `?`, `[`, `]` inside a segment) are rejected. | `:123-131` |
| BR-87 | Literal segments are restricted to `[A-Za-z0-9._-]`. | `:132-137` |
| BR-88 | `*` matches exactly one path segment; non-recursive patterns require an exact segment-count match. | `:170-178` |
| BR-89 | `**` matches **one or more** remaining segments — `refs/heads/**` does not match `refs/heads`. | `:151-168`; test `pattern_recursive_requires_at_least_one_segment` (`git_perms.rs:676-681`) |

#### Update classification

| # | Rule | Enforced at |
|---|------|-------------|
| BR-90 | `old_oid == 40 zeros` → Create; else `new_oid == 40 zeros` → Delete; else `is_ancestor` → FastForward; else NonFastForward. Create is checked before Delete. | `git_perms.rs:206-221` |

#### Rule parsing

| # | Rule | Enforced at |
|---|------|-------------|
| BR-91 | A `buzz-protect` tag needs at least 2 values (pattern + ≥1 rule), else `TooFewValues`. | `:306-308` |
| BR-92 | `push:<role>` is parsed via `MemberRole::FromStr`; unknown roles → `InvalidRole`. | `:317-320` |
| BR-93 | `push:bot` and `push:guest` are rejected as nonsensical. | `:321-326`; test `parse_protection_tag_rejects_push_bot_and_guest` (`git_perms.rs:760-771`) |
| BR-94 | Multiple `push:` rules in one tag collapse to the **strictest** (highest permission level). | `:327-336` |
| BR-95 | Recognized flag rules are exactly `no-force-push`, `no-delete`, `require-patch`; unknown rules are skipped and reported for logging (forward compatibility). | `:337-346`; test `parse_protection_tag_unknown_rule_skipped` (`git_perms.rs:743-750`) |
| BR-96 | At most 50 protection rules per repo (`MAX_PROTECTION_RULES`); the 51st `buzz-protect` tag errors with `TooManyRules`. | `:379-395` |
| BR-97 | Non-`buzz-protect` tags are skipped rather than erroring. | `:384-386` |

#### Effective rules + evaluation

| # | Rule | Enforced at |
|---|------|-------------|
| BR-98 | `EffectiveRules` unions every matching pattern: strictest `push_role`, logical OR of `no_force_push` / `no_delete` / `require_patch`, and `has_explicit_match` if any matched. | `:447-475`; test `effective_rules_union_strictest_role` (`git_perms.rs:773-783`) |
| BR-99 | Built-in defaults when no rule matches: branch/tag Create → Member; other Create → Admin; branch FastForward → Member; tag FastForward (tag move) → Admin; other FastForward → Admin; NonFastForward → Admin; Delete → Admin. | `:403-425` |
| BR-100 | If no explicit rule matches, only the default minimum role is checked; the denial message states "using built-in defaults". | `:511-525` |
| BR-101 | `require-patch` denies **all** direct pushes regardless of role or update kind. | `:527-533`; doc `:248-253`; test `evaluate_require_patch_blocks_all` (`git_perms.rs:861-872`) |
| BR-102 | An explicit `push:<role>` can never *weaken* the built-in default: the effective minimum is `max(explicit, default)`. | `:535-552`; test `evaluate_push_member_cannot_weaken_destructive_defaults` (`git_perms.rs:943-978`) |
| BR-103 | A rule that sets only flags (no `push:`) still enforces the built-in default role — a Guest cannot slip through. | `:546-551`; regression test `evaluate_guest_denied_even_with_only_no_force_push_rule` (`git_perms.rs:926-941`) |
| BR-104 | `no-force-push` blocks NonFastForward updates for **everyone**, including Owner. | `:554-560`; test `evaluate_no_force_push_blocks_owner` (`git_perms.rs:816-830`) |
| BR-105 | `no-force-push` must not block FastForward updates. | `:559`; test `evaluate_no_force_push_allows_fast_forward` (`git_perms.rs:832-846`) |
| BR-106 | `no-delete` blocks Delete updates for everyone, including Admin. | `:562-568`; test `evaluate_no_delete_blocks_admin` (`git_perms.rs:848-859`) |
| BR-107 | Check order inside an explicit match: require-patch → role → no-force-push → no-delete. | sequential structure `:527-570` |
| BR-108 | A push is atomic: `evaluate_push` collects denials from every ref and fails the whole push if any ref is denied. | `:584-597`; doc `:580-582`; test `evaluate_push_multiple_refs_partial_deny` (`git_perms.rs:980-1002`) |
| BR-109 | Permission model statement: channel role = repo role; `buzz-protect` constraints apply to everyone including the owner. | doc `git_perms.rs:5-7` |

---

### 12. NIP-AB pairing rules

#### QR payload (`src/pairing/qr.rs`)

| # | Rule | Enforced at |
|---|------|-------------|
| BR-110 | URI length must not exceed 2048 characters. | `qr.rs:105-111` |
| BR-111 | URI must start with `nostrpair://` and contain a `?` query string. | `qr.rs:113-127` |
| BR-112 | Pubkey must be exactly 64 **lowercase** hex chars (uppercase rejected). | `qr.rs:129-137`, helper `is_lowercase_hex` `:240-242`; test `reject_uppercase_hex_in_pubkey` (`qr.rs:555-571`) |
| BR-113 | `secret` must be present and exactly 64 lowercase hex chars → 32 bytes. | `qr.rs:153-168` |
| BR-114 | An all-zeros `session_secret` is rejected. | `qr.rs:169-174`; test `reject_all_zeros_session_secret` (`qr.rs:538-553`) |
| BR-115 | At least one `relay` parameter is required. | `qr.rs:176-181` |
| BR-116 | Every relay URL must fully parse, use scheme `ws` or `wss`, and have a host. | `qr.rs:183-204` |
| BR-117 | Version defaults to 1 when `v=` is absent; any version ≠ 1 is rejected. | `qr.rs:145-151`; tests `reject_unsupported_version` (`qr.rs:517-527`) and `default_version_when_absent` (`qr.rs:529-536`) |
| BR-118 | Unknown query parameters are ignored. | `qr.rs:141` |
| BR-119 | Relay values are percent-encoded with `NON_ALPHANUMERIC` on encode and percent-decoded (lossy UTF-8) on decode. | `qr.rs:220-236` |
| BR-120 | The session secret is zeroized when a `QrPayload` drops. | `qr.rs:56-60` |

#### Key derivation (`src/pairing/crypto.rs`)

| # | Rule | Enforced at |
|---|------|-------------|
| BR-121 | `session_id = HKDF-SHA256(IKM=session_secret, salt=[], info="nostr-pair-session-id", L=32)`. | `crypto.rs:29-42`, `:54-56`, info const `:23` |
| BR-122 | `sas_input = HKDF-SHA256(IKM=ecdh_shared, salt=session_secret, info="nostr-pair-sas-v1", L=32)`; `sas_code = be_u32(sas_input[0..4]) mod 1_000_000`. | `crypto.rs:70-75`, info const `:24` |
| BR-123 | `transcript = session_id ‖ source_pubkey ‖ target_pubkey ‖ sas_input` (128 bytes, order-sensitive); `transcript_hash = HKDF-SHA256(IKM=transcript, salt=session_secret, info="nostr-pair-transcript-v1", L=32)`. | `crypto.rs:89-105`; order-sensitivity test `transcript_hash_sensitive_to_pubkey_order` (`crypto.rs:325-351`) |
| BR-124 | SAS is displayed zero-padded to exactly 6 digits. | `crypto.rs:116-118`; tests `format_sas_zero_padding` (`crypto.rs:353-360`) and `format_sas_always_six_chars` (`crypto.rs:362-369`) |
| BR-125 | Comparisons of 32-byte secret-derived values use constant-time equality. | `crypto.rs:126-129` |
| BR-126 | Pinned spec test vectors (pubkeys, ECDH, session id, sas_input, SAS `863346`, transcript hash) must not drift. | test `all_test_vectors` (`crypto.rs:272-323`) |

#### Session state machine (`src/pairing/session.rs`)

| # | Rule | Enforced at |
|---|------|-------------|
| BR-127 | Default session lifetime is 120 s; every handler calls `check_expired()` first and fails with `SessionExpired`. | `session.rs:43` (`DEFAULT_TIMEOUT = 120s`), applied at `:138`/`:309`; `check_expired` at `:698-703`; test `expired_session_rejects_operations` (`session.rs:1130-1145`) |
| BR-128 | Every handler asserts both the exact expected state and the expected role before doing work. | `expect_state` `:706-715`, `expect_role` `:717-726`; called at the top of each handler (e.g. `:150-152`, `:330-332`, `:392-394`) |
| BR-129 | Offer handling requires protocol `version == 1`. | `:169-175` |
| BR-130 | The offer's `session_id` must equal the locally derived session id, compared in constant time; mismatch → `InvalidSessionId`. | `:177-185`; test `reject_invalid_session_id` (`session.rs:986-1002`) |
| BR-131 | The source locks onto the offering pubkey as its peer; every later event must come from that exact pubkey. | `:187-188`, `validate_event_from_peer` `:681-696`; test `reject_event_from_wrong_pubkey` (`session.rs:1004-1043`) |
| BR-132 | The ECDH shared secret is zeroized immediately after SAS derivation. | `:186-191` (source, `ecdh.zeroize()` at `:189`), `:290-295` (target, `:295`) |
| BR-133 | Basic event validation: full `event.verify()` (id + sig, `:632-635`), duplicate-id check (`:638-643`), kind must equal 24134 (`:645-650`), and a `p` tag must match our ephemeral pubkey (`:653-666`). | `session.rs:631-674` |
| BR-134 | Duplicate event ids are rejected; ids are recorded **only after** a message is fully accepted, so speculative probes (e.g. `handle_abort` used as a detector) cannot poison the dedup set. | `record_event` `:676-678`, called at `:193`, `:264`, `:369`, `:404`, `:468`; tests `duplicate_event_id_is_rejected` (`session.rs:1148-1184`), `speculative_abort_does_not_poison_dedup` (`:1186-1212`), `wrong_type_message_not_recorded` (`:1214-1267`) |
| BR-135 | Target must obtain explicit user confirmation: `handle_sas_confirm` moves to `AwaitingConfirmation`, and only `confirm_target_sas()` unlocks `Transferring` so a payload can be accepted. | `:368`, `:376-382`; test `target_must_confirm_sas_before_payload` (`session.rs:1056-1089`) |
| BR-136 | Transcript-hash mismatch on the target aborts the session immediately (`state = Aborted`, `TranscriptMismatch`). | `:359-367` |
| BR-137 | Exactly one payload per session — state advances to `PayloadExchanged`, so a second send or receive fails. | `:249`, `:403`; test `reject_duplicate_payload` (`session.rs:1091-1119`) |
| BR-138 | `complete(success = false)` sets state to `Aborted`, returns an error, and does **not** record the event id. | `:267-278`; test `complete_failure_aborts_without_recording` (`session.rs:1269-1314`) |
| BR-139 | `abort()` is rejected from terminal states (`Completed`/`Aborted`) — a finished session cannot regress. | `:431-437`; test `local_abort_after_completed_is_rejected` (`session.rs:871-898`) |
| BR-140 | `abort()` with no known peer transitions to `Aborted` and returns `None` (nothing to encrypt). | `:438-441`; test `abort_without_peer_returns_none` (`session.rs:862-869`) |
| BR-141 | `handle_abort()` requires a known peer, so an anonymous relay observer cannot kill a session; late aborts in terminal states are ignored. | `:449-462`; tests `reject_handle_abort_before_peer_known` (`session.rs:900-931`) and `reject_abort_after_completed` (`session.rs:933-984`) |
| BR-142 | Outbound events carry a `created_at` reduced by random 0–30 s jitter for metadata privacy, exactly one `p` tag (the peer), kind 24134, NIP-44 v2 content. | `session.rs:561-590` (jitter at `:578-579`, single `p` tag at `:582`, NIP-44 V2 at `:566-571`) |
| BR-143 | Inbound content length must be 132–87,472 chars before decryption; decrypted plaintext must be ≤ 65,535 bytes, otherwise it is zeroized and rejected. | `session.rs:592-629` (length gate `:595`, plaintext cap `:609-615`) |
| BR-144 | Serialized plaintext is zeroized after encryption; decrypted plaintext is zeroized on both success and parse failure; the transient payload clone is zeroized in `send_payload`. | `:573` (serialized plaintext), `:610`/`:619` (decrypted plaintext), `:241-247` (payload clone) |
| BR-145 | `qr_uri()` returns `None` for a Target session (only the source displays a QR). | `session.rs:517-527` (`role != Source` → `None` at `:518-520`) |
| BR-146 | Session secret, session id, and SAS input are zeroized on drop. | `Drop` impl `session.rs:731-739` |

#### Message wire rules (`src/pairing/types.rs`)

| # | Rule | Enforced at |
|---|------|-------------|
| BR-147 | Messages are JSON tagged unions on `"type"` in kebab-case (`offer`, `sas-confirm`, `payload`, `complete`, `abort`). | `types.rs:18-63`; tests `:104-126`, `:147-155` |
| BR-148 | `Offer.version` defaults to 1 when absent (pre-versioned compatibility). | `types.rs:12-14`, `:28-31`; test `:117-131` |
| BR-149 | Unknown `abort.reason` strings deserialize to `Unknown` and are to be treated as `protocol_error`; callers must not emit `Unknown` outbound. | `types.rs:88-101`; tests `:196-206`, `:208-215` |

---

### 13. SSRF classification rules (`src/network.rs`)

Documented as: webhook targets must not resolve to these addresses (`network.rs:22-24`). Exact ranges in the security aspect doc; the rule itself:

| # | Rule | Enforced at |
|---|------|-------------|
| BR-150 | An IPv4-mapped IPv6 address is re-checked recursively against the IPv4 rule set, so `::ffff:10.0.0.1` is blocked. **Widened in `c26bf594`**: the guard is now `to_ipv4()` instead of `to_ipv4_mapped()`, so IPv4-**compatible** `::a.b.c.d` recurses as well (`::10.0.0.1` is blocked). | `network.rs:62-65` |
| BR-150a | Two further embedded-IPv4 forms recurse into the IPv4 rule set via `embedded_ipv4` (`network.rs:15-20`): NAT64 well-known `64:ff9b::/96` and legacy SIIT IPv4-translated `::ffff:0:0:0/96`. Public embedded IPv4 stays reachable; private/reserved embedded IPv4 is blocked. | `network.rs:69-73`, `:75-79` |
| BR-151 | IPv4 blocked if loopback, RFC1918 private, link-local, first octet 0, broadcast, CGNAT `100.64.0.0/10`, or benchmarking `198.18.0.0/15`. (Unchanged by the sync.) | `network.rs:48-60` |
| BR-152 | IPv6 blocked if loopback, unspecified, ULA `fc00::/7`, link-local `fe80::/10`, multicast `ff00::/8`, documentation `2001:db8::/32`, and — added in `c26bf594` — NAT64 local-use `64:ff9b:1::/48`, Teredo `2001::/32`, 6to4 `2002::/16`. | `network.rs:81-92` |


## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: Business Rules

All rules execute **before** the caller signs — every builder returns
`Result<EventBuilder, SdkError>` and short-circuits on the first violation
(`crates/buzz-sdk/src/lib.rs:11-13`).

---

### 1. Shared validation primitives

| Rule | What it enforces | File:line | Trigger |
|---|---|---|---|
| `check_content` | `content.len()` (bytes) ≤ `max`, else `ContentTooLarge{max,got}` | `builders.rs:35-41` | every content-bearing builder |
| `check_hex_len` | ≥ `min_len` chars **and** all ASCII hex, else `InvalidDiffMeta` | `builders.rs:44-52` | `build_diff_message` commit/parent, `build_add_member`/`build_remove_member` pubkeys |
| `check_commit_hex` | length is exactly 40 (SHA-1) **or** 64 (SHA-256) and all hex; abbreviated refs rejected | `builders.rs:59-66` | patch/status/PR commit, parent-commit, euc, merge-commit, merge-base, applied-as-commits |
| `check_pubkey_hex` | exactly 64 hex chars; returns lowercased | `builders.rs:69-77` | observer frame, git owner/recipients, workflow delete, DM, moderation, identity archival, auth tag |
| `check_hex_exact` | exactly `len` hex chars; returns lowercased | `builders.rs:79-89` | patch `reply_to`, status `root_event`/`accepted_revision_root`/`applied_patch`, PR `revision_of`, `pr_event`, `report_event_id` |
| `check_repo_id` | non-empty, ≤64 chars, charset `[A-Za-z0-9._-]`, no leading `.`, no `..` | `builders.rs:92-121` | `build_repo_announcement`, `build_repo_announcement_with_tags`, and every `GitRepoCoord::to_a_tag_value` call |
| `normalize_custom_emoji_shortcode` | trims whitespace and `:`; non-empty; ≤64 bytes; charset `[A-Za-z0-9_-]`; lowercases (prevents `party_parrot` vs `Party_Parrot` collision) | `builders.rs:127-150` | `build_custom_emoji_reaction`, `build_custom_emoji_set` |
| `check_custom_emoji_url` | non-empty, ≤2048 bytes, must start `http://` or `https://` | `builders.rs:152-170` | same two builders |
| `check_reason` | ≤64 **UTF-8 bytes** (`MAX_REASON_BYTES`), no control characters | `builders.rs:1704-1721` | identity archive/unarchive `reason` |
| `check_auth_tag_shape` | element 0 == `"auth"`, element 1 is 64-hex pubkey, element 3 is 128-hex signature (structure only; relay does full verification) | `builders.rs:1723-1737` | identity archive/unarchive `auth` |

---

### 2. Tag-construction rules

| Rule | Enforcement | File:line | Trigger |
|---|---|---|---|
| NIP-29 channel scoping via `h` tag | every channel-scoped builder emits `["h", channel_uuid]` as the **first** tag; no `e`-tag substitution | `builders.rs:224`, `283`, `297`, `339`, `382`, `415`, `438`, `456`, `530`, `569`, `586`, `596`, `630`, `653`, `662`, `689`, `704`, `711`, `720`, `728`, `1476`, `1494`, `1562` | any channel operation |
| NIP-10 threading markers | `root == parent` ⇒ single `["e",root,"","reply"]`; `root ≠ parent` ⇒ `["e",root,"","root"]` + `["e",parent,"","reply"]` | `builders.rs:173-186` (documented `lib.rs:24-27`) | `build_message`, `build_forum_comment`, `build_diff_message` |
| Flat reply model for kind 1 | `build_note` emits only `["e",id,"","reply"]` — full NIP-10 root/reply/p-tags explicitly deferred | `builders.rs:732-748` | global text notes |
| Mention dedup + cap | > `MENTION_CAP` (50) raw entries ⇒ `TooManyMentions`; cap is checked **before** dedup, then remaining entries lowercased and deduped into `p` tags | `builders.rs:188-201`; cap `mentions.rs:38` | any builder taking `mentions` |
| Community-command events carry **no** `h` tag | moderation kinds 9040–9044 intentionally omit `h`; tenant is bound by connection host, a stray `h` would be rejected relay-side | `builders.rs:1587-1595` (comment), builders `1597-1690` | moderation commands |
| NIP-70 protection marker | identity archival requests always emit `["-"]` as the first tag | `builders.rs:1748` | kinds 9035/9036 |
| `allow_self_tagging()` | required so nostr 0.44 does not strip the `["p", target]` tag when actor == target (self archive/unarchive path) | `builders.rs:1798-1801`, `1819-1822` | kinds 9035/9036 |
| Non-standard `h` on NIP-09 kind 5 | `build_delete_compat` adds `h` so channel-scoped subscriptions observe the delete (documented as intentionally non-standard) | `builders.rs:433-443` | compat deletes |
| `a`-tag coordinate format | repo coordinate rendered as `30617:<owner>:<id>` with owner and id revalidated at render time | `builders.rs:975-982` | all NIP-34 builders |
| Workflow deletion coordinate | `["a", "<KIND_WORKFLOW_DEF>:<pubkey>:<workflow_id>"]` (30620) | `builders.rs:1498-1508` | workflow delete |
| Multi-value tags | `clone`, `relays`, `applied-as-commits` are single tags with N values, not N tags | `builders.rs:925-940`, `1289-1297`, `1367-1369`, `1449-1451` | repo announcement, PR, PR update, merged status |
| Duplicated `r` tags for commits | `commit`, `merge-commit`, and each `applied-as-commit` also emit a bare `["r", <commit>]` | `builders.rs:1050-1052`, `1284-1287`, `1296-1298` | patch, merged status |
| NIP-22 uppercase root tags | PR update references its PR with `["E", pr_event]` + `["P", pr_author]` | `builders.rs:1443-1444` | kind 1619 |
| Emoji set d-tag pinned | `["d","buzz:custom-emoji"]` always first, so the set is parameterized-replaceable per member | `builders.rs:503`, `513` | kind 30030 |

---

### 3. Per-builder business rules

| Rule | Enforces | File:line | Trigger |
|---|---|---|---|
| Message content bound | ≤ 64 KiB | `builders.rs:227` | `build_message` |
| Diff content bound | ≤ 60 KiB (tighter than messages) | `builders.rs:314` | `build_diff_message` |
| Diff `repo_url` scheme | must start `http://` or `https://` | `builders.rs:317-321` | `build_diff_message` |
| Diff commit SHA | ≥7 hex chars (abbreviated refs allowed here, unlike NIP-34 builders) | `builders.rs:322` | `build_diff_message` |
| Diff parent commit | ≥7 hex chars when present | `builders.rs:323-325` | `build_diff_message` |
| Diff branch pair | both source and target must be non-empty | `builders.rs:326-332` | `build_diff_message` |
| Diff PR number | must be > 0 | `builders.rs:333-339` | `build_diff_message` |
| Observer frame direction | `frame` must equal `"telemetry"` or `"control"` | `builders.rs:251-255` | `build_agent_observer_frame` |
| Observer frame encryption | `content_looks_like_nip44` must hold (length in 132..=87 472, `crates/buzz-core/src/observer.rs:53-55`) — plaintext refused | `builders.rs:256-260` | `build_agent_observer_frame` |
| Reaction emoji length | > 64 **chars** ⇒ `EmojiTooLong` (char count, not bytes) | `builders.rs:467-469` | `build_reaction` |
| Emoji set uniqueness | duplicate normalized shortcode ⇒ `InvalidInput` (hard error, not silent dedup) | `builders.rs:517-521` | `build_custom_emoji_set` |
| Profile field omission | only `Some` fields land in the kind-0 JSON object | `builders.rs:542-561` | `build_profile` |
| Channel update non-empty | at least one of name/about/visibility/ttl required | `builders.rs:610-614` | `build_update_channel` |
| Channel visibility vocabulary | free-form `&str` restricted to `"open"`/`"private"` | `builders.rs:615-621` | `build_update_channel` |
| Channel name canonicalization | leading `#`/whitespace stripped via `canonical_channel_name`; result must not be blank | `builders.rs:622-627`, `634-639` (update); `675-679` (create) | create + update channel |
| TTL tri-state | `None` = unchanged; `Some(Some(secs))` = set; `Some(None)` = clear via `["ttl",""]` | `builders.rs:641-646` | `build_update_channel` |
| Contact list cap | > 10 000 contacts ⇒ `InvalidInput`; checked before dedup | `builders.rs:751`, `770-776` | `build_contact_list` |
| Contact pubkey format | exactly 64 hex chars, any case, normalized lowercase | `builders.rs:779-784` | `build_contact_list` |
| Contact relay URL / petname bounds | ≤2048 bytes / ≤256 bytes | `builders.rs:785-799` | `build_contact_list` |
| Contact dedup | duplicate pubkeys silently skipped, first occurrence kept | `builders.rs:800-804` | `build_contact_list` |
| Repo announcement bounds | `name` ≤128, `description` ≤1024, ≤5 `clone_urls` each non-empty and ≤512, `web_url` http(s) and ≤512, ≤10 `relays` each `ws://`/`wss://` and ≤256 | `builders.rs:840-919` | `build_repo_announcement` |
| Read-modify-write d-tag canonicalization | all caller `d` tags removed, exactly one validated `d` inserted at index 0; every other tag preserved | `builders.rs:958-963` | `build_repo_announcement_with_tags` |
| Patch content must be appliable | blank/whitespace-only content rejected ("refusing to publish an unappliable patch") | `builders.rs:1018-1022` | `build_git_patch` |
| Patch size bound | ≤60 KiB per NIP-34's patch-vs-PR guidance; never silently truncated | `builders.rs:1007-1012`, `1023` | `build_git_patch` |
| Patch root exclusivity | `root` and `root_revision` are mutually exclusive | `builders.rs:1038-1042` | `build_git_patch` |
| Issue subject | non-empty, ≤256 chars | `builders.rs:1086-1094` | `build_git_issue` |
| Merge-only status fields | `applied_patches`/`merge_commit`/`applied_as_commits` allowed only on `AppliedOrResolved` (1631) | `builders.rs:1250-1259` | `build_git_status` |
| `q`-tag hint ordering | pubkey hint without a relay hint ⇒ `InvalidInput` (NIP-34 positional shape) | `builders.rs:1260-1275` | `build_git_status` |
| PR subject | non-empty, ≤256 chars | `builders.rs:1335-1343` | `build_git_pull_request` |
| PR reachability inputs | full-length `commit` plus ≥1 `clone_urls` entry required | `builders.rs:1344-1350` | `build_git_pull_request` |
| PR channel id | `channel_id` must parse as a UUID; re-rendered canonically (lowercased, untrimmed input rejected) | `builders.rs:1370-1374`; tests `builders.rs:3491-3505` and `3506-3517` | `build_git_pull_request` |
| PR update required refs | `pr_event` 64-hex, `pr_author` 64-hex pubkey, full `commit`, ≥1 clone URL | `builders.rs:1421-1431` | `build_git_pr_update` |
| Workflow YAML bound | ≤64 KiB | `builders.rs:1468`, `1486` | workflow def/update |
| Approval token hash | exactly 64 hex chars (SHA-256 digest) | `builders.rs:1528-1532` | `build_workflow_approval` |
| Approval kind selection | `approved == true` ⇒ 46030, else 46031 | `builders.rs:1533-1537` | `build_workflow_approval` |
| DM participant count | 1–8 pubkeys | `builders.rs:1545-1549` | `build_dm_open` |
| Presence vocabulary | `"online"`/`"away"`/`"offline"` only; value duplicated into content and `status` tag | `builders.rs:1571-1584` | `build_presence_update` |
| Ban expiry semantics | `None` ⇒ permanent (no `expiration` tag); `Some(unix)` ⇒ temporary | `builders.rs:1593-1606` | `build_moderation_ban` |
| Timeout expiry required | `expires_at: u64` is a non-optional parameter | `builders.rs:1623-1633` | `build_moderation_timeout` |
| Report resolution vocabulary | `status` ∈ {resolved, dismissed}; `action` ∈ {delete, kick, ban, timeout, dismiss, escalate}; the status/action **pairing** is left to the relay | `builders.rs:1660-1680` | `build_moderation_resolve_report` |
| Rotation pointer distinctness | `replaced_by` must differ from `target_pubkey` | `builders.rs:1765-1774` | `build_archive_identity_request` |
| Unarchive has no rotation pointer | `replaced_by` is not a parameter at all; `identity_archive_tags` called with `None` | `builders.rs:1810-1823` | kind 9036 |

---

### 4. Mention-resolution rules (`mentions.rs`)

| Rule | Enforces | File:line | Trigger |
|---|---|---|---|
| `@` boundary rule | `@` must be at index 0 or preceded by ASCII whitespace — excludes `user@host` email forms | `mentions.rs:78-80`, `128-131` | both extractors |
| Name charset | `[A-Za-z0-9._-]` terminates the single-word token | `mentions.rs:85-90`, `144-147` | both extractors |
| Case folding + dedup | all names lowercased, first-seen order preserved | `mentions.rs:93-97`, `149-152` | both extractors |
| Longest-known-name-first | known names sorted by descending length; a match also requires a trailing word boundary | `mentions.rs:117-121`, `134-141`, `154-159` | `extract_at_mentions_with_known` |
| UTF-8 boundary safety | `rest.get(..k.len())` returns `None` on non-char-boundary rather than panicking | `mentions.rs:135-138`; test `mentions.rs:520-531` | multi-byte content |
| Profile name precedence | `display_name` first; `name` used **only when `display_name` is absent** (an empty `display_name` blocks the fallback) | `mentions.rs:186-193`; test `mentions.rs:560-569` | `match_names_to_profiles` |
| Silent skip on bad profiles | unparseable JSON, missing or non-string name ⇒ profile skipped | `mentions.rs:183-195` | `match_names_to_profiles` |
| Ambiguity allowed | duplicate display names yield multiple pubkeys by design | `mentions.rs:174-177`, `199-203` | `match_names_to_profiles` |
| Output order = profile order | matched pubkeys follow the profile slice, not text position | `mentions.rs:165-170`, `181-205`; test `mentions.rs:571-583` | `match_names_to_profiles` |
| Explicit-mentions priority | merge appends auto-resolved entries only within `cap - explicit.len()` budget | `mentions.rs:208-220` | `merge_mentions` |
| No self-mention | sender pubkey removed case-insensitively | `mentions.rs:228-241` | `normalize_mention_pubkeys` |
| Code regions excluded from scanning | fenced blocks (``` at line start) and same-line inline spans replaced by a single space; original content stored verbatim | `mentions.rs:244-341` | pre-pass to `extract_nostr_uris` |
| NIP-27 URI shape | requires literal `nostr:npub1` + exactly 58 bech32 chars; window must land on a char boundary; uppercase normalized before decode; invalid bech32 silently skipped | `mentions.rs:353-387` | `extract_nostr_uris` |

---

### 5. NIP-OA rules (`nip_oa.rs`)

| Rule | Enforces | File:line | Trigger |
|---|---|---|---|
| Signing preimage | `"nostr:agent-auth:" ‖ agent_pubkey_hex ‖ ":" ‖ conditions`, hashed with SHA-256, signed BIP-340 Schnorr | `nip_oa.rs:109-116`, `146-176` | compute + verify |
| Self-attestation rejected | `owner_pubkey == agent_pubkey` fails at **both** compute and verify | `nip_oa.rs:154-159`, `212-217` | both paths |
| Conditions grammar | empty allowed; otherwise `&`-joined clauses, each `kind=<0..65535>`, `created_at<<0..4294967295>`, or `created_at><0..4294967295>` | `nip_oa.rs:36-73` | compute, verify, parse |
| No whitespace in conditions | any ASCII whitespace anywhere ⇒ error | `nip_oa.rs:42-47` | all three |
| No empty clauses | leading/trailing/double `&` ⇒ error | `nip_oa.rs:50-57` | all three |
| Canonical decimals | no leading zeros (except `"0"`), digits only, range-checked | `nip_oa.rs:75-107` | all three |
| Case-sensitive clause labels | `Kind=`/`CREATED_AT<` rejected; `created_at=` (wrong operator) rejected | `nip_oa.rs:61-73`; tests `nip_oa.rs:541-546` | all three |
| Tag arity | JSON must be an array of exactly 4 elements, all strings, element 0 == `"auth"` | `nip_oa.rs:124-133`, `181-205`, `254-272` | verify + parse |
| Lowercase-hex requirement (parse path) | owner pubkey 64 lowercase hex, signature 128 lowercase hex — uppercase rejected | `nip_oa.rs:120-122`, `274-296`; test `nip_oa.rs:458-486` | `parse_auth_tag` |
| Crypto-free fast path | `parse_auth_tag` performs no signature verification (documented as the MCP-startup path) | `nip_oa.rs:239-251` | agent startup |

---

### 6. Defaulting and timestamp handling

- **No timestamp is ever set.** No builder calls `custom_created_at`; `created_at`
  is whatever `nostr::EventBuilder` assigns at signing time. The only
  timestamp-shaped values the SDK writes are caller-supplied: `["expiration",
  <unix>]` for ban/timeout (`builders.rs:1602`, `1632`), `["ttl", <secs>]` for
  channels (`builders.rs:642`, `697`), and the committer tuple's `ts`/`tz`
  strings (`builders.rs:1063-1065`).
- Defaulting is limited to: `DeleteMessageOptions::default()` for the
  no-metadata delete path (`builders.rs:403-409`), `#[derive(Default)]` on the
  five Git meta structs, and `unwrap_or("")` for NIP-02 relay/petname slots
  (`builders.rs:806-811`).
- Normalization defaults: pubkeys and event ids are lowercased on the way out
  (`builders.rs:69-89`), channel UUIDs are re-rendered from `Uuid`
  (canonical lowercase hyphenated form) rather than echoed from input strings.


## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: Business Rules

Rules are grouped by stage. Each row states what is enforced, where, and what triggers it.

---

### A. `.persona.md` parsing rules

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| A1 | File must begin with the literal `---` | `crates/buzz-persona/src/persona.rs:279-281` | any call to `split_frontmatter` / `parse_persona_md` | `PersonaError::NoFrontmatter` |
| A2 | Opening `---` must be followed by `\n` (an optional single `\r` is tolerated first) | `crates/buzz-persona/src/persona.rs:283-285` | `---junk` on line 1, or leading whitespace/newline before `---` | `PersonaError::NoFrontmatter` |
| A3 | Closing delimiter must be `\n---` followed by `\n`, `\r`, or EOF; `---junk` is skipped and the search continues | `crates/buzz-persona/src/persona.rs:287-306` | `---junk` inside a YAML block scalar | scanner advances past it and finds the real close (test: `crates/buzz-persona/src/persona.rs:492-501`) |
| A4 | No valid closing delimiter → parse failure | `crates/buzz-persona/src/persona.rs:293` | unterminated frontmatter | `PersonaError::NoFrontmatter` |
| A5 | Frontmatter ≤ 1 MiB | `crates/buzz-persona/src/persona.rs:213-215` (limit at `:21`) | oversized YAML block | `PersonaError::FrontmatterTooLarge` |
| A6 | Body ≤ 256 KiB | `crates/buzz-persona/src/persona.rs:216-218` (limit at `:24`) | oversized markdown body | `PersonaError::BodyTooLarge` |
| A7 | File size pre-check before read: `metadata.len() > MAX_FRONTMATTER + MAX_BODY + 100` | `crates/buzz-persona/src/persona.rs:263-267` | `parse_persona_file` on a >~1.25 MiB file | `PersonaError::TooLarge` (avoids the large allocation) |
| A8 | Unknown frontmatter keys are a **hard error** | `#[serde(deny_unknown_fields)]` at `crates/buzz-persona/src/persona.rs:175` | any key not in the `Frontmatter` struct (`:176-196`) | `PersonaError::Yaml` |
| A9 | Unknown keys inside `hooks:` are a hard error | `deny_unknown_fields` at `crates/buzz-persona/src/persona.rs:83` | e.g. `hooks.on_init` | `PersonaError::Yaml` (test: `:412-418`) |
| A10 | `name`, `display_name`, `description` are required | `crates/buzz-persona/src/persona.rs:222-228` | key absent | `PersonaError::MissingField("<field>")` |
| A11 | Those three must be non-empty after `trim()` | `crates/buzz-persona/src/persona.rs:231-239` | `name: ""` or `name: "   "` | `PersonaError::MissingField("name (empty)")` etc. |
| A12 | `respond_to` is accepted as an alias for `triggers` | `#[serde(alias = "respond_to")]` at `crates/buzz-persona/src/persona.rs:186` | legacy persona file | parsed into `triggers` |
| A13 | The markdown body — everything after the closing delimiter, with one leading `\r\n` or `\n` stripped — becomes `prompt` | `crates/buzz-persona/src/persona.rs:312-318`, assigned at `:257` | every parse | `PersonaConfig.prompt` |
| A14 | `prompt` cannot be set from YAML | `prompt` is absent from `Frontmatter` (`crates/buzz-persona/src/persona.rs:176-196`) | frontmatter containing `prompt:` | rejected by A8 |
| A15 | Empty body is valid | no non-empty check on `body`; test at `crates/buzz-persona/src/persona.rs:355-359` | persona with no prose | `prompt == ""` |

---

### B. `plugin.json` manifest rules

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| B1 | `id`, `name`, `version` required | `crates/buzz-persona/src/manifest.rs:155-159` | key absent | `ManifestError::MissingField` |
| B2 | Those three must be non-empty after `trim()` | `crates/buzz-persona/src/manifest.rs:161-170` | `"id": "   "` | `ManifestError::MissingField("id (empty)")` |
| B3 | Unknown top-level manifest keys are **silently accepted** at parse time (OPS superset) | no `deny_unknown_fields` on `RawManifest`, rationale at `crates/buzz-persona/src/manifest.rs:123-130` | foreign OPS fields | ignored (advisory warning later — see E2) |
| B4 | `personas` is optional and defaults to empty | `#[serde(default)]` at `crates/buzz-persona/src/manifest.rs:141-142`; test at `:245-251` | manifest with no `personas` key | `Vec::new()` (later rejected by D1) |
| B5 | `defaults.respond_to` accepted as alias for `defaults.triggers` | `alias = "respond_to"` at `crates/buzz-persona/src/manifest.rs:62-63` | legacy manifest | parsed into `BehavioralDefaults.triggers` |
| B6 | Wrong types inside `defaults` are rejected by serde at parse time | typed `BehavioralDefaults` (`crates/buzz-persona/src/manifest.rs:49-70`) | `"mentions": "yes"` | `ManifestError::Json` → surfaces as a load error (test: `crates/buzz-persona/src/validate.rs:900-936`) |
| B7 | `engines.buzz` is parsed but **never evaluated** | field at `crates/buzz-persona/src/manifest.rs:39-40`; no version comparison exists anywhere in the crate | pack declaring `engines.buzz: ">=99.0.0"` | no rejection (spec §2 says buzz-acp rejects; not implemented here) |
| B8 | `hooks_config` is parsed into `PackManifest` but dropped by the loader | dropped at `crates/buzz-persona/src/pack.rs:111-112` (comment: "hooks are a runtime concern loaded separately by buzz-acp") | manifest declaring `hooks_config` | pack-level hooks never reach `LoadedPack` |

---

### C. Pack loading rules (`load_pack`)

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| C1 | Pack directory must canonicalize (must exist) | `crates/buzz-persona/src/pack.rs:126-129` | nonexistent dir | `PackError::Io` |
| C2 | `.plugin/plugin.json` must exist | `crates/buzz-persona/src/pack.rs:132-135` | missing manifest | `PackError::ManifestNotFound` |
| C3 | Every path in `personas[]` must exist | `crates/buzz-persona/src/pack.rs:157-159` | dangling reference | `PackError::PersonaNotFound` |
| C4 | Persona files are size-capped at `MAX_FRONTMATTER + MAX_BODY + 200` bytes | limit computed at `crates/buzz-persona/src/pack.rs:153-154`, applied at `:160` via `read_bounded_file` (`:374-389`) | oversized file | `PackError::FileParse` ("file too large") |
| C5 | Absolute paths in the manifest are rejected (`/` prefix; on Windows also `X:`) | `crates/buzz-persona/src/pack.rs:330-337` | `"personas": ["/etc/passwd"]` | `PackError::PathTraversal` |
| C6 | Any `..` path component is rejected before canonicalization | `crates/buzz-persona/src/pack.rs:339-345` | `"../../etc/passwd"`, `"personas/../../x"` | `PackError::PathTraversal` (tests `:561-575`) |
| C7 | After canonicalization the path must still be inside `pack_root` (catches symlink escapes) | `crates/buzz-persona/src/pack.rs:359-364` | symlink in `personas/` pointing outside the pack | `PackError::PathEscape` (test `:590-607`) |
| C8 | Non-existent joined paths skip canonicalization and are returned as-is so the caller can emit "not found" | `crates/buzz-persona/src/pack.rs:349-355` | manifest referencing a missing file | `Ok(joined)` → C3 fires |
| C9 | Explicit `pack_instructions` path must exist; **implicit** `instructions.md` is optional | `crates/buzz-persona/src/pack.rs:170-181` vs `:182-188` | declared-but-missing file | `PackError::FileParse`; implicit-missing → `None` |
| C10 | Explicit `mcp_config` path must exist; **implicit** `.mcp.json` is optional | `crates/buzz-persona/src/pack.rs:198-207` vs `:208-217` | declared-but-missing file | `PackError::FileParse`; implicit-missing → `None` |
| C11 | `.mcp.json` must be syntactically valid JSON (any shape) | `crates/buzz-persona/src/pack.rs:192-197` | malformed JSON | `PackError::McpConfigParse` |
| C12 | `skills/` presence is detected only if it is a directory | `crates/buzz-persona/src/pack.rs:223-230` | `skills` as a plain file | `skills_dir == None` |
| C13 | Persona `runtime` is **not** subject to pack-defaults merging | `crates/buzz-persona/src/pack.rs:428` copies `pc.runtime` directly; `runtime` is not a field of `BehavioralDefaults` (`crates/buzz-persona/src/manifest.rs:49-70`) | pack trying to set a default runtime | ignored (advisory warning via E2) |
| C14 | MCP server entries missing `name`/`command` are dropped silently during JSON re-encode | `filter_map(...ok())` at `crates/buzz-persona/src/pack.rs:415-419` | serialization failure | entry omitted |

---

### D. Semantic validation (applied in `resolve_loaded_pack`)

These duplicate the checks in `validate.rs` but are enforced as hard errors on the
resolve path.

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| D1 | A pack must contain ≥ 1 persona | `crates/buzz-persona/src/resolve.rs:119-123` | `personas: []` | `PackError::ManifestParse("pack contains zero personas")` |
| D2 | Persona names must be unique within a pack | `crates/buzz-persona/src/resolve.rs:125-133` | two personas named `bot` | `PackError::FileParse("duplicate persona name ...")` |
| D3 | Persona name charset: `[a-zA-Z0-9_-]` only | `crates/buzz-persona/src/resolve.rs:134-145` | `name: "my bot"` | `PackError::FileParse("... invalid characters ...")` |
| D4 | Persona name ≤ 64 characters | `crates/buzz-persona/src/resolve.rs:146-155` | 65-char name | `PackError::FileParse("... exceeds 64 characters ...")` |

Note: `name.len()` is byte length, not char count — a multi-byte name shorter than 64
chars can still trip D4. Non-ASCII names are already rejected by D3, so the practical
effect is limited to ASCII input.

---

### E. Precedence / resolution rules (levels 3–5)

The module doc states this crate handles levels 3–5 only; levels 1–2 (operator env
vars, desktop UI) are resolved at runtime by the consumer
(`crates/buzz-persona/src/merge.rs:1-9`).

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| E1 | Per-persona frontmatter beats pack `defaults` beats built-in defaults | `merge_behavioral_config` at `crates/buzz-persona/src/merge.rs:47-83` | any field present on both sides | persona value wins (`:68-73`) |
| E2 | Persona value of `null` is treated as **absent** → falls through to pack default | `Some(Value::Null) | None => default_val.clone()` at `crates/buzz-persona/src/merge.rs:69-71` | `temperature: null` | pack default applies (test `:225-231`) |
| E3 | Persona keys not present in `defaults` are carried through unless null | `crates/buzz-persona/src/merge.rs:76-81` | persona sets a field the pack does not | persona value used |
| E4 | Non-object inputs short-circuit the merge | `crates/buzz-persona/src/merge.rs:53-60` | scalar JSON passed in | returns the other side unchanged |
| E5 | `subscribe: []` is **present**, not absent — it overrides the pack default | dedicated match at `crates/buzz-persona/src/merge.rs:106-113` | persona sets empty array | `Some(vec![])` = "subscribe to nothing" (test `:445-451`) |
| E6 | `subscribe` absent/null falls through to the pack default; if neither, `None` | `crates/buzz-persona/src/merge.rs:114-121` | neither side sets it | `None` → later becomes `vec![]` at `crates/buzz-persona/src/pack.rs:432` |
| E7 | Non-string entries inside `subscribe` are silently dropped | `filter_map(|v| v.as_str()...)` at `crates/buzz-persona/src/merge.rs:108-112` | `subscribe: [1, "x"]` in defaults JSON | only `"x"` survives |
| E8 | `triggers` uses **shallow replacement** — a persona `triggers` object discards the pack's object entirely | `crates/buzz-persona/src/merge.rs:131-152` (persona arm at `:147-149`) | persona sets only `mentions` | pack `keywords` are lost; missing sub-fields take **built-in** defaults (test `:353-368`) |
| E9 | `triggers: {}` is present-but-empty → all sub-fields take built-in defaults | same match arm as E8 + `parse_triggers` (`crates/buzz-persona/src/merge.rs:177-199`) | `triggers: {}` | `mentions=true`, `keywords=[]`, `all_messages=false` (test `:370-388`) |
| E10 | `triggers: null` or absent → pack default object used verbatim | `crates/buzz-persona/src/merge.rs:150-151` | legacy "unset" pattern | pack triggers applied (tests `:390-397`, `:340-351`) |
| E11 | Built-in trigger sub-defaults: `mentions=true`, `keywords=[]`, `all_messages=false` | `crates/buzz-persona/src/merge.rs:180-198` | any trigger object with missing sub-keys | those literals |
| E12 | Built-in `thread_replies=true`, `broadcast_replies=false` | constants at `crates/buzz-persona/src/merge.rs:38-39`, applied `:159-168` | neither persona nor pack sets them | those literals |
| E13 | `model` / `temperature` / `max_context_tokens` have **no** built-in default — they stay `None` | `crates/buzz-persona/src/merge.rs:95-97` | nothing sets them | `None` (consumer falls back) |
| E14 | Non-`Option` reads are type-tolerant: a wrong-typed merged value degrades to the default instead of erroring | `and_then(|v| v.as_f64())`, `as_bool()`, `as_u64()` at `crates/buzz-persona/src/merge.rs:96-97,159-168` | `temperature: "hot"` reaching the merge JSON | field becomes `None`/default silently |

---

### F. Prompt composition and template rules

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| F1 | `system_prompt` is the persona markdown body **verbatim** — no templating, no variable substitution, no escaping | `let system_prompt = lp.prompt.clone();` at `crates/buzz-persona/src/resolve.rs:200` | every resolve | body copied byte-for-byte |
| F2 | Pack instructions are kept **separate** from the system prompt (not concatenated) | `crates/buzz-persona/src/resolve.rs:201-204`, field at `:34`; asserted in test `crates/buzz-persona/tests/integration.rs:394-407` | pack with `instructions.md` | `ResolvedPersona.pack_instructions` |
| F3 | Pack instructions are trimmed; whitespace-only content becomes `None` | `.map(str::trim).filter(|s| !s.is_empty())` at `crates/buzz-persona/src/resolve.rs:202-203` | `instructions.md` containing only newlines | `None` |
| F4 | There is **no** `${VAR}` interpolation anywhere — MCP `env` values pass through as literals | `parse_mcp_server_config` at `crates/buzz-persona/src/resolve.rs:325-333`; doc note at `:274`; test `crates/buzz-persona/src/resolve.rs:558-572` | `env: { SECRET: "${MY_SECRET}" }` | stored as the literal string `"${MY_SECRET}"` |
| F5 | No per-persona version: `ResolvedPersona.version` is always the pack version | `crates/buzz-persona/src/resolve.rs:225-231` | persona declaring `version: 1.2.3` in frontmatter | frontmatter `version` is parsed into `PersonaConfig` (`crates/buzz-persona/src/persona.rs:116`) but **not** propagated to `LoadedPersona`/`ResolvedPersona` |

---

### G. MCP server merge rules

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| G1 | Pack `.mcp.json` servers form the base set, read from the `mcpServers` object | `crates/buzz-persona/src/resolve.rs:283-291` | `.mcp.json` present | keyed by object key |
| G2 | Per-persona `mcp_servers[]` are layered on top; **name collision → persona wins entirely** (no field merge) | `crates/buzz-persona/src/resolve.rs:293-300` | same server name on both sides | persona entry replaces (test `:513-535`) |
| G3 | A server entry without a string `command` is dropped silently | `?` on `command` in `parse_mcp_server_config` at `crates/buzz-persona/src/resolve.rs:312` | `.mcp.json` entry with no `command` | entry omitted, no diagnostic |
| G4 | A persona entry without a string `name` is skipped | `crates/buzz-persona/src/resolve.rs:294` | malformed persona MCP entry | entry omitted |
| G5 | Non-string `args` / `env` values are dropped; missing → empty | `crates/buzz-persona/src/resolve.rs:313-333` | `args: [1,2]` | empty args |
| G6 | Output order is deterministic: sorted by server name | `crates/buzz-persona/src/resolve.rs:303-306` | more than one server | alphabetical (test `:537-556`) |
| G7 | Any `.mcp.json` shape other than `{ "mcpServers": {...} }` contributes nothing | `get("mcpServers").and_then(as_object)` at `crates/buzz-persona/src/resolve.rs:285` | flat map of servers | silently ignored |

---

### H. Hooks rules

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| H1 | Hooks are parsed and carried, **never executed** by this crate | field comment at `crates/buzz-persona/src/resolve.rs:57`; no `Command`/`process` usage in the crate | pack with `hooks:` | stored only |
| H2 | Hook paths are stored as **raw relative strings**, deliberately not resolved | `resolve_hooks` at `crates/buzz-persona/src/resolve.rs:339-357`, security rationale at `:339-346` | `on_start: "../../evil.sh"` | stored verbatim; the crate documents that the PR wiring execution MUST run `safe_resolve()` first |
| H3 | A hooks object with all three fields `None` collapses to `None` | `crates/buzz-persona/src/resolve.rs:349-351` | `hooks: {}` in frontmatter | `ResolvedPersona.hooks == None` (test `:592-600`) |
| H4 | Pack-level `hooks/hooks.json` is never loaded | see B8 (`crates/buzz-persona/src/pack.rs:111-112`) | pack shipping `hooks_config` | ignored entirely |

---

### I. Env-var projection rules

`runtime_env_vars` — `crates/buzz-persona/src/resolve.rs:365-397`. Pure: it does **not**
read the process environment (doc at `:359-364`).

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| I1 | `model` splits on the **first** colon into provider + model id | `split_model` at `crates/buzz-persona/src/persona.rs:324-329`; used at `resolve.rs:371` and `:210-218` | `"databricks:gpt-5:preview"` | provider `databricks`, id `gpt-5:preview` (test `crates/buzz-persona/src/persona.rs:559-563`) |
| I2 | `runtime == "buzz-agent"` → emit `BUZZ_AGENT_MODEL` (+ `BUZZ_AGENT_PROVIDER` when a provider exists) | `crates/buzz-persona/src/resolve.rs:374-379` | persona `runtime: "buzz-agent"` | BUZZ_AGENT_* only; no GOOSE_* model/provider (test `crates/buzz-persona/tests/e2e_env_flow.rs:139-166`) |
| I3 | Any other runtime — including `None` and `"goose"` — falls into the GOOSE_* branch | wildcard `_ =>` arm at `crates/buzz-persona/src/resolve.rs:380-386` | `runtime: "claude"`, or absent | `GOOSE_PROVIDER` + `GOOSE_MODEL` (tests `crates/buzz-persona/src/resolve.rs:688-707`, `:709-717`) |
| I4 | Provider env var is omitted when the model has no colon prefix | `if let Some(p) = provider` at `crates/buzz-persona/src/resolve.rs:376,381` | `model: "gpt-4o"` | `GOOSE_MODEL` only (test `crates/buzz-persona/tests/e2e_env_flow.rs:333-356`) |
| I5 | `temperature` → `GOOSE_TEMPERATURE`, `max_context_tokens` → `GOOSE_CONTEXT_LIMIT`, regardless of runtime | `crates/buzz-persona/src/resolve.rs:390-396` with comment at `:389` | any runtime | always GOOSE_-prefixed |
| I6 | No model → no model/provider vars at all | `if let Some(model_str)` at `crates/buzz-persona/src/resolve.rs:369` | persona with no `model` | vars vector may be empty (test `crates/buzz-persona/src/resolve.rs:664-669`) |
| I7 | `llm_provider`/`model` on `ResolvedPersona` are `None` when the model string is empty or whitespace | guard at `crates/buzz-persona/src/resolve.rs:210-218` | `model: "   "` | both `None` — note `runtime_env_vars` uses a different guard (`Some(model_str)` only), so a whitespace model string would still project an env var |

---

### J. Skill scoping rules

`resolve_skills` — `crates/buzz-persona/src/pack.rs:249-321`. Not called from anywhere
else in the crate (`load_pack`/`resolve_pack` do not invoke it).

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| J1 | Declared skill paths are normalized to the final path component | `normalize_skill_name` at `crates/buzz-persona/src/pack.rs:253-260` | `"./skills/security-review/"` | `"security-review"` (test `:667-682`) |
| J2 | Only directories under `skills/` count; dotfiles/dot-dirs are skipped | `crates/buzz-persona/src/pack.rs:275-293` | `skills/.hidden`, `skills/README.md` | excluded |
| J3 | A skill claimed by ≥ 1 persona is private to the claiming personas | `claimed` set at `crates/buzz-persona/src/pack.rs:263-268` + shared filter `:296-301` | one persona lists `web-search` | other personas don't get it (test `:611-635`) |
| J4 | An unclaimed skill directory is granted to **all** personas | `crates/buzz-persona/src/pack.rs:296-312` | `skills/code-review` listed by nobody | every persona gets it (test `:637-652`) |
| J5 | Personas can claim skills that don't exist on disk — they are still returned | claimed names are emitted from `p.skills` unconditionally (`crates/buzz-persona/src/pack.rs:305`) with no existence check | `skills: ["ghost"]` | `"ghost"` appears in the map |
| J6 | No `skills/` directory → shared set is empty | `crates/buzz-persona/src/pack.rs:290-293` | pack without `skills/` | personas get only their declared names (test `:654-659`) |

---

### K. Validation rules (`validate_pack`)

Architecture: structural validation is delegated to `load_pack()`; advisory checks run
on raw files afterwards (`crates/buzz-persona/src/validate.rs:1-11`).

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| K1 | If `load_pack()` fails, one error is recorded and validation stops | `crates/buzz-persona/src/validate.rs:145-152` | any structural fault (C1–C14) | single `Error("pack failed to load: ...")` — later checks skipped, so only the first fault is reported |
| K2 | Zero personas → error, and duplicate/name checks are skipped | `crates/buzz-persona/src/validate.rs:189-192` | `personas: []` | `Error("pack contains zero personas")` |
| K3 | Duplicate persona names → error (one per extra duplicate) | `crates/buzz-persona/src/validate.rs:195-203` | two `bot` personas | `Error("duplicate persona name \"bot\"")` |
| K4 | Persona name ≤ 64 chars and `[a-zA-Z0-9_-]` only | `validate_persona_name` at `crates/buzz-persona/src/validate.rs:167-185` | `"my bot"`, 65-char name | `Error` (duplicates rules D3/D4) |
| K5 | Unknown top-level `plugin.json` key → **warning** | `crates/buzz-persona/src/validate.rs:319-324` against `KNOWN_MANIFEST_KEYS` (`:99-118`) | `"totally_made_up": true` | `Warning("plugin.json unknown key ...")` |
| K6 | Unknown key in `defaults` → warning | `crates/buzz-persona/src/validate.rs:326-333` against `KNOWN_BEHAVIORAL_KEYS` (`:121-130`) | `"temprature": 0.5` | `Warning` (test `:568-604`) |
| K7 | Unknown sub-key in `defaults.triggers` / `defaults.respond_to` → warning | `crates/buzz-persona/src/validate.rs:335-348` against `KNOWN_RESPOND_TO_KEYS` (`:133`) | `triggers.mentioned` | `Warning` |
| K8 | `defaults.triggers` sub-key type errors → **error** | `check_respond_to_value` at `crates/buzz-persona/src/validate.rs:236-287`, invoked `:210-234` | `mentions: "yes"`, `keywords: [42]` | `Error(...)` — in practice `load_pack` (B6) already fails first, so K1 short-circuits this path |
| K9 | `SKILL.md` `name:` differing from its directory name → warning | `advisory_check_skill_names` at `crates/buzz-persona/src/validate.rs:354-434`, comparison `:420-431` | dir `code-review` with `name: code_review` | `Warning` (test `:735-770`) |
| K10 | Skill dirs with no `SKILL.md`, unreadable content, or malformed frontmatter are skipped silently | `crates/buzz-persona/src/validate.rs:392-418` | missing/broken `SKILL.md` | no diagnostic (spec PF-5 lists this as unimplemented) |
| K11 | `SKILL.md` missing `name:`/`description:` is **not** an error | only `name` is read (`crates/buzz-persona/src/validate.rs:420`); `description` is never inspected | skill without `description:` | no diagnostic |
| K12 | Exit code contract: 0 clean, 1 errors, 2 warnings-only | `ValidationReport::exit_code` at `crates/buzz-persona/src/validate.rs:63-71` | — | i32 (note: `buzz-cli` does not use this value; it maps errors to `CliError::Usage` at `crates/buzz-cli/src/commands/pack.rs:37-44`) |
| K13 | Advisory checks read the raw manifest again and bail silently if unreadable/unparseable | `crates/buzz-persona/src/validate.rs:212-219`, `:304-312` | manifest deleted between load and check | checks skipped |

---

### L. Capability / permission gating

| # | Finding | Evidence |
|---|---|---|
| L1 | The crate has **no** tool-permission, allow-list, or capability model. There is no `permissions`, `allowed_tools`, `tools`, or `permission_mode` field in any struct. | full field lists in `crates/buzz-persona/src/persona.rs:101-170`, `manifest.rs:79-121`, `resolve.rs:23-65` |
| L2 | Agent capability is granted **indirectly** — a persona's `mcp_servers` entries (`command` + `args` + `env`) define which external tool processes the runtime will launch. | `crates/buzz-persona/src/persona.rs:70-79`; merged at `crates/buzz-persona/src/resolve.rs:277-307` |
| L3 | Operator-owned knobs are deliberately **rejected** in persona frontmatter — `idle_timeout`, `max_turn_duration`, `agents`, `heartbeat_interval`, `permission_mode` all fail `deny_unknown_fields`. The test calls this "the security boundary: pack authors define behavior, operators define limits". | `crates/buzz-persona/tests/integration.rs:637-650` |


## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: Business Rules

All rules below are read directly from `src/connection.rs` and `src/message.rs`.
The crate encodes protocol rules only (NIP-42 handshake, NIP-01 framing, timeouts);
it contains no domain/authorization logic.

---

### 1. NIP-42 authentication handshake — step by step

| # | Step | Code | file:line |
|---|---|---|---|
| 1 | Parse the URL string with `url::Url`; failure → `WsClientError::Url` | `url.parse::<url::Url>()` | `connection.rs:49`–`51` |
| 2 | Open the WebSocket with the **normalized** URL (`parsed.as_str()`, not the raw input) | `connect_async(parsed.as_str())` | `connection.rs:53`–`55` |
| 3 | Store the **original, un-normalized** `url` string as `relay_url` for later AUTH-event construction | `relay_url: url.to_string()` | `connection.rs:63` |
| 4 | Wait for the relay's `["AUTH", <challenge>]`, bounded by `AUTH_CHALLENGE_TIMEOUT_SECS` (20 s) | `wait_for_auth_challenge(Duration::from_secs(AUTH_CHALLENGE_TIMEOUT_SECS))` | `connection.rs:75`–`77`, const at `:17` |
| 5 | Challenge resolution order: (a) `pending_challenge` if already captured, (b) first buffered `RelayMessage::Auth`, (c) read from the socket | `pending_challenge.take()` / `buffer` scan / read loop | `connection.rs:161`–`163`, `:165`–`174`, `:178`–`214` |
| 6 | Reject a socket-read challenge longer than 1024 bytes → `AuthFailed("challenge exceeds 1024 bytes")` | `if challenge.len() > 1024` | `connection.rs:198`–`202` |
| 7 | Non-`AUTH` messages seen while waiting are pushed to `buffer` (never dropped) | `other => self.buffer.push_back(other)` | `connection.rs:205` |
| 8 | Build the AUTH event: parse `relay_url` into `nostr::RelayUrl`, call `EventBuilder::auth(challenge, url)`, append `auth_tag` when supplied, sign with `keys` | `build_auth_event` | `message.rs:174`–`190` (parse `:180`, builder `:181`, tag `:182`–`186`, sign `:188`) |
| 9 | Capture the AUTH event's hex id as the correlation key | `auth_event.id.to_hex()` | `connection.rs:80` |
| 10 | Send `["AUTH", <event>]` as a text frame | `send_raw(&json!(["AUTH", auth_event]))` | `connection.rs:82`, `:121`–`124` |
| 11 | Wait for `OK` whose `event_id` equals that hex id, bounded by `AUTH_OK_TIMEOUT_SECS` (20 s) | `wait_for_ok(&event_id, …)` | `connection.rs:84`–`86`, const at `:20` |
| 12 | If `ok.accepted == false` → `WsClientError::AuthFailed(ok.message)` (relay's reason string is propagated verbatim) | `if !ok.accepted` | `connection.rs:87`–`89` |
| 13 | On success, log at debug and return `Ok(())` — no state flag is recorded | `debug!("NIP-42 authentication successful")` | `connection.rs:91`–`92` |

`connect_authenticated` is exactly steps 1–13 composed
(`connection.rs:42`–`44`). `publish_event` runs steps 1–13, then `EVENT` + `OK`,
then close (`connection.rs:285`–`288`).

---

### 2. Rule catalogue

| Rule | What it enforces | Trigger | file:line |
|---|---|---|---|
| Challenge size cap | Challenge string must be ≤ 1024 bytes (byte length via `str::len`) | An `AUTH` message read from the socket inside `wait_for_auth_challenge` | `connection.rs:198`–`202` |
| Cap not applied on alternate paths | The same cap is **not** checked when the challenge comes from `pending_challenge` or from `buffer` | Challenge captured earlier by `recv_one`/`wait_for_ok` | `connection.rs:161`–`163`, `:170`–`171` (vs `:198`) |
| AUTH acceptance gate | Authentication only succeeds when the relay's `OK` for the AUTH event has `accepted == true` | `authenticate` after sending AUTH | `connection.rs:87`–`89` |
| OK correlation | An `OK` is only accepted when `ok.event_id` string-equals the locally computed hex event id; non-matching `OK`s are buffered | Every `OK` observed in `wait_for_ok` | `connection.rs:227`, `:254`, `:259` |
| Message preservation | Any relay message that is not the one being awaited is queued in `buffer` rather than discarded, so a later `next_event` still sees it | All wait loops | `connection.rs:205`, `:257`, `:259`; drain at `:108`, `:129` |
| Mid-flight AUTH capture | An `AUTH` challenge arriving while awaiting an `OK` is recorded in `pending_challenge` **and** buffered (so it is observable twice: as state and as a message) | `wait_for_ok` sees `RelayMessage::Auth` | `connection.rs:255`–`258` |
| Opportunistic AUTH capture | An `AUTH` returned to the caller via `next_event`/`recv_one` also updates `pending_challenge` | `recv_one` parses an `AUTH` | `connection.rs:143`–`145` |
| Keepalive | Inbound `Ping` must be answered with `Pong` carrying the same payload; the loop then continues waiting | Any `Ping` frame in any of the three loops | `connection.rs:148`–`150`, `:208`–`:210`, `:262`–`:264` |
| Close handling | An inbound `Close` frame, or a stream end (`None`), terminates the operation with `ConnectionClosed` | Any read loop | `connection.rs:137`, `:151`; `:190`, `:211`; `:247`, `:265` |
| Unknown frames ignored | Binary/Pong/raw-Frame variants are skipped without error | Any read loop | `connection.rs:152`, `:212`, `:266` |
| Strict message typing | The first array element must be a string and one of the **seven** known types; otherwise `UnexpectedMessage`. Was six at report time — the post-analysis sync added a `"COUNT"` arm (`message.rs:147`–`162`). | `parse_relay_message` | `message.rs:65`–`68`, `:163`–`165` |
| Required positional fields | `EVENT` requires `arr[1]` (sub id, string) and `arr[2]` (event object); `OK`/`EOSE`/`CLOSED`/`AUTH` require `arr[1]` as string | `parse_relay_message` | `message.rs:72`–`81`, `:88`–`92`, `:106`–`110`, `:116`–`120`, `:140`–`144` |
| Lenient optional fields | `OK.accepted` defaults `false` if missing/not a bool; `OK.message`, `CLOSED.message`, `NOTICE.message` default to `""` | `parse_relay_message` | `message.rs:93`, `:94`–`98`, `:121`–`125`, `:132`–`136` |
| Relay-URL validity for AUTH | `relay_url` must parse as `nostr::RelayUrl`, else `WsClientError::Url` | `build_auth_event` | `message.rs:180` |
| Optional NIP-OA tag | When `auth_tag` is `Some`, exactly that one tag is cloned onto the AUTH event; when `None`, the builder is left untouched | `build_auth_event` | `message.rs:182`–`186` |

---

### 3. Timeout values and how they are applied

| Constant / parameter | Value | Applied to | file:line |
|---|---|---|---|
| `AUTH_CHALLENGE_TIMEOUT_SECS` | 20 s | Waiting for the `AUTH` challenge; expiry → `NoAuthChallenge` | const `connection.rs:17`; use `:76`; expiry `:184`, `:189` |
| `AUTH_OK_TIMEOUT_SECS` | 20 s | Waiting for the `OK` to the AUTH event; expiry → `Timeout` | const `connection.rs:20`; use `:85`; expiry `:241`, `:246` |
| `PUBLISH_OK_TIMEOUT_SECS` | 30 s | Waiting for the `OK` to a published `EVENT` | const `connection.rs:23`; use `:99` |
| `timeout_dur` (caller-supplied) | caller's choice | `next_event`/`recv_one` single-message read; expiry → `Timeout` | `connection.rs:104`, `:134`, `:136` |
| `timeout_secs` (caller-supplied) | caller's choice (`buzz-cli` passes `75`) | Wraps the whole connect+auth+publish+close sequence in `publish_event`; expiry → `Timeout` | `connection.rs:282`, `:284`, `:292`; caller `crates/buzz-cli/src/client.rs:1080` |

Deadline discipline: both multi-iteration waiters compute an absolute deadline once
(`connection.rs:176`, `:222`) and re-derive `remaining` each iteration with
`checked_duration_since(...).unwrap_or(Duration::ZERO)` (`:179`–`181`, `:236`–`238`),
returning the timeout error when `remaining.is_zero()` (`:183`–`185`, `:240`–`242`).
So the budget is for the whole wait, not per frame. `recv_one` by contrast applies
`timeout_dur` **per socket read** inside its loop (`connection.rs:133`–`134`), so a
stream of `Ping`s can extend its total wall time.

---

### 4. Retry / reconnect / backoff

**None.** Verified across all 314 lines of `connection.rs`: there is no retry loop,
no attempt counter, no backoff constant, no reconnect helper, and no
`connect_authenticated` re-invocation on failure. `publish_event` establishes one
fresh connection per call (`connection.rs:285`) and returns the first error it hits;
its `disconnect` result is explicitly discarded (`let _ = conn.disconnect().await;`
— `connection.rs:288`), so a failing close never masks the `OK` result. Recovery
after `ConnectionClosed` is entirely the caller's responsibility.

Re-authentication is also not automated: a mid-session `AUTH` challenge is only
recorded/buffered (`connection.rs:255`–`258`); the caller must call `authenticate`
again to consume it via `pending_challenge` (`connection.rs:161`).

---

### 5. Error classification

| Condition | Error produced | file:line |
|---|---|---|
| Bad URL string | `Url(String)` | `connection.rs:51`; `message.rs:180` |
| Upgrade/transport failure, read error, send error | `WebSocket(tungstenite::Error)` | `connection.rs:55`, `:138`, `:191`, `:248`; `#[from]` at `error.rs:8` |
| JSON encode of an outbound frame, or decode of an inbound frame | `Json(serde_json::Error)` | `connection.rs:122`; `message.rs:63`, `:77`–`74`; `#[from]` at `error.rs:12` |
| Malformed/unknown relay message | `UnexpectedMessage(String)` (carries the raw frame, or `"unknown message type: {other}"`) | `message.rs:68`, `:75`, `:80`, `:91`, `:109`, `:119`, `:143`, `:163`–`165` |
| No challenge within budget | `NoAuthChallenge` | `connection.rs:184`, `:189` |
| Oversized challenge, or relay `OK` with `accepted == false` for the AUTH event | `AuthFailed(String)` | `connection.rs:199`–`201`, `:88` |
| Awaited `OK` did not arrive in time; or whole-operation timeout | `Timeout` | `connection.rs:136`, `:241`, `:246`, `:292` |
| Stream ended or `Close` frame received | `ConnectionClosed` | `connection.rs:137`, `:151`, `:190`, `:211`, `:247`, `:265` |
| Event signing / builder failure | `EventBuilder(String)` | `message.rs:189`; `From` impl at `error.rs:47`–`51` |
| Relay rejected a published event | **Not raised by this crate.** `send_event` returns `Ok(OkResponse { accepted: false, … })` and leaves interpretation to the caller; `EventRejected` (`error.rs:40`) is never constructed here | `connection.rs:96`–`101` vs `error.rs:40` |

---

### 6. Challenge validation / replay considerations (as coded)

- The only validation performed on a relay challenge is the 1024-byte length cap
  (`connection.rs:198`). There is no charset check, no entropy check, no
  nonce-uniqueness tracking, and no rejection of a repeated challenge value.
- Freshness is delegated: the AUTH event's `created_at` is set by
  `nostr::EventBuilder::auth` (`message.rs:181`); this crate performs no timestamp
  computation or tolerance check of its own (verified — no `Timestamp`, `now()`, or
  clock-skew constant anywhere in the crate).
- `pending_challenge` holds only the most recent challenge — a second `AUTH` frame
  overwrites the first (`connection.rs:144`, `:256`), and the value is cleared only
  by `take()` in `wait_for_auth_challenge` (`connection.rs:161`).


## Module: buzz-db (`crates/buzz-db`)

### Aspect: Business Rules

Rules are grouped by concern. "Trigger" is the call/statement path that
evaluates the rule.

---

#### 1. Event admission

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 1.1 | Kind 22242 (`KIND_AUTH`) is never stored — it carries bearer tokens | `crates/buzz-db/src/event.rs:264-266` (also `:1024-1026` in the tx path) → `DbError::AuthEventRejected` (`crates/buzz-db/src/error.rs:20-22`) | `insert_event`, `insert_event_with_thread_metadata` |
| 1.2 | Ephemeral kinds 20000–29999 are never stored (`is_ephemeral`, from buzz-core) | `crates/buzz-db/src/event.rs:267-269`, `:1027-1029` → `DbError::EphemeralEventRejected(kind)` (`crates/buzz-db/src/error.rs:24-26`) | same |
| 1.3 | Event dedup is `ON CONFLICT DO NOTHING` on `(community_id, created_at, id)`; the caller learns whether the row was new via `was_inserted` | `crates/buzz-db/src/event.rs:286-290`, `:294-299` | every insert path |
| 1.4 | Dedup is per community — the same signed event may exist in two communities | PK `(community_id, created_at, id)` `migrations/0001_initial_schema.sql:234`; test `crates/buzz-db/src/event.rs:1642-1689` | insert into two communities |
| 1.5 | `d_tag` is materialized only for NIP-33 kinds (30000–39999); a missing `d` tag stores `""` | `crates/buzz-db/src/event.rs:157-178` | every insert |
| 1.6 | `not_before` is materialized only for kind 30300 and only when the tag parses as `i64` | `crates/buzz-db/src/event.rs:182-197` | every insert |
| 1.7 | Rows whose stored JSON cannot be rebuilt into a `nostr::Event` are **skipped** (warn), not fatal | `crates/buzz-db/src/event.rs:579-584` (the `warn!` is at `:582`); same policy in `thread.rs:454-457`, `:657-660` | any read path |
| 1.8 | A channel-bearing `events` row whose `created_at` is more than `buzz.created_at_floor` seconds before **commit** time aborts the transaction (`check_violation`). Channel-NULL rows are structurally exempt; sessions without the GUC are unguarded | `migrations/0021_created_at_fence_floor.sql:44-68` (function), `:70-74` (deferred constraint trigger); GUC armed at `crates/buzz-db/src/lib.rs:394-405` | COMMIT of any insert/update of `created_at`/`channel_id` |

#### 2. Replaceable-event ordering (NIP-16 / NIP-33)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 2.1 | Replacement is serialized per `(community, kind, pubkey, [coordinate])` by a transaction-scoped advisory lock keyed with FNV-1a | `crates/buzz-db/src/lib.rs:63-89`, used at `:3320-3336`, `:3502-3512`, `:3654-3664` | `replace_addressable_event`, `publish_nip43_membership_locked`, `replace_parameterized_event` |
| 2.2 | Canonical ordering: newer `created_at` wins; on a same-second tie the **lowest** event id wins. Incoming events that are dominated are rejected with `(event, false)` and the tx rolls back | `crates/buzz-db/src/lib.rs:3358-3372` (addressable), `:3719-3737` (parameterized) | both replace paths |
| 2.3 | Addressable replacement keys on `(kind, pubkey, channel_id)` using `IS NOT DISTINCT FROM` for NULL safety | `crates/buzz-db/src/lib.rs:3345-3352`, `:3378-3386` | `replace_addressable_event` |
| 2.4 | NIP-33 replacement keys on `(kind, pubkey, d_tag)` **globally** — `channel_id` is stored for query scoping only and is not part of the replacement key | documented and implemented at `crates/buzz-db/src/lib.rs:3607-3626`, `:3697-3706` | `replace_parameterized_event` |
| 2.5 | If the insert hits `ON CONFLICT` after the old row was already soft-deleted, the whole tx rolls back so the previous replaceable event is not lost | `crates/buzz-db/src/lib.rs:3407-3417`, `:3798-3805` | duplicate id replay |
| 2.6 | Reads of the latest replaceable row use the same `ORDER BY created_at DESC, id ASC LIMIT 1` as the write path (defensive against historical duplicate survivors) | `crates/buzz-db/src/event.rs:935-962`; comment at `crates/buzz-db/src/lib.rs:3338-3340` | `get_latest_global_replaceable` |

#### 3. NIP-RS (kind 30078 read-state) retention

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 3.1 | An event is classified NIP-RS only under **exact** tag cardinality: kind 30078, exactly one `d` tag whose value equals the coordinate, `d_tag` matching `^read-state:[0-9a-f]{32}$` (lowercase only), and exactly one `["t","read-state"]` tag | `crates/buzz-db/src/lib.rs:3672-3687`; DB mirror `migrations/0011_nip_rs_exact_tag_cardinality.sql:66-88` | `replace_parameterized_event` and every `events` INSERT |
| 3.2 | Superseded NIP-RS payloads are **hard-deleted** (not soft-deleted), and their `event_mentions` rows are deleted after the event row (fixed lock order to avoid deadlock with the mention fence) | `crates/buzz-db/src/lib.rs:3739-3782` | NIP-RS replacement |
| 3.3 | A compact ordering watermark `(community, kind, pubkey, d_tag) → (created_at, event_id)` survives payload deletion and blocks stale resurrection even after a NIP-09 coordinate delete | table `migrations/0007_nip_rs_retention.sql:14-22`; Rust probe `crates/buzz-db/src/lib.rs:3709-3723`; upsert `:3784-3796`; DB trigger `migrations/0011_…:89-121` | replacement + any raw INSERT |
| 3.4 | Exact replay of an already-watermarked coordinate is a silent no-op (`RETURN NULL`) rather than an error — so concurrent physical deletion cannot open a resurrection window | `migrations/0010_nip_rs_exact_replay_guard.sql:34-47`; carried forward at `migrations/0011_…:104-116` | BEFORE INSERT trigger |
| 3.5 | Any other stale NIP-RS insert raises `check_violation` with `'stale NIP-RS event rejected by durable watermark'` | `migrations/0011_nip_rs_exact_tag_cardinality.sql:118-119` | BEFORE INSERT trigger |
| 3.6 | A legacy soft-delete of a NIP-RS row is converted to a physical purge (event + mentions) by an AFTER UPDATE trigger, covering pre-fix relay binaries during rolling deploys | `migrations/0009_nip_rs_database_guards.sql:77-108`; corrected body `migrations/0011_…:123-160` | `UPDATE events SET deleted_at` |
| 3.7 | Direct `DELETE` of a read-state-coordinate row is **refused** unless the transaction has opted in with `set_config('buzz.nip_rs_hard_delete','on',true)` | `migrations/0011_nip_rs_exact_tag_cardinality.sql:45-60`; opt-in at `crates/buzz-db/src/lib.rs:3742-3747` | any `DELETE FROM events` on a matching row |
| 3.8 | Mention inserts for kind 30078 take `FOR KEY SHARE` on the live event and are silently dropped if the event is already gone | `migrations/0009_nip_rs_database_guards.sql:111-137` | `INSERT INTO event_mentions` |
| 3.9 | Migration `0007` refuses to run when a pre-0007 database holds kind-30078 rows with ambiguous `d`/`t` cardinality; the operator must repair first | `crates/buzz-db/src/migration.rs:34-96` (`reject_legacy_nip_rs_cardinality_ambiguity`) | `run_migrations` on a DB at version < 7 |

#### 4. Mesh-status retention (kind 30003, `buzz-mesh-member-status:*`)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 4.1 | A kind:30003 coordinate whose `d_tag` starts with `buzz-mesh-member-status:` and which carries `["k","buzz-mesh-status"]` hard-deletes its superseded payload (heartbeat, no historical value) | `crates/buzz-db/src/lib.rs:3688-3695`, `:3739-3782` | `replace_parameterized_event` |
| 4.2 | Legacy soft-deletes of those rows are purged physically by trigger | `migrations/0019_mesh_status_retention.sql:21-45` | `UPDATE events SET deleted_at` |

#### 5. Channel membership and roles

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 5.1 | `created_by` / member `pubkey` must be exactly 32 bytes | `crates/buzz-db/src/channel.rs:96-101`, `:184-189`, `:355-360`; DB CHECK on `users.pubkey` `migrations/0001_initial_schema.sql:171` | `create_channel*`, `add_member` |
| 5.2 | Channel name is canonicalised (`buzz_core::channel::canonical_channel_name`) and must be non-empty | `crates/buzz-db/src/channel.rs:103-106`, `:191-194`, `:1063-1068` | `create_channel*`, `update_channel` |
| 5.3 | A client-supplied channel id may not be the nil UUID (reserved for global fan-out) | `crates/buzz-db/src/channel.rs:191-195` | `create_channel_with_id` |
| 5.4 | The creator is bootstrapped as `'owner'` in the **same transaction** as the channel insert | `crates/buzz-db/src/channel.rs:110-152` and `:220-241` | `create_channel*` |
| 5.5 | Private channel joins require an `invited_by` who is an active member; the creator adding themselves is the only bootstrap exception | `crates/buzz-db/src/channel.rs:368-385` | `add_member` |
| 5.6 | Elevated roles (`owner`, `admin`) may only be granted by an active `owner`/`admin` — checked for both private and open channels; open-channel self-join always yields `Member` | `crates/buzz-db/src/channel.rs:392-397` (private), `:399-417` (open) | `add_member` |
| 5.7 | The entire read-role → insert sequence runs in one transaction (TOCTOU-safe: the inviter cannot be removed between check and insert) | `crates/buzz-db/src/channel.rs:362`, `:453-456` | `add_member` |
| 5.8 | Removing another member requires an elevated actor **or** being the target's `agent_owner_pubkey`; self-removal always allowed | `crates/buzz-db/src/channel.rs:470-489` | `remove_member` |
| 5.9 | The **last owner** of a channel cannot be removed, regardless of caller ("transfer ownership first") | `crates/buzz-db/src/channel.rs:491-510` | `remove_member` |
| 5.10 | Member removal is a **soft delete** (`removed_at`, `removed_by`); re-adding reverses it via `ON CONFLICT … DO UPDATE SET removed_at=NULL, removed_by=NULL, role=EXCLUDED.role` | `crates/buzz-db/src/channel.rs:512-527` (remove) and `:419-433` (re-add) | `remove_member` / `add_member` |
| 5.11 | Every membership read joins `channels … deleted_at IS NULL`, so a soft-deleted channel has no members | `crates/buzz-db/src/channel.rs:531-552`, `:581-608`, `:610-636`, `:1363-1385` | membership reads |
| 5.12 | "Accessible channels" = active memberships ∪ all `visibility='open'` channels | `crates/buzz-db/src/channel.rs:638-667`; DM rows additionally require `cm.hidden_at IS NULL` at `:822` | REQ filter resolution |
| 5.13 | `channels.community_id` is immutable — a BEFORE UPDATE trigger raises `check_violation` on any change | `migrations/0001_initial_schema.sql:115-128` | any `UPDATE channels` |

#### 6. Channel TTL / ephemeral channels

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 6.1 | `ttl_deadline` is initialised to `NOW() + ttl_seconds` at create, and reset on any TTL change | `crates/buzz-db/src/channel.rs:117-119`, `:1099-1105` | `create_channel*`, `update_channel` |
| 6.2 | Every durable channel-scoped event **except kind 9007** refreshes `ttl_deadline` for ephemeral channels, inside the commit of that event | `migrations/0024_event_ttl_refresh_shared_lock.sql:25-57` (replacing `0022`) | deferred AFTER INSERT on `events` |
| 6.3 | TTL refresh is synchronized with TTL transitions via a per-channel advisory lock: **shared** on the event path, **exclusive** in `update_channel`. Permanent channels take no row lock at all | `migrations/0024_…:31-33` (shared); `crates/buzz-db/src/channel.rs:1131-1147` (exclusive) | event commit vs TTL change |
| 6.4 | A TTL-refresh failure must never reject an otherwise valid event — the trigger swallows exceptions into a `WARNING` | `migrations/0024_…:48-53` | trigger body |
| 6.5 | Expired ephemeral channels are archived idempotently (`archived_at IS NULL` guard), skipping archived communities; the reaper returns `(community_id, host, channel_id)` so side effects run under the right tenant | `crates/buzz-db/src/channel.rs:1387-1417` | `reap_expired_ephemeral_channels` |
| 6.6 | Unarchiving an expired ephemeral channel renews its lease so the reaper does not immediately re-archive it | `crates/buzz-db/src/channel.rs:1276-1288` | `unarchive_channel` |
| 6.7 | `archive_channel` rejects an already-archived channel and `unarchive_channel` rejects a non-archived one, both with `AccessDenied` | `crates/buzz-db/src/channel.rs:1219-1230`, `:1261-1271` | those calls |

#### 7. Thread counter materialization

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 7.1 | `reply_count` counts **direct** children; `descendant_count` counts all descendants — the root is bumped even when root == parent | `crates/buzz-db/src/thread.rs:236-249` (increment), `:292-333` (decrement) | reply insert/delete |
| 7.2 | Counters are only bumped when the `thread_metadata` row was actually inserted (`rows_affected > 0` after `ON CONFLICT DO NOTHING`) | `crates/buzz-db/src/thread.rs:152-154`; `crates/buzz-db/src/event.rs:1154-1156` | reply insert |
| 7.3 | Missing parent/root rows are stubbed (`depth=0, broadcast=false`) so the counter `UPDATE` has a target | `crates/buzz-db/src/thread.rs:158-207`; `crates/buzz-db/src/event.rs:1157-1211` | reply insert |
| 7.4 | Insert + all counter updates share one transaction, so a crash cannot desynchronise counters from rows | `crates/buzz-db/src/thread.rs:130`, `:245` (commit); `crates/buzz-db/src/event.rs:1221-1240` | reply insert |
| 7.5 | Decrements floor at zero (`GREATEST(x - 1, 0)`) | `crates/buzz-db/src/thread.rs:296`, `:319`; `crates/buzz-db/src/event.rs:818`, `:789` | reply delete |
| 7.6 | Soft-deleting an event and decrementing counters is atomic | `crates/buzz-db/src/event.rs:797-844` | `soft_delete_event_and_update_thread` |

#### 8. Reactions

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 8.1 | One reaction per `(community, target event, pubkey, emoji)`; the PK is the uniqueness | `migrations/0001_initial_schema.sql:548` | insert |
| 8.2 | `add_reaction` is a single upsert whose `DO UPDATE … WHERE reactions.removed_at IS NOT NULL` gives three outcomes: new (1 row), reactivation (1 row), active duplicate (0 rows) — no TOCTOU window | `crates/buzz-db/src/reaction.rs:66-112` | `add_reaction`, `add_reaction_tx` |
| 8.3 | `reaction_event_id` is only ever filled in, never cleared, on reactivation (`COALESCE(EXCLUDED…, reactions…)`) | `crates/buzz-db/src/reaction.rs:71` | reactivation |
| 8.4 | Reaction removal is a soft delete (`removed_at`), so history is retained | `crates/buzz-db/src/reaction.rs:140-172`, `:174-195` | `remove_reaction*` |
| 8.5 | A kind:7 reaction event is stored only after the reaction row is confirmed new/reactivated; an active duplicate short-circuits **before** the event insert and rolls back | `crates/buzz-db/src/event.rs:1271-1285` | `insert_reaction_event_with_thread_metadata` |
| 8.6 | A reaction whose target does not exist (or is soft-deleted) **in this community** commits nothing and returns `TargetMissing` | `crates/buzz-db/src/event.rs:1254-1269` | same |
| 8.7 | `get_reactions`' `limit` bounds **emoji groups**, not raw rows, so one busy emoji cannot hide other groups | `crates/buzz-db/src/reaction.rs:283-315` | `get_reactions` |

#### 9. Feeds

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 9.1 | Hard cap `FEED_MAX_LIMIT = 100`: every feed query applies `limit.min(FEED_MAX_LIMIT)` before building SQL | `crates/buzz-db/src/feed.rs:29`, applied at `:96`, `:157`, `:209` | all three feed queries |
| 9.2 | An **empty** accessible-channel list means "community-global events only", never "all channels" | `crates/buzz-db/src/feed.rs:59-74`; unit tests `:766-786` | all three feed queries |
| 9.3 | Mentions are restricted to kinds `{9, 40002 (v2), forum post, forum comment}`; needs-action to `{workflow approval requested, stream reminder}`; activity to `{stream msg, v2, forum post, job request/progress/result}` — workflow execution kinds are deliberately excluded from activity | `crates/buzz-db/src/feed.rs:104-107`, `:165-167`, `:216-219`; assertions `:882-935` | feed queries |
| 9.4 | Feed reads never surface soft-deleted events (`e.deleted_at IS NULL`) | `crates/buzz-db/src/feed.rs:103`, `:164`, `:214` | feed queries |

#### 10. Query scoping and pagination

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 10.1 | `before_id` requires `until`; `global_only` and `channel_id` are mutually exclusive — both are `InvalidData` errors | `crates/buzz-db/src/event.rs:320-333` | `query_events` |
| 10.2 | An explicitly empty `kinds`/`authors`/`ids`/`e_tags` list means "match nothing": early `Ok(vec![])` / `Ok(0)` | `crates/buzz-db/src/event.rs:336-348`, `:559-571` | `query_events`, `count_events` |
| 10.3 | `Some(empty)` `channel_ids` means "no channel access" → only `channel_id IS NULL` rows (explicitly annotated `SECURITY`) | `crates/buzz-db/src/event.rs:389-394`, `:601-604` | `query_events`, `count_events` |
| 10.4 | Default limit 100, clamped to `max_limit` (default 1000) | `crates/buzz-db/src/event.rs:350-352` | `query_events` |
| 10.5 | Channel-access filtering is pushed into SQL so `LIMIT` applies to the **visible** set (exact exhaustion semantics) | `crates/buzz-db/src/event.rs:387-406`; regression test `:1740-1789` | `query_events` |
| 10.6 | `has_more` for a channel window comes from an internal `LIMIT n+1` probe evaluated after all predicates; the sentinel row never leaves the module and callers must not re-derive exhaustion from row counts | `crates/buzz-db/src/thread.rs:556-563`, `:645-664` | `get_channel_window` |
| 10.7 | The channel-window cursor is captured from the last **raw** row before event reconstruction, so a row that fails to reconstruct cannot stall the cursor | `crates/buzz-db/src/thread.rs:650-663` | `get_channel_window` |
| 10.8 | Thread pagination uses a composite `(event_created_at, event_id)` keyset; the legacy 8-byte timestamp-only cursor is still accepted and documented as unsafe across same-second ties | `crates/buzz-db/src/thread.rs:337-344`, `:365-379`, `:400-419` | `get_thread_replies` |
| 10.9 | A channel window's "top-level" set is `depth IS NULL OR depth = 0 OR (depth = 1 AND broadcast)` | `crates/buzz-db/src/thread.rs:589-593` | `get_channel_window` |
| 10.10 | `get_events_by_ids` expects callers to bound batches (`debug_assert!(ids.len() <= 500)`) | `crates/buzz-db/src/event.rs:997` | `get_events_by_ids` |

#### 11. Read-replica routing (replica fence)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 11.1 | The fence starts **closed**; a closed or stale (> 30 s) fence routes every cursor page to the writer | `crates/buzz-db/src/replica_fence.rs:89-93`, `:114-127` | any routed read |
| 11.2 | Head fetches (`cursor: None`) always read the writer | `crates/buzz-db/src/lib.rs:2004-2043` (thread), `:2063-2077` (channel window) | routed reads |
| 11.3 | A channel-window cursor page may use the replica only when `fence.covers(cursor_ts)` | `crates/buzz-db/src/lib.rs:2070-2073` | `get_channel_window` |
| 11.4 | A thread cursor page from the replica is accepted only when it is **full** (≥ limit) **and** its tail `created_at` is ≤ fence; otherwise it is re-run on the writer (prevents a lag-truncated false EOF and mid-page holes) | `crates/buzz-db/src/lib.rs:2016-2032` | `get_thread_replies` |
| 11.5 | Fence value = `min(oldest_open_xact_start, sampled_at) − CREATED_AT_FLOOR_SECS(960) − FENCE_CLOCK_MARGIN_SECS(5)`, and only after the replica has replayed past the sampled writer LSN | `crates/buzz-db/src/replica_fence.rs:466-486` | `probe_once` |
| 11.6 | The handshake must sample in order S → activity scan → L, as three separately awaited statements on one pinned connection | `crates/buzz-db/src/replica_fence.rs:404-447` | `sample_writer` |
| 11.7 | Fails closed on: any probe error, masked `pg_stat_activity` rows, NULL/absent replica replay LSN, or a target that is not in recovery | `crates/buzz-db/src/replica_fence.rs:378-395`, `:449-463`, `:492-502` | probe loop |
| 11.8 | The probe is only spawned after **two** verifications pass: catalog shape of the floor-guard trigger on the parent and every partition, and observed behaviour through the armed pool | `crates/buzz-db/src/lib.rs:449-462`; `crates/buzz-db/src/replica_fence.rs:145-196`, `:199-330` | `spawn_fence_probe` |
| 11.9 | `run_migrations` itself fails closed if the floor-guard trigger is missing or mis-shaped anywhere | `crates/buzz-db/src/migration.rs:25` | `run_migrations` |

#### 12. Community / tenant lifecycle

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 12.1 | Host → community resolution is case-insensitive and **excludes archived** communities; the caller turns `None` into a fail-closed connection error | `crates/buzz-db/src/lib.rs:656-683` | `lookup_community_by_host` |
| 12.2 | An operator-plane variant deliberately resolves archived communities too | `crates/buzz-db/src/lib.rs:696-715` | `lookup_community_by_host_for_management` |
| 12.3 | `MAX_COMMUNITIES_PER_OWNER = 3`, enforced inside the same transaction that holds a per-owner FNV-1a advisory lock, for both create and transfer | `crates/buzz-db/src/relay_members.rs:379`, `:410-417`; create `crates/buzz-db/src/lib.rs:869-916`; transfer `crates/buzz-db/src/relay_members.rs:444-493` | `create_community_with_owner`, `transfer_ownership` |
| 12.4 | Limit rejection **rolls back** the freshly inserted `communities` row. The threshold is no longer the bare constant: `2a051a40` changed the comparison to `relay_members::max_communities_per_owner()`, so the effective cap can be raised via `BUZZ_MAX_COMMUNITIES_PER_OWNER`. The rollback behaviour itself is unchanged. | `crates/buzz-db/src/lib.rs:901-904`; test `create_community_with_owner_enforces_per_owner_limit` `:4856-4887` (rollback assertion `:4880-4886`) | `create_community_with_owner` |
| 12.5 | An identical create retry by the same owner returns the original row; a different owner on the same host gets `HostExists` | `crates/buzz-db/src/lib.rs:918-940` | `create_community_with_owner` |
| 12.6 | Archival is idempotent (`COALESCE(archived_at, now())`), owner-gated, and refuses the protected deployment host | `crates/buzz-db/src/lib.rs:947-978` | `archive_community_owned_by` |
| 12.7 | `bootstrap_owner` upserts the configured owner and demotes other owners to `'admin'`; `transfer_ownership` demotes them to `'member'` | `crates/buzz-db/src/relay_members.rs:320-360` vs `:495-515`; the difference is called out at `:432-434` | startup vs user transfer |
| 12.8 | Ownership transfer requires the caller's `expected_owner_pubkey` to match a current owner row locked `FOR UPDATE`; otherwise `OwnerConflict` and the caller must re-read | `crates/buzz-db/src/relay_members.rs:453-474` | `transfer_ownership` |
| 12.9 | `relay_members` removal never deletes an owner (`role <> 'owner'` in the DELETE), and the role-conditional variant distinguishes `RoleMismatch` from `IsOwner` | `crates/buzz-db/src/relay_members.rs:196-240`, `:242-285` | `remove_relay_member*` |
| 12.10 | Relay-member role updates cannot touch an owner (`role <> 'owner'`) | `crates/buzz-db/src/relay_members.rs:287-303` | `update_relay_member_role` |
| 12.11 | Invite-claimed membership persists the accepted policy version **in the same transaction** — membership cannot be granted without its acceptance record | `crates/buzz-db/src/relay_members.rs:122-158` | `claim_relay_membership` |
| 12.12 | Allowlist backfill runs only when the target community has **no** `relay_members` rows, so intentionally removed members are not resurrected | `crates/buzz-db/src/relay_members.rs:553-563` | `backfill_from_allowlist` |
| 12.13 | Archiving an identity is not a ban: it does not affect membership, relay access, or repo permissions; re-archiving is idempotent and does not mutate the existing row | `crates/buzz-db/src/archived_identities.rs:3-6`, `:44-79` | `archive` |

#### 13. API tokens

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 13.1 | Only SHA-256 hashes are stored; the hash is 32 bytes (DB CHECK) | `migrations/0001_initial_schema.sql:488`; callers pass the hash (`crates/buzz-db/src/api_token.rs:8-9`) | insert |
| 13.2 | 10 active tokens per `(community, owner)`, enforced by a subquery inside the INSERT (`… WHERE (SELECT COUNT(*) …) < 10`) — no count-then-insert race | `crates/buzz-db/src/api_token.rs:91-126` | `create_api_token_if_under_limit` |
| 13.3 | "Active" for the quota means `revoked_at IS NULL AND (expires_at IS NULL OR expires_at > NOW())` | `crates/buzz-db/src/api_token.rs:113-118` | same |
| 13.4 | Lookup is keyed on `(community_id, token_hash)`, not hash alone — a token minted in A can never authorize in B | `crates/buzz-db/src/lib.rs:2337-2340`; `crates/buzz-db/src/api_token.rs:155-158` and rationale `:129-142` | auth path |
| 13.5 | Revocation is scoped to `(community, id, owner)` and skips already-revoked rows (idempotent) | `crates/buzz-db/src/api_token.rs:272-301`, `:303-329` | `revoke_token`, `revoke_all_tokens` |
| 13.6 | Self-minted tokens are flagged (`created_by_self_mint = TRUE`) | `crates/buzz-db/src/api_token.rs:100` | `create_api_token_if_under_limit` |

#### 14. Workflows, runs, approvals, cron claims

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 14.1 | A NIP-33 workflow upsert may only update a row with the **same owner and same channel**; otherwise `AccessDenied` — a learned workflow UUID is not a cross-user overwrite primitive | `crates/buzz-db/src/workflow.rs:320-352` | `upsert_workflow` |
| 14.2 | Owner-scoped delete keeps the owner predicate inside the DELETE (`… AND owner_pubkey = $3`), so no check-then-delete race | `crates/buzz-db/src/workflow.rs:738-763` | `delete_workflow_for_owner` |
| 14.3 | List queries are bounded: `LIST_DEFAULT_LIMIT = 100`, clamped to `LIST_MAX_LIMIT = 1000`; run lists clamp to 1000 | `crates/buzz-db/src/workflow.rs:25-27`, `:395`, `:441`, `:466`, `:824` | list calls |
| 14.4 | The scheduler scan only returns `status='active' AND enabled AND definition->'trigger'->>'on' = 'schedule'` in non-archived communities, and carries each row's `community_id` | `crates/buzz-db/src/workflow.rs:457-478` | `list_all_enabled_workflows` |
| 14.5 | At-most-once cron fire: the first pod to insert `(community_id, workflow_id, scheduled_for)` wins; everyone else gets `None` and must not create a run. The claim binds **both** community and id because workflow UUIDs are not globally unique | `crates/buzz-db/src/workflow.rs:496-534`; PK `migrations/0001_initial_schema.sql:457` | `claim_scheduled_workflow_fire` |
| 14.6 | Interval anchoring reads `MAX(scheduled_for)` from the **claim** table, not `workflow_runs` | `crates/buzz-db/src/workflow.rs:528-560` | `latest_scheduled_workflow_fire` |
| 14.7 | A won claim whose run creation fails keeps `workflow_run_id` NULL on purpose — the instant stays claimed and must not fire twice | `crates/buzz-db/src/workflow.rs:556-561` | `attach_scheduled_workflow_run` |
| 14.8 | Claim retention prune must stay older than the largest supported interval or interval workflows lose their anchor (documented operational constraint) | `crates/buzz-db/src/workflow.rs:589-596` | `prune_scheduled_workflow_fires_before` |
| 14.9 | `create_approval` receives the **raw** token and hashes it with SHA-256 internally; the plaintext never reaches the database | `crates/buzz-db/src/workflow.rs:33-36`, `:915-946` | `create_approval` |
| 14.10 | Approval grant/deny includes `AND status = 'pending'` in the WHERE clause, so two concurrent decisions cannot both succeed (`false` ⇒ conflict / HTTP 409) | `crates/buzz-db/src/workflow.rs:1043-1051`, `:1074-1087` | `update_approval*` |
| 14.11 | Approval lookups bind `(community_id, token)` so an approval action for A/X can never act on B/X | `crates/buzz-db/src/workflow.rs:983-991` | `get_approval*`, `update_approval*` |
| 14.12 | `started_at` is stamped from the **bind parameter** (`$5 = 'running' AND started_at IS NULL`), fixing the earlier post-SET read; `completed_at` is stamped for `completed`/`failed`/`cancelled` | `crates/buzz-db/src/workflow.rs:843-880` | `update_workflow_run` |
| 14.13 | Status/enum strings are parsed back with strict `FromStr`; unknown values are `InvalidData`, never silently coerced | `crates/buzz-db/src/workflow.rs:61-71`, `:103-116`, `:148-160` | every row mapping |

#### 15. DMs

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 15.1 | DM identity is `SHA-256(sorted, deduped participant pubkeys)` with no separator (fixed-width inputs) — order-independent | `crates/buzz-db/src/dm.rs:48-60`; UNIQUE index `migrations/0001_initial_schema.sql:104` | `create_dm`, `open_dm` |
| 15.2 | 2 ≤ participants ≤ 9, each exactly 32 bytes | `crates/buzz-db/src/dm.rs:107-124`, `:365-370` | `create_dm`, `open_dm` |
| 15.3 | DMs are `channel_type='dm', visibility='private'`, participants added as `'member'` | `crates/buzz-db/src/dm.rs:160-190` | `create_dm` |
| 15.4 | `create_dm` is idempotent: the participant-hash probe runs inside the transaction | `crates/buzz-db/src/dm.rs:126-155` | `create_dm` |
| 15.5 | `open_dm` always adds the caller to the participant set | `crates/buzz-db/src/dm.rs:361-364` | `open_dm` |
| 15.6 | Hiding a DM is per-user (`channel_members.hidden_at`), never a delete; re-opening the same participant set unhides it | `crates/buzz-db/src/dm.rs:377-381`, `:390-424`, `:429-446` | `hide_dm`, `open_dm` |
| 15.7 | Hidden DMs are filtered out of channel listings (`c.channel_type != 'dm' OR cm.hidden_at IS NULL`) and of `list_dms_for_user` | `crates/buzz-db/src/channel.rs:822`; `crates/buzz-db/src/dm.rs:257`, `:283` | listings |
| 15.8 | `list_dms_for_user` caps `limit` at 200 | `crates/buzz-db/src/dm.rs:233` | `list_dms_for_user` |

#### 16. Moderation

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 16.1 | Report ingest is idempotent per `(community, report_event_id)` | `crates/buzz-db/src/moderation.rs:186-193`; UNIQUE `migrations/0006_moderation.sql:63` | `insert_report` |
| 16.2 | Exactly one target class per report row; `target_kind` is authoritative and the DB CHECK forbids mixed rows. An unrecognised `target_kind` on read is `InvalidData` | `migrations/0006_moderation.sql:42-46`; `crates/buzz-db/src/moderation.rs:580-590` | insert / read |
| 16.3 | Only an `open` report can be resolved/dismissed/escalated (`AND status='open'`) | `crates/buzz-db/src/moderation.rs:298-300` | `resolve_report` |
| 16.4 | Ban expiry is computed on read: `banned AND (ban_expires_at IS NULL OR ban_expires_at > now())`; NULL expiry with `banned` ⇒ permanent | `crates/buzz-db/src/moderation.rs:447-451`, `:476-479`, `:497-508` | enforcement reads |
| 16.5 | `muted_until` is only surfaced when in the future | `crates/buzz-db/src/moderation.rs:450` | `restriction_state` |
| 16.6 | `unban_member` only affects a currently banned row; `untimeout_member` only a currently active timeout | `crates/buzz-db/src/moderation.rs:359`, `:415` | those calls |
| 16.7 | A report may only be resolved by an action row **in its own community** (composite FK) | `migrations/0006_moderation.sql:128-130` | `resolve_report` |
| 16.8 | The 12-value action vocabulary is duplicated in Rust and asserted against the SQL CHECK by a test | `crates/buzz-db/src/moderation.rs:104-118`; assertion `crates/buzz-db/src/migration.rs:640-645` | build/test |
| 16.9 | The deployment-admin plane is the **only** moderation repository allowed to omit a `CommunityId`, and it clamps pages to 200 | `crates/buzz-db/src/admin_moderation.rs:1-8`, `:15-18` | admin reads |

#### 17. Push (NIP-PL)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 17.1 | Lease acceptance takes three locks in a fixed global order: address lock → author lock → per-community push-gate lock (activations only) | `crates/buzz-db/src/push.rs:216-243` | `accept_lease_event` |
| 17.2 | A signed lease event wins only on `source_created_at` DESC with lowest `source_event_id` as tiebreak, **and** strictly increasing `generation`; otherwise `StaleEvent` / `StaleGeneration` | `crates/buzz-db/src/push.rs:266-280`; upsert predicate `:475-483` | `accept_lease_event`, `replace_lease` |
| 17.3 | Reusing a `source_event_id` under a different `(author, installation)` is `SourceEventCollision` | `crates/buzz-db/src/push.rs:245-259`, `:398-402` | `accept_lease_event` |
| 17.4 | Expired active leases are deactivated before quota/endpoint checks so they cannot hold quota or endpoint uniqueness forever | `crates/buzz-db/src/push.rs:285-296` | `accept_lease_event` |
| 17.5 | Per-author active-lease quota (`max_active_leases`, caller-supplied) and per-`(author, app_profile, endpoint_hash)` uniqueness are checked in-transaction, backed by a partial unique index | `crates/buzz-db/src/push.rs:298-323`; index `migrations/0012_push_leases.sql:23-25` |`accept_lease_event` |
| 17.6 | Any `23xxx` integrity violation on a validated lease is mapped to a protocol outcome, not a 500 | `crates/buzz-db/src/push.rs:392-410` | `accept_lease_event` |
| 17.7 | `push_match_queue` enqueue happens in the event's own transaction, only for kinds `{7, 9, 1059, 40007, 46010}`, and **only** when the community has an active + endpoint-enabled + unexpired lease | `migrations/0023_push_match_gate.sql:22-42` | AFTER INSERT on `events` |
| 17.8 | The lost-wake race is closed by a per-community advisory lock: event inserts take `'buzz_push_gate:<community>'` **shared**, lease activations take it **exclusive** | `migrations/0023_…:34-35`; `crates/buzz-db/src/push.rs:15-33`, `:239-243`, `:464-468` | insert vs activation |
| 17.9 | On activation, recent gate-skipped events (`received_at > now() - 120s`, relay clock, same kind allowlist) are backfilled into the queue **inside** the activation transaction | `crates/buzz-db/src/push.rs:35-66`, `:373-375`, `:496-499` | activation |
| 17.10 | Wake enqueue copies `endpoint_hash` and `endpoint_grant` from the current lease — a caller cannot redirect a wake | `crates/buzz-db/src/push.rs:502-511`, `:650-701` | `enqueue_wakes` |
| 17.11 | Wake dedup key is `(community_id, endpoint_hash, event_id)`; the first request to claim an inserted key reports `Enqueued`, later same-key requests report `Duplicate` | `migrations/0012_push_leases.sql:47`; `crates/buzz-db/src/push.rs:703-763` | `enqueue_wakes` |
| 17.12 | Lease rows are locked `FOR UPDATE` in `(author, installation_id)` order so batches and single-row replacements cannot deadlock | `crates/buzz-db/src/push.rs:624-660` | `enqueue_wakes` |
| 17.13 | An enqueue whose generation no longer matches the current active lease returns `InactiveLease` | `crates/buzz-db/src/push.rs:667-675` | `enqueue_wakes` |
| 17.14 | Matcher claims are one-community-at-a-time, `SKIP LOCKED`, `attempts < MAX_MATCH_ATTEMPTS(8)`, and increment `attempts` | `crates/buzz-db/src/push.rs:70`, `:840-882` | `claim_due_match_batch` |
| 17.15 | A claimed job whose source event is absent or soft-deleted is deleted — a deliberate privacy-preserving terminal outcome | `crates/buzz-db/src/push.rs:899-916` | `claim_due_match_batch` |
| 17.16 | Exhausted jobs are reaped by a **separate** periodic sweep, never on the claim path | `crates/buzz-db/src/push.rs:925-941` | `reap_exhausted_matches` |
| 17.17 | Every completion/retry/fail requires the fencing `claim_id` and `state='sending'`/`'matching'`; stale workers are no-ops | `crates/buzz-db/src/push.rs:970-1017`, `:1132-1195` | those calls |
| 17.18 | The load-bearing authorization gate is `revalidate_wake_for_send`, re-joining community + generation + endpoint_hash + live event immediately before the transport call; claim-time checks are only optimizations | `crates/buzz-db/src/push.rs:1080-1130` | send path |
| 17.19 | Endpoint disable only applies to the exact current generation (`AND generation=$4 AND active AND endpoint_enabled`), making stale gateway responses no-ops | `crates/buzz-db/src/push.rs:1193-1221` | `disable_endpoint_generation` |
| 17.20 | Outbox pruning skips rows whose event still has a matcher job | `crates/buzz-db/src/push.rs:1223-1243` | `prune_wake_outbox` |
| 17.21 | Structural coupling: an `active` lease must have all five effective columns non-NULL; an inactive lease must have them all NULL | `migrations/0012_push_leases.sql:20-21` | any write |

#### 18. Reminders (NIP-ER, kind 30300)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 18.1 | Due set = latest head per `(community, pubkey, d_tag)` with `not_before <= now`, not deleted, not delivered, in a non-archived community | `crates/buzz-db/src/event.rs:1334-1380` | `query_due_reminders` |
| 18.2 | Exactly one pod wins a reminder: `UPDATE … SET delivered_at=$1 WHERE … delivered_at IS NULL` | `crates/buzz-db/src/event.rs:1411-1441`; race test `:2065-2107` | `claim_due_reminder*` |
| 18.3 | Release is compare-and-clear on the pod's own stamp (`AND delivered_at = $4`), so one pod cannot roll back another's later claim | `crates/buzz-db/src/event.rs:1443-1472` | `release_due_reminder` |
| 18.4 | Claim and release are community-scoped, because the same event id may exist in two communities | `crates/buzz-db/src/event.rs:1400-1409`, `:1394-1400`; test `:2183-2263` | claim/release |

#### 19. Git repo names

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 19.1 | Names are unique **within a community**, enforced atomically by `INSERT … ON CONFLICT (community_id, repo_id) DO NOTHING RETURNING owner_pubkey` (replaces the old filesystem `create_dir` race guard) | `crates/buzz-db/src/git_repo.rs:96-113`; PK `migrations/0002_git_repo_names.sql:25` | `reserve_repo_name` |
| 19.2 | Same-owner re-announce is `AlreadyOwned` (no quota re-check); different owner is `TakenByOther` | `crates/buzz-db/src/git_repo.rs:115-140` | `reserve_repo_name` |
| 19.3 | A conflicting row that vanished between INSERT and the classification read is treated as `TakenByOther` — a name is never granted without an atomic claim | `crates/buzz-db/src/git_repo.rs:134-139` | race |
| 19.4 | Per-pubkey quota is a caller responsibility using `count_repos_for_owner`; this module returns no quota error | `crates/buzz-db/src/git_repo.rs:88-92`, `:136-158` | announce handler |
| 19.5 | Rollback release is owner-scoped, so it can never delete another owner's concurrent reservation | `crates/buzz-db/src/git_repo.rs:160-180` | seed failure |

#### 20. Users / agents

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 20.1 | Agent ownership is first-mint-wins, set by a conditional `UPDATE … WHERE agent_owner_pubkey IS NULL`; `false` means an owner already exists, an error means the agent row is missing | `crates/buzz-db/src/user.rs:283-325` | `set_agent_owner` |
| 20.2 | `agent_owner_pubkey` must reference a user in the **same** community (composite self-FK, `ON DELETE SET NULL`) | `migrations/0001_initial_schema.sql:173-174` | insert/update |
| 20.3 | Because `agent_owner_pubkey` is immutable after mint, `remove_member` may read it outside its transaction | documented at `crates/buzz-db/src/channel.rs:451-456`, `:483-485` | `remove_member` |
| 20.4 | `channel_add_policy` values are validated in Rust before the enum cast | `crates/buzz-db/src/user.rs:374-380` | `set_channel_add_policy` |
| 20.5 | Empty profile strings are stored as NULL — required for kind:0 absolute-state semantics and to avoid `nip05_handle` uniqueness collisions on `''` | `crates/buzz-db/src/user.rs:95-101`, `:135-140` | `update_user_profile` |
| 20.6 | User search escapes LIKE metacharacters and clamps `limit` to `[1, 500]` | `crates/buzz-db/src/user.rs:207-222`, `:233` | `search_users` |

#### 21. Retention / TTL / pruning summary

| Object | Policy | Where |
|--------|--------|-------|
| Superseded NIP-RS payloads | hard delete + watermark retained | `crates/buzz-db/src/lib.rs:3739-3796`, `migrations/0007`, `0009`, `0011` |
| Superseded mesh-status payloads | hard delete | `crates/buzz-db/src/lib.rs:3688-3695`, `migrations/0019` |
| Other superseded replaceables | soft delete (`deleted_at`) — history retained | `crates/buzz-db/src/lib.rs:3752-3757` |
| Events, channels, channel members, reactions | soft delete only; no purge path in this crate | `deleted_at` / `removed_at` / `removed_at` columns |
| Ephemeral channels | archived when `ttl_deadline < NOW()` | `crates/buzz-db/src/channel.rs:1387-1417` |
| `scheduled_workflow_fires` | operator-driven `DELETE … claimed_at < cutoff` | `crates/buzz-db/src/workflow.rs:597-611` |
| `push_wake_outbox` | `DELETE` terminal/expired rows older than a cutoff, unless a matcher job still references the event | `crates/buzz-db/src/push.rs:1223-1243` |
| `push_match_queue` | `DELETE` when `attempts >= 8`, or on load-miss | `crates/buzz-db/src/push.rs:899-941` |
| `events` / `delivery_log` partitions | created ahead; **no drop/detach path exists in this crate** | `crates/buzz-db/src/partition.rs:15-73` |
| `audit_log`, `delivery_log`, `subscriptions`, `rate_limit_violations` | no retention code in buzz-db | — |

#### 22. Partition management rules

| # | Rule | Enforced at |
|---|------|-------------|
| 22.1 | Only `events` and `delivery_log` may be partition-managed (`PARTITIONED_TABLES` allowlist), re-checked inside `ensure_partition` | `crates/buzz-db/src/partition.rs:12`, `:84-88` |
| 22.2 | Partition suffix must be digits/underscores only | `crates/buzz-db/src/partition.rs:58-60`, `:89-93` |
| 22.3 | Boundary dates must be exactly `YYYY-MM-DD` (length 10, digits, dashes at 4 and 7) | `crates/buzz-db/src/partition.rs:63-73`, `:94-104` |
| 22.4 | Existence is checked against `pg_catalog` with `relispartition = true` and `current_schema()` before any DDL | `crates/buzz-db/src/partition.rs:108-127` |
| 22.5 | A `42P17` "would overlap partition" error is treated as "already ensured" (the right-edge `*_p_future` catch-all covers the month) rather than failing startup | `crates/buzz-db/src/partition.rs:136-148` |
| 22.6 | Month arithmetic rolls the year over and rejects impossible dates with `InvalidData` | `crates/buzz-db/src/partition.rs:18-49` |

#### 23. Tenant scoping (community_id) as a rule

| # | Rule | Enforced at |
|---|------|-------------|
| 23.1 | Every tenant-scoped table carries `community_id NOT NULL`; asserted over the concatenated SQL of **all** migrations | `crates/buzz-db/src/migration.rs:635-651` (`all_non_operator_global_tables_have_not_null_community_id`) |
| 23.2 | Every PK / UNIQUE / FK / unique index on a tenant-scoped table must lead with `community_id`; the sole exception is `delivery_log`'s partition PK `(delivered_at, id)` | `crates/buzz-db/src/migration.rs:495-512`, `:653-670` |
| 23.3 | "Tenant-scoped" is defined negatively: any table not listed in `_operator_global_tables` | `crates/buzz-db/src/migration.rs:330-360` |
| 23.4 | Migrations may not re-tenant `channels`: no `UPDATE … SET community_id`, no drop/alter/rename of the column, no drop/disable of the guard trigger, no `DROP TABLE channels` | `crates/buzz-db/src/migration.rs:527-556`, `:672-688` |
| 23.5 | Cross-community reads are allowed only in the explicitly documented operator/scheduler paths (see `buzz-db-security.md` §tenant isolation) | `crates/buzz-db/src/admin_moderation.rs:1-8`, `crates/buzz-db/src/usage.rs:1-14`, `crates/buzz-db/src/workflow.rs:449-456` |


## Module: buzz-auth (`crates/buzz-auth`)

### Aspect: Business Rules

---

### 1. NIP-42 verification sequence

Entry point `verify_nip42_event(event, expected_challenge, relay_url)`
(`crates/buzz-auth/src/nip42.rs:47-86`). Guard-clause style: each step returns
early on failure; there is no accumulation of errors. Order matters — the first
failing check determines the error variant.

| Step | Rule enforced | Failure | file:line |
|------|---------------|---------|-----------|
| 1 | `event.kind` must equal `Kind::Authentication` (kind 22242) | `AuthError::InvalidSignature` | `crates/buzz-auth/src/nip42.rs:52-54` |
| 2 | Event id hash + Schnorr signature must verify via `buzz_core::verify_event` | `AuthError::InvalidSignature` (original error discarded) | `crates/buzz-auth/src/nip42.rs:56` |
| 3 | A `challenge` tag must be present with content | `AuthError::ChallengeMismatch` | `crates/buzz-auth/src/nip42.rs:58-62` |
| 4 | `challenge` tag content must equal `expected_challenge` byte-for-byte (`!=` on `&str`) | `AuthError::ChallengeMismatch` | `crates/buzz-auth/src/nip42.rs:64-66` |
| 5 | A `relay` tag must be present with content | `AuthError::RelayUrlMismatch` | `crates/buzz-auth/src/nip42.rs:68-72` |
| 6 | `normalize_relay_url(relay_tag)` must equal `normalize_relay_url(relay_url)` | `AuthError::RelayUrlMismatch` | `crates/buzz-auth/src/nip42.rs:74-76` |
| 7 | `abs_diff(now, event.created_at)` must be `<= 60` | `AuthError::EventExpired` | `crates/buzz-auth/src/nip42.rs:78-83` |

Trigger: called from `AuthService::verify_auth_event`, which wraps it in
`tokio::task::spawn_blocking` because Schnorr verification is CPU-bound
(`crates/buzz-auth/src/lib.rs:125-132`). A `spawn_blocking` panic maps to
`AuthError::Internal("spawn_blocking panicked")`
(`crates/buzz-auth/src/lib.rs:132`). In the relay, the caller is
`handlers::auth::handle_auth` (`crates/buzz-relay/src/handlers/auth.rs:87-89`),
which supplies the connection's pending challenge
(`crates/buzz-relay/src/handlers/auth.rs:48`) and a tenant-derived relay URL
(`crates/buzz-relay/src/handlers/auth.rs:80-81`).

Notable ordering consequence: signature verification (step 2) happens **before**
the challenge and relay-URL checks, so an unsigned/forged event can never reach
the tag comparisons. Conversely a validly-signed event for a different relay
still consumes a Schnorr verification.

---

### 2. NIP-42 timestamp tolerance

| Rule | Value | file:line |
|------|-------|-----------|
| `TIMESTAMP_TOLERANCE_SECS` | `60` (private `const u64`) | `crates/buzz-auth/src/nip42.rs:35` |
| Comparison | `delta = now.abs_diff(event_ts); if delta > TIMESTAMP_TOLERANCE_SECS { EventExpired }` | `crates/buzz-auth/src/nip42.rs:78-83` |

`abs_diff` makes the window symmetric: future-dated events are rejected the same
way as stale ones, so the effective window is ±60s inclusive (delta of exactly 60
passes; 61 fails). Test pins rejection at 120s in the past
(`crates/buzz-auth/src/nip42.rs:143-157`). `AuthError::EventExpired`'s Display
text also states "±60 seconds" (`crates/buzz-auth/src/error.rs:22-24`).

---

### 3. Relay-URL comparison rules for AUTH

`normalize_relay_url` (`crates/buzz-auth/src/nip42.rs:19-33`), applied to **both**
sides before comparison (`crates/buzz-auth/src/nip42.rs:74`):

| Rule | Behaviour | file:line |
|------|-----------|-----------|
| Unparseable input | returned verbatim (`raw.to_string()`) — falls back to exact string comparison | `crates/buzz-auth/src/nip42.rs:20-23` |
| Loopback aliasing | host `localhost` **or** `::1` is rewritten to `127.0.0.1`; the three are treated as equivalent | `crates/buzz-auth/src/nip42.rs:25-29` |
| Trailing slash | path is `trim_end_matches('/')` then re-set | `crates/buzz-auth/src/nip42.rs:30-31` |
| Scheme/host case, default ports, percent-encoding | delegated to `url::Url` parsing + `to_string()` | `crates/buzz-auth/src/nip42.rs:20`, `:32` |

Scheme is **not** normalised across `ws`/`wss` — `ws://h` and `wss://h` produce
different strings and would mismatch. Port is part of the comparison (the
localhost test uses matching `:3030` on both sides,
`crates/buzz-auth/src/nip42.rs:171-175`).

This is deliberately **different** from NIP-98's URL rule (rule 5 below):
NIP-42 collapses loopback aliases, NIP-98 does not.

---

### 4. NIP-98 verification sequence

Entry point `verify_nip98_event(event_json, expected_url, expected_method, body)`
(`crates/buzz-auth/src/nip98.rs:55-131`). Steps are numbered in the code itself.

| Step | Rule enforced | Failure message (all wrapped in `AuthError::Nip98Invalid`) | file:line |
|------|---------------|-----------------------------------------------------------|-----------|
| 1 | `event_json` must deserialise into `nostr::Event` | `event JSON parse error: {e}` | `crates/buzz-auth/src/nip98.rs:62-63` |
| 2 | `event.kind` must equal `Kind::HttpAuth` (27235) | `expected kind 27235, got {n}` | `crates/buzz-auth/src/nip98.rs:66-71` |
| 3 | Event id hash + Schnorr signature via `buzz_core::verify_event` | `invalid Schnorr signature` | `crates/buzz-auth/src/nip98.rs:74-75` |
| 4 | `abs_diff(now, created_at) <= 60` | `event timestamp outside ±60s window (delta: {d}s)` | `crates/buzz-auth/src/nip98.rs:78-85` |
| 5a | A single-letter lowercase `u` tag must be present with content | `missing \`u\` tag` | `crates/buzz-auth/src/nip98.rs:89-95` |
| 5b | `normalize_url(u_tag)` must equal `normalize_url(expected_url)` | `URL mismatch: event has \`..\`, expected \`..\`` | `crates/buzz-auth/src/nip98.rs:97-101` |
| 6a | A `method` tag must be present with content | `missing \`method\` tag` | `crates/buzz-auth/src/nip98.rs:104-108` |
| 6b | `method_tag.eq_ignore_ascii_case(expected_method)` | `method mismatch: ...` | `crates/buzz-auth/src/nip98.rs:110-114` |
| 7 | **Only if** a `payload` tag exists **and** `body` is `Some`: `hex(SHA-256(body))` must equal the tag value | `payload tag SHA-256 mismatch: request body does not match signed hash` | `crates/buzz-auth/src/nip98.rs:117-127` |
| 8 | Return `event.pubkey` | — | `crates/buzz-auth/src/nip98.rs:130` |

Step 7 is conditional in both directions: a missing `payload` tag with a present
body passes (test: `crates/buzz-auth/src/nip98.rs:269-276`), and a present
`payload` tag with `body == None` is silently ignored. The tuple destructure
`(Some(payload_hex), Some(body_bytes))` is what makes both cases skip
(`crates/buzz-auth/src/nip98.rs:119`). Enforcing the tag's presence is the
caller's job — `buzz-relay` does that with a `require_payload` flag before
calling in (`crates/buzz-relay/src/api/bridge.rs:99-112`).

Trigger: HTTP requests carrying `Authorization: Nostr <base64(event JSON)>`; the
caller base64-decodes and passes the JSON string
(`crates/buzz-auth/src/nip98.rs:38`, caller at
`crates/buzz-relay/src/api/bridge.rs:81-112`). This function is **synchronous**
and is not wrapped in `spawn_blocking` by this crate, unlike the NIP-42 path.

---

### 5. NIP-98 URL comparison rules

`normalize_url` (`crates/buzz-auth/src/nip98.rs:145-153`), applied to both sides:

| Rule | Behaviour | file:line |
|------|-----------|-----------|
| Unparseable input | `raw.to_lowercase()` (note: differs from NIP-42's verbatim fallback) | `crates/buzz-auth/src/nip98.rs:147-148` |
| Scheme/host case | lowercased by `url::Url` | `crates/buzz-auth/src/nip98.rs:135` |
| Trailing slash | stripped from path | `crates/buzz-auth/src/nip98.rs:150-151` |
| Loopback aliasing | **explicitly not performed** — `localhost`, `::1`, `127.0.0.1` are three distinct hosts | `crates/buzz-auth/src/nip98.rs:138-144` |

Stated rationale: under multi-tenant the `u`-tag host is the row-zero community
binding, so collapsing loopback aliases would be a host-binding side door
(`crates/buzz-auth/src/nip98.rs:138-144`). Pinned by a three-way test asserting
both directions reject and same-host still passes
(`crates/buzz-auth/src/nip98.rs:288-316`).

Query string and fragment are part of `Url::to_string()` and therefore part of
the comparison — no explicit stripping.

---

### 6. Scope-granting rules — which auth path grants what

| Auth path | Scopes granted | file:line |
|-----------|----------------|-----------|
| NIP-42 (`AuthService::verify_auth_event`) | `Scope::all_known()` — all **16** known variants, unconditionally, for every successfully-authenticated pubkey | `crates/buzz-auth/src/lib.rs:136-142`, list at `crates/buzz-auth/src/scope.rs:68-87` |
| NIP-98 (`verify_nip98_event`) | **none** — returns only a `PublicKey`; no `AuthContext`, no scope set | `crates/buzz-auth/src/nip98.rs:60`, `:130` |

Other `AuthContext` fields on the NIP-42 path are fixed:
`channel_ids: None` (unrestricted), `auth_method: AuthMethod::Nip42`,
`agent_owner_pubkey: None` with the comment "Set later by relay membership gate
if NIP-OA" (`crates/buzz-auth/src/lib.rs:139-141`).

Stated justification: "In pure Nostr mode, all authenticated connections get full
scopes. Per-channel access is enforced by the relay's membership checks (NIP-29)"
(`crates/buzz-auth/src/lib.rs:134-135`; same claim in the module doc at
`crates/buzz-auth/src/scope.rs:3-5`).

`Scope::all_non_admin()` (14 variants, excludes `AdminChannels` and `AdminUsers`)
exists as an alternative grant set (`crates/buzz-auth/src/scope.rs:94-111`) but is
never called anywhere in the repo — no auth path in this crate or outside it uses
it. Its doc comment describes a dev-mode `X-Pubkey` / `require_auth_token=false`
scenario (`crates/buzz-auth/src/scope.rs:91-93`); the same doc text is
copy-pasted onto `all_known()` (`crates/buzz-auth/src/scope.rs:66-67`) even though
`all_known()` is what the production NIP-42 path uses.

Repo-wide, the only external caller of either is
`crates/buzz-relay/src/api/bridge.rs:829`, which builds its own auth struct with
`buzz_auth::Scope::all_known()`.

---

### 7. Scope-checking logic

| Rule | Implementation | file:line |
|------|----------------|-----------|
| `require_scope` | `scopes.contains(&required)` → `Ok(())`, else `AuthError::InsufficientScope { required, have }` where `have` is the caller's full scope list stringified | `crates/buzz-auth/src/access.rs:60-69` |
| `AuthContext::has_scope` | `self.scopes.contains(scope)` — bool, no error | `crates/buzz-auth/src/lib.rs:84-86` |
| Matching semantics | exact `PartialEq` on the enum. No hierarchy, no wildcards, no implication (`admin:channels` does **not** imply `channels:write`; `messages:write` does not imply `messages:read`) | derive at `crates/buzz-auth/src/scope.rs:15`; enumerated grant list at `crates/buzz-auth/src/scope.rs:69-86` |
| `Unknown(s)` scopes | compare by inner string only; never satisfy a known-variant requirement | `crates/buzz-auth/src/scope.rs:60`, `:164` |

Because NIP-42 grants all 16 known scopes, `require_scope` can only fail for a
caller whose scope list was built some other way (e.g. an empty list, as in the
test at `crates/buzz-auth/src/access.rs:189`).

---

### 8. Combined access rules (scope + membership)

| Function | Sequence | file:line |
|----------|----------|-----------|
| `check_read_access` | 1) `require_scope(scopes, Scope::MessagesRead)?` 2) `checker.can_access(ctx, pubkey, channel_id)` → `Ok(())` or `AuthError::ChannelAccessDenied` | `crates/buzz-auth/src/access.rs:72-85` |
| `check_write_access` | identical but requires `Scope::MessagesWrite` | `crates/buzz-auth/src/access.rs:88-101` |

Ordering rule: the scope check runs **before** the membership lookup, so an
under-scoped caller never triggers a DB round-trip
(`crates/buzz-auth/src/access.rs:79-80`, `:95-96`). A `Err` from the checker
propagates as-is via `?` (fail-closed, no swallow).

`can_access` default implementation: fetch the whole accessible set, then
`contains` (`crates/buzz-auth/src/access.rs:52-55`).

Tenant rule for implementors: every query MUST be scoped by `ctx.community()`
because the `channels` PK is `(community_id, id)` and a bare `WHERE id = $1`
would be a cross-community existence oracle
(`crates/buzz-auth/src/access.rs:22-30`). Pinned by
`access_does_not_cross_communities` (`crates/buzz-auth/src/access.rs:225-251`).

Neither `check_read_access` nor `check_write_access` is called anywhere outside
this crate's own tests.

---

### 9. Rate-limit contract rules (interface-level, defined here)

| Rule | Statement | file:line |
|------|-----------|-----------|
| Pubkey-keyed quotas are per-community | key is `buzz:{community}:ratelimit:{pubkey_hex}:{suffix}`; same pubkey in two communities = two independent quotas | `crates/buzz-auth/src/rate_limit.rs:201-208`, doc `:151-156` |
| IP-keyed quotas are operator-global | key `buzz:ratelimit:ip:{ip}:conn`, deliberately no `TenantContext` (runs before host→community resolution) | `crates/buzz-auth/src/rate_limit.rs:213-215`, doc `:158-163` |
| Key stability | keys must be all-lowercase ASCII, else the same principal would get two counters (effective 2× quota) — pinned by test | `crates/buzz-auth/src/rate_limit.rs:290-306` |
| Fixed-window burst | documented as allowing up to 2× at window boundaries | `crates/buzz-auth/src/rate_limit.rs:6-7`, `:165-167` |
| Allow/deny boundary | decided by the implementor, not here. `RedisRateLimiter` uses `count <= limit` → allowed | `crates/buzz-pubsub/src/rate_limiter.rs:74-78` |

---

### 10. NIP-98 replay rules (contract defined here, enforced elsewhere)

| Rule | Statement | file:line |
|------|-----------|-----------|
| Verify-then-mark ordering | Verify the event first, then claim the seen-set slot. Marking before verifying would let an attacker who knows a future event id DoS the legitimate event | `crates/buzz-auth/src/nip98_replay.rs:19-27` |
| Return semantics | `Ok(true)` = newly inserted, proceed. `Ok(false)` = already seen, caller MUST reject as replay | `crates/buzz-auth/src/nip98_replay.rs:78-81` |
| Fail-closed on error | On `Err` (e.g. Redis unreachable) callers MUST reject, not admit | `crates/buzz-auth/src/nip98_replay.rs:83-86` |
| Atomicity | implementations MUST use atomic set-if-absent; read-then-write forfeits the freshness proof | `crates/buzz-auth/src/nip98_replay.rs:88-90` |
| TTL floor | `ttl_secs` ≥ `DEFAULT_REPLAY_TTL_SECS` (120); implementations MAY clamp up, MUST NOT honour smaller | `crates/buzz-auth/src/nip98_replay.rs:92-94`, const `:46` |
| TTL ceiling | clamp down to `MAX_REPLAY_TTL_SECS` (3600); larger values risk Redis `EX` (i64) parse failure | `crates/buzz-auth/src/nip98_replay.rs:96-100`, const `:59` |
| Why 120s | 2× the ±60s verifier tolerance — the full span over which a duplicate id is physically plausible | `crates/buzz-auth/src/nip98_replay.rs:29-31`, `:42-45` |
| No in-process caching | with multiple relay pods an in-process cache (moka, DashMap) does not carry the freshness proof across pods | `crates/buzz-auth/src/nip98_replay.rs:6-10` |

Three const-drift tripwire tests pin these bounds: TTL ≥ 120
(`crates/buzz-auth/src/nip98_replay.rs:210-218`), floor < ceiling
(`:220-230`), ceiling fits `i64::MAX` (`:232-237`).

---

### 11. Dev-only username→key derivation rule

| Rule | Statement | file:line |
|------|-----------|-----------|
| Derivation | secret key material = `SHA-256("buzz-test-key:{username}")`, then `nostr::SecretKey::from_slice` → `Keys::new(...).public_key()` | `crates/buzz-auth/src/lib.rs:162-166` |
| Gate | `#[cfg(any(test, feature = "dev"))]` | `crates/buzz-auth/src/lib.rs:159` |
| Stated invariant | "must never be compiled into a production release build"; keys are predictable from the username alone | `crates/buzz-auth/src/lib.rs:152-158` |
| Failure mode | `from_slice` error → `AuthError::Internal("key derivation failed: {e}")` | `crates/buzz-auth/src/lib.rs:164-165` |

Stated purpose: match the desktop's `set_test_identity` derivation so the relay
can resolve usernames to pubkeys in dev mode
(`crates/buzz-auth/src/lib.rs:148-151`). No caller exists anywhere in the repo.

---

### 12. Security invariants asserted by the crate docs

| Invariant | file:line | Verified in this crate? |
|-----------|-----------|-------------------------|
| "AUTH events (kind:22242) are NEVER stored or logged" | `crates/buzz-auth/src/lib.rs:14`, `crates/buzz-auth/src/nip42.rs:7` | Negatively — this crate has no logging or persistence on the NIP-42 path. Nothing enforces it for callers. |
| "All paths produce an `AuthContext` bound to the connection" | `crates/buzz-auth/src/lib.rs:15` | **Not accurate for NIP-98** — `verify_nip98_event` returns a `PublicKey` (`crates/buzz-auth/src/nip98.rs:60`). |
| "No JWT validation, no token management, no IdP runtime dependency" | `crates/buzz-auth/src/lib.rs:16`, `:98` | Accurate — no such types or deps exist in the crate. |


## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Business Rules

Rules extracted from all 10 `.rs` files. Each row is an enforced behaviour with its
enforcement site; "Tested" cites the asserting test.

---

### 1. Tenant scoping rules

| # | Rule | Enforcement | Tested |
|---|---|---|---|
| BR-PS-01 | Every event topic key embeds the server-resolved community id; the same channel UUID in two communities is two distinct Redis channels | `topic.rs:45-50` | `topic.rs:138-148` |
| BR-PS-02 | The community id comes only from `TenantContext::community()`, never from a caller argument | `EventTopicKey::from_context` `topic.rs:35-40` | — |
| BR-PS-03 | Presence keys are community-scoped by the same rule | `presence.rs:19-25` | `presence.rs:124-134` |
| BR-PS-04 | Cache-invalidation channels are community-scoped | `cache_invalidation.rs:30-35` | `cache_invalidation.rs:186-199` |
| BR-PS-05 | Conn-control channels are community-scoped | `conn_control.rs:33-35` | `conn_control.rs:172-181` |
| BR-PS-06 | NIP-98 replay seen-sets are per-community: the same event id is a *first* claim in a second community | `nip98_replay.rs:81` (scope-keyed) | `nip98_replay.rs:144-162` |
| BR-PS-07 | IP-keyed rate limits are deliberately **not** community-scoped (operator-global) | `rate_limiter.rs:118` → `buzz-auth/src/rate_limit.rs:213`; rationale `buzz-auth/src/rate_limit.rs:158` | `buzz-auth/src/rate_limit.rs:309-313` |
| BR-PS-08 | Releasing a topic in community A must not tear down the identical channel id in community B | refcount keyed by `EventTopicKey` not channel id — `lib.rs:193`, `:216`, `subscriber.rs:21` | `lib.rs:510-590` (explicitly written to catch the channel-id-only bug, comment `lib.rs:556-557`) |

### 2. Topic-key parsing rules (`topic.rs:53-99`)

| # | Rule | Line |
|---|---|---|
| BR-PS-09 | Prefix must be exactly `buzz` | `:58-63` |
| BR-PS-10 | Segment 2 must parse as a UUID | `:68-71` |
| BR-PS-11 | `global` topics must have exactly 3 segments — a 4th is rejected | `:78-82` |
| BR-PS-12 | `channel` topics must have exactly 4 segments; the 4th must parse as a UUID; a 5th is rejected | `:84-94` |
| BR-PS-13 | Any other scope word (including `presence`) is rejected | `:95` |

All 13 negative cases asserted at `topic.rs:179-195`, including
`buzz:{uuid}:presence:abc` (`:187`).

### 3. Subscription lifecycle rules

| # | Rule | Enforcement |
|---|---|---|
| BR-PS-14 | The local desired-refcount map is the single source of truth for what should be subscribed; the Redis connection's `active_topics` set is derived | doc `lib.rs:108`, `subscriber.rs:36-38`; rebuild `subscriber.rs:90-101` |
| BR-PS-15 | Only the **first** retain of a topic emits `SUBSCRIBE`; later retains just increment | `lib.rs:194-201` (`was_zero` gate) |
| BR-PS-16 | Only the **last** release schedules an unsubscribe; the entry is removed from the map at zero | `lib.rs:217-231` |
| BR-PS-17 | Unsubscribe is debounced by `unsubscribe_debounce` (default 500 ms) and re-checks the refcount at execution time, so a re-retain during the window cancels it | schedule `lib.rs:233-244`; re-check `subscriber.rs:123-130` |
| BR-PS-18 | Releasing a topic that was never retained is a logged no-op, not an underflow | `lib.rs:219-222` (`warn` + early return) — prevents `usize` underflow panic |
| BR-PS-19 | On reconnect, subscriptions are re-established from the refcount snapshot **before** any message is processed | `subscriber.rs:90-101` precedes the `loop`/`select!` at `:109` |
| BR-PS-20 | Duplicate `SUBSCRIBE` for an already-active topic is suppressed | `active_topics.insert(...)` guard `subscriber.rs:118-120` |
| BR-PS-21 | The fan-out loop can only be started once; a second `run_subscriber` logs an error and returns | `subscription_rx` take-once `lib.rs:149-152` |

### 4. Event fan-out rules

| # | Rule | Enforcement |
|---|---|---|
| BR-PS-22 | Published payload is the complete signed Nostr event JSON, not a delta | `publisher.rs:31` (`event.as_json()`) |
| BR-PS-23 | A message whose channel name fails to parse is dropped with a warning, never fanned out | `subscriber.rs:141-148` |
| BR-PS-24 | A payload that fails `Event::from_json` is dropped with a warning; the loop continues | `subscriber.rs:150-157` |
| BR-PS-25 | A payload-retrieval failure is dropped with a warning; the loop continues | `subscriber.rs:132-139` |
| BR-PS-26 | Zero local receivers is not an error — the message is dropped at `trace` level | `subscriber.rs:165-167` |
| BR-PS-27 | The community id delivered to local receivers is taken from the **parsed Redis channel**, not from the event body | `subscriber.rs:159-163` |
| BR-PS-28 | The topic key is a routing label, **not** an authorization boundary; per-recipient access is re-checked by the relay's `filter_fanout_by_access` on both the in-process and Redis paths | doc `topic.rs:3-6`, `lib.rs:305-320`; relay side `buzz-relay/src/handlers/event.rs:135`, `main.rs:696` |
| BR-PS-29 | Author-private reminders (kind 30300) are deliberately *not* isolated by Redis routing; author-only delivery is enforced downstream | `lib.rs:307-319` |

### 5. Presence rules

| # | Rule | Value / enforcement |
|---|---|---|
| BR-PS-30 | Presence TTL is 90 s — 3× the 30 s heartbeat so one missed beat doesn't flap | `presence.rs:16`, rationale `presence.rs:4-6`, `:15` |
| BR-PS-31 | Presence is written with `SET key status EX 90` (TTL refreshed on every heartbeat) | `presence.rs:36-43` |
| BR-PS-32 | Clean disconnect deletes the key immediately rather than waiting for TTL | `presence.rs:52-56`; callers `buzz-relay/src/connection.rs:280`, `handlers/event.rs:793` |
| BR-PS-33 | Bulk lookup short-circuits on an empty input list without touching Redis | `presence.rs:79-81` |
| BR-PS-34 | Bulk lookup returns only keys that exist — absent pubkeys are omitted rather than mapped to a default | `filter_map` `presence.rs:89-93` | 
| BR-PS-35 | Status is an arbitrary caller-supplied string; the crate defines no enum and validates nothing | `presence.rs:28-33` (`status: &str` written verbatim) |

BR-PS-34 tested at `presence.rs:159-184` (`pk3` asserted absent, `:180`).
BR-PS-30 tested indirectly by TTL-range assertions at `presence.rs:186-207` and
`lib.rs:490-499`.

### 6. Rate-limit rules (`rate_limiter.rs`)

| # | Rule | Enforcement |
|---|---|---|
| BR-PS-36 | Counter increment and TTL creation are atomic via a single Lua script — no crash window where a key exists without expiry | `RATE_LIMIT_SCRIPT` `rate_limiter.rs:24-31`; `EXPIRE` only when `count == 1` (`:26-28`) |
| BR-PS-37 | The window is **fixed**, accepting up to 2× burst at boundaries | documented `rate_limiter.rs:9-10`; also flagged upstream `buzz-relay/src/admission.rs:4-7` |
| BR-PS-38 | A key found with no TTL (`ttl < 0`) is repaired with a fresh `EXPIRE` and a warning; the reported reset becomes the full window | `rate_limiter.rs:57-70` |
| BR-PS-39 | Allow boundary is inclusive: `count <= limit` allows | `rate_limiter.rs:74-78` |
| BR-PS-40 | Redis failure surfaces as `AuthError::Internal`, and the relay treats that as **deny** (fail-closed) | mapping `rate_limiter.rs:44-46`, `:52`, `:66`; relay decision `buzz-relay/src/admission.rs:29-33` (`AdmissionError::Unavailable`) |
| BR-PS-41 | WS admission converts a per-second limit into a 5 s fixed window (`limit × 5`) to tolerate desktop startup bursts | `buzz-relay/src/admission.rs:8`, `:35-40`; asserted `ws_admission_budget(10) == (5, 50)` (`admission.rs:101`) |
| BR-PS-42 | Rate limiting applies only to `EVENT`, `REQ`, and `COUNT`; all other WS frames bypass it | `buzz-relay/src/connection.rs:598-601` |
| BR-PS-43 | Unauthenticated connections bypass rate limiting entirely (the limiter is pubkey-keyed) | `buzz-relay/src/connection.rs:602-609` (`_ => return true`) |
| BR-PS-44 | Agents get `agent_standard_messages_per_min`, humans `human_messages_per_min`, on a fixed 60 s window | `buzz-relay/src/connection.rs:628-646` |

### 7. NIP-98 replay rules (`nip98_replay.rs`)

| # | Rule | Enforcement |
|---|---|---|
| BR-PS-45 | Freshness is proven by `SET … NX EX` returning `OK` only on the first claim | `nip98_replay.rs:68-78`, `:86-88` |
| BR-PS-46 | An existing key (`nil` reply) yields `Ok(false)` so the caller rejects as replay | `nip98_replay.rs:87` |
| BR-PS-47 | Requested TTL is clamped into `[DEFAULT_REPLAY_TTL_SECS, MAX_REPLAY_TTL_SECS]` — sub-floor is lifted, above-ceiling is pushed down so a buggy caller cannot send a Redis-invalid `EX` or pin a slot indefinitely | `nip98_replay.rs:47`, rationale `:42-46` |
| BR-PS-48 | Any reply other than `OK`/`nil` is an internal error, logged at `error` — never treated as a successful claim | `nip98_replay.rs:88-98` |
| BR-PS-49 | Redis unavailability returns `Err`, and the contract requires callers to fail **closed** | `nip98_replay.rs:49-58`, `:74-77` (log text states "caller MUST fail closed") |

BR-PS-45/46 tested `nip98_replay.rs:127-142`; BR-PS-47 both directions tested at
`:164-177` (sub-floor) and `:179-200` (`u64::MAX` clamp).

### 8. Cross-pod control-message rules

| # | Rule | Enforcement |
|---|---|---|
| BR-PS-50 | Cache-invalidation messages are pure idempotent key drops — never "evict these subscriptions" payloads | stated invariant `cache_invalidation.rs:10-14`; enum carries only ids (`:58-80`) |
| BR-PS-51 | Each `CacheInvalidation` variant mirrors exactly one relay-local `invalidate_*` operation | `cache_invalidation.rs:53-80` |
| BR-PS-52 | Imperative, non-idempotent actions (disconnects) must live on a **separate** channel from cache drops, precisely to preserve BR-PS-50 | rationale `conn_control.rs:12-18`; distinct suffix `conn_control.rs:26` |
| BR-PS-53 | Both control publishes are fire-and-forget: the DB row (ban) or the next DB read (cache) is the durable backstop, so a dropped publish degrades gracefully | `lib.rs:266-271`, `:285-291`, `conn_control.rs:18-21` |
| BR-PS-54 | An unparseable/unknown control payload is skipped without breaking the subscriber loop | `cache_invalidation.rs:159-165`, `conn_control.rs:150-156`; regression test `conn_control.rs:209-217` |

### 9. Reconnect rules (all three subscribers)

| # | Rule | Enforcement |
|---|---|---|
| BR-PS-55 | Reconnect backoff starts at 1 s and doubles to a 30 s ceiling | `subscriber.rs:16-19`, `:46`, `:71`; `cache_invalidation.rs:91-94`, `:120-122`; `conn_control.rs:81-84`, `:110-112` |
| BR-PS-56 | A *clean* stream end resets backoff to 1 s; only errors let it grow | `subscriber.rs:57-63`, `cache_invalidation.rs:105-111`, `conn_control.rs:95-101` |
| BR-PS-57 | All three subscriber loops are infinite — they never return to the caller | `subscriber.rs:41-76`, `cache_invalidation.rs:100-127`, `conn_control.rs:90-117` |

**Rule count: 57.** Zero rules are enforced by type-state or the compiler; all are
runtime checks or conventions. No rule in this crate performs authorization — the
crate consistently defers that to `buzz-auth`/`buzz-relay` (BR-PS-28).


## Module: buzz-search (`crates/buzz-search`)

### Aspect: Business Rules

All rules live in `query::search` (`crates/buzz-search/src/query.rs:216-323`) and its
two private helpers. `SearchService` adds no rules (`lib.rs:51-53`).

#### R1 — Community scoping is unconditional and first

| What it enforces | Site | Trigger |
|---|---|---|
| `WHERE community_id = $1` is the first predicate of every executed statement; the bound value is `*query.community.as_uuid()` | `query.rs:240-241` | every non-empty-`q` call |

`SearchQuery.community` is a non-`Option` `CommunityId` (`query.rs:76`), and
`CommunityId` can only be minted from a server-trusted UUID
(`crates/buzz-core/src/tenant.rs:43-45`). There is no code path in the crate that
builds the statement without this predicate — the builder is seeded with the
`FROM` clause and the `WHERE community_id = ` fragment before any optional
predicate is considered (`query.rs:233-242`).

#### R2 — Empty / whitespace-only search text short-circuits with no SQL

| What it enforces | Site | Trigger |
|---|---|---|
| Returns `SearchResult { hits: [], page: clamp(page,1,PAGE_MAX) }` without touching the pool | `query.rs:217-222` | `normalized_search_text(&q)` returns `None`, i.e. `q.trim()` empty, or NUL-scrub + trim leaves it empty (`query.rs:180-190`) |

#### R3 — Search-text normalization: trim, NUL→space, hard char cap

| What it enforces | Site | Trigger |
|---|---|---|
| `q.trim()` applied first | `query.rs:180` | always |
| Each `'\0'` replaced with `' '` (Postgres text-search input cannot contain NUL) | `query.rs:186` | any NUL in text |
| Truncation to the first `SEARCH_TEXT_MAX_CHARS = 4096` **chars** (not bytes) | `query.rs:134`, `query.rs:185` | longer input |
| Re-trim after cleaning; `None` if now empty | `query.rs:188-193` | trailing-whitespace-only remainder |

Stated rationale: bound Postgres text-search parser CPU/memory per request
(`query.rs:131-134`).

#### R4 — NIP-50 term handling is parameter-bound in both modes (no escaping by string surgery)

| What it enforces | Site | Trigger |
|---|---|---|
| `FullText`: term passed to `websearch_to_tsquery('simple', $n)` as a bind, so Postgres' own websearch grammar (quoted phrases, `or`, `-`) applies and no operator can escape into SQL text | `query.rs:143-145` | `mode == FullText` |
| `Prefix`: term bound once (`query.rs:168`) then split/normalized entirely inside SQL — `regexp_split_to_table($n, '\s+') WITH ORDINALITY`, `to_tsvector('simple', token)`, `tsvector_to_array`, `unnest ... WITH ORDINALITY` | `query.rs:154-176` | `mode == Prefix` |
| `Prefix`: each lexeme wrapped in `quote_literal(...)` before concatenation, explicitly to stop tsquery-syntax injection from punctuation/operators | `query.rs:150-153`, `query.rs:157` | `mode == Prefix` |
| `Prefix`: only the **trailing whitespace-delimited token** gets `:*`; completed tokens stay exact (`CASE WHEN token_ord = max_token_ord`) | `query.rs:157-158` | `mode == Prefix` |
| `Prefix`: lexemes joined with `' & '` ordered by `(token_ord, lex_ord)`; `COALESCE(..., '')::tsquery` so an all-stopword/empty tokenization degrades to the empty tsquery rather than an error | `query.rs:155-162` | `mode == Prefix` |

The `simple` text-search configuration is used on both the query side
(`query.rs:143`, `query.rs:173`) and the storage side
(`migrations/0001_initial_schema.sql:224`), so query normalization matches the
generated column by construction — this symmetry is asserted in the code comment
at `query.rs:149-152`.

#### R5 — `ChannelScope` semantics (one rule per variant)

| Variant | Predicate emitted | Site | Effect |
|---|---|---|---|
| `Any` | none | `query.rs:249-251` | no channel constraint; every community row (subject to other predicates) is a candidate, including channel-scoped rows the caller may not be able to read |
| `ChannelLessOnly` | `AND channel_id IS NULL` | `query.rs:252-254` | only channel-less/global events |
| `Channels(ids)` | `AND channel_id = ANY($n)` | `query.rs:255-259` | only the listed channels; `ids = []` → `= ANY('{}')` → zero rows (documented at `query.rs:33-36`, asserted at `tests/fts_integration.rs:645-668`) |
| `ChannelsOrChannelLess(ids)` | `AND (channel_id = ANY($n) OR channel_id IS NULL)` | `query.rs:260-264` | listed channels plus channel-less; `ids = []` is equivalent to `ChannelLessOnly` (`query.rs:36-39`) |

Caller-side mapping from the legacy `(accessible_channels, include_global)` pair
lives outside this crate in `crates/buzz-relay/src/handlers/req.rs:484-501`; the
`empty && !include_global` case returns `None` there and the caller EOSEs instead
of calling search (`req.rs:517-524`).

#### R6 — Soft-deleted events are never candidates

| What it enforces | Site | Trigger |
|---|---|---|
| `AND deleted_at IS NULL` in every statement | `query.rs:242` | every non-empty-`q` call |

Regression test: `tests/fts_integration.rs:674-714`.

#### R7 — Optional NIP-01 pushdowns treat `Some(empty)` as "no constraint"

| What it enforces | Site | Trigger |
|---|---|---|
| `kinds`: predicate added only if `Some` **and** non-empty | `query.rs:267-273` | `kinds = Some(vec![...])` |
| `authors`: predicate added only if `Some` **and** non-empty | `query.rs:275-281` | `authors = Some(vec![...])` |
| `since`: `AND created_at >= to_timestamp($n)` (inclusive) | `query.rs:283-287` | `since.is_some()` |
| `until`: `AND created_at <= to_timestamp($n)` (inclusive) | `query.rs:289-293` | `until.is_some()` |

No validation that `since <= until`; an inverted range simply yields zero rows.
No validation that author byte-vectors are 32 bytes long.

#### R8 — Which kinds are searchable is a **storage** decision, not a crate rule

| What it enforces | Site | Trigger |
|---|---|---|
| The crate applies no allow/deny kind list; it only forwards the caller's `kinds` filter | `query.rs:267-273` (only kind logic in the crate) | always |
| Exclusion is realized by `search_tsv` being `NULL` for excluded kinds, which `@@` can never match | `migrations/0001_initial_schema.sql:222-226`, `migrations/0005_agent_turn_metric_fts.sql:26-31`, `migrations/0008_fresh_install_search_allowlist.sql:15-20`, `migrations/0014_push_lease_fts.sql:28-32`, `schema/schema.sql:211-215` | row insert |

Effective policy depends on which migration path a database took:

| Path | `search_tsv` expression | Reference |
|---|---|---|
| Fresh schema file | `CASE WHEN kind IN (1059, 30300, 30350, 30622, 44100, 44101, 44200) THEN NULL ELSE to_tsvector('simple', content) END` | `schema/schema.sql:211-215` |
| Migration 0001 | excludes `1059, 30300, 30622, 44100, 44101` | `migrations/0001_initial_schema.sql:222-226` |
| + 0005 | excludes `1059, 30300, 30622, 44100, 44101, 44200` | `migrations/0005_agent_turn_metric_fts.sql:26-31` |
| + 0008, **only if `events` is empty** | replaced by positive allowlist `kind IN (0, 9, 40002, 45001, 45003)` | `migrations/0008_fresh_install_search_allowlist.sql:13-23` |
| + 0014 | wraps whatever exists: `CASE WHEN kind = 30350 THEN NULL ELSE (<previous>) END` | `migrations/0014_push_lease_fts.sql:28-32` |
| Brownfield opt-in to allowlist | operator runs `scripts/maintenance/nip_rs_search_allowlist.sql:11-18` out of band | that file |

#### R9 — Ordering is fixed: relevance, then recency, then id

| What it enforces | Site |
|---|---|
| `ORDER BY rank DESC, created_at DESC, id` where `rank = ts_rank_cd(search_tsv, search_query.query)` | `query.rs:236`, `query.rs:295` |

`id` (BYTEA, ascending) is the deterministic tiebreak, which is what makes
OFFSET pagination stable across pages.

#### R10 — Pagination and limit clamps

| What it enforces | Value | Site |
|---|---|---|
| `per_page` clamped to `1..=PER_PAGE_MAX` | `PER_PAGE_MAX = 500` | `query.rs:129`, `query.rs:224` |
| `per_page == 0` → `PER_PAGE_DEFAULT` | `PER_PAGE_DEFAULT = 100` | `query.rs:130`, `query.rs:225-229` |
| `page` clamped to `1..=PAGE_MAX` (both on the SQL path and the empty-query path) | `PAGE_MAX = 1000` | `query.rs:138`, `query.rs:220`, `query.rs:230` |
| `OFFSET = (page - 1) * per_page_actual`, computed in `i64` | — | `query.rs:231`, `query.rs:297-298` |
| Returned `SearchResult.page` is the clamped page, not the requested one | — | `query.rs:322` (test: `tests/fts_integration.rs:1044`, expects `1000` for `u32::MAX`) |

Stated reason for the page clamp: pages are server-generated today (WS iterates
`1..=MAX_SEARCH_PAGES`, bridge passes a page), but clamp anyway so future
untrusted input cannot produce a huge OFFSET (`query.rs:135-138`). Caller values:
`MAX_SEARCH_PAGES = 10` (`crates/buzz-relay/src/handlers/req.rs:421`), WS
`per_page = 100` (`req.rs:591`), bridge `per_page = limit.min(500)`
(`crates/buzz-relay/src/api/bridge.rs:1660`) and bridge page from the raw JSON
`page`/`search_page`/`searchPage` field, defaulting to 1
(`bridge.rs:345-353`).

#### R11 — Authorization is explicitly NOT applied here

| What it enforces | Site |
|---|---|
| No membership check, no role check, no `#p` check, no owner gate, no author-only filter anywhere in the crate — the only visibility-affecting predicates are `community_id`, `deleted_at IS NULL`, the caller-supplied `ChannelScope`, and the storage-level NULL tsvector | entire `query.rs:216-323`; documented as such at `lib.rs:19-22` and `query.rs:3-9` |

Where authorization *is* applied (caller side, outside this crate):

| Check | Site |
|---|---|
| Re-fetch canonical events scoped by `(community, ids)` before use | `crates/buzz-relay/src/api/bridge.rs:1727-1731`, `crates/buzz-relay/src/handlers/req.rs:601-621` |
| Full NIP-01 filter re-match + channel-accessibility + reader authorization per hit (`search_hit_accepted`) | `crates/buzz-relay/src/api/bridge.rs:1594-1625`, invoked at `bridge.rs:1743` |
| Author-only kinds dropped (now inside `event_visible_to_reader`) | `crates/buzz-relay/src/api/bridge.rs:1753`; helper `handlers/req.rs:1222-1234` |
| WS path equivalent post-filter + channel check | `crates/buzz-relay/src/handlers/req.rs:685-701` |

Consequence if a caller forgets: the crate will happily return hits from channels
the reader cannot access whenever `ChannelScope::Any` is passed, and hits whose
non-pushed-down filter constraints (`#p`, `#e`, `#d`, `ids`) were never evaluated.
The relay's own comment states this exact failure mode for the `/query` bridge
(`crates/buzz-relay/src/api/bridge.rs:1582-1592`).


## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Business Rules

---

### BR-1 — Hash pre-image composition (the crate's core invariant)

`compute_hash` (`crates/buzz-audit/src/hash.rs:42-73`) feeds a single `Sha256`
in exactly this order. **9 hasher updates for the always-present fields, plus a
1-byte presence tag ahead of each of the two optional fields.**

| # | Field fed | Byte encoding | Code (file:line) |
|---|---|---|---|
| 1 | `community_id` | `Uuid::as_bytes()` → 16 raw big-endian UUID bytes | `hash.rs:45` |
| 2 | `seq` | `i64::to_be_bytes()` → 8 bytes, big-endian | `hash.rs:46` |
| 3 | `created_at` | `to_storage_precision(created_at).to_rfc3339().as_bytes()` — RFC3339 UTF-8, variable length (chrono emits 0/3/6/9 fractional digits) | `hash.rs:47-51` |
| 4 | `action` | `AuditAction::as_str().as_bytes()` — the snake_case string (`action.rs:35-49`) | `hash.rs:52` |
| 5 | `actor_pubkey` | presence tag `[1u8]` then raw pubkey bytes; or `[0u8]` alone when `None` | `hash.rs:53-59` |
| 6 | `object_id` | presence tag `[1u8]` then `id.as_bytes()` (UTF-8); or `[0u8]` alone when `None` | `hash.rs:60-66` |
| 7 | `detail` | `canonical_json(detail)?.as_bytes()` — sorted-key JSON, UTF-8; error propagates | `hash.rs:67` |
| 8 | `prev_hash` | the 32 stored bytes when `Some`; `GENESIS_HASH` (32 zero bytes) when `None` | `hash.rs:68-71` |

Digest is `hasher.finalize().into()` → `[u8; 32]` (`hash.rs:72`).

Encoding notes verified in code:
- No length prefixes or field separators other than the two 1-byte presence tags —
  the concatenation is otherwise unframed (`hash.rs:45-71`).
- The pre-image contains **no** `event_id`, `event_kind`, or `channel_id` field; the
  relay carries those inside `detail` (`crates/buzz-relay/src/handlers/event.rs:589-592`),
  so they are covered transitively through field 7.
- `community_id` is hashed **first**, explicitly to bind chain identity to the tenant
  (`hash.rs:28-30`, `hash.rs:44`). Test: `community_id_is_part_of_identity`
  (`hash.rs:216-224`).
- Trigger: every append (`service.rs:126`) and every verification step
  (`service.rs:197`) run the same function, so write and read pre-images cannot
  diverge by construction.

### BR-2 — `created_at` must be at storage precision before hashing

`to_storage_precision` truncates to microseconds (`hash.rs:22-24`), matching what
`TIMESTAMPTZ` round-trips (`hash.rs:13-18`). Enforced twice:
- write path stamps `log_timestamp()` = `to_storage_precision(Utc::now())`
  (`service.rs:21-23`, used at `service.rs:112`);
- `compute_hash` re-normalizes at the single consuming point so a caller that forgets
  cannot split the write/read pre-image (`hash.rs:32-37`, `hash.rs:47-51`).

Rationale recorded in code: RFC3339 sub-second width follows the value, so a
nanosecond timestamp and its microsecond truncation are different strings and every
entry would fail `verify_chain` with `HashMismatch`
(`hash.rs:167-183`, assertions at `hash.rs:177-182`). Unit guard without Postgres:
`log_timestamp_carries_no_sub_microsecond_digits` (`service.rs:283-292`).

### BR-3 — Canonical JSON determinism for `detail`

`canonical_json` recursively re-serializes objects with keys sorted through a
`BTreeMap<&str, &Value>` (`hash.rs:85-100`), arrays in order (`hash.rs:101-113`), and
scalars via `serde_json::to_string` (`hash.rs:114`). Serialization failure is
propagated, never substituted with an empty placeholder (`hash.rs:39-41`, `:77-79`,
`:94`, `:96`, `:109`). Trigger: any hash computation with a non-null `detail`.
Test: `canonical_json_key_order_is_stable` (`hash.rs:266-271`).

Not normalized: numeric formatting is whatever `serde_json` emits for the parsed
`Value`, and duplicate JSON keys are already collapsed by `serde_json`'s map before
this function sees them (`hash.rs:114`).

### BR-4 — Presence tags disambiguate `None` from `Some(empty)`

`actor_pubkey` and `object_id` each get a leading `1`/`0` byte (`hash.rs:53-66`), so
`Some(vec![])` and `None` hash differently. Test:
`presence_tag_distinguishes_none_from_empty` (`hash.rs:256-264`). Without the tags the
unframed concatenation would collide.

### BR-5 — Chain linking is per-community and read inside the transaction

`log_inner` reads the head of *this* community's chain only:
`SELECT seq, hash FROM audit_log WHERE community_id = $1 ORDER BY seq DESC LIMIT 1`
(`service.rs:94-101`). New entry takes `seq = prev_seq + 1` (`service.rs:110`) and
`prev_hash = Some(head.hash)` (`service.rs:104-107`). Head read and insert share one
transaction (`service.rs:87`, `:146`, commit `:149`). Trigger: every `log` call.

### BR-6 — Genesis handling

When no row exists for the community, the head query yields `None` → `(prev_seq, prev_hash) = (0, None)`
(`service.rs:103-109`), so the first entry gets `seq = 1` and `prev_hash = NULL`
stored (`service.rs:110`, bind at `service.rs:140`). `compute_hash` substitutes the
32-zero-byte `GENESIS_HASH` for the `None` (`hash.rs:9`, `:68-71`). Test:
`community_chain_starts_at_seq_1_with_null_prev` (`service.rs:318-336`).

### BR-7 — Single-writer guarantee is **per-community**, not global

`log` builds a namespaced lock key `format!("buzz_audit:{community_id}")`
(`service.rs:29`, `:58`) and takes `SELECT pg_advisory_lock(hashtextextended($1, 0))`
(`service.rs:59-62`) on the same pooled connection later used for the transaction
(`service.rs:54`, `:67`). Documented intent: two communities never serialize each
other's writes, avoiding both a bottleneck and a cross-tenant timing oracle
(`service.rs:25-28`, `lib.rs:13-15`). Lock ordering: acquire *before* `begin`, release
*after* commit (`service.rs:49-51`).

Because `hashtextextended` maps a text key into `bigint`, two distinct community keys
can in principle collide onto the same advisory-lock key (no collision handling exists
in code) — that would over-serialize, not corrupt.

### BR-8 — Lock release on every path including panic

`log_inner` is wrapped in `AssertUnwindSafe(...).catch_unwind().await`
(`service.rs:67-69`); the unlock query runs unconditionally afterwards with its result
discarded (`service.rs:71-74`); a captured panic is re-raised with
`std::panic::resume_unwind` after the unlock (`service.rs:76-79`). Trigger: any panic
or error inside the append. Caveats visible in code: an `Err` from the *unlock* query
itself is swallowed (`let _ =`, `service.rs:71`), and cancellation of the `log` future
before `catch_unwind` resolves would drop the connection without an explicit unlock
(session-scoped locks are then released by Postgres when the session ends — but a
pooled connection is reused rather than closed, so that release is not immediate).
No test covers the panic path (no test in `service.rs` induces a panic inside
`log_inner`).

### BR-9 — Verification algorithm

`verify_chain(community, from_seq, to_seq)` (`service.rs:160-206`):
1. `SELECT` the full projection for `community_id = $1 AND seq BETWEEN $2 AND $3
   ORDER BY seq ASC` (`service.rs:166-179`).
2. Empty result → `Ok(false)` (`service.rs:181-183`).
3. Walk rows in ascending `seq`, decoding each with `row_to_audit_entry`
   (`service.rs:188`).
4. For every row after the first in the range, require
   `entry.prev_hash == Some(previous_row.hash)` else `Err(ChainViolation { seq })`
   (`service.rs:190-195`).
5. Recompute `compute_hash(&entry)` and require equality with the stored `hash`, else
   `Err(HashMismatch { seq })` (`service.rs:197-200`).
6. All rows pass → `Ok(true)` (`service.rs:205`).

Rules this algorithm does **not** enforce (verified absent from `service.rs:160-206`):
- The first row of the range is not checked against `seq = 1` or against
  `GENESIS_HASH` — a range starting mid-chain has its `prev_hash` accepted unchecked.
- `seq` contiguity is not asserted; a gap between two rows is invisible unless the
  `prev_hash` link also breaks (a deleted row *does* break the link, so an interior
  deletion is caught, but a truncation of the chain **tail** is not).
- The stored `hash` uniqueness/`community_id` label are not cross-checked beyond what
  the recomputation implicitly covers.

Tests: `chain_links_within_one_community` (`service.rs:338-374`),
`verify_detects_tampering_within_a_community` (`service.rs:437-473`, asserts
`HashMismatch` at the tampered `seq`), `verify_empty_range_is_false` (`service.rs:512-526`).

### BR-10 — Cross-community replay is rejected

Because `community_id` is in the pre-image (BR-1), a row copied from community A into
community B recomputes to a different digest and fails as `HashMismatch`. Proven by
`cross_community_row_does_not_verify` (`service.rs:475-510`), which inserts A's hash
under B's id and asserts `Err(AuditError::HashMismatch { seq: 1 })` (`service.rs:508-509`).

### BR-11 — Per-community chain scoping is total across all three DB paths

Every statement filters on `community_id`: head read (`service.rs:96`), verify
(`service.rs:171`), list (`service.rs:223`). Schema reinforces it with
`PRIMARY KEY (community_id, seq)` and `UNIQUE (community_id, hash)`
(`migrations/0001_initial_schema.sql:616`, `:619`). Isolation test:
`chains_are_independent_per_community` (`service.rs:376-435`) — interleaved A/B writes
each start at `seq 1` and link only within their own chain (`service.rs:411-414`), and
`get_entries` scoped to A returns only A rows (`service.rs:425-433`).

### BR-12 — Provenance rule: `community_id` cannot come from client input

`NewAuditEntry.community_id` is typed `buzz_core::CommunityId` (`entry.rs:57`), a
newtype with no `Deserialize` (`crates/buzz-core/src/tenant.rs:37`), and
`NewAuditEntry` itself is deliberately not `Serialize`/`Deserialize`
(`entry.rs:46-51`). The raw `Uuid` is only unwrapped at the DB boundary
(`service.rs:89-91`). `buzz-core` documents this as a lint-and-review fence rather
than a compiler fence, since `CommunityId::from_uuid` is public
(`crates/buzz-core/src/tenant.rs:19-25`, `:44-47`).

### BR-13 — Unknown action strings fail closed on read

`row_to_audit_entry` parses the stored `action` via `FromStr` and maps any failure to
`AuditError::UnknownAction` after a `warn!` (`service.rs:238-243`; parser at
`action.rs:75-81`). Trigger: reading a row whose `action` string is not one of the 11.
Consequence: one unrecognised row fails the entire `verify_chain`/`get_entries` call
(`service.rs:188`, `:234`). The DB column is `VARCHAR(64)` with no CHECK
(`migrations/0001_initial_schema.sql:611`), so such rows are insertable out-of-band.

### BR-14 — Which events are refused: nothing is refused *inside this crate*

There is no kind-based, ephemeral-based, or content-based rejection anywhere in
`crates/buzz-audit/src`. `log` accepts any `NewAuditEntry` (`service.rs:53`); the only
error sources are DB failures, JSON canonicalisation failures, and (on read) unknown
actions. No `AuditError::AuthEventForbidden` variant exists (`error.rs:12-41`), and
grep across the repo finds that identifier only in `ARCHITECTURE.md:505`.

The refusals described in ARCHITECTURE.md live in the relay, upstream of the audit
sink:
- `KIND_AUTH` submissions are rejected during ingest with
  `IngestError::Rejected("invalid: AUTH events cannot be submitted")`
  (`crates/buzz-relay/src/handlers/ingest.rs:1438-1442`), so no audit entry is ever
  enqueued for them.
- Audit enqueue happens only from the persistent-event path
  (`crates/buzz-relay/src/handlers/event.rs:335`, `:486`, helper at `:540-577`) and
  from media upload (`crates/buzz-relay/src/api/media.rs:422-442`); ephemeral events
  are dispatched on a separate branch and never reach `enqueue_event_created_audit`.

### BR-15 — Append-only by absence of any other write

The crate issues exactly one mutating statement, the `INSERT`
(`service.rs:130-147`). No `UPDATE`, `DELETE`, or `TRUNCATE` appears in
`crates/buzz-audit/src`. Append-only is therefore a property of this code path only —
it is not enforced by DB privileges, triggers, or constraints in
`migrations/0001_initial_schema.sql:606-619`.

### BR-16 — `detail` must not carry secrets (documented, unenforced)

`NewAuditEntry.detail` is documented as never bearer-token material; callers must not
write tokens or passwords (`entry.rs:64-71`). This is a doc-comment obligation only —
no validation, redaction, or size limit exists on the field in
`crates/buzz-audit/src`.


## Module: buzz-media (`crates/buzz-media`)

### Aspect: Business Rules

Every rule below was read in the source; the "Trigger" column names the entry point that enforces it.

---

### 1. Content addressing and integrity

| # | Rule | Enforcement | Trigger |
|---|---|---|---|
| BR-1 | Blob key is `{sha256}.{ext}` where sha256 is computed server-side over the received bytes | `crates/buzz-media/src/upload.rs:84`, `crates/buzz-media/src/upload.rs:94` | every buffered upload |
| BR-2 | Streaming (video) hash is computed incrementally over the same bytes written to the temp file | `crates/buzz-media/src/upload.rs:365-381`, `crates/buzz-media/src/upload.rs:397` | `process_video_upload` |
| BR-3 | The client-declared hash (Blossom `x` tag) must equal the server-computed hash, else `HashMismatch` | `crates/buzz-media/src/auth.rs:190-196` | `verify_blossom_upload_auth`, called at `crates/buzz-media/src/upload.rs:85` and `crates/buzz-media/src/upload.rs:412` |
| BR-4 | The stored key is derived from the *server-computed* hash, not the `x` tag — a spoofed `x` cannot mis-key a blob | `crates/buzz-media/src/upload.rs:94`, `crates/buzz-media/src/upload.rs:421` | all upload paths |
| BR-5 | Extension is derived from sniffed MIME only; no client filename is consulted anywhere in the crate | `crates/buzz-media/src/validation.rs:930-939`, `crates/buzz-media/src/validation.rs:203-206`, `crates/buzz-media/src/upload.rs:420` | all upload paths |

There is **no re-read-and-verify** after the S3 PUT, and no verification on read: `MediaStorage::get`/`get_stream`/`get_range` return bytes without re-hashing (`crates/buzz-media/src/storage.rs:105-146`).

---

### 2. Size limits

| # | Rule | Value | file:line |
|---|---|---|---|
| BR-6 | Image upload cap (non-GIF) | `config.max_image_bytes` (no in-crate default; required serde field) | `crates/buzz-media/src/validation.rs:259-268`, `crates/buzz-media/src/config.rs:41` |
| BR-7 | Animated-GIF cap, applied when sniffed MIME == `image/gif` | `config.max_gif_bytes` | `crates/buzz-media/src/validation.rs:259-262` |
| BR-8 | Generic-file cap | `config.max_file_bytes`, default **104_857_600 (100 MB)** | `crates/buzz-media/src/validation.rs:167-172`, `crates/buzz-media/src/config.rs:7-9` |
| BR-9 | Video cap, pre-checked against `Content-Length` before streaming | `config.max_video_bytes`, default **524_288_000 (500 MB)** | `crates/buzz-media/src/upload.rs:311-320`, `crates/buzz-media/src/config.rs:3-5` |
| BR-10 | Video cap re-checked per read chunk during streaming (running total) | same | `crates/buzz-media/src/upload.rs:372-378` |
| BR-11 | Video cap re-checked a third time against the temp file's on-disk size | same | `crates/buzz-media/src/validation.rs:298-304` |
| BR-12 | Config coherence: all four caps must be > 0 and `max_gif_bytes <= max_image_bytes` | startup failure | `crates/buzz-media/src/config.rs:76-90` |
| BR-13 | Pixel-count cap (image bomb): `width * height <= 25_000_000` | 25 MP | `crates/buzz-media/src/validation.rs:269-273` |
| BR-14 | Video duration `> 600.0 s` → `DurationTooLong`; `<= 0.0` → `InvalidVideo` | 600 s | `crates/buzz-media/src/validation.rs:354-359` |
| BR-15 | Video resolution `> 3840 × 2160` → `ResolutionTooHigh` | 4K | `crates/buzz-media/src/validation.rs:364-366` |

`max_image_bytes` and `max_gif_bytes` have **no `#[serde(default)]`** — the caller must supply them (`crates/buzz-media/src/config.rs:38-44`); the relay defaults them to 50 MB / 10 MB (`crates/buzz-relay/src/config.rs:626-632`).

---

### 3. MIME sniffing vs declared content type

| # | Rule | file:line |
|---|---|---|
| BR-16 | MIME is always sniffed from magic bytes via `infer`; the HTTP `Content-Type` header is never read in this crate | `crates/buzz-media/src/validation.rs:239-242` ("never trust Content-Type header"), `crates/buzz-media/src/validation.rs:176-201` |
| BR-17 | Image path allowlist: exactly `image/jpeg`, `image/png`, `image/gif`, `image/webp` | `crates/buzz-media/src/validation.rs:15`, enforced `crates/buzz-media/src/validation.rs:245-247` |
| BR-18 | Unsniffable bytes on the image path → `UnknownContentType` (fail closed) | `crates/buzz-media/src/validation.rs:239-242` |
| BR-19 | `video/mp4` uploaded through the image path is rejected (`DisallowedContentType`) — spoofing an MP4 as an image cannot skip video validation | `crates/buzz-media/src/validation.rs:9-13`, `crates/buzz-media/src/validation.rs:245-247` |
| BR-20 | Generic path: any sniffed `image/*`, `video/*`, or `audio/*` is rejected so recognized media cannot bypass its format-specific validator; audio is rejected outright pending a sanitizer | `crates/buzz-media/src/validation.rs:186-196` |
| BR-21 | Generic path: any structurally valid ISO-BMFF `ftyp` container is rejected even when `infer` doesn't recognize the brand | `crates/buzz-media/src/validation.rs:174-181` |
| BR-22 | Generic path deny-list (defence in depth against stored XSS / malware): `text/html`, `application/xhtml+xml`, `image/svg+xml`, `application/javascript`, `text/javascript`, `application/x-msdownload`, `application/x-executable`, `application/vnd.microsoft.portable-executable`, `application/x-mach-binary`, `application/x-sharedlib`, `application/x-elf`, `application/x-msi`, `application/vnd.android.package-archive`, `application/x-apple-diskimage` | `crates/buzz-media/src/validation.rs:75-95`, enforced `crates/buzz-media/src/validation.rs:198-200` |
| BR-23 | Unsniffable bytes on the generic path are accepted as `application/octet-stream` / ext `bin` (text, CSV, JSON, source code have no magic bytes) | `crates/buzz-media/src/validation.rs:207` |
| BR-24 | Serve policy: only `image/*` and `video/*` are inline; everything else (including PDF) is an attachment | `crates/buzz-media/src/validation.rs:216-218` |

---

### 4. Metadata-free (privacy) enforcement — no transformation, reject instead

The crate does **not strip** EXIF/XMP/ICC; it *rejects* any file carrying a metadata channel, on the stated premise that client encoders sanitize before upload (`crates/buzz-media/src/validation.rs:487-491`).

| # | Container | Rule | file:line |
|---|---|---|---|
| BR-25 | JPEG | Only canonical JFIF `APP0` (fixed length formula) and 12-byte `Adobe` `APP14` allowed; `APP1`–`APP13`, `APP15`, and `COM` → `MetadataForbidden`; trailing bytes after `EOI` → `MetadataForbidden` | `crates/buzz-media/src/validation.rs:539-563`, `crates/buzz-media/src/validation.rs:528-532` |
| BR-26 | PNG | `eXIf`/`zTXt`/`iTXt`/`iCCP` forbidden; unknown ancillary chunks forbidden; only `cHRM gAMA sBIT sRGB bKGD hIST tRNS sPLT acTL fcTL fdAT` allowed; `pHYs` deliberately excluded; bytes after `IEND` forbidden | `crates/buzz-media/src/validation.rs:614-651` |
| BR-27 | PNG (product exception) | Exactly one `tEXt` chunk whose keyword is `buzz_agent_snapshot` or `buzz_team_snapshot` followed by NUL is permitted (agent/team snapshot manifests); a second such chunk or any other keyword is forbidden | `crates/buzz-media/src/validation.rs:579-590`, `crates/buzz-media/src/validation.rs:601-611` |
| BR-28 | WebP | RIFF declared size must equal `len-8`; only `VP8 `/`VP8L`/`VP8X`/`ALPH`/`ANIM`/`ANMF` chunks; `VP8X` ICC/EXIF/XMP flags (`0x20/0x08/0x04`) forbidden; `ANMF` sub-chunks recursively restricted to `ALPH`/`VP8 `/`VP8L` | `crates/buzz-media/src/validation.rs:690-732`, `crates/buzz-media/src/validation.rs:660-687` |
| BR-29 | GIF | Only Graphic Control Extensions with exact spec shape and `NETSCAPE2.0`/`ANIMEXTS1.0` loop application extensions; comment/plain-text/other app extensions forbidden; trailing bytes after `0x3B` forbidden | `crates/buzz-media/src/validation.rs:734-829` |
| BR-30 | MP4 | Box allowlist (35 types); explicit forbidden list `meta ilst keys data uuid xml  bxml loci ©xyz name chap`; any box not in the allowlist → `MetadataForbidden`; `udta` permitted **only** as the exact 53-byte empty-ffmpeg `udta` payload | `crates/buzz-media/src/validation.rs:831-861`, `crates/buzz-media/src/validation.rs:895-908` |
| BR-31 | MP4 tracks | Exactly one video track; ≥2 video or ≥2 audio tracks → `MetadataForbidden`; any non-video/non-audio track type (e.g. timed metadata) → `MetadataForbidden` | `crates/buzz-media/src/validation.rs:325-331`, `crates/buzz-media/src/validation.rs:369-386` |

Only transformation performed anywhere in the crate: JPEG **thumbnail** generation (≤320×320, aspect preserved) plus a 4×3 blurhash, both derived artifacts written to separate keys; the original bytes are stored unmodified (`crates/buzz-media/src/thumbnail.rs:30-38`, `crates/buzz-media/src/upload.rs:129`). No transcoding, no re-encoding of the stored blob, no EXIF stripping.

---

### 5. Codec / container policy (video)

| # | Rule | file:line |
|---|---|---|
| BR-32 | Container must be structurally ISO-BMFF `ftyp` with an MP4 brand (major **or** any compatible brand in `isom iso2..iso9 mp41 mp42 avc1 dash M4V `) — proprietary major brands accepted when a compatible brand matches | `crates/buzz-media/src/validation.rs:17-20`, `crates/buzz-media/src/validation.rs:52-62`, called `crates/buzz-media/src/upload.rs:403-407` |
| BR-33 | QuickTime major brand `qt  ` → `UnsupportedContainer` | `crates/buzz-media/src/validation.rs:316-322` |
| BR-34 | Video codec must be H.264 (`mp4::MediaType::H264`), else `WrongCodec` (rejects HEVC/VP9/AV1) | `crates/buzz-media/src/validation.rs:335-341` |
| BR-35 | Audio codec must be AAC (`mp4::MediaType::AAC`), else `WrongCodec` | `crates/buzz-media/src/validation.rs:378-382` |
| BR-36 | `moov` must precede `mdat` (fast-start), verified by a header-only top-level atom scan before the MP4 parser runs | `crates/buzz-media/src/validation.rs:291-293`, `crates/buzz-media/src/validation.rs:408-484` |
| BR-37 | Audio-only MP4 (no video track, has audio) → `DisallowedContentType("audio/mp4")`; no tracks at all → `InvalidVideo` | `crates/buzz-media/src/validation.rs:388-395` |
| BR-38 | `track.timescale() == 0` rejected before calling `duration()` to avoid a division-by-zero panic in the `mp4` crate | `crates/buzz-media/src/validation.rs:346-351` |

---

### 6. Blossom auth rules (kind 24242)

| # | Rule | file:line |
|---|---|---|
| BR-39 | Schnorr signature must verify | `crates/buzz-media/src/auth.rs:37-39` |
| BR-40 | Kind must be exactly 24242 | `crates/buzz-media/src/auth.rs:42-44` |
| BR-41 | `content` must be non-empty after trim (BUD-11 human-readable string) | `crates/buzz-media/src/auth.rs:47-49` |
| BR-42 | `t` tag required and must equal the expected verb (`upload`/`get`) | `crates/buzz-media/src/auth.rs:59-66`, `crates/buzz-media/src/auth.rs:97-99` |
| BR-43 | `expiration` tag required and must be strictly in the future | `crates/buzz-media/src/auth.rs:101-110` |
| BR-44 | `created_at` must not be > 5 s in the future | `crates/buzz-media/src/auth.rs:116-119` |
| BR-45 | `created_at` must be within `max_age_secs`: **600 s** for buffered image/file uploads, **3600 s** for video uploads | `crates/buzz-media/src/auth.rs:120-122`; windows at `crates/buzz-media/src/upload.rs:85` and `crates/buzz-media/src/upload.rs:412` |
| BR-46 | If any `server` tag is present, the bound tenant host must match one of them after `normalize_host` (case, trailing dot, default port, scheme/path tolerant); if the bound host is unknown (`None`) → reject (fail closed) | `crates/buzz-media/src/auth.rs:135-143`, `crates/buzz-media/src/auth.rs:163-170` |
| BR-47 | Upload: at least one `x` tag must equal the computed sha256 | `crates/buzz-media/src/auth.rs:190-196` |
| BR-48 | Get: `x` match **or** `server` match suffices; neither → `InsufficientScope`. A `server`-scoped get token grants reads of all blobs on that host until expiry (documented, deliberate) | `crates/buzz-media/src/auth.rs:201-236` |

---

### 7. Deduplication / idempotency

| # | Rule | file:line |
|---|---|---|
| BR-49 | Upload short-circuits only when **both** sidecar and blob exist; a sidecar without its blob falls through to re-upload | `crates/buzz-media/src/upload.rs:97-103`, `crates/buzz-media/src/upload.rs:424-427` |
| BR-50 | Short-circuit returns the descriptor built from the existing sidecar with the **original** `uploaded_at`, not now | `crates/buzz-media/src/upload.rs:120-128` |
| BR-51 | Identical bytes across communities share one blob object but get separate sidecars (dedup is global, visibility is per-community) | `crates/buzz-media/src/storage.rs:177-185`, test `crates/buzz-media/src/storage.rs:352-377` |
| BR-52 | A re-upload of known bytes still writes an upload record when records are enabled, because no blob PUT occurs and the event would otherwise be invisible to moderation | `crates/buzz-media/src/upload.rs:104-119`, `crates/buzz-media/src/upload.rs:428-443` |

---

### 8. Write ordering (publish gate)

| # | Rule | file:line |
|---|---|---|
| BR-53 | Order is: blob → derived artifacts (thumb) → upload record → sidecar. The sidecar is the serve gate, so a failed moderation record cannot publish media | `crates/buzz-media/src/upload.rs:129-172`, `crates/buzz-media/src/upload.rs:434-475` |
| BR-54 | Record existence implies the referenced objects are readable (record written after blob durability) | `crates/buzz-media/src/upload_record.rs:35-40`, `crates/buzz-media/src/upload.rs:158-172` |
| BR-55 | On metadata-generation failure the orphan blob is deliberately **not** deleted (concurrent same-hash uploads could race); logged at `warn` and left for GC | `crates/buzz-media/src/upload.rs:122-127`, `crates/buzz-media/src/upload.rs:132-141` |

---

### 9. Deletion / GC / orphan sweeping / retention

| # | Rule | file:line |
|---|---|---|
| BR-56 | The crate exposes a raw `delete(key)` and nothing else — no blob-lifecycle delete, no cascade to sidecar/thumb/record, no `DELETE /{sha256}` implementation | `crates/buzz-media/src/storage.rs:158-164` |
| BR-57 | GC is **not implemented**; a background job is named as future work ("A V2 background GC job can sweep blobs with no matching sidecar after a grace period") | `crates/buzz-media/src/upload.rs:122-127` |
| BR-58 | Orphan *accounting* exists: a blob sha with no sidecar binding in any community counts as an orphan blob; a sidecar binding whose sha has no blob counts as an orphan sidecar | `crates/buzz-media/src/bucket_index.rs:294-315` |
| BR-59 | Multi-variant anomaly: one sha with >1 blob extension is counted and all its variant bytes are billed | `crates/buzz-media/src/bucket_index.rs:298-306` |
| BR-60 | Logical billing double-counts a blob bound to N communities (documented as intentional); thumb bytes are attributed to the blob's sha | `crates/buzz-media/src/bucket_index.rs:210-213`, `crates/buzz-media/src/bucket_index.rs:317-334` |
| BR-61 | `_uploads/` auxiliary objects count toward physical totals only, never logical/orphan math | `crates/buzz-media/src/bucket_index.rs:266-268` |
| BR-62 | Sweep is bounded: cumulative object count checked **before** folding each page; breach → `CapExceeded` and the whole sweep fails (no partial snapshot) | `crates/buzz-media/src/bucket_index.rs:394-398`, `crates/buzz-media/src/bucket_index.rs:339-341` |
| BR-63 | `is_truncated == true` with no continuation token → `MalformedPage` (fail, don't return partial data) | `crates/buzz-media/src/bucket_index.rs:404-409` |
| BR-64 | Key classification is strict: 64-char lowercase hex sha, 1–8 alphanumeric ext, canonical lowercase UUID, uppercase-only 26-char ULID; anything malformed falls to `Unknown` rather than being coerced | `crates/buzz-media/src/bucket_index.rs:75-195` |

No retention/TTL/expiry logic exists anywhere in the crate (no lifecycle-policy code, no `expires_at` field).

---

### 10. Upload attribution / IP collection rules

| # | Rule | file:line |
|---|---|---|
| BR-65 | Upload records are off unless `upload_records_enabled` | `crates/buzz-media/src/config.rs:49-50`, gating at `crates/buzz-media/src/upload.rs:104`, `:158`, `:428`, `:464` |
| BR-66 | IP recording is fail-empty: only a single syntactically valid **public** address is recorded; private, loopback, link-local, CGNAT, documentation, ULA, Teredo, 6to4, v4-mapped, comma lists, `ip:port` all record nothing | `crates/buzz-media/src/upload_record.rs:191-256` |
| BR-67 | A port is recorded only alongside a valid IP (`ip.and(port)`) | `crates/buzz-media/src/upload_record.rs:151-153` |
| BR-68 | Startup fails if an IP header is configured without records enabled, or a port header without an IP header, or if either header name is not a valid HTTP header name | `crates/buzz-media/src/config.rs:91-124` |
| BR-69 | Community id/host and uploader pubkey in the record are server-resolved (`TenantContext`, verified auth event pubkey), never client-supplied | `crates/buzz-media/src/upload_record.rs:154-168`, `crates/buzz-media/src/storage.rs:204-209` |


## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Business Rules

---

### 1. Definition validation (`WorkflowDef::validate`, `schema.rs:152-229`)

| Rule | Enforcement | Line | Trigger |
|---|---|---|---|
| `name` must be non-empty after `trim()` | `InvalidDefinition("name is required and must not be empty")` | `schema.rs:153-157` | every parse via `parse_yaml` |
| At least one step | `InvalidDefinition("at least one step is required")` | `schema.rs:159-163` | every parse |
| Step id non-empty after trim | `InvalidDefinition("step id must not be empty")` | `schema.rs:176-180` | per step |
| Step id ≤ 64 chars, only `[A-Za-z0-9_]` | closure `valid_step_id` (`schema.rs:168-172`); error names the offending id | `schema.rs:181-186` | per step — protects evalexpr variable names (`steps_{id}_output_{field}`) |
| Step ids unique | `HashSet` insert; `InvalidDefinition("duplicate step id: …")` | `schema.rs:187-192` | per step |
| `schedule` requires `cron` **or** `interval` | `InvalidDefinition("schedule trigger requires either 'cron' or 'interval'")` | `schema.rs:196-200` | schedule triggers only |
| `schedule` forbids both `cron` and `interval` | `InvalidDefinition("… cannot specify both …")` | `schema.rs:202-206` | schedule triggers only |
| `cron` must parse after normalization | `validate_cron` → `cron::Schedule` parse | `schema.rs:208-210`, `schema.rs:237-243` | schedule with cron |
| `interval` must parse and be ≥ 60 s | `parse_duration_secs`; `InvalidDefinition("interval must be at least 60s (cron loop ticks every 60 seconds)")` at `schema.rs:220-224` | `schema.rs:212-225` | schedule with interval |

Cron normalization: 5 fields → `0 {expr} *`; 6 fields → `{expr} *`; otherwise unchanged (`schema.rs:250-257`). Nothing else is validated at definition time — notably `call_webhook.url` scheme/host, `send_dm.to`, `add_reaction.emoji`, `delay.duration`, and `request_approval.timeout` are **not** checked by `validate()`.

---

### 2. Trigger matching semantics

Kind gate (`trigger_matches_event`, `lib.rs:955-964`):

| Trigger | Fires only on kind | Constant value |
|---|---|---|
| `message_posted` | `KIND_STREAM_MESSAGE` | 9 (`crates/buzz-core/src/kind.rs:419`) |
| `reaction_added` | `KIND_REACTION` | 7 (`crates/buzz-core/src/kind.rs:58`) |
| `diff_posted` | `KIND_STREAM_MESSAGE_DIFF` | 40008 (`crates/buzz-core/src/kind.rs:433`) |
| `schedule`, `webhook` | never from events (`false`) | `lib.rs:962` |

Pre-conditions inside `on_event` (`lib.rs:276-383`), evaluated in order:
1. Event must carry `channel_id`, else skip (`lib.rs:281-289`).
2. `is_workflow_execution_kind(kind)` (46001–46012) ⇒ skip (`lib.rs:293-295`).
3. Enabled-workflow list for `(community_id, channel_id)` from the 10 s moka cache, else DB (`lib.rs:297-311`); empty ⇒ return (`lib.rs:313-315`).
4. Definition JSON must deserialize; parse failure logs a warning and skips that workflow (`lib.rs:331-337`).
5. `def.enabled` must be true **and** the kind must match (`lib.rs:335`).
6. `should_fire_workflow` gate (below).
7. Run row created; execution spawned as a detached tokio task (`lib.rs:344-381`).

`should_fire_workflow` (`lib.rs:806-882`):
- `reaction_added` with `emoji: Some(e)`: exact string equality against `trigger_ctx.emoji` — no normalization, no shortcode↔unicode mapping (`lib.rs:807-822`). `emoji: None` matches any reaction.
- `message_posted` with `filter`: `evaluate_condition(expr, ctx, &HashMap::new())`; `false` ⇒ skip, `Err` ⇒ skip with a warning (fail-closed) (`lib.rs:824-846`).
- `diff_posted` with `filter`: identical logic, duplicated block (`lib.rs:848-880`).
- Note: step outputs are always empty at trigger-filter time, so `steps_*` variables are unavailable in filters.

`TriggerContext` construction (`build_trigger_context`, `lib.rs:884-953`):
- `text` = event content (`lib.rs:886`, `lib.rs:945`).
- `author` = first tag whose kind stringifies to `actor`, else `event.pubkey` hex (`lib.rs:888-897`).
- `emoji` = content when kind == `KIND_REACTION`, else empty (`lib.rs:901-905`).
- `message_id` = for reactions, the **last** `e` tag whose value is 64 ASCII-hex chars, falling back to the reaction's own id; for other kinds, the event's own id (`lib.rs:910-938`).
- `channel_id` = event channel UUID string or empty (`lib.rs:947-950`); `timestamp` = `created_at` seconds as string; `webhook_fields` always empty on this path (`lib.rs:952`).

---

### 3. Condition evaluation

| Rule | Detail | Line |
|---|---|---|
| Engine | `evalexpr` v11, `HashMapContext`, `eval_boolean_with_context` | `executor.rs:224-231`, `executor.rs:373` |
| Name mangling | dots → underscores: `trigger_text`, `trigger_author`, `trigger_channel_id`, `trigger_timestamp`, `trigger_emoji`, `trigger_message_id`; step outputs `steps_{step_id}_output_{field}` | `executor.rs:288-296`, `executor.rs:306-315` |
| Custom functions | exactly four: `str_contains(h, n)`→bool, `str_starts_with(s, p)`→bool, `str_ends_with(s, s2)`→bool, `str_len(s)`→int (byte length via `s.len()`) | `executor.rs:236-283` |
| Webhook field vars | registered as `trigger_{key}` **before** the standard fields so standard fields always win; keys already starting `trigger_`/`steps_` are skipped outright | `executor.rs:285-296` |
| Typing of step outputs | JSON string→String, bool→Boolean, i64→Int, other numbers→Float, null→Empty, arrays/objects→their JSON text as String | `executor.rs:318-335` |
| Expression length cap | `MAX_EXPR_LEN = 4096` bytes; longer ⇒ `ConditionError` before evaluation | `executor.rs:362-368` |
| Timeout | `EVAL_TIMEOUT = 100 ms`, applied via `tokio::time::timeout` around `spawn_blocking` | `executor.rs:342`, `executor.rs:370-380` |
| Timeout caveat (in-code) | the comment states the `spawn_blocking` thread is **not cancelled** by the timeout and runs to completion; the length cap is the actual mitigation | `executor.rs:358-361` |
| Failure semantics | eval error, task panic, or timeout all map to `ConditionError`; in `execute_steps` this **fails the run** (not "skip") | `executor.rs:375-383`, `executor.rs:1113-1121` |

---

### 4. Template variable resolution

| Rule | Detail | Line |
|---|---|---|
| Fast path | no `{{` in the string ⇒ returned unchanged | `executor.rs:78-80` |
| Single pass | one left-to-right scan; substituted values are **not** re-scanned, so `{{` inside a resolved value is never re-expanded | `executor.rs:82-121` |
| Supported paths | `trigger.<field>` (incl. any `webhook_fields` key via `get_field`'s fallback arm) and `steps.<ID>.output.<FIELD>` — exactly 3 dot segments after `steps.`, middle segment must literally be `output` | `executor.rs:126-151`, `executor.rs:49-59` |
| Depth | one level only: `json_get_str` reads a single key off a JSON **object**; nested paths and non-object outputs resolve to `None` | `executor.rs:154-163` |
| Unknown variable | the original `{{expr}}` is re-emitted verbatim, no error | `executor.rs:110-117` |
| Unclosed `{{` | emitted literally, remainder appended, `Ok` returned | `executor.rs:96-104` |
| Filters | `| truncate(N)` (char-wise `chars().take(n)`); `| npub` / `| truncate_pubkey` (hex pubkey → full bech32; non-pubkey passes through). Any other filter ⇒ `TemplateError` | `executor.rs:178-201` |
| `truncate` arg | must parse as `usize`, else `TemplateError` | `executor.rs:183-185` |
| JSON coercion | string as-is, bool/number via `to_string`, null → empty string, arrays/objects → JSON text | `executor.rs:165-174` |
| Which fields are templated | `send_message.text` + `.channel`; `send_dm.to` + `.text`; `set_channel_topic.topic`; `add_reaction.emoji`; `call_webhook.url` + header **values** + `.body`. **Not** templated: `call_webhook.method`, header **names**, `request_approval.timeout`, `delay.duration` | `executor.rs:406-452` |

---

### 5. Per-action semantics

| Action | Behaviour today | Validation performed | Line |
|---|---|---|---|
| `send_message` | Loads the run (`get_workflow_run`) then the workflow (`get_workflow`), both community-scoped; resolves the destination; hex-encodes `workflow.owner_pubkey` for attribution; calls `ActionSink::send_message`. Output `{sent, event_id}` | destination rules below; DB/lookup failure ⇒ `WebhookError`; missing sink ⇒ `InvalidDefinition` | `executor.rs:530-578` |
| `send_dm` | logs a warning, returns `WorkflowError::NotImplemented("SendDm")` — always fails the run | none | `executor.rs:580-584` |
| `set_channel_topic` | logs a warning, returns `WorkflowError::NotImplemented("SetChannelTopic")` | none | `executor.rs:586-590` |
| `add_reaction` | requires non-empty `trigger_ctx.message_id`; with feature `reqwest` performs `POST {BUZZ_RELAY_BASE_URL}/api/messages/{message_id}/reactions`; without the feature returns `{added:false, skipped:true}` | empty `message_id` ⇒ `InvalidDefinition` | `executor.rs:592-617`, `executor.rs:888-933` |
| `call_webhook` | with `reqwest`: SSRF check, DNS-pinned per-request client, redirects disabled, 10 s timeout, 1 MiB response cap, output `{status, body}`; without the feature returns `{status:0, body:null, skipped:true}` | URL must parse and have a host; method must be a valid HTTP method token | `executor.rs:619-648`, `executor.rs:781-869` |
| `request_approval` | logs, generates a v4 UUID token, returns `StepResult::Suspended` — **no DB record is created, no event is emitted** (`TODO (WF-08)` at `executor.rs:663`) | none; `timeout` string is neither parsed nor validated (default literal `"24h"` used only for the log line) | `executor.rs:650-668` |
| `delay` | parses the duration, rejects > `MAX_DELAY_SECS = 270`, then `tokio::time::sleep`; output `{slept_secs}` | `> 270 s` ⇒ `InvalidDefinition`; unparseable ⇒ `InvalidDefinition` | `executor.rs:671-690` |

`send_message` destination rules (`resolve_send_message_channel`, `executor.rs:468-517`):
1. Blank/whitespace `channel` overrides are treated as absent (`executor.rs:473-475`).
2. If the workflow row has `channel_id`: any override must parse as a UUID **and equal** that channel, else `InvalidDefinition("channel override must match the workflow channel …")`; the bound channel is always used (`executor.rs:477-493`).
3. Workflow with no bound channel: a valid UUID override is used (`executor.rs:495-503`).
4. Otherwise the trigger's channel is used; if that is blank ⇒ `InvalidDefinition("no channel_id available …")` (`executor.rs:505-513`).

---

### 6. Step sequencing and failure handling (`execute_steps`, `executor.rs:1083-1217`)

1. Steps run strictly in definition order; indices `< start_index` are skipped with a debug log (`executor.rs:1091-1094`).
2. `if:` false ⇒ trace `{"status":"skipped"}` and continue; `if:` error ⇒ return `(ConditionError, PartialProgress{step_index: i, trace})` (`executor.rs:1096-1124`).
3. Template resolution error ⇒ return with partial progress (`executor.rs:1126-1135`).
4. Per-step timeout = `step.timeout_secs` or `engine.config.default_timeout_secs` (300 s default), wrapped around `dispatch_action`; expiry ⇒ `StepTimeout { step_id, timeout_secs }` (`executor.rs:1136-1177`).
5. Any action error aborts the whole run — there is no per-step retry, no continue-on-error, and no compensation (`executor.rs:1160-1167`).
6. `Completed` ⇒ trace entry with output and `step_outputs.insert(step.id, output)` (`executor.rs:1179-1187`).
7. `Suspended` ⇒ returns immediately with `approval_token: Some(_)`, `step_index: i`, accumulated outputs and trace (`executor.rs:1188-1199`).
8. `Skipped` ⇒ trace entry (unreachable: `dispatch_action` never returns `StepResult::Skipped`) (`executor.rs:1200-1206`).
9. Normal end ⇒ `ExecutionResult { approval_token: None, step_index: def.steps.len(), … }` (`executor.rs:1208-1216`).

---

### 7. Run status state machine

Only three statuses are ever written by this crate.

```
                     create_workflow_run (relay / on_event / cron)
                                 │  status = Pending (buzz-db default)
                                 ▼
            ┌──────────── try_acquire() permit ────────────┐
            │ fail                                          │ ok
            ▼                                               ▼
   CapacityExceeded error                            update_workflow_run(Running)
   (status stays Pending;                       executor.rs:985-994 / :1044-1053
    finalize_run then writes Failed)                        │
                                                            ▼
                                                    execute_steps loop
                                       ┌────────────┬───────┴────────────┐
                                       │ Ok, token  │ Ok, no token       │ Err(e, progress)
                                       ▼            ▼                    ▼
                                    Failed      Completed             Failed
                        "approval gates not   lib.rs:218-238     error = e.to_string()
                         yet implemented —                       lib.rs:242-261
                         see WF-08"
                         lib.rs:192-215
```

`WaitingApproval`, `Pending` (explicitly) and `Cancelled` are never written by this crate — repo-wide grep finds `WaitingApproval` only in the comment at `lib.rs:193` and in two relay guards (`crates/buzz-relay/src/handlers/command_executor.rs:1197`, `:1253`).

`finalize_run` is the single mapping point and prepends `existing_trace` when supplied (`lib.rs:180`, `lib.rs:218-221`, `lib.rs:242-245`). DB update failures are logged, not propagated (`lib.rs:207-213`, `lib.rs:231-236`, `lib.rs:256-260`).

---

### 8. Approval gate lifecycle (as implemented)

| Step | Reality | Line |
|---|---|---|
| Token generation | `Uuid::new_v4().to_string()`; `run_id`/`step_id` args deliberately unused | `executor.rs:698-700` |
| Persistence | none — `TODO (WF-08): create approval record in DB, emit kind:46010` | `executor.rs:663` |
| Executor return | `StepResult::Suspended { approval_token }`, bubbled up as `ExecutionResult.approval_token` | `executor.rs:665-667`, `executor.rs:1188-1199` |
| Finalization | `finalize_run` logs `"Workflow hit approval gate — not yet implemented, marking as failed"` and writes `RunStatus::Failed` with error text `"approval gates not yet implemented — see WF-08"` | `lib.rs:192-215` |
| Resume | `execute_from_step` supports resumption (start index + `initial_outputs` + trace preservation) and the relay calls it, but the relay's `resume_workflow` refuses unless `run.status == WaitingApproval` — a status this codebase never writes | `executor.rs:1018-1075`, `crates/buzz-relay/src/handlers/command_executor.rs:1244-1320` |

---

### 9. Concurrency admission

| Rule | Detail | Line |
|---|---|---|
| Mechanism | `Arc<Semaphore>` created with `config.max_concurrent.max(1)` permits (default 100) | `lib.rs:68`, `lib.rs:110-111` |
| Admission | `try_acquire()` — non-blocking, no queueing; failure ⇒ `WorkflowError::CapacityExceeded` with an empty `PartialProgress` | `executor.rs:978-983`, `executor.rs:1028-1033` |
| Scope | permit held for the whole run including `delay` sleeps and step timeouts (`_permit` binding lives to end of fn) | `executor.rs:978`, `executor.rs:1028` |
| Rejected runs | `finalize_run` records them as `Failed` with message `"capacity exceeded"` | `lib.rs:242-261`, `error.rs:52-54` |

---

### 10. Delay bounds and duration parsing

- `MAX_DELAY_SECS = 270` (`executor.rs:676`), justified in-code as "must be less than default_timeout_secs (300s)" (`executor.rs:673-675`). Exceeding it ⇒ `InvalidDefinition`, i.e. run failure at execution time, not at parse time.
- `parse_duration_secs` (`executor.rs:705-735`): suffix `h` → `×3600` checked, `m` → `×60` checked, `s` → as-is, bare number → seconds; any other form ⇒ `InvalidDefinition`. Overflow ⇒ `InvalidDefinition("duration overflow: …")`. Fractional values (`"1.5h"`) are rejected.

---

### 11. Cron / interval matching (`run`, `lib.rs:428-672`)

| Rule | Detail | Line |
|---|---|---|
| Tick | `tokio::time::sleep(60 s)` **first**, then work — nothing fires in the first minute after startup | `lib.rs:430-432` |
| Scan | `list_all_enabled_workflows()` each tick (no community filter; each row carries its own `community_id`) | `lib.rs:436-442` |
| Definition parse failure | warn + skip that workflow | `lib.rs:449-459` |
| Disabled defs | `def.enabled == false` ⇒ skip | `lib.rs:461-463` |
| Missing channel | schedule workflow with `channel_id == None` ⇒ warn + skip ("Fix 2") | `lib.rs:465-472` |
| Trigger dispatch | `match &def.trigger` computing `(scheduled_for, trigger_type)`; cron arm `lib.rs:481-486`, interval arm `lib.rs:487-536` | `lib.rs:480-538` |
| Cron match | `cron_fire_instant(expr, now, 60, id)`: `schedule.after(now - 60s).next()` filtered to `<= now`; returns the **scheduled** instant so all pods agree; invalid expression ⇒ warn + `None` | `lib.rs:484`, `lib.rs:688-706` |
| Interval prefilter | in-memory `last_fired`, falling back to `latest_scheduled_workflow_fire`; DB read failure ⇒ `None` ⇒ suppress this tick (fail-closed) with a warning | `lib.rs:498-517` |
| Interval elapse test | `elapsed = (now - last).num_seconds().unsigned_abs() >= interval_secs`; `last = None` treated as `now` ⇒ no immediate fire | `lib.rs:753-774` |
| Cold-start seed | on the `None` suppress path the anchor is seeded to `now` so the next tick has a real anchor; an existing `Some` anchor is never advanced on suppress | `lib.rs:784-797` |
| Interval instant | `floor(now.timestamp() / interval) * interval` bucket boundary (`div_euclid`) so pods collide on one claim; zero/unparseable ⇒ warn + skip | `lib.rs:532`, `lib.rs:719-745` |
| Durable claim | `claim_scheduled_workflow_fire(community_id, workflow_id, scheduled_for)`; `Ok(None)` ⇒ another pod won, skip (interval anchors still advanced at `lib.rs:559`); `Err` ⇒ log + skip | `lib.rs:547-570` |
| Trigger-context serialization | built at `lib.rs:574-578`, serialization failure ⇒ error log + skip ("Fix 5") | `lib.rs:572-589` |
| Claim/run failure policy | if run creation fails after a won claim, the claim row is intentionally kept — at-most-once beats exactly-once | `lib.rs:590-611` |
| Anchor write ordering | `last_fired` updated only after claim **and** run insert succeed | `lib.rs:630-636` |
| Map hygiene | `last_fired.retain(...)` prunes entries whose `(community_id, id)` is no longer in the enabled scan ("Fix 1") | `lib.rs:664-670` |
| Non-schedule triggers | `_ => continue` — event triggers are never handled here | `lib.rs:537` |

---

### 12. Tenant scoping rules

Every DB call and side effect carries the server-resolved `CommunityId`: workflow lookup (`lib.rs:301-306`), run creation (`lib.rs:346`, `lib.rs:592`), status updates (`executor.rs:987`, `lib.rs:201`), `send_message` metadata lookups (`executor.rs:535-556`), and the sink call itself (`executor.rs:569-571`). The cron loop uses the workflow row's own `community_id` (`lib.rs:455`) rather than a deployment default. `last_fired` and `workflow_cache` are both keyed `(CommunityId, Uuid)` (`lib.rs:87`, `lib.rs:104`).


## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Business Rules

All rules below were read out of the code, not from docs. Where a doc comment and the code disagree, the delta is stated explicitly.

---

#### A. Startup ordering and preconditions (`main.rs:83-1060`)

`main()` is a single 977-line function. The order is load-bearing in several places (the code says so at `main.rs:176-183`, `main.rs:220-232`, `main.rs:265-269`, `main.rs:592-593`).

| # | Rule | Cite |
|---|------|------|
| **BR-RC-01** | The ring rustls `CryptoProvider` MUST be installed before any TLS use (`rediss://`, `wss://`, S3-over-TLS). Failure is a hard `expect` panic. | `main.rs:88-90` |
| **BR-RC-02** | The tracing subscriber is installed **after** `telemetry::try_init_tracer` returns but **before** any log is emitted; an exporter build failure is therefore logged only at `main.rs:118-120`, never inside `telemetry.rs` (documented `telemetry.rs:75-78`). | `main.rs:98-120` |
| **BR-RC-03** | Log filtering is `EnvFilter::from_default_env()` (i.e. `RUST_LOG`) **plus a forced `buzz_relay=info` directive**, so `buzz_relay` can never be filtered below `info` by env alone. | `main.rs:111` |
| **BR-RC-04** | Config load failure is fatal: `Config::from_env()` errors abort startup. | `main.rs:122-127` |
| **BR-RC-05** | The Prometheus exporter is installed **before** the DB connects, so a DB failure is still observable via `/metrics`. `metrics::install` panics if a recorder already exists or the port is taken. | `main.rs:138`, `metrics.rs:143-146` |
| **BR-RC-06** | Gauge idle-timeout MUST be at least 3× the usage-poll interval; the configured value is raised to `max(configured_or_900, interval*3)`. | `main.rs:1273-1277`, called `main.rs:136-138` |
| **BR-RC-07** | Usage-poll interval has a hard floor of 5 s (`.max(5)`) to prevent a busy loop; default 300 s. | `main.rs:1257-1262` |
| **BR-RC-08** | Postgres connect failure is fatal. Read-replica presence is logged but never required. | `main.rs:151-160` |
| **BR-RC-09** | Migrations run **only** when `BUZZ_AUTO_MIGRATE` ∈ {`true`,`1`,`yes`,`on`} (case-insensitive, trimmed). Opt-in, not opt-out. Migration failure is fatal. | `main.rs:25-33`, `main.rs:161-172` |
| **BR-RC-10** | Partition pre-creation (`ensure_future_partitions(3)`) failure is logged at `error` and **non-fatal** — the relay serves on with no future partitions. | `main.rs:173-176` |
| **BR-RC-11** | The replica freshness fence probe MUST start after the migration decision so the commit-time floor guard is verified against the *live* schema. Verification failure is loud but non-fatal: the fence stays closed and every cursor page routes to the writer. | `main.rs:177-198` |
| **BR-RC-12** | `require_relay_membership=true` requires a valid `RELAY_OWNER_PUBKEY`; otherwise abort. `config.rs:526-546` already warn-and-drops invalid values, so this catches the resulting `None`. | `main.rs:199-212` |
| **BR-RC-13** | `require_relay_membership=true` requires `BUZZ_RELAY_PRIVATE_KEY`; otherwise abort. Checked **before any DB mutation** (documented `main.rs:214-216`). | `main.rs:213-220` |
| **BR-RC-14** | The deployment's own community is seeded from `relay_url_authority(config.relay_url)` — the *same* normalization live request resolution uses — so the bootstrapped owner lands in the community requests actually resolve to. | `main.rs:238-240`, `tenant.rs:139`, `tenant.rs:80` |
| **BR-RC-15** | An empty derived authority is fatal **only when** `require_relay_membership=true`; otherwise it logs `error` and skips backfill/bootstrap. | `main.rs:241-256` |
| **BR-RC-16** | `ensure_configured_community` failure is fatal iff membership is enforced, else non-fatal error log. | `main.rs:247-263` |
| **BR-RC-17** | `pubkey_allowlist` → `relay_members` backfill MUST run **before** owner bootstrap, otherwise enabling membership locks every existing user out (documented `main.rs:265-269`). Failure fatal iff membership enforced. | `main.rs:270-289` |
| **BR-RC-18** | Owner bootstrap runs only when both a community and an owner pubkey exist. Failure fatal iff membership enforced. | `main.rs:290-311` |
| **BR-RC-19** | NIP-33 `d_tag` backfill runs unconditionally every startup and is always non-fatal (`error!` only). Idempotent. | `main.rs:312-320` |
| **BR-RC-20** | The audit service opens a **separate** Postgres pool (max 5, min 1) only when `audit_enabled`. Connect failure is fatal. | `main.rs:322-334` |
| **BR-RC-21** | Redis pool creation and `PubSubManager::new` failures are both fatal. Pool size = `config.redis_pool_size` (default 16; deadpool's own default of `CPU*2` is explicitly rejected as too small, `config.rs:61-65`). | `main.rs:335-348` |
| **BR-RC-22** | Search opens a **third** Postgres pool, preferring `read_database_url` when set (search is lag-tolerant). Connect failure is fatal. No pool sizing knobs are set — sqlx defaults apply. | `main.rs:369-386` |
| **BR-RC-23** | Relay signing keypair resolution, in strict priority: (a) `BUZZ_RELAY_PRIVATE_KEY` → parse or abort; (b) else if `!require_auth_token` → **hard-coded dev key `0x…01`** with a `warn`; (c) else `panic!`. | `main.rs:392-414` |
| **BR-RC-24** | Media config is validated (`config.media.validate()`) and storage constructed before `AppState`; both failures fatal. | `main.rs:415-422` |
| **BR-RC-25** | `AppState::new` spawns the audit worker eagerly even when audit is disabled; the disabled worker just awaits cancellation and returns (`state.rs:659-663`). `audit_tx` is `None` when disabled, so nothing can enqueue. | `state.rs:653-690`, `state.rs:763` |
| **BR-RC-26** | Mesh boot (`BUZZ_MESH` seam) returns `None` when off ⇒ **nothing bound, published or spawned**. When on, a bind/Redis failure is fatal (`?` at `main.rs:451`). Consumers are wired **before** the handle is published to `AppState.mesh`, so no peer can route traffic to an unwired dispatcher. | `main.rs:437-464` |
| **BR-RC-27** | The git object-store A3 conformance probe runs by default (opt-*out* via `BUZZ_GIT_CONFORMANCE_PROBE=false`) and its failure is **fatal** — a backend that cannot do linearizable conditional writes invalidates the manifest-pointer protocol. | `main.rs:466-503` |
| **BR-RC-28** | NIP-43 membership-snapshot reconciliation runs at startup **before the listener binds** and then periodically, only when `require_relay_membership=true`. Startup failure is `warn`-only. | `main.rs:505-546` |
| **BR-RC-29** | Channel-event reconciliation runs only when `BUZZ_RECONCILE_CHANNELS` is *present* (any value, including empty — `is_ok()`), is community-scoped via `bind_deployment_community`, and retries 24× at 5 s (≈2 min). An unmapped relay host skips it with a `warn`. | `main.rs:547-590` |
| **BR-RC-30** | The workflow action sink MUST be wired after `AppState` (which owns `sub_registry`/`conn_manager`) and **before** the cron loop starts. | `main.rs:592-600` |
| **BR-RC-31** | Both routers are built (`main.rs:939-940`) **before** the pool-metrics and usage-metrics pollers are spawned, and `serve()` is the last call. | `main.rs:939-1043` |

#### B. Tenancy resolution rules (`tenant.rs`)

| # | Rule | Cite |
|---|------|------|
| **BR-RC-32** | Row zero: `req.community = resolve_host(connection.host)`, bound at connection establishment, **before the WebSocket upgrade**, so no frame is ever read on an unbound connection. | `tenant.rs:1-9`, `router.rs:279-300` |
| **BR-RC-33** | The host is normalized exactly once, by `buzz_core::tenant::normalize_host`, before lookup — so `RELAY.EXAMPLE`, `relay.example.`, and `relay.example:443` all bind to one community (`tenant.rs:203-219`). Implementors of `HostResolver` MUST assume pre-normalized input (`tenant.rs:22-26`). | `tenant.rs:80` |
| **BR-RC-34** | An empty or whitespace-only host fails closed **before** the resolver is consulted, because the schema does not forbid a `host = ''` row in `communities`. The rejection reuses `UnmappedHost`, byte-identical to any other unmapped host, so a caller cannot probe for an empty-host row. | `tenant.rs:81-88` |
| **BR-RC-35** | Both "host maps to nothing" (`Ok(None)`) and "lookup failed" (`Err`) reject. There is deliberately **no** default/fallback community path. | `tenant.rs:89-95`, `tenant.rs:149-151` |
| **BR-RC-36** | The rejection surfaced to the caller is a single generic `404 "relay: no community is configured for this host"` that never echoes the host and never distinguishes unmapped from lookup error. | `router.rs:288-299` |
| **BR-RC-37** | The returned `TenantContext` carries the **normalized** host so downstream NIP-05 labelling, audit labelling, and the NIP-98 `u`-host check all see one canonical form. | `tenant.rs:74-77`, `tenant.rs:91` |
| **BR-RC-38** | Server-internal paths with no inbound `Host` (git Smart-HTTP, the localhost pre-receive hook, the workflow sink, startup tasks) resolve `config.relay_url`'s authority through the *same* fail-closed `bind_community`. Not a fallback: an unmapped `relay_url` errors identically. | `tenant.rs:120-133` |
| **BR-RC-39** | Background sweeps that cross communities MUST build a per-row `TenantContext` from the DB `RETURNING`, never a default tenant. Enforced by construction in the ephemeral reaper (`main.rs:637-644`) and the reminder scheduler (`main.rs:741-746`). | `main.rs:632-644`, `main.rs:707-746` |
| **BR-RC-40** | The relay's *own* deployment community is ensured with the same authority derivation used by live request resolution, so bootstrap and requests cannot diverge. | `main.rs:220-236` |
| **BR-RC-41** | Local-echo dedup keys include the community, because the same Nostr event id can legitimately exist in two communities; keying on the bare id would let a publish in A suppress a distinct event in B. | `state.rs:530-540`, `state.rs:821-824` |
| **BR-RC-42** | Cross-pod cache invalidation applies **only** the matching tenant-local drop; a mutation in A must not flush B's derived state. | `main.rs:856-867`, `state.rs:980-998` |
| **BR-RC-43** | Ban-driven disconnects are fenced to the banning community: the same pubkey live in A and B loses only A's sockets. | `state.rs:296-308`, `state.rs:328-334`, tests `state.rs:1734-1780` |
| **BR-RC-44** | Graceful-shutdown drain is deliberately **not** tenant-fenced — `drain_all` signals every connection across all communities. | `state.rs:336-366`, test `state.rs:1782-1836` |
| **BR-RC-45** | Community-scoped cache invalidation uses moka predicate invalidation; if invalidation closures are ever unavailable the code **over-invalidates** (full flush) rather than serving stale access state. | `state.rs:881-895`, `state.rs:928-975` |

#### C. Admission rules (`admission.rs`, `router.rs`, `state.rs`)

| # | Rule | Cite |
|---|------|------|
| **BR-RC-46** | Shared rate-limit admission is **fail-closed**: a limiter error (Redis unavailable) maps to `AdmissionError::Unavailable` and denies, after a `warn`. Confirmed. | `admission.rs:29-33`, test `admission.rs:141-158` |
| **BR-RC-47** | A denied counter returns `Exceeded { reset_in_secs }` taken from the limiter result, so callers can emit a retry hint. | `admission.rs:25-28` |
| **BR-RC-48** | WebSocket admission converts a per-second limit into a fixed 5-second window with 5× the budget, preserving the average rate while allowing desktop's burst of simultaneous subscriptions. Saturating multiply. | `admission.rs:6-10`, `admission.rs:39-44`, tests `:107-115` |
| **BR-RC-49** | Total concurrent connections are bounded by `conn_semaphore` = `config.max_connections` (default 10,000), acquired with `try_acquire_owned` (immediate refusal, no queuing) by both ordinary WS (`connection.rs:149`) and huddle audio (`audio/handler.rs:113`). | `state.rs:729`, `config.rs:449-452` |
| **BR-RC-50** | Concurrent message handlers are bounded by `handler_semaphore` = `config.max_concurrent_handlers` (default 1,024), also `try_acquire_owned`. | `state.rs:730`, `connection.rs:513/541/563` |
| **BR-RC-51** | A new WebSocket upgrade is refused with `503 "relay restarting"` once `shutting_down` is set, because readiness-503 only stops K8s routing and direct/in-flight upgrades still arrive during the pre-drain grace window. | `router.rs:305-313` |
| **BR-RC-52** | Inbound frame size is capped **at the parser**, before tungstenite assembles the message, by setting both `max_message_size` and `max_frame_size`; the application-level check in `recv_loop` remains as defence in depth. | `router.rs:324-332`, test `router.rs:504-518` |
| **BR-RC-53** | Slow-client backpressure: `grace_limit` consecutive full send buffers cancels the connection; any successful send resets the counter to 0. The counter is shared (`Arc<AtomicU8>`) between direct sends and fan-out, so a mixed pattern still trips the limit. Default grace 15 (`config.rs:470-473`). | `state.rs:451-478`, tests `state.rs:1447-1533` |
| **BR-RC-54** | Registration and drain are race-free by an insert-then-check / store-then-iterate pair: a connection that registers after the drain snapshot observes the sticky `draining` flag and self-signals its own 1012 close + cancel. The flag is one-way. | `state.rs:227-235`, `state.rs:353-356`, test `state.rs:1875-1930` |
| **BR-RC-55** | Long-lived-socket community admission ordering: register → durably revalidate → run. Archive-before-query is caught by the query; archive-after-registration is caught by the token. A guard deregisters on **every** exit path. | `state.rs:132-158`, tests `state.rs:1602-1683` |
| **BR-RC-56** | The periodic community revalidator only inspects communities that have **live local sockets** (`bound_communities`), never a global DB scan; a per-community lookup failure retains that community's sockets until the next tick and does not abort the sweep. | `state.rs:160-181`, `state.rs:1076-1087`, test `state.rs:1685-1730` |
| **BR-RC-57** | `check_ip_connection` (per-IP connection flood limiting) exists in `buzz-auth` (`crates/buzz-auth/src/rate_limit.rs:188`) and is implemented by `RedisRateLimiter` (`crates/buzz-pubsub/src/rate_limiter.rs:112`), but has **zero production callers** — the only other reference is the test stub at `admission.rs:85`. There is no IP-level admission rule in force. | verified by workspace grep |

#### D. NIP-11 advertisement rules (`nip11.rs`)

| # | Rule | Cite |
|---|------|------|
| **BR-RC-58** | `relay_self` (JSON `self`) is advertised iff the relay has a **stable** signing key (`relay_private_key.is_some()`). Ephemeral keys are excluded because they change on restart, leaving previously-signed NIP-29 group metadata unverifiable. | `nip11.rs:288-305` |
| **BR-RC-59** | NIP-43 is advertised iff stable key **AND** `require_relay_membership`. It is deliberately absent from the static `SUPPORTED_NIPS`; the desktop pairing probe keys off it, and advertising it on an open relay misroutes pairing peers to a non-existent `/pair` sidecar (`nip11.rs:475-484`). | `nip11.rs:17-21`, `nip11.rs:148-151`, `nip11.rs:304` |
| **BR-RC-60** | Advertising NIP-43 without `self` is a programmer error, caught by `debug_assert!` in debug/test builds only — a release build would ship the inconsistent document. | `nip11.rs:143-146`, test `nip11.rs:514-521` |
| **BR-RC-61** | `RelayInfo::build` MUST accept only static/scalar inputs. This is mechanically enforced: `_RELAY_INFO_BUILD_STATIC_INPUT_FENCE` (`nip11.rs:329-335`) is a `const` function pointer pinned to `build`'s exact signature — adding a `&Db`, `&AppState`, search, or audit handle breaks the build. | `nip11.rs:307-335` |
| **BR-RC-62** | NIP-11 is served **before** tenant binding and fails **open**: an unmapped host still gets the document with host-scoped fields (`icon`) simply absent, so the doc cannot leak which hosts are mapped. | `router.rs:275-286`, `nip11.rs:271-286` |
| **BR-RC-63** | The push descriptor is emitted iff `push_gateway_delivery_url.is_some()` **and** the request host binds to a community; the `origin` scheme is derived from whether `relay_url` starts with `wss://`. | `nip11.rs:183-233`, `nip11.rs:244-259`, test `nip11.rs:340-356` |
| **BR-RC-64** | `supported_nips` MUST be sorted — asserted by test (`nip11.rs:462-471`). |
| **BR-RC-65** | An empty-string icon is treated the same as absent and omitted from the JSON, so the document matches pre-icon output byte-for-byte. | `nip11.rs:154`, `nip11.rs:31-32`, test `nip11.rs:412-437` |
| **BR-RC-66** (delta) | `auth_required: true`, `restricted_writes: true`, and `due_delivery_mode: "push"` are **unconditional literals** (`nip11.rs:109/111/112`) — advertised even on a fully open relay and even when push delivery is disabled and no push worker was spawned (`main.rs:686-691`). |
| **BR-RC-67** (delta) | `max_limit: 10_000` (`nip11.rs:106`) is not enforced: `crates/buzz-db/src/event.rs:347` clamps a filter limit to `max_limit.unwrap_or(1000)`. |

#### E. Shutdown / graceful-drain rules (`main.rs:1113-1240`)

| # | Rule | Cite |
|---|------|------|
| **BR-RC-68** | The signal set is SIGTERM **or** Ctrl+C on unix; Ctrl+C only elsewhere. `signal(SIGTERM)` installation failure is an `expect` panic. | `main.rs:1226-1240` |
| **BR-RC-69** | Shutdown sequence: set `shutting_down=true` → readiness returns 503 → **sleep 5 s** so K8s stops routing → signal the listeners to drain → send a `1012 Service Restart` close frame to every live WebSocket → **sleep 30 s** → `std::process::exit(1)`. | `main.rs:1130-1154` |
| **BR-RC-70** | Every live WebSocket is explicitly closed with 1012 rather than left to ride the dying pod; without it a client would learn about the restart only from a TCP reset (or up to 60 s of stall-watchdog silence). | `state.rs:336-350`, `main.rs:1138-1149` |
| **BR-RC-71** | Close frames are delivered "queue on ctrl, then cancel": the send loop drains queued control frames before its `biased` cancel branch closes the socket. Best-effort — a full 1-slot control buffer still gets the cancel, just without the frame. | `state.rs:344-350`, `state.rs:357-362`, test `state.rs:1838-1873` |
| **BR-RC-72** | The health listener is spawned **without** graceful shutdown so `/_readiness` keeps answering 503 throughout the drain. | `main.rs:1120-1124` |
| **BR-RC-73** | The community revalidator is cancelled only *after* `serve()` returns, and only that loop — via its own dedicated `CancellationToken`, not the global flag. | `main.rs:1043-1045`, `state.rs:510` |
| **BR-RC-74** | The audit worker drain is signalled by a `CancellationToken` (not by dropping `Arc<AppState>`) so it works while background tasks still hold state clones; timeout 5 s. On cancel it closes the receiver, then drains everything already buffered. | `state.rs:664-681`, `state.rs:1182-1196`, `main.rs:1047-1051` |
| **BR-RC-75** | OTEL spans are flushed last, after the audit drain; a shutdown error is `warn`-only. | `main.rs:1053-1058` |
| **BR-RC-76** (delta) | The 30 s watchdog `std::process::exit(1)` (`main.rs:1153`) bypasses **both** BR-RC-74 and BR-RC-75 and exits non-zero. A shutdown that overruns the drain window therefore loses buffered audit entries and pending spans, and K8s records a failed termination. |
| **BR-RC-77** | On the UDS path, the UDS server task is `abort()`ed once the TCP server has drained; the function returns early and never falls through to the TCP-only branch. | `main.rs:1197-1205` |
| **BR-RC-78** | `BUZZ_UDS_PATH` pointing at an existing **non-socket** file is a fatal error; an existing socket is unlinked and rebound. | `main.rs:1163-1176` |
| **BR-RC-79** | On non-unix, a set `BUZZ_UDS_PATH` is a `warn` and silently ignored. | `main.rs:1208-1211` |

#### F. Metrics-emission rules (`main.rs:1279-1806`)

| # | Rule | Cite |
|---|------|------|
| **BR-RC-80** | The first usage tick is jittered by `rand::random::<u64>() % interval_secs` using true per-process randomness — PID-derived seeds are explicitly rejected because the relay is typically PID 1 in every pod. | `main.rs:1009-1017` |
| **BR-RC-81** | The usage interval uses `MissedTickBehavior::Skip` so a slow tick never schedules a catch-up burst. Same for the periodic-until-cancelled helper. | `main.rs:1021`, `main.rs:1089` |
| **BR-RC-82** | Exactly one pod owns the DB-derived snapshot, via a Postgres advisory lock (`USAGE_METRICS_LOCK_KEY`). Leadership is re-checked each tick with `is_live()`; a failed liveness check demotes and **skips** re-acquisition on that same tick (`!demoted` guard). | `main.rs:1417-1433` |
| **BR-RC-83** | All DB queries for a tick complete **before any gauge is set** ("collect then publish", C4). Any query error aborts the whole tick so a mixed fresh/stale snapshot is impossible. | `main.rs:1481-1516` |
| **BR-RC-84** | In-memory gauges are emitted by **every** pod (dashboard sums); DB gauges only by the leader (dashboard maxes). Documented `main.rs:989-993`. |
| **BR-RC-85** | Per-community series are zero-filled from the host map, not from query rows, so a community that drops to zero reports 0 instead of retaining its last non-zero value. Applied to users, channels, messages, relay members, workflows, git repos, active users, active channels. | `main.rs:1524-1806` |
| **BR-RC-86** | An unrecognised `channel_type` / `role` / `workflow status` row is **skipped with a `warn`**, not defaulted. | `main.rs:1573-1580`, `main.rs:1633-1640`, `main.rs:1676-1683` |
| **BR-RC-87** | A label key that disappears between ticks receives one final `0.0` before being forgotten; the key stores the resolved **host label** so a renamed/removed community can still be zeroed. | `main.rs:1316-1382`, test `main.rs:1861-1875` |
| **BR-RC-88** | Legacy lifecycle gauges (`buzz_ws_connections_active`, `buzz_subscriptions_active`) are kept "recent" for the exporter's idle-timeout policy by `increment(0.0)` rather than a snapshot `set()`, so a poller refresh cannot race the lifecycle inc/dec. | `main.rs:1306-1317`, tests `main.rs:1877-1927` |
| **BR-RC-89** | If the host-map query fails, the leader demotes, in-memory gauges are still emitted with `None` host map (totals only), and the tick returns `Err`. | `main.rs:1390-1403` |
| **BR-RC-90** | Fleet-wide `buzz_total_*` gauges always emit regardless of `EmissionScope`; only per-community series are gated. | `main.rs:44-47`, `main.rs:1506-1516` |
| **BR-RC-91** | The storage sweep has its own always-on config and a hard kill switch independent of `EmissionScope`; disabled means **no** storage-family gauge is touched, including health gauges, so a relay without `s3:ListBucket` can turn the feature off cleanly. | `main.rs:1436-1453` |
| **BR-RC-92** | An unknown `BUZZ_USAGE_METRICS_PER_COMMUNITY` value logs a `warn` and defaults to `all` (fail-open on cost, not on correctness). | `main.rs:57-73` |
| **BR-RC-93** | The pool-metrics interval is floored at 1 s because `tokio::time::interval` panics on `Duration::ZERO`. | `main.rs:945-949` |
| **BR-RC-94** | Replica-fence observability is reported as `buzz_db_replica_fence_open` 1/0 plus `..._lag_seconds`; a closed or stale fence reports `open=0` and leaves the lag gauge untouched. | `main.rs:967-978` |

#### G. Config validation rules (`config.rs`)

| # | Rule | Cite |
|---|------|------|
| **BR-RC-95** | `RELAY_OWNER_PUBKEY` must be exactly 64 hex chars; an invalid value is **warn-and-ignore** (lowercased, trimmed first). | `config.rs:526-546` |
| **BR-RC-96** | `RELAY_OPERATOR_PUBKEYS` entries must each be 64 hex chars; an invalid entry is a **hard config error** (asymmetric to BR-RC-95 by design — silently dropping an operator pubkey would silently disable provisioning). Duplicates are deduped, order preserved. | `config.rs:548-576` |
| **BR-RC-97** | A non-empty `RELAY_OPERATOR_PUBKEYS` **requires** `RELAY_OPERATOR_API_ORIGIN`; the origin must be `http`/`https` with a host and **no** credentials, path, query, or fragment. | `config.rs:577-582`, `config.rs:319-338` |
| **BR-RC-98** | Every `BUZZ_RATE_LIMIT_*` override must parse as a **positive** integer; zero and junk are hard errors (not silent fallbacks). | `config.rs:270-282`, tests `config.rs:1107-1121` |
| **BR-RC-99** | `BUZZ_PUSH_GATEWAY_DELIVERY_URL` must be an **exact** `https://…/v1/deliveries/apns` URL — no `http`, no trailing slash, no query, no fragment, no credentials. Explicit empty string disables push; **unset falls back to the hard-coded `https://push.buzz.xyz/v1/deliveries/apns`**, i.e. push is on by default. | `config.rs:340-360`, `config.rs:332`, `config.rs:752-758` |
| **BR-RC-100** | `BUZZ_PUSH_GATEWAY_TIMEOUT_MS` must be in `100..=10000`; out-of-range is a hard error, not a silent default. Unset ⇒ 2000 ms. | `config.rs:759-773` |
| **BR-RC-101** | `BUZZ_PUSH_EXECUTOR_KEY_ID` must be 1..=64 bytes. | `config.rs:745-751` |
| **BR-RC-102** | Policy Markdown documents are capped at 256 KiB each; `join_policy` is `None` unless at least one document or the age attestation is configured. Its `version` is `SHA-256(terms ‖ 0x00 ‖ privacy ‖ 0x00 ‖ age_flag)` — a content-derived id binding receipts to the exact revision. | `config.rs:775-810` |
| **BR-RC-103** | `BUZZ_ADMIN_HOST` must be an exact authority — rejecting any `/`, `\`, or `@`. `BUZZ_ADMIN_WEB_DIR` and `BUZZ_WEB_DIR` must each contain `index.html` or startup fails. | `config.rs:813-838`, `config.rs:850-859` |
| **BR-RC-104** | `BUZZ_GIT_HOOK_HMAC_SECRET`, when explicitly set, must be ≥32 chars. When unset a fresh random 32-byte hex secret is generated per process — so in a multi-pod deployment the hook secret differs per pod. | `config.rs:739-744`, `config.rs:862-871` |
| **BR-RC-105** | `BUZZ_PAIRING_RELAY_URL` must parse as `ws://` or `wss://` with a host; `https://` is rejected. | `config.rs:430-447`, test `config.rs:1287-1305` |
| **BR-RC-106** | `git_repo_path` and `git_pack_cache_path` are **created** (`create_dir_all`) at config load; an uncreatable path is a config error. `git_pack_cache_path` defaults to `<git_repo_path>/.pack-cache`. | `config.rs:377-398`, `config.rs:704-712` |
| **BR-RC-107** | Derived defaults chain: `git_max_repo_bytes = git_max_pack_bytes * 2`, `git_pack_cache_max_bytes = git_max_repo_bytes * 5` (saturating), so raising the pack limit silently raises both. | `config.rs:717-725` |
| **BR-RC-108** | `media_max_concurrent_uploads_per_pubkey` is clamped to `min(configured, media_max_concurrent_uploads)`. | `config.rs:669-675` |
| **BR-RC-109** | Boolean parsing is **inconsistent by design across three helpers**: `parse_bool` accepts `true/1/on` vs `false/0/off/""` and *errors* on anything else (`config.rs:363-377`); most flags use a bare `v == "true" \|\| v == "1"` and silently treat anything else as false (`config.rs:475`, `:479`, `:483`, `:520`, `:651`, `:848`); `huddle_audio_available` inverts (`!(v=="false"\|\|v=="0")`, `config.rs:489`); `BUZZ_MESH`/`BUZZ_MESH_DEMO_ECHO` accept `on` case-insensitively plus `true`/`1` (`config.rs:498`, `:516`); `BUZZ_REQUIRE_MEDIA_GET_AUTH` additionally accepts `yes`/`on` (`config.rs:682-689`); `BUZZ_AUTO_MIGRATE` accepts `true/1/yes/on` trimmed and case-insensitive (`main.rs:25-33`); `BUZZ_GIT_CONFORMANCE_PROBE` is `v != "false"` (`main.rs:469-471`); `BUZZ_RECONCILE_CHANNELS` is mere presence (`main.rs:549`). **Eight distinct boolean grammars.** |
| **BR-RC-110** | `!require_auth_token` emits a startup `warn` that REST bypasses token auth. `BUZZ_EPHEMERAL_TTL_OVERRIDE` emits a `warn` that it overrides client TTLs. | `config.rs:590-594`, `config.rs:696-702` |

#### H. CORS rules (`router.rs:409-432`)

| # | Rule | Cite |
|---|------|------|
| **BR-RC-111** | Empty `BUZZ_CORS_ORIGINS` ⇒ `CorsLayer::permissive()` — the **default**, since the var is unset by default (`config.rs:595-600`). | `router.rs:410-412` |
| **BR-RC-112** | If origins are configured but **none** parse as a `HeaderValue`, the layer refuses to fall back to permissive and returns a bare `CorsLayer::new()` (no CORS headers) after an `error!`. | `router.rs:419-426` |
| **BR-RC-113** | With valid origins the layer allows those origins with `Any` methods and `Any` headers — no credentials allowance, no method/header restriction. | `router.rs:428-431` |


## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Business Rules

Authoritative rule set for the WebSocket path. Every rule cites its enforcement site. Rules marked **⚠** are behaviours that deviate from NIP-01, from an in-repo doc, or from the sibling HTTP path.

---

#### A. Connection admission and lifecycle

| ID | Rule | Site |
|---|---|---|
| BR-WS-01 | The community is bound from the HTTP `Host` **before** the WebSocket upgrade. An unmapped host or a lookup error is rejected with a generic `404 relay: no community is configured for this host` — never a default tenant, and the host is never echoed back. | `router.rs:286-300` |
| BR-WS-02 | A socket is refused with `503 relay restarting` if `shutting_down` is set, even though readiness already returns 503 (direct dials and in-flight upgrades still arrive). | `router.rs:302-312` |
| BR-WS-03 | The parser's `max_message_size`/`max_frame_size` are set from `config.max_frame_bytes` before the handler exists; the app-level size check in `recv_loop` is defence in depth. | `router.rs:334-342`; `connection.rs:421`, `:440` |
| BR-WS-04 | Every socket registers in the community lifecycle registry, then the community's `is_community_active` is revalidated **after** registration. Archive-before-check is observed by the check; archive-after-registration is observed by the cancellation token. Either ordering closes the socket. | `connection.rs:132-140`; `state.rs:135-163` |
| BR-WS-05 | A connection slot is a `conn_semaphore` permit taken with `try_acquire_owned` — no queueing. Exhaustion logs and returns with **no** frame sent to the client. The permit is held for the whole connection and dropped last. | `connection.rs:149-155`, `:287` |
| BR-WS-06 | The AUTH challenge is sent as the very first frame, before the connection is registered in `ConnectionManager` and before `buzz_ws_connections_active` is incremented — so an immediate client disconnect leaks neither a registry entry nor a gauge count. | `connection.rs:182-212` (ordering comments `:194-196`, `:198`) |
| BR-WS-07 | A socket that has not reached `Authenticated` within **5 s** is cancelled. The timer starts at task spawn, is independent of client activity, and is cancelled by the shared token on normal close. | `connection.rs:27`, `:228-252` |
| BR-WS-08 | Heartbeat: `Ping` every 30 s on the **control** channel. The counter is pre-incremented, so the 3rd unanswered tick cancels. Any `Pong` resets it to 0. A full control channel is itself terminal (writer stalled). | `connection.rs:383-403`; reset `:461-463` |
| BR-WS-09 | Cleanup order on any exit: `cancel()` → await send+heartbeat+auth-timeout tasks → `sub_registry.remove_connection` + one `release_topic` per removed subscription → `conn_manager.deregister` → presence clear → gauge decrement → drop permit. | `connection.rs:263-287` |
| BR-WS-10 | Presence is cleared in Redis **only** when this was the last connection for that pubkey **in that community**. A session held in another community does not keep presence alive, and a second session in the same community does. | `connection.rs:274-287`; `state.rs:261-282` |
| BR-WS-11 | Graceful shutdown sends WS close code **1012 "relay restarting"** to every live socket regardless of tenant, and the drain flag is sticky so a registration that races past the drain snapshot self-signals. | `state.rs:352-374`, `:227-233` |

---

#### B. AUTH handshake sequence (NIP-42)

| ID | Rule | Site |
|---|---|---|
| BR-WS-12 | The handshake is relay-initiated: `["AUTH", challenge]` is pushed unsolicited; the client replies `["AUTH", <kind:22242 event>]`. `generate_challenge()` produces the nonce. | `connection.rs:157`, `:182-186`; `buzz-auth/src/nip42.rs:38` |
| BR-WS-13 | AUTH is processed **inline and awaited** in the recv loop (not spawned), so no two AUTH frames on one socket can interleave. | `connection.rs:503-509` |
| BR-WS-14 | AUTH is accepted only from `Pending`. From `Authenticated` → `OK false "auth-required: already authenticated"`. From `Failed` → `OK false "auth-required: authentication already failed"`. **There is no way to recover from `Failed`** — the socket is authentication-dead. | `auth.rs:45-74` |
| BR-WS-15 | The NIP-OA `auth` tag is extracted from the signed event **before** verification and is trusted only because the Schnorr signature covers it. Exactly one `auth` tag is required: zero → `None`; two or more → `None` (fail closed per NIP-OA). | `auth.rs:28-42`, extraction `:88-90` |
| BR-WS-16 | NIP-42 verification is pure crypto (challenge + expected relay URL); no DB, no tokens, no JWT. The expected relay URL is derived per-tenant. | `auth.rs:80-90`; `api/bridge.rs::nip42_expected_relay_url` |
| BR-WS-17 | **Gate order after verification is fixed**: (1) ban seam, (2) pubkey allowlist, (3) relay membership, (4) NIP-OA owner backfill/materialisation. The ban gate is first by design (MOD-7/M20: a ban must block connection auth even for open channels). | `auth.rs:105-275` |
| BR-WS-18 | Ban check is a **cascade**: the authenticated pubkey is checked first; only if clear is the cryptographically-proven NIP-OA owner checked. Owner ban ⇒ agent blocked; agent ban is agent-only. | `auth.rs:132-156` |
| BR-WS-19 | Ban lookups **fail closed but distinguish the cause**: `banned` → `blocked: you are banned from this community`; DB error → `error: internal error checking restriction state`. Both set `Failed` and cancel. | `auth.rs:113-183` |
| BR-WS-20 | The ban denial frame is queued on `ctrl_tx` (not `send_tx`) and *then* the token is cancelled — the send loop's cancel branch drains ctrl before emitting `Close`, so the client learns why. This "queue on ctrl, then cancel" idiom is load-bearing and regression-tested. | `auth.rs:173-182`; drain `connection.rs:334-345`; test `connection.rs:856-882` |
| BR-WS-21 | The pubkey allowlist applies **only** when `pubkey_allowlist_enabled` **and** `auth_method == Nip42`. A DB error is treated as "not allowed" (fail closed). | `auth.rs:189-212` |
| BR-WS-22 | Relay membership is enforced through the shared helper with NIP-OA owner-delegation fallback; failure → `restricted: not a relay member` + `Failed`. | `auth.rs:216-237` |
| BR-WS-23 | On an **open** relay (`require_relay_membership == false`) the NIP-OA owner is still extracted, unconditionally, purely for the agent→owner mapping the observer path needs. No feature flag gates this because the attestation is self-proving. | `auth.rs:239-253` |
| BR-WS-24 | `agent_owner_pubkey` is set on the `AuthContext` **only after** `materialize_nip_oa_owner` confirms the first-write-wins relationship; a failed materialisation logs a warning and leaves the field `None` (auth still succeeds). | `auth.rs:255-275` |
| BR-WS-25 | On success the pubkey is written into `ConnEntry.authenticated_pubkey` — the copy every fan-out access decision reads. `ConnectionState` never holds it. | `auth.rs:279-281`; reads `event.rs:146`, `:184`, `:460` |

##### Auth requirement matrix

| Message | Auth required | Behaviour when unauthenticated |
|---|---|---|
| `AUTH` | n/a | the only productive pre-auth message |
| `EVENT` | **yes** | `OK false "auth-required: not authenticated"` (`event.rs:667-674`) |
| `REQ` | **yes** | `NOTICE` + `CLOSED "auth-required: not authenticated"` (`req.rs:76-85`) |
| `COUNT` | **yes** | `CLOSED "auth-required: not authenticated"`, no NOTICE (`count.rs:42-48`) |
| `CLOSE` | **no** | fully processed pre-auth: mutates `conn.subscriptions`, calls `sub_registry.remove_subscription`, may call `release_topic`, and returns `CLOSED ""` (`connection.rs:581-583`; `close.rs:12-30`) |

BR-WS-26 **⚠** `CLOSE` is the one message an unauthenticated socket can execute end-to-end. It is harmless in effect (an unauthenticated socket owns no subscriptions, so `remove_subscription` returns `None` and no `release_topic` fires — `subscription.rs:162-178`) but it is unmetered, unauthenticated, and always answered — an unbounded free-work channel. See the security aspect.

---

#### C. Per-message admission (rate limiting)

| ID | Rule | Site |
|---|---|---|
| BR-WS-27 | Admission runs **after** parse and **before** dispatch, for every frame. | `connection.rs:498-500` |
| BR-WS-28 | Only `EVENT`, `REQ`, `COUNT` are metered. `AUTH` and `CLOSE` return `true` immediately. | `connection.rs:599-602` |
| BR-WS-29 | **Unauthenticated frames bypass admission entirely** — the limiter is pubkey-keyed, so a non-`Authenticated` state short-circuits to `true`. | `connection.rs:604-609` (`_ => return true`) |
| BR-WS-30 | Every metered frame consumes one `LimitType::WsEvents` token. The budget is a **5 s fixed window** of `human_ws_events_per_sec × 5` (default 10 → **50 per 5 s**), deliberately allowing a bounded burst for desktop startup. | `connection.rs:612-623`; `admission.rs:10`, `:44-49`; default `buzz-auth/src/rate_limit.rs:116-118` |
| BR-WS-31 | `EVENT` additionally consumes one `LimitType::Messages` token in a **60 s** window: `agent_standard_messages_per_min` (**120**) if `agent_owner_pubkey.is_some()`, else `human_messages_per_min` (**60**). | `connection.rs:632-650`; defaults `buzz-auth/src/rate_limit.rs:110-121` |
| BR-WS-32 | The agent tier is selected by `agent_owner_pubkey.is_some()`. Since that field is only set when NIP-OA materialisation succeeded (BR-WS-24), an agent whose materialisation failed is silently metered as a **human**. | `connection.rs:607`, `:633-637` |
| BR-WS-33 | Admission **fails closed**: a Redis error becomes `AdmissionError::Unavailable` → the frame is rejected with `rate-limited: shared admission unavailable`. | `admission.rs:34-40`; `connection.rs:670-677` |
| BR-WS-34 | Rejections are subscription-scoped only for `REQ` (`CLOSED`); `EVENT` and `COUNT` get a `NOTICE`. | `connection.rs:587-592`, `:624-627`, `:647` |
| BR-WS-35 **⚠** | A rate-limited `EVENT` therefore receives **no `OK` frame at all** — only a `NOTICE`. NIP-01 clients that key delivery confirmation on `OK` will hang on that event id. Same for a `COUNT` (no `COUNT` response, only a `NOTICE`). | `connection.rs:647` (passes `None`), `:566-569` |
| BR-WS-36 | Rate limits are community-scoped: the Redis key is `buzz:{community}:ratelimit:{pubkey}:{suffix}`, so the same key in two communities holds two independent quotas. | `buzz-auth/src/rate_limit.rs:153-156` |
| BR-WS-37 | A second, independent bound is the `handler_semaphore` (`max_concurrent_handlers`, default **1024**), taken with `try_acquire_owned` per EVENT/REQ/COUNT. Exhaustion → `rate-limited: too many concurrent requests`. `CLOSE` and `AUTH` do not take a permit. | `connection.rs:513-521`, `:541-550`, `:563-571`; default `config.rs:454-457` |
| BR-WS-38 | Observer frames (kind 24200) carry their **own** in-process limiter: 100/s per `(community, agent pubkey)`, fixed 1 s window. Control frames (owner→agent) are exempt so bursty telemetry cannot starve them. | `event.rs:917-939`, `:1032-1044` |

**Delta vs `ARCHITECTURE.md`.** §9 item 2 (`ARCHITECTURE.md:823`) states "**No rate limiting implementation** … none are enforced". That is false: `enforce_ws_admission` runs on every EVENT/REQ/COUNT (`connection.rs:498-500` → `:594-653`) against `state.admission_rate_limiter`, a `RedisRateLimiter` (`state.rs:712`, impl `buzz-pubsub/src/rate_limiter.rs`), using `RateLimitConfig` values that are env-overridable and validated non-zero (`config.rs:284-317`). What *is* true is narrower: the `IpConnections` tier is unused (see BR-WS-39).

| ID | Rule | Site |
|---|---|---|
| BR-WS-39 **⚠** | `LimitType::IpConnections` / `RateLimiter::check_ip_connection` are implemented (`buzz-pubsub/src/rate_limiter.rs:112`) but **never called** anywhere in the relay. There is no per-IP connection cap on the WS accept path. | `grep check_ip_connection` → only trait (`buzz-auth/src/rate_limit.rs:188`, `:234`), impl (`buzz-pubsub/src/rate_limiter.rs:112`), test stub (`admission.rs:85`) |

---

#### D. EVENT write path

| ID | Rule | Site |
|---|---|---|
| BR-WS-40 | Order of gates before any branch: auth → pubkey match → kind-22242 reject → observer route → ephemeral route → persistent ingest. | `event.rs:611-705` |
| BR-WS-41 | `event.pubkey` must equal the authenticated pubkey, **except** kind 1059 (gift wrap), whose sealed-sender design requires a mismatched author. | `event.rs:659-668` |
| BR-WS-42 | Kind 22242 (`KIND_AUTH`) can never be submitted via `EVENT`. | `event.rs:670-678` |
| BR-WS-43 | Scope enforcement is **empty-scopes-permissive**: `if !scopes.is_empty() && !scopes.contains(&MessagesWrite)`. A NIP-42 pubkey-only session has `scopes == []` and therefore passes every WS scope gate. | `event.rs:681`, `:676`; `req.rs:54` |
| BR-WS-44 | Kind 24200 observer frames are routed **before** the ephemeral branch even though 24200 is inside 20000–29999, so they never take the generic ephemeral path. | `event.rs:657-669` |
| BR-WS-45 | Ephemeral kinds (20000–29999) bypass the ingest pipeline entirely: verify → optional presence side-effect → optional membership check → `mark_local_event` → Redis publish → local fan-out → `OK true`. Never stored. | `event.rs:762-897` |
| BR-WS-46 | Ephemeral verification runs on `spawn_blocking`; a join error is reported as `error: internal error`, never as a validation failure. | `event.rs:771-791` |
| BR-WS-47 | Presence (kind 20001) accepts both a bare status string and legacy `{"status":…}` JSON; a bare string longer than 128 bytes is truncated on a UTF-8 char boundary. `"offline"` clears Redis presence; anything else sets it. | `event.rs:795-823` |
| BR-WS-48 | Presence then **falls through** to the ordinary channel-less ephemeral publish path so other nodes see the delta. | `event.rs:802-806`, `:843-871` |
| BR-WS-49 | A channel-scoped ephemeral event requires channel membership *before* publish; failure returns the membership error verbatim as `OK false`. | `event.rs:808-820` |
| BR-WS-50 | `mark_local_event` is always called **before** `publish_event`; if the publish fails the mark is immediately invalidated, so a failed publish cannot suppress a later legitimate Redis delivery of the same id. | `event.rs:847-861`, `:852-865`, `:1046-1058`, `:394-406` |
| BR-WS-51 | Channel-less ephemeral events publish to `EventTopic::Global`. The in-code "nil UUID sentinel" narrative at `event.rs:868-874` is **stale**: `EventTopic` is a two-variant enum (`Channel(Uuid)` / `Global`) and `publish_event` is called with `EventTopic::Global` (`event.rs:880`), with no `Uuid::nil()` anywhere on the path. | `event.rs:868-880` vs `buzz-pubsub` `EventTopic` |
| BR-WS-52 | Persistent events delegate to `super::ingest::ingest_event` with `IngestAuth::Nip42 { pubkey, scopes, channel_ids, conn_id }`; the WS handler only maps the outcome to `OK`. | `event.rs:698-735` |
| BR-WS-53 | `IngestError::Internal` is **sanitised** to `error: internal server error` before it reaches the wire; `Rejected`/`AuthFailed` messages pass through. | `event.rs:749-756` |
| BR-WS-54 | `OK true` means *durably accepted*, not *delivered*: `dispatch_persistent_event` awaits only the bounded audit enqueue, then spawns Redis publish + fan-out + workflow triggering and returns `0`. | `event.rs:311-371` (doc `:313-322`) |
| BR-WS-55 | The audit enqueue stays on the awaited path deliberately, using `send().await` on a capacity-1000 channel, so a saturated audit DB back-pressures the event handler rather than growing memory. | `event.rs:540-578` (rationale `:551-557`) |
| BR-WS-56 | The audit actor is the **caller-resolved actor hex**, not `event.pubkey` — for relay-signed events the claimed author is the relay key and deriving from the event would erase the human. | `event.rs:561-567`; regression test `event.rs:1761-1845` |
| BR-WS-57 | Workflow triggering is skipped for workflow-execution kinds, command kinds, relay-signed events tagged `buzz:workflow`, and kind 1059. | `event.rs:503-534` |

##### Observer frames (kind 24200)

| ID | Rule | Site |
|---|---|---|
| BR-WS-58 | Content must look NIP-44 encrypted, else `invalid: observer content must be NIP-44 encrypted`. | `event.rs:1095-1097` |
| BR-WS-59 | Exactly one each of `p`, `agent`, `frame` tags; zero or duplicates → `invalid`. | `event.rs:1117-1130` |
| BR-WS-60 | Direction is derived structurally: `pubkey == agent && p != agent` ⇒ Telemetry (owner = `p`); `p == agent && pubkey != agent` ⇒ Control (owner = `pubkey`); anything else rejected. | `event.rs:1082-1097` |
| BR-WS-61 | The `frame` tag value must match the direction's expected constant; a mismatch is **silently accepted** (`OK true`) and dropped, deliberately not signalled to the publisher. | `event.rs:1099-1102`, `:963-967` |
| BR-WS-62 | Timestamp must be within ±300 s of server time. | `event.rs:949-958` |
| BR-WS-63 | The publisher must be authorised for the `(agent, owner)` pair. Fast path: the session's verified `agent_owner_pubkey` equals the frame's owner (no DB read). Otherwise a 5-min `observer_owner_cache` lookup, then `db.is_agent_owner`. A DB error → `error: internal server error` (deny). | `event.rs:986-1030` |
| BR-WS-64 | Observer frames are routed as **global** ephemeral events (`EventTopic::Global`, `StoredEvent` with `channel_id = None`), never stored. Subscription-side gating is the REQ `#p` p-gate. | `event.rs:1069-1089` (doc `:915-919`) |

---

#### E. REQ read path

| ID | Rule | Site |
|---|---|---|
| BR-WS-65 | REQ is refused without `MessagesRead` (when scopes are non-empty) with a `NOTICE` **and** a `CLOSED`. | `req.rs:54-61` |
| BR-WS-66 | The 1024-subscription cap is checked against `conn.subscriptions` and only for a **new** `sub_id`; replacing an existing id is always allowed. | `req.rs:65-72` |
| BR-WS-67 | The accessible-channel set is resolved once per REQ from a 10 s cache, then narrowed by any scoped-token `channel_ids`. Every downstream consumer (search scope, registration, historical delivery, per-filter access) reads that one vector. | `req.rs:89-104`, `:105-107` |
| BR-WS-68 | Optimisation: a REQ whose every filter targets **only** kind 13534 (NIP-43 membership list) skips accessible-channel resolution entirely, because that kind is globally stored. | `req.rs:89-93`, `:829-840` |
| BR-WS-69 | Channel access for the subscription-level `#h` is confirmed **up front**, before the search branch, so a `#h` search against a just-joined channel is not scoped against a stale cache. | `req.rs:130-174` (rationale `:118-129`) |
| BR-WS-70 | Request-local access repair truth table (`resolve_request_local_access`): token denies → deny, no DB, no repair; cached hit → allow, no DB; cache-miss + DB member → allow **and push into the vector**; cache-miss + DB non-member → deny, vector untouched. The scoped-token bound is an absolute ceiling — a DB positive can never push a channel past a narrower token. | `req.rs:448-465` (doc `:423-447`); tests `:1299-1360` |
| BR-WS-71 | Denial for an inaccessible `#h` is `CLOSED "restricted: not a channel member"`. | `req.rs:167-171` |

##### The p-gate (the "kinds required" rule)

| ID | Rule | Site |
|---|---|---|
| BR-WS-72 | **The p-gate.** For a filter that *can* match a p-gated kind, the filter must carry a non-empty `#p` whose **every** value equals the authenticated pubkey. "Can match" is `filter.kinds.is_none_or(|ks| ks.iter().any(|k| P_GATED_KINDS.contains(k)))` — so **a filter that omits `kinds` always can match** and is therefore gated. This is the exact mechanism behind AGENTS.md's "relay queries must specify `kinds`". | `req.rs:1042-1074`; kind set `buzz-core/src/kind.rs:146-156` (24200, member-added, member-removed, 1059, 30622, 44200) |
| BR-WS-73 | **`ids` exemption**, with two carve-outs. A filter carrying a non-empty `ids` is exempt ("knowing the id implies authorisation") — *unless* the filter explicitly names `KIND_DM_VISIBILITY` (30622) or `KIND_AGENT_TURN_METRIC` (44200), which are relay-signed / metadata-leaking and keep the `#p` requirement. A **kindless** `ids` filter keeps the exemption. | `req.rs:1052-1068`; tests `:1454-1487`, `:1493-1546` |
| BR-WS-74 **⚠** | **On the WS REQ path the p-gate only runs when the subscription is global** (`channel_id.is_none()`). A REQ carrying a resolvable single `#h` skips all three sensitive-kind gates. | `req.rs:182-205` |
| BR-WS-75 | The p-gate's transport-specific outcome differs: WS → `CLOSED "restricted: p-gated events require #p matching your pubkey"`; HTTP `/query` and `/count` → **HTTP 403** `"restricted: p-gated kinds require #p tag matching your pubkey"`; WS COUNT → `CLOSED` with the *HTTP* wording. Three surfaces, two wordings. | `req.rs:184-189`; `api/bridge.rs:981-985`, `:1406-1410`; `count.rs:56-61` |
| BR-WS-76 | The engram gate (kind 30174): a filter that can match it needs `authors` all-self **or** `#p` all-self; non-empty `ids` is exempt; a kindless filter is gated. | `req.rs:1099-1135` |
| BR-WS-77 | The author-only gate (kinds in `AUTHOR_ONLY_KINDS` = event reminder + push lease, `buzz-core/src/kind.rs:120`): a filter targeting **only** author-only kinds must set `authors` all-self. Mixed-kind filters pass and rely on the per-event filter. | `req.rs:1249-1268`, per-event `:1186-1190` |
| BR-WS-78 | All three gates run **before** the NIP-50 search branch, deliberately, because search hits are fetched by id and skip the per-filter post-check. | `req.rs:175-205` (rationale `:175-181`) |

##### Search REQ

| ID | Rule | Site |
|---|---|---|
| BR-WS-79 | Mixing `search` and non-`search` filters in one REQ → `CLOSED "error: mixed search and non-search filters not supported"`. | `req.rs:211-219` |
| BR-WS-80 | A search REQ is **one-shot**: no `conn.subscriptions` entry, no registry entry, no `retain_topic`. Results then EOSE. | `req.rs:220-234` |
| BR-WS-81 **⚠** | Because the search branch returns before `subs.insert` (`:238`) and `register_scoped` (`:241`), **a search REQ reusing a live `sub_id` does not replace the existing subscription.** NIP-01 requires a REQ with an existing sub_id to overwrite it; here the old subscription keeps live-fanning-out under an id the client believes was replaced by a one-shot query. | `req.rs:212-234` vs `:236-247` |
| BR-WS-82 | A search REQ also never counts against the 1024 cap (the cap reads `conn.subscriptions`, which search never writes). | `req.rs:66` vs `:236-239` |
| BR-WS-83 | Search scope resolution: no accessible channels **and** no global access → immediate EOSE. Otherwise `ChannelsOrChannelLess` / `Channels` / `ChannelLessOnly` per the 4-case table. `include_global` is `token_channel_ids.is_none()` — i.e. a scoped token never broadens to channel-less rows. | `req.rs:483-501`, `:224`, `:520-525`; test `:1650-1666` |
| BR-WS-84 | A filter's `#h` values are intersected with accessible channels; if all are invalid or inaccessible the filter is **skipped** (matches nothing) rather than broadened. | `req.rs:568-583` |
| BR-WS-85 | Search paginates up to 10 pages of 100 hits per filter, always fetching full pages regardless of the requested limit, because post-filtering discards hits. `limit == 0` skips the filter (NIP-01). A short page terminates the loop. | `req.rs:421`, `:536-545`, `:586-600`, `:604-606`, `:725-727` |
| BR-WS-86 | Search results are post-filtered against **that filter only** (not the OR of all filters), then channel-access checked, then `reader_authorized_for_event`, then author-only, then deduped. | `req.rs:685-712` |

##### Subscription registration and historical delivery

| ID | Rule | Site |
|---|---|---|
| BR-WS-87 | Registration replaces same-`sub_id` (NIP-01). `register_scoped` returns the replaced entry's scope so exactly one `release_topic` is issued for it, then `retain_topic` for the new scope. | `req.rs:236-257`; `subscription.rs:69-160` |
| BR-WS-88 | Subscription scope is `Some(id)` only if **every** filter carries an `h` tag and all resolvable ids agree; a kindless-`h` filter or two distinct ids ⇒ global. | `req.rs:1013-1039` |
| BR-WS-89 | Historical delivery is **one DB query per filter**, not one merged query, so per-filter `limit` and time windows are respected (NIP-01 OR semantics). | `req.rs:261-330` (rationale `:261-265`) |
| BR-WS-90 | Queries run with bounded concurrency 4 via `buffered` (not `buffer_unordered`), so post-processing observes filters in original order — preserving dedupe order, conformance-trace order, and first-error-wins. Pinned by a test. | `req.rs:299-314`; test `:1271-1297` |
| BR-WS-91 | Per-filter channel scope for query construction prefers the filter's own single `#h` over the subscription scope, so cross-channel rows do not consume the LIMIT. | `req.rs:272-289` |
| BR-WS-92 | A logically global query gets `channel_ids = accessible_channels` pushed into SQL so `LIMIT` counts only visible rows; an explicit single-channel filter keeps its narrower `channel_id`. | `req.rs:989-998`; tests `:1236-1260` |
| BR-WS-93 | Per-row acceptance order in the delivery loop: `filters_match` (this filter only) → channel accessible → `reader_authorized_for_event` → not-author-only → dedupe. | `req.rs:336-407` |
| BR-WS-94 | **Dedupe happens after acceptance**, so an event rejected by filter A stays eligible for filter B. | `req.rs:389-392` |
| BR-WS-95 | Every 100 delivered events the task yields, so one large REQ cannot starve the runtime. | `req.rs:401-404` |
| BR-WS-96 **⚠** | A failed historical query sends `EOSE` and returns — **not** `CLOSED`. The subscription stays registered and live. The client sees a normal (empty) end-of-stored-events and cannot distinguish "no matches" from "DB error". | `req.rs:318-326` |
| BR-WS-97 **⚠** | If `conn.send` fails mid-delivery (buffer full), the handler returns **without EOSE**, leaving the client waiting on a subscription that is registered and live. | `req.rs:398-400`; search `:720-722` |
| BR-WS-98 | `limit` is clamped to `MAX_HISTORICAL_LIMIT` (2000) per filter; `kinds: []` produces `Some(vec![])`, which the DB layer treats as "no matching kinds". | `req.rs:879-882`, `:861-871` |
| BR-WS-99 | SQL pushdown set: kinds, single-author (`pubkey`) or multi-author (`authors`), `ids`, `since`, `until`, `#h`→`channel_id`, single `#p`, `#e` (any count), `#d` **only** when every kind is NIP-33 (30000–39999) because `d_tag` is NULL otherwise. Everything else post-filters. | `req.rs:860-991`; pushability predicate `:777-827`; test `:1594-1648` |

---

#### F. CLOSE

| ID | Rule | Site |
|---|---|---|
| BR-WS-100 | CLOSE removes from `conn.subscriptions` first, then deregisters from the fan-out index, then releases the Redis topic, and only then sends `CLOSED ""` — so no event is routed to the sub after the ack. | `close.rs:12-30` (ordering comment `:18-19`) |
| BR-WS-101 | An unknown `sub_id` still receives `CLOSED ""`; `remove_subscription` returning `None` simply skips `release_topic`. Idempotent, unauthenticated, and always answered. | `close.rs:20-27`; `subscription.rs:162-178` |
| BR-WS-102 | An empty-string `sub_id` is accepted for CLOSE (unlike REQ/COUNT, which reject it at parse). | `protocol.rs:152-162` vs `:89-93` |

---

#### G. Fan-out and delivery

| ID | Rule | Site |
|---|---|---|
| BR-WS-103 | **Registration is never sufficient for delivery.** Every fan-out path re-validates access on the sending pod, so a subscription surviving a membership change or an open→private flip cannot leak. | `event.rs:196-217` (doc), enforced `:224`, `:284`, `:409` |
| BR-WS-104 | `filter_fanout_by_access` applies four fences, in this order: (1) receiver-side tenant label — `community_for_conn(conn) == Some(event_community)`; (2) author-only kinds → author's connection only; (3) channel-less short-circuit (pass through); (4) private-channel membership. | `event.rs:115-195` |
| BR-WS-105 | Fence 1 is a hard cross-tenant fence at the send chokepoint, independent of the (already community-scoped) subscription indexes. | `event.rs:126-133`; regression test `:2436-2457` |
| BR-WS-106 | The author-only fence runs **before** the channel fence because author-only kinds are stored globally (`channel_id = None`) and would otherwise short-circuit past it. An unauthenticated recipient fails closed. | `event.rs:135-157`; test `:2211-2255` |
| BR-WS-107 | Channel visibility: only `"private"` triggers per-recipient membership filtering. A visibility lookup **error returns an empty recipient list** (fail closed). Unknown/unauthenticated recipients on a private channel are dropped; a membership lookup error drops that recipient only. | `event.rs:159-195` |
| BR-WS-108 | A threaded visibility value is honoured **only** when its `(community_id, channel_id)` exactly match this fan-out's; a mismatch or absence falls back to the fresh fail-closed lookup, never to "assume open". Membership checks always stay fresh. | `event.rs:161-180`; fence tests `:2259-2325` |
| BR-WS-109 | The visibility cache caches **only** `"private"`, never a non-private value — so the worst stale entry is over-restrictive, never a leak. | `state.rs:1124-1150` |
| BR-WS-110 | `dispatch_persistent_event_inner` layers a fifth fence for kinds 30622 / 44200: the recipient's authenticated pubkey must equal the event's `#p` value. | `event.rs:436-472` |
| BR-WS-111 | Frames are serialised once per event and cached per distinct `sub_id` within one fan-out cycle; the cache must not be reused across cycles (pinned by test). | `event.rs:63-98`; test `:1168-1188` |
| BR-WS-112 | Subscription scoping is **symmetric**: global subscriptions never receive channel-scoped events, and channel-scoped subscriptions never receive global events. Achieved structurally by index partitioning, not by a skip check. | `subscription.rs:265-330` (invariant comment `:320-327`); tests `:1005-1041`, `:1191-1227` |
| BR-WS-113 | Global fan-out consults the `(kind, #p)` index for **each** distinct `p` tag value on the event, then the generic kind index, then the global wildcard index; duplicates are suppressed by a `seen` set. | `subscription.rs:289-318`, `:369-387` |
| BR-WS-114 | A `kinds: []` subscription is never indexed and can never receive an event. | `subscription.rs:95-100`, `:415-418`; test `:952-1002` |

##### Local-echo dedup

| ID | Rule | Site |
|---|---|---|
| BR-WS-115 | The dedup key is `(community_id, event_id)`. A same-id event arriving for a **different** community is a distinct delivery and must not be suppressed. | `event.rs:295-304`; `state.rs:530-540`; test `:1560-1600` |
| BR-WS-116 | The check is **consume-on-read**: a hit invalidates the entry and returns, so only the first Redis echo per `(community, id)` is suppressed. | `event.rs:300-304` |
| BR-WS-117 | Entries expire after 60 s with a 10 000-entry cap, so the cache is bounded regardless of subscriber health. | `state.rs:734-739` |
| BR-WS-118 | Mark-before-publish and invalidate-on-publish-failure (BR-WS-50) is what keeps dedup from swallowing a real delivery. | `event.rs:394-406` |
| BR-WS-119 | The Redis consumer loop applies the same `filter_fanout_by_access` gate as the local path, then writes frames. Broadcast lag increments `buzz_multinode_fanout_lag_total`; a closed channel ends the loop. | `main.rs:818-845`; gate `event.rs:307` |

---

#### H. COUNT (NIP-45)

| ID | Rule | Site |
|---|---|---|
| BR-WS-120 | COUNT requires `Authenticated`; unauthenticated → `CLOSED "auth-required: not authenticated"`. | `count.rs:37-51` |
| BR-WS-121 **⚠** | COUNT does **not** check `MessagesRead` scope, unlike REQ (`req.rs:54`). A scoped session without read scope can still COUNT. | `count.rs:37-51` (no scope branch) |
| BR-WS-122 | COUNT applies the p-gate, engram gate, and author-only gate **unconditionally** — not only for global filters. It is therefore *stricter* than REQ (BR-WS-74). | `count.rs:53-76` |
| BR-WS-123 | Accessible channels are resolved and narrowed by the token exactly as in REQ; without the `retain` narrowing a scoped token could count outside its scope via the no-channel-filter pushdown. | `count.rs:78-97` (rationale `:90-94`) |
| BR-WS-124 | **Substance changed by `ab3af828`.** Per filter, the fast SQL `count_events` path is taken only when **all four** hold: `filter_fully_pushable`; author-only cannot match **or** `authors` is all-self; result-gated cannot match **or** `#p` is all-self; **and the filter cannot match kind 30175**. Otherwise a bounded candidate query + per-row post-filter. The 30175 condition has no self-authorized escape, so even `{kinds:[30175], authors:[self]}` loses the exact path (see DEBT-58 in `debt.md`). | `count.rs:105`, `:110`, `:117-118`; guards `:170-174`, `:239-243` |
| BR-WS-125 | The fallback fetches 5001 rows; > 5000 candidates → `CLOSED "restricted: count filter requires narrower constraints"` and `buzz_count_fallback_rejections_total`. COUNT must be exact, so a truncated count is never returned. | `req.rs:753-765`; `count.rs:185-195`, `:254-264` |
| BR-WS-126 | **Substance changed by `ab3af828`.** Fallback rows are counted only if they pass `filters_match` and then the single combined result gate `event_visible_to_reader` — which covers author-only kinds, the kind-30175 shared-gate, and `reader_authorized_for_event`. The three predicates are no longer inlined separately. | `count.rs:202`, `:271`; helper `req.rs:1222-1234` |
| BR-WS-126a | **Added by `ab3af828`.** For any 30175-matching filter the fallback's candidate query also carries the SQL visibility clause (`query.persona_reader`), so unshared foreign personas are excluded before `ORDER … LIMIT` rather than only after fetch. | `count.rs:162-163`, `:230-231`; clause `crates/buzz-db/src/event.rs:504-527` |
| BR-WS-127 | A filter targeting an inaccessible channel is `continue`d (contributes 0), not an error — so a partially-inaccessible COUNT silently returns a smaller number. | `count.rs:142-151` |
| BR-WS-128 | A filter with no `#h` gets `channel_ids = accessible_channels` and `limit = None` on the fast path. | `count.rs:221-228`, `:245` |
| BR-WS-129 **⚠** | COUNT DB errors are forwarded verbatim: `CLOSED "error: {e}"`. Unlike the EVENT path (BR-WS-53), there is no sanitisation — raw `sqlx`/Postgres text can reach an authenticated client. | `count.rs:179`, `:209`, `:249`, `:278` |

---

#### I. Backpressure and slow clients

| ID | Rule | Site |
|---|---|---|
| BR-WS-130 | All data sends are `try_send`, never awaited. A full buffer increments a shared strike counter; a successful send resets it to 0. At `grace_limit` (default **15**) strikes the connection is cancelled. | `connection.rs:88-113`; `state.rs:436-475` |
| BR-WS-131 | The strike counter is shared (`Arc<AtomicU8>`) between direct `ConnectionState::send` and fan-out `ConnectionManager::send_to*`, so mixed-path pressure accumulates on one counter. | `connection.rs:77`; `state.rs:54`, `:212`; test `state.rs:1343-1405` |
| BR-WS-132 | The send loop drains **all** pending control frames before touching data on every iteration, and the `select!` is `biased` (cancel > control > data). | `connection.rs:308-372` (`biased;` `:326`) |
| BR-WS-133 | On cancel the loop drains queued control frames **again** before emitting `Close`, which is what makes the "queue frame on ctrl, then cancel" idiom safe. | `connection.rs:328-345` |
| BR-WS-134 | Data frames are batched up to 64 per `flush()` and the batch size is recorded as a histogram. | `connection.rs:347-369` |
| BR-WS-135 | A full **control** channel is always terminal (client Ping, heartbeat Ping): it means the socket writer is completely stalled. | `connection.rs:396-400`, `:464-472` |
| BR-WS-136 | Live ban enforcement is cluster-wide: pod-local `disconnect_pubkey` (tenant-fenced) plus a fire-and-forget conn-control publish; the DB ban row is the durable backstop. | `state.rs:290-325`, `:508-556` |


## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: Business Rules

The ingest pipeline is one linear function, `ingest_event_inner`
(`ingest.rs:1453-2531`, **1 079 lines**), preceded by a tracing wrapper
`ingest_event` (`ingest.rs:1367-1425`). Rules below are numbered in **exact source
execution order**. Anything marked ⚠ is a verified deviation from the doc comment or from
the intent stated in the surrounding code.

---

### Phase A — envelope gates (kind-independent, pre-crypto)

| ID | Rule | Enforced at | Failure |
|---|---|---|---|
| **BR-IN-01** | kind 22242 (`KIND_AUTH`) may never be submitted through ingest. | `ingest.rs:1464-1467` | `Rejected("invalid: AUTH events cannot be submitted")` |
| **BR-IN-02** | kinds 44100 / 44101 (membership notifications) are relay-signed only. | `ingest.rs:1469-1473` | `Rejected("invalid: membership notifications are relay-signed only")` |
| **BR-IN-03** | Over HTTP, kinds 1059 (gift wrap) and 20001 (presence) are refused; they are WebSocket-only. | `ingest.rs:1475-1479` | `Rejected("invalid: kind {n} is only accepted via WebSocket")` |
| **BR-IN-04** | `is_relay_only_kind` (13534, 30622, 39005, 39006, 40901, 40902) is refused from any client. | `ingest.rs:1481-1483`; predicate `buzz-core/src/kind.rs:758-769` | `Rejected("restricted: relay-only kind")` |

⚠ BR-IN-01..04 all run **before** signature verification (BR-IN-05). An unauthenticated
malformed event can therefore be classified by kind without paying the Schnorr cost —
intentional, but it means these four rules provide no cryptographic assurance about the
claimed kind.

---

### Phase B — cryptographic and size gates

| ID | Rule | Enforced at | Failure |
|---|---|---|---|
| **BR-IN-05** | Event id and Schnorr signature must verify. Verification runs on `spawn_blocking`; the event is shared by `Arc` to avoid deep-cloning up to 256 KiB of content. | `ingest.rs:1486-1504`; `buzz_core::verification::verify_event` | `Rejected("invalid: {e}")`; task panic → `Internal("error: internal verification error")` |
| **BR-IN-06** | `\|created_at − server_now\| ≤ 900 s` (±15 min), `MAX_TIMESTAMP_DRIFT_SECS`. Applies to **all** kinds. | `ingest.rs:1506-1513` | `Rejected("invalid: event timestamp too far from server time")` |
| **BR-IN-07** | `content.len() ≤ 262 144` bytes (256 KiB), `MAX_EVENT_CONTENT_BYTES`. | `ingest.rs:1515-1523` | `Rejected("invalid: content exceeds maximum size of 262144 bytes (got n)")` |
| **BR-IN-08** | The event's signing pubkey must equal the authenticated principal — **except** for kind 1059 (NIP-59 gift wrap), which deliberately uses an unrelated ephemeral key. | `ingest.rs:1524-1529` | `AuthFailed("invalid: event pubkey does not match authenticated identity")` |

BR-IN-06 is a **symmetric** fence: events more than 15 minutes in the *past* are also
rejected, which makes back-dated import impossible through ingest. Two kinds tighten it
further to ±120 s: 28936 (`ingest.rs:1856-1867`) and 9040–9044 (in
`moderation_commands.rs:116`).

---

### Phase C — kind allowlist and scope

| ID | Rule | Enforced at | Failure |
|---|---|---|---|
| **BR-IN-09** | The kind must appear in `required_scope_for_kind`'s match — an **81-kind allowlist**. Unknown kinds (including 47 of the 127 `ALL_KINDS` entries) are refused. This is the real kind gate. | `ingest.rs:198-306`; raised `:1505-1508` | `Rejected("restricted: unknown event kind")` |
| **BR-IN-10** | kind 9002's required scope is *content-dependent*: `AdminChannels` if any `archived` tag is present, otherwise `ChannelsWrite`. | `ingest.rs:276-287` | — |
| **BR-IN-11** | Relay-admin kinds (9030–9033) may not be issued by a channel-scoped token. | `ingest.rs:1535-1544` | `AuthFailed("restricted: relay admin commands require a global token, not a channel-scoped token")` |
| **BR-IN-12** | kind 28936 may not be issued by a channel-scoped token. | `ingest.rs:1545-1550` | `AuthFailed("restricted: leave requests require a global token")` |
| **BR-IN-13** | The connection's scopes must contain the kind's required scope. | `ingest.rs:1551-1556` | `AuthFailed("restricted: insufficient scope (need {scope})")` |

⚠ **BR-IN-13 is unreachable in production.** Both transports construct `IngestAuth` with
`buzz_auth::Scope::all_known()` — WS at `buzz-auth/src/lib.rs:134-141`, HTTP at
`api/bridge.rs:829`. `Scope::all_known()` returns all 16 scopes
(`buzz-auth/src/scope.rs:68-87`). No production code path produces a reduced scope set, so
`!auth.scopes().contains(&required)` is always `false`. The 81-row scope column in the
dispatch table is documentation, not enforcement.

⚠ **BR-IN-11 and BR-IN-12 are also unreachable.** `AuthContext.channel_ids` is
hard-coded `None` on the WS auth path (`buzz-auth/src/lib.rs:138`), and
`IngestAuth::Http` has no `channel_ids` field, so `auth.channel_ids()`
(`ingest.rs:117-126`) never returns `Some`. The code's own doc comment admits this:
"In pure Nostr mode this always returns None" (`ingest.rs:113-115`).

---

### Phase D — early routing (bypasses generic storage)

Order is load-bearing. Each of these returns before the write-block gate or before storage.

| ID | Rule | Enforced at |
|---|---|---|
| **BR-IN-14** | Command kinds (30620, 41010–41012, 46020, 46030, 46031) are routed to `command_executor::handle_command`. The comment at `ingest.rs:1558-1559` pins the invariant: routed **after** signature, timestamp, pubkey-match, and scope — never before. | `ingest.rs:1560-1562` |
| **BR-IN-15** | kind 42000 (product feedback) is sidecarred to a private deployment table; it never enters `events` and never fans out. | `ingest.rs:1564-1584` |
| **BR-IN-16** | kind 1984 (NIP-56 report) is persisted only to `moderation_reports`; not stored publicly, never fanned out. Reports remain submittable while the author is timed out **and** in the missed-disconnect window after a ban, because they are non-actioning signals. | `ingest.rs:1586-1596` |
| **BR-IN-17** | Moderation commands 9040–9044 execute directly and are never stored or fanned out. Routed **before** the ban/timeout gate so a timed-out admin can lift a timeout; the handler re-checks the durable ban itself (`moderation_commands.rs:103-108`). | `ingest.rs:1598-1614` |

---

### Phase E — community restriction (ban / timeout write-block)

| ID | Rule | Enforced at | Failure |
|---|---|---|---|
| **BR-IN-18** | For every kind **except** 9040–9044 and 9030–9033, a durable `moderation_restriction_state` lookup runs. `banned = true` → refuse. | `ingest.rs:1639-1650` | `AuthFailed("blocked: you are banned from this community")` |
| **BR-IN-19** | `muted_until > now` → refuse the write (connection stays open so the client can render a countdown). | `ingest.rs:1651-1658` | `AuthFailed("restricted: you are timed out until {unix_ts}")` |
| **BR-IN-20** | A DB error in the restriction lookup **fails closed**. | `ingest.rs:1659-1667` | `Internal("error: internal error checking restriction state: {e}")` |
| **BR-IN-21** | The gate checks the *authoring* pubkey only — no NIP-OA owner→agent cascade. Documented as a deliberate Phase-1 asymmetry: ban cascades structurally at the auth seam (a banned owner's agent can never authenticate), timeout has no auth-seam presence and therefore does **not** cascade. | `ingest.rs:1602-1612` (rationale) | — |

This gate is one DB round-trip on **every** write. The code names the intended fix
(a restriction-state cache) at `ingest.rs:1609-1612`.

---

### Phase F — channel resolution and tenancy

| ID | Rule | Enforced at | Failure |
|---|---|---|---|
| **BR-IN-22** | kind 7: `channel_id` is derived from the **last** `e` tag's target event's `channel_id`, not from an `h` tag (`derive_reaction_channel`). Missing target → reject; missing/invalid `e` → reject; DB error → `Internal`. | `ingest.rs:331-377`, `:1644-1663` | `invalid: reaction target event not found` / `invalid: reaction must reference a target event via e tag` / `error: internal error looking up reaction target: {e}` |
| **BR-IN-23** | kind 1059: `channel_id` forced `None`. | `ingest.rs:1690-1691` | — |
| **BR-IN-24** | kind 5: `channel_id` derived from the **first** `e` tag's target's channel; absent target ⇒ `None` and deferred to BR-IN-33. | `ingest.rs:1692-1732` | `invalid: malformed deletion target id`; `error: looking up deletion target: {e}` |
| **BR-IN-25** | All other kinds: `channel_id` = first parseable UUID in an `h` tag. | `extract_channel_id` `ingest.rs:308-319`, called `:1707` |  — |
| **BR-IN-26** | `is_global_only_kind` (44 kinds, `ingest.rs:379-453`) forces `channel_id = None` regardless of any `h` tag. | `ingest.rs:1735-1737` | — |
| **BR-IN-27** | `requires_h_channel_scope` (23 kinds, `ingest.rs:455-491`) must yield a channel. | `ingest.rs:1739-1743` | `Rejected("invalid: channel-scoped events must include an h tag")` |
| **BR-IN-28** | The two sets are provably disjoint — asserted by an exhaustive test over all 65 536 kind values. | test `global_only_and_channel_scoped_are_disjoint` `ingest.rs:2779-2788` | — |
| **BR-IN-29** | Token-scoped channel access check (dead — see BR-IN-11 note). A channel-scoped token may not publish a global event. | `check_token_channel_access` `ingest.rs:525-532`; `:1719-1728` | `AuthFailed("restricted: channel-scoped tokens cannot publish global events")` |
| **BR-IN-30** | **Tenancy fence.** Every DB access in the pipeline is `tenant.community()`-scoped, resolved server-side from the request host — never from the event. Verified: `get_channel` `:1739`, `is_member_cached` (via `check_channel_membership` `:501`), `get_event_by_id` `:1690`/`:2245`, all inserts `:2298`/`:2371`/`:2385`/`:2394`. `claimed_community_from_event(&event)` is recorded in the trace **only as a claim**, never used as an authorization basis (`ingest.rs:1803-1808` documents this explicitly). | throughout | — |

⚠ **BR-IN-26 leaks through the read path.** The doc comment at `ingest.rs:369-377`
concedes it: the raw `h` tag stays on the signed event, and `filter.rs` treats explicit
`h` tags as authoritative, so a stray `h` on a global-only kind still matches `#h`
queries. Nulling `channel_id` fixes the write side only. Flagged in the source as
"a known limitation … should be addressed in the filter layer as a follow-up".

---

### Phase G — membership / authorization

| ID | Rule | Enforced at | Failure |
|---|---|---|---|
| **BR-IN-31** | If a channel is resolved, the author must be an active member **or** the channel's `visibility == "open"`. Uses the request's already-fetched `channel_row` to avoid a second SELECT. | `check_channel_membership` `ingest.rs:493-523`; called `:1785` | `Rejected("restricted: not a channel member")` |
| **BR-IN-31b** | A membership-lookup DB failure yields `error: database error: {e}` classified as **`Rejected`**, not `Internal`. ⚠ Inconsistent with BR-IN-20's fail-closed-as-`Internal` convention; over HTTP this surfaces as 400 instead of 500. | `ingest.rs:501`, mapped `:1802` | `Rejected("error: database error: …")` |
| **BR-IN-32** | Six kinds **skip** BR-IN-31 and rely on their own validator: 9021 (join needs no prior membership), 9007 (channel does not exist yet), 40003, 9002, 9005, 9008 (per-kind validators are the authority, so an agent's owning human can act without being a member — "OQ1 decision", `ingest.rs:1789-1796`). | `ingest.rs:1796-1801` | — |
| **BR-IN-32b** | A conformance `AuthCheck` verdict (`Allow`/`Deny`) is emitted at the call site, carrying the *claimed* community from the event alongside the server-resolved verdict basis. | `ingest.rs:1802-1827` | — |

---

### Phase H — kinds handled directly (post-authorization, pre-storage)

| ID | Rule | Enforced at |
|---|---|---|
| **BR-IN-33** | Relay-admin kinds 9030–9033 mutate `relay_members` and return; **never stored**. | `ingest.rs:1834-1844` |
| **BR-IN-34** | kind 28936 requires `require_relay_membership = true`, a ±120 s freshness window, and a NIP-70 `["-"]` tag. `remove_relay_member` handles NotFound / IsOwner atomically; the relay owner may not leave. Then two fire-and-forget publishes (8001 delta + 13534 snapshot), whose failures are `warn!`-logged only. **Never stored.** | `ingest.rs:1846-1928` |

---

### Phase I — per-kind pre-storage validators

| ID | Rule | Enforced at |
|---|---|---|
| **BR-IN-35** | `is_admin_kind` = `9000..=9022` → `validate_admin_event`. 9007 returns `Ok` immediately (no channel yet). All others must carry a resolvable `h` tag, and the channel must not be archived unless the event is a 9002 `archived=false` unarchive. | `side_effects.rs:26-28`, `:259-283`; called `ingest.rs:1931-1939` |
| **BR-IN-36** | 9000 (PUT_USER): `role` tag must parse. On **private** channels the actor must be an existing member, and only owner/admin may grant an elevated role. Self-add is always allowed. Third-party add consults the target's `channel_add_policy`: `owner_only` → only the registered owner; `nobody` → refuse; `anyone`/unknown → allow. ⚠ On **open** channels no relay-layer authorization runs at all — the only elevated-role gate is inside `buzz_db::channel::add_member` (`buzz-db/src/channel.rs:391-410`). | `side_effects.rs:284-372` |
| **BR-IN-37** | 9001 (REMOVE_USER): self-removal requires active membership and forbids removing the last owner. Removing others requires owner/admin, or member + NIP-OA ownership of the target. Non-members are refused outright — the code explicitly declines to check `is_agent_owner` for non-members ("you must be in the channel to remove anyone, even your own bot", `side_effects.rs:365-367`). | `side_effects.rs:373-409` |
| **BR-IN-38** | 9002 (EDIT_METADATA): ≥1 recognised tag from {name, about, archived, topic, purpose, visibility, ttl}. `archived` ∈ {true,false}; `name` non-empty post-canonicalisation; `visibility` ∈ {open,private}; `ttl` = `""` (clear) or a positive `i32`. Privileged set {name, about, archived, visibility, ttl} needs owner/admin **or** owner-of-any-active-owner-agent; {topic, purpose} needs only membership. | `side_effects.rs:410-624` |
| **BR-IN-39** | 9005 (DELETE_EVENT): optional `action_id` must be a UUID; `e` tag required; target must exist and belong to the h-tag channel (fail-closed on missing target — prevents an admin of A deleting events in B). Author self-delete path is re-gated on membership/open-visibility, and is **disabled** when moderation metadata (`action_id`/`reason_code`/`public_reason`) is present, forcing the owner/admin path. Fallback: owner/admin, or NIP-OA owner of the message's agent author. | `side_effects.rs:508-624`, `author_delete_can_use_self_delete_path` `:2353-2355` |
| **BR-IN-40** | 9008 (DELETE_GROUP): owner only, or owner-of-any-active-owner-agent. | `side_effects.rs:625-644` |
| **BR-IN-41** | 9022 (LEAVE_REQUEST): active member, not the last owner. | `side_effects.rs:645-663` |
| **BR-IN-42** | 9035/9036 (NIP-IA): `handle_identity_archive_event` verifies the consent path (self / admin-role / owner-via-live-kind:0) and mutates `archived_identities`, emitting the relay-signed 8002/8003 delta and 13535 snapshot. The request then **falls through to normal storage** so the delta's `["e", request_id]` resolves. | `ingest.rs:1941-1945` |
| **BR-IN-43** | kind 5: `validate_standard_deletion_event`. `a`-tag path checks the coordinate's pubkey equals the actor or the actor owns that agent. `e`-tag path resolves each target *including deleted* and requires the same. | `side_effects.rs:179-238`; called `ingest.rs:1947-1951` |
| **BR-IN-44** | If a channel was resolved, `channel.archived_at IS NOT NULL` refuses the write — unless the event is 9002 with an `archived=false` tag. | `ingest.rs:1953-1969` |
| **BR-IN-45** | kinds 5 and 9005: `e_count + a_count` must be **exactly 1**. `count_e_tags` counts *all* `e` tags including malformed ones, so a valid + malformed pair is rejected (regression-tested at `ingest.rs:3092-3107`). | `ingest.rs:1972-1984`, `count_e_tags` `:719-726` |
| **BR-IN-46** | kind 40003: `validate_edit_ownership` — target exists, same channel, actor is the target's *effective* author (relay-signed events resolve via `actor` then `p` tag) or the agent's owning human. Author path re-gates membership so a removed private-channel member cannot mutate old messages. | `ingest.rs:763-842`, `effective_message_author` `:729-761`; called `:1960-1964` |
| **BR-IN-47** | kind 45002: target must exist, be kind 45001 or 45003, and live in the same channel. | `ingest.rs:844-894`; called `:1966-1970` |
| **BR-IN-48** | kind 40008: content ≤ 61 440 B; `repo` tag required and `http(s)`; `commit` tag required and ≥7 hex chars; `parent-commit` ≥7 hex; `branch` needs both source and target; `pr` a positive integer. | `ingest.rs:896-963`; called `:1972-1974` |
| **BR-IN-49** | kind 30174: exactly one 64-**lowercase**-hex `d`, exactly one 64-lowercase-hex `p`. Uppercase is refused because readers query `#p` lowercase and Nostr tag matching is byte-exact — an uppercase head would win NIP-33 replacement then be invisible (`ingest.rs:1010-1013`). Content must be a plausible NIP-44 v2 payload. | `ingest.rs:965-1025`; called `:1976-1979` |
| **BR-IN-50** | NIP-44 v2 content shape: non-empty; standard base64 alphabet only; length a multiple of 4; padding only in the final two positions, ≤2 chars; decoded length ≥ 99 bytes; first decoded byte `0x02`. Envelope sanity only — the MAC is never checked at the relay. | `validate_engram_nip44_content` `ingest.rs:1084-1149` |
| **BR-IN-51** | kind 44200: exactly one 64-lc-hex `p`, exactly one 64-lc-hex `agent` equal to `event.pubkey`, **no** `h` tag (channel identity lives inside the ciphertext), NIP-44 v2 content. Then a DB check that `p` is the registered owner of `event.pubkey`. | `ingest.rs:1177-1247`; called `:1981-2016` |
| **BR-IN-52** | kind 30300: exactly one non-empty `d`; at most one `not_before`; `not_before` must be canonical decimal (no sign, whitespace, decimal point, or leading zero except `"0"`), ≤ 9 007 199 254 740 991, and ≤ `now + SPROUT_MAX_NOT_BEFORE_DELTA` (default 31 536 000 s). When both are present, `expiration > not_before`. A malformed `expiration` is ignored (NIP-40's concern). | `validate_not_before` `ingest.rs:1249-1276`, `validate_event_reminder` `:1252-1326`; called `:2018-2021` |
| **BR-IN-53** | kind 30175: exactly one `d`, non-empty, ≤64 chars, matching `^[a-z0-9][a-z0-9_-]{0,63}$`. Rationale: an empty `d` collapses every persona into the `(pubkey, 30175, "")` slot — LWW data loss (`ingest.rs:1022-1024`). | `ingest.rs:1027-1082`; called `:2023-2026` |
| **BR-IN-54** | kind 9007: `name` required and non-empty after `canonical_channel_name`; `visibility` (default `open`) and `channel_type` (default `stream`) must parse — validated for **all** 9007 events, with or without an `h` tag, so invalid enums are refused pre-storage. With an `h` tag, the channel is created eagerly via `create_channel_with_id`; `was_created == false` returns `accepted:false, "duplicate: channel already exists"`. | `ingest.rs:2029-2132` |
| **BR-IN-55** | kind 9021: an `h` tag is mandatory; the channel must exist and be `open`. | `ingest.rs:2160-2180` |
| **BR-IN-56** | kind 30350: `push_lease::accept` returns one of 7 outcomes; 6 map to distinct `Rejected` strings and `Accepted` returns immediately. Infrastructure failures map to `Internal`, never `invalid:` — regression-tested at `ingest.rs:2758-2777`. | `map_push_accept_error` `ingest.rs:186-195`; `:2156-2204` |
| **BR-IN-57** | Every `imeta` tag is validated structurally, then every referenced blob is verified to exist in storage with matching MIME / size / duration, plus thumbnails and poster frames. | `imeta.rs:11-335`; called `ingest.rs:2232-2244` |
| **BR-IN-58** | `requires_h_channel_scope` kinds resolve NIP-10 ancestry: `e root` / `e reply` markers (4-element tags only); parent must exist and be in the same channel; a parent with no channel is refused; the client's `root` must match the resolved ancestry; **depth ≤ 100**. | `resolve_nip10_thread_meta` `ingest.rs:564-717`; called `:2220-2231` |
| **BR-IN-59** | kind 0 content must parse as JSON, checked *before* storage so the profile-sync side effect cannot silently fail on a stored event. | `ingest.rs:2233-2239` |
| **BR-IN-60** | kind 7 emoji: `content` defaults to `"+"` when empty; `chars().count() ≤ 64` (mirrors the SDK's `check_emoji_len` so raw clients cannot bypass it). | `ingest.rs:2300-2318` |

---

### Phase J — persistence and replacement semantics

| ID | Rule | Enforced at |
|---|---|---|
| **BR-IN-61** | kind 7 uses a single transaction that upserts the reaction row (`ON CONFLICT` dedup) with `reaction_event_id` already set, then stores the kind:7 event. Ordering is load-bearing: an **active duplicate returns before** a duplicate kind:7 event is stored. Three outcomes: `TargetMissing` → reject, `Duplicate` → `accepted:false`, `Inserted` → continue. | `ingest.rs:2320-2351`; `buzz-db/src/event.rs:1242-1302` (fn head `:1242`, duplicate short-circuit `:1281-1285`, event insert `:1287`) |
| **BR-IN-62** | `is_replaceable` (0, 3, 41, 10000–19999) → `replace_addressable_event` with NIP-16 stale-write protection. | `ingest.rs:2393-2400` |
| **BR-IN-63** | `is_parameterized_replaceable` (30000–39999) → `d_tag` extracted, length ≤ `D_TAG_MAX_LEN` = 1024 (`buzz-db/src/event.rs:140`), then `replace_parameterized_event` keyed `(kind, pubkey, d_tag)`. | `ingest.rs:2401-2416` |
| **BR-IN-64** | Everything else → `insert_event_with_thread_metadata`. On DB failure, if a 9007 channel was pre-created it is compensated by `soft_delete_channel` + cache invalidation; `DbError::AuthEventRejected` maps to `invalid: AUTH events cannot be stored`, all else to `Internal`. | `ingest.rs:2417-2450` |
| **BR-IN-65** | **NIP-33/NIP-16 LWW conflict signal.** When the storage call reports `was_inserted == false` — the write lost the LWW race or the id already existed — ingest returns `accepted: true, message: "duplicate:"`. This is the relay-side conflict detection; `buzz-cli` converts it to `CliError::Conflict` → **exit code 5** (`buzz-cli/src/commands/mem.rs:105-108`, `commands/notes.rs:560-563`, `buzz-cli/src/error.rs:103`). | `ingest.rs:2452-2458` |
| **BR-IN-66** | Command kinds get their own LWW: for a NIP-33 `d_tag`, `persist_command_event` takes an FNV-1a-derived `pg_advisory_xact_lock` over `(community, kind, pubkey, d_tag)`, reads the current head, and treats `created_at < existing` or `created_at == existing && incoming_id >= existing_id` as **dominated** → `PersistResult::Duplicate`. Otherwise it soft-deletes the old row and inserts. | `command_executor.rs:135-224` |
| **BR-IN-67** | Command-event dedup: raw `INSERT … ON CONFLICT DO NOTHING`; `rows_affected() == 0` → `Duplicate`, transaction rolled back on drop. | `command_executor.rs:196-232` |

---

### Phase K — post-storage side effects

| ID | Rule | Enforced at |
|---|---|---|
| **BR-IN-68** | `is_side_effect_kind` gates `handle_side_effects`. Predicate: `0 \| 5 \| 9000..=9022 \| 30617 \| 10100 \| 41001..=41003 \| 40099`. | `side_effects.rs:35-37`; called `ingest.rs:2460-2467` |
| **BR-IN-69** | ⚠ **Side-effect failure does not fail the write.** `handle_side_effects` errors are `warn!`-logged and discarded; the event is already committed and is fanned out immediately afterwards. This is the module's core partial-failure semantic. | `ingest.rs:2460-2467`, then `:2513` |
| **BR-IN-70** | A freshly-inserted reply triggers a fire-and-forget relay-signed kind:39005 thread summary (`emit_live_thread_summary`), which **re-reads** counts from `thread_metadata` post-commit rather than incrementing, so the emitted summary is exact under concurrency. Fan-out only — never stored. | `ingest.rs:2469-2481`; `side_effects.rs:724-815` |
| **BR-IN-71** | kind 7 is deliberately **excluded** from `is_side_effect_kind` because dedup + DB writes already happened pre-storage (`side_effects.rs:31-34`). The old `handle_reaction()` is gone — noted at `side_effects.rs:1974-1976`. | `side_effects.rs:31-37` |
| **BR-IN-72** | The relay-signed system message (kind 40099) emitted by most NIP-29 side effects is best-effort: an insert failure is `warn!`-logged and fan-out proceeds. | `emit_system_message` `side_effects.rs:677-722` |
| **BR-IN-73** | Discovery events (39000/39001/39002) force `created_at > any existing event of the same (kind, pubkey, channel)` so two updates in the same second cannot lose to the random-event-id NIP-16 tiebreaker. Same guard in `publish_dm_visibility_snapshot`. | `emit_addressable_discovery_event` `side_effects.rs:902-960`; `:3092-3115` |
| **BR-IN-74** | 9002 `visibility` flips invalidate the accessible-channels and channel-visibility caches **before** any later event for that channel fans out, and an open→private flip eagerly evicts non-member subscriptions (with per-event fan-out membership re-checking as the cluster-wide backstop). | `side_effects.rs:1413-1447`; `evict_non_member_channel_subscriptions` `:95-121` |
| **BR-IN-75** | Deletions (kind 5 and 9005) look up thread metadata and call `soft_delete_event_and_update_thread`, which soft-deletes and **decrements** `reply_count`/`descendant_count` in one transaction, then emits a fresh 39005 so live badges count down. `deleted == false` short-circuits with no system message, to avoid false audit records. | `side_effects.rs:1615-1645`, `:2138-2168` |
| **BR-IN-76** | kind 5 targeting a kind:7 also removes the reaction row: first by `reaction_event_id`, falling back to `(target, actor, emoji)` derived from the reaction event itself. | `side_effects.rs:2170-2231` |
| **BR-IN-77** | kind 5 with no `e` tag at all routes to `handle_a_tag_deletion`. Routing keys on the **presence** of any `e` tag (not on decoded-target count) so a malformed `e` alongside an `a` can never silently soft-delete a coordinate. | `has_e_tag` `side_effects.rs:2300-2302`; `:2113-2118` |
| **BR-IN-78** | `a`-tag deletion dispatch: 30350 is ignored (revocation is a higher-generation replacement only); 30620 deletes the workflow by UUID or owner+name and invalidates the workflow cache; any other parameterized-replaceable kind is soft-deleted by coordinate; non-NIP-33 kinds are a no-op. | `side_effects.rs:1979-2106` |
| **BR-IN-79** | kind 30617: repo id `[a-zA-Z0-9._-]{1,64}`, no leading dot, no `..`. Same-owner re-announce is idempotent; a name held by another pubkey is a hard collision. Per-pubkey quota (`git_max_repos_per_pubkey`, default 100) is checked **before** claiming. Only a fresh `Reserved` claim may be rolled back on pointer failure; only a fresh claim emits the initial empty-refs kind:30618 (re-emitting would shadow real pushed refs under NIP-16). | `side_effects.rs:2391-2604` |
| **BR-IN-80** | 9001/9022 both re-check the last-owner guard inside their side-effect handler, duplicating the pre-storage check in `validate_admin_event`. | `side_effects.rs:1273-1287`, `:1919-1930` |
| **BR-IN-81** | 9021 join short-circuits when the actor is already a member, and fails closed on DB error rather than falling through to `add_member`. | `side_effects.rs:1856-1866` |
| **BR-IN-82** | Post-unarchive (9002 `archived=false`), a kind:44100 notification is re-emitted to **every** member purely as a resubscribe trigger, because archiving evicted their live subscriptions and unarchive otherwise emits no resubscribe signal. Documented known limitation: four sub-second archive/unarchive toggles by the same actor could collide event ids and skip a fan-out. | `side_effects.rs:1508-1546` |

---

### Phase L — command-executor rules

| ID | Rule | Enforced at |
|---|---|---|
| **BR-IN-83** | `handle_command` calls `ensure_user` first (FK requirement); failure is `warn!`-logged and execution continues. | `command_executor.rs:44-64` |
| **BR-IN-84** | 41010: 1–8 `p` tags; participant set = self + others deduplicated; then `open_dm`. A re-open of an existing DM clears the caller's `hidden_at` and republishes their NIP-DV snapshot. | `command_executor.rs:310-441` |
| **BR-IN-85** | 41011: caller must be a member; channel must be `channel_type == "dm"`; merged set ≤9; creates a **new** DM because DM participant sets are immutable. | `command_executor.rs:443-578` |
| **BR-IN-86** | 41012: caller must be a member of a `dm` channel; `hide_dm`; then republish the NIP-DV snapshot. | `command_executor.rs:580-651` |
| **BR-IN-87** | 30620: caller must be a channel member; the YAML must parse; an existing workflow at the same UUID must have the same owner **and** channel; the webhook secret is preserved across updates and returned only on first creation; the definition hash is computed **after** secret injection; the workflow's community is always `tenant.community()`, never re-derived from the client-supplied channel id, and the channel is re-verified inside that community. | `command_executor.rs:653-807` |
| **BR-IN-88** | 46020: the workflow lookup is community-scoped (a bare-id lookup could load another community's workflow and then satisfy membership against *its* colliding channel). Only the workflow **owner** may trigger — channel membership is explicitly insufficient because manual triggers run with owner authority. The trigger event is persisted under `workflow.channel_id` so workflow ids do not leak to unrelated relay members. | `command_executor.rs:809-959` |
| **BR-IN-89** | 46030/46031: approval must be `Pending` and unexpired; `check_approver_spec` accepts `""`/`"any"` (anyone) or an exact 64-hex pubkey (case-insensitive compare) and **fails closed** on role-based or unrecognised specs. `update_approval_by_stored_hash` returning `false` is treated as a lost race. Grant resumes the run at `step_index + 1`; deny cancels it. Both guard on `RunStatus::WaitingApproval` before acting. | `command_executor.rs:961-1234`, `resume_workflow_after_approval` `:1236-1327` |
| **BR-IN-90** | ⚠ Commands are **not atomic**. The doc header claims "validate → begin tx → insert event → execute mutations → commit" (`command_executor.rs:3-4`), but the implementation note at `:92-98` admits domain mutations execute on the pool, not in the transaction: "if a mutation succeeds but commit fails, the mutation persists without the event record". Safety rests on the mutations being idempotent. | `command_executor.rs:92-98` |

---

### Retention / TTL

There is **no retention or TTL enforcement on events** in this module. The only TTL is
per-channel: `resolve_ttl` (`handlers/mod.rs:42-62`) reads a `ttl` tag and lets
`BUZZ_EPHEMERAL_TTL_OVERRIDE` clobber it; the value is stored on the channel row at 9007
creation (`ingest.rs:2125`, `side_effects.rs:1681`) or updated by 9002
(`side_effects.rs:1449-1481`). Actual expiry is done by an out-of-module reaper
(`main.rs:646-680`). NIP-40 `expiration` tags are read only for the 30300 ordering check
(BR-IN-52) and are never acted upon at ingest.

---

### Conformance-trace coverage rule

**BR-IN-91**: `ingest_event` arms an `EmitGuard` (`ingest.rs:1407-1412`) whose `Drop`
records an `ImplBug` if the request emitted **no** trace record. Terminal errors are
mapped centrally (`ingest.rs:1436-1443`), and the success paths emit
`WriteInsert`/`WriteInsertGlobal`/`WriteDuplicate` at their dispatch points
(`:2353-2376`, `:2483-2513`, `:2215-2222`) plus an explicit
`emit_product_feedback_success` for 42000 (`:1572`, helper `:133-154`).

⚠ At least **six** success exit paths return `Ok` without any emit and without a prior
`AuthCheck` (which only fires for channel-bearing kinds): kind 1984
(`ingest.rs:1591-1595`), moderation commands (`:1609-1613`), relay-admin
(`:1838-1842`), 28936 (`:1924-1928`), the 9007 duplicate-channel branch (`:2143-2149`),
and all seven command kinds via `handle_command`. Under a recording tracer this is a
CoverageBreach. Production impact is nil because `AppState::tracer` is `NoopTracer`
(`state.rs:798`), so the guard is inert outside conformance tests.


## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Business Rules

Numbered BR-API-nn. Every rule cites the enforcing line.

---

#### A. Cross-cutting request pipeline

| ID | Rule | Enforcement | `file:line` |
|---|---|---|---|
| BR-API-01 | Every tenant-scoped HTTP door binds the community from the `Host` header **before** any auth or DB work; unmapped host or lookup failure fails closed with a generic 404 that never echoes the host. | `crate::tenant::bind_community` | `bridge.rs:626-633`, `:894-901`, `:1327-1334`, `:1783-1785`, `:2018-2025`; `invites.rs:200-207`; `media.rs:158-166`, `:480-487` |
| BR-API-02 | The NIP-98 `u`-tag expectation is built from `tenant.host()`, never `config.relay_url`'s host — closing both cross-host token reuse and false rejections on multi-tenant deployments. | `nip98_expected_url` | `bridge.rs:195-206`; tests `:2417-2449`, `:2636-2654` |
| BR-API-03 | For query-bearing NIP-98 GETs the expected URL appends the **verbatim** raw query string (not a re-serialized parse), so param order and encoding stay byte-exact with what the client signed. | `authorize_moderation_read`, `authorize_operator_request` | `bridge.rs:2027-2031`; `operator.rs:73-77`; tests `bridge.rs:2529-2630` |
| BR-API-04 | Every NIP-98 request is checked against a shared, community-scoped Redis seen-set (`SET NX EX`, TTL `DEFAULT_REPLAY_TTL_SECS`); a second presentation of the same event id is 401. | `check_nip98_replay` → `check_nip98_replay_with_guard` | `bridge.rs:136-176`; tests `:2262-2317`, `invites.rs:1103-1188` |
| BR-API-05 | A replay-guard **error** (Redis down) rejects the request — fail closed, never admit. | `Err(e) => 401 "replay check unavailable"` | `bridge.rs:167-176`; operator sibling `operator.rs:126-137`; no-infra test `bridge.rs:2340-2377` |
| BR-API-06 | Replay detection is **skipped** when the event id is the zero hash — the sentinel written by the `X-Pubkey` dev path. | early `return Ok(())` | `bridge.rs:150-153`, sentinel produced at `:122-125` |
| BR-API-07 | `X-Pubkey: <hex>` is accepted in place of a NIP-98 signature **only** when `require_auth_token == false`. | `if !require_auth_token` | `bridge.rs:118-127` |
| BR-API-08 | Bodies on the three bridge POSTs are capped at 1 MiB; media routes at `max(max_image, max_video)`; admin at 1024 bytes; git policy at 1 MiB. | `RequestBodyLimitLayer` | `router.rs:130`, `:33-46`; `admin/mod.rs:39`; `git/mod.rs:63` |

#### B. Rate limiting and admission

| ID | Rule | Enforcement | `file:line` |
|---|---|---|---|
| BR-API-09 | `POST /events`, `/query`, `/count` consume one `LimitType::ApiCalls` token per request against a **Redis** fixed 60 s window sized by `human_api_calls_per_min` (default 300). | `enforce_http_admission` → `admission::check_principal` | `bridge.rs:24-56`, called `:760`, `:955`, `:1386`; limiter `state.rs:712`; default `buzz-auth/src/rate_limit.rs:113-115` |
| BR-API-10 | Quota exceeded ⇒ **429** with `retry in {reset_in_secs}s`; limiter unavailable ⇒ **503** — i.e. the limiter fails **closed**. | `AdmissionError` match | `bridge.rs:39-55`; mapping `admission.rs:24-33` |
| BR-API-11 | Admission and replay run **before** body parse, so a 429/replay reject on a malformed body is still attributed. | ordering in `submit_event_authed` | `bridge.rs:758-770` |
| BR-API-12 | The three `/moderation/*` GETs have **no** admission check — they are the only NIP-98 bridge routes without a rate limit. | absence in `authorize_moderation_read` | `bridge.rs:2008-2054` |
| BR-API-13 | Media upload enforces a **process-local** fixed 60 s window of `media_uploads_per_minute` (default 30) per `(community, pubkey)`. | `upload_rate_limited` | `media.rs:88-111`, `:66`; called `media.rs:215` |
| BR-API-14 | Media upload enforces two concurrency ceilings: a global `Semaphore(media_max_concurrent_uploads)` (default 8) and a per-`(community,pubkey)` counter ≤ `media_max_concurrent_uploads_per_pubkey` (default 2, clamped to the global). | `acquire_upload_permit` | `media.rs:113-136`; state `state.rs:730`, `:781`; clamp `config.rs:669-675` |
| BR-API-15 | Both media limits are **per pod** (DashMap / in-process Semaphore), not cluster-wide — unlike the bridge's Redis limiter. | types | `state.rs:38-39`, `:523`, `:592`, `:600` |
| BR-API-16 | `/api/invites/claim` allows ≤10 attempts per `(community, pubkey)` per 60 s, evaluated **before** code verification; over-limit ⇒ 429. | `claim_rate_limited` → `claim_key_rate_limited` | `invites.rs:37`, `:293-298`, `:374-390`; test `:1194-1252` |
| BR-API-17 | The claim limiter is bounded by capacity (10 000) **and** TTL (60 s) because a pre-membership caller can mint keypairs freely — an unbounded map would itself be the DoS. | moka builder | `invites.rs:36-43`; `state.rs:775-780`; tests `invites.rs:459-503` |
| BR-API-18 | `POST /api/invites/accept-policy`, `GET /api/join-policy*`, and `GET/HEAD /media/*` have **no** rate limit at all. | absence | `invites.rs:75`, `:95`, `:104`, `:162`; `media.rs:604`, `:798` |

#### C. `POST /events` (ingest bridge)

| ID | Rule | Enforcement | `file:line` |
|---|---|---|---|
| BR-API-19 | Relay membership is enforced for the submitting pubkey, with a NIP-OA owner-delegation fallback read from `X-Auth-Tag`. | `enforce_relay_membership` | `bridge.rs:798-819`; gate `api/mod.rs:124-149` |
| BR-API-20 | On open relays (`require_relay_membership=false`) the membership check short-circuits, but a NIP-OA tag is still opportunistically parsed so the agent→owner edge can be materialized. The NIP-OA signature is self-proving, so no feature flag guards this. | `extract_nip_oa_owner` | `bridge.rs:806-813`; `api/mod.rs:151-170` |
| BR-API-21 | A verified agent→owner mapping is persisted first-write-wins; an existing mapping is accepted only if it names the same owner. Both principals are `ensure_user`'d first because the FK is community-scoped. | `materialize_nip_oa_owner` | `api/mod.rs:174-224` |
| BR-API-22 | HTTP ingest is granted `Scope::all_known()` — **all 16 scopes**, including `AdminChannels`/`AdminUsers`; channel authority comes from membership, not scopes. | `IngestAuth::Http` | `bridge.rs:827-831`; `buzz-auth/src/scope.rs:68-87` |
| BR-API-23 | `auth_method` is hardcoded `HttpAuthMethod::Nip98` regardless of whether the caller actually signed — the `X-Pubkey` path is indistinguishable downstream. | literal | `bridge.rs:830`; `DevPubkey` never constructed (`handlers/ingest.rs:58`) |
| BR-API-24 | `IngestError` maps: `Rejected`→400, `AuthFailed`→403, `Internal`→500; each increments `buzz_events_rejected_total{transport="http", reason=…}`. | match arms | `bridge.rs:842-871` |
| BR-API-25 | Rejection reasons can embed event-controlled text, so they are truncated to 256 bytes at a UTF-8 char boundary **before logging** — but the untruncated message is still returned in the HTTP body. | `truncate_reason` | `bridge.rs:595-611`, `:850-856`; tests `:3208-3241` |
| BR-API-26 | `serde_json` parse failures log only `category`/`line`/`column`, never the error's Display string (which embeds the offending input). | `ParseFail` variant | `bridge.rs:727-748` |
| BR-API-27 | Exactly one terminal `"HTTP bridge request"` log line is emitted per authenticated request, covering every outcome including early admission/replay/membership failures. | `SubmitOutcome` + thin wrapper | `bridge.rs:646-700`, `:704-747`; tests `:3586-3688` |

#### D. `POST /query` — read authorization

| ID | Rule | Enforcement | `file:line` |
|---|---|---|---|
| BR-API-28 | P-gated kinds (gift wraps, member notifications, observer frames) require the caller's own pubkey in `#p`; violation ⇒ **403**. | `p_gated_filters_authorized` | `bridge.rs:981-986`; `handlers/req.rs:1038` |
| BR-API-29 | Agent-engram reads require `authors=[self]` or `#p=[self]`; violation ⇒ 403. | `engram_filters_authorized` | `bridge.rs:987-992` |
| BR-API-30 | Author-only kinds require `authors=[self]`; violation ⇒ 403. | `author_only_filters_authorized` | `bridge.rs:993-998` |
| BR-API-31 | **The three gates above are applied unconditionally on HTTP.** The WS REQ path applies the identical three gates only when `channel_id.is_none()` (global subscriptions), because channel-scoped subs cannot receive globally-stored events. HTTP is therefore strictly stricter. | comparison | `bridge.rs:981-998`, `:1404-1421` vs `handlers/req.rs:183-205` |
| BR-API-32 | The caller's accessible-channel set is resolved once per request and every result is re-checked against it; channel-less (global) events are always allowed through. | `get_accessible_channel_ids_cached` + `event_in_accessible_channel` | `bridge.rs:1000-1005`, `:583-588` |
| BR-API-33 | Filters naming a single inaccessible `#h` channel are silently skipped (empty contribution), never an error. | `continue` | `bridge.rs:1141-1145`, `:1193-1196`, `:449-451` |
| BR-API-34 | Every delivered event passes `reader_authorized_for_event` — result-level read auth for viewer-private kinds (e.g. kind:30622 DM-visibility, kind:44200) even when reached via a kindless `ids` filter. | 4 call sites | `bridge.rs:1080-1083`, `:1170-1174`, `:1283-1288`, `:1601-1603`; test `:3125-3160` |
| BR-API-35 | Author-only events authored by someone else are dropped post-query. Since `ab3af828` the check is no longer a standalone `is_author_only_event` call on these paths — it is folded into `event_visible_to_reader`. | `event_visible_to_reader` (`handlers/req.rs:1222-1234`) | `bridge.rs:1295`, `:1753` |
| BR-API-36 | Filter dispatch precedence is fixed: channel-window (`top_level`) → feed (`feed_types`) → thread (`depth_limit` + one `#e`) → catch-all. A filter handled by an earlier stage is never re-processed. | `handled: HashSet<usize>` | `bridge.rs:1017-1027`, `:1029`, `:1112`, `:1188` |
| BR-API-37 | Catch-all filters are built and validated in filter order (phase 1) **before** any DB read, so deterministic client errors surface ahead of transient DB errors; DB reads then run bounded-concurrent (`FILTER_QUERY_CONCURRENCY=4`) while post-processing preserves filter order. | 3-phase pipeline | `bridge.rs:1186-1264`; `handlers/req.rs:35` |
| BR-API-38 | `before_id` must be 64 hex chars — malformed ⇒ **400**, never demoted to "absent". It also requires `until` to be set ⇒ 400 otherwise. | `BeforeId::Malformed` | `bridge.rs:273-291`, `:1198-1216`; tests `:2940-2977` |
| BR-API-39 | Extension flags opt in only on a literal JSON `true`; `"true"`, `1`, and absent all read false, so a malformed filter degrades to a normal query rather than a wrong window. | `extension_flag` | `bridge.rs:295-297`; test `:2979-3000` |

##### Channel-window sub-rules (`top_level: true`)

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-40 | Requires exactly one `#h` channel ⇒ else 400. | `bridge.rs:414-419` |
| BR-API-41 | Cursor is composite: `until` **and** `before_id` together, or neither ⇒ else 400. No timestamp-only fallback (that ambiguity is the dense-second dup/loss bug). | `bridge.rs:436-458` |
| BR-API-42 | `until` out of `DateTime` range ⇒ 400. | `bridge.rs:441-444` |
| BR-API-43 | Row budget = `min(limit, 200)`, default 50, floor 1. Summary/bounds overlays and the aux closure do **not** consume it. | `bridge.rs:374-375`, `:460-465` |
| BR-API-44 | Aux closure walks exactly two hops: reactions/deletions/edits targeting retained rows, then deletions targeting those aux events. Each hop bounded at 1000 rows; duplicates deduped by event id. | `bridge.rs:379-390`, `:483-521` |
| BR-API-45 | Aux events are access-**checked** rather than channel-constrained, because deletions can be stored channel-less. | `bridge.rs:504-508` |
| BR-API-46 | Exactly one kind:39006 window-bounds overlay per response is the sole authority on exhaustion — `rows < limit` proves nothing on an exact-multiple final page. | `bridge.rs:558-576` |
| BR-API-47 | Overlays (39005, 39006) and synthesized presence (20001) are signed with `state.relay_keypair`. | `bridge.rs:523-527`, `:1972` |

##### Feed / thread / presence sub-rules

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-48 | Feed limit is `min(limit, 100)` default 20, shared across all requested feed types; `agent_activity` canonicalizes to `activity`; duplicate types deduped; unknown types silently ignored. | `bridge.rs:1035-1064` |
| BR-API-49 | Thread reads require exactly one `#e` value that decodes to 32 bytes; limit `min(limit, 500)` default 100. | `bridge.rs:1126-1140`, `:1147-1151` |
| BR-API-50 | Thread cursor decodes to BE-i64 seconds optionally followed by raw event-id bytes; a malformed id hex degrades to timestamp-only rather than erroring. | `bridge.rs:305-345`; test `:3072-3082` |
| BR-API-51 | If **every** filter targets only kind:20001 or kind:40902 **with** non-empty `authors`, the request is served entirely from Redis presence and never touches Postgres. Any other filter shape falls through. | `synthesize_presence` | `bridge.rs:1920-1985` |
| BR-API-52 | Presence lookup failure degrades to an empty result (`unwrap_or_default`), not an error. | `bridge.rs:1969-1975` |

##### NIP-50 search routing on `/query`

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-53 | Any filter with `search` routes the whole request to `buzz-search` (Postgres FTS); mixing search and non-search filters in one request ⇒ **400**. | `bridge.rs:1006-1021`, `:1578-1580` |
| BR-API-54 | The three read gates (BR-API-28/29/30) run **before** the search branch, so search cannot be used to harvest p-gated or engram kinds. | `bridge.rs:981-998` then `:1006` |
| BR-API-55 | Channel scope: intersect the filter's `#h` values with accessible channels — all-inaccessible ⇒ skip that filter; no `#h` ⇒ community-wide scope with `include_global = true`; no scope at all ⇒ return `[]`. | `bridge.rs:1622-1650` |
| BR-API-56 | Only kind/authors/since/until are pushed down to FTS; **every** other filter constraint (`#p`, `#h`, `#e`, `#d`, `ids`, …) is re-enforced against the full stored event by `search_hit_accepted`, plus channel scope and `reader_authorized_for_event`. | `bridge.rs:1601-1618`, `:1652-1671`, `:1717-1719`; tests `:2842-2934`, `:3125-3160` |
| BR-API-57 | Per-filter FTS page size is `min(limit, 500)`; `limit == 0` skips the filter; `page` defaults to 1 and must be > 0. | `bridge.rs:1664-1668`, `:380-388` |
| BR-API-58 | FTS relevance ordering is preserved by iterating hit ids and looking each up in a map built from the DB fetch; results are deduped across filters. | `bridge.rs:1700-1735` |
| BR-API-59 | On non-search queries, `page` (1-based) becomes SQL `OFFSET = (page-1)*limit`, but only when a positive `limit` is present; page ≤ 1 leaves the default offset untouched. | `bridge.rs:390-410`, `:1218-1229`; tests `:3002-3030` |

#### E. `POST /count` semantics

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-60 | The same three read gates apply, unconditionally, as on `/query`. | `bridge.rs:1404-1421` |
| BR-API-61 | Filters naming an inaccessible `#h` channel are skipped (contribute 0), not an error. | `bridge.rs:1454-1457` |
| BR-API-62 | With no `#h`, `query.channel_ids` is set to the accessible set so the scope is pushed into SQL, and `limit` is cleared for the pushdown path. | `bridge.rs:1524-1527`, `:1544` |
| BR-API-63 | SQL `COUNT(*)` pushdown is used only when **all three** hold: `filter_fully_pushable`, author-only filtering unnecessary or `authors == [self]`, and result-gated filtering unnecessary. | `bridge.rs:1467-1479`, `:1535-1543` |
| BR-API-64 | Result-gated kinds (44200 / 30622) force the per-event fallback unless `#p=[self]` is safely pushed down — otherwise the count itself leaks existence. | `filter_can_match_result_gated_kinds` + `result_gated_count_safe_for_pushdown` | `bridge.rs:1436-1443` |
| BR-API-65 | The fallback path bounds its candidate set (`apply_count_fallback_limit`); if the candidate set is exceeded, the request is **rejected 400** `"count filter requires narrower constraints"` and `buzz_count_fallback_rejections_total` is incremented — the count is never silently truncated. | `bridge.rs:1489-1497`, `:1551-1559` |
| BR-API-66 | **Substance changed by `ab3af828`.** Fallback counting applies `filters_match` and then the single combined result gate `event_visible_to_reader` (author-only + kind-30175 shared-gate + `reader_authorized_for_event`) per candidate before incrementing. The three predicates were previously inlined separately. | `bridge.rs:1503-1508`, `:1569-1574`; helper `handlers/req.rs:1222-1234` |
| BR-API-66a | **Added by `ab3af828`.** `POST /count` bypasses the fast SQL pushdown whenever a filter can match kind 30175, and sets the SQL persona-visibility pre-filter on both fallback branches. | `bridge.rs:1447-1448`; guards `:1477`, `:1543`; pushdown `:1465-1466`, `:1530-1531` |
| BR-API-67 | Response is the summed total across all filters — **not** deduplicated across overlapping filters, so an event matching two filters is counted twice. | `bridge.rs:1432`, `:1575` |

#### F. Media upload / download

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-68 | Tenant binding runs **first** in the extractor, before Blossom verification, so the `server` tag is validated against the bound tenant host rather than a process-global domain — and it reads only the `Host` header, preserving the pre-body rejection guarantee. | `media.rs:145-167` |
| BR-API-69 | All upload authorization happens in `FromRequestParts`, i.e. before the body is buffered, so an unauthenticated caller cannot force the server to buffer up to the body limit. | `media.rs:29-38`, `:140-234` |
| BR-API-70 | Only `/upload` and `/media/upload` are valid upload paths; anything else ⇒ `MediaError::NotFound`. | `media.rs:57-63` |
| BR-API-71 | `X-SHA-256` is **mandatory** (BUD-11) and must be exactly 64 lowercase hex chars. | `media.rs:169-186` |
| BR-API-72 | `X-SHA-256` must match at least one `x` tag in the Blossom auth event, else `HashMismatch` (→401). | `media.rs:188-194` |
| BR-API-73 | Relay membership is the **only** upload authority — independent of bearer tokens and of `require_auth_token`. On open relays any valid Blossom signer may upload. | `media.rs:196-219` |
| BR-API-74 | The extractor verifies Blossom auth with the permissive 3600 s window because the content type is not yet known; `buzz-media` re-verifies with the per-type window (600 s images / 3600 s video) after the SHA-256 is computed. | `media.rs:171-177` |
| BR-API-75 | Content type is decided from **actual bytes** (a bounded 4096-byte sniff, replayed into the pipeline so the hashed body stays byte-identical), never from `Content-Type`. | `media.rs:319-336` |
| BR-API-76 | Video takes a streaming-to-disk path (never fully buffered); non-video buffers up to `max(max_image_bytes, max_file_bytes)` and then splits image vs generic-file by sniffed MIME. | `media.rs:338-400` |
| BR-API-77 | The legacy `/media/upload` alias accepts images only — any other sniffed type ⇒ **415** `DisallowedContentType`; it also increments `buzz_media_legacy_upload_route_total`. | `media.rs:313-317`, `:384-388` |
| BR-API-78 | Descriptor URLs are rewritten to the bound tenant host before returning, with the extension validated by `is_safe_ext` and defaulted to `bin`. | `media.rs:402-407`, `:458-476` |
| BR-API-79 | Upload MIME metric labels are normalized to a 6-value allowlist to bound cardinality. | `media.rs:410-424` |
| BR-API-80 | A `MediaUploaded` audit entry is sent over the bounded audit channel; a closed channel logs and increments `buzz_audit_send_errors_total` but does **not** fail the upload. | `media.rs:426-445` |
| BR-API-81 | Read auth (`GET`/`HEAD`) is applied only when `require_media_get_auth == true`: Blossom `get` verb + `x`-tag hash binding + `server` = tenant host + relay membership. Default is **off**. | `media.rs:489-514`; `config.rs:682-689` |
| BR-API-82 | Path validation runs before any auth or storage lookup and rejects anything that is not `{64-hex}[.ext][.thumb.jpg]`; traversal, uppercase hashes, `_uploads/`/`_meta/` keys, and >3 segments are all 404. | `media.rs:547-583`; tests `:1163-1272` |
| BR-API-83 | The sidecar is consulted **before** any blob I/O and is authoritative for content type and extension; storage metadata is never trusted. A missing sidecar is 404, not a metadata fallback. | `media.rs:629-660`, `:794-836` |
| BR-API-84 | For explicit `{hash}.{ext}` requests, `requested_ext` must equal `sidecar.ext` or the response is 404. | `media.rs:646-658`, `:820-834` |
| BR-API-85 | Non-inline content types are served `Content-Disposition: attachment` with `nosniff` and `CSP: default-src 'none'` — the primary defence against an uploaded file rendering as active content. | `media.rs:662-668` |
| BR-API-86 | Range handling: no `Range` ⇒ 200 streamed; single range ⇒ 206 capped at 16 MiB; `start >= total` or unparseable ⇒ **416** with `Content-Range: bytes */TOTAL`; multi-range (comma) is ignored and the full body served. | `media.rs:670-757` |
| BR-API-87 | Suffix ranges (`bytes=-N`) are supported and clamped; `bytes=-0` and any suffix on an empty file ⇒ `None` ⇒ 416; `start > end` ⇒ 416. | `media.rs:759-792`; tests `:1310-1370` |
| BR-API-88 | Blob content hash is verified **on upload** (`buzz-media/src/upload.rs:83`, `:392-393`, `auth.rs:195`) and **not** re-verified on read — reads trust the content-addressed key. | see cited lines |

#### G. Invite issuance, redemption, expiry

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-89 | Minting requires the caller to hold role `owner` or `admin` in the bound community — mirroring the kind:9030 authz; absent membership defaults to `""` and is rejected 403. | `invites.rs:233-247` |
| BR-API-90 | Invite and claim endpoints always require a real NIP-98 signature **and** a `payload` tag covering the body — no `X-Pubkey` fallback, regardless of `require_auth_token`. | `invites.rs:210-218`; payload check `bridge.rs:97-108` |
| BR-API-91 | `ttl_secs` is clamped to `[60, 30 days]`; default 72 h. | `invite_token.rs:129`, `:52`, `:55` |
| BR-API-92 | The invite role is fixed to `"member"` at mint **and re-checked on verify**, so a hand-crafted correctly-signed payload with an elevated role is still rejected. | `invite_token.rs:135`, `:189-191`; test `:311-331` |
| BR-API-93 | Verification order is MAC-first: nothing in the payload is trusted before `verify_slice` succeeds. | `invite_token.rs:171-176` |
| BR-API-94 | A code minted for community A fails on community B on the same deployment. | `invite_token.rs:186-188`; endpoint test `invites.rs:1004-1046` |
| BR-API-95 | `Expired` is the only distinguishable rejection (403 `invite_expired`); every other failure collapses to 403 `invite_invalid` so the endpoint is a poor forging oracle. | `invites.rs:303-312`; test `invites.rs:1071-1101` |
| BR-API-96 | Claim is idempotent: a repeat claim by an existing member returns 200 `already_member` and publishes nothing. | `invites.rs:340-366`; test `invites.rs:704-713` |
| BR-API-97 | On first insert, NIP-43 member-added and membership-list events are published best-effort — a publish failure is logged, never converted into an HTTP error. | `invites.rs:344-355` |
| BR-API-98 | The claim endpoint is **deliberately exempt** from the relay-membership gate; NIP-98 proves control of the joining key and the HMAC proves an admin authorized the join. | `invites.rs:8-13`, `:291-293` |
| BR-API-99 | Codes are multi-use until expiry — there is no server-side "used" bit; revocation is coarse (rotate the relay keypair or remove the member). | `invite_token.rs:32-34`, `:43-46` |
| BR-API-100 | Landing-page URL scheme follows the deployment's TLS posture (`wss://` ⇒ `https`, else `http`), same rule as `nip98_expected_url`. | `invites.rs:266-270` |

#### H. Join-policy acceptance

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-101 | The policy version is a SHA-256 over `terms ‖ 0x00 ‖ privacy ‖ 0x00 ‖ age_required`, so editing any document invalidates every outstanding receipt. | `config.rs:794-811` |
| BR-API-102 | `accept-policy` requires `policy_version == configured version` **and**, if `age_attestation_required`, `age_confirmed == true`; else 400 `join_policy_not_accepted`. | `invites.rs:176-184` |
| BR-API-103 | The receipt binds `hex(sha256(code))`, the policy version, and a 10-minute expiry, HMAC'd with the derived invite key. | `invite_token.rs:346-359` |
| BR-API-104 | When a join policy is configured, `claim` **requires** a receipt; absent ⇒ 403 `join_policy_required`; forged/cross-invite/stale-version ⇒ same 403. | `invites.rs:314-322`; tests `invites.rs:790-901` |
| BR-API-105 | The accepted policy version is persisted alongside the membership row on successful claim. | `invites.rs:324-338`; assertion `invites.rs:967-978` |
| BR-API-106 | `accept-policy` has **no caller authentication and no rate limit** — anyone who knows a code can mint a receipt, so the gate proves "someone clicked", not "this pubkey accepted". | `invites.rs:162-190` |
| BR-API-107 | Operator-supplied policy Markdown is rendered with raw HTML escaped to text, so the endpoint cannot serve arbitrary operator markup; the (literal) title is escaped too. | `invites.rs:126-155`; test `invites.rs:1254-1271` |
| BR-API-108 | Policy documents 404 until configured; the JSON endpoint returns `{}` rather than 404. | `invites.rs:112-124`, `:75-90` |
| BR-API-109 | The join policy is **deployment-global** config, not per-community — every tenant on a deployment shares one policy and version. | `config.rs:794-811`; read at `invites.rs:76`, `:171`, `:314` |

#### I. Moderation queue reads

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-110 | Authorization goes through the single capability helper with `ModerationAction::ViewQueue` and `ModerationTarget::None`, never an inline role check. | `bridge.rs:2054-2070` |
| BR-API-111 | `ViewQueue` is granted to community `owner` and `admin` only — channel-level roles do **not** confer it. | `handlers/moderation_authz.rs:83-96`; tests `:297-320` |
| BR-API-112 | Every authorization failure collapses to 403 `restricted: moderator access required` — the underlying reason is discarded. | `bridge.rs:2063-2070` |
| BR-API-113 | Row counts are clamped to `1..=500`; a non-positive or absent `limit` yields the maximum 500. | `bridge.rs:2077`, `:2066-2072` |
| BR-API-114 | Queue reads are community-wide with no channel context. | `bridge.rs:1998-2000` |

#### J. Workflow webhook

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-115 | The `Host` header — not the workflow row — determines the tenant, so a request for community A can only reach A's workflows even when the same UUID exists in B. | `bridge.rs:1773-1793` |
| BR-API-116 | The workflow must have `TriggerDef::Webhook`, else 400. | `bridge.rs:1816-1821` |
| BR-API-117 | Secret precedence is header (`X-Webhook-Secret`) then `?secret=`; comparison is XOR-fold constant-time after a length check. | `bridge.rs:1823-1831`; `webhook_secret.rs:70-90` |
| BR-API-118 | A workflow with **no** stored secret is rejected 401 with an explanatory message rather than allowed through. | `bridge.rs:1820-1826` |
| BR-API-119 | The response is **202 Accepted** with a `run_id`; execution is spawned detached and its outcome never affects the HTTP status. | `bridge.rs:1866-1913` |
| BR-API-120 | Every top-level key of the JSON body becomes a `webhook_fields` string (non-strings via `to_string()`); the workflow's own `channel_id` seeds the trigger context. | `bridge.rs:1846-1873` |
| BR-API-121 | A definition that fails to re-parse inside the spawned task marks the run `Failed` with the parse error, so the run row is never left `Pending`. | `bridge.rs:1891-1911` |

#### K. Operator control plane

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-122 | Operator requests authenticate against the configured `relay_operator_api_origin`, **not** the inbound `Host`, and never bind a tenant. | `operator.rs:57-60`, `:66-77` |
| BR-API-123 | A `payload` tag is required iff a body is present (`body.is_some()`), so GETs are exempt. | `operator.rs:84` |
| BR-API-124 | Operator replay uses a dedicated deployment-global scope `"operator-management"` rather than a community scope. | `operator.rs:55`, `:104-122` |
| BR-API-125 | The signer must appear in `relay_operator_pubkeys` (exact hex match) or 403. | `operator.rs:86-98` |
| BR-API-126 | If `relay_operator_api_origin` is unset the request fails with **500**, not 403/404 — and config refuses to start with pubkeys but no origin. | `operator.rs:69-72`; `config.rs:577-581` |
| BR-API-127 | Hosts are normalized and canonically validated (normalized-shape equality, no scheme/path/query/userinfo, valid domain labels, ≤253 bytes) before create; availability accepts normalizable-but-non-canonical input. | `handlers/community_provisioning.rs:83-165`, `:167+` |
| BR-API-128 | All pubkeys are validated as exactly 64 hex chars, lowercased. | `handlers/community_provisioning.rs:71-75`; call sites `operator.rs:227`, `:281`, `:317`, `:381`, `:389` |
| BR-API-129 | The deployment's own community host cannot be archived ⇒ 409. | `operator.rs:220-226` |
| BR-API-130 | Archive/unarchive require the asserted `owner_pubkey` to actually own the community; a wrong owner and a nonexistent host both return the same 404 `community not found`. | `operator.rs:234-240`, `:288-293`; test `operator.rs:900-919` |
| BR-API-131 | Archive must propagate a cluster-wide disconnect; a propagation failure returns **503** carrying the archived state and `propagation:"pending"` so the caller retries — the DB mutation is already committed and idempotent. | `operator.rs:241-264`; test `operator.rs:960-1029` |
| BR-API-132 | Transfer is compare-and-swap on `expected_owner_pubkey`; mismatch ⇒ 409 `owner_conflict`, no owner ⇒ 404, transferee at the per-owner cap ⇒ 409 `limit_reached`. | `operator.rs:398-431` |
| BR-API-133 | The previous owner is demoted to `member`, not `admin`. | `operator.rs:348`; test `operator.rs:1148-1157` |
| BR-API-134 | Post-transfer NIP-43 snapshot publication is best-effort and only attempted when `require_relay_membership`; failure logs a warning and still returns 200. | `operator.rs:433-457` |
| BR-API-135 | Provisioning error strings are mapped by prefix: `actor not authorized`→403, `community already exists`/`limit_reached:`→409, persistence prefixes→500 (generic body), everything else→400 (message passed through). | `operator.rs:180-199` |

#### L. Deployment-admin reads

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-136 | Every handler calls `authorize` as its first statement, before any DB access. | `admin/mod.rs:100`, `:130`, `:155`, `:182`, `:196`; test `:377-391` |
| BR-API-137 | `authorize` requires exact `Host` equality with `config.admin.host`, and — only when `Origin` is present — an origin whose host matches. A request with **no** `Origin` passes. | `admin/auth.rs:16-33` |
| BR-API-138 | `limit` must be `1..=200`, else 400 `invalid_limit`; `status` and `target_kind` are allowlisted. | `admin/mod.rs:75-91`, `:101-112` |
| BR-API-139 | Feedback listing is a hard-coded 100 rows with no client-controllable limit. | `admin/mod.rs:151-156` |
| BR-API-140 | Feedback summaries strip Markdown lines whose link target matches an `imeta` attachment URL, then truncate to 240 **chars** (Unicode-safe) with `…`. | `admin/mod.rs:295-322`; tests `:441-458` |
| BR-API-141 | Attachment reads require: 64-lowercase-hex path, an existing feedback row, an `imeta` tag whose `x` equals the requested hash **and** whose `url` resolves to the same community's `/media/{hash}.{safe-ext}` with no query/fragment/extra path. All failures collapse to 404. | `admin/mod.rs:196-240`, `:241-286`; tests `:460-506` |
| BR-API-142 | The tenant for an attachment read is derived from server-owned provenance (`feedback.community_host`), and the resolved community must still equal `feedback.community_id` — otherwise 404 plus a warning. | `admin/mod.rs:206-224` |
| BR-API-143 | Attachment reads go through `serve_blob_for_tenant`, which **bypasses** `authenticate_media_read` — `BUZZ_REQUIRE_MEDIA_GET_AUTH` is not applied on this route. | `admin/mod.rs:226`; `media.rs:619-631` |
| BR-API-144 | Only `GET` is routed for admin paths; write verbs get 405. | `admin/mod.rs:29-38`; test `:406-429` |
| BR-API-145 | Admin reads are **deployment-wide**: `community_id` is an optional filter on `reports`, and `report_detail`/`feedback_detail`/`feedback_attachment` take no community argument at all. | `admin/mod.rs:100-110`, `:125-138`, `:177-189` |
| BR-API-146 | Every `DbError` collapses to 500 `internal_error` with a fixed message. | `admin/error.rs:79-83` |

#### M. NIP-05 and mesh demo

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-147 | The NIP-05 lookup is host-scoped: the community is bound from `Host` and the handle domain must match that same tenant host, not `config.relay_url`. | `nip05.rs:36-58`, `:79-102` |
| BR-API-148 | Absent `name`, unmapped host, and a miss are **indistinguishable** — all return `{names:{},relays:{}}` with 200. | `nip05.rs:41-62` |
| BR-API-149 | The advertised relay URL keeps the deployment scheme from config but uses the tenant host. | `nip05.rs:104-111` |
| BR-API-150 | `Access-Control-Allow-Origin: *` is set unconditionally on the NIP-05 response. | `nip05.rs:65-69` |
| BR-API-151 | `canonicalize_nip05` requires `local@domain` with `domain == tenant host`, lowercasing both parts. | `nip05.rs:79-102`; consumer `handlers/side_effects.rs:1145` |
| BR-API-152 | `/_mesh/demo/echo` returns 404 unless **both** mesh is enabled and `mesh_demo_echo` is on — 404 not 403 so a non-demo deployment looks route-less. | `mesh_demo.rs:64-70` |
| BR-API-153 | The `Owned` branch acquires a fenced Redis lease and **deliberately does not renew it** — it expires with its TTL (30 s default). | `mesh_demo.rs:29-33`, `:108-115` |
| BR-API-154 | The `Forwarded` branch waits at most 10 s for the owner's echo, then 504. | `mesh_demo.rs:44`, `:132-152` |
| BR-API-155 | The demo takes `community_id` from the **request body**, bypassing the host-derived tenant boundary every other route enforces. | `mesh_demo.rs:50-51`, `:100` |


## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: Business Rules

Rules are numbered BR-GIT-nn. Each cites the enforcing site. §7 is the CAS correctness argument.

---

#### 1. Transport-level admission (all three git routes)

| # | Rule | Enforcement | Failure |
|---|---|---|---|
| BR-GIT-01 | Every git route requires a NIP-98 `Authorization: Nostr <base64>` header; it is validated **before** the request body is read (`FromRequestParts`). | `transport.rs:76-231` | 401 + `WWW-Authenticate` |
| BR-GIT-02 | The request `Host` must resolve, through the authoritative `communities` table, to a community. Forwarded headers are never trusted. | `transport.rs:122-130` → `tenant::bind_community` (`tenant.rs:71-92`) | 404 "repository not found" (deliberately indistinguishable from a missing repo) |
| BR-GIT-03 | The NIP-98 `u` tag must equal `<scheme>://<resolved-tenant-host><repo-root-path>`. Scheme derives from `config.relay_url` (`wss://`⇒https, else http); the **host is never taken from config**. | `transport.rs:132-142`, `git_expected_url` `:235-253`; pinned `:2067-2141` | 401 URL mismatch |
| BR-GIT-04 | The HTTP method is **deliberately not bound**: the method fed to the verifier is read out of the signed event itself, making the check tautological. Reason: git's credential helper signs once with `GET` and reuses for `POST`. | `transport.rs:158-183` (explicit SECURITY comment) | n/a |
| BR-GIT-05 | The NIP-98 payload hash is **not** verified (`body = None`) because the pack stream cannot be buffered. | `transport.rs:183-190` | n/a |
| BR-GIT-06 | NIP-98 event-id replay dedup is **deliberately disabled** on git routes; the ±60 s timestamp window is the only replay bound. | `transport.rs:192-198` | n/a |
| BR-GIT-07 | The pusher must pass the relay-membership gate. A NIP-OA attestation may travel in the signed event's `auth` tag or an `x-auth-tag` header, event tag winning. | `transport.rs:204-222`; `handlers/auth.rs:26-36`; `api/mod.rs:124-145` | 403 "restricted: not a relay member" |
| BR-GIT-08 | On an **open** relay (`require_relay_membership = false`, the default) BR-GIT-07 is a no-op — any well-formed NIP-98 signature from any keypair passes. | `api/mod.rs:130-131`; default at `config.rs:483-485`, pinned `config.rs:954-955` | n/a |
| BR-GIT-09 | `owner` must be exactly 64 **lowercase** hex chars. | `transport.rs:268-275` | 400 |
| BR-GIT-10 | `repo` (after stripping one trailing `.git`) must be 1–64 chars of `[A-Za-z0-9._-]`, must not start with `.`, must not contain `..`. | `transport.rs:278-292` | 400 |
| BR-GIT-11 | `service` must be exactly `git-upload-pack` or `git-receive-pack`. | `transport.rs:541-548` | 400 |
| BR-GIT-12 | **There is no per-repo read authorization.** Neither `info_refs` nor `upload_pack` consults kind:30617, channel membership, or the repo-name registry. Any caller past BR-GIT-01/07 can clone any repo in the community. | absence of any check in `transport.rs:539-594`, `:786-827` | — |

---

#### 2. Resource admission and limits

| # | Rule | Value | Site |
|---|---|---|---|
| BR-GIT-13 | Compressed request body ≤ `git_max_pack_bytes` (default 500 MiB) on all three routes. | config | `transport.rs:1757`, `:1763` |
| BR-GIT-14 | gzip-encoded bodies are transparently inflated; **decoded** bytes are capped independently — 64 MiB for upload-pack, `git_max_pack_bytes` for receive-pack. Exceeding it errors the stream, which the stdin pump surfaces to git as early EOF. | `transport.rs:59`, `:745-784`, `:789`, `:861` | pinned `:1823-1887` |
| BR-GIT-15 | Only `gzip`/`x-gzip` is decoded; any other non-identity encoding is passed through unchanged rather than rejected. | `transport.rs:757-765` | |
| BR-GIT-16 | Every git subprocess must hold a permit from the global `git_semaphore` (`git_max_concurrent_ops`, default 20), acquired **non-blocking**. | `transport.rs:318-338`; sized `state.rs:729` | 503 + `Retry-After: 5`, metric `buzz_git_semaphore_rejections_total` |
| BR-GIT-17 | The `info/refs` **fast path acquires no permit** — it is the dominant clone case and does no subprocess work. | `transport.rs:552-577` (permit only in `info_refs_subprocess` `:598`) | |
| BR-GIT-18 | For streamed responses the permit must outlive the body, not the handler. | `GitPermitStream` `transport.rs:1293-1310`, moved in at `:1477-1480`; pinned `:1924-1941` | |
| BR-GIT-19 | `info/refs` subprocess output ≤ 4 MiB; receive-pack buffered output ≤ 1 MiB. | `transport.rs:52`, `:50`; enforced `:680-698`, `:1092-1108` | 413 |
| BR-GIT-20 | `info/refs` subprocess timeout 120 s; pack ops 300 s (both the buffered push and the streaming fetch deadline). | `transport.rs:42`, `:44`; `:660-668`, `:1067-1078`, `:1471` | 504, or in-band stream error + child kill (`:1315-1331`) |
| BR-GIT-21 | Manifest object read ≤ 4 MiB; ref snapshot stdout ≤ 4 MiB; `count-objects -v` output ≤ 64 KiB; stderr capture ≤ 64 KiB. | `hydrate.rs:45`; `cas_publish.rs:270`, `:595`, `:678`, `:1091` etc. | 413 / `ResourceLimit` |
| BR-GIT-22 | Each pack object ≤ `git_max_pack_bytes`, enforced twice: `head_object` Content-Length pre-check then post-read length re-check. | `store.rs:406-444`; `hydrate.rs:305-317` | `ObjectTooLarge` → `ResourceLimit` → 413 |
| BR-GIT-23 | Total hydrated bytes for one repo ≤ `git_max_repo_bytes` (default `pack*2` = 1 GiB), accumulated with `checked_add`. | `hydrate.rs:318-329` | 413 |
| BR-GIT-24 | Generated/fetched `.idx` sidecar ≤ `max_pack_bytes`. | `hydrate.rs:431-446` | `ResourceLimit` |
| BR-GIT-25 | Manifest cardinality: ≤ 128 packs, ≤ 10 000 refs, every pack key canonical. | `manifest.rs:205-237` | `ManifestInvalid` → 400 |
| BR-GIT-26 | Workspace snapshot refuses a **new** ref past 10 000. | `cas_publish.rs:322-326` | `ResourceLimit` → 413 |
| BR-GIT-27 | Repos per pubkey ≤ `git_max_repos_per_pubkey` (default 100), checked at announce, not at push. | `handlers/side_effects.rs:2486-2492`; config `config.rs:731-734` | announce rejected |
| BR-GIT-28 | Protection rules per repo ≤ 50; pattern ≤ 256 chars; ≤ 3 wildcards per pattern. | `buzz-core/src/git_perms.rs:19-23`, `:387` | `RuleParseError` → 403 |
| BR-GIT-29 | Policy callback ref-update count must be `1..=500`. | `policy.rs:213-215` | 403 |
| BR-GIT-30 | `pack-objects` is pinned to `--threads=1`, `--window 10`, `--window-memory=64MiB` on both the delta and compaction paths. | `cas_publish.rs:83-84`, `:524-537`, `:707-727` | |

---

#### 3. Push authorization chain (in execution order)

1. **BR-GIT-31** — BR-GIT-01…11 (auth + id validation) run first (`transport.rs:858-861`).
2. **BR-GIT-32** — semaphore permit (`transport.rs:863`).
3. **BR-GIT-33** — `hydrate_for_write` observes the pointer exactly once and returns `(HydratedRepo, ParentState)`; pointer-absent yields a fresh empty bare repo + `ParentState::fresh()` (`hydrate.rs:207-244`, `transport.rs:878-892`).
4. **BR-GIT-34** — the pre-receive hook is written into the ephemeral workspace at 0o755 (`hook.rs:152-178`, called `transport.rs:906`). Hook install failure ⇒ 500, push aborted.
5. **BR-GIT-35** — `core.hooksPath` is force-overridden via `GIT_CONFIG_COUNT/KEY_0/VALUE_0` so a repo-local setting cannot redirect the hook (`transport.rs:924-930`).
6. **BR-GIT-36** — hook env carries exactly `BUZZ_HOOK_URL`, `BUZZ_HOOK_SECRET`, `BUZZ_REPO_ID` (stripped id), `BUZZ_REPO_OWNER`, `BUZZ_COMMUNITY_ID` (server-resolved), `BUZZ_PUSHER_PUBKEY` (`transport.rs:912-923`). The hook fails closed if any is unset (`hook.rs:42-47`).
7. **BR-GIT-37** — the hook computes `is_ancestor` per ref via `git merge-base --is-ancestor`, inheriting git's quarantine env; exit 128 (error) is treated as *not* ancestor, i.e. fail-closed toward NFF (`hook.rs:59-70`).
8. **BR-GIT-38** — the hook POSTs to `BUZZ_HOOK_URL` with `curl --max-time 10`; any network error or any non-200 ⇒ `exit 1` ⇒ push rejected (`hook.rs:129-148`).
9. **BR-GIT-39** — `BUZZ_HOOK_URL` is hardcoded to `http://127.0.0.1:<bind_addr.port()>/internal/git/policy` (`transport.rs:911-915`). A relay bound to a non-loopback-inclusive address makes every push fail closed.

##### Policy endpoint rules (`policy.rs:173-417`), evaluated in this order

| # | Rule | Site |
|---|---|---|
| BR-GIT-40 | Structural validation runs **before** HMAC (explicitly, to avoid HMAC CPU on garbage): `repo_id` 1–64; `repo_owner`/`pusher_pubkey` 64 lowercase hex; `community_id` a parseable UUID; 1–500 ref updates; each oid exactly 40 hex; each `ref_name` non-empty, ≤256, `refs/`-prefixed, no `..`, no byte ≤ 0x20 or == 0x7f. | `policy.rs:176-234` |
| BR-GIT-41 | HMAC-SHA256 over the length-prefixed canonical payload must match, compared in constant time (`subtle::ConstantTimeEq`). | `policy.rs:113-171`, `:236-241` |
| BR-GIT-42 | Callback age ≤ 30 s; future skew ≤ 5 s. | `policy.rs:51`, `:243-256` |
| BR-GIT-43 | A kind:30617 must exist for `(community, pubkey=repo_owner, d=repo_id)`, `global_only`. Absence ⇒ 403 "repository not found". | `policy.rs:258-284` |
| BR-GIT-44 | Malformed `buzz-protect` tags ⇒ deny (fail-closed). Unknown rule strings are warned and skipped. | `policy.rs:286-306` |
| BR-GIT-45 | If kind:30617 carries a `buzz-channel` tag and that channel is archived, **all** pushes are denied — including the owner's. | `policy.rs:308-330` |
| BR-GIT-46 | The repo owner pubkey, **or** a cryptographically verified NIP-OA managed-agent owner of that pubkey, is treated as `MemberRole::Owner`. | `policy.rs:332-368` (`db.is_agent_owner` `:352`) |
| BR-GIT-47 | Otherwise a `buzz-channel` binding is mandatory; without one ⇒ 403 "no channel binding". | `policy.rs:369-375` |
| BR-GIT-48 | Non-owner role comes from `get_member_role(community, channel, pusher)`; non-member ⇒ 403; unparseable role ⇒ 403. | `policy.rs:376-395` |
| BR-GIT-49 | `MemberRole::Bot` is promoted to `Member` for git only; the promotion is local to this module. | `policy.rs:397-402` |
| BR-GIT-50 | Any DB error at any lookup ⇒ 403, never 5xx. | `policy.rs:277-282`, `:322-326`, `:355-362`, `:388-392` |

##### Ref-protection rules (`buzz-core/src/git_perms.rs`)

| # | Rule | Site |
|---|---|---|
| BR-GIT-51 | Update kind is classified from the zero-oid and `is_ancestor`: zero-old ⇒ Create, zero-new ⇒ Delete, ancestor ⇒ FastForward, else NonFastForward. | `git_perms.rs:207-226` |
| BR-GIT-52 | Built-in defaults when no pattern matches: branch/tag **create** ⇒ Member; branch **FF** ⇒ Member; tag FF (i.e. tag move) ⇒ Admin; any non-`heads`/`tags` ref ⇒ Admin; **NFF ⇒ Admin; Delete ⇒ Admin**. | `git_perms.rs:403-428` |
| BR-GIT-53 | All matching `buzz-protect` patterns are **unioned**: strictest `push:<role>` wins; `no-force-push`/`no-delete`/`require-patch` are sticky-true. | `git_perms.rs:445-486` |
| BR-GIT-54 | An explicit `push:<role>` can **never weaken** the built-in default — the higher permission level of (explicit, default) is used. | `git_perms.rs:534-550` |
| BR-GIT-55 | `require-patch` denies *all* update kinds on the matched ref, not just fast-forwards. | `git_perms.rs:522-529`, doc `:255-262` |
| BR-GIT-56 | A push is **atomic**: any single denied ref rejects the whole push. | `git_perms.rs:584-600`; hook exits 1 on any 403 (`hook.rs:141-148`) |

---

#### 4. Publish gating (`finalize_push`, `transport.rs:1548-1754`)

| # | Rule | Site |
|---|---|---|
| BR-GIT-57 | `finalize_push` is the **unique** constructor of a push response; both `build_git_response` call sites are inside it. | `transport.rs:1574`, `:1748` |
| BR-GIT-58 | A push publishes nothing unless `PackOutput.ok`, where `ok = subprocess exit 0 && no `ng ` line in report-status`. A pre-receive decline exits **0**, so the report-status scan — not the exit code — is the primary signal. | `transport.rs:1113-1139`, `:1567-1580`; `receive_pack_report_rejected` `:1181-1208` |
| BR-GIT-59 | Report-status parsing must de-frame **two** pkt-line levels: side-band-64k nests the status stream inside a band-1 outer frame. Band 2/3 payloads are never treated as status. | `transport.rs:1181-1208`; pinned `:1989-2064` |
| BR-GIT-60 | On `!ok` the buffered in-band report-status is still returned (200) so the client prints the rejection — only the side effects are suppressed. | `transport.rs:1567-1579` |
| BR-GIT-61 | kind:30618 is emitted **after** a successful CAS, never as the commit, and only when `parent_digest != committed_digest`. | `transport.rs:1662-1683` |
| BR-GIT-62 | kind:30618 build failure, DB insert failure, or DB dedup are all **non-fatal** — the push stays durable and still returns 200. | `transport.rs:1712-1735` |
| BR-GIT-63 | kind:30618 is relay-signed and bypasses `dispatch_persistent_event`, going straight to `db.insert_event` + `fan_out_event_to_local_subscribers`. | `transport.rs:1693-1710` |
| BR-GIT-64 | The workspace tempdir is dropped only **after** the response bytes are built. | `transport.rs:1748-1751` |
| BR-GIT-65 | `CasError` → HTTP mapping is total and typed so `?`-bubbling cannot turn a 412 into a 500: `Conflict`→409, `ManifestInvalid`→400, `ResourceLimit`→413, `{ManifestReadFailed, Backend, PackCapture}`→500. | `transport.rs:1601-1657` |

---

#### 5. Read-path (hydration) rules

| # | Rule | Site |
|---|---|---|
| BR-GIT-66 | Pointer absent ⇒ `Ok(None)` ⇒ 404. Every **below-pointer** failure (non-64-hex pointer body, manifest 404, digest mismatch, unsupported version, oversized object, `index-pack` failure) ⇒ error ⇒ 5xx/413. Never a synthesized empty repo. | `hydrate.rs:98-121`, `:246-270`; `transport.rs:341-355` |
| BR-GIT-67 | Pointer body must be exactly 64 ASCII-hex after trim. | `hydrate.rs:255-260` |
| BR-GIT-68 | Manifests are read digest-verified and then `validate()`d on the read side too, duplicating the write-side predicates. | `hydrate.rs:262-268` |
| BR-GIT-69 | **Phase ordering is load-bearing**: all packs fetched, written, and indexed before any ref file or HEAD is written. A failed hydrate leaves a repo with no advertised refs, never refs pointing at missing objects. | `hydrate.rs:288-292`, `:300-329`, `:351-375` |
| BR-GIT-70 | Refnames and oids are re-validated at ref-write time as defense in depth, and `manifest.head` is re-validated before writing `HEAD`. | `hydrate.rs:337-350`, `:366-372` |
| BR-GIT-71 | A pack key must literally begin `packs/`; the remainder is the expected digest. | `hydrate.rs:303-305` |
| BR-GIT-72 | A first-push (pointer-absent) workspace is `git init --bare` + `symbolic-ref HEAD refs/heads/main` — HEAD defaults to `main`, never to git's environment default. | `hydrate.rs:181-184`; pinned `:508-521` |
| BR-GIT-73 | `index-pack` runs **without** `--strict`: connectivity checks are deliberately skipped because `Inv_Closed` is a write-path invariant. Structural integrity (CRC, type tags) is still validated. | `hydrate.rs:410-416` |
| BR-GIT-74 | An `.idx` sidecar from the store is trusted only after `git verify-pack` passes; on failure it is deleted and regenerated. A missing sidecar or a read error is a cache miss, not a failure. | `hydrate.rs:381-408`, `store.rs:317-334` |
| BR-GIT-75 | The `info/refs` fast path is eligible only when: no `refs/tags/*` at all (annotated tags need a `^{}` peel line the manifest cannot reproduce), `head ∈ refs`, and every emitted refname/oid re-passes `is_safe_refname`/`is_hex_oid` with refname ≤ 4096. Any failure degrades to the subprocess path. | `transport.rs:380-419`; pinned `:2164-2211` |

---

#### 6. Pack-cache rules

| # | Rule | Site |
|---|---|---|
| BR-GIT-76 | Cache keys must be 64 ASCII-hex — the path-traversal guard for the shard/entry directory names. | `pack_cache.rs:458-470`; pinned `:586-593` |
| BR-GIT-77 | The cache parent must not be a symlink. | `pack_cache.rs:118-126`; pinned `:575-585` |
| BR-GIT-78 | Concurrent misses for one digest share a single population *flight* (per-digest async mutex + refcount); a cancelled waiter must not deregister a live flight. | `pack_cache.rs:76-104`, `:186-238`, `:392-399`; pinned `:594-661` |
| BR-GIT-79 | Population concurrency ≤ `git_pack_cache_max_concurrent_populations` (default 2, must be > 0). | `pack_cache.rs:110-113`, `:253-269`; config `config.rs:725-730` |
| BR-GIT-80 | An entry is published only by **atomic directory rename** after both pack and idx exist; a lost rename race is resolved by adopting the winner's directory. | `pack_cache.rs:311-333` |
| BR-GIT-81 | `max_bytes == 0`, or a single entry larger than `max_bytes`, **bypasses** the cache: the entry is served from a `TempDir` owned by the returned `CachedPack` and never indexed. | `pack_cache.rs:303-313`; metric `buzz_git_pack_cache_bypasses_total` |
| BR-GIT-82 | Eviction is least-recently-used by logical tick, and **only entries with `Arc::strong_count == 1`** are evictable; if every entry is in flight the loop breaks and the cache temporarily exceeds `max_bytes`. | `pack_cache.rs:365-390` |
| BR-GIT-83 | A cached pair is installed into the workspace by hard link, falling back to copy (counted). Both destination files are removed before the copy attempt. | `pack_cache.rs:428-456` |
| BR-GIT-84 | At construction, sibling `session-*` directories whose `.heartbeat` mtime is older than 10 minutes are recursively deleted; the live session touches its heartbeat every 60 s. | `pack_cache.rs:20-21`, `:127-146`, `:482-509` |
| BR-GIT-85 | A cache hit whose files have vanished is treated as a miss and (on install failure) invalidated. | `pack_cache.rs:241-251`, `:172-183` |

---

#### 7. The CAS protocol — rules and correctness argument

##### 7.1 Rules

| # | Rule | Site |
|---|---|---|
| BR-GIT-86 | The pointer is the **sole** commit point. `manifests/`, `packs/`, `idx/` writes are create-only and idempotent; a 412 on a content-addressed key is *success* because the key is the digest of the bytes being written. | `store.rs:243-280`, `:294-321` |
| BR-GIT-87 | The caller does **not** choose the content-addressed key — `put_immutable` derives it internally. This is what makes the 412-is-success argument constructive rather than a trust assumption. | `store.rs:243-260` |
| BR-GIT-88 | `get_pointer` returns ETag and body from **one** GET so the CAS predicate and the observed state cannot straddle a concurrent writer. A missing ETag header is a hard error. | `store.rs:446-479` |
| BR-GIT-89 | A 2xx CAS PUT with **no** ETag header is treated as a non-conforming backend and fails the operation, rather than returning `ETag("")`. | `store.rs:511-537` |
| BR-GIT-90 | 412 is a *value* (`CasOutcome::LostRace`), not an error; every other non-2xx bubbles as `StoreError::Backend`. | `store.rs:58-64`, `:538-544`; pinned `:930-943` |
| BR-GIT-91 | The CAS predicate is `parent_state.if_match` — the ETag observed at hydrate time — never a re-read. First push uses `If-None-Match: *`. | `cas_publish.rs:1208-1226` |
| BR-GIT-92 | **No retry on `LostRace`.** The loser re-reads the pointer only to report the winner, then returns `Conflict` ⇒ 409. Retrying in-handler would reuse receive-pack output derived against a superseded parent. | `cas_publish.rs:1260-1291`, `read_winner_after_conflict` `:1294-1329` |
| BR-GIT-93 | **No advisory lock.** Writer serialization is exclusively the CAS. Concurrent same-repo pushes each hydrate and run receive-pack; the loser's work is discarded. | `transport.rs:865-877`; `state.rs:517-521` |
| BR-GIT-94 | `m_after.parent` is the parent's **bare** digest (not `manifests/<digest>`), so `parent == pointer.digest` reads literally. | `manifest.rs:238-242`, `cas_publish.rs:907-929`; pinned `manifest.rs:424-447` |
| BR-GIT-95 | `m_after.packs ⊇ parent.packs` on the normal path, sorted and deduped for byte-stability. | `cas_publish.rs:907-929`; pinned `:1440-1487` |
| BR-GIT-96 | `Manifest::validate()` runs between compose and `put_manifest`, so an un-clone-able manifest becomes a push-time 400 instead of a permanent read-time 5xx. | `cas_publish.rs:1177-1190`; call-site pinned `:1834-1878` |
| BR-GIT-97 | Winner resolution after a lost race fails closed at every step: vanished pointer, non-64-hex body, digest mismatch, or missing manifest all become `ManifestReadFailed` ⇒ 500. | `cas_publish.rs:1294-1329` |
| BR-GIT-98 | The published HEAD is resolved, not copied: observed symref if it names a published branch, else the parent's head if it does, else `refs/heads/main`, else `refs/heads/master`, else the first `refs/heads/*`, else observed, else parent. | `cas_publish.rs:355-381`; pinned `:1379-1396` |
| BR-GIT-99 | Objects the push introduced are captured by `git pack-objects --revs --stdout` with `refs_after` tips positive and **`parent_state.parent.refs`** tips negative. | `cas_publish.rs:486-570` |
| BR-GIT-100 | An empty `pack-objects` output, or an empty `refs_after`, is legitimate (refs-only or delete-all push) and still publishes a new manifest reusing the parent's packs. | `cas_publish.rs:507-511`, `:560-562`, `:1153-1157` |
| BR-GIT-101 | `parent_hydrated_bytes + new_pack_bytes ≤ max_repo_bytes`, with `checked_add` overflow handling. | `cas_publish.rs:1132-1146` |
| BR-GIT-102 | `idx` sidecar write failure is **non-fatal** — a warning; hydrate regenerates. | `cas_publish.rs:1148-1152`, `:828-844` |

##### 7.2 Pack compaction rules

| # | Rule | Site |
|---|---|---|
| BR-GIT-103 | Compaction is attempted when `parent.packs.len() >= PACK_COMPACTION_THRESHOLD` (96) — a quarter of manifest capacity kept in reserve. | `cas_publish.rs:572-574`, `:1046`; pinned `:1499-1513` |
| BR-GIT-104 | Compaction is serialized process-wide by a `Semaphore::const_new(1)` with a 300 s acquire timeout and a 600 s operation timeout. | `cas_publish.rs:86`, `:576-586`, `:82`, `:1058` |
| BR-GIT-105 | Compaction refuses repos with more than 1 000 000 objects (`count-objects -v`, `count:` + `in-pack:`, with `checked_add`). | `cas_publish.rs:85`, `:593-664` |
| BR-GIT-106 | Compaction feeds **deduplicated** `refs_after` tips with **no** negative revisions and `--max-pack-size=max_pack_bytes`, producing a self-contained closure. | `cas_publish.rs:694-727` |
| BR-GIT-107 | The compacted pack set is usable only if it reduces the pack count, or (at the hard cap) matches it while staying within `MAX_MANIFEST_PACKS`. | `cas_publish.rs:588-591`, `:853-875`; pinned `:1515-1533` |
| BR-GIT-108 | A compacted manifest **replaces** the parent's pack list rather than extending it; old objects are never deleted, so readers holding an earlier manifest stay valid. | `cas_publish.rs:931-948`; pinned `:1535-1560` |
| BR-GIT-109 | Compaction failure (any cause) degrades to the normal delta path with a `fallback` metric; but if `packs_before >= 128` **and** a new pack is needed, the push is rejected with the compaction error. | `cas_publish.rs:1096-1103`, `:1120-1131`; metric `buzz_git_pack_compaction_required_failures_total` |
| BR-GIT-110 | Compacted pack bytes are re-checked against the on-disk size after read; a size change between validation and upload is an error. | `cas_publish.rs:812-827` |

##### 7.3 Why the CAS is correct here

**Claim.** For every accepted push, the manifest installed by the winning CAS is derived from exactly the pointer state the workspace was hydrated from; concurrent pushes never lose an update; and no unauthorized ref value can be published.

**(a) Parent is observed once, and the observation is the predicate.** `hydrate_for_write` performs a single `get_pointer` and packages `(ETag, digest, Manifest)` into `ParentState` (`hydrate.rs:207-244` → `cas_publish.rs:245-255`). That value travels on `PushContext.parent_state` (`transport.rs:944-955`) into `cas_publish`, which uses `parent_state.if_match` verbatim as the precondition (`cas_publish.rs:1208-1226`) and `parent_state.parent_digest` verbatim as `m_after.parent` (`:1159-1165`, `:907-929`). There is no code path between hydrate and CAS that re-reads the pointer. So "built on `d_old`, published against `d_new`" is not expressible.

**(b) The delta pack is computed against the same parent.** `capture_pack`'s negative set is `parent_state.parent.refs` (`cas_publish.rs:1112-1119`), i.e. the manifest the workspace was materialized from — and `materialize_manifest` wrote exactly those refs to disk (`hydrate.rs:355-371`). So `refs_before` (manifest) and the workspace's pre-push refs are equal by construction, and `m_after.packs = parent.packs ∪ {new}` covers the reachable closure of `refs_after`.

**(c) At most one CAS predicated on a given ETag succeeds.** Delegated to the backend (axiom A3) and *admitted*, not assumed: `run_conformance_probe` races `race_width` (default 32) writers over `race_rounds` (default 3) rounds for both `If-Match` and `If-None-Match: *`, requiring exactly one winner among *classified* observers, ≥ 2 classified observers, and post-race byte equality (`store.rs:571-877`). It is a **fatal** startup gate (`main.rs:466-503`), opt-out via `BUZZ_GIT_CONFORMANCE_PROBE=false`.

**(d) Transport-unknown outcomes are dropped, not counted.** `S3Error::{Reqwest, Http, Io}` racers never received a classified response, so they are excluded from the observer set and surfaced as `ProbeReport.transport_drops` (`store.rs:126-149`, `:673-700`, `:781-800`). This is what keeps the probe a conformance test rather than a network-stack test, and prevents a false pass at `classified < 2`.

**(e) The loser cannot publish.** `LostRace` is returned as `CasError::Conflict` — a variant distinct from `Backend` — so `?`-bubbling cannot silently convert it (`cas_publish.rs:105-124`, mapped `transport.rs:1605-1618`). No retry loop exists in the handler; `git` re-drives the whole push, which re-hydrates against the advanced pointer.

**(f) The re-read after a loss is allowed to race.** `read_winner_after_conflict` may observe a *third* winner; that is explicitly accepted because the value is diagnostic only and the client re-pushes regardless (`cas_publish.rs:1272-1279`, `:1294-1305`).

**(g) The response fence.** The success 2xx is constructed only after `cas_publish` returns `Ok` (`transport.rs:1582-1660` then `:1748`). `run_git_at` deliberately returns an owned `PackOutput` instead of a `Response` so no earlier code can build a push response (`transport.rs:974-991`). `finalize_push` is a single sequential `async fn` with no detached `tokio::spawn`.

**(h) A false-positive `ok` cannot publish unauthorized refs.** This is the deeper reason the fence is safe, and it is *not* stated in the doc. If report-status parsing ever missed an `ng` line — e.g. a client that never negotiated `report-status`, so receive-pack emits no status at all — `ok` would be `true` and the CAS would run. But a pre-receive decline means git rejected **all** commands (pre-receive is all-or-nothing) and never migrated the quarantined objects, so the workspace refs still equal `parent.refs`. `snapshot_workspace_state` therefore yields `refs_after == parent.refs`, `capture_pack` yields empty, `compose_after` reproduces the parent's `(head, refs, packs)`, and `canonical_bytes` is deterministic ⇒ the same digest ⇒ the CAS installs the identical value and `manifest_changed` is false, so no kind:30618 is emitted. **Ref integrity comes from the workspace, not from the status parse**; `PackOutput.ok` is a cost/noise optimization on top of it.

**(i) The residual hole is pointer *creation*, not ref content.** Nothing in `receive_pack` requires the repo to be announced. A receive-pack request with **zero** ref commands causes git to skip `execute_commands`, so the pre-receive hook never runs, `ok` is true, and `cas_publish` proceeds: for an existing repo it re-installs the same digest under a fresh ETag (a 409-inducing contention vector for concurrent legitimate pushers); for an **absent** pointer it takes `IfNoneMatchStar` and creates a pointer to the canonical empty manifest for an arbitrary `(owner, repo)` — bypassing `git_repo_names` reservation (`handlers/side_effects.rs:2441-2513`) and the per-pubkey quota. This contradicts the doc's "pointer-absence means never announced" (doc §Implementation Correspondence). It cannot publish ref *content* (that requires ≥1 command ⇒ the hook ⇒ kind:30617), and the squatted manifest is byte-identical to the announce seed so a later legitimate announce still succeeds via `seed_manifest_pointer`'s digest-equality check (`handlers/side_effects.rs:2656-2688`). *Inferred from code structure plus git's `receive-pack` semantics; not confirmed by a live behavioral test.*

**(j) One correctness gap in the snapshot.** BR-GIT-111: `snapshot_workspace_state` silently `warn!`s-and-skips any ref whose oid is not exactly 40 hex (`cas_publish.rs:325-328`), while `is_hex_oid` (`manifest.rs:156`), `manifest_event::is_valid_oid` (`:129`), and the advertisement's `object-format` derivation (`transport.rs:473-479`) all accept 64. In a SHA-256 repository every ref would be dropped and the CAS would publish `refs: {}` — a silent full ref deletion that `validate()` accepts. Unreachable today because `init_bare_repo` always creates SHA-1 repos (`hydrate.rs:181-184`), but the failure mode is silent-drop rather than fail-closed.

##### 7.4 Rules the implementation adds beyond the spec

| # | Rule | Site | Spec status |
|---|---|---|---|
| BR-GIT-112 | HEAD is a published manifest field, resolved via a 6-step fallback (BR-GIT-98). | `cas_publish.rs:355-381` | spec's `m` has only `packs` + `refs` |
| BR-GIT-113 | Manifest carries `version` and is rejected on mismatch. | `manifest.rs:263-272` | not in spec |
| BR-GIT-114 | Pack compaction (BR-GIT-103…110). | `cas_publish.rs` | doc §Theorem 2 Remark covers it |
| BR-GIT-115 | `idx/` sidecar tier. | `store.rs:227-334`, `hydrate.rs:381-429` | not in spec |
| BR-GIT-116 | The `info/refs` manifest fast path bypasses hydrate entirely. | `transport.rs:380-537` | not in spec (spec §Read always hydrates) |
| BR-GIT-117 | Byte/count/time limits (BR-GIT-13…30). | throughout | spec is silent on resource bounds |
| BR-GIT-118 | Spec §Push step 4 ("validate Δ against `m_before.refs`") has **no** counterpart inside `cas_publish` — the fast-forward/protection check lives entirely in the pre-receive hook against the *workspace*, which was materialized from `m_before`. Equivalent in effect, different location. | `hook.rs:32-150` + `policy.rs:173-417`; absence in `cas_publish.rs:1021-1292` | delta |
| BR-GIT-119 | `capture_pack`'s comment claims it deduplicates the `X ^X` case; it does not (`cas_publish.rs:494-517` pushes every value unconditionally). `capture_compacted_packs` **does** dedup (`:696-702`). | `cas_publish.rs:490-517` | comment/behavior delta |


## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: Business Rules

All rules below are read out of code, not documentation. Where a doc comment or `ARCHITECTURE.md` disagrees, the delta is called out inline.

---

#### A. Moderation authorization (BR-MOD-01 … BR-MOD-12)

**BR-MOD-01 — Single authorization seam.** Every moderation decision routes through `authorize_moderation_action` (`moderation_authz.rs:83-144`); no moderation handler performs an inline role check. Verified: `moderation_commands.rs` contains zero `get_relay_member`/`get_member_role` calls; all 5 command paths call the seam (`:156`, `:235`, `:274`, `:338`, `:399`).

**BR-MOD-02 — Community owner is unconditionally authorized.** `actor_role == "owner"` ⇒ `Ok(CommunityOwner)` for all 8 actions with **no guard rail**, including against another owner or admin target (`moderation_authz.rs:149-150`; test `:207-217`).

**BR-MOD-03 — Community admin holds every capability except restricting a peer.** `actor_role == "admin"` ⇒ `Ok(CommunityAdmin)` unless the action is `Ban` or `Timeout` **and** the target's `relay_members.role` is `owner` or `admin`, in which case: `an admin cannot ban or time out a community owner or fellow admin` (`moderation_authz.rs:163-171`).

**BR-MOD-04 — The peer guard trips on role, never on a missing row.** A target with no `relay_members` row (`target_role == None`) is bannable by an admin (`moderation_authz.rs:165-167` requires `Some("owner") | Some("admin")`; test `:246-266`). Rationale in-code: a drive-by spammer who already left must stay bannable (`moderation_authz.rs:157-159`).

**BR-MOD-05 — Restriction *lifting* is deliberately unguarded at the role seam.** `Unban` and `Untimeout` are excluded from the peer guard (`moderation_authz.rs:165`), so an admin may lift another admin's or the owner's restriction. Documented as intentional and "benign, audited, and owner-reversible" (`moderation_authz.rs:158-162`); test `:269-291`.

**BR-MOD-06 — Channel role grants only `DeleteMessage`/`Kick`, and only within `channel_id`.** `moderation_authz.rs:174-179`. **This rule is unreachable in production:** all 6 call sites pass `channel_id: None` and none request `DeleteMessage`/`Kick`, so `channel_role` is always `None` (`moderation_authz.rs:120-131`). Exercised by unit tests only (`:294-310`).

**BR-MOD-07 — Everyone else is denied with `moderator access required`.** Plain channel members and strangers fail every action (`moderation_authz.rs:178`; test `:313-327`).

**BR-MOD-08 — Authority never crosses the tenant fence.** Both role reads are keyed on `tenant.community()` (`moderation_authz.rs:96-131`). The module states callers must pre-resolve the target inside the same tenant (`:15-19`); `moderation_commands.rs` satisfies this for the report path (`:414`) but for pubkey targets there is nothing to resolve — a raw `p`-tag pubkey is accepted without any in-tenant existence check (`:147`, `:228`, `:264`, `:331`).

**BR-MOD-09 — Target-role lookup is conditional, not unconditional.** The second `get_relay_member` fires only for `(admin, Ban|Timeout, Pubkey)` (`moderation_authz.rs:105-116`). Owner and channel-role paths stay at one query. Side effect: an admin banning an *event* target would skip the guard entirely — currently unreachable because `Ban` is only ever paired with `ModerationTarget::Pubkey`.

**BR-MOD-10 — There is no Moderator tier in v1.** Roles are exactly community `owner`/`admin` plus channel `owner`/`admin` (`moderation_authz.rs:5-10`). Adding a tier is a change to `decide_authority` only.

**BR-MOD-11 — `ViewQueue` gates the read APIs.** `GET /moderation/reports` and `/moderation/audit` both require `ViewQueue` under `ModerationTarget::None` (`api/bridge.rs:2054-2067`), which by BR-MOD-02/03/07 means community owner or admin only. Denial is normalised to a fixed 403 `restricted: moderator access required` regardless of the underlying reason (`bridge.rs:2062-2067`).

**BR-MOD-12 — `ModerationAuthority` is computed and thrown away.** The doc comment says the matched authority is "recorded in the audit row" (`moderation_authz.rs:61`), but `insert_audit` accepts no authority argument (`moderation_commands.rs:518-527`) and both callers discard the return value (`moderation_commands.rs:165`, `bridge.rs:2062`). **Documentation delta.**

---

#### B. Self-action and last-owner protection (BR-MOD-13 … BR-MOD-18)

**BR-MOD-13 — Community moderation has NO self-action prevention.** `moderation_commands.rs` never compares `actor` to `target`. An owner can self-ban (9040 with own `p` tag) — `decide_authority` returns `CommunityOwner` (`moderation_authz.rs:149`), the ban row is written (`:169`), and the actor's own live sockets are then closed by `disconnect_pubkey_clusterwide` (`:195-200`). Recovery requires another owner/admin or direct DB access, because the ban re-check at `moderation_commands.rs:103-108` blocks the self-unban. An admin cannot self-ban only incidentally: the peer guard trips on their own `admin` role (`moderation_authz.rs:165-167`).

**BR-MOD-14 — Relay-admin removal has explicit self-protection.** `cannot remove yourself` (`relay_admin.rs:231-233`) and `cannot change your own role` (`relay_admin.rs:290-292`). Note the asymmetry with BR-MOD-13.

**BR-MOD-15 — The relay owner cannot be removed or demoted.** Owner removal is refused atomically in the DB, surfaced as `RemoveResult::IsOwner` → `cannot remove the relay owner` (`relay_admin.rs:257-259`). Role change on an owner fails the update and is disambiguated by a second read into `cannot change the relay owner's role` vs `member not found` (`relay_admin.rs:313-325`). This is the last-owner protection: there is no owner-count check, the owner row is simply immutable through 9031/9032.

**BR-MOD-16 — Ownership transfer via 9032 is blocked by design.** `cannot set role to owner` (`relay_admin.rs:300-302`) with an in-code DESIGN note that transfer is high-risk and must go through `RELAY_OWNER_PUBKEY` config (`relay_admin.rs:296-299`). 9030 likewise refuses `role=owner` with `invalid role: use kind:9032 to promote to owner` (`relay_admin.rs:184-186`) — a **misleading message**, since 9032 also refuses it.

**BR-MOD-17 — Admins cannot mint admins.** Only an owner may add a member at `role=admin` (`relay_admin.rs:187-189`). Role vocabulary is closed to `admin`/`member` (`relay_admin.rs:190-192`).

**BR-MOD-18 — Admin removal is TOCTOU-safe by construction.** Admins use `remove_relay_member_if_role(…, "member")` — a single conditional delete — rather than read-then-delete, explicitly to close the promote-between-read-and-delete race (`relay_admin.rs:235-249`). Owners use the unconditional `remove_relay_member` which itself refuses owners.

---

#### C. Freshness window (BR-MOD-19 … BR-MOD-21)

**BR-MOD-19 — ±120 s signed-timestamp window on every direct command.** Applied identically in four places with three different spellings:

| Path | Constant | Site |
|---|---|---|
| 9040–9044 | `MAX_COMMAND_SKEW_SECS = 120` | `moderation_commands.rs:81`, check `:113-121` |
| 9030–9033 | bare literal `120` | `relay_admin.rs:125-130` |
| 9035/9036 | bare literal `120` | `identity_archive.rs:141-153` |
| 30350 | `ALLOWED_SKEW = 120` (only against `expiration`, not `created_at`) | `push_lease.rs:476`, `:138` |

Rationale: these events are never stored, so replay of a captured command is the only threat and a tight window is the mitigation (`moderation_commands.rs:78-80`).

**BR-MOD-20 — The window is checked *after* the ban re-check for moderation, *after* nothing for relay-admin.** `moderation_commands.rs` reads restriction state at `:103-108` and only then checks freshness at `:113`. `relay_admin.rs` checks freshness first (`:122-131`) then reads the member row (`:133-137`).

**BR-MOD-21 — Clock-read failure fails closed.** `SystemTime::now().duration_since(UNIX_EPOCH)` errors map to `now = 0` (`moderation_commands.rs:115`, `relay_admin.rs:124`, `identity_archive.rs:146`), which makes `|event_ts - 0| > 120` true for any real timestamp ⇒ universal rejection. Correct direction, but it produces a confusing `now=0` in the error string.

---

#### D. Ban vs timeout semantics (BR-MOD-22 … BR-MOD-29)

**BR-MOD-22 — A ban is an admission boundary; a timeout is a write-block.** Ban is enforced at the NIP-42 auth seam (`handlers/auth.rs:107-186`) *and* as a durable backstop on the ingest write path (`ingest.rs:1616-1650`). Timeout has **no auth-seam presence at all** — it appears only on the ingest write path (`ingest.rs:1663-1669`), producing `restricted: you are timed out until <unix ts>` so the desktop can render a countdown (`ingest.rs:1584-1588` comment; `:1639-1641`).

**BR-MOD-23 — Ban closes live sockets cluster-wide, timeout does not.** 9040 calls `disconnect_pubkey_clusterwide` (`moderation_commands.rs:195-200`), which closes this pod's sockets synchronously and fire-and-forget-publishes a Redis `ConnControl::DisconnectPubkey` to every other pod (`state.rs:1018-1050`). 9042 has **no** disconnect call — a timed-out user keeps their socket and continues receiving events, only writes are refused.

**BR-MOD-24 — The live ban fan-out is best-effort; the durable row is the guarantee.** The Redis publish is `tokio::spawn`ed and its failure only logs (`state.rs:1043-1047`), justified in-code because the durable ban row rejects the member again at auth (`state.rs:1039-1042`). The ingest write-path gate exists precisely to cover a missed disconnect (`ingest.rs:1589-1596`).

**BR-MOD-25 — Ban expiry is permitted but not surfaced to the handler.** 9040 accepts an optional `expiration` tag ⇒ `None` means permanent (`moderation_commands.rs:148`, doc `:29`). The handler's own re-check reads only `RestrictionState.banned` (`moderation_commands.rs:135-142`), which omits `ban_expires_at` (`buzz-db/src/moderation.rs:432-436`); expiry semantics live entirely in the DB query.

**BR-MOD-26 — Timeout requires an expiration.** `invalid: timeout requires an expiration tag` (`moderation_commands.rs:270-271`). Unlike a ban, an unbounded timeout is not expressible.

**BR-MOD-27 — Moderation commands are exempt from the ban/timeout write gate so a timeout cannot disarm the tool that lifts it.** `ingest.rs:1639` excludes both `is_moderation_command_kind` and `is_relay_admin_kind`; the handler then re-imposes only the *ban* half (`moderation_commands.rs:103-108`), letting a timed-out admin issue 9043. Pinned by a unit test (`moderation_commands.rs:623-641`).

**BR-MOD-28 — The ban cascades from NIP-OA owner to agent; the timeout does not.** At the auth seam, if the authenticated pubkey is clear, the cryptographically-proven NIP-OA owner is checked and an owner ban denies the agent (`handlers/auth.rs:134-155`). The ingest write gate checks the **authoring pubkey only, with no cascade** (`ingest.rs:1624-1637`) — explicitly documented as a deliberate Phase-1 asymmetry because `IngestAuth` does not carry the self-proving auth tag (`ingest.rs:1632-1637`). Net effect: **an owner timeout does not silence that owner's agents.**

**BR-MOD-29 — Ban-state DB errors deny, but are distinguished from real bans.** Auth seam: `BanOutcome::DbError` ⇒ `error: internal error checking restriction state` rather than claiming a ban (`handlers/auth.rs:112-176`). Ingest gate: `IngestError::Internal("error: internal error checking restriction state: {e}")` (`ingest.rs:1645-1650`). Handler: `error: database error checking restriction state: {e}` (`moderation_commands.rs:108`). Three different strings for the same condition.

---

#### E. Report and resolution rules (BR-MOD-30 … BR-MOD-40)

**BR-MOD-30 — Reports are signals, never triggers.** The relay persists to the tenant queue and never auto-actions or fans out publicly (`report.rs:1-6`; not stored as an event — `ingest.rs:1551-1570`).

**BR-MOD-31 — Reports stay submittable while timed out and in the post-ban window.** 1984 is dispatched at `ingest.rs:1562`, before the restriction gate at `:1613`. Documented as intentional: users must be able to signal abuse during a write-block, and a banned actor in a missed-disconnect window is tolerated because reports are non-actioning and mod-only (`ingest.rs:1551-1559`).

**BR-MOD-32 — Exactly one `p` tag, at most one of `e` or `x`, never both.** `report.rs:122-142`. The `p` tag is mandatory even for `e`/`x` targets (NIP-56 shape) but its value is **not trusted** for authorship — the annotation says e-target author truth comes from the stored row (`report.rs:106-107`) and blob authorship is not inserted at all (`:112-113`).

**BR-MOD-33 — `e` targets must resolve in-tenant or the report is rejected.** `get_event_by_id(tenant.community(), …)` ⇒ `invalid: report target event not found` (`report.rs:55-60`); never a cross-tenant search (`report.rs:11-14`).

**BR-MOD-34 — `x` blob targets must have a tenant-scoped sidecar.** `media_storage.get_sidecar(tenant, sha_hex)` (`report.rs:66-71`), so a bare SHA-256 shared across tenants grants no cross-tenant visibility (`report.rs:15-18`). Known limitation: all lookup failures — including transient storage errors — collapse to `invalid: report target blob not found` (`report.rs:67-70`).

**BR-MOD-35 — `channel_id` is inferred only from an `e` target.** `Blob` and `Pubkey` targets store `channel_id = None` (`report.rs:72`, `:74`), which means a pubkey-only report is never visible in a channel-scoped queue view.

**BR-MOD-36 — Report type must be one of 7 NIP-56 values, read from the *target tag's third element*.** `REPORT_TYPES` (`report.rs:29-37`); extracted at `report.rs:186` and validated at `:200-209`. Note the type comes from the `e`/`x`/`p` tag itself, not a separate tag.

**BR-MOD-37 — Report insertion is idempotent on the signed event id.** `report_event_id` is the per-community idempotency key (`buzz-db/src/moderation.rs:39`, `:170-171`), so re-submitting the same signed 1984 is a no-op returning the existing row.

**BR-MOD-38 — 9044 vocabulary is re-validated server-side.** Status ∈ `{resolved, dismissed}` (`moderation_commands.rs:380-385`); action ∈ `{delete, kick, ban, timeout, dismiss, escalate}` (`:386-392`); `dismiss` ⇔ `dismissed` is an exclusive-or check (`:393-397`). In-code rationale: the SDK validates at build time but the relay must not trust the client (`:377-379`).

**BR-MOD-39 — 9044 records a *decision*, not an *enforcement*.** `resolve:*` prefixed audit actions distinguish the moderator's one-click resolution from the paired 9040/9042 that writes the unprefixed enforcement row (`moderation_commands.rs:502-513`, rationale `:443-450`). `dismiss` → `dismiss_report`, `escalate` → `escalate` — both unprefixed so escalations stay queryable for platform safety (`:503-504`, doc `:52-54`).

**BR-MOD-40 — A closed report cannot be re-resolved, with an accepted residual race.** An early `report.status != "open"` check (`moderation_commands.rs:426-430`) prevents an orphan audit row; the real guard is the DB's `WHERE status='open'` inside `resolve_moderation_report` (`:461-468`). The comment concedes a window where the row flips between read and write, yielding an audit row plus a failed resolve — explicitly tolerated (`moderation_commands.rs:419-425`).

**Additional 9044 facts:** the audit row is written **before** the resolve (`:453-460` then `:461`), so a failed resolve leaves an audit row; and the audit row inherits the *report's* target (`:434-439`), with `ReportTarget::Blob` mapping to `(None, None)` — a blob resolution audits with no target at all.

---

#### F. Notice-delivery rules (BR-MOD-41 … BR-MOD-46)

**BR-MOD-41 — Notice delivery is best-effort and never fails the command.** All three call sites log and continue (`moderation_commands.rs:204-220`, `:309-321`, `:481-495`), with in-code rationale that the enforcement has already landed and been audited (`:214-216`).

**BR-MOD-42 — One DM thread per (community, user), created on first use.** `open_dm` is participant-hash idempotent (`moderation_notices.rs:97-107`).

**BR-MOD-43 — A hidden moderation thread is force-resurfaced.** `unhide_dm` is called on every send because `open_dm` only clears `hidden_at` for `created_by` (the relay key) — otherwise a user who hid the thread would never see a later ban notice (`moderation_notices.rs:121-129`).

**BR-MOD-44 — Idempotency is crash-safe but not concurrency-safe.** `notice_already_sent` is a query-then-insert keyed on the `moderation_source` tag (`moderation_notices.rs:227-252`); the comment states two simultaneous deliveries for the same source can both miss the pre-query and that hard serialization is a noted follow-up (`:132-138`). The scan limit is pinned to the query clamp of 1000 rather than 1 or the default 100, because matching happens post-query in Rust (`:222-226`).

**BR-MOD-45 — Discovery and profile are re-emitted on every send, deliberately.** Not gated on `was_created`, because a first-delivery discovery failure would otherwise leave the thread permanently undiscoverable (`moderation_notices.rs:141-151`). The kind:0 profile failure is downgraded to a warning (`:152-154`) while discovery failure is `?`-propagated (`:155`) — so a discovery failure *does* abort the notice.

**BR-MOD-46 — Privacy invariant: notices never name reporters or quote report notes.** Bodies are built only from status/summary/`public_reason` (`moderation_notices.rs:267-306`, invariant `:268-272`, module doc `:20-23`). The reporter-facing summary defaults to a generic sentence when no `reason` tag is present (`moderation_commands.rs:477-481`). Note that `reason` from a 9044 tag is relayed verbatim to the reporter (`:479`) — the doc requires it be "safe for the reporter's eyes" (`moderation_commands.rs:44-47`) but nothing sanitizes it. A 32-byte recipient length check guards the DM (`moderation_notices.rs:82-87`) and a self-DM to the relay key is silently skipped (`:94-96`).

---

#### G. Relay-membership admin rules (BR-MOD-47 … BR-MOD-51)

**BR-MOD-47 — Permission matrix.** 9030 add / 9031 remove / 9033 workspace-profile require `admin`|`owner`; 9032 change-role requires `owner` (`relay_admin.rs:20-29` doc table; enforced `:148`, `:177`, `:227`, `:286`).

**BR-MOD-48 — 9030 is silently idempotent and never overwrites a role.** `add_relay_member` returns `was_inserted`; an existing member at any role is a no-op with `Ok(())` (`relay_admin.rs:194-212`, doc `:194-196`). NIP-43 announcements are suppressed on a no-op re-add to avoid spurious kind:8000 events (`:212-221`).

**BR-MOD-49 — 9033 is handled before p-tag extraction because it targets the relay, not a member.** `relay_admin.rs:144-166`. Missing or empty `icon` tag clears the icon (`:152`, `:157-161`).

**BR-MOD-50 — Workspace-icon policy.** Empty ⇒ clear; no control or whitespace characters; `data:image/*` up to 98,304 bytes; otherwise must start `https://` or `http://` and be ≤ 2048 bytes (`relay_admin.rs:69-94`). **`http://` is accepted** (`:83`), so an icon can be a plaintext URL. `data:text/html` and `javascript:` are rejected (test `:441-444`).

**BR-MOD-51 — NIP-43 publication failures never fail the command.** All 5 publish calls warn-and-continue (`relay_admin.rs:214-220`, `:274-279`, `:334-336`), so `relay_members` and the event-backed membership view can diverge silently.

---

#### H. Community-provisioning rules (BR-MOD-52 … BR-MOD-58)

**BR-MOD-52 — Provisioning authority is deployment-level, not community-level.** The gate is the `RELAY_OPERATOR_PUBKEYS` allowlist, deliberately **not** a `relay_members` lookup, because the effect of the operation is the creation of tenancy itself (`community_provisioning.rs:3-14`, `:255-266`).

**BR-MOD-53 — Empty allowlist fails closed.** An empty `relay_operator_pubkeys` (the default, `config.rs:962-963`) rejects everyone (`community_provisioning.rs:258-266`). Config additionally refuses to boot if the allowlist is set without `RELAY_OPERATOR_API_ORIGIN` (`config.rs:577-580`).

**BR-MOD-54 — `create_only=true` requires an owner and refuses an existing host.** `initial_owner_pubkey is required when create_only is true` (`:281-283`); `HostExists` ⇒ `community already exists`, `LimitReached` ⇒ `limit_reached: …` (`:290-299`).

**BR-MOD-55 — Legacy convergence mode can *rotate an existing community's owner*.** Without `create_only`, `ensure_configured_community` + `bootstrap_owner` runs even when the community already existed, demoting any previous owner to admin (`community_provisioning.rs:236-247`, `:321-334`). Documented as the price of retry convergence and as the reason the allowlist is "deployment-root authority, not create-only authority" (`:243-247`). Clients acting for end users are told to use `create_only` (`:317-320`) — this is guidance, not enforcement.

**BR-MOD-56 — Create requires a pre-normalized canonical host; availability checks do not.** `validate_host` insists `normalize_host(host) == host` so the stored value is byte-identical to request-time lookup (`community_provisioning.rs:75-100`). `normalize_candidate_host` normalizes first, accepting uppercase/trailing-dot/default-port, for read-only endpoints (`:174-207`).

**BR-MOD-57 — Host must be a canonical bare authority.** No scheme/path/query/fragment/userinfo (`:106-118`); parsed via `Url::parse("http://{authority}/")` and required to round-trip byte-identically (`:120-146`); DNS labels are alphanumeric-or-hyphen, non-empty, ≤63, not hyphen-terminal, total ≤253 (`:150-172`). Underscores are rejected (test `:409`), IPv6 brackets accepted (test `:421-424`).

**BR-MOD-58 — Membership snapshot publication is best-effort and only when membership is required.** Skipped entirely if `!require_relay_membership` (`:210-217`); a failure warns because provisioning is already committed and turning a stored success into an HTTP failure would make create-only retries report a misleading conflict (`:202-208`, `:218-227`).

---

#### I. Push-lease rules (BR-MOD-59 … BR-MOD-72)

**BR-MOD-59 — Lease acceptance is disabled without a gateway URL.** First check in `accept`: `push not supported` (`push_lease.rs:480-482`). Because the URL defaults to `https://push.buzz.xyz/v1/deliveries/apns` (`config.rs:339`, `:752-758`), leases are accepted **by default** on any deployment that does not explicitly set the var to empty.

**BR-MOD-60 — Only 4 public tags are permitted and duplicates are fatal.** `d`, `expiration`, `exec`, `alt`; every other tag ⇒ `unexpected public tag: {name}`; a repeat ⇒ `duplicate {name} tag`; any tag with ≠2 elements ⇒ rejected (`push_lease.rs:106-126`). This is the metadata-leak guard: nothing about subscriptions is public.

**BR-MOD-61 — Lease TTL is 30 days max, 120 s skew tolerance.** `expiration <= now - 120` ⇒ `lease already expired`; `expiration > now + 2_592_000` ⇒ `lease ttl too long` (`push_lease.rs:138-143`, consts `:475-476`).

**BR-MOD-62 — The executor key id must match deployment config exactly.** `unknown executor key` (`push_lease.rs:484-486`), default `relay-v1` (`config.rs:745-746`).

**BR-MOD-63 — Plaintext is NIP-44-decrypted with the relay's own secret key.** `nip44::decrypt(state.relay_keypair.secret_key(), &event.pubkey, &event.content)` (`push_lease.rs:487-492`) — any failure collapses to `invalid encrypted content`.

**BR-MOD-64 — Origin must equal the server-resolved tenant authority, never the decrypted claim.** `canonical_origin(config.relay_url, tenant.host())` derives the scheme from the relay URL and the authority from the tenant (`push_lease.rs:585-596`); `body.origin != expected` ⇒ `origin mismatch` (`:200-202`). Module doc pins that tenant selection never comes from the decrypted `origin` (`:3-5`).

**BR-MOD-65 — Generation must be a positive JSON-safe integer and strictly advance.** `1..=2^53-1` at validation (`push_lease.rs:198-200`); the strict-advance watermark is enforced in the DB, surfacing as `AcceptLeaseOutcome::StaleGeneration` → `invalid: stale generation` (`ingest.rs:2191-2193`).

**BR-MOD-66 — Duplicate JSON keys are rejected at every depth.** Custom `DeserializeSeed` (`push_lease.rs:383-455`); pinned by a test that covers both top-level and nested-filter duplicates (`:637-649`).

**BR-MOD-67 — An inactive lease must use the minimal schema.** Only `v`/`origin`/`generation`/`active` are permitted; presence of `app_profile`/`transport`/`endpoint`/`subscriptions` ⇒ `inactive lease must use minimal schema` (`push_lease.rs:210-218`), and the key-set check rejects them earlier as `unknown field: {key}` (`:159-190`; test `:651-663`).

**BR-MOD-68 — Only 2 app profiles and 3 classes are advertised, and transport must match the profile.** Profiles `buzz-ios-production` / `buzz-ios-sandbox`, both `apns` (`push_lease.rs:503-512`); classes `silent`/`default`/`time_sensitive` (`:509`). `transport mismatch` at `:230-232`.

**BR-MOD-69 — The `urgent` class is structurally unreachable.** `URGENT_KINDS` is empty (`push_lease.rs:16`) and `"urgent"` is absent from `supported_classes` (`:509`), so `class not supported` (`:245-247`) fires before the urgent-kind confinement check at `:281-283` can ever run. `class_rank` still ranks `urgent` highest in both copies (`:582`, `push_runtime.rs:574`), and NIP-11 advertises `urgent_kinds: []` (`nip11.rs:209`).

**BR-MOD-70 — Every positive lease filter must be narrowed and self-scoped.** A positive filter needs at least one of `authors`/`#p`/`#h` ⇒ else `lease filter not narrowed` (`push_lease.rs:292-295`); every `#p` value must equal the lease author ⇒ `p-tag must be self` (`:298-303`). `ignore` filters are exempt from narrowing and from urgent confinement because they can only subtract (`:251-255`, `:263` with `require_narrowing=false`).

**BR-MOD-71 — Push-eligible kinds are a closed set of 5.** `PUSH_KINDS = [7, 9, 1059, 40007, 46010]` (`push_lease.rs:15`); any other kind ⇒ `kind not push-eligible` (`:277-280`). A unit test asserts the DB trigger allowlist in `migrations/0018_push_match_queue.sql` uses this exact list (`push_lease.rs:696-710`) — a genuinely strong cross-artifact invariant.

**BR-MOD-72 — Quotas are hard-coded, not configurable.** 16 active leases per author (`push_lease.rs:479`), 16 subscriptions, 16 kinds, 20 authors, 50 `#h`, 20 generic tag values, 8 ignore filters, 4096-byte endpoint, 512-byte strings (`:513-521`), 64 KiB ciphertext (`:477`), 32 KiB plaintext (`:478`). Note `max_h = 50` is deliberately independent of `max_tag_values = 20` (test `:734-741`).

**Lease replacement rules** (enforced in the DB transaction, surfaced by ingest): NIP-01 addressable ordering (`StaleEvent`), generation watermark (`StaleGeneration`), one active address per endpoint tuple (`EndpointAlreadyLeased`), per-author quota (`LeaseQuotaExceeded`), source-event uniqueness (`SourceEventCollision`) — `ingest.rs:2185-2212`, outcomes at `buzz-db/src/push.rs:190-208`.

---

#### J. Push matcher / delivery rules (BR-MOD-73 … BR-MOD-83)

**BR-MOD-73 — Batch claim with a 30 s lease and a 64-job ceiling.** `CLAIM_SECS = 30`, `MATCH_BATCH_LIMIT = 64`, bounded under `get_events_by_ids`' 500-id contract (`push_runtime.rs:15`, `:18-20`, claim `:69-74`).

**BR-MOD-74 — Idle backoff 250 ms → 2 s, reset on any work.** `IDLE_POLL_FLOOR`/`IDLE_POLL_CEILING` (`push_runtime.rs:22-25`); doubling at `:80-81`, reset at `:76`. Delivery worker uses its own 500 ms floor with the same 2 s ceiling (`:317`, `:341-345`).

**BR-MOD-75 — Poison-job reaping runs off the claim path, every 30 s.** `REAP_INTERVAL` (`push_runtime.rs:26-29`), because the reap scan is not served by the due partial index and would slow claims exactly when a backlog needs them fastest (`:26-28`); executed at `:60-68`.

**BR-MOD-76 — Whole-batch retry when shared context fails to load.** No job was evaluated, so all ids are released with a 2 s delay (`push_runtime.rs:127-149`).

**BR-MOD-77 — 8 match attempts then discard, not infinite retry.** `job.attempt >= buzz_db::push::MAX_MATCH_ATTEMPTS` (=8, `buzz-db/src/push.rs:70`) ⇒ marked complete so a poison event cannot retry forever or pin outbox retention (`push_runtime.rs:168-176`).

**BR-MOD-78 — Match evaluation is pure; the wake enqueue is one transaction per batch.** `match_job` does no DB access (`push_runtime.rs:216-290`); all wakes flush in a single `enqueue_push_wakes` (`:184-198`). Enqueue failure sends only the contributing jobs back for an idempotent rematch, absorbed by the outbox dedup key (`:179-183`).

**BR-MOD-79 — Three authorization layers per (event, lease) pair.** `reader_authorized_for_event` on the lease author (`push_runtime.rs:222-225`); channel membership from the pre-loaded `(channel, author)` pair set (`:226-233`); and per-filter `push_filter_authorized_for_event` (`:250`).

**BR-MOD-80 — Gift wraps (kind 1059) may only match a lease whose filter is self-`#p` AND whose event is addressed to that author.** `push_runtime.rs:292-310` — a match-time counterpart of REQ's `#p` gate, because 1059 is globally stored and leaks recipient activity through wake timing (`:287-290`). All non-1059 kinds pass this check unconditionally (`:296-298`). Tested at `:580-620`.

**BR-MOD-81 — Wake class is the max across matching subscriptions; suppression and ignore-filters subtract.** Highest `class_rank` wins (`push_runtime.rs:268-270`); a matching `ignore` filter or a `p` tag count exceeding `suppress.p_tags_max` skips the subscription (`:255-267`).

**BR-MOD-82 — Wake deadline is `min(lease expiry, event created_at + 3600 s)`, and an already-expired wake is dropped.** `EVENT_USEFUL_SECS = 3600` (`push_runtime.rs:16`); computed `:272-276`.

**BR-MOD-83 — Delivery revalidates three times, with membership sandwiched between two generation fences.** `revalidate_push_wake` (`push_runtime.rs:354-370`) → channel membership (`:372-402`) → `revalidate_push_wake` again as the final DB op before transport (`:403-420`), because membership I/O can race lease replacement (`:403-404`). `Suppressed` ⇒ `fail_push_wake`; a revalidation *error* returns without touching the row, letting the claim lease expire.

**Retry schedule:** `retry_or_fail` backs off `delay * 2^(attempt-1)` clamped to `2^6`, failing permanently at `MAX_ATTEMPTS = 8` (`push_runtime.rs:531-550`, const `:17`).

**Endpoint invalidation is generation-fenced:** a 410 only disables the endpoint when the gateway-reported generation equals the wake's (`push_runtime.rs:452-466`) — so a stale 410 cannot kill a freshly rotated lease.

**404 after the first attempt is treated as success.** A timed-out terminal attempt burns the stable request id; its replay is indistinguishable from an invalid-grant 404, and sending a fresh id would double-deliver (`push_runtime.rs:488-497`). This deliberately swallows genuine 404s on retry.

---

#### K. Identity-archive consent rules (BR-MOD-84 … BR-MOD-91)

**BR-MOD-84 — Three consent paths, evaluated in a fixed order.** Self (`actor == target`) → community `owner`|`admin` role → cryptographic NIP-OA owner (`identity_archive.rs:228-251`). Note the naming inversion: a community owner/admin yields `ConsentPath::Admin` (`:245-247`); `ConsentPath::Owner` means *key owner of the target agent* (`:250`).

**BR-MOD-85 — The scope gate is deliberately weak because the handler owns authorization.** 9035/9036 require only `Scope::UsersWrite`, not `AdminUsers`, since self and owner-of-agent paths are open to ordinary users (`ingest.rs:258-266`).

**BR-MOD-86 — Owner consent requires a valid NIP-OA auth tag on the *request*, owned by the request signer.** Exactly one 4-element `auth` tag (`identity_archive.rs:300-318`); `verify_auth_tag(tag, target_pubkey)` must return the actor (`:260-266`).

**BR-MOD-87 — Owner consent additionally requires the target's *live* kind:0 to still attest the same owner.** A global-only kind:0 query for the target (`identity_archive.rs:270-284`), author cross-check (`:286-289`), and a second `verify_auth_tag` whose owner must equal the actor ⇒ else `live kind:0 no longer attests to request signer` (`:290-295`). This is the revocation path: replacing your kind:0 auth tag immediately invalidates outstanding owner-signed archive requests. Covered by the module's one integration test (`:515-578`).

**BR-MOD-88 — Auth-tag `created_at<`/`created_at>` conditions are enforced against the request's own timestamp.** `identity_archive.rs:328-355`; unknown clauses such as `kind=1` are ignored (test `:424-431`).

**BR-MOD-89 — Exactly one NIP-70 `["-"]` protected tag is mandatory.** Zero or ≥2 ⇒ rejected (`identity_archive.rs:156-167`).

**BR-MOD-90 — Exactly one valid `p` tag; `replaced-by` must be lowercase 64-hex, distinct from the target, and singular.** `extract_single_p_tag_hex` returns `None` on a second `p` tag (`:170-187`); `replaced-by` rules at `:200-226`. `replaced-by` on a 9036 unarchive is rejected (`:60-62`).

**BR-MOD-91 — Archiving is idempotent and event emission is gated on actual change.** `archive` uses `ON CONFLICT DO NOTHING` and does not mutate an existing row (`buzz-db/src/archived_identities.rs:66`); `!changed` ⇒ early `Ok(())` with no delta or snapshot published (`identity_archive.rs:100-102`). Delta/snapshot publish failures only warn (`:130-136`), so `archived_identities` can diverge from the 8002/8003/13535 event view.

**Archiving is explicitly not a ban** — it does not affect membership, relay access, or repository permissions (`buzz-db/src/archived_identities.rs:3-6`).

---

#### L. Product-feedback rules (BR-MOD-92 … BR-MOD-94)

**BR-MOD-92 — Feedback is sidecarred out of the event store entirely.** No `events` row, no fan-out (`ingest.rs:1562-1578`).

**BR-MOD-93 — At most one category from a closed 3-value set; category is optional.** `["bug","praise","needs-work"]` (`product_feedback.rs:11`, `:78-92`).

**BR-MOD-94 — Body must be non-blank and ≤32 KiB; the full tag array is stored and capped at 64 KiB.** `:94-104`, `:70-76`. `imeta` tags, if present, are URL-validated against the tenant media base and their blobs verified before insert (`:26-36`).

---

#### M. Storage-sweep rules (BR-MOD-95 … BR-MOD-101)

**BR-MOD-95 — Leader-only, single-flight, never awaited by the tick.** Both entry points are called from the leader-only branch of `run_usage_metrics_tick` (`main.rs:1423-1430`); `maybe_spawn_sweep` returns immediately if `in_flight` is unfinished (`storage_sweep.rs:161-165`). This is the module's leader election — a real Postgres advisory lock via `buzz_db::UsageMetricsLeader` (`main.rs:1414-1421`).

**BR-MOD-96 — Harvest and spawn share one lock acquisition, by design.** Splitting them would let a tick observe a freshly-emptied `in_flight` from a harvest that has not yet updated `cached`, and spawn a redundant sweep (`storage_sweep.rs:143-149`).

**BR-MOD-97 — Cadence rule: spawn iff nothing in flight AND (never attempted OR last attempt failed OR cache older than `interval`).** `should_spawn` (`storage_sweep.rs:105-127`). A failed attempt respawns on the very next usage tick (default 300 s, `main.rs:1257-1262`) rather than waiting the full hour — documented as intentional because a permission failure costs one cheap LIST and the sweep self-heals when fixed (`storage_sweep.rs:89-103`).

**BR-MOD-98 — A warm cache survives later failures; a cold cache stays cold.** Failure increments `failures_total` and updates `last_attempt` but never clears `cached` (`storage_sweep.rs:175-184`); tests `:531-591`, `:493-528`.

**BR-MOD-99 — Task panic is counted as a failure, not propagated.** `JoinError` ⇒ `failures_total += 1`, `last_attempt = { ok: false, duration: ZERO }` (`storage_sweep.rs:194-202`).

**BR-MOD-100 — The kill switch suppresses *all* storage gauges including health ones.** `BUZZ_STORAGE_METRICS=off` ⇒ `run_storage_sweep_tick` returns before touching anything (`main.rs:1454-1456`), so a relay without `s3:ListBucket` can disable the feature cleanly (`storage_sweep.rs:42-45`).

**BR-MOD-101 — A completed-but-unharvested sweep publishes nothing.** Harvest only happens inside `maybe_spawn_sweep`, which a demoted leader stops calling — so its snapshot sits in `in_flight` forever (`storage_sweep.rs:643-670` test; rationale `:611-628`). Per-community series that disappear from the snapshot (removed, host-renamed, or scope-excluded) are explicitly zeroed rather than left stale (`storage_sweep.rs:333-345`; test `:857-1088`). Unmapped community UUIDs roll into `buzz_storage_unmapped_community_bytes` instead of a per-community series (`:318-323`, `:330`).

**Budget:** `interval` floored at 60 s (`storage_sweep.rs:59-60`), whole-attempt `timeout` (`:249-252`), cumulative `max_objects` cap checked before folding each page (`buzz-media/src/bucket_index.rs:392-395`), 1000 objects per LIST page (`main.rs:1466`). Every `SweepError` variant means "keep the old snapshot", never a partial one (`bucket_index.rs:337-338`).

---

#### N. Workflow-action authorization (BR-MOD-102 … BR-MOD-108)

**BR-MOD-102 — The run's community is authoritative; the tenant is never re-derived from config.** `lookup_community_host(community_id)` reads the host back for labelling only, and fails closed if the community no longer maps to a host (`workflow_sink.rs:190-210`). In-code rationale: re-deriving from `config.relay_url` would post a community-B workflow's output into the deployment default under N>1 (`:191-199`).

**BR-MOD-103 — Empty/whitespace-only text is refused.** `ActionSinkError::EmptyContent` (`workflow_sink.rs:212-214`).

**BR-MOD-104 — Channel must exist, be a canonical UUID, and not be archived.** `Uuid::parse_str` then canonicalized (`:217-220`); `ChannelNotFound` mapped from `DbError::ChannelNotFound|NotFound` (`:222-231`); `ChannelArchived` at `:232-236`.

**BR-MOD-105 — The workflow *owner* must be a member unless the channel is open.** `!is_member && channel.visibility != "open"` ⇒ `workflow owner does not have access to destination channel` (`workflow_sink.rs:243-251`). This is the only authorization on the action; the event is then signed by the relay key (`:304`) with a `p` tag attributing the owner (`:260-261`).

**BR-MOD-106 — Mention resolution is deliberately conservative and defines the wake contract.** Members only, exact display name, case-insensitive, greedy-longest non-overlapping, ambiguous names wake nobody (`workflow_sink.rs:22-43`, impl `:45-140`). Boundary rule: `@` must sit at start/whitespace/`(` and the name must not be followed by an alphanumeric or `_` (`:98-101`). Case folding is done on the fly in original-char coordinates because `İ`(U+0130) lowercases to two code points (`:78-96`) — regression-tested at `:490-503` and `:544-560`.

**BR-MOD-107 — Workflow messages are always top-level.** `depth: 0`, no parent, no root, `broadcast: false` (`workflow_sink.rs:318-335`). A workflow can never reply into a thread.

**BR-MOD-108 — `buzz:workflow` tag prevents recursive triggering; post-persist side effects only fire on real insert.** Tag at `workflow_sink.rs:264-266`; `if was_inserted` guard at `:349`. The `dispatch_persistent_event` result is discarded with `let _ =` (`:351`), so fan-out/search/audit failures are invisible.

> **Verified against ARCHITECTURE.md §9:** items 5 (`:826`, approval gates) and 6 (`:827`, `send_dm`/`set_channel_topic` `NotImplemented`) are accurate. From the sink side the gap is wider — `ActionSink` declares exactly one method (`buzz-workflow/src/action_sink.rs:44-64`), so only `send_message` is wired end-to-end, and `add_reaction` is a **third** broken action (POSTs to `/api/messages/{id}/reactions`, `buzz-workflow/src/executor.rs:886-888`, which is not registered in `router.rs`). ARCHITECTURE.md:541 lists `add_reaction` as a working action with no caveat.


## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Business Rules

Every rule is transcribed from code, not doc comments. Where the doc and the code
disagree, the delta is called out inline.

---

#### 1. Admission & capacity

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-01 | A huddle-audio connection is bound to a community from the **request `Host` header** before the WebSocket upgrade; an unmapped host fails closed with a generic 404, never a default tenant | `tenant::bind_community(&state.db, raw_host)` | `handler.rs:74-88` |
| BR-AU-02 | An audio connection consumes one permit from the **same** `conn_semaphore` as ordinary relay WebSockets; exhaustion → 503 | `try_acquire_owned()`; permit is moved into the connection task and dropped on exit | `handler.rs:90-99`, `:110-114`, `:167` |
| BR-AU-03 | A connection to a community whose `is_community_active` is false is dropped silently | `run_registered_community_connection` | `handler.rs:156-164` |
| BR-AU-04 | Every WS message is capped at 8192 bytes **at the parser**, before the handler reads it | `max_message_size` + `max_frame_size` | `handler.rs:52`, `:116-119` |
| BR-AU-05 | Auth must complete within 5 s of the challenge or the socket is closed silently | `tokio::time::timeout(AUTH_TIMEOUT, …)` | `handler.rs:61`, `:190` |
| BR-AU-06 | Room capacity is `min(MAX_PEERS_PER_ROOM=25, 255-index-space)`. The cap check is **inside** the admission mutex, so two concurrent joiners cannot both pass | `self.peers.len() >= MAX_PEERS_PER_ROOM` under `guard.lock()` | `room.rs:50`, `:233-238` |
| BR-AU-07 | `peer_index` allocation is 0..=254; `alloc()` returns `None` at `next_fresh == 255`, so **index 255 is never issued** and the effective hard cap is 255 peers | `AdmissionGuard::alloc` | `room.rs:143-153` |
| BR-AU-08 | Freed indices are recycled LIFO (`Vec::pop`) before fresh ones are minted | `AdmissionGuard::{alloc, release}` | `room.rs:144-146`, `:155-157`; pinned `room.rs:730-757` |
| BR-AU-09 | Admission error precedence is **`Ended` > `Full` > `VersionMismatch`** — a client that could not get a seat must not learn the room's pinned protocol version | ordered `if` chain under one lock | `room.rs:220-224` (rationale), `:230-243`; pinned `room.rs:759-782` |
| BR-AU-10 | A room is keyed by **(community, channel)**, so two tenants reusing one channel UUID never share peers, pins, or frames | `DashMap<(CommunityId, Uuid), …>` | `room.rs:490`, `:503-509`; pinned `room.rs:684-728` |

---

#### 2. Authentication & membership

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-11 | Authentication is NIP-42: the relay issues a challenge, the client returns a signed auth event, and `state.auth.verify_auth_event(event, challenge, relay_url)` must accept it | same `buzz_auth` verifier the main relay door uses | `handler.rs:176-238` |
| BR-AU-12 | The expected relay URL for the NIP-42 event is derived **per tenant**, not from a global constant | `api::bridge::nip42_expected_relay_url(&state.config.relay_url, &tenant)` | `handler.rs:219` |
| BR-AU-13 | Relay-level membership is checked via `enforce_relay_membership`, which is a **no-op when `require_relay_membership == false` — the default** | `api/mod.rs:67` early-returns `OpenRelay`; `api/mod.rs:130-131` maps that to `Ok` | `handler.rs:244-262` |
| BR-AU-14 | An **archived** channel is rejected before any membership check, so an auto-ended huddle can never be rejoined | `channel.archived_at.is_some()` first in `ensure_membership` | `handler.rs:1162-1170` |
| BR-AU-15 | For an ephemeral (TTL) channel, the lifecycle parent is **not** taken from the client-supplied `parent_channel_id` on trust: a creator-signed kind-48100 link must exist in the DB | `db.huddle_started_link_exists(community, parent, channel, channel.created_by)` | `handler.rs:1172-1191` |
| BR-AU-16 | A non-TTL channel's lifecycle parent is the channel itself | `else { channel_id }` | `handler.rs:1192-1194` |
| BR-AU-17 | Admission order is: existing member → `visibility == "open"` → auto-add. Anything else is `"not a member"` | sequential checks | `handler.rs:1196-1234` |
| BR-AU-18 | Auto-add applies **only** to a TTL channel whose caller is already a member of the resolved parent; the new row is `MemberRole::Member`, added_by = `channel.created_by` | `db.add_member(...)` then `invalidate_membership` | `handler.rs:1208-1231` |
| BR-AU-19 | After `get_or_create`, the archived check is **repeated** against the DB to close the cross-boundary race where a joiner passed `ensure_membership` before the last peer archived | second `db.get_channel` | `handler.rs:384-413` |
| BR-AU-20 | A DB error on that re-check **fails closed** (silent teardown), it does not fall through to admission | `Err(e) => { … return; }` | `handler.rs:404-410` |

---

#### 3. Protocol version negotiation

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-21 | `protocol_version` defaults to **1** when the auth message omits it, so pre-v2 clients keep working | `#[serde(default = "default_protocol_version")]` | `handler.rs:134-142` |
| BR-AU-22 | A requested version of `0` or `> CURRENT_PROTOCOL_VERSION (2)` is refused with `unsupported_version` **before** the room is touched, so an unknown version can never become a room's pin | range check after the archived re-check | `handler.rs:123-124`, `:415-441` |
| BR-AU-23 | The **first successful admission pins the room's version**; the pin is set *after* index allocation so a `Full` rejection cannot pin | `g.pinned_version.get_or_insert(requested_version)` following `g.alloc()?` | `room.rs:244-247`, `:311` |
| BR-AU-24 | A later joiner whose version differs from the pin is refused `upgrade_required` with both `pinned_version` and `requested_version` | `AdmissionError::VersionMismatch` → `handler.rs:534-548` | `room.rs:239-243` |
| BR-AU-25 | The pin **survives full peer churn** while the `Room` object lives, and is only cleared when `cleanup_if_empty` evicts the room | pin lives in `AdmissionGuard`, dropped with the `Room` | `room.rs:110-127`; pinned `room.rs:661-682`, `:730-757` |
| BR-AU-26 | Frames from a v≥2 room must carry the 8-byte header **plus** a non-empty Opus payload; otherwise they are dropped (frame only, not the connection) | `data.len() <= V2_HEADER_LEN` then `FrameHeader::parse` with `!payload.is_empty()` | `handler.rs:975-1017` |
| BR-AU-27 | The relay never strips, rewrites, or re-encodes a client frame. A malformed `level_dbov` is **clamped in the parsed view only** — the forwarded bytes are unchanged, and the frame is never dropped for bad telemetry | `parse` clamps to `-127`; `broadcast_frame`/`forward_media` take the original `data` | `wire.rs:73-77`, `:14-20`; `handler.rs:1017-1022` |
| BR-AU-28 | `level_dbov` MUST NOT feed any trust decision (admission, moderation, kicks) | stated invariant `wire.rs:16-20`; verified: the only consumer is `tracing::trace!` | `handler.rs:996-1003` |

---

#### 4. Frame fan-out

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-29 | Fan-out is "everyone except the sender", identified by peer UUID on the local path | `if *entry.key() == sender_id { continue; }` | `room.rs:406-408` |
| BR-AU-30 | On the mesh path the author is identified by **`peer_index`**, not peer UUID, so a participant whose own frame round-tripped owner→their-pod does not hear themselves | `deliver_prefixed` skips `entry.peer_index == author_index` | `room.rs:422-429`; `mesh.rs:246-247` |
| BR-AU-31 | Audio uses `try_send` **everywhere** — a full 8-slot per-peer queue drops the frame silently, with no counter and no log | 3 sites | `room.rs:409`, `:427`; `handler.rs:1115` |
| BR-AU-32 | Control JSON uses a separate 32-slot queue; a drop is logged as a warning because joined/left are state-bearing | `broadcast_control` warns on `try_send` failure | `room.rs:437-449` |
| BR-AU-33 | The outbound WS loop **drains all pending control frames before any data frame**, on every iteration and again in a `biased` select | `while let Ok(ctrl_msg) = ctrl_rx.try_recv()` then `biased` | `handler.rs:1060-1086` |
| BR-AU-34 | A frame from a sender not present in the peer map is discarded without error | `peers.get(&sender_id)` → `None => return` | `room.rs:394-397` |
| BR-AU-35 | On the ingress (non-owner) path there is **no local fan-out at all**: media goes only to the owner, which fans it back — including to co-located clients | `match remote_session { Some(s) => s.forward_media(&data), None => room.broadcast_frame(...) }` | `handler.rs:1017-1022`; rationale `join.rs:1453-1462` |
| BR-AU-36 | Media forwarding to the owner is drop-on-error; a dead link never blocks the receive loop | `if let Err(e) = self.transport.send_datagram(...) { debug!(...) }` | `join.rs:1760-1765` |

---

#### 5. Speaking / mute semantics

| ID | Rule | Status |
|---|---|---|
| BR-AU-37 | The relay has **no** notion of mute, speaking, active speaker, or dominant talker. There is no server-side mute state, no `mute` control message, and no speaker-election logic | Verified: grep for `mute\|speaking\|active_speaker\|dominant` across `crates/buzz-relay/src/audio/` finds only two doc-comment mentions (`mod.rs:5`, `wire.rs:14`) |
| BR-AU-38 | Active-speaker detection is entirely client-side, from `level_dbov` | `desktop/src-tauri/src/huddle/playout.rs:220` emits `huddle-active-speakers` |
| BR-AU-39 | Mute is client-side suppression of transmit; the relay cannot distinguish a muted peer from a silent one, except that a DTX-flagged frame is still forwarded | `FLAG_DTX` is parsed and traced only (`handler.rs:1001`) |
| BR-AU-40 | The `PeerCtrl::Json(...)` doc claims control messages include a `speakers` type (`room.rs:33`). **No `speakers` message is ever produced.** Documented delta | Verified: no `speakers` literal in `crates/buzz-relay/src` |

---

#### 6. Room ownership & ending (single-pod)

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-41 | A huddle has **no human owner or moderator**. Room lifetime is purely reference-counted on connected peers | no owner/creator field on `Room` (`room.rs:157-170`) |
| BR-AU-42 | When the **last** peer leaves, the room auto-ends: removal, index recycle, roster bump and `ended = true` all happen under one mutex acquisition | `remove_peer_and_check_ended` | `room.rs:358-388` |
| BR-AU-43 | Exactly one leaver wins the auto-end, so two simultaneous disconnects cannot both archive or both emit 48103 | `if !g.ended && self.peers.is_empty() { g.ended = true; true }` | `room.rs:378-384` |
| BR-AU-44 | Auto-end archives the channel in the DB **before** emitting 48103 | `db.archive_channel` then `emit_participant_event(48103)` | `handler.rs:835-860` |
| BR-AU-45 | If `archive_channel` fails, the end is **rolled back**: `clear_ended()` runs, `room_emptied = false`, no 48103 is emitted, and the huddle stays alive | `Err(e) => { room.clear_ended(); room_emptied = false; }` | `handler.rs:840-845` |
| BR-AU-46 | 48102 (participant left) is emitted on **every** disconnect path, including the ingress path and including when the archive fails | unconditional call | `handler.rs:822-831` |
| BR-AU-47 | An **ingress (non-owner) peer never archives** huddle state: it removes locally and leaves room lifetime to the owner | `if remote_session.is_some() { room.remove_peer(peer_id); false }` | `handler.rs:803-810` |
| BR-AU-48 | An ingress peer also does **not** broadcast `left` locally — the owner's roster delta carries it | `if remote_session.is_none() { room.broadcast_control(left_msg) }` | `handler.rs:818-820` |
| BR-AU-49 | A lifecycle event whose DB insert reports a **duplicate** skips fan-out entirely, to avoid double delivery | `Ok((_, false)) => return` | `handler.rs:1285-1295` |
| BR-AU-50 | A lifecycle event whose DB insert **errors** is still fanned out from an in-memory `StoredEvent`, accepting that late joiners will never see it | `StoredEvent::new(event.clone(), …)` | `handler.rs:1296-1307` |
| BR-AU-51 | Local fan-out precedes Redis publish and is preceded by `mark_local_event`, so the Redis echo is suppressed on this node; a publish failure invalidates that marker | `mark_local_event` → `fan_out_event_to_local_subscribers` → `publish_event` → invalidate on error | `handler.rs:1309-1331` |

---

#### 7. Reconnection / rejoin

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-52 | Heartbeat: a ping every 30 s; 3 consecutive missed pongs closes the connection. `missed_pongs` is incremented **before** the ping is sent and reset by any inbound Pong | `fetch_add(1) + 1 >= MAX_MISSED_PONGS` | `handler.rs:55-58`, `:1127-1151`, `:1038-1040` |
| BR-AU-53 | Effective idle detection window is 60–90 s (increment happens at tick, so failure is observed on the 3rd tick) | derived from `handler.rs:1132-1140` |
| BR-AU-54 | There is **no session resumption**. A reconnect is a brand-new NIP-42 handshake, a new peer UUID, and (usually) a different `peer_index` — clients must rebuild their index→pubkey map from `joined`/`roster` | `Uuid::new_v4()` per admission (`room.rs:255`) |
| BR-AU-55 | A rejoin is refused if the channel was archived by the previous session's auto-end | BR-AU-14 + BR-AU-19 | `handler.rs:1162-1170`, `:389-403` |
| BR-AU-56 | Owner-loss and owner-drain both **close local owner clients so they rejoin**; the relay never migrates a session | `owner_cancel.cancel(); fence.forget(channel_id)` | `handler.rs:727-772` |
| BR-AU-57 | On any teardown cause (`OwnerLost`, `OwnerDraining`, `SessionEnded`, `StreamClosed`) the ingress path takes the **same** action — cancel + `fence.forget`. The cause is observability only | `teardown_remote_huddle` ignores `cause` except for logging | `handler.rs:897-910`; doc `join.rs:1483-1495` |
| BR-AU-58 | `GenerationFloor::forget` clears **local stale-frame suppression only**; it never authorises ownership. Redis fenced CAS remains the arbiter | stated `handler.rs:889-895`, `mesh_boot.rs:154-163`; verified `forget` only does `seen.remove` | `mesh.rs:131-133` |

---

#### 8. Mesh owner / forwarding protocol

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-59 | A huddle's `session_id` **is** its `channel_id` | `resolve_join_owner_ready(…, channel_id, …)` | `handler.rs:324`; doc `join.rs:20` |
| BR-AU-60 | Exactly one pod owns a huddle: the holder of the Redis fenced CAS lease. Mesh membership is a routing hint and **never** grants ownership | doc `join.rs:22-38`; verified — every ownership decision goes through `HuddleDirectory` | `join.rs:317-379` |
| BR-AU-61 | `resolve_join` looks up the live lease **first** and only attempts `acquire` when unowned, so the steady state avoids the generation-INCR race window | `owner_of` → `None` → `acquire` | `join.rs:329-347` |
| BR-AU-62 | A lost CAS is not an error: the winner becomes the routing target (`RemoteOwner`) | `AcquireOutcome::Held(o) => o` | `join.rs:344-346`; pinned `join.rs:2035-2054` |
| BR-AU-63 | `ResolvedJoin.acquired` is `Some` **only** on the CAS-win arm. The steady-state owner reuses the registry's existing renewer instead of rebuilding authority from a generation snapshot | `acquired: None` on both non-acquire arms | `join.rs:293-303`, `:349-378` |
| BR-AU-64 | The remote-owner arm is fence-validated against Redis **before** a control stream is opened (origin-side hop) | `directory.validate(community, &fenced).await?` | `join.rs:365-372`; pinned `join.rs:2018-2033`, `:2056-2066` |
| BR-AU-65 | **The registry, not the `resolve_join` snapshot, gates reuse.** A `LocalOwner`+`acquired=None` result with no live registry entry is re-resolved in a bounded loop rather than admitting an owner peer with no loss watcher | `if owners.lost_for(session_id).is_some() { return }` else sleep+retry | `join.rs:426-441` |
| BR-AU-66 | That loop runs at most **25 attempts × 20 ms ≈ 500 ms** and then **fails closed** with a transient transport error, surfaced to the client as `join_rejected` | `OWNER_READY_MAX_ATTEMPTS`, `OWNER_READY_RETRY_INTERVAL` | `join.rs:387-388`, `:444-447`; three paths pinned `join.rs:2657-2745` |
| BR-AU-67 | Only the reuse arm may loop; CAS-win and remote-owner arms return immediately | `_ => return Ok(resolved)` | `join.rs:439` |
| BR-AU-68 | A `LocalOwner` reuse that finds `lost_for == None` after the loop is an **invariant violation**, logged at `error!` — not a benign race | `error!("huddle owner-ready invariant violated: …")` | `handler.rs:586-601` |
| BR-AU-69 | The owner-lease renewer is **per-room, not per-connection**: N local owner joiners must not each spawn a renewer | `HuddleOwnerRegistry` keyed by `session_id` | `join.rs:566-598`, `:665-724` |
| BR-AU-70 | If `attach_signals` finds a live entry, the existing renewer wins and the just-acquired extra lease is released by cancelling a throwaway renewer — **no lease leak, no double renewer** | pre-cancelled token handed to a discarded renewer | `join.rs:679-692`; pinned `join.rs:2470-2504` |
| BR-AU-71 | `release` and `drain` are **generation-fenced**: a stale room-empty cannot tear down a newer epoch installed by a re-acquire | `remove_if(|_, e| e.generation == generation)` | `join.rs:734-742`, `:750-761`; pinned `join.rs:2506-2552`, `:2626-2650` |
| BR-AU-72 | Exactly one `release` fires per room epoch, because only the last leaver sets `room_emptied` | `if room_emptied { mesh.owners.release(channel_id, generation) }` | `handler.rs:868-877` |
| BR-AU-73 | Renewal cadence is 10 s against a 30 s lease TTL → three attempts per lifetime. Missed ticks use `MissedTickBehavior::Delay` | `DEFAULT_HUDDLE_RENEW_INTERVAL`, mirrored from the reliable lane | `join.rs:452`, `:501-502`; `directory.rs:17`; `reliable.rs:34` |
| BR-AU-74 | Owner-loss is `Lost`, a renew error, a release `NotOwner`, **or** a release error on a non-caller-cancel exit. A caller-initiated cancel is normal teardown and stays silent | `if !caller_cancelled { lost_for_task.cancel() }` | `join.rs:504-560`; pinned `join.rs:2377-2422` |
| BR-AU-75 | Once `drain_all()` runs the registry is **permanently** draining: every later `attach_signals` immediately releases the new lease and returns a pre-cancelled `draining` token | sticky `AtomicBool` + early return | `join.rs:667-677`, `:765-773`; pinned `join.rs:2600-2624` |
| BR-AU-76 | `attach_signals` closes its own check→insert race with `drain_all` by re-checking after publication and retracting the exact epoch it installed | second `is_draining()` + generation-fenced `remove_if` | `join.rs:706-720` |
| BR-AU-77 | A new owner admission is refused up-front when the registry is draining (`huddle_relay_draining`) | `if mesh.owners.is_draining()` before `resolve_join_owner_ready` | `handler.rs:308-320` |
| BR-AU-78 | Owner-side: the community is **learned from the first `RegisterPeer`** and latched; a later frame naming a different community tears the stream down | `stream_community` latch | `join.rs:1200-1209` |
| BR-AU-79 | **Validate before admit.** The Redis fence runs on `RegisterPeer` before `room.add_peer`; a wrong community keys a lease that does not exist → `no_active_lease`, so no peer is admitted | `directory.validate(community, &fenced)` then `register_remote_peer` | `join.rs:1231-1245`; pinned `join.rs:2332-2392` |
| BR-AU-80 | A **non-fence** validate error (Redis unreachable, decode) is not a clean rejection — it tears the stream down | `None => break Err(e)` | `join.rs:1240-1244` |
| BR-AU-81 | `UnregisterPeer` needs no fence because it can only remove entries from **this stream's own** `registered` map | `registered.remove(&pubkey)` gate | `join.rs:1264-1288` |
| BR-AU-82 | Every control frame must carry the **same** fenced header the `Hello` did; a lease that moves mid-stream rejects subsequent frames | `if f != fenced { break Err(OwnerMismatch) }` | `join.rs:1191-1198` |
| BR-AU-83 | Control-stream teardown **always** drops every peer that stream registered, whatever the exit path, so no leaked remote peer holds an index | unconditional loop after the `result` binding | `join.rs:1345-1367`; pinned `join.rs:2253-2330` |
| BR-AU-84 | The owner is the **sole allocator** of `peer_index`, so indices cannot collide across pods | ingress calls `add_peer_at_index(owner_assigned)` | `handler.rs:507-509`; doc `mesh.rs:18-24` |
| BR-AU-85 | Remote registration happens **before** local ingress admission, so the owner-assigned index is the only index the client ever has — no frame or `joined` can escape with a placeholder | `dial_remote_owner` at `handler.rs:444-503` precedes `add_peer_at_index` at `:507` | `handler.rs:443-511` |
| BR-AU-86 | `add_peer_at_index` refuses an index already in use, removes it from the free list, and advances `next_fresh` past it so a later local allocation cannot collide | `peers.iter().any(...)`, `free.retain`, `next_fresh = idx+1` | `room.rs:288-306`; pinned `room.rs:559-575` |
| BR-AU-87 | Roster deltas are ordered and monotone; an ingress receiver that detects a gap requests `RosterResync` and **does not apply the gapped delta** | `next == revision.wrapping_add(1)` guard, else send `RosterResync` | `join.rs:1560-1597`; pinned `join.rs:2113-2183` |
| BR-AU-88 | A stale delta (`next <= revision`) is silently ignored | `Ok(RosterDelta{revision: next, ..}) if next <= revision => {}` | `join.rs:1583` |
| BR-AU-89 | Owner-side roster subscription happens **before** admission, and `PeerRegistered` carries a post-admission snapshot, so queued deltas at or below that revision are safely ignorable | `subscribe_roster()` at `:1229`, snapshot in the reply at `:1401` | `join.rs:1225-1263` |
| BR-AU-90 | A lagged owner-side `roster_rx` recovers by sending a full `RosterSnapshot`, never by skipping deltas | `RecvError::Lagged(_) => roster_snapshot_msg(&room)` | `join.rs:1174-1182` |
| BR-AU-91 | The media fence is **monotone-only**: accept `>= floor`, advance on `>`, reject `<`. A rejected frame does not move the floor | `GenerationFloor::check` | `mesh.rs:102-128`; pinned `mesh.rs:298-330` |
| BR-AU-92 | Dropping a stale-generation datagram is deliberately indistinguishable from packet loss for lossy audio | stated `mesh.rs:44-51`; verified — `RejectStale` returns before delivery | `mesh.rs:212-220` |
| BR-AU-93 | A media datagram whose channel UUID is **ambiguous** across two active communities is dropped (fail closed) rather than delivered to the wrong tenant | `get_unambiguous_by_channel` returns `None` on ≥2 matches | `room.rs:526-541`, `mesh.rs:221-227`; pinned `room.rs:684-728` |
| BR-AU-94 | An empty datagram payload is dropped with a warning | `payload.split_first()` → `None` | `mesh.rs:229-232` |
| BR-AU-95 | `RealtimeMedia` arriving as a **stream** (rather than a datagram) is a protocol violation and is rejected without routing | dispatcher match arm | `mesh_boot.rs:113-121`; pinned `mesh_boot.rs:611-616` |
| BR-AU-96 | Inbound mesh traffic arriving before its per-profile handler is registered is logged and **dropped**; fencing makes the peer's retry safe | `OnceLock::get()` → `None` | `mesh_boot.rs:52-55`, `:92-100`, `:122-129`; pinned `mesh_boot.rs:621-663` |
| BR-AU-97 | First handler registration wins; later registrations are logged and ignored | `OnceLock::set(...).is_err()` | `mesh_boot.rs:70-88`; pinned `mesh_boot.rs:645-647` |
| BR-AU-98 | On SIGTERM the mesh gossips `draining=true` **then** drains locally-owned huddles, each generation-fenced | drain watcher polls `shutting_down` every 500 ms | `mesh_boot.rs:481-496` |
| BR-AU-99 | `BUZZ_MESH` off is a hard no-op: no endpoint bind, no Redis write, no spawned task | early `return Ok(None)` | `mesh_boot.rs:418-421`; pinned `mesh_boot.rs:527-541` |
| BR-AU-100 | A **misconfigured but enabled** mesh is fatal at boot (bind failure or ready-registry publish failure), by explicit policy | `?` on `MeshEndpoint::bind` and `publish_ready` | `mesh_boot.rs:423-465`; rationale `:404-410` |

---

#### 9. Reliable-stream ordering & retransmission

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-101 | There is **no relay-level retransmission or ACK protocol.** Ordering and reliability come entirely from the underlying QUIC bi-stream; the relay only frames and validates | verified: no seq, no ack, no retry, no timer in `reliable.rs` |
| BR-AU-102 | First join acquires the lease and becomes owner; a join on the owner stays local; a join elsewhere opens a fenced bi-stream to the owner and sends the `Hello` first | `join` | `reliable.rs:78-126`; pinned `reliable.rs:759-841` |
| BR-AU-103 | A lease whose `profile != ReliableStream` is refused — profiles never share a session id | `ProfileMismatch` | `reliable.rs:99-105` |
| BR-AU-104 | Outbound payloads are chunked at **1 MiB** into ordered `Data` frames; QUIC preserves order across chunks | `bytes.chunks(MAX_RELIABLE_PAYLOAD_BYTES)` | `reliable.rs:274-294` |
| BR-AU-105 | Inbound acceptance validates `hello.sender == from`, `role == Session`, `profile == ReliableStream`, and `fenced.owner == local` — all structural, no Redis | `accept_inbound` | `reliable.rs:135-172` |
| BR-AU-106 | Every inbound `Data`/`Goodbye` is validated against **both** the stream's pinned fence and Redis. Unlike the media floor, a mismatch **fails the session** rather than dropping silently | `recv_validated` → `validate_frame_fence` | `reliable.rs:332-389`; rationale `:326-331` |
| BR-AU-107 | The community is latched from the first validated frame; any later frame naming a different community is `CommunityMismatch` | `validate_frame_fence` / `ensure_outbound_community` | `reliable.rs:370-388`, `:391-408` |
| BR-AU-108 | `MeshStreamFrame::{Goodbye, Hello, Gossip}` on a reliable session stream are all `UnexpectedFrame` errors — session close rides the **inner** `ReliableWireFrame::Goodbye`, not the outer mesh frame | three explicit arms | `reliable.rs:353-357` |
| BR-AU-109 | The inner frame format is validated strictly: `len >= 18`, `version == 1`, known kind, and `Goodbye` must be exactly 19 bytes with a known reason byte | `decode` | `reliable.rs:461-492`, `:509-518` |
| BR-AU-110 | Reliable-lease renewal follows the same loss contract as huddles (10 s / 30 s, caller-cancel silent) — but **no production site spawns it** | `spawn_lease_renewer_with_interval`; zero callers of `spawn_renewer`/`spawn_observable_renewer` | `reliable.rs:571-657`; stated `mesh_boot.rs:212-215` |
| BR-AU-111 | The demo echo consumer polls `shutting_down` every 100 ms and, on drain, sends `Goodbye(Draining)` if a community is latched, else bare `finish()` | `drain_tick` arm | `mesh_boot.rs:313-336` |

---

#### 10. Session-directory lease rules

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-112 | Acquire is a single Lua script: if a live lease exists, return it and **do not touch** the generation counter; otherwise `INCR` the counter and `SET` the lease with `PX ttl` | `ACQUIRE_SCRIPT` | `directory.rs:20-34`; pinned `directory.rs:650-711` |
| BR-AU-113 | The generation counter key **never expires**, so generation is monotone across lease expiry and pod death | no TTL on `…:generation` | `directory.rs:453-461`; pinned `directory.rs:712-777` |
| BR-AU-114 | Renew extends the TTL **only** if `(owner, generation)` match exactly; otherwise `Lost` | `RENEW_SCRIPT` string match | `directory.rs:36-52`, `:246-273` |
| BR-AU-115 | Release deletes **only** on an exact `(owner, generation)` match; otherwise `NotOwner` | `RELEASE_SCRIPT` | `directory.rs:54-70`, `:277-303` |
| BR-AU-116 | A malformed lease value (wrong part count, `generation == 0`, non-hex owner, unknown profile) is an error, never a silent default | `parse_lease` | `directory.rs:495-531`; pinned `directory.rs:639-648` |
| BR-AU-117 | Fence verdict order is fixed: `StaleGeneration` → `NoActiveLease` → `FutureGeneration` → `OwnerMismatch`, with `known = max(lease.generation, counter)` | `validate_fenced_header` | `directory.rs:348-439`; pinned `directory.rs:779-880` |
| BR-AU-118 | `StaleGeneration` is only raised when `known > 0`, so a never-used session does not reject generation 0 | `if known > 0 && …` | `directory.rs:382` |
| BR-AU-119 | Every fence rejection increments `mesh_fence_rejections_total{reason}` with the exact taxonomy string | `record_fence_rejection` | `directory.rs:481-483` |
| BR-AU-120 | A lease TTL that cannot be expressed as a positive `i64` ms is rejected before any Redis call | `ttl_ms` | `directory.rs:570-575` |
| BR-AU-121 | `takeover` is documented as a distinct operation but is a **verbatim delegate to `acquire`** — and has zero callers | `self.acquire(...)` | `directory.rs:233-242` |

---

#### 11. Conformance emit-coverage guard

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-122 | Every entry to a critical seam must arm an `EmitGuard`; if the seam exits with zero emits, `Drop` records `TraceAction::ImplBug{kind}`, which the checker treats as a coverage breach | `Drop for EmitGuard` | `conformance/mod.rs:403-414`; contract `:26-30` |
| BR-AU-123 | Counting is done by a **wrapper tracer** returned from `arm`; production code calls `tracer.record` as before and must thread the wrapper downstream or the counter never moves | `CountingTracer` + `arm` returning a tuple | `conformance/mod.rs:357-372`, `:383-400` |
| BR-AU-124 | The synthetic `ImplBug` lands on the **inner** tracer, not the wrapper, so it cannot recursively bump the counter | `self.inner.record(step)` | `conformance/mod.rs:412` |
| BR-AU-125 | Exactly one seam is armed today: `ingest_event`, kind `"ingest_event_exited_without_trace"` | one production `EmitGuard::arm` call site | `ingest.rs:1408-1412` |
| BR-AU-126 | The ingest wrapper also maps `IngestError → SanitizedError` in one place, so individual `return Err(...)` sites need no emit. On `Ok` it emits nothing — the inner fn's dispatch points are responsible | `if let Err(err) = &result { emit(...) }` | `ingest.rs:1432-1444`; mapping `conformance/mod.rs:422-429` |
| BR-AU-127 | `SanitizedReason` is a closed 3-value alphabet asserted 1:1 with `IngestError`; a fourth variant makes the match non-exhaustive and CI catches it | exhaustive `match` with no `_` arm | `conformance/mod.rs:422-429` |
| BR-AU-128 | **Don't normalize away violations**: `claimed_community` (from the event `h` tag) is recorded separately from `resolved_community` (from `TenantContext`), because the checker's claim≠resolved bite depends on seeing both | `claimed_community_from_event` + `state_for_request` | `conformance/mod.rs:16-21`, `:94-117` |
| BR-AU-129 | On the REQ read path `claimed_community` is **unconditionally `None`** — the REQ wire carries no client-asserted community. Encoding `None` (not a copy of resolved) is load-bearing | hard-coded `None` | `conformance/mod.rs:128-146`, `:152` |
| BR-AU-130 | A channel-scoped row whose channel id is missing from the lookup map is a **coverage breach**, emitted as `ImplBug{"row_community_lookup_missing"}`, never silently substituted with the resolved label | `RowCommunityProjection::MissingLookup` → `ImplBug` | `conformance/mod.rs:234-255`, `:286-294`, `:321-329`; pinned `:660-696` |
| BR-AU-131 | A channel-**less** row legitimately projects as the resolved community; the distinction comes from the row's own `channel_id`, not from the query filter, so a channel-scoped row cannot masquerade as channel-less | `match row_channel_id { None => resolved, Some(ch) => lookup }` | `conformance/mod.rs:186-197`; rationale `:167-184` |
| BR-AU-132 | Production binds `NoopTracer`, so **all of the above is inert in production**: every emit and every `ImplBug` is discarded | `tracer: Arc::new(crate::conformance::NoopTracer)` | `state.rs:794-798`; `tracers.rs:20-24` |
| BR-AU-133 | A `JsonlTracer` write failure loses one step but must never fail the request; the coverage guard is the safety net for systemic loss | all-`let _ =` writes | `tracers.rs:63-71` |
| BR-AU-134 | This audio module emits **no trace steps at all** — no `EmitGuard`, no `Tracer` reference anywhere in `audio/`, `tunnel/`, or `mesh_boot.rs`. Huddle joins, cross-pod registrations, and the three lifecycle-event writes are entirely outside the conformance seam | verified by grep for `conformance`/`tracer` across the group |


## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Business Rules

Numbered BR-MESH-nn. Every rule cites its enforcement site. Where a rule is
*documented but not implemented*, that is stated explicitly.

---

#### A. The governing law

**BR-MESH-01 — Membership is a hint; the Redis fenced generation is the arbiter.**
Stated at `lib.rs:33-35` and `wire.rs:19-27`. Structurally enforced by omission:
nothing in this crate reads, compares, or mutates `FencedHeader.generation`.
Verified — the four fence-rejection variants (`MeshError::StaleGeneration`,
`NoActiveLease`, `OwnerMismatch`, `FutureGeneration`, `lib.rs:96-130`) have **zero
constructors inside `crates/buzz-relay-mesh`**; all construction sites are in
`buzz-relay` (`tunnel/directory.rs:378,395,413,430,824,842,870,914`;
`audio/join.rs:1081,1201,1888,2854`). The crate defines the taxonomy and never
raises it.

**BR-MESH-02 — The mesh may say "don't dial"; it may never say "take over."**
`wire.rs:25-27`. Enforced by the seam split: `RelayMeshMembership` (`lib.rs:144-151`)
exposes only `peers()`, `local_runtime_id()`, `begin_drain()` — no ownership verb.
`RelayPeerTransport` (`lib.rs:158-175`) is explicitly documented as *not* performing
generation fencing (`lib.rs:161-163`). The consumer even keeps a compile-time
reminder: `fn _peer_info_is_not_an_owner_signal(_peer: PeerInfo)`
(`tunnel/reliable.rs:949`).

**BR-MESH-03 — Fencing is checked at every hop, by every consumer, not by the pipe.**
`wire.rs:21-24`, `lib.rs:161-163`. The crate provides the header slot; each
consumer validates. In practice the validators are
`SessionDirectory::validate_fenced_header` (`tunnel/directory.rs`) and the huddle
`GenerationFloor` (`audio/mesh.rs`).

---

#### B. Membership join

**BR-MESH-04 — The Redis ready registry is the only way *into* the mesh.**
`registry.rs:3-6`. A fresh runtime learns dialable peers exclusively from
`ReadyRegistry::scan_ready` (`registry.rs:211-257`), invoked at
`runtime.rs:311` (reconcile) and `runtime.rs:318` (inbound-admission rescan).
Gossip then propagates records peer-to-peer (BR-MESH-19).

**BR-MESH-05 — Ready records are only published while the relay is ready.**
`ReadyRegistry::publish_ready` deliberately has no internal readiness probe
(`registry.rs:177-181`); the caller owns the predicate. The relay's predicate is
`!shutting_down` (`mesh_boot.rs:469`), evaluated every `refresh_interval`
(`runtime.rs:602`).

**BR-MESH-06 — The first ready-registry publish is part of boot; failure is fatal.**
`mesh_boot.rs:456-463` — `.map_err(|e| anyhow!("mesh ready-registry publish failed: {e}"))?`
propagated through `main.rs:442`'s `?`. Rationale documented at
`mesh_boot.rs:456-457` and `mesh_boot.rs:406-411`: "an operator who sets
`BUZZ_MESH=on` wants the mesh or wants to know why not."

**BR-MESH-07 — A record cannot be published unless its own attestation verifies.**
`publish_ready` calls `record.verify_attestation()?` before touching Redis
(`registry.rs:183`). Self-check, so it only catches local construction bugs.

**BR-MESH-08 — Ready-record TTL is exactly 3× the refresh interval, floored at 1 s.**
`REGISTRY_EXPIRY_MULTIPLIER = 3` (`registry.rs:21`), `expiry_for`
(`registry.rs:154-156`), `ttl_secs = self.expiry().as_secs().max(1)`
(`registry.rs:187`). With the relay's hardcoded 15 s refresh (`config.rs:511`) the
TTL is 45 s. A crashed pod therefore disappears from the registry within 45 s
(`registry.rs:198` — "A crash is handled by TTL expiry"). Test:
`expiry_is_three_refreshes` (`registry.rs:322-325`).

**BR-MESH-09 — A scanned record whose Redis key does not match its own
`runtime_id` is discarded.** `scan_ready` requires `record.key() == key`
(`registry.rs:233`); mismatch logs and skips (`registry.rs:240-244`). Blocks
key-substitution.

**BR-MESH-10 — One malformed registry entry must not block bootstrap from healthy
peers.** `scan_ready` warns-and-skips on decode failure, key mismatch, and
attestation failure (`registry.rs:233-247`), documented at `registry.rs:206-208`.
Only Redis transport errors abort the scan (`registry.rs:225`, `?`).

---

#### C. Admission and identity

**BR-MESH-11 — Ready records must be attested by *this deployment's* relay key,
checked before signature verification.** `apply_ready_records`
(`membership.rs:85-118`): first `ready.relay_pubkey == expected` (`:90-92`), then
`ready.verify_attestation()` (`:103`). Ordering is deliberate and documented at
`membership.rs:80-84`: "a record signed by a key we don't recognize is foreign no
matter how valid its signature is." Rejections bump `foreign_relay_rejections`
(`:93-94`) and log with `anchored` (`:99`).
Tests: `ready_records_from_foreign_relay_identity_are_rejected` (`:451-462`),
`ready_records_must_have_valid_attestation` (`:437-448`).

**BR-MESH-12 — An unanchored membership table admits nothing (fail-closed).**
`expected_relay_pubkey: None` falls into the reject arm (`membership.rs:93`, the
`anchor =>` catch-all). Documented `membership.rs:36-40`; test
`unanchored_membership_rejects_all_ready_records` (`:465-471`). The relay always
anchors (`mesh_boot.rs:444-445`).

**BR-MESH-13 — Attestation binds the boot-unique endpoint pubkey to the deployment
identity, over a versioned textual preimage.** Preimage
`buzz-relay-mesh-ready-v1\nruntime_pubkey={hex}\nrelay_pubkey={hex}`
(`registry.rs:85-91`), SHA-256'd (`registry.rs:93-96`), schnorr-signed
(`registry.rs:41`) / verified (`registry.rs:78-80`). Textual-and-versioned so
"integration can reproduce it exactly without depending on JSON key order"
(`registry.rs:82-84`).

**BR-MESH-14 — `runtime_pubkey` must equal `runtime_id.to_hex()`.**
`verify_attestation` rejects the mismatch before signature checking
(`registry.rs:140-145`). Tests
`ready_record_attestation_verifies_and_binds_runtime_pubkey` (`:341-349`) and
`attestation_rejects_signature_for_other_runtime` (`:352-357`).

**BR-MESH-15 — The mesh identity must NOT be the deployment Nostr key.**
`wire.rs:52-60`: all pods share one `BUZZ_RELAY_PRIVATE_KEY` Secret, so using it
would give every pod the same runtime id and "collapse the ownership plane."
Enforced by construction — the only RuntimeId source is
`SecretKey::generate()`/`public()` (`endpoint.rs:20`, `:38`).

**BR-MESH-16 — RuntimeId is boot-unique.** Fresh keypair every
`MeshEndpoint::bind` (`endpoint.rs:19-21`), documented `endpoint.rs:8-9`,
`wire.rs:47-50`. `bind_with_secret_key` exists only for stable test identities
(`endpoint.rs:23-25`).

**BR-MESH-17 — Inbound connections are admitted only from runtime ids present in the
membership table; unknown ids get exactly one registry rescan first.**
`accept_loop` → `is_known_peer` (`runtime.rs:262-270`, `:305-323`): `has_peer` →
if false and a registry exists, one `scan_ready` + `apply_ready_records`, then
re-check. Rejection logs `"rejected inbound connection from unattested runtime id"`
(`runtime.rs:265-269`) and `continue`s. Documented as gating *dialability*, never
ownership (`runtime.rs:11-14`, `membership.rs:184-186`).

**BR-MESH-18 — ALPN mismatch is a hard reject at connection construction.**
`MeshPeer::from_connection` returns `MeshError::Transport("unexpected mesh ALPN …")`
when `conn.alpn() != ALPN` (`peer.rs:50-55`). ALPN is `buzz/mesh/1`
(`wire.rs:37`) and is documented to change on every wire-version bump so rolling
deploys never half-speak (`wire.rs:34-36`).

---

#### D. Gossip, anti-entropy, record versioning

**BR-MESH-19 — Scuttlebutt push-pull: a received `Digest` is answered with a
`Delta`; a received `Delta` is applied.** `control_stream_exchange`
(`runtime.rs:531-551`): `Digest{entries}` → `membership.delta_for(&entries)` →
send back as `MeshStreamFrame::Gossip`; `Delta{records}` → `apply_gossip_record`
per record.

**BR-MESH-20 — Only the owning runtime may increment its own record version.**
`gossip.rs:23-24`. Enforced locally: `update_local` is the only version bumper
(`membership.rs:166-178`, `gossip.rs:100-109`) and it only ever touches
`self.local_record`. Enforced remotely by BR-MESH-22 (self-records rejected).
**Not enforced cryptographically** — see `-security.md` S-01.

**BR-MESH-21 — Conflict resolution is strict last-version-wins; equal versions
lose.** `membership.rs:128` (`record.version <= peer.record.version => false`) and
`gossip.rs:154-157` (`record.version > existing.version`). No vector clocks, no
timestamp tie-break. Test `stale_gossip_record_is_ignored` (`membership.rs:474-479`)
and `apply_delta_ignores_stale_versions` (`gossip.rs:253-266`).

**BR-MESH-22 — A runtime never accepts a gossiped record about itself.**
`membership.rs:121-123` (returns `false`), `membership.rs:87-89` (`continue`).
Prevents a peer from rewriting our own advertised addrs/draining flag.
Test `ready_records_seed_peers_but_skip_self` (`membership.rs:425-434`).

**BR-MESH-23 — A version bump always refreshes `heartbeat_millis`.**
`membership.rs:174-176`, `gossip.rs:105-107`. Version is `saturating_add(1)`, so it
never wraps (it saturates at `u64::MAX`).

**BR-MESH-24 — Ready-registry seeds enter as version-1 hints; existing newer gossip
records win.** `apply_ready_records` builds a fresh `GossipRecord::new(...)`
(version 1, `gossip.rs:38`) and routes it through `apply_gossip_record`
(`membership.rs:110-116`), which rejects it if the table already holds version ≥ 1.
Documented `membership.rs:76-78`. Consequence: after the first gossip tick a peer's
registry-published `endpoint_addrs` can never again correct a stale gossiped
address, because the seed is permanently version 1.

**BR-MESH-25 — Digest and Delta contents are deterministically ordered by
`runtime_id.to_hex()`.** `membership.rs:216`, `:241`; `gossip.rs:118`, `:144`.

**BR-MESH-26 — A digest/delta covers *all* records the runtime knows, local
included.** `records()` appends the local record (`membership.rs:204`), and both
`digest()` (`:210`) and `delta_for()` (`:232`) build from it. This is how a new pod's
own record reaches peers that have not yet rescanned Redis.

**BR-MESH-27 — Gossip payload version must equal `GOSSIP_PAYLOAD_VERSION`.**
`decode_message` rejects any other value with
`MeshError::Transport("unknown gossip payload version {v}")` (`gossip.rs:68-76`).
Note this is a `Transport` string, not a typed variant — unlike the outer frame's
`UnknownWireVersion` (`wire.rs:185`).

**BR-MESH-28 — Gossip frames carry no fence and no attestation.**
`MeshStreamFrame::Gossip { payload }` (`wire.rs:144-146`) has no `FencedHeader`;
`apply_gossip_record` (`membership.rs:120-153`) performs **no** signature or
relay-anchor check — unlike `apply_ready_records`. This is the single largest
asymmetry in the trust model; see `-security.md` S-01.

**BR-MESH-29 — Gossip is periodic and idempotent, so dropping a frame under
backpressure is preferred over blocking.** `send_control_frame` uses `try_send` and
discards the error (`runtime.rs:553-558`, comment at `:554-555`). Queue depth 64
(`runtime.rs:46`).

**BR-MESH-30 — Every gossip tick bumps the local record even when nothing changed.**
`gossip_tick_loop` calls `update_local(|_| {})` (`runtime.rs:566`) purely so peers'
phi accrual sees life, then broadcasts a digest to every peer with a live control
queue (`runtime.rs:567-585`). Interval `DEFAULT_GOSSIP_INTERVAL = 2s`
(`runtime.rs:44`).

---

#### E. Failure detection

**BR-MESH-31 — Heartbeat cadence is 2 s (gossip tick); registry refresh is 15 s.**
`runtime.rs:44`; `config.rs:511` / `registry.rs:20`. The gossip tick sleeps *before*
its first bump (`runtime.rs:564`), so the first heartbeat lands one interval after
`MeshRuntime::start`.

**BR-MESH-32 — Phi is computed only from observed heartbeat *intervals*, and only
advances on a strictly newer record.** `PhiAccrual::observe` is called only from
`apply_gossip_record` (`membership.rs:132`, `:139`) — i.e. inside the branches that
accepted a newer version. A peer whose gossip stalls but whose QUIC connection is
alive will still accrue suspicion.

**BR-MESH-33 — Phi formula: `(elapsed / mean_interval) / ln(10)`.**
`gossip.rs:203-215`, comment `:213`. Returns `None` when no heartbeat seen, when
fewer than 2 heartbeats have been observed (`samples.is_empty()`, `:207`), or when
mean ≈ 0 (`:211`). **Sample variance is unused** (`mean_secs`, `:217-220`, is the
only statistic) — this is an exponential approximation, not the
distribution-based phi-accrual detector the type name implies.

**BR-MESH-34 — Suspicion threshold is 8.0, hardcoded in production.**
`DEFAULT_PHI_SUSPECT_THRESHOLD = 8.0` (`membership.rs:17`). `with_phi_suspect_threshold`
(`membership.rs:66-69`) exists but has **zero callers** — the relay never overrides
it (`mesh_boot.rs:444-445` only anchors the relay pubkey). With a ~2 s mean, phi ≥ 8
requires `8 × ln(10) × 2s ≈ 36.8 s` of gossip silence.

**BR-MESH-35 — A suspect peer is filtered out of `peers()` entirely and reported as
`connection_state: "suspect"` in `/_mesh`.** `RelayMeshMembership::peers`
(`membership.rs:359-379`) returns `None` for `phi >= threshold` (`:366-369`);
`peer_statuses` overrides the stored state to `Suspect` (`membership.rs:340-344`).
Suspicion is recomputed per call — never persisted.

**BR-MESH-36 — Suspicion does not evict, disconnect, or stop dialing.** Verified:
`membership.rs` has no `remove`/`retain`/`clear`; `reconcile_once`
(`runtime.rs:296-326`) filters only on `runtime_id != local && !record.draining`,
so a suspect peer is still dialed every 5 s and still passes `has_peer` admission
(BR-MESH-17) forever. Suspicion is purely advisory — and since `peers()` has zero
production readers (see `-api-surface.md` §6), it currently has **no effect at all**.

**BR-MESH-37 — A peer whose recv loop errors is removed from the *connection* table
but not the membership table.** `datagram_recv_loop` / `stream_accept_loop` call
`remove_peer` on any error (`runtime.rs:359-363`, `:379-383`); `remove_peer`
(`runtime.rs:267-281`) aborts the entry's tasks and sets
`ConnectionState::Disconnected`. The `MeshMembership.peers` entry survives.

---

#### F. Peer selection, dialing, reconnect

**BR-MESH-38 — The mesh is a warm full mesh: every known non-draining peer is dialed
proactively.** `reconcile_once` (`runtime.rs:296-326`) dials all candidates not
already connected; rationale at `runtime.rs:15-19` — "failover is 'next frame goes
elsewhere,' not 'wait for a handshake.'" There is **no peer-selection or fan-out
sampling** — the "gossip fan-out" is 1:N to every connected peer
(`runtime.rs:575-585`), not the log-N sampling classical scuttlebutt uses.

**BR-MESH-39 — Draining peers are never dialed.** `reconcile_once` filter
`!record.draining` (`runtime.rs:302`).

**BR-MESH-40 — Reconcile cadence 5 s, preceded by one immediate pass at boot.**
`DEFAULT_RECONCILE_INTERVAL` (`runtime.rs:42`); `reconcile_loop` runs
`reconcile_once` *then* sleeps (`runtime.rs:288-293`); the relay additionally forces
one pass before returning the handle (`mesh_boot.rs:478`, rationale
`runtime.rs:148-149`).

**BR-MESH-41 — Dial tries a peer's advertised addresses in order and stops at the
first success.** `dial_peer` (`runtime.rs:328-355`): unparseable addr → warn +
next; bad runtime id → warn + `return` (abandons the whole peer); connect success →
`install_peer` + `return`; connect failure → warn + next addr. All addrs exhausted →
mark `Disconnected` (`runtime.rs:351-354`).

**BR-MESH-42 — There is no reconnect backoff.** Verified: no jitter, no exponential
delay, no failure counter anywhere in `runtime.rs`. A permanently unreachable peer
record is re-dialed on every 5 s tick, and each dial `await`s inside the sequential
loop (`runtime.rs:326`), so N dead peers serialize their connect timeouts into one
reconcile pass. Combined with BR-MESH-36 (no eviction) this is unbounded churn.

**BR-MESH-43 — Simultaneous-dial tie-break: the connection dialed by the numerically
smaller runtime id wins.** `new_connection_wins(local, remote, new_dialed_by_us)`
(`runtime.rs:204-213`): if `local.0 < remote.0` the winner is our outbound
(`new_dialed_by_us`), else the peer's inbound (`!new_dialed_by_us`). Byte-wise
lexicographic comparison of the 32-byte arrays. Deterministic and symmetric on both
ends (`runtime.rs:20-22`); tests `tie_break_is_symmetric` (`runtime.rs:846-856`) and
`simultaneous_dial_converges_to_one_connection` (`runtime.rs:858-...`).

**BR-MESH-44 — The losing connection's tasks are aborted and its entry replaced.**
`install_peer` (`runtime.rs:216-260`): if an entry exists and the new connection
loses, log + `return false` (keep existing, `:224-227`); if it wins,
`existing.abort()` (`:228`) then insert. `PeerEntry::abort` aborts every
`JoinHandle` (`runtime.rs:56-62`).

**BR-MESH-45 — Exactly one control stream per connection, opened by the dialer.**
`install_peer` spawns `open_control_stream` only when `dialed_by_us`
(`runtime.rs:236-247`); the acceptor learns its writer queue when
`Hello{Control}` arrives (`runtime.rs:441-448`). Documented `wire.rs:149-151`,
`runtime.rs:20-21`.

**BR-MESH-46 — Only peers with a live control queue receive gossip digests.**
`gossip_tick_loop` filters `control_tx.is_some()` (`runtime.rs:576`). An acceptor
that never receives a `Hello{Control}` is gossip-silent but still connected.

---

#### G. Wire discipline / framing rules

**BR-MESH-47 — Every encoded frame is prefixed with a one-byte wire version;
unknown versions are rejected loudly, never guessed.** `encode`
(`wire.rs:176-179`), `decode` (`wire.rs:182-188`) → `UnknownWireVersion` /
`EmptyFrame`. Documented `wire.rs:38-41`. Test `unknown_version_rejected`
(`wire.rs:246-257`).

**BR-MESH-48 — The first frame on any stream, in both directions, MUST be `Hello`.**
`wire.rs:29-31`, `wire.rs:128`. Enforced on the accept side: a non-`Hello` first
frame logs `"stream without Hello — dropped"` and `continue`s
(`runtime.rs:400-411`). **Delta vs doc:** `wire.rs:31` says the stream "is reset";
the implementation drops the `MeshStream` value and moves on — no explicit QUIC
`reset()` call exists in the crate. Also, the contract says "in both directions,"
but nothing validates the *peer's* Hello on a stream we opened
(`runtime.rs:190-191` sends ours and returns the stream unread).

**BR-MESH-49 — Stream frames are u32-LE length-delimited, capped at 16 MiB.**
Send: `write_all(len.to_le_bytes())` then body (`peer.rs:148-158`), with
`FrameTooLarge` pre-check (`peer.rs:142-147`). Recv: `read_exact(4)` → length →
`FrameTooLarge` if over cap → `read_exact(len)` (`peer.rs:169-191`).
`FinishedEarly(0)` maps to a clean `Ok(None)` end-of-stream (`peer.rs:172`).

**BR-MESH-50 — Datagram boundary is the frame boundary; senders must check size and
fail loud, never truncate.** `wire.rs:19-22`; `encode_datagram_checked`
(`lib.rs:213-226`) and `MeshPeer::send_datagram` (`peer.rs:105-116`) →
`DatagramTooLarge`. A peer with no datagram support yields
`Transport("peer does not support QUIC datagrams")` (`peer.rs:109`).
Test `oversized_datagram_is_rejected_before_send` (`endpoint.rs:239-254`).

**BR-MESH-51 — Datagram receivers tolerate gaps and reordering and never wait.**
`wire.rs:114-116`; `seq` is observability-only. `datagram_recv_loop`
(`runtime.rs:358-372`) never buffers or reorders.

**BR-MESH-52 — `RealtimeMedia` is datagram-only; `HuddleControl` is stream-only.**
`wire.rs:99-108` (HuddleControl "rides a reliable stream … never datagrams,
[because] a dropped roster delta is an unrecoverable peer-index desync"). Enforced
in the consumer's dispatcher: a stream claiming `RealtimeMedia` is rejected
(`mesh_boot.rs:118-126`), test `dispatcher_routes_session_streams_by_profile`
(`mesh_boot.rs:601-635`).

**BR-MESH-53 — Non-gossip frames on the control stream are logged and ignored, not
fatal.** `runtime.rs:545-547`.

**BR-MESH-54 — Control streams never reach the inbound handler; session streams
without a registered handler are dropped with a warning.** `runtime.rs:437-459`
(Control handled in-runtime); `runtime.rs:465-470` (no handler → warn + drop).
Consumer-side equivalent at `mesh_boot.rs:99-104`, `:128-134`. Documented as a
bounded boot-window race made safe by the peer's fenced retry
(`mesh_boot.rs:53-55`).

**BR-MESH-55 — Changes to `wire.rs` require a post in the mesh thread before the
edit.** `wire.rs:10-13`. Process rule, unenforced by tooling — no CODEOWNERS entry
or CI check references `wire.rs` (verified: `.github/CODEOWNERS` has no mesh entry).

---

#### H. Drain / lifecycle

**BR-MESH-56 — Drain sets a local flag and gossips `draining=true` (one version
bump).** `begin_drain` (`membership.rs:385-388`): `draining.store(true)` +
`update_local(|r| r.draining = true)`. Test `begin_drain_updates_local_record`
asserts version goes 1→2 (`membership.rs:482-489`).

**BR-MESH-57 — Drain is triggered by the relay's `shutting_down` flag, polled every
500 ms, one-shot.** `mesh_boot.rs:481-497` — the watcher `return`s after firing, so
drain cannot be un-set.

**BR-MESH-58 — `GoodbyeReason::Draining` tells the peer to re-establish elsewhere;
`SessionEnded` is normal; `StaleGeneration` is the sender fencing itself out.**
`wire.rs:166-174`. All three are produced by `buzz-relay` consumers, not by this
crate.

**BR-MESH-59 — A clean `Goodbye` is semantically distinct from a QUIC reset;
receivers treat reset as abnormal.** `wire.rs:135-137`.

**BR-MESH-60 — The heartbeat clears the registry record on the ready→not-ready
edge.** `ReadyHeartbeat::tick` (`registry.rs:295-304`): publish while ready; `DEL`
once on the falling edge (guarded by `published`), then stop.
`ReadyHeartbeat::shutdown()` (`:306-312`) does the same explicitly but has **zero
callers**. Clean-shutdown removal therefore depends entirely on the heartbeat task
getting at least one post-SIGTERM tick within its 15 s interval before the process
exits — the relay's graceful window is 30 s (`main.rs:1108`), so this usually holds
but is not guaranteed.

**BR-MESH-61 — `MeshRuntime` loops outlive all handle clones; only `shutdown()`
stops them.** `runtime.rs:75-76`, `shutdown` (`runtime.rs:155-164`) aborts the 3
loops and every peer entry. **Zero production callers** — the relay never calls it.
`spawn_registry_heartbeat`'s task (`runtime.rs:598-607`) has no shutdown path at all.

---

#### I. Version negotiation (exhaustive)

There are **four independent version channels**, and only two of them actually
negotiate anything:

| # | Channel | Value | Advertised | Checked | Effect of mismatch |
|---|---|---|---|---|---|
| V1 | **ALPN** `buzz/mesh/1` | `wire.rs:37` | offered at bind (`endpoint.rs:34`) and dial (`endpoint.rs:88`) | by QUIC/TLS, plus a belt-and-braces check `conn.alpn() != ALPN` (`peer.rs:50-55`) | connection never establishes; no half-speaking (`wire.rs:34-36`) |
| V2 | **frame version byte** `WIRE_VERSION = 1` | `wire.rs:42` | first byte of every datagram and stream frame (`wire.rs:177`) | `decode` (`wire.rs:183-186`) | `MeshError::UnknownWireVersion(v)`; frame dropped, loop continues |
| V3 | **gossip payload version** `GOSSIP_PAYLOAD_VERSION = 1` | `gossip.rs:13` | in-struct field on both `Digest` and `Delta` (`gossip.rs:52-59`) | `decode_message` (`gossip.rs:68-76`) | `MeshError::Transport("unknown gossip payload version …")`; frame logged + dropped (`runtime.rs:548-550`) |
| V4 | **`proto_version: u16`** | `WIRE_VERSION as u16` (`mesh_boot.rs:367`) | in `ReadyRecord.proto_version` (`registry.rs:110`) and `GossipRecord.proto_version` (`gossip.rs:19`) | **never** | none — see below |

**BR-MESH-62 — ALPN is the real negotiation boundary; V1 is redundant with it but
harmless.** Because ALPN embeds the version, two pods with different wire versions
cannot connect at all, which makes V2's `UnknownWireVersion` unreachable across a
rolling deploy and reachable only from a same-ALPN peer sending a corrupt byte.

**BR-MESH-63 — `proto_version` is advertised and never compared.** Verified: grep
for `proto_version` in `crates/**` shows only assignment sites
(`gossip.rs:19,33,34`, `registry.rs:110,126`, `membership.rs:112`, `status.rs:21`,
`membership.rs:337`, `mesh_boot.rs:367,439,451`) and the `/_mesh` echo — no
comparison, no gate, no rejection. The field is pure observability.

**BR-MESH-64 — `capabilities` is advertised and never consulted.**
`mesh_boot.rs:371-377` ships a static `["reliable-stream", "realtime-media",
"huddle-control"]`; `apply_ready_records` copies it onto the gossip record
(`membership.rs:113`); nothing reads it. There is **no capability negotiation** —
`open_session_stream` will happily open a `HuddleControl` stream to a peer that
never advertised it, and the only backstop is the receiver's dispatcher
(`mesh_boot.rs:112-134`).

**BR-MESH-65 — Wire enums are exhaustive and postcard-strict, so forward
compatibility within one ALPN is zero.** No `#[non_exhaustive]` anywhere
(verified). A future 5th `MeshStreamFrame` variant decodes to `MeshError::Decode`
on an older peer at the same ALPN — the design intent (bump ALPN with the wire
version) is the only safe path, and it is unenforced.

---

#### J. Ownership / lease rules (what this crate does *not* do)

**BR-MESH-66 — This crate grants no ownership and holds no lease.** `lib.rs:33-35`,
`registry.rs:3-6`, `gossip.rs:3-5`, `membership.rs:3-6`. Verified by absence: no
Redis `SET NX`, no `INCR`, no `EVAL`/Lua, no CAS anywhere in the crate. The only
Redis commands are `SET … EX`, `DEL`, `SCAN`, `GET` (`registry.rs:188-228`), all
against `mesh:ready:*`.

**BR-MESH-67 — `owner_runtime_id` in the fenced header is advisory; the generation
is what fences.** `wire.rs:90-92`. Consistent with the consumer, where
`OwnerMismatch` is raised by the Redis-backed directory
(`tunnel/directory.rs:430`), not by the transport.

**BR-MESH-68 — Generation monotonicity across owner death is guaranteed elsewhere.**
Stated by the consumer at `audio/mesh.rs:44-46` ("the directory's companion INCR
counter … this module trusts that"). Nothing in `buzz-relay-mesh` maintains or
validates that counter.

---

#### K. Rules documented but not implemented (delta list)

| Claim | Where claimed | Reality |
|---|---|---|
| Counter `mesh_fence_rejections_total{reason=…}` | `lib.rs:102-109` | does not exist — grep finds no such metric in the repo |
| `MeshCounters.stale_generation_rejections` reflects fence rejects | `status.rs:43` | always 0; the only writer `record_stale_generation_rejection` (`membership.rs:285`) has no caller but the test at `:486` |
| `BUZZ_MESH` is "`on` (default when replicas can exist)" | `lib.rs:55-56` | default is **off** (`config.rs:498-500`); tested `mesh_boot.rs:546-555` |
| `PeerInfo.load` is "gossiped by the peer (0.0..)" | `lib.rs:137-138` | structurally always `0.0` — nothing ever writes a non-zero `load` |
| Non-Hello first frame ⇒ "the stream is reset" | `wire.rs:31` | dropped, not reset (`runtime.rs:404-406`) |
| `MeshStream` halves are "placeholder … pre-transport" | `lib.rs:186-188` | they are the real iroh halves (`peer.rs:132-192`) |
| Relay consumes the crate "exclusively through two seams" | `lib.rs:11-19` | also uses `InboundHandler`, `MeshStream`, both half traits, `MeshEndpoint`, `MeshPeer`, `GossipRecord`, `ReadyRecord`/`ReadyRegistry`, `MeshRuntime`, `spawn_registry_heartbeat` |


## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: Business Rules

The crate's rules are the accept/reject conditions of `check_trace`
(`src/checker.rs:74`) and `check_step` (`src/transitions.rs:138`). Every rule below is a
predicate that either returns `Ok(())` or one of the four `TransitionError` variants
(`src/transitions.rs:61-102`).

**Naming correction up front.** The invariant names actually referenced by this crate are
`Inv_NonInterference` and `Inv_ReadConfinement` (`src/transitions.rs:53-54`, `:296-297`,
`src/lib.rs:238`), plus `Inv_SanitizedErrors` (`src/lib.rs:125`) and `Inv_AdmissionFence`
(`src/transitions.rs:222-224`). `Inv_RowZero`, `Inv_NoFork`, `Inv_Closed`,
`Inv_RefEffectApplied`, and `Inv_RefDerivedFromParent` belong to **other** specs and modules:
`Inv_RowZero` to the relay's host-binding seam
(`crates/buzz-relay/src/tenant.rs:76`, `:291`, `:312`), and the four `Inv_Ref*`/`Inv_NoFork`/
`Inv_Closed` names to `docs/spec/GitOnObjectStore.tla:211`, `:221`, `:230`, `:253` as consumed
by `crates/buzz-relay/src/api/git/cas_publish.rs`. Neither set has any relationship to
`buzz-conformance`.

---

### BR-CONF-01 — Non-empty trace (fail-closed on silence)

- **Asserts:** a critical seam that was reached must have emitted at least one action.
- **Predicate:** `if scenario.trace.is_empty()` → `CoverageBreach`
  (`src/checker.rs:75-82`). Detail string: "trace is empty — seam reached without emitting
  any action" (`:78-80`).
- **Preconditions:** none. This is checked before bootstrap, so it is the first rule.
- **Spec grounding:** no TLA+ counterpart. This is a harness-level rule the crate doc calls
  "load-bearing … without it, trace conformance is decorative logging" (`src/lib.rs:35-36`).
- **Tests:** `checker::tests::empty_trace_is_coverage_breach` (`src/checker.rs:165-170`),
  `replay_fixtures::empty_trace_is_coverage_breach` (`tests/replay_fixtures.rs:295-304`).
- **Not producible by the relay:** the relay never calls `check_trace`, so this rule fires only
  against test-constructed scenarios. The equivalent production signal is `EmitGuard`'s
  `ImplBug` (BR-CONF-09).

---

### BR-CONF-02 — Schema-version equality

- **Asserts:** every step's `schema_version` equals `SCHEMA_VERSION` (= 1, `src/lib.rs:86`).
- **Predicate:** checked twice — once on `trace[0]` before bootstrap (`src/checker.rs:84-93`)
  and again for every step inside the loop (`:97-107`). Both return
  `IllegalTransition { step_index, detail: "trace schema_version=… but checker
  schema_version=…" }`.
- **Classification note:** a version skew is mapped to `IllegalTransition`, not to a distinct
  variant; the rationale is at `src/checker.rs:69-71` ("no transition rule applies").
- **Reachability:** unreachable through the typed API — `TraceStep::new` (`src/lib.rs:303-309`)
  is the only constructor and always stamps `SCHEMA_VERSION`. Only hand-edited JSONL can
  trigger it. No test covers it (verified: no test constructs a mismatched version).

---

### BR-CONF-03 — Model bootstrap from the trace itself

- **Asserts:** the model state for the whole trace is whatever `trace[0].state_after` says.
- **Predicate:** `ModelState::bootstrap(&first.state_after)` (`src/checker.rs:94`, impl at
  `src/transitions.rs:123-129`) copies all three fields verbatim.
- **Consequence (self-consistency, not correctness):** the checker never independently derives
  the tenant. A relay that resolved the *wrong* community and then reported that wrong
  community consistently across every step passes all state rules. The crate is explicit that
  independence means "no shared code" (`Cargo.toml:8-24`), not "independent derivation";
  `src/transitions.rs:110-114` states the checker "does NOT know `HostCommunity[_]` at large".
- **Immutability:** `check_step` takes `&ModelState` and updates nothing
  (`src/transitions.rs:131-132`), so no rule can express state evolution.

---

### BR-CONF-04 — Mid-request state stability (three fields)

- **Asserts:** `resolved_community`, `bound_host`, and `actor` are identical on every step of
  a trace.
- **Predicates** (checked in order, before any action-specific logic):
  | Field | Guard | Line | Error |
  |---|---|---|---|
  | `resolved_community` | `!=` model | `src/transitions.rs:143-151` | `StateMismatch` |
  | `bound_host` | `!=` model | `:152-160` | `StateMismatch` |
  | `actor` | `!=` model | `:161-169` | `StateMismatch` |
- **What it detects:** "the relay either reassigned the tenant context mid-request, or emitted
  a step from a context other than `TenantContext`" (`src/transitions.rs:71-73`).
- **Spec grounding:** implicit. TLA+ models `observations` as an unordered set with a per-
  observation `community` field (`docs/spec/MultiTenantRelay.tla:801-803`); the "one request =
  one tenant" collapse is a modeling reduction the crate documents at
  `src/transitions.rs:24-30`.
- **Tests:** `checker::tests::state_after_changing_mid_request_is_state_mismatch`
  (`src/checker.rs:262-287`); property P5 `state_flip_bites_state_mismatch`
  (`tests/proptest_checker.rs:354-399`) flips exactly one of the three fields per case
  (`:373-386`).

---

### BR-CONF-05 — Row-label confinement (`Inv_NonInterference` / `Inv_ReadConfinement`)

The crate's central rule and the only one that translates a TLA+ invariant directly.

- **TLA+ source:**
  `Inv_NonInterference == \A o \in observations : o.labels \subseteq {o.community}`
  (`docs/spec/MultiTenantRelay.tla:985-986`), and
  `Inv_ReadConfinement == \A o : o.kind = "ResultRows" => \A r \in o.rows : r.community =
  o.community` (`:999-1001`).
- **Rust predicate:** `check_row_labels` (`src/transitions.rs:294-312`) —
  `row_communities.iter().find(|c| **c != model.resolved_community)`; a hit returns
  `NonInterference { step_index, detail }` (`:303-310`).
- **Applied to:** all three read variants, which share one match arm —
  `ReadMessageRows | ReadByIdRows` (`src/transitions.rs:258-268`) and `ReadHostFeedRows`
  (`:269-271`).
- **Vec-not-Set is a rule, not an accident:** documented at `src/lib.rs:236-239` and
  `src/transitions.rs:281-292`. Presence of a foreign label is the entire bar; count and
  duplication are irrelevant, and a de-duping emitter still bites.
- **Not enforced on writes:** `WriteInsert`, `WriteInsertGlobal`, and `WriteDuplicate` carry no
  row labels at all, so this rule cannot apply to the write seam.
- **Tests:** `checker::tests::cross_community_row_bites_non_interference`
  (`src/checker.rs:210-225`); property P1 `foreign_row_label_is_rejected`
  (`tests/proptest_checker.rs:213-267`) which rotates across all three read surfaces
  (`:246-258`); fixture `bad_foreign_row_leak.jsonl` via
  `foreign_row_leak_is_non_interference` (`tests/replay_fixtures.rs:281-292`, builder
  `:154-168`).
- **Uncheckable companion:** `Inv_NoTenantContextFailsClosed`
  (`docs/spec/MultiTenantRelay.tla:1116-1118`) says `labels = {} => rows = {}`. The Rust rule
  cannot express it: an empty `row_communities` vec passes `check_row_labels` vacuously
  (`src/transitions.rs:299-311`, the `find` returns `None`), and the schema has no separate
  row count.

---

### BR-CONF-06 — `AuthCheck` Allow requires claim agreement (M2/M8 bite)

- **TLA+ source:** `AuthCheck(w)` (`docs/spec/MultiTenantRelay.tla:794-809`). The spec's
  authorization is `hostAgrees == real \in Communities /\ HostCommunity[host] = real` (`:798`)
  and `allowed == hostAgrees /\ ch \in ScopedAccessible(real, a)` (`:799`), with
  `real == ChannelCommunity(ch)` (`:797`). The observation carries
  `labels |-> {real}` and `community |-> real` (`:801-803`).
- **Rust predicate:** `src/transitions.rs:228-250`:
  ```
  (Verdict::Allow, Some(c)) if c != &model.resolved_community => IllegalTransition
  _ => Ok(())
  ```
  So the bite fires **only** when the verdict is `Allow` **and** a claim is present **and** it
  differs from the resolved community.
- **What is deliberately NOT checked:** `Deny` with any claim is in-spec
  (`src/transitions.rs:224-226`, `:243-248`); `Allow` with `claimed_community: None` passes
  unconditionally; and `ScopedAccessible` cannot be recomputed — stated at
  `src/transitions.rs:216-218` ("that's production state").
- **Coverage hole this creates:** the REQ-path emitter hard-wires `claimed_community: None`
  (`crates/buzz-relay/src/conformance/mod.rs:152`, rationale `:135-145`), so on the entire read
  path this rule is structurally inert — the guard's `Some(c)` pattern can never match.
- **Inverse hazard on the write path:** the ingest emitter populates the claim from the first
  `h` tag (`crates/buzz-relay/src/conformance/mod.rs:101-119`, call site
  `crates/buzz-relay/src/handlers/ingest.rs:1788`), and in Buzz an `h` tag carries a **channel**
  UUID, not a community UUID (the emitter's own comment concedes the ambiguity at
  `mod.rs:103-105`). `channel_uuid != resolved_community` for essentially every real event, so
  with a recording tracer this rule would fire `IllegalTransition` on nearly every authorized
  channel write. See the data-model doc's D-CONF-02.
- **Tests:** `checker::tests::auth_allow_with_foreign_claim_bites_m2` (`src/checker.rs:228-244`)
  and `auth_deny_with_foreign_claim_is_fine` (`:247-259`); properties P3a
  `auth_allow_foreign_claim_bites` (`tests/proptest_checker.rs:272-297`) and P3b
  `auth_deny_any_claim_is_ok` (`:304-323`); fixture `bad_host_channel_mismatch.jsonl`
  (`tests/replay_fixtures.rs:253-264`, builder `:107-129`).

---

### BR-CONF-07 — Write actions carry no obligation beyond BR-CONF-04

- **Predicates:** all three write variants return `Ok(())` unconditionally —
  `WriteInsert` (`src/transitions.rs:187`), `WriteInsertGlobal` (`:191`),
  `WriteDuplicate` (`:198`).
- **Stated rationale:** "The spec ignores `claimed_community` ('host wins'), so a mismatch is
  allowed at this exact action — the gate that bites it is the next read's row labels"
  (`src/transitions.rs:183-186`, repeated `:47-51`).
- **TLA+ obligations left unchecked:** `Inv_ResolutionFence`
  (`docs/spec/MultiTenantRelay.tla:1011-1035`) requires
  `w.community = ChannelCommunity(w.channel)` for channel-bearing writes, and
  `Inv_HostBindingFence` (`:1038-1055`) requires `w.community = HostCommunity[w.host]`. Neither
  is computable from the trace: `AbstractState` carries no channel→community map and no
  host→community map (`src/lib.rs:150-161`). `src/transitions.rs:110-114` acknowledges the
  latter explicitly.
- **Net effect:** the write seam contributes only the three state-stability checks and, for
  channel-bearing writes, the `AuthCheck` step that precedes it. A write that persisted a row
  under the wrong community would produce an in-spec trace unless a later read in the same
  request re-surfaced the row.

---

### BR-CONF-08 — `SanitizedError` closed alphabet

- **TLA+ source:** `SanitizedError(w)` binds `e \in SanitizedErrors` and emits
  `labels |-> {}` (`docs/spec/MultiTenantRelay.tla:778-788`);
  `Inv_SanitizedErrors == \A o : o.kind = "SanitizedError" => o.error \in SanitizedErrors`
  (`:1124-1126`).
- **Rust predicate:** `src/transitions.rs:206-210` matches all three `SanitizedReason`
  variants and returns `Ok(())` for each. The code comment concedes it is
  "trivially true by construction" (`src/transitions.rs:203-205`).
- **Where closure is actually enforced:** at the relay, by exhaustive match —
  `sanitized_reason_for` (`crates/buzz-relay/src/conformance/mod.rs:422-430`) maps
  `IngestError::{Rejected, AuthFailed, Internal}` → `{Invalid, Restricted, ServerError}`. A
  fourth `IngestError` variant breaks the build. That is a compile-time rule, not a trace rule.
- **Alphabet-width gap:** `docs/spec/MultiTenantRelay.cfg:26` declares nine sanitized errors;
  the Rust enum has three (`src/lib.rs:132-140`). A mis-bucketed reason (rate-limit reported as
  `Invalid`) is not detectable by any rule here. Detail is in the data-model doc, BR-CONF-05
  there.
- **Test:** `checker::tests::sanitized_error_alone_is_well_formed` (`src/checker.rs:326-336`)
  loops all three variants and asserts `Ok`.

---

### BR-CONF-09 — `ImplBug` is always a coverage breach

- **Predicate:** `TraceAction::ImplBug { kind } => Err(CoverageBreach { detail: "ImplBug action
  emitted by Drop guard: kind=…" })` (`src/transitions.rs:277-279`).
- **No spec counterpart:** `src/lib.rs:193-194` says so directly; it is a runtime witness, not
  a `Next` disjunct.
- **Two producers:** `EmitGuard::drop` when the counting tracer saw zero records
  (`crates/buzz-relay/src/conformance/mod.rs:404-414`), and the row-projection
  missing-lookup path in `record_read_message_rows` / `record_read_by_id_rows`
  (`mod.rs:284-296`, `:319-331`) with `kind = "row_community_lookup_missing"`
  (`mod.rs:250`).
- **Fail-closed on DB error, by accident of construction:** when `communities_of_channels`
  errors, `req.rs` substitutes an **empty** map (`crates/buzz-relay/src/handlers/req.rs:352`,
  `:668`), so every channel-scoped row misses the lookup and the seam emits `ImplBug` instead
  of a labelled read. That is the correct direction, but it means a transient DB error is
  indistinguishable from a real projection bug.
- **Tests:** `checker::tests::impl_bug_action_bites_coverage_breach` (`src/checker.rs:290-303`);
  property P4 `impl_bug_bites_coverage_breach` (`tests/proptest_checker.rs:328-350`);
  fixture `bad_coverage_breach.jsonl` (`tests/replay_fixtures.rs:267-278`); producer-side
  `emit_guard_drop_records_exactly_one_impl_bug_when_no_emit`
  (`crates/buzz-relay/src/conformance/mod.rs:497-527`) and
  `record_read_message_rows_missing_lookup_emits_impl_bug` (`mod.rs:674-696`).

---

### BR-CONF-10 — Scenario-required actions must all appear

- **Predicate:** after every step passes, the set of `action.kind()` strings seen
  (`src/checker.rs:113-118`) must be a superset of `required_critical_actions`; the missing
  names are sorted and formatted into `CoverageBreach`
  (`src/checker.rs:119-129`).
- **Comparison is by string,** using `TraceAction::kind()` (`src/lib.rs:266-277`) — a typo in a
  required-action name silently becomes an always-missing requirement, since nothing validates
  the strings against the enum.
- **Opt-in, and mostly opted out:** `Scenario::unstructured` sets an empty set
  (`src/checker.rs:45-50`) and accounts for 18 of the 22 scenario constructions in the crate
  (`src/checker.rs:166`, `:220`, `:239`, `:258`, `:282`, `:298`, `:334`;
  `tests/proptest_checker.rs:200`, `:262`, `:294`, `:320`, `:342`, `:394`, `:422`;
  `tests/replay_fixtures.rs:257`, `:271`, `:285`, `:298`). Only four sites declare
  requirements: `src/checker.rs:200-205`, `:315-317`, `tests/replay_fixtures.rs:240-247`,
  `:308-315`.
- **Tests:** `required_critical_action_missing_bites_coverage_breach` (`src/checker.rs:306-324`),
  `missing_required_action_is_coverage_breach` (`tests/replay_fixtures.rs:307-320`).

---

### Fail-fast ordering (a rule about the rules)

`check_step` returns on the first failure and `check_trace` propagates with `?`
(`src/checker.rs:109`). Because the three state checks run before the action match
(`src/transitions.rs:143-169` vs `:172+`), a step that violates both state stability and row
confinement reports `StateMismatch` only. The property tests are explicitly built around this
constraint — see the fail-fast discipline note at `tests/proptest_checker.rs:27-33`.

---

### Spec-action coverage of the rule set

`Next` (`docs/spec/MultiTenantRelay.tla:933-956`) offers **23** actions. The trace vocabulary
covers **8** of them; 15 have no representation and therefore no rule:

| Modeled (8) | Unmodeled (15) |
|---|---|
| `WriteInsert` (`:514`), `WriteInsertGlobal` (`:559`), `WriteDuplicate` (`:606`), `ReadMessageRows` (`:643`), `ReadByIdRows` (`:681`), `ReadHostFeedRows` (`:703`), `SanitizedError` (`:778`), `AuthCheck` (`:794`) | `ReadProjectionRows`, `ReadHostAuxRows` (`:726`), `ReadForgotPredicateWithRLS`, `ReadNoTenantContext`, `AppendAudit` (`:811`), `ObserveAuditHead` (`:819`), `RebuildProjections` (`:833`), `AuthenticateOpenCommunity`, `CreateChannel` (`:860`), `AdmitMember` (`:873`), `RevokeMember` (`:882`), `AddMembership`, `RemoveMembership`, `OpenChannel`, `CloseChannel` (`:926`) |

Of the 13 invariants in `Safety` (`docs/spec/MultiTenantRelay.tla:1129-1142`), the Rust rules
enforce **one and a half**: `Inv_NonInterference` fully for read observations (BR-CONF-05), and
a fragment of `AuthCheck`'s consequences (BR-CONF-06). The remaining eleven —
`Inv_LabelPropagation` (`:990`), `Inv_ReadConfinement` as a distinct check (`:999`),
`Inv_ResolutionFence` (`:1011`), `Inv_HostBindingFence` (`:1038`),
`Inv_ChannelCommunityImmutable` (`:1057`), `Inv_AdmissionFence` (`:1071`),
`Inv_AcceptedWritesPersist` (`:1104`), `Inv_MessageKeyUnique` (`:1110`),
`Inv_NoTenantContextFailsClosed` (`:1116`), `Inv_ProjectionDerived` (`:1121`),
`Inv_SanitizedErrors` beyond the 3-variant collapse (`:1124`) — have no Rust predicate,
because the state they quantify over is absent from `AbstractState`.


## Module: buzz-acp — harness core & orchestration (`crates/buzz-acp/src`)
### Aspect: Business Rules

#### Inbound event pipeline order (fixed, per event)

The main loop's relay branch (`lib.rs:1855-2300`) applies gates in a strict order. Order is load-bearing and commented as such at several sites.

| # | Gate | Line | Behaviour on reject |
|---|---|---|---|
| 1 | membership notification intercept (44100/44101) | `lib.rs:1861-1949` | `continue` — never reaches the agent |
| 2 | `ignore_self` (own pubkey) | `lib.rs:2027-2030` | drop |
| 3 | `!shutdown` from owner | `lib.rs:2033-2059` | consume |
| 4 | `!cancel` from owner | `lib.rs:2064-2092` | consume |
| 5 | `!rotate` from owner | `lib.rs:2102-2133` | consume |
| 6 | inbound author gate | `lib.rs:2135-2172` | drop (debug log only) |
| 7 | subscription rule match | `lib.rs:2172-2181` | drop |
| 8 | `queue.push` + dedup | `lib.rs:2196-2201` | `DedupMode::Drop` returns `false`, no 👀 |
| 9 | mid-turn mode gate | `lib.rs:2215-2258` | may steer/interrupt the running turn |

Owner control commands are deliberately checked **before** the author gate (comment `lib.rs:2133-2136`) so `--respond-to nobody` cannot lock the owner out.

#### Author gate (`author_allowed`, `lib.rs:235-256`)

| Mode | Non-DM channel | DM channel |
|---|---|---|
| `Anyone` | accept all | **owner ∪ verified siblings only** |
| `Nobody` | reject all | reject all |
| `OwnerOnly` | owner ∪ siblings | owner ∪ siblings |
| `Allowlist` | allowlist ∪ owner ∪ siblings | **owner ∪ siblings only — allowlist ignored** |

The DM branch short-circuits first (`lib.rs:242-247`). Rationale documented at `lib.rs:220-233`: clients auto-`p`-tag every DM participant, so in a DM any participant's message reads as a mention, and since the agent can be asked to DM a third party, `anyone`/`allowlist` would become a transitive access grant.

`is_dm_channel` (`lib.rs:273-286`) **fails closed** — an unresolvable channel type is treated as a DM and the misclassification is deliberately *not* cached so a later event retries (documented `lib.rs:261-272`, enforced by `ChannelInfoResolver::resolve`).

`is_owner_or_sibling` (`lib.rs:192-215`) also fails closed: no configured owner ⇒ `false` (`lib.rs:200`). Sibling proof requires a locally verified NIP-OA Schnorr signature over the `["auth", owner, conditions, sig]` tag from the author's kind:0 profile — `check_sibling_via_profile` (`lib.rs:291-362`) re-verifies rather than trusting the relay (comment `lib.rs:322-323`), with a 2,000 ms timeout that fails closed (`lib.rs:310-315`).

Sibling verdicts cache for the process lifetime; at 256 entries the whole map is cleared rather than evicted LRU (`lib.rs:183-186`), so a hot sibling can be forced back through a REST round-trip by cache churn.

#### Owner resolution priority

`resolve_agent_owner` (`lib.rs:123-148`): `BUZZ_AUTH_TAG` NIP-OA attestation verified against the agent's own pubkey (`lib.rs:125-137`), else `--agent-owner` / `BUZZ_ACP_AGENT_OWNER` (`lib.rs:147`). A failed attestation verification logs a warning and silently falls through to the unverified flag (`lib.rs:138-141`).

Owner is resolved exactly once at startup. `OwnerCache.pubkey` is immutable (`lib.rs:161-163` returns `&str`, no setter). Consequences: with `respond-to=owner-only` and no owner, every event is dropped for the process lifetime — warned at `lib.rs:1379-1384` but never retried. `--relay-observer` is silently downgraded to disabled when no owner resolved (`lib.rs:1421-1425`).

#### Turn lifecycle

Dispatch (`dispatch_pending`, `lib.rs:2889-3000`) loops until the queue or the pool is exhausted:

1. `queue.flush_next()` drains one channel's whole backlog into one `FlushBatch` (`lib.rs:2892`).
2. Typing scope is derived from the **last** event in the batch (`lib.rs:2897-2900`).
3. `pool.try_claim(Some(channel_id))` prefers session affinity (`lib.rs:2902`).
4. On pool exhaustion: `requeue_preserve_timestamps` then `mark_complete`, then `break` (`lib.rs:2906-2910`) — timestamps preserved so retry ordering is not reset.
5. A recovery copy of the batch is retained **only** under `DedupMode::Queue` (`lib.rs:2915-2918`); `DedupMode::Drop` stores `None`, so a panic loses the batch.
6. A capacity-1 steer channel is installed on **every** prompt task regardless of agent capability (`lib.rs:2933-2934`, comment `lib.rs:2929-2932`) — "try-and-tolerate".
7. `turn_id = Uuid::new_v4()` (`lib.rs:2963`), task spawned into `pool.join_set`, `TaskMeta` recorded keyed by `abort_handle.id()` (`lib.rs:2977-2987`).

Exactly one prompt per channel is in flight; concurrency across channels is bounded by `config.agents` (1–32, enforced in `config.rs:292-296`).

#### Mid-turn arrival: `mode_gate_signal` (`lib.rs:2741-2757`)

| `--multiple-event-handling` | Signal | Author re-check |
|---|---|---|
| `queue` | `None` — event waits | — |
| `steer` | `ControlSignal::Steer` | none (gate already ran) |
| `interrupt` | `ControlSignal::Interrupt` | none |
| `owner-interrupt` | `ControlSignal::Interrupt` | owner only (`lib.rs:2751-2754`) |

`Steer` takes a two-stage path. `try_native_steer` (`lib.rs:2803-2887`) first attempts the non-cancelling ACP steer:

- Builds the body from `queue::native_steer_framing()` + `queue::format_event_block` so native and fallback framing cannot drift (comment `lib.rs:2812-2825`).
- Deliberately passes `None` for `channel_info` and `profile_lookup` — a steer is a *delta*, the agent already saw that context (`lib.rs:2820-2825`).
- On `Ok(())` it withholds the queued event **synchronously before** spawning the ack watcher (`lib.rs:2836-2839`) to close the `mark_complete` → stray-`flush_next` re-delivery race.
- On `Err` it returns `false` and the caller falls through to cancel+merge (`lib.rs:2249-2256`).

The ack arm (`lib.rs:2417-2500`) resolves a three-way decision `(release_withheld, drop_withheld, signal_fallback)`:

| Ack | release | drop | fallback |
|---|---|---|---|
| `Success` | no | yes | no |
| `AgentError{code == -32601}` | yes | no | **yes** |
| `AgentError{other}` | yes | no | no |
| other `SteerError` (transport / `ExpectedRunIdMissing`) | yes | no | **yes** |
| `PromptCompletedNeutral` | yes | no | no |
| watcher `Err` (oneshot dropped) | yes | no | no |

On `Success` the in-flight deadline is extended by `max_turn_duration_secs` (`lib.rs:2487-2489`) — a steered turn gets a fresh full budget, so repeated steering can extend a single turn indefinitely past the configured cap.

#### Outcome → retry policy (`handle_prompt_result`, `lib.rs:3034-3399`)

Requeue happens **before** `mark_complete` because `requeue()` sets `retry_after` that `mark_complete()` reads to decide whether to preserve `retry_counts` (comment `lib.rs:3048-3051`). Inverting the order silently resets every retry to attempt 1.

| Outcome | Batch fate | Agent fate |
|---|---|---|
| `Ok` | consumed | returned to pool (`lib.rs:3241`) |
| `Cancelled` / `CancelDrainTimeout` | `requeue_as_cancelled(reason)`, default `Steer` if unset (`lib.rs:3071-3072`) | `Cancelled` → pool (`lib.rs:3335`); `CancelDrainTimeout` → **respawn** (`lib.rs:3290-3305`) |
| `Timeout(Hard{recently_active:false})` | dead-lettered immediately + user notice (`lib.rs:3082-3096`) | respawn |
| `Timeout(Hard{recently_active:true})` | `requeue()`; notice only if budget exhausted (`lib.rs:3097-3117`) | respawn |
| `Timeout(Idle)` | `requeue()` (`lib.rs:3135`) | respawn |
| `AgentExited` | `requeue()` | respawn |
| `Error(e)` where `is_auth_error(e)` | dead-lettered immediately, no retry (`lib.rs:3118-3133`) | pool (pipe intact) |
| `Error(transport-class)` — `Io`/`WriteTimeout`/`Timeout`/`Protocol` | `requeue()` | respawn (`lib.rs:3341-3346`) |
| `Error(application-class)` | `requeue()` | pool (`lib.rs:3373-3383`) |
| any outcome, channel in `removed_channels` | dropped (`lib.rs:3149-3155`) | unchanged |

`is_auth_error` (`lib.rs:3003-3011`) matches on **substrings of vendor error text**: `"Re-authenticate"` or `"API Error: 401"`. Rationale for high-precision matching is documented at `lib.rs:2989-3002`. This is a string-matching contract against upstream CLI messages with no version pin.

Sessions for channels the agent lost membership to are stripped when the agent returns (`lib.rs:3166-3169`), covering the gap that `invalidate_channel_sessions` only reaches *idle* agents.

#### Circuit breaker (`SlotCircuit`, `lib.rs:1048-1134`)

- 3 crashes inside a 60 s window opens the circuit for 300 s (`lib.rs:1082-1085`).
- Backoff before threshold: `1s × 2^(recent-1)` capped at 30 s, ±20 % jitter derived from `SystemTime` subsec nanos (`lib.rs:1088-1097`) — not a PRNG, so co-started slots crashing in the same nanosecond window get correlated jitter.
- Half-open probe pre-seeds `crash_times` to `THRESHOLD-1` so a single probe failure re-opens immediately (`lib.rs:1064-1074`, mirrored in `can_refill` `lib.rs:1113-1131`) — "prove stability for one full window".
- `mark_spawn_failed` (`lib.rs:1103-1105`) uses a fresh `Instant::now()` so spawn latency cannot shorten the cooldown.
- `record_crash` is documented as the single canonical crash→respawn path (`lib.rs:1050-1053`), called from `spawn_respawn_task` (`lib.rs:3646`), `recover_panicked_agent` (`lib.rs:3465`), and slot refill via `can_refill` (`lib.rs:1762`).

Exit condition: `pool.live_count() == 0 && !any_respawn_in_flight(&crash_history)` (`lib.rs:2373`, `3277`, `3306`, `3355`, `3527`). The `any_respawn_in_flight` conjunct (`lib.rs:1136-1138`) prevents a premature "all agents dead" exit while a respawn is pending.

#### Heartbeat rules (`lib.rs:2265-2298`, `dispatch_heartbeat` `lib.rs:3537-3586`)

Priority order, all enforced in the tick arm:

1. skipped if `!pool_ready` (`lib.rs:2270`);
2. queued work wins — the tick becomes a `dispatch_pending` call instead (`lib.rs:2272-2278`);
3. requires an idle agent (`lib.rs:2279`), else dropped with `heartbeat_skipped_busy` — no queuing;
4. at most one globally, guarded by `heartbeat_in_flight` (`lib.rs:3541-3543`).

Interval must be `0` or ≥ 10 s (README; validated in `config.rs`). Default prompt (`default_heartbeat_prompt`, `lib.rs:3609-3633`) instructs `buzz feed get --types needs_action` then `--types mentions`, and explicitly tells the agent not to run `buzz channels list` or `buzz messages search`. The README (`README.md § Heartbeat Semantics`) still documents the older `get_feed_actions()` / `get_feed_mentions()` call names.

#### Membership notification dedup (two layers, `lib.rs:1865-1889`)

1. Exact event-ID rejection across two generations (`seen_membership_current` ∪ `seen_membership_previous`), rotated at 1,000 entries so there is no amnesia window (`lib.rs:1877-1881`).
2. Timestamp watermark per channel using strict `<` (`lib.rs:1882-1888`). Strict is deliberate (`lib.rs:1653-1658`, `1867-1876`): `<=` would suppress a legitimate live add→remove pair in the same second, leaving the harness with wrong membership state.

Known accepted race, documented at `lib.rs:1670-1680`: a batch in flight when a channel is removed **and re-added** before it returns may be requeued. The comment states the fix would need per-channel epoch tracking on `TaskMeta` and `PromptResult` and judges it not worth the complexity.

#### Observer frame pacing and trimming

- Pacer (`lib.rs:372-409`) enforces 167 ms spacing plus a 90-frames-per-rolling-minute ceiling, with **no initial burst** — even the first snapshot frame waits a slot (`lib.rs:375-376`).
- Snapshot/live overlap is deduped by the snapshot's high-water `seq` (`lib.rs:443`, `lib.rs:470-473`); subscribe happens before snapshot so no event can fall between them (`lib.rs:420-423`).
- Oversized frames are trimmed rather than dropped. `fit_observer_event_to_budget` (`lib.rs:659-687`) repeatedly elides the largest *strictly shrinkable* string leaf and reserializes, falling back to a stub payload when no leaf can shrink. Termination is argued in the doc comment (`lib.rs:645-651`): monotone decrease bounded below by the stub. `leaf_shrinks` (`lib.rs:739-747`) includes a marker-pushback guard so a leaf near its retained floor is never touched.
- Elision boundaries snap to UTF-8 char boundaries (`lib.rs:770-789`).

#### Owner control-frame acceptance (`lib.rs:837-870`)

Three defence-in-depth checks before decryption, each rejecting outright:

1. `buzz_core::verify_event` signature re-verification (`lib.rs:845-848`);
2. sender pubkey must equal the resolved owner hex (`lib.rs:851-858`);
3. `created_at` within ±300 s (`lib.rs:861-869`).


## Module: buzz-acp — relay client & observer (`crates/buzz-acp/src`)
### Aspect: Business Rules

#### NIP-42 handshake sequence

`do_connect` (`relay.rs:3825-3862`) is the whole handshake:

1. Parse the URL with `url::Url` (`relay.rs:3830-3832`) — parse only, no scheme assertion.
2. `connect_async` wrapped in `CONNECT_TIMEOUT` = 30 s (`relay.rs:3834-3837`); a timeout maps to `RelayError::ConnectionClosed` (`relay.rs:3836`).
3. `wait_for_auth_challenge(ws, &mut buffer, AUTH_TIMEOUT)` — `AUTH_TIMEOUT` = 20 s (`relay.rs:64`, called at `relay.rs:3843`). Non-AUTH frames are pushed into `buffer` (`relay.rs:3902`); `Ping` is answered with `Pong` (`relay.rs:3903-3907`); `Close` → `ConnectionClosed` (`relay.rs:3909`).
4. `send_auth_response` (`relay.rs:3845`, defined `:3433-3461`). With no `auth_tag` it uses `EventBuilder::auth(challenge, relay_nostr_url)` (`relay.rs:3457`); with one it hand-builds `["relay", url]`, `["challenge", challenge]`, plus the NIP-OA tag and signs `Kind::Authentication` (`relay.rs:3444-3456`), because `EventBuilder::auth` accepts no extra tags (`relay.rs:3443`).
5. `wait_for_any_ok` (`relay.rs:3849`, defined `:3924-3982`) accepts the **first** OK of any event id. The inline comment concedes the event id is not tracked (`relay.rs:3847-3854`). `accepted == false` → `RelayError::AuthFailed(ok.message)` (`relay.rs:3850-3852`).
6. Returns `(ws, buffer)`; the buffer is replayed by `process_handshake_buffer` (`relay.rs:2393-2450`).

`retry_initial_connect` (`relay.rs:3786-3822`) wraps step 1-6: 1 immediate
attempt + 5 delayed retries over `STARTUP_CONNECT_BACKOFFS` = 1/2/4/8/16 s
(`relay.rs:83-89`), each jittered ±20 %. A terminal error short-circuits
(`relay.rs:3809-3812`).

Terminal-vs-transient classification is exhaustive and compile-checked:
`is_terminal_connect_error` (`relay.rs:3657-3664`) → `is_terminal_ws_error`
(`relay.rs:3669-3717`, no wildcard arm) → `is_terminal_rustls_io_error`
(`relay.rs:3724-3766`, source-chain downcast to `rustls::Error` with a
5-variant terminal allowlist at `:3757-3764`). Auth denials split on the NIP-01
prefix: only `error:` is retried (`is_terminal_auth_failure`,
`relay.rs:3773-3775`).

#### Mid-session AUTH: this module re-authenticates

On a mid-session `AUTH` frame `handle_ws_message` calls `send_auth_response`
immediately and forces a reconnect if the send fails
(`relay.rs:2344-2353`). A rejected re-auth (`OK false` with a message starting
`auth`) also forces reconnect (`relay.rs:2359-2363`).

This is a functional divergence from the shared crate:
`buzz-ws-client` only records the challenge — `recv_one` stores it in
`pending_challenge` and returns the message
(`buzz-ws-client/src/connection.rs:144`); `wait_for_ok` does the same and
re-buffers the frame (`buzz-ws-client/src/connection.rs:256-258`). No
re-authentication is ever issued.

Buffered `AUTH` frames from the handshake window are **not** answered — they are
skipped on replay (`relay.rs:2432-2433`).

#### Subscription lifecycle

Subscribe intent is recorded before, and independently of, the wire send.
`execute_connected_command` (`relay.rs:1346-1531`):

- Rate-gated → `apply_command_to_state` + park in `rate_limited_pending`, return `true` ("connection is fine") (`relay.rs:1360-1375`).
- Otherwise seed `subscribe_since` (`relay.rs:1380-1384`), compute `since = last_seen.or(subscribe_since)` (`relay.rs:1385-1389`), send.
- On success: insert into `active_subscriptions` + `active_filters`, evict stale drain entries (`relay.rs:1392-1404`).
- On failure: `apply_command_to_state` records intent, return `false` → caller reconnects (`relay.rs:1405-1417`).

`since` policy in `send_subscribe` (`relay.rs:3185-3194`): `Some(ts)` →
`ts - SINCE_SKEW_SECS` (5 s, `relay.rs:57`); `None` → `now`, deliberately
skipping history.

`Unsubscribe` is best-effort at the socket level: a failed CLOSE frame does not
fail the command because state has already been mutated
(`relay.rs:1419-1432`).

CLOSED handling is a four-way branch (`relay.rs:2210-2346`):

1. Exact per-channel denial → drop just that channel, keep the socket. Only the two literals in `CHANNEL_ACCESS_DENIED_REASONS` match — `"restricted: not a channel member"`, `"restricted: channel access revoked"` (`relay.rs:3498-3501`) — compared with `contains(&message)`, never `starts_with`, so connection-level `restricted: insufficient scope` still reconnects (`relay.rs:3515-3517`, rationale at `:3489-3496`).
2. `rate-limited:` prefix → arm the gate, park, keep the socket (`relay.rs:2226-2246`).
3. `auth-required` / `starts_with("restricted")` / `contains("auth")` → return `false`, full reconnect (`relay.rs:2250-2264`).
4. Anything else → targeted resubscribe of that one subscription, with a fail-closed guard: a missing `active_filters` entry triggers reconnect rather than a wildcard REQ (`relay.rs:2308-2318`, and `:2513-2521` on the reconnect path). A CLOSED for an already-unsubscribed channel is ignored so it cannot resurrect a subscription (`relay.rs:2300-2306`).

#### EOSE handling

`RelayMessage::Eose` is logged at debug and otherwise discarded
(`relay.rs:2190-2192`). Nothing in `BgState` records EOSE, so the module cannot
distinguish stored-history replay from live events, and no caller is told when
initial replay finished.

#### Dedup

Channel events go through `record_event` (`relay.rs:1091-1107`): insert into
`TwoGenDedup`, return `false` on duplicate, otherwise advance `last_seen` with
`max` (`relay.rs:1100-1105`).

Membership notifications deliberately bypass `record_event` and touch
`seen_ids` directly (`relay.rs:2093-2100`) so membership timestamps cannot
contaminate per-channel replay watermarks — the reason is spelled out at
`relay.rs:2086-2092`.

On backpressure the id is **removed** from the dedup set so reconnect replay can
re-deliver it (`relay.rs:2124` for membership, `:2162` for channel events).

#### Backpressure → proactive resubscribe

`event_tx.try_send` full (`relay.rs:2121-2143` membership, `:2155-2181`
channel):
1. `seen_ids.remove(id)`.
2. Record the oldest dropped ts (`membership_dropped_since` via `min`, or `channel_dropped_since` entry via `min`).
3. Set `proactive_resubscribe_needed = true`.

The main loop checks the flag at the top of every iteration and resubscribes on
the **existing** socket with `is_fresh_connection = false`, explicitly
preserving the rate-limit gate (`relay.rs:1628-1640`). An 80 %-capacity warning
fires before the drop (`relay.rs:2109-2113`, `:2144-2151`).

#### Rate-limit gate

`set_rate_limit_gate` (`relay.rs:1156-1165`): a hint below 2 s (including the
no-hint 0) floors to 5 s; the deadline is jittered; the gate takes the **max**
of any existing deadline so overlapping CLOSED/NOTICE cannot shorten it.
`check_rate_gate` lazily clears an expired gate (`relay.rs:1173-1181`).
`parse_rate_limit_retry_secs` splits on `"retry in "` and takes ASCII digits —
no regex (`relay.rs:3328-3332`).

The gate is armed from both `NOTICE` (`relay.rs:2193-2208`) and `CLOSED`
(`relay.rs:2226-2246`). A NOTICE additionally requeues all unacked observer
frames because NOTICE carries no event id (`relay.rs:2200`, implementation
`relay.rs:1202-1210`).

Drain pacing: `REQ_PACING_INTERVAL` = 125 ms (`relay.rs:107`) and
`DRAIN_BUDGET_PER_ITER` = 1 (`relay.rs:113`) — one frame per select-loop tick,
stated to stay under the relay's 50-frames/5 s window
(`relay.rs:103-106`). Drain order per tick (`relay.rs:1706-1779`): membership
control sub → observer control sub → `rate_limited_pending` →
`resubscribe_retry` → `gated_observer_pending`. When nothing is sent because the
gate is still armed, the pacing timer is set to the gate deadline so parked
observer frames drain promptly (`relay.rs:1774-1779`).

A failed drain REQ re-queues the channel with a flat +5 s penalty
(`relay.rs:2712-2716`).

#### Observer frame durability rule

Kind 24200 is treated as durable telemetry, every other WS publish as droppable
ephemera:

- While gated **or** while a parked backlog exists, observer frames are parked to preserve order (`relay.rs:1470-1481`).
- Every other publish is silently dropped while gated (`relay.rs:1483-1495`); the invariant that only ephemeral kinds reach this path is documented rather than enforced (`relay.rs:1487-1493`).
- Successful writes move to `observer_in_flight` (`relay.rs:1500-1502`), retired on the matching `OK` (`relay.rs:2364`, implementation `relay.rs:1224-1232`).
- Both queues are bounded at `GATED_OBSERVER_QUEUE_CAP` = 256 with drop-oldest and a visible counter (`relay.rs:1187-1200`, `:1212-1222`, summarised at `:2655-2662`).
- While disconnected, `apply_command_to_state` parks observer frames and drops everything else (`relay.rs:1229-1237`); `retain_failed_command_intent` does the same after a live send failure (`relay.rs:1307-1316`).

#### Reconnect / backoff

Three distinct loops:

| Loop | Location | Ladder | DNS handling |
|---|---|---|---|
| `retry_initial_connect` | `relay.rs:3786-3822` | 1 immediate + 5 delayed (all of `STARTUP_CONNECT_BACKOFFS`) | none — DNS is transient `Io`, consumes a rung |
| `try_autonomous_reconnect` | `relay.rs:2893-3018` | 5 attempts, first 4 delays used (sleep skipped on the last, `relay.rs:3000`) | flat 2 s (`DNS_RETRY_INTERVAL`, `relay.rs:98`), capped at 10 retries (`relay.rs:2914`, `:3006-3008`) |
| `wait_for_reconnect` | `relay.rs:3022-3151` | 1/2/4/8/16/32 s then flat 60 s (`relay.rs:3055-3062`, `:3128-3132`), resumes from `state.backoff_step` (`relay.rs:3063`) | flat 2 s, **unbounded** (`relay.rs:3111-3117`) |

Backoff sleeps are deadline-based (`sleep_until`) and processed inside a
`select!` so `Shutdown` is honoured mid-sleep without resetting the timer — the
stated reason is that 3-second typing refreshes would otherwise collapse the
backoff into a reconnect storm (`relay.rs:3122-3127`, `:2993-2999`).
`backoff_step` is persisted before sleeping (`relay.rs:3125`) and reset to 0 by
the stability block after `STABLE_CONNECTION_SECS` = 60 s
(`relay.rs:2026-2035`).

A handshake-buffer drop signal after reconnect falls through to the backoff
sleep rather than tight-looping (`relay.rs:2932-2940`, `:3084-3090`).

#### Resubscribe after reconnect

`resubscribe_after_reconnect` (`relay.rs:2489-2607`):
- On a fresh connection, `rate_limited_pending` and `resubscribe_retry` are cleared but the gate is deliberately preserved, because the relay's quota is keyed by community+pubkey and survives socket replacement (`relay.rs:2496-2503`, rationale `:2481-2485`).
- Channels are replayed with 125 ms pacing via `pacing_sleep` (`relay.rs:2543-2547`), which defers non-`Shutdown` commands in arrival order (`relay.rs:3369-3392`).
- A gate re-armed mid-burst parks the remaining channels instead of sending (`relay.rs:2518-2526`).
- A failed **channel** REQ is parked in `resubscribe_retry` and does not abort the reconnect (`relay.rs:2548-2557`); a failed **membership** or **observer-control** REQ, or a failed deferred command, returns `RetryConnection` (`relay.rs:2576-2578`, `:2597-2601`).
- Deferred commands are then executed via `drain_commands` (`relay.rs:2600`), which preserves FIFO order across deferred and newly-arrived commands and paces only actual live sends (`relay.rs:2793-2865`).

#### Membership-notification rules

Both membership kinds require an `h` tag; without one the event is dropped
(`relay.rs:2078-2085`, extractor at `relay.rs:3418-3427`). Successful enqueue
advances `membership_last_seen` with `max` (`relay.rs:2117-2119`).
`SetStartupWatermark` seeds `membership_last_seen` if unset
(`relay.rs:1263-1266`, `:1523-1525`).

#### Archived-channel exclusion

`merge_discovered_channels` (`relay.rs:171-232`) drops any channel whose
kind:39000 metadata carries `["archived","true"]` — even when the agent is still
a listed member. The stated purpose is to stop a `CLOSED restricted` reconnect
loop for channels reaped while the agent was offline (`relay.rs:163-170`).
`archived=false` is kept (test at `relay.rs:4140-4149`). A channel with no
metadata becomes `("unknown","unknown")` (`relay.rs:219-221`) so security
consumers can fail closed.

#### Engram fetch ordering and caching

`decode_core_body` (`engram_fetch.rs:110-165`) implements a three-way
fail-closed rule:

| Relay result | Outcome | Rendered section |
|---|---|---|
| empty array | `Ok(None)` — confirmed absence | `[Agent Memory — core]\n<ONBOARDING_NUDGE>` (`engram_fetch.rs:47`) |
| ≥1 event decrypts, head is `Body::Core` | `Ok(Some(profile))` | `[Agent Memory — core]\n<profile>` (`engram_fetch.rs:46`) |
| ≥1 event decrypts, head is tombstone/other | `Ok(None)` (`engram_fetch.rs:159-161`) | nudge |
| non-empty but nothing decrypts | `Err` (`engram_fetch.rs:139-148`) | **no section** (`engram_fetch.rs:49-56`) |
| transport / non-array / parse failure | `Err` (`engram_fetch.rs:92-99`) | no section |

The `Err` → no-section rule exists so a relay outage is never mistaken for "no
core", which would invite the agent to overwrite real memory
(`engram_fetch.rs:8-11`, `:63-70`). Every candidate is signature-verified
locally before decryption (`engram_fetch.rs:126-128`) — the relay is not
trusted. `select_head` resolves the LWW winner (`engram_fetch.rs:149`) and the
body is matched back by event id (`engram_fetch.rs:152-156`).

Caching lives in the caller, not here: `pool.rs:1382-1414` fetches once per new
channel session under a 3 s timeout and stores the rendered string in
`SessionState::core_sections`; a timeout yields `None` (`pool.rs:1394-1404`).
Invalidation is via `SessionState::invalidate_channel` / `invalidate_all`. No
mid-session refresh.

#### Observer gating

`observer.rs` applies almost no gating: `emit` always assigns a `seq` and always
attempts the broadcast send (`observer.rs:106-135`). The only bounds are the
1,000-entry drop-front replay buffer (`observer.rs:126-129`) and the
1,000-slot broadcast channel (`observer.rs:46`). A poisoned buffer mutex is
logged and skipped, not propagated (`observer.rs:96-100`, `:129-131`).
The handle is only constructed when `config.relay_observer` is true
(`lib.rs:1244-1246`), which is the actual on/off gate.

Relay-side gating of observer control frames is in `lib.rs`, not this module:
signature re-verification, owner-pubkey match, and a ±300 s freshness window
(`OBSERVER_CONTROL_FRESHNESS_SECS`). `relay.rs:2069-2076` forwards the raw
signed event with no checks beyond subscription-id routing.


## Module: buzz-acp — agent pool & lifecycle (`crates/buzz-acp/src`)
### Aspect: Business Rules

#### Admission: claiming an agent

`try_claim` (`pool.rs:558-575`) runs two passes and never blocks:

1. Affinity pass — if a `channel_id` is supplied, take the first idle agent whose `state.sessions` already contains that channel (`pool.rs:560-568`). Session reuse is the sole affinity criterion; no load, age, or model is considered.
2. Fallback pass — take the first idle slot in index order (`pool.rs:571-574`).

`None` means every agent is checked out. The caller then requeues the batch preserving timestamps, marks the channel complete, and **breaks out of the dispatch loop entirely** rather than trying the next channel (`lib.rs:2909-2916`), logging `pool_exhausted` at `debug`. One exhausted claim therefore stalls dispatch for all other pending channels until the next loop iteration.

Concurrency limits:
- Pool width is `config.agents`, clamped to `1..=32` by clap (`config.rs:292-295`), default `1`. The `agents` vector is allocated once and never resized.
- At most one prompt in flight per channel — enforced by `EventQueue`, not the pool; `dispatch_pending` calls `queue.mark_complete(channel_id)` and `flush_next` skips in-flight channels (`lib.rs:2910-2914`).
- Heartbeats claim with `try_claim(None)` and are simply dropped when nothing is idle (`lib.rs:3545-3552`); `heartbeat_in_flight` is a single bool, so at most one heartbeat turn exists at a time.
- No per-agent queue depth, no fairness or aging, no priority. There is no idle-eviction path anywhere in the module: an idle agent is never reaped for inactivity, only on crash, poisoning, or process shutdown.

#### Return path and the double-return rule

`return_agent` (`pool.rs:577-591`) writes `agents[agent.index] = Some(agent)`. If the slot is already occupied it logs `tracing::error!("BUG: return_agent called for slot {idx} which is already occupied — overwriting")` and overwrites, deliberately choosing to leak the previously-resident `AcpClient` (dropping its child via best-effort `Drop`) over permanently leaking the slot (`pool.rs:579-588`). Nothing prevents the caller from returning an agent whose index is out of range — `self.agents[idx]` would panic.

Every `PromptResult`-based return funnels through `send_prompt_result` (`pool.rs:1235-1252`), which clears the per-turn steer receiver first so the next dispatch's `install_steer_rx` assertion cannot trip (`pool.rs:1223-1233`).

#### Lifecycle state machine — deferred pool start

States: `Listening`, `Waking { attempt }`, `Ready(P)`, `Failed { attempt, retry_at, error }` (`pool_lifecycle.rs:14-25`). Complete transition table:

| From | Call | Guard | To | Return | Site |
|---|---|---|---|---|---|
| `Listening` | `start_wake_if_due(false, _)` | no pending work | `Listening` | `None` | `pool_lifecycle.rs:41-43` |
| `Listening` | `start_wake_if_due(true, now)` | — | `Waking{1}` | `Some(1)` | `:47`, `:55-58` |
| `Failed{a, retry_at}` | `start_wake_if_due(true, now)` | `now >= retry_at` | `Waking{a+1}` | `Some(a+1)` | `:48-51`, `:55-58` |
| `Failed{a, retry_at}` | `start_wake_if_due(true, now)` | `now < retry_at` | unchanged | `None` | `:52` |
| `Failed` | `start_wake_if_due(false, _)` | no pending work | unchanged | `None` | `:41-43` |
| `Waking{a}` | `start_wake_if_due(true, _)` | — | unchanged | `None` | `:52` |
| `Ready(p)` | `start_wake_if_due(true, _)` | — | unchanged | `None` | `:52` |
| `Waking{a}` | `complete_wake(a, Ok(p), _)` | attempt matches | `Ready(p)` | `Ok(())` | `:101`, `:110-111` |
| `Waking{a}` | `complete_wake(a, Err(e), now)` | attempt matches | `Failed{a, now+retry_delay(a), e}` | `Ok(())` | `:112-117` |
| `Waking{a}` | `complete_wake(b, _, _)` | `b != a` | unchanged | `Err("wake result attempt did not match Waking attempt")` | `:105-106` |
| `Listening`/`Ready`/`Failed` | `complete_wake(..)` | — | unchanged | `Err("wake completed while lifecycle was not Waking")` | `:107` |
| `Waking{a}` | `cancel_wake(a, e, now)` | attempt matches | `Failed{a, now+retry_delay(a), e}` | `true` | `:91-93` |
| `Waking{a}` | `cancel_wake(b, ..)` | `b != a` | unchanged | `false` | `:91-93` via `:105` |
| `Ready(p)` | `take_ready()` | — | `Listening` | `Some(p)` | `:60-68` |
| any other | `take_ready()` | — | unchanged | `None` | `:63-66` |

Invariants the machine enforces:
- Exactly one wake task per `Waking` entry: the attempt token is handed out only on the transition, and a second `start_wake_if_due` in the same state returns `None` (`pool_lifecycle.rs:31-35`, test `first_pending_event_starts_exactly_one_wake` at `:139`).
- A stale attempt's success cannot replace a newer pool — the doc states accepting it "could replace a newer pool" (`pool_lifecycle.rs:95-98`), test `stale_attempt_result_cannot_replace_current_wake` (`:244`).
- `Ready` is transient: `take_ready` mem-replaces with `Listening` (`:61`), so after a successful wake the machine no longer records success. Suppression of further wakes depends on the caller's separate `pool_ready` flag (`lib.rs:1322`, gate at `lib.rs:1714`).
- A wake starts only when work is buffered, so an idle harness never spawns subprocesses (`pool_lifecycle.rs:41-43`; caller feeds `queue.has_flushable_work()` at `lib.rs:1715`).

Backoff policy (`pool_lifecycle.rs:122-131`): `retry_delay(attempt) = min(5s * 2^(attempt-1), 300s)`, with the shift exponent clamped to 63 and `checked_shl` falling back to `u64::MAX`, then `saturating_mul`. Sequence: 5, 10, 20, 40, 80, 160, then 300 s forever. Retries are unbounded — there is no attempt ceiling and no circuit breaker on the wake path, unlike the per-slot respawn circuit. Tests pin `retry_delay(7) == 300s` and `retry_delay(u32::MAX) == 300s` (`pool_lifecycle.rs:196-197`).

Caller-side coupling: the retry sleep arm is gated on `lazy_wake_work_pending` because a past `retry_at` would otherwise resolve instantly every iteration — an explicit busy-spin guard (`lib.rs:1709-1713`, `lib.rs:1852-1861`). A panicked wake task is converted into `cancel_wake` on the current attempt (`lib.rs:1862-1878`). A wake result that arrives after the channel send fails is shut down rather than leaked (`lib.rs:1736-1741`).

#### Turn lifecycle inside `run_prompt_task`

Ordered phases (`pool.rs:1265-2212`):

1. Classify source from `batch` presence (`pool.rs:1276-1279`), set observer context, emit `turn_started` (`pool.rs:1295-1304`).
2. Install `TurnCompletionGuard` (`:1309`), start liveness task + `LivenessGuard` (`:1325-1342`), install `ReactionGuard` (`:1351`).
3. NIP-AE core fetch — only when `memory_enabled`, source is a channel, an owner pubkey exists, the channel has no session, and no cached section; bounded by 3 s, all failures inject nothing (`pool.rs:1379-1416`).
4. Canvas fetch — same once-per-new-session lifecycle; DM status resolved from the resolver and **unknown is treated as DM (fail closed, no canvas)** (`pool.rs:1428-1449`); the fetched section is held in `pending_canvas` and committed only after session creation succeeds (`pool.rs:1429`, commit at `:1489-1491`).
5. Session resolve or create (`pool.rs:1466-1560`). Creation applies, in order: combined system prompt → Goose custom system-prompt probe → capability capture → `desired_model` switch → `session_config_captured` observer emit → permission mode (`pool.rs:804-936`).
6. Optional `initial_message` on brand-new channel sessions only (`pool.rs:1580-1712`).
7. Prompt assembly: slash-command extraction (`pool.rs:1761-1769`), context fetch when `context_message_limit > 0`, profile lookup, `format_prompt`.
8. 💬 reaction fired without awaiting (`pool.rs:1802-1809`).
9. The prompt itself, wrapped in `biased` `select!` against the control channel when `control_rx` is `Some` (`pool.rs:1827-1990`).
10. Outcome classification (`pool.rs:1992-2209`), then guards drop.

Rotation rules on success (`pool.rs:1994-2022`): rotate when `stop_reason` is `MaxTokens` or `MaxTurnRequests`, **or** when `max_turns_per_session > 0` and the per-source counter reaches it. Rotation means `state.invalidate(&source)` — the next turn creates a fresh session, re-fetching core and canvas.

Error → session-state policy:

| Condition | Session action | Outcome | Site |
|---|---|---|---|
| `session/new` returns `AgentExited` | `invalidate_all` | `AgentExited` | `pool.rs:1494-1505` |
| `session/new` other error | none | `Error(e)` | `pool.rs:1507-1518` |
| `initial_message` idle timeout | `cancel_with_cleanup`, then `invalidate(&source)` | `Timeout(Idle)` | `pool.rs:1640-1681` |
| `initial_message` hard timeout | `invalidate_all` | `Timeout(Hard{recently_active})` | `pool.rs:1683-1700` |
| prompt `AgentExited` | `invalidate_all` | `AgentExited` | `pool.rs:2047-2069` |
| prompt idle timeout | cancel, then `invalidate(&source)` on cancel error | `Timeout(Idle)` | `pool.rs:2070-2152` |
| prompt hard timeout | `invalidate_all` | `Timeout(Hard{..})` | `pool.rs:2155-2180` |
| prompt `AgentError{..}` | **session preserved** — agent caught the problem before mutating state | `Error(e)` | `pool.rs:2182-2188` |
| any other prompt error | `invalidate(&source)` | `Error(e)` | `pool.rs:2186-2188` |

`recently_active` is `silence < RECENT_ACTIVITY_WINDOW` (60 s) at hard-cap firing (`pool.rs:1683`, `pool.rs:2154`), and is the flag the queue uses to decide requeue-vs-dead-letter for a hard-capped batch.

#### Cancellation semantics

Five control signals with distinct batch fates (`pool.rs:263-289`, mapping at `pool.rs:2986-3004`):

| Signal | Cancels turn | Batch fate | Session |
|---|---|---|---|
| `Cancel` | yes | dropped | invalidated |
| `Interrupt` | yes | requeued, `CancelReason::Interrupt` | invalidated |
| `Steer` | yes | requeued, `CancelReason::Steer` | invalidated |
| `Rotate` | yes | dropped | invalidated |
| `SwitchModel(id)` | yes | requeued, `CancelReason::Interrupt` | invalidated after `desired_model` is set |

`requeue_cancelled_batch` returns `None` for `Cancel`/`Rotate` before consulting dedup mode (`pool.rs:2992-2995`); for the others it defers to `requeue_batch_if_queue`, so in `DedupMode::Drop` even a steer loses the batch (`pool.rs:2973-2979`).

Race 1 — the signal arrives after the turn already completed. `has_in_flight_prompt()` is false (`pool.rs:1861`), so no cancel is sent; instead the harness synthesizes `PromptOutcome::Ok(StopReason::EndTurn)` with **no batch requeue** (`pool.rs:1931-1988`). `apply_completed_before_control_signal` (`pool.rs:242-257`) still applies the load-bearing part: `Rotate` and `SwitchModel` invalidate the session; `Cancel`/`Interrupt`/`Steer` leave it intact. The code notes this branch is unreachable during the pre-prompt phase because `biased` polls the prompt arm first and it sets `last_prompt_id` before its first yield (`pool.rs:1935-1943`). The comment "MUST send a PromptResult or the main loop deadlocks" (`pool.rs:1945`) states the invariant: the main loop's `task_map`/slot accounting only clears on a `PromptResult` or a `JoinError`.

Cancel drain deadline: control-signal cancels use `cancel_with_cleanup_grace(session_id, CONTROL_CANCEL_GRACE)` — a fixed 5 s (`pool.rs:788-793`, call at `pool.rs:1863-1866`). Failure is classified once by `classify_control_cancel_failure` (`pool.rs:3029-3056`):

| Error | Outcome | `invalidate_all` |
|---|---|---|
| `AgentExited` | `AgentExited` | yes |
| `IdleTimeout(_)` | `Timeout(Idle)` | no |
| `CancelDrainTimeout(g)` | `CancelDrainTimeout(g)` | no |
| `HardTimeout{..}` | `CancelDrainTimeout(CONTROL_CANCEL_GRACE)` — deliberately **not** `Timeout(Hard)` so it can never claim the configured cap or trigger hard-cap dead-lettering | no |
| anything else | `Error(other)` | no |

Non-cancelling steer (Goose-native): `send_steer` finds the `TaskMeta` whose `channel_id` matches, requiring an installed `steer_tx`, and `try_send`s into a capacity-1 channel (`pool.rs:646-662`). `Full`/`Closed` become `SteerError::Transport`; a missing task becomes `SteerError::PromptCompleted`. Every failure path is documented to fall back to the universal `ControlSignal::Steer` cancel+merge route. Ack semantics are locked (`pool.rs:375-392`): `Success` → drop the withheld event; `Err(_)` → release it and fire the fallback; `PromptCompletedNeutral` → release it and do **not** fire the fallback.

#### What happens to in-flight turns when a child dies

The pool never observes the child directly — death surfaces as `AcpError::AgentExited` from a read, which becomes `PromptOutcome::AgentExited` and returns the (now-useless) `OwnedAgent` through `result_tx`. `handle_prompt_result` then removes every `task_map` entry for that agent index (`lib.rs:3047-3051`), requeues the batch before `mark_complete` so retry backoff survives (`lib.rs:3061-3069`), and routes `AgentExited | Timeout(_)` and `CancelDrainTimeout(_)` to `spawn_respawn_task` (`lib.rs:3239-3283`, `:3292-3323`). `Cancelled` and non-transport `Error` return the agent to the pool; transport-class errors (`Io`, `WriteTimeout`, `Timeout`, `Protocol`) respawn (`lib.rs:3347-3396`).

If the task panics instead of returning, `recover_panicked_agent` (`lib.rs:3402-3495`) reconstructs state purely from `TaskMeta`: requeue `recoverable_batch` (only populated in `DedupMode::Queue`, `lib.rs:2919-2922`), `mark_complete` the channel or clear `heartbeat_in_flight`, emit `agent_panic`, count the panic against the circuit breaker, and respawn in the background. A panic in `DedupMode::Drop` therefore silently loses the message.

Respawn/backoff (all in `lib.rs`, outside this module's files): `SlotCircuit` (`lib.rs:1027-1035`) with `CIRCUIT_BREAKER_THRESHOLD = 3` crashes per `CIRCUIT_BREAKER_WINDOW = 60s`, `CIRCUIT_BREAKER_COOLDOWN = 300s`, respawn delay `RESPAWN_BASE_DELAY = 1s` doubling to `RESPAWN_MAX_DELAY = 30s` (`lib.rs:1008-1016`). Circuit-open slots stay empty until the maintenance loop's refill pass (`lib.rs:1748-1768`). When `live_count() == 0` and no respawn is in flight, the harness exits (`lib.rs:3275-3278`, `:3315-3318`, `:3378-3382`, `:3529-3531`).

#### Health checks

There is no active health probe. The only liveness mechanism is outbound and observational: `run_turn_liveness` (`pool.rs:3162-3204`) emits a `turn_liveness` observer frame every `turn_liveness_interval` while a turn runs, skipping the immediate first tick so the first ping lands one interval after `turn_started` (`pool.rs:3184-3187`). A zero interval parks forever without emitting (`pool.rs:3172-3174`), as does a missing observer handle (`pool.rs:3169-3171`). Failure detection is entirely reactive: idle timeout, hard cap, EOF-on-read, or a task panic.


## Module: buzz-acp — work queue, filtering & usage accounting (`crates/buzz-acp/src`)
### Aspect: Business Rules

#### 1. Admission (`push`, `queue.rs:230-252`)

| Rule | Evidence |
|---|---|
| In `DedupMode::Drop`, an event whose channel is in `in_flight_channels` is discarded and `push` returns `false`; only a `debug!` is logged | `queue.rs:231-241` |
| In `DedupMode::Queue`, every event is accepted regardless of in-flight state | `queue.rs:231` (guard is `matches!(Drop)`) |
| At depth ≥ 500 the **oldest** event is `pop_front`-ed to make room, with a `warn!` | `queue.rs:242-249` |
| Events for other channels always queue, even in `Drop` mode | `queue.rs:232` scopes the check to `event.channel_id`; test `test_drop_mode_queues_other_channels` (`queue.rs:2522`) |
| The 👀 "seen" reaction is only sent when `push` returns `true` | `lib.rs:2209-2217` |

#### 2. In-flight exclusion and expiry

| Rule | Evidence |
|---|---|
| At most one batch per channel is in flight; multiple channels run concurrently | `queue.rs:294-298` filter excludes `in_flight_channels`; test `test_multiple_channels_in_flight_simultaneously` (`queue.rs:2541`) |
| `flush_next` and `has_flushable_work` both auto-expire any channel where `now >= deadline`, logging `ERROR "BUG: in-flight channel expired without mark_complete"` with `lost_events` | `queue.rs:263-287` and `:558-590` (duplicated block) |
| Expiry releases `in_flight_channels` + `in_flight_deadlines` but the dispatched batch is **not** recovered — it is counted as orphaned | `queue.rs:271-281`; comment `queue.rs:275-277` |
| Deadline = `max_turn_duration + 100 s`, so a turn hitting the hard cap returns via `mark_complete` before the backstop fires | `queue.rs:39`, `:197-201`; invariant stated `queue.rs:167-169`; test `default_in_flight_deadline_exceeds_default_max_turn_duration` (`queue.rs:4530`) |
| `extend_in_flight_deadline` is monotonic (`max(current, now+budget)`) and a **no-op** when the channel is not in flight | `queue.rs:211-227`; tests `extend_in_flight_deadline_is_monotonic` (`:4574`), `extend_in_flight_deadline_noop_after_mark_complete` (`:4587`) |
| `compact_expired_state` preserves an extended deadline | test `compact_expired_state_preserves_extended_in_flight_deadline` (`queue.rs:4606`) |

#### 3. Flush selection and batching (`flush_next`, `queue.rs:260-380`)

| Rule | Evidence |
|---|---|
| Candidate = non-empty queue AND not in flight AND (`retry_after` absent OR ≤ now) | `queue.rs:291-300` |
| Winner = channel whose **head** event has the oldest `received_at` (cross-channel FIFO fairness) | `queue.rs:299`; tests `test_fifo_fairness_picks_oldest_channel` (`:1810`), `test_flush_next_picks_oldest_non_throttled` (`:2615`) |
| Ties are broken by `HashMap` iteration order — nondeterministic | `queue.rs:293-300` iterates `self.queues` with no secondary key |
| Drains `min(50, len)` events; the remainder stays queued | `queue.rs:336-345` (`MAX_BATCH_EVENTS`, `queue.rs:27`) |
| The drained batch is re-sorted ascending by `event.created_at` (stable), because relay replay arrives newest-first | `queue.rs:346-350`; test `test_flush_orders_replayed_events_chronologically` (`:1782`) |
| Empty queue entries are removed from the map after drain | `queue.rs:352-355` |
| Any stored `cancelled_batches[channel]` is merged into `FlushBatch::cancelled_events` on the same flush | `queue.rs:362-366`; test `test_requeue_as_cancelled_merges_in_flush_next` (`:3674`) |
| `cancel_reasons[channel]` is **always removed** on flush; the value is only propagated when `cancelled_events` is non-empty, otherwise `cancel_reason` is `None` | `queue.rs:367-372` |
| **Fallback:** if no channel has ready queued events but some channel has `cancelled_batches`, that channel is flushed with the cancelled events moved into the regular `events` slot and `cancelled_events` empty (plain re-dispatch, no merge framing) | `queue.rs:305-333`; test `test_requeue_as_cancelled_no_new_events_fallback` (`:3760`) |
| The fallback path checks only `in_flight_channels` — it **ignores `retry_after`**, so a throttled channel's cancelled batch can flush during its backoff window | `queue.rs:308-312` (no `retry_after` predicate); same omission in `has_flushable_work` (`queue.rs:586-590`) |
| The fallback picks with `keys().find(...)` — nondeterministic and not FIFO-ordered across cancelled channels | `queue.rs:308-312` |

#### 4. Completion vs requeue ordering (the load-bearing invariant)

`mark_complete` decides whether to keep the retry counter by **reading `retry_after`** (`queue.rs:397-409`):

- `Some(deadline)` with `deadline > now` → channel was just requeued → `retry_counts` is left intact so backoff continues.
- `Some(_)` expired → clear both `retry_after` and `retry_counts`.
- `None` → clear `retry_counts`.

Only `requeue` writes `retry_after` (`queue.rs:496`). Therefore **`requeue()` must be called before `mark_complete()`** or every retry restarts at attempt 1, silently defeating both exponential backoff and dead-lettering. `handle_prompt_result` enforces this ordering explicitly, with the rationale in a comment at `lib.rs:3061-3065`; the batch-fate block runs at `lib.rs:3067-3163` and `mark_complete` only at `lib.rs:3171`. The same ordering holds at the pool-exhausted site (`requeue_preserve_timestamps` `lib.rs:2912` then `mark_complete` `lib.rs:2913`) and in panic recovery (`requeue` `lib.rs:3428` then `mark_complete` `lib.rs:3440`). Neither `requeue` nor `requeue_preserve_timestamps` clears `in_flight_channels` themselves — documented at `queue.rs:426-427` and `:506-507`. Nothing in the type system or a debug assertion enforces the order; it is carried by comments and the tests `test_mark_complete_enables_flush` (`queue.rs:1736`) / `test_retry_throttle_blocks_requeue_channel` (`queue.rs:2854`).

#### 5. Retry budget, backoff, dead-lettering (`requeue`, `queue.rs:429-498`)

| Rule | Evidence |
|---|---|
| Attempt counter is per **channel**, not per batch | `queue.rs:431-435` |
| `attempt > MAX_RETRIES` (10) → dead-letter: `ERROR` log, `retry_counts` **and** `retry_after` cleared, batch returned to caller | `queue.rs:437-451`; test `test_requeue_dead_letters_after_max_retries` (`:2824-2846`) |
| `retry_after` is cleared on dead-letter so fresh traffic is not throttled by the poison batch's backoff | `queue.rs:447-449` |
| Backoff = `5 s × 2^min(attempt-1, 6)` capped at 300 s → 5, 10, 20, 40, 80, 160, then 300 for attempts 7-10 (≈25 min total budget) | `queue.rs:454-455` (`BASE_RETRY_DELAY_SECS` `:33`, `MAX_RETRY_DELAY_SECS` `:36`) |
| Jitter is ±20 % derived from `SystemTime::now().subsec_nanos()` — not a CSPRNG | `queue.rs:457-464` |
| Events are pushed to the **front** in reverse order, preserving original order and original `received_at` (fairness position) | `queue.rs:476-484`; doc `queue.rs:416-421` |
| Requeue overflow past 500 trims from the **back** (newest) | `queue.rs:486-495`; tests `test_requeue_preserve_timestamps_enforces_cap` (`:2715`), `test_requeue_preserve_timestamps_overflow_keeps_requeued_events` (`:2751`) |
| `requeue_preserve_timestamps` is the no-agent-available path: same front-insert, **no** `retry_after`, **no** attempt increment, so it can loop indefinitely without ever dead-lettering | `queue.rs:508-529`; test `test_requeue_preserve_timestamps_no_retry_after` (`:2699`) |

#### 6. Which failures retry, which dead-letter (decided in `lib.rs`, executed by the queue)

| Outcome | Queue action | Evidence |
|---|---|---|
| Channel in `removed_channels` | batch dropped, no queue call | `lib.rs:3157-3163` |
| `Cancelled` or `CancelDrainTimeout` | `requeue_as_cancelled(batch, reason)` — no retry accounting at all | `lib.rs:3072-3090` |
| `Timeout(Hard { recently_active: false })` | dead-lettered **immediately**, no `requeue` call, user notice posted | `lib.rs:3091-3109` |
| `Timeout(Hard { recently_active: true })` | `requeue(batch)`; only dead-letters when `requeue` returns `Some` (retry budget exhausted per §5) | `lib.rs:3110-3129` |
| `Error(e)` where `is_auth_error(e)` | dead-lettered immediately, no `requeue`, re-auth notice | `lib.rs:3130-3144` |
| everything else (`Idle` timeout, `AgentExited`, other `Error`) | `requeue(batch)` with generic notice on dead-letter | `lib.rs:3145-3156` |

"Recently active" means agent activity within `RECENT_ACTIVITY_WINDOW = 60 s` before the hard cap (`pool.rs:44-45`).

#### 7. Requeue-vs-drop per cancel reason

`requeue_as_cancelled` never drops (`queue.rs:542-548`); the drop decision happens upstream:

| `ControlSignal` | `CancelReason` | Batch fate | Evidence |
|---|---|---|---|
| `Steer` | `Steer` | requeued in `Queue` mode | `pool.rs:2988` |
| `Interrupt`, `SwitchModel(_)` | `Interrupt` | requeued in `Queue` mode | `pool.rs:2989` |
| `Cancel`, `Rotate` | — | **dropped outright** | `pool.rs:2990-2991` |

`requeue_batch_if_queue` returns `None` for `DedupMode::Drop` on **every** failure path including a steer (`pool.rs:2971-2977`), so in `Drop` mode a cancel loses the batch. The same asymmetry applies to panic recovery: `recoverable_batch` is populated only under `DedupMode::Queue` (`lib.rs:2196-2199`), so a task panic in `Drop` mode drops the batch. `DedupMode` defaults to `queue` on the CLI (`config.rs:344`), which is what keeps these paths lossless in practice.

Double-cancel accumulates: `requeue_as_cancelled` `extend`s both the prior `cancelled_events` and the new `events` into the same vec, and the **latest reason wins** (`queue.rs:543-547`). Tests `test_double_cancel_latest_reason_wins` (`:3740`), `test_double_cancel_preserves_all_events` (`:3825`).

#### 8. Goose-native steer withhold (side table)

| Rule | Evidence |
|---|---|
| `mark_native_steer_pending` moves the event **out of `queues`** into `withheld_native_steer`, making it invisible to `flush_next` / `has_flushable_work` / `drain_channel` without touching the drain path | `queue.rs:673-691`; rationale `queue.rs:157-165`, `:649-660` |
| Returns `false` (race-safe no-op) if the event id is absent — caller logs a possible-duplicate warning | `queue.rs:674-680`; caller `lib.rs:2847-2865` |
| Must be called synchronously after `send_steer` returns `Ok` and **before** the ack watcher is spawned, closing the `mark_complete` → `flush_next` race | doc `queue.rs:667-671`; enforced by ordering at `lib.rs:2841-2872`; also stated `pool.rs:636-640` |
| `SteerAck::Success` → `remove_event` (drop from both stores; the agent already got it) | `queue.rs:738-751`; caller `lib.rs:2512` |
| `SteerAck::Err` / `PromptCompletedNeutral` → `release_native_steer` pushes it back to the queue **front** with original `received_at` | `queue.rs:703-730`; caller `lib.rs:2515` |
| `SteerAck::Success` also extends the in-flight deadline (fresh turn budget) | `lib.rs:2509` |
| In-flight deadline expiry bulk-recovers withheld events in reverse order so FIFO is preserved at the queue front — recover, not log-and-drop, because they were never delivered | `queue.rs:766-789`; tests `test_native_steer_expiry_recovers_withheld` (`:4355`), `test_native_steer_bulk_release_preserves_fifo` (`:4402`) |
| A channel whose only events are withheld is **not** flushable | test `test_native_steer_withhold_only_channel_not_flushable` (`:4278`) |
| Earlier non-withheld events in the same channel still flush during the ack window | test `test_native_steer_earlier_events_flush_during_ack_window` (`:4306`) |

#### 9. Channel teardown and compaction

| Rule | Evidence |
|---|---|
| `drain_channel` removes `queues`, `retry_after`, `retry_counts`, `cancelled_batches`, `cancel_reasons`, `withheld_native_steer` and returns dropped hex event ids | `queue.rs:625-642` |
| It deliberately **preserves** `in_flight_channels` **and** `in_flight_deadlines` — removing deadlines alone would disable auto-expiry and wedge the channel forever | `queue.rs:636-641` comment; test `test_drain_channel_does_not_affect_in_flight` (`:3578`) |
| `compact_expired_state` drops past-deadline `retry_after` entries | `queue.rs:809` |
| `retry_counts` is retained only while the channel has an active throttle **or** queued events **or** an in-flight prompt; the in-flight clause prevents resetting backoff mid-retry | `queue.rs:813-817`; tests `test_compact_cleans_orphaned_retry_counts` (`:3596`), `test_compact_preserves_retry_counts_when_in_flight` (`:3630`), `test_compact_preserves_retry_counts_with_queued_events` (`:3658`) |
| `cancelled_batches`, `cancel_reasons`, `withheld_native_steer`, `in_flight_batch_sizes` are **not** compacted | `queue.rs:807-818` touches only two maps |

#### 10. Subscription rule matching (`filter::match_event`, `filter.rs:368-459`)

Rules are evaluated in order; **first match wins** (`filter.rs:376`, `:451-456`). Per rule, in this exact order:

1. **Channel scope** — `ChannelScope::All(s)` matches only when `s == "all"` literally (`filter.rs:68-70`); `List` compares `id == &channel_id.to_string()` (lowercase-hyphenated UUID string compare, `filter.rs:70`). Non-match → `continue` (`filter.rs:378-380`). Tests `test_channel_scope_all_invalid_string` (`filter.rs:692`), `test_channel_scope_list` (`:702`).
2. **Kinds** — empty vec = wildcard; otherwise `rule.kinds.contains(&(event.kind.as_u16() as u32))` (`filter.rs:383-385`).
3. **`require_mention`** — a tag whose slice is `["p", agent_pubkey_hex, …]` must exist; matched via `tag.as_slice()` to avoid depending on the tag-kind `Display` impl (`filter.rs:390-398`). Comparison is exact-case hex.
4. **`filter` expression** — evalexpr boolean, using the pre-compiled `Arc<Node>` when present (`filter.rs:402-449`).

Fail-closed semantics (`filter.rs:357-366`): **any** filter error or timeout returns `None` for the whole `match_event` call, never falling through to later rules, because falling through would silently widen the subscription. Test `test_filter_error_fails_closed_no_fallthrough` (`filter.rs:733`).

Timeout circuit breaker: `consecutive_timeouts` increments on `FilterError::Timeout` and resets to 0 on any `Ok` (`filter.rs:417-448`). At `MAX_CONSECUTIVE_TIMEOUTS = 5` (`filter.rs:341`) the rule is logged at `ERROR` and `match_event` returns `None` immediately (`filter.rs:405-415`). Because the counter is a shared `Arc<AtomicU32>` (`filter.rs:131-145`), the breaker is global across rule clones. Test `test_consecutive_timeouts_disables_rule` (`filter.rs:766`).

Expression evaluation bounds (`evaluate_filter`, `filter.rs:197-262`): length cap 4096 bytes checked **before** dispatch (`filter.rs:203-208`); a `100 ms` `EVAL_TIMEOUT` applied both to semaphore acquisition and to the blocking eval (`filter.rs:220-232`, `:234-249`); at most 4 concurrent blocking evals, with the permit moved *into* the closure so it is held until the thread actually finishes rather than until the caller times out (`filter.rs:173-183`, `:227`, `:239-241`). Variables exposed: `content`, `author`, `kind`, `channel_id`, `timestamp` plus four hand-registered helpers `str_contains` / `str_starts_with` / `str_ends_with` / `str_len` (`filter.rs:268-323`).

Rule construction rules live in `lib.rs:1439-1474`:

| `SubscribeMode` | kinds | `require_mention` | `prompt_tag` |
|---|---|---|---|
| `Mentions` | `kinds_override` else `[9, 46010, 40007]` (`KIND_STREAM_MESSAGE`/`KIND_WORKFLOW_APPROVAL_REQUESTED`/`KIND_STREAM_REMINDER`, `buzz-core/src/kind.rs:343`, `:442`, `:355`) | `!no_mention_filter` | `"@mention"` |
| `All` | `kinds_override.unwrap_or_default()` — **empty vec when `--kinds` is unset** | `false` | `"all"` |
| `Config` | from `config::load_rules` | per rule | rule `prompt_tag` else `name` |

An empty kinds vec is a wildcard *inside* `match_event` (`filter.rs:383`), but the same `kinds_override` feeds `resolve_channel_filters` (`lib.rs:1476`, `config.rs:1180`) where `kinds: None` produces a relay REQ with no `kinds` — which trips the relay p-gate and returns 403 (`AGENTS.md § Common Gotchas #2`). So `--subscribe all` without `--kinds` matches everything locally and receives nothing.

`load_rules` validation (`config.rs:1060-1129`): ≤100 rules, non-empty unique names, filter ≤4096 bytes, expression compiled eagerly (typos fail startup, not runtime), `ChannelScope::All` must be exactly `"all"`, `consecutive_timeouts` reset to a fresh `Arc`.

#### 11. Prompt framing (`format_prompt`, `queue.rs:1406-1564`)

| Rule | Evidence |
|---|---|
| Empty batch → `ERROR` log and empty `Vec` (no panic) | `queue.rs:1411-1417` |
| Scope is derived from the **last** event in the batch, preventing mixed batches from being mislabeled `thread` | `queue.rs:1407-1417` comment + `batch.events.last()` |
| Section order: `[Base]`, `[System]`, `[Team Instructions]`, `[Agent Memory — core]`, `[Channel Canvas]`, `[Context]`, `[Thread/Conversation Context]`, cancelled section, event section, closing note | `queue.rs:1430-1562`; test `test_format_prompt_ordering_with_full_context` (`:2446`) |
| `has_system_prompt_support` (protocol ≥ 2) suppresses `[Base]`/`[System]`/`[Team Instructions]`/core/canvas from the user message — they ride the system role in `session/new` | `queue.rs:1432-1462`; tests `test_format_prompt_modern_agent_suppresses_base_and_system` (`:2407`), `test_format_prompt_canvas_omitted_for_modern_agent` (`:4481`) |
| Blank/whitespace-only `team_instructions` are skipped | `queue.rs:1444-1450` |
| Each section is returned as its own `String` so the observer size trimmer can elide one section's body while keeping every `[Header]` line | doc `queue.rs:1394-1400`; trimmer `lib.rs:659` |
| Single event, no cancel → `[Buzz event: {prompt_tag}]`; multiple → `[Buzz events — N events]` with `--- Event i (tag) ---` separators | `queue.rs:1528-1552` |
| Merged (cancel) framing selected by `MergeFraming::for_reason`; `None` falls back to the **gentler `Steer`** wording | `queue.rs:1584-1609`; tests `test_format_prompt_steer_framing` (`:1916`), `test_format_prompt_interrupt_framing` (`:1940`), `test_format_prompt_no_reason_defaults_to_steer_framing` (`:1960`) |
| The `Steer` `prior_header` is deliberately `[What you were working on]` and not a transcript claim, because `session/cancel` returns nothing | comment `queue.rs:1589-1592` |
| Native-steer transport reuses the same strings via `native_steer_framing()` (returns `new_header_single` + `closing_note` only) and the same `format_event_block`, so the two transports cannot drift | `queue.rs:1612-1626`; call site `lib.rs:2812-2831` |

Reply-anchor rules (`queue.rs:1465-1497`, helpers `:1182-1231`):

- DM → anchor to the triggering event id, but only if the event has thread tags (`queue.rs:1475-1479`).
- Non-DM human-facing turn in a thread → anchor to the thread **root** (keeps replies flat at layer 1) (`queue.rs:1216-1222`).
- Non-DM human-facing top-level turn → anchor to the triggering event, which becomes the new root, with explicit "do NOT reply into any other (older) thread" wording (`queue.rs:1164-1180`).
- Agent↔agent turn → **no** anchor; deep nesting is intentional (`queue.rs:1213-1215`).
- "Human-facing" = sender is not an agent, OR some non-agent pubkey is `p`-tagged. Identity comes from `PromptProfile::is_agent` (NIP-OA `auth` tag), not raw `p`-tag presence, and an **unclassified** pubkey is treated as human (`queue.rs:1188-1195`, `:1205-1206`). Documented as a UX routing heuristic, explicitly "not a security boundary" (`queue.rs:1005-1007`).
- Tests: `test_anchor_human_in_thread_uses_root` (`:3278`) through `test_anchor_agent_only_p_tags_do_not_flatten` (`:3325`); `test_human_thread_reply_anchors_to_root_not_triggering_or_parent` (`:4008`).

Label resolution: `display_name` first, then `nip05_handle`, both passed through `sanitize_prompt_label` which drops control characters and truncates at 64 chars; empty result → fall through to the raw pubkey (`queue.rs:1028-1065`). Tests `test_sanitize_prompt_label_strips_newlines_and_control_chars` (`:3334`), `test_resolve_prompt_label_skips_whitespace_only_display_name` (`:3229`).

#### 12. Thread-tag parsing (`parse_thread_tags`, `queue.rs:849-900`)

- Only NIP-10 **marker** form is honoured: an `e` tag must have `parts.len() >= 4` and `parts[3] ∈ {"root", "reply"}` (`queue.rs:857-866`). Deprecated positional `["e", id, relay]` tags are silently ignored — documented at `queue.rs:843-848`.
- `p` tags with ≥2 parts accumulate into `mentioned_pubkeys` (`queue.rs:867-869`).
- Resolution: `(root, reply)` → both; `(root, None)` → parent = root; `(None, reply)` → root = reply; `(None, None)` → both `None` (`queue.rs:876-884`). Tests `test_parse_thread_tags_*` (`:2901`-`:2961`).

#### 13. Slash-command pass-through

`slash_command_for_batch` returns `Some` only when the batch has **exactly one** event and **no** cancelled carryover — a merged prompt needs the full context format (`queue.rs:961-967`). `extract_slash_command` (`queue.rs:902-959`) strips leading mention tokens in a loop: NIP-27 `nostr:npub1…` / `nostr:nprofile1…` whole tokens (`:911-915`), then `@` + longest matching known display name (case-insensitive, must end at whitespace or EOS) (`:922-935`), then `@` + a single token of `[A-Za-z0-9._-]` (`:936-943`). A bare `@` aborts and returns `None` (`:946`). The remainder qualifies only if it starts with `/` followed by an ASCII alphanumeric, so `"@Eva see /tmp/foo"` never matches (`queue.rs:956-958`). Tests `test_extract_slash_command_basic` (`:4174`), `..._multi_word_display_name` (`:4199`), `..._rejects_non_commands` (`:4214`), `test_slash_command_for_batch_gating` (`:4231`).

#### 14. Usage delta accounting (`usage.rs`)

Lifecycle: `begin_turn` before `session/prompt` → zero or more `record` → `take` at turn completion (`usage.rs:143-171`; call sites `acp.rs:690`, `:1659`, `:784`).

| Rule | Evidence |
|---|---|
| `begin_turn` sets `in_flight_session` and **clears** any leftover `pending` | `usage.rs:182-185` |
| No baseline for the session → `delta_reliable = false`, all `turn_*` fields `None`, `turn_seq = 1` | `usage.rs:222-226`; test `first_turn_no_prior_delta_unreliable` (`:461`) |
| A token counter **decrease** → unreliable, all `turn_*` `None`, but `turn_seq` still advances | `usage.rs:233-235`; test `counter_decrease_delta_unreliable_no_negatives` (`:483`) |
| A **cost** decrease also nulls *all* turn fields, not just cost | `usage.rs:241-254`; test `cost_decrease_sets_delta_unreliable_and_nulls_all_turn_fields` (`:506`) |
| Cost absent on either side → `turn_cost_usd = None` but token delta stays **reliable** | `usage.rs:245`; test `cost_absent_on_one_side_leaves_tokens_reliable` (`:540`) |
| Unknown `session_id` behaves as a first turn (session-restart case) | `usage.rs:222-226`; test `session_restart_new_session_id_treated_as_first_turn` (`:565`) |
| Delta is always measured from the **last published** baseline, so multiple notifications in one turn cannot compound; last notification wins on cumulative values and `turn_seq` stays constant within the turn | `usage.rs:231`, `:259-273`; test `last_update_wins_multiple_updates_same_turn` (`:620`) |
| Notification while **no** session is in flight (e.g. during `session/new`) advances the baseline but produces **no** publishable record | `usage.rs:274-291`; test `setup_notification_before_begin_turn_returns_none` (`:351`) |
| Notification for session X while Y is in flight is **ignored entirely** — advancing X's baseline would undercount X's next delta | `usage.rs:292-295`; test `cross_session_notification_does_not_corrupt_other_sessions_delta` (`:404`) |
| `take` clears `in_flight_session` **before** the `?` early-return, so a no-usage turn still ends the in-flight marker | `usage.rs:305-306`; tests `begin_turn_then_take_without_record_returns_none` (`:814`), `take_returns_none_after_drain` (`:611`) |
| `take` advances `published_seq` and the cumulative baseline to the published record only | `usage.rs:307-318` |
| Non-`usage_update` variants deserialize to `Other` and are ignored | `usage.rs:69-71`; test `other_variant_deserializes_without_error` (`:716`) |
| Downstream, `publish_agent_turn_metric` re-checks `delta_reliable` before emitting `turn` counts (defence in depth) and skips publishing entirely when `usage` is `None` or the owner pubkey is unset | `pool.rs:3332-3352` |


## Module: buzz-acp — ACP protocol, config & setup mode (`crates/buzz-acp/src`)
### Aspect: Business Rules

#### Framing

- Transport is newline-delimited JSON over the child's stdio (`acp.rs:1-2`).
- Reads go through `FramedRead::new(stdout, LinesCodec::new_with_max_length(MAX_LINE_SIZE))` (`acp.rs:487`), `MAX_LINE_SIZE = 10_000_000` (10 MB, `acp.rs:21`).
- A line exceeding the cap surfaces as `LinesCodecError::MaxLineLengthExceeded` and is mapped to `AcpError::Protocol("agent stdout line exceeded 10MB limit")` (`acp.rs:1082-1086`, `acp.rs:1373-1381`) — the turn dies; the codec does not attempt resynchronisation.
- Writes are one `to_string` + `\n` + `flush`, wrapped in a fixed 30 s `WRITE_TIMEOUT` (`acp.rs:952-963`); expiry is `AcpError::WriteTimeout`.
- Empty/whitespace-only lines are skipped without resetting anything (`acp.rs:1097-1099`, `acp.rs:1391-1393`).
- Unparseable lines emit an `acp_parse_error` observer event, log a warning, and are skipped — they do **not** fail the turn (`acp.rs:1103-1118`, `acp.rs:1397-1412`).

#### Request/response correlation

- Ids are harness-generated `u64`s from a monotonic `next_id` (`acp.rs:149-150`, incremented at `acp.rs:981`, `acp.rs:690`, `acp.rs:1336`).
- A frame counts as *our* response only when `msg["id"] == json!(expected_id)` **and** `msg.get("method").is_none()` (`acp.rs:1123-1130`, `acp.rs:1444-1447`). The `method` guard exists so an agent-initiated request whose id happens to collide is not mistaken for a response (`acp.rs:1120-1122`).
- Comparison is done on `serde_json::Value`, so a string-typed id from the agent would not match a numeric expectation (`acp.rs:1072-1073`).
- **Non-matching ids are silently skipped** — the loop simply continues. Late responses from a previously timed-out request are therefore consumed and discarded on the next read (documented at `acp.rs:1010-1015`).
- Notifications are distinguished purely by absence of `id` on the outbound side (`acp.rs:1049-1057`) and by the presence of `method` on the inbound side (`acp.rs:1136`).
- Steer-response routing is checked **before** the prompt-response check, both under the same `no method` guard, disambiguating on id (`acp.rs:1439-1447`).

#### Timeouts

| Bound | Value | Site |
|---|---|---|
| Write | 30 s | `acp.rs:952` |
| Non-prompt request (read phase) | `REQUEST_TIMEOUT = 60 s` | `acp.rs:968`, applied `acp.rs:1000-1003` |
| Non-prompt request (write phase) | reuses the same 60 s wrapper | `acp.rs:996-999` — worst case ≈ 90 s, stated at `acp.rs:974-976` |
| Prompt idle | `Config::idle_timeout_secs`, reset on every valid JSON line | `acp.rs:1416-1418` |
| Prompt hard cap | `Config::max_turn_duration_secs`, absolute `Instant` | `acp.rs:681-682` |
| Cancel drain idle | fixed 30 s | `acp.rs:932-936` |
| Cancel-drain floor | 30 s grace when the inherited deadline is expired/near | `acp.rs:840-861` |
| Child reap after SIGKILL | 5 s | `acp.rs:394` |

The idle clock resets on **any** valid JSON frame (`acp.rs:1416-1418`) and is reset a second time on `session/update` variants that report a tool call starting (`acp.rs:1483-1490`) — `handle_session_update` returns `true` only for `tool_call` (`acp.rs:1550`).

Classification is decided *before* sleeping: `idle_fires_first = idle_deadline < hard_deadline` (`acp.rs:1247`), so scheduler jitter cannot mislabel which timeout fired (`acp.rs:1245-1246`).

A pre-select deadline check runs at the top of every loop iteration (`acp.rs:1256-1274`). The comment at `acp.rs:1257-1262` explains why: `tokio::select!` uses `biased` with the reader arm first (`acp.rs:1276-1278`), so an agent producing a continuous output stream would win every poll and `sleep_until` would never be reached, silently defeating the hard cap.

#### Cancellation

`cancel_with_cleanup_until` (`acp.rs:897-946`) is the single drain implementation:

1. **Precondition first** — `last_prompt_id.take()` must be `Some`, else `AcpError::Protocol("cancel_with_cleanup called with no in-flight prompt")` (`acp.rs:903-905`). Validated before any write so no stray frames reach an idle agent (`acp.rs:899-902`).
2. If a permission request is pending and `permission_responded == false`, write `outcome: "cancelled"` (`acp.rs:909-919`).
3. Send the `session/cancel` notification (`acp.rs:922`).
4. Drain with 30 s idle + the inherited hard deadline until the original prompt id responds (`acp.rs:937-945`).

Deadline inheritance rules (`acp.rs:840-861`): a stored deadline more than 30 s out is used as-is; otherwise a 30 s floor applies (with a debug log); a missing deadline logs a **warning** and uses the same 30 s fallback. `session_prompt_blocks_with_idle_timeout` deliberately leaves `last_prompt_id` and `current_hard_deadline` set on `IdleTimeout`/`HardTimeout` so the drain can inherit them, and clears them on every other outcome (`acp.rs:722-737`).

`cancel_with_cleanup_grace` (`acp.rs:881-895`) discards the stored deadline, uses a caller-supplied grace window, and remaps `HardTimeout` → `CancelDrainTimeout` (`acp.rs:891-892`) so a Stop-button drain expiry is distinguishable from a genuine configured hard-cap breach (`acp.rs:874-879`).

#### Permission auto-approval

`handle_permission_request` (`acp.rs:1680-1755`):

- Missing `id` or missing `params.options` → `AcpError::Protocol` (`acp.rs:1682-1685`, `acp.rs:1692-1694`).
- The option is found **by `kind`, never by hardcoded `optionId`** (`acp.rs:1676`, `acp.rs:1703-1705`). Preference order: `allow_once`, then `reject_once` (`acp.rs:1720-1722`).
- Neither present → `AcpError::Protocol("no suitable permission option found …")` (`acp.rs:1728-1731`).
- On the `reject_once` fallback, a missing `optionId` degrades to the literal `"reject"` (`acp.rs:1725`) rather than erroring, unlike the `allow_once` path which requires it (`acp.rs:1707-1709`).
- Ordering is write-then-flag (`acp.rs:1749-1751`). The comment at `acp.rs:1735-1748` records why: flag-before-write could leave `permission_responded == true` after a failed write, making the cancel path skip the cancelled outcome and deadlocking the agent forever. The residual double-response window is one memory store wide.

#### Capability probing and latching

- `initialize` requests `protocolVersion: 2` unconditionally — an intentional squat ahead of the upstream ACP RFD (`acp.rs:536-537`, `acp.rs:126`).
- The negotiated version is read by the caller as `init_result["protocolVersion"].as_u64().unwrap_or(1) as u32` (`lib.rs:3776`, `lib.rs:3864`). A missing or non-numeric field therefore silently means **legacy v1**, which changes prompt composition (`session_new_system_prompt` branches on it at `pool.rs:829-833`; base-prompt/heartbeat prefixing at `lib.rs:4197-4210`). `acp.rs` itself neither validates nor stores the version.
- Goose system-prompt support is latched negatively: `pool.rs:849` matches `Err(AcpError::AgentError { code: -32601, .. })` and records `goose_system_prompt_supported = Some(false)`, after which `pool.rs:838` skips the probe for the rest of the process. The code path that makes this possible is the `AgentError` code preservation in `agent_error_from_json` (`acp.rs:115-122`).
- `session/set_model` is the unstable path; `resolve_model_switch_method` prefers the stable `configOptions` route and only falls back to `SetModel` (`acp.rs:1873-1919`). The fresh `session/new` response is treated as authoritative (`acp.rs:1873-1874`).
- `model_in_catalog` (`acp.rs:1922-1951`) mirrors the same match against the two pre-extracted catalog halves for the idle-path pre-cancel guard.

#### Steer (goose-native, non-cancelling)

- At most one steer in flight: the select arm is gated on `pending_steer.is_none()` (`acp.rs:1288`).
- `active_run_id == None` at write time → nothing is written; the request is acked `SteerError::ExpectedRunIdMissing` and the caller falls back to cancel+merge (`acp.rs:1319-1333`).
- `expectedRunId` is read at write time, not snapshotted at dispatch, so it matches goose's current run (`acp.rs:1300-1310`).
- A successful steer **renews** the hard deadline to `now + max_duration` if that is later than the current one (`acp.rs:1465-1474`).
- Every early-return path drains `pending_steer` with `SteerAck::PromptCompletedNeutral` so the oneshot is never leaked: pre-select expiry (`acp.rs:1263-1269`), sleep arm (`acp.rs:1362-1364`), `AgentExited` (`acp.rs:1369-1371`), max-line (`acp.rs:1374-1376`), IO error (`acp.rs:1382-1384`), prompt error (`acp.rs:1449-1452`), prompt success (`acp.rs:1454-1457`).
- `install_steer_rx` **panics** if a receiver is already installed (`acp.rs:801-805`); `clear_steer_rx` is called on every exit path of `run_prompt_task` to keep that invariant (`acp.rs:811-814`, `pool.rs:1243`).

#### `session/new` and stop-reason mapping

- `session_new_full` requires `result.sessionId` to be a string, else `AcpError::Protocol("session/new response missing sessionId")` (`acp.rs:576-579`).
- `systemPrompt` is added to the params only when `Some`; the doc comment relies on JSON-RPC's ignore-unknown-fields behaviour for agents that don't support it (`acp.rs:559-561`, `acp.rs:572-574`).
- `parse_stop_reason` (`acp.rs:1758-1765`): missing `stopReason` → `Protocol("session/prompt response missing stopReason")`; unrecognised value → `Protocol("unknown stopReason: …")`. Parsing is case-insensitive (`acp.rs:62-64`).

#### `CODEX_CONFIG` merge contract

`build_codex_config_env` (`acp.rs:257-345`), gated entirely on `has_generated_codex_config`:

1. `false` → returns `Ok(None)` immediately; any persona `CODEX_CONFIG` falls through to plain operator-wins (`acp.rs:262-265`).
2. Collect all `CODEX_CONFIG` entries from `extra_env` in order (`acp.rs:268-273`); empty → `Ok(None)` rather than a panic (`acp.rs:275-279`).
3. Every entry must parse as a JSON **object**; otherwise `AcpError::Protocol`, with the message distinguishing `persona` (index 0) from `generated` (`acp.rs:284-302`).
4. First entry is the base; the rest are `deep_merge`d in (`acp.rs:305-309`).
5. `parent_codex_config` (the parent process's `CODEX_CONFIG`) is deep-merged last-but-one, so **parent wins on every colliding key at every nesting level** (`acp.rs:311-327`).
6. `sandbox_workspace_write.network_access` is **force-set to `true`** as the final step and always wins (`acp.rs:329-343`). The entry is created if absent (`acp.rs:330-332`); a non-object `sandbox_workspace_write` errors (`acp.rs:335-341`).

`deep_merge` (`acp.rs:208-227`) recurses only when both sides hold objects; scalars, arrays, and type mismatches let overlay win outright (`acp.rs:220-224`).

`codex_network_env` (`config.rs:646-677`) only fires for normalized commands `codex` / `codex-acp` (`config.rs:647-650`) and only when the relay URL parses **and** has a host — otherwise it logs a warning and skips injection rather than widening the sandbox (`config.rs:652-670`). The extracted `host` is logged (`config.rs:672`) but not used in the emitted value, which is the constant `{"sandbox_workspace_write":{"network_access":true}}` (`config.rs:674-677`).

#### Env injection precedence at spawn

`spawn` (`acp.rs:449-461`): for each `(key, value)` in `extra_env`, inject only `if std::env::var(key).is_err()` — **the parent environment wins** (`acp.rs:455-457`). `CODEX_CONFIG` is excluded from this loop when the merge path produced a value and is set unconditionally afterwards (`acp.rs:451-453`, `acp.rs:459-461`).

#### Agent command / args normalization

- `normalize_agent_command_identity` (`config.rs:600-615`): trim, `\` → `/`, strip trailing `/`, take basename, lowercase, strip `.exe`, then map space and `_` to `-`.
- `default_agent_args` (`config.rs:617-624`): `goose` → `["acp"]`; `codex`, `codex-acp`, `claude-agent-acp`, `claude-code-acp`, `claude-code`, `claudecode`, `buzz-agent` → `[]`; anything else → `None` (no normalization).
- `normalize_agent_args` (`config.rs:679-706`): trims and drops empty args; empty result takes the default; and a lone case-insensitive `"acp"` is treated as "no args" for zero-arg providers so legacy desktop launches behave the same as env-based launches (`config.rs:695-700`).

#### Config validation and precedence — `Config::from_args` (`config.rs:740-1009`)

Executed in this order:

| Rule | Behaviour | Line |
|---|---|---|
| Key parse | `Keys::parse(&args.private_key)`, error → `ConfigError::KeyParse` | `config.rs:741` |
| Key scrub | raw string overwritten with `0`s then cleared — best-effort, explicitly not a `zeroize` guarantee | `config.rs:742-748` |
| System prompt | `--system-prompt` wins over `--system-prompt-file`; clap also marks them mutually exclusive | `config.rs:750-756` |
| Heartbeat interval | `>0 && <10` → **error** | `config.rs:758-762` |
| Turn liveness | `>0 && <5` → **error** | `config.rs:764-768` |
| Heartbeat prompt | inline wins over file | `config.rs:770-776` |
| Base prompt file | read only when `!no_base_prompt`; `>1_048_576` bytes → **error** | `config.rs:778-791` |
| Config-mode warnings | `--kinds`, `--channels`, `--no-mention-filter` each log "ignored in config mode" | `config.rs:793-804` |
| Empty agent command | trimmed-empty → **error** | `config.rs:808-812` |
| Channel UUIDs | non-UUID entries **warn and are ignored**, never error | `config.rs:816-824` |
| Heartbeat cap | `>86400` → warn and clamp to `86400` | `config.rs:826-834` |
| Liveness cap | `>86400` → warn and clamp to `86400` | `config.rs:836-845` |
| Idle-timeout ladder | see below | `config.rs:849-874` |
| Max turn duration | `0` → warn + clamp to `60`; `>604_800` → **error** | `config.rs:876-890` |
| Ordering invariant | `idle_timeout_secs >= max_turn_duration_secs` → **error** | `config.rs:894-899` |
| Allowlist | `allowlist` mode with empty list → **error** (`:903-907`); entries validated (`:908`); list supplied in another mode → **warn and discard** (`:909-915`) | `config.rs:901-916` |
| `allowed_respond_to` | each entry must parse as a `RespondTo`, else **error**; non-empty list not containing the active mode → **error** | `config.rs:918-941` |
| Codex env | `codex_network_env` pushes into `persona_env_vars` and sets `has_generated_codex_config` | `config.rs:951-957` |
| Handling/dedup | `validate_multiple_event_handling` | `config.rs:959` |

The idle-timeout ladder (`config.rs:849-874`) resolves `--idle-timeout` > `--turn-timeout` (deprecated) > `DEFAULT_IDLE_TIMEOUT_SECS = 900` (`config.rs:27`):

- both set → `--idle-timeout` wins, deprecation warning logged (`config.rs:851-857`)
- only `--turn-timeout` → used, deprecation warning logged (`config.rs:859-864`)
- neither → `900`
- a resolved value of `0` → warn and clamp to `1` (`config.rs:868-873`)

The strict-less-than rule (`config.rs:894-899`) is justified in the comment above it: with `idle >= max`, the wall-clock cap always fires first and the idle timeout becomes a dead letter. With the shipped defaults (900 vs 7200) the invariant holds.

`validate_multiple_event_handling` (`config.rs:579-597`) rejects `Steer`/`Interrupt`/`OwnerInterrupt` combined with `DedupMode::Drop`, because `Drop` discards events during the cancel drain window and would yield incomplete merged prompts.

`validate_allowlist` (`config.rs:558-572`) trims + lowercases each entry, requires exactly 64 ASCII hex chars, errors otherwise, and dedupes via `HashSet`.

`allowed_respond_to` is validated at startup but is otherwise **display-only** — the stored `Vec<String>` (`config.rs:532`) is read nowhere except `summary()` (`config.rs:1019-1025`, emitted at `config.rs:1049`). No runtime enforcement re-checks it.

#### Legacy env-var propagation

`propagate_legacy_env_vars` (`config.rs:715-726`) maps `BUZZ_ACP_PRIVATE_KEY` → `BUZZ_PRIVATE_KEY` and `BUZZ_ACP_API_TOKEN` → `BUZZ_API_TOKEN`, and only when the canonical name is unset (`config.rs:720`). It uses `std::env::set_var` and must run before the tokio runtime starts because Rust 2024 requires a single-threaded process for that call (`config.rs:708-714`); it is invoked from the sync wrapper at `lib.rs:1234`. `Config::from_cli` explicitly does **not** call it (`config.rs:730-732`).

`BUZZ_API_TOKEN` is written by this function and then never read anywhere in the crate — the only other occurrence of the name in the repo's buzz-acp tree is the README table row (`crates/buzz-acp/README.md:107`).

#### Subscription filter construction

`resolve_channel_filters` (`config.rs:1134-1234`):

- Target set = `channels_override` ∩ `discovered_channels` when the override is set (non-UUID strings dropped by `filter_map`), else all discovered channels (`config.rs:1143-1153`).
- `Mentions`: `kinds = kinds_override` or the default triple `[KIND_STREAM_MESSAGE, KIND_WORKFLOW_APPROVAL_REQUESTED, KIND_STREAM_REMINDER]`; `require_mention = !no_mention_filter` (`config.rs:1158-1174`).
- `All`: `kinds = config.kinds_override.clone()` — **`None` when `--kinds` is absent** — and `require_mention = false` (`config.rs:1175-1185`).
- `Config`: per channel, union the `kinds` of every applicable rule; **any rule with an empty `kinds` list collapses the merge to `None` (wildcard)**; `require_mention` is the most permissive across matching rules; channels matched by no rule are omitted entirely (`config.rs:1186-1231`).

`resolve_dynamic_channel_filter` (`config.rs:1236-1314`) repeats the same three branches for a single newly-discovered channel, with `channels_override` enforced in Mentions/All and ignored in Config mode per the CLI contract (`config.rs:1246-1258`); `All` again yields `kinds: config.kinds_override.clone()` (`config.rs:1271-1274`).

`rule_applies_to_channel` (`config.rs:1316-1325`) matches `ChannelScope::All("all")` or a `List` containing the channel UUID; anything else is a non-match.

Consequence: `--subscribe all` without `--kinds` produces `ChannelFilter { kinds: None, .. }`, and Config mode does the same for any rule with an empty `kinds` array. Per `AGENTS.md § Common Gotchas #2`, a relay REQ with no `kinds` trips the p-gate and returns 403. Neither branch warns about the wildcard; the only `kinds`-related warning in the whole file is the "ignored in config mode" notice (`config.rs:794-796`).

#### TOML rule loading — `load_rules` (`config.rs:1060-1132`)

| Rule | Behaviour | Line |
|---|---|---|
| File read | missing file → `ConfigError::Io` | `config.rs:1064` |
| Parse | TOML error → `ConfigFile` | `config.rs:1065-1066` |
| Rule count | `>100` → **error** | `config.rs:1068-1073` |
| Zero rules | **warn** only: "agent will receive no events in Config mode" | `config.rs:1075-1080` |
| Empty name | **error** | `config.rs:1084-1088` |
| Duplicate name | **error** | `config.rs:1089-1094` |
| Filter length | `>4096` bytes → **error** | `config.rs:1096-1103` |
| Filter syntax | `evalexpr::build_operator_tree` at load time; failure → **error**; success cached in `compiled_filter` | `config.rs:1104-1116` |
| Channel scope | `ChannelScope::All(s)` where `s != "all"` → **error** (catches `"ALL"`, `"All"`) | `config.rs:1118-1125` |
| Counter reset | `consecutive_timeouts` explicitly reset to 0 after deserialization | `config.rs:1127-1128` |

#### Setup mode

`SetupPayload::from_raw_env_value` (`setup_mode.rs:226-234`): `None` or empty string → `Ok(None)` (normal startup); non-empty invalid JSON → `Err`. When it returns `Some`, `lib.rs:1290-1295` returns straight into `run_setup_listener` and the agent pool never starts.

Per-event nudge gates, in the order the loop applies them (`setup_mode.rs:388-478`):

1. Membership kinds are handled and `continue`d (`setup_mode.rs:405-411`).
2. Kinds other than `KIND_STREAM_MESSAGE` / `KIND_WORKFLOW_APPROVAL_REQUESTED` are dropped (`setup_mode.rs:414-416`).
3. `ignore_self` by pubkey equality (`setup_mode.rs:419-421`).
4. `event_mentions_agent` — an explicit @mention is **required even when `subscribe_mode = all`** (`setup_mode.rs:424-426`).
5. `author_allowed` with the real config's `respond_to` + allowlist, plus DM hardening (`setup_mode.rs:430-442`).
6. `filter::match_event` against the setup rules (`setup_mode.rs:444-451`).
7. `should_nudge_for_event` — author verdict, then filter verdict, then event-id dedup insert (`setup_mode.rs:495-509`).

`should_nudge_for_event` returns early on a blocked author **before** touching the dedup set (`setup_mode.rs:501-504`), so a blocked event leaves no phantom entry — asserted by `test_non_allowlisted_author_returns_no_nudge` (`setup_mode.rs:673-690`).

`build_setup_subscription_rules` (`setup_mode.rs:521-542`) uses a single hardcoded mentions rule for Mentions/All, and in Config mode loads the real rules but falls back to the mentions rule with a warning if `load_rules` fails (`setup_mode.rs:530-539`). Its default kinds are only `[KIND_STREAM_MESSAGE, KIND_WORKFLOW_APPROVAL_REQUESTED]` (`setup_mode.rs:526`) — no `KIND_STREAM_REMINDER`, unlike `resolve_channel_filters` (`config.rs:1161-1165`).

Nudge threading (`publish_setup_nudge`, `setup_mode.rs:595-646`): if the trigger carries a NIP-10 root, both `root_event_id` and `parent_event_id` point at that root (a flat reply); otherwise both point at the triggering event (`setup_mode.rs:608-624`). The asker is p-tagged (`setup_mode.rs:632`).

Nudge copy selection in `nudge_body` (`setup_mode.rs:243-303`): empty requirements → generic "needs configuration" prose (`setup_mode.rs:245-249`); footer is chosen by requirement mix — Git-Bash present, all-external (`CliConfigInvalid`), mixed, or all-Buzz-managed (`setup_mode.rs:271-289`). The structured payload is always appended as a fenced `buzz:config-nudge` block for the desktop card (`setup_mode.rs:296-302`); `serde_json::to_string` there uses `.expect(...)` guarded by a SAFETY comment (`setup_mode.rs:294-296`).


## Module: buzz-agent — core loop, ACP wire types & handoff (`crates/buzz-agent/src`)
### Aspect: Business Rules
#### Turn loop: fixed ordering per round
`RunCtx::run` (`agent.rs:66-257`) executes this order every round, and the order is load-bearing:

1. Round cap: `if cfg.max_rounds > 0 && round >= cfg.max_rounds → StopReason::MaxTurnRequests` (`agent.rs:88-90`). `0` means unlimited (default `0`, `config.rs:801`).
2. Cancel poll: `if *self.cancel.borrow() → Cancelled` (`agent.rs:91-93`).
3. Steer drain (`agent.rs:98`) — queued steers become `User` history items *before* the handoff decision, so a steer is included in the summarized context rather than lost to it.
4. Handoff gate (`agent.rs:99-112`). `Performed` clears `last_request_input_tokens` **and** `last_request_history_bytes` (`agent.rs:104-106`) so a stale over-threshold reading can't immediately re-fire; `Skipped` instead runs `truncate_history` (`agent.rs:111`); `Cancelled` returns.
5. Tool catalog assembly, with `load_skill` appended only when the session discovered skills (`agent.rs:115-119`).
6. `round += 1` (saturating, `agent.rs:120`), then the provider call.

Pre-loop validation: prompt blocks are flattened and joined with `\n` (`agent.rs:618-631`), rejected over `MAX_PROMPT_BYTES` = 1 MiB (`agent.rs:69-73`, `config.rs:638`), and the first prompt of a session is memoized as `original_task` (`agent.rs:78-80`) for later handoff summaries.

#### Cancellation semantics
Cancel always wins: every wait uses `tokio::select! { biased; _ = cancel.changed() => … }` — the provider call (`agent.rs:121-146`), the handoff summarizer (`handoff.rs:45-48`), and the tool `JoinSet` loop (`agent.rs:424-441`). Additional rules:
- Preflight cancel fills every empty result slot with a synthetic `cancelled` error and emits a terminal `failed` for calls that already got `pending` (`agent.rs:300-313`).
- On cancel during execution the semaphore is closed rather than tasks aborted (`agent.rs:430-435`), so each in-flight MCP call can send `notifications/cancelled` itself; late permit acquisition emits `failed` and returns (`agent.rs:378-384`).
- The post-cancel drain is bounded to 5 s, then `set.abort_all()` (`agent.rs:455-478`).
- Any still-unfilled runnable slot becomes a synthetic `cancelled` result plus a `failed` update, so no client is left with a permanently `pending` tool (`agent.rs:483-489`).
- Results are appended to history even on the cancel path (`agent.rs:311`, `agent.rs:341`) — that is what keeps history valid for the next prompt (test `cancel_leaves_history_valid_for_next_prompt`, `tests/regressions.rs:456`).
- A cancelled turn still emits `usage_update` if any tokens were seen (`lib.rs:714`; test `cancelled_turn_with_usage_emits_notification_before_response`, `tests/fake_llm.rs:926`).

#### Timeouts
| Rule | Value | Site |
|---|---|---|
| Per-tool call timeout | `cfg.tool_timeout` (default 660 s, `config.rs:804`) | `agent.rs:509-517` |
| On tool timeout, kill the owning MCP server's process group | — | `agent.rs:531-537` |
| Except when the session was cancelled — then do not kill a healthy server | — | `agent.rs:526-530` |
| Post-cancel drain bound | hardcoded 5 s | `agent.rs:455` |
| Keepalive tick while awaiting the provider | hardcoded 30 s, first tick skipped | `agent.rs:129-131` |
| `_Stop` / `_PostCompact` hook budget | `cfg.hook_timeout` (default 2500 ms, `config.rs:822`) | `agent.rs:228`, `handoff.rs:75` |

The timeout message is deliberately instructive to the model: `"tool: timeout after {n}s. The command took too long. Try a faster approach."` (`agent.rs:538-541`). Similarly, every failed tool result gets `ERROR_REFLECTION_SUFFIX` appended — "[Reflect] Before retrying, identify the cause and change your approach." (`agent.rs:21-22`, applied `agent.rs:361-366`). Both are prompt-engineering rules encoded in control flow, with no test asserting the text.

#### Concurrency and admission rules
- One prompt per session: `acquire_session` rejects with `"prompt already in flight"` when `busy` (`lib.rs:786-788`); surfaced as `-32602` `session/prompt: prompt already in flight` (`lib.rs:653-661`). Tests: `rejects_concurrent_prompts` (`tests/fake_llm.rs:320`), `test_concurrent_prompt_rejected` (`tests/golden_transcripts.rs:409`).
- Session cap checked twice: once before the (slow) MCP spawn without holding the lock (`lib.rs:346-355`) and again after, under the lock (`lib.rs:399-409`). No test covers either rejection — `grep -n 'max sessions reached' tests/` returns zero matches.
- Tool fan-out is bounded by `Semaphore(max(cfg.max_parallel_tools, 1))` (`agent.rs:371-372`); `1` degenerates to sequential execution.
- Per-turn tool-call cap: over `MAX_TOOL_CALLS_PER_TURN` = 64 the list is truncated with a `warn!` and the *truncated* list is what enters history (`agent.rs:242-252`, `config.rs:650`). Test `per_turn_tool_call_cap_enforced` (`tests/regressions.rs:606`).

#### Validation rules
| Rule | Enforcement | Test |
|---|---|---|
| `cwd` non-empty and absolute | `lib.rs:334-343` | `test_session_new_rejects_relative_cwd` (`tests/golden_transcripts.rs:314`) |
| Combined system prompt ≤ 512 KiB | `lib.rs:375-387` | `session_new_rejects_oversized_system_prompt` (`tests/fake_llm.rs:415`) |
| Prompt ≤ 1 MiB | `agent.rs:69-73` | none found (`grep 'MAX_PROMPT_BYTES' tests/` → 0 matches) |
| Inbound frame ≤ `max_line_bytes` | `wire.rs:194-198` | `rejects_oversized_line` (`tests/fake_llm.rs:387`), `test_oversized_line_kills_agent` (`tests/golden_transcripts.rs:471`) |
| Frame must be valid UTF-8 | `wire.rs:207-214` | none found |
| `modelId` non-blank | `lib.rs:509-516` | `session_set_model_empty_model_id_returns_error` (`tests/databricks_oauth.rs:906`) |
| Steer `prompt` non-empty | `lib.rs:559-566` | `steer_rejected_on_empty_prompt` (`tests/fake_llm.rs:1092`) |
| Steer `expectedRunId` non-empty | `lib.rs:567-575` | none found |
| Provider claims `stop=tool_use` with zero calls → hard error | `agent.rs:208-212` | none found |

The `cwd` check is *shape only* — existence, readability and canonicalization are not verified before the path is handed to MCP child spawn (`lib.rs:390`).

#### Precedence rules
| Decision | Winner | Site |
|---|---|---|
| Effective model | session `effective_model` (from `session/set_model`) over `cfg.model` | `lib.rs:671-673` |
| System prompt | client `systemPrompt` when non-blank, else `cfg.system_prompt` | `lib.rs:361-370` |
| Hints | appended after the chosen base with a blank line, when `cfg.hints_enabled` | `lib.rs:356-374` |
| Models catalog | cached successful discovery > fresh discovery > provider-aware fallback (never cached). Since `8eb6e3eb` the `DatabricksV2` fallback is the configured model plus `DATABRICKS_V2_KNOWN_MODELS`, not the known list alone (`catalog.rs:61-76`) | `lib.rs:312-327`, `lib.rs:324` |

`session/new` reports `currentModelId: cfg.model` (`lib.rs:470`) — it does not consult any session override, which is correct only because the session is brand new.

#### Steering rules (`_goose/unstable/session/steer`)
Accepted only when all hold (`lib.rs:554-605`): non-empty prompt, non-empty `expectedRunId`, known session, `active_run_id.is_some()`, `expectedRunId == active_run_id`, and `steer_tx.send()` succeeds. Any failure is `-32602`; the mismatch message names both ids (`lib.rs:590-596`). Delivery is *queue-only* — the message is folded into history at the **next round boundary**, never mid-round, and the turn is not restarted (`agent.rs:94-98`, `agent.rs:265-280`). A steer whose blocks fail to render is dropped with `warn!` rather than aborting the turn (`agent.rs:274-278`); an empty-after-render steer is dropped with `debug!` (`agent.rs:271-273`). Tests: `steer_folds_into_active_turn_without_cancelling` (`tests/fake_llm.rs:596`), `steer_rejected_when_no_active_run` (`tests/fake_llm.rs:683`), `steer_rejected_on_run_id_mismatch` (`tests/fake_llm.rs:705`).

Consequence worth noting: a steer arriving after the final round (while the provider is producing the last response) is queued but never drained, because the loop returns before the next boundary (`agent.rs:207-240`). Nothing reports the dropped steer to the client.

#### `_Stop` hook rules (agent sovereignty)
- Consulted **only** when the mapped stop reason is exactly `EndTurn` (`agent.rs:217-219`) — `max_tokens` and `refusal` are never overridden.
- Budget is per prompt, not per session: `stop_rejections` is a loop-local counter (`agent.rs:86`), checked before calling hooks (`agent.rs:220-222`). `cfg.stop_max_rejections = 0` disables the hook entirely (default 3, `config.rs:823`).
- Objections are injected as synthetic assistant-tool-call + tool-result pairs, never as user or system messages (`agent.rs:660-695`), then the loop `continue`s (`agent.rs:233-236`).
- Fail-open: `call_hooks` drops errors, timeouts and blank output (`mcp.rs:309-313`), so a broken hook can never trap the agent.
Tests: `hook_stop_blocks_premature_end` (`tests/regressions.rs:787`), `hook_stop_budget_exhausted` (`:872`), `hook_stop_consecutive_end_turn_uses_rejection_budget` (`:927`), `hook_stop_budget_resets_per_prompt` (`:979`), `hook_stop_timeout_failopen` (`:1514`), `hook_tools_hidden_from_llm` (`:1035`).

Hook tools are unreachable from the model by two independent rules: they are filtered out of `mcp.tools()` and any direct invocation is answered `unknown tool: {name}` (`agent.rs:330-337`).

#### Handoff (context compaction) rules
`maybe_handoff` (`handoff.rs:31-107`):
1. Gate — `should_handoff()` (`handoff.rs:109-129`). With provider usage known: `projected_input_tokens >= token_threshold(max_context_tokens, max_output_tokens)`. Without usage: `context_pressure_bytes > byte_fallback_threshold(...)`. **The comparison operators differ (`>=` vs `>`)** between the two branches (`handoff.rs:111-113` vs `handoff.rs:115-126`) — an off-by-one asymmetry with no stated rationale.
2. Cap — `handoff_count >= cfg.max_handoffs` (default 10, `config.rs:820`) logs and falls back to truncation (`handoff.rs:34-41`).
3. Summarize with a dedicated system prompt (`handoff.rs:25-29`, "Stay under 8192 tokens") and `HANDOFF_MAX_OUTPUT_TOKENS = 8192` (`config.rs:652`), on the session's effective model (`handoff.rs:53`).
4. Empty or failed summary → `Skipped` (truncate instead), never a hard error (`handoff.rs:54-63`).
5. History is cleared **before** calling `_PostCompact`, so the hook injects into the fresh context (`handoff.rs:66-77`).
6. Fresh context is exactly: one `User` item holding `[Context Handoff]\n{summary}` (plus optionally the labeled `_PostCompact` block), then the current user prompt re-appended verbatim (`handoff.rs:84-95`). The comment at `handoff.rs:78-83` explains the rule: tool results must not be orphaned because OpenAI Chat/Responses require them to follow an assistant tool call.

Threshold math is the best unit-tested logic in the group: `token_threshold` = `min(window*9/10, window - max_output_tokens)` (`handoff.rs:348-353`) with tests at `handoff.rs:388`, `:395`, `:403`; `byte_fallback_threshold` clamps to `max_history_bytes*9/10` (`handoff.rs:359-368`) with test at `handoff.rs:410`; `handoff_prompt_budget_bytes` tests at `handoff.rs:378`, `:383`; `estimate_tokens_from_bytes` at `handoff.rs:422`. `CONSERVATIVE_BYTES_PER_TOKEN = 1` (`handoff.rs:326`) is justified as an unconditional upper bound on tokens — the gate is intentionally biased toward handing off early.

Integration coverage: `token_usage_over_budget_triggers_handoff` (`tests/regressions.rs:1354`), `stale_usage_plus_history_growth_triggers_handoff` (`:1436`), `handoff_summary_prompt_includes_full_history_within_context_budget` (`:1224`), `handoff_summary_prompt_keeps_latest_item_when_one_item_exceeds_budget` (`:1292`), `hook_post_compact_injects_after_handoff` (`:1112`).

#### History truncation rule
`truncate_history` (`agent.rs:711-738`) drops whole leading segments up to (but not including) the next `User` item, so the window always begins at a user turn and never orphans a tool result. Two silent behaviors: if no later `User` item exists it `break`s and **returns over budget** (`agent.rs:723-725`), and it uses `estimated_bytes` (real wire size), not `context_pressure_bytes` — deliberately, per `types.rs:24-28`. Covered indirectly by `history_budget_evicts_old_turns` (`tests/regressions.rs:555`); the over-budget break path has no test.

#### Rules enforced only by comment or convention
- `lib.rs:255-259` — gating `[Base]` prompt injection on `protocol_version < 2` is described as a temporary measure, but that gate lives in the client (`crates/buzz-acp/src/pool.rs:181`), not here. Nothing in this crate enforces or checks it.
- `lib.rs:714-718` — the "usage notification must precede the response" contract is a comment plus statement ordering; no assertion or type prevents a future reorder (integration tests do catch it).
- `agent.rs:693-696` — `unique_nonce`'s stated uniqueness target ("no collision within the lifetime of one history vec") is satisfied by a process-wide `AtomicU64` with no wrap handling.
- `lib.rs:599-601` — "A live run always has a `steer_tx`" is an invariant asserted in prose; the code degrades gracefully instead of checking it.
- `docs/MCP_DRIVEN_HOOKS.md:14-17` states hook output is injected as tool-result messages and JSON-encoded; that holds for `_Stop` (`agent.rs:641-651`) but **not** for `_PostCompact`, which is plain text inside a user message (`handoff.rs:85-92`, `handoff.rs:247-253`).


## Module: buzz-agent — LLM providers & configuration (`crates/buzz-agent/src`)
### Aspect: Business Rules
The rules here fall into five clusters: provider/endpoint selection, the new mesh "Auto" collective policy, retry and auth recovery, thinking-effort normalization, and configuration precedence and validation.

#### Configuration precedence
There is no config file and no CLI flag layer, so precedence is only two-deep.

| Rule | Site |
|---|---|
| `BUZZ_AGENT_PROVIDER` is mandatory — absent or whitespace-only is a startup error, with no inference from which API keys are present | `config.rs:1015-1018` |
| `BUZZ_AGENT_MODEL` beats the provider-specific model var (`ANTHROPIC_MODEL` / `OPENAI_COMPAT_MODEL` / `DATABRICKS_MODEL`) | `resolve_model`, `config.rs:979-985`; call sites `config.rs:766-769`, `config.rs:776-779`, `config.rs:786` |
| Provider name matching is case-insensitive and trimmed; aliases `openai-compat`, `databricks-v2` accepted | `config.rs:996-1010` |
| An unsupported provider value is echoed back with the user's original casing (not the lowercased form) | `config.rs:1010-1012`; tested `config.rs:1269` |
| `BUZZ_AGENT_SYSTEM_PROMPT` and `BUZZ_AGENT_SYSTEM_PROMPT_FILE` are mutually exclusive — both set is a hard error | `config.rs:792-794` |
| When neither is set, the built-in prompt is used | `config.rs:658-659`, selected at `config.rs:797` |
| Every numeric var is parsed via `parse_env`, which returns the default only when the var is *absent*; a present-but-unparseable value is a startup error | `config.rs:1049-1056` |
| `OPENAI_COMPAT_API` is only parsed on the `Provider::OpenAi` arm; Anthropic gets a hard-coded `Auto` and Databricks a hard-coded `Chat` | `config.rs:782` vs `config.rs:772` and `config.rs:789` |
| `BUZZ_AGENT_PREFER_MESH_FOR_AUTO` is parsed as a `u8` and compared `!= 0`, so any non-zero integer enables it and a non-numeric value is a startup error | `config.rs:807` |

The `openai_api` inert values are inconsistent: Anthropic gets `OpenAiApi::Auto` (`config.rs:772`, commented "unused for Anthropic") while both Databricks providers get `OpenAiApi::Chat` (`config.rs:789`). For legacy `Provider::Databricks` the value is *not* inert — it is read at `llm.rs:545-546` and `llm.rs:563`. Setting it to `Chat` therefore permanently disables the chat→responses auto-upgrade for legacy Databricks, because the upgrade guard at `llm.rs:563` requires `cfg.openai_api == OpenAiApi::Auto`. This is a real behavioural rule that exists only as a side effect of an "inert default", with no comment saying so.

`prefer_mesh_for_auto` is unconditional at parse time — `from_env` reads it on every provider arm (`config.rs:807` sits in the shared struct literal, not inside a provider branch) — and is then *ignored* for every provider but `OpenAi` by the guard at `llm.rs:411-415`. So unlike `openai_api`, there is no per-provider default fiction; the narrowing is a request-time rule.

#### Provider selection and key requirements
| Rule | Site |
|---|---|
| `anthropic` requires a non-empty, non-whitespace `ANTHROPIC_API_KEY`; otherwise error | `config.rs:999-1002`, emptiness via `present_nonempty` `config.rs:986-988` |
| `openai`/`openai-compat` requires a non-empty `OPENAI_COMPAT_API_KEY` | `config.rs:1003-1006` |
| `databricks` and `databricks_v2` require **no** key at `resolve_provider` time — `DATABRICKS_TOKEN` is optional and defaults to `""` | `config.rs:1007-1008`, `config.rs:785` |
| An empty `api_key` for either Databricks provider means "use OAuth 2.0 PKCE" | `llm.rs:1535-1553` |
| `DATABRICKS_HOST` and a model are required, but validated later in `from_env`, not in `resolve_provider` | `config.rs:786-788` |
| There is no fallback between providers — a missing key never silently degrades to Databricks | tested `config.rs:1236`, `config.rs:1253` |

#### Endpoint selection (OpenAI family)
`use_responses` is a three-term disjunction at `llm.rs:544-546`:
1. the sticky `auto_upgraded` latch, or
2. `openai_api == Responses` (pinned), or
3. `openai_api == Auto` **and** `is_openai_host(base_url)`.

The rule itself is unchanged by `16d4ec33`; it simply moved from `openai_request` into the extracted `openai_request_for_model` (`llm.rs:535-574`), whose doc comment states the precedence explicitly: "pinned > sticky-upgraded > host default" (`llm.rs:533`). Model resolution now happens *before* endpoint selection (`llm.rs:354` then `llm.rs:544`), so the two decisions are independent: a `mesh` request still goes to `/chat/completions` or `/responses` by exactly the rules above.

`is_openai_host` (`config.rs:1037-1047`) accepts a URL only if it starts with `https://` **or** `http://` (`config.rs:1039-1040`), then takes the host up to the first `/` or `:` and matches `api.openai.com` or any `*.openai.com` suffix (`config.rs:1046`). Lookalike hosts such as `api.openai.com.evil.example` return `false` (tested `config.rs:1285`), and a non-URL string returns `false` (tested `config.rs:1286`). The rule is documented and tested by `is_openai_host_matrix` (`config.rs:1275-1291`).

Note the rule is host-only, **not** scheme-gated: `http://eu.api.openai.com/v1` returns `true` and is asserted to do so (`config.rs:1281`). A plaintext base URL for an OpenAI host is therefore treated as fully trusted for endpoint-selection purposes.

Legacy Databricks ignores `path` entirely: `post_openai` rewrites the URL to `{base}/serving-endpoints/{model}/invocations` for `Provider::Databricks` regardless of whether the caller asked for `/chat/completions` or `/responses` (`llm.rs:608-616`). Combined with the forced `OpenAiApi::Chat` above, this is currently unreachable, but the code shape allows a Responses-shaped body to be POSTed to the Chat invocation URL if the upgrade latch were ever set for a Databricks process.

#### Mesh "Auto" collective (Mixture-of-Agents) routing
New in `16d4ec33`, with live discovery corrected in `8eb6e3eb`. The policy rewrites the request model for one specific combination and never any other.

**Eligibility — all three must hold** (`llm.rs:411-415`): `cfg.provider == Provider::OpenAi`, `cfg.prefer_mesh_for_auto`, and `effective_model == "auto"` (exact string equality against `MESH_AUTO_MODEL_ID`, `llm.rs:29`). Any miss returns `effective_model` unchanged (`llm.rs:416`). Consequences that are asserted, not merely implied:
- an explicit real model is never rewritten and never probes the catalog — `explicit_real_model_bypasses_mesh_auto_policy` (`llm.rs:2155`);
- `prefer_mesh_for_auto == false` with model `auto` neither probes nor selects `mesh` — `generic_openai_auto_does_not_probe_or_select_mesh` (`llm.rs:2121`);
- an explicit `mesh` request is *not* adaptive, so it gets no fallback retry and the mesh error surfaces to the caller — `explicit_models_are_never_rewritten_or_fallback_retried` (`llm.rs:2136`). The `adaptive_mesh` flag that gates every fallback requires *both* `effective_model == "auto"` and `request_model == "mesh"` (`llm.rs:355-356`).

**Catalog probe.** `observe_mesh_virtual_model` (`llm.rs:472-530`) issues `GET {base_url}/models` (`llm.rs:473`) with the same bearer as inference (`llm.rs:474`, applied `llm.rs:487`) and a hard `MESH_AUTO_CATALOG_TIMEOUT` of 2 s (`llm.rs:488`). Every failure mode returns `Unknown` and is logged at `debug`: auth unavailable (`llm.rs:475-483`), transport error (`llm.rs:492-499`), non-2xx (`llm.rs:500-506`), missing `data` array (`llm.rs:509-514`), invalid JSON (`llm.rs:521-528`).

**Collective viability.** `mesh_catalog_supports_collective` (`llm.rs:47-64`) returns `None` when there is no `data` array (`llm.rs:48`) and otherwise `Some(has_virtual_mesh && physical_models.len() >= 2)` (`llm.rs:63`). The counting rules are exact-match, not substring:
- an id equal to `"mesh"` sets `has_virtual_mesh` (`llm.rs:57-58`);
- an id equal to `"auto"` is skipped entirely — the router's own virtual entry is not a peer (`llm.rs:59`);
- every other id is normalised by removing the literal `@main` and inserted into a `BTreeSet` (`llm.rs:60`), so two spellings of one revision collapse to one model. This is the `8eb6e3eb` correction, asserted by `collective_catalog_requires_two_distinct_physical_models` (`llm.rs:1891-1915`), which pins all five cases including `org/model@main:Q4` + `org/model:Q4` → `Some(false)` (`llm.rs:1899-1908`);
- ids are trimmed and empty ids dropped before any of this (`llm.rs:54-56`).

**Hysteresis, TTL, and cooldown** — the state machine in `resolve_openai_model` (`llm.rs:418-468`), evaluated in this order:

| Rule | Site |
|---|---|
| If a cooldown is active (`now < cooldown_until`), return `auto` immediately — no probe, no state change | `llm.rs:420-423` |
| An expired cooldown is cleared before proceeding | `llm.rs:424` |
| If the last probe is younger than `MESH_AUTO_CATALOG_TTL` (5 s, `llm.rs:30`), reuse the cached decision without probing | `llm.rs:426-435` |
| Otherwise probe, and stamp `last_checked` **after** the probe returns | `llm.rs:437-439` |
| `Available` increments `consecutive_available` (saturating) and flips `collective_enabled` only once it reaches `MESH_AUTO_ENABLE_OBSERVATIONS` = 2 (`llm.rs:33`) | `llm.rs:441-446` |
| `Unavailable` resets the counter and disables the collective; **and** starts a 30 s cooldown, but only if the collective had been enabled | `llm.rs:447-451` |
| `Unknown` is a no-op — it preserves the last confirmed state rather than treating a failed probe as evidence of absence | `llm.rs:452-457` |
| The resolved model is `mesh` iff `collective_enabled`, else `auto`, and the choice is logged at `debug` | `llm.rs:458-468` |

Two consequences worth stating as rules in their own right:
- **Two consecutive confirmations are required to opt in, one to opt out.** So a mesh that appears is adopted on the *second* qualifying probe (`mesh_auto_requires_two_stable_catalog_observations`, `llm.rs:1826`), while a mesh that contracts is abandoned on the first (`llm.rs:447-451`). A peer that flaps in and out never accumulates two in a row and the collective stays off — `mesh_auto_does_not_enable_while_second_model_flaps` (`llm.rs:1861`) drives four probes and asserts the posted models are `auto, auto, auto, mesh`, i.e. adoption happens only after the flap settles.
- **The full appear → adopt → contract → reappear → re-adopt cycle is a supported lifecycle, not a restart-required change.** `mesh_auto_tracks_models_appearing_disappearing_and_reappearing` (`llm.rs:1917`) walks eight calls and asserts `auto, auto, auto, mesh, auto, auto, auto, mesh` with exactly eight probes.

**Inference-time fallback.** The catalog is advisory; inference is authoritative. `openai_request` (`llm.rs:344-395`) retries **once** through `auto` in exactly two situations, and enters cooldown in both:

| Trigger | Detection | Site |
|---|---|---|
| A 503 whose body is exactly the mesh's MoA-unavailable message | `is_mesh_moa_unavailable_body` on `/error/message` == `MESH_MOA_UNAVAILABLE_MESSAGE` | classifier `llm.rs:1364-1375`, raised in `post` `llm.rs:1446-1448`, handled `llm.rs:361-372` |
| Any 5xx whose body carries a structured `moa_failure` | `is_mesh_moa_failure_body` on `/error/type` == `"moa_failure"` | classifier `llm.rs:1377-1388`, raised `llm.rs:1449`, same handler |
| A 200 that returned no structured tool calls but whose text *looks like* tool-call markup, when tools were supplied | `looks_like_unstructured_tool_call` | classifier `llm.rs:66-72`, guard `llm.rs:373-378`, handler `llm.rs:379-387` |

Rules the code makes explicit:
- Fallback is **once only** — the retry calls `openai_request_for_model` directly (`llm.rs:371`, `llm.rs:388`) and maps any second failure straight to an `AgentError` (`llm.rs:372`, `llm.rs:389`), so there is no loop.
- Both fallback paths call `cool_down_collective` (`llm.rs:363`, `llm.rs:381`) *before* retrying, which zeroes the counter, disables the collective, and sets a 30 s cooldown (`llm.rs:398-403`). The next call therefore goes straight to `auto` **without re-probing** — `mesh_auto_falls_back_once_and_cools_down_when_mesh_contracts` (`llm.rs:1966`) asserts exactly two `/v1/models` requests across four inference calls, with the message "cooldown request must not re-probe the catalog".
- The mesh-fallback classification is gated on `detect_mesh_fallback`, which `post_openai` sets only when the request model is literally `mesh` (`llm.rs:637`). A non-mesh request can never be reclassified, however its provider phrases its 5xx body.
- Mesh fallback **pre-empts the ordinary retry loop**: the check at `llm.rs:1446-1451` runs before the `attempt + 1 < MAX_RETRIES` branch at `llm.rs:1453`, so a MoA failure is not retried three times against `mesh` first.
- The pseudo-tool-call rule only fires when `tools_supplied` (`llm.rs:375`) and `response.tool_calls.is_empty()` (`llm.rs:376`), so a collective that returns prose containing the phrase is unaffected unless tools were actually offered.
- **Fail-open on an unusable catalog.** A failed probe yields `Unknown`, `collective_enabled` stays `false` on a cold start, and the request goes to `auto` — `mesh_auto_catalog_failure_fails_open_to_auto` (`llm.rs:2104`) drives a 500 from `/models` and asserts the single posted model is `auto`.

**Classifier exactness is a stated rule, and it is tested as such.** Both body classifiers parse the body as JSON and compare a single pointer to an exact string (`llm.rs:1366-1374`, `llm.rs:1379-1387`); a substring match or a differently-shaped body does not qualify. `mesh_unavailable_classifier_requires_exact_openai_error_message` (`llm.rs:2171`) and `mesh_failure_classifier_requires_structured_moa_failure_type` (`llm.rs:2185`) each assert one positive and one negative. Note the asymmetry with the *other* classifier in this file: `is_responses_required_error` (`llm.rs:963-968`) is deliberately a lowercase substring test, because it matches free-form provider prose; the mesh classifiers match a contract.

**Where the coupling is implicit.** `MESH_MOA_UNAVAILABLE_MESSAGE` (`llm.rs:34`) is a verbatim copy of a message emitted by the mesh gateway, including the `≥` character. Nothing links the two; a wording change upstream silently disables the 503 fallback path (the `moa_failure` path is typed and would survive). The tests reuse the same constant (`llm.rs:1971`, `llm.rs:2139`, `llm.rs:2148`, `llm.rs:2173`), so they cannot detect the drift either.

#### Chat → Responses auto-upgrade (one-way, process-wide)
| Rule | Site |
|---|---|
| Triggered only from `AgentError::Llm` (never auth or transport errors) | `llm.rs:658-662` |
| Matcher is a lowercase substring test on the provider error body: `/v1/responses`, `responses api instead`, or `use the responses api` | `llm.rs:963-968` |
| Latch is a one-way `AtomicBool::swap(true)`; warns exactly once | `llm.rs:665-672` |
| Only retried when `openai_api == Auto` | `llm.rs:563` |
| The upgrade now triggers only from `PostError::Agent` — a `PostError::MeshFallback` cannot reach `try_upgrade` | pattern at `llm.rs:562` |
| Latch is per-`Llm`-instance, i.e. per-process, and never reset | `llm.rs:90-94`; no `store(false)` anywhere — grep for `auto_upgraded` in `llm.rs` returned only `llm.rs:94`, `llm.rs:117`, `llm.rs:544`, `llm.rs:665`, `llm.rs:3640` |

The matcher's false-positive risk is bounded by design and tested: `is_responses_required_error_matrix` (`llm.rs:2448-2462`) covers the real Databricks GPT-5.5 message, a forward-compat prose variant, and three negatives including `{"error":"invalid_api_key"}` and the empty string. What is **not** tested is the latch behaviour itself — no test exercises `try_upgrade` (`llm.rs:657`) or asserts that a second call skips Chat Completions.

#### DatabricksV2 route selection
`databricks_v2_route_for_model` (`llm.rs:970-982`) lowercases and does plain `contains` checks:

| Condition | Route | Path |
|---|---|---|
| contains `gpt-5` or `gpt5` | `OpenAiResponses` | `/ai-gateway/openai/v1/responses` (`llm.rs:985`) |
| else contains `claude` | `AnthropicMessages` | `/ai-gateway/anthropic/v1/messages` (`llm.rs:986`) |
| else | `MlflowChatCompletions` | `/ai-gateway/mlflow/v1/chat/completions` (`llm.rs:987`) |

This is still a **looser** model-family classifier than the boundary-safe one in `config.rs`. `config.rs` deliberately built `gpt5_token_matches` (`config.rs:239-254`) and `gpt5_base_matches` (`config.rs:267-311`) to prevent `gpt-5.1` matching `gpt-5.10`, `gpt-5-1` matching `gpt-5-1106`, and `gpt-5-4` matching `gpt-5-4o` — with eight boundary tests (`config.rs:2286-2409`). `llm.rs:974-976` throws all of that away for the routing decision. Consequence: a model named `databricks-gpt-5-10` routes to the OpenAI Responses path (`contains("gpt-5")` is true) while `openai_efforts_for_model("databricks-gpt-5-10")` returns `None` (asserted at `config.rs:2346`) and its effort is passed through unverified. The two classifiers disagree on the same string in the same request, and no test covers that pairing — `databricks_v2_routes_by_model_family` (`llm.rs:2464-2487`) only tests three unambiguous names.

The `contains("claude")` check is also unanchored: a model named `my-claude-killer-llama` would route to the Anthropic Messages path.

**Re-verified after `16d4ec33`: the new mesh code does *not* add a third loose classifier.** Every mesh model decision is exact string equality or set membership — `effective_model != MESH_AUTO_MODEL_ID` (`llm.rs:413`), `id == MESH_VIRTUAL_MODEL_ID` (`llm.rs:57`), `id != MESH_AUTO_MODEL_ID` (`llm.rs:59`), `effective_model == MESH_VIRTUAL_MODEL_ID` (`llm.rs:637`) — so it sides with `config.rs`'s discipline rather than `databricks_v2_route_for_model`'s. `databricks_v2_route_for_model` (`llm.rs:974-976`) remains the file's only naked-`contains` family classifier. The one substring matcher the mesh work *did* add, `looks_like_unstructured_tool_call` (`llm.rs:66-72`), classifies model *output*, not a model *name*, and is anchored to the start of the trimmed text (`starts_with`, `llm.rs:69-71`) rather than using `contains`.

#### Retry and backoff
All retry logic lives in `post` (`llm.rs:1390-1517`), which now returns `PostError` rather than `AgentError`. Every give-up path wraps its result in `PostError::Agent` (`llm.rs:1424`, `llm.rs:1440`, `llm.rs:1463`, `llm.rs:1473`, `llm.rs:1479`, `llm.rs:1486`, `llm.rs:1497`, `llm.rs:1505`, `llm.rs:1514`), so the retry *semantics* are unchanged — only the wrapper is new.

| Parameter | Value | Site |
|---|---|---|
| `MAX_RETRIES` (total attempts, not extra retries) | 3 | `llm.rs:1285` |
| `BASE_BACKOFF_MS` | 500 | `llm.rs:1286` |
| `MAX_BACKOFF_MS` | 8000 | `llm.rs:1287` |
| Backoff formula | `min(500 << attempt, 8000)`, then subtract a uniform jitter in `[0, base/2)` | `llm.rs:1290-1301` |
| Effective delays | attempt 0 → 250-500 ms, attempt 1 → 500-1000 ms | derived from `llm.rs:1290-1293` |
| Jitter source | `getrandom::fill`; on RNG failure the full `base` is used with no jitter | `llm.rs:1295-1300` |

Retryable conditions:
- Transport errors where `is_timeout() || is_connect() || is_request()` (`llm.rs:1311-1313`). The `is_request()` term is the important one — it catches a socket accepted then dropped before any response bytes, and is regression-tested by `post_retries_on_dropped_connection_before_response` (`llm.rs:3122`).
- Any 5xx, plus 429, plus the non-standard 499 (`llm.rs:1444`) — **unless** the mesh-fallback classifier claims the body first (`llm.rs:1446-1451`). 499 handling is tested twice: recovery (`post_retries_499_and_succeeds`, `llm.rs:3187`) and exhaustion (`post_exhausts_retries_on_persistent_499`, `llm.rs:3251`, which asserts exactly `MAX_RETRIES` server-side accepts).

Non-retryable: 401/403 (routed to auth recovery instead, `llm.rs:1439-1442`), 404 (`llm.rs:1472-1477`), any other non-2xx (`llm.rs:1478-1483`), JSON decode failure (`llm.rs:1514-1515`), and — new — a classified mesh failure (`llm.rs:1450-1452`), which returns `PostError::MeshFallback` for the caller to handle instead of retrying in place. A **body-read** error mid-stream is explicitly not retried (`llm.rs:1499-1512`) even though the request was already sent — the comment does not say why, and there is no test.

The retry loop still ends in `unreachable!()` (`llm.rs:1516`) with a message asserting the final iteration always returns. **Re-verified against the new control flow:** the `unreachable!` is still genuinely unreachable for `MAX_RETRIES >= 1`, and the `PostError` refactor did not introduce a new `continue` that could skip the terminal return — the only two `continue`s remain the transport retry (`llm.rs:1422`) and the retryable-status retry (`llm.rs:1461`), both guarded by `attempt + 1 < MAX_RETRIES` (`llm.rs:1414`, `llm.rs:1453`). It remains a production-path panic guarded only by `MAX_RETRIES > 0`, not an `#[allow]`-suppressed dead branch.

`terminal_llm_error` (`llm.rs:1323-1336`) is unchanged: it is still the single exit-formatter for give-up paths, still appends `(cumulative <dur>, N attempt[s])` (`llm.rs:1332-1335`), and still emits a `tracing::warn!` when cumulative elapsed `>= STALL_NOTICE_THRESHOLD` (300 s, `llm.rs:27`) at `llm.rs:1324-1330`. It returns a bare `AgentError`, so all three of its call sites now wrap it in `PostError::Agent` (`llm.rs:1424`, `llm.rs:1463`, `llm.rs:1505`). The threshold is a strict `>=` and both sides of the boundary are mutation-tested: `terminal_llm_error_below_threshold_emits_no_stall_warning` at 299 s (`llm.rs:3424`) and `terminal_llm_error_at_threshold_emits_one_stall_warning` at exactly 300 s (`llm.rs:3437`), using a scoped `tracing` subscriber (`llm.rs:3357-3416`). Pluralization is tested too (`llm.rs:3339`).

Three paths deliberately bypass `terminal_llm_error` and two of them are annotated: 401/403 (`llm.rs:1433-1438`) and 404 (`llm.rs:1468-1471`). The third — `PostError::MeshFallback` (`llm.rs:1450-1452`) — carries no annotation, and it is the one where a cumulative-cost note would arguably be wrong for a different reason: the call is not over, it is about to be retried against a different model.

#### Timeouts
| Timeout | Value | Site |
|---|---|---|
| Connect | 10 s, hard-coded | `llm.rs:110` |
| Read (per-read inactivity) | `cfg.llm_timeout`, default 240 s | `llm.rs:111`, default `config.rs:810` |
| Mesh catalog probe (total, per-request) | 2 s | `llm.rs:488` |
| Total wall-clock for an inference POST | **none** | see below |

**Re-verified: the inference path still has no wall-clock bound.** `Llm::new` (`llm.rs:109-113`) sets only `connect_timeout` and `read_timeout`; there is no `.timeout(...)` on the client builder. A provider that trickles one byte every 239 s can hold a request open indefinitely; the only backstop is `STALL_NOTICE_THRESHOLD`, which merely logs (`llm.rs:1324-1330`). The `README.md:150` wording "per-read inactivity, not wall-clock" is accurate and honest about this.

What *did* change is the evidence. `16d4ec33` introduced the file's **first production `.timeout(...)`** — `MESH_AUTO_CATALOG_TIMEOUT` at `llm.rs:488` — so the old "grep for `.timeout(` in `llm.rs` finds only test-module matches" formulation is no longer true. Current matches: one production site (`llm.rs:488`, the 2 s catalog probe) and four test-module sites (`llm.rs:3171`, `llm.rs:3235`, `llm.rs:3286`, `llm.rs:3637`). The bound is per-request on a `RequestBuilder`, applies only to `GET /models`, and does not propagate to the POST that follows. The tests still run the *auth* suite against a stricter client than production, because `llm_with` (`llm.rs:3634-3648`) keeps building `Llm` by struct literal with `.timeout(5s)` (`llm.rs:3637`); the mesh tests, by contrast, go through `Llm::new` (e.g. `llm.rs:1837`), so the production timeout wiring is now at least constructed.

#### Auth recovery (401/403 refresh-once)
| Rule | Site |
|---|---|
| A bearer is fetched once per `post_openai` call, before the loop | `llm.rs:630` |
| 401 **and** 403 both map to `AgentError::LlmAuth` | `llm.rs:1439-1442` |
| On the first `LlmAuth`, force `refresh_now(&rejected_bearer)` and retry once | `llm.rs:642-649` |
| The match now unwraps one layer deeper — `Err(PostError::Agent(AgentError::LlmAuth(_)))` | `llm.rs:642` |
| The `refreshed` guard is a local, so an earlier turn cannot suppress a later turn's retry | `llm.rs:631`, rationale `llm.rs:622-629` |
| Anthropic never uses this path for inference — it sends `x-api-key` directly from `cfg.api_key` | `llm.rs:330-338`, rationale `llm.rs:99-103` |
| The mesh catalog probe uses the bearer but has **no** refresh-once behaviour — a 401 there is just `Unknown` | `llm.rs:474-483`, `llm.rs:500-506` |

Well covered: `post_openai_refreshes_once_per_call_on_401` (`llm.rs:3650`) asserts one refresh *and* that a second call gets its own; `post_openai_persistent_401_propagates_after_one_retry` (`llm.rs:3686`); `post_openai_persistent_403_propagates_after_one_retry` (`llm.rs:3717`); `post_openai_refreshes_once_on_403` (`llm.rs:3748`). The two persistent-failure tests were updated by `16d4ec33` to match on the new wrapper (`llm.rs:3703`, `llm.rs:3734`).

Treating 403 as refreshable is a deliberate trade documented at `llm.rs:1433-1438` and `llm.rs:622-629`: a pure authorization 403 costs one wasted refresh.

#### Thinking-effort normalization
Startup validation is asymmetric by design (`config.rs:940-957`): only `Provider::Anthropic` rejects `none`/`minimal` at startup (`config.rs:949-956`); OpenAI, Databricks, and DatabricksV2 accept all seven values and defer to request time, because `session/set_model` can change the model after startup. The rationale is a comment block at `config.rs:940-946`. All four provider × 7 effort combinations are tested (`config.rs:1867`, `config.rs:1885`, `config.rs:1893`, `config.rs:1914`, `config.rs:1934`, `config.rs:1954`, `config.rs:1964`, `config.rs:1971`, `config.rs:1978`).

Request-time normalization:

| Route | Normalizer | Site |
|---|---|---|
| Pure Anthropic (`Provider::Anthropic`) | **none** — `cfg.thinking_effort` is passed raw to `anthropic_body` | `llm.rs:131`, `llm.rs:135-145` |
| OpenAI / legacy Databricks | `normalize_effort_for_openai_route` | called `llm.rs:158-159`, defined `config.rs:538-550` |
| DBv2 OpenAI Responses route | `normalize_effort_for_openai_route` | `llm.rs:184-185` |
| DBv2 Anthropic Messages route | `normalize_effort_for_anthropic_route` | `llm.rs:193` |
| DBv2 MLflow Chat route | `normalize_effort_for_openai_route` | `llm.rs:202-203` |

One consequence of the mesh work worth recording as a rule: the OpenAI-family normalizer is now applied to the **resolved** model, not the configured one. The build closure receives `request_model` (`llm.rs:152`) and passes it to `normalize_effort_for_openai_route` (`llm.rs:158-159`), so a request rewritten to `mesh` is effort-normalised against the string `"mesh"` — an unknown family, which means `max` clamps to `xhigh` with a warn (`config.rs:541-548`) and every other level passes through (`config.rs:549`). No test covers that pairing; the mesh tests all use the fixture's `thinking_effort` (`llm.rs:1608`).

The pure-Anthropic asymmetry is real: `Provider::Anthropic` never calls `normalize_effort_for_anthropic_route`, so `none`/`minimal` reaching `anthropic_body` would be handled only by the defensive fallbacks at `config.rs:41-43` (budget 0) and `config.rs:69-70` (string `"low"`). Startup validation makes that unreachable *today*, but the two defensive fallbacks are the only thing standing between a validation-bypass and a wrong `output_config.effort`. Both fallbacks are tested (`config.rs:1368-1370`, `config.rs:1391-1393`), and both carry comments saying they exist only because startup rejects those values.

`normalize_effort_for_openai_route` (`config.rs:538-550`):
- known model family → `resolve_openai_effort` nearest-supported substitution;
- unknown family + `Max` → clamp to `XHigh` with a warn (`config.rs:541-548`);
- unknown family + anything else → pass through unchanged (`config.rs:549`).

`resolve_openai_effort` (`config.rs:466-520`) implements two rules:
1. `none ↔ minimal` are each other's first fallback — explicit `peer` branch at `config.rs:479-483`, applied at `config.rs:494-498`.
2. Otherwise, nearest by ordinal distance with upward ties winning — `config.rs:486-491`.

The doc comment at `config.rs:459` states as a rule that "`xhigh` falls back to `high` when not supported (no model skips from `high` to `xhigh`)". **This rule is not implemented explicitly** — it is an emergent property of the distance sort at `config.rs:486-491` plus the shape of the tables at `config.rs:334-359`. If a future table ever supported `max` but not `xhigh`, `xhigh` would resolve upward to `max`, contradicting the stated rule while every existing test still passed. It is tested only for `gpt-5.1` (`config.rs:2488`). Unchanged by these commits; anchors confirmed.

`normalize_effort_for_anthropic_route` (`config.rs:563-577`) collapses `none`/`minimal` to `None` (omit thinking fields entirely) with a warn, and passes everything else through. Its warning text is honest that omission is not the same as disabling thinking (`config.rs:571-573`: "default-on/always-on adaptive models may still think"). Tested at `config.rs:2026`, `config.rs:2035`, `config.rs:2044`.

#### Anthropic thinking-shape rules
`anthropic_thinking_config` (`config.rs:124-178`) picks between three shapes after stripping catalog prefixes (`config.rs:134`, helper `config.rs:89-97`):

| Bucket | Predicate | Emitted fields |
|---|---|---|
| Manual budget | `is_manual_budget_model` — `claude-3*` prefix **or** exactly `claude-opus-4-5` | `thinking: {type:"enabled", budget_tokens}`, no `output_config` (`config.rs:136-160`) |
| Adaptive | `is_adaptive_thinking_model` — explicit list of nine prefixes | `thinking: {type:"adaptive"}` + `output_config: {effort}` (`config.rs:161-170`) |
| Unknown | neither | both omitted (`config.rs:171-176`) |

Manual-budget clamp rule (`config.rs:141-158`): `budget = min(level_budget, max_output_tokens - 1024)`; if the result is `< 1024`, thinking is **omitted entirely** with a `warn!` rather than sending an answer-starving budget. The 1024 reserve is a local `const MIN_ANSWER_TOKENS` (`config.rs:145`). Boundary behaviour is tested on both sides: omit at `max_output_tokens = 2047` (`config.rs:1423`) and at 1025 (`config.rs:1521`); emit at exactly 2048 with `budget_tokens == 1024` (`config.rs:1435`, mirrored at `llm.rs:2665`).

The bucket boundaries are asserted as *doc-verified* against Anthropic's published tables with dated comments (`config.rs:100-122`, `config.rs:578-620`), and the "omit rather than guess" rule for unrecognised `claude-*` names is explicitly tested for five names including a hypothetical `claude-opus-4-9` (`config.rs:1542-1565`).

`clamp_adaptive_effort` (`config.rs:205-236`) enforces one rule: `xhigh` → `high` for adaptive models that do not support `xhigh`; everything else including `max` passes through. The xhigh-support set is `anthropic_model_supports_xhigh` (`config.rs:184-190`), shared with `anthropic_efforts_for_model` (`config.rs:443`) — genuinely a single source of truth for that one predicate, unlike the manual/adaptive split which is duplicated.

Note the clamp is asymmetric with the capability tables: `anthropic_efforts_for_model` returns `ADAPTIVE_NO_XHIGH = [low, medium, high, max]` for non-xhigh adaptive models (`config.rs:428-433`, `config.rs:446`), so `max` is advertised as valid — and `clamp_adaptive_effort` agrees by passing `max` through (`config.rs:212-214`). Consistent, and tested (`config.rs:1704`, `config.rs:2141`).

#### Tool-call parsing rules
| Rule | Site |
|---|---|
| `arguments` must parse as JSON; a parse failure is a hard `AgentError::Llm`, not a silent `{}` | Responses `llm.rs:1025-1030`; Chat `llm.rs:1226-1228` |
| A missing `arguments` string defaults to the literal `"{}"` before parsing | `llm.rs:1026`, `llm.rs:1226` |
| A tool call with empty `id` or empty `name` is rejected | `llm.rs:1249-1251` |
| `arguments` must be a JSON object; `null` becomes `{}`; any other type is rejected | `llm.rs:1252-1260` |
| Unknown Responses `output[]` item types are ignored for forward compatibility | `llm.rs:1059-1060` |
| Responses `output_text` parts accept the alias `text`; `summary_text` accepts `text` | `llm.rs:1011-1016`, `llm.rs:1046-1050` |
| An empty Anthropic assistant turn (no text, no tool calls) is skipped rather than sent as an empty block | `llm.rs:708-714` |
| An empty Responses assistant text is skipped but its tool calls still emit | `llm.rs:890-903`; tested `llm.rs:2346` |
| Tool-call-shaped *text* from a mesh collective is never parsed into a `ToolCall` — it triggers one retry through `auto` and the retried response is what the caller sees | `llm.rs:66-72`, `llm.rs:373-387` |

Malformed-arguments rejection is tested for Responses (`llm.rs:2489`) but **not** for Chat Completions — grep for `tool_call.arguments not valid JSON` in the `llm.rs` test module returned zero matches, so the `llm.rs:1228` error path is untested. Likewise `make_tool_call`'s three rejection branches (`llm.rs:1249`, `llm.rs:1255`) have no direct test.

#### Truncation and context-window handling
Neither file truncates history or counts tokens. `max_history_bytes` (`config.rs:700`) and `max_context_tokens` (`config.rs:712`) are only *defined* and *validated* here; the enforcement lives in `handoff.rs` and `agent.rs` (`handoff.rs:162` references `MAX_PROMPT_BYTES`; `agent.rs:68` enforces it). `llm.rs` sends whatever history it is handed — grep for `truncat`, `elide`, and `max_history_bytes` in `llm.rs` returned zero matches outside the test `Config` literal at `llm.rs:1593`.

Response-size caps are enforced, however, and the `PostError` refactor preserved all three: a `Content-Length` precheck against `MAX_LLM_RESPONSE_BYTES` (`llm.rs:1484-1491`) plus a streaming-buffer cap (`llm.rs:1492-1513`), and a 4 KiB cap on error bodies (`read_error_body`, `llm.rs:1268-1284`) — which is now also the *only* thing bounding how much of a hostile body the two mesh classifiers parse, since both are handed the already-truncated string (`llm.rs:1445`, `llm.rs:1447-1449`). No test covers either response-size cap — grep for `MAX_LLM_RESPONSE_BYTES` and `response too large` in the `llm.rs` test module returned zero matches.

#### Rules enforced only by comment or convention
1. Responses replay ordering (`function_call` before `function_call_output`) — comment at `llm.rs:867-872`, no type or runtime check; relies on `HistoryItem` insertion order.
2. `xhigh → high` fallback preference — stated at `config.rs:459`, implemented only as an emergent property of the distance sort (`config.rs:486-491`).
3. `ThinkingEffort` discriminants being contiguous 0..=6 — required by the `as i32` distance metric at `config.rs:495`, guaranteed only by declaration order (`config.rs:19-27`).
4. `MAX_RETRIES >= 1` — required to make `llm.rs:1516`'s `unreachable!()` actually unreachable; asserted only in that panic message.
5. `for_discovery` being exempt from `validate` — stated at `config.rs:839-843`, enforced by nothing.
6. `anthropic_efforts_for_model` being "the single production source of truth" for Anthropic family routing (`config.rs:407`) — contradicted by `anthropic_thinking_config` classifying independently (`config.rs:136`, `config.rs:161`).
7. The `*` wildcard in `MCP_HOOK_SERVERS` being honoured only as a sole entry, on the grounds that `*` cannot pass the MCP name validator (`config.rs:1104-1110`) — a cross-module assumption with no compile-time link. Tested behaviourally at `config.rs:1166` and `config.rs:1194`, including the deliberate `hs.allows("*") == true` at `config.rs:1202`.
8. **New:** `MESH_MOA_UNAVAILABLE_MESSAGE` (`llm.rs:34`) matching the mesh gateway's 503 text exactly. Enforced by nothing on either side of the boundary.
9. **New:** `prefer_mesh_for_auto` only being set when the base URL actually points at a mesh router. The env var is provider-agnostic at parse time (`config.rs:807`) and the request-time guard only checks `Provider::OpenAi` (`llm.rs:411`), so pointing an unrelated OpenAI-compatible gateway at `auto` with the flag on will make the agent probe `{base_url}/models` on every 5 s window. The convention lives in the desktop caller, which sets the flag only for its own relay-mesh provider (`desktop/src-tauri/src/managed_agents/relay_mesh.rs:37-44`).

#### Tests that re-implement production rules locally
`valid_effort_values_for_provider_model` (`config.rs:2567-2658`) is a test-only function that re-implements the provider→effort-table routing which the TypeScript `getProviderEffortConfig` performs. The fixture guard `effort_table_fixture_matches_rust_implementation` (`config.rs:2674-2708`) tests *this re-implementation* against `effortTable.fixture.json`, not any production function. Three rules exist only inside it:
- the catalog-prefix strip at `config.rs:2578-2590` duplicates `strip_catalog_prefix` (`config.rs:89-97`) line-for-line rather than calling it;
- the default-effort-per-family derivation at `config.rs:2607-2615` (`GPT5_PRO → high`, `GPT5_1 → none`, else `medium`) exists nowhere in production;
- the DBv2 gpt-5 family predicate at `config.rs:2631-2644` duplicates the chain from `config.rs:367-392` while omitting the `gpt-5-6`/`gpt5-6` dash variants that production checks at `config.rs:371-372`.

That last omission is harmless today only because both the `is_gpt5` branch (`config.rs:2645-2647`) and the fall-through (`config.rs:2648-2651`) return the identical `openai_result(&m)` — making the 14-line `is_gpt5` computation dead for every non-empty model name. The fixture can therefore go green while the production and test routing rules diverge.


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

- Auth is obtained with `bearer_no_browser()` (`catalog.rs:117-118`), so discovery never blocks on user interaction.
- Dispatch by provider; anything other than the two Databricks providers is `InvalidParams` (`catalog.rs:123-130`). `host` is `cfg.base_url` with trailing slashes trimmed (`catalog.rs:121`).
- v1 (`api/2.0/serving-endpoints`): a missing `endpoints` array is an error (`catalog.rs:172-180`); each entry needs a string `name` (`catalog.rs:185`); `state.ready` must equal `"READY"` **when present** and `task` must be `llm/v1/chat` or `llm/v1/completions` **when present** — absent fields include the endpoint ("prefer including over silently dropping", `catalog.rs:166-170`, logic at `catalog.rs:187-206`).
- v2 (`api/ai-gateway/v2/endpoints`): `page_size=100`, at most 20 pages (`catalog.rs:244-245`); the loop stops on an absent, empty, or repeated `next_page_token` (`catalog.rs:281-284`, `catalog.rs:383-387`); if the accumulated list is empty it is replaced with `DATABRICKS_V2_KNOWN_MODELS` (`catalog.rs:287-296`).
- v2 chat-capability filter (new in `8eb6e3eb`): an endpoint is dropped unless `is_chat_capable_endpoint` accepts its name (`catalog.rs:370-373`). The rule is name-only because the v2 payload carries neither `task` nor a readiness field (`catalog.rs:85-88`): reject when the lowercased name contains `embedding` anywhere, or when any `-`-delimited segment is exactly `bge` or `gte`; otherwise accept (`catalog.rs:97-106`). Segment matching is deliberate so `databricks-budget-gtex-model` survives (`catalog.rs:102`, asserted `catalog.rs:583`). Unrecognised names are kept, matching the v1 "prefer including over silently dropping" stance (`catalog.rs:93-96`); image endpoints are explicitly in scope of that (`catalog.rs:514-517`). Tests: `v2_parse_drops_embedding_endpoints` (`catalog.rs:498`), `is_chat_capable_endpoint_keeps_unrecognised_names` (`catalog.rs:578`).
- v2 ordering rule (new in `8eb6e3eb`): pages are accumulated in wire order (`catalog.rs:279`) and sorted exactly once, after pagination completes (`catalog.rs:298`). The comparator is descending `created_timestamp` then ascending name (`catalog.rs:337-344`). Because `None < Some(_)`, reversing the timestamp comparison puts endpoints with no usable timestamp **last** (`catalog.rs:339-341`). The name tiebreak is load-bearing, not cosmetic: several managed endpoints share one placeholder timestamp, so without it their order would be arbitrary (`catalog.rs:335-336`). Test: `v2_endpoints_sort_newest_first_then_by_name` (`catalog.rs:542`).
- `created_timestamp` parsing rule: accept a JSON string of digits (the shape the gateway actually sends) or a bare JSON number; anything else yields `None` rather than an error (`endpoint_created_ms`, `catalog.rs:320-325`). Test: `v2_parse_reads_created_timestamp_in_either_wire_shape` (`catalog.rs:522`).
- Discovery-failure fallback (`discovery_failure_fallback`, `catalog.rs:51-80`) is provider-split on purpose. **Changed in `8eb6e3eb`:** `DatabricksV2` → the configured model *first*, then `DATABRICKS_V2_KNOWN_MODELS` with any entry equal to the configured model filtered out (`catalog.rs:61-76`); previously it returned only the two known gateway models. The stated reason is that a fallback catalog which omits the running model leaves the picker unable to represent the current selection (`catalog.rs:40-44`). Legacy `Databricks` → still only the configured model, because gateway IDs may not be servable by `/serving-endpoints/{model}/invocations` (`catalog.rs:45-47`, `catalog.rs:77`).
- Fallback normalisation rule (new in `8eb6e3eb`): the configured model is `trim()`ed once on entry (`catalog.rs:55`) because `resolve_model` does not trim, so a padded `DATABRICKS_MODEL` would otherwise defeat the dedupe and list the model twice (`catalog.rs:52-54`). A configured model that is empty after trimming is omitted entirely (`catalog.rs:63-65`), so the result is exactly `DATABRICKS_V2_KNOWN_MODELS`. Tests: `v2_discovery_failure_fallback_leads_with_configured_model` (`catalog.rs:590`), `..._does_not_duplicate_configured_model` (`catalog.rs:603`), `..._tolerates_blank_configured_model` (`catalog.rs:612`), `..._dedupes_a_padded_configured_model` (`catalog.rs:621`). Three tests in `lib.rs` lock the provider split (`lib.rs:891`, `lib.rs:919`, `lib.rs:945`); the v2 one now asserts `DATABRICKS_V2_KNOWN_MODELS.len() + 1` entries (`lib.rs:926-941`).
- Caller-side rule: a *successful* discovery is cached in a `OnceCell`; a failure deliberately leaves the cell empty so the next `session/new` retries (`lib.rs:312-329`, test `lib.rs:840`).

Contradiction: the doc comment says "Returns a non-empty `Vec<ModelEntry>` on success" (`catalog.rs:110`), but the v1 path returns whatever `parse_v1_endpoints` produced, including an empty vector (`catalog.rs:163`; `v1_parse_empty_endpoints_returns_empty_vec`, `catalog.rs:435`). Only v2 has an empty-list fallback (`catalog.rs:287-296`). Downstream, `session_new` would advertise an empty `availableModels` (`lib.rs:447-457`), while the desktop caller treats empty as an error (`desktop/src-tauri/src/commands/agent_models.rs:740-742`). Re-verified after `8eb6e3eb`: **still true, and unchanged in kind.** The new v2 chat-capability filter runs *before* the empty check (`catalog.rs:370-373` then `catalog.rs:288`), so a v2 workspace serving only embedding endpoints now also parses to an empty list — but that case is caught by the known-model fallback, so v1 remains the only path that can return empty.

#### Test coverage — Business Rules

Covered: server-count cap (`tests/regressions.rs:420 mcp_server_count_cap`), tool-count/description caps (`tests/regressions.rs:354 tool_metadata_caps_enforced`, `681 description_clamping_enforced`), init timeout (`tests/regressions.rs:307 mcp_init_timeout_kills_child`), hook visibility and the `_Stop` budget/fail-open rules (`tests/regressions.rs:787, 872, 927, 979, 1035, 1112, 1514`), `notifications/cancelled` on cancel (`tests/regressions.rs:1573, 1710`), hint chain precedence and skill dedupe (`hints.rs:321, 384, 499, 531, 544, 576`), all six `load_skill` resolution rules (`builtin.rs:277-573`), the catalog parsers, the v2 chat-capability filter, the newest-first sort, the `created_timestamp` wire shapes and the fallback split including its trim/dedupe rules (`catalog.rs:401-630` — 15 unit tests, up from 6 in `8eb6e3eb`; `lib.rs:886-959`), and the whole OAuth ladder except step 5 (`tests/databricks_oauth.rs:105, 144, 179, 212, 261, 305`; `auth.rs:722, 763, 812`).

Not covered: restart attempts and backoff (no test constructs a `Dead` state — `grep -rn 'RESTART_MAX_ATTEMPTS\|RESTART_BASE_MS' crates/buzz-agent/tests` returns zero matches), the `kill_server`→restart cycle described above, hook determinism with more than one server, the 128 KiB hint cap, the 16 KiB hook-result cap, v2 pagination and the 20-page ceiling, the v2 empty-list fallback (`catalog.rs:287-296` — still reached only through `fetch_v2_models`, which no test calls), the interaction between the new chat-capability filter and that fallback (a page of only embedding endpoints), and the browser flow (`grep -rn 'browser_pkce_flow' crates/buzz-agent` matches only `auth.rs:237, 293, 527` — no test file).

One test is written to accept either of two contradictory behaviours: `tool_metadata_caps_enforced` (`tests/regressions.rs:348-352`, branch at `:383-390`) passes whether `spawn_all` rejects a 200-tool server or truncates it. The production rule is "reject" (`mcp.rs:232-236`), so a regression from reject to truncate would not be caught.


## Module: buzz-dev-mcp (`crates/buzz-dev-mcp`)
### Aspect: Business Rules

#### Cross-cutting: path resolution

One function resolves paths for every file tool: `paths::resolve_path`
(`crates/buzz-dev-mcp/src/paths.rs:20-45`).

| Rule | Evidence |
|---|---|
| Absolute inputs are used verbatim | `if raw.is_absolute() { raw.to_path_buf() }` (`paths.rs:32-33`) |
| Relative inputs are joined onto the workspace root | `root.join(raw)` (`paths.rs:34-36`) |
| The joined path is then `std::fs::canonicalize`d | `paths.rs:38-39` |
| Canonicalization failure is the **only** rejection | error string `"path not accessible: {} ({e})"` (`paths.rs:38-41`) |
| No `..` rejection, no containment check, no symlink restriction | module doc states it outright: "No containment enforcement — the resolved path may land anywhere on the filesystem" (`paths.rs:3-6`) |
| The workspace root is whatever the caller passed as `workdir`, unvalidated | `PathBuf::from(w)` with no checks (`paths.rs:107-110`) |

`resolve_path` requires the target to already exist (canonicalize fails
otherwise), so all four file tools are read/modify-only — none can create a new
file at a fresh path.

On Windows only, MSYS-form absolute paths are rewritten before resolution
(`msys_to_windows`, `paths.rs:62-99`): `/c/Users/x` → `C:\Users\x`,
`/c` → `C:\`, `//server/share/x` → `\\server\share\x`. Two-letter leading
segments (`/cc/x`) and root-anchored MSYS paths (`/tmp`, `/usr/bin/git`) are
deliberately left untranslated so they fail with a clear error instead of being
mis-mapped (`paths.rs:51-59`, `paths.rs:96-98`).

`paths::read_text_file` (`paths.rs:102-180`) layers the shared read pipeline on
top:

| Rule | Limit / behaviour | Evidence |
|---|---|---|
| Must be a regular file | `invalid_params` `"not a regular file: …"` | `paths.rs:127-132` |
| Size cap | `MAX_FILE_BYTES = 10 MiB`; `invalid_params` naming both actual and limit | `paths.rs:15`, `paths.rs:131-142` |
| TOCTOU re-check | read is `take(MAX_FILE_BYTES + 1)`; over-cap read yields `"file grew past … bytes during read"` | `paths.rs:154-161` |
| Encoding | strict UTF-8 via `String::from_utf8`; invalid bytes are an `internal_error`, never lossily decoded | `paths.rs:163-176` |

#### `shell`

| Rule | Behaviour | Evidence |
|---|---|---|
| Command size | over `MAX_COMMAND_BYTES = 1_000_000` → `invalid_params` | `shell.rs:18`, `shell.rs:135-140` |
| Timeout default | `120_000 ms` when omitted | `shell.rs:16`, `shell.rs:141-144` |
| Timeout cap | silently `min`-clamped to `MAX_TIMEOUT_MS = 600_000`; a larger request is *not* an error | `shell.rs:17`, `shell.rs:141-144` |
| Working directory | `workdir` if given, else `state.cwd`; must satisfy `is_dir()` or `invalid_params` — no containment check of any kind | `shell.rs:145-159` |
| Interpreter | a shell **is** invoked: `Command::new(shell).arg(flag).arg(command)`; the command string is parsed by the shell | `shell.rs:166-167` |
| Shell flag dispatch | `cmd` → `/C`, `powershell`/`pwsh` → `-Command`, everything else → `-c` | `shell.rs:336-348` |
| Command allow-list / deny-list | none exists anywhere in the crate | `shell.rs:130-323` |
| stdin | always `Stdio::null()` — commands cannot read input | `shell.rs:175` |
| Process group | `process_group(0)` on Unix; Windows Job Object with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` | `shell.rs:674-677`, `shell.rs:759-796` |
| Capture cap | reader stops storing at `CAPTURE_CAP = 10 MiB` but keeps counting `total_bytes` | `shell.rs:19`, `shell.rs:867-892` |
| Truncation trigger | any of: capture capped, over `MAX_BYTES = 50 KiB`, or over `MAX_LINES = 2000` newlines | `shell.rs:20-21`, `shell.rs:906-908` |
| Truncation shape | full captured bytes written to an artifact file, then the **last** `TAIL_BYTES = 8 KiB` returned behind a `[truncated: …]` notice | `shell.rs:22`, `shell.rs:914-956` |
| UTF-8 boundary | tail start is advanced past continuation bytes (`b & 0xC0 == 0x80`) before slicing | `shell.rs:930-931`, `shell.rs:959-965` |
| Encoding of output | `lossy()` — valid UTF-8 passes through, otherwise `from_utf8_lossy` | `shell.rs:967-969` |
| Artifact rotation | ring of `ARTIFACT_RING_SIZE = 8`; the evicted file is deleted | `shell.rs:23`, `shell.rs:971-981` |
| Artifact write failure | non-fatal; appends a `notes` entry and returns `artifact: null` | `shell.rs:917-928` |
| Timeout kill sequence | `SIGTERM` to the process group → 200 ms sleep → `SIGKILL`, then a 2-second `try_wait` poll loop, then `start_kill` + `wait` | `shell.rs:713-724`, `shell.rs:242-269` |
| Timeout exit code | `124` when the process yielded no code and timed out; `-1` otherwise | `shell.rs:298-301` |
| Post-success kill | `kill_graceful()` is called even on a clean exit, to reap orphaned grandchildren holding the pipes | `shell.rs:271-273` |
| Cancellation | `notifications/cancelled` → immediate `SIGKILL` of the group, 1-second bounded reap, reader tasks aborted, tool returns the text `"cancelled"` | `shell.rs:218-239` |
| Reader stall | each reader gets 5 s to finish after process exit; on timeout it is aborted, an empty stream is substituted, and a `notes` entry is added | `shell.rs:275-295` |
| Drop safety | `KillGroup::drop` re-`SIGKILL`s the group unless `disarm()` ran; `disarm()` happens only on the success path and the clean cancel-reap | `shell.rs:731-736`, `shell.rs:226`, `shell.rs:322` |

#### `read_file`

| Rule | Behaviour | Evidence |
|---|---|---|
| Empty file | short-circuits with `"{path} is empty (0 lines)"` before any windowing | `read_file.rs:27-31` |
| Offset | 0-based; clamped with `offset.min(total)` so an out-of-range offset yields an empty slice, not a panic | `read_file.rs:36` |
| Limit | default 2000; `limit: 0` produces the "no lines in range" message | `read_file.rs:34`, `read_file.rs:37-44` |
| Line numbering | output uses 1-based numbers (`offset + i + 1`) while `offset` itself is 0-based | `read_file.rs:46-58` |
| Line splitting | `content.lines()` — `\r\n` line endings lose the `\r`, and a missing trailing newline still counts as a line | `read_file.rs:24`, test `read_file_without_trailing_newline` (`read_file.rs:218-233`) |
| Continuation hint | emitted only when `end_line < total`, advising `offset={end_line}` | `read_file.rs:59-63` |
| Output size | **no byte cap on the returned string** — 2000 arbitrarily long lines are returned in full | `read_file.rs:48-58` |

#### `str_replace`

| Rule | Behaviour | Evidence |
|---|---|---|
| `old_str` non-empty | required; empty → `invalid_params` | `str_replace.rs:26-31` |
| Input size | `old_str`/`new_str` each capped at `MAX_INPUT_BYTES = 1 MiB` | `str_replace.rs:9`, `str_replace.rs:32-37` |
| Match semantics | plain byte-substring search (`str::find` / `str::matches`) — **not** regex, whitespace- and case-sensitive, no normalisation | `str_replace.rs:41-45`, `str_replace.rs:108-122` |
| Uniqueness | without `replace_all`, exactly one match is required; ≥2 → `invalid_params` advising more context | `str_replace.rs:60-68` |
| Counting shortcut | the non-`replace_all` path uses `count_occurrences_capped`, which stops at 2 | `str_replace.rs:108-122` |
| `replace_all` | replaces every occurrence; zero matches is still an error | `str_replace.rs:41-43`, `str_replace.rs:46-58`, `str_replace.rs:84-88` |
| Miss diagnostics | echoes `old_str` truncated to 80 chars plus a fuzzy nearest-line hint | `str_replace.rs:46-58`, `str_replace.rs:157-164` |
| Hint scope | only the first `HINT_SCAN_LINE_LIMIT = 200` lines are scanned, similarity threshold `> 0.6`, comparison strings truncated to 512 bytes | `str_replace.rs:10`, `str_replace.rs:177-217` |
| Growth preflight | projected size computed before allocation; negative or over 10 MiB → `invalid_params` | `str_replace.rs:70-82` |
| Write atomicity | `NamedTempFile::new_in(parent)` → write → flush → `persist(target)` rename | `str_replace.rs:124-138` |
| Permission preservation | original `Permissions` re-applied after the rename; failure is ignored (`let _ =`) | `str_replace.rs:127`, `str_replace.rs:134-137` |
| Diff budget | hunks appended until 64 KiB, then `"[diff truncated]"` | `str_replace.rs:140-155` |
| Encoding | inherits `read_text_file`'s strict UTF-8 rule — binary files cannot be edited | `paths.rs:163-176` |

#### `view_image`

| Rule | Behaviour | Evidence |
|---|---|---|
| `max_dim` | default 1568, `clamp(64, 2048)` — out-of-range values are silently clamped | `view_image.rs:38-40`, `view_image.rs:89-92` |
| Source dispatch order | `data:` → `http(s)://` → any other `scheme://` rejected → filesystem path | `view_image.rs:117-188` |
| Non-http scheme guard | `src.contains("://")` rejects e.g. `ftp://`, preventing it becoming a relative path | `view_image.rs:130-136` |
| Data URL rules | must be `data:image/*`, must declare `;base64`; percent-encoded forms rejected; encoded length prechecked at `ceil(20MiB/3)*4 + 4` chars; decoded length re-checked | `view_image.rs:190-231`, `view_image.rs:118-129` |
| File source | must be a regular file; ≤ `MAX_SOURCE_BYTES = 20 MiB`; read is `take(cap + 1)` for TOCTOU | `view_image.rs:31`, `view_image.rs:137-171` |
| URL source | 10 s connect **and** read timeout; up-front `Content-Length` rejection; per-chunk cumulative cap enforcement | `view_image.rs:50`, `view_image.rs:325-390` |
| Format detection | magic bytes only — extension and `Content-Type` are ignored. Accepts PNG, JPEG, GIF87a/89a, WebP (RIFF…WEBP) | `view_image.rs:394-421` |
| Animation rejection | animated GIF (≥2 image descriptors, byte-level scan) and animated WebP (VP8X flags byte offset 20, bit 1) are rejected outright | `view_image.rs:424-527`, `view_image.rs:534-536` |
| Decompression-bomb guard | dimensions read header-only, then `w*h > MAX_PIXELS = 64 Mpx` → reject *before* decoding | `view_image.rs:45`, `view_image.rs:549-558` |
| Decoder allocation cap | `Limits::max_alloc = 256 MiB` applied on the resize path | `view_image.rs:48`, `view_image.rs:55-61`, `view_image.rs:562-566` |
| Pass-through | if longest edge ≤ `max_dim` **and** bytes ≤ `MAX_FINAL_RAW_BYTES = 3 MiB`, original bytes are returned verbatim with the sniffed MIME | `view_image.rs:35`, `view_image.rs:560-567` |
| Resize | Lanczos3 `resize_exact`, aspect preserved by longest edge, output dims floored at 1 px | `view_image.rs:626-634` |
| Output format choice | driven by the **decoded** colour type, not the input MIME: alpha → PNG, else JPEG q85 | `view_image.rs:640-654` |
| Second pass | if still over 3 MiB, one retry at 75 % of target (floored at 64 px); still over → error | `view_image.rs:586-596` |
| Relay-media auth gate | header attached only when scheme is http/https **and** path starts with `/media/` **and** host and effective port both equal `BUZZ_RELAY_URL`'s | `view_image.rs:236-247` |
| Token shape | Blossom BUD-01 kind-24242 event with `t=get`, `expiration = now + 600 s`, `server=<authority>`; no `x` tag | `view_image.rs:53`, `view_image.rs:252-274` |
| Redirect policy | when a token is attached, redirects are disabled entirely because reqwest does not treat `x-auth-tag` as sensitive | `view_image.rs:327-333` |
| Fail-open auth | missing/invalid `BUZZ_PRIVATE_KEY` or a signing error degrades to an unsigned fetch with a `tracing::warn!` | `view_image.rs:286-318` |
| 401/403 messaging | unauthenticated 401/403 produces a distinct error naming `BUZZ_PRIVATE_KEY` and `BUZZ_RELAY_URL` | `view_image.rs:347-361` |

#### `todo`, `_Stop`, `_PostCompact`

| Rule | Behaviour | Evidence |
|---|---|---|
| Read vs write | `todos` omitted or explicitly `null` → read; array → full replacement | `todo.rs:71-94`, test `explicit_null_is_read` (`todo.rs:400-412`) |
| Item cap | `MAX_ITEMS = 50` | `todo.rs:20`, `todo.rs:147-149` |
| Text length | `MAX_TEXT_CHARS = 200`, measured **after trim**, in `chars()` not bytes | `todo.rs:21`, `todo.rs:154-159` |
| Empty text | rejected after trim | `todo.rs:151-153` |
| Character allow-list | rejects all control chars, all whitespace except ASCII space, U+200B–200F, U+202A–202E, U+2060–206F, U+FEFF — spoofing defence for the rendered list | `todo.rs:126-144`, `todo.rs:160-166` |
| Duplicates | duplicate text after trim is rejected, because silent-removal diffing has no ids to disambiguate | `todo.rs:167-181` |
| Normalisation | text is trimmed on store so diffing and rendering share one canonical form | `todo.rs:75-81` |
| Silent-removal warning | any `!done` item in the old list whose text is absent from the new list triggers a `⚠️` block; removing a `done` item does not | `todo.rs:183-218` |
| Atomicity | validation, mutation, warning computation, and rendering all occur under one lock hold | `todo.rs:58-64`, `todo.rs:78-92` |
| Lock poisoning | `PoisonError::into_inner()` — a poisoned mutex is used anyway rather than panicking | `todo.rs:59-62` (same pattern at `shell.rs:66-69`, `shell.rs:972-975`) |
| Render format | `[x]`/`[ ] {1-based index}. {text}`, with `"  ← next"` appended to the first open item | `todo.rs:219-239` |
| `_Stop` semantics | empty string when the list is empty or fully done; otherwise `"You have open todo items. Keep working."` plus the rendering | `todo.rs:99-112` |
| `_PostCompact` semantics | empty string when the list is empty; otherwise `"# Todo List\n"` plus the rendering | `todo.rs:113-124` |

#### `rg` personality

| Rule | Behaviour | Evidence |
|---|---|---|
| Delegation first | if a real `rg` is found on a self-filtered PATH, args are passed through **unmodified** and its exit code is propagated | `rg.rs:11-29` |
| Self-recursion guard | `clean_path` drops every PATH entry whose `rg` canonicalizes to this binary | `rg.rs:31-47` |
| PATH parsing | hardcoded `split(':')` in both `clean_path` (`rg.rs:34`) and `which_rg` (`rg.rs:50`), and the probe name is bare `"rg"` with no `.exe` (`rg.rs:39`, `rg.rs:54`) — so on Windows the system-`rg` path effectively never resolves and the fallback always runs |
| Executable check | Unix requires mode `& 0o111`; non-Unix returns `true` unconditionally | `rg.rs:62-73` |
| Fallback flag allow-list | only `--files`, `-n/--line-number`, `-i/--ignore-case`, `-l/--files-with-matches`, `-C/--context`, `-g/--glob`, `--`; every other `-`-prefixed token errors with exit 2 | `rg.rs:87-135` |
| Fallback match semantics | literal substring (`line.contains(needle)`), lowercased on both sides for `-i` — **not** regex, diverging from real `rg` | `rg.rs:198-204`, `rg.rs:289-296` |
| Context cap | `-C` value clamped to `MAX_CONTEXT = 100` | `rg.rs:8`, `rg.rs:99-103` |
| Glob engine | hand-rolled `*`/`?` matcher tried against both the file name and the full path; no `**`, no character classes, no brace expansion | `rg.rs:388-434` |
| Output caps | `MAX_OUTPUT_BYTES = 50 KiB` / `MAX_OUTPUT_LINES = 2000`; on breach the sink latches `capped`, logs a `tracing::warn!`, and silently stops printing — **no truncation marker reaches stdout** | `rg.rs:6-7`, `rg.rs:137-166` |
| Long-line skip | a line exceeding `MAX_LINE_BYTES = 1 MiB` aborts the whole file scan | `rg.rs:5`, `rg.rs:228-261`, `rg.rs:283-287` |
| Non-UTF-8 files | a decode failure aborts the file scan and returns whatever was already found | `rg.rs:252-259`, `rg.rs:284-286` |
| Fallback walk exclusions | skips dot-prefixed names, all symlinks, and the fixed set `target`, `node_modules`, `dist`, `build`; depth capped at `MAX_WALK_DEPTH = 50`. Does **not** read `.gitignore` | `rg.rs:345-386` |
| Exit codes | `0` found, `1` not found, `2` bad args | `rg.rs:168-227` |

#### `tree` personality

| Rule | Behaviour | Evidence |
|---|---|---|
| Depth | default and hard cap both `MAX_WALK_DEPTH = 50`; requested depth is `min`-clamped, never rejected | `tree.rs:8`, `tree.rs:138`, `tree.rs:167` |
| Argument rules | one positional path only; a second positional or a second path after `--` errors "multiple paths not supported"; unknown flags error; exit 2 | `tree.rs:137-168` |
| Root validation | non-directory root → `"tree: not a directory"` + exit 2 | `tree.rs:26-29` |
| Ignore semantics | uses `ignore::WalkBuilder` with `git_ignore`, `git_exclude`, `git_global`, `ignore` all on, `require_git(false)`, and `hidden(true)` — unlike the `rg` fallback, gitignore rules apply | `tree.rs:41-50` |
| Ordering | `sort_by_file_name` for deterministic output | `tree.rs:49` |
| Annotation | every file gets `[line-count]`; every non-leaf directory gets a subtree total; the root line carries the grand total | `tree.rs:53-117` |
| Depth-boundary directories | directories at exactly `max_depth` are marked `leaf` and get **no** count annotation | `tree.rs:80-86`, `tree.rs:69-74` |
| Line counting | files over `MAX_FILE_BYTES = 10 MiB`, or unreadable, count as `0` — silently, with no marker | `tree.rs:9`, `tree.rs:170-184` |
| Output caps | line budget `MAX_OUTPUT_LINES - 1 = 1999`, byte budget `50 KiB`; both emit an explicit `[truncated]` line | `tree.rs:38`, `tree.rs:76-79`, `tree.rs:120-134` |
| Write failure | a failed `writeln!` returns exit `0`, so a broken pipe looks like success | `tree.rs:106-108`, `tree.rs:126-128` |


## Module: buzz-cli — dispatch, relay client & validation (`crates/buzz-cli/src`)
### Aspect: Business Rules

#### Startup ordering rules

1. Install the `ring` rustls provider, ignoring failure (`lib.rs:39`,
   `let _ = rustls::crypto::ring::default_provider().install_default()`). The
   swallow is deliberate and documented (`lib.rs:30-38`): when `buzz-dev-mcp`
   delegates to `run_from_args`, it has already installed a provider
   (`crates/buzz-dev-mcp/src/lib.rs:165`), and the second install returns `Err`.
2. Parse argv. Usage errors print the JSON error envelope and return 1;
   help output prints verbatim and returns 0 (`lib.rs:41-53`).
3. Normalize the relay URL *before* any auth check (`lib.rs:1734`).
4. `pack` short-circuits: no key, no client, no relay (`lib.rs:1736-1743`).
   This makes `pack validate|inspect` the only commands runnable without
   `BUZZ_PRIVATE_KEY`.
5. Require the private key (`lib.rs:1746-1748`), parse it (`lib.rs:1749-1750`),
   then parse *and verify* the NIP-OA auth tag (`lib.rs:1752-1767`), then build
   the client (`lib.rs:1768`).

#### Auth rules

| Rule | Behavior | Site |
|---|---|---|
| Flag beats env | clap `env =` fallback, explicit arg wins | `lib.rs:81-89`; verified: `BUZZ_RELAY_URL=http://127.0.0.1:9 buzz --relay notaurl channels list` fails on `notaurl` |
| Key is mandatory | missing → `CliError::Auth`, exit 3 | `lib.rs:1746-1748`; verified exit 3 |
| Key format | `Keys::parse` accepts hex or nsec; failure → `CliError::Key`, exit 3 | `lib.rs:1749-1750`; verified with `BUZZ_PRIVATE_KEY=zzz` |
| Empty auth tag == unset | `Some(json) if !json.is_empty()` | `lib.rs:1755`; verified `BUZZ_AUTH_TAG=''` proceeds to the relay call |
| Auth tag must parse **and** verify against the signer's pubkey | `parse_auth_tag` then `verify_auth_tag(json, keys.public_key())`; either failure → `CliError::Auth`, exit 3 | `lib.rs:1756-1766`; verified with a bogus clause: `auth error: BUZZ_AUTH_TAG is malformed: … unsupported clause: "{}"` |
| The keypair *is* the identity | in-code comment "no tokens, no other auth" | `lib.rs:1745-1746` |

Per-request auth: every HTTP call signs a fresh NIP-98 kind-27235 event with
`u`, `method`, a UUIDv4 `nonce` and (for bodies) a `payload` SHA-256 tag
(`client.rs:84-110`); the nonce exists specifically so retries survive the
relay's replay guard (`client.rs:886-887`, `client.rs:1041-1042`).

#### Auth-tag injection rules

`sign_event` (`client.rs:588-614`) injects `self.auth_tag` and then **counts**
`auth` tags on the signed event, erroring if the count differs from the expected
0-or-1 — i.e. "callers must not add auth tags manually" is enforced at runtime,
not just by comment (`client.rs:606-613`).

`sign_event_unchecked` (`client.rs:743-747`) deliberately bypasses both
injection and the count check, and is reserved by *convention only* (a doc
comment, `client.rs:729-742`) for NIP-IA kinds 9035/9036. Nothing restricts it
to those kinds. Two tests pin the intent:
`sign_event_unchecked_does_not_inject_ambient_auth_tag` (`client.rs:2373-2400`)
and `sign_event_unchecked_preserves_callers_content_auth_tag`
(`client.rs:2401-2437`).

#### Output-format selection

`--format` is threaded to only 5 of the 21 dispatchers — `messages`,
`channels`, `users`, `feed`, `moderation` (`lib.rs:1772,1773,1778,1780,1790`).
The other 16 dispatch calls omit it, so `buzz --format compact social notes …`
parses successfully and is silently ignored. No warning is emitted; there is no
code path that rejects `compact` for an unsupported group
(`grep -n 'cli.format' lib.rs` → exactly those 5 lines).

#### Retry, timeout and idempotency rules

Two policies, chosen by event kind. `is_moderation_kind` = `9040..=9044`
(`client.rs:211-213`, test `client.rs:1511-1517` plus a negative test at
`1518-1527`).

| Failure | Stored events (all other kinds) | Moderation kinds 9040-9044 |
|---|---|---|
| TCP connect error | retry, and on exhaustion stay `Network` (retryable) | same (`client.rs:900-910`) |
| Timeout / request / body / decode | retry (`client.rs:653-661`), exhaustion → `DeliveryUnknown` (`client.rs:1060-1069`) | **no retry**, immediate `DeliveryUnknown` (`client.rs:911-923`) |
| 429 with body starting `rate-limited:` | retry honoring `retry in Ns` capped at 30 s (`client.rs:662-668`) | retry; exhaustion → `Relay{429}` (still retryable) (`client.rs:930-957`) |
| 429 otherwise (proxy) | retried as a plain 429 by `with_retry_body` | `DeliveryUnknown` immediately (`client.rs:958-963`) |
| 502-504 | retry, exhaustion → `DeliveryUnknown` (`client.rs:669-671`, `client.rs:1060-1069`) | `DeliveryUnknown` immediately (`client.rs:964-972`) |
| Other 4xx | no retry | no retry (`client.rs:983-1005`) |

Attempt budget is 3 (`client.rs:122`); inter-attempt delay is full jitter in
`[0, RETRY_BASE_SECS[attempt])` = `[0,0.5)` then `[0,1.5)`
(`client.rs:126-135`). The ambiguity classifier
`is_stored_event_exhaustion_ambiguous` (`client.rs:229-248`) is the rule that
decides retryability on exhaustion: connect → not ambiguous, canonical 429 →
not ambiguous, everything else → ambiguous.

Timeouts: per-request 30 s and connect 15 s, overridable by
`BUZZ_TIMEOUT_SECS` / `BUZZ_CONNECT_TIMEOUT_SECS`, with **zero treated as
invalid** so timeouts can never be disabled (`client.rs:140-150`,
`client.rs:547-551`; test `env_duration_secs_parsing`, `client.rs:1555-1580`).
Uploads override per-request: 120 s images, 600 s video
(`client.rs:1141-1145`); downloads use a dedicated 120 s client
(`client.rs:1231-1237`); WS publish is capped at 75 s (`client.rs:1084`).

Retry rules are well covered by behavioral integration tests over a local axum
server (`client.rs:1582-2295`): `moderation_kind_non_ingest_429_returns_delivery_unknown`
(`:1658`), `moderation_kind_ingest_429_is_retried_until_success` (`:1688`),
`exhausted_ingest_429_returns_relay_429_retryable` (`:1731`),
`moderation_kind_502_returns_delivery_unknown` (`:1764`),
`exhausted_connect_failures_return_network_retryable` (`:1787`),
`stored_event_502_is_retried_under_standard_policy` (`:1814`),
`query_403_is_not_retried` (`:1948`),
`with_retry_body_retries_on_body_transfer_failure` (`:1978`),
`stored_event_body_loss_is_retried_with_same_event_bytes` (`:2038`),
`upload_body_loss_is_retried_with_same_file_bytes` (`:2116`),
`stored_event_all_body_losses_return_delivery_unknown` (`:2207`),
`stored_event_all_502s_return_delivery_unknown` (`:2277`).

#### Exit-code mapping rules

See the API Surface aspect for the full table. The mapping rule of note:
`Relay{status}` splits on 401/403 → 3, else → 2 (`error.rs:96-100`), and the
same split drives the `error` category string (`error.rs:111-117`).
`is_retryable_error` (`error.rs:74-88`) publishes retryability separately from
the exit code, and explicitly refuses to mark `DeliveryUnknown` retryable
(`error.rs:85`).

#### Query, kinds and pagination rules

- `kinds` is **not** defaulted or required anywhere in this layer.
  `grep -c '"kinds"' client.rs` → 1, and that single hit is a test fixture
  (`client.rs:2306`). The relay p-gate documented in `AGENTS.md` gotcha 2 is
  satisfied only by the sibling command modules (e.g.
  `commands/messages.rs:276,320,361`).
- Page size is fixed at 500 (`client.rs:498`); each request asks for
  `min(remaining, 500)` (`client.rs:687-691`).
- Termination rule: a short page ends the loop; a full page advances the
  `(until, before_id)` cursor (`client.rs:692-700`).
- `query_all` (`client.rs:724-727`) has **no page or memory cap** — it loops
  until the relay returns a short page, accumulating every event in a `Vec`.
- `channels search --limit` defaults to 1000 metadata events (`lib.rs:527-528`);
  `moderation reports`/`audit --limit` default to 50 (`lib.rs:1656-1657`,
  `lib.rs:1729-1730`); most other `--limit` flags are `Option<u32>` with the
  default deferred to the relay or the sibling handler.

#### Channel scoping

No `h`-tag logic exists in this group: `grep -c '"h"' client.rs` → 0. NIP-29
scoping is entirely the command modules' concern. The one place this group
touches `h` is a *negative* assertion — agent draft frames must carry no `h`
tag (`agent_management.rs:225-227`).

#### NIP-33 last-write-wins

The `Conflict` variant and its exit code 5 exist here (`error.rs:26-29`,
`error.rs:103`) with a doc comment naming `buzz mem` set/rm as the producer, but
nothing in the six in-scope files constructs a `Conflict`:
`grep -n 'CliError::Conflict' lib.rs client.rs validate.rs agent_management.rs`
returns zero matches outside `error.rs` itself. The LWW rule is implemented in
the sibling-owned `commands/mem.rs`; this layer only carries the code.

#### Media rules

`media_url_from_input` (`client.rs:270-323`) enforces, in order:
absolute URLs must be `http(s)` (`:272`), path must start `/media/`
(`:275-284`), the segment must be `sha256`, `sha256.ext` or `sha256.thumb.jpg`
with lowercase hex and an `[a-z0-9]{1,8}` extension (`client.rs:250-268`), and
the origin (scheme + host + effective port) must equal the configured relay —
otherwise "refusing to sign media GET for a non-relay origin" (`:305-315`).
Bare inputs get `/media/` prefixed after the same segment check (`:317-322`).
Three tests cover this (`client.rs:391-451`).

Upload rules: regular file required (`client.rs:1102-1106`), MIME sniffed from
magic bytes rather than the extension (`client.rs:1109-1112`), MIME must be in
the 5-entry allowlist (`client.rs:1114-1116`), size ≤ 50 MiB image / 500 MiB
video (`client.rs:1118-1130`). Endpoint fallback is narrow by rule: only 404 and
405 switch to `/media/upload`, and the switch itself is not retried
(`client.rs:200-207`, `client.rs:1180-1193`; test
`legacy_upload_retry_statuses_are_narrow`, `client.rs:483-496`).

Blossom auth expiry rules: get = now+600 s (`client.rs:329-330`); upload =
now+3600 s for `video/*`, else now+600 s (`client.rs:359-363`).

#### Validation rules (`validate.rs`)

| Rule | Detail | Site |
|---|---|---|
| UUID | strict `Uuid::parse_str` | `:23-27` |
| 64-hex | length 64 **and** `is_ascii_hexdigit` — uppercase accepted despite the doc saying "lowercase" | `:28-36` |
| repo id | 1-64 chars, no leading `.`, no `..`, `[A-Za-z0-9._-]` only | `:39-61` |
| content size | ≤ 65,536 **bytes** (`str::len`, not chars) | `:64-72` |
| diff truncation | cut at the last `\n@@` inside the budget, else last newline, always UTF-8-boundary safe, notice appended | `:103-121` |
| language inference | extension → language, 26 mappings, `None` when unknown | `:124-152` |
| SDK error mapping | `SdkError::InvalidInput` → `Usage` (exit 1), all else → `Other` (exit 4) | `:155-160` |
| `-` means stdin | `read_or_stdin` treats any other value as literal content; `read_file_or_stdin` always treats it as a path | `:162-193` |

The `read_or_stdin` vs `read_file_or_stdin` split is itself a bug-fix rule with
a regression test — `read_file_or_stdin_does_not_treat_path_as_literal_content`
(`validate.rs:498-505`) documents the prior bug where `--patch-file` echoed the
path as content.

#### Agent-draft rules (`agent_management.rs`)

- Every string field is trimmed, must be non-empty, and is length-capped in
  characters (not bytes): 120 for names, 20,000 for prompts, 300 for other
  optional fields (`agent_management.rs:70-85`).
- `channel` must be a valid UUID *and* ≤128 chars (`:129-131`, `:145-147`).
- `respond_to` must be exactly `owner-only` or `anyone` (`:148-155`) — a second,
  string-level check on top of the `RespondToArg` value enum (`lib.rs:242-257`),
  because the public struct takes a free-form `Option<String>`.
- An update must change at least one field, else `Usage`
  (`agent_management.rs:169-179`; test `update_requires_a_change`, `:243-262`).
- Drafts are encrypted to the owner pubkey and published as observer telemetry
  frames (`:107-121`), so the rule "owner reviews before anything is created" is
  structural: the CLI never writes an agent record itself.

#### Rules enforced only by comment or convention

- `sign_event_unchecked` is "NIP-IA only" by doc comment (`client.rs:729-742`).
- "Callers MUST NOT add `auth` tags before calling `sign_event`" is documented
  (`client.rs:585-587`) *and* enforced (`client.rs:606-613`) — the good case.
- `AGENTS.md`'s no-`unwrap`/`expect` rule is not enforced by lint; three
  production panic sites remain (`client.rs:506`, `client.rs:1379`, plus
  `unreachable!` at `client.rs:679`, `client.rs:1018`, `lib.rs:1791`).
- The `retry in Ns` cap of 30 s is described as "defensive … real relay hints
  observed up to ~24 s" (`client.rs:128-130`) — a convention derived from
  observation, not a negotiated protocol value.


## Module: buzz-cli — channel, message & social commands (`crates/buzz-cli/src/commands`)
### Aspect: Business Rules

#### Enumerated value gates

| Rule | Accepted values | Site | Rejection |
|------|----------------|------|-----------|
| channel type | `stream`, `forum` | `channels.rs:283-290` | `Usage` |
| channel visibility | `open`, `private` | `channels.rs:291-298` | `Usage` |
| template type/visibility override | same two sets, re-checked after the template supplies defaults | `channels.rs:680-697` | `Usage` |
| member role | `owner`,`admin`,`member`,`guest`,`bot` | `channels.rs:964-976` | `Usage` |
| add policy | `anyone`,`owner_only`,`nobody` | `channels.rs:1006-1013` | `Usage` |
| message kind | `9` (or omitted), `45001`, `45003` | `messages.rs:540-572` | `Usage` at `:568` |
| vote direction | `up`,`down` | `messages.rs:730-737` | `Usage` |
| feed types | `mentions`,`needs_action`,`activity`,`agent_activity` | `feed.rs:6`, checked at `:31-39` | `Usage` |
| social list kind | `10000,10001,10002,10003,30000,30003` | `social.rs:127-139` | `Usage` |
| emoji export scope | `own`,`workspace` | clap enum `lib.rs:145-150`, matched at `emoji.rs:201`/`:216` | clap |

`channels create` and `channels create --template` duplicate the type/visibility
validation and the string→enum mapping four times total
(`channels.rs:283-298` + `:300-320` vs `channels.rs:680-697` + `:699-709`), each
ending in `unreachable!()` (`:314`, `:319`, `:703`, `:708`).

#### Mutual exclusions and required combinations

| Rule | Enforced where | Notes |
|------|---------------|-------|
| `channels create` needs `--type`+`--visibility` unless `--template` | clap `required_unless_present` (`lib.rs:551`,`:554`), re-checked defensively in dispatch (`channels.rs:1136-1139`) | dispatch comment says clap guarantees it |
| `channels update` needs at least one of `--name/--description/--ttl/--no-ttl` | `channels.rs:843-847` | SDK repeats the check (`builders.rs:610-614`) |
| `--ttl` vs `--no-ttl` | clap `conflicts_with` (`lib.rs:586`) plus the tri-state match at `channels.rs:836-840` | `Some(None)` clears TTL, emitting `["ttl",""]` (`builders.rs:643-646`) |
| `messages send-diff` branch pair | `messages.rs:601-608` — both or neither | |
| `messages search` needs `--query` or `--author` | `messages.rs:345-349` | |
| `messages send --kind 45003` needs `--reply-to` | `messages.rs:546-548` | |
| `notes get`: exactly one of `--naddr`/`--name`; `--author` and `--latest` only with `--name`; `--author` xor `--latest` | `validate_get_args` `notes.rs:588-610`, called at `notes.rs:794` | not expressed in clap, so absent from `--help` |
| `notes set`: `--clear-tags` xor `--tag` | `notes.rs:775-779` (in dispatch, not `cmd_set`) | |
| `dms open`: 1–8 pubkeys | `dms.rs:52-54` | re-implements `builders.rs:1544-1548`, which the CLI bypasses |
| `emoji list --format compact` | not a rule — silently ignored (no format arg reaches `emoji::dispatch`, `emoji.rs:311`) | |

A silent-drop case: `messages send --kind 45001 --reply-to <id>` resolves the
thread ref (`messages.rs:534-538`, costing a relay round trip) and then calls
`build_forum_post`, which takes no thread ref (`messages.rs:542`,
`builders.rs:278-289`). The reply linkage is discarded with no warning. Only
kinds 9 and 45003 thread.

#### `h`-tag scoping

Every channel-scoped write carries `h` and never `e` for the channel, matching
AGENTS.md. Reads are less uniform:

| Read | Channel scoping | Site |
|------|----------------|------|
| `messages get` | `#h` | `messages.rs:277` |
| `messages thread` (reply filter) | `#h` + `#e` | `messages.rs:321-322` |
| `messages thread` (root filter) | none — `ids` only | `messages.rs:328-331` |
| `canvas get` | `#h` | `channels.rs:266` |
| `channels get`/`members`, mention resolution | `#d` (39000/39002 are NIP-33-addressed by channel UUID) | `channels.rs:228`,`:251`, `messages.rs:140` |
| `messages search` | **no channel scoping at all** | `messages.rs:360-372` |
| `reactions get`/`remove` | `#e` only | `reactions.rs:83-85`, `:45-48` |
| `feed get` | `#p` only | `feed.rs:19-22` |
| `social` reads, `notes` reads | not channel-scoped by design (global kinds) | `social.rs:74-121`, `notes.rs:170-192` |

`messages search` is community-wide: the relay intersects with accessible
channels server-side (`bridge.rs:1661-1675`), so there is no client-side
`--channel` filter available. `buzz messages search --query x` cannot be scoped
to one channel through this CLI.

#### `kinds` in filters and the p-gate

AGENTS.md says a query without `kinds` "triggers the p-gate (403)". The actual
rule (`req.rs:1038-1071`, `p_gated_filters_authorized`) is: a kindless filter is
treated as *possibly* matching a p-gated kind, and then passes if it has non-empty
`ids`, or if every `#p` value equals the authed pubkey. Checked against the four
kindless filters in scope, all pass legitimately:

| Kindless filter | Why it passes |
|-----------------|---------------|
| `social event` `{ids:[…]}` (`social.rs:74-76`) | `ids` exemption (`req.rs:1064`) |
| `resolve_thread_ref` `{ids:[…],limit:1}` (`messages.rs:63`) | `ids` exemption |
| `resolve_channel_id` `{ids:[…]}` (`messages.rs:90-92`) | `ids` exemption |
| `feed get` `{#p:[me],limit:N}` (`feed.rs:19-22`) | `#p` equals self (`req.rs:1068-1070`) |

AGENTS.md gotcha 3 ("`messages search` must include `--kinds`, pass at least
`--kinds 9,45001,45003`") is stale: `MessagesCmd::Search` has no `--kinds` flag
(`lib.rs:475-491`) and `cmd_search` hardcodes `kinds: [9,40002,45001,45003]`
(`messages.rs:361`). `crates/buzz-cli/TESTING.md:223` documents the working form
without `--kinds`, so the code and the runbook agree and AGENTS.md is the outlier.

The three message read paths use three different kind sets:

| Command | Kinds | Site |
|---------|-------|------|
| `messages get` | 9, 40002, 40008, 45001, 45003 | `messages.rs:276` |
| `messages thread` | 9, 40002, 40003, 40008, 45003 | `messages.rs:320` |
| `messages search` | 9, 40002, 45001, 45003 | `messages.rs:361` |

So `get` omits edits (40003), `thread` omits forum roots (45001) but includes
edits, and `search` omits both edits and diffs (40008). Nothing in the code
explains the divergence.

`messages get --kinds` overrides the default set, parsed with
`filter_map(|s| s.trim().parse().ok())` (`messages.rs:284`). Unparseable entries
are dropped silently, and if *every* entry is bad the `if !kind_list.is_empty()`
guard (`:285`) leaves the default kinds in place — `--kinds nine` returns the
default result set instead of erroring.

#### Threading rules

`resolve_thread_ref` (`messages.rs:58-86`) treats `--reply-to` as the **immediate
parent** and derives the root from the parent's own NIP-10 tags via a relay
fetch. `find_root_from_tags` (`messages.rs:25-56`) prefers an explicit `root`
marker, falls back to a `reply` marker, ignores unmarked `e` tags (NIP-10
deprecated positional markers), and defensively ignores marker values that aren't
64-hex (`messages.rs:39-45`) so a malformed parent can't block a send. When no
marker is found, root == parent (`messages.rs:81-84`). The SDK then emits one
`["e",root,"","reply"]` for a direct reply, or root+reply pairs for nested
(`builders.rs:171-181`).

`reply_count` / `descendant_count`: AGENTS.md requires any code inserting replies
to maintain these materialized counters. `grep -rn 'reply_count\|descendant_count'
crates/buzz-cli/src` returns **zero matches** — the CLI never reads or writes
them, delegating entirely to the relay's ingest path. That is consistent with the
CLI submitting signed events rather than rows, but it means the CLI has no
guardrail if the relay ever stops materializing them.

`messages thread` asks the relay for reply expansion via the non-standard
`depth_limit` filter field (`messages.rs:326-327`, consumed at
`bridge.rs:1140-1146`) and ORs in a root-by-id filter so the root event is
returned alongside its replies (`messages.rs:328-332`).

#### Pagination, limits and defaults

| Command | Default | Client cap | Site |
|---------|---------|-----------|------|
| `messages get` | 50 | 200 | `messages.rs:273` |
| `messages thread` | 100 | 500 | `messages.rs:314` |
| `messages search` | 20 | 100 | `messages.rs:353` |
| `feed get` | 20 | 50 | `feed.rs:15` |
| `dms list` | 50 | 200 | `dms.rs:10` |
| `notes ls` | 50 | 200 | `notes.rs:672` |
| `social notes` | 50 | 100 | `social.rs:93` |
| `social list` | fixed 10 | — | `social.rs:202` |
| `notes get --name` candidate scan | fixed 50 | — | `notes.rs:191` |
| author-name resolution | fixed 100 | — | `messages.rs:409`, `notes.rs:216` |
| `channels list` | 500 | none | `channels.rs:31` |
| `channels search` | 1000 (clap) | none | `lib.rs:539`, `channels.rs:133` |
| `emoji list`/`export --scope workspace`, `reactions get`/`remove`, `canvas get` | **no `limit` sent** | — | `emoji.rs:78-81`,`:218-221`; `reactions.rs:82-85`,`:44-48`; `channels.rs:264-267` |

Filters with no `limit` fall to the relay's `MAX_HISTORICAL_LIMIT` of 2000
(`req.rs:25`, applied at `req.rs:878-882`). So the workspace emoji palette
silently truncates at 2000 kind-30030 events and a message's reaction list at
2000 kind-7 events, with no indication to the caller.

`--limit` for `channels list` and `channels search` bounds *events fetched*, not
matches returned: filtering happens client-side after pagination
(`channels.rs:66-95` for visibility, `channels.rs:134-142` for name/archived). A
`channels search --query x --limit 10` can therefore return zero rows while
matches exist beyond the first 10 events.

Only `channels list`/`search` and the template roster scan paginate
(`query_paginated`/`query_all`, `channels.rs:42`,`:65`,`:133`,`:449`); the
roster scan is deliberately keyset-paginated because 30177 is
parameterized-replaceable and offset drift could skip a live instance
(`channels.rs:434-439`, doc comment).

`social notes` forwards `--before`→`until` and `--before_id`→`before_id`
(`social.rs:102-108`), the composite cursor the relay requires as both-or-neither
(`bridge.rs:1233-1247`); the CLI does **not** enforce that pairing, so
`--before-id` alone yields a relay 400.

#### Ordering rules

| Command | Ordering | Site |
|---------|----------|------|
| `messages get` / `thread` | ascending `created_at`, client-side | `messages.rs:298`, `:334` |
| `messages search` with `--query` | relay relevance order preserved | `messages.rs:375-381` (explicit comment) |
| `messages search` without `--query` | descending `created_at`, client-side | `messages.rs:377-381` |
| `feed get` | descending `created_at` | `feed.rs:44` |
| `channels search` | by name, then channel_id | `channels.rs:143-147` |
| `emoji list`/`export` | by shortcode (then url for `--scope own`) | `emoji.rs:71`, `:210` |
| `reactions get` | by emoji string | `reactions.rs:116-122` |
| `notes ls` / candidates | descending `updated_at` | `notes.rs:388`, `:281-283` |
| `canvas get`, `channels get`, `dms list` | **none** — takes `events.first()` / relay order as-is | `channels.rs:270`, `:233`, `dms.rs:20` |

`notes.rs` sorts defensively before taking the head of a
parameterized-replaceable coordinate ("the relay keeps only the latest… but be
defensive", `notes.rs:178-181`, `:361-363`); `channels.rs:270` and `emoji.rs:107`
rely on relay ordering instead (`emoji.rs:106` uses `events.last()` with the same
"be defensive" comment but no sort, so with multiple events it takes the *last*
returned, i.e. the oldest under `created_at DESC`).

#### Edit / delete semantics

- `messages edit` publishes kind 40003 with an `e` tag to the target and no
  ownership check (`messages.rs:701-720`); authorization is left to the relay.
- `messages delete` publishes Buzz-native kind 9005, not NIP-09 kind 5
  (`builders.rs:429`), and can attach `action_id`/`reason_code`/`public_reason`
  for moderator tombstones (`messages.rs:685-689`) with no validation of those
  free-text values.
- `notes rm` publishes NIP-09 kind 5 with an **`a` tag only**. The no-`e`-tag
  property is load-bearing: the relay routes to its coordinate soft-delete path
  only when the kind:5 has no `e` targets (`notes.rs:704-711` doc). A test pins
  both halves (`notes.rs:1264-1296`).
- `reactions remove` is a read-then-delete: it queries the caller's own kind-7
  events on the target and deletes the first whose `content` equals the `--emoji`
  string exactly (`reactions.rs:60-73`). Custom-emoji reactions store content as
  `:shortcode:` (`builders.rs:487-491`), so `reactions add --emoji party
  --emoji-url …` followed by `reactions remove --emoji party` fails with "no
  reaction with emoji 'party' found"; the user must pass `:party:`. Nothing in the
  code or `--help` (`lib.rs:717-722`) states this.
- `notes rm` refuses when the caller has no note under the slug
  (`notes.rs:721-726`), giving `NotFound` rather than emitting a pointless kind 5.

#### Reaction dedupe and counting

`reactions add` performs no duplicate check, so the same user can stack identical
reactions (`reactions.rs:9-32`). `reactions get` groups by `content`, defaults
empty content to `"+"` (`reactions.rs:99-104`), and reports
`count = pubkeys.len()` **without deduplicating by pubkey**
(`reactions.rs:107-114`) — a user who reacted twice is counted twice and appears
twice in `pubkeys`. Whether the relay filters deleted reactions out of the kind-7
query is not determinable from this module; I did not verify it.

#### Emoji scope and merge rules

- Each member publishes their own kind:30030 under the fixed d-tag
  `buzz:custom-emoji` (`emoji.rs:9`); the workspace palette is a **client-side
  union** of all members' sets (`emoji.rs:49-75`).
- Union rule: newest `created_at` wins per shortcode; equal timestamps tie-break
  to the lexicographically smallest URL, making the result independent of fetch
  order (`emoji.rs:55-63`, doc `:44-48`). Two tests pin it, including the
  reversed-input invariance check (`emoji.rs:331-372`, `:374-388`).
- `emoji set`/`rm` are read-modify-write over the caller's own set only
  (`emoji.rs:128-139`, `:141-157`); `rm` short-circuits without publishing when
  the shortcode is absent (`emoji.rs:146-154`).
- `emoji import` normalizes every shortcode, dedupes within the batch
  (first occurrence wins, `emoji.rs:276-279`), then either replaces the whole set
  (`--replace`) or merges keeping **existing** entries on conflict
  (`emoji.rs:281-296`). `--dry-run` prints the computed set to stdout and the
  marker to stderr without publishing (`emoji.rs:298-307`).
- Shortcode normalization (trim, strip colons, ≤64 bytes, `[A-Za-z0-9_-]`,
  lowercased) and URL checks (non-empty, ≤2048 bytes, http/https) live in the SDK
  (`builders.rs:127-168`), invoked at `emoji.rs:129`, `:142`, `:269`.

#### Membership, ownership and add-policy rules

- No command checks membership locally. `channels add-member`, `remove-member`,
  `join`, `leave`, `archive`, `delete`, `messages edit`/`delete` all sign and
  submit, relying on relay-side authorization.
- `channels set-add-policy` has a client-side deployment gate: if
  `BUZZ_ACP_ALLOWED_CHANNEL_ADD_POLICIES` is set and non-empty, the requested
  policy must be in the comma-separated list (`channels.rs:1021-1033`). The
  in-code comment is explicit that this covers only the CLI path and a direct
  kind:10100 submission bypasses it (`channels.rs:1015-1020`) — a rule enforced by
  convention, not by the relay.
- Template roster owner invariant (labelled F1): the effective owner is the
  NIP-OA auth-tag owner if present, else the signing pubkey, with **no**
  sole-author fallback so a same-slug 30176/30177 from another principal can never
  be selected (`channels.rs:645-651`, `client.rs:576-584`).
- Template cardinality rule (labelled F4, `channels.rs:472-503`): zero live
  instances for a persona slug is a skip, exactly one is added as
  `MemberRole::Bot` (`channels.rs:750`), more than one is a hard `Usage` error
  listing candidate pubkeys and aborting before any write.
- Archive filtering (NIP-IA) runs before cardinality and **fails open**: if the
  archived-identities snapshot can't be trusted, the filter proceeds with an empty
  archived set and threads a warning into both stderr and the report
  (`channels.rs:511-587`, `:589-603`). Skips are labelled `"all instances
  archived"` vs `"no live instances"` (`channels.rs:568-580`).
- Template writes are best-effort after creation: canvas failure sets
  `canvas_applied:false` (`channels.rs:736-737`), per-member failures land in
  `member_failures` and downgrade `status` to `"partial"`
  (`channels.rs:763-772`). Members are added sequentially because concurrent
  kind:9000 writes are LWW on the relay (`channels.rs:741-743`).

#### NIP-33 LWW conflict handling

Only `notes set` treats a `duplicate:`/`duplicate` relay message as a conflict,
returning `CliError::Conflict` → exit 5 (`notes.rs:556-566`). `notes rm` parses
`accepted` but not `duplicate:` (`notes.rs:734-745`). `emoji set`/`rm`/`import`
publish parameterized-replaceable kind 30030 and never inspect the message, so a
dominated emoji-set write reports success (`emoji.rs:118-124`). The same is true
of `social set-list` for kinds 30000/30003 (`social.rs:179-180`, which prints the
raw response).

#### DM handling

`dms` implements Buzz's channel-style DMs, not NIP-17: `open` creates kind 41010
with `p` tags plus a client-generated `d` UUID (`dms.rs:57-70`), `add-member` is
41011, `hide` is 41012, and `list` reads relay-confirmed 41001
(`dms.rs:11-15`). `dms open` builds the event by hand because
`buzz_sdk::build_dm_open` accepts no `d` tag (`dms.rs:57-59` comment), and then
prefers the relay-assigned `channel_id` from the `response:{…}` message over the
locally generated UUID (`dms.rs:74-84`). Actual DM *messages* are sent with
`messages send --channel <dm-uuid>` as plaintext kind 9 — `grep -rn
'nip44\|nip04\|encrypt\|gift_wrap' crates/buzz-cli/src` matches only
`agent_management.rs` (observer frames), so nothing in this module encrypts DM
content. Kind 1059 gift wrap (`kind.rs:60`) is never used here.

#### Note carry-forward semantics

`build_set_event` (`notes.rs:418-469`) implements a ratified omit-vs-clear matrix:
`None` carries the prior value (usage error for `title` on first publish, since
NIP-23 requires it), `Some("")` clears, `Some(v)` sets; `tags: None` carries,
`Some(&[])` clears, `Some(slice)` replaces wholesale (not merged);
`published_at` is preserved from the prior event or stamped with `now` on first
publish. Ten tests cover the matrix (`notes.rs:1057-1229`), including a malformed
prior `published_at` re-stamping to `now` (`notes.rs:1231-1240`).

Slug rules: 1–80 chars, `[a-z0-9._-]` only, lowercase enforced to avoid
`Dco-Recipe` vs `dco-recipe` ambiguity (`notes.rs:50-70`, six tests at
`notes.rs:826-857`).

`notes get --name` ambiguity: >1 author for a slug is an error listing candidates
unless `--latest` picks the newest (`notes.rs:638-652`). The candidate scan is
capped at 50 events (`notes.rs:191`), so ambiguity detection is itself bounded.

#### Author resolution — two incompatible implementations

| Behaviour | `messages.rs:394-439` | `notes.rs:204-252` |
|-----------|----------------------|--------------------|
| `"me"` accepted | no | yes (`notes.rs:205-207`) |
| 64-hex accepted | yes, lowercased (`:396-398`) | yes (`:208-211`) |
| `npub1…` accepted | yes (`:399-404`) | **no** |
| name search | kind 0 + NIP-50 `search`, limit 100 | identical filter (`:213-217`) |
| match rule | exact case-insensitive on `display_name` or `name` (`:441-467`) | same rule, re-implemented inline (`:220-235`) |
| ambiguity error | lists up to 5 candidates + "… and N more" (`:425-437`) | reports only the count (`:248-250`) |
| dedupe of duplicate kind:0 rows | yes (`:465`) | no |

So `buzz messages search --author npub1…` works while `buzz notes ls --author
npub1…` is treated as a display name and almost certainly fails. `notes ls` also
gives `--author all` a special meaning (skip the author filter,
`notes.rs:679-683`) that `messages search` has no equivalent for.

#### Mention resolution rules

`resolve_content_mentions` (`messages.rs:128-201`) only runs when the body
contains `@` (`:133-135`), resolves against the channel's kind-39002 member list
plus those members' kind-0 profiles, matches multi-word display names first then
single words (`extract_at_mentions_with_known`, `:191-193`), and returns an empty
vec on **any** I/O or parse failure so auto-tagging never blocks a send
(`:124-127` doc, `:144-147`, `:155-158`). Consequences: `@name` for a
non-member is silently unresolved, and a relay hiccup silently drops all
mentions from an otherwise successful message.

NIP-27 `nostr:npub1…` URIs are extracted separately after stripping code regions
and merged under `MENTION_CAP` (`messages.rs:526-531`). Mentions are computed
from the author-written body only, never the appended media markdown
(`messages.rs:523-525` comment). The SDK dedupes and enforces the cap
(`builders.rs:186-198`).

#### Rules enforced only by comment or convention

| Rule | Where stated | Enforcement |
|------|-------------|-------------|
| add-policy allowlist covers only this CLI path | `channels.rs:1015-1020` | none relay-side, by team decision |
| callers must not add `auth` tags manually | `client.rs:585-587` | actually enforced post-signing (`client.rs:598-610`) |
| template members added sequentially to avoid LWW races | `channels.rs:741-743` | code shape only |
| kind:5 for notes must carry no `e` tag | `notes.rs:704-711` | one unit test (`notes.rs:1288-1295`) |
| `emoji` 30030 relay keeps only latest per `(pubkey,d)` | `emoji.rs:105` | relies on relay; client takes `events.last()` unsorted |
| `--content -` is the escape hatch for shell-metacharacter-heavy text | `messages.rs:484-487` | convention |

#### Test coverage — business rules

Covered: TTL validation (3 tests, `channels.rs:1276-1290`), add-policy env gate
(4 tests, `channels.rs:1294-1382`, one through the real
`cmd_set_add_policy`), roster cardinality (6 tests, `channels.rs:1386-1466`),
archive filter + fail-open (6 tests, `channels.rs:1470-1597`), warning wiring at
both observable boundaries (3 tests, `channels.rs:1607-1683`), NIP-10 root
derivation (8 tests, `messages.rs:781-861`), author name matching (4 tests,
`messages.rs:1112-1163`), social list kind validation and d-tag requirement
(5 tests, `social.rs:239-283`), `validate_get_args` matrix (4 tests,
`notes.rs:1298-1330`), note carry-forward (10 tests, `notes.rs:1057-1229`), emoji
union LWW (2 tests, `emoji.rs:330-388`), template lookup and case-insensitivity
(5 tests, `channel_templates.rs:137-197`).

Uncovered business rules, each verified by grepping every `#[cfg(test)]` block in
the ten files for the function name with zero hits: the message-kind gate
(`messages.rs:540-572`), the 45001-drops-`--reply-to` behaviour, the branch-pair
rule (`messages.rs:601-608`), `--kinds` parse fallback (`messages.rs:283-287`),
role mapping (`channels.rs:964-976`), channel type/visibility gates
(`channels.rs:283-298`), the visibility post-filter (`channels.rs:66-95`), feed
type validation (`feed.rs:31-39`), the DM 1–8 bound (`dms.rs:52-54`), reaction
emoji matching (`reactions.rs:60-73`), reaction grouping/counting
(`reactions.rs:96-124`), emoji merge-vs-replace (`emoji.rs:281-296`), and the
LWW `duplicate:` handling in `cmd_set` (`notes.rs:556-566`).

One test re-implements a production rule rather than calling it:
`check_allowed_channel_add_policy` (`channels.rs:1296-1307`) is a verbatim copy of
the parsing/matching logic at `channels.rs:1022-1033`, so the three tests using it
(`channels.rs:1310-1343`) would stay green if the production gate changed. The
file acknowledges the gap and adds `set_add_policy_env_gate_rejects_disallowed_via_full_path`
(`channels.rs:1362-1382`) to cover the real path. A second, benign instance:
`cli_pipeline_resolves_multiword_display_names` (`messages.rs:1017-1061`)
re-implements the profile-parsing loop from `resolve_content_mentions` inline
("Simulate the single-parse pipeline", `messages.rs:1026`), so a change to the
real loop would not fail it.


## Module: buzz-cli — repo, agent, memory & moderation commands (`crates/buzz-cli/src/commands`)
### Aspect: Business Rules

#### Identity-perspective rules (`mem`)

`resolve_reader` (`mem.rs:54-79`) produces the `(agent, owner, their_pubkey)` triple that
drives both the query filter and the NIP-44 decrypt key:

| Flags | agent | owner | NIP-44 counterparty | Site |
|---|---|---|---|---|
| neither | CLI identity | from `--owner` or `BUZZ_AUTH_TAG` | owner | `mem.rs:76-78` |
| `--agent <pk>` | the supplied pubkey | CLI identity | the supplied agent | `mem.rs:63-74` |
| both | **rejected** | — | — | `mem.rs:56-60` |
| `--agent == CLI identity` | **rejected** | — | — | `mem.rs:67-72` |

Rules enforced:

- R1. `--owner` and `--agent` are mutually exclusive on read commands (`mem.rs:56-60`).
  Tested: `resolve_reader_rejects_owner_with_agent_flag` (`mem.rs:821`).
- R2. `--agent` must differ from the CLI identity (`mem.rs:67-72`). Tested:
  `resolve_reader_rejects_agent_flag_matching_cli_identity` (`mem.rs:837`).
- R3. Owner resolution order is `--owner` flag first, then the `auth_tag` owner slot
  (index 1), else a `Usage` error (`resolve_owner`, `mem.rs:33-49`).
- R4. Write commands (`set`, `patch`, `rm`) accept `--owner` but **not** `--agent`
  (`lib.rs:1571-1622` defines no `agent` field on `Set`/`Patch`/`Rm`), so owner-side
  recovery is read-only by construction.

`fetch_head` picks the ECDH counterparty by comparing the CLI identity to the agent
(`mem.rs:143-147`) — the conversation key is symmetric per `engram.rs:134-138`, so both
perspectives derive the same `d` tag and the same decrypt key.

#### NIP-33 LWW conflict detection and exit code 5

Two independent mechanisms, both landing on `CliError::Conflict` → exit 5.

**Mechanism A — relay-reported domination.** `submit_engram` (`mem.rs:93-105`) parses the
`POST /events` reply and applies three rules in order:

1. `accepted` missing or false → `Other` (exit 4), message passed through (`mem.rs:97-104`).
2. `accepted == true` **and** `message` starts with `"duplicate:"` or equals `"duplicate"`
   → `Conflict` (`mem.rs:100-104`).
3. otherwise → success.

`repos.rs`'s `validate_write_response` (`repos.rs:173-193`) is a byte-for-byte re-derivation
of the same rule with the same two message forms and a repo-flavoured error string
(`repos.rs:154-158`). It is not shared code — see Integrations.

The relay side that produces the `"duplicate:"` message is
`crates/buzz-relay/src/handlers/ingest.rs:2451-2457` (`was_inserted == false`), and for
NIP-33 kinds `was_inserted` is false exactly when the incoming `(created_at, id)` tuple is
dominated by the stored head:
`created_at < accepted_ts || (created_at == accepted_ts && incoming_id >= accepted_id)`
(`crates/buzz-db/src/lib.rs:3719-3735`). So the CLI's interpretation is faithful for a
genuinely-stale write.

**Important caveat the code does not handle.** The identical `"duplicate:"` reply is also
returned for a byte-identical *re-submission* of an event that already landed (same event id
⇒ `incoming_id >= accepted_id` holds trivially). `client.rs`'s `submit_stored_event` retries
the **same serialized event bytes** on a transient failure (`client.rs:1030-1050`: `body` is
cloned per attempt, only the NIP-98 wrapper is re-signed). If attempt 1 stores the event but
its response body is lost, attempt 2 receives `"duplicate:"` and the CLI reports exit 5 —
a *false* conflict on a write that actually succeeded. There is no test for this path;
`repos.rs:628`'s test only pins the string-matching rule, not the retry interaction.

**Mechanism B — client-side base-hash gate.** `mem patch` compares `sha256_hex(current)`
against `--base-hash` (case-normalized) and raises `Conflict` on mismatch
(`mem.rs:592-599`). This is the only optimistic-concurrency check in the group that fires
before any write is attempted.

#### Monotonic timestamp rules

- Engram writes compute `created_at = max(now, prior_head + 1)` via
  `engram::monotonic_created_at` (`engram.rs:588-593`), called at `mem.rs:353`,
  `mem.rs:677-678`, `mem.rs:722`. Each write therefore performs a read of the current head
  first (`mem.rs:351`, `mem.rs:590`, `mem.rs:720`) — the write path is always read-then-write.
- Repo announcement updates use a **different** rule: `existing.created_at + 1` with
  `checked_add`, deliberately *not* wall-clock (`repos.rs:132-136`). The comment at
  `repos.rs:129-130` states the reason: wall-clock would let a delayed writer leapfrog an
  intervening update and erase metadata. Overflow is an `Other` error, not a panic.
  Tested: `protection_update_preserves_metadata_and_replaces_only_matching_pattern` asserts
  `created_at == 101` from an input of `100` (`repos.rs:445`).

#### Memory value and patch rules

| Rule | Enforcement | Site | Test |
|---|---|---|---|
| slug grammar: `core` or `mem/<seg>(/<seg>)*`, ≤ 255 bytes, segments ≤ 64 bytes of `[a-z0-9][a-z0-9_-]*` | `engram::normalize_slug` → `validate_slug` | called at `mem.rs:279`, `:315`, `:510`, `:539`, `:707`; rules at `engram.rs:67-112` | in `buzz-core` |
| a bare slug is auto-prefixed with `mem/` | `normalize_slug` | `engram.rs:123-131` | in `buzz-core` |
| stdin read is bounded to `NIP44_PLAINTEXT_MAX + 1` (65 536 B) | explicit `.take()` | `mem.rs:322-325`, `mem.rs:576` | none |
| value over 65 535 B rejected | length check after read | `mem.rs:326-329` (set), `mem.rs:630-636` (patch result) | none |
| empty **stdin** value rejected unless `--allow-empty` | `mem.rs:331-338` | — | none |
| a literal `""` positional value is **accepted** | the guard only runs on the `-` branch (`mem.rs:319`) | documented at `mem.rs:309-311` | none |
| empty patch from stdin rejected unconditionally (no `--allow-empty` escape) | `mem.rs:586-591` | — | none |
| `--base-hash` required unless `--no-base-hash`; the two are mutually exclusive | `mem.rs:545-561` | — | none |
| `--base-hash` must be exactly 64 ASCII hex chars | `mem.rs:562-568` | — | none |
| multi-file patch rejected | count of `"--- "` lines > 1 | `mem.rs:600-608` | `multi_file_header_count` (`mem.rs:1035`) — but the test only exercises the counting expression, not `cmd_patch` |
| hunks must apply at their declared line numbers, no fuzz, no slide | `verify_hunks_at_declared_position` | `mem.rs:400-482`, invoked `mem.rs:611-618` | 5 tests, `mem.rs:938`-`mem.rs:1033` |
| no-context insertion into a non-empty value refused | `mem.rs:435-443` | documented limitation, "failure mode is rejection, not corruption" | none (the accepted empty-file case is tested at `mem.rs:968`) |
| empty patch **result** rejected unless `--allow-empty` | `mem.rs:637-643` | — | none |
| `mem rm core` refused; operator told to `mem set core ''` | `mem.rs:711-716` | rationale at `mem.rs:701-704`: NIP-AE defines tombstones only for memory entries | none |
| `mem ls` excludes `core` and tombstones | `mem.rs:238-250` | per spec, comment at `mem.rs:238` | none |
| listings sorted by slug | `mem.rs:260` | — | none |
| a corrupt event must not deny-of-service the whole listing | `parse_events` skips undeserializable entries (`mem.rs:114-133`); signature/decrypt failures `continue` (`mem.rs:164-172`, `mem.rs:207-231`) | — | none |

`mem get` and `mem hash` funnel through `fetch_value` (`mem.rs:484-506`), which maps absent
head, absent body, and tombstone all to `NotFound` and unwraps `Body::Core`'s `profile` as
if it were a value (`mem.rs:503`) — so `mem hash core` hashes the profile text.

#### Repo id and protection-rule semantics

- `validate_repo_id` (`validate.rs:40-61`) requires 1-64 chars of `[A-Za-z0-9._-]`, forbids a
  leading `.` and any `..`. Applied at `repos.rs:207`, `repos.rs:236`, `repos.rs:288`, plus
  every `pr`/`patches`/`issues` command that names a repo. Nine tests in `validate.rs:410-449`.
- `repos protect *` operates **only on the caller's own announcement**: the head fetch pins
  `authors: [self]` (`repos.rs:22`) and a miss is `NotFound` (`repos.rs:287-293`). There is
  no `--owner` escape hatch, so the CLI cannot even attempt to edit someone else's repo.
- `protect set` requires **at least one** constraint: `build_protection_tag` assembles the
  rule words then round-trips them through `parse_protection_tag`, which rejects a
  pattern-only tag (`repos.rs:60-85`; `git_perms.rs:280`). Tested:
  `protection_set_requires_at_least_one_rule` (`repos.rs:521`).
- `protect set` is a **full replacement** for the exact ref pattern: every existing
  `buzz-protect` tag with the same pattern is dropped and the new one appended
  (`repos.rs:108-116`). Any constraint omitted from the command is therefore removed —
  stated in `crates/buzz-cli/README.md:80-82` and pinned by the `count == 1` assertion at
  `repos.rs:474-486`.
- Non-protection tags are **preserved verbatim** across the rewrite, including unknown
  future metadata; only `auth` and the matching `buzz-protect` are filtered
  (`repos.rs:103-106`). Tested for `buzz-channel` and a synthetic `future-metadata` tag at
  `repos.rs:456-465`.
- The update **fails closed** if the stored announcement already contains a malformed
  protection rule: `parse_protection_tags` over the full rebuilt tag set must succeed
  (`repos.rs:118-131`). Tested: `protection_update_rejects_malformed_existing_rules`
  (`repos.rs:526`).
- The 50-rules-per-repo ceiling (`git_perms.rs:19`, `MAX_PROTECTION_RULES`) is enforced
  transitively by the same `parse_protection_tags` call — the CLI never counts rules itself.
  Tested at the boundary: `protection_update_enforces_repository_rule_limit` builds 50 rules
  and asserts the 51st is rejected (`repos.rs:551-573`).
- `protect list` deliberately does **not** fail closed: it reports `unknown_rules` and a
  string `validation_error` so an owner can see and repair a broken rule
  (`repos.rs:141-171`). Two tests, `repos.rs:575` and `repos.rs:601`.
- `protect remove` validates the ref pattern (`RefPattern::parse`, `repos.rs:331-332`), then
  requires an existing rule for that exact pattern before writing (`repos.rs:334-338`) —
  removal is not idempotent by design.
- Ref-pattern grammar (must start with `refs/`, ≤ 256 chars, ≤ 3 wildcards, `**` only as the
  last segment, no partial globs like `v*`) lives in `git_perms.rs:80-125` and is reached
  from the CLI only through `parse_protection_tag`/`RefPattern::parse`.

#### PR / patch / issue state transitions

- Status words map to kinds through one shared function, `parse_status`
  (`patches.rs:194-205`): `open`→1630, `merged`|`resolved`→1631, `closed`→1632,
  `draft`→1633. `merged` and `resolved` are explicit synonyms for the same kind — the
  comment at `patches.rs:190-193` justifies this as NIP-34 using different words for patches
  vs issues over one status kind. Tested: `parse_status_accepts_known_words` (`patches.rs:304`),
  `parse_status_rejects_unknown_word` (`patches.rs:319`).
- clap narrows the accepted words **per command** before `parse_status` ever runs:
  `patches status` allows `open|merged|closed|draft` (`lib.rs:1263`), `pr status` the same
  (`lib.rs:1413`), `issues status` allows `open|resolved|closed|draft` (`lib.rs:1490`). So
  `patches status --status resolved` is rejected by clap even though `parse_status` accepts
  it. That divergence is a convention enforced only by the clap `value_parser` lists, not by
  any shared constant.
- `--repo-owner` and `--repo-id` must be given **together** on all three status commands.
  Enforced twice: clap `requires` (`lib.rs:1249-1254`, `lib.rs:1420-1425`, `lib.rs:1497-1502`)
  and a defensive `match` arm returning `Usage` (`patches.rs:154-158`, `pr.rs:192-196`,
  `issues.rs:121-125`). The `match` arm is unreachable through the CLI but protects the
  library entry points.
- Recipient assembly rule, replicated in all three: seed with the repo owner when known,
  then append each `--to` after `validate_hex64`, skipping duplicates
  (`patches.rs:170-178`, `pr.rs:186-194`, `issues.rs:127-135`). Note the seeded repo owner
  is **not** re-validated at that point (it was validated earlier at `patches.rs:150`),
  and the dedup is `Vec::contains` — O(n²) but n is tiny.
- `pr status` hardcodes `accepted_revision_root: None` and empty `applied_patches` /
  `applied_as_commits` (`pr.rs:199-207`), so the `--revision` and `--q` affordances that
  `patches status` exposes are simply unavailable for PRs. `issues status` additionally
  hardcodes `merge_commit: None` (`issues.rs:141`).
- `--revision`, `--q`, `--merge-commit`, `--applied-as-commit` are documented as
  "status=merged only" (`lib.rs:1256`, `:1273`, `:1276`, `:1279`) but **nothing enforces
  that**: `cmd_patch_status` passes them into `GitStatusMeta` regardless of the resolved
  status (`patches.rs:180-188`). A convention enforced only by help text.
- `--root` / `--root-revision` on `patches send` are two independent booleans
  (`patches.rs:37-38`) with no mutual-exclusion check in the CLI and no clap `conflicts_with`
  (`lib.rs:1215-1221`); whatever the SDK does with both set is not constrained here.
- `--committer` must be exactly four `|`-separated fields (`parse_committer`,
  `patches.rs:58-71`). Fields are **not** individually validated — the timestamp and tz
  offset are passed through as strings. Tested for arity only (`patches.rs:284`, `:298`).

#### Workflow trigger and approval rules

- `workflows create` generates a fresh v4 UUID client-side (`workflows.rs:107`) but prefers
  a relay-supplied `workflow_id` from a `response:{…}` message when present
  (`workflows.rs:111-112`, via `extract_relay_response_field`, `client.rs:1407-1418`).
- `workflows trigger --inputs` must parse as JSON **and** be an object
  (`workflows.rs:163-169`). When present, the SDK builder is bypassed and the event is
  hand-assembled with only a `d` tag (`workflows.rs:170-181`) — which means the 64 KiB
  `check_content` cap that `build_workflow_def` applies (`builders.rs:1468`) is **not**
  applied to `--inputs`. Uncapped input content on this branch.
- `workflows approve` sends `hex(SHA256(token))` as the `d` tag, never the raw token
  (`workflows.rs:205`). The comment at `workflows.rs:204` states the relay expects the hash.
  The raw token is validated as a UUID first (`workflows.rs:196`).
- `--approved` defaults to `true` with `ArgAction::Set` (`lib.rs:909`), so a bare
  `workflows approve --token X` grants. Grant vs deny selects 46030 vs 46031 inside the SDK
  (`builders.rs:1532-1536`).
- `workflows runs` clamps `--limit` to `min(limit.unwrap_or(20), 100)` (`workflows.rs:72`) —
  the only client-side limit clamp in the entire group.
- `workflows runs` is a **known-dead read path**: its own doc comment says the relay does not
  emit 46001-46003 (`workflows.rs:62-65`). Verified: `grep -rn '46001'` across `crates/`
  finds only kind-registry entries, exclusion comments
  (`crates/buzz-db/src/feed.rs:232`, `crates/buzz-workflow/src/lib.rs:266`), a name-mapping
  arm (`crates/buzz-relay/src/handlers/event.rs:48`), and this CLI query — no producer.
- `workflows get` filters on `#d` with **no `authors` constraint** (`workflows.rs:40-43`) and
  then takes `events.first()` (`workflows.rs:47`). Two different authors can publish kind
  30620 with the same `d` value; whichever the relay returns first wins. `workflows update`
  and `delete`, by contrast, are author-scoped implicitly — `build_workflow_delete` embeds
  the caller's own pubkey in the `a` coordinate (`workflows.rs:143`, `builders.rs:1502-1506`).
- `workflows update` requires `--channel` in addition to `--workflow` (`lib.rs:867-877`)
  because the `h` tag is re-emitted on every 30620 write (`builders.rs:1487-1490`).

#### Moderation action semantics

| Rule | Site |
|---|---|
| `--expires-in` is converted to an absolute unix second via `Timestamp::now() + secs`; `--expires-at` is used as-is; `--expires-in` wins if both were somehow set | `resolve_expiry`, `moderation.rs:26-32` |
| the two flags are mutually exclusive (clap only) | `lib.rs:1697`, `lib.rs:1723` — `conflicts_with = "expires_at"` on `expires_in` |
| `ban` with no expiry is permanent | `moderation.rs:39` passes `None` through |
| `timeout` **requires** an expiry — this is the one duration rule the CLI enforces itself | `moderation.rs:69-71` |
| `resolve --status` must be `resolved`\|`dismissed` and `--action` one of six words | **not** in `moderation.rs`; enforced in `buzz_sdk::build_moderation_resolve_report` (`builders.rs:1661-1676`). The CLI passes both strings through unvalidated (`moderation.rs:97`) |
| the community is the relay host; moderation commands carry no channel scope | module doc `moderation.rs:16-18`, mirrored in `lib.rs:1638-1643` |
| authorization (owner/admin) is entirely the relay's job | module doc `moderation.rs:6-8`; nothing in `moderation.rs` checks a local role |
| `reports --limit` / `audit --limit` default to 50 (`lib.rs:1663`, `lib.rs:1768`) and are `i64` with **no client-side validation** | `moderation.rs:105-117`, `moderation.rs:125-130`; the relay clamps to `1..=500` and maps non-positive to 500 (`crates/buzz-relay/src/api/bridge.rs:2084-2088`) |
| `--reason` on ban/timeout/resolve is uncapped and unchecked for control characters | `builders.rs:1607-1609` pushes it straight into a tag; contrast `check_reason`'s 64-byte + control-char rule applied to NIP-IA reasons (`builders.rs:1706-1719`) |

`resolve_expiry`'s `Timestamp::now().as_secs() + secs` (`moderation.rs:29`) is unchecked
`u64` addition. A large `--expires-in` overflows: panic in a debug build, silent wrap to a
tiny timestamp in release — which would produce an already-expired ban. Every other
timestamp arithmetic in the group uses `checked_add` (`repos.rs:134`) or
`saturating_add` (`engram.rs:590`). No test covers `resolve_expiry`.

#### Agent archive / draft rules

- Draft commands **require** `BUZZ_AUTH_TAG`: `require_owner` (`agents.rs:158-166`) returns
  `Auth` (exit 3) when absent. Archive/unarchive do not require it.
- Drafts are advisory only. The output injects `"saved": false` and a message stating nothing
  changes until the owner saves (`agents.rs:41-45`, `agents.rs:84-88`). Payload is NIP-44
  encrypted to the owner (`agent_management.rs:107-108`).
- Draft field caps: display name ≤ 120 chars, system prompt ≤ 20 000 chars, channel ≤ 128
  chars, other optional fields ≤ 300 chars — all measured in **chars, not bytes**, and all
  trimmed first (`agent_management.rs:11-12`, `:70-84`). Channel must parse as a UUID
  (`agent_management.rs:130-131`). `draft-update` requires at least one field to change
  (`agent_management.rs:170-179`); `--respond-to` must be `owner-only` or `anyone`
  (`agent_management.rs:147-153`). Tested: `update_requires_a_change`
  (`agent_management.rs:239`), `create_rejects_invalid_channel` (`agent_management.rs:261`).
- The NIP-OA `auth` tag on 9035/9036 is resolved by `resolve_auth` (`agents.rs:172-204`):
  self-path (`target == signer`, case-insensitive) → `None`; otherwise fetch the target's
  kind:0 and extract. Query or network failure is an **error**, not a silent downgrade
  (`agents.rs:181-183`) — the comment at `agents.rs:167-170` explains that degrading to bare
  would make the relay's rejection misleading.
- `extract_owner_auth_tag` (`agents.rs:206-248`) applies a **set-level** rule: there must be
  exactly one `auth`-labelled tag on the kind:0 (`agents.rs:216-219`). A structurally valid,
  owner-matching tag sitting next to a second `auth`-labelled tag — duplicate or malformed —
  yields `None` (bare). Owner match is case-insensitive; owner must be 64 hex, sig 128 hex.
  Fourteen tests, `agents.rs:387`-`agents.rs:503`, including the two discriminating
  set-level cases at `agents.rs:483` and `agents.rs:494`.
- Archive/unarchive use `sign_event_unchecked` (`agents.rs:113`, `agents.rs:142`), bypassing
  the "exactly one auth tag" invariant that `sign_event` enforces (`client.rs:597-609`).
  Rationale documented at `client.rs:735-742`: the 9035 `auth` tag is a content-level
  attestation about the *target*, not this client's membership delegation, so injecting
  `self.auth_tag` would either double up or drop the caller's attestation.
- The 13535 snapshot is a strict tri-state (`fetch_archived_snapshot`, `agents.rs:270-305`):
  no events → `Ok(vec![])`; a fully-verified event → `Ok(pubkeys)`; any trust failure →
  `Err`. Verification (`verify_archived_event`, `agents.rs:320-372`) requires, in order:
  kind == 13535; author == the NIP-11 `self` pubkey; **exactly one** NIP-70 `-` tag with
  arity exactly 1 (a valid `["-"]` alongside a malformed `["-","extra"]` poisons the
  snapshot — `agents.rs:335-352`, tested at `agents.rs:665`); then `event.verify()`.
  Non-hex or short `p` values are dropped silently rather than failing the whole snapshot
  (`agents.rs:353-366`, tested at `agents.rs:686` and `agents.rs:704`).
- The NIP-11 `self` field must be 64 hex and is lowercased before comparison
  (`normalize_relay_self_hex`, `agents.rs:250-258`) — without this an uppercase `self` would
  never match the always-lowercase event author. Tested at `agents.rs:527`, `:534`, `:539`,
  and end-to-end at `agents.rs:548`.
- `agents archived` treats a trust failure as fatal; the `--template` roster resolver in
  `channels.rs` fails *open* on the same failure. The split is documented at
  `agents.rs:255-269` and `agents.rs:306-309`, with the rationale living in
  `channels.rs:526`'s doc comment.

#### Users rules

- `--name` and `--pubkey` are mutually exclusive (`users.rs:16-20`).
- `--pubkey` accepts at most 200 values (`users.rs:35-37`). **`users presence` has no such
  cap** — it splits an arbitrary CSV and sends every entry as an author (`users.rs:248-256`).
- `--name` must be non-blank after trim (`users.rs:85-87`); the relay's NIP-50 result set is
  additionally narrowed client-side by a case-insensitive substring match on `display_name`
  or `name` (`users.rs:120-134`). The doc comment at `users.rs:78-79` states an empty array is
  returned when the relay does not implement NIP-50.
- `users set-profile` requires at least one field (`users.rs:157-161`) and is
  read-merge-write: caller values win, absent ones fall back to the current kind:0 content,
  with `display_name` falling back to `name` (`users.rs:167-179`). The `name` (username)
  field is hardcoded to `None` on write (`users.rs:200`), so a merge that promoted `name` to
  `display_name` silently drops the original `name`.
- `users presence` prefers a `p`-tag subject over the event author (`presence_subject`,
  `users.rs:279-289`), because the relay may synthesize relay-authored presence snapshots.
  Three tests at `users.rs:343`, `:349`, `:355`.
- Kind 20001 is ephemeral, so `set-presence` must go over WebSocket, not `POST /events`
  (`users.rs:292-296` doc, `users.rs:301`).

#### Pack rules

`pack validate` (`pack.rs:15-49`): path must exist and be a directory (`pack.rs:17-22`);
diagnostics print to stderr; errors → `Usage` (exit 1); warnings only → exit 0 with
`Valid (with warnings).`; clean → `Valid.`. The doc comment at `pack.rs:11-13` matches the
implementation. `pack inspect` (`pack.rs:52-150`) applies the same path checks then
`resolve_pack`, and truncates the system-prompt preview at 77 **chars** with an ellipsis
(`pack.rs:143-148`) while reporting the length in **bytes** (`pack.rs:150`) — a mixed unit
in one output line.

#### Pagination and limit defaults

The group sets almost no limits of its own; effective bounds come from the relay.

| Command | Client limit | Server-side outcome |
|---|---|---|
| `mem ls` | `5000` (`mem.rs:201`) | clamped to `MAX_HISTORICAL_LIMIT` 2000 (`crates/buzz-relay/src/handlers/req.rs:25`, applied `:879-882`), then to 1000 by `max_limit.unwrap_or(1000)` in `crates/buzz-db/src/event.rs:346-347`. Effective cap ≈ 1000 rows, silently |
| `mem get`/`hash`/`patch` | `16` (`mem.rs:155`) | fine |
| `repos get`, `workflows list`, `workflows get`, `pr get`/`list`, `patches get`/`list`, `issues get`/`list` (without `--limit`) | none | relay default = `MAX_HISTORICAL_LIMIT` (2000) then DB clamp 1000 |
| `repos list`, `pr list`, `patches list`, `issues list` with `--limit` | passed through, unvalidated | relay clamps |
| `workflows runs` | `min(limit.unwrap_or(20), 100)` (`workflows.rs:72`) | fine |
| `users get` | `authors.len()` (`users.rs:44`) | fine |
| `users get --name` | `100` (`users.rs:93`) | fine |
| `users presence` | `pubkeys.len()` (`users.rs:260`) — `0` when the CSV is blank, which yields an empty result rather than an unbounded scan | fine |
| `moderation reports`/`audit` | 50 default (`lib.rs:1663`, `:1768`) | relay clamps `1..=500`; a non-positive value becomes 500 |

No command in this group uses `BuzzClient::query_paginated` or `query_all`
(`client.rs:715-729`): `grep -n 'query_paginated\|query_all' ` across the eleven files
returns zero matches. Every read is a single unpaginated `POST /query`, so a result set
larger than the server clamp is silently truncated with no signal to the caller. `mem ls` is
the clearest exposure: it *asks* for 5000 and can receive at most about 1000.

