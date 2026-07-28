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
