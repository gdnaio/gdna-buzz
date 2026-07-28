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
`ingest.rs:1427-2505`

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
`ingest.rs:2434-2441`

```rust
if crate::handlers::side_effects::is_side_effect_kind(kind_u32) {
    if let Err(e) = handle_side_effects(tenant, kind_u32, &event, state).await {
        warn!(event_id = %event_id_hex, kind = kind_u32, "Side effect failed: {e}");
    }
}
```

The event is already committed; `dispatch_persistent_event` runs unconditionally 48 lines
later (`:2489`). So a rejected `add_member`, a failed `update_channel`, or a failed
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
`ingest.rs:1808-1818` (9030–9033), `:1579-1588` (9040–9044), `command_executor.rs:36-78`
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
`ingest.rs:198-306`, `:1525-1530`

109 lines mapping 81 kinds to 7 scopes, plus a gate that can never fail because both
transports pass `Scope::all_known()` (`buzz-auth/src/lib.rs:137`, `api/bridge.rs:827`).
`AdminUsers` on 9030–9033 and `AdminChannels` on 9000/9001/9008 look like access control
and are not. Nine tests assert scope values (`ingest.rs:2824`, `:2861`, `:2897`, …),
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
`ingest.rs:2220-2231` computes `thread_meta` for any `requires_h_channel_scope` kind, but
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
- `restricted: ` maps to `Rejected` at `ingest.rs:1456`, `:1507`, `:521` and to `AuthFailed`
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
addressing shape (asserted at `buzz-core/src/kind.rs:709-711`), are validated in the same
scope arm (`ingest.rs:224`), and get only the generic 1024-byte length check
(`ingest.rs:2378`). An empty `d` collapses every team into one slot.

#### D-15 — `SPROUT_MAX_NOT_BEFORE_DELTA` is read per-request and duplicated
`ingest.rs:1299-1302` does `std::env::var(...).parse()` on **every** kind:30300 ingest — the
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
| D-21 | 6 panic sites in production paths, against AGENTS.md's "do not introduce new `unwrap()`/`expect()` in production paths": `ingest.rs:1998`, `:2000`, `:2338`, `:2344`, `side_effects.rs:311`, `:314` | as listed |
| D-22 | `IngestResult.message` is a stringly-typed tagged union (`""` / `duplicate:` / `info:` / `response:{json}` / `{}`). `buzz-cli` string-matches it to derive exit code 5 (`buzz-cli/src/commands/mem.rs:105`, `commands/notes.rs:560`). A typo in either side silently changes CLI exit-code behaviour. | `ingest.rs:166-173` |
| D-23 | `ReactionEventInsertOutcome::Duplicate` returns `accepted: false` (`ingest.rs:2316-2321`) while the generic LWW duplicate returns `accepted: true` (`:2427-2431`). Two duplicate semantics, two response shapes, no documentation of the difference. | as listed |
| D-24 | 8 relay-minted kinds are absent from `is_relay_only_kind` (8000, 8001, 8002, 8003, 13535, 39000, 39001, 39002) plus 40099, so client submission yields the vague `restricted: unknown event kind` rather than `restricted: relay-only kind`. | `buzz-core/src/kind.rs:682-693` |
| D-25 | 26 declared kinds have no ingest path at all: 43001–43006 (jobs), 46001–46012 (workflow execution), 9009, 39003, 41, 1063, 41001, 48001, 49001. Declared, documented, unreachable. | `buzz-core/src/kind.rs:380-446`, `:452`, `:466` |
| D-26 | Workflow `add_reaction` POSTs to `/api/messages/{id}/reactions`, a route the relay never registers (`router.rs` registers 3 `/api/*` paths, none matching). Ingest has a native kind:7 path the action does not use. Any workflow with an `add_reaction` step 404s. | `buzz-workflow/src/executor.rs:883-917` |
| D-27 | `buzz_channels_created_total` is emitted from 5 sites; the argument that they do not double-count is prose, not structure (`side_effects.rs:1670-1679`). | `ingest.rs:2124`; `side_effects.rs:1704`, `:1731`; `command_executor.rs:373`, `:527` |
| D-28 | 40004–40007 (pin/bookmark/schedule/reminder), 48100–48106 (huddles), 40100 (canvas) are stored with only the `h`-tag gate — no target validation, unlike 40003 and 45002 which do check. Nothing prevents a `HUDDLE_ENDED` for a huddle that never started, or a pin of an event in another channel. | `ingest.rs:455-491` |
| D-29 | 30618 (`GIT_REPO_STATE`) is client-submittable with no repo-ownership check, and is also relay-minted (`side_effects.rs:2733`). A client cannot overwrite the relay head (different `d` coordinate owner) but can pollute the kind. | `ingest.rs:294` |
| D-30 | The 9002 `ttl` update path does not apply `ephemeral_ttl_override`, so a client can raise a channel's TTL post-creation to escape the deployment override that 9007 enforces. | `side_effects.rs:1449-1481` vs `ingest.rs:2099` |
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
