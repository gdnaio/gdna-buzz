## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: Features

Five capabilities, split cleanly by whether the relay actually drives them.

| # | Feature | Where it lives | Wired into the relay? |
|---|---|---|---|
| F1 | Trace emission (the seam) | `crates/buzz-relay/src/conformance/mod.rs`, `handlers/ingest.rs`, `handlers/req.rs` | **Yes** — but always into `NoopTracer` (`crates/buzz-relay/src/state.rs:798`) |
| F2 | Coverage-breach guard (`EmitGuard`) | `crates/buzz-relay/src/conformance/mod.rs:344-415` | **Yes** — armed at one seam only (`handlers/ingest.rs:1382-1386`) |
| F3 | Row-community projection | `crates/buzz-relay/src/conformance/mod.rs:186-333` | **Yes** — both read lanes in `req.rs` |
| F4 | Replay checking (`check_trace`) | `crates/buzz-conformance/src/checker.rs:74` | **No** — zero production callers |
| F5 | Test lanes: property tests + golden fixtures | `crates/buzz-conformance/tests/` | Test-only by construction |

---

### F1 — Trace emission

Nine actions in the vocabulary (`src/lib.rs:179-261`); the relay emits **six** of them.

| Action | Relay emit site(s) | Producer |
|---|---|---|
| `SanitizedError` | `handlers/ingest.rs:1410-1418` | terminal-error wrapper, one place for all `Err` returns |
| `AuthCheck` (write path) | `handlers/ingest.rs:1794-1802` | after `check_channel_membership` (`:1785-1787`) |
| `AuthCheck` (read path) | `handlers/req.rs:143-151` → `conformance/mod.rs:148-168` | after `db.is_member` (`req.rs:137-142`), only on the cache-miss branch (`:134-136`) |
| `WriteInsertGlobal` | `handlers/ingest.rs:139-146` (product feedback, called at `:1547`), `:2189-2195` (push lease), `:2480-2483` (trailing dispatch, channel-less arm) | |
| `WriteInsert` | `handlers/ingest.rs:2336-2340` (reaction lane), `:2470-2474` (trailing dispatch) | |
| `WriteDuplicate` | `handlers/ingest.rs:2342-2346` (reaction lane), `:2475-2479` (trailing dispatch) | |
| `ReadMessageRows` | `handlers/req.rs:355-361` → `conformance/mod.rs:265-298` | non-search REQ lane, per filter |
| `ReadByIdRows` | `handlers/req.rs:671-677` → `conformance/mod.rs:300-333` | NIP-50 search refetch lane |
| `ImplBug` | `conformance/mod.rs:404-414` (guard drop), `:284-296` / `:319-331` (projection miss) | |
| `ReadHostFeedRows` | **none** — grep across `crates/buzz-relay/` finds no producer | |

Emission is per-decision, synchronous, and untimed. There is no batching, no sampling, no
async queue; `emit` is `tracer.record(TraceStep::new(action, state))`
(`conformance/mod.rs:127-129`).

**One emit arm is unreachable.** `handlers/ingest.rs:2426-2432` returns early on
`!was_inserted`, so the trailing dispatch's `(Some(ch), false) => WriteDuplicate` arm
(`:2475-2479`) can never execute. Only the reaction lane's `WriteDuplicate` (`:2342`) is
reachable, and only when `buzz_db::ReactionEventInsertOutcome::Inserted` carries
`was_inserted: false` (`:2322-2325`) — the explicit `Duplicate` outcome returns earlier at
`:2315-2321` without emitting.

**Fan-out is deliberately not a feature.** `handlers/event.rs:382-387` documents the decision:
the spec has no fan-out action, and acceptance is already recorded at the ingest seam.

---

### F2 — Coverage-breach guard

`EmitGuard::arm` (`conformance/mod.rs:383-400`) returns a pair: the guard, and a
`CountingTracer`-wrapped `Arc<dyn Tracer>` the caller must use in place of the original. The
wrapper bumps a `Relaxed` `AtomicUsize` on each `record` (`:367-373`); `Drop` emits
`ImplBug { kind }` on the **inner** tracer when the count is still zero (`:403-415`).

Callers never disarm — the design note at `:337-343` is explicit that production paths just
call `record` as before.

**Armed at exactly one seam:** `ingest_event` (`handlers/ingest.rs:1382-1386`, `kind =
"ingest_event_exited_without_trace"`). The REQ path arms nothing; `handle_req` builds a
`trace_state` (`handlers/req.rs:116-118`) but no guard, so a REQ that returns before any read
emits produces no `ImplBug`. Six paths through `ingest_event_inner` return `Ok` without any
emit, so the guard fires `ImplBug` on each:

| Path | Line | Kind(s) |
|---|---|---|
| Command routing | `ingest.rs:1534-1536` | 7 kinds: 30620, 41010, 41011, 41012, 46020, 46030, 46031 (`buzz-core/src/kind.rs:667-678`) |
| NIP-56 report | `ingest.rs:1561-1570` | 1984 (`buzz-core/src/kind.rs:191`) |
| Moderation commands | `ingest.rs:1579-1588` | 5 kinds (`buzz-core/src/kind.rs:240-249`) |
| Relay-admin kinds | `ingest.rs:1808-1816` | 4 kinds (`buzz-core/src/kind.rs:647-655`) |
| NIP-43 leave request | `ingest.rs:1820-1902` | 28936 (`buzz-core/src/kind.rs:268`) |
| Reaction duplicate | `ingest.rs:2315-2321` | 7 (`buzz-core/src/kind.rs:58`) |
| Non-reaction duplicate | `ingest.rs:2426-2432` | all stored kinds |

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
