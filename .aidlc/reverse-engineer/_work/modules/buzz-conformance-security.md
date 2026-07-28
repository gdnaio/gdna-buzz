## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: Security

The crate is a **detection** mechanism, not an enforcement one. It never influences a request
outcome: `Tracer::record` returns `()` (`src/lib.rs:317`), no relay code branches on a
`TraceStep`, and `check_trace` has no production caller. `crates/buzz-conformance/LIMITS.md:82-86`
states the design intent directly — the gate "is observation only — it does not feed back into
the decision."

---

### What the invariants are meant to protect

| Property | Threat it addresses | Rust predicate |
|---|---|---|
| Tenant non-interference | a request bound to community A observing a row labelled B | `check_row_labels` (`src/transitions.rs:294-312`) |
| Confused-deputy on the host axis | an A-host connection driving a B-channel authorization | `AuthCheck` Allow + foreign claim → `IllegalTransition` (`src/transitions.rs:228-250`) |
| Tenant-context stability | a handler swapping tenant mid-request or emitting from a non-`TenantContext` scope | three `StateMismatch` guards (`src/transitions.rs:143-169`) |
| Error-channel sanitization | tenant-derived information leaking through error strings | 3-variant closed enum (`src/lib.rs:132-140`) + exhaustive relay mapping (`crates/buzz-relay/src/conformance/mod.rs:422-430`) |
| Instrumentation completeness | a seam silently losing its trace (regression hiding a leak) | `ImplBug` → `CoverageBreach` (`src/transitions.rs:277-279`) |

The TLA+ statement of the first is
`Inv_NonInterference == \A o \in observations : o.labels \subseteq {o.community}`
(`docs/spec/MultiTenantRelay.tla:985-986`) — described in the spec comment as "the single-run
label/taint encoding of the two-execution non-interference theorem" (`:982-984`).

---

### Secret handling in the trace payload

The schema is designed around not carrying secrets, and the implementation mostly holds:

| Field | Value on the wire | Producer |
|---|---|---|
| `resolved_community` | community UUID | `conformance/mod.rs:57` — `tenant.community().as_uuid()` |
| `bound_host` | **raw hostname string** | `conformance/mod.rs:58` — `tenant.host().to_string()` |
| `actor` | first 16 hex chars of the authed pubkey | `conformance/mod.rs:70-75` |
| `msg_id` | first 8 bytes of the event id, hex | `conformance/mod.rs:78-85` |
| `channel` | channel UUID, direct | `conformance/mod.rs:89-91` |

No payload content, signature, private key, NIP-98 token, or wall-clock timestamp is projected
— confirmed by reading `AbstractState` (`src/lib.rs:150-175`) and all nine `TraceAction`
variants (`src/lib.rs:179-261`).

Three doc/implementation mismatches are relevant to a security reader (all catalogued in the
data-model doc): `ActorLabel`'s doc claims `blake3(pubkey)` (`src/lib.rs:110`) where the code
takes a raw pubkey prefix; `HostLabel`'s doc claims a registry-mediated opaque label
(`src/lib.rs:96-98`) where the code stores the raw host; `OpaqueId`'s doc claims a hash
(`src/lib.rs:88-91`) where the code takes an event-id prefix. The relay-side comments are
honest about the pubkey case and give the reasoning — the pubkey is already a public
Schnorr X-only key and full hexes appear in relay logs elsewhere
(`conformance/mod.rs:64-70`, `:76-77`).

The one field that is *not* covered by that reasoning is `bound_host`: it is a plaintext
hostname, and it is the field that reveals which tenant a trace belongs to. In a multi-tenant
deployment a JSONL trace file is therefore a cross-tenant correlation artifact. `JsonlTracer`
writes it to a caller-supplied path with default permissions (`tracers.rs:37-43`) and no
redaction hook.

**Production fence preserved.** The crate deliberately does not add `Serialize`/`From<Uuid>` to
`buzz_core::CommunityId`, so that type's "cannot be conjured from client input" property is
untouched; the reasoning is at `Cargo.toml:9-14` and `src/lib.rs:47-58`. Instead the crate owns
a parallel `CommunityLabel` with a public `from_uuid` (`src/lib.rs:74`) — safe because
`CommunityLabel` never feeds a decision. `LIMITS.md:70-73` correctly notes the gate cannot
detect someone *removing* that fence.

---

### Consequence of `NoopTracer` in production

`AppState` binds `Arc::new(crate::conformance::NoopTracer)` at
`crates/buzz-relay/src/state.rs:798` and nothing overwrites it — the field is assigned exactly
once repo-wide. `NoopTracer::record` has an empty body (`tracers.rs:18-20`). So in every
deployed configuration:

- Every `TraceStep` is constructed, then discarded.
- Every `EmitGuard` `ImplBug` on drop is constructed, then discarded
  (`conformance/mod.rs:404-414`).
- Every `RowCommunityProjection::MissingLookup` becomes an `ImplBug` that is discarded
  (`conformance/mod.rs:284-296`, `:319-331`).
- No file, log line, metric, or counter records any of it — `Tracer` is the only sink, and
  there is no `tracing::warn!` fallback on the `ImplBug` paths.

**What still runs anyway** (real work performed for a discarded result):

| Work | Site | Per |
|---|---|---|
| `state_for_request` — 2 `to_string()`/hex allocations + an `AbstractState` clone | `conformance/mod.rs:55-61`; called at `ingest.rs:1381`, `:1801`, `:2348`, `:2485`, `:2194`, `:145` | every ingest, plus once per write/auth emit |
| `EmitGuard::arm` — 2 `Arc` allocations + an `AtomicUsize` | `conformance/mod.rs:383-400`, armed at `ingest.rs:1382` | every ingest |
| `CountingTracer::record` indirection on every emit | `conformance/mod.rs:367-373` | every emit |
| `claimed_community_from_event` — tag scan + UUID parse | `conformance/mod.rs:101-119`; `ingest.rs:143`, `:1788`, `:2193`, `:2334`, `:2468` | every channel-bearing write |
| `msg_id_label` — 8-byte hex format | `conformance/mod.rs:78-85` | every write emit |
| **`db.communities_of_channels` — an extra Postgres round-trip** | `handlers/req.rs:345` and `:661` | **every REQ filter and every search page** |
| `BTreeSet` build over distinct row channel ids | `handlers/req.rs:337-344`, `:648-660` | same |
| per-row label `Vec` allocation | `conformance/mod.rs:238-260` | same |

The DB round-trip is the material one, and it is **not** gated on the tracer type. The gate is
`if let Some(state_snap) = trace_state` (`handlers/req.rs:334`, `:649`), and `trace_state` is
`Some` whenever the authenticated pubkey bytes parse (`handlers/req.rs:116-118`, comment
`:112-115` — "on the hot read path this is always `Some`"). So an authenticated REQ pays one
extra query per filter to build a trace step that `NoopTracer` throws away. On the failure
branch the code substitutes an empty map and logs `warn!` (`:347-353`, `:663-669`), so a
`communities_of_channels` outage produces log noise on a purely observational path.

**Security posture summary:** the invariants above protect nothing at runtime today. They are
assertions that hold over test-constructed traces. Any tenant-isolation regression in the
relay would be caught by this crate only if someone (a) bound `JsonlTracer`, (b) exercised the
regressing path, and (c) fed the resulting file to `check_trace` — and step (c) has no
implementation in the repo (grep: `check_trace` appears in no non-test file).

---

### Uninstrumented paths (blind spots even with a real tracer bound)

Seven `Ok`-returning paths through `ingest_event_inner` emit nothing, so the guard converts
each into an `ImplBug`/`CoverageBreach` rather than a modeled write action:

| Path | Line | Kinds affected |
|---|---|---|
| command routing → `handle_command` | `ingest.rs:1534-1536` | 30620, 41010, 41011, 41012, 46020, 46030, 46031 (`buzz-core/src/kind.rs:667-678`) |
| NIP-56 report | `ingest.rs:1561-1570` | 1984 (`buzz-core/src/kind.rs:191`) |
| moderation commands | `ingest.rs:1579-1588` | 5 kinds (`buzz-core/src/kind.rs:240-249`) |
| relay-admin | `ingest.rs:1808-1816` | 4 kinds (`buzz-core/src/kind.rs:647-655`) |
| NIP-43 leave | `ingest.rs:1820-1902` | 28936 (`buzz-core/src/kind.rs:268`) |
| reaction duplicate | `ingest.rs:2315-2321` | 7 |
| stored-event duplicate | `ingest.rs:2426-2432` | all stored kinds |

Security relevance: the command lane (`handle_command`) and the moderation lane both perform
community-scoped authorization and durable mutation, and neither surfaces an `AuthCheck` or a
`Write*` observation. A tenant-boundary bug inside `command_executor::handle_command` or
`moderation_commands::handle_moderation_command` is structurally invisible to this gate — it
would present as a `CoverageBreach`, identical to a legitimate command.

Additional structural blind spots:

- **REQ arms no guard.** `handle_req` builds `trace_state` (`req.rs:116-118`) but never calls
  `EmitGuard::arm`. A REQ that closes early — the three p-gate rejections at `req.rs:184-189`,
  `:191-196`, `:198-203`, or the `query_events` failure at `:321-325` — emits nothing and
  produces no `ImplBug`, so read-path emit loss is undetectable. The read path also emits no
  `SanitizedError` for any of those rejections: the only `SanitizedError` producer in the relay
  is the ingest wrapper (`ingest.rs:1410-1418`).
- **`AuthCheck` is skipped for six kinds** by `skip_membership`
  (`ingest.rs:1770-1776`: `KIND_NIP29_JOIN_REQUEST`, `KIND_NIP29_CREATE_GROUP`,
  `KIND_STREAM_MESSAGE_EDIT`, `KIND_NIP29_EDIT_METADATA`, `KIND_NIP29_DELETE_EVENT`,
  `KIND_NIP29_DELETE_GROUP`) — no membership check runs, so no verdict is traced.
- **`ReadHostFeedRows` has no producer.** Grep across `crates/buzz-relay/` finds none, so the
  kinds-only host-feed read surface (`docs/spec/MultiTenantRelay.tla:703`) is untraced.
- **The M2/M8 rule is inert on the read path** because `record_req_authcheck` hard-wires
  `claimed_community: None` (`conformance/mod.rs:152`); the checker's guard requires
  `Some(c)` (`src/transitions.rs:233`).
- **Row-label confinement is the only leak detector, and it needs a same-request read.** A
  write that persisted under the wrong community produces an in-spec trace: all three write
  arms return `Ok(())` unconditionally (`src/transitions.rs:187`, `:191`, `:198`).
- **`Inv_NoTenantContextFailsClosed`** (`docs/spec/MultiTenantRelay.tla:1116-1118`) — the
  "missing TenantContext serves no rows" backstop — has no Rust predicate; an empty
  `row_communities` passes `check_row_labels` vacuously.

`LIMITS.md:47-79` documents several of these classes honestly (DB-layer leaks the projection
doesn't read, cross-pod leaks, timing, fan-out, type-fence removal, spec bugs), but it also
says "the read-seam half of the gate is **not yet armed**" (`:56-57`) — stale, since
`handlers/req.rs:334-361` and `:649-677` now emit.
