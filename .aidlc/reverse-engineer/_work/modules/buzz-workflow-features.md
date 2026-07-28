## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Features

---

### 1. Capabilities

| Capability | Where | Status |
|---|---|---|
| YAML → validated `WorkflowDef` → canonical JSON | `schema.rs:262-268`, `lib.rs:163-165` | full |
| Definition validation (name, steps, step-id charset/uniqueness, cron/interval invariants) | `schema.rs:152-229` | full |
| Event-driven trigger matching with cache | `lib.rs:276-383` | full |
| Cron + interval scheduling with cross-pod at-most-once claim | `lib.rs:428-672` | full |
| Sequential step execution with per-step timeout | `executor.rs:1080-1214` | full |
| `if:` conditions via evalexpr with 4 custom string helpers | `executor.rs:224-384` | full |
| Template substitution with 2 filters | `executor.rs:70-201` | full (single-pass only) |
| Execution trace persistence + partial-progress on failure | `executor.rs:1105-1201`, `error.rs:9-15`, `lib.rs:175-263` | full |
| Concurrency admission control | `executor.rs:975-980`, `lib.rs:110-111` | full |
| Approval-gate suspension + resume plumbing | `executor.rs:650-668`, `executor.rs:1015-1072` | partial — token is generated but never persisted; runs end `Failed` |
| Side-effect abstraction (`ActionSink`) | `action_sink.rs:48-70` | partial — only `send_message` exists on the trait |
| Outbound HTTP with SSRF protection | `executor.rs:745-868` | full, gated behind the `reqwest` feature |

---

### 2. Trigger completeness

| Trigger | Status | Evidence |
|---|---|---|
| `message_posted` | **full** — kind-9 gate + optional evalexpr filter | `lib.rs:958`, `lib.rs:824-846` |
| `reaction_added` | **full** — kind-7 gate + exact emoji equality (no shortcode/unicode normalization) | `lib.rs:959`, `lib.rs:807-822` |
| `diff_posted` | **full** — kind-40008 gate + optional filter | `lib.rs:960`, `lib.rs:848-880` |
| `schedule` (`cron`) | **full** — window-based match, deterministic claim anchor | `lib.rs:526-534`, `lib.rs:688-706` |
| `schedule` (`interval`) | **full** — bucket-quantized anchor, restart-safe prefilter, ≥ 60 s enforced at parse | `lib.rs:535-538`, `lib.rs:719-797`, `schema.rs:212-225` |
| `webhook` | **schema + relay-side only** — this crate never fires it; the entry point is the relay's `POST /hooks/{id}` handler, which builds the `TriggerContext` and calls `execute_from_step` directly | `lib.rs:962`, `crates/buzz-relay/src/router.rs:120`, `crates/buzz-relay/src/api/bridge.rs:1759-1893` |

---

### 3. Action completeness (precise, as of the code read)

| Action | Status | Detail |
|---|---|---|
| `send_message` | **full** (requires a wired `ActionSink`) | Community-scoped run/workflow lookup, destination validation, owner attribution, real event emission through `RelayActionSink` (`executor.rs:530-578`; `crates/buzz-relay/src/workflow_sink.rs:159`). Without `set_action_sink` it fails with `InvalidDefinition` (`lib.rs:148-156`). |
| `send_dm` | **stubbed** | Returns `WorkflowError::NotImplemented("SendDm")` — the step and therefore the run fails (`executor.rs:580-584`). |
| `set_channel_topic` | **stubbed** | Returns `WorkflowError::NotImplemented("SetChannelTopic")` (`executor.rs:586-590`). |
| `add_reaction` | **partial / non-functional against the current relay** | Code path is complete under `feature = "reqwest"` (which the relay enables, `crates/buzz-relay/Cargo.toml:63`) but targets `POST {BUZZ_RELAY_BASE_URL}/api/messages/{message_id}/reactions` (`executor.rs:885-891`). No such route is registered in the relay router (`crates/buzz-relay/src/router.rs:39-125` — verified by grep for `reactions` and `/api/messages`), so the call returns a non-success status and the step fails with `WebhookError` (`executor.rs:911-919`). Without the `reqwest` feature it silently returns `{added:false, skipped:true}` (`executor.rs:606-616`). |
| `call_webhook` | **full** under `feature = "reqwest"` | SSRF guard, DNS pinning, redirects disabled, 10 s timeout, 1 MiB cap (`executor.rs:781-866`). Without the feature it is a no-op returning `{status:0, body:null, skipped:true}` (`executor.rs:636-647`). |
| `request_approval` | **partial / effectively non-functional** | Generates a token and suspends, but nothing persists an approval record or emits kind 46010 (`executor.rs:650-668`); `finalize_run` converts the suspension into `RunStatus::Failed` (`lib.rs:192-215`). |
| `delay` | **full, bounded** | Max 270 s; longer delays fail the step (`executor.rs:671-690`). |

Header note: the module doc comment still claims "Action dispatch uses placeholder implementations that log intent. Real event emission is wired in WF-07/08" (`executor.rs:9-10`) and `dispatch_action`'s doc says "For MVP, most actions log their intent and return a success output" (`executor.rs:521-522`). Both are stale relative to the `send_message`/`call_webhook` implementations.

---

### 4. Feature-flag effects

`feature = "reqwest"` (`Cargo.toml:28-29`) toggles `add_reaction` and `call_webhook` between real HTTP and skip-stubs. `buzz-relay` enables it (`crates/buzz-relay/Cargo.toml:63`); `buzz-admin` depends on the crate without it (`crates/buzz-admin/Cargo.toml:21`), so an admin-side build compiles the skip-stubs.

---

### 5. All TODO/FIXME/HACK/XXX comments (verbatim)

Repo-grep over `crates/buzz-workflow/` found exactly three markers, all `TODO`:

| Marker | Verbatim text | Location |
|---|---|---|
| TODO | `// TODO (WF-07): emit DM event.` | `crates/buzz-workflow/src/executor.rs:582` |
| TODO | `// TODO (WF-07): update channel topic via DB.` | `crates/buzz-workflow/src/executor.rs:588` |
| TODO | `// TODO (WF-08): create approval record in DB, emit kind:46010.` | `crates/buzz-workflow/src/executor.rs:663` |

No `FIXME`, `HACK`, or `XXX` comments exist in the crate.

Adjacent in-code work markers that are not TODO-tagged but state incomplete behaviour:
- `"// Approval gates are not yet implemented (WF-08). / Fail explicitly rather than creating unreachable WaitingApproval rows."` (`lib.rs:192-193`).
- `"Long delays (hours/days) should use the scheduled resume pattern (future work: WF-09)."` (`executor.rs:674-675`).
- `"In-memory only — lost on restart. Missed fires during downtime are not replayed (acceptable for MVP)."` (`lib.rs:85-86`).
- Numbered "Fix N" comments referencing prior review rounds: Fix 4 (`schema.rs:218`), Fix 2 (`lib.rs:465`), Fix 5 (`lib.rs:572`), Fix 6 (`lib.rs:638`), Fix 1 (`lib.rs:664`).
