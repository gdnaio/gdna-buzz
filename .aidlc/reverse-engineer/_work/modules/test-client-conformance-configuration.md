## Module: buzz-test-client — Nostr interop & multi-tenant conformance E2E (`crates/buzz-test-client/tests`)
### Aspect: Configuration

---

#### 1. Every environment variable read across the two files

| Env var | File | Read site | Default (if unset) | Documented in `TESTING.md`? | Documented in `.env.example`? |
|---|---|---|---|---|---|
| `RELAY_URL` | `e2e_nostr_interop.rs` | `relay_url()`, `:28` | `ws://localhost:3000` | **Yes** — `TESTING.md:275` (config reference table), `:106` (port-override callout) | **Yes** — `.env.example:49` |
| `RELAY_URL_A` | `conformance_multitenant.rs` | `url_a()`, `:46` | `http://a.localhost:3000` | **No** | **No** |
| `RELAY_URL_B` | `conformance_multitenant.rs` | `url_b()`, `:51` | `http://b.localhost:3000` | **No** | **No** |
| `RELAY_URL_UNKNOWN` | `conformance_multitenant.rs` | `url_unknown()`, `:59-60` | `http://unknown.localhost:3000` | **No** | **No** |

That is the complete set — a repo-wide-style search restricted to these two files
(`grep -n 'std::env::var' crates/buzz-test-client/tests/e2e_nostr_interop.rs
crates/buzz-test-client/tests/conformance_multitenant.rs`) returns exactly these four call sites
and no others. Neither file reads any CLI flag (`std::env::args`) — both are pure `cargo test`
binaries with no custom flag parsing; the only invocation-time controls available are the
standard `cargo test [-- --ignored]` mechanism (see below) and these four env vars.

##### `RELAY_URL_A`/`RELAY_URL_B`/`RELAY_URL_UNKNOWN` — documentation gap, verified

Searched for these three variable names across every markdown file at the repo root and in
`docs/`:

```
grep -rn 'RELAY_URL_A\|RELAY_URL_B\|RELAY_URL_UNKNOWN' TESTING.md CONTRIBUTING.md ARCHITECTURE.md \
  AGENTS.md docs/multi-tenant-conformance.md .env.example
```

Zero matches in all six files. The **only** place in the entire repository where these three
variable names appear is inside `conformance_multitenant.rs` itself — in its own top-of-file
usage example (`:20-26`):

```text
RELAY_URL_A=http://a.localhost:3000 \
RELAY_URL_B=http://b.localhost:3000 \
cargo test -p buzz-test-client --test conformance_multitenant -- --ignored
```

This is a self-contained example with no cross-reference from `TESTING.md`'s "Automated Tests"
section (which documents exactly one `#[ignore]`d-suite invocation pattern: `cargo test -p
buzz-test-client -- --ignored`, `TESTING.md:12-13`) or from anywhere else a contributor or CI
author would look for how to run this specific file. Contrast with `RELAY_URL`, which is
documented in a dedicated configuration-reference table with its default and semantic note ("no
`BUZZ_` prefix," `TESTING.md:275`).

##### `RELAY_URL_UNKNOWN` — parsed and read exactly once, only inside a stub-adjacent live test

Unlike `RELAY_URL_A`/`RELAY_URL_B`, which each feed roughly a dozen live tests via `url_a()`/
`url_b()` helper reuse across every non-stub module, `RELAY_URL_UNKNOWN` is consumed by exactly one
test function: `row_zero_host_binding::unmapped_host_fails_closed_generically`
(`conformance_multitenant.rs:116-232`, which calls `url_unknown()` at `:130`, `:180`, `:225`) and
by `nip11_relay_info::nip11_is_not_a_cross_community_enumeration_oracle` (`:519`). It is not
"parsed but never read" — both call sites are inside live (non-stub) tests — but its total
blast radius in the file is narrow relative to its sibling vars.

---

#### 2. Indirect configuration dependency: the relay's own env vars, assumed pre-configured

Neither file sets or reads any relay-side environment variable directly, but both files' tests are
only meaningful if the relay-under-test was started with specific configuration. Cross-referencing
`TESTING.md`'s configuration-reference table (`:271-282`) against what each file's fixtures
require:

| Relay env var | Default | Why each file's tests need a specific value | Which file |
|---|---|---|---|
| `BUZZ_REQUIRE_AUTH_TOKEN` | `false` | both files use the dev-mode `X-Pubkey` REST header fallback exclusively (never a minted NIP-98/API token for ordinary posts) — this only works when `false` | both |
| `RELAY_URL` (relay-side, distinct from the test client's own `RELAY_URL` env var of the same name) | `ws://localhost:3000` | advertised in NIP-42 AUTH challenges and NIP-98 `u`-tag expectations; `e2e_nostr_interop.rs`'s NIP-98-adjacent behavior is indirect (it never builds NIP-98 headers itself), but `conformance_multitenant.rs`'s `api_tokens_nip98_replay` module hand-builds `u` tags against `url_a()`/`url_b()` and depends on the relay computing a matching per-tenant expected URL (per that module's own doc comment, `:815-828`) | `conformance_multitenant.rs` |
| (no documented var for a second community host mapping) | n/a | `conformance_multitenant.rs`'s entire live-test set requires the relay to have been started against a **database already seeded with two `communities` rows** (`a.localhost:3000`, `b.localhost:3000`) — there is no relay env var, documented or otherwise, that this agent could find which provisions this. See Integrations doc for the full search. This is the most consequential configuration gap in this module: the file's tests are unconditionally non-functional without a manual or undocumented seeding step. | `conformance_multitenant.rs` |

---

#### 3. Configuration read but arguably never meaningfully exercised

`RELAY_URL_UNKNOWN`'s default value, `http://unknown.localhost:3000`, is architecturally
significant: per the file's own doc comment (`:53-57`), `*.localhost` is relied upon to resolve to
`127.0.0.1` via OS/browser convention, so the same relay process addressed by `url_a()`/`url_b()`
is also addressed by this URL, differing only in the `Host` header. If the two-host fixture this
file depends on has never been provisioned (see §1/Integrations), then this variable's default is
never validated against a live unmapped-host scenario in practice — it is "configured" only in the
sense that the fallback string exists and would be syntactically usable if the rest of the fixture
existed.

---

#### 4. Comparison: how thoroughly each file's configuration surface is documented

| | `e2e_nostr_interop.rs` | `conformance_multitenant.rs` |
|---|---|---|
| Env vars read | 1 (`RELAY_URL`) | 3 (`RELAY_URL_A`, `RELAY_URL_B`, `RELAY_URL_UNKNOWN`) |
| Documented in `TESTING.md` | Yes, in the shared config table | No |
| Documented in `.env.example` | Yes | No |
| Has its own in-file usage example | Yes (`:15-19`) | Yes (`:20-26`) |
| Invocation pattern documented in `TESTING.md`'s "Automated Tests" section | Implicitly, via the generic `cargo test -p buzz-test-client -- --ignored` example (`:12-13`) — this pattern *would* include `e2e_nostr_interop.rs`'s tests since they carry no extra required env var beyond the already-documented `RELAY_URL` | **No** — the generic pattern in `TESTING.md` would run `conformance_multitenant.rs`'s tests too (they're `--ignored` under the same crate), but every one of them would immediately fail (`RELAY_URL_A`/`_B` falling back to `a.localhost`/`b.localhost`, which almost certainly aren't bound to a two-community relay in the reader's local setup) — `TESTING.md` gives no warning about this |

This asymmetry is the clearest configuration-level evidence, independent of the CI-execution
question addressed in the Debt doc, that `conformance_multitenant.rs` was not brought up to the
same operational-readiness bar as `e2e_nostr_interop.rs` at the time these files were authored.
