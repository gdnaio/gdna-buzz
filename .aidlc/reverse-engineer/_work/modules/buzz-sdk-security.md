## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: Security

---

### 1. Signing and key handling

| Finding | Detail | File:line |
|---|---|---|
| Builders never sign | All 51 builders return an unsigned `nostr::EventBuilder`; the caller signs with its own `Keys`. Documented as "No keys are held here." | `crates/buzz-sdk/src/lib.rs:11-13` |
| No key material in builder signatures | No builder takes `Keys`, `SecretKey`, or any secret parameter — pubkeys are accepted as hex `&str` only | `builders.rs:219-1823` (all signatures) |
| One signing path in the crate | `nip_oa::compute_auth_tag` takes `&Keys` (owner) and calls `owner_keys.sign_schnorr(&message)`. The `Keys` value is borrowed, used once, never stored or logged | `nip_oa.rs:146-176`, sign at `nip_oa.rs:170` |
| Verification path | `verify_auth_tag` uses `SECP256K1.verify_schnorr` against the x-only owner pubkey extracted from the tag | `nip_oa.rs:237-244` |
| Cryptography used | SHA-256 digest (`nostr::hashes::sha256`) + BIP-340 Schnorr over secp256k1 — current, not deprecated | `nip_oa.rs:22-26`, `109-116` |
| Self-attestation rejected on both sides | `owner == agent` fails at compute (`nip_oa.rs:154-159`) and at verify (`nip_oa.rs:212-217`), so a stolen agent key cannot self-authorize | `nip_oa.rs:154-159`, `212-217` |
| Structure-only fast path is explicit | `parse_auth_tag` performs **no** signature verification and says so; used at MCP startup | `nip_oa.rs:239-251` |
| `verify_auth_tag` returns the owner key | Callers get the attesting owner pubkey back for authorization decisions rather than a bare bool | `nip_oa.rs:179-248` |

**Secret material in memory:** the `zeroize` crate is not a dependency and is not
used anywhere in this crate (`crates/buzz-sdk/Cargo.toml:10-16`). The only place
a secret exists is the borrowed `&Keys` inside `compute_auth_tag`
(`nip_oa.rs:146`) — the SDK creates no secret copies, so there is nothing local
to zeroize; lifetime of the key material is the caller's responsibility. By
contrast, `buzz-core`'s observer encryption path does zeroize its plaintext
(`crates/buzz-core/src/observer.rs:63-68`), so the pattern exists in the
workspace but is not needed here.

The example binary accepts an owner secret key **as a command-line argument**
(`examples/compute_auth_tag.rs:11-21`), which exposes it to the process table
and shell history. It is an example, not a shipped binary, but it is the one
place in the crate where secret handling is user-facing.

---

### 2. Input validation posture

Validation is the crate's primary security function — it is the last gate before
a signature is applied to a wire form the relay will trust.

| Control | Coverage | File:line |
|---|---|---|
| Pubkey format | exactly 64 ASCII hex, lowercased on output — used in 20+ builders. Tests explicitly pin rejection of **over-long** (65-char) pubkeys because the relay's `extract_p_tag_bytes` requires exactly 64 and would silently drop a longer value | `builders.rs:69-77`; test `builders.rs:3613-3618` |
| Event-id format | exactly 64 hex, lowercased | `builders.rs:79-89` |
| Commit id format | exactly 40 or 64 hex — abbreviated refs refused for NIP-34 canonical tags | `builders.rs:59-66` |
| Path-traversal defense on repo ids | `check_repo_id` blocks `..`, leading `.`, and any character outside `[A-Za-z0-9._-]`; enforced both in `build_repo_announcement` and inside `GitRepoCoord::to_a_tag_value`, so a coordinate built directly through the SDK cannot smuggle an invalid `d` value into an `a` tag. Regression test uses `../etc/passwd` | `builders.rs:92-121`, `975-982`; test `builders.rs:3113-3121` |
| Control-character filtering | `reason` codes for identity archival reject any `char::is_control()` | `builders.rs:1706-1721`; test `builders.rs:3790-3795` |
| Size bounds | content caps 60/64 KiB, emoji 64 chars, shortcode 64 bytes, URL 2048 bytes, repo name 128, description 1024, clone URL 512, relay 256, petname 256, reason 64 bytes, contacts 10 000, mentions 50, DM participants 8, clone URLs 5, relays 10 | `builders.rs:35-41`, `152-170`, `751`, `840-919`, `1545-1549`, `1704`; `mentions.rs:38` |
| Encryption-required gate | `build_agent_observer_frame` refuses content that does not pass `content_looks_like_nip44` (length 132–87 472), so plaintext telemetry cannot be published on kind 24200. Dedicated negative test exists | `builders.rs:256-260`; `crates/buzz-core/src/observer.rs:53-55`; test `builders.rs:1914-1924` |
| NIP-70 protection | identity archival requests are always marked `["-"]` (protected) | `builders.rs:1748` |
| Enum-constrained vocabularies | presence status, moderation status/action, channel visibility, observer frame direction are all whitelist-matched | `builders.rs:1571-1584`, `1660-1680`, `615-621`, `251-255` |
| NIP-OA conditions grammar | strict canonical-decimal parser: no whitespace, no empty clauses, no leading zeros, range-bounded, case-sensitive labels — reduces the chance that relay and client disagree on an attestation's scope | `nip_oa.rs:36-107` |
| Lowercase-hex enforcement on auth tags | `parse_auth_tag` rejects uppercase/mixed-case owner pubkey and signature, preventing two encodings of the same tag | `nip_oa.rs:120-122`, `274-296`; test `nip_oa.rs:458-486` |
| UTF-8 boundary safety | `extract_nostr_uris` checks `is_char_boundary` before slicing a fixed-width window; `extract_at_mentions_with_known` uses `get(..)` rather than direct indexing. Both have explicit no-panic tests | `mentions.rs:135-138`, `374-377`; tests `mentions.rs:520-531`, `787-800` |

---

### 3. Validation gaps observed

| Gap | Consequence | File:line |
|---|---|---|
| `build_set_canvas` has no content-length check | a canvas document of unbounded size can be signed; every other content builder caps at 60–64 KiB | `builders.rs:529-532` |
| `build_workflow_approval` note is unbounded | free-text `note` becomes event content with no size check | `builders.rs:1522-1541` |
| PR / PR-update `clone_urls` entries are unvalidated | only list non-emptiness is checked; individual URLs get no scheme or length check (unlike `build_repo_announcement`, which validates each) | `builders.rs:1344-1350`, `1425-1431` vs `868-882` |
| `q`-tag relay hints pass through raw | relay-url hint on applied patches is written verbatim with no scheme/length validation | `builders.rs:1266-1272` |
| NIP-02 `relay_url` scheme unchecked | length-capped at 2048 bytes but any scheme accepted | `builders.rs:785-792` |
| `build_diff_message` accepts 7-char commit SHAs | intentionally looser than `check_commit_hex`; a truncated SHA is ambiguous but is written into the `commit` tag | `builders.rs:322-325` |
| Free-text tag values are not sanitized | `about`, `topic`, `purpose`, `description`, `reason`, `public_reason`, `alt`, `subject`, `branch-name`, labels, and petnames accept any UTF-8 including control characters and newlines (only the NIP-IA `reason` is control-char filtered). Consumers render these | `builders.rs:632-638`, `655`, `664`, `1093`, `1361`, `1607-1609`, `1685-1687` |
| `build_repo_announcement_with_tags` trusts caller tags | every non-`d` tag is preserved verbatim in a read-modify-write update; only the `d` tag is canonicalized | `builders.rs:952-963` |
| `build_profile` does not validate `picture`/`nip05` | any string is accepted into the kind-0 JSON | `builders.rs:537-562` |
| `check_auth_tag_shape` does not validate the conditions element | the builder-side auth-tag check verifies label/pubkey/signature shape but not `auth[2]`, while `nip_oa::parse_auth_tag` does validate conditions — the two entry points differ | `builders.rs:1723-1737` vs `nip_oa.rs:283-286` |

These are documented as observations; several are explicitly delegated to the
relay in the source comments (e.g. `builders.rs:1697-1699` for NIP-IA consent,
`builders.rs:1648-1651` for moderation status/action pairing).

---

### 4. Hardcoded values

| Value | Nature | File:line |
|---|---|---|
| `CUSTOM_EMOJI_SET_D_TAG = "buzz:custom-emoji"` | protocol constant (public) | `builders.rs:503` |
| `MAX_CONTACTS = 10_000`, `MAX_REASON_BYTES = 64`, `MENTION_CAP = 50` | limit constants | `builders.rs:751`, `1704`; `mentions.rs:38` |
| Inline size limits (`64 * 1024`, `60 * 1024`, `2048`, `1024`, `512`, `256`, `128`, `64`, `8`, `5`, `10`) | magic numbers used directly at call sites rather than named constants | `builders.rs:227`, `314`, `157`, `856`, `875`, `907`, `845`, `1092`, `1546`, `868`, `900` |
| NIP-OA preimage prefix `"nostr:agent-auth:"` | protocol constant | `nip_oa.rs:110` |
| Bech32 window `"nostr:npub1"` + 58 chars | protocol constant | `mentions.rs:354-355` |
| Test key material: spec owner/agent pubkeys, spec signature, `0x…02` secret key | **test-only** constants, no production use | `nip_oa.rs:303-310`; `builders.rs:3701-3705`, `3756-3759` |

No credentials, tokens, URLs, or hostnames are hardcoded outside tests.

---

### 5. Unsafe code

`#![deny(unsafe_code)]` at the crate root (`crates/buzz-sdk/src/lib.rs:1`).
A search for `unsafe` across `src/` and `examples/` returns only that lint
attribute — **zero unsafe blocks**, confirmed.

### 6. Panic surface

No `unwrap()` or `expect()` exists outside `#[cfg(test)]` code (verified by scan
of all three source files). `GitAppliedPatchRef::parse` uses
`unwrap_or_default()` (`builders.rs:1145`), which cannot panic. Slicing in
`GitAppliedPatchRef::parse` (`builders.rs:1157`, `1161-1162`) and
`strip_code_regions` operates on byte offsets derived from `find`/`rfind` and
`char_indices`, so indices are char-boundaries by construction; the one
fixed-width window in the crate is explicitly boundary-guarded
(`mentions.rs:374-377`). The example binary panics by design on malformed CLI
arguments (`examples/compute_auth_tag.rs:21-27`).
