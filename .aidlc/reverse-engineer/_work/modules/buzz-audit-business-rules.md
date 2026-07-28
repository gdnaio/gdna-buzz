## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Business Rules

---

### BR-1 — Hash pre-image composition (the crate's core invariant)

`compute_hash` (`crates/buzz-audit/src/hash.rs:42-73`) feeds a single `Sha256`
in exactly this order. **9 hasher updates for the always-present fields, plus a
1-byte presence tag ahead of each of the two optional fields.**

| # | Field fed | Byte encoding | Code (file:line) |
|---|---|---|---|
| 1 | `community_id` | `Uuid::as_bytes()` → 16 raw big-endian UUID bytes | `hash.rs:45` |
| 2 | `seq` | `i64::to_be_bytes()` → 8 bytes, big-endian | `hash.rs:46` |
| 3 | `created_at` | `to_storage_precision(created_at).to_rfc3339().as_bytes()` — RFC3339 UTF-8, variable length (chrono emits 0/3/6/9 fractional digits) | `hash.rs:47-51` |
| 4 | `action` | `AuditAction::as_str().as_bytes()` — the snake_case string (`action.rs:35-49`) | `hash.rs:52` |
| 5 | `actor_pubkey` | presence tag `[1u8]` then raw pubkey bytes; or `[0u8]` alone when `None` | `hash.rs:53-59` |
| 6 | `object_id` | presence tag `[1u8]` then `id.as_bytes()` (UTF-8); or `[0u8]` alone when `None` | `hash.rs:60-66` |
| 7 | `detail` | `canonical_json(detail)?.as_bytes()` — sorted-key JSON, UTF-8; error propagates | `hash.rs:67` |
| 8 | `prev_hash` | the 32 stored bytes when `Some`; `GENESIS_HASH` (32 zero bytes) when `None` | `hash.rs:68-71` |

Digest is `hasher.finalize().into()` → `[u8; 32]` (`hash.rs:72`).

Encoding notes verified in code:
- No length prefixes or field separators other than the two 1-byte presence tags —
  the concatenation is otherwise unframed (`hash.rs:45-71`).
- The pre-image contains **no** `event_id`, `event_kind`, or `channel_id` field; the
  relay carries those inside `detail` (`crates/buzz-relay/src/handlers/event.rs:566-569`),
  so they are covered transitively through field 7.
- `community_id` is hashed **first**, explicitly to bind chain identity to the tenant
  (`hash.rs:28-30`, `hash.rs:44`). Test: `community_id_is_part_of_identity`
  (`hash.rs:216-224`).
- Trigger: every append (`service.rs:126`) and every verification step
  (`service.rs:197`) run the same function, so write and read pre-images cannot
  diverge by construction.

### BR-2 — `created_at` must be at storage precision before hashing

`to_storage_precision` truncates to microseconds (`hash.rs:22-24`), matching what
`TIMESTAMPTZ` round-trips (`hash.rs:13-18`). Enforced twice:
- write path stamps `log_timestamp()` = `to_storage_precision(Utc::now())`
  (`service.rs:21-23`, used at `service.rs:112`);
- `compute_hash` re-normalizes at the single consuming point so a caller that forgets
  cannot split the write/read pre-image (`hash.rs:32-37`, `hash.rs:47-51`).

Rationale recorded in code: RFC3339 sub-second width follows the value, so a
nanosecond timestamp and its microsecond truncation are different strings and every
entry would fail `verify_chain` with `HashMismatch`
(`hash.rs:167-183`, assertions at `hash.rs:177-182`). Unit guard without Postgres:
`log_timestamp_carries_no_sub_microsecond_digits` (`service.rs:283-292`).

### BR-3 — Canonical JSON determinism for `detail`

`canonical_json` recursively re-serializes objects with keys sorted through a
`BTreeMap<&str, &Value>` (`hash.rs:85-100`), arrays in order (`hash.rs:101-113`), and
scalars via `serde_json::to_string` (`hash.rs:114`). Serialization failure is
propagated, never substituted with an empty placeholder (`hash.rs:39-41`, `:77-79`,
`:94`, `:96`, `:109`). Trigger: any hash computation with a non-null `detail`.
Test: `canonical_json_key_order_is_stable` (`hash.rs:266-271`).

Not normalized: numeric formatting is whatever `serde_json` emits for the parsed
`Value`, and duplicate JSON keys are already collapsed by `serde_json`'s map before
this function sees them (`hash.rs:114`).

### BR-4 — Presence tags disambiguate `None` from `Some(empty)`

`actor_pubkey` and `object_id` each get a leading `1`/`0` byte (`hash.rs:53-66`), so
`Some(vec![])` and `None` hash differently. Test:
`presence_tag_distinguishes_none_from_empty` (`hash.rs:256-264`). Without the tags the
unframed concatenation would collide.

### BR-5 — Chain linking is per-community and read inside the transaction

`log_inner` reads the head of *this* community's chain only:
`SELECT seq, hash FROM audit_log WHERE community_id = $1 ORDER BY seq DESC LIMIT 1`
(`service.rs:94-101`). New entry takes `seq = prev_seq + 1` (`service.rs:110`) and
`prev_hash = Some(head.hash)` (`service.rs:104-107`). Head read and insert share one
transaction (`service.rs:87`, `:146`, commit `:149`). Trigger: every `log` call.

### BR-6 — Genesis handling

When no row exists for the community, the head query yields `None` → `(prev_seq, prev_hash) = (0, None)`
(`service.rs:103-109`), so the first entry gets `seq = 1` and `prev_hash = NULL`
stored (`service.rs:110`, bind at `service.rs:140`). `compute_hash` substitutes the
32-zero-byte `GENESIS_HASH` for the `None` (`hash.rs:9`, `:68-71`). Test:
`community_chain_starts_at_seq_1_with_null_prev` (`service.rs:318-336`).

### BR-7 — Single-writer guarantee is **per-community**, not global

`log` builds a namespaced lock key `format!("buzz_audit:{community_id}")`
(`service.rs:29`, `:58`) and takes `SELECT pg_advisory_lock(hashtextextended($1, 0))`
(`service.rs:59-62`) on the same pooled connection later used for the transaction
(`service.rs:54`, `:67`). Documented intent: two communities never serialize each
other's writes, avoiding both a bottleneck and a cross-tenant timing oracle
(`service.rs:25-28`, `lib.rs:13-15`). Lock ordering: acquire *before* `begin`, release
*after* commit (`service.rs:49-51`).

Because `hashtextextended` maps a text key into `bigint`, two distinct community keys
can in principle collide onto the same advisory-lock key (no collision handling exists
in code) — that would over-serialize, not corrupt.

### BR-8 — Lock release on every path including panic

`log_inner` is wrapped in `AssertUnwindSafe(...).catch_unwind().await`
(`service.rs:67-69`); the unlock query runs unconditionally afterwards with its result
discarded (`service.rs:71-74`); a captured panic is re-raised with
`std::panic::resume_unwind` after the unlock (`service.rs:76-79`). Trigger: any panic
or error inside the append. Caveats visible in code: an `Err` from the *unlock* query
itself is swallowed (`let _ =`, `service.rs:71`), and cancellation of the `log` future
before `catch_unwind` resolves would drop the connection without an explicit unlock
(session-scoped locks are then released by Postgres when the session ends — but a
pooled connection is reused rather than closed, so that release is not immediate).
No test covers the panic path (no test in `service.rs` induces a panic inside
`log_inner`).

### BR-9 — Verification algorithm

`verify_chain(community, from_seq, to_seq)` (`service.rs:160-206`):
1. `SELECT` the full projection for `community_id = $1 AND seq BETWEEN $2 AND $3
   ORDER BY seq ASC` (`service.rs:166-179`).
2. Empty result → `Ok(false)` (`service.rs:181-183`).
3. Walk rows in ascending `seq`, decoding each with `row_to_audit_entry`
   (`service.rs:188`).
4. For every row after the first in the range, require
   `entry.prev_hash == Some(previous_row.hash)` else `Err(ChainViolation { seq })`
   (`service.rs:190-195`).
5. Recompute `compute_hash(&entry)` and require equality with the stored `hash`, else
   `Err(HashMismatch { seq })` (`service.rs:197-200`).
6. All rows pass → `Ok(true)` (`service.rs:205`).

Rules this algorithm does **not** enforce (verified absent from `service.rs:160-206`):
- The first row of the range is not checked against `seq = 1` or against
  `GENESIS_HASH` — a range starting mid-chain has its `prev_hash` accepted unchecked.
- `seq` contiguity is not asserted; a gap between two rows is invisible unless the
  `prev_hash` link also breaks (a deleted row *does* break the link, so an interior
  deletion is caught, but a truncation of the chain **tail** is not).
- The stored `hash` uniqueness/`community_id` label are not cross-checked beyond what
  the recomputation implicitly covers.

Tests: `chain_links_within_one_community` (`service.rs:338-374`),
`verify_detects_tampering_within_a_community` (`service.rs:437-473`, asserts
`HashMismatch` at the tampered `seq`), `verify_empty_range_is_false` (`service.rs:512-526`).

### BR-10 — Cross-community replay is rejected

Because `community_id` is in the pre-image (BR-1), a row copied from community A into
community B recomputes to a different digest and fails as `HashMismatch`. Proven by
`cross_community_row_does_not_verify` (`service.rs:475-510`), which inserts A's hash
under B's id and asserts `Err(AuditError::HashMismatch { seq: 1 })` (`service.rs:508-509`).

### BR-11 — Per-community chain scoping is total across all three DB paths

Every statement filters on `community_id`: head read (`service.rs:96`), verify
(`service.rs:171`), list (`service.rs:223`). Schema reinforces it with
`PRIMARY KEY (community_id, seq)` and `UNIQUE (community_id, hash)`
(`migrations/0001_initial_schema.sql:616`, `:619`). Isolation test:
`chains_are_independent_per_community` (`service.rs:376-435`) — interleaved A/B writes
each start at `seq 1` and link only within their own chain (`service.rs:411-414`), and
`get_entries` scoped to A returns only A rows (`service.rs:425-433`).

### BR-12 — Provenance rule: `community_id` cannot come from client input

`NewAuditEntry.community_id` is typed `buzz_core::CommunityId` (`entry.rs:57`), a
newtype with no `Deserialize` (`crates/buzz-core/src/tenant.rs:37`), and
`NewAuditEntry` itself is deliberately not `Serialize`/`Deserialize`
(`entry.rs:46-51`). The raw `Uuid` is only unwrapped at the DB boundary
(`service.rs:89-91`). `buzz-core` documents this as a lint-and-review fence rather
than a compiler fence, since `CommunityId::from_uuid` is public
(`crates/buzz-core/src/tenant.rs:19-25`, `:44-47`).

### BR-13 — Unknown action strings fail closed on read

`row_to_audit_entry` parses the stored `action` via `FromStr` and maps any failure to
`AuditError::UnknownAction` after a `warn!` (`service.rs:238-243`; parser at
`action.rs:75-81`). Trigger: reading a row whose `action` string is not one of the 11.
Consequence: one unrecognised row fails the entire `verify_chain`/`get_entries` call
(`service.rs:188`, `:234`). The DB column is `VARCHAR(64)` with no CHECK
(`migrations/0001_initial_schema.sql:611`), so such rows are insertable out-of-band.

### BR-14 — Which events are refused: nothing is refused *inside this crate*

There is no kind-based, ephemeral-based, or content-based rejection anywhere in
`crates/buzz-audit/src`. `log` accepts any `NewAuditEntry` (`service.rs:53`); the only
error sources are DB failures, JSON canonicalisation failures, and (on read) unknown
actions. No `AuditError::AuthEventForbidden` variant exists (`error.rs:12-41`), and
grep across the repo finds that identifier only in `ARCHITECTURE.md:505`.

The refusals described in ARCHITECTURE.md live in the relay, upstream of the audit
sink:
- `KIND_AUTH` submissions are rejected during ingest with
  `IngestError::Rejected("invalid: AUTH events cannot be submitted")`
  (`crates/buzz-relay/src/handlers/ingest.rs:1438-1442`), so no audit entry is ever
  enqueued for them.
- Audit enqueue happens only from the persistent-event path
  (`crates/buzz-relay/src/handlers/event.rs:335`, `:486`, helper at `:540-577`) and
  from media upload (`crates/buzz-relay/src/api/media.rs:422-442`); ephemeral events
  are dispatched on a separate branch and never reach `enqueue_event_created_audit`.

### BR-15 — Append-only by absence of any other write

The crate issues exactly one mutating statement, the `INSERT`
(`service.rs:130-147`). No `UPDATE`, `DELETE`, or `TRUNCATE` appears in
`crates/buzz-audit/src`. Append-only is therefore a property of this code path only —
it is not enforced by DB privileges, triggers, or constraints in
`migrations/0001_initial_schema.sql:606-619`.

### BR-16 — `detail` must not carry secrets (documented, unenforced)

`NewAuditEntry.detail` is documented as never bearer-token material; callers must not
write tokens or passwords (`entry.rs:64-71`). This is a doc-comment obligation only —
no validation, redaction, or size limit exists on the field in
`crates/buzz-audit/src`.
