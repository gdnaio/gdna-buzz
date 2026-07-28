## Module: buzz-test-client — Nostr interop & multi-tenant conformance E2E (`crates/buzz-test-client/tests`)
### Aspect: Integrations

---

#### 1. What `e2e_nostr_interop.rs` spins up

**Exactly one relay instance, externally provided.** The file starts nothing itself. Every test
calls `relay_url()` (`crates/buzz-test-client/tests/e2e_nostr_interop.rs:27-29`), which reads
`RELAY_URL` (default `ws://localhost:3000`) and `relay_http_url()` (`:31-37`), which derives the
HTTP form by string substitution (`wss://`→`https://`, `ws://`→`http://`). There is no
`docker compose`, `cargo run -p buzz-relay`, or process-spawn call anywhere in the file — the doc
comment at the top states plainly that "these tests require a running relay instance"
(`:4-5`) and defers startup entirely to whatever invokes `cargo test --test e2e_nostr_interop
-- --ignored` (per `TESTING.md`, or CI — see below).

Real dependent services touched **transitively through the relay** (never connected to directly
by this file): Postgres (event storage, thread metadata, FTS `search_tsv`), Redis (implied by the
relay's architecture for pub/sub — not directly exercised by any assertion in this file, since it
contains no presence/typing/fan-out test). No S3/Blossom, no git-smart-HTTP, no ACP harness.

The file's own network clients: `reqwest::Client` (REST, instantiated fresh per helper call —
e.g. `create_test_channel`, `:45`) and `buzz_test_client::BuzzTestClient` (WS, wraps
`buzz_ws_client::NostrWsConnection` per the harness doc read for context).

---

#### 2. What `conformance_multitenant.rs` spins up

**Also externally provided — but the fixture shape is qualitatively different and, per the task
brief's specific question, is worth stating precisely.** The module doc-comment is explicit
(`conformance_multitenant.rs:29-31`):

> "Both URLs MUST address the same relay process (same pod, same Postgres, same Redis); only the
> `Host` header differs. That is the whole point: one binary, one DB, two communities, provably
> isolated by `community_id` derived from the host."

So this is **not** "spin up two separate relay instances." It is **one relay process, addressed
via two different `Host` headers** (`a.localhost:3000` / `b.localhost:3000`, both of which resolve
to `127.0.0.1` per the standard `*.localhost` wildcard convention noted in the doc comment at
`:181-183`), where the relay's own host→community resolution logic (`bind_community`, referenced
throughout the file's doc comments but never called directly by this file) is expected to have
been configured with two community rows mapped to those two host strings.

The file provides **three URL-producing functions**, all env-var-driven with hardcoded
`a.localhost`/`b.localhost`/`unknown.localhost` fallbacks:

```rust
fn url_a()       // RELAY_URL_A,       default http://a.localhost:3000  (:45-47)
fn url_b()       // RELAY_URL_B,       default http://b.localhost:3000  (:50-52)
fn url_unknown() // RELAY_URL_UNKNOWN, default http://unknown.localhost:3000  (:58-61)
```

Nothing in this file, or anywhere else this agent could find in the repository (see §4 below),
constructs the "two communities on one relay" fixture these functions assume exists. The file is
written entirely on the assumption that some *other* process — a script, a CI job, a manually
operated dev relay — has already seeded a `communities` table with rows for `a.localhost:3000`
and `b.localhost:3000` before `cargo test --test conformance_multitenant -- --ignored` is run.

Real dependent services touched transitively (assuming the fixture exists): Postgres
(`api_tokens_nip98_replay`'s doc comment explicitly names the storage-layer isolation fence being
indirectly exercised, `:606-624`), Redis (`pubsub_presence_typing`'s presence half is explicitly
documented as Redis-backed with "no DB fallback," `:2465-2469` per the doc comment cross-referenced
in the earlier `buzz-pubsub`-focused analysis batch).

---

#### 3. Does either file link `buzz-conformance` as a dependency? — No

Checked directly against `crates/buzz-test-client/Cargo.toml`:

```toml
[dependencies]
anyhow, buzz-core, buzz-ws-client, nostr, tokio, tokio-tungstenite,
futures-util, serde, serde_json, tracing, tracing-subscriber, thiserror,
uuid, url, rustls

[dev-dependencies]
tracing-subscriber, uuid, futures-util, reqwest, base64, hex, rand, sha2,
sqlx, chrono, s3 (rust-s3), buzz-sdk
```

`buzz-conformance` appears in **neither** list. This is independently conclusive (a crate cannot
be `use`d if it isn't a dependency) and confirms the grep-based finding in the Data Model and API
Surface docs. Of the declared dev-dependencies, this agent's two owned files use only: `reqwest`
(both files, extensively), `base64`+`sha2` (`conformance_multitenant.rs`'s hand-rolled NIP-98
header builder, `:870-880`), `uuid` (both files). `sqlx`, `chrono`, `s3`, `rand`, and `buzz-sdk`
are declared at the crate level but — per direct grep against both files — **not used by either**;
those dependencies exist to serve sibling test files this agent does not own (`e2e_media*.rs`,
`e2e_git.rs`, etc.), which is expected given `Cargo.toml`'s dependencies are crate-wide, not
per-test-binary.

---

#### 4. What would have to exist for `conformance_multitenant.rs`'s live tests to actually run, and whether it does

Searched for any two-host / multi-community test-fixture provisioning:

- `grep` for `a.localhost`, `b.localhost`, `RELAY_URL_A`, `RELAY_URL_B`, `two-host`, `multi-tenant`
  across `docker-compose*.yml` and `scripts/**`: **zero matches** outside the test file itself.
- `docs/multi-tenant-conformance.md` (the design document this test file's own doc comment says it
  "mirrors ... one row per module," `conformance_multitenant.rs:8-9`) is a **checklist of what the
  multi-tenant rewrite must satisfy**, not a description of an existing runnable fixture — its
  final section, "Migration gates," frames the whole document as pre-implementation planning
  ("Before multi-tenant mode is admitted, the implementation must have automated gates...").
- `scripts/start-relay-for-tests.sh` — the script CI uses to boot a relay for `e2e_nostr_interop.rs`
  and friends — seeds exactly **one** community row (`localhost:3000`) and has no parameter for a
  second host (`scripts/start-relay-for-tests.sh:104-127`).
- `crates/buzz-relay/src/tenant.rs` confirms `bind_community`/`relay_url_authority` exist as the
  production host-resolution mechanism the test file's doc comments describe, but this is relay
  *source*, not a test-time fixture — nothing in `buzz-test-client` or the CI/script layer drives
  it with two hosts.

**Conclusion:** the two-host multi-tenant relay fixture that every live test in
`conformance_multitenant.rs` requires does not appear to be provisioned anywhere in this
repository as of the commit reviewed. This is consistent with the earlier `buzz-conformance`-batch
finding that multi-tenancy itself is a forward-looking rewrite the codebase is preparing for, not
a currently-deployed capability — `docs/multi-tenant-conformance.md`'s own framing ("adding
first-class communities without changing the observed behavior of a single-community Buzz
deployment") confirms single-community is still the only shipped mode. See the Debt doc for the
consequence this has for CI executability.

---

#### 5. Cross-references to this file from elsewhere in the repository

Grepping the already-completed reverse-engineering docs from other batches turns up several
citations *of* `conformance_multitenant.rs` as supporting evidence for claims made in other
modules' analysis (all pre-existing content from other agents' batches, cited here only to
establish this file's role as a cross-cutting reference point, not re-analyzed):

- `buzz-pubsub`-adjacent integrations doc cites `conformance_multitenant.rs:2371` and `:2484` for
  the claim that presence reads come from Redis with no DB fallback.
- `buzz-audit`-adjacent integrations doc cites `conformance_multitenant.rs:2665-2710` for the claim
  that audit is deliberately not black-box wire-testable.
- `buzz-workflow`-adjacent debt doc cites `conformance_multitenant.rs:1863-1946` for the approval-
  gate coverage gap (WF-08).
- The `buzz-conformance`-batch debt doc cites this file by absence: "`crates/buzz-test-client/tests/
  conformance_multitenant.rs` never references `buzz_conformance`" — the same finding this batch's
  Data Model and API Surface docs independently re-verify against the current source.

This confirms the file is treated elsewhere in the repository's own reverse-engineering corpus as
an authoritative black-box source for multi-tenant behavioral claims, despite (per this batch's
own findings) most of its rows being unimplemented stubs and none of its rows being runnable
without external, unprovisioned infrastructure.
