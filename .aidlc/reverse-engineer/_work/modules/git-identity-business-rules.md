## Module: git-sign-nostr & git-credential-nostr (`crates/git-sign-nostr`, `crates/git-credential-nostr`)
### Aspect: Business Rules

#### Overview

The "business logic" of both tools is entirely protocol conformance: each
implements one half of a well-specified interface (git's GPG-signing-program
interface for `git-sign-nostr`; git's credential-helper interface for
`git-credential-nostr`) plus a Nostr-specific signing/verification step. There
is no user-facing business domain beyond "prove this git operation came from
this Nostr key."

#### `git-sign-nostr`: signing procedure

`do_sign` (`crates/git-sign-nostr/src/lib.rs:943-1082`) executes, in order:

1. **Load key** — `load_key()` (`lib.rs:397-441`), priority
   `NOSTR_PRIVATE_KEY` env → `BUZZ_PRIVATE_KEY` env → `nostr.keyfile` git
   config, matching the NIP-GS spec's Key Loading order exactly. Both env
   vars are capped at 128 bytes and the process env var is removed
   immediately after reading regardless of success (`lib.rs:399-436`), to
   shrink the exposure window (see Security doc).
2. **Parse into `SecretKey` directly**, bypassing `nostr::Keys`
   (`lib.rs:947-957`) — a deliberate choice documented at `lib.rs:41-44` to
   avoid a second, non-zeroizable copy of the secret. The `Keypair` is
   wrapped in `KeypairGuard`, whose `Drop` impl calls
   `Keypair::non_secure_erase()` unconditionally (`lib.rs:93-110`), so every
   early-return path (including argument-mismatch errors below) still
   erases key material.
3. **Verify `-u <key>` argument matches the loaded key**, if non-empty
   (`lib.rs:980-993`). Rule enforced: if `<key>` parses as hex or `npub1...`
   via `normalize_key_id`, it *must* equal the derived pubkey or signing
   fails (`lib.rs:984-988`); if `<key>` is non-empty but unrecognized as
   either format, signing *also* fails (`lib.rs:990-993`) — the code is
   stricter here than the NIP-GS spec's SHOULD-level requirement ("the
   signing program SHOULD verify... If they do not match, MUST exit with an
   error... If `<key>` is empty or not recognizable, MAY ignore it") — this
   implementation treats "not recognizable but non-empty" as a hard failure
   rather than exercising the spec's MAY-ignore option.
4. **Read payload from stdin**, bounded to 100 MiB (`lib.rs:1000,
   1552-1567`).
5. **Capture timestamp `t`** from `SystemTime::now()`, rejecting if it would
   exceed `u32::MAX` (`lib.rs:1002-1010`) — a Y2106 guard matching the
   NIP-GS `t` range constraint.
6. **Load and validate the optional NIP-OA auth tag** (`lib.rs:1013-1049`):
   - Rule: **fail closed if `BUZZ_AUTH_TAG`/`nostr.authtag` is configured but
     malformed** — `load_auth_tag()` returns `Err` rather than `Ok(None)` for
     a syntactically present-but-invalid tag (`lib.rs:463-475` return-type
     doc comment states this explicitly: "Callers MUST treat this as a hard
     error to prevent signing without the intended attestation").
   - Rule: **self-attestation rejected** — owner pubkey must differ from the
     signer's own pubkey (`lib.rs:1026-1030`).
   - Rule: **owner signature is verified before embedding**, not just
     structurally checked — `verify_oa(&pk_hex, oa_val)` (`lib.rs:1032-1038`)
     runs a real BIP-340 verification against the NIP-OA preimage
     (`"nostr:agent-auth:" || agent_pk || ":" || conditions`,
     `lib.rs:1517-1519`) and refuses to sign if it fails. This is stricter
     than NIP-OA's own text (which only requires *verifiers* to check the
     signature) — `git-sign-nostr` also self-checks before it will emit a
     signature carrying that attestation.
   - Rule: **temporal conditions are enforced against the signing
     timestamp** — `enforce_conditions(&oa_val.1, t)` (`lib.rs:1043-1045`)
     rejects signing if `created_at<N`/`created_at>N` clauses in the
     attestation are violated by the *current* `t`, preventing an expired
     auth tag from being freshly embedded into a new signature.
   - Rule: **`kind=` clauses are accepted but not enforceable** — a warning
     is printed but signing proceeds (`lib.rs:1046-1048`), matching the
     NIP-GS spec's explicit carve-out that `kind=` has no git equivalent.
7. **Compute the domain-separated signing hash** and produce a BIP-340
   Schnorr signature over it with `SECP256K1.sign_schnorr` (`lib.rs:1055-1056`).
8. **Write the armored envelope to stdout**, then **GnuPG status lines**
   (`BEGIN_SIGNING`, `SIG_CREATED D 8 1 00 <t> <pk>`) to the status-fd
   (`lib.rs:1063-1077`). Rule: stdout write errors are fatal
   (`lib.rs:1067-1070`) because a partial/failed signature write must not be
   silently swallowed — git would otherwise treat empty stdout as an
   (incorrectly) absent signature rather than surfacing the real I/O error.

#### `git-sign-nostr`: verification procedure

`do_verify` (`lib.rs:1099-1335`) is the mirror: parse armor → validate
base64-line-length (≤4096, `MAX_BASE64_LINE`) → base64-decode → check
decoded size (≤2048, `MAX_JSON_DECODED`) → validate UTF-8 → **reject any
whitespace outside string values** (`has_non_string_whitespace`,
`lib.rs:1700-1719`, called at `:1213-1220`) → parse the envelope structurally
→ **rebuild the canonical JSON from the parsed fields and byte-compare it
against the original decoded string** (`lib.rs:1223-1231`) — this is the
anti-malleability check: any reordering, extra whitespace, or non-canonical
number formatting in the original signature fails verification even if the
cryptographic signature itself would otherwise check out. Then: validate `pk`
is a real BIP-340 point (`PublicKey::from_hex`, `lib.rs:1234-1240`), re-read
the payload from stdin, recompute the signing hash, and
`SECP256K1.verify_schnorr` the signature (`lib.rs:1187`).

Rule: **on cryptographic failure, `BADSIG` is emitted and verification stops
before any OA processing** (`lib.rs:1188-1196`) — an invalid base signature
short-circuits before the code even looks at the `oa` field.

Rule: **OA verification failure does not invalidate the base signature** —
per NIP-GS design, if `sig` verifies but `oa[2]` does not, the code still
emits `GOODSIG`/`VALIDSIG` for the commit and separately reports
`nostr-oa-status: invalid_signature` via `NOTATION_DATA` (`lib.rs:1215-1218,
1305-1318`). The commit is "signed", the *owner authorization* is
independently "unverified" — the two facts are deliberately decoupled, as
required by NIP-GS § Owner Attestation → Trust Display.

Rule: **trust determination is local-config-only** — `determine_trust`
(`lib.rs:1673-1682`) returns `TRUST_FULLY` iff the verified `pk` matches
`git config user.signingkey` (normalized via `normalize_key_id`), else
`TRUST_UNDEFINED`. This is explicitly documented as advisory-only (not a PKI
trust root) both in the module doc (`lib.rs:20-27`) and via an extra
`NOTATION_DATA advisory-config-match-only` status line emitted alongside
every `TRUST_*` line (`lib.rs:1315-1318`) — a belt-and-suspenders
disclosure not required by the NIP-GS spec text itself but added defensively.

#### `git-credential-nostr`: decision tree for whether to emit a credential

`run()` (`crates/git-credential-nostr/src/lib.rs:152-264`) implements a
strict ordered decline sequence, each step a "fall through silently" rule
except where noted:

1. Verb must be `get` or absent, else exit `0` silently (`lib.rs:154-155`).
2. If `capability[]=authtype` was not seen on stdin (i.e. git predates 2.46 or
   didn't advertise the capability), print a bare blank line and exit `0`
   (`lib.rs:160-164`) — this is the git-version compatibility rule from the
   crate's `README.md`.
3. **Rule, explicitly ordered first among the "hard" checks**: if no
   `wwwauth[]=Nostr ...` line was seen, exit `0` with no output at all
   (`lib.rs:180-183`). The comment at `lib.rs:177-179` states this ordering
   is deliberate — "so non-Buzz remotes never hit validation errors" —
   meaning a GitHub/GitLab credential request reaching this helper (e.g. via
   a global `credential.helper=nostr` config) is *not* an error case, it's
   the expected common path, and must produce zero stderr noise.
4. If `wwwauth` was present but `parse_method` (`lib.rs:135-149`) cannot
   extract a `method="..."` token from it, also exit `0` silently
   (`lib.rs:184-187`) — same rationale, covers a Buzz-shaped challenge from
   an older/misconfigured server.
5. **Only past this point do missing `protocol`/`host`/`path` become hard
   errors** (exit `1`, `lib.rs:190-199`) — because by now the code has
   established this genuinely is a Nostr-challenging remote, so an
   incomplete request is a real problem (most commonly: missing `path`
   means `credential.useHttpPath` is not set, and the error message says so
   verbatim: `"credential.useHttpPath must be true for NIP-98 auth"`,
   `lib.rs:196-198`).
6. `repo_path` is derived by stripping the git-service suffix from `path`
   (`/info/refs`, `/git-upload-pack`, `/git-receive-pack`,
   `lib.rs:200-205`) — this reconstructs the repo-root URL the relay expects
   as the NIP-98 `u` tag, independent of which of the three git-HTTP
   sub-requests triggered the credential lookup. Confirmed to match the
   relay's own reconstruction in `git_expected_url`
   (`crates/buzz-relay/src/api/git/transport.rs:245-251`), which strips the
   identical three suffixes.
7. Key load, event build/sign, and output — errors from this point are hard
   (`lib.rs:210-252`).

Rule (implicit, from the ordering in step 3-4 vs step 5): **a non-Buzz
remote and a Buzz remote with a broken server config are treated
identically (silent decline)** — the helper cannot distinguish "this isn't a
Buzz server" from "this is a Buzz server whose `WWW-Authenticate` header is
missing `method=`" (the latter is called out by the crate's own `README.md`
troubleshooting table as `method hint` / "Upgrade the Buzz server"). Both
produce the same observable behavior: git silently falls through to the next
credential helper.

#### `git-credential-nostr`: credential-helper protocol correctness

Reads exactly the subset of the documented git credential protocol it needs
(`protocol=`, `host=`, `path=`, `capability[]=authtype`, `wwwauth[]=`,
`parse_stdin` `lib.rs:107-132`) and ignores everything else (`username=`,
`password=`, `state[]=`, etc. all fall through with no branch). It writes
back exactly the fields defined for the `authtype`/`credential` extension:
`capability[]=authtype`, `authtype=Nostr`, `credential=<b64>`,
`ephemeral=true`, `quit=true`, terminated by a blank line
(`lib.rs:258-263`). This matches git's documented `get` response format for
helpers that opt into `capability[]=authtype`.

#### Token/credential lifecycle

There is **no lifecycle** in the caching sense — every invocation mints a
brand-new kind:27235 event with a fresh `created_at` (delegated to
`nostr::EventBuilder::http_auth`, `lib.rs:235`) and marks it `ephemeral=true`
so git does not persist it (`lib.rs:261`). "Expiry" is therefore entirely
server-side: the relay's `verify_nip98_event` enforces a ±60-second window
against wall-clock time (`crates/buzz-auth/src/nip98.rs:18,32,77-79`, outside
this module's scope but the mechanism the credential's short lifetime relies
on). `git-credential-nostr` itself has no retry logic — a single failed
attempt (e.g. bad key) is a single exit `1`; git's own credential-helper
retry/fallback behavior (trying the next configured helper) is what recovers,
not any loop inside this crate.

#### Rules enforced only by comment or convention (not compiler/runtime)

- **"Env vars are removed from process environment to minimize exposure"**
  (`git-sign-nostr/src/lib.rs:397-441`) is enforced in `git-sign-nostr`'s
  `load_key`, but the equivalent function in `git-credential-nostr`
  (`load_key`, `lib.rs:50-73`) never calls
  `std::env::remove_var("NOSTR_PRIVATE_KEY")` — the convention is stated as a
  security property of the *ecosystem* (`buzz-dev-mcp`'s `shim.rs:47-49`
  removes it once, at the shim layer, before any child sees it) but is not
  independently enforced inside `git-credential-nostr` itself. See Security
  doc for the full asymmetry.
- **"NOTATION_DATA advisory-config-match-only" / TRUST_FULLY is not a PKI
  trust root"** (`git-sign-nostr/src/lib.rs:20-27, 1673-1682`) is a rule
  stated only in doc comments and a machine-readable-but-optional status
  line — nothing prevents a caller from treating `TRUST_FULLY` as a real
  trust signal; the code can only annotate, not enforce, correct
  interpretation downstream.
- **"Implementations SHOULD use process-scoped configuration"**
  (`docs/nips/NIP-GS.md` § Git Configuration) — neither crate itself sets up
  `GIT_CONFIG_COUNT`/`GIT_CONFIG_KEY_*`; that responsibility is fully
  delegated to `buzz-dev-mcp`'s `shim.rs:186-213` (out of scope here, but the
  rule is *followed* by the ecosystem, just not *by these two crates*).

#### Contradiction check against `AGENTS.md` one-liners

`AGENTS.md:57-58` describes the two crates as "Sign git objects with a Nostr
key" and "Git credential helper for Nostr-authed push/fetch" respectively.
Both descriptions hold up against the code with no contradiction found:
`git-sign-nostr` is exactly a `gpg.x509.program`-shaped commit/tag signer
producing BIP-340 signatures (confirmed: `lib.rs:1-15` module doc, `Mode::Sign`
at `:186`, `SECP256K1.sign_schnorr` at `:1056`), and `git-credential-nostr` is
exactly a credential helper minting NIP-98 kind:27235 events for git's
HTTP push/fetch auth (confirmed: `lib.rs:1-6` module doc, `EventBuilder::
http_auth` at `:235`). No rewording is needed; the one-line descriptions in
`AGENTS.md` accurately summarize the implementation.
