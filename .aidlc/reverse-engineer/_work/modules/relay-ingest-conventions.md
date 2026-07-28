## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: Conventions

---

### 1. Handler shape

Three distinct shapes coexist, distinguished by return type and error style.

**A. The ingest pipeline** — one linear function, no sub-handlers.

```rust
async fn ingest_event_inner(
    state: &Arc<AppState>,
    tracer: &Arc<dyn buzz_conformance::Tracer>,
    tenant: &TenantContext,
    event: Event,
    auth: IngestAuth,
) -> Result<IngestResult, IngestError>          // ingest.rs:1427
```

Argument order is stable across the group: `tenant` first (or after `state`), then
`state`, then `event`, then auth. Two orderings are actually in use —
`(state, tenant, …)` in `ingest.rs` and `(tenant, state, …)` in `side_effects.rs` and
`command_executor.rs`. No file mixes them internally.

**B. Side-effect handlers** — `anyhow::Result<()>`, private, one per kind.

```rust
async fn handle_put_user(
    tenant: &TenantContext,
    event: &Event,
    state: &Arc<AppState>,
) -> anyhow::Result<()>                          // side_effects.rs:1203
```

All 13 follow this exactly: `handle_kind0_profile` `:1113`, `handle_agent_profile` `:1078`,
`handle_put_user` `:1203`, `handle_remove_user` `:1265`, `handle_edit_metadata` `:1335`,
`handle_delete_event_side_effect` `:1560`, `handle_create_group` `:1660`,
`handle_delete_group` `:1783`, `handle_join_request` `:1835`, `handle_leave_request` `:1913`,
`handle_a_tag_deletion` `:1979`, `handle_standard_deletion_event` `:2108`,
`handle_git_repo_announcement` `:2412`.

**C. Command handlers** — `Result<IngestResult, IngestError>`, taking `&IngestAuth`.

```rust
async fn handle_dm_open(
    tenant: &TenantContext,
    state: &Arc<AppState>,
    event: &Event,
    auth: &IngestAuth,
) -> Result<IngestResult, IngestError>           // command_executor.rs:310
```

All 7 identical (`:310`, `:443`, `:580`, `:653`, `:809`, `:986`, `:1098`). Each follows the
same 6-step comment skeleton — `// 1. Extract`, `// 2. Validate`, `// Persist the command
event`, `// 4. Execute`, `// Commit`, `// 5./6. Side effects` + `// Return response` —
which makes the group readable as a template. `handle_dm_open` `:310-441` is the reference
implementation.

---

### 2. Validation-function conventions

| Convention | Detail |
|---|---|
| Naming | `validate_*` for pre-storage gates. `validate_edit_ownership` `ingest.rs:763`, `validate_forum_vote_target` `:844`, `validate_diff_event` `:896`, `validate_engram_envelope` `:965`, `validate_persona_envelope` `:1027`, `validate_engram_nip44_content` `:1084`, `validate_agent_turn_metric_envelope` `:1151`, `validate_not_before` `:1223`, `validate_event_reminder` `:1252`, `validate_admin_event` `side_effects.rs:259`, `validate_standard_deletion_event` `:179`, `validate_imeta_tags` `imeta.rs:11`, `validate_repo_id` `side_effects.rs:2391`. |
| Error type | Pure/synchronous validators return `Result<(), String>`; the two `&'static str` returners (`validate_not_before`, `validate_event_reminder`) are the only exceptions, and their strings are closed-set wire values the spec pins for client matching (`ingest.rs:1259-1261`). Async DB-touching validators in `side_effects.rs` return `anyhow::Result<()>`. |
| Error prefix | Validators return **bare** messages; the ingest call site adds the prefix: `.map_err(\|e\| IngestError::Rejected(format!("invalid: {e}")))`. 12 sites use this exact line (`ingest.rs:1908`, `:1918`, `:1924`, `:1962`, `:1968`, `:1973`, `:1978`, `:1983`, `:2020`, `:2025`, `:2214`, `:2217`). ⚠ This produces `invalid: restricted: not a channel member` for edit-ownership failures (`ingest.rs:838`), i.e. a double prefix. |
| Predicate naming | `is_*` for classification: `is_global_only_kind` `ingest.rs:379`, `requires_h_channel_scope` `:455`, `is_admin_kind` `side_effects.rs:26`, `is_side_effect_kind` `:35`, `is_local_media_url` `imeta.rs:373`, `is_well_formed_mime` `:340`, `has_e_tag` `side_effects.rs:2300`, `actor_is_channel_owner_or_admin` `:2357`, `author_delete_can_use_self_delete_path` `:2353`. |
| Position | Every validator runs **pre-storage**. The only post-storage logic is `handle_side_effects`, whose failures are non-fatal by design (`ingest.rs:2434-2441`). |

---

### 3. Error-string wire format

A three-token prefix vocabulary, matched by clients:

| Prefix | Meaning | Typical mapping |
|---|---|---|
| `invalid: ` | client-side protocol/data error | `Rejected` → HTTP 400 |
| `restricted: ` | authorization refusal | either `Rejected` or `AuthFailed`; HTTP 400 or 403 |
| `blocked: ` | community ban | `AuthFailed` → 403 |
| `error: ` | server-side failure | mostly `Internal` → 500 |
| `duplicate: ` | idempotent no-op | `Ok` |
| `info: ` | successful non-storage action | `Ok` |
| `response:` | JSON payload follows (command kinds) | `Ok` |
| `forbidden: ` | **command-executor only** | `Rejected` → 400 |

⚠ Three inconsistencies:
1. `forbidden: ` appears only in `command_executor.rs` (`:509`, `:625`, `:711`, `:845`,
   `:975`, `:982`) and always as `Rejected`, so an authorization failure on a command kind
   returns HTTP **400**, while the same class of failure on any other kind returns **403**.
2. `restricted: ` maps to `Rejected` in some places (`ingest.rs:1456`, `:1507`, `:521`) and
   `AuthFailed` in others (`:1513`, `:1521`, `:1526`, `:1726`, `:2012`).
3. `error: database error: {e}` from `check_channel_membership` (`ingest.rs:501`) is
   surfaced as `Rejected` (`:1802`), giving a 400 for a server fault.

The prefix set is not centralised anywhere. `crate::conformance::sanitized_reason_for`
(`ingest.rs:1411`) is the only place that classifies `IngestError` variants into a closed
alphabet, and it is for the trace, not the wire.

---

### 4. Tag access convention

Every tag read in this group goes through `tag.kind().to_string() == "name"` and
`tag.content()`, or through `tag.as_slice()` for positional access. Two families:

| Helper | Semantics | Copies |
|---|---|---|
| first-match extractor | returns the first matching tag's content | `extract_channel_id` `ingest.rs:308`; `extract_h_tag_channel` `side_effects.rs:2237`; `extract_p_tag` `:2251`; `extract_tag_value` `:2325`; `extract_h_tag` `command_executor.rs:250`; `extract_d_tag` `:261`; `extract_e_tag` `:272`; `extract_tag` `:283` |
| all-match collector | `extract_target_event_ids` `side_effects.rs:2304`; `extract_p_tags` `command_executor.rs:235`; `count_e_tags` `ingest.rs:719` |

⚠ **`e`-tag selection direction is inconsistent and load-bearing.** Reactions take the
**last** `e` tag (`.rev()` — `ingest.rs:334`, `:2251`, `side_effects.rs:2192`), per NIP-25.
Edits (`ingest.rs:766`), votes (`:847`), deletion channel derivation (`:1670`), and 9005
(`side_effects.rs:531`) take the **first**. Nothing names or documents the rule; it must be
read off each call site.

⚠ **Duplicated helpers.** `effective_message_author` exists twice with identical bodies —
`ingest.rs:729-761` (`pub(crate)`) and `side_effects.rs:2271-2298` (private). The
`side_effects.rs` copy uses `extract_tag_value(event, "actor")` where the `ingest.rs` copy
inlines the same loop. `side_effects.rs:2195` then reaches back for
`super::ingest::effective_message_author`, so both copies are live in the same file's call
graph. Similarly `extract_channel_id` (`ingest.rs:308`) and `extract_h_tag_channel`
(`side_effects.rs:2237`) are byte-equivalent, and `command_executor.rs:250` has a third
variant returning `Option<String>` instead of `Option<Uuid>`.

---

### 5. How a new kind is added (the actual sequence)

Derived from the code, not from docs — no doc describes this.

1. Add the constant to `crates/buzz-core/src/kind.rs` and to `ALL_KINDS`
   (`kind.rs:490-617`). Add a compile-time range assertion if the kind is
   replaceable/parameterized (`kind.rs:707-744`).
2. Add a match arm to `required_scope_for_kind` (`ingest.rs:198-306`). **Without this the
   kind is rejected with `restricted: unknown event kind`.** This is the real gate.
3. Add it to exactly one of `is_global_only_kind` (`ingest.rs:379-453`) or
   `requires_h_channel_scope` (`:455-491`), or neither if the channel is derived some other
   way. The disjointness test (`:2753-2762`) will catch getting both.
4. If the kind needs pre-storage validation, write a `validate_*` returning
   `Result<(), String>` and call it in the `ingest_event_inner` gauntlet
   (`ingest.rs:1960-2026` is where the per-kind validators cluster).
5. If it needs post-storage effects, add it to `is_side_effect_kind`
   (`side_effects.rs:35-37`) **and** add a `handle_side_effects` arm
   (`:143-176`). Both, or the kind is silently ignored.
6. If it is a transactional command, add it to `buzz_core::kind::is_command_kind`
   (`kind.rs:667-679`), then a `handle_command` arm (`command_executor.rs:66-77`) plus a
   handler following shape C.
7. If it is relay-signed only, add it to `is_relay_only_kind` (`kind.rs:682-693`) so the
   reject message is `restricted: relay-only kind` rather than
   `restricted: unknown event kind`. ⚠ Eight relay-minted kinds skip this step today
   (8000, 8001, 8002, 8003, 13535, 39000, 39001, 39002, 40099) — see features.md §3.
8. Add unit tests to the `ingest.rs` test module asserting scope, global-only, and
   `requires_h` classification. The existing suite has one test per property per kind
   family (e.g. `:2683-2718` for NIP-51 lists, `:2909-2927` for teams/managed agents), and
   `per_kind_scope_allowlist_covers_all_migrated_kinds` (`:2822-2879`) is the running
   checklist — 44 kinds are listed there, out of 81 accepted, so it is not exhaustive.

There is **no** single registry or trait. Adding a kind touches 3–6 disjoint `match`
statements across 2 crates, none of which are exhaustive over `ALL_KINDS`. Nothing fails to
compile if a step is skipped.

---

### 6. Test conventions

| Convention | Detail |
|---|---|
| Location | `#[cfg(test)] mod tests` at the bottom of the file: `ingest.rs:2506`, `side_effects.rs:3266`, `imeta.rs:419`. `command_executor.rs` has none. |
| Style | Pure-function unit tests only. **Zero** `#[tokio::test]`, so no handler with a DB dependency is tested in-file. All 111 tests are synchronous. |
| Builders | `ingest.rs` has a small builder set: `make_dummy_event()` `:3045`, `make_event_with_tags(kind, content, &[&[&str]])` `:3053`, then kind-specific wrappers `make_engram` `:3083`, `make_reminder` `:3283`, `make_persona` `:3421`, `make_agent_turn_metric` `:3541`, plus `fake_nip44_v2()` `:3090` producing a shape-valid 99-byte NIP-44 v2 payload. |
| Assertion style | Property assertions loop over kind arrays with a message naming the kind: `assert!(is_global_only_kind(kind), "kind {kind} must be global-only")` (`ingest.rs:2571`, `:2707`, `:2865`, …). Error assertions match on substrings, not equality: `assert!(err.contains("`p` tag"), "got: {err}")` (`ingest.rs:3111`). |
| Exhaustive properties | One brute-force test over the whole kind space: `global_only_and_channel_scoped_are_disjoint` iterates `0..=65535` (`ingest.rs:2753-2762`). |
| Regression documentation | Regressions carry a doc comment explaining the failure they prevent, e.g. the uppercase-`p` invisibility bug (`ingest.rs:2612-2616`) and the non-base64 replacement bug (`:3169-3174`). |
| Metrics tests | `reject_with_transport_labels_http_and_ws_as_separate_series` (`ingest.rs:3643-3684`) uses `metrics_util::debugging::DebuggingRecorder` + `with_local_recorder` to assert label cardinality. The only metrics test in the group. |
| Conformance tests | `feedback_success_action_satisfies_ingest_emit_guard` (`ingest.rs:2531-2565`) arms a real `EmitGuard` against a `VecTracer` (`:2520-2528`) and asserts exactly one `WriteInsertGlobal`. The pattern exists for one kind only. |
| `side_effects.rs` tests | All 5 cover pure helpers: `delete_tombstone_content` (a `#[cfg(test)]`-only function at `:2363-2391` that duplicates the production tombstone builder at `:1650-1656`), `author_delete_can_use_self_delete_path`, `actor_is_channel_owner_or_admin`. Nothing touches the 418-line `validate_admin_event`. |
| Integration coverage | Behaviour of this group is covered out-of-file in `crates/buzz-test-client/tests/` — e.g. `e2e_human_edit_agent_content.rs:5-6` names `validate_standard_deletion_event` and the `validate_admin_event` 9005 branch as its subjects. Per AGENTS.md these need Postgres + Redis (`just test`). |

---

### 7. Logging conventions

| Convention | Detail |
|---|---|
| Levels | `debug!` for pipeline entry (`ingest.rs:1432`); `info!` for accepted writes and completed side effects (`ingest.rs:2359`, `:2499`); `warn!` for every swallowed side-effect failure (32 sites in `side_effects.rs`); `error!` for genuine bugs (`ingest.rs:1472` spawn panic, 8 sites in `command_executor.rs` for spawned-task failures). |
| Structured fields | `event_id = %hex`, `kind = u32`, `channel = %uuid`, `target = %hex`, `pubkey = %hex`, `error = %e`. Consistent throughout. |
| Sensitive values | Pubkeys are logged as hex — acceptable (they are public). Event content is never logged. Reject reasons that may embed event-controlled data are truncated **at the transport**, not here (`api/bridge.rs:842-851`). |
| Success/failure symmetry | `ingest.rs` logs one `info!` per accepted event (`:2359` reaction, `:2499` generic) but nothing on rejection — rejection telemetry is the `buzz_events_rejected_total` counter, emitted by the transport (`reject_with_transport` `:156`), not by ingest. |

---

### 8. Comment conventions

This group is unusually heavily commented, and the comments carry design decisions rather
than restating code. Recurring patterns:

- **Decision references.** `(OQ1 decision …)` `ingest.rs:1768`; `(E1 within-request
  threading; correctness ruling §4.8)` `:485`; `(§4.8 phase-2 addendum)` `:1742`;
  `(COMMUNITY_MODERATION_PLAN.md §0 decision 4)` `:1589`; `(C5)` double-count analysis
  `side_effects.rs:1670-1679`; `(F9)` `buzz-db/src/thread.rs:114`;
  `(spec line 794)` `ingest.rs:1776`.
- **Load-bearing-ordering markers.** "Ordering is load-bearing" `ingest.rs:2294`;
  "Redis before local fan-out so subscribers on other relay pods receive it too"
  `side_effects.rs:783`; "Listed after the workflow branch so workflow's bespoke deletion
  … takes precedence" `:2051-2056`.
- **Known-limitation blocks.** `ingest.rs:369-377` (stray `h` on the read path);
  `side_effects.rs:952-960` (channel-scoped discovery vs live global subs);
  `:1516-1524` (four sub-second archive toggles);
  `command_executor.rs:92-98` (non-atomic command mutations).
- **Negative rationale** — why something is *not* done: "we intentionally do NOT check
  is_agent_owner for non-members" `side_effects.rs:365-367`; "Not reachable in practice …
  so we don't engineer around it" `:1522-1524`; "diverges from kind:9001 intentionally"
  `:598-600`, `:632-634`.
- **Unreachable-arm annotations.** `RemoveResult::RoleMismatch` is documented as
  "unreachable but exhaustiveness requires it" (`ingest.rs:1875-1876`).

---

### 9. Metrics conventions

Label cardinality is reasoned about explicitly. Fleet-wide counters carry `kind` but
**no** `community`, because `bounded_kind_label` passes through all 10 000 ephemeral kind
values and crossing kind × community "would produce up to millions of series"
(`handlers/event.rs:629-632`). Per-community counters carry `community` but no `kind`.
`author_type` is accepted as a label only because it is 2-valued and "merely doubles the
kind series" (`ingest.rs:1391-1394`). `reject_with_transport`'s `reason` is documented as
"one of a small closed set … bounded, no cardinality risk" (`ingest.rs:150-155`).

---

### 10. AGENTS.md compliance

| Rule | Status |
|---|---|
| No `unsafe` | ✅ 0 occurrences in 8 911 lines |
| No new `unwrap()`/`expect()` in production paths | ❌ 4 `expect()` (`ingest.rs:1998`, `:2000`, `:2338`, `:2344`) + 2 `unwrap()` (`side_effects.rs:311`, `:314`) |
| New public API must have doc comments | ✅ every `pub` item in all four files is documented; `IngestAuth`'s fields are individually documented (`ingest.rs:64-85`) |
| Channels use `h` tags, not `e` tags | ✅ `extract_channel_id` reads only `h` (`ingest.rs:308-319`) |
| Event kinds defined in `buzz-core/src/kind.rs` | ⚠ mostly — `KIND_PUSH_LEASE` is defined in **both** `buzz-core/src/kind.rs:109` and `handlers/push_lease.rs:19`, and `ingest.rs` imports the `push_lease` copy (`ingest.rs:216`, `:451`, `:2156`) rather than the `buzz-core` one. Two sources of truth for one integer. |
| Prefer Nostr events over new HTTP endpoints | ✅ this group adds no HTTP surface |
| Thread counters updated on every reply-insert path | ✅ verified — see data-model.md §7 |
| 1 000-line file ceiling | ❌ not applicable to Rust (guard is JS/mobile only), but `ingest.rs` (3 686) and `side_effects.rs` (3 347) are 3.3–3.7× over the ceiling the repo applies elsewhere — see debt.md D-01 |
