## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: Data Model

The crate has **no persistent data model** — no database, no migrations, no ORM. Its "data model" is a
serde wire schema (`TraceStep` JSONL) plus an in-memory `ModelState` the replay checker carries across
one trace. Everything below is a value type.

Crate size: 1,029 LOC of Rust across 3 src files + 755 LOC of tests
(`Cargo.toml` 35, `src/lib.rs` 327, `src/checker.rs` 337, `src/transitions.rs` 330,
`tests/proptest_checker.rs` 431, `tests/replay_fixtures.rs` 324). Two undeclared docs also live in the
crate: `LIMITS.md` (125) and `TRACE_SCHEMA.md` (163).

---

### 1. Opaque label newtypes

All five are `#[serde(transparent)]` newtypes — the JSON carries a bare scalar, not an object.

| Type | Definition | Inner | Derives | Relay-side source |
|---|---|---|---|---|
| `CommunityLabel` | `src/lib.rs:66` | `uuid::Uuid` | `Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord, Serialize, Deserialize` (`src/lib.rs:65`) | `CommunityLabel::from_uuid(*tenant.community().as_uuid())` — `buzz-relay/src/conformance/mod.rs:57` |
| `OpaqueId` | `src/lib.rs:93` | `String` | same minus `Copy` (`src/lib.rs:92`) | first 16 hex chars of the event id — `buzz-relay/src/conformance/mod.rs:78-85` |
| `HostLabel` | `src/lib.rs:100` | `String` | `src/lib.rs:99` | `tenant.host().to_string()` — `buzz-relay/src/conformance/mod.rs:58` |
| `ChannelLabel` | `src/lib.rs:106` | `uuid::Uuid` | `src/lib.rs:105` | direct wrap of the channel UUID — `buzz-relay/src/conformance/mod.rs:89-91` |
| `ActorLabel` | `src/lib.rs:112` | `String` | `src/lib.rs:111` | first 16 hex chars of the authed pubkey — `buzz-relay/src/conformance/mod.rs:71-75` |

`CommunityLabel::from_uuid` is a public `const fn` (`src/lib.rs:74`). `Display` is implemented only for
`CommunityLabel` (`src/lib.rs:79-83`).

### Documentation deltas on the label types (verified)

1. **`ActorLabel` doc vs. implementation.** `src/lib.rs:110` documents the actor label as *"the lower 16
   bytes of `blake3(pubkey)`. Stable, non-reversible, secret-free."* The only producer,
   `buzz-relay/src/conformance/mod.rs:71-75`, takes `actor.to_hex()[..16]` — a **prefix of the raw
   pubkey**, no hashing at all. The relay-side doc comment (`mod.rs:64-70`) is honest about this and
   explains why; the schema doc comment in this crate is stale. `TRACE_SCHEMA.md:31` also says "first 16
   hex chars of authed pubkey", agreeing with the code and contradicting `lib.rs:110`.
2. **`HostLabel` doc vs. implementation.** `src/lib.rs:96-98` says the label is *"produced by the relay
   from the bound `Host` header via a configured registry, never the raw `Host` string."* There is no
   registry: `mod.rs:58` stores `tenant.host().to_string()` directly. The committed fixture confirms a
   raw hostname on the wire — `tests/fixtures/good.jsonl` line 1 carries `"bound_host":"a.example.test"`.
3. **`OpaqueId` doc** (`src/lib.rs:88-91`) says "derived from an event id or other secret material …
   Implementations pick a hash" — the implementation is a hex prefix (`mod.rs:78-85`), not a hash.

---

### 2. Closed enums

| Type | Variants | Line | Wire form |
|---|---|---|---|
| `Verdict` | `Allow`, `Deny` | `src/lib.rs:117-123` | `snake_case` (`src/lib.rs:116`) → `"allow"` / `"deny"` |
| `SanitizedReason` | `Restricted`, `Invalid`, `ServerError` | `src/lib.rs:132-140` | `snake_case` → `"restricted"` / `"invalid"` / `"server_error"` |

`Verdict` is derived from the TLA+ `AuthCheck` verdict (`docs/spec/MultiTenantRelay.tla:801`, `verdict ==
IF allowed THEN "Allow" ELSE "Deny"`) — faithful, 2-for-2.

**`SanitizedReason` is a 3-way collapse of a 9-element spec alphabet.** `docs/spec/MultiTenantRelay.cfg:26`
sets `SanitizedErrors = {"auth-required", "restricted", "invalid", "duplicate", "pow", "rate-limited",
"blocked", "error", "frame-too-large"}`. The Rust enum has 3 variants. The doc comment at
`src/lib.rs:125-129` calls the set "the sanitized error alphabet (spec `Inv_SanitizedErrors`, M6
mutation)" and `TRACE_SCHEMA.md:110-116` calls it "closed", but closure is asserted against
`IngestError`'s 3 variants (`buzz-relay/src/conformance/mod.rs:422-430`), not against the spec's 9. A
mis-bucketed reason (e.g. relay maps a rate-limit to `Invalid`) is undetectable — see business-rules
BR-CONF-05.

---

### 3. `AbstractState` — the per-request abstract state

`src/lib.rs:150-175`. Three fields, all required, `Debug + Clone + PartialEq + Eq + Serialize +
Deserialize` (`src/lib.rs:149`):

| Field | Type | Line | Maps to relay concept | Spec counterpart |
|---|---|---|---|---|
| `resolved_community` | `CommunityLabel` | `src/lib.rs:155` | `TenantContext::community()` — host-derived, server-resolved | `o.community` on every observation (`MultiTenantRelay.tla:985`) |
| `bound_host` | `HostLabel` | `src/lib.rs:158` | `TenantContext::host()` | the `host` bound variable in `WriteInsert`/`AuthCheck`/reads; `HostCommunity[host]` |
| `actor` | `ActorLabel` | `src/lib.rs:160` | the NIP-42/NIP-98 authenticated pubkey | `o.actor` |

### What the abstract state deliberately omits — and the coverage consequence

The doc comment (`src/lib.rs:147-148`) says the state carries "deliberately … not raw payloads, pubkey
bytes, signatures, or wall-clock timestamps." Verified accurate. But the omissions also remove spec
state the checker would need:

| Spec state variable (`MultiTenantRelay.tla`) | Present in `AbstractState`? | Consequence |
|---|---|---|
| `messages` (row set) | No | `Inv_AcceptedWritesPersist` (`tla:1104`), `Inv_MessageKeyUnique` (`tla:1110`) uncheckable |
| `projections` | No | `Inv_ProjectionDerived` (`tla:1121`) uncheckable |
| `memberships` / `admittedMembers` | No | `Inv_AdmissionFence` (`tla:1071`) uncheckable — acknowledged at `src/transitions.rs:222-224` and `:253-256` |
| `HostCommunity[_]` (the full map) | No — only the one bound host | acknowledged at `src/transitions.rs:110-114`: the checker "does NOT know `HostCommunity[_]` at large" |
| `ChannelCommunity(_)` (channel→community map) | No | the checker cannot recompute a channel's real community, so `WriteInsert`/`WriteDuplicate` legality is unverifiable (BR-CONF-01/03) |
| `createdChannels`, `openChannels`, `auditHeads`, `queryFaults`, `feedReads`, `auxReads`, `authRegistrations` | No | `Inv_HostBindingFence` (`tla:1038`), `Inv_ChannelCommunityImmutable` (`tla:1057`), `Inv_NoTenantContextFailsClosed` (`tla:1116`) uncheckable |
| `worker` / observation ordering | No — explicitly rejected at `src/checker.rs:26-28` | fine: the spec models `observations` as an unordered set |

`src/transitions.rs:24-30` states the modeling reduction honestly: "a runtime trace covers ONE worker
handling ONE request — so the model state we carry is much smaller."

---

### 4. `TraceAction` — 9 variants (all of them)

`src/lib.rs:179-261`. Serde representation is internally-tagged: `#[serde(tag = "type", rename_all =
"snake_case")]` (`src/lib.rs:178`), so each JSON object carries a `"type"` discriminant.

| # | Variant | Line | Fields | `kind()` string | Spec action + line cited in code | Actual spec line |
|---|---|---|---|---|---|---|
| 1 | `WriteInsert` | `src/lib.rs:181-194` | `msg_id: OpaqueId`, `channel: ChannelLabel`, `claimed_community: Option<CommunityLabel>` | `write_insert` (`:267`) | `WriteInsert` 514–550 (`src/lib.rs:186`) | `tla:514` ✅ |
| 2 | `WriteInsertGlobal` | `src/lib.rs:195-203` | `msg_id`, `claimed_community` | `write_insert_global` (`:268`) | 559–595 (`src/lib.rs:187`) | `tla:559` ✅ |
| 3 | `WriteDuplicate` | `src/lib.rs:204-213` | `msg_id`, `channel`, `claimed_community` | `write_duplicate` (`:269`) | 606–637 (`src/lib.rs:188`) | `tla:606` ✅ |
| 4 | `SanitizedError` | `src/lib.rs:214-220` | `reason: SanitizedReason` | `sanitized_error` (`:270`) | 778 (`src/lib.rs:189`) | `tla:778` ✅ |
| 5 | `AuthCheck` | `src/lib.rs:221-230` | `channel`, `claimed_community`, `verdict: Verdict` | `auth_check` (`:271`) | 794 (`src/lib.rs:190`) | `tla:794` ✅ |
| 6 | `ReadMessageRows` | `src/lib.rs:231-240` | `channel: Option<ChannelLabel>`, `row_communities: Vec<CommunityLabel>` | `read_message_rows` (`:272`) | 643 (`src/lib.rs:191`) | `tla:643` ✅ |
| 7 | `ReadByIdRows` | `src/lib.rs:241-249` | `channel: Option<…>`, `row_communities: Vec<…>` | `read_by_id_rows` (`:273`) | 681 (`src/lib.rs:192`) | `tla:681` ✅ |
| 8 | `ReadHostFeedRows` | `src/lib.rs:250-255` | `row_communities: Vec<…>` | `read_host_feed_rows` (`:274`) | "line ~720" (`src/lib.rs:193`) | `tla:703` ⚠️ off by 17 |
| 9 | `ImplBug` | `src/lib.rs:256-261` | `kind: String` | `impl_bug` (`:275`) | not a spec action (`src/lib.rs:194`) | n/a ✅ |

`TraceAction::kind()` (`src/lib.rs:266-277`) is the canonical string used by `Scenario`'s coverage set.
`TraceAction::is_critical()` (`src/lib.rs:283-285`) is a `const fn` that returns `true` unconditionally.

### Spec-line-citation drift in the prose docs

`TRACE_SCHEMA.md` cites different lines than the Rust for the same actions: `WriteInsertGlobal` "line 562"
(`TRACE_SCHEMA.md:57`) vs. actual 559; `WriteDuplicate` "line 612" (`TRACE_SCHEMA.md:69`) vs. actual 606.
The invariant citations in `src/transitions.rs` also drift: `Inv_NonInterference` "line ~983"
(`src/transitions.rs:53`, `:287`) vs. actual `tla:985`; `Inv_ReadConfinement` "line ~1003"
(`src/transitions.rs:54`) vs. actual `tla:999`. All are annotated `~`, so they are approximations, not
false claims — but the un-tilde'd `WriteInsert` range "514–550" and `WriteInsertGlobal` "559–595" are
exact and correct.

### The `claimed_community` / `resolved_community` split

Three of the four write/auth variants carry `claimed_community: Option<CommunityLabel>` separately from
`state_after.resolved_community`. This is the load-bearing non-normalization documented at
`src/lib.rs:188-192` and `buzz-relay/src/conformance/mod.rs:17-19`. Producer:
`claimed_community_from_event` (`buzz-relay/src/conformance/mod.rs:101-119`) reads the event's first `h`
tag and parses it as a UUID; `None` when absent or unparseable.

**Semantic hazard, verified:** the `h` tag on the Buzz write path carries a **channel** UUID, not a
community UUID (`AGENTS.md` "Channel scoping: Channels use `h` tags (NIP-29 group tag)"; and the relay's
own kind:9007 handler treats `h` as the channel id — see `crates/buzz-test-client/tests/conformance_multitenant.rs:319`
building `Tag::parse(["h", &channel_uuid])`). `mod.rs:103-105` admits the ambiguity in a comment ("or
channel uuid, ambiguous"). So `claimed_community` is in practice populated with a **channel** UUID, which
will essentially never equal `resolved_community`. The single rule that consumes it — the AuthCheck M2/M8
bite (`src/transitions.rs:233-242`) — would therefore fire on nearly every real ingest, if a recording
tracer were ever enabled. See debt D-CONF-02.

On the read path `claimed_community` is unconditionally `None` by design, with a rationale doc comment at
`buzz-relay/src/conformance/mod.rs:135-145`.

### `row_communities` is a `Vec`, not a `Set` — deliberately

`src/lib.rs:236-239` and `src/transitions.rs:281-292` both state the reason: the checker must see every
leaked label, and de-duping would let a buggy emitter hide a leak. `check_row_labels`
(`src/transitions.rs:294-310`) walks the slice and fails on the first non-matching entry.

---

### 5. `TraceStep` — the JSONL record

`src/lib.rs:290-299`:

| Field | Type | Line |
|---|---|---|
| `schema_version` | `u32` | `src/lib.rs:292` |
| `action` | `TraceAction` | `src/lib.rs:294` |
| `state_after` | `AbstractState` | `src/lib.rs:298` |

`TraceStep::new` (`src/lib.rs:303-309`) stamps `SCHEMA_VERSION` (= `1`, `src/lib.rs:86`) automatically;
there is no constructor that lets a caller set a different version, so the only way to produce a
version-mismatched step is hand-written JSON.

### Replay-fixture format (JSONL)

One `TraceStep` per line, no envelope, no trailing metadata. Serializer:
`tests/replay_fixtures.rs:178-186` (`to_jsonl`); parser: `tests/replay_fixtures.rs:190-198` (`from_jsonl`,
skips blank lines, panics with the 1-based line number on a parse error). Four fixtures on disk:

| Fixture | Steps | Expected verdict | Builder | Test |
|---|---|---|---|---|
| `tests/fixtures/good.jsonl` | 3 (`auth_check` → `write_insert` → `read_message_rows`) | `Ok(())` | `tests/replay_fixtures.rs:74-101` | `:237-249` |
| `tests/fixtures/bad_host_channel_mismatch.jsonl` | 2 | `IllegalTransition` | `:107-129` | `:251-264` |
| `tests/fixtures/bad_coverage_breach.jsonl` | 1 (`impl_bug`) | `CoverageBreach` | `:134-141` | `:266-278` |
| `tests/fixtures/bad_foreign_row_leak.jsonl` | 1 | `NonInterference` | `:155-168` | `:280-292` |

The actual on-disk key names are `schema_version`, `action`, `state_after` — e.g.
`tests/fixtures/bad_coverage_breach.jsonl:1`. **`TRACE_SCHEMA.md:37-46` documents the record as
`{"schema": 1, "action": …, "state": {…}}`** — three of the four top-level keys are wrong in the doc
(`schema`/`state` vs. `schema_version`/`state_after`). Anyone hand-writing a fixture from the doc gets a
serde error.

Fixtures are byte-compared against the typed builders (`assert_fixture_matches`,
`tests/replay_fixtures.rs:205-235`) and regenerated with `BUZZ_CONFORMANCE_UPDATE=1`
(`tests/replay_fixtures.rs:210`), so a schema change forces a deliberate re-commit. This round-trip
guard works and is the crate's strongest anti-drift mechanism.

---

### 6. `ModelState` — the checker's in-memory model

`src/transitions.rs:105-118`. Fields mirror `AbstractState` 1:1 (`resolved_community`
`:108`, `bound_host` `:115`, `actor` `:117`) and are populated from the **first** trace step by
`ModelState::bootstrap` (`src/transitions.rs:123-129`). The model is **immutable for the lifetime of a
trace** — `check_step` takes `&ModelState` and the doc at `src/transitions.rs:131-132` says it "Updates
nothing." Consequence: the checker verifies *self-consistency across one request*, not evolution of relay
state. It cannot express any spec action that mutates state (`AdmitMember`, `CreateChannel`,
`RebuildProjections`, …).

Because the model is bootstrapped **from the trace itself**, a relay that consistently reported the wrong
resolved community for a whole request would pass every state check. The checker's independence
(`Cargo.toml:8-24`) is about not sharing *code*, not about independently deriving the tenant.

---

### 7. Dead type: `Verdict_`

`src/transitions.rs:53-56` declares `pub enum Verdict_ { Ok }`, documented "Reserved — internal
placeholder." Zero constructors, zero matches, zero references anywhere in `crates/**` (verified by
repo-wide grep). It survives `cargo clippy -p buzz-conformance --all-targets` with no warning because it
is `pub` in a library and the trailing underscore does not trip `non_camel_case_types`.
