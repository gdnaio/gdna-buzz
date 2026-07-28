## Module: buzz-core (`crates/buzz-core`)

### Aspect: Security

---

### 1. SSRF helper — `is_private_ip` exact coverage

Function: `crates/buzz-core/src/network.rs:25-53`. Purpose per doc: "webhook targets must not resolve to these addresses" (`network.rs:9-11`).

#### IPv4 branch (`network.rs:26-40`)

| Check as coded | CIDR / value covered | Line |
|---|---|---|
| `v4.is_loopback()` | `127.0.0.0/8` | `:29` |
| `v4.is_private()` | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` (std semantics) | `:30` |
| `v4.is_link_local()` | `169.254.0.0/16` | `:31` |
| `octets[0] == 0` | `0.0.0.0/8` (whole first octet, not just `0.0.0.0`) | `:32` |
| `v4.is_broadcast()` | `255.255.255.255` exactly | `:33` |
| `octets[0] == 100 && (octets[1] & 0xC0) == 64` | `100.64.0.0/10` — CGNAT, RFC 6598; comment cites cloud metadata routing risk | `:36` |
| `octets[0] == 198 && (octets[1] & 0xFE) == 18` | `198.18.0.0/15` — benchmarking, RFC 2544 | `:38` |

Boundary tests pin the CGNAT and benchmarking edges: `100.64.0.1` and `100.127.255.254` blocked, `100.63.255.255` and `100.128.0.0` allowed (`network.rs:151-171`); `198.18.0.1` and `198.19.255.254` blocked, `198.17.255.255` and `198.20.0.0` allowed (`network.rs:173-192`).

#### IPv6 branch (`network.rs:41-52`)

| Check as coded | Range covered | Line |
|---|---|---|
| `v6.to_ipv4_mapped()` → recurse into the IPv4 rules | `::ffff:0:0/96` (IPv4-mapped) | `:42-44` |
| `v6.is_loopback()` | `::1` | `:45` |
| `v6.is_unspecified()` | `::` | `:46` |
| `segments()[0] & 0xfe00 == 0xfc00` | `fc00::/7` — ULA | `:47` |
| `segments()[0] & 0xffc0 == 0xfe80` | `fe80::/10` — link-local | `:48` |
| `segments()[0] & 0xff00 == 0xff00` | `ff00::/8` — multicast | `:49` |
| `segments()[0] == 0x2001 && segments()[1] == 0x0db8` | `2001:db8::/32` — RFC 3849 documentation | `:51` |

#### Ranges **not** covered (verified absent from the code)

| Range | Status in code |
|---|---|
| IPv4 `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24` (RFC 5737 documentation) | not checked |
| IPv4 `192.88.99.0/24` (6to4 relay anycast) | not checked |
| IPv4 `224.0.0.0/4` multicast, `240.0.0.0/4` reserved | not checked (IPv6 multicast *is* checked; IPv4 multicast is not) |
| IPv6 `64:ff9b::/96` (NAT64 well-known prefix), `2002::/16` (6to4), `100::/64` (discard-only) | not checked |
| IPv6 IPv4-**compatible** addresses (`::a.b.c.d`, deprecated) | `to_ipv4_mapped()` matches only `::ffff:*`, so `::10.0.0.1` is not re-checked against IPv4 rules |
| DNS-rebinding / TOCTOU between resolution and connect | out of scope for this pure predicate — the caller must resolve then check then connect |

The function is a pure classifier over an already-resolved `IpAddr`; it performs no DNS resolution, so protection depends entirely on the caller checking every resolved address and pinning the connection.

---

### 2. Signature / ID verification

| Control | Implementation | file:line |
|---|---|---|
| Event ID must equal the canonical hash | `event.verify_id()`; on failure the expected ID is recomputed with `EventId::new(pubkey, created_at, kind, tags, content)` and both values are surfaced in the error | `verification.rs:12-25` |
| Schnorr signature check | `event.verify_signature()`, evaluated only after the ID check | `verification.rs:27-29` |
| CPU-bound warning | module doc mandates `tokio::task::spawn_blocking` in async contexts (this crate cannot enforce it — it has no runtime) | `verification.rs:1-2`, `:9` |
| Underlying crypto | delegated to `nostr` (which wraps `secp256k1`); the only direct secp256k1 surface here is the error type | `error.rs:19` |
| Verification inside the pairing state machine | `event.verify()` (id **and** signature) before any other check on inbound pairing events | `pairing/session.rs:632-635` |
| Verification for engrams | explicitly the **caller's** responsibility before `validate_and_decrypt` — documented, not enforced | `engram.rs:478-482` |
| Tamper tests | ID tamper (`verification.rs:44-58`), signature tamper (`:60-68`), and an independent check that a `sig`-only tamper leaves `verify_id()` true (`event.rs:57-66`) | |

`StoredEvent.verified` is a private bool with only a read accessor (`event.rs:17`, `:33-35`); `new()` forces `false` (`event.rs:28`) so a verified flag can only be asserted deliberately through `with_received_at` (`event.rs:38-48`).

---

### 3. `unsafe` code

| Finding | Evidence |
|---|---|
| Crate-level deny | `#![deny(unsafe_code)]` at `src/lib.rs:1` |
| Actual `unsafe` blocks/fns | **none**. The only textual matches for "unsafe" in `src/` are the lint attribute (`lib.rs:1`) and two prose uses of the word in `pairing/qr.rs:23` and `:226` ("characters unsafe in a query-parameter value") |
| Misleading comment | `engram.rs:410-412` contains a `// SAFETY:` comment justifying a slice index (`&body[start..j]` at `:413`), but the code is a **safe** slice operation, not an `unsafe` block. The comment style is borrowed from unsafe-code convention; no unsafe code is present |

---

### 4. Constant-time comparison

| Where | Implementation | file:line |
|---|---|---|
| Primitive | `ct_eq(&[u8;32], &[u8;32]) -> bool` via `subtle::ConstantTimeEq`, with a doc note on preventing compiler short-circuiting | `pairing/crypto.rs:120-129` |
| Pairing session-ID comparison | offer's `session_id` compared with `ct_eq` | `pairing/session.rs:173-181` |
| Transcript-hash comparison | received hash decoded then compared with `ct_eq` | `pairing/session.rs:357-363` |
| **Not** constant-time (by contrast) | engram `d`-tag comparison uses plain `!=` on `String` (`engram.rs:551`); `p`-tag owner comparison uses `eq_ignore_ascii_case` (`engram.rs:533`); the filter `#p` reader gate compares with `==` on `&str` (`filter.rs:32`). These compare values that are derivable by the legitimate reader or already public, but the asymmetry with the pairing path is a factual difference worth noting |

---

### 5. Secret handling

| Control | Implementation | file:line |
|---|---|---|
| Session secret generated from `rand::fill` (32 bytes) | `pairing/session.rs:113-115` |
| All-zero session secret rejected on QR decode | `pairing/qr.rs:169-174` |
| `QrPayload` zeroizes `session_secret` on drop | `pairing/qr.rs:52-56` |
| `PairingSession` zeroizes `session_secret`, `session_id`, `sas_input` on drop | `pairing/session.rs:731-739` |
| ECDH shared secret zeroized immediately after SAS derivation (both roles) | `pairing/session.rs:189` (source), `:295` (target) |
| Serialized plaintext zeroized after encryption | `pairing/session.rs:573`; `observer.rs:79` |
| Decrypted plaintext zeroized on success **and** on failure paths | `pairing/session.rs:610`, `:619`; `observer.rs:66`, `:101`, `:109` |
| Payload secrets travel as `Zeroizing<String>` across the API boundary | `pairing/session.rs:227-230`, `:388-391`, `:405` |
| Transient payload clone zeroized even on error (deferred `?`) | `pairing/session.rs:241-247` |
| Documented residual exposure | `serde_json::to_string` intermediate buffers and `nip44::encrypt`'s internal copy cannot be zeroized — stated at `pairing/session.rs:556-560` |
| Ephemeral private keys | zeroing delegated to `nostr::SecretKey`'s own `Drop`, which uses `write_volatile` (noted at `pairing/session.rs:729-730`) |
| Metadata privacy | outbound pairing events subtract random 0–30 s jitter from `created_at` | `pairing/session.rs:577-579` |

---

### 6. Authorization / access-control logic in this crate

| Control | Rule | file:line |
|---|---|---|
| Result-level private-event gate | reader must appear in the event's `p` tag for kinds 30622 and 44200; every other kind passes | `filter.rs:23-33` |
| Owner-only read for turn metrics | even the authoring agent is denied (test asserts it) | gate `filter.rs:28-32`, assertion `filter.rs:294-298` |
| Declared read-gate kind sets consumed by the relay | `AUTHOR_ONLY_KINDS` (`kind.rs:120`), `P_GATED_KINDS` (`kind.rs:146-156`), `RESULT_GATED_KINDS` (`kind.rs:129`); contracts documented at `kind.rs:112-144` |
| Relay-only kinds (client submission must be rejected) | `is_relay_only_kind` — 13534, 40901, 40902, 30622, 39005, 39006 | `kind.rs:682-693` |
| Cross-channel leak prevention in filters | an explicit `h` tag is authoritative; the `channel_id` fallback applies only when the event has no `h` tags | `filter.rs:78-100` |
| Role ladder used for git push authorization | Owner 4 / Admin 3 / Member 2 / Guest 1 / Bot 0; Bot satisfies nothing at this layer | `channel.rs:142-157` |
| Explicit rules can never weaken destructive defaults | effective minimum is `max(explicit push:role, built-in default)` | `git_perms.rs:535-552` |
| Protection constraints bind everyone including Owner | `no-force-push` / `no-delete` / `require-patch` are checked after the role check and have no role exemption | `git_perms.rs:527-570`; doc `git_perms.rs:5-7` |
| Atomic push semantics | any denied ref fails the whole push | `git_perms.rs:584-597` |
| Pairing peer binding | source locks to the offering pubkey; all later events must match it | `pairing/session.rs:187-188`, `:681-696` |
| Anonymous abort DoS prevention | `handle_abort` refuses before a peer is known | `pairing/session.rs:456-462` |
| Terminal-state immutability | `abort`/`handle_abort` refuse from `Completed`/`Aborted` | `pairing/session.rs:431-437`, `:449-454` |
| Replay/duplicate protection | per-session processed-id set; ids recorded only after full acceptance so probes cannot poison it | `pairing/session.rs:638-643`, `:676-678` |
| Session expiry | 120 s default, checked at the top of every handler | `pairing/session.rs:43`, `:698-703` |
| Tenant fence | `CommunityId`/`TenantContext` cannot be deserialized from client input; single `resolved()` constructor | `tenant.rs:17-30`, `:37-91` |

---

### 7. Feature-gated / test-only code and what it exposes

| Gate | What it exposes | file:line |
|---|---|---|
| `#[cfg(any(test, feature = "test-utils"))]` | `pub mod test_helpers`: `make_event` (`lib.rs:55`), `make_event_with_keys` (`:64`), `make_stored_event` (`:72`). These generate random `Keys` and sign test events; `make_stored_event` constructs a `StoredEvent` with **`verified = true`** without running `verify_event` | `lib.rs:47-74` (verified flag passed at `:73`) |
| `#[cfg(test)] impl PairingSession` | `has_processed()` (reads the dedup set) and `set_timeout()` (shrinks the session lifetime) — deliberately kept out of the public API per the doc comment | `pairing/session.rs:530-544` |

Notes:
- There is **no** `dev` feature in this crate (the `dev` feature seen in `crates/buzz-relay/Cargo.toml:84` belongs to `buzz-auth`, not `buzz-core`). `buzz-core` declares exactly one feature: `test-utils` (`Cargo.toml:10-11`).
- `test-utils` is enabled only by `crates/buzz-relay/Cargo.toml:89` inside `[dev-dependencies]`, so the helper cannot leak into a release relay binary through that edge.
- Because `test_helpers::make_stored_event` sets `verified = true`, any production code path that accepted a `StoredEvent` from this helper would bypass verification. It is compile-gated, so this is a hygiene observation rather than a live exposure.

---

### 8. Input validation inventory and gaps

Validated (fail-closed) inputs:

| Input | Validation | file:line |
|---|---|---|
| Nostr event (id, sig) | full verification | `verification.rs:11-32`; pairing path `session.rs:632-635` |
| NIP-44 ciphertext length | 132–87,472 chars before decryption | `observer.rs:53-55`, `:85-90`; `session.rs:595-601` |
| Decrypted plaintext size | ≤ 65,535 bytes, zeroized on rejection | `observer.rs:96-104`; `session.rs:609-615` |
| Engram slug | grammar + 255-byte cap + per-segment 64-byte cap | `engram.rs:67-112` |
| Engram body JSON | strict parse, duplicate member names rejected at any depth, trailing data rejected, field types checked | `engram.rs:216-259`, `:283-380` |
| Engram envelope | kind, author, exactly one `d`, 64 lowercase hex `d`, exactly one `p`, owner match, slug↔d-tag re-derivation | `engram.rs:489-557` |
| Engram body size | ≤ 65,535 bytes at build time | `engram.rs:436-440` |
| QR URI | ≤ 2048 chars, scheme, 64-char lowercase hex pubkey, 64-char lowercase hex secret, non-zero secret, ≥ 1 relay, each relay a parseable `ws`/`wss` URL with a host, version must be 1 | `qr.rs:104-206` |
| Relay URL for runtime identity | scheme, no credentials, no fragment, host required | `relay.rs:37-64` |
| Host header | normalization with fail-closed empty result | `tenant.rs:121-139` |
| Git ref pattern | length, `refs/` prefix, segment charset, wildcard count, `**` position | `git_perms.rs:83-146` |
| `buzz-protect` tag | min value count, role validity, `push:bot`/`push:guest` rejection, per-repo rule cap | `git_perms.rs:303-362`, `:379-399` |
| NIP-AM payload numbers | `cost_usd` finite and ≥ 0 on both encrypt and decrypt | `agent_turn_metric.rs:147-190` |
| Presence status | strict serde enum | `presence.rs:10-18` |
| Pairing protocol version | must be 1 (QR and offer) | `qr.rs:145-151`, `session.rs:169-175` |

Gaps and asymmetries found in the code (stated factually, no severity judgement):

| # | Gap | Evidence |
|---|---|---|
| G-1 | NIP-AM structural requirements are documented but unenforced: `session_id`/`turn_seq` required when `cumulative` is present, `turn_seq` strictly increasing, `timestamp` must be RFC 3339. `validate()` checks only `cost_usd`; `timestamp` is an unparsed `String` | doc `agent_turn_metric.rs:97-111`; `validate()` `:147-169`; field type `:111` |
| G-2 | `harness`, `model`, `session_id`, `turn_id`, `channel_id` have no length or format bounds; `channel_id` is a free `String` rather than a UUID type | `agent_turn_metric.rs:89-100` |
| G-3 | `reader_authorized_for_event` hard-codes its two kinds instead of reading `RESULT_GATED_KINDS`, so adding a kind to the constant does not extend the gate | `filter.rs:25` vs `kind.rs:129` |
| G-4 | Filter matching implements no `limit` and no NIP-50 `search` handling; a caller relying on `filters_match` alone gets no result-count bound | `filter.rs:35-104` |
| G-5 | `filters_match` treats a present-but-empty `kinds`/`authors`/`ids` collection as "match nothing" purely as a side effect of `contains`; the NIP-01 empty-vs-absent distinction is not addressed explicitly | `filter.rs:36-66` |
| G-6 | Event ID prefix matching uses `starts_with` on hex (`filter.rs:60-66`); a 1-character filter id would match a large set. No minimum prefix length is enforced |
| G-7 | `normalize_host` strips only exact `:443`/`:80` suffixes, so an IPv6 literal with a non-default port keeps its port while `[::1]:443` collapses — correct per the doc, but it means host strings must already be authority-shaped; no validation rejects a malformed host string | `tenant.rs:121-139` |
| G-8 | `engram::extract_refs` silently drops malformed candidates (documented) — no signal to the caller that a reference was present but invalid | `engram.rs:373-381`, `:421-425` |
| G-9 | `git_perms::parse_protection_tags` checks the rule count **before** parsing each tag (`rules.len() >= MAX_PROTECTION_RULES` at `:387-389` precedes the parse at `:391`), so a 51st malformed tag reports `TooManyRules` rather than the parse error | `git_perms.rs:383-394` |
| G-10 | `UpdateKind::classify` trusts the caller-supplied `is_ancestor` boolean and does not validate OID shape (any non-zero string is treated as a real OID) | `git_perms.rs:206-221` |
| G-11 | `url_decode` in the QR path uses `decode_utf8_lossy`, so invalid UTF-8 percent-escapes become replacement characters rather than an error (the subsequent `Url::parse` is the effective guard) | `qr.rs:236-238`, validation at `:183-204` |
| G-12 | Pairing `PayloadType::Custom` content is passed through unvalidated by design ("interpretation is out-of-band") | `pairing/types.rs:75-76` |
| G-13 | `PairingSession::processed_ids` grows unbounded within a session; the bound is argued from the 120 s lifetime and ~6 expected events rather than enforced by a cap | doc `pairing/session.rs:626-628`, field `:105-106` |
| G-14 | `engram::select_head` performs no validation of its inputs (kind, tags, signature) — documented as the caller's job | `engram.rs:560-563` |
| G-15 | `is_private_ip` does not cover the RFC 5737 documentation ranges, IPv4 multicast/reserved space, NAT64/6to4 prefixes, or IPv4-compatible IPv6 addresses (see §1) | `network.rs:26-56` |
