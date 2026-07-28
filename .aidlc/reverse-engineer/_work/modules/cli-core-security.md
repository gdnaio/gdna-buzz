## Module: buzz-cli — dispatch, relay client & validation (`crates/buzz-cli/src`)
### Aspect: Security

#### Private key: where it is read, held, and where it can escape

Read: clap arg `private_key` with `env = "BUZZ_PRIVATE_KEY"` (`lib.rs:84-85`),
consumed in `run` (`lib.rs:1746-1750`) and parsed by `Keys::parse`, which accepts
raw hex or `nsec1…`. Held: `BuzzClient.keys: nostr::Keys` for the process
lifetime (`client.rs:524`), exposed crate-internally via `keys()`
(`client.rs:562-564`).

No secret wrapper of any kind: `grep -rn 'zeroize\|Secret\|redact' crates/buzz-cli/src`
returns zero matches. The key is not zeroized on drop, the plaintext `String`
from clap is never cleared, and `BuzzClient` derives no `Debug` (so at least
there is no accidental debug-print path — `client.rs:521`).

**The key does reach terminal output.** clap prints live env values in help text
and `hide_env` is never set (`grep -rn 'hide_env' crates/buzz-cli/src` → zero
matches). Verified against the built binary:

```
$ BUZZ_PRIVATE_KEY=nsec1SUPERSECRETVALUE buzz --help
      --private-key <PRIVATE_KEY>
          Nostr private key (hex or nsec). This is the CLI's identity
          [env: BUZZ_PRIVATE_KEY=nsec1SUPERSECRETVALUE]
```

`buzz -h` leaks identically. The same block prints `BUZZ_AUTH_TAG`'s full value.
This matters more than usual here because (a) the ACP harness injects
`BUZZ_PRIVATE_KEY` into managed agent subprocesses (`AGENTS.md:161-164`), (b)
`buzz --help` is the first thing an LLM-driven agent runs against an unfamiliar
CLI, and (c) help output goes to **stdout with exit 0** (`lib.rs:48-52`), so it
lands in agent transcripts, CI logs and screen recordings as ordinary success
output. Fix is a one-line `hide_env_values`/`hide_env` on the two secret args.

Other potential escape routes, checked:

| Route | Result |
|---|---|
| Error messages | Key never interpolated. Parse failure prints only the upstream reason — verified: `BUZZ_PRIVATE_KEY=zzz` → `key error: invalid BUZZ_PRIVATE_KEY: Invalid secret key` (`lib.rs:1749-1750`) |
| Auth-tag verification failure | Interpolates the **public** key only (`lib.rs:1760-1764`) |
| Log lines | No logging framework exists (`grep -rn 'tracing\|log::'` → zero matches) |
| Serialized output | Only signatures/pubkeys are serialized; `sign_*` helpers emit signed events, never the secret (`client.rs:84-110`, `:325-385`) |
| Child process env | No subprocess is spawned (`grep -n 'Command::new' …` → zero matches) |
| Panic backtraces | Three production panic sites (`client.rs:506`, `:1379`, `:679`) carry no key material |

#### TLS and scheme handling

`normalize_relay_url` (`client.rs:1291-1297`) maps `wss://`→`https://` and
`ws://`→`http://`, then trims a trailing slash. `to_ws_url`
(`client.rs:1299-1305`) maps back. There is **no downgrade**: `wss://` round-trips
to `wss://`, `https://` to `wss://`. Both use `str::replace`, which substitutes
anywhere in the string rather than only at the prefix — a relay URL containing
`ws://` inside a path or query would be corrupted; low practical risk, but it is
not a prefix-anchored parse.

Weaknesses:

- **Plaintext is the default.** `--relay` defaults to `http://localhost:3000`
  (`lib.rs:81`), and nothing rejects, warns about, or upgrades `http://` for a
  non-loopback host. `grep -n 'https\|scheme' client.rs` shows scheme checks only
  inside `media_url_from_input` (`client.rs:272`, `:302-309`) for origin
  matching, never as a policy gate. Over `http://`, every request carries a
  signed NIP-98 event and (when configured) the `x-auth-tag` header in the clear,
  so a network observer can replay a captured NIP-98 event to the same URL with
  the same method and body for as long as the relay's freshness window allows —
  the event has a `nonce` but **no `expiration` tag** (`client.rs:88-105`).
- **No scheme validation at all.** `buzz --relay notaurl channels list` reaches
  `reqwest` and fails as `network_error: builder error: relative URL without a
  base` with exit 2 (verified) rather than being rejected as bad input. A typo
  like `htp://relay` is likewise a runtime failure, not a parse failure.
- No `--insecure`, custom CA, client-cert or pinning options exist, so TLS
  verification is `reqwest`'s default and cannot be weakened by a flag — the one
  security-positive consequence of having no transport knobs.

#### NIP-42 AUTH and signing-oracle surface

NIP-42 challenge handling is not implemented here; it is delegated to
`buzz_ws_client::publish_event` (`client.rs:1084`,
`crates/buzz-ws-client/src/connection.rs:277-293`), so this group has no
challenge-parsing or signature-echo logic of its own to audit.

Signing surfaces this group *does* own, ranked by capability granted:

1. **Blossom `t=get` auth (`sign_blossom_get`, `client.rs:325-348`)** — a signed
   kind-24242 event with `t=get`, `expiration = now+600`, `server=<authority>`
   and **no `x` (blob hash) tag**, deliberately (asserted at
   `client.rs:479-481`). Anyone who captures that header has a 10-minute,
   server-wide read capability for that identity, not a single-blob token. Over
   `http://` that capture is trivial.
2. **Blossom `t=upload` auth (`client.rs:350-385`)** — bound to the file hash via
   `x`, expiry 600 s (3600 s for video). Tighter.
3. **NIP-98 (`sign_nip98`, `client.rs:84-110`)** — bound to URL + method +
   body hash, so replay is confined to the identical request; no expiration tag.
4. **`sign_event` (`client.rs:588-614`)** — refuses to emit an event whose `auth`
   tag count differs from expectation, blocking a caller from smuggling a forged
   owner attestation into an otherwise-normal event.
5. **`sign_event_unchecked` (`client.rs:743-747`)** — signs whatever it is given,
   including caller-supplied `auth` tags. Restricted to NIP-IA by comment only;
   nothing enforces the kind.

Positive controls worth naming: `download_media` builds a dedicated client with
`redirect::Policy::none()` explicitly so `Authorization` and `x-auth-tag` are
never forwarded to a redirect target (`client.rs:1231-1237`), and
`media_url_from_input` refuses to sign a GET for any origin other than the
configured relay (`client.rs:302-315`) — both with tests
(`client.rs:404-451`).

#### Input validation

| Input | Validation | Site |
|---|---|---|
| media URL / hash | scheme allowlist, `/media/` path prefix, segment must be `sha256`, `sha256.ext` or `sha256.thumb.jpg` with **lowercase** hex and `[a-z0-9]{1,8}` ext, origin must equal relay | `client.rs:250-323` |
| event id / pubkey | 64 chars, `is_ascii_hexdigit` — **uppercase accepted** though the doc says lowercase | `validate.rs:28-36` |
| UUID | strict parse | `validate.rs:23-27` |
| repo id | 1-64 chars, no leading `.`, no `..`, `[A-Za-z0-9._-]` — an explicit path-traversal guard, since repo ids become git paths | `validate.rs:39-61` |
| content | ≤ 64 KiB | `validate.rs:64-72` |
| upload file | must be a regular file, MIME from magic bytes (not extension), 5-MIME allowlist, size cap | `client.rs:1102-1130` |
| cursor id from relay | 64 ASCII hex before reuse in the next filter | `client.rs:511-514` |
| agent draft fields | trim, non-empty, char caps, UUID channel, `respond_to` allowlist | `agent_management.rs:70-85`, `:128-179` |

Path handling: `upload_file` (`client.rs:1100`) and `read_file_or_stdin`
(`validate.rs:180-193`) accept any path the caller can name, with no confinement
to a working directory. For a local CLI this is expected, but it means an agent
that will run `buzz` with attacker-influenced arguments can read arbitrary files
— `buzz upload file --file ~/.ssh/id_rsa` fails only because the MIME allowlist
rejects it (`client.rs:1114-1116`), while `--patch-file /etc/passwd` succeeds and
its content is published (`validate.rs:189-192`).

#### Command injection

None reachable from this group: there is no shell invocation and no subprocess
(`grep -n 'Command::new\|process::Command\|sh -c' lib.rs client.rs validate.rs error.rs agent_management.rs main.rs`
→ zero matches). `read_or_stdin` passes values through verbatim with no
evaluation, and a test pins that backticks and `$vars` survive untouched
(`read_or_stdin_passthrough_returns_value`, `validate.rs:463-471`).

#### Output injection

The error envelope is safe: `serde_json` escapes control bytes, so ANSI escapes
in user input cannot reach the terminal raw. Verified —
`buzz channels get --channel $'x\033[31mRED'` prints
`{"error":"user_error","message":"invalid UUID: x\u001b[31mRED","retryable":false}`.
`print_create_response` (`client.rs:1401-1403`) likewise emits
`serde_json`-serialized text. The residual risk is outside this group: read
commands that `println!` the relay body verbatim (e.g.
`commands/social.rs:78`) forward whatever bytes the relay sent, so terminal
safety there depends on the relay's serializer.

#### Resource exhaustion

- `query_all` (`client.rs:724-727`) pages until the relay returns a short page
  with no page-count or memory cap; a broad filter accumulates every matching
  event in RAM.
- `upload_file` reads the whole file into memory before hashing
  (`client.rs:1108`), so a 500 MiB video is a 500 MiB allocation (plus the
  `bytes::Bytes` clone per retry attempt, `client.rs:1148`).
- `read_or_stdin` / `read_file_or_stdin` slurp unbounded input into a `String`
  (`validate.rs:164-170`, `:182-187`); `validate_content_size` is a separate,
  opt-in call that the sibling handlers must remember to make (3 call sites in
  `commands/`).
- Retry amplification is bounded: 3 attempts, ≤30 s honored hint
  (`client.rs:122-130`), and the moderation path deliberately declines to retry
  ambiguous outcomes so a ban cannot be applied three times
  (`client.rs:911-972`).

#### Information disclosure in error text

`handle_response` and `submit_moderation_event` append
"(BUZZ_AUTH_TAG is set — it may be stale or revoked; try unsetting it)" to any
403 when the env var is present (`client.rs:1271-1279`, `client.rs:991-997`).
The value is not printed, only its presence. Note the check reads
`std::env::var("BUZZ_AUTH_TAG")` directly, so the hint is keyed to the
environment rather than to the effective configuration: a tag supplied via
`--auth-tag` produces no hint, and a tag overridden by `--auth-tag` still
produces one.
