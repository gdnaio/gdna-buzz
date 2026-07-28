## Module: git-sign-nostr & git-credential-nostr (`crates/git-sign-nostr`, `crates/git-credential-nostr`)
### Aspect: Security

#### Overview

Both crates handle raw Nostr secret keys, so the central security question
for each is: how is the key loaded, and can it leak? A second question,
specific to `git-credential-nostr`, is whether a credential can leak to an
unintended remote. A third, specific to `git-sign-nostr`'s verify mode, is
whether adversarial signature-file input is safely bounded. This doc also
covers the asymmetry between the two crates' key-handling rigor, which is
significant enough to be a security-relevant finding in its own right.

#### Key loading and leak surface — `git-sign-nostr`

Priority order `NOSTR_PRIVATE_KEY` > `BUZZ_PRIVATE_KEY` > `nostr.keyfile`
(`crates/git-sign-nostr/src/lib.rs:397-441`). Concrete hardening observed:

- Both env vars are capped at 128 bytes before use, and **removed from the
  process environment immediately after being read**, regardless of whether
  parsing succeeds (`lib.rs:399-409, 416-436`) — this shrinks the window
  during which a child process or a `/proc/<pid>/environ` read (same-UID or
  root, per `docs/nips/NIP-GS.md` § Key Exposure via Environment Variables)
  could observe the raw key, though it does not eliminate exposure to
  anything that reads the environment *before* this function runs (e.g. the
  shell that launched git, or `buzz-dev-mcp`'s own shim — which is exactly
  why the shim independently scrubs `NOSTR_PRIVATE_KEY` at `shim.rs:47-49`,
  outside this crate's control).
- Keyfile path is opened with `O_NOFOLLOW` (`open_keyfile`,
  `crates/git-sign-nostr/src/lib.rs:776-800`) — **atomically rejects
  symlinks at the kernel level**, closing the TOCTOU window a separate
  `lstat`-then-`open` pattern would have. `O_NONBLOCK` is also set during
  open to prevent a FIFO at that path from hanging the process, then cleared
  once confirmed to be a regular file (`lib.rs:806-816`).
- Permission check requires exactly `0600` or `0400`
  (`mode & 0o177 != 0`, `lib.rs:820-824`) — narrower than NIP-GS's own text
  ("MUST verify file permissions are no broader than 0600"); the code's
  `0o177` mask also accepts `0400` (read-only), which is a reasonable
  superset the spec's prose doesn't explicitly rule out.
- **Ownership check**: the keyfile's UID must equal the current process's
  UID (`libc::getuid()`, `lib.rs:826-833`) — this is *not* required by the
  NIP-GS spec text and is a genuine hardening beyond spec: a `0600` file
  owned by a different UID could still be readable via ACLs or setuid
  execution, and this check closes that gap.
- Keyfile contents are read directly into `zeroize::Zeroizing<String>`
  (`read_keyfile_secure`, `lib.rs:853-887`) — allocated *before* the read so
  even an error mid-read still zeroizes on drop.
- The raw key string, once parsed into a `SecretKey`, is explicitly
  zeroized (`raw_key.zeroize()`, `lib.rs:952, 956`), and the `SecretKey`
  stack slot itself is manually overwritten with `ptr::write_bytes` before
  `drop` (`lib.rs:963-972`) — an extra step the code's own comments
  (`lib.rs:38-46, 60-62`) acknowledge is best-effort, since `secp256k1::
  SecretKey` upstream lacks a real `Zeroize` impl and the compiler may
  retain copies in registers/spilled stack slots regardless.
- The derived `Keypair` is wrapped in `KeypairGuard`
  (`lib.rs:93-110`), whose `Drop` calls `non_secure_erase()` — this runs on
  **every** exit path from `do_sign`, including the early-return error
  paths for key-id mismatch, OA validation failure, etc., because it is a
  RAII guard rather than an explicit end-of-function call.
- The key is **never written to stdout** — stdout in sign mode carries only
  the armored signature (`lib.rs:1063-1066`); in verify mode no key material
  is involved at all (verification uses only the embedded public key).
- The key is **never included in an error message** — every error string
  referencing key material uses derived, non-secret values only (`pk_hex`,
  the `-u <key>` argument as typed by the caller, never the secret itself).
  Confirmed by reading every `Error::Fatal`/`eprintln!` call site in
  `do_sign`/`load_key`/`read_keyfile_secure` — none interpolates `raw_key`
  or `secret_key`.

#### Key loading and leak surface — `git-credential-nostr`

Priority order `NOSTR_PRIVATE_KEY` > `nostr.keyfile`
(`crates/git-credential-nostr/src/lib.rs:50-73`) — note this crate does
**not** consult `BUZZ_PRIVATE_KEY` at all (confirmed: no reference to
`BUZZ_PRIVATE_KEY` anywhere in `git-credential-nostr/src/lib.rs`), unlike
`git-sign-nostr`'s three-tier fallback and unlike the NIP-GS spec's own key
loading order which both crates are nominally implementing the same
philosophy for. This is a **real asymmetry**: a deployment that only sets
`BUZZ_PRIVATE_KEY` (the variable name `buzz-acp`/`buzz-cli` actually use per
`AGENTS.md:164`, `.env.example`) would authenticate fine via `buzz-cli`
but silently fail `git-credential-nostr`'s key lookup (falling to `"no
nostr key configured"`, `lib.rs:53`) unless `NOSTR_PRIVATE_KEY` or a keyfile
is *also* set. `buzz-dev-mcp`'s shim works around this by writing a
dedicated keyfile from whatever `NOSTR_PRIVATE_KEY` it was given
(`shim.rs:80-116`), so the gap is invisible in the primary deployed path,
but a user following `git-credential-nostr`'s own `README.md` in isolation
(which does document `NOSTR_PRIVATE_KEY`, not `BUZZ_PRIVATE_KEY`, correctly
matching the code) would not hit this — the gap only bites someone who
assumes `BUZZ_PRIVATE_KEY` is honored by both git helpers symmetrically,
which the NIP-GS spec (written to describe `git-sign-nostr`'s behavior) does
imply.

Rigor differences from `git-sign-nostr`, all confirmed by reading:

- **No `O_NOFOLLOW` on keyfile open.** `load_key`
  (`crates/git-credential-nostr/src/lib.rs:56-73`) calls
  `std::fs::metadata(&path)` then separately `std::fs::read_to_string(&path)`
  — a classic TOCTOU window: between the permission-check `metadata` call
  (`check_keyfile_permissions`, `lib.rs:29-38`) and the actual read
  (`lib.rs:70-71`), the path could be swapped to point at a different file
  (e.g. via a symlink race), and there is no `O_NOFOLLOW` or fd-based
  re-check the way `git-sign-nostr/src/lib.rs:776-816` performs. This is a
  strictly weaker guarantee than `git-sign-nostr` provides for the
  structurally identical operation.
- **No ownership (UID) check.** `check_keyfile_permissions`
  (`lib.rs:29-38`) checks `mode & 0o177 != 0` (same mask as `git-sign-nostr`)
  but never compares the file's owning UID against the current process UID
  — `git-sign-nostr`'s extra `getuid()` check (`lib.rs:826-833`) has no
  counterpart here.
- **Max keyfile size is 256 bytes** (`MAX_KEYFILE_BYTES`, `lib.rs:48`) vs.
  `git-sign-nostr`'s 1024 bytes (`MAX_KEYFILE`, `lib.rs:854-856`) — both are
  generous relative to a ~64-byte key, so this difference has no practical
  security effect, just noted as an inconsistency.
- **The key, once loaded as `String`, is zeroized** at the two points it is
  discarded (`raw_key.zeroize()`, `lib.rs:219, 224`) — this part matches
  `git-sign-nostr`'s discipline. However, the crate uses `nostr::Keys::parse`
  (`lib.rs:216`) rather than parsing directly into `SecretKey` — meaning,
  per `git-sign-nostr`'s own documented rationale for avoiding `Keys`
  (`git-sign-nostr/src/lib.rs:41-44`: "we bypass `nostr::Keys` which caches
  non-zeroizable copies"), `git-credential-nostr`'s `Keys` value likely
  retains an un-zeroized internal copy of the secret for the remainder of
  the process's (short) lifetime. This is the same class of residual-copy
  risk `git-sign-nostr`'s own doc comments describe as a known ecosystem
  limitation (`git-sign-nostr/src/lib.rs:38-46`), just not documented or
  mitigated at all in `git-credential-nostr`.
- **No `git_config` subprocess env scrubbing.** `git-sign-nostr`'s
  `git_config`/`git_config_strict` (`lib.rs:661, 715`) both
  `.env_remove("NOSTR_PRIVATE_KEY")` and `.env_remove("BUZZ_PRIVATE_KEY")`
  before spawning `git config --get ...` (`lib.rs:665-667, 719-721`) — a
  defense-in-depth measure so a malicious `git` binary on `$PATH` cannot
  read the secret back out of its own inherited environment.
  `git-credential-nostr`'s `git_config` (`lib.rs:16-25`) does **no** such
  scrubbing — it spawns `git config --get <key>` with the full inherited
  environment, including `NOSTR_PRIVATE_KEY` if still set at that point in
  the call sequence. In practice `load_key` calls `git_config("nostr.
  keyfile")` (`lib.rs:58`) only *after* checking `NOSTR_PRIVATE_KEY` is
  absent/empty (`lib.rs:51-55`), so the env var that would matter most is
  usually already known-absent by the time `git_config` runs — but
  `load_auth_tag`'s call to `git_config("nostr.authtag")` (`lib.rs:82`) has
  no such ordering guarantee relative to `NOSTR_PRIVATE_KEY`'s presence, so
  a malicious `git` on `$PATH` invoked during that call could read the
  still-set secret from its own environment. This is a concrete, if narrow,
  gap relative to `git-sign-nostr`'s equivalent path.

**Net assessment**: `git-sign-nostr` treats the secret key with
meaningfully more paranoia (symlink rejection, UID check, subprocess env
scrubbing, manual stack-zeroing, avoiding `nostr::Keys`) than
`git-credential-nostr` does, despite both handling the same class of secret
for the same threat model (a process invoked by git, potentially in a
repository whose config or `$PATH` is not fully trusted). Whether this
asymmetry is *acceptable* depends on the deployment: in the primary
`buzz-dev-mcp` shim path both keyfiles are freshly written into a `0700`
tempdir the shim itself controls (`shim.rs:24-30`), which narrows the
practical blast radius of the weaker checks; a manual, standalone install of
`git-credential-nostr` per its own README (pointing `nostr.keyfile` at
`~/.nostr/key`) would be more exposed to the gaps listed above than an
equivalent manual `git-sign-nostr` install.

#### Can a credential leak to an unintended git remote?

`git-credential-nostr`'s decision tree (see Business Rules doc) declines to
emit a credential unless a `wwwauth[]=Nostr ...` challenge with an extractable
`method="..."` was actually present on stdin (`lib.rs:180-187`). Because git
itself is the one populating `wwwauth[]=` from the *actual* server's `401`
response for the *actual* request being made, this means the helper cannot be
tricked into emitting a Nostr credential for a remote that never issued a
Nostr challenge — the check is inherently scoped to the request git is
making, not to any helper-side allowlist of hosts. There is no separate
host-allowlist in the code (no `if host == "relay.example.com"` anywhere) —
the *only* gate is "did this specific request receive a Nostr challenge,"
which is the correct trust boundary for a credential helper (it should not
need to know Buzz's hostname in advance) but does mean a **malicious server
capable of issuing a syntactically valid `WWW-Authenticate: Nostr realm="x",
method="GET"` header for any request could induce this helper to sign and
hand over a fresh NIP-98 event scoped to *that* server's URL**. This is not a
credential-reuse leak (the signed event's `u` tag is locked to the requesting
URL, so it cannot be replayed against a different host), but it does mean
any HTTPS server the user's git ever talks to can, by sending the right
challenge header, obtain a validly-signed proof that the user's Nostr key
made a request to *that server's URL* at *that time* — a privacy/
fingerprinting consideration more than a credential-theft one, and inherent
to the additive design goal (work automatically against any Nostr-challenging
remote) rather than a bug.

#### Replay protection

- **`git-credential-nostr`**: replay protection is entirely server-side — the
  relay's ±60-second `verify_nip98_event` window
  (`crates/buzz-auth/src/nip98.rs:18,32,77-79`) is what limits how long a
  captured token remains valid; `git-credential-nostr` itself applies no
  independent nonce or single-use marker (`ephemeral=true` only tells git not
  to *cache* the credential locally, it does not affect the signed event's
  own properties). The relay additionally documents that it does *not*
  dedupe by event ID for git routes specifically, because a single signed
  token is legitimately reused across the info/refs GET and the following
  POST within one clone/push session (`transport.rs:165-176` — "NIP-98
  event-ID dedup intentionally NOT implemented here... Git's credential
  protocol reuses one signed token across multiple requests in a session").
  So replay protection for this crate's output is exactly the ±60s window,
  nothing more, by explicit relay-side design.
- **`git-sign-nostr`**: the signed envelope has **no expiry at all** by
  design — a commit signature is meant to remain valid indefinitely (per
  `docs/nips/NIP-GS.md` § Replay Across Repositories: "A signed git object...
  is valid wherever it appears... This is intentional and consistent with how
  GPG-signed commits behave"). The `t` field is signer-controlled and
  verifiers are only advised, not required, to cross-check it against the
  commit's own `author`/`committer` timestamps (`docs/nips/NIP-GS.md` §
  Signer-Controlled Timestamp) — `git-sign-nostr`'s `do_verify` performs no
  such cross-check itself (confirmed: no reference to `author`/`committer`
  timestamp comparison anywhere in `do_verify`, `lib.rs:1099-1335`).

#### TLS / scheme handling

Neither crate makes an HTTP(S) network call directly — `git-credential-nostr`
only *signs* data; the actual HTTP request (and its TLS) is made by `git`
itself. The `protocol=` field read from stdin (`lib.rs:118-119`) is trusted
as-is to build the signed URL's scheme (`format!("{protocol}://{host}/
{repo_path}")`, `lib.rs:206`) — if git is configured to talk to the relay
over plain HTTP (as the relay's own dev config does, `RELAY_URL=ws://
localhost:3000` per `.env.example`), the resulting NIP-98 token is
transmitted over that same unencrypted channel, meaning a network observer
could capture and reuse it within the relay's own ±60-second window. This is
flagged as a known, accepted tradeoff in `docs/nips/NIP-GS.md` for the
*signing* side ("HTTPS in production (prevents token theft)",
`transport.rs:180`) and applies identically here — `git-credential-nostr`
itself performs no scheme enforcement (no rejection of `protocol=http`), so
the security of the token in transit is entirely a deployment-configuration
concern (use HTTPS in production), not something this crate's code enforces
or warns about.

#### Input validation on adversarial stdin/signature-file input

`git-sign-nostr` verify mode (`do_verify`) is the surface most exposed to
adversarial input, since a malicious repository could carry a crafted
`gpgsig` header that ends up in the signature file `git` passes to
`--verify`. Bounds observed:

| Guard | Limit | Site |
|---|---|---|
| Signature file size | 8 KiB (`MAX_SIG_FILE`) | `lib.rs:123-124`, enforced in `read_bounded_file` `lib.rs:1576-1618` |
| Base64 line length | 4096 bytes (`MAX_BASE64_LINE`) | `lib.rs:129`, checked `lib.rs:1150-1156` |
| Decoded JSON size | 2048 bytes (`MAX_JSON_DECODED`) | `lib.rs:126`, checked `lib.rs:1164-1170` |
| Payload (git object) size | 100 MiB (`MAX_PAYLOAD`) | `lib.rs:119-121`, enforced `read_payload_stdin` `lib.rs:1552-1567` |
| Auth tag size (env/config) | 1024 bytes (`MAX_AUTH_TAG`) | `lib.rs:539-543` |
| Signature file must be a regular file | rejects FIFOs/devices via `O_NONBLOCK` + `is_file()` check | `lib.rs:1576-1596` |

All of these match or are tighter than the NIP-GS spec's own stated limits
(2048-byte JSON, 4096-byte base64, 100 MB payload — `docs/nips/NIP-GS.md` §
Denial of Service), and are exercised by dedicated tests
(`test_armor_rejects_oversized_b64` `lib.rs:2072-2080`,
`test_envelope_rejects_t_out_of_range` `lib.rs:2404-2411`, etc.). JSON parsing
also explicitly rejects unknown keys (`lib.rs:1352-1357`), non-canonical
whitespace (`lib.rs:1213-1220`), and duplicate-key-style ambiguity is avoided
by the canonical-reconstruction byte-comparison (`lib.rs:1223-1231`) rather
than a dedicated duplicate-key detector — since `serde_json::from_str` into a
`serde_json::Value` silently keeps the *last* occurrence of a duplicate key
(standard serde_json behavior), and the canonical-reconstruction check would
then compare the rebuilt single-value JSON against the original *multi-key*
source string and fail to match (because the original had two `"pk":...`
occurrences and the rebuild has one) — so duplicate keys are caught
transitively by the anti-malleability check rather than by an explicit
"reject duplicate keys" branch. This satisfies the *outcome* NIP-GS requires
("verifiers MUST reject the signature" on duplicate keys) without a
purpose-built duplicate-key scanner; no test explicitly exercises a
duplicate-key JSON payload, so this reasoning is inferred from the code path,
not confirmed by a passing test — flagged as a coverage gap.

`git-credential-nostr`'s stdin parser (`parse_stdin`, `lib.rs:107-132`) has
**no explicit byte/line-count bound** — it reads `stdin.lock().lines()` in an
unbounded loop until a blank line or EOF. Since the loop only terminates on a
blank line (`lib.rs:115-117`) or read error (`lib.rs:112-114`), a malicious or
buggy git build that never sends a terminating blank line, or sends an
extremely long single line, could cause unbounded memory growth or an
unbounded read — there is no equivalent to `git-sign-nostr`'s `MAX_PAYLOAD`/
`MAX_SIG_FILE` pattern here. In practice the input is produced by git itself,
not directly by a remote server, which narrows the realistic threat model
(the task's framing — "a git remote or malicious repo config could feed
adversarial input to a credential helper" — is more applicable to `host=`/
`wwwauth[]=` *content* than to the transport's line count, since those two
fields' *values* do originate from server-controlled data: the `Host` the
remote redirected to, and the literal `WWW-Authenticate` header the server
sent back). No test in `tests/integration.rs` exercises an oversized or
malformed stdin stream, an unterminated input, or adversarial control
characters in `host=`/`wwwauth[]=` — this is a genuine untested gap, stated
here as a finding rather than assumed safe.

#### Validation of server-controlled `wwwauth[]=`/`host=`/`path=` content

`parse_method` (`lib.rs:135-149`) does simple substring/split parsing of the
`wwwauth[]=` value with no length bound and no charset restriction — a
`method="..."` value containing arbitrary bytes (as long as it parses as an
`HttpMethod`, delegated to the `nostr` crate's `FromStr` impl) would be
accepted. `host`/`path` are interpolated directly into a `format!` URL string
(`lib.rs:206`) with no validation beyond what `Url::parse` will accept (or
reject, triggering the `panic!` documented in the Conventions doc) — there is
no allowlist of expected characters, no length cap, and no rejection of
control characters. Given these three fields ultimately originate from data
the *remote server* controls (the `Host` header of the redirect chain git
followed, and the literal `WWW-Authenticate` response header), this is the
one place in either crate where genuinely server-controlled input reaches a
`panic!`-capable code path (`Url::parse(&url).unwrap_or_else(|e|
panic!(...))`, `lib.rs:226`) rather than a clean `Result`-based error — a
crash-on-malformed-input outcome (denial of service against the git
operation, not against any other process) rather than the graceful exit `1`
every other error path in this crate uses.

#### Summary table

| Concern | `git-sign-nostr` | `git-credential-nostr` |
|---|---|---|
| Key source precedence | `NOSTR_PRIVATE_KEY` > `BUZZ_PRIVATE_KEY` > keyfile | `NOSTR_PRIVATE_KEY` > keyfile (no `BUZZ_PRIVATE_KEY`) |
| Keyfile symlink rejection | yes (`O_NOFOLLOW`) | no |
| Keyfile ownership check | yes (`getuid`) | no |
| Keyfile size cap | 1024 bytes | 256 bytes |
| Secret zeroized on drop | yes, plus manual stack overwrite | yes (`String::zeroize`) but via `nostr::Keys` (residual copy likely) |
| `git config` subprocess env scrubbed | yes | no |
| Key ever in stdout/stderr/error | no | no |
| Adversarial-input size bounds | yes, five explicit constants | no bound on stdin line loop |
| Crash-capable (`panic!`) path reachable from external input | no | yes (`lib.rs:226`) |
