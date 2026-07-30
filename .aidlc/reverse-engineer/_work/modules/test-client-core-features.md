## Module: buzz-test-client — core protocol, status & team E2E (`crates/buzz-test-client`)
### Aspect: Features

This group delivers three kinds of capability: a reusable test harness other crate consumers build on, two operator/dev tools, and E2E proof for four relay-level features (core NIP-01/NIP-42 protocol, NIP-42 host-binding, NIP-38 status, NIP-AP teams).

#### Shared test harness (`src/lib.rs`)

`BuzzTestClient` is the one reusable feature this group ships to the rest of the crate (and, per `Cargo.toml`, is a `[dependencies]`-level export usable by any crate that lists `buzz-test-client` as a dev-dependency — currently used only within this crate's own `tests/` directory and bin targets). It packages the otherwise-tedious NIP-01/NIP-42 handshake, event signing, subscription bookkeeping, and EOSE-draining into eleven methods (see API Surface), so every downstream E2E file — in this group and beyond — writes tests in terms of "connect, send, subscribe, collect" rather than hand-rolling WebSocket frame parsing. This is the single biggest leverage point in the crate: all 219 `#[tokio::test]` functions across the 16 test files in `crates/buzz-test-client/tests/` depend on it (directly or via `RelayMessage`/`TestClientError`), and this group's four files alone account for 50 of those 219.

The harness deliberately stays thin — it has no retry logic, no connection pooling, no fixture management, and (per Debt) doesn't even track the outstanding-subscription set it half-exposes through `subscribe`/`close_subscription`. That minimalism is itself a feature choice: it keeps the harness's own behavior legible enough that a test failure is almost always attributable to the relay under test, not to hidden harness-side retry/backoff logic masking a race.

#### Manual testing CLI (`buzz-test-cli`, `src/main.rs`)

A tiny, dependency-light send/subscribe tool for a human to point at a running relay without writing a Rust test. Two modes only: fire-and-forget send (`--send`), and a blocking subscribe-and-print loop (`--subscribe`) that runs until Ctrl+C, a `CLOSED` frame, or an unrecoverable error. It's positioned as the fast manual-verification complement to the `#[ignore]`d automated suites — useful for a developer who wants to eyeball live event delivery while iterating on relay code, without paying the cost of writing/running a full test. It is not wired into any CI job or `justfile` recipe (see Configuration/Debt) — its entire audience is a developer running it by hand.

#### Write-amplification benchmark (`wamp-bench`, `src/bin/wamp_bench.rs`)

A purpose-built load generator answering one specific operational question: at a sustained target write rate against one channel, what does per-event OK-latency look like, and does the relay start rejecting writes under load? It opens N synchronous connections (one shared identity), paces each on its own `tokio::interval` so the *aggregate* rate matches the requested QPS, and reports p50/p95/p99/max latency plus accept/reject counts as a single JSON blob — the shape an operator would pipe into a spreadsheet or a regression-tracking script rather than read by eye. The tool's own doc comment (`wamp_bench.rs:1-12`) frames it explicitly as measuring "relay write-amplification" — i.e., this is infrastructure for capacity/regression testing of the ingest pipeline's throughput characteristics, not a correctness test. No test file in this group (or, per a repo-wide grep, anywhere else) invokes it programmatically — it's a standalone operator tool, run by hand or from an as-yet-unwritten CI perf job.

#### One-shot @mention sender (`mention`, `src/bin/mention.rs`)

The smallest tool in this group: build one kind:9 event carrying both an `h`-tag (channel) and a `p`-tag (target pubkey), sign it with a fresh throwaway identity, and print whether the relay accepted it. Its natural use case is manually triggering an `buzz-acp`-observed `@mention` event against a channel to watch an agent pick it up, without needing a full desktop/CLI session. Because it always generates a new keypair, it can't be used to test "the same sender mentions multiple times" scenarios — each invocation is a stranger to the relay from the channel's perspective (subject to whatever open/private membership rules that channel enforces).

#### Core relay protocol E2E (`e2e_relay.rs`)

This is the group's primary correctness-proof feature: 38 tests giving black-box coverage of the WebSocket/HTTP surface documented in `ARCHITECTURE.md` §§2-5 — connection lifecycle, the EVENT pipeline's auth/pubkey-match/kind-gating steps, the REQ/subscription registry's filter and limit behavior, NIP-11 self-description accuracy, NIP-29 group administration (creation, discovery events, membership policy, private-channel role gating, archive/unarchive), a NIP-05/kind:0 profile-sync side effect, relay-signed membership notifications, and relay membership invite tokens. Functionally, this file is the closest thing in the repository to an executable specification of "what does a compliant NIP-01/NIP-29 client need to know about this relay's behavior" — a new client implementation (or a new relay implementation aiming for interop) could derive most of its integration-test plan directly from this file's test names and assertions.

#### NIP-42 host-binding live proof (`nip42_host_binding_live.rs`)

A narrow, deliberately manual-only (all four tests `#[ignore]`d, requiring a live two-host relay and `--test-threads=1`) feature: proving that a specific multi-tenant security fix holds under real network conditions, not just in a relay-internal unit test. This exists because the underlying unit test (`bridge.rs:2722-2726`) can only prove the *helper function* computes the right expected URL — it can't prove the WebSocket handshake, tenant resolution from `Host` header, and AUTH verification pipeline actually wire that helper's output into the real accept/reject decision end-to-end across two genuinely different hostnames. This file is the closing link in that chain.

#### User status E2E (`e2e_user_status.rs`)

Five tests giving the NIP-38 (kind:30315) feature its own dedicated black-box proof, independent of the channel/message machinery: acceptance, retrievability, NIP-33 replacement (newer wins), multiple independent `d`-tag slots coexisting under one pubkey, and stale-write rejection (older `created_at` can't overwrite newer). This is a small, self-contained feature — user status has no channel scoping, no membership gate, and no encryption; the five tests together constitute essentially the entire behavioral contract for the kind.

#### Team E2E (`e2e_team.rs`)

Three tests proving the NIP-AP team-event feature (kind:30176) round-trips correctly through the *generic* NIP-33 parameterized-replaceable path — publish-and-query, replacement (newer wins), and `a`-tag tombstone deletion. The file's own header comment is explicit that its purpose is partly comparative: proving the team kind is validated by the same generic mechanism as any other parameterized-replaceable kind, in contrast to the stricter, kind-specific slug-grammar envelope personas get. This is a deliberately minimal feature-proof file (3 tests, 212 lines) — commensurate with how little kind-specific logic actually exists for teams on the relay side (a `d`-tag length check and nothing more).
