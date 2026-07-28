## Module: buzz-cli — repo, agent, memory & moderation commands (`crates/buzz-cli/src/commands`)
### Aspect: Data Model

The eleven modules in this group hold no persistent state of their own. Their data model is
(a) the Nostr event kinds they read and write, (b) the tag vocabularies on those events,
(c) the filter shapes they build, (d) the JSON shapes they print, and (e) three local file
formats they read (memory patch files, git patch files, persona pack directories).

#### Event kinds written

Every integer below is defined in `crates/buzz-core/src/kind.rs`. Where a module hardcodes
the literal instead of importing the constant, that is called out — it is a real
inconsistency inside this group (see Debt).

| Kind | Constant (`kind.rs:LINE`) | Written by | Use site | Constant or literal? |
|---|---|---|---|---|
| 30174 | `KIND_AGENT_ENGRAM` (`kind.rs:94`) | `mem set` / `mem patch` / `mem rm` | via `engram::build_event`, called at `mem.rs:359`, `mem.rs:684`, `mem.rs:728` | constant, imported `mem.rs:15` |
| 24200 | `KIND_AGENT_OBSERVER_FRAME` | `agents draft-create` / `draft-update` | via `build_create`/`build_update` (`agents.rs:23`, `agents.rs:60`) → `buzz_sdk::build_agent_observer_frame` | constant (in `agent_management.rs`) |
| 9035 | `KIND_IA_ARCHIVE_REQUEST` (`kind.rs:348`) | `agents archive` | `build_archive_identity_request` at `agents.rs:110` | constant (in SDK) |
| 9036 | `KIND_IA_UNARCHIVE_REQUEST` (`kind.rs:350`) | `agents unarchive` | `build_unarchive_identity_request` at `agents.rs:139` | constant (in SDK) |
| 30617 | `KIND_GIT_REPO_ANNOUNCEMENT` (`kind.rs:545`) | `repos create`, `repos protect set/remove` | `buzz_sdk::build_repo_announcement` (`repos.rs:213`), `build_repo_announcement_with_tags` (`repos.rs:136`) | constant (in SDK) |
| 1617 | `KIND_GIT_PATCH` (`kind.rs:549`) | `patches send` | `buzz_sdk::build_git_patch` at `patches.rs:50` | constant (in SDK) |
| 1618 | `KIND_GIT_PULL_REQUEST` (`kind.rs:551`) | `pr open` | `buzz_sdk::build_git_pull_request` at `pr.rs:57` | constant (in SDK) |
| 1619 | `KIND_GIT_PR_UPDATE` (`kind.rs:553`) | `pr update` | `buzz_sdk::build_git_pr_update` at `pr.rs:101` | constant (in SDK) |
| 1621 | `KIND_GIT_ISSUE` (`kind.rs:555`) | `issues create` | `buzz_sdk::build_git_issue` at `issues.rs:29` | constant (in SDK) |
| 1630/1631/1632/1633 | `KIND_GIT_STATUS_OPEN`/`_MERGED`/`_CLOSED`/`_DRAFT` (`kind.rs:557-563`) | `patches status`, `pr status`, `issues status` | `buzz_sdk::build_git_status` at `patches.rs:186`, `pr.rs:210`, `issues.rs:140` | constant (in SDK), selected by `parse_status` (`patches.rs:194-205`) |
| 0 | metadata | `users set-profile` | `buzz_sdk::build_profile` at `users.rs:198` | implicit in SDK |
| 20001 | `KIND_PRESENCE_UPDATE` (`kind.rs:403`) | `users set-presence` | `buzz_sdk::build_presence_update` at `users.rs:299` | constant (in SDK) |
| 30620 | `KIND_WORKFLOW_DEF` (`kind.rs:382`) | `workflows create` / `update` | `build_workflow_def` (`workflows.rs:108`), `build_workflow_update` (`workflows.rs:127`) | constant (in SDK) |
| 5 | `KIND_DELETION` | `workflows delete` | `buzz_sdk::build_workflow_delete` at `workflows.rs:143` | constant (in SDK) |
| 46020 | `KIND_WORKFLOW_TRIGGER` (`kind.rs:498`) | `workflows trigger` | `workflows.rs:177` (hand-built) and `workflows.rs:186` (SDK) | constant, but referenced as `buzz_sdk::kind::KIND_WORKFLOW_TRIGGER` only on the hand-built branch |
| 46030 / 46031 | `KIND_APPROVAL_GRANT` / `KIND_APPROVAL_DENY` (`kind.rs:500`, `kind.rs:502`) | `workflows approve` | `buzz_sdk::build_workflow_approval` at `workflows.rs:207` | constant (in SDK); the doc comment at `workflows.rs:192` naming "46030 (grant) or 46031 (deny)" matches the registry |
| 9040 | `KIND_MODERATION_BAN` (`kind.rs:298`) | `moderation ban` | `build_moderation_ban` at `moderation.rs:41` | constant (in SDK) |
| 9041 | `KIND_MODERATION_UNBAN` (`kind.rs:300`) | `moderation unban` | `moderation.rs:53` | constant (in SDK) |
| 9042 | `KIND_MODERATION_TIMEOUT` (`kind.rs:303`) | `moderation timeout` | `moderation.rs:71` | constant (in SDK) |
| 9043 | `KIND_MODERATION_UNTIMEOUT` (`kind.rs:305`) | `moderation untimeout` | `moderation.rs:81` | constant (in SDK) |
| 9044 | `KIND_MODERATION_RESOLVE_REPORT` (`kind.rs:310`) | `moderation resolve` | `moderation.rs:96` | constant (in SDK) |

`upload.rs` writes no Nostr event at all: `upload file` PUTs bytes to Blossom
(`upload.rs:7`) and `media get` GETs them (`upload.rs:20`). `pack.rs` writes nothing —
both subcommands are read-only local filesystem operations (`pack.rs:15`, `pack.rs:52`).

#### Event kinds read (filter shapes)

Every filter in this group carries an explicit `kinds` array. Grep for filters in these
eleven files (`grep -n '"kinds"' mem.rs agents.rs repos.rs users.rs pr.rs patches.rs
workflows.rs issues.rs`) returns 18 matches and every one of them is a `"kinds": [...]`
entry — there is no kindless filter anywhere in this group, so none of these commands can
trip the relay's `p_gated_filters_authorized` "kindless filter" branch
(`crates/buzz-relay/src/handlers/req.rs:1040-1046`).

| Command | Filter (file:line) | Kinds | Scoping keys | Limit |
|---|---|---|---|---|
| `mem get/hash/patch` (head fetch) | `mem.rs:150-156` | `[KIND_AGENT_ENGRAM]` | `authors:[agent]`, `#d:[d]`, `#p:[owner]` | 16 |
| `mem ls` | `mem.rs:197-202` | `[KIND_AGENT_ENGRAM]` | `authors:[agent]`, `#p:[owner]` | 5000 |
| `agents archive/unarchive` (auth-tag probe) | `agents.rs:180` | `[0]` | `authors:[target]` | 1 |
| `agents archived` | `agents.rs:285` | `[KIND_IA_ARCHIVED_LIST]` (13535, `kind.rs:358`) | `authors:[relay self]` | 1 |
| `repos protect *` (own head) | `repos.rs:20-25` | `[KIND_GIT_REPO_ANNOUNCEMENT]` | `authors:[self]`, `#d:[repo_id]` | 1 |
| `repos get` | `repos.rs:239-242` | `[30617]` **literal** | `#d:[repo_id]`, optional `authors:[owner]` | none |
| `repos list` | `repos.rs:270-273` | `[30617]` **literal** | `authors:[owner or self]` | optional `--limit` |
| `users get` | `users.rs:41-45` | `[0]` | `authors:[…]` | `authors.len()` |
| `users get --name` | `users.rs:90-94` | `[0]` | NIP-50 `search` | 100 |
| `users set-profile` (read-merge-write) | `users.rs:222-226` | `[0]` | `authors:[self]` | 1 |
| `users presence` | `users.rs:257-261` | `[40902]` **literal** (`KIND_PRESENCE_SNAPSHOT`, `kind.rs:443`) | `authors:[csv]` | `pubkeys.len()` |
| `pr get` | `pr.rs:109-112` | `[1618]` **literal** | `ids:[event]` | none |
| `pr list` | `pr.rs:130-133` | `[1618]` **literal** | `#a:["30617:owner:id"]`, optional `authors`, `#t` | optional |
| `patches get` | `patches.rs:75-78` | `[1617]` **literal** | `ids:[event]` | none |
| `patches list` | `patches.rs:95-98` | `[1617]` **literal** | `#a`, optional `authors` | optional |
| `issues get` | `issues.rs:38-41` | `[1621]` **literal** | `ids:[event]` | none |
| `issues list` | `issues.rs:59-62` | `[1621]` **literal** | `#a`, optional `authors`, `#t` | optional |
| `workflows list` | `workflows.rs:15-18` | `[30620]` **literal** | `#h:[channel_id]` | none |
| `workflows get` | `workflows.rs:40-43` | `[30620]` **literal** | `#d:[workflow_id]` | none |
| `workflows runs` | `workflows.rs:73-77` | `[46001, 46002, 46003]` **literals** | `#d:[workflow_id]` | `min(limit, 100)` (`workflows.rs:72`) |

Only `mem.rs` and `repos.rs` import kind constants from `buzz_core::kind`
(`mem.rs:16`, `repos.rs:3`). The other six read paths embed decimal literals, so a kind
renumbering in `kind.rs` would not be caught by the compiler for `repos get`/`list`,
`users presence`, `pr`, `patches`, `issues`, and `workflows`.

#### NIP-33 addressable identifiers and `d`-tag conventions

Four distinct `d`-tag conventions cross this group:

| Kind | `d` value | Derived at | Notes |
|---|---|---|---|
| 30174 engram | `lower_hex(HMAC-SHA256(K_c, "agent-memory/v1/d-tag" ‖ 0x00 ‖ slug))` | `engram::d_tag` (`crates/buzz-core/src/engram.rs:144-153`), called from `mem.rs:148` | opaque — the slug never appears on the wire; `validate_and_decrypt` re-derives it from the decrypted body and rejects a mismatch (`engram.rs:546-552`) |
| 30617 repo | the raw `repo_id` string | `repos.rs:23` (filter), `buzz_sdk::build_repo_announcement_with_tags` re-inserts it at index 0 (`crates/buzz-sdk/src/builders.rs:958-959`) | validated by `validate_repo_id` (`crates/buzz-cli/src/validate.rs:40-61`) |
| 30620 workflow | the workflow UUID string | `build_workflow_def`/`_update` (`builders.rs:1470`, `builders.rs:1488`) | client-generated UUID at `workflows.rs:107`; the relay may return a different id, preferred at `workflows.rs:111-112` |
| 46030/46031 approval | `hex(SHA256(approval_token_uuid_ascii))` | `workflows.rs:205` | the raw token UUID is validated as a UUID at `workflows.rs:196` but never sent; only its digest is |

The addressable coordinate string `30617:<owner_hex>:<repo_id>` is built by hand in three
places — `pr.rs:127`, `patches.rs:94`, `issues.rs:58` — and once inside the SDK
(`GitRepoCoord::to_a_tag_value`, `builders.rs:976-981`). The three CLI copies use
`format!("30617:{repo_owner}:{repo_id}")` with the kind hardcoded.

#### Tag vocabularies

| Tag | Shape | Where produced / consumed |
|---|---|---|
| `d` | `["d", <see table above>]` | produced by every NIP-33 write in this group; read by `repo_id_from_event` (`repos.rs:32-43`) and `client::extract_d_tag` (used at `workflows.rs:23`, `workflows.rs:48`) |
| `p` | `["p", <hex64>]` | engram owner (`engram.rs:465`); moderation target (`builders.rs:1603`); NIP-IA target (`builders.rs:1748`); NIP-34 status recipients assembled at `patches.rs:170-178`, `pr.rs:186-194`, `issues.rs:127-135`; read back by `verify_archived_event` (`agents.rs:353-366`) and `presence_subject` (`users.rs:279-289`) |
| `buzz-protect` | `["buzz-protect", <ref-pattern>, <rule>…]` where rule ∈ `push:{owner,admin,member}`, `no-force-push`, `no-delete`, `require-patch` | built at `repos.rs:60-85`; matched by `protection_pattern` (`repos.rs:49-54`); projected to JSON at `repos.rs:152-163`; grammar owned by `crates/buzz-core/src/git_perms.rs` |
| `auth` | `["auth", owner_hex, conditions, sig_hex128]` | NIP-OA membership tag injected by `BuzzClient::sign_event` (`client.rs:588-591`); a *different*, content-level owner-of-agent `auth` tag is extracted from the target's kind:0 by `extract_owner_auth_tag` (`agents.rs:206-248`) and re-emitted on 9035/9036. `repos.rs:100-107` strips any inherited `auth` tag before rebuilding the announcement so the client re-injects exactly one |
| `-` | `["-"]`, arity exactly 1 | NIP-70 protection marker; required on 13535 and enforced at `agents.rs:335-352` (exactly one, and any `-`-labelled tag of arity ≠ 1 is a hard error) |
| `expiration` | `["expiration", <unix-secs>]` | ban/timeout expiry (`builders.rs:1604-1606`), computed by `resolve_expiry` (`moderation.rs:26-32`) |
| `reason` | `["reason", <free text>]` | moderation (`builders.rs:1607-1609`, uncapped) and NIP-IA (`builders.rs:1750-1753`, capped at 64 bytes + no control chars via `check_reason`, `builders.rs:1706-1719`) |
| `report` / `status` / `action` | `["report", <hex64>]`, `["status", resolved\|dismissed]`, `["action", delete\|kick\|ban\|timeout\|dismiss\|escalate]` | `moderation resolve`, validated in the SDK at `builders.rs:1661-1681` — **not** in `moderation.rs` |
| `replaced-by` | `["replaced-by", <hex64>]` | `agents archive --replaced-by`; must differ from the target (`builders.rs:1757-1761`) |
| `h` | `["h", <channel uuid>]` | workflow definitions (`builders.rs:1471`); `pr open --channel` passes a channel id into `GitPullRequestMeta.channel_id` (`pr.rs:44`) |
| `#t` | label filter | `pr list --label` (`pr.rs:140`), `issues list --label` (`issues.rs:69`) |

#### Output JSON shapes

| Command(s) | Shape | Site |
|---|---|---|
`repos create`, `repos get`, `repos list`, `pr open/update/get/list/status`, `patches send/get/list/status`, `issues create/get/list/status` | **raw relay body, unmodified** (`println!("{resp}")`) | `repos.rs:219`, `repos.rs:252`, `repos.rs:280`; `pr.rs:60`, `pr.rs:104`, `pr.rs:114`, `pr.rs:147`, `pr.rs:213`; `patches.rs:53`, `patches.rs:80`, `patches.rs:110`, `patches.rs:189`; `issues.rs:32`, `issues.rs:43`, `issues.rs:77`, `issues.rs:143` |
| `repos protect list` | `{repo_id, protections:[{ref, rules:[…]}], unknown_rules:[…], validation_error: string\|null}` | `protection_rules_json`, `repos.rs:165-170` |
| `repos protect set/remove` | `{event_id, accepted, message}` via `normalize_write_response` | `repos.rs:159`, `repos.rs:198` |
| `users get`, `users get --name` | `[{…kind:0 content fields…, pubkey}]`; with `--format compact`, `[{pubkey, display_name}]` | `users.rs:52-74`, `users.rs:118-145` |
| `users presence` | `[{pubkey, status, updated_at}]` | `users.rs:264-273` |
| `users set-profile`, `users set-presence` | `{event_id, accepted, message}` | `users.rs:206`, `users.rs:302` |
| `workflows list` | `[{workflow_id, content, created_at, pubkey}]` | `workflows.rs:21-31` |
| `workflows get` | one such object, or the bare literal `null` | `workflows.rs:47-58` |
| `workflows runs` | `[{event_id, kind, content, created_at, tags}]` | `workflows.rs:80-91` |
| `workflows create` | `{event_id, accepted, message, workflow_id}` | `print_create_response`, `workflows.rs:113` |
| `workflows update/delete/trigger/approve` | `{event_id, accepted, message}` | `workflows.rs:132`, `:147`, `:180`/`:189`, `:210` |
| `agents draft-create`, `draft-update` | relay WS response object plus injected `request_id`, `action`, `saved:false`, `message` | `agents.rs:33-46`, `agents.rs:76-89` |
| `agents archive`, `unarchive` | `{ok:true, event_id, action, target}` | `agents.rs:117-125`, `agents.rs:146-154` |
| `agents archived` | `{archived:[hex64…]}` | `agents.rs:312` |
| `moderation reports/restricted/audit` | **raw relay array**, unmodified | `moderation.rs:115`, `:120`, `:128` |
| `moderation ban/unban/timeout/untimeout/resolve` | `{event_id, accepted, message}` | `moderation.rs:44`, `:56`, `:74`, `:84`, `:99` |
| `mem ls` | with `--json`, `[{slug, event_id, created_at}]` (`engram::Listing`, `engram.rs:596-605`); otherwise TSV `slug\tcreated_at\tevent_id` on stdout | `mem.rs:262-272` |
| `mem get` | **raw value bytes, no trailing newline, no JSON** | `mem.rs:295-303` |
| `mem hash` | 64 hex chars + `\n` | `mem.rs:517` |
| `mem set/patch/rm` | **nothing on stdout**; a human line on stderr | `mem.rs:363`, `mem.rs:700`, `mem.rs:732` |
| `pack validate` | plain text `Valid.` / `Valid (with warnings).` on stdout, diagnostics on stderr | `pack.rs:26-45` |
| `pack inspect` | multi-line indented plain text | `pack.rs:66-149` |
| `upload file` | `serde_json::to_string_pretty(BlobDescriptor)` — the only pretty-printed JSON in the group | `upload.rs:9-10`; struct at `client.rs:10-33` |
| `media get` | raw bytes to a file or stdout | `upload.rs:21-33` |

`AGENTS.md` states reads "return sig-stripped JSON arrays". That holds for the messaging
commands (which call `client::normalize_events`, `client.rs:1307-1323`) but **not** for this
group: `grep -rn 'normalize_events' crates/buzz-cli/src/` returns hits only in
`client.rs`, `feed.rs`, and `messages.rs` — zero in any of the eleven files here. The
pass-through read commands print the relay array verbatim, and the relay serializes a full
`nostr::Event` including `sig` (`crates/buzz-relay/src/api/bridge.rs:1298`). See Debt.

#### Memory patch / diff representation

`mem patch` treats a slug as a single virtual file and consumes a **unified diff**:

- Multi-file patches are rejected by counting lines starting with `"--- "` before parsing
  (`mem.rs:600-608`); more than one header is a `Usage` error.
- The diff is parsed by `diffy::Patch::from_str` (`mem.rs:610`).
- Hunk positions are checked strictly against the *unmodified* current value: preimage
  lines (`Context` + `Delete`) must match byte-for-byte starting at the declared 1-based
  `@@ -N,M @@` line (`verify_hunks_at_declared_position`, `mem.rs:400-482`). Lines are split
  with `split_inclusive('\n')` (`mem.rs:404`) so trailing-newline state is part of the match.
- `@@ -0,0 +1,M @@` (pure insertion into an empty value) is the only accepted empty-preimage
  hunk (`mem.rs:432-434`); a no-context insertion into a non-empty value is refused
  (`mem.rs:435-443`).
- The base hash is `hex(SHA256(utf8 bytes))` of the exact value (`sha256_hex`, `mem.rs:377-381`),
  matching `printf '%s' "$value" | sha256sum` per the doc comment at `mem.rs:373-375`.
- The echoed audit trail is the **input** diff verbatim plus the resulting digest
  (`mem.rs:696-698`), not a regenerated diff.

The engram body itself is a hand-rolled, whitespace-free JSON object — `{"slug":…,"value":…}`
for memories (with `"value":null` as the tombstone) and `{"slug":"core","profile":…}` for
core (`engram.rs:189-214`) — NIP-44-v2-encrypted into the event content
(`engram.rs:455-461`). Duplicate object keys are rejected at any nesting depth
(`engram.rs:216-224`). The CLI's `Body` handling maps `slug == engram::CORE_SLUG` to
`Body::Core` and everything else to `Body::Memory` at `mem.rs:339-346` and `mem.rs:665-675`.

#### Persona-pack structures as consumed here

`pack.rs` consumes `buzz_persona::resolve::ResolvedPack` / `ResolvedPersona` only — it never
touches the on-disk schema directly. Fields read at `pack.rs:66-148`: `name`, `id`,
`version`, `personas[]`, and per persona `name`, `display_name`, `description`,
`llm_provider`, `model`, `temperature`, `max_context_tokens`, `subscribe[]`,
`triggers{mentions, keywords[], all_messages}`, `thread_replies`, `broadcast_replies`,
`mcp_servers[]` (count only), `skills[]`, `avatar`, `system_prompt`, `runtime_env_vars[]`.
`runtime_env_vars` is a *derived* closed set (`BUZZ_AGENT_MODEL`, `BUZZ_AGENT_PROVIDER`,
`GOOSE_PROVIDER`, `GOOSE_MODEL`, `GOOSE_TEMPERATURE`, `GOOSE_CONTEXT_LIMIT` —
`crates/buzz-persona/src/resolve.rs:365-392`), not arbitrary operator-supplied vars.
`pack validate` consumes `buzz_persona::validate::ValidationReport` with its
`Error`/`Warning` diagnostic enum (`pack.rs:24-36`).

#### Local file formats read or written

| Path source | Direction | Read/write site | Format |
|---|---|---|---|
| `mem patch --patch-file <path>` | read whole file | `mem.rs:578-579` (`std::fs::read_to_string`) | unified diff |
| `mem set <slug> -` / `mem patch` stdin | read, bounded to `NIP44_PLAINTEXT_MAX + 1` = 65 536 bytes | `mem.rs:322-330`, `mem.rs:582-585` | raw UTF-8 |
| `patches send --patch-file <path\|->` | read whole file | `read_file_or_stdin`, `validate.rs:180-192`, called at `patches.rs:29` | `git format-patch` output |
| `pr open/update/status --body-file <path\|->` | read whole file | `read_file_or_stdin` via `read_optional_body`, `pr.rs:8-18` | markdown |
| `pack validate/inspect <path>` | read pack directory | `pack.rs:16-23`, `pack.rs:53-60` | persona pack tree |
| `upload file --file <path>` | read whole file | `client.rs:1102-1109` | any of the allowlisted MIME types (`client.rs:64-71`) |
| `media get -o <path>` | **write** | `upload.rs:24-25` (`std::fs::write`) | raw blob bytes |

`media get`'s output path is the only file this group writes, and it does so with no
overwrite guard and no confinement (see Security).

#### Test coverage for the data model

| Behavior | Test | Site |
|---|---|---|
| slug→hash byte-exactness (empty, `abc`, `abc\n`) | `sha256_hex_empty`, `sha256_hex_abc`, `sha256_hex_handles_newline_terminated_value` | `mem.rs:847`, `:855`, `:863` |
| unified-diff strict-position semantics | `strict_position_rejects_offset_slide`, `strict_position_accepts_exact_match`, `strict_position_accepts_pure_insertion_into_empty`, `strict_position_accepts_multi_hunk_against_original`, `strict_position_handles_no_trailing_newline` | `mem.rs:938`, `:958`, `:968`, `:993`, `:1017` |
| multi-file header count | `multi_file_header_count` | `mem.rs:1035` |
| `buzz-protect` tag construction and rule limits | `protection_update_preserves_metadata_and_replaces_only_matching_pattern`, `protection_update_enforces_repository_rule_limit` | `repos.rs:423`, `repos.rs:551` |
| `repos protect list` JSON shape | `protection_list_keeps_unknown_rules_visible`, `protection_list_surfaces_malformed_rules_for_recovery` | `repos.rs:575`, `repos.rs:601` |
| 13535 tag vocabulary (`-`, `p`) | 8 tests, `archived_state2_valid_event_returns_pubkeys` … `archived_short_p_tag_dropped` | `agents.rs:578`-`agents.rs:717` |
| presence `p`-tag-vs-author subject resolution | `presence_subject_uses_p_tag` and two fallbacks | `users.rs:343`, `:349`, `:355` |

Not covered by any test in this group: the shape of every pass-through read output
(`repos get/list`, `pr *`, `patches get/list`, `issues *`), the `workflows` normalized
shapes, the `moderation` outputs, the `a`-tag coordinate string built in three places,
`upload`/`media` shapes, and `pack inspect`'s text layout. `grep -c '#\[test\]'` returns
0 for `workflows.rs`, `issues.rs`, `moderation.rs`, `pack.rs`, and `upload.rs`.
