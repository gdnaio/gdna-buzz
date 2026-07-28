## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: Conventions

---

### 1. Module organization

| File | LOC | Role |
|---|---|---|
| `crates/buzz-sdk/src/lib.rs` | 112 | crate docs, lints, module declarations, shared input types, `SdkError`, `buzz-core` re-exports |
| `crates/buzz-sdk/src/builders.rs` | 3 824 | all 51 event builders + private validation helpers + 162 tests |
| `crates/buzz-sdk/src/mentions.rs` | 820 | pure mention-resolution helpers + 51 tests |
| `crates/buzz-sdk/src/nip_oa.rs` | 595 | NIP-OA auth-tag compute/verify/parse + 22 tests |
| `crates/buzz-sdk/examples/compute_auth_tag.rs` | 29 | example binary wrapping `nip_oa::compute_auth_tag` |

Flat two-level layout — no submodule directories. Types shared by more than one
builder live in `lib.rs`; types used by a single builder family live next to
that family in `builders.rs` (e.g. `GitPatchMeta` at `builders.rs:985`,
`DeleteMessageOptions` at `builders.rs:393`).

`pub use builders::*` (`lib.rs:19`) flattens the builder namespace so consumers
write `buzz_sdk::build_message`, while `mentions` and `nip_oa` stay
namespace-qualified (`lib.rs:15-17`).

Section banners with `// ---` rules mark thematic groups inside `builders.rs`
(`builders.rs:1587-1595` for moderation, `1692-1701` for NIP-IA,
`builders.rs:3286` and `3577` for test groups using `// ──` box-drawing rules).

---

### 2. Naming patterns

| Pattern | Convention | Examples |
|---|---|---|
| Event builders | `build_<operation>` | `build_message`, `build_forum_post`, `build_git_pr_update` (`builders.rs:219`, `278`, `1416`) |
| Variant builders | base name + `_with_<what>` suffix | `build_delete_message_with_options` (`builders.rs:411`), `build_repo_announcement_with_tags` (`builders.rs:952`) |
| Compatibility shims | `_compat` suffix | `build_delete_compat` (kind 5 vs native 9005) (`builders.rs:434`) |
| Private validators | `check_<subject>` returning `Result<(), SdkError>` or `Result<String, SdkError>` | `check_content`, `check_hex_len`, `check_commit_hex`, `check_pubkey_hex`, `check_hex_exact`, `check_repo_id`, `check_custom_emoji_url`, `check_reason`, `check_auth_tag_shape` (`builders.rs:35-170`, `1706-1737`) |
| Public normalizers | `normalize_<subject>` | `normalize_custom_emoji_shortcode` (`builders.rs:127`), `normalize_mention_pubkeys` (`mentions.rs:228`) |
| Tag emitters | `<subject>_tags(…, tags: &mut Vec<Tag>)` | `thread_tags`, `mention_tags`, `imeta_tags`, `identity_archive_tags` (`builders.rs:173`, `188`, `203`, `1739`) |
| Extractors | `extract_<subject>` | `extract_channel_id` (`builders.rs:816`), `extract_at_names`, `extract_nostr_uris` (`mentions.rs:64`, `353`) |
| Input bundles | `<Domain><Thing>Meta` / `Options` | `GitPatchMeta`, `GitIssueMeta`, `GitStatusMeta`, `GitPullRequestMeta`, `GitPrUpdateMeta`, `DeleteMessageOptions` |
| Constants | SCREAMING_SNAKE with unit in the name where relevant | `MENTION_CAP` (`mentions.rs:38`), `MAX_CONTACTS` (`builders.rs:751`), `MAX_REASON_BYTES` (`builders.rs:1704`), `CUSTOM_EMOJI_SET_D_TAG` (`builders.rs:503`) |
| `nip_oa` functions | verb-first, NIP-term nouns | `compute_auth_tag`, `verify_auth_tag`, `parse_auth_tag` |

Local shorthand inside tests: `ev`, `cid`, `eid`, `pk`, `wid`, `rid`
(`builders.rs:1958-1975`).

---

### 3. Builder-pattern style

There is **no fluent/typestate builder struct**. The style is:

1. Free function taking positional parameters, plus one `&Meta` struct when the
   optional-tag count grows large (Git family, `builders.rs:1013`, `1081`,
   `1222`, `1330`, `1416`).
2. Validate everything first, returning early on error.
3. Accumulate `Vec<Tag>` in wire order — `h`/`a`/`d` first, then optional tags.
4. Return `EventBuilder::new(Kind::Custom(n), content).tags(tags)` as the final
   expression. Signing is deliberately excluded (`lib.rs:11-13`).

Optionality is expressed through the type system rather than setters:
`Option<&str>` for optional strings, `Option<MemberRole>` for enums,
`Option<Option<i32>>` for the tri-state TTL (`builders.rs:604-609`), `bool` flags
for presence-only tags (`broadcast`, `truncated`, `root`, `root_revision`).

`#[derive(Default)]` on the five Git meta structs enables `..Default::default()`
partial initialization, which is how tests and the CLI construct them
(`builders.rs:984`, `1072`, `1200`, `1302`, `1396`).

Every kind integer reaches nostr through `Kind::Custom(... as u16)` — 26 sites
use `buzz_core::kind::KIND_*` constants (`builders.rs:6-19`), the rest use bare
literals.

---

### 4. Error handling

- Single crate-level error enum, `SdkError`, derived with `thiserror::Error`
  (`lib.rs:87-113`); six variants, three of which carry a free-form `String`
  message (`InvalidTag`, `InvalidDiffMeta`, `InvalidInput`) and one structured
  (`ContentTooLarge { max, got }`).
- Uniform return type: every fallible public function returns
  `Result<_, SdkError>`. There is no `Box<dyn Error>`, no `anyhow`, and no `From`
  impl chain — third-party errors are stringified at the boundary
  (`builders.rs:31`, `1372`; `nip_oa.rs:126`, `209`, `220`, `242`).
- **No `unwrap()`/`expect()` in non-test code** (verified by scan of `src/*.rs`
  outside `#[cfg(test)]`). The one near-miss is
  `parts.next().unwrap_or_default()` in `GitAppliedPatchRef::parse`
  (`builders.rs:1145`), which cannot panic.
- Guard-clause style: validation blocks sit at the top of each function and
  `return Err(...)` immediately; the tag-building section is unconditional
  afterwards (e.g. `builders.rs:314-340`, `840-919`).
- Error messages embed the offending value or measured size for diagnosis:
  `"{field} must be at least {min_len} hex characters (got {:?})"`
  (`builders.rs:47-49`), `"repo_id exceeds 64 characters (got {})"`
  (`builders.rs:97-100`).
- The example binary is the only place that panics on bad input, via `.expect()`
  on argument parsing (`examples/compute_auth_tag.rs:21-27`).

---

### 5. Doc-comment practice

- `#![warn(missing_docs)]` (`lib.rs:2`) — every public item is documented,
  including struct fields (e.g. `DiffMeta`'s ten fields each carry a `///`
  line, `lib.rs:37-56`) and enum variants (`lib.rs:62-65`).
- Crate doc opens with an ASCII "Mental Model" pipeline diagram
  (`lib.rs:5-13`); `mentions.rs:1-30` and `nip_oa.rs:1-19` do the same for their
  modules (pipeline diagram and tag-format/preimage blocks respectively).
- Builder docs state the kind number in prose ("Build a stream message (kind 9)",
  `builders.rs:211`) and enumerate parameters as a bullet list when there are
  more than three (`builders.rs:211-218`, `1465-1469`).
- Rationale comments cite the governing NIP and, frequently, the exact
  cross-repo file being mirrored: `desktop/src-tauri/src/events.rs:624-743`
  (`builders.rs:1700`), `desktop/src-tauri/src/events.rs:635-647`
  (`builders.rs:1702-1704`), `moderation_commands.rs` (`builders.rs:1593-1594`),
  `NIP-IA.md §Vector 1` (`builders.rs:3625`).
- Deviations from a NIP are called out inline rather than hidden — e.g. "The `h`
  tag is non-standard for NIP-09 but is required so channel-scoped subscriptions
  observe the delete." (`builders.rs:432-433`).
- `# Errors` sections appear in `nip_oa` public functions
  (`nip_oa.rs:140-144`, `173-177`, `245-250`).

---

### 6. Testing patterns

| Aspect | Convention | Evidence |
|---|---|---|
| Location | one `#[cfg(test)] mod tests` per source file; no `tests/` directory in the crate | `builders.rs:1825`, `mentions.rs:389`, `nip_oa.rs:301` |
| Volume | 235 `#[test]` functions total: 162 in `builders.rs`, 51 in `mentions.rs`, 22 in `nip_oa.rs` | counted per file |
| Framework | stock `#[test]` only — no proptest, no rstest, no async tests | no `proptest`/`quickcheck` reference anywhere in the crate |
| Test helpers | small fixture fns at the top of the module: `keys()`, `sign()`, `event_id()`, `uuid()`, `tag_values()`, `has_tag()` | `builders.rs:1830-1873` |
| Domain fixtures | per-family fixture fns: `good_diff_meta()` (`builders.rs:2054`), `pr_repo()` (`builders.rs:3396`), `full_clone_tag()` (`builders.rs:3403`), `profile()` (`mentions.rs:545`) |
| Naming | `<subject>_<scenario>`: `*_happy_path`, `*_minimal`, `*_all_fields`, `*_rejects_*`, `*_too_large` | `builders.rs:1874`, `2821`, `2145`, `2872` |
| Assertion style | sign the builder, then assert on `ev.kind.as_u16()`, `ev.content`, and tag presence via `has_tag`/`tag_values`; positional tag layout asserted where wire order matters | `builders.rs:3707-3735`, `3744-3771` |
| Error assertions | `assert!(matches!(result, Err(SdkError::Variant { .. })))` | `builders.rs:2080-2087`, `2177-2185` |
| Boundary pairs | max-allowed and max+1 tested together | `builders.rs:2009-2020` (64 KiB), `builders.rs:2260-2273` (64-char emoji), `builders.rs:3773-3788` (64-byte reason) |
| Regression comments | tests carry a comment naming the bug or cross-component contract they pin | `builders.rs:3101-3105` (whitespace-only patch), `builders.rs:3613-3615` (relay `extract_p_tag_bytes`), `builders.rs:3688-3690` (relay `extract_report_tag`) |
| Spec vectors | NIP-OA spec pubkeys/conditions/signature and NIP-IA target/owner hex are hoisted into module `const`s and asserted against | `nip_oa.rs:303-310`, `builders.rs:3701-3705` |
| Unicode/panic-safety tests | explicit non-panic tests for multi-byte input | `mentions.rs:520-531`, `mentions.rs:787-800` |
| Cross-client pinning | test comments state that the assertions mirror `desktop/src-tauri/src/events.rs`'s own tests so both clients stay on one wire form | `builders.rs:3696-3699` |
