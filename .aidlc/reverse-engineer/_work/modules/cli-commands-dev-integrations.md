## Module: buzz-cli — repo, agent, memory & moderation commands (`crates/buzz-cli/src/commands`)
### Aspect: Integrations

#### Event kinds published, per command

Every write in this group goes out as a signed Nostr event. Kind integers are owned by
`crates/buzz-core/src/kind.rs`; `buzz-sdk` re-exports that module wholesale
(`crates/buzz-sdk/src/lib.rs:22` — `pub use buzz_core::kind;`), so the builders name the
constant even where the CLI does not.

| Command | Kind | Constant (`kind.rs:LINE`) | Kind materialized at | Transport |
|---|---|---|---|---|
| `agents draft-create` / `draft-update` | 24200 | `KIND_AGENT_OBSERVER_FRAME`, `kind.rs:409` | `buzz-sdk/src/builders.rs:271` via `build_agent_observer_frame` (called `agent_management.rs:109`) | `publish_ephemeral_event` (WSS), `agents.rs:33`, `agents.rs:75` |
| `agents archive` | 9035 | `KIND_IA_ARCHIVE_REQUEST`, `kind.rs:348` | `builders.rs:1798` | `submit_event`, `agents.rs:110` |
| `agents unarchive` | 9036 | `KIND_IA_UNARCHIVE_REQUEST`, `kind.rs:350` | `builders.rs:1819` | `submit_event`, `agents.rs:140` |
| `mem set` / `patch` / `rm` | 30174 | `KIND_AGENT_ENGRAM`, `kind.rs:94` | `buzz_core::engram::build_event`, called `mem.rs:368`, `mem.rs:694`, `mem.rs:731` | `submit_engram` → `submit_event`, `mem.rs:93-108` |
| `repos create` | 30617 | `KIND_GIT_REPO_ANNOUNCEMENT`, `kind.rs:545` | `build_repo_announcement`, `repos.rs:216` | `submit_event`, `repos.rs:226` |
| `repos protect set` / `remove` | 30617 | same | `build_repo_announcement_with_tags`, `repos.rs:136` | `submit_repo_update`, `repos.rs:195-200` |
| `patches send` | 1617 | `KIND_GIT_PATCH`, `kind.rs:549` | `build_git_patch`, `patches.rs:50` | `submit_event`, `patches.rs:52` |
| `pr open` | 1618 | `KIND_GIT_PULL_REQUEST`, `kind.rs:551` | `build_git_pull_request`, `pr.rs:58` (kind set `builders.rs:1390`) | `submit_event`, `pr.rs:60` |
| `pr update` | 1619 | `KIND_GIT_PR_UPDATE`, `kind.rs:553` | `build_git_pr_update`, `pr.rs:100` | `submit_event`, `pr.rs:102` |
| `issues create` | 1621 | `KIND_GIT_ISSUE`, `kind.rs:555` | `build_git_issue`, `issues.rs:29` | `submit_event`, `issues.rs:31` |
| `patches status` / `pr status` / `issues status` | 1630 / 1631 / 1632 / 1633 | `KIND_GIT_STATUS_{OPEN,MERGED,CLOSED,DRAFT}`, `kind.rs:557-563` | `build_git_status`, `patches.rs:183`, `pr.rs:209`, `issues.rs:140`; word→kind map in `patches::parse_status`, `patches.rs:194-205` | `submit_event` |
| `users set-profile` | 0 | `KIND_PROFILE`, `kind.rs:8` | `build_profile`, `users.rs:200` | `submit_event`, `users.rs:211` |
| `users set-presence` | 20001 | `KIND_PRESENCE_UPDATE`, `kind.rs:403` | `build_presence_update`, `users.rs:299` (kind `builders.rs:1580`) | `publish_ephemeral_event` (WSS), `users.rs:302` |
| `workflows create` / `update` | 30620 | `KIND_WORKFLOW_DEF`, `kind.rs:382` | `build_workflow_def` / `build_workflow_update`, `workflows.rs:107`, `workflows.rs:129` | `submit_event` |
| `workflows delete` | 5 | `KIND_DELETION`, `kind.rs:56` | `build_workflow_delete`, `workflows.rs:144`; `a`-tag value is `"{KIND_WORKFLOW_DEF}:{pk}:{id}"` (`builders.rs:1505`, kind at `builders.rs:1507`) | `submit_event` |
| `workflows trigger` | 46020 | `KIND_WORKFLOW_TRIGGER`, `kind.rs:498` | **two paths** — `build_workflow_trigger` (`workflows.rs:184`) or, with `--inputs`, a hand-rolled `EventBuilder` naming `buzz_sdk::kind::KIND_WORKFLOW_TRIGGER` (`workflows.rs:175-178`) | `submit_event` |
| `workflows approve` | 46030 / 46031 | `KIND_APPROVAL_GRANT` / `KIND_APPROVAL_DENY`, `kind.rs:500`, `kind.rs:502` | `build_workflow_approval`, `workflows.rs:206` | `submit_event` |
| `moderation ban/unban/timeout/untimeout/resolve` | 9040-9044 | `KIND_MODERATION_*`, `kind.rs:298`, `:300`, `:303`, `:305`, `:310` | `build_moderation_*`, `moderation.rs:42`, `:53`, `:71`, `:81`, `:97` | `submit_event`, which routes 9040-9044 through the non-idempotent policy (`client.rs:863-870`) |

#### Event kinds queried, and the bare-literal audit

The task asked specifically for bare integers used in place of the constant. `grep -n '"kinds"'`
across the eleven files returns 20 filter sites; only **five** name a constant.

| Site | Filter kinds | Constant named? |
|---|---|---|
| `mem.rs:151`, `mem.rs:198` | `[KIND_AGENT_ENGRAM]` | yes |
| `agents.rs:285` | `[KIND_IA_ARCHIVED_LIST]` | yes |
| `repos.rs:21` | `[KIND_GIT_REPO_ANNOUNCEMENT]` | yes |
| `agents.rs:180` | `[0]` | no — bare literal for kind:0 (`KIND_PROFILE`, `kind.rs:8`) |
| `repos.rs:240`, `repos.rs:271` | `[30617]` | **no** — and this is the same file that imports the constant at `repos.rs:3` and uses it at `repos.rs:21` |
| `users.rs:42`, `users.rs:91`, `users.rs:223` | `[0]` | no |
| `users.rs:258` | `[40902]` | no (`KIND_PRESENCE_SNAPSHOT`, `kind.rs:443`) |
| `pr.rs:110`, `pr.rs:131` | `[1618]` | no (`kind.rs:551`) |
| `patches.rs:76`, `patches.rs:96` | `[1617]` | no (`kind.rs:549`) |
| `issues.rs:39`, `issues.rs:60` | `[1621]` | no (`kind.rs:555`) |
| `workflows.rs:16`, `workflows.rs:41` | `[30620]` | no (`kind.rs:382`) |
| `workflows.rs:74` | `[46001, 46002, 46003]` | no (`kind.rs:504-508`) |

A second class of bare literal is the NIP-34 `a`-tag address, hardcoded as
`format!("30617:{repo_owner}:{repo_id}")` in three files — `pr.rs:129`, `patches.rs:94`,
`issues.rs:58` — where the SDK's own `GitRepoCoord::to_a_tag_value` already exists
(`builders.rs:976`) and is used on the write side (`builders.rs:1024`, `:1097`, `:1240`,
`:1352`, `:1431`). And `repos.rs:411` builds
`Kind::Custom(30617)` in a test helper.

The `repos.rs` split (constant at `:21`, literal at `:240` and `:271`) is the clearest
internal inconsistency in the group: one file, one kind, two spellings.

#### HTTP paths reached, and one endpoint the docs do not name

`AGENTS.md:122` states the relay's HTTP surface is "NIP-11/NIP-05 metadata, `POST /events`,
`POST /query`, `POST /count`, workflow webhooks at `/hooks/{id}`, Blossom media, git smart
HTTP, git policy hooks, and health probes."

The `/moderation/*` triple is **not** on that list, and three commands here depend on it.
Verified on both sides:

| Path | CLI call site | Relay route | Relay handler |
|---|---|---|---|
| `GET /moderation/reports?limit=N[&status=S]` | `moderation.rs:110-114`, via `client.get_authed` (`client.rs:836-853`) | `crates/buzz-relay/src/router.rs:113` | `api/bridge.rs:2091-2115` |
| `GET /moderation/restricted` | `moderation.rs:120` | `router.rs:116` | `api/bridge.rs:2135-2145` |
| `GET /moderation/audit?limit=N` | `moderation.rs:126-128` | `router.rs:114` | `api/bridge.rs:2117-2133` |

`moderation.rs:8-13` documents and defends the choice ("reports and audit rows are
structured queue rows, not public nostr events — serving them over a REQ filter would mean
synthesizing fake events"). So this is a deliberate, argued exception that `AGENTS.md` has
not caught up with — a code-vs-docs contradiction with the code side carrying the better
rationale. Both endpoints keep the host-derived community boundary `AGENTS.md:122` promises:
`authorize_moderation_read` binds the tenant from the `Host` header before any lookup
(`api/bridge.rs:2032-2043`).

Everything else in the group stays inside the documented surface: `POST /query`
(`client.rs:767-771`), `POST /events` (`client.rs:863-870`), NIP-11 `GET /`
(`client.rs:753-765`, reached only from `agents.rs:272`), Blossom `PUT /upload` with legacy
`PUT /media/upload` fallback (`client.rs:1139`, `client.rs:1195`) and `GET /media/<sha256>`
(`client.rs:1230`), and WSS for ephemeral kinds (`client.rs:1073-1096`).

`POST /count` is implemented (`client.rs:803`) but reached by nothing here —
`grep -rn '\.count(' ` across the eleven files returns zero matches.

#### NIPs implemented or relied on

| NIP | Where, in this group |
|---|---|
| NIP-01 | event shape everywhere; `ev.verify()` before trusting a fetched event, `mem.rs:162-164`, `agents.rs:364-368` |
| NIP-11 | `agents archived` reads the relay info document for `self`, `agents.rs:271-284`; `normalize_relay_self_hex` guards it, `agents.rs:250-256` |
| NIP-33 | addressable `d`-tag replaceables for 30174/30617/30620; LWW-dominance detection at `mem.rs:105-108` and `repos.rs:187-192` |
| NIP-34 | git patches/PRs/issues/status — `patches.rs`, `pr.rs`, `issues.rs`, `repos.rs`; the `p`-tag-the-owner rule is spelled out at `patches.rs:152-155` and mirrored at `issues.rs:118-121`, `pr.rs:185-187` |
| NIP-42 | WSS auth for ephemeral publishes, delegated to `buzz_ws_client::publish_event`, `client.rs:1080-1082` |
| NIP-43 | moderation commands are modeled on the 9030-series relay-admin pattern, `moderation.rs:4-7` |
| NIP-44 v2 | engram body encryption; conversation key at `mem.rs:147`, plaintext cap `NIP44_PLAINTEXT_MAX = 65_535` (`buzz-core/src/engram.rs:28`) enforced at `mem.rs:333-338` and `mem.rs:650-656` |
| NIP-50 | `users get --name` sends a `search` filter, `users.rs:89-94`; the doc comment at `users.rs:80` notes it degrades to `[]` on relays without FTS |
| NIP-70 | `-` protected-event marker required exactly once on the 13535 snapshot, `agents.rs:337-358` |
| NIP-98 | every `/query`, `/events` and `/moderation/*` request is signed, `client.rs:769`, `client.rs:841`; NIP-98 is re-signed per retry attempt so the relay's replay guard stays happy (`client.rs:891-893`) |
| NIP-AE | the whole `mem` surface, `mem.rs:1-15` |
| NIP-IA | archive/unarchive + archived-identities snapshot, `agents.rs:93-156`, `agents.rs:259-317` |
| NIP-OA | owner-of-agent attestation; owner pubkey read from slot 1 of the `auth` tag (`client.rs:576-583`), consumed at `mem.rs:36-43` and `agents.rs:160-166` |

`grep -n 'NIP-' ` across the eleven files returns zero hits for NIP-05, NIP-29 or NIP-17 —
no command here is channel-scoped by `h` tag except `workflows list` (`workflows.rs:17`),
`workflows create`/`update` (`builders.rs:1471`, `:1489`) and `pr open --channel`
(`builders.rs:1373`).

#### Relationship to the relay's git surface

There is **no direct coupling** — the CLI never speaks git wire protocol. The connection is
data-mediated:

1. `repos create` publishes the 30617 announcement carrying `clone` URLs
   (`repos.rs:216-224`), which is what a client needs to find the smart-HTTP endpoint the
   relay serves at `/git/{owner}/{repo}/info/refs`, `/git-upload-pack`, `/git-receive-pack`
   (`crates/buzz-relay/src/api/git/transport.rs:1760-1762`).
2. `repos protect set/remove` writes `buzz-protect` tags onto that same event
   (`build_protection_tag`, `repos.rs:64-90`). The relay's git policy hook reads them back:
   `parse_protection_tags` + `evaluate_push` at
   `crates/buzz-relay/src/api/git/policy.rs:45` and `:285`, served at
   `POST /internal/git/policy` (`api/git/mod.rs:62`). `buzz_core::git_perms` is the shared
   contract — `grep -rln 'git_perms' crates/` returns exactly three files:
   `buzz-cli/src/commands/repos.rs`, `buzz-core/src/lib.rs`, and that policy module. So
   `repos.rs` is the **write** end of a rule the relay is the **enforce** end of, and the two
   agree only because both call into `buzz-core`.
3. `git-sign-nostr` and `git-credential-nostr` are **not** referenced from `buzz-cli` at all
   — `grep -rn 'git-sign-nostr\|git_sign_nostr\|git-credential-nostr\|git_credential_nostr'
   crates/buzz-cli/` returns zero matches. They are peer tools invoked by `git` itself:
   `git-sign-nostr` as a `gpg.x509.program` producing BIP-340 signatures over commits
   (`crates/git-sign-nostr/src/lib.rs:1-13`), `git-credential-nostr` as a credential helper
   minting kind:27235 NIP-98 events for push/fetch
   (`crates/git-credential-nostr/src/lib.rs:1-6`). The only place their output surfaces in
   this group is `patches send --commit-pgp-sig`, which the CLI passes through untouched
   (`patches.rs:41`).

#### `buzz-workflow` as reached from `workflows.rs`

`buzz-workflow` is **not a declared dependency** of `buzz-cli` — it is absent from
`crates/buzz-cli/Cargo.toml`. The integration is therefore entirely by document:

- `workflows create` / `update` read a YAML blob (`read_or_stdin`, `workflows.rs:104`,
  `workflows.rs:127`) and publish it verbatim as the kind:30620 content. **Nothing parses or
  validates the YAML client-side.** The only gate is the SDK's byte cap
  (`check_content(yaml, 64 * 1024)`, `builders.rs:1468`, `:1486`). A workflow with a
  malformed `if:` expression, an unknown step type, or a syntactically invalid document is
  accepted by `buzz create` and only fails later, relay-side.
- The `if:` conditions inside that YAML are evaluated by `evalexpr` in the relay's executor
  (`crates/buzz-workflow/src/executor.rs:15`, `:203-232`, `:317-318`). The CLI has no view
  of the variable namespace, the underscore-not-dot naming rule
  (`executor.rs:205-216`), or the `str_contains`/`str_starts_with`/`str_ends_with` helpers
  registered at `executor.rs:232`. So `buzz workflows create` cannot tell an operator that a
  condition references an undefined variable.
- The webhook side (`POST /hooks/{id}`, `crates/buzz-relay/src/router.rs:120`, handler
  `api/bridge.rs:1780-1795`) is **not reachable from any command in this group** —
  `grep -n '/hooks' ` across the eleven files returns zero matches. `workflows trigger`
  reaches the same executor through the kind:46020 event path instead
  (`workflows.rs:156-190`).
- `workflows runs` queries 46001/46002/46003 and its own doc comment
  (`workflows.rs:60-64`) says these events are never emitted — run history lives in the
  `workflow_runs` table. The command is wired to a surface that does not exist yet and will
  return `[]`.

#### `buzz-persona` as consumed by `pack.rs`

`buzz-persona` (declared `Cargo.toml:70-71`) is reached from **exactly four call sites, all
in `pack.rs`** — `grep -rn 'buzz_persona' crates/buzz-cli/src/` returns
`pack.rs:24`, `:28`, `:31`, `:62`:

| Call | Site | What it pulls in |
|---|---|---|
| `validate::validate_pack(pack_dir)` | `pack.rs:24` | full pack load + diagnostics, returned as `Vec<ValidationDiagnostic>` |
| `validate::ValidationDiagnostic::{Error,Warning}` | `pack.rs:28`, `pack.rs:31` | the two-level diagnostic enum, printed to stderr |
| `resolve::resolve_pack(pack_dir)` | `pack.rs:62` | `ResolvedPack` — post-merge, post-split effective config |

`pack.rs` reads a wide slice of `ResolvedPersona` (`runtime_env_vars` field declared at
`buzz-persona/src/resolve.rs:64`, projected by `resolve.rs:365`):
`name`, `display_name`, `description`, `avatar`, `llm_provider`, `model`, `temperature`,
`max_context_tokens`, `subscribe`, `triggers`, `thread_replies`, `broadcast_replies`,
`mcp_servers` (count only, `pack.rs:116`), `skills`, `system_prompt`, `runtime_env_vars`.
`pack_instructions`, `hooks` and `version`-per-persona are resolved by the crate but never
printed — the two `pack` commands are the only relay-free commands in the whole CLI
(`pack.rs:3` — "No relay connection needed"), and `lib.rs:1737-1740` short-circuits them
before a `BuzzClient` is built.

#### Filesystem interaction

`grep -rn 'std::fs::\|Path::new\|canonicalize'` across the eleven files returns four sites:

| Site | Operation | Bound? | Confined to a root? |
|---|---|---|---|
| `mem.rs:581` | `std::fs::read_to_string(patch_path)` for `mem patch --patch-file` | **no** — the `limit` computed at `mem.rs:577` is applied only in the stdin arm (`mem.rs:585`) | no |
| `upload.rs:23` | `std::fs::write(path, &bytes)` for `media get -o` | n/a | no |
| `pack.rs:16`, `pack.rs:53` | `Path::new(path)` + `exists()`/`is_dir()` for the pack root | n/a | no |
| (indirect) `validate.rs:189-192` | `read_file_or_stdin` → `std::fs::read_to_string`, used by `patches send --patch-file` (`patches.rs:26`) and `pr open/update/status --body-file` (`pr.rs:15`) | **no** | no |
| (indirect) `client.rs:1101-1110` | `metadata` + `read` for `upload file`; `is_file()` check and MIME/size gates at `client.rs:1103-1131` | yes — `MAX_IMAGE_BYTES` 50 MiB (`client.rs:73`) / `MAX_VIDEO_BYTES` 500 MiB (`client.rs:76`), selected at `client.rs:1120-1125` | no |

Memory "files" are virtual — a slug is a single virtual file (`mem.rs:610-613`) held in a
30174 event, never on disk. No command in this group writes a dotfile, cache, or config;
`dirs` (`Cargo.toml:73-75`) is used by `channels.rs`, not here.

#### Subprocesses

**None.** `grep -rn 'Command::new\|std::process\|tokio::process'` across the eleven files
returns zero matches. `pack inspect` prints MCP server `command`/`args` values but never
executes them (`pack.rs:116` prints only the count), and `buzz-persona` deliberately does
not resolve or run hook paths (`buzz-persona/src/resolve.rs:340-347` — "hooks are not
executed in this PR"). The only subprocess anywhere near this feature area is `git`'s own
invocation
of `git-credential-nostr`, which shells out to `git config`
(`crates/git-credential-nostr/src/lib.rs:16-19`) — outside this group.

#### Workspace crates and third-party dependencies used from this group

| Crate | Reached from | Notes |
|---|---|---|
| `buzz-core` | `mem.rs:22-25` (`engram`, `kind`), `agents.rs:1` (`kind`), `repos.rs:1-4` (`git_perms`, `kind`), `agent_management.rs:2` (`observer`) | the shared-contract crate |
| `buzz-sdk` | all eight write-bearing modules | typed builders; `buzz_sdk::kind` re-export used once, `workflows.rs:176` |
| `buzz-persona` | `pack.rs` only (4 sites) | **reached only from this group** |
| `buzz-ws-client` | indirectly, via `client.rs:1080` from `agents.rs:33`, `agents.rs:75`, `users.rs:302` | ephemeral publish |
| `nostr` | `mem.rs:26`, `agents.rs:3`, `repos.rs:5`, `moderation.rs:17`, `workflows.rs:174` | `Event`, `Keys`, `PublicKey`, `Tag`, `Timestamp`, `EventBuilder` |
| `diffy` 0.5 | `mem.rs` only — 31 occurrences; `grep -rn 'diffy' crates/buzz-cli/src/` outside `mem.rs` returns zero | **declared dependency reached exclusively from this group**, `Cargo.toml:52-53` |
| `sha2` | `mem.rs:20` (base-hash), `workflows.rs:1` (approval-token hash) — also `client.rs:7` | shared |
| `hex` | `mem.rs:381`, `workflows.rs:204` | shared with `client.rs` |
| `uuid` | `workflows.rs:105`, via `validate::parse_uuid` | shared |
| `chrono` | not from here — only `agent_management.rs:97`, which `agents.rs:6` pulls in transitively | so `agents draft-*` is the sole reason `chrono` is a dependency (`Cargo.toml:41-42`) |
| `serde_json` | everywhere | filters, responses, output |

Notably absent as dependencies given the surface: `buzz-workflow` (YAML/evalexpr — see
above), and any YAML parser at all (`grep -n 'serde_yaml\|yaml' crates/buzz-cli/Cargo.toml`
returns zero matches), which is why `workflows create` cannot pre-validate.

#### Logic duplicated across these eleven files rather than shared

| Duplicated logic | Copies | Comment |
|---|---|---|
| NIP-34 `a`-tag address string | `pr.rs:129`, `patches.rs:94`, `issues.rs:58` | three identical `format!("30617:{repo_owner}:{repo_id}")`; `GitRepoCoord::to_a_tag_value` (`builders.rs:1353`) already does this on the write side |
| `--repo-owner`/`--repo-id` must-pair match block | `pr.rs:169-183`, `patches.rs:136-150`, `issues.rs:99-113` | byte-identical including the error string |
| owner-first recipient list + dedupe | `pr.rs:185-196`, `patches.rs:152-165`, `issues.rs:118-127` | three copies of the same `p`-tag defaulting rule; each carries its own comment restating NIP-34's intent |
| relay "duplicate" → conflict detection | `mem.rs:93-108` (`submit_engram`), `repos.rs:173-192` (`validate_write_response`) | same two-clause rule (`== "duplicate"` ‖ `starts_with("duplicate:")`), two implementations, opposite clause order, different error text. Only the `repos.rs` copy is tested (`duplicate_write_response_is_a_conflict`, `repos.rs:619`) |
| `parse_events` JSON→`Vec<Event>` | `mem.rs:114-134`, `repos.rs:11-14` | **not** equivalent: `mem.rs` skips undeserializable entries by design (`mem.rs:120-128`), `repos.rs` fails the whole response. Same name, different failure semantics, same crate |
| `--format compact` projection | `users.rs:63-72`, `users.rs:134-143` | identical `{pubkey, display_name}` reduction inside one file |
| relay-response JSON reparse + field injection | `agents.rs:35-47`, `agents.rs:77-89` | two ~13-line copies differing only in which draft was sent |
| `pack` path precondition (`exists` / `is_dir`) | `pack.rs:16-22`, `pack.rs:54-59` | identical six-line preamble |

None of these have a shared helper, though `client.rs` already hosts the group's other
shared normalizers (`normalize_write_response` at `client.rs:1420`,
`print_create_response` at `client.rs:1401`).

#### Where I am uncertain

- I did not run the relay, so the `/moderation/*` request/response bodies are inferred from
  the handler signatures at `api/bridge.rs:2091-2145`, not observed.
- Whether the relay validates workflow YAML on ingest of kind:30620 (as opposed to at
  execution time) is not something I traced; I only established that the CLI does not.
- `workflows runs` returning `[]` is what `workflows.rs:60-64` asserts about the relay. I
  confirmed no 46001-46003 emission from `buzz-workflow`'s executor by absence of a publish
  call in the files I read, but I did not exhaustively grep the relay for every emission
  path.
