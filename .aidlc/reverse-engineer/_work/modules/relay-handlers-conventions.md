## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: Conventions

---

#### 1. Handler signature shapes

Three distinct shapes coexist across 6 event handlers. Argument order is **not** consistent.

| Handler | Signature | Site |
|---|---|---|
| `moderation_commands::handle_moderation_command` | `(tenant, state, event) -> Result<(), String>` | `:91-95` |
| `relay_admin::handle_relay_admin_event` | `(tenant, state, event) -> Result<(), String>` | `:108-112` |
| `identity_archive::handle_identity_archive_event` | `(tenant, state, event) -> Result<(), String>` | `:40-44` |
| `report::handle_report_event` | `(tenant, event, state) -> Result<(), String>` — **state last** | `:44-48` |
| `product_feedback::handle` | `(tenant, event, state) -> Result<(), String>` — **state last** | `:19-23` |
| `push_lease::accept` | `(tenant, state, event, now: i64) -> Result<AcceptLeaseOutcome, AcceptError>` — **typed error, typed outcome, injected clock** | `:469-474` |

`push_lease::accept` is the only handler with a typed error and the only one taking an injected `now`. The other five call `SystemTime::now()` internally, making their freshness checks untestable without wall-clock manipulation.

Naming is also inconsistent: four handlers use `handle_<domain>_event`, `moderation_commands` uses `handle_<domain>_command`, `product_feedback` uses the bare `handle`.

`state` is always `&Arc<AppState>` except in `push_runtime`, which takes `&AppState` for helper functions (`push_runtime.rs:98`, `:125`, `:349`, `:531`) and `Arc<AppState>` by value for the two spawned loops (`:57`, `:312`).

---

#### 2. Error-string conventions

##### 2.1 Three competing prefix strategies

| Strategy | Handlers | Effect at the wire |
|---|---|---|
| **Self-prefixing** with `invalid:` / `error:` / `restricted:` | `moderation_commands` (via helpers `:548-558`), `report` (inline literals), `product_feedback` (inline literals) | ingest passes through verbatim |
| **Unprefixed**, ingest wraps as `invalid: {e}` | `relay_admin` (`ingest.rs:1811`), `identity_archive` (`ingest.rs:1917`) | authorization failures surface as `invalid: actor not authorized: …` — semantically wrong |
| **Typed error enum**, ingest maps | `push_lease` → `AcceptError::{Validation, Internal}` (`ingest.rs:187-195`) | correct 400/500 split |

`moderation_commands` is the only handler with named prefix helpers and the only one with a **test that pins the prefixes** (`moderation_commands.rs:669-680`):

```rust
fn authz_denial(e: anyhow::Error) -> String { format!("restricted: {e}") }   // :548
fn invalid(message: impl Into<String>) -> String { format!("invalid: {}", …) } // :552
fn error(message: impl Into<String>) -> String { format!("error: {}", …) }     // :556
```

`report.rs` and `product_feedback.rs` inline the same prefixes as string literals at 20+ sites, with no helper and no test pinning them.

##### 2.2 The `blocked:` prefix is unique and unprefixed

`blocked: you are banned from this community` appears at `moderation_commands.rs:139` (returned bare, not via `invalid()`), `:199` (as the disconnect reason), `ingest.rs:1622`, and `handlers/auth.rs:171` — four independent literals of the same string with no shared constant.

##### 2.3 Freshness error text is duplicated verbatim three times

```
"event timestamp out of range: created_at={event_ts}, now={now}, delta={}s (max ±120s)"
```
`moderation_commands.rs:117-120` (interpolating the named const), `relay_admin.rs:126-129` (hard-coded `±120s`), `identity_archive.rs:148-151` (hard-coded `±120s`).

##### 2.4 DB error text reaches clients

`error: database error: {e}` (`moderation_commands.rs:174` and 5 more), `database error: {e}` (`relay_admin.rs:137` and 6 more), `restricted: {e}` where `e` may be a `sqlx` error (`moderation_commands.rs:549` wrapping `moderation_authz.rs:99`). Only `push_lease.rs:572` deliberately opaques it — and in doing so discards the diagnostic entirely.

---

#### 3. Tag-extraction conventions

Four near-duplicate helper families, all private, all reimplemented per file.

| Helper | Copies | Divergence |
|---|---|---|
| `extract_tag_value(event, name) -> Option<String>` | 3 — `moderation_commands.rs:608`, `relay_admin.rs:49`, `identity_archive.rs:189` | bodies are functionally identical; `identity_archive` uses `find_map`, the others use a `for` loop |
| p-tag extraction | 3 — `extract_p_tag_bytes` (`moderation_commands.rs:561`, returns `Vec<u8>`), `extract_p_tag_hex` (`relay_admin.rs:33`, returns `String`), `extract_single_p_tag_hex` (`identity_archive.rs:170`, returns `String` **and rejects a second `p` tag**) | three different contracts for the same tag |
| 64-hex validation | 4 inline copies — `moderation_commands.rs:567`/`:582`, `relay_admin.rs:41`, `identity_archive.rs:178`, plus typed variants `report.rs:211-220` (`decode_32_byte_hex`) and `push_lease.rs:365-374` (`check_exact_hex`, lowercase-only) | `push_lease` is the only one rejecting uppercase hex |
| tag-name matching | two idioms: `tag.as_slice().first().map(\|s\| s.as_str()) == Some("p")` (`moderation_commands.rs:564`, `relay_admin.rs:36`, `identity_archive.rs:173`) vs `tag.kind().to_string() == "imeta"` (`product_feedback.rs:24`, `:81`, `push_runtime.rs:263`) | the second allocates a `String` per tag per call |

Case handling also diverges: `relay_admin.rs:169` and `identity_archive.rs:58` lowercase the target pubkey with `to_ascii_lowercase()`; `moderation_commands.rs:561-574` does not (it accepts mixed-case hex and `hex::decode`s it, so the bytes normalize anyway).

---

#### 4. Logging conventions

##### 4.1 Level usage

| Level | Convention | Examples |
|---|---|---|
| `info!` | one line per successful privileged mutation | `moderation_commands.rs:223` (`"community ban applied"`), `:258`, `:325`, `:362`, `:497`; `relay_admin.rs:164`, `:203-209`, `:268-272`, `:327-332`; `identity_archive.rs:90-97`; `community_provisioning.rs:302-308`, `:336-343`; `workflow_sink.rs:316-321` |
| `warn!` | best-effort side effect failed | `relay_admin.rs:215`, `:218`, `:275`, `:278`, `:335`; `identity_archive.rs:131`, `:135`; `moderation_notices.rs:153`; `community_provisioning.rs:220-226`; `push_runtime.rs:63`, `:130`, `:187`, `:367`, `:386`, `:415`, `:427`, `:464`, `:467` |
| `error!` | loop-level failure that will retry | `push_runtime.rs:65`, `:83`, `:337`; `storage_sweep.rs:176-181`, `:195` |
| `info!` for a *failed* side effect | **inconsistent** — notice-DM failures use `info!`, not `warn!` | `moderation_commands.rs:217`, `:319`, `:493` |

The notice-DM failure at `info!` level is the outlier: a user who was banned and never told is an `info`, while a failed NIP-43 announcement is a `warn`.

##### 4.2 Structured field conventions

Consistent field names across the group: `sender`, `target`, `actor`, `operator`, `community`, `host`, `role`, `new_role`, `kind`, `error`, `wake`, `event_id`, `attempt`, `reaped`, `changed`. `%`-sigil display formatting is used for hex/UUID values (`moderation_commands.rs:223`, `push_runtime.rs:171`).

Notably **absent**: `moderation_commands` logs `target` but never the `actor` on success (`:223`, `:258`, `:325`, `:362`) — `relay_admin` logs both `sender` and `target` (`:203-209`). The moderation success lines are therefore not attributable from logs alone.

##### 4.3 Secret handling in logs

No file logs a pubkey secret, token, or ciphertext. Verified specifics:
- `relay_admin.rs:164` logs `icon_len`, not the icon value.
- `push_runtime.rs` never logs `endpoint_grant`; the closest is `wake=%claimed.id` (a UUID).
- `push_lease.rs` logs nothing at all — zero `tracing` calls in the file.
- `moderation_notices.rs` never logs the notice body.

---

#### 5. Concurrency and background-loop conventions

| Convention | Followed by | Site |
|---|---|---|
| `loop { claim → work → backoff }` with exponential idle backoff capped at 2 s | `push_runtime::run_matcher` (`:57-90`), `run_delivery_worker` (`:312-347`) | both reset the delay on finding work |
| Off-claim-path periodic sweep using `tokio::time::Instant::elapsed()` rather than a second task | `push_runtime.rs:59-68` | rationale at `:26-28` |
| Single-flight via a stored `JoinHandle` + `is_finished()` | `storage_sweep.rs:161-165` | harvest and spawn deliberately share one lock (`:143-149`) |
| Leader election via Postgres advisory lock | `storage_sweep` (through `main.rs:1414-1430`) | **not** followed by `push_runtime`, which runs on every pod |
| `Weak<AppState>` to break `Arc` cycles | `workflow_sink.rs:159-161` | rationale `:150-155` |
| Function-local `OnceLock` for localized feature config | `main.rs:1447-1453` | rationale `:1448-1451` |
| Cross-tick state in `AppState` behind `tokio::sync::Mutex` | `storage_sweep` (`state.rs:561`) | pattern documented at `storage_sweep.rs:128-130` |

Pure/impure separation is applied consistently in the two most-tested files: `decide_authority` is factored out of `authorize_moderation_action` "so it is exhaustively unit-testable" (`moderation_authz.rs:137-139`), `match_job` is documented as "Pure match evaluation: no DB access" (`push_runtime.rs:216-218`), `should_spawn` is a pure cadence predicate (`storage_sweep.rs:105-127`), and `resolve_mention_pubkeys` is a pure function (`workflow_sink.rs:45`).

---

#### 6. Test conventions

##### 6.1 Counts

| File | LOC | Tests | `#[ignore]` | Test-mod start |
|---|---|---|---|---|
| `handlers/moderation_commands.rs` | 768 | 10 | 0 | `:619` |
| `handlers/moderation_notices.rs` | 398 | 4 | 0 | `:310` |
| `handlers/moderation_authz.rs` | 335 | 7 | 0 | `:184` |
| `handlers/relay_admin.rs` | 468 | 15 | 0 | `:348` |
| `handlers/community_provisioning.rs` | 445 | 13 | 0 | `:354` |
| `handlers/push_lease.rs` | 771 | 10 | 0 | `:600` |
| `handlers/identity_archive.rs` | 580 | 6 | 0 | `:360` |
| `handlers/report.rs` | 337 | 6 | 0 | `:231` |
| `handlers/product_feedback.rs` | 161 | 4 | 0 | `:100` |
| `push_runtime.rs` | 656 | **2** | 0 | `:578` |
| `storage_sweep.rs` | 1090 | 15 | 0 | `:360` |
| `workflow_sink.rs` | 711 | 18 | **1** (`:613`) | `:368` + `integration_tests` `:560` |
| **Total** | **6,720** | **110** | **1** | |

Test density is wildly uneven: `storage_sweep` has 15 tests for 1090 LOC (~1 per 73), `push_runtime` has **2** for 656 LOC (~1 per 328) — and one of those two is an HTTP-level test of request-id stability, not of the delivery state machine. `deliver_one`'s 10-branch response handling and `retry_or_fail`'s backoff have zero coverage.

##### 6.2 Test-module structure

Standard: `#[cfg(test)] mod tests { use super::*; … }`. `workflow_sink.rs` is the only file with a second module — `#[cfg(test)] mod integration_tests` (`:560`) with a module-level doc comment naming the commits it regresses and the exact command to run it (`:561-567`).

##### 6.3 Event-builder helper convention

Every event-handling file defines a local `make_*` helper, all slightly different:

| Helper | Signature | Site |
|---|---|---|
| `make_event(kind: u16, created_at_secs: u64, tags: Vec<Vec<String>>)` | includes a timestamp | `moderation_commands.rs:646-657` |
| `make_test_event(kind: u16, tags: Vec<Vec<&'static str>>)` | no timestamp; needs `Box::leak` at call sites (`:391`, `:369`) | `relay_admin.rs:355-365`, `identity_archive.rs:363-373` |
| `report_with_tags(tags: &[&[&str]])` | `&str` slices | `report.rs:234-245` |
| `feedback(tags: Vec<Tag>)` | pre-built `Tag`s | `product_feedback.rs:107-115` |
| `event(tags: Vec<Tag>)` | fixed `created_at = 1000` | `push_lease.rs:604-611` |

The `Box::leak(hex.clone().into_boxed_str())` idiom (`relay_admin.rs:391`, `identity_archive.rs:369`) is a deliberate test-only leak to satisfy the `&'static str` parameter — a smell caused by the helper's signature, not by the test.

##### 6.4 Postgres-gated tests: three different strategies

| Strategy | File | Behaviour without Postgres |
|---|---|---|
| `#[ignore = "requires Postgres"]` | `workflow_sink.rs:613` | **skipped and reported as ignored** — correct |
| Silent `return` on connect failure | `identity_archive.rs:515-527` | **passes green** — three bailouts: `test_pool()` returns `None` (`:517-519`), a probe `SELECT` fails (`:520-526`), `test_state()` returns `None` (`:527-529`) |
| n/a | all others | pure unit tests |

`identity_archive.rs:515` is a false-green: the module's only integration test — and the only coverage of the live-kind:0 revocation rule — silently no-ops in CI without Postgres.

`storage_sweep.rs:381-397` uses a fourth variant: it returns early if any of the four `BUZZ_STORAGE_*` env vars is externally set (`:386-394`), with an honest in-code comment "externally forced — skip rather than assert a lie" (`:392`).

##### 6.5 Cross-artifact invariant tests (the strongest convention here)

Three tests assert Rust constants match non-Rust artifacts:

| Test | Asserts | Site |
|---|---|---|
| `resolve_audit_actions_are_allowed_by_db_check_vocabulary` | every 9044 action maps into `MODERATION_ACTION_CHECK_VOCAB`, with a failure message naming `migrations/0006_moderation.sql` | `moderation_commands.rs:659-667` |
| `migration_trigger_allowlist_matches_advertised_push_kinds` | `include_str!("../../../../migrations/0018_push_match_queue.sql")` contains the literal `NEW.kind IN (7, 9, 1059, 40007, 46010)` | `push_lease.rs:696-710` |
| `command_error_prefix_helpers_preserve_machine_readable_token` | the three wire prefixes | `moderation_commands.rs:669-680` |

The `include_str!` migration test is the most valuable pattern in the group — it makes Rust/SQL drift a compile-adjacent failure. It is applied once.

##### 6.6 Async-test conventions in `storage_sweep`

- `#[tokio::test(start_paused = true)]` for timeout and multi-tick behaviour (`:686`, `:743`).
- `tokio::task::yield_now()` between `maybe_spawn_sweep` calls, with an in-code explanation that harvest happens on the *next* call so two failures need three calls (`:507-513`).
- `async { panic!("must not spawn …") }` as an assertion that a future is never polled — used 5× (`:493`, `:546`, `:671`, `:760`, `:781`), relying on the documented guarantee that an unpolled async value has not started its body (`:151-153`).
- `metrics_util::debugging::DebuggingRecorder` + `metrics::with_local_recorder` + `futures::executor::block_on` to capture gauges (`:652-656`, snapshot helpers `:800-812`, `:820-838`).
- A bounded `for _ in 0..50 { … if finished { break } yield_now() }` poll loop instead of a fixed poll count, with rationale (`:697-711`).

##### 6.7 Documentation-in-tests convention

Test names encode the rule (`admin_cannot_ban_or_timeout_owner_or_fellow_admin`, `lowercase_expansion_does_not_shift_later_mentions`, `a_completed_but_unharvested_sweep_never_emits_its_snapshot`), and several tests carry multi-paragraph rationale comments naming the reviewer or counterexample that motivated them: "Wren's redteam counterexample" (`workflow_sink.rs:490-497`), "Quinn's re-review" (`:526-528`), "Wren's L5 lesson: never a UUID where an event id belongs" (`moderation_commands.rs:717`), "Rev 3 required tests" (`storage_sweep.rs:602-628`).

Two tests document *why a test was dropped* rather than deleting silently: `workflow_sink.rs:526-528` explains the two `ẞ→ss`-premised cases were vacuous because `ẞ` lowercases to one char.

---

#### 7. Documentation conventions

##### 7.1 Module-doc structure

Every file opens with `//!` docs. The moderation files follow a house style unique to this group: a summary table of kinds → operations → side effects, then an explicitly labelled pinned-contract section, then a lane-ownership footer.

| Convention | Examples |
|---|---|
| Markdown table of kinds/permissions in the module doc | `moderation_commands.rs:14-21`, `relay_admin.rs:9-16` |
| `## Routing (pinned — …)` / `## Tag vocabulary (pinned — …)` sections with a date and reviewer | `moderation_commands.rs:23-27`, `:29-55` |
| `Lane ownership: L<n> (<name>)` footer naming the owning engineer | `moderation_commands.rs:57-60`, `moderation_notices.rs:30`, `moderation_authz.rs:21`, `report.rs:23-25`, `buzz-db/src/moderation.rs:15-16` |
| Cross-lane coordination instruction | `moderation_commands.rs:58-60` ("coordinate, don't edit ingest.rs") |
| Reference to a `PLANS/` design doc | `moderation_authz.rs:4-5`, `storage_sweep.rs:5-7`, `:35`, `moderation_notices.rs:3` |
| Reference to a TLA+ spec | `report.rs:11`, `buzz-db/src/moderation.rs:12-13` |
| `## Privacy` invariant section | `moderation_notices.rs:20-23` |
| `DESIGN:` inline marker for a deliberate refusal | `relay_admin.rs:296-299` |
| Named-thread / event-id citation for a pinned decision | `moderation_commands.rs:33` ("thread event `86f46207`"), `workflow_sink.rs:561` ("`e3661764` / `7899c1a8`") |

This is unusually good provenance documentation. The cost is that several pinned claims have drifted from the code (see the api-surface and features aspects: the "reject channel-scoped API tokens" claim at `moderation_commands.rs:50`, the "recorded in the audit row" claim at `moderation_authz.rs:61`, and the "fan out through the existing 9005/9001 + 9040 paths" claim at `moderation_commands.rs:20`).

##### 7.2 Rationale-comment convention

Long inline comments explaining *why*, including accepted tradeoffs stated explicitly rather than hidden. Representative examples:
- accepted residual race, named as tolerated — `moderation_commands.rs:419-425`
- crash-safe but not concurrency-safe, with the follow-up named — `moderation_notices.rs:132-138`
- why discovery is re-emitted unconditionally — `moderation_notices.rs:141-151`
- why `limit: Some(1000)` and not `Some(1)` — `moderation_notices.rs:222-226`
- why the operator allowlist is deployment-root, not create-only, authority — `community_provisioning.rs:236-247`
- why the storage sweep respawns on every tick after a failure — `storage_sweep.rs:89-103`
- why harvest and spawn share one lock — `storage_sweep.rs:143-149`
- why the gateway request id must be stable across retries — `push_runtime.rs:486-490`
- why case folding must run in original-char coordinates — `workflow_sink.rs:78-96`
- why the deliberately non-standard `moderation_source` tag is not an `e` tag — `moderation_notices.rs:35-38`

##### 7.3 Public-API doc coverage

All `pub` items carry doc comments, satisfying the AGENTS.md rule. Verified across `push_lease.rs` (every `pub` struct field documented, `:24-81`), `storage_sweep.rs` (`:34-46`, `:105-127`, `:143-153`, `:258-282`), `moderation_authz.rs` (`:28-69`), `buzz-db/src/moderation.rs` (every field, `:37-170`).

---

#### 8. Rust-hygiene conventions (measured)

| Rule | Count in the 12 files | Detail |
|---|---|---|
| `unsafe` | **0** | none anywhere |
| `unwrap()` in production paths | **0** | all 49 occurrences are inside `#[cfg(test)]` modules |
| `.expect()` in production paths | **7** | `push_lease.rs:534`, `:539`, `:543`, `:548`, `:552` (all justified by prior validation — "validated active endpoint" etc.); `push_runtime.rs:316` (`"push HTTP client"`), `:514` (`"closed delivery body"`) |
| `panic!` in production paths | **0** | 6 occurrences, all test assertions (`storage_sweep.rs:493/546/671/760/781`, `workflow_sink.rs:631`) |
| `todo!` / `unimplemented!` | **0** | — |
| `TODO` / `FIXME` / `HACK` / `XXX` markers | **0** | the two `// TODO (WF-07)` markers live in `buzz-workflow/src/executor.rs:577`, `:582` — outside this module |
| `#[allow(…)]` | **0** in these 12 files | `#[allow(clippy::too_many_arguments)]` appears on the DB functions they call (`buzz-db/src/push.rs:210`, `archived_identities.rs:49`) |
| `unwrap_or(0)` on `SystemTime` | 3 | `moderation_commands.rs:115`, `relay_admin.rs:124`, `identity_archive.rs:146` — fails closed but produces `now=0` in the error string |

The 5 `.expect()` calls in `push_lease.rs:530-556` are a direct consequence of `LeasePlaintext` using `Option<T>` for fields that are mandatory when `active == true`. A `LeasePlaintext::Active { … } | Inactive { … }` enum would make them unrepresentable. This is the module's clearest type-modelling debt against the "no new `expect()` in production paths" rule.

---

#### 9. Deviations from repo-wide conventions (AGENTS.md)

| AGENTS.md rule | Compliance |
|---|---|
| No `unsafe` | ✅ 0 |
| No new `unwrap()`/`expect()` in production paths | ⚠️ 7 `.expect()` (5 in `push_lease.rs`, 2 in `push_runtime.rs`) |
| New public API must have doc comments | ✅ all `pub` items documented |
| Event kinds defined in `buzz-core/src/kind.rs` | ⚠️ `KIND_PUSH_LEASE` is defined **twice** — `buzz-core/src/kind.rs:109` and `push_lease.rs:19`; ingest imports the `push_lease` copy (`ingest.rs:204`, `:450`, `:2156`) while `req.rs:1689` imports the `buzz-core` one |
| Channels scoped by `h` tags, not `e` | ✅ `workflow_sink.rs:262-263`, `moderation_notices.rs:161`; `moderation_notices.rs:35-38` explicitly refuses to abuse `e` for a row UUID |
| Prefer Nostr events over new HTTP endpoints | ✅ 13 of 14 operations are event kinds; the one HTTP endpoint (`/operator/communities`) is justified in-code as necessarily above the tenant fence (`community_provisioning.rs:3-14`) |
| Thread counters (`reply_count`/`descendant_count`) updated by reply inserters | n/a — `workflow_sink` inserts only top-level (`depth: 0`, `workflow_sink.rs:333`) and `moderation_notices` inserts via `insert_event` with no thread metadata (`:174`) |
