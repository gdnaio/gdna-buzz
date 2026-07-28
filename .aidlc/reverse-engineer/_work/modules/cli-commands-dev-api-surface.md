## Module: buzz-cli — repo, agent, memory & moderation commands (`crates/buzz-cli/src/commands`)
### Aspect: API Surface

#### Dispatch entry points

Eleven modules, nine `dispatch` functions plus two directly-called free functions. All are
reached from the single `match cli.command` in `lib.rs:1769-1793`.

| Entry | Signature site | Called from | Accepts `&OutputFormat`? |
|---|---|---|---|
| `agents::dispatch` | `agents.rs:12` | `lib.rs:1771` | no |
| `users::dispatch` | `users.rs:307-311` | `lib.rs:1778` | **yes**, and honors it |
| `workflows::dispatch` | `workflows.rs:214` | `lib.rs:1779` | no |
| `repos::dispatch` | `repos.rs:349` | `lib.rs:1783` | no |
| `patches::dispatch` | `patches.rs:206` | `lib.rs:1784` | no |
| `issues::dispatch` | `issues.rs:147` | `lib.rs:1785` | no |
| `pr::dispatch` | `pr.rs:216` | `lib.rs:1786` | no |
| `upload::dispatch_media` | `upload.rs:17` | `lib.rs:1787` | no |
| `upload::dispatch` | `upload.rs:4` | `lib.rs:1788` | no |
| `mem::dispatch` | `mem.rs:737` | `lib.rs:1789` | no (has its own `--json` on `ls`) |
| `moderation::dispatch` | `moderation.rs:133-137` | `lib.rs:1790` | **yes, but ignores it** — parameter is bound as `_format` at `moderation.rs:136` |
| `pack::cmd_validate` / `cmd_inspect` | `pack.rs:15`, `pack.rs:52` | `lib.rs:1737-1740`, short-circuited before any client is constructed | no |

`--format` is a global flag (`lib.rs:90-92`, default `json`). Of the eleven modules here,
exactly one (`users.rs`) reads it. `moderation.rs` takes it and discards it. The other nine
never see it — so `buzz --format compact repos list`, `… pr list`, `… mem ls`,
`… workflows list` and `… moderation reports` all silently produce full JSON. This
contradicts nothing in `AGENTS.md` (which only documents flag *position*), but the CLI's
own `--format` help text ("Output format… 'compact' (reduced fields)", `lib.rs:90`) promises
behavior these commands do not implement.

#### Subcommand handler table

| Command | Handler | Relay call | Kind / path | Output |
|---|---|---|---|---|
| `agents draft-create` | inline, `agents.rs:16-48` | `publish_ephemeral_event` (WSS) | 24200 observer frame | relay OK object + `request_id`/`action`/`saved:false`/`message` |
| `agents draft-update` | inline, `agents.rs:50-91` | `publish_ephemeral_event` (WSS) | 24200 | same |
| `agents archive` | inline, `agents.rs:93-127` | `query` (kind:0 probe) then `submit_event` | 9035 | `{ok, event_id, action, target}` |
| `agents unarchive` | inline, `agents.rs:129-156` | `query` then `submit_event` | 9036 | `{ok, event_id, action, target}` |
| `agents archived` | `cmd_archived`, `agents.rs:310` | `get_public("/")` + `query` | NIP-11 doc, then 13535 | `{archived:[…]}` |
| `users get` | `cmd_get_users`, `users.rs:12` | `query` | kind:0 | profile array (format-aware) |
| `users set-profile` | `cmd_set_profile`, `users.rs:150` | `query` then `submit_event` | kind:0 | write response |
| `users presence` | `cmd_get_presence`, `users.rs:247` | `query` | 40902 | `[{pubkey,status,updated_at}]` |
| `users set-presence` | `cmd_set_presence`, `users.rs:298` | `publish_ephemeral_event` (WSS) | 20001 | write response |
| `workflows list` | `cmd_list_workflows`, `workflows.rs:13` | `query` | 30620 by `#h` | normalized array |
| `workflows get` | `cmd_get_workflow`, `workflows.rs:38` | `query` | 30620 by `#d` | object or `null` |
| `workflows create` | `cmd_create_workflow`, `workflows.rs:98` | `submit_event` | 30620 | write response + `workflow_id` |
| `workflows update` | `cmd_update_workflow`, `workflows.rs:119` | `submit_event` | 30620 | write response |
| `workflows delete` | `cmd_delete_workflow`, `workflows.rs:139` | `submit_event` | kind:5 with `a` tag | write response |
| `workflows trigger` | `cmd_trigger_workflow`, `workflows.rs:156` | `submit_event` | 46020 | write response |
| `workflows runs` | `cmd_get_workflow_runs`, `workflows.rs:66` | `query` | 46001/46002/46003 | normalized array (see below) |
| `workflows approve` | `cmd_approve_step`, `workflows.rs:193` | `submit_event` | 46030 / 46031 | write response |
| `repos create` | `cmd_create_repo`, `repos.rs:202` | `submit_event` | 30617 | raw relay body |
| `repos get` | `cmd_get_repo`, `repos.rs:232` | `query` | 30617 | raw relay array |
| `repos list` | `cmd_list_repos`, `repos.rs:256` | `query` | 30617 | raw relay array |
| `repos protect list` | `cmd_protect_list`, `repos.rs:295` | `query` | 30617 (own) | derived protection JSON |
| `repos protect set` | `cmd_protect_set`, `repos.rs:301` | `query` then `submit_event` | 30617 | normalized write response |
| `repos protect remove` | `cmd_protect_remove`, `repos.rs:327` | `query` then `submit_event` | 30617 | normalized write response |
| `patches send` | `cmd_send_patch`, `patches.rs:9` | `submit_event` | 1617 | raw relay body |
| `patches get` | `cmd_get_patch`, `patches.rs:73` | `query` | 1617 by `ids` | raw relay array |
| `patches list` | `cmd_list_patches`, `patches.rs:84` | `query` | 1617 by `#a` | raw relay array |
| `patches status` | `cmd_patch_status`, `patches.rs:114` | `submit_event` | 1630-1633 | raw relay body |
| `pr open` | `cmd_open_pr`, `pr.rs:20` | `submit_event` | 1618 | raw relay body |
| `pr update` | `cmd_update_pr`, `pr.rs:66` | `submit_event` | 1619 | raw relay body |
| `pr get` | `cmd_get_pr`, `pr.rs:107` | `query` | 1618 by `ids` | raw relay array |
| `pr list` | `cmd_list_prs`, `pr.rs:118` | `query` | 1618 by `#a` | raw relay array |
| `pr status` | `cmd_pr_status`, `pr.rs:152` | `submit_event` | 1630-1633 | raw relay body |
| `issues create` | `cmd_create_issue`, `issues.rs:6` | `submit_event` | 1621 | raw relay body |
| `issues get` | `cmd_get_issue`, `issues.rs:36` | `query` | 1621 by `ids` | raw relay array |
| `issues list` | `cmd_list_issues`, `issues.rs:47` | `query` | 1621 by `#a` | raw relay array |
| `issues status` | `cmd_issue_status`, `issues.rs:81` | `submit_event` | 1630-1633 | raw relay body |
| `moderation ban` | `cmd_ban`, `moderation.rs:34` | `submit_event` (moderation retry policy) | 9040 | normalized write response |
| `moderation unban` | `cmd_unban`, `moderation.rs:51` | `submit_event` | 9041 | normalized write response |
| `moderation timeout` | `cmd_timeout`, `moderation.rs:61` | `submit_event` | 9042 | normalized write response |
| `moderation untimeout` | `cmd_untimeout`, `moderation.rs:79` | `submit_event` | 9043 | normalized write response |
| `moderation resolve` | `cmd_resolve`, `moderation.rs:89` | `submit_event` | 9044 | normalized write response |
| `moderation reports` | `cmd_reports`, `moderation.rs:105` | `get_authed` | `GET /moderation/reports?limit=…[&status=…]` | raw relay array |
| `moderation restricted` | `cmd_restricted`, `moderation.rs:119` | `get_authed` | `GET /moderation/restricted` | raw relay array |
| `moderation audit` | `cmd_audit`, `moderation.rs:125` | `get_authed` | `GET /moderation/audit?limit=…` | raw relay array |
| `mem ls` | `cmd_ls`, `mem.rs:189` | `query` | 30174 | TSV or `--json` array |
| `mem get` | `cmd_get`, `mem.rs:277` | `query` | 30174 | raw value bytes |
| `mem hash` | `cmd_hash`, `mem.rs:508` | `query` | 30174 | 64 hex + `\n` |
| `mem set` | `cmd_set`, `mem.rs:314` | `query` (head) then `submit_event` | 30174 | stderr line only |
| `mem patch` | `cmd_patch`, `mem.rs:538` | `query` (head) then `submit_event` | 30174 | stderr diff + digest |
| `mem rm` | `cmd_rm`, `mem.rs:706` | `query` (head) then `submit_event` | 30174 tombstone | stderr line only |
| `upload file` | inline, `upload.rs:6-13` | `upload_file` | `PUT /upload`, falling back to `PUT /media/upload` | pretty-printed `BlobDescriptor` |
| `media get` | inline, `upload.rs:19-33` | `download_media` | `GET /media/<sha256[.ext]>` | raw bytes |
| `pack validate` | `cmd_validate`, `pack.rs:15` | none — local | — | plain text |
| `pack inspect` | `cmd_inspect`, `pack.rs:52` | none — local | — | plain text |

#### HTTP surface reached

| Path | Method | Reached by | Auth |
|---|---|---|---|
| `/query` | POST | every read command in this group, via `BuzzClient::query` (`client.rs:767-771`) | NIP-98 + optional `x-auth-tag` |
| `/events` | POST | every stored-event write, via `submit_event` (`client.rs:863-870`) | NIP-98 + optional `x-auth-tag` |
| `/` (NIP-11 relay info) | GET | `agents archived`, via `get_public` (`client.rs:753-765`) | **none** — deliberately unauthenticated |
| `/moderation/reports` | GET | `moderation reports` (`moderation.rs:107-112`) | NIP-98 via `get_authed`, plus relay-side mod authz |
| `/moderation/restricted` | GET | `moderation restricted` (`moderation.rs:120`) | same |
| `/moderation/audit` | GET | `moderation audit` (`moderation.rs:126-127`) | same |
| `/upload` then `/media/upload` | PUT | `upload file` (`client.rs:1139`, `client.rs:1195`) | Blossom BUD-02 auth |
| `/media/<sha256[.ext]>` | GET | `media get` (`client.rs:1230`) | Blossom BUD-01 `t=get` |
| WSS `/` | EVENT | `agents draft-*`, `users set-presence` (`client.rs:1073-1096`) | NIP-42 |

`AGENTS.md` names `POST /events`, `POST /query`, `POST /count`, `/hooks/{id}`, Blossom
media, git smart HTTP, git policy hooks, NIP-11/NIP-05, and health probes as the whole HTTP
surface. The `/moderation/*` triple is **not** in that list, yet three commands here depend
on it. The `moderation.rs` module doc (`moderation.rs:9-15`) explains and defends the
choice; `AGENTS.md` has not been updated to match. Routes confirmed at
`crates/buzz-relay/src/router.rs:113-116`. Conversely, `POST /count` is defined
(`client.rs:803`) but marked `#[allow(dead_code)]` and is called by no command in this
group — `grep -n 'client.count\|\.count(' ` across the eleven files returns zero matches.

#### Undocumented / semi-public items

- `agents::fetch_archived_snapshot` is `pub(crate)` (`agents.rs:270`) and is the one item in
  this group consumed by a *sibling* module: `channels.rs:11` imports it for
  `--template` roster resolution. Its doc comment (`agents.rs:255-269`) is the only
  description of the fail-open-vs-fail-closed split between the two callers.
- `patches::parse_status` is `pub(crate)` (`patches.rs:194`) and is called cross-module by
  `pr.rs:169` and `issues.rs:83`. It is the single source of the status-word vocabulary.
- `repos.rs` exposes `cmd_create_repo`, `cmd_get_repo`, `cmd_list_repos` as `pub`
  (`repos.rs:202`, `:232`, `:256`) while the three `protect` handlers are private
  (`repos.rs:295`, `:301`, `:327`) — inconsistent visibility with no external caller for
  either set beyond the local `dispatch`.
- `users.rs` and `mem.rs` mark all their command functions `pub` even though only
  `dispatch` calls them; `mem.rs`'s are documented, `users.rs`'s partly so
  (`cmd_get_users` at `users.rs:7-11` yes, `cmd_set_profile` at `users.rs:150` no doc).

#### Output contract deviations within the group

Three commands break the "JSON out" contract stated in `crates/buzz-cli/README.md:3`
("JSON in, JSON out") and `README.md:159`:

1. `mem get` writes raw value bytes with no newline (`mem.rs:295-303`) — documented as
   intentional at `mem.rs:296` so it round-trips with `mem set <slug> -`.
2. `mem ls` without `--json` emits TSV (`mem.rs:268-270`) and an empty listing prints
   `(no memories besides core)` to **stderr** (`mem.rs:266`) while stdout stays empty.
3. `pack validate` / `pack inspect` emit only human-readable text (`pack.rs:26-45`,
   `pack.rs:66-149`) with no JSON mode at all.

And one formatting outlier: `upload file` is the only command in the whole group that
pretty-prints (`upload.rs:9-10`); everything else emits single-line JSON.

#### Exit-code classes produced

Mapping is `error::exit_code` (`error.rs:89-107`); the categories printed on stderr come
from `error::print_error` (`error.rs:112-137`).

| Code | `CliError` variant | Produced in this group by |
|---|---|---|
| 0 | — | every success path |
| 1 | `Usage` | all `validate_hex64` / `validate_repo_id` / `validate_uuid` failures; mutual-exclusion errors (`users.rs:17-19`, `mem.rs:57-60`, `mem.rs:551-555`, `pr.rs:9-11`); `parse_status` (`patches.rs:202-204`); `parse_committer` (`patches.rs:66-68`); `--inputs` not-an-object (`workflows.rs:165-169`); empty-stdin guards (`mem.rs:331-338`, `mem.rs:586-591`); `pack validate` failure (`pack.rs:40`); `pack` path errors (`pack.rs:17-22`, `pack.rs:54-59`); patch-application failures (`mem.rs:611-628`); size-cap breaches (`mem.rs:326-329`, `mem.rs:630-636`) |
| 1 | `NotFound` | `mem get`/`hash`/`patch` on absent or tombstoned slug (`mem.rs:294`, `mem.rs:296`, `mem.rs:487-491`); `repos protect *` when the repo is not the caller's (`repos.rs:290-292`); `repos protect remove` for an absent rule (`repos.rs:334-338`) |
| 2 | `Relay` (non-401/403), `Network`, `DeliveryUnknown` | any `/query`, `/events`, `/moderation/*` failure surfaced by `client.rs` |
| 3 | `Auth`, `Key`, `Relay{401,403}` | `require_owner` when `BUZZ_AUTH_TAG` is absent (`agents.rs:161-165`); relay 403 on any command |
| 4 | `Other` | build/sign failures (`mem.rs:356`, `mem.rs:681`, `mem.rs:725`); relay-response parse failures (`mem.rs:95-96`, `agents.rs:36`, `repos.rs:174-175`); all 13535 trust failures (`agents.rs:322-372`); `repos` invalid-existing-rules refusal (`repos.rs:126-131`); `pack inspect` resolve failure (`pack.rs:62-63`) |
| 5 | `Conflict` | **`mem set`/`patch`/`rm`** via `submit_engram` (`mem.rs:100-104`); **`mem patch`** base-hash mismatch (`mem.rs:594-598`); **`repos protect set`/`remove`** via `validate_write_response` (`repos.rs:154-158`) |

`AGENTS.md` names `mem set`/`mem rm` as "the documented producers" of exit 5. That is
incomplete in two directions: `mem patch` produces it on two distinct paths (base-hash
mismatch and relay duplicate), and `repos protect set`/`remove` produce it too — the only
non-`mem` producers in the codebase, added at `repos.rs:154-158` with a test
(`duplicate_write_response_is_a_conflict`, `repos.rs:628`). No other command in this group
maps anything to exit 5: `grep -n 'CliError::Conflict' ` across the eleven files matches
only `mem.rs:101`, `mem.rs:595`, and `repos.rs:155`.

#### Test coverage of the API surface

Covered: `read_optional_body`'s mutual exclusion and empty default (`pr.rs:334`, `pr.rs:339`);
`parse_status` word set and rejection (`patches.rs:304`, `patches.rs:319`); `parse_committer`
arity (`patches.rs:284`, `patches.rs:298`); the write-response normalization/conflict split
(`repos.rs:628`, `repos.rs:640`); `resolve_reader`'s three-way perspective resolution and
both mutual-exclusion rules (`mem.rs:793`-`mem.rs:845`).

Not covered anywhere: every `dispatch` arm (no test constructs a `*Cmd` and dispatches);
`--format` propagation; the group's exit-code mapping end to end; the `/moderation/*` URL
construction; `resolve_expiry`; `cmd_get_workflow`'s `null` branch. There is no
`crates/buzz-cli/tests/` directory (`ls crates/buzz-cli/tests` → "No such file or
directory") and no `#[tokio::test]` in any of the eleven files
(`grep -rn 'tokio::test' crates/buzz-cli/src/commands/` returns zero matches).
