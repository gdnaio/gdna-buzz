## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Security

### 1. Tamper-evidence: what the chain actually proves

| Property | Mechanism (file:line) | Holds? |
|---|---|---|
| Mutating any hashed field of a stored row is detectable | `verify_chain` recomputes `compute_hash` per row and compares to stored `hash` (`service.rs:197-200`) | Yes — test `verify_detects_tampering_within_a_community` (`service.rs:437-473`) |
| Deleting an interior row is detectable | the next row's `prev_hash` no longer matches its predecessor's `hash` → `ChainViolation` (`service.rs:190-195`) | Yes, **provided** the verified range spans the deletion |
| Deleting/truncating the chain **tail** is detectable | nothing in the crate persists an external head pointer; `log_inner` derives the head from `MAX(seq)` in the table itself (`service.rs:94-101`) | **No** — see §4 |
| Re-ordering rows | `seq` is inside the pre-image (`hash.rs:46`) and rows are read `ORDER BY seq ASC` (`service.rs:172`) | Yes for content, but `seq` gaps are not asserted (§4) |
| Moving a row into another community's chain | `community_id` leads the pre-image (`hash.rs:44-45`) | Yes — test `cross_community_row_does_not_verify` (`service.rs:475-510`) |

### 2. Can an attacker with DB write access rewrite the whole chain?

**Yes.** The chain is keyed only by SHA-256 over data the attacker can also see and
write; there is no secret key, no signature, and no external anchor anywhere in
`crates/buzz-audit/src` (the crate's only crypto import is `sha2` at `hash.rs:2`; no
`hmac`, no signing crate is declared in `crates/buzz-audit/Cargo.toml:11-22`).

Concretely, an attacker who can `UPDATE`/`INSERT` on `audit_log` can:
1. modify or drop entries,
2. recompute each subsequent `hash` with the same public algorithm (`hash.rs:42-73`) —
   all inputs are stored in the row itself,
3. rewrite the chain so `verify_chain` returns `Ok(true)`.

Nothing detects this: there is no signed head, no periodic checkpoint export, no
transparency-log submission, no WORM storage, and no DB-level protection —
`migrations/0001_initial_schema.sql:606-619` creates a plain table with a PK and a
unique index, no triggers, no revoked `DELETE`/`UPDATE` grants, no row-level security.
The crate itself never issues `UPDATE`/`DELETE` (only the `INSERT` at
`service.rs:130-147`), so append-only is a property of *this code path*, not of the
datastore.

The relay uses the **same** database credentials for the audit pool as for everything
else (`crates/buzz-relay/src/main.rs:322-328` connects with `config.database_url`), so
a compromised relay process holds write access to the audit table.

Accurate statement of the guarantee: the chain gives **tamper evidence against actors
who can modify rows but cannot recompute and rewrite the suffix** (e.g. partial
corruption, a single UPDATE, restore of a stale row, accidental edit) and against
**cross-tenant row replay**. It does not give tamper-*proofing* against a full-DB
adversary.

### 3. Hash construction weaknesses

- **Unframed concatenation.** Fields are appended with no length prefixes and no
  domain separator (`hash.rs:45-71`). Only the two optional fields get a 1-byte
  presence tag (`hash.rs:55`, `:58`, `:62`, `:65`). Because `action` (field 4) is a
  variable-length string immediately followed by the `actor_pubkey` presence tag, and
  `object_id` is variable-length UTF-8 followed by the canonical-JSON `detail` string,
  the boundaries between adjacent variable-length fields are not cryptographically
  marked. Practical exploitability is limited by (a) `action` being drawn from a fixed
  11-element set (`action.rs:37-47`) — an attacker cannot choose it freely, and
  (b) `detail` always starting with a JSON-legal character. Still, "distinct field
  tuples ⇒ distinct pre-images" is not guaranteed by construction, only by the
  restricted value domains. A length-prefixed or tagged encoding would close it.
- **Length-extension.** SHA-256 is Merkle–Damgård and therefore length-extendable, but
  the digest is not used as a MAC and no secret is prefixed, so length extension buys
  an attacker nothing here: any forged successor must still be a row whose *stored*
  fields recompute to the digest (`service.rs:197-200`). No `hash.rs` code accepts an
  externally supplied digest as a pre-image continuation.
- **`prev_hash` length is unchecked.** `compute_hash` feeds whatever bytes are in the
  `Option<Vec<u8>>` (`hash.rs:69`); the column is untyped `BYTEA`
  (`migrations/0001_initial_schema.sql:610`). A row with a 5-byte `prev_hash` hashes
  fine; only the link check would catch it, and only if the predecessor is in range
  (`service.rs:192`).
- **No depth limit on `canonical_json` recursion** (`hash.rs:80-116`). `detail` is
  caller-supplied JSON from the relay (`crates/buzz-relay/src/handlers/event.rs:566-569`,
  `crates/buzz-relay/src/api/media.rs:431-435`); a deeply nested value would recurse
  proportionally to nesting depth. Both current producers build shallow literals, so
  this is a latent rather than reachable issue.

### 4. Deletion / truncation detectability

| Scenario | Detected? | Why (file:line) |
|---|---|---|
| Interior row deleted, range covers the hole | Yes — `ChainViolation` | `service.rs:190-195` |
| Interior row deleted, verification range starts *after* the hole | No | the first row in a range is compared against nothing; `expected_prev` starts as `None` (`service.rs:185-190`) |
| Tail rows deleted (chain truncated) | **No** | `verify_chain(from, to)` returns `Ok(true)` for whatever exists; an empty range returns `Ok(false)` not an error (`service.rs:181-183`); there is no stored head/length to compare against |
| Whole community's chain deleted | **No** | subsequent `log` sees no head and starts again at `seq = 1` with `prev_hash = NULL` (`service.rs:103-110`) — indistinguishable from a fresh community |
| Genesis integrity (first row must be `seq = 1` with `prev_hash = NULL`) | Not checked | no such assertion exists in `verify_chain` (`service.rs:160-206`) |
| `seq` contiguity within a range | Not checked | loop iterates rows as returned, never compares `entry.seq` to `previous.seq + 1` (`service.rs:187-203`) |

The `UNIQUE (community_id, hash)` index (`migrations/0001_initial_schema.sql:619`)
prevents storing two identical digests in one community, which blocks a naive
duplicate-row replay.

### 5. Canonical-serialization determinism

`canonical_json` sorts object keys via `BTreeMap<&str, &Value>` (`hash.rs:86`),
preserves array order (`hash.rs:101-113`), and delegates scalars to `serde_json`
(`hash.rs:114`). Determinism therefore depends on `serde_json`'s scalar rendering
(notably float formatting) being stable across versions — the doc claims stability
"across machines and Rust versions" (`hash.rs:39-41`) but nothing in the crate pins or
tests float formatting. Sorting is by `&str` byte order, which is well-defined.
Timestamp determinism is handled separately by `to_storage_precision`
(`hash.rs:22-24`), and the RFC3339 variable-width trap is documented and tested
(`hash.rs:167-183`).

Serialization failure is a hard error rather than a silent empty value
(`hash.rs:67`, `error.rs:39-40`) — important because an "empty on failure" fallback
would let two different `detail` payloads share a digest.

### 6. Lock-release safety on panic

`log` wraps the append in `AssertUnwindSafe(...).catch_unwind()` (`service.rs:67-69`),
runs the unlock unconditionally (`service.rs:71-74`), then re-raises any captured panic
with `resume_unwind` (`service.rs:76-79`). Residual risks visible in code:

- The unlock's own error is discarded (`let _ =`, `service.rs:71`). If the unlock fails
  (connection already broken), the connection returns to the pool still holding the
  advisory lock for that community, and subsequent `log` calls for that community would
  block until the session ends.
- **Future cancellation** (the `log` future dropped mid-await, e.g. by a `select!` or
  task abort) bypasses the unlock entirely — `catch_unwind` covers unwinding, not
  cancellation. In the relay this path is single-task and not raced
  (`crates/buzz-relay/src/state.rs:657-672`), so it is latent.
- `AssertUnwindSafe` is an assertion, not a proof: state mutated before a panic
  (`audit_entry` is local; the transaction is dropped and rolls back) is confined to the
  call, so the assertion looks sound for the current body (`service.rs:82-152`).
- No test induces a panic inside `log_inner`, so the branch is untested.

### 7. Tenant isolation of chains

- `community_id` is the first hashed field (`hash.rs:44-45`) and leads the PK
  (`migrations/0001_initial_schema.sql:616`).
- Every statement filters on `community_id`: head read (`service.rs:96`), verify
  (`service.rs:171`), list (`service.rs:223`). No query in the crate reads across
  communities and there is no "all communities" API.
- Input type is `CommunityId` (`entry.rs:57`), a `buzz-core` newtype with no
  `Deserialize` (`crates/buzz-core/src/tenant.rs:37`), and `NewAuditEntry` is
  intentionally non-deserializable (`entry.rs:46-51`). `buzz-core` states plainly that
  this is a lint-and-review fence, not a compiler fence, because
  `CommunityId::from_uuid` is public (`crates/buzz-core/src/tenant.rs:19-25`).
- Error text is sanitized so a failure while verifying one tenant cannot name another:
  variants hold only `seq` (`error.rs:22-32`), asserted by
  `audit_error_text_carries_no_community_id_or_constraint` (`error.rs:58-107`). Note the
  `Database(sqlx::Error)` variant is *not* covered by that test (it iterates only the
  three domain variants, `error.rs:79-83`) — a Postgres error string can carry the
  constraint name `audit_log_pkey` and would render through `database error: {0}`
  (`error.rs:14-15`). Mitigated by the fact that audit errors are logged
  operator-side, never returned to clients (`error.rs:5-7`;
  `crates/buzz-relay/src/state.rs:1201-1203`).
- The advisory lock is per-community precisely to avoid a cross-tenant timing oracle
  (`service.rs:25-28`). Residual: `hashtextextended` can collide two community keys onto
  one lock key; the code has no collision handling, so colliding tenants would serialize
  (a timing side channel, not a data leak).

### 8. `unsafe` code

None. `#![deny(unsafe_code)]` at `crates/buzz-audit/src/lib.rs:1`, and a grep for
`unsafe` across `crates/buzz-audit/src` matches only that lint line.

### 9. Input validation gaps

| Input | Validation present | Gap |
|---|---|---|
| `actor_pubkey` | none | any length accepted (`entry.rs:61`, bound at `service.rs:142`); a 32-byte pubkey is a caller convention only. Relay passes `hex::decode(...).ok()` so a malformed hex actor silently becomes `None` (`crates/buzz-relay/src/handlers/event.rs:566`) |
| `object_id` | none | unbounded `TEXT` (`entry.rs:63`, `migrations/0001_initial_schema.sql:613`) |
| `detail` | none | unbounded JSONB, arbitrary nesting; documented-but-unenforced "no secrets" rule (`entry.rs:64-71`) |
| `action` on read | `FromStr` over the 11 known strings, else `UnknownAction` (`service.rs:240-243`) | one bad row fails an entire read/verify call (`service.rs:234`) |
| `from_seq`/`to_seq`/`limit` | none | negative or inverted ranges are passed straight to SQL (`service.rs:176-177`, `:229-230`); an inverted range simply yields no rows → `Ok(false)`, and an unbounded `limit` is the caller's responsibility |
| `community_id` | type-level only | see the fence caveat in §7 |

Positive: all values are bound as parameters, so no SQL injection surface — the only
string interpolation is the advisory-lock key, itself passed as `$1`
(`service.rs:58-60`).

### 10. Secrets handling

The crate reads no credentials and logs no payloads: `log`'s span skips the entry
(`service.rs:52`), the `debug!` records only `seq` (`service.rs:128`), and the
unknown-action `warn!` omits the offending string (`service.rs:241`). The only
environment read is `DATABASE_URL` inside a test helper (`service.rs:275-279`).
