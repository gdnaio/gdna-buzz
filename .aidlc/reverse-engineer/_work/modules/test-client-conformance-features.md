## Module: buzz-test-client — Nostr interop & multi-tenant conformance E2E (`crates/buzz-test-client/tests`)
### Aspect: Features

Read from an external/protocol-compliance reader's perspective: what capability does this test
suite collectively demonstrate the relay has, if every test in it were green?

---

#### 1. NIP-level interoperability the suite (would) prove

If every `#[ignore]`d test in `e2e_nostr_interop.rs` passed against a live relay, an external
reader could conclude the relay:

- **Implements NIP-50 search** with EOSE-bounded historical results, one-shot semantics (no live
  tail on a search subscription), rejection of the search-plus-non-search "mixed filter" shape
  that the NIP-50 spec leaves relay-defined, and genuine text-relevance ranking rather than
  chronological ordering dressed up as search (`e2e_nostr_interop.rs:262-435`, `:1055-1113`).
- **Implements NIP-10 thread markers** (`e`-tag `reply`/`root` positional markers) with
  server-side validation: a reply must name a real parent, a claimed root must agree with the
  actual parent chain, and the relay maintains its own `depth`/`broadcast`-driven notion of which
  replies surface at "top level" independent of the thread-query surface
  (`e2e_nostr_interop.rs:439-586`, `:878-967`).
- **Implements NIP-17-adjacent gift-wrap transport handling**: kind:1059 events are accepted from
  a signer distinct from the authenticated connection (a structural requirement for gift wraps,
  whose whole purpose is sender-anonymity via an ephemeral signing key), gated behind a mandatory
  `#p` filter on read, correctly routed to the addressed recipient, and — the property closest to
  an actual privacy guarantee — **excluded from full-text search indexing** so that encrypted DM
  envelopes can never surface as search hits (`e2e_nostr_interop.rs:589-756`, `:971-1051`).
- **Supports a Buzz-specific DM-discovery layer** (kinds 39000/44100/41010/41012) with relay-signed
  visibility snapshots (kind:30622) that are per-viewer, privacy-gated against third parties across
  four distinct bypass shapes (plain REST, WS REQ, kindless `ids` escape hatch, NIP-50 search
  escape hatch), and monotonically correct under same-second hide/reopen races
  (`e2e_nostr_interop.rs:760-874`, `:1254-1699`).
- **Implements a channel-window read model** (kinds 39005/39006 overlay events over `POST /query`)
  that separates top-level rows from thread replies, attaches relay-signed (not client-forgeable)
  summary/bounds metadata, and paginates correctly through exact-multiple boundaries — a
  capability well beyond bare NIP-01 filter semantics (`e2e_nostr_interop.rs:1776-1986`).
- **Correctly implements two NIP-01 edge cases** relays commonly get wrong: `kinds:[]` matching
  nothing (not everything), and multi-filter OR semantics surviving post-match deduplication
  (`e2e_nostr_interop.rs:1116-1206`).

From a protocol-compliance reader's standpoint, this file is a fairly serious interop
conformance suite for a relay that layers substantial custom behavior (DM discovery, channel
windows) on top of standard Nostr NIPs — it is not merely smoke-testing that events round-trip.

---

#### 2. Multi-tenant isolation guarantee the suite (would) prove

If every non-stub test in `conformance_multitenant.rs` passed against a genuinely two-host relay
deployment, an external reader could conclude:

- **Host-derived tenancy binds correctly and fails closed.** An unmapped host is rejected
  generically (no tenant enumeration via status code or error body) at both the HTTP and
  WebSocket-upgrade doors, and a client cannot override the host-derived community by supplying a
  disagreeing `#h` tag (`row_zero_host_binding`).
- **The unauthenticated relay-info surface (NIP-11) cannot be used to enumerate tenants** — every
  community serves an identical document modulo its own icon, including unmapped hosts
  (`nip11_relay_info`).
- **NIP-98 replay protection is genuinely wired into the request path** (not a decorative
  no-op) — a repeated authenticated event is rejected the second time, in the same community
  (`api_tokens_nip98_replay`).
- **Profile, NIP-05 identity, channel content, workflow triggers, full-text search results and
  deletes, presence, and typing fan-out are all correctly partitioned per community** even under
  the adversarial fixture shape of a *shared* UUID/pubkey/local-part used simultaneously in both
  communities — the isolation holds despite maximal opportunity for the fence to be missing
  (`users_profiles_nip05`, `channels_membership`, `workflows`, `search_fts`,
  `pubsub_presence_typing`).

**But** — and this is the feature-level headline finding for this module — as of the code
reviewed, **11 of the file's 19 obligation rows have no live test at all** (8 explicit
`pending_lane` stubs plus 2 doc-only modules with zero test functions), and the file's own
top-of-file doc comment states these obligations require "a running multi-tenant relay with two
host mappings" (`conformance_multitenant.rs:20-26`) that no script, compose file, or CI job in
this repository is confirmed to provision (see Integrations/Configuration/Debt docs). So the
*feature the test suite is written to prove* — "Buzz's relay correctly isolates multiple tenants
sharing one process/DB/Redis" — is only asserted in the source tree for roughly 8 of 19 named
obligation areas, and this agent could not establish that even those 8 have ever actually been
run against a live two-host fixture (as opposed to merely compiling). See Debt doc for the CI
evidence.

---

#### 3. Relationship between the two files as a combined feature statement

Per the file's own module doc-comment (`conformance_multitenant.rs:9-19`), `e2e_nostr_interop.rs`
(along with `e2e_relay.rs`, `e2e_media.rs`, `e2e_git.rs`) is explicitly named as the **N=1 parity
oracle** for the eventual multi-tenant rewrite: the obligation is that these suites keep passing
unchanged, run with `RELAY_URL` pointed at whatever new multi-tenant-capable relay build
eventually exists, as proof that single-tenant behavior didn't regress. The `n1_parity` module
(`:2728-2739`) states this explicitly and assigns no new assertion to itself — it just documents
that `e2e_nostr_interop.rs` and its siblings **are** the N=1 half of the conformance story. This
means the two files this agent owns are not just adjacent in the same crate; one of them
(`e2e_nostr_interop.rs`) is *referenced by name in spirit* as a dependency of the other's
stated purpose, even though neither imports the other and `conformance_multitenant.rs` never
executes `e2e_nostr_interop.rs`'s tests itself.
