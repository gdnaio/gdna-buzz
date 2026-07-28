## Module: buzz-cli — repo, agent, memory & moderation commands (`crates/buzz-cli/src/commands`)
### Aspect: Security

#### Input validation on ids, names and content

Identifier validation is consistent and applied first in almost every handler.

| Input class | Validator | Rule | Applied at |
|---|---|---|---|
| pubkey / event id | `validate_hex64` (`validate.rs:29-37`) | exactly 64 chars, `is_ascii_hexdigit` (so **uppercase hex is accepted** despite the doc comment at `validate.rs:28` saying "lowercase") | `pr.rs:37`, `:39`, `:40`, `:80`, `:82`, `:83`, `:108`, `:126`, `:135`, `:166`, `:176`, `:192`; `patches.rs:26`, `:27`, `:74`, `:91`, `:104`, `:135`, `:143`, `:161`; `issues.rs:19`, `:20`, `:37`, `:54`, `:67`, `:98`, `:106`, `:123`; `moderation.rs:40`, `:52`, `:69`, `:80`, `:96`; `agents.rs:100`, `:130`; `repos.rs:242`, `:261`; `users.rs:32`, `:255` |
| repo id | `validate_repo_id` (`validate.rs:39-63`) | 1-64 bytes, no leading `.`, no `..`, charset `[A-Za-z0-9._-]` | every git command: `pr.rs:38`, `patches.rs:27`, `issues.rs:20`, `repos.rs:213`, `:238`, `:288` |
| UUID | `validate_uuid` / `parse_uuid` (`validate.rs:19-26`) | `Uuid::parse_str` | `workflows.rs:14`, `:39`, `:70`, `:103`, `:124`, `:125`, `:141`, `:163`, `:198`; `agent_management.rs:130`, `:147` |
| engram slug | `normalize_slug` → `validate_slug` (`buzz-core/src/engram.rs:123-131`, `:66-90`) | `core`, or `mem/` + `[a-z0-9][a-z0-9_-]{0,63}` segments, ≤ 255 bytes total | every `mem` subcommand: `mem.rs:280`, `:311`, `:512`, `:541`, `:709` |
| base-hash | inline (`mem.rs:566-572`) | 64 chars, `is_ascii_hexdigit`, compared case-insensitively at `mem.rs:607` | `mem patch` |
| relay `self` pubkey | `normalize_relay_self_hex` (`agents.rs:250-256`) | 64 hex, lowercased before comparison | `agents.rs:284` |

`validate_repo_id`'s `..` and leading-`.` rules exist precisely because the repo id becomes a
path segment in the relay's git URL (`/git/{owner}/{repo}/…`,
`crates/buzz-relay/src/api/git/transport.rs:1760-1762`) — so this one input **is** traversal-hardened.

Gaps:

- **`--status` and `--limit` on the moderation reads are unvalidated and interpolated raw
  into a URL query string.** `moderation.rs:110-112` builds
  `format!("/moderation/reports?limit={limit}")` then `push_str(&format!("&status={s}"))`
  with no charset check and no encoding. `limit` is `i64` with `default_value_t = 50` and no
  clamp (`lib.rs:1653-1655`), so `--limit -1` and `--limit 9223372036854775807` both reach
  the relay verbatim; `--status 'open&limit=99999'` injects a second query parameter. The
  CLI *has* a percent-encoder — `percent_encode` at `validate.rs:78` — but it is
  **`#[cfg(test)]`-gated at `validate.rs:77`**, so no production path can call it. Severity
  is low, not nil: the injected query is inside the caller's own NIP-98 signature
  (`client.rs:841`) and the relay re-parses it (`api/bridge.rs:2045-2049`), so this lets the
  key-holder shape their own request rather than forge someone else's. It still means the
  CLI's advertised limit is not a limit.
- **`validate_content_size` is never called from this group.**
  `grep -rn 'validate_content_size' crates/buzz-cli/src/` returns matches only in
  `messages.rs:9`, `:492`, `:707` and `validate.rs` itself. So the 64 KiB `MAX_CONTENT_BYTES`
  guard (`validate.rs:4`, `:64-73`) that the messaging commands enforce locally is absent
  from every git/issue/PR/workflow/moderation write here.
- `repos create --clone-urls` and `--web` are published unvalidated (`repos.rs:216-224`): no
  scheme check, no length check. A `javascript:` or `file:` URL lands in the `clone` tag and
  will be rendered by whatever client reads the announcement.
- `patches send --commit-pgp-sig` and `--committer` are pass-through. `parse_committer`
  (`patches.rs:61-72`) only checks the pipe-separated field *count* — the timestamp and
  tz-offset fields are never parsed as numbers.

#### Length capping before publish

Capping happens in `buzz-sdk`, not in these files. `check_content`
(`buzz-sdk/src/builders.rs:35-41`) is applied by the builders this group calls:

| Write | Cap | Enforced at |
|---|---|---|
| `patches send` | 60 KiB | `builders.rs:1023` |
| `issues create` | 64 KiB | `builders.rs:1087` |
| `pr open` | 64 KiB | `builders.rs:1335` |
| `pr update` | 64 KiB | `builders.rs:1421` |
| `*/status` | 64 KiB | `builders.rs:1227` |
| `repos create` | 64 KiB | `builders.rs:742` |
| `workflows create` / `update` (YAML) | 64 KiB | `builders.rs:1468`, `:1486` |
| `agents archive` / `unarchive` | 64 KiB | `builders.rs:1795`, `:1816` |
| `mem set` / `patch` | 65 535 B (NIP-44 plaintext max) | `mem.rs:333-338`, `mem.rs:650-656` — the only cap enforced *in this group's own code* |
| `agents draft-*` | 20 000 chars prompt, 120 chars name, 300 chars for other fields | `agent_management.rs:10-11`, `:83` |

**Uncapped:** `moderation --reason`. `build_moderation_ban` / `_timeout` /
`_resolve_report` place it into a `reason` tag with no length check
(`builders.rs:1605-1607`, `:1631-1633`, `:1677-1679`) and no `check_content` call — those
builders set content to `""`. Nothing between argv and the relay bounds it. `pr open
--subject` is capped at 256 (`builders.rs:1338-1345`) but `labels` (`--label`, repeatable)
are not bounded in count or length (`builders.rs:1366-1368`).

Escaping: none, anywhere, and correctly so for the Nostr path — tag values and content are
serialized by `serde_json` inside `nostr::Event`, so JSON-level injection is not possible.
The escaping gap is at the *terminal*, covered below.

#### Path traversal on every file argument

This is the weakest area. Five file/directory arguments reach the filesystem; **none** is
confined to a root, and none uses the repo's own traversal-safe primitive.

| Argument | Reaches | Traversal check | Size bound | Symlink resolution |
|---|---|---|---|---|
| `mem patch --patch-file <path>` | `std::fs::read_to_string(path)`, `mem.rs:581` | none | **none** — the `limit` computed at `mem.rs:577` is applied only in the stdin arm (`mem.rs:584-585`) | none |
| `patches send --patch-file <path>` | `read_file_or_stdin` → `std::fs::read_to_string`, `validate.rs:195` (called `patches.rs:26`) | none | none | none |
| `pr open/update/status --body-file <path>` | same, `pr.rs:15` via `read_optional_body`, `pr.rs:9-18` | none | none | none |
| `media get -o <path>` | `std::fs::write(path, &bytes)`, `upload.rs:23` | none | n/a | none |
| `pack validate/inspect <path>` | `Path::new(path)` + `exists()`/`is_dir()`, `pack.rs:16-22`, `:53-59` | none | see below | none |

Compare against the repo's own answer to exactly this problem —
`buzz_persona::safe_resolve` (`crates/buzz-persona/src/pack.rs:323-364`), which does three
things these five sites do none of:

1. rejects absolute paths, both `/`-prefixed and Windows drive letters (`pack.rs:325-333`);
2. rejects any `Component::ParentDir` before joining (`pack.rs:335-341`);
3. canonicalizes (resolving symlinks) and asserts the result `starts_with(pack_root)`
   (`pack.rs:352-360`), returning `PackError::PathEscape` otherwise.

And `read_bounded_file` (`pack.rs:374-386`) stats the file and refuses it above `max_bytes`
*before* reading — the pattern `mem.rs:581` should have used and does not.

So the codebase already contains a correct, tested confinement helper, applied one crate
over, and the CLI's own file arguments do not use it. The asymmetry is sharp inside
`pack.rs` itself: the **pack root** the operator names is unconstrained (`pack.rs:16`), but
every file *inside* the pack is resolved through `safe_resolve` by `buzz-persona`. A
malicious pack cannot escape its own directory; the operator (or an agent) can point the
pack root anywhere.

How much this matters depends on the trust model, and here it is worth being blunt: these are
CLI arguments, so a human operator can already read and write any file their uid can reach.
The relevant threat is the one `AGENTS.md` describes — an **agent** subprocess with
`BUZZ_RELAY_URL` / `BUZZ_PRIVATE_KEY` / `BUZZ_AUTH_TAG` auto-injected by the ACP harness,
choosing its own argv. For that caller:

- `buzz patches send --patch-file /Users/x/.ssh/id_ed25519 --repo-owner … --repo-id …`
  publishes the file's contents as a kind:1617 event body. There is no content inspection —
  `read_file_or_stdin` returns whatever it read (`validate.rs:195-196`) and
  `build_git_patch` only length-checks (`builders.rs:1023`). **This is an arbitrary-file
  exfiltration primitive** reachable with three flags, and it is the single most significant
  finding in this aspect. The same shape applies to `pr open --body-file` and
  `mem patch --patch-file` (the latter exfiltrates into an encrypted engram, so only the
  owner can read it back).
- `buzz media get <sha256> -o <path>` writes relay-supplied bytes to an arbitrary path
  (`upload.rs:23`) with no extension, MIME or path check on the way out — an
  arbitrary-file-write primitive. `upload_file` validates MIME on the way *in*
  (`client.rs:1116-1118`, allow-list `ALLOWED_MIMES`), but download does not validate on the
  way out.

Nothing here is a memory-safety or privilege-escalation bug; it is missing defence-in-depth
against a caller the architecture explicitly anticipates.

#### Secret exposure to stdout, stderr, errors, or published events

Private keys: the secret key is used in exactly two ways in this group — deriving the NIP-44
conversation key (`mem.rs:147`, `mem.rs:167`, `mem.rs:226`) and signing
(`client.rs:588-616`, `client.rs:743-750`). `grep -n 'secret_key\|to_bech32\|nsec'` across the
eleven files returns only the three `mem.rs` derivation sites; none is printed, formatted, or
placed in an event. `keys().public_key()` appears widely and is not secret.

Secrets that *can* reach a terminal:

- `pack inspect` prints `runtime_env_vars` as `k=v` pairs (`pack.rs:139-146`). I checked what
  can be in that field: `buzz-persona`'s `runtime_env_vars` projection
  (`resolve.rs:365-397`) emits only `GOOSE_PROVIDER`, `GOOSE_MODEL`, `GOOSE_TEMPERATURE`,
  `GOOSE_CONTEXT_LIMIT`, `BUZZ_AGENT_MODEL`, `BUZZ_AGENT_PROVIDER` — derived from
  `model`/`temperature`/`max_context_tokens`, no credentials. So **this print is not a leak
  today**, but it is a leak-shaped API: the field name promises "env vars", and if the
  projection ever grows to carry pack-declared secrets the printer needs no change to start
  emitting them.
- MCP server `env` values *do* carry literal credentials — `buzz-persona` preserves them
  verbatim with no interpolation and its own test uses `"TOKEN": "abc"` and
  `"SECRET": "${MY_SECRET}"` as fixtures. `pack inspect` holds those in memory (they are
  inside `ResolvedPersona::mcp_servers`) but prints **only the count**
  (`pack.rs:116`). That is the right call and worth preserving — printing the servers would
  turn `pack inspect` into a credential dump.
- Error messages echo the offending value: `validate_hex64` formats the input into its
  message (`validate.rs:33-36`), as do `validate_repo_id` (`validate.rs:57-61`) and
  `validate_uuid` (`validate.rs:24`). If a caller ever mis-slots a token into a `--pubkey`
  argument, the token lands in the stderr JSON error object (`error.rs:132-137`). Low
  severity, but it is a value-echoing error path.
- `handle_response` (`client.rs:1266-1274`) appends a hint when a 403 arrives and
  `BUZZ_AUTH_TAG` is set. It names the variable and does **not** print its value — correct.

Published events: `sign_event` (`client.rs:588-616`) injects the NIP-OA `auth` tag into every
event and then asserts the resulting event carries exactly the expected number of `auth` tags
(`client.rs:601-614`), refusing to publish otherwise. That auth tag is a public attestation
(pubkey + conditions + signature), not a secret. `sign_event_unchecked`
(`client.rs:743-750`) bypasses both the injection and the assertion and is used **only** by
`agents archive`/`unarchive` (`agents.rs:105`, `:135`); the reason is documented at
`client.rs:727-742` — the `auth` tag there is a content-level attestation about the *target*
identity, not the caller's membership delegation. That is a narrow, argued exemption, but it
is the one path in the group where the "no manual auth tags" invariant is not enforced.

One genuine sanitization step exists: `build_updated_repo_announcement` strips any `auth` tag
off the event it read back from the relay before re-signing (`repos.rs:110-117`), so a
relay-supplied auth tag cannot be laundered into the caller's new announcement. That is a
good pattern and it is tested (`repos.rs:450-453`).

#### Authorization: what the CLI checks and what it delegates

The CLI enforces **no permission locally**. There is no role check, no owner check, no
membership check in any of the eleven files. Concretely:

- `moderation ban/unban/timeout/untimeout/resolve` build and submit the command event with no
  local notion of whether the caller is an owner or admin (`moderation.rs:34-102`). The
  relay authorizes: `authorize_moderation_action` (`api/bridge.rs:2055-2056`) after
  `verify_bridge_auth` + NIP-98 replay check (`api/bridge.rs:2050-2052`). The module doc says
  so explicitly (`moderation.rs:4-7`: "the relay validates, authorizes (owner/admin only),
  and executes them directly").
- `repos protect set/remove` do check that the announcement being modified is *the caller's
  own* — `fetch_own_repo_announcement` filters on `authors: [self]` (`repos.rs:23`) and
  `current_repo` returns `NotFound` otherwise (`repos.rs:286-294`). That is scoping, not
  authorization: it prevents accidentally editing someone else's repo, and it also means a
  non-owner cannot even see the rules to edit them.
- `mem` reads and writes are cryptographically scoped rather than authorized: the `d` tag is
  `HMAC(K_c, slug)` (`buzz-core/src/engram.rs:144-154`), so without the agent↔owner
  conversation key you cannot even construct the filter. `resolve_reader`
  (`mem.rs:52-79`) refuses `--owner` with `--agent` and refuses `--agent` equal to the CLI
  identity — both are correctness guards, not permission checks.
- `agents archive/unarchive` attach an owner-of-agent attestation when one exists on the
  target's kind:0 (`resolve_auth`, `agents.rs:172-196`) and otherwise send the request bare
  (`Ok(None)`). The relay decides. Note `resolve_auth` fails **closed on network error**
  (`agents.rs:182-184` maps a query failure to `Err`) rather than silently degrading to bare
  — the doc comment at `agents.rs:167-171` explains why. Good choice, correctly reasoned.

What that means for an agent holding a delegated `BUZZ_AUTH_TAG`: the tag is injected into
every event by `sign_event` (`client.rs:589-593`) and into every HTTP request as the
`x-auth-tag` header (`client.rs:616-621`), so **the agent acts with the full authority the
owner delegated, on every command, with no per-command consent step and no local narrowing.**
If the owner is a community moderator, the agent can ban members; if the owner owns repos,
the agent can rewrite branch protection. The only guardrails are relay-side: the
`conditions` field of the NIP-OA tag and whatever `authorize_moderation_action` decides.
Nothing in this group inspects `conditions` — `auth_tag_owner_hex` (`client.rs:576-583`)
reads slot 1 (owner pubkey) and ignores slot 2 (conditions) entirely, and no file here reads
it either. So an agent cannot tell, locally, whether an action is within its delegation; it
finds out from a relay 403. `handle_response` at least makes that failure legible
(`client.rs:1266-1274`).

The one place the CLI *adds* a review gate is `agents draft-create` / `draft-update`: they
publish an encrypted observer frame to the owner and print `"saved": false` with an explicit
"Nothing changes until the owner saves it" message (`agents.rs:38-41`, `:80-83`) rather than
mutating anything. That is a deliberate human-in-the-loop step, and it is the only one in the
group.

#### The moderation surface's exposure

`/moderation/reports`, `/moderation/restricted` and `/moderation/audit` return the community's
report queue, its restricted-member list, and its enforcement audit log — the most
privacy-sensitive reads in the CLI. Protections, verified end to end:

| Layer | Mechanism | Citation |
|---|---|---|
| Transport auth | NIP-98 signed GET, signature over the **full URL including query** | `client.rs:841`, expected-URL recomputed relay-side at `api/bridge.rs:2049` |
| Replay | `check_nip98_replay` | `api/bridge.rs:2052` |
| Tenant binding | `bind_community` from the `Host` header **before** any lookup | `api/bridge.rs:2036-2043` |
| Authorization | `authorize_moderation_action` (owner/admin) | `api/bridge.rs:2055-2056` |
| Delegation | `x-auth-tag` forwarded | `client.rs:838`, `client.rs:616-621` |

The residual exposure is on the client side: all three commands print the relay body
**verbatim to stdout** (`moderation.rs:115`, `:121`, `:129`) with no redaction and no
field selection. `moderation --reason` on the write side is documented as reader-visible
("relayed to the reporter", `builders.rs:1650-1653`; and the clap help at `lib.rs:1671`
says "keep it tombstone-safe"), while `ban --reason` is labelled "private ban reason (audit
only)" (`lib.rs:1689`) — yet `moderation audit` will print exactly those private reasons to
whatever captured stdout. In an agent harness stdout is the model's context, so a single
`buzz moderation audit` pulls the whole private enforcement history into an LLM transcript.
Nothing warns about that.

`--format compact` is accepted by `moderation::dispatch` and then discarded — the parameter
is bound as `_format` (`moderation.rs:136`) — so there is no way to ask for fewer fields.

#### Output injection into terminals

Relay- and file-supplied bytes are printed with no sanitization at these sites:

| Site | Source | Sanitization |
|---|---|---|
| `mem.rs:296`, `mem.rs:300` | decrypted engram value / core profile, raw bytes via `write_all` | **none** — deliberately verbatim so `mem get` round-trips with `mem set -` (`mem.rs:295`) |
| `mem.rs:671` | the input patch text, echoed back | none (`trim_end_matches('\n')` only) |
| `mem.rs:268` | slug TSV | slug charset is `[a-z0-9_/-]` (`engram.rs:94-116`), so this one is safe by construction |
| `repos.rs:228`, `:252`, `:280`; `pr.rs:61`, `:103`, `:114`, `:147`, `:212`; `patches.rs:53`, `:80`, `:109`, `:186`; `issues.rs:32`, `:43`, `:76`, `:143`; `moderation.rs:115`, `:121`, `:129` | raw relay response body | none — passed through as received |
| `pack.rs:74`, `:75`, `:94`, `:120`, `:124`, `:133-136` | persona `display_name`, `description`, `subscribe`, `skills`, `avatar`, system-prompt preview | only `replace('\n', " ")` on the prompt preview (`pack.rs:136`) |
| `pack.rs:29`, `:32` | `buzz-persona` diagnostic messages, which embed file paths and parse errors | none |

The relay bodies are JSON, so control characters inside string values arrive as `\u00XX`
escapes and are inert. The two paths that are genuinely raw are `mem get`/`mem patch`'s echo
and `pack inspect`'s persona fields: an owner who writes ANSI escape sequences into an engram
value, or a persona pack whose `description` contains them, gets those sequences executed by
the operator's terminal on the next `buzz mem get` / `buzz pack inspect`. This can rewrite
visible scrollback, hide subsequent output, or set the window title. Low severity for a
human, more interesting for an agent harness where the same bytes become model context —
`mem get` output is attacker-controlled text that flows straight into a prompt with no
delimiter, which is a prompt-injection channel rather than a terminal one.

`grep -n 'escape\|sanitize\|strip_ansi\|is_control'` across the eleven files returns zero
matches.

#### Resource exhaustion

Unbounded reads:

| Read | Bound | Site |
|---|---|---|
| `mem set <slug> -` (stdin) | `NIP44_PLAINTEXT_MAX + 1` via `.take()` — **the only bounded stdin read in the CLI** | `mem.rs:324-338` |
| `mem patch` stdin | same `.take(limit)` | `mem.rs:584-585` |
| `mem patch --patch-file` | **none** — `limit` is not applied to the file arm | `mem.rs:581` |
| `read_or_stdin("-")` — `agents draft-* --system-prompt`, `issues create --content`, `*/status --content`, `pr --body`, `workflows create/update` YAML | **none** — plain `read_to_string` | `validate.rs:168-179` |
| `read_file_or_stdin` — `patches send --patch-file`, `pr --body-file` | **none**, for both the file and stdin arms | `validate.rs:186-198` |
| `upload file` | `MAX_IMAGE_BYTES` 50 MiB / `MAX_VIDEO_BYTES` 500 MiB, checked after a full `fs::read` into memory | `client.rs:73`, `:76`, `:1108`, `:1116-1131` |
| `media get` | none — whole blob buffered via `resp.bytes()` | `client.rs:1252` |

So a runaway producer piping into `buzz issues create --content -` will be read entirely into
memory before the SDK's 64 KiB check rejects it — the cap is applied after the read, not
during. `mem` is the one family that got this right, and even there the `--patch-file` arm
was missed.

Unbounded queries:

| Query | Limit |
|---|---|
| `repos get` | **no `limit` at all** (`repos.rs:238-241`) — and no `authors` unless `--owner` is passed, so this is "every 30617 with this `d` tag, from anyone" |
| `repos list` | only if `--limit` given (`repos.rs:275-277`) |
| `pr list`, `patches list`, `issues list` | only if `--limit` given (`pr.rs:139-141`, `patches.rs:104-106`, `issues.rs:71-73`) |
| `workflows list` | **no limit** (`workflows.rs:15-18`) |
| `mem ls` | hardcoded `5000` (`mem.rs:201`), then every returned event is signature-verified **and NIP-44-decrypted** in a loop (`mem.rs:206-232`) — 5 000 decryptions is the worst case, and the events are all held in a `HashMap<String, Vec<(Event, Body)>>` simultaneously |
| `users get` | `limit: authors.len()`, and `--pubkey` is capped at 200 (`users.rs:30-32`) — the group's best-behaved read |
| `users get --name` | `100` (`users.rs:93`) |
| `users presence` | `limit: pubkeys.len()`, **no cap on the CSV length** (`users.rs:247-261`) — unlike `--pubkey`, which has the 200 check |
| `workflows runs` | `limit.unwrap_or(20).min(100)` (`workflows.rs:72`) — the only clamped limit in the group |
| `moderation reports` / `audit` | `--limit` passed through unclamped, negative allowed (`moderation.rs:110`, `:127`) |

Unbounded diffs: `validate.rs` defines `MAX_DIFF_BYTES = 61_440` (`validate.rs:7`) and
`truncate_diff` (`validate.rs:103-121`) to cut a diff at a hunk boundary.
`grep -n 'truncate_diff\|MAX_DIFF_BYTES'` across the eleven files returns **zero matches** —
both are used only by `messages.rs`. So `patches send` accepts a diff of any size up to the
SDK's 60 KiB hard rejection (`builders.rs:1023`) with no truncation option, and
`mem patch`'s multi-file guard (`mem.rs:618`) is a count of `--- ` headers over the whole
in-memory string, computed after the unbounded read.

The relay-facing retry policy is the one place resource use is deliberately bounded:
`RETRY_MAX_ATTEMPTS` with jittered backoff (`client.rs:638-681`), a 429 `retry in Ns` hint
capped at `RETRY_IN_MAX_SECS` (`client.rs:661-665`), and moderation kinds excluded from
blind retry so an ambiguous outcome surfaces as `DeliveryUnknown` rather than a duplicated
ban (`client.rs:863-870`, `error.rs:38-42`).

#### Where I am uncertain

- I did not attempt any of the exfiltration or injection paths described above; they are
  read from the code, not demonstrated. In particular I did not verify that a relay accepts a
  kind:1617 whose body is arbitrary non-diff bytes — `build_git_patch` does not appear to
  parse the content (`builders.rs:1013-1068`), but the relay may.
- The claim that `runtime_env_vars` cannot carry credentials rests on
  `buzz-persona/src/resolve.rs:365-397` being the only producer of that field. I confirmed
  the single assignment at `resolve.rs:221`/`:250` but did not audit every `buzz-persona`
  code path.
- Terminal-escape severity depends on the operator's terminal emulator; I have not tested
  which sequences a given emulator honours.
- I did not check whether the relay clamps `limit` on `/moderation/*` server-side, so the
  practical effect of an unclamped `--limit` is unknown.
