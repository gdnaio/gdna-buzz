## Module: buzz-cli — dispatch, relay client & validation (`crates/buzz-cli/src`)
### Aspect: Features

This group is the CLI's front door and transport: what it *offers* is a single
non-interactive binary that turns argv into signed Nostr events and relay JSON,
with machine-readable success and failure envelopes. The per-command behavior
lives in `commands/` (sibling scope); everything below is what this layer itself
makes possible or prevents.

#### What an operator or agent can do through this layer

| Capability | How | Site |
|---|---|---|
| Run 21 command groups / ~100 subcommands from one binary | `enum Cmd` + dispatch match | `lib.rs:175-240`, `lib.rs:1771-1792` |
| Point at any relay per invocation | `--relay` / `BUZZ_RELAY_URL`, normalized from ws/wss | `lib.rs:81-82`, `client.rs:1291-1297` |
| Act as any identity per invocation | `--private-key` / `BUZZ_PRIVATE_KEY` | `lib.rs:84-85`, `lib.rs:1749-1750` |
| Act on behalf of an owner (NIP-OA delegation) | `--auth-tag` / `BUZZ_AUTH_TAG`, verified locally then injected into every event and sent as `x-auth-tag` | `lib.rs:1752-1767`, `client.rs:588-621` |
| Get agent-friendly reduced output | `--format compact` | `lib.rs:92-93` — but only for `messages`, `channels`, `users`, `feed`, `moderation` (`lib.rs:1772-1790`) |
| Work offline | `pack validate` / `pack inspect` short-circuit before any auth or network use | `lib.rs:1736-1743` |
| Pipe content in | `-` means stdin for content-style flags; `read_file_or_stdin` for path-style flags | `validate.rs:162-193` |
| Survive flaky links | 3-attempt retry with full jitter, `retry in Ns` honoring, body-transfer failures inside the retry boundary | `client.rs:638-681` |
| Know whether a failed write is safe to re-run | `retryable` field + `DeliveryUnknown` category | `error.rs:74-88`, `error.rs:117-125` |
| Page through large histories without flags | automatic `(until, before_id)` cursor following, 500/page | `client.rs:683-727` |
| Publish ephemeral kinds the HTTP bridge rejects | WS publish with NIP-42 AUTH | `client.rs:1073-1098` |
| Upload/download relay media with Blossom auth | `upload_file`, `download_media` | `client.rs:1100-1256` |
| Read structured moderation queue/audit rows | NIP-98-authed GET of arbitrary relay paths | `client.rs:836-856`, used by `commands/moderation.rs:114-128` |
| Request owner-reviewed agent creation/edits | encrypted observer frames, kind 24200 | `agent_management.rs:87-181` |
| Script against stable command names | inventory-drift tests fail the build on rename | `lib.rs:1806-2034` |

#### Limits and non-features

- **No `--version`.** Not declared on `#[command(...)]` (`lib.rs:62-78`);
  `buzz --version` exits 1 as an unknown argument (verified against the built
  binary), despite `lib.rs:48-52` handling a `--version` branch that can never
  fire.
- **`--format` is positional-sensitive and mostly inert.** It must precede the
  subcommand (verified: after-subcommand use exits 1), and 16 of 21 groups
  ignore it with no diagnostic.
- **No shell completions, no man page, no config file.** `grep -rn 'clap_complete\|generate(' lib.rs`
  → no matches; nothing in this group reads a dotfile or app-data path
  (`grep -rn 'dirs::' lib.rs client.rs validate.rs error.rs agent_management.rs`
  → zero matches).
- **No verbosity, quiet, colour or logging controls.** The crate has no logging
  framework at all (`grep -rn 'tracing\|log::' lib.rs client.rs validate.rs error.rs agent_management.rs main.rs`
  → zero matches). Diagnostics are limited to the single-line error envelope.
- **No proxy, CA-bundle, client-cert or TLS-pinning options**, and no
  `--insecure`. Transport config is whatever `reqwest` defaults to
  (`client.rs:547-551`).
- **Retry/backoff is not tunable.** Attempts, jitter bases and the 30 s hint cap
  are `const` (`client.rs:122-130`); only the two timeout values are env-tunable
  (`client.rs:140-150`).
- **One relay, one identity, one command per process.** No batch/multi-relay
  mode, no fan-out; `BuzzClient` is constructed once per run (`lib.rs:1768`).
- **No interactive prompts or confirmations** anywhere in this layer — every
  destructive subcommand (`channels delete`, `moderation ban`, `mem rm`) is
  immediate once parsed.
- **Content ceilings are hard.** 64 KiB per content field
  (`validate.rs:4`, `:64-72`), 60 KiB diffs before truncation
  (`validate.rs:7`), 50 MiB image / 500 MiB video uploads
  (`client.rs:73-76`), 5 allowed MIME types (`client.rs:64-71`) — so PDFs, SVGs,
  archives and non-mp4 video cannot be uploaded through `buzz upload file`.
- **Media downloads are relay-origin-only.** A URL for any other host is
  refused rather than fetched (`client.rs:305-315`), so the CLI cannot be used
  as a general fetcher — deliberate, since it would otherwise sign a Blossom
  capability for a foreign host.
- **`count` is unreachable.** `POST /count` is implemented but has zero callers
  and is marked `#[allow(dead_code)]` (`client.rs:802-834`;
  `grep -rn 'client.count(' commands/` → no matches), so COUNT filters are not
  exposed to users.
- **Unbounded reads are possible.** `query_all` (`client.rs:724-727`) keeps
  paging with no cap, and `read_or_stdin` (`validate.rs:162-175`) slurps stdin
  into memory with no size check, so a broad filter or a huge pipe is bounded
  only by RAM.
- **No `--kinds` on `messages search`** (`lib.rs:472-489`), so a caller cannot
  narrow a search the way `AGENTS.md` gotcha 3 instructs; verified as an exit-1
  usage error.
