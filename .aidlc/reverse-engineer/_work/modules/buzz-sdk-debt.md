## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: Technical Debt

---

### 1. File-size distribution

| File | Total LOC | Production LOC | Test LOC | Tests start |
|---|---|---|---|---|
| `crates/buzz-sdk/src/builders.rs` | 3 824 | 1 824 | 1 999 | `builders.rs:1825` |
| `crates/buzz-sdk/src/mentions.rs` | 820 | 388 | 432 | `mentions.rs:389` |
| `crates/buzz-sdk/src/nip_oa.rs` | 595 | 300 | 295 | `nip_oa.rs:301` |
| `crates/buzz-sdk/src/lib.rs` | 112 | 112 | 0 | — |
| `crates/buzz-sdk/examples/compute_auth_tag.rs` | 29 | 29 | 0 | — |
| **Total** | **5 380** | **2 653** | **2 726** | — |

`builders.rs` is 71 % of the crate and holds 51 of the 61 public functions in
one file. Slightly more than half the crate's lines are tests.

For reference, the repo enforces a 1 000-line ceiling on mobile files
(`AGENTS.md` § Mobile Rules, `mobile/scripts/check-file-sizes.mjs`) and the same
guard exists for desktop/web; no equivalent guard applies to Rust crates, so
`builders.rs` at 3 824 lines is unconstrained.

---

### 2. Complexity hotspots (largest production functions)

| Function | Body lines | File:line | Why it is large |
|---|---|---|---|
| `build_repo_announcement` | 112 | `builders.rs:834-945` | six sequential validation blocks (repo_id, name, description, clone_urls loop, web_url, relays loop) then tag assembly; each block is an inline `if len > N` rather than a shared helper |
| `build_git_status` | 70 | `builders.rs:1222-1291` | validates root/revision/recipients/repo/euc, enforces the merged-only rule, then a four-arm match over `(relay, pubkey)` for `q` tags plus three commit-tag paths |
| `build_diff_message` | 68 | `builders.rs:308-375` | validates 6 `DiffMeta` fields and conditionally emits 10 optional tags |
| `build_git_pull_request` | 62 | `builders.rs:1330-1391` | 5 validations + 11 conditional tag emissions |
| `build_git_patch` | 57 | `builders.rs:1013-1069` | 9 conditional tag emissions including the root/root-revision exclusivity check |
| `build_contact_list` | 48 | `builders.rs:764-811` | per-contact validation loop with three inline bound checks |
| `build_update_channel` | 46 | `builders.rs:604-649` | at-least-one-field check + tri-state TTL handling |
| `strip_code_regions` | 96 | `mentions.rs:244-339` | hand-rolled markdown scanner: nested `loop` inside `while let` with manual byte-offset arithmetic, four separate "is this a line start" predicates, and manual iterator advancement. The highest-complexity function in the crate |
| `verify_auth_tag` | 58 | `nip_oa.rs:179-236` | 8 sequential `ok_or_else` element extractions duplicating `parse_auth_tag`'s first half |
| `parse_auth_tag` | 48 | `nip_oa.rs:252-299` | same extraction sequence as `verify_auth_tag` with different length rules |

---

### 3. Duplication

**Within the crate**

| Duplication | Detail | File:line |
|---|---|---|
| `build_workflow_def` vs `build_workflow_update` | Identical bodies — same kind, same two tags, same content cap, same 64 KiB check. Only the doc comment differs | `builders.rs:1463-1478` vs `1481-1494` |
| `verify_auth_tag` / `parse_auth_tag` element extraction | Lines `nip_oa.rs:181-206` and `254-286` repeat the arity check, label check, and four `as_str().ok_or_else(...)` extractions verbatim; no shared "destructure" helper | `nip_oa.rs:179-299` |
| Five overlapping hex validators | `check_hex_len`, `check_commit_hex`, `check_pubkey_hex`, `check_hex_exact`, plus an inline 64-hex check in `build_contact_list` (`builders.rs:779-784`) and another in `build_workflow_approval` (`builders.rs:1528-1532`) — six variants of "is this hex of length N" | `builders.rs:44-89`, `779-784`, `1528-1532` |
| `check_hex_len` misuse for pubkeys | `build_add_member`/`build_remove_member` validate pubkeys with `check_hex_len(pubkey, 64, …)`, which is a **minimum**-length check and produces an `InvalidDiffMeta` error variant for a membership problem — while every other pubkey site uses `check_pubkey_hex` (exact length, `InvalidInput`). A 65-hex pubkey passes here but is rejected in the moderation builders | `builders.rs:569`, `586` vs `69-77` |
| `build_archive` / `build_unarchive` | Same body modulo the boolean literal | `builders.rs:709-724` |
| `build_set_topic` / `build_set_purpose` | Same body modulo the tag name | `builders.rs:652-667` |
| Inline size literals | 13 sites pass `64 * 1024`, 2 pass `60 * 1024`, plus 2048/1024/512/256/128 scattered inline instead of named constants (contrast with `MAX_CONTACTS`/`MAX_REASON_BYTES` which are named) | `builders.rs:227`–`1816` (see configuration doc) |

**Across crates**

| Duplication | Detail | Evidence |
|---|---|---|
| Desktop maintains a parallel builder library | `desktop/src-tauri/src/events.rs` defines its own `build_create_channel`, `build_join`, `build_leave`, `build_update_channel`, `build_set_topic`, `build_set_purpose`, `build_archive`, `build_unarchive`, `build_delete_channel`, `build_add_member`, `build_remove_member`, `build_message`, `build_forum_post`, `build_forum_comment`, `build_delete_compat`, `build_reaction`, `build_remove_reaction`, `build_set_canvas`, `build_profile`, `build_note`, `build_contact_list`, `build_dm_open`, `build_workflow_delete`, `build_workflow_trigger`, `build_archive_identity_request`, `build_unarchive_identity_request` — 36 `EventBuilder::new` sites, returning `Result<_, String>` instead of `SdkError`. `desktop/src-tauri` declares `buzz-sdk` as a dependency but references no `buzz_sdk::` path in `src/` | `desktop/src-tauri/src/events.rs:143-842`; SDK equivalents `builders.rs:219-1823` |
| Wire-form drift is managed by comment + mirrored tests, not shared code | The SDK's NIP-IA section states it mirrors `desktop/src-tauri/src/events.rs:624-743` "so both clients emit the same wire form", and duplicates the desktop's byte-based reason check with a comment explaining the desktop's misleading error text | `builders.rs:1697-1704`, test comment `builders.rs:3696-3699` |
| `repo_id` validation exists twice | `buzz-cli`'s `validate_repo_id` (`crates/buzz-cli/src/validate.rs:39`) has its own copy with its own 7 tests; the SDK's `check_repo_id` was added specifically because a coordinate could bypass it ("bypassing CLI-side `validate_repo_id`") | `builders.rs:87-91`; `crates/buzz-cli/src/validate.rs:39`, `423-461` |
| `validate_pubkey_hex` in desktop huddle path | third independent pubkey-hex validator | `desktop/src-tauri/src/huddle/relay_api.rs:29` |

---

### 4. Dead code / unused API

- No `#[allow(dead_code)]` and no `#[deprecated]` attributes anywhere in the crate.
- Every one of the 51 public builders plus all 10 public helper functions is
  referenced from at least one `.rs` file outside `crates/buzz-sdk/` (verified by
  per-symbol search across `crates/` and `desktop/`). There is no unreachable
  public API by that measure.
- Reference concentration is uneven: `compute_auth_tag` (12 files),
  `verify_auth_tag` (10), `parse_auth_tag` (8), `build_message` (6),
  `build_archive_identity_request` (6) at the top; 22 builders are referenced from
  exactly one external file, all of them `buzz-cli` command modules (spot-checked:
  `build_custom_emoji_set` → `crates/buzz-cli/src/commands/emoji.rs:119`,
  `build_diff_message` → `crates/buzz-cli/src/commands/messages.rs:659`,
  `build_vote` → `crates/buzz-cli/src/commands/messages.rs:744`).
- `serde` is declared as a dependency (`crates/buzz-sdk/Cargo.toml:14`) but no
  `use serde` or serde derive appears in `src/` — it is only needed transitively
  by `serde_json`, so the direct declaration is unused.

---

### 5. Test coverage gaps

235 tests exist (162 builders / 51 mentions / 22 nip_oa). Gaps observed by
comparing the test module against the production surface:

| Untested or thinly tested | Detail | File:line |
|---|---|---|
| `build_git_pr_update` merge-base / euc validation | happy path, bad `pr_event`, and missing clone URL are covered; invalid `merge_base`/`euc` hex are not | `builders.rs:1416-1460`; tests `builders.rs:3533-3577` |
| `build_repo_announcement` bound checks | repo_id rules have 5 tests, but the `name` > 128, `description` > 1024, `clone_urls` > 5, `clone_url` > 512, `web_url` scheme/length, `relays` > 10, and relay-scheme rejections have no tests | validations `builders.rs:840-919`; tests `builders.rs:2788-2925` |
| `build_repo_announcement_with_tags` | one test (`builders.rs:2837-2866`); no test that an invalid `repo_id` is rejected on this path | `builders.rs:952-963` |
| `build_custom_emoji_set` / `build_custom_emoji_reaction` negatives | happy paths only; no test for duplicate shortcode, empty shortcode, > 64-byte shortcode, non-`http(s)` URL, or > 2048-byte URL | validations `builders.rs:127-170`, `517-521`; tests `builders.rs:2276-2301` |
| `normalize_custom_emoji_shortcode` | no direct unit test; exercised only indirectly through the reaction happy path | `builders.rs:127-150` |
| `build_set_canvas` | one happy-path test; nothing pins the (absent) content bound | `builders.rs:529-532`; test `builders.rs:2310-2317` |
| `build_workflow_approval` note length | no bound exists and no test asserts either way | `builders.rs:1522-1541` |
| `build_dm_add_member` / `build_dm_open` boundary | 8-pubkey upper bound tested at 9 (reject); exactly-8 accept case not tested | `builders.rs:1546`; tests `builders.rs:3324-3346` |
| `extract_channel_id` on multiple `h` tags | first-match behavior is not pinned by a test | `builders.rs:816-826` |
| `strip_code_regions` edge cases | 6 tests; no coverage for indented fences, nested/triple-backtick-inside-fence, unclosed fenced block at EOF, or the `before.chars().all(is_ascii_whitespace)` branch | `mentions.rs:244-339`; tests `mentions.rs:662-700` |
| `merge_mentions` with a custom `cap` | all three tests pass `MENTION_CAP`; no test uses a different cap value | `mentions.rs:208-220`; tests `mentions.rs:616-638` |
| `compute_auth_tag` byte-exactness | round-trip and spec-vector verification are tested, but no test asserts `compute_auth_tag` reproduces the spec signature byte-for-byte (the test comment calls this out: "round-trip without byte comparison") | `nip_oa.rs:335-350` |
| Kind literals vs `buzz-core` constants | 20 builders pass bare integer literals to `Kind::Custom` (9, 45001, 45003, 40008, 40003, 9005, 5, 45002, 7, 40100, 0, 9000, 9001, 9002, 9007, 9021, 9008, 9022, 1, 3) and the tests assert the same literals, so a divergence from `buzz-core/src/kind.rs` would not be caught | `builders.rs:241`, `288`, `304`, …; registry `crates/buzz-core/src/kind.rs` |

No property-based tests (no `proptest`/`quickcheck` anywhere in the crate), and
no integration-test directory (`crates/buzz-sdk/tests/` does not exist) despite
the crate producing wire forms consumed by the relay.

---

### 6. Other debt signals

| Signal | Detail | File:line |
|---|---|---|
| Kind sourcing is inconsistent | 26 builders use `buzz_core::kind::KIND_*` constants (`builders.rs:6-19`), 20 use bare literals. `AGENTS.md` states all kind integers are defined in `buzz-core/src/kind.rs`; the literal sites bypass that registry | `builders.rs:241` (literal) vs `builders.rs:270-271` (constant) |
| Error-variant semantics leak | `InvalidDiffMeta` is used for non-diff failures because `check_hex_len` hardcodes that variant — e.g. an invalid membership pubkey reports "invalid diff metadata" | `builders.rs:44-52` used at `builders.rs:569`, `586` |
| `check_hex_len` is a minimum-length check used where an exact length is meant | see duplication table above — permits over-long pubkeys on kinds 9000/9001 | `builders.rs:44-52`, `569`, `586` |
| Stringly-typed parameters where enums exist | `build_update_channel` takes `visibility: Option<&str>` and re-validates the vocabulary by hand, while `build_create_channel` takes the typed `Option<Visibility>`; `build_presence_update` and `build_moderation_resolve_report` also take raw `&str` vocabularies | `builders.rs:604-621` vs `674-696`; `1570-1584`; `1654-1680` |
| Positional-parameter builders with 5–6 arguments | `build_message` (6), `build_profile` (5), `build_update_channel` (5), `build_repo_announcement` (6), `build_create_channel` (6) — no options struct, so call sites are order-sensitive; the crate already has the `…Meta`/`Options` pattern available | `builders.rs:219`, `537`, `604`, `674`, `834` |
| Cross-repo coupling encoded in comments | correctness depends on line-referenced comments pointing at `desktop/src-tauri/src/events.rs:624-743` / `:635-647`, `moderation_commands.rs`, relay helpers `extract_p_tag_bytes` and `extract_report_tag` — these references will silently rot as those files change | `builders.rs:1697-1704`, `1593-1594`; test comments `builders.rs:3613-3615`, `3688-3690` |
| Manual byte-offset string scanning | `strip_code_regions` (`mentions.rs:244-339`) and `GitAppliedPatchRef::parse` (`builders.rs:1143-1185`) do their own index arithmetic instead of using a parser; both are correct today and guarded by tests, but they are the crate's most fragile code |
| Repeated `as u16` narrowing casts | `buzz_core` kind constants are `u32` and every use is cast `as u16`; a kind ≥ 65536 would wrap silently | 27 sites: `builders.rs:271`, `525`, `944`, `961`, `1068`, `1110`, `1190-1193`, `1390`, `1455`, `1473`, `1491`, `1507`, `1513`, `1538`, `1555`, `1562`, `1580`, `1610`, `1617`, `1636`, `1643`, `1685`, `1798`, `1819` |
| No deprecated API usage | no `#[deprecated]` items and no calls into deprecated `nostr` APIs observed; the crate is current with nostr 0.44 semantics (including the `allow_self_tagging` change at `builders.rs:1801`) | — |
