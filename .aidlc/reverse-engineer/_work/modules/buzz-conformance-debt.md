## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: Technical Debt

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
(`LIMITS.md:116-118`) — but the read-seam patch has landed (`handlers/req.rs:334-361`,
`:649-677`) and the replay did not follow.

**Evidence the emit half is nonetheless live:** `handlers/ingest.rs:1382`, `:1410-1418`,
`:1547`, `:1794`, `:2189`, `:2348`, `:2485`; `handlers/req.rs:144`, `:355`, `:671`.

---

### D-CONF-02 — `NoopTracer` in production, with real cost still paid

`AppState` binds `Arc::new(crate::conformance::NoopTracer)` (`crates/buzz-relay/src/state.rs:798`)
and nothing overwrites it. `NoopTracer::record` is empty (`tracers.rs:18-20`).

The relay nonetheless performs, per request, work whose only consumer is that empty body:

| Work | Site | Frequency |
|---|---|---|
| `state_for_request` — host `to_string()`, pubkey hex slice, `AbstractState` | `conformance/mod.rs:55-61` | once per ingest at `:1381` + once per emit at `ingest.rs:1801`, `:2194`, `:2348`, `:2485`, `:145` |
| `EmitGuard::arm` — 2 `Arc`s + `AtomicUsize` | `conformance/mod.rs:383-400` | every ingest (`ingest.rs:1382`) |
| `claimed_community_from_event` — tag scan + `Uuid::parse_str` | `conformance/mod.rs:101-119` | `ingest.rs:143`, `:1788`, `:2193`, `:2334`, `:2468` |
| **extra `db.communities_of_channels` query** | `handlers/req.rs:345`, `:661` | **every REQ filter, every search page** |
| `BTreeSet` of distinct row channels + per-row label `Vec` | `handlers/req.rs:337-344`, `:648-660`; `conformance/mod.rs:238-260` | same |

The DB round-trip is the material item. Its gate is `trace_state.is_some()`
(`handlers/req.rs:334`, `:649`), and `trace_state` is derived solely from whether the
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
| 1 | `handle_command` routing | `ingest.rs:1534-1536` | 30620, 41010, 41011, 41012, 46020, 46030, 46031 (`buzz-core/src/kind.rs:667-678`) |
| 2 | NIP-56 report | `ingest.rs:1561-1570` | 1984 (`buzz-core/src/kind.rs:191`) |
| 3 | moderation commands | `ingest.rs:1579-1588` | 5 kinds (`buzz-core/src/kind.rs:240-249`) |
| 4 | relay-admin | `ingest.rs:1808-1816` | 4 kinds (`buzz-core/src/kind.rs:647-655`) |
| 5 | NIP-43 leave | `ingest.rs:1820-1902` | 28936 (`buzz-core/src/kind.rs:268`) |
| 6 | reaction duplicate | `ingest.rs:2315-2321` | 7 (`buzz-core/src/kind.rs:58`) |
| 7 | stored-event duplicate | `ingest.rs:2426-2432` | all stored kinds |

Paths 1 and 3 are the security-relevant ones: both perform community-scoped authorization and
durable mutation inside their own handlers, so a tenant-boundary bug there produces no
`AuthCheck` and no `Write*` observation.

Contrast with the paths that *were* instrumented for exactly this reason: product feedback
(`ingest.rs:1541-1554`, with a dedicated regression test at `:2530-2565` asserting the guard is
satisfied) and push lease (`:2189-2196`). The pattern exists; it was applied to two of nine
early-return success paths.

---

### D-CONF-04 — `WriteDuplicate` is unreachable on the main write path

`handlers/ingest.rs:2426-2432` returns early when `!was_inserted`. The trailing emit block's
match therefore never sees `was_inserted == false`:

```
(Some(ch), false) => TraceAction::WriteDuplicate { … }   // ingest.rs:2475-2479 — dead
(None, _)         => TraceAction::WriteInsertGlobal { … } // ingest.rs:2480-2483 — `_` only ever true
```

The comment above the block (`:2450-2456`) reasons carefully about the duplicate case as if it
were live. The only reachable `WriteDuplicate` producer is the reaction lane
(`:2342-2346`), which requires `buzz_db::ReactionEventInsertOutcome::Inserted` to carry
`was_inserted: false` (`:2322-2325`) — the explicit `Duplicate` outcome returns at `:2315-2321`
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
`handlers/ingest.rs:2530-2565` — the ones that prove `EmitGuard` fires, that
`member → Allow` / `!member → Deny`, and that a missing projection lookup becomes `ImplBug`
rather than a substituted label — do not run in the pre-PR gate.
`run_integration_tests`' catch-all is `cargo test --test '*'` (`scripts/run-tests.sh:118-120`),
which selects integration targets, not `--lib` unit tests. `LIMITS.md:105-110` names
`cargo test -p buzz-relay --lib conformance::` a required CI surface; nothing runs it.

The same omission applies to `buzz-relay-mesh` (`Cargo.toml:27`), which appears in no unit
step either — so this is a general gap in the unit gate's crate list rather than a
conformance-specific oversight.
