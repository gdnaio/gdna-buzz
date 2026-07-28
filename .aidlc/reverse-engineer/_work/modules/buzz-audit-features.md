## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Features

### Capability inventory

| Capability | Where | Completeness |
|---|---|---|
| Append an entry to a per-community hash chain | `AuditService::log` (`crates/buzz-audit/src/service.rs:53-80`) + `log_inner` (`:82-152`) | Complete: assigns `seq`, links `prev_hash`, stamps `created_at`, computes hash, inserts in one transaction |
| Per-community write serialization across processes | `pg_advisory_lock(hashtextextended(...))` (`service.rs:56-62`, key at `:29`, `:58`) | Complete for the happy and error paths; unlock errors are discarded (`service.rs:71`) |
| Panic-safe lock release | `catch_unwind` + unconditional unlock + `resume_unwind` (`service.rs:67-79`) | Implemented; **no test exercises it** (no panic-inducing test in `service.rs:258-527`) |
| Deterministic hashing of an entry | `compute_hash` (`hash.rs:42-73`) | Complete; 9-field fixed order, presence tags, canonical JSON |
| Timestamp precision normalization | `to_storage_precision` (`hash.rs:22-24`), `log_timestamp` (`service.rs:21-23`), re-normalization in `compute_hash` (`hash.rs:47-51`) | Complete and double-enforced |
| Canonical JSON serialization | `canonical_json` (`hash.rs:80-116`) | Complete for object/array/scalar; recursion has no depth guard |
| Chain verification over a `seq` range | `verify_chain` (`service.rs:160-206`) | Partial by design: checks links + recomputed hashes inside the range; does not check range start against genesis, does not check `seq` contiguity, does not detect tail truncation |
| Paginated chain read | `get_entries` (`service.rs:212-235`) | Complete: `seq >= from_seq`, `ORDER BY seq ASC`, `LIMIT` |
| Action enum ↔ string round-trip | `as_str` (`action.rs:35-49`), `Display` (`:66-70`), `FromStr` (`:72-82`) | Complete for all 11 variants; unknown strings error |
| Error taxonomy | `AuditError` (`error.rs:12-41`) | 5 variants; see Conventions |
| Genesis sentinel | `GENESIS_HASH` (`hash.rs:9`) | Complete |

### Not implemented / absent (verified by reading, not inferred)

- **No DDL / migrations in the crate.** Stated at `lib.rs:17-18`; table lives in
  `migrations/0001_initial_schema.sql:606-619`.
- **No head/tip accessor.** There is no public `head()`/`latest_seq()`; head is read
  privately inside `log_inner` (`service.rs:94-101`).
- **No count/aggregate, no filter-by-action, no time-range query.** The only reads are
  `verify_chain` and `get_entries`, both keyed on `seq` (`service.rs:160`, `:212`).
- **No external anchoring, signing, or notarization.** No signature, HMAC, Merkle
  root, or export path exists in `crates/buzz-audit/src` (the crate's only crypto
  dependency use is `sha2` at `hash.rs:2`).
- **No retention, archival, or pruning.** No `DELETE`/`TRUNCATE` statement in the
  crate.
- **No batch append.** `log` takes a single `NewAuditEntry` (`service.rs:53`).
- **No retry/backoff on DB failure.** Errors are returned to the caller; the relay
  worker logs and drops them (`crates/buzz-relay/src/state.rs:1199-1207`).
- **No cargo features.** `crates/buzz-audit/Cargo.toml` has no `[features]` section.
- **No integration/`tests/` directory.** All tests are inline `#[cfg(test)]` modules.

### Actions defined vs. actions actually produced

11 variants are defined (`action.rs:8-31`), but repo-wide grep for `AuditAction::`
outside this crate finds only two producers:

| Action | Produced at |
|---|---|
| `EventCreated` | `crates/buzz-relay/src/handlers/event.rs:560` |
| `MediaUploaded` | `crates/buzz-relay/src/api/media.rs:428` |

The remaining 9 (`EventDeleted`, `ChannelCreated`, `ChannelUpdated`, `ChannelDeleted`,
`MemberAdded`, `MemberRemoved`, `AuthSuccess`, `AuthFailure`, `RateLimitExceeded`) have
no production call site in the repo; they appear only in this crate's own tests
(e.g. `service.rs:352`, `:356`, `:452`, `:455`).

### TODO / FIXME / HACK / XXX comments

A case-insensitive search for `todo`, `fixme`, `hack`, `xxx`, `unimplemented!`,
`todo!` across `crates/buzz-audit/src` returns **zero matches**. There are no
deferred-work markers in the crate.

### Documentation-vs-code drift found while reading

| ARCHITECTURE.md claim (line) | Code reality |
|---|---|
| "10 audit actions" (`ARCHITECTURE.md:503`) | 11 variants (`action.rs:8-31`); `MediaUploaded` is missing from the doc list |
| Hash covers `event_id`, `event_kind`, `channel_id` (`ARCHITECTURE.md:501`) | Those fields do not exist on `AuditEntry` (`entry.rs:14-37`); pre-image has `community_id`, `seq`, `created_at`, `action`, `actor_pubkey`, `object_id`, `detail`, `prev_hash` (`hash.rs:45-71`) |
| `AuditError::AuthEventForbidden` returned for `KIND_AUTH` (`ARCHITECTURE.md:505`) | No such variant (`error.rs:12-41`); the refusal is in the relay ingest path (`crates/buzz-relay/src/handlers/ingest.rs:1438-1442`) |
| `GENESIS_HASH` is "64 zeros" (`ARCHITECTURE.md:497`) | `[u8; 32]` of zeros, i.e. 32 bytes = 64 zero hex chars (`hash.rs:9`) |
| "`pg_advisory_lock` before each transaction" implying one lock (`ARCHITECTURE.md:501`) | Lock key is per-community (`service.rs:29`, `:58`) |
