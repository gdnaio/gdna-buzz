## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: Conventions

#### 1. Module layout

| File | LOC | Layer | Depends on |
|---|---|---|---|
| `mod.rs` | 66 | wiring + `require_localhost` | `policy`, `transport` |
| `transport.rs` | 2 288 | HTTP handlers, subprocess runners, fence | all others |
| `cas_publish.rs` | 1 891 | commit protocol | `manifest`, `store`, `transport::harden_git_env` |
| `store.rs` | 1 164 | object-store primitives + conformance probe | none (leaf) |
| `hydrate.rs` | 893 | read/write materialization | `manifest`, `store`, `pack_cache`, `cas_publish::ParentState` |
| `policy.rs` | 775 | HMAC callback endpoint | `buzz_core::git_perms`, `buzz_db` |
| `pack_cache.rs` | 686 | bounded local cache | `hydrate` (`pub(super)` helpers), `store` |
| `manifest.rs` | 570 | schema + predicates | leaf except `CommunityId` |
| `manifest_event.rs` | 395 | manifest → kind:30618 | `nostr` only (pure) |
| `hook.rs` | 207 | bash hook literal + installer | none |

Dependency direction is mostly clean, with two documented back-edges: `pack_cache` → `hydrate` (`hydrate.rs:272`, `:381` are `pub(super)` for exactly this) and `cas_publish`/`hydrate` → `transport::harden_git_env` (a `pub(crate)` helper living in the HTTP layer, which is where it least belongs).

#### 2. Error handling

Three distinct idioms, one per layer:

| Layer | Error type | Idiom |
|---|---|---|
| HTTP handlers | `Result<Response, Response>` | errors *are* responses, built at the failure site. `#[allow(clippy::result_large_err)]` is applied where needed (`transport.rs:265`, `:317`, `:1413`) |
| Protocol layers | `thiserror` enums | `CasError` (7 variants, `cas_publish.rs:90-145`), `HydrateError` (5, `hydrate.rs:96-121`), `StoreError` (6, `store.rs:65-102`), `ManifestError` (9, `manifest.rs:74-131`), `BuildError` (2, `manifest_event.rs:44-52`) |
| Hook installer | `anyhow::Result` | the only `anyhow` in the module (`hook.rs:152`) |

Conventions worth naming:

- **Variants encode HTTP class, not just cause.** `CasError::Conflict` is deliberately separate from `Backend` so `?`-bubbling cannot turn a 412 into a 500 (`cas_publish.rs:105-124`); `ManifestInvalid` is 4xx-class while `ManifestReadFailed` is 5xx-class (`:126-141`). `finalize_push` maps all seven (`transport.rs:1601-1657`).
- **"Not found" is a type, not a variant.** `hydrate_for_read` returns `Result<Option<HydratedRepo>>` so 404-vs-5xx is enforced by the type system (`hydrate.rs:94-97`).
- **Semantic non-errors are values.** `CasOutcome::LostRace` for HTTP 412 (`store.rs:52-64`); a 412 on a content-addressed PUT is folded into `Ok(key)` (`store.rs:271-275`).
- **Fail closed on ambiguity.** A 2xx CAS without an ETag becomes an error rather than `ETag("")` (`store.rs:522-536`); `get_pointer` errors on a missing ETag (`:462-471`); the policy endpoint returns 403 for *every* failure including DB errors (`policy.rs:277-282`).
- **Non-fatal degradations are explicit `warn!` + continue**, never swallowed: idx sidecar write (`cas_publish.rs:1148-1152`, `:828-844`), idx validation failure (`hydrate.rs:398-406`), kind:30618 build/insert (`transport.rs:1712-1735`), compaction fallback (`cas_publish.rs:1096-1103`).
- **Error strings are opaque to clients.** All git 5xx bodies are the literal `"git error"` (`transport.rs:630-700`, `:1004-1124`); detail goes to `tracing`. Policy denials leak only role/rule text.
- **`?` is used freely inside protocol layers; handlers use explicit `match` + `map_err`** so each arm can pick a status.

#### 3. Panic policy

AGENTS.md forbids new `unwrap()`/`expect()` in production paths. Current production count (outside `#[cfg(test)]`): **17**.

| File | Count | Lines | Assessment |
|---|---|---|---|
| `transport.rs` | 10 | 97, 108, 334, 572, 721, 1037, 1442, 1469, 1493, 1507 | 7 are `Response::builder().body(..).unwrap()` (infallible); 3 are `child.stdin/stdout.take()` immediately after `Stdio::piped()` (infallible by construction, one of them written as `.unwrap()` rather than `.expect()`) |
| `store.rs` | 5 | 262, 297, 490, 892 (`"*".parse().unwrap()` — const header value), 718 (`new_etag.expect("winner exists")` — guarded by `winners == 1` two lines above) | all structurally infallible |
| `cas_publish.rs` | 1 | 410 — `pack_path.to_str().unwrap()` | path is `tempdir/pack-<hex>.pack`; the sibling compaction path handles the same case with a real error (`:702-705`), so this one is an inconsistency |
| `policy.rs` | 1 | 126 — `Hmac::new_from_slice(..).expect("HMAC can take key of any size")` | infallible for HMAC |
| `hydrate.rs`, `pack_cache.rs`, `manifest.rs`, `manifest_event.rs`, `hook.rs`, `mod.rs` | 0 | — | clean |

Two additional panic-avoidance conventions:

- `pack_cache` never `unwrap()`s a poisoned mutex: `self.state.lock().unwrap_or_else(|e| e.into_inner())` at `:242`, `:335`, `:350`, `:367`.
- `pkt_line` degrades an over-long payload to an empty `0004` frame + `error!` rather than panicking or truncating a length prefix, "non-panicking in every build profile" (`transport.rs:424-454`, pinned `:2231-2237`).

`unsafe`: **0** occurrences anywhere in the module (the only matches for the string are `unsafe_refname` identifiers and error text).

`TODO`/`FIXME`/`XXX`/`HACK`: **0** occurrences.

#### 4. Streaming patterns

Two mutually exclusive strategies, chosen by whether the operation mutates published state (`transport.rs:1244-1260`, `:1405-1412`):

**Read paths — stream.** `stream_git_read` (`transport.rs:1414-1498`) composes four layers:
1. `ReaderStream` over `ChildStdout`.
2. `TimedByteStream` — hard deadline, byte/duration histograms in `Drop` (`:1282-1391`).
3. `StreamingGit` — parks the `Child` and the `HydratedRepo` to extend their lifetimes past the last byte, aborts the stdin pump on `Drop`, and kills the child when it observes a `TimedOut` item (`:1226-1332`).
4. `GitPermitStream` — holds the semaphore permit to EOF (`:1293-1310`).

**Write path — buffer.** `run_git_at` returns an owned `PackOutput` (`transport.rs:971-991`) precisely so no `Response` can exist before the CAS. The rationale is documented at the type (`:963-970`) and at the streaming helper (`:1244-1260`).

Request bodies are always pumped by a **spawned task**, never awaited inline, so stdin backpressure cannot deadlock against stdout reads (`transport.rs:1039-1064`, `:1442-1467`). Both pumps log body/decode errors at `warn` and then close stdin so git sees EOF rather than an opaque hang.

Body decoding is a stream transformer with a running counter, not a buffer-then-check (`transport.rs:766-783`).

#### 5. Temp-file and cleanup discipline

| Rule | Practice |
|---|---|
| Scratch location | Every tempdir/tempfile is created with `*_in(scratch_dir)` so it lands on the mounted volume, never `/tmp`. `cas_publish` derives its scratch as `repo_path.parent()` rather than taking another argument (`cas_publish.rs:1028-1030`). |
| Subprocess output | `NamedTempFile::new_in(...)` + `.reopen()` → `Stdio::from(file)`. The `NamedTempFile` handle stays in scope so `Drop` unlinks it; the reopened descriptor is what the child writes. Repeated verbatim at 8 sites (`transport.rs:622-644`, `:996-1018`; `cas_publish.rs:272-284`, `:511-523`, `:595-607`, `:704-712`). |
| Metadata-before-read | Every buffered read is `tokio::fs::metadata(...).len()` → size check → `tokio::fs::read(...)`, never read-then-check (`transport.rs:680-701`, `:1085-1126`; `cas_publish.rs:307-315`, `:552-568`). |
| Bounded stderr | `read_log_prefix` / `read_prefix` cap at 64 KiB using `AsyncReadExt::take`, returning `"<stderr unavailable>"` on failure — duplicated in two files (`transport.rs:1141-1155`, `cas_publish.rs:877-891`). |
| Lifetime-as-cleanup | `HydratedRepo` owns its `TempDir` (`hydrate.rs:51-56`); the explicit `drop(repo)` / `drop(ctx.repo_handle)` calls at `transport.rs:707`, `:1576`, `:1750` are load-bearing ordering, and are commented as such. |
| `kill_on_drop(true)` | On 6 of 10 subprocess sites; absent on the four `.output()`/`.status()` sites that already await completion (`cas_publish.rs:284`, `:337`, `:409`) — except `hydrate::run_git`, which sets it despite using `.output()` (`hydrate.rs:453`). |
| Cache publication | staging `TempDir` → atomic `rename` into the final digest directory; on rename failure the winner's directory is adopted (`pack_cache.rs:274-333`). |
| Process-lifetime GC | Startup sweep of stale `session-*` dirs plus a 60 s heartbeat writer aborted in `Drop` (`pack_cache.rs:127-146`, `:420-426`, `:482-509`). |
| Bash hook | `WORK_DIR=$(mktemp -d)` with `trap 'rm -rf "$WORK_DIR"' EXIT` (`hook.rs:49-53`). |

#### 6. Concurrency conventions

| Mechanism | Scope | Site |
|---|---|---|
| `git_semaphore` | global, `try_acquire_owned` (never blocks), 503 on exhaustion | `transport.rs:318-338` |
| `PACK_COMPACTION_SEMAPHORE` | process-global `const_new(1)`, `acquire` with 300 s timeout | `cas_publish.rs:86`, `:576-586` |
| `population_semaphore` | per-cache, bounds concurrent object-store pack fetches | `pack_cache.rs:110`, `:253-269` |
| Per-digest single-flight | `DashMap<String, Arc<PopulationFlight>>` with an `AtomicUsize` refcount and an RAII `FlightParticipant` that deregisters on the last drop | `pack_cache.rs:76-104`, `:186-238`, `:392-399` |
| **No per-repo lock** | deliberate; the CAS is the only writer serialization, with a 14-line justification comment | `transport.rs:865-877` |
| Sync mutex in async | `std::sync::Mutex` for cache index (never held across `await`) | `pack_cache.rs:63`, `:241-390` |

#### 7. Documentation conventions

- Every file opens with a `//!` module doc that maps code to the spec by section (`cas_publish.rs:1-49` walks §Push steps 1–8; `hydrate.rs:1-27` walks §Read).
- Invariant names from the TLA+ model (`Inv_NoFork`, `Inv_Closed`, `Inv_RefEffectApplied`, `Inv_RefDerivedFromParent`) are cited inline at the code that establishes them (`cas_publish.rs:200-213`, `:897-906`; `transport.rs:832-856`).
- Negative documentation is a first-class pattern: "What this function deliberately does *not* do" (`cas_publish.rs:34-49`), the `SECURITY:` note explaining why method binding is tautological (`transport.rs:174-183`), the "no advisory lock — by design" block (`transport.rs:865-877`).
- Named-reviewer attribution appears in production comments ("Max's blocker", "Eva's call, on record in #proj-git-on-s3", "Sami #2 / Max / Dawn") — `transport.rs:530-537`, `:875-877`; `cas_publish.rs:1171-1176`. This ages badly and encodes decisions in people's names rather than in the argument.
- All public items carry doc comments, satisfying the AGENTS.md rule.
- Two module doc comments are **stale**: `store.rs:25` ("wired in … in a follow-up commit") and `hydrate.rs:24-30` (describes an `#[allow(dead_code)]` that no longer exists).

#### 8. Test conventions

Totals: **99 unit tests** in-module (76 `#[test]`, 23 `#[tokio::test]`), **0** `#[ignore]`d. Plus 2 `#[ignore]`d live E2E tests in `crates/buzz-test-client/tests/e2e_git.rs:195-475`.

| File | `#[test]` | `#[tokio::test]` | Notes |
|---|---|---|---|
| `manifest.rs` | 23 | 0 | schema + predicates + byte-pinning |
| `transport.rs` | 20 | 7 | gzip decode, pkt-line framing, report-status de-framing, NIP-98 host binding, fast-path eligibility |
| `cas_publish.rs` | 16 | 2 | pure composition + 2 that shell out to real `git` |
| `policy.rs` | 14 | 0 | 12 HMAC unit + 2 cross-language bash/Rust |
| `manifest_event.rs` | 9 | 0 | tag shape and filtering |
| `store.rs` | 6 | 4 | 2 pure (`classify_cas`) + 2 config + 4 live-MinIO probes |
| `pack_cache.rs` | 5 | 3 | eviction, symlink guard, flight coalescing |
| `hydrate.rs` | 2 | 6 | predicates + 4 live-MinIO roundtrips |
| `hook.rs` | 1 | 0 | Dockerfile parse assertion |
| `mod.rs` | 0 | 0 | **`require_localhost` is untested** |

Conventions:

- **Environment-gated live tests instead of `#[ignore]`.** `probe_enabled()` reads `BUZZ_GIT_S3_PROBE == "1"` and returns early otherwise, so the tests always "pass" in CI. Duplicated in three files (`store.rs:996-998`, `hydrate.rs:579-581`, `cas_publish.rs:1578-1580`). Cost: a silently skipped test is indistinguishable from a passing one in CI output — only `store.rs:1020-1022` prints a skip notice.
- **Named mutation-bite tests.** Tests are written to fail under a specific regression and say so: `git_nip98_rejects_token_signed_for_wrong_community_host` (`transport.rs:2101-2122`), `same_owner_repo_pointers_do_not_bleed_between_communities` (`manifest.rs:505-524`), `validate_invoked_between_compose_and_put_manifest` (`cas_publish.rs:1833-1878`).
- **Byte-pinning.** Canonical manifest bytes (`manifest.rs:544-568`, `:367-385`) and pkt-line framing against a stated git-2.51 oracle (`transport.rs:2239-2277`) are asserted literally.
- **Cross-language verification.** The bash HMAC is re-implemented inside the test and diffed against Rust (`policy.rs:592-773`) — the only test of the module's most security-critical contract, and it depends on `bash` + `openssl` being present.
- **Real-`git` helpers.** `run_test_git` asserts success and applies `harden_git_env` (`cas_publish.rs:1530-1541`); `build_source_repo` constructs a real repo and extracts its pack (`hydrate.rs:595-640`).
- **Test-only accessors** are gated: `GitPackCache::flight` is `#[cfg(test)]` (`pack_cache.rs:401-418`).
- Test module names are mostly `tests`, except `transport.rs`'s `track_c_tests` (`:1768`) and `store.rs`'s second module `probe` (`:984`), which document *why* they exist.

#### 9. Naming and style conventions

| Convention | Example |
|---|---|
| Spec vocabulary in identifiers | `ParentState`, `CasSuccess`, `CasOutcome`, `Precond`, `PublishLimits` |
| `m_before` / `m_after` mirroring the spec | `cas_publish.rs:1105`, `:1177` |
| Predicate functions named `is_*` | `is_safe_refname`, `is_hex_oid`, `is_pack_key`, `is_manifest_digest`, `is_emittable_ref`, `is_valid_oid`, `fast_path_eligible`, `should_compact`, `compacted_pack_set_is_usable` |
| `*_inner` for the metric-wrapped body | `hydrate_for_read` → `hydrate_for_read_inner` (`hydrate.rs:124-168`); `cas_publish` → `cas_publish_inner` (`:997-1021`) |
| Limit constants `SCREAMING_SNAKE` with a doc paragraph explaining the number | `transport.rs:42-59`, `cas_publish.rs:81-86`, `manifest.rs:34-47` |
| `_`-prefixed fields that exist only for lifetime | `_tempdir`, `_repo`, `_permit`, `_temporary`, `_session_dir` |
| `#[derive(Debug, Clone, Copy)]` on small option structs | `PublishLimits`, `PublishOptions`, `HydrationOptions` |
| Options bundled into a struct rather than long argument lists | `HydrationOptions` (`hydrate.rs:79-89`), `PublishLimits` (`cas_publish.rs:147-155`) |

#### 10. Convention violations and inconsistencies

| # | Issue | Site |
|---|---|---|
| 1 | 17 production `unwrap()`/`expect()` against the AGENTS.md rule (all structurally infallible, but the rule is absolute) | §3 above |
| 2 | Two distinct git-env hardening implementations that claim to match but do not (`GIT_CONFIG_GLOBAL` missing in one) | `transport.rs:294-310` vs `hydrate.rs:451-465` |
| 3 | `read_log_prefix` and `read_prefix` are the same 14-line function in two files | `transport.rs:1141-1155`, `cas_publish.rs:877-891` |
| 4 | `probe_enabled()` triplicated | `store.rs:996`, `hydrate.rs:579`, `cas_publish.rs:1578` |
| 5 | `tenant()` test helper duplicated three ways | `hydrate.rs:536`, `cas_publish.rs:1603`, `transport.rs:1902` |
| 6 | Path-to-`&str` handled inconsistently: `.unwrap()` in one place, a typed error two functions away | `cas_publish.rs:410` vs `:702-705` |
| 7 | Module-wide `#![allow(dead_code)]` in `store.rs` will hide any future genuinely-dead item | `store.rs:25` |
| 8 | `harden_git_env` lives in the HTTP transport module but is consumed by two storage layers | `transport.rs:302` |
| 9 | `stream_git_read` carries an unused `extra_args` parameter | `transport.rs:1418`, called with `&[]` at `:824` |
| 10 | `transport.rs` at 2 288 lines mixes auth extractor, pkt-line codec, three handlers, two subprocess runners, four stream adapters, and the fence | whole file |
