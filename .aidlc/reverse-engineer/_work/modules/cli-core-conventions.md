## Module: buzz-cli — dispatch, relay client & validation (`crates/buzz-cli/src`)
### Aspect: Conventions

#### Error handling

`thiserror` enum + `?` propagation throughout; every fallible function returns
`Result<_, CliError>` (`error.rs:1-45`). Conversion from `reqwest::Error` is
`#[from]` (`error.rs:15-16`); everything else is mapped explicitly at the call
site, e.g. `Tag::parse(...).map_err(|e| CliError::Other(format!("tag error: {e}")))`
(`client.rs:93-95`). SDK errors get a single mapping helper so exit codes stay
consistent: `sdk_err` sends `InvalidInput` → `Usage` (1) and everything else →
`Other` (4) (`validate.rs:155-160`), and it is used 20 times in `commands/`
(`grep -rn 'sdk_err' commands/ | wc -l` → 20).

Error text is deliberately machine-shaped: one JSON object per failure with
`error` (category), `message`, `retryable` (`error.rs:127-136`). The
`fmt_reqwest_error` helper walks the full `source()` chain and de-duplicates
substrings so network failures carry the real cause rather than
`error sending request` (`error.rs:49-61`; test
`network_display_includes_detail_beyond_prefix`, `error.rs:225-235`).

#### Output discipline

Exactly two output statements exist in the whole group:
`println!` in `print_create_response` (`client.rs:1402`) and `eprintln!` in
`print_error` (`error.rs:135`) — verified by
`grep -n 'println!\|eprintln!\|print!(' lib.rs client.rs validate.rs error.rs agent_management.rs main.rs`.
Payload goes to stdout, diagnostics to stderr, both single-line JSON. Human
prose reaches the terminal only through clap's own help rendering
(`lib.rs:50`).

#### Logging

None. `grep -rn 'tracing\|log::' lib.rs client.rs validate.rs error.rs agent_management.rs main.rs`
returns zero matches, and no logging crate is declared in `Cargo.toml`. The
upside is that no secret can reach a log sink from this layer; the downside is
that retries, endpoint fallbacks and cursor pagination are invisible — a caller
sees a 60-second hang with no way to observe the three attempts and two sleeps.

#### `unsafe`, lint attributes, panics

- **No `unsafe`**: `grep -n 'unsafe' lib.rs client.rs validate.rs error.rs agent_management.rs main.rs`
  → zero matches, satisfying the AGENTS.md rule.
- Only two `#[allow(...)]` attributes in the group, both `dead_code` on unused
  public methods (`client.rs:567`, `client.rs:802`) — used in place of deletion
  or a tracking marker.
- `AGENTS.md` forbids new `unwrap()`/`expect()` on production paths. Remaining
  violations, all in `client.rs`:
  `advance_query_cursor`'s `.expect("a full query page always has a last event")`
  (`client.rs:504-506`) and `extract_p_tags`'s `t.as_array().unwrap()`
  (`client.rs:1379`). Both are locally justified (the caller only calls
  `advance_query_cursor` on a full page; the filter already matched `as_array`),
  but both are reachable panics on malformed relay data rather than `?`.
- Three `unreachable!` sites: loop-exhaustion guards in `with_retry_body`
  (`client.rs:679`) and `submit_moderation_event` (`client.rs:1018`), and the
  `Cmd::Pack(_) => unreachable!("handled above")` arm (`lib.rs:1791`) that pairs
  with the early return at `lib.rs:1737-1743`.
- `lib.rs:1948-1957` uses `.expect("repos command")` / `.expect("repos protect command")`
  but inside `#[cfg(test)]`, which is idiomatic.

#### Naming

Command enums are `<Group>Cmd` (`AgentsCmd`, `MessagesCmd`, …, `lib.rs:260-1731`);
value enums are noun-shaped with explicit `#[value(name = "…")]` kebab-case wire
names (`lib.rs:101-172`); relay-facing verbs are `submit_*`/`query_*`/`get_*`;
predicates read as questions (`is_moderation_kind`, `is_safe_media_ext`,
`resp_was_success`, `is_stored_event_exhaustion_ambiguous`). Validators are
`validate_*` when they return `()` and `parse_*` when they return the value — a
distinction called out explicitly in the doc comment on `parse_uuid`
(`validate.rs:15-18`). One naming defect: `validate_hex64`'s doc says
"64-character lowercase hex string" (`validate.rs:28`) but the body accepts
uppercase via `is_ascii_hexdigit` (`validate.rs:30`), while the media path
checker in the same crate enforces lowercase explicitly
(`is_lower_hex_sha256`, `client.rs:260-262`).

#### Doc-comment discipline

Mixed. `client.rs` is exemplary: near-every item carries a `///`, and the long
comments on retry/idempotency (`client.rs:1024-1039`), `sign_event_unchecked`
(`client.rs:729-742`) and the rustls install (`lib.rs:30-38`) explain *why*, not
what. `lib.rs`'s clap surface is documented for users (every flag has a `///`
that becomes help text, plus 14 `after_help` example blocks), but the public Rust
types are not: `ChannelType`, `ChannelVisibility`, `PresenceStatus`, `EmojiScope`
and 11 of the `*Cmd` enums have no doc comment (`lib.rs:101`, `:118`, `:135`,
`:145`, `:260`, `:348`, `:502`, `:679`, `:698`, `:729`, `:771`, `:802`, `:844`,
`:923`, `:939`). `agent_management.rs` has a module-level `//!` (`:1`) but no
doc comment on any of its five public items. `CliError` variants are documented
individually while the enum itself is not (`error.rs:4`).

#### File-size discipline

`client.rs` is 2,477 lines and `lib.rs` 2,035 — both far past the 1,000-line
ceiling the repo enforces for mobile Dart (`justfile:617`,
`mobile/scripts/check-file-sizes.mjs`). There is no equivalent guard for Rust:
`grep -rn 'check-file-sizes' justfile` matches only the mobile recipe. Mitigating
factor for `client.rs`: production code ends at line 1,433 and lines 1,434-2,477
(42%) are test modules. `lib.rs` has no such excuse — it is a single flat clap
declaration with no submodule split, and `enum Cmd`'s dispatch match plus 22
subcommand enums live in one file. Largest single function bodies:
`submit_moderation_event` (`client.rs:873-1022`, ~150 lines with 5 distinct
retry branches) and `upload_file` (`client.rs:1100-1227`, ~128 lines including
the legacy-endpoint duplicate block).

#### Test organization

95 tests in the group: `validate.rs` 42, `client.rs` 29 `#[test]` + 14
`#[tokio::test]`, `error.rs` 7, `lib.rs` 4, `agent_management.rs` 3, `main.rs` 0
(counts via `grep -c '#\[test\]'` / `'#\[tokio::test\]'` per file). Convention is
inline `#[cfg(test)] mod tests` at the bottom of each file, with `client.rs`
splitting into four purpose-named modules — `media_download_tests`
(`client.rs:387`), `retry_tests` (`:1434`), `retry_policy_tests` (`:1582`),
`tests` (`:2297`) — and using `// ---- name ----` banner comments to group
assertions (`client.rs:1440`, `:1479`, `:1508`, `:1528`, `:1545`, `:1554`).
Integration-style tests spin a real axum server or raw `TcpListener` and assert
observable behavior (attempt counts, elapsed time, identical request bytes)
rather than internals — see `test_server` (`client.rs:1603-1636`) and
`stored_event_body_loss_is_retried_with_same_event_bytes` (`client.rs:2038-2114`).

Two convention breaks worth flagging:

1. **Tests that re-implement the production rule.** `error.rs`'s
   `json_error_includes_retryable_field_for_network` (`error.rs:197-210`) and
   `json_error_retryable_false_for_usage` (`error.rs:213-221`) rebuild
   `print_error`'s JSON object inline with `serde_json::json!` instead of calling
   `print_error`, so the actual production serializer and its category strings
   (`error.rs:109-126`) are never executed by a test.
   `grep -c 'print_error' <(awk 'NR>=137' error.rs)` finds one hit and it is a
   banner comment.
2. **Production-file helpers that exist only for tests.**
   `parse_retry_in_secs` (`client.rs:172-186`) and `percent_encode`
   (`validate.rs:75-99`) are `#[cfg(test)]`-gated functions in production files,
   each with its own test suite (6 tests at `client.rs:1444-1477`; 5 at
   `validate.rs:277-306`) — coverage on code that never runs in production.

#### Formatting and toolchain conventions

`rustfmt` defaults appear respected (100-col wrapping, trailing commas). Test
attributes use `#[tokio::test]` without flavour arguments, relying on the
`macros` + `rt-multi-thread` features declared at `Cargo.toml:25`. Cargo.toml
groups dependencies with a one-line rationale comment above each
(`Cargo.toml:18-86`) — a good convention undermined by two comments that have
gone stale (see the Integrations aspect).
