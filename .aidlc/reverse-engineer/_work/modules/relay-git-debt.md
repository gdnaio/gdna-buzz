## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: Debt

#### 1. Quantified baseline

| Metric | Value |
|---|---|
| Files | 10 |
| Total LOC | 8 935 |
| Production LOC (excluding `#[cfg(test)]` regions) | ≈ 5 830 |
| Test LOC | ≈ 3 105 (35 % of the module) |
| Production functions | 124 |
| Unit tests | 99 (76 `#[test]`, 23 `#[tokio::test]`) |
| `#[ignore]`d tests in-module | **0** |
| `#[ignore]`d E2E tests covering this module | 2 (`crates/buzz-test-client/tests/e2e_git.rs:195`, `:333`) |
| Env-gated tests that silently pass when the gate is off | 10 (`store.rs:1018`, `:1046`, `:1127`, `:1146`; `hydrate.rs:642`, `:766`, `:797`; `cas_publish.rs:1618`) |
| Production `unwrap()`/`expect()` | **17** (`transport.rs` 10, `store.rs` 5, `cas_publish.rs` 1, `policy.rs` 1) |
| `unsafe` | **0** |
| `TODO`/`FIXME`/`XXX`/`HACK` markers | **0** |
| Production `Command::new("git")` sites | **10** |
| HTTP endpoints | **4** |
| Public items with zero production callers | **1** (`policy::generate_hook_hmac`) |
| Public items with zero *external* callers (over-export) | 12 |
| `#![allow(dead_code)]` blanket suppressions | 1 (`store.rs:25`) |
| Stale doc comments identified | 4 |
| Metrics emitted | 8 counters, 11 histograms, 3 gauges |

##### File sizes

| File | LOC | Assessment |
|---|---|---|
| `transport.rs` | 2 288 | over-scoped: auth extractor + pkt-line codec + 3 handlers + 2 subprocess runners + 4 stream adapters + the fence |
| `cas_publish.rs` | 1 891 | 560 lines of that is the compaction subsystem |
| `store.rs` | 1 164 | 308 of them are the conformance probe |
| `hydrate.rs` | 893 | 412 production / 412 test |
| `policy.rs` | 775 | 334 production / 332 test (two of which embed bash scripts) |
| `pack_cache.rs` | 686 | cohesive |
| `manifest.rs` | 570 | cohesive |
| `manifest_event.rs` | 395 | cohesive, pure |
| `hook.rs` | 207 | 118 of them are an embedded bash script |
| `mod.rs` | 66 | cohesive |

##### Longest production functions

| Lines | Location | Function |
|---|---|---|
| 308 | `store.rs:571-878` | `run_conformance_probe` |
| 265 | `cas_publish.rs:1021-1285` | `cas_publish_inner` |
| 242 | `policy.rs:173-414` | `hook_policy_check` |
| 204 | `transport.rs:1548-1751` | `finalize_push` |
| 147 | `transport.rs:79-225` | `GitAuth::from_request_parts` |
| 146 | `transport.rs:994-1139` | `run_git_at` |
| 145 | `cas_publish.rs:666-810` | `capture_compacted_packs` |
| 127 | `transport.rs:596-722` | `info_refs_subprocess` |
| 105 | `transport.rs:858-962` | `receive_pack` |
| 87 | `hydrate.rs:293-379` | `materialize_manifest` |

Four functions exceed 200 lines. `run_conformance_probe` is four independent phases in one body with three near-duplicate error-construction blocks; `cas_publish_inner` carries five mutable locals (`compaction_failure`, `compaction_observation`, `compacted_manifest`, plus the pack/manifest keys) threaded through a compaction branch and six early returns; `hook_policy_check` is nine numbered steps with a `return 403` in each; `finalize_push` interleaves the fence, a five-arm error mapping, and the 30618 emission.

---

#### 2. Prioritized findings

##### D-GIT-01 — Critical: pointer creation is ungated by announcement, reservation, or quota

`receive_pack` performs no kind:30617 or `git_repo_names` lookup (`transport.rs:858-962`). Because git skips `execute_commands` — and therefore the pre-receive hook — when a client sends zero ref commands, `PackOutput.ok` is true (`transport.rs:1136-1139`) and `finalize_push` proceeds to `cas_publish`, which will `put_pointer(IfNoneMatchStar)` a pointer into existence for any `(owner, repo)` (`cas_publish.rs:1208-1237`). Two effects: pointer-namespace squatting that bypasses the reservation and per-pubkey quota (`handlers/side_effects.rs:2441-2513`), and an authenticated ETag-bump that turns concurrent legitimate pushes into 409s.

Also falsifies the design's stated invariant "pointer-absence means never announced" (`docs/git-on-object-storage.md` §Implementation Correspondence), on which `info_refs`'s unambiguous 404 depends.

**Fix:** gate `receive_pack` on an existing pointer *or* a live `git_repo_names` reservation before hydrating, and/or reject a receive-pack request whose report-status shows zero ref commands before reaching `cas_publish`.

*Basis: code structure plus `git receive-pack` command-dispatch semantics; no behavioral test exists to confirm.*

##### D-GIT-02 — Critical: no read authorization on any git path

Neither `info_refs` (`transport.rs:539-594`) nor `upload_pack` (`:786-827`) consults kind:30617, the `buzz-channel` binding, channel membership, or channel visibility. Since `require_relay_membership` defaults to **false** (`config.rs:483-485`, pinned `config.rs:954-955`), the effective read policy on a default relay is "anyone with a keypair." The module doc says "No public repos for v1" (`transport.rs:60-66`), which does not describe the code.

**Fix:** mirror the push path's kind:30617 + channel-role resolution on the read path, or state the model honestly in the doc and require membership for git.

##### D-GIT-03 — High: silent ref-wipe path for non-40-hex OIDs

`snapshot_workspace_state` `warn!`s and drops any ref whose oid is not exactly 40 hex (`cas_publish.rs:325-328`), while `is_hex_oid` (`manifest.rs:156`), `manifest_event::is_valid_oid` (`:129`), and the advertisement's `object-format` derivation (`transport.rs:473-479`) all accept 64. A SHA-256 repository would snapshot to `refs: {}` and the CAS would publish a manifest that deletes every ref — `validate()` accepts it (`manifest.rs:204-245`). Unreachable today only because `init_bare_repo` hardcodes SHA-1 (`hydrate.rs:181-184`).

**Fix:** make the snapshot fail closed (`CasError::PackCapture`) on any unparseable oid line, and either commit to SHA-1 everywhere or support SHA-256 end to end.

##### D-GIT-04 — High: `hydrate::run_git` has no timeout and runs `index-pack` on attacker-influenced bytes

`run_git` (`hydrate.rs:451-470`) awaits `.output()` with no `tokio::time::timeout`, and one of its four call sites is `git index-pack <pack>` (`hydrate.rs:420`) over pack bytes fetched from the object store. Every other subprocess in the module is bounded (120/300/600 s). Combined with `index-pack` running without `--strict` (deliberately, `hydrate.rs:410-416`), a pathological pack can occupy a `git_semaphore` permit indefinitely; 20 such requests stop git relay-wide.

**Fix:** wrap `run_git` in a timeout; consider a separate, smaller bound for `verify-pack`/`index-pack`.

##### D-GIT-05 — High: the publish fence's real invariant is undocumented

`transport.rs:1128-1135` presents `receive_pack_report_rejected` as "the primary fence for a denied push." It is not: a client that never negotiates `report-status` produces no status stream, so a hook decline yields `ok == true` and the CAS runs. It is safe anyway because a pre-receive decline rejects all commands and never migrates quarantined objects, so `refs_after == parent.refs` and the CAS installs an identical digest. **The safety comes from the workspace snapshot, not the status parse** — and that argument appears nowhere in the code or in `docs/git-on-object-storage.md`. A future refactor that reorders the snapshot or trusts `ok` for something other than "skip a redundant CAS" would silently lose the property.

**Fix:** write the invariant down at `finalize_push`, and add a test that a hook-declined push publishes nothing *with* report-status disabled.

##### D-GIT-06 — High: `git_semaphore` is global, and the fast path bypasses it entirely

One `Semaphore::new(20)` per process, not per tenant and not per pubkey (`state.rs:729`). Any authenticated caller can hold all 20 permits with slow pushes and starve git for every community on the pod. Separately, the `info/refs` fast path takes **no** permit (`transport.rs:552-577`), so unbounded concurrent requests each issue two S3 round-trips with no local backpressure. No git route is rate-limited (no `enforce_http_admission`, no `admission_rate_limiter` — verified by absence in `transport.rs`).

**Fix:** per-tenant (or per-pubkey) sub-limits, and an admission limiter on the fast path.

##### D-GIT-07 — High: no metrics for push outcome or authorization decisions

There is no counter for push success / 409 conflict / 400 invalid-manifest / 413, and none for policy allow/deny. The design explicitly defers the "add a best-effort local lock" decision to metrics ("if contention ever shows up in metrics" — `docs/git-on-object-storage.md` §Scope; echoed at `transport.rs:872-875`), but the CAS-conflict signal that decision depends on **does not exist**. `buzz_git_pack_compactions_total{outcome="cas_conflict"}` fires only on the compaction path (`cas_publish.rs:1265-1272`), so a normal-path conflict is invisible.

**Fix:** `buzz_git_push_total{outcome}` and `buzz_git_policy_decisions_total{decision}`.

##### D-GIT-08 — Medium: the `idx/` object is the only non-content-addressed object

`idx/<pack-digest>` is keyed by the *pack's* digest, not by its own bytes (`store.rs:227-241`), so `get_idx` cannot digest-verify it (`hydrate.rs:388-397`). The only defense is `git verify-pack` with regeneration on failure (`:398-406`). Every other object in the design is digest-verified, and `store.rs:19-24` advertises exactly that property. An attacker with bucket write who wins the create race for a pack lacking a sidecar controls bytes git will parse.

**Fix:** key the sidecar on its own digest and record it in the manifest, or drop the sidecar tier and always regenerate (measuring the cost first).

##### D-GIT-09 — Medium: four stale doc comments, one of which masks dead code

| Site | Claim | Reality |
|---|---|---|
| `store.rs:25` | `#![allow(dead_code)] // wired in by the push path in a follow-up commit` | the push path is wired in (`cas_publish.rs:826`, `:1194`, `:1235`); the blanket allow now hides genuinely dead items module-wide |
| `hydrate.rs:24-30` | "We narrow `#[allow(dead_code)]` to those specific items" | there is no `#[allow(dead_code)]` in `hydrate.rs` |
| `hydrate.rs:456-457` | "Match transport.rs's `harden_git_env` semantics" | omits `GIT_CONFIG_GLOBAL`; relies on `HOME=<cwd>` instead |
| `cas_publish.rs:490-497` | "Deduplicate against the same-oid case — no point feeding `X ^X`" | no dedup; every ref value is pushed unconditionally (`:498-517`). `capture_compacted_packs` **does** dedup (`:696-702`) |

##### D-GIT-10 — Medium: design-doc drift

| Doc claim | Site | Reality |
|---|---|---|
| `build_git_response` "has two call sites … shared with the read paths — info_refs, upload_pack" | doc §Implementation Correspondence | both call sites are inside `finalize_push` (`transport.rs:1574`, `:1748`); read paths use `stream_git_read` (`:1414`) and an inline builder (`:715-721`). The doc's caveat about the discriminator is now moot. |
| `ParentState` at `cas_publish.rs:154`, `cas_publish` at `:410`, `CasError::Conflict` at `:92`, `PushContext` at `transport.rs:643`, `finalize_push` at `:674`, `build_git_response` at `:627` | doc §Implementation Correspondence table | actual: `:215`, `:997`, `:105`, `transport.rs:1514`, `:1548`, `:1500`. The doc warns line numbers are pinned at landing time, but the table is the only index a reviewer has. |
| "pointer-absence means never announced" | doc §Implementation Correspondence | see D-GIT-01 |
| Spec §Push step 4 "validate Δ against `m_before.refs`" | doc §Protocol | no counterpart inside `cas_publish`; the check lives entirely in the pre-receive hook against the workspace (`hook.rs:32-150` + `policy.rs:173-417`). Equivalent in effect, but the spec-to-code map is misleading. |
| "A behavioral integration test for runtime ordering (publish-before-response) … is the belt-and-suspenders item to add once a mockable-CAS seam exists" | doc §Implementation Correspondence | still absent; no mockable-CAS seam exists (`GitStore` is a concrete struct with no trait, `store.rs:168-172`) |

##### D-GIT-11 — Medium: `GitStore` is not mockable, so the CAS protocol has no unit-test coverage

`GitStore` is a concrete struct (`store.rs:168-172`) with no trait abstraction. Consequence: `cas_publish` — the 265-line heart of the module — has **zero** tests that exercise the CAS itself. Its 18 tests cover only pure helpers (`compose_after`, `digest_from_*_key`, `resolve_published_head`, `should_compact`, `compacted_pack_set_is_usable`) plus two that need a real `git` binary. The lost-race path, the winner re-read, the compaction/CAS interaction, and the conflict-to-409 mapping are exercised only by an env-gated live test (`cas_publish.rs:1618-1832`) and an `#[ignore]`d E2E (`e2e_git.rs:333`). This is the largest coverage gap in the module and the direct blocker for the fence test the doc still owes.

##### D-GIT-12 — Medium: `require_localhost` is untested and silently incompatible with the UDS listener

`mod.rs` has no test module. `require_localhost` returns 403 whenever `ConnectInfo` is absent (`mod.rs:41-49`), and the UDS listener serves the same router with `.into_make_service()` (`main.rs:1182`) while the TCP listeners use `.into_make_service_with_connect_info` (`:1193`, `:1214`). Harmless today because the hook always dials `127.0.0.1` (`transport.rs:911-915`), but it is an undocumented coupling: a UDS-only deployment mode would silently break every push, and nothing tests either the fail-closed path or the loopback-accept path.

##### D-GIT-13 — Medium: 17 production `unwrap()`/`expect()` against an absolute repo rule

AGENTS.md forbids new `unwrap()`/`expect()` in production paths. All 17 are structurally infallible (`Response::builder().body(..)`, `Stdio::piped()` handle takes, a const header parse, a guarded `winners == 1`), but the rule admits no exceptions and the count will drift. `cas_publish.rs:410` (`pack_path.to_str().unwrap()`) is the one worth fixing on its merits — the sibling compaction path handles the identical case with a typed error 290 lines later (`:702-705`).

##### D-GIT-14 — Medium: pack-cache accounting can exceed its bound and drift

`prune` only evicts entries with `Arc::strong_count == 1`; if every entry is in flight the loop breaks and the cache stays over `max_bytes` (`pack_cache.rs:365-390`). Bypassed over-capacity entries are never counted at all (`:299-313`). And a `lookup` that finds an entry whose files have vanished returns `None` without removing the record, so its bytes keep counting until the next `insert` replaces it (`:241-251`). Self-healing but unbounded in the worst case, on a volume shared with ephemeral workspaces by default (`config.rs:707-712`).

##### D-GIT-15 — Medium: four subprocess sites have no timeout, and five inherit the relay's stdin

No timeout: `for-each-ref` (`cas_publish.rs:284-296`), `symbolic-ref` (`:337-345`), `index-pack` for the idx sidecar (`:409-415`), and all four `hydrate::run_git` arg sets (`hydrate.rs:451-470` — see D-GIT-04). No `stdin` configured (so the child inherits the relay's): `transport.rs:645-653`, `cas_publish.rs:284`, `:337`, `:409`, `hydrate.rs:452`.

##### D-GIT-16 — Low: kind:30618 bypasses the cross-pod publication path

Both emission sites call `fan_out_event_to_local_subscribers` directly (`transport.rs:1701-1710`, `handlers/side_effects.rs:2755-2761`) rather than `dispatch_persistent_event`. The comments justify the bypass only in terms of the access gate being a no-op for `channel_id = None`, and say nothing about Redis fan-out. Net effect: a subscriber on a different pod does not receive the ref-state event in real time. Given the design's own instruction that subscribers must "treat [30618] only as a signal to re-read `pointer(R)`" (doc §System Model), this is a UX gap rather than a correctness one — but it is undocumented.

##### D-GIT-17 — Low: kind:30618 silently omits refs and oids

`is_emittable_ref` drops everything outside `refs/heads/*` and `refs/tags/*` (`manifest_event.rs:117-127`), and malformed oids/names are skipped rather than failing (`:82-93`). A repo with `refs/notes/*`, `refs/stash`, or `refs/pull/*` publishes an incomplete ref set, and any consumer treating 30618 as authoritative (the web repo browser's branch list, `web/src/features/repos/use-repo-refs.ts:54-57`) sees a partial view with no signal that truncation occurred.

##### D-GIT-18 — Low: the conformance probe writes permanently to the production media bucket

`git_store` is built from the **media** S3 config (`state.rs:694-701`), and the probe writes immutable objects it deliberately never deletes ("immutable probe writes accumulate by design", `store.rs:864-866`): at default settings 3 `packs/<digest>` + 3 `probe/inm-race/<digest>` objects per boot, alongside user media. Only `probe/pointer-<uuid>` is cleaned up (`:618`, `:871`).

##### D-GIT-19 — Low: numeric config parses fail silently

Every `BUZZ_GIT_*` numeric var uses `.ok().and_then(parse)` with no warning on failure and no range check (`config.rs:713-738`, `main.rs:474-481`). `BUZZ_GIT_MAX_CONCURRENT_OPS=20x` silently yields 20. Contrast `BUZZ_PUSH_GATEWAY_TIMEOUT_MS`, which hard-errors with a range message (`config.rs:759-771`).

##### D-GIT-20 — Low: `.env.example` documents 6 of 11 vars

Missing: `BUZZ_GIT_MAX_REPOS_PER_PUBKEY`, `BUZZ_GIT_MAX_CONCURRENT_OPS`, `BUZZ_GIT_HOOK_HMAC_SECRET`, `BUZZ_GIT_CONFORMANCE_PROBE`, `BUZZ_GIT_PROBE_WRITERS`, `BUZZ_GIT_PROBE_ROUNDS` (`.env.example:71-79`). The Helm chart also never sets `BUZZ_GIT_MAX_REPO_BYTES`, so raising `maxPackBytes` silently multiplies both the repo bound (×2) and the cache bound (×10) while `packCacheVolumeSize` stays put.

##### D-GIT-21 — Low: dead and over-exported API

- `policy::generate_hook_hmac` (`policy.rs:419-437`) — **zero** production callers; used only at `:581`, `:626`, `:720`, all `#[cfg(test)]`. Either move it into the test module or `#[cfg(test)]`-gate it.
- `stream_git_read`'s `extra_args` parameter (`transport.rs:1418`) — always `&[]` (`:824`).
- Twelve `pub` items with no caller outside the module: `GitStore::{content_key, idx_key_for_pack_digest, get, get_limited}`, `MAX_MANIFEST_PACKS`, `PACK_COMPACTION_THRESHOLD`, `MAX_MANIFEST_REFS`, `is_safe_refname`, `is_hex_oid`, `is_pack_key`, `HydratedRepo::hydrated_packs`, `ProbeFailure`. `InfoRefsQuery`/`GitRepoParams` are `pub` with private fields, so they are unconstructible externally anyway.

##### D-GIT-22 — Low: duplicated code and triplicated test helpers

`read_log_prefix` (`transport.rs:1141-1155`) and `read_prefix` (`cas_publish.rs:877-891`) are the same function. `probe_enabled()` appears three times (`store.rs:996`, `hydrate.rs:579`, `cas_publish.rs:1578`); `tenant()` three times (`hydrate.rs:536`, `cas_publish.rs:1603`, `transport.rs:1902`); the `NamedTempFile::new_in` + `.reopen()` + `Stdio::from` idiom eight times. `harden_git_env` lives in the HTTP transport module (`transport.rs:302`) but is consumed by two storage layers.

##### D-GIT-23 — Low: env-gated live tests are indistinguishable from passing tests

Ten tests return early when `BUZZ_GIT_S3_PROBE != "1"` and report success. Only one prints a skip notice (`store.rs:1020-1022`). CI therefore shows 99/99 passing while the CAS, hydrate-roundtrip, empty-repo, and compaction paths were never executed. Prefer `#[ignore]` (which CI reports as skipped) or an explicit `eprintln!` in all ten.

##### D-GIT-24 — Low: named-reviewer attribution in production comments

"Max's blocker" (`transport.rs:530`), "Eva's call, on record in #proj-git-on-s3" (`:875-877`), "Sami #2 / Max / Dawn" (`cas_publish.rs:1171`), "Dawn's `canonical_bytes`" (`transport.rs:1670`). These encode decisions in people's names and a Slack channel rather than in the argument, and they will outlive both.

##### D-GIT-25 — Low: refname directory/file conflict bricks a repo permanently

`is_safe_refname` allows a manifest holding both `refs/heads/a` and `refs/heads/a/b`. Hydration writes `a` as a file first (BTreeMap order) and then `create_dir_all(.../a)` fails, so every subsequent read returns 500 (`hydrate.rs:351-364`). Unreachable via push (`git receive-pack` rejects D/F conflicts), but there is no repair path and no validation that would catch such a manifest before it is published.

##### D-GIT-26 — Low: `cas_publish_inner`'s compaction branch is the module's densest control flow

265 lines, five mutable locals, six early returns, and five `record_compaction` call sites that must each be kept in sync with a new error path (`cas_publish.rs:1021-1285`, plus `:961-979`). The `record_compaction` calls are copy-pasted into four separate error arms (`:1179-1189`, `:1195-1204`, `:1215-1225`, `:1252-1260`), which is exactly the shape where a new arm silently loses its metric.

---

#### 3. Recommended order of work

1. D-GIT-01, D-GIT-02 — authorization holes; both are small changes with large blast radius.
2. D-GIT-11 — introduce a `GitStore` trait. This unblocks the fence test the design doc still owes, plus tests for the lost-race path, D-GIT-05's invariant, and D-GIT-01's regression fence.
3. D-GIT-04, D-GIT-15 — bound the unbounded subprocesses.
4. D-GIT-03 — make the ref snapshot fail closed.
5. D-GIT-07 — add push-outcome and policy-decision metrics; the "do we need a local lock?" decision is currently unmeasurable.
6. D-GIT-05, D-GIT-09, D-GIT-10 — correct the comments and the doc's correspondence table while the reasoning is fresh.
7. D-GIT-06, D-GIT-08, D-GIT-14 — resource isolation and cache correctness.
8. Everything else as opportunistic cleanup; split `transport.rs` when next touching it (D-GIT-26 and the file-size row in §1).
