## Module: git-sign-nostr & git-credential-nostr (`crates/git-sign-nostr`, `crates/git-credential-nostr`)
### Aspect: Features

#### Overview

Together these two crates let a git user or CI pipeline authenticate to, and
sign commits for, Buzz's git-over-HTTP surface using nothing but a Nostr
keypair — no GPG key, no SSH key, no username/password, no PAT. Each crate
delivers one half of that story and each can be adopted independently.

#### Feature: password-less push/fetch against Buzz git remotes

Installing `git-credential-nostr` and configuring
`credential.helper=nostr` + `credential.useHttpPath=true`
(`crates/git-credential-nostr/README.md` § Setup) means every `git clone` /
`git push` / `git fetch` against a Buzz relay's `/git/{owner}/{repo}` HTTP
surface transparently authenticates via a freshly-minted NIP-98 event, with
no interactive password prompt and nothing stored on disk beyond the user's
own Nostr key. Concretely:

- The credential is derived, not looked up — `run()`
  (`crates/git-credential-nostr/src/lib.rs:152-264`) builds a brand-new
  kind:27235 event scoped to the exact request URL and method on every
  invocation.
- `ephemeral=true` in the response (`lib.rs:261`) tells git not to cache it,
  so there is no stale-credential problem and no credential store to clear
  on key rotation.
- CI/CD gets the same UX via `$NOSTR_PRIVATE_KEY` with zero filesystem
  writes (`crates/git-credential-nostr/README.md` § CI / CD).

**Limit:** this only activates for hosts that challenge with
`WWW-Authenticate: Nostr ...` — it is *additive*, not a replacement for
existing credential helpers. A non-Buzz remote (GitHub, GitLab, etc.) falls
straight through to the next configured helper because the decision tree
declines silently before any Nostr-specific work happens
(`lib.rs:180-187`, see Business Rules doc). This is what makes it safe to set
globally (`buzz-dev-mcp`'s shim does exactly that — see Integrations doc).

**Limit:** requires **git ≥ 2.46** for the `authtype`/`credential`/
`ephemeral` capability extension (`crates/git-credential-nostr/README.md` §
Requirements; enforced implicitly — an older git simply never sends
`capability[]=authtype`, so the helper always takes the silent-decline path
at `lib.rs:160-164`, and the request falls back to git's basic
username/password prompt, which the relay's NIP-98-only git auth cannot
satisfy).

**Limit:** requires `credential.useHttpPath=true` — without it, git omits
`path=` from the request and the helper cannot construct the repo-root URL
NIP-98 needs, and hard-fails with a clear error rather than silently
mis-authenticating (`lib.rs:195-198`).

#### Feature: NIP-OA owner delegation carried through git push auth

If a `BUZZ_AUTH_TAG` (or `nostr.authtag` git config) is present,
`git-credential-nostr` embeds it as a signed tag on the same NIP-98 event
(`lib.rs:236-239`), so an agent pushing on an owner's behalf can prove that
delegation to the relay in the same request that authenticates the push —
no separate header or side-channel is needed (the code comment at
`lib.rs:75-77` explains this is because git's credential protocol can only
return a single opaque credential value, not extra headers). Verified
end-to-end by
`includes_nip_oa_auth_tag_in_signed_event`
(`crates/git-credential-nostr/tests/integration.rs:126-172`), which confirms
the tag survives inside the *signed* event (i.e. tampering would break the
signature) and that a malformed tag fails closed rather than silently
omitting delegation
(`malformed_nip_oa_auth_tag_fails_closed`, `tests/integration.rs:174-186`).

**Limit:** this is push-auth delegation only — it says nothing about who
authored the *commits themselves*. Commit-level delegation is a separate
feature, provided by `git-sign-nostr`'s own (optional) `oa` field.

#### Feature: commits/tags cryptographically signed by a Nostr identity

Installing `git-sign-nostr` as `gpg.x509.program` and enabling
`commit.gpgsign`/`tag.gpgsign` (`crates/git-sign-nostr/README.md` § Usage)
gives every commit and tag a detached BIP-340 Schnorr signature over a
domain-separated hash of the git object, verifiable with plain
`git verify-commit`/`git verify-tag`/`git log --show-signature` — no
Nostr-aware tooling is needed to *see* that a commit is signed, because the
armor format (`-----BEGIN SIGNED MESSAGE-----`) is git's standard x509
signature marker (chosen specifically to avoid colliding with PGP/SSH
markers that platforms like GitHub already parse — `docs/nips/NIP-GS.md` §
How Git Invokes Signing Programs, point 1).

**Limit:** this establishes "this Nostr key signed this content," not
"this content is trustworthy" or "this signer is who they claim" — the
module doc is explicit that `TRUST_FULLY` is advisory-only, matching only
the locally-configured `user.signingkey`, never a real trust root
(`crates/git-sign-nostr/src/lib.rs:20-27`).

**Limit (platform):** `git-sign-nostr` is Unix-only — its own module doc
states this plainly (`lib.rs:13`), because status-fd handling requires
Unix file-descriptor passing (`libc::fcntl`/`from_raw_fd`,
`lib.rs:320-334`). There is no Windows fallback path in this crate (contrast
with `git-credential-nostr`, which has an explicit `#[cfg(not(unix))]`
degraded-but-functional branch for permission checks,
`crates/git-credential-nostr/src/lib.rs:41-44`).

#### Feature: verifiable owner attestation embedded in the commit signature

`git-sign-nostr` can optionally embed a NIP-OA `oa` field directly in the
signature envelope (`build_envelope`, `lib.rs:924-936`), so "this commit was
made by agent key X, authorized by owner key Y" is a single, offline-
verifiable fact carried inside the git object itself — no relay round-trip
needed to check delegation, unlike the push-time NIP-98 case. `git-sign-nostr`
additionally surfaces this as a machine-readable status line
(`NOTATION_NAME nostr-oa-status` / `NOTATION_DATA valid|invalid_signature|
expired|kind_not_applicable|none`, `lib.rs:1305-1318`) so any tool consuming
`--status-fd` output — not just humans reading `git log` — can branch on
delegation validity.

**Limit:** because the `oa` field is folded into the signing hash, it is
immutable once signed (`docs/nips/NIP-GS.md` § Immutability) — an owner
cannot revoke a specific commit's attestation after the fact; only future
signing can omit or replace it. There is no revocation-list mechanism in
either crate.

#### Feature: single Nostr identity spans relay, git auth, and git signing

Because both crates and the relay's own NIP-42/NIP-98 auth all consume the
same key-loading precedence
(`NOSTR_PRIVATE_KEY` > `BUZZ_PRIVATE_KEY` > keyfile), a single Nostr keypair
is simultaneously: the identity that authenticates WebSocket connections to
Buzz, the identity that authenticates git push/fetch
(`git-credential-nostr`), and the identity that signs the resulting commits
(`git-sign-nostr`) — exactly the "one identity, one key, across all surfaces"
motivation stated in `docs/nips/NIP-GS.md` § Motivation. This module does
not implement that unification itself (it's a consequence of shared
conventions, not shared code — see Debt doc for the resulting duplication),
but it is the headline capability the two crates jointly unlock.

#### Feature: automatic wiring into agent-spawned shells

Neither crate wires itself into a shell automatically — that orchestration
lives in `buzz-dev-mcp`'s `Shim::install`
(`crates/buzz-dev-mcp/src/shim.rs:24-70`, out of this module's scope), which
symlinks both binaries into a per-session `PATH` prefix and derives the
full `GIT_CONFIG_*` env-var set (`gpg.format=x509`,
`gpg.x509.program=git-sign-nostr`, `credential.helper=nostr`,
`credential.useHttpPath=true`, `user.signingkey=<hex pubkey>`, etc.,
`shim.rs:186-213`) from a single `NOSTR_PRIVATE_KEY`. The *feature* this
enables — an AI agent's shell tool calls get committing/pushing identity for
free, without per-repo `git config` setup — is real and worth recording here
because it is the primary deployed use case per NIP-GS's own Motivation
section ("autonomous agents that commit code on behalf of their owners"),
but the mechanism itself belongs to `buzz-dev-mcp`, not to these two crates.

#### Feature comparison table

| Capability | Provided by | Verifiable via | Platform limit |
|---|---|---|---|
| Password-less git push/fetch to Buzz | `git-credential-nostr` | relay's NIP-98 verification (±60s window) | requires git ≥ 2.46 |
| Push-time NIP-OA delegation | `git-credential-nostr` | relay checks the `auth` tag on the NIP-98 event | none noted |
| Commit/tag signing with Nostr key | `git-sign-nostr` | `git verify-commit` / `git log --show-signature` | Unix-only |
| Commit-time NIP-OA delegation (offline-verifiable) | `git-sign-nostr` | `nostr-oa-status` NOTATION line, or manual verification per NIP-GS | Unix-only; immutable once signed |
