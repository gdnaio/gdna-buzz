## Module: buzz-test-client — core protocol, status & team E2E (`crates/buzz-test-client`)
### Aspect: Debt

#### `ARCHITECTURE.md`'s "buzz-test-client" section is substantially stale

`ARCHITECTURE.md`'s dedicated subsection (its "Test coverage" table, around line 700) claims:

```
| tests/e2e_relay.rs         | 27  | ... |
| tests/e2e_media.rs         | 7   | ... |
| tests/e2e_media_extended.rs | 18 | ... |
| tests/e2e_nostr_interop.rs | 15  | ... |
...
Total: 134 e2e tests.
```

None of these numbers match the current repository:

- `e2e_relay.rs` has **38** `#[tokio::test]` functions today (confirmed by direct grep), not 27 — an undercount of 11, roughly 40% higher than documented.
- The crate has **16** test files under `tests/`, not the 4 enumerated in the doc's table — the doc is missing `e2e_event_reminder.rs` (29 tests), `e2e_human_edit_agent_content.rs` (19), `e2e_long_form.rs` (8), `e2e_managed_agent.rs` (5), `e2e_media_video.rs` (7), `e2e_mesh_llm.rs` (4), `e2e_persona.rs` (24), `e2e_team.rs` (3), `e2e_user_status.rs` (5), `nip42_host_binding_live.rs` (4), `conformance_multitenant.rs` (18), and `e2e_git.rs` (2) — twelve entire files unlisted.
- The actual total across all 16 files is **219** `#[tokio::test]` functions (directly counted), not 134 — the documented total is off by 85 tests, a 63% undercount.
- The doc's own summary line ("Total: 134 e2e tests") appears to be a stale sum of exactly the 4 listed files (27+7+18+15=67, which doesn't even match 134 either) — so the number wasn't even internally consistent with the table it followed at the time it was last accurate, or the table itself has since been edited without updating either the per-row counts or the total.

This group's own four files alone (`e2e_relay.rs` 38, `e2e_user_status.rs` 5, `e2e_team.rs` 3, `nip42_host_binding_live.rs` 4 = 50 tests) are more than a third of the doc's claimed *crate-wide* total by themselves. Anyone using `ARCHITECTURE.md` to scope "how much E2E coverage exists" for this crate will significantly underestimate it, and anyone using it to find *which* files exist will miss three-quarters of them, including both of this group's newer additions (`e2e_team.rs`, `e2e_user_status.rs`, `nip42_host_binding_live.rs` — all three postdate the doc's table, along with eight other files outside this group's scope).

#### `ARCHITECTURE.md` misattributes `BuzzTestClient`'s buffering to the wrong struct

The same section states: *"`BuzzTestClient` wraps a WebSocket connection with a `VecDeque<RelayMessage>` buffer for message interleaving."* This is incorrect as stated. `BuzzTestClient` (`src/lib.rs:87-89`) has exactly one field, `inner: NostrWsConnection`, and holds no buffer itself. The `VecDeque<RelayMessage>` actually lives inside `NostrWsConnection` (`crates/buzz-ws-client/src/connection.rs:28`, field `buffer`), one crate down. This is a minor but genuine architectural misstatement — the *behavior* the doc describes (interleaved AUTH/OK/EVENT frames can be buffered and replayed in the right order to the right waiter) is real and correctly attributed in spirit, but the doc names the wrong type as the owner of that behavior. Anyone reading `buzz-test-client`'s source expecting to find a `VecDeque` field on `BuzzTestClient` (e.g. to extend it) will not find one there.

#### `MAX_HISTORICAL_LIMIT` documentation drift (relay-side, but directly relevant to this group's REQ tests)

`ARCHITECTURE.md` states in two places: "Max historical results per filter: 500" (protocol section) and a constants table entry `MAX_HISTORICAL_LIMIT | 500 | Per-filter historical query cap`. The actual relay constant is `const MAX_HISTORICAL_LIMIT: i64 = 2_000;` (`crates/buzz-relay/src/handlers/req.rs:25`) — 4x the documented value. No test in this group (or, per inspection, in the whole `e2e_relay.rs`-style suite) directly asserts the actual historical-result cap by requesting more than 500 events and counting the returned batch, so this drift is invisible to this group's own test suite; it was only caught by direct source inspection during this analysis pass. `MAX_SUBSCRIPTIONS` (1024) and `MAX_FRAME_BYTES` (`DEFAULT_MAX_FRAME_BYTES = 512 * 1024`, i.e. 524,288, not the doc's stated 65,536) show the same kind of staleness — `test_subscription_limit_enforced` and `test_nip11_relay_info` in this group *do* correctly assert the current 1024 value against both the enforcement constant and the NIP-11 payload, so that specific number is not drifted in the tests, only in `ARCHITECTURE.md`'s "Max frame size: 65,536 bytes" line, which contradicts the doc's own later constants table entry claiming `65,536` for `MAX_FRAME_BYTES` — both of which are stale against the actual `512 * 1024` default. `test_large_event_frame_below_configured_limit_is_accepted` (this group, `e2e_relay.rs:324-364`) is itself a **regression test written specifically because this cap changed** from 64 KiB to 512 KiB — the test's own doc comment states the old cap, meaning the codebase has already outpaced `ARCHITECTURE.md` on this exact fact and nobody updated the doc to match.

#### Loose multi-outcome assertions reduce two tests' regression-catching power

Both discussed in more depth in Security: `test_unauthenticated_rejected` (`e2e_relay.rs:756-786`) accepts `OK false` / `ConnectionClosed` / `Timeout` as equally valid, when the relay's actual behavior is deterministically `OK false`. `test_user_status_stale_write_rejected` (`e2e_user_status.rs:214-272`) explicitly declines to assert on the stale write's own `ok.accepted` value (comment at `:255-256`: "Stale write may be rejected or accepted-as-duplicate — either way..."), asserting only on the subsequent query result. Both patterns are defensible engineering choices (the tests are deliberately tolerant of multiple valid relay implementations), but both also mean a regression to a *worse* behavior than the relay currently exhibits (e.g., a genuine hang instead of a fast `OK false`) would not be caught.

#### `nip42_host_binding_live.rs` naming breaks this group's `test_`-prefix convention

All 46 test functions in this group's other three files use the `test_<subject>_<outcome>` naming pattern; this file's four functions (`nip42_matching_host_accepted_a`, `nip42_matching_host_accepted_b`, `nip42_cross_host_rejected_a_relay_tag_on_b_connection`, `nip42_cross_host_rejected_b_relay_tag_on_a_connection`) have no `test_` prefix at all. Functionally harmless (Rust's `#[tokio::test]` doesn't require any particular name), but it's an inconsistency a linter or convention-checker scanning for `fn test_*` wouldn't catch, and it stands out against the other 15 files in the crate.

#### Two different env var names for "which relay to hit" across this group's own files

`e2e_relay.rs`, `e2e_user_status.rs`, and `e2e_team.rs` all read `RELAY_URL`. `wamp_bench.rs` and `mention.rs` both read `BUZZ_RELAY_URL` instead — a different variable name for the identical purpose, within the same crate, in files sitting one directory apart (`tests/` vs `src/bin/`). `nip42_host_binding_live.rs` reads neither (its two target hosts are hardcoded constants). An operator switching between running the automated test suite and the manual bench/mention tools against a non-default relay URL has to remember to set (or export both of) two different variable names, which is an easy source of "why isn't my override taking effect" confusion. `buzz-relay` itself, `buzz-cli`, and the ACP harness all standardize on `RELAY_URL` (relay-side) and `BUZZ_RELAY_URL` (client-side, per `AGENTS.md`'s own "Auth env vars" note) as *intentionally distinct* variables for two different roles — so the bin targets' choice of `BUZZ_RELAY_URL` is arguably the *more* correct one relative to that established client/relay split, and it's the three test files that are the outliers by using `RELAY_URL` (the relay-side name) for what is, from the test client's perspective, a client-side configuration value. Either framing is defensible; the inconsistency itself, within one crate, is the debt.

#### Rustls crypto-provider selection is inconsistent between the two bin targets

Covered in depth in Conventions: `wamp_bench.rs` installs `aws_lc_rs`, `mention.rs` installs `ring`, for the identical underlying requirement, with neither following the rest of the repo's established pattern of pinning the provider via a `Cargo.toml` feature flag (as `buzz-relay`, `buzz-admin`, `buzz-cli`, `buzz-dev-mcp`, and `buzz-acp` all do). `buzz-test-client`'s own `Cargo.toml:25` declares `rustls = "0.23"` with no feature selection, leaving each binary to resolve the ambiguity itself, inconsistently.

#### `authenticate_with_nip_oa` and `send_raw` are unreachable from this group

Confirmed by exhaustive grep across all eight files: neither method is called anywhere in this group. Both are real, used methods elsewhere in the crate (`authenticate_with_nip_oa` by `e2e_human_edit_agent_content.rs`; `send_raw` internally by `subscribe`/`close_subscription`, and directly by `e2e_event_reminder.rs` and `e2e_persona.rs` for hand-built COUNT frames) — so neither is dead code at the crate level, but this group's four files exercise 9 of the harness's 11 public methods, leaving these two entirely unexercised from this vantage point. Not a defect, just a scope note: a reviewer auditing "does this group's test suite cover the harness surface it depends on" should know these two are proven only by sibling groups.

#### No COUNT-door coverage anywhere in this group

Restated from Security because it is also a structural test-suite gap, not just a security concern: zero `["COUNT", ...]` frames are sent and zero `RelayMessage::Count` matches occur across all four test files in this group, despite the task's framing explicitly listing COUNT as part of this group's expected core-protocol surface. The membership-notification `#p`-filter gate (proven for REQ across six tests) has no COUNT-side counterpart test in this group.

#### Hardcoded relay-side constants duplicated into test assertions with no shared source of truth

`test_subscription_limit_enforced`'s `for i in 0..1024` loop bound and `test_nip11_relay_info`'s `Some(1024)` assertion, and `test_large_event_frame_below_configured_limit_is_accepted`'s `65_536`/`512 * 1024` boundary assertions, are all literal numbers typed independently in the test file with no shared constant imported from `buzz-relay` or `buzz-core`. This is consistent with the crate-wide "hardcode kind numbers, don't import `buzz_core::kind`" convention already noted, extended here to relay-internal tuning constants too. It means a future change to `MAX_SUBSCRIPTIONS`, `DEFAULT_MAX_FRAME_BYTES`, or `MAX_HISTORICAL_LIMIT` requires a human to remember to go update these test files by hand — there is no compiler-enforced link between the relay's actual constant and the test's expected value (unlike, say, a `pub const` re-exported from `buzz-relay` that the test could import). Given that `MAX_HISTORICAL_LIMIT` has *already* drifted from `ARCHITECTURE.md`'s documentation without any test catching it, the same silent-drift risk applies to the test-vs-relay relationship for every one of these constants, not just the doc-vs-relay relationship.

#### No CI wiring for any test in this group (operational risk, elaborated from Configuration)

All 50 tests in this group run only on manual invocation. Combined with the `ARCHITECTURE.md` staleness above, this means: (a) nobody is verifying these tests still pass on an ongoing basis, and (b) the one document that might tell a future maintainer "here's what's covered and how much" is itself significantly out of date. The two problems compound — a coverage gap that isn't run automatically and isn't accurately documented is effectively invisible to anyone who doesn't independently read the source, which is exactly the gap this analysis pass exists to close.

#### `buzz-test-cli`'s `--channel default` silently accepts a non-UUID channel identifier

`main.rs`'s `--channel <ID>` flag defaults to the literal string `"default"` (`main.rs:47`, referenced at `:52`) with no format validation anywhere in `parse_args` or the send/subscribe call sites — it's passed straight through to `Tag::parse(["h", channel_id])`, which will happily build a syntactically valid tag out of a non-UUID string. Since the relay's channel model is UUID-keyed, sending to `--channel default` without an explicit override will simply never match any real channel and never error — the tool gives no feedback that the default is almost certainly not what the operator wants. This is a minor UX rough edge in a hand-run manual tool, not a correctness bug, but it's an easy first-run trap.
