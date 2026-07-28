## Module: git-sign-nostr & git-credential-nostr (`crates/git-sign-nostr`, `crates/git-credential-nostr`)
### Aspect: Data Model

#### Overview

Neither crate has a database, cache, or persistent state beyond the ephemeral
key material each reads at process start. Both are short-lived CLI processes
invoked once per git operation. The two "data models" that matter are the
**on-the-wire formats each binary produces/consumes** and the **key
representations** each accepts.

#### NIP-GS signature envelope (`git-sign-nostr`)

The signature `git-sign-nostr` writes to stdout in sign mode, and reads back
in verify mode, is a JSON object wrapped in PEM-style armor:

```
-----BEGIN SIGNED MESSAGE-----
<base64>
-----END SIGNED MESSAGE-----
```

Built by `armor()` (`crates/git-sign-nostr/src/lib.rs:938-941`), which base64s
the JSON with the standard alphabet and wraps it between fixed
`ARMOR_BEGIN`/`ARMOR_END` markers (`lib.rs:113-114`). Parsed back by
`parse_armor()` (`lib.rs:1464-1499`), which requires exactly 3 lines, a
trailing `\n`, no trailing whitespace on any line, and the literal begin/end
markers.

The decoded JSON envelope, built field-by-field with `format!` rather than a
serde `Serialize` impl (deliberately — see below), is:

| Field | Type | Required | Constraints (as enforced in code) | Built/parsed at |
|---|---|---|---|---|
| `v` | integer | yes | must equal `1`, or `parse_envelope` rejects (`lib.rs:1364-1367`) | `build_envelope` `lib.rs:924-936`; `parse_envelope` `lib.rs:1345-1441` |
| `pk` | string | yes | exactly 64 lowercase hex chars, validated by `validate_hex_field(pk, 64, "pk")` (`lib.rs:1374`) | same |
| `sig` | string | yes | exactly 128 lowercase hex chars (`lib.rs:1381`) | same |
| `t` | integer | yes | non-negative, ≤ 4294967295, rejected if float (`lib.rs:1385-1392`) | same |
| `oa` | array of 3 strings | no | `[owner_pk_hex(64), conditions, owner_sig_hex(128)]`; owner pk validated as a real BIP-340 point via `PublicKey::from_hex`, and `owner != pk` is enforced (self-attestation rejected) (`lib.rs:1394-1436`) | same |

`build_envelope` (`lib.rs:924-936`) hand-assembles the JSON string with a fixed
field order (`v, pk, sig, t[, oa]`) and no whitespace, specifically so that
`do_verify` can reconstruct the exact same byte string from parsed fields and
compare it against the original (`lib.rs:1223-1231`) — this is the "canonical
JSON" anti-malleability check the NIP-GS spec requires (`docs/nips/NIP-GS.md`
§ Verification Procedure step 2). Using serde's `Value`/`Map` here would not
guarantee this byte-exact round trip, which is why the comment at
`lib.rs:919-923` explicitly rejects serde for this purpose.

The envelope's `oa` field is a 3-tuple `(String, String, String)` in Rust
(owner pubkey hex, NIP-OA conditions string, owner signature hex) — see
`Envelope` struct (`lib.rs:1338-1343`) and `load_auth_tag`'s return type
`Result<Option<(String, String, String)>, Error>` (`lib.rs:463`).

#### Signing hash preimage

`compute_signing_hash` (`lib.rs:895-921`) builds the SHA-256 preimage:

```
"nostr:git:v1:" || decimal(t) || ":" || [oa[0] || ":" || oa[1] || ":" || oa[2] || ":"] || payload
```

matching `docs/nips/NIP-GS.md` § Signing Hash exactly, including the
bracketed `oa_binding` being omitted entirely (zero bytes, not an empty
string) when no `oa` is present. `DOMAIN_SEPARATOR = "nostr:git:v1:"`
(`lib.rs:112`). Verified against the spec's test vector by
`test_signing_hash_matches_spec` and `test_signing_hash_with_oa_matches_spec`
(`lib.rs:1806-1821`).

#### NIP-98 credential event (`git-credential-nostr`)

`git-credential-nostr` produces a standard Nostr **kind:27235** event (NIP-98
HTTP Auth), not a bespoke envelope. It is built via the `nostr` crate's own
`EventBuilder::http_auth(HttpData::new(parsed_url, method))`
(`crates/git-credential-nostr/src/lib.rs:227-235`) — the crate does not
construct the event JSON by hand the way `git-sign-nostr` does for its
envelope. The event's shape (id, pubkey, created_at, kind, tags, content,
sig) is entirely delegated to the `nostr` crate (workspace dependency,
version `0.44.3` per `Cargo.lock`). The only local addition is an optional
`auth` tag (NIP-OA) appended via `builder.tag(tag)` when configured
(`lib.rs:236-239`).

The finished event is serialized with `serde_json::to_string(&event)`
(`lib.rs:246`) then base64-standard-encoded (`lib.rs:255`) and emitted as the
`credential=` line of the git credential-helper protocol (`lib.rs:260`). This
base64(JSON(signed-event)) string is exactly what appears as the git
`Authorization: Nostr <token>` header value downstream (confirmed against the
server side at `crates/buzz-relay/src/api/git/transport.rs:88-113`, which
base64-decodes the token and parses it back into a `nostr::Event`).

Test `happy_path` (`crates/git-credential-nostr/tests/integration.rs:63-116`)
decodes this credential value and asserts `kind == 27235` and the presence of
`id`/`pubkey`/`sig`/`tags`, confirming the on-wire shape empirically.

#### Key representations

Both crates accept the Nostr secret key in the same two textual forms,
per the NIP-GS "Key Loading" section:

| Form | Length | Example prefix | Parsed by |
|---|---|---|---|
| Hex | 64 chars | (raw hex, no prefix) | `SecretKey::parse` (`git-sign-nostr/src/lib.rs:947`), `Keys::parse` (`git-credential-nostr/src/lib.rs:216`) |
| NIP-19 bech32 | ~63 chars | `nsec1...` | same functions — both delegate bech32 handling to the `nostr` crate |

`git-sign-nostr` parses directly into `nostr::SecretKey` (`lib.rs:947`) and
explicitly avoids `nostr::Keys`, per the doc comment at `lib.rs:41-44`,
because `Keys` caches a non-zeroizable copy of the secret internally.
`git-credential-nostr` has no such concern documented and uses `Keys::parse`
directly (`lib.rs:216`), so it does retain whatever internal copies `Keys`
holds for the lifetime of the process (a short-lived process, but see the
Security doc for the implication).

Public keys are represented as 64-char lowercase hex (BIP-340 x-only, i.e. the
32-byte x-coordinate) throughout both crates — there is no compressed/parity
byte anywhere in either envelope format. `git-sign-nostr` additionally accepts
`npub1...` bech32 for the `-u <key>` signing-key argument, normalized via
`normalize_key_id` (`lib.rs:1644-1658`).

#### Signature format: BIP-340 Schnorr

Both crates use `nostr::secp256k1::SECP256K1.sign_schnorr` /
`.verify_schnorr` (e.g. `git-sign-nostr/src/lib.rs:1056`, `:1187`) — i.e. raw
BIP-340 Schnorr signatures over a 32-byte message digest, not NIP-01 event
signing. `git-credential-nostr` does not sign anything itself; it hands a
`HttpData` to `nostr::EventBuilder::http_auth(..).sign_with_keys(&keys)`
(`git-credential-nostr/src/lib.rs:240`), which internally performs the same
class of BIP-340 signature as part of standard Nostr event signing (NIP-01),
just over the kind:27235 event hash rather than a custom envelope hash. This
matches the two "signature format" claims in the task: `git-sign-nostr`
produces a bespoke BIP-340 signature over a domain-separated hash; the NIP-98
event `git-credential-nostr` produces is a standard NIP-01-signed event whose
signature is *also* BIP-340 Schnorr, by construction of the `nostr` crate.

#### On-disk state: keyfiles only, no cache

Neither crate writes any file. Both may *read* a keyfile whose path comes
from `git config nostr.keyfile`:

- `git-sign-nostr`: `open_keyfile` (`lib.rs:776-838`) opens with `O_NOFOLLOW`
  (rejects symlinks) and `O_NONBLOCK` (rejects blocking on FIFOs), then
  `read_keyfile_secure` (`lib.rs:853-887`) reads up to 1024 bytes into a
  `zeroize::Zeroizing<String>` and trims it.
- `git-credential-nostr`: `load_key` (`lib.rs:50-73`) opens the file with
  plain `std::fs::metadata`/`std::fs::read_to_string` (no `O_NOFOLLOW`,
  see Security doc), capped at `MAX_KEYFILE_BYTES = 256` (`lib.rs:48`).

Neither crate ever persists a derived value (pubkey, token, session) to disk;
every value is recomputed per invocation. This is consistent with both being
invoked fresh, once per git request, by git itself.

#### NIP-OA auth-tag data shape

Both crates accept the same 4-element JSON array shape for owner attestation,
matching `docs/nips/NIP-OA.md`:

```json
["auth", "<owner-pubkey-hex>", "<conditions>", "<owner-sig-hex>"]
```

- `git-sign-nostr`'s `load_auth_tag` (`lib.rs:463-522`) parses this via
  `serde_json::Value` and returns the inner 3-tuple, with structural
  validation (`arr.len() != 4`, label check, hex-length/charset checks,
  `validate_conditions`) — see `lib.rs:475-521`.
  It embeds the 3-tuple as the envelope's `oa` field (Data Model above).
- `git-credential-nostr`'s `load_auth_tag` (`lib.rs:78-96`) parses the same
  4-element array into a `nostr::Tag` directly (via `Tag::parse(parts)`,
  `lib.rs:92`) and attaches it as a literal event tag on the signed NIP-98
  event, rather than re-deriving a private 3-tuple. This is a data-shape
  divergence between the two crates worth noting: `git-sign-nostr` keeps the
  raw strings and rebuilds JSON by hand; `git-credential-nostr` hands the
  same 4 strings straight to the `nostr` crate's own `Tag` type. Both accept
  identical *input* shape, but represent it differently internally — see
  Debt doc for the duplication this implies.

#### No shared data-model crate

Both crates independently define their own "auth tag" parsing/validation
logic (`git-sign-nostr/src/lib.rs:463-522` vs
`git-credential-nostr/src/lib.rs:78-96`) rather than sharing a struct or
parser from `buzz-core`/`buzz-sdk`. Confirmed by `Cargo.toml` for both crates
(`crates/git-sign-nostr/Cargo.toml:20-42`,
`crates/git-credential-nostr/Cargo.toml:9-12`): neither depends on
`buzz-core`, `buzz-sdk`, or `buzz-auth`. Their only common Rust dependency
touching this data is the `nostr` crate itself (`Tag`, `Keys`, hex/bech32
codecs).
