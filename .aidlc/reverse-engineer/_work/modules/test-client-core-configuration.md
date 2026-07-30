## Module: buzz-test-client — core protocol, status & team E2E (`crates/buzz-test-client`)
### Aspect: Configuration

#### Environment variables consumed by this group's eight files

| Variable | File(s) | Default if unset | Purpose |
|---|---|---|---|
| `RELAY_URL` | `e2e_relay.rs:31`, `e2e_user_status.rs:29`, `e2e_team.rs:29` | `ws://localhost:3000` | Target relay WebSocket URL for the harness's `connect`/`connect_unauthenticated` calls |
| `DATABASE_URL` | `e2e_relay.rs:75` | `postgres://buzz:buzz_dev@localhost:5432/buzz` | Direct Postgres connection for test fixture seeding (`communities`, `relay_members` — see Integrations) |
| `BUZZ_TEST_OWNER_PRIVATE_KEY` | `e2e_relay.rs:47` | `Keys::generate()` (ephemeral) | Identity used by `test_owner_keys()` for the invite-mint tests; letting this default to an ephemeral key means each test run mints/seeds a fresh, disposable "owner" unless the operator deliberately pins one |
| `BUZZ_PRIVATE_KEY` | `main.rs:50` | `Keys::generate()` (ephemeral) | Identity for the manual `buzz-test-cli` tool |
| `RUST_LOG` | `main.rs:37` | `buzz_test_client=debug` | Tracing filter for `buzz-test-cli` |
| `BUZZ_RELAY_URL` | `wamp_bench.rs:35`, `mention.rs:20` | `ws://localhost:3000` | Target relay for the two bin targets (note: **not** `RELAY_URL` — a different variable name than the one the test files use; see Debt) |
| `BENCH_PRIVATE_KEY` | `wamp_bench.rs:36-39` | none — **hard failure via `anyhow::bail!`** if unset | Shared channel-member identity for the load generator; required because `wamp-bench` sends real writes into a real channel and needs pre-existing membership |

`mention.rs` reads no identity-related env var at all — it always generates a fresh keypair (see Features/Conventions).

Neither `e2e_relay.rs` nor any file in this group reads `BUZZ_AUTH_TAG`, despite `AGENTS.md`'s statement that "Auth env vars (`BUZZ_RELAY_URL`, `BUZZ_PRIVATE_KEY`, `BUZZ_AUTH_TAG`) are auto-injected by the ACP harness into managed agent subprocesses" — that auto-injection is specific to `buzz-cli`/`buzz-acp` subprocess launches, not this test crate, so its absence here is expected, not a gap.

#### Two-host live-relay bring-up (`nip42_host_binding_live.rs`)

This file's module doc comment (`:6-19`) is a self-contained runbook, not just a code comment — it specifies exact SQL to seed two communities and the exact relay-process env vars needed to run a second co-located instance without port/feature collisions:

```sql
INSERT INTO communities (id, host) VALUES
  ('11111111-1111-4111-8111-111111111111', 'a.localhost:3100'),
  ('22222222-2222-4222-8222-222222222222', 'b.localhost:3100');
```

Plus, for the relay process itself: `BUZZ_HEALTH_PORT=8180`, `BUZZ_METRICS_PORT=9202`, `BUZZ_RECONCILE_CHANNELS=false`, `BUZZ_GIT_CONFORMANCE_PROBE=false` — the last two specifically to suppress background reconciliation/probe work that isn't relevant to (and would only slow down) this narrow host-binding check. The two hardcoded hostnames (`HOST_A = "ws://a.localhost:3100"`, `HOST_B = "ws://b.localhost:3100"`, `:20-21`) are not overridable via any env var in this file — unlike every other file in this group, there is no `relay_url()`-style indirection, so running this file against a differently-hostnamed two-tenant setup requires editing the constants directly.

The required invocation, `cargo test -p buzz-test-client --test nip42_host_binding_live -- --ignored --test-threads=1` (stated in both the module doc comment `:18-19` and repeated in each test's context), pins `--test-threads=1` specifically — this matters because the four tests share the same two live relay hosts and each opens a real WebSocket, so parallel execution could interleave connections and challenges across tests in ways the single-connection-per-test design doesn't account for. No other file in this group needs (or specifies) single-threaded execution.

#### Relay-side configuration this group's tests implicitly depend on

None of this group's tests set relay-side env vars themselves (they're pure clients), but several tests only pass under specific relay configuration postures, and the group doesn't flag this dependency inline:

| Relay config | Default | This group's dependency |
|---|---|---|
| `BUZZ_REQUIRE_AUTH_TOKEN` | `false` (`config.rs:475-477`) | Most of `e2e_relay.rs`'s `POST /events` calls use the `X-Pubkey` dev-mode header, which the relay only honors when this is `false` — see Security. Local dev's `.env.example` doesn't set it, but the production `deploy/compose/.env.example` sets it `true` |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | `false` (`config.rs:483-485`) | `test_invite_mint_and_claim_admits_new_pubkey` and friends test the *closed-relay* invite flow; they work regardless of this flag because they explicitly seed `relay_members` rows via direct SQL rather than depending on the open/closed posture, but the flag's practical meaning (closed relay requires membership; open relay doesn't gate on it) is exercised only implicitly |
| `MAX_SUBSCRIPTIONS` (relay-internal `const`, not an env var) | `1024` (`handlers/req.rs:26`) | `test_subscription_limit_enforced` hardcodes this value directly in its loop bound (`for i in 0..1024`, `:706`) — if the relay's constant ever changes, this test's loop bound must be manually updated in lockstep; there's no shared constant or env var either side reads to stay in sync (see Debt) |
| `DEFAULT_MAX_FRAME_BYTES` (relay-internal `const`) | `512 * 1024` (`config.rs:14`) | `test_large_event_frame_below_configured_limit_is_accepted` hardcodes both the old (65,536) and an assumed new-cap ceiling (512 KiB) directly in its assertions (`:340-348`) rather than reading the relay's advertised value from anywhere |

#### `Cargo.toml` — bin target declarations

Only one of this crate's two extra binaries is declared via `[[bin]]` in `Cargo.toml:34-36` (`name = "buzz-test-cli"`, `path = "src/main.rs"`). `wamp_bench.rs` and `mention.rs` live under `src/bin/` and are picked up by Cargo's default binary-discovery convention (any `.rs` file directly under `src/bin/` becomes an implicitly-named binary matching the filename minus extension) — so `wamp-bench` and `mention` are real, buildable targets (`cargo build --release -p buzz-test-client` produces `target/release/wamp-bench` and `target/release/mention`) without needing an explicit `[[bin]]` entry. This is valid, idiomatic Cargo, but it means the crate's binary surface isn't fully visible from `Cargo.toml` alone — a reader checking only the manifest would miss two of the three shipped binaries.

#### No CI wiring for any test in this group

`justfile`'s `test`, `test-unit`, and `test-integration` recipes (`justfile:271-300`) and `scripts/run-tests.sh` were checked directly: none reference `buzz-test-client`, `--ignored`, or any of this group's four test file names. `just ci` (`justfile:1`) composes `check`, `test-unit`, `desktop-test`, `desktop-build`, `desktop-tauri-check`, `desktop-tauri-test`, `web-build`, `mobile-test` — none of which touch this crate's `#[ignore]`d suites either. `.github/workflows/*.yml` was grepped for `buzz-test-client`, `--ignored`, and all four file basenames — zero matches. This means every test in this group (all 50 of them: 38+5+3+4) runs **only** when a human explicitly types the documented `cargo test -p buzz-test-client -- --ignored` (or the file-scoped variant) after manually starting a relay per `TESTING.md`'s runbook. There is no automated regression signal for any of this group's tests on any PR — a behavior change that silently breaks, say, the membership-notification `#p`-filter gate would only be caught by someone remembering to run this suite by hand. See Debt for the operational-risk framing of this gap.
