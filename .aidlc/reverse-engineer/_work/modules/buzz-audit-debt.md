## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Technical Debt

### Explicit debt markers

None. A case-insensitive search for `TODO`, `FIXME`, `HACK`, `XXX`, `todo!`,
`unimplemented!` across `crates/buzz-audit/src` returns **zero matches**. All debt below
is inferred from reading the code, not from author annotations.

### Complexity hotspots

| Item | Size / shape | Note |
|---|---|---|
| `AuditService::log` + `log_inner` | `service.rs:53-152` (~95 lines across two fns) | The most intricate control flow in the crate: pinned connection + advisory lock + `catch_unwind` + transaction + `resume_unwind`. Four interacting concerns in one place, no test for the panic branch |
| `compute_hash` | `hash.rs:42-73` (32 lines, 8 sequential `update` groups) | Straight-line but order-critical: any reordering silently invalidates every existing chain (warned at `hash.rs:28`). No golden/known-answer test pins the digest — the tests only assert determinism and per-field sensitivity (`hash.rs:152-254`), so an accidental field reorder would still pass CI |
| `canonical_json` | `hash.rs:80-116` | Hand-rolled recursive serializer with manual string building and `first` flags for both objects and arrays; duplicated comma logic in two branches. No recursion-depth guard |
| `verify_chain` | `service.rs:160-206` | Loads the whole `[from_seq, to_seq]` range into memory via `fetch_all` (`service.rs:178`) before walking it — memory grows linearly with the requested range; no streaming/cursor variant |
| `get_entries` | `service.rs:212-235` | Same `fetch_all` pattern; `limit` is caller-supplied with no cap (`service.rs:216`, `:230`) |

### Correctness/robustness gaps (see Security aspect for the threat framing)

1. **No golden-hash regression test.** Nothing asserts a fixed expected digest for a
   fixed entry, so the "field order is frozen" invariant (`hash.rs:28`) is unenforced by
   tests. `deterministic` (`hash.rs:152-157`) only compares the function to itself.
2. **`verify_chain` cannot detect tail truncation or a missing genesis.** No stored head
   pointer; the first row of a range is not validated against `seq = 1`/`GENESIS_HASH`,
   and `seq` contiguity is never asserted (`service.rs:185-203`).
3. **Empty range returns `Ok(false)`, not a distinct outcome.** `Ok(false)` conflates
   "nothing to verify" with a caller-visible boolean result (`service.rs:181-183`), so a
   caller cannot distinguish "no entries" from "verified nothing meaningful" without a
   second query. No enum/`Option` return.
4. **Advisory unlock errors swallowed.** `let _ = sqlx::query(...unlock...)`
   (`service.rs:71`) — a failed unlock leaves the lock held on a pooled connection with
   no log line and no metric.
5. **Cancellation leaks the lock.** `catch_unwind` covers unwinding only; dropping the
   `log` future skips the unlock (`service.rs:67-74`).
6. **Advisory-lock key collisions unhandled.** `hashtextextended` maps text → `bigint`
   (`service.rs:59`); two communities can collide and serialize each other, defeating
   the per-tenant design goal stated at `service.rs:25-28`. No detection or mitigation.
7. **`prev_hash` length never validated** (`hash.rs:69`; column is untyped `BYTEA`,
   `migrations/0001_initial_schema.sql:610`).
8. **Unframed hash pre-image.** Adjacent variable-length fields (`action`, `object_id`,
   canonical `detail`) have no length prefixes (`hash.rs:52-67`); safety currently rests
   on the restricted value domains rather than on the encoding.
9. **One unparsable `action` row fails an entire read.** `get_entries` collects
   `Result` (`service.rs:234`) and `verify_chain` decodes before checking
   (`service.rs:188`), so a single legacy/foreign row makes the whole call error with
   `UnknownAction` — there is no skip-and-report path. The DB column has no CHECK
   constraint (`migrations/0001_initial_schema.sql:611`), so such rows are insertable.
10. **`detail` "no secrets" rule is documentation only** (`entry.rs:64-71`) — no
    redaction, key denylist, or size limit.
11. **Untyped sqlx queries throughout.** `sqlx::query` + runtime `Row::get`
    (`service.rs:94`, `:130`, `:166`, `:218`, `:238-255`) instead of the compile-time
    checked macros, so a column rename in a future migration fails at runtime rather
    than at build time.

### Dead / unused code

| Item | Evidence |
|---|---|
| `hex` dependency | Declared at `crates/buzz-audit/Cargo.toml:21`; grep for `hex::` in `crates/buzz-audit/src` returns nothing |
| `tokio` as a normal dependency | `crates/buzz-audit/Cargo.toml:13`, but used only in `#[cfg(test)]` code (`service.rs:265`, `#[tokio::test]` at `:318`–`:512`) — belongs in `[dev-dependencies]` |
| `verify_chain` / `get_entries` | No production caller in the repo; only this crate's `#[ignore]` tests (`service.rs:368`, `:417`, `:427`, `:468`, `:508`, `:523`) and relay tests (`crates/buzz-relay/src/handlers/event.rs:1906-1952`). `buzz-admin` declares the dependency (`crates/buzz-admin/Cargo.toml:20`) but `crates/buzz-admin/src` contains no `audit` reference — the operator verification surface described in the docs does not exist yet |
| 9 of 11 `AuditAction` variants | Only `EventCreated` (`crates/buzz-relay/src/handlers/event.rs:560`) and `MediaUploaded` (`crates/buzz-relay/src/api/media.rs:428`) are produced in production. `EventDeleted`, `ChannelCreated`, `ChannelUpdated`, `ChannelDeleted`, `MemberAdded`, `MemberRemoved`, `AuthSuccess`, `AuthFailure`, `RateLimitExceeded` are defined (`action.rs:11-28`) but never logged |
| `to_storage_precision` public but unexported | `pub` at `hash.rs:22` yet absent from the root re-export list (`lib.rs:34`); reachable only as `buzz_audit::hash::to_storage_precision` |
| `AuditEntry`'s `Serialize`/`Deserialize` derives | `entry.rs:13` — no code in this crate or its consumers serializes an `AuditEntry` (relay never sends one on the wire); unverified whether any downstream needs it |

### Test coverage gaps

19 tests total: 13 `#[test]` + 6 `#[tokio::test]`, all 6 async ones
`#[ignore = "requires Postgres"]` (`service.rs:319`, `:339`, `:377`, `:438`, `:476`,
`:513`). Uncovered behaviour:

| Gap | Why it matters |
|---|---|
| Panic path through `catch_unwind`/`resume_unwind` | The documented single-writer safety property (`service.rs:64-79`) is never exercised |
| Advisory lock behaviour (contention, per-community independence at the lock level) | Tests serialize themselves with an in-process `Mutex` (`service.rs:263-267`), so the Postgres lock is never actually contended |
| Concurrent `log` calls / duplicate-`seq` race | No test attempts two simultaneous appends to one community |
| Tail truncation and genesis validation | No test — consistent with the code not implementing the check |
| `ChainViolation` variant | Constructed at `service.rs:193` but no test asserts it; the tampering test hits `HashMismatch` instead (`service.rs:472`) |
| `UnknownAction` on read | Constructed at `service.rs:242`; no test inserts a bogus `action` string |
| `Database` / `Serialization` error variants | Never asserted; the `Database` variant is also excluded from the error-sanitization test (`error.rs:79-83`) |
| `get_entries` pagination edges (`limit = 0`, negative `from_seq`, `limit` boundary) | Only the happy path is tested (`service.rs:425-433`) |
| Non-trivial `detail` payloads (nested objects, arrays, floats, unicode keys) | `canonical_json_key_order_is_stable` uses a flat 3-key object (`hash.rs:266-271`); the array branch (`hash.rs:101-113`) is untested |
| Whole-crate CI signal without Postgres | Only 13 tests run in `just test-unit`; every chain-behaviour assertion sits behind `--ignored` |

### Deprecated API usage

None found. `chrono` `trunc_subsecs`/`to_rfc3339` (`hash.rs:23`, `:49`), `sha2` 0.11
`Digest` (`hash.rs:2`), `sqlx` 0.9 `Acquire`/`Row` (`service.rs:3`), and `thiserror` 2
(`error.rs:1`) are all current for the pinned workspace versions
(`Cargo.toml:52-54`, `:85`, `:90`, `:96`). No `#[deprecated]` items and no
`#[allow(deprecated)]` in the crate.

### Documentation drift (repo docs vs. this crate)

`ARCHITECTURE.md:493-505` describes an older shape of this crate and is now wrong on
five points: the action count (10 vs 11), the hash pre-image field set (`event_id`,
`event_kind`, `channel_id` do not exist on `AuditEntry`, `entry.rs:14-37`), the
non-existent `AuditError::AuthEventForbidden` variant (`error.rs:12-41`), "64 zeros"
for a 32-byte constant (`hash.rs:9`), and an implied single global advisory lock
(actually per-community, `service.rs:29`, `:58`). Details in the Features aspect.
