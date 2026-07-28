## Module: buzz-test-client — Nostr interop & multi-tenant conformance E2E (`crates/buzz-test-client/tests`)
### Aspect: Security

---

#### 1. `e2e_nostr_interop.rs` — does the NIP-17 test verify metadata-hiding, or just round-trip?

Per the task brief's specific question: **the file verifies transport-level metadata-hiding
properties, not payload confidentiality, and it does not exercise NIP-17's cryptographic unwrap
chain at all.**

Precisely, across the three gift-wrap tests:

| Test | What it proves | What it does *not* prove |
|---|---|---|
| `test_nip17_gift_wrap_accepted` (`:589-618`) | the relay accepts a kind:1059 signed by a signer **different** from the connection's authenticated key — i.e. the relay does not enforce "event pubkey == auth pubkey" for this kind, which is the structural precondition NIP-17 needs (a gift wrap's outer signature is an ephemeral throwaway key, not the sender's real identity) | nothing about the *content* — `"encrypted-content"` is a literal placeholder string, not NIP-44 ciphertext |
| `test_nip17_gift_wrap_requires_p_filter` (`:622-673`) | a bare `{kinds:[1059]}` subscription with no `#p` filter is refused — this is a **recipient-metadata-hiding property at the query level**: nobody can enumerate all gift wraps on the relay; they can only ever ask "show me gift wraps addressed to a pubkey I can prove interest in via the filter" | nothing about whether the `#p` tag itself could be used to link sender/recipient pairs — the test does not check whether an observer with a valid `#p` filter learns anything about the *sender* |
| `test_nip17_gift_wrap_recipient_receives` (`:677-756`) | correct routing: the addressed recipient receives the exact event | nothing about the sender's real identity being hidden — the test itself signs with an `ephemeral_keys` throwaway key (`:706`), so by construction there's no real sender identity in this test to leak or protect |
| `test_nip17_gift_wrap_not_searchable` (`:971-1051`, cross-listed in Business Rules) | **this is the one genuine privacy-regression test in the file**: it proves a kind:1059 event with a known unique token is excluded from NIP-50 search results while a kind:9 control with the same token is included — i.e. the relay's full-text index cannot be used as a side-channel to discover the existence/content of gift-wrapped DMs | nothing about metadata beyond searchability — no check of whether gift-wrap timing, size, or `#p` tag presence leaks information through some *other* channel (feed reads, count queries, etc.) |

**Overall characterization**: this file treats "gift wrap" as a transport envelope whose privacy
property is *the relay must not treat kind:1059 like an ordinary readable/searchable event*, and
it proves exactly that — acceptance despite signer mismatch, `#p`-gated read, correct delivery,
search exclusion. It explicitly does **not** test anything downstream of the wire event itself:
no NIP-44 encryption/decryption, no seal/rumor unwrapping, no verification that a relay operator
reading raw Postgres rows would see ciphertext rather than plaintext (that would require a DB-level
test, out of scope for a WS/REST black-box client). A reader should not conclude from this file
that NIP-17's end-to-end confidentiality guarantee (sender/recipient/content hidden from the relay
operator) is verified anywhere in this test suite — only the relay's *behavioral* treatment of the
already-encrypted envelope is verified.

The NIP-DV snapshot tests (`:1383-1699`, five tests) are a stronger and more thorough security
suite by comparison, deliberately probing four distinct bypass shapes (plain REST cross-viewer
query, WS `REQ`, kindless `ids` escape hatch, NIP-50 `search` escape hatch) for a single privacy
invariant (a viewer's hidden-DM snapshot is unreadable by anyone else). This is the file's most
rigorous security-testing pattern, more rigorous than its NIP-17 coverage.

---

#### 2. `conformance_multitenant.rs` — what it does and does not prove about the isolation boundary

Per the task brief: this file is indeed the closest thing in the crate to a genuine security test
of the multi-tenant isolation boundary. Characterizing precisely:

##### What it proves, when a test is live and its fixture exists

- **Fail-closed host binding** at both the HTTP and WS-upgrade doors, with a specific check that
  the rejection body doesn't leak the queried host string (`row_zero_host_binding`).
- **No enumeration oracle via NIP-11** — a genuinely adversarial check, since NIP-11 is by design
  unauthenticated (`nip11_relay_info`).
- **Client-supplied claims cannot override host-derived tenancy** — tested via the specific
  adversarial shape of a channel that exists in exactly one community, with a positive control
  proving the channel is otherwise genuinely functional (`row_zero_host_binding`'s second test).
- **Storage/cache/search fences hold under maximum-collision fixtures** — every positive test in
  `channels_membership`, `search_fts`, `pubsub_presence_typing`, `users_profiles_nip05`, and
  `workflows` deliberately uses the **same** UUID, pubkey, or local-part across both communities,
  which the files' own doc comments correctly identify as the necessary fixture shape to make a
  missing fence observable (as opposed to merely "different data happened not to collide").

##### What it does not prove — and where it explicitly says so

- **8 of 19 obligation rows are unimplemented stubs** (`membership_allowlist`,
  `channelless_global_events_dms` ×2, `feed_read_side_isolation`, `workflows::approval_token_is_community_confined`,
  `media_blossom`, `git_hosting`, `mesh_agents_cli`) — no security assertion exists for archived-
  identity isolation, DM-does-not-cross-deliver, feed-mention isolation, approval-token isolation,
  media-metadata isolation, git-repo isolation, or mesh/ACP cross-community bleed.
- **2 obligation rows have no wire test even in principle** (`audit_log`, `n1_parity`) — the
  `audit_log` module's doc comment explicitly argues no client-reachable surface for audit exists,
  so black-box testing this file's own contract is impossible for that row; it defers to
  `buzz-audit`'s unit tests.
- **1 row is a compiling no-op** (`token_minted_in_a_does_not_authorize_in_b`) — the `#[test]`
  function has an empty body; the doc comment explains the mint-token HTTP surface doesn't exist
  to test against, and defers to `buzz-db` unit tests.
- **The two-host fixture itself is unverified as ever having existed** — see Integrations doc.
  Every live assertion in this file is conditional on infrastructure this agent could not confirm
  is provisioned anywhere in the repository. A security reader should treat every "proves X" claim
  above as "proves X, *if this test has ever actually been executed against the described
  fixture*" — which this agent could not establish (see Debt doc for the CI evidence).

##### The specific question: does this suite's own assertions catch a tenant-isolation regression independent of the disabled `buzz-conformance` tracer?

**Yes, structurally — its assertions are entirely independent of `buzz_conformance::Tracer`/
`NoopTracer`.** As established in Data Model and API Surface, this file contains zero references
to any `buzz_conformance` type and is not a declared dependent of that crate. Its assertions are
wire-level: HTTP status codes, response body JSON field equality, WS message variants. A
regression in the relay's tenant-boundary logic (e.g. dropping a `WHERE community_id = $1` clause,
per the exact mutate-bites this file's own doc comments name) would be caught by *this file's*
assertions regardless of whether `AppState.tracer` is bound to `NoopTracer` or `JsonlTracer` in the
relay build under test — because this file never reads `AppState.tracer` or any trace artifact; it
only reads the HTTP/WS response the regression would itself corrupt.

**However — and this is the load-bearing caveat** — that independence is only as good as (a)
whether the specific regression's blast radius is one this file actually has a live (non-stub)
test for, and (b) whether the file is ever actually executed with the two-host fixture live. Given
that 11 of 19 rows are stub/doc-only, and given this agent could not confirm the fixture is ever
provisioned in CI (see Debt doc), the honest characterization is: **this file's design is sound
and its assertion mechanism is genuinely tracer-independent, but its current *actual* test-time
coverage of the tenant-isolation boundary is partial (roughly 8/19 obligation areas) and its
*execution* status is unconfirmed.** It is not fair to call this file "the test-time verification
of the multi-tenant gate" in present tense — it is better described as a partially-built,
never-confirmed-executed black-box isolation suite that is architecturally independent of, and a
credible eventual substitute for, the `buzz-conformance` crate's own (also-unexercised-in-
production) trace-replay mechanism.

##### Connection to the earlier `buzz-conformance` finding (production tracer is `NoopTracer`)

The two facts compound rather than compensate for each other:

1. Production binds `Arc::new(NoopTracer)` (per the `buzz-conformance` batch's security doc,
   `crates/buzz-relay/src/state.rs:798`) — so the `buzz-conformance` crate's own invariants protect
   nothing at runtime.
2. This file — the only other repo artifact that could plausibly serve as an independent,
   wire-level test-time check of the same tenant-isolation properties — is itself largely
   unimplemented (11/19 stubs) and its live tests' execution status against a real fixture is
   unconfirmed.

So the combined picture is: **neither the runtime conformance gate nor its most plausible
black-box test-suite substitute currently provides confirmed, exercised, end-to-end coverage of
the multi-tenant isolation boundary this repository's own design docs (`docs/multi-tenant-
conformance.md`) describe as a required migration gate.** The isolation properties that *are*
covered live in unit tests at lower layers (`buzz-db`, `buzz-auth`, `buzz-pubsub` — cited
extensively by this file's own doc comments as the "real" proof locations for several rows), which
this agent did not re-verify as being in scope for this analysis batch.

---

#### 3. Secrets and credential handling in both files

Neither file handles or logs a production secret. Both generate fresh, disposable `nostr::Keys`
per test (`Keys::generate()`, e.g. `e2e_nostr_interop.rs:264`, `conformance_multitenant.rs:263`) —
no hardcoded private keys, no `.env` values read for auth material. `conformance_multitenant.rs`'s
NIP-98 header builder (`build_nip98_header`, `:870-880`) constructs and immediately discards a
signed event in memory; nothing is written to disk. Both files rely on the relay's dev-mode
`X-Pubkey` header fallback (implied by `BUZZ_REQUIRE_AUTH_TOKEN=false`, per `TESTING.md`) rather
than minting real API tokens, consistent with `conformance_multitenant.rs`'s own doc-comment
observation that no token-minting wire surface exists to test against.
