## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Security

---

### 1. Unsafe code

`#![deny(unsafe_code)]` at `crates/buzz-workflow/src/lib.rs:1`. Grep over the crate finds no `unsafe` blocks. **Verified: none.**

---

### 2. Expression-evaluation sandboxing (`evalexpr`)

| Question | Answer | Evidence |
|---|---|---|
| Can an expression read engine/DB state? | No. The context is a fresh `HashMapContext` populated only with the 6 trigger fields, prefixed webhook fields, and step outputs of the current run (`build_eval_context`). No handle to `Db`, the sink, or the filesystem is exposed. | `executor.rs:224-316` |
| Can an expression write state? | Not into the engine. `eval_boolean_with_context` takes `&ctx` (immutable), so assignment operators cannot mutate anything the engine later reads; the context is dropped after evaluation. | `executor.rs:373` |
| Can it call arbitrary functions? | Only the four registered helpers plus `evalexpr`'s own builtins: `str_contains`, `str_starts_with`, `str_ends_with`, `str_len`. No I/O, process, or reflection functions are registered. | `executor.rs:236-283` |
| Can it DoS the engine? | Partially mitigated. Two controls: a 4096-byte expression length cap (`MAX_EXPR_LEN`) and a 100 ms `tokio::time::timeout` around a `spawn_blocking` evaluation. **The code itself documents the gap**: "The spawn_blocking thread cannot be cancelled by tokio::time::timeout — it will run to completion even after timeout" (`executor.rs:358-360`). A pathological ≤ 4 KiB expression therefore keeps occupying a blocking-pool thread after the caller has already errored out. | `executor.rs:342`, `executor.rs:362-368`, `executor.rs:370-380` |
| Who can supply expressions? | Workflow authors only — `filter` and `if:` come from the stored definition, which the relay gates on workflow ownership before upsert (`crates/buzz-relay/src/handlers/command_executor.rs:684`, ownership check at `:838-842`). Not attacker-controlled from the message path. |

Secondary hardening: webhook-derived variables are registered **before** the standard trigger fields so a webhook body cannot shadow `trigger_text`/`trigger_author`, and any webhook key starting with `trigger_` or `steps_` is dropped entirely (`executor.rs:285-296`).

---

### 3. Step-ID validation (evalexpr variable injection)

`WorkflowDef::validate` requires each step id to be non-empty, ≤ 64 chars, and `[A-Za-z0-9_]` only (`schema.rs:168-186`). Rationale is stated in-code: ids become `steps_{id}_output_{field}` variables (`schema.rs:165-167`). A dash id like `my-step` would otherwise be parsed by evalexpr as subtraction — locked down by the test at `schema.rs:756-772`; special characters (`step;drop table`) are rejected at `schema.rs:786-798`. Uniqueness is enforced separately (`schema.rs:187-192`), preventing later steps from silently overwriting an earlier step's output variables.

Residual note: the field portion of `steps_{id}_output_{field}` comes from action **output** JSON keys, all of which are engine-generated literals (`sent`, `event_id`, `status`, `body`, `added`, `response`, `slept_secs`, `skipped`) plus, for `call_webhook`, nothing attacker-controlled — the remote body lands in a single `body` string, not as object keys (`executor.rs:862-865`).

---

### 4. Template-injection surface

- Substitution is single-pass and never re-scans substituted output, so a value containing `{{trigger.text}}` cannot cause a second expansion (`executor.rs:82-121`).
- Unknown variables are emitted literally rather than erroring or resolving to empty, so a typo cannot silently blank a field (`executor.rs:110-117`).
- Templates are **not** evaluated as expressions — `resolve_variable` only performs prefix matching on `trigger.` / `steps.` (`executor.rs:126-151`); there is no code path from a template into `evalexpr`.
- Injection risk is downstream, not in the resolver:
  - `call_webhook.url` **is** templated (`executor.rs:427`), so trigger text can influence the request target. The SSRF check runs *after* resolution (`executor.rs:790-794`), so private targets remain blocked, but an attacker who controls message text of a workflow that templates the URL can steer requests to arbitrary **public** hosts, and header values are likewise templated (`executor.rs:417-424`) — a credential-bearing header could be sent to an attacker-chosen host.
  - `send_message.text` is templated and posted as a relay-signed message (see §7).
  - `send_message.channel` is templated but then must parse as a UUID and, for channel-bound workflows, must equal the bound channel (`executor.rs:477-493`).
- Header **names** and `call_webhook.method` are not templated (`executor.rs:419-426`), so header/method smuggling via trigger data is not possible.
- No shell, SQL, or filesystem sink exists in this crate; all DB access is through parameterized `buzz-db` methods.

---

### 5. SSRF protection on outbound webhooks

| Control | Exact implementation | Line |
|---|---|---|
| Guard | `check_ssrf(host, port)`: resolve via OS resolver on `spawn_blocking`; reject resolver error, reject empty address list, reject if **any** resolved address matches `buzz_core::network::is_private_ip`; return `addrs[0]` | `executor.rs:745-776` |
| Blocked ranges | IPv4 loopback, RFC1918 private, link-local, `0.0.0.0/8`, broadcast, CGNAT `100.64.0.0/10` (called out for cloud metadata reachability), benchmarking `198.18.0.0/15`; IPv6 loopback, unspecified, `fc00::/7`, `fe80::/10`, `ff00::/8`, `2001:db8::/32`, and IPv4-mapped addresses recursed through the IPv4 rules | `crates/buzz-core/src/network.rs:25-54` |
| Rebinding TOCTOU | `.resolve(host, SocketAddr::new(safe_ip, port))` pins the validated IP; the client is rebuilt per request specifically for this, at the documented cost of connection pooling | `executor.rs:800-812` |
| Redirect policy | `reqwest::redirect::Policy::none()` — redirects fully disabled with the stated reason that a redirect to an internal host would bypass the check | `executor.rs:809` |
| Response cap | 1 MiB (`WEBHOOK_MAX_RESPONSE_BYTES = 1024 * 1024`), enforced by incremental `resp.chunk()` reads that abort early rather than `resp.bytes()` | `executor.rs:778`, `executor.rs:838-860` |
| Timeout | 10 s total | `executor.rs:807` |

Gaps:
- **No scheme allowlist.** The schema doc says "must be a public HTTPS endpoint" (`schema.rs:120`) but the code only requires `reqwest::Url::parse` to succeed and a host to be present (`executor.rs:786-794`); `http://` is accepted and the port default falls back to 80 (`executor.rs:796-798`).
- **No port restriction** — any port on a public IP is reachable.
- **`add_reaction` has no SSRF guard, no redirect policy, and no response cap** (`executor.rs:885-930`); its base URL comes from `BUZZ_RELAY_BASE_URL`, i.e. operator-controlled rather than workflow-controlled, which is why the guard is absent, but a misconfigured env var is unchecked.
- Multi-address hosts: all resolved addresses are validated, but only `addrs[0]` is used, so there is no fallback attempt — a correct fail-safe rather than a gap.

---

### 6. Inbound webhook secret comparison

Not implemented in this crate. The `webhook` trigger's only inbound path is the relay's `POST /hooks/{id}` handler (`crates/buzz-relay/src/router.rs:120`, `crates/buzz-relay/src/api/bridge.rs:1759-1893`). Findings, for boundary completeness:

| Aspect | Finding | Line |
|---|---|---|
| What is compared | The **secret itself**, not an HMAC of the request body. Provided secret comes from the `X-Webhook-Secret` header, falling back to the `?secret=` query parameter; the stored value is read out of the definition JSON key `_webhook_secret`. There is no request-body signature, so bodies are unauthenticated beyond bearer-secret possession, and no replay protection. | `crates/buzz-relay/src/api/bridge.rs:1797-1817`, `crates/buzz-relay/src/webhook_secret.rs:41-47` |
| Constant-time? | Yes for content: `verify_secret` XOR-folds all bytes without short-circuiting. Length mismatch returns early — documented as acceptable because generated secrets are always 36-byte UUIDs. | `crates/buzz-relay/src/webhook_secret.rs:68-87` |
| Secret entropy | `Uuid::new_v4()` string, documented as 122 bits. | `crates/buzz-relay/src/webhook_secret.rs:22-27` |
| Missing secret | Fails closed with 401 ("webhook secret required but not configured"). | `crates/buzz-relay/src/api/bridge.rs:1810-1815` |
| Storage | Plaintext inside the workflow definition JSON (covered by `definition_hash`), stripped before API responses via `strip_secret`. Because `WorkflowDef` does not set `deny_unknown_fields` (`schema.rs:13-14`), the extra `_webhook_secret` key deserializes cleanly in this crate and is silently dropped on re-serialization. | `crates/buzz-relay/src/webhook_secret.rs:29-66` |
| Tenant binding | The community is bound from the HTTP `Host` header before any lookup; unmapped host / missing workflow both return the same generic 404. | `crates/buzz-relay/src/api/bridge.rs:1772-1788` |

---

### 7. Privilege of relay-signed actions

`send_message` executes with **relay authority**: the run's community and the workflow's `owner_pubkey` are looked up server-side (`executor.rs:535-556`), the owner pubkey is passed only as an attribution value, and the relay keypair signs the event (`action_sink.rs:62-68` documents "the relay keypair signs the event"). Consequences and containment:

- The destination is constrained to the workflow's bound channel when one exists; cross-channel overrides are rejected (`executor.rs:477-493`). Unbound workflows may target any valid channel UUID (`executor.rs:495-503`), so the sink's own channel-existence/archived checks are the remaining fence (`action_sink.rs:22-27`).
- Message **content** is fully attacker-influenceable when a workflow templates trigger text, and it is posted under the relay identity. Impersonation of a *user* is not possible (the signing key is the relay's), but relay-signed content can be produced by any channel member whose message satisfies the trigger filter.
- Manual triggers are restricted to the workflow owner by the relay ("Manual triggers execute with the workflow owner's authority, so only the owner may start them", `crates/buzz-relay/src/handlers/command_executor.rs:836-843`) — channel membership alone is insufficient.
- Approval authorization (`approver_spec`) is enforced relay-side and fails closed for role-style specs (`crates/buzz-relay/src/handlers/command_executor.rs:942-976`), not in this crate.

---

### 8. Approval-token generation / storage / single-use

| Property | Finding | Line |
|---|---|---|
| Generation | `Uuid::new_v4().to_string()`; the doc states it draws from the OS CSPRNG via `getrandom` and deliberately does **not** mix in `run_id`/`step_id` to avoid time-based predictability | `executor.rs:693-700` |
| Storage | **Never stored by this crate.** `TODO (WF-08): create approval record in DB, emit kind:46010` (`executor.rs:663`). Repo-wide grep confirms `Db::create_approval` has no caller outside `buzz-db` itself | `executor.rs:650-668` |
| Transport | Returned in-memory as `StepResult::Suspended { approval_token }` → `ExecutionResult.approval_token`; logged only as `token: <redacted>` (`executor.rs:1186-1190`) | `executor.rs:665-667` |
| Single-use enforcement | Not exercised: the primitive exists in `buzz-db` (tokens stored as SHA-256, `crates/buzz-db/src/workflow.rs:29-35`; status-guarded `UPDATE` documented as TOCTOU-safe so grant and deny cannot both win, `crates/buzz-db/src/workflow.rs:1020-1031`) but no token is ever written, so the guard is unreachable | — |
| Net effect | A run reaching an approval gate is finalized as `Failed` (`lib.rs:192-215`); the token is discarded. There is no persisted secret to leak, and equally no working approval gate |

---

### 9. Workflow-loop prevention

Two layers; **most of it lives in `buzz-relay`, not in this crate**:

| Guard | Where |
|---|---|
| Kinds 46001–46012 excluded | Both: relay pre-check `!is_workflow_execution_kind(kind_u32)` (`crates/buzz-relay/src/handlers/event.rs:505`) and again inside `on_event` (`crates/buzz-workflow/src/lib.rs:293-295`). `is_workflow_execution_kind` is defined in `crates/buzz-core/src/kind.rs:641-643` |
| Relay-signed `buzz:workflow`-tagged messages excluded | **Relay only** — requires `state.relay_keypair` and the tag scan, unavailable to this crate (`crates/buzz-relay/src/handlers/event.rs:498-503`, used at `:507`) |
| `KIND_GIFT_WRAP` (1059) excluded | **Relay only** (`crates/buzz-relay/src/handlers/event.rs:508`) |
| Command kinds excluded | **Relay only** — `is_command_kind(kind_u32)` (`crates/buzz-relay/src/handlers/event.rs:506`) |
| Workflow-authored messages carry the loop-breaking tag | Relay sink emits the `buzz:workflow` tag (`crates/buzz-relay/src/handlers/event.rs:1759-1770`) |

Consequence: a caller invoking `WorkflowEngine::on_event` directly (bypassing the relay handler) gets only the 46001–46012 exclusion. There is also **no depth/fan-out counter and no per-run loop budget** anywhere — a workflow whose `send_message` output satisfies another workflow's trigger in the same channel would be stopped only by the `buzz:workflow` relay-side tag check.

---

### 10. Input-validation gaps

| Gap | Detail | Line |
|---|---|---|
| No `deny_unknown_fields` on any schema type | Unknown YAML/JSON keys silently accepted (relied on for `_webhook_secret`, but also means typo'd action fields pass validation) | `schema.rs:13`, `:36`, `:71`, `:90` |
| `call_webhook.url` unvalidated at parse time | No scheme/host check in `validate()`; only checked at execution | `schema.rs:152-229` vs `executor.rs:786-798` |
| `send_dm.to` unvalidated | No pubkey/npub format check anywhere (the action is stubbed) | `schema.rs:102-107`, `executor.rs:580-584` |
| `request_approval.timeout` never parsed | The string is only interpolated into a log line; `"24h"` default is a literal, not a duration | `executor.rs:653-658` |
| `request_approval.from` never parsed | Passed through templating only; the approver spec check lives relay-side | `executor.rs:446-450` |
| `delay.duration` validated only at run time | Over-270 s delays fail the run instead of failing the save | `executor.rs:671-684` |
| `add_reaction.emoji` unvalidated | Sent verbatim in a JSON body | `executor.rs:592-617` |
| `step.timeout_secs` unbounded | Any `u64` accepted; a large value effectively disables the step timeout | `schema.rs:82-83`, `executor.rs:1133-1148` |
| Definition size unbounded | No cap on step count, template length, or header count in `validate()` | `schema.rs:152-229` |
| Webhook header values templated | Trigger-derived data can be injected into outbound header values (see §4) | `executor.rs:417-424` |
| Trace growth unbounded | Every step appends a trace entry containing the full action output, including up to 1 MiB of webhook response body, and the whole array is written to `execution_trace` | `executor.rs:1176-1180`, `executor.rs:862-865` |
| Emoji trigger matching is byte-exact | `reaction_added { emoji }` compares raw strings, so `"👍"` vs `"thumbsup"` vs `"+"` are distinct — a filter can be bypassed by using an equivalent representation | `lib.rs:807-822` |
