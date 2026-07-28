## Module: buzz-cli — channel, message & social commands (`crates/buzz-cli/src/commands`)
### Aspect: Conventions

#### The dominant handler shape

Nine of the ten files follow one template: `pub async fn cmd_<verb>(client, …) ->
Result<(), CliError>` that validates flags, calls a `buzz_sdk::build_*` builder,
signs via `client.sign_event`, submits, and prints
`normalize_write_response(&resp)`. `channels.rs:864-878` (`cmd_set_channel_topic`)
is the canonical 15-line instance; `join`, `leave`, `archive`, `unarchive`,
`delete`, `add-member`, `remove-member`, `set-canvas`, `dms hide`,
`dms add-member`, `reactions add`, `social publish`, `social set-contacts` are all
the same shape. Reads mirror it with a `serde_json::json!` filter,
`client.query`, and a print.

`notes.rs` deliberately departs: it declares its own kind constant
(`notes.rs:38`), builds events with a local pure `build_set_event`
(`notes.rs:418-469`) instead of the SDK, deserializes into typed `nostr::Event`
(`notes.rs:156-159`), and prints plain-text receipts. Its module doc
(`notes.rs:1-31`) explains the verbs and implementation state; no other file in
scope has a module doc except `channel_templates.rs:1-8`.

#### Error handling

`CliError` variants are used consistently by intent: `Usage` for anything the
caller can fix, `NotFound` for absent resources, `Other` for build/parse/internal
failures, `Conflict` for NIP-33 LWW. Builder failures are wrapped with the builder
name in the message, e.g. `"build_forum_post failed: {e}"`
(`messages.rs:543`), `"build_create_channel failed: {e}"` (`channels.rs:311`) —
consistent across ~24 call sites.

Two inconsistencies:

- `validate.rs:167-173` provides `sdk_err`, which maps
  `SdkError::InvalidInput` → `Usage` (exit 1) and everything else → `Other`
  (exit 4). Only `dms.rs:118` uses it. Every other builder call site wraps the SDK
  error in `CliError::Other` unconditionally, so a user-input SDK rejection (empty
  channel name, bad emoji URL, over-long content) exits **4** instead of 1 —
  e.g. `channels.rs:311`, `messages.rs:543`, `emoji.rs:120`, `reactions.rs:21`,
  `social.rs:35`.
- `resolve_channel_id`'s "event not found" and "no h-tag" cases are `Other`
  (`messages.rs:98`, `:112`) where `NotFound`/`Usage` would match the taxonomy;
  `notes.rs:618` uses `NotFound` for the same situation.

Failure-swallowing is used in exactly one place and is documented as intentional:
`resolve_content_mentions` returns an empty vec on any error
(`messages.rs:124-127` doc, `:144-147`, `:155-158`, `:203-211`). Everywhere else
`?` propagates.

`unwrap_or_default()` on `serde_json::from_str` is the module's idiom for
tolerating a malformed relay body (`channels.rs:233`, `:256`, `:269`;
`messages.rs:297`, `:333`, `:374`, `:412`; `dms.rs:17`; `feed.rs:42`;
`reactions.rs:87`), which turns a parse failure into an empty result set rather
than an error. `notes.rs:156-159` and `emoji.rs:83-84` take the opposite line and
surface a parse error.

#### Output discipline

| Rule | Followed by | Exceptions |
|------|------------|-----------|
| stdout is exactly one line of JSON | most commands | `notes set` prints 5 plain-text lines (`notes.rs:571-580`); `notes rm` prints 2 (`:747-748`); `notes get`/`ls` print **pretty-printed** multi-line JSON (`notes.rs:367-372`, `:380-385`); `notes get --content-only` prints raw markdown (`:661-664`); `canvas get` prints raw markdown (`channels.rs:275`) |
| reads are sig-stripped | via `normalize_events` | `social.rs:78`, `:110`, `:123`, `:207` print the relay body verbatim, including `sig` |
| writes print `{event_id,accepted,message}` | via `normalize_write_response` | `social.rs:180` prints raw; `dms.rs:91` prints a hand-assembled object; `emoji.rs:148-153` prints `{accepted,message}` with no `event_id`; `channels.rs:785` prints a bespoke report |
| stderr is for errors only | `error.rs:135` | `emoji.rs:303` writes `(dry run — not published)` as bare text (not JSON) to stderr; `channels.rs:597` writes a JSON `{"warning":…}` line to stderr |
| absent single resource prints `null` | `channels.rs:239`, `:277` | `notes get` errors with `NotFound` instead (`notes.rs:637`) |

So there are two conventions for "not found on a single-item read" and four
distinct write-response shapes.

#### Naming

- Handlers: `cmd_<verb>_<noun>` in `channels.rs`/`messages.rs`/`dms.rs`
  (`cmd_list_channels`, `cmd_send_message`, `cmd_hide_dm`), but bare `cmd_<verb>`
  in `emoji.rs` and `notes.rs` (`cmd_list`, `cmd_set`, `cmd_rm`, `cmd_ls`), which
  are also `async fn` (private) in `emoji.rs` vs `pub async fn` in `notes.rs`.
- Filter builders: inline `serde_json::json!` literals everywhere; no named filter
  constructors.
- Helper naming for relay reads is inconsistent: `fetch_own_note`
  (`notes.rs:168`), `fetch_own_emoji` (`emoji.rs:93`), `fetch_by_slug`
  (`notes.rs:187`), `fetch_by_coord` (`notes.rs:349`), `fetch_events`
  (`messages.rs:203`), `fetch_member_pubkeys` (`messages.rs:213`),
  `fetch_team_persona_slugs` (`channels.rs:399`), `scan_managed_agents_by_owner`
  (`channels.rs:440`) — `fetch_*` vs `scan_*` for the same operation shape.
- Two different local names for the same idea: `resolve_author` exists in both
  `messages.rs:394` and `notes.rs:204` with different signatures
  (`String` vs `PublicKey`) and different accepted inputs.
- The template-resolution code uses requirement IDs in comments as identifiers
  ("Owner invariant (F1)" `channels.rs:645`, "the F4 cardinality rule"
  `channels.rs:472`) with no in-repo definition of F1/F4 — an external-doc
  reference that cannot be resolved from the codebase.

#### `unsafe`, lint attributes, `unwrap`/`expect`

`grep -rn 'unsafe' crates/buzz-cli/src` returns **zero matches** — the no-`unsafe`
rule holds.

Lint attributes in scope: `#[allow(clippy::too_many_arguments)]` twice
(`channels.rs:654`, `:794`), on `cmd_create_channel_from_template` (8 params) and
`build_template_report` (7 params). No `#[allow(dead_code)]` in these ten files
(`grep -n 'allow(dead_code)'` → zero matches; the two in the crate are in
`client.rs:567` and `client.rs:802`).

Production-path `expect()`/`unwrap()` (AGENTS.md forbids new ones):

| Site | Call | Reachability |
|------|------|-------------|
| `channels.rs:148` | `serde_json::to_string(&matches).expect("serializing ChannelSummary")` | infallible in practice; inconsistent with `unwrap_or_default()` used 4 lines later at `channels.rs:104`/`:108` and at `:236`, `:258` |
| `notes.rs:632` | `name.expect("dispatch enforces --name xor --naddr")` | panics if `cmd_get` (a `pub fn`) is called without `validate_get_args` — the invariant lives in the caller, not the type |
| `channels.rs:314`, `:319`, `:703`, `:708` | `unreachable!()` | guarded by the preceding string match; correct but four copies |

`client.rs:501` also has an `expect("a full query page always has a last event")`
that these files depend on transitively via `query_paginated`.

#### Doc-comment discipline

Strong: `notes.rs` (module doc + every public item, including a "Carry-forward
semantics (ratified)" contract block at `:390-417`), `channel_templates.rs`
(module doc + every item), the template-resolution cluster in `channels.rs`
(`:329-398`, `:466-530`, `:583-654`, `:789-794` — unusually thorough, explaining
fail-open, testability seams, and why members are added sequentially),
`messages.rs`'s NIP-10 helpers (`:19-24`, `:41-56`, `:118-127`).

Weak: 30+ `pub` items with no doc comment, enumerated in the API Surface aspect —
including all nine `dispatch` functions, all `channels` lifecycle handlers, and
the `SendMessageParams`/`SendDiffParams` structs and their fields
(`messages.rs:474-482`, `:581-595`). `mod.rs` has no doc at all (`mod.rs:1-20`).
`reactions.rs`, `dms.rs` and `feed.rs` have per-function docs on some handlers
(`dms.rs:7`, `:50`, `:95`, `:110`; `feed.rs:8`) but none on `cmd_add_reaction`,
`cmd_remove_reaction`, `cmd_get_reactions`.

#### File-size discipline

| File | Total | Production (pre-`#[cfg(test)]`) | Test |
|------|-------|-------------------------------|------|
| `channels.rs` | 1713 | 1175 | 538 |
| `notes.rs` | 1330 | 820 | 510 |
| `messages.rs` | 1167 | 876 | 291 |
| `emoji.rs` | 389 | 325 | 64 |
| `social.rs` | 284 | 238 | 46 |
| `channel_templates.rs` | 195 | 125 | 70 |
| `reactions.rs` / `dms.rs` / `feed.rs` / `mod.rs` | 138 / 136 / 80 / 20 | all | none |

The repo enforces a 1000-line ceiling for mobile, desktop and web
(`justfile:123`, `:585`, `:617` invoking `check-file-sizes.mjs`), but there is no
equivalent guard for Rust — `grep -n 'check-file-sizes\|max-lines' justfile`
matches only those three JS/Dart invocations. `channels.rs` (1713) and `notes.rs`
(1330) exceed that ceiling; `channels.rs` is the largest file in the crate after
`lib.rs` and `client.rs`.

Longest functions: `cmd_create_channel_from_template` (`channels.rs:655-790`, 136
lines, 8 params, 3 sequential write stages), `messages::dispatch`
(`messages.rs:754-875`, 122 lines of struct-shuffling), `channels::dispatch`
(`channels.rs:1066-1167`, 102 lines including branching logic for
`create --template`), `cmd_send_message` (`messages.rs:483-579`, 97 lines doing
stdin, upload, threading, mentions and kind selection), `cmd_set`
(`notes.rs:487-582`, 96 lines), `cmd_import` (`emoji.rs:234-309`, 76 lines with
numbered steps 1–8 in comments).

`channels::dispatch` is the only dispatcher containing business logic rather than
pure delegation: it decides template-vs-plain creation and re-derives the
"required" flags (`channels.rs:1128-1148`).

#### Test organization

All tests are inline `#[cfg(test)] mod tests` at file bottom; there is no
`crates/buzz-cli/tests/` directory. Conventions observed:

- Named, intent-revealing test functions (`cardinality_multiple_instances_is_hard_error_listing_candidates`,
  `channels.rs:1423`; `malformed_root_does_not_shadow_valid_reply`,
  `messages.rs:848`).
- Section banner comments grouping tests (`channels.rs:1292`, `:1384`, `:1468`,
  `:1599`; `messages.rs:988`, `:1108`; `notes.rs:875`, `:1032`, `:1258`).
- Local fixture builders: `event()` (`channels.rs:1192`), `agent()`
  (`channels.rs:1388`), `profile_event()` (`messages.rs:1112`), `build_30023()`
  (`notes.rs:884`), `prior_snapshot()` (`notes.rs:1036`), `write_store()`
  (`channel_templates.rs:131`).
- Explanatory comments stating *why* a case matters, including several that
  document a previously-shipped bug (`messages.rs:995-998` "regression guard for
  the previous stub that always returned `vec![]`"; `channels.rs:1599-1606`
  "deleting the stderr emission … left all of them green").
- Constants for fixture hex values (`messages.rs:882-891`), with an honest
  comment about what `PublicKey::from_hex` actually validates
  (`messages.rs:1085-1088`).

Deviations: `reactions.rs`, `dms.rs`, `feed.rs` have no `#[cfg(test)]` module at
all; `channels.rs:1296-1307` re-implements a production rule inside the test
module (see Business Rules); `channels.rs:1363-1366` mutates process env with
`std::env::set_var`/`remove_var` without serialization, which is unsound if
another test in the same binary ever reads that variable concurrently.
