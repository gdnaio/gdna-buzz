## Module: git-sign-nostr & git-credential-nostr (`crates/git-sign-nostr`, `crates/git-credential-nostr`)
### Aspect: Integrations

#### Overview

Both crates integrate with three things: git itself (via two different
pluggable-program interfaces), the Nostr/BIP-340 primitives supplied by the
`nostr` crate, and — indirectly, never as a compile-time dependency — Buzz's
relay git-HTTP surface and `buzz-dev-mcp`'s shim. Neither crate depends on
any other Buzz crate (`buzz-core`, `buzz-auth`, `buzz-sdk`, etc.) or on each
other.

#### Integration with git: config keys involved

Verified by reading, not assumed:

| Config key | Read/written by | Site |
|---|---|---|
| `gpg.format` | git itself (not read by either crate) — must be `x509` for git to invoke `gpg.x509.program` | `docs/nips/NIP-GS.md` § Git Configuration; set by `buzz-dev-mcp/src/shim.rs:206` |
| `gpg.x509.program` | git itself — points at `git-sign-nostr` | `crates/git-sign-nostr/README.md` § Usage; set by `shim.rs:207` |
| `commit.gpgsign` / `tag.gpgsign` | git itself | `git-sign-nostr/README.md`; set by `shim.rs:208-209` |
| `user.signingkey` | **read** by `git-sign-nostr` via `git_config("user.signingkey")` in `determine_trust` (`crates/git-sign-nostr/src/lib.rs:1673-1682`); also arrives as the `-u <key>` argv value checked in `do_sign` (`lib.rs:980-993`) | `lib.rs:1673-1682, 980-993` |
| `nostr.keyfile` | **read** by both crates as the fallback key source — `git-sign-nostr`: `git_config("nostr.keyfile")` (`lib.rs:437-441`); `git-credential-nostr`: `git_config("nostr.keyfile")` (`crates/git-credential-nostr/src/lib.rs:58`) | both `lib.rs` files |
| `nostr.authtag` | **read** by both crates as the fallback NIP-OA source when `BUZZ_AUTH_TAG` env is unset — `git-sign-nostr` via `git_config_strict("nostr.authtag")` (`lib.rs:472-473`); `git-credential-nostr` via `git_config("nostr.authtag")` (`crates/git-credential-nostr/src/lib.rs:82`) | both `lib.rs` files |
| `credential.helper` | git itself — must name `nostr` (resolves to the `git-credential-nostr` binary via `PATH` or full path) | `git-credential-nostr/README.md` § Setup; set by `shim.rs:203` |
| `credential.useHttpPath` | git itself — must be `true` or `path=` is never sent to the helper | `git-credential-nostr/README.md`; enforced client-side at `lib.rs:195-198`; set by `shim.rs:204` |

Neither crate reads `gpg.format` or `credential.helper` itself — those are
consumed entirely by git's own dispatch logic before either binary is ever
invoked. This module only reads the config keys that affect its *own*
runtime behavior (`nostr.keyfile`, `nostr.authtag`, `user.signingkey`).

#### Integration with git: invocation protocols

`git-sign-nostr` implements git's GPG-signing-program interface
(`gpg-interface.c`'s `sign_buffer_gpg`/`verify_gpg_signed_buffer`, per
`docs/nips/NIP-GS.md` § How Git Invokes Signing Programs), confirmed by the
argv shapes `parse_args` accepts (`-bsau <key>` for signing, `--verify <file>
-` for verification, `crates/git-sign-nostr/src/lib.rs:196-280`) and by the
`[GNUPG:]`-prefixed status protocol it emits
(`GNUPG_PREFIX = "[GNUPG:] "`, `lib.rs:133`).

`git-credential-nostr` implements git's credential-helper protocol (the
`get`/`store`/`erase` verb dispatch, `key=value\n`-line stdin,
`key=value\n`-line stdout, terminated by a blank line) — confirmed by
`parse_stdin`/the response block described in the API Surface doc. It
specifically opts into the newer `authtype`/`credential`/`ephemeral`
capability extension (requires git ≥ 2.46 per the crate's own `README.md`)
rather than the legacy `username=`/`password=` fields.

#### Integration with the `nostr` crate

Both crates depend on the workspace `nostr = { workspace = true }` dependency
(pinned to `0.44.3` via `Cargo.lock`), but use different surface area:

- `git-sign-nostr` (`crates/git-sign-nostr/Cargo.toml:29-33`) uses the
  low-level re-exports: `nostr::secp256k1::{Keypair, Message, schnorr::
  Signature}`, `nostr::hashes::sha256::Hash`, `nostr::{FromBech32, PublicKey,
  SecretKey, SECP256K1}` (`crates/git-sign-nostr/src/lib.rs:81-89`) — it
  builds its own signing/verification/hash pipeline on top of the crate's
  cryptographic primitives rather than using `nostr::Keys`/`EventBuilder` for
  anything (deliberately, per the zeroization rationale in the Data Model
  doc).
- `git-credential-nostr` (`crates/git-credential-nostr/Cargo.toml:9-12`) uses
  the high-level API: `nostr::nips::nip98::{HttpData, HttpMethod}`,
  `nostr::{EventBuilder, Keys, Tag}`, `nostr::types::Url`
  (`crates/git-credential-nostr/src/lib.rs:9-11`) — it delegates the entire
  NIP-98 event construction and signing to the `nostr` crate's own
  `EventBuilder::http_auth`/`sign_with_keys`.

This is the same `nostr` dependency the rest of the Buzz workspace uses
(`Cargo.toml:61`, features `["nip44", "nip98"]` at the workspace level), so
no separate crypto stack is introduced.

#### Integration with the relay's git smart-HTTP surface

`git-credential-nostr` is the client half of an auth handshake whose server
half lives in `crates/buzz-relay/src/api/git/transport.rs` (out of this
module's scope to analyze in depth, but the wire contract is directly
relevant and was read to confirm it matches):

1. Relay responds `401` with header
   `WWW-Authenticate: Nostr realm="buzz", method="<GET|POST>"`
   (`crates/buzz-relay/src/api/git/transport.rs:88-94, 99-105`).
2. Git surfaces this via `wwwauth[]=` on the credential-helper stdin protocol;
   `git-credential-nostr`'s `parse_method` extracts the `method="..."` token
   (`crates/git-credential-nostr/src/lib.rs:135-149`).
3. `git-credential-nostr` reconstructs the repo-root URL by stripping
   `/info/refs`, `/git-upload-pack`, or `/git-receive-pack` from `path`
   (`lib.rs:200-205`) — this **exactly mirrors** the relay's own
   `git_expected_url` reconstruction (`transport.rs:235-253`, confirmed by
   reading both: same three suffixes, same fallback order), which is what
   lets the relay's NIP-98 verification (`u` tag = repo-root URL) match the
   token the client signed.
4. `git-credential-nostr` builds and signs a kind:27235 event via
   `EventBuilder::http_auth(HttpData::new(parsed_url, method))`
   (`lib.rs:226-235`) — a standard NIP-98 event, not a Buzz-specific format.
5. The relay decodes the `Authorization: Nostr <base64>` header back into a
   `nostr::Event` and calls `buzz_auth::nip98::verify_nip98_event(&event_json,
   &expected_url, &event_method, None)` (`transport.rs:183-185`), which
   enforces a ±60-second timestamp window
   (`crates/buzz-auth/src/nip98.rs:18,32,77-79`) — this is the real
   expiry/replay-limiting mechanism behind `git-credential-nostr`'s
   `ephemeral=true` response field.
6. The relay explicitly does **not** verify the HTTP method on git routes
   (`transport.rs:150-166`, code comment: "Git's credential helper signs with
   `method=GET`... then reuses the token for POST... Method binding can't
   work here") — meaning `git-credential-nostr`'s `parse_method` extraction
   is used only to build the initial signed event, and the relay's own
   method check is a documented no-op for this path. This is a genuine
   protocol looseness on the server side, not a client-side bug in this
   module — noted here because a reader of `git-credential-nostr` alone
   might assume method-scoping is enforced end-to-end; it is not.
7. If `git-credential-nostr` embedded a NIP-OA `auth` tag (from `BUZZ_AUTH_TAG`/
   `nostr.authtag`), the relay extracts it from the *event* itself
   (`crate::handlers::auth::extract_auth_tag_json(&event)`,
   `transport.rs:198-199`) rather than from any HTTP header — confirming the
   doc-comment rationale in `git-credential-nostr/src/lib.rs:75-77` ("Git's
   credential protocol... cannot add a separate HTTP header") is accurate:
   the relay's design accommodates exactly this constraint.

`git-sign-nostr` has **no relay integration at all** — NIP-GS is explicitly
scoped as relay-uninvolved (`docs/nips/NIP-GS.md` § Non-Goals: "This NIP does
not require relay changes"). Confirmed by grep: no reference to
`git-sign-nostr`/`git_sign_nostr` exists anywhere under `crates/buzz-relay/`.

#### Relationship to `buzz-dev-mcp`'s `Shim::install`

Confirmed directly by reading `crates/buzz-dev-mcp/src/shim.rs` (read only to
verify this specific relationship, per the task's scope boundary — no
broader analysis of `buzz-dev-mcp` performed here):

- `Shim::install()` (`shim.rs:24-70`) creates a `0700` tempdir and symlinks
  the running `buzz-dev-mcp` executable to five names inside it, including
  `git-credential-nostr` and `git-sign-nostr` (`shim.rs:31-38`) — a
  multicall pattern.
- `buzz-dev-mcp`'s own `run()` dispatches on `argv[0]`
  (`crates/buzz-dev-mcp/src/lib.rs:138-160`) and for these two names calls
  straight into this module's library functions:
  `"git-credential-nostr" => std::process::exit(git_credential_nostr::run())`
  and `"git-sign-nostr" => std::process::exit(git_sign_nostr::run())`
  (`lib.rs:151-152`). This is the **only** place either crate's `pub fn
  run()` is called from outside its own `main.rs` — confirmed by grep across
  the workspace for `git_sign_nostr::run\|git_credential_nostr::run`
  (matches only in each crate's own `main.rs` and here).
- The shim also derives the full ephemeral `GIT_CONFIG_*` environment
  (`build_git_env`, `shim.rs:186-213`) that wires `gpg.x509.program=
  git-sign-nostr`, `credential.helper=nostr`, `credential.useHttpPath=true`,
  `nostr.keyfile=<shim-dir>/.nostr-key`, and `user.signingkey=<pubkey>` —
  every config key both crates read (see table above) is populated
  exclusively by this shim when running under `buzz-dev-mcp`, not by any
  logic inside `git-sign-nostr`/`git-credential-nostr` themselves. Confirming
  `buzz-dev-mcp`'s own README/behavior in depth is out of scope for this
  module; this note only records the specific coupling the task asked to
  verify.
- Confirmed by `Cargo.toml`: `buzz-dev-mcp` depends on both crates by path
  (`crates/buzz-dev-mcp/Cargo.toml:18-19`:
  `git-credential-nostr = { path = "../git-credential-nostr" }`,
  `git-sign-nostr = { path = "../git-sign-nostr" }`), which is the compile-time
  half of the same relationship.

#### Relationship to `buzz-cli`

None at the protocol level. `buzz-cli` never invokes, links, or shells out to
either crate — confirmed by the existing top-level analysis at
`.aidlc/reverse-engineer/integrations.md:136` and
`_work/modules/cli-commands-dev-integrations.md:135-143`, which grepped
`crates/buzz-cli/` for `git-sign-nostr\|git_sign_nostr\|git-credential-nostr\|
git_credential_nostr` and found zero matches. The one indirect touchpoint is
data-only: `buzz-cli patches send --commit-pgp-sig` accepts a pre-existing
PGP/NIP-GS signature string as a CLI flag and passes it through verbatim into
a `GitPatchMeta.commit_pgp_sig` field (`crates/buzz-cli/src/commands/
patches.rs:8-13, 21, 41` — confirmed by reading) without generating,
parsing, or validating it — the signature would have to be produced by
`git-sign-nostr` (or GPG) upstream of the CLI call. `buzz-cli` itself never
depends on these crates in `Cargo.toml`.

#### Dependencies each crate declares

`git-sign-nostr` (`crates/git-sign-nostr/Cargo.toml:20-42`): `base64 = "0.22"`,
`hex` (workspace), `zeroize` (workspace, `features = ["derive"]` — see Debt
doc for whether the derive feature is actually used), `nostr` (workspace),
`serde_json` (workspace), `chrono` (workspace), and Unix-only `libc = "0.2"`.

`git-credential-nostr` (`crates/git-credential-nostr/Cargo.toml:9-12`):
`nostr` (workspace), `serde_json` (workspace), `zeroize` (workspace),
`base64 = "0.22"`.

Neither declares a dependency on the other, nor on `buzz-core`, `buzz-sdk`,
`buzz-auth`, or any other in-repo crate — both are leaf crates in the
dependency graph except for being depended *on* by `buzz-dev-mcp`.

#### Test-side integration: `buzz-test-client`

`crates/buzz-test-client/tests/e2e_git.rs` (out of scope to analyze deeply,
but its coupling to `git-credential-nostr` is a direct integration worth
recording) spawns real `git` subprocesses configured with
`credential.helper=<path-to-git-credential-nostr-binary>` via `-c` flags
(`e2e_git.rs:66-77`) to drive full clone/push/fetch/force-push/concurrent-push
scenarios against a live relay + MinIO backend, per `docs/git-on-object-storage.md`.
This is the only place in the repository that exercises `git-credential-nostr`
against a real relay end-to-end; it does not exercise `git-sign-nostr` at
all (every `git` invocation in that file explicitly sets
`-c commit.gpgsign=false -c tag.gpgsign=false`, `e2e_git.rs:70-71`) — see the
Conventions/Debt docs for what this implies about `git-sign-nostr` test
coverage.
