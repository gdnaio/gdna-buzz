## Module: buzz-cli — dispatch, relay client & validation (`crates/buzz-cli/src`)
### Aspect: Configuration

#### Environment variables read by this group

| Var | Read site | Default | Effect | `.env.example` | `AGENTS.md` | `crates/buzz-cli/README.md` |
|---|---|---|---|---|---|---|
| `BUZZ_RELAY_URL` | clap `env` (`lib.rs:81`) | `http://localhost:3000` | relay base URL; `ws/wss` normalized to `http/https` (`client.rs:1291-1297`) | yes, but as `ws://localhost:3000` under the ACP section (`.env.example:130`) | yes (`AGENTS.md:162`) | yes ("defaults to http://localhost:3000") |
| `BUZZ_PRIVATE_KEY` | clap `env` (`lib.rs:85`) | none — required for every group except `pack` (`lib.rs:1746-1748`) | the identity; hex or nsec | yes (`.env.example:125`) | yes (`AGENTS.md:162`) | yes (auth table) |
| `BUZZ_AUTH_TAG` | clap `env` (`lib.rs:89`) | none; empty string treated as unset (`lib.rs:1755`) | NIP-OA owner attestation: verified locally, injected into every signed event, sent as `x-auth-tag` | **no** (`grep -c 'BUZZ_AUTH_TAG' .env.example` → 0) | yes (`AGENTS.md:162`) | no |
| `BUZZ_AUTH_TAG` (second read) | `std::env::var` (`client.rs:991`, `client.rs:1271`) | — | appends a "may be stale or revoked" hint to 403 errors | as above | no | no |
| `BUZZ_TIMEOUT_SECS` | `env_duration_secs` (`client.rs:548`) | 30 s; `0` and unparseable fall back (`client.rs:140-150`) | per-request total timeout | **no** | **no** | **no** |
| `BUZZ_CONNECT_TIMEOUT_SECS` | `env_duration_secs` (`client.rs:549`) | 15 s; same fallback rule | TCP connect timeout | **no** | **no** | **no** |

The two timeout vars are documented **only** in a Rust doc comment
(`client.rs:535-540`). Repo-wide check:
`grep -rn 'BUZZ_TIMEOUT_SECS\|BUZZ_CONNECT_TIMEOUT_SECS' --include='*.md' --include='*.toml' --include='*.example' .`
(excluding `target/`) returns hits only in `client.rs` — zero documentation
matches. They are the CLI's only operational tuning knobs and an operator cannot
discover them without reading the source.

One further env var is read by the sibling-owned command layer and worth naming
so the configuration picture is complete:
`BUZZ_ACP_ALLOWED_CHANNEL_ADD_POLICIES` (`commands/channels.rs:1021`). Nothing
in the six in-scope files reads it.

The double read of `BUZZ_AUTH_TAG` is a genuine configuration inconsistency: the
403 hint keys off the *environment* rather than the *effective* value, so
`--auth-tag <json>` with no env var set produces no hint, and an env var
overridden by `--auth-tag` still produces one.

#### CLI flags at this level

| Flag | Type | Default | Read site | Documented |
|---|---|---|---|---|
| `--relay` | `String` | `http://localhost:3000` | `lib.rs:80-82` | `lib.rs:66` long_about, README |
| `--private-key` | `Option<String>` | none | `lib.rs:84-85` | `lib.rs:67`, README |
| `--auth-tag` | `Option<String>` | none | `lib.rs:87-89` | `lib.rs:68` |
| `--format` | `OutputFormat` | `json` | `lib.rs:92-93` | `AGENTS.md:192-193` |
| `-h`/`--help` | flag | — | clap builtin | — |

Precedence is clap's standard "explicit arg beats env" — verified empirically:
with `BUZZ_RELAY_URL=http://127.0.0.1:9` and `--relay notaurl`, the failure comes
from `notaurl` (`builder error: relative URL without a base`), proving the flag
won.

#### Parsed but not read

`--format` is parsed for every invocation but forwarded to only 5 of 21
dispatchers (`lib.rs:1772`, `:1773`, `:1778`, `:1780`, `:1790`). For the other 16
groups — `agents`, `canvas`, `reactions`, `emoji`, `dms`, `workflows`, `social`,
`notes`, `repos`, `patches`, `issues`, `pr`, `media`, `upload`, `mem`, `pack` — it
is accepted and silently discarded. `grep -n 'cli.format' lib.rs` returns exactly
those five lines.

`--relay` is also parsed on the `pack` path and then never used, because `pack`
short-circuits before `BuzzClient::new` (`lib.rs:1734-1743`) — harmless, but it
means `buzz --relay <garbage> pack validate .` succeeds.

#### Configuration that only exists as constants

None of the following can be changed without a rebuild:

| Constant | Value | Site |
|---|---|---|
| `RETRY_MAX_ATTEMPTS` | 3 | `client.rs:122` |
| `RETRY_BASE_SECS` | `[0.5, 1.5]` s jitter ceilings | `client.rs:126` |
| `RETRY_IN_MAX_SECS` | 30 s cap on relay `retry in Ns` hints | `client.rs:130` |
| `QUERY_PAGE_SIZE` | 500 events/page | `client.rs:498` |
| upload timeouts | 120 s image, 600 s video | `client.rs:1141-1145` |
| download timeout | 120 s | `client.rs:1233` |
| WS publish budget | 75 s | `client.rs:1084` |
| Blossom auth expiry | 600 s get; 600/3600 s upload | `client.rs:329-330`, `:359-363` |
| `ALLOWED_MIMES` | jpeg, png, gif, webp, mp4 | `client.rs:64-71` |
| `MAX_IMAGE_BYTES` / `MAX_VIDEO_BYTES` | 50 MiB / 500 MiB | `client.rs:73-76` |
| `MAX_CONTENT_BYTES` / `MAX_DIFF_BYTES` | 64 KiB / 60 KiB | `validate.rs:4`, `:7` |
| agent draft caps | 120 name / 20,000 prompt / 300 other chars | `agent_management.rs:10-11`, `:83-85` |

#### No config file, no profile

This layer reads no file for configuration: `grep -rn 'dirs::\|home_dir\|config.toml\|\.buzzrc' lib.rs client.rs validate.rs error.rs agent_management.rs main.rs`
returns zero matches. The `dirs` dependency (`Cargo.toml:70-72`) is used only by
`commands/channel_templates.rs` to locate the desktop app's
`channel-templates.json`, which `channels create --template/--templates-file`
consumes (`lib.rs:565-575`) — that is the only file-based configuration reachable
from the CLI, and it is sibling-owned.

#### Documentation drift on configuration

1. `.env.example:130` presents `BUZZ_RELAY_URL=ws://localhost:3000` while
   `--relay`'s help says "Relay URL (http:// or https://)" (`lib.rs:80`). Both
   work — `normalize_relay_url` converts (`client.rs:1291-1297`) — but the flag
   help understates what is accepted, and `.env.example` documents the var only
   inside the "ACP (Agent Communication Protocol — buzz-acp harness)" block
   (`.env.example:113-130`), with no buzz-cli section at all.
2. `BUZZ_AUTH_TAG` is absent from `.env.example` even though `AGENTS.md:161-164`
   names it as harness-injected and this group both verifies and transmits it.
3. `Cargo.toml:19` credits clap's env feature with "(BUZZ_API_TOKEN
   auto-wired)"; that variable is read nowhere in this crate
   (`grep -rn 'BUZZ_API_TOKEN' crates/buzz-cli/src` → zero matches) and belongs
   to `buzz-acp`/`buzz-workflow`.
4. `crates/buzz-cli/TESTING.md:37-73` still describes a token/scope
   configuration model (`BUZZ_REQUIRE_AUTH_TOKEN=false`,
   `buzz-admin mint-token --scopes …`, a per-command scope table) that this layer
   contradicts in code: "The keypair IS the identity — no tokens, no other auth"
   (`lib.rs:1745-1746`). The same section points at a stale checkout path
   (`cd REPOS/buzz-nostr`, `TESTING.md:26`).
5. The `x-auth-tag` HTTP header this group sends (`client.rs:616-621`) appears in
   no markdown doc: `grep -rln 'x-auth-tag' --include='*.md' .` (excluding
   `.aidlc/`) returns nothing.
