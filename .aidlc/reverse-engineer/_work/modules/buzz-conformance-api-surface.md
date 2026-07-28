## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: API Surface

The crate's entire surface is a Rust library API. It exposes **no HTTP route, no WebSocket
frame, no CLI subcommand, and no Nostr event kind** — the only dependencies are `serde`,
`serde_json`, `thiserror`, `uuid` (`crates/buzz-conformance/Cargo.toml:26-29`), none of which
carry a transport. `crates/buzz-relay/src/conformance/mod.rs` re-exports the types into the
relay but adds no route either; the relay's router is untouched by this module.

Two crate-level lints frame the whole API: `#![deny(unsafe_code)]` and
`#![warn(missing_docs)]` (`src/lib.rs:38-39`). Two public modules:
`checker` (`src/lib.rs:41`) and `transitions` (`src/lib.rs:42`).

---

### Exported value types (`src/lib.rs`)

| Item | Line | Shape | Notes |
|---|---|---|---|
| `CommunityLabel(pub uuid::Uuid)` | `src/lib.rs:66` | `#[serde(transparent)]` newtype | field is `pub`, so construction is unrestricted |
| `CommunityLabel::from_uuid` | `src/lib.rs:74` | `pub const fn (Uuid) -> Self` | the documented conversion seam |
| `impl Display for CommunityLabel` | `src/lib.rs:79-83` | forwards to the inner `Uuid` | the only `Display` impl in the crate |
| `SCHEMA_VERSION: u32 = 1` | `src/lib.rs:86` | `pub const` | compared by `check_trace` |
| `OpaqueId(pub String)` | `src/lib.rs:93` | transparent newtype | |
| `HostLabel(pub String)` | `src/lib.rs:100` | transparent newtype | |
| `ChannelLabel(pub uuid::Uuid)` | `src/lib.rs:106` | transparent newtype | |
| `ActorLabel(pub String)` | `src/lib.rs:112` | transparent newtype | |
| `Verdict { Allow, Deny }` | `src/lib.rs:117-123` | `snake_case` serde | |
| `SanitizedReason { Restricted, Invalid, ServerError }` | `src/lib.rs:132-140` | `snake_case` serde | |
| `AbstractState` | `src/lib.rs:150-175` | 3 pub fields | `resolved_community`, `bound_host`, `actor` |
| `TraceAction` | `src/lib.rs:179-261` | 9 variants, `#[serde(tag = "type")]` | see data-model doc for the variant table |
| `TraceAction::kind()` | `src/lib.rs:266` | `pub fn (&self) -> &'static str` | canonical coverage-set strings |
| `TraceAction::is_critical()` | `src/lib.rs:283` | `pub const fn (&self) -> bool` | returns `true` unconditionally (`src/lib.rs:284`); **zero callers repo-wide** |
| `TraceStep` | `src/lib.rs:290-299` | 3 pub fields | `schema_version`, `action`, `state_after` |
| `TraceStep::new` | `src/lib.rs:303` | `pub fn (TraceAction, AbstractState) -> Self` | stamps `SCHEMA_VERSION`; no version-override constructor exists |

All newtype inner fields are `pub`, so any consumer can mint any label. The only fenced type
is the production `buzz_core::CommunityId` the crate deliberately does **not** reuse
(`Cargo.toml:8-18`, `src/lib.rs:47-63`).

---

### The tracer trait — the sole production-facing surface

```
pub trait Tracer: Send + Sync {
    fn record(&self, step: TraceStep);
}
```
`src/lib.rs:314-318`. One method, no return value, no error channel, `&self` (interior
mutability is the implementor's problem). `Send + Sync` is what lets the relay hold it as
`Arc<dyn buzz_conformance::Tracer>` in `AppState` (`crates/buzz-relay/src/state.rs:620`).

`NoopTracer` (`src/lib.rs:323`, `impl` at `:325-327`) is a unit struct deriving
`Debug, Default, Clone, Copy` — its `record` body is empty (`src/lib.rs:326`).

Note the duplication: the relay declares a **second, independent** `NoopTracer`
(`crates/buzz-relay/src/conformance/tracers.rs:16-20`) with the same name and behavior, and
re-exports *that* one (`crates/buzz-relay/src/conformance/mod.rs:46`). The crate's own
`NoopTracer` therefore has no consumer; `AppState::default` binds the relay's copy
(`crates/buzz-relay/src/state.rs:798`).

---

### `checker` module (`src/checker.rs`)

| Item | Line | Signature |
|---|---|---|
| `Scenario` | `:24-38` | struct with `pub trace: Vec<TraceStep>` (`:29`) and `pub required_critical_actions: HashSet<String>` (`:37`) |
| `Scenario::unstructured` | `:45` | `pub fn (Vec<TraceStep>) -> Self` — empty coverage set |
| `Scenario::require` | `:54` | `pub fn (self, &str) -> Self` builder; **zero callers repo-wide** — every construction site uses `unstructured` or a struct literal |
| `check_trace` | `:74` | `pub fn (&Scenario) -> Result<(), TransitionError>` |

`check_trace` is the crate's whole entry point. Four stages, documented at `:63-73` and
implemented as: empty-trace guard (`:75-82`), first-step schema check (`:84-93`), bootstrap
(`:94`), per-step loop with a redundant per-step schema check (`:96-110`), coverage-set diff
(`:113-129`). Callers observed: only the crate's own tests
(`src/checker.rs`, `tests/proptest_checker.rs`, `tests/replay_fixtures.rs`) — no production
call site exists anywhere in `crates/`.

---

### `transitions` module (`src/transitions.rs`)

| Item | Line | Signature / shape |
|---|---|---|
| `Verdict_ { Ok }` | `:53-56` | `pub enum`, doc says "Reserved — internal placeholder"; zero references |
| `TransitionError` | `:61-102` | `thiserror::Error`, 4 variants: `IllegalTransition{step_index,detail}` (`:65`), `StateMismatch{…}` (`:77`), `NonInterference{…}` (`:87`), `CoverageBreach{detail}` (`:97`) |
| `ModelState` | `:105-118` | `pub` fields `resolved_community` (`:108`), `bound_host` (`:115`), `actor` (`:117`); `Debug + Clone` only — no `PartialEq` |
| `ModelState::bootstrap` | `:123` | `pub fn (&AbstractState) -> Self` |
| `check_step` | `:138` | `pub fn (usize, &ModelState, &TraceStep) -> Result<(), TransitionError>` |
| `action_channel` | `:318` | `pub fn (&TraceAction) -> Option<&ChannelLabel>`; **zero callers repo-wide** |
| `check_row_labels` | `:294` | private `fn (usize, &ModelState, &[CommunityLabel])` |

`TransitionError` carries no machine-readable payload beyond `step_index` — the offending
community/host/channel values are formatted into the `detail: String`
(`:146-151`, `:236-241`, `:304-309`), so a mechanical consumer can match the variant but not
extract the values.

---

### Feature gates

`crates/buzz-conformance/Cargo.toml` has **no `[features]` section** — verified by grep. There
is no `cfg` gate on any public item in `src/`; the emit path is switched at runtime by which
`Tracer` impl is bound, not at compile time. `src/lib.rs:321-322` and
`crates/buzz-relay/src/conformance/tracers.rs:9-13` both describe a hypothetical
feature-flagged elimination ("the build can omit emission entirely behind a feature") that
does not exist.

---

### Relay-side helper API (`crates/buzz-relay/src/conformance/mod.rs`)

Public because `handlers/ingest.rs` and `handlers/req.rs` call across module boundaries;
not part of `buzz-conformance` itself.

| Item | Line | Purpose |
|---|---|---|
| re-export block | `:40-44` | `AbstractState, ActorLabel, ChannelLabel, CommunityLabel, HostLabel, OpaqueId, SanitizedReason, TraceAction, TraceStep, Tracer, Verdict` |
| `state_for_request` | `:55` | `(&TenantContext, &PublicKey) -> AbstractState` |
| `actor_label` (private) | `:70` | pubkey hex prefix, ≤16 chars |
| `msg_id_label` | `:78` | first 8 bytes of the event id, hex |
| `channel_label` | `:89` | direct `Uuid` wrap |
| `claimed_community_from_event` | `:101` | first `h` tag parsed as `Uuid`, else `None` |
| `step` | `:121` | thin `TraceStep::new` wrapper; only caller is… none (grep finds no `conformance::step(` call) |
| `emit` | `:127` | `tracer.record(TraceStep::new(...))` |
| `record_req_authcheck` | `:148` | REQ-path `AuthCheck`, `claimed_community` hard-wired to `None` (`:152`) |
| `project_row_community` (private) | `:186` | per-row label, `None` on lookup miss |
| `RowCommunityProjection { Ok, MissingLookup }` | `:210-224` | discriminated projection outcome |
| `project_row_communities` | `:234` | vectorized projection |
| `record_read_message_rows` | `:265` | `ReadMessageRows` or `ImplBug` |
| `record_read_by_id_rows` | `:300` | `ReadByIdRows` or `ImplBug` |
| `EmitGuard` | `:344-354` | RAII coverage guard |
| `EmitGuard::arm` | `:383` | `(Arc<dyn Tracer>, AbstractState, &'static str) -> (Self, Arc<dyn Tracer>)` |
| `impl Drop for EmitGuard` | `:403-415` | emits `ImplBug` when the counter is 0 |
| `sanitized_reason_for` | `:422` | `&IngestError -> SanitizedReason` |
| `JsonlTracer` / `NoopTracer` re-export | `:46` | from `tracers.rs:16`, `tracers.rs:30` |
| `JsonlTracer::create` | `tracers.rs:37` | `<P: AsRef<Path>>(P) -> io::Result<Self>`; truncating open (`:39-43`) |

`CountingTracer` (`:356-359`) is private; it is the `Arc<dyn Tracer>` `EmitGuard::arm` hands
back, and its `record` bumps a `Relaxed` atomic before delegating (`:368-372`).

There is **no relay helper for `TraceAction::ReadHostFeedRows`** — grep for
`ReadHostFeedRows` across `crates/buzz-relay/` returns nothing. The variant is
constructible only from tests (`tests/proptest_checker.rs:164-166` and `:255-257`).
