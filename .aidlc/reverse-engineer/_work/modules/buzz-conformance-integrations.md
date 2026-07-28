## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: Integrations

Two integration edges only: **inward** from `buzz-relay` (the sole dependent crate) and
**outward** to a TLA+ spec file that is read by humans, never by code. No network client, no
external service, no message broker.

---

### Dependency graph

`buzz-conformance` is declared in exactly two manifests:

| Manifest | Line | Form |
|---|---|---|
| workspace root | `Cargo.toml:5` | workspace member |
| workspace root | `Cargo.toml:125` | `buzz-conformance = { path = "crates/buzz-conformance" }` |
| `crates/buzz-relay/Cargo.toml` | `:20` | `buzz-conformance = { workspace = true }` |

**`buzz-admin` does not declare this dependency.** Grep for `conformance` in
`crates/buzz-admin/Cargo.toml` returns nothing, and grep for `buzz_conformance` /
`conformance` across `crates/buzz-admin/src/` returns nothing. Repo-wide, the only
`Cargo.toml` files mentioning the crate are the workspace root and `buzz-relay`.

Outbound, the crate depends on `serde`, `serde_json`, `thiserror`, `uuid`
(`crates/buzz-conformance/Cargo.toml:26-29`) and `proptest` as a dev-dep (`:34`). The
"independence rule" comment (`:7-24`) enumerates the crates it must never depend on
(`buzz-db`, `buzz-relay`, `buzz-pubsub`, `buzz-auth`, `buzz-search`, `buzz-audit`) and the
manifest honors it — including the deliberate refusal to reuse `buzz_core::CommunityId`, so
that type's "no Serde, no `From<Uuid>`" fence stays intact (`:9-14`, restated
`src/lib.rs:47-63`).

---

### Relay call-in path

**1. Tracer binding.** `AppState` holds `pub tracer: Arc<dyn buzz_conformance::Tracer>`
(`crates/buzz-relay/src/state.rs:620`, doc `:615-619`). The constructor binds
`Arc::new(crate::conformance::NoopTracer)` (`crates/buzz-relay/src/state.rs:798`, comment
`:794-797`). Nothing else ever writes the field — grep for `tracer:` and `.tracer =` across
`crates/buzz-relay/src/` finds only this one assignment plus reads at
`handlers/ingest.rs:1383`, `handlers/req.rs:145`, `:356`, `:672`. The constructor comment
promises "Conformance tests overwrite this with a JsonlTracer … (see test helpers in
`crates/buzz-test-client` once those land)" (`:795-797`); those helpers do not exist —
`crates/buzz-test-client/tests/conformance_multitenant.rs` never references
`buzz_conformance`.

**2. Two tracer impls, relay-side.** `crates/buzz-relay/src/conformance/tracers.rs` declares
`NoopTracer` (`:16-20`, empty `record`) and `JsonlTracer` (`:30-45`) which serializes one JSON
object per line into a truncating-open file (`:37-43`) behind a `Mutex<BufWriter<File>>`
(`:31`). `JsonlTracer::record` (`:55-72`) recovers from lock poisoning by
`e.into_inner()` (`:60-63`) and swallows serialization/IO failures (`:68-70`). `JsonlTracer`
is never constructed anywhere in the repo.

The relay's `NoopTracer` shadows the identically-named one in the crate
(`crates/buzz-conformance/src/lib.rs:323-327`) and is what `mod.rs:46` re-exports, so the
crate's own no-op impl is dead.

**3. Ingest seam.** `handlers/ingest.rs:47-50` imports the helper set
(`self as conf, channel_label, claimed_community_from_event, emit, msg_id_label,
state_for_request, EmitGuard, TraceAction, Verdict`). `ingest_event`:

| Step | Line |
|---|---|
| build `AbstractState` | `:1381` |
| arm guard, receive counting tracer | `:1382-1386` |
| delegate to `ingest_event_inner(state, &tracer, …)` | `:1388` |
| map terminal `IngestError` → `SanitizedError` | `:1410-1418` |
| guard drops (implicit) | comment `:1419-1423` |

`ingest_event_inner` takes `tracer: &Arc<dyn buzz_conformance::Tracer>` (`:1429`) and emits at
`:1547` (via `emit_product_feedback_success`, `:133-147`), `:1794-1802`, `:2189-2195`,
`:2348`, `:2485`.

**4. REQ seam.** `handlers/req.rs:116-118` builds `trace_state: Option<AbstractState>` from
`PublicKey::from_slice(&pubkey_bytes).ok().map(…)`. It is `Option` **only** because the pubkey
bytes may be malformed (comment `:112-115`) — it has no relationship to which tracer is bound,
so `trace_state` is `Some` on essentially every authenticated REQ even under `NoopTracer`.

Plumbed by value into the search lane: `handle_search_req(..., trace_state.as_ref())`
(`:230`), parameter `trace_state: Option<&crate::conformance::AbstractState>` (`:514`).

Three gated blocks:

| Block | Gate | Extra DB query | Emit |
|---|---|---|---|
| membership confirmation | `:143` | none (reuses the `db.is_member` result from `:137-141`) | `record_req_authcheck` `:144-150` |
| non-search read | `:334` | `db.communities_of_channels` `:345` | `record_read_message_rows` `:355-361` |
| search read | `:649` | `db.communities_of_channels` `:661` | `record_read_by_id_rows` `:671-677` |

The two `communities_of_channels` calls are inside the `trace_state.is_some()` blocks, not
inside a tracer-type check — so both run and are awaited even though the resulting
`TraceStep` is immediately discarded by `NoopTracer::record`. On DB error the code substitutes
an empty `HashMap` (`:352`, `:668`), which turns every channel-scoped row into a
missing-lookup and yields a single `ImplBug` step for the whole page.

**5. Guard arm/disarm.** There is no disarm API. `EmitGuard::arm`
(`conformance/mod.rs:383-400`) hands back a `CountingTracer` (`:356-373`) and the caller
substitutes it for the duration; `Drop` (`:403-415`) decides based on the counter. The ingest
site names the seam `"ingest_event_exited_without_trace"` (`ingest.rs:1385`) and the
guard-drop test asserts that string flows through
(`conformance/mod.rs:516-521`). One guard exists in the whole relay — the REQ path arms none.

**6. Error-alphabet coupling.** `sanitized_reason_for` (`conformance/mod.rs:422-430`) is the
only place `buzz-relay`'s error type touches the schema, and it is an exhaustive match over
`crate::handlers::ingest::IngestError` — a new variant is a compile error, which is the
mechanism `crates/buzz-conformance/TRACE_SCHEMA.md:120-124` calls "closed".

---

### TLA+ spec relationship

The relationship is documentary: no build step, test, or CI job reads
`docs/spec/MultiTenantRelay.tla`. The coupling is doc comments carrying line numbers.

| Rust site | Cites | Actual spec line | Match |
|---|---|---|---|
| `src/lib.rs:186` | `WriteInsert` 514–550 | `:514` | yes |
| `src/lib.rs:187` | `WriteInsertGlobal` 559–595 | `:559` | yes |
| `src/lib.rs:188` | `WriteDuplicate` 606–637 | `:606` | yes |
| `src/lib.rs:189` | `SanitizedError` 778 | `:778` | yes |
| `src/lib.rs:190` | `AuthCheck` 794 | `:794` | yes |
| `src/lib.rs:191` | `ReadMessageRows` 643 | `:643` | yes |
| `src/lib.rs:192` | `ReadByIdRows` 681 | `:681` | yes |
| `src/lib.rs:193` | `ReadHostFeedRows` "line ~720" | `:703` | off by 17 |
| `src/transitions.rs:53` / `:296` | `Inv_NonInterference` "line ~983" | `:985` | ~approximate |
| `src/transitions.rs:54` | `Inv_ReadConfinement` "line ~1003" | `:999` | ~approximate |
| `src/lib.rs:115` | `AuthCheck` verdict alphabet, "spec line 794" | `:800` for `verdict ==` | close |

The relay repeats the citations: `ingest.rs:1777` ("Spec AuthCheck (line 794)"),
`:2327-2329` ("WriteInsert (line 514) / WriteDuplicate (line 606)"), `:2450-2456`
("WriteInsert (line 514) / WriteInsertGlobal (line 559) / WriteDuplicate (line 606) …
lines 559-595"), `conformance/mod.rs:422-423` ("spec line 778"). All resolve correctly.

`TRACE_SCHEMA.md` drifts from both: it cites `WriteInsertGlobal` at "line 562"
(`:57`) and `WriteDuplicate` at "line 612" (`:69`) — actual `:559` and `:606`.

**Spec surface not integrated.** `Next` has 23 disjuncts
(`docs/spec/MultiTenantRelay.tla:933-956`); the trace vocabulary covers 8. `Safety` conjoins
13 invariants (`:1129-1142`); the Rust checker enforces `Inv_NonInterference` for reads and a
fragment of `AuthCheck`. `docs/spec/MultiTenantRelay.cfg:26` declares a 9-element
`SanitizedErrors` set against the Rust enum's 3. `docs/spec/GitOnObjectStore.tla` is a
separate spec consumed by `crates/buzz-relay/src/api/git/cas_publish.rs` — unrelated to this
crate.

---

### Build / CI integration

| Surface | Where | Runs the crate? |
|---|---|---|
| `just test-unit` | `justfile:275-296`, conformance step at `:286-290` (`cargo nextest run -p buzz-conformance`) | **Yes** — all targets, lib + both integration tests |
| `just ci` | `justfile:266` → `test-unit` | Yes, transitively |
| `scripts/run-tests.sh unit` | `:78-103`, conformance at `:96-99` (`cargo test -p buzz-conformance`) | Yes (nextest-absent fallback) |
| relay-side emitter tests | `crates/buzz-relay/src/conformance/mod.rs:431-726`, `handlers/ingest.rs:2530-2565` | **No** — `buzz-relay` appears in neither unit list |
| GitHub Actions | grep `conformance` in `.github/workflows/` | No hits |

Contrast with the `buzz-relay-mesh` crate (`Cargo.toml:27`), which appears in no
`test-unit`/`run-tests.sh` step either — the same omission pattern, so `buzz-conformance`
being present in the unit gate is the exception rather than the rule for non-core crates.

**Documentation integration is absent.** Grep for `buzz-conformance`, `MultiTenantRelay.tla`,
`conformance`, `TLA`, and `formal` across `AGENTS.md`, `ARCHITECTURE.md`, and
`CONTRIBUTING.md` returns nothing. The crate is missing from `AGENTS.md`'s repo-structure
table (which lists `buzz-audit` at `AGENTS.md:46` and its neighbours but no
`buzz-conformance`), and `ARCHITECTURE.md` has no mention of the formal-methods lane at all.
The crate's own `LIMITS.md` (125 lines) and `TRACE_SCHEMA.md` (163 lines) are the only prose,
and neither is linked from any top-level doc.
