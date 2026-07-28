## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: Conventions

### Naming

**Spec-mirroring is the dominant convention.** Every `TraceAction` variant is named after its
TLA+ action, character-for-character:

| Rust variant | TLA+ action | Spec line |
|---|---|---|
| `WriteInsert` (`src/lib.rs:181`) | `WriteInsert(w)` | `docs/spec/MultiTenantRelay.tla:514` |
| `WriteInsertGlobal` (`:195`) | `WriteInsertGlobal(w)` | `:559` |
| `WriteDuplicate` (`:204`) | `WriteDuplicate(w)` | `:606` |
| `SanitizedError` (`:214`) | `SanitizedError(w)` | `:778` |
| `AuthCheck` (`:221`) | `AuthCheck(w)` | `:794` |
| `ReadMessageRows` (`:231`) | `ReadMessageRows(w)` | `:643` |
| `ReadByIdRows` (`:241`) | `ReadByIdRows(w)` | `:681` |
| `ReadHostFeedRows` (`:250`) | `ReadHostFeedRows(w)` | `:703` |

`ImplBug` (`:256`) is the sole variant with no spec counterpart, and the doc comment says so
explicitly (`:193-194`).

**`Inv_*` names appear only in prose, never as Rust identifiers.** The crate references
`Inv_NonInterference` (`src/transitions.rs:53`, `:296`, `src/lib.rs:238`),
`Inv_ReadConfinement` (`src/transitions.rs:54`), `Inv_SanitizedErrors` (`src/lib.rs:125`),
`Inv_AdmissionFence` (`src/transitions.rs:223`) — all inside `///` or `//` comments. No
function or type is named after an invariant; the mapping from invariant to enforcing predicate
lives in comments, not in the type system.

**Mutation IDs (`M1`…`M8`) are a second comment-only vocabulary.** Used at `src/lib.rs:127`
("M6 mutation"), `:190` ("M2/M8 target"), `:238` ("M1/M4/M7"), `src/transitions.rs:218-221`,
`:239-240`, `crates/buzz-relay/src/conformance/mod.rs:18-19`,
`crates/buzz-relay/src/handlers/ingest.rs:1779`. No file in the repo defines what M1–M8 are —
grep for `M1 ` / `M4` / `M7` outside these comment sites finds no legend. The identifiers are
inherited from an external mutation-testing plan ("skill-runtime-formal-compliance", cited at
`Cargo.toml:7`, `src/lib.rs:6`, `src/checker.rs:9`, `tests/proptest_checker.rs:5-6`) that is
not in the repo.

**Label newtypes use a `*Label` suffix** — `CommunityLabel`, `HostLabel`, `ChannelLabel`,
`ActorLabel` (`src/lib.rs:66`, `:100`, `:106`, `:112`) — except `OpaqueId` (`:93`), which
breaks the pattern despite being the same kind of thing.

**Tracer impls use a `*Tracer` suffix** and are named for their sink: `NoopTracer`
(`src/lib.rs:323` and again `crates/buzz-relay/src/conformance/tracers.rs:16`), `JsonlTracer`
(`tracers.rs:30`), `CountingTracer` (`conformance/mod.rs:356`, private), `VecTracer` — declared
twice, independently, as a test-local sink (`conformance/mod.rs:447-456` and
`handlers/ingest.rs:2519-2528`) rather than shared through a test-support module.

**Emitter helpers are `record_*` or `emit*`.** `record_req_authcheck` (`mod.rs:148`),
`record_read_message_rows` (`:265`), `record_read_by_id_rows` (`:300`) all end in the action
they emit; `emit` (`:127`) and `emit_product_feedback_success`
(`handlers/ingest.rs:133`) are the generic and one-off forms. `step` (`mod.rs:121`) is the
odd one out — a bare noun with no caller.

**Trailing-underscore placeholder.** `Verdict_` (`src/transitions.rs:53-56`) uses a trailing
underscore to avoid colliding with the schema's `Verdict`. It has zero uses; the underscore
suffix also keeps it out of `non_camel_case_types` lint range.

---

### Error handling

- **No panics in library code.** `#![deny(unsafe_code)]` (`src/lib.rs:38`) and no `unwrap()` /
  `expect()` anywhere in `src/` — verified by reading all three source files. The one
  `expect` in the relay-side helper is a documented invariant restatement:
  `row.expect("project_row_community returns None only for Some(ch)")`
  (`conformance/mod.rs:248`).
- **`thiserror` for the single error type.** `TransitionError`
  (`src/transitions.rs:60-102`) derives `thiserror::Error` with `#[error(...)]` format strings
  carrying the step index (`:63`, `:75`, `:85`, `:95`).
- **Human-readable detail, machine-readable variant.** Every variant carries
  `detail: String` built with `format!` (`:146-151`, `:155-160`, `:164-169`, `:236-241`,
  `:278`, `:304-309`). The convention documented at `:58-59` — "the string payload is
  human-readable; mechanical consumers should match on the variant" — means offending values
  are only recoverable by parsing prose.
- **Fail-fast, first-error-wins.** `check_step` returns on the first violation and
  `check_trace` propagates with `?` (`src/checker.rs:109`). The test suites are written around
  this constraint; the discipline is spelled out at `tests/proptest_checker.rs:27-33`.
- **Fail-closed defaults.** Empty trace → `CoverageBreach` (`src/checker.rs:75-82`);
  missing projection lookup → `ImplBug`, never a substituted label
  (`conformance/mod.rs:246-253`, rationale `:203-208`).
- **Observability code never breaks the request.** `JsonlTracer::record` recovers from a
  poisoned mutex via `into_inner()` (`tracers.rs:59-63`) and discards write errors
  (`:66-71`); `req.rs` logs a `warn!` and continues with an empty lookup map on DB failure
  (`:347-353`, `:663-669`). The `Tracer` trait returns `()` (`src/lib.rs:317`), so there is no
  error channel to propagate even if a caller wanted one.

---

### Comment style

Unusually heavy — roughly half the crate is doc prose. Three recurring shapes:

1. **"Why not the obvious thing"** blocks. `Cargo.toml:7-24` (independence rule),
   `src/lib.rs:47-63` (why not `buzz_core::CommunityId`), `src/lib.rs:236-239` (why `Vec` not
   `Set`), `conformance/mod.rs:135-145` (why `claimed_community: None` on REQ).
2. **Spec-line citations** inline with each match arm (`src/transitions.rs:172-186`,
   `:188-191`, `:193-198`, `:200-205`, `:211-227`, `:251-257`, `:266-268`, `:272-276`).
3. **Named-reviewer / thread references.** `conformance/mod.rs:37-38` ("held back as additive
   patch for Eva to apply onto Max's req.rs writes — see thread `c882c9b1…`"),
   `tests/replay_fixtures.rs:19-20` ("Eva's review (thread `06aaf3f7…`)"),
   `tests/replay_fixtures.rs:145-152`, `conformance/mod.rs:170-172` ("the (B)
   projection-strategy guard-rail Eva specified"). These leave the code coupled to
   conversations that are not in the repo, and several are now stale (the req.rs patch has
   landed — `handlers/req.rs:334-361`, `:649-677` — while the comment still says "held back",
   as does `TRACE_SCHEMA.md:137`).

---

### Test organization

| Lane | Location | Convention |
|---|---|---|
| Unit | `src/checker.rs:134-337` (`#[cfg(test)] mod tests`) | one passing case + one bite case per failure mode; tiny helpers `cid`/`ch`/`state`/`step` (`:144-162`) |
| Property | `tests/proptest_checker.rs` | one `proptest!` block (`:191-431`), all 7 cases inside; generators prefixed `arb_*` (`:73-93`, `:115-170`, `:184-189`) |
| Fixture | `tests/replay_fixtures.rs` | typed builder → serialize → byte-compare → re-parse → replay (`assert_fixture_matches`, `:206-235`) |
| Emitter | `crates/buzz-relay/src/conformance/mod.rs:431-726` | in-crate `#[cfg(test)] mod tests` with a local `VecTracer` sink (`:447-456`) |

**Test names encode the assertion, not the target.** `*_bites_*` for expected failures
(`cross_community_row_bites_non_interference` `src/checker.rs:210`,
`auth_allow_with_foreign_claim_bites_m2` `:228`, `impl_bug_action_bites_coverage_breach`
`:290`, `state_flip_bites_state_mismatch` `tests/proptest_checker.rs:354`); `*_is_fine` /
`*_is_ok` / `*_passes` / `*_is_accepted` for expected successes (`:247`, `:172`,
`tests/proptest_checker.rs:199`, `:304`).

**Property tests carry `P<n>` doc-comment IDs** — P1 (`tests/proptest_checker.rs:207`),
P2 (`:195`), P3a (`:269`), P3b (`:299`), P4 (`:325`), P5 (`:351`), P6 (`:401`) — matching the
"invariant properties, NOT a parallel oracle" design note at `:9-25`.

**Fixture regeneration is env-gated, not flag-gated:** `BUZZ_CONFORMANCE_UPDATE=1`
(`tests/replay_fixtures.rs:210`), so a schema change forces a deliberate re-commit rather than
silently rewriting the golden files.

**Deterministic fixture constants.** `community_a`/`community_b`/`channel_in_a`/`channel_in_b`
are hand-picked `Uuid::from_u128` values with mnemonic hex prefixes
(`tests/replay_fixtures.rs:48-62`: `0xAAAA…`, `0xBBBB…`, `0xCAFE…`, `0xDEAD…`); the property
lane uses prefix-tagged pools instead (`0x0c00…` for communities, `0x0ca0…` for channels,
`tests/proptest_checker.rs:53-63`). The rationale — reproducible serialized JSONL — is at
`tests/replay_fixtures.rs:42-46`.

---

### Serde conventions

- Every newtype is `#[serde(transparent)]` (`src/lib.rs:65`, `:92`, `:99`, `:105`, `:111`), so
  the wire form is a bare scalar.
- Both enums use `#[serde(rename_all = "snake_case")]` (`:116`, `:131`).
- `TraceAction` uses internal tagging: `#[serde(tag = "type", rename_all = "snake_case")]`
  (`:178`), so each action object carries a `"type"` discriminant matching
  `TraceAction::kind()` (`:266-277`).
- Field names are snake_case Rust identifiers with no renames, so `schema_version` /
  `state_after` appear verbatim on the wire — which is where `TRACE_SCHEMA.md:37-46` diverges
  from reality (it documents `schema` / `state`).
- JSONL, one `TraceStep` per line, no envelope: writer `tracers.rs:66-71`, test-side
  serializer `tests/replay_fixtures.rs:179-187`, parser `:191-198` (skips blank lines, panics
  with a 1-based line number).

---

### Dead-code tolerance

Four public items have zero callers anywhere in `crates/` and produce no warning because they
are `pub` in a library:

| Item | Line |
|---|---|
| `Verdict_` | `src/transitions.rs:53-56` |
| `action_channel` | `src/transitions.rs:318-330` |
| `TraceAction::is_critical` | `src/lib.rs:283-285` |
| `Scenario::require` | `src/checker.rs:54-57` |

Two more on the relay side: `conformance::step` (`mod.rs:121-123`) and the crate's own
`NoopTracer` (`src/lib.rs:323-327`), shadowed by the relay's copy (`tracers.rs:16-20`). The
convention here is evidently "keep the reserved surface"; none of the six carries a
`#[allow(dead_code)]` or a TODO.
