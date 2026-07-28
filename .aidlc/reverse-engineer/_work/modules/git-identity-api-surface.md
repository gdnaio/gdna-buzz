## Module: git-sign-nostr & git-credential-nostr (`crates/git-sign-nostr`, `crates/git-credential-nostr`)
### Aspect: API Surface

#### Overview

Both crates are single-binary CLI tools invoked by `git` itself through two
different pluggable-program protocols. Neither exposes a conventional
`clap`-style `--help`; their "API" is the exact argv/stdin/stdout/status-fd
contract git expects from a `gpg.x509.program` (for `git-sign-nostr`) and a
`credential.helper` (for `git-credential-nostr`). Both also expose exactly one
public Rust function each, used as a library entry point by `buzz-dev-mcp`'s
multicall dispatch (see Integrations doc).

#### `git-sign-nostr` CLI argument surface

`parse_args()` (`crates/git-sign-nostr/src/lib.rs:196-280`) recognizes:

| Argument form | Meaning | Notes |
|---|---|---|
| `--status-fd=<N>` | status output fd | `lib.rs:210` |
| `--status-fd <N>` (two args) | status output fd | `lib.rs:212-217` |
| `--verify <file>` | verification mode, `<file>` = detached-sig path | `lib.rs:219-233`; rejects if `-bsau` already seen (`:227-231`) or `--verify` given twice (`:221-225`) |
| `-bsau <key>` | signing mode, `<key>` = signing-key id from `user.signingkey` | `lib.rs:240-256`; rejects if `--verify` already seen (`:248-252`) or `-bsau` given twice (`:242-246`) |
| `-` (bare dash) | stdin marker, required after `--verify <file>` | `lib.rs:257-259`; enforced at `lib.rs:265-269` |
| anything else | silently ignored | `lib.rs:270-271`, per NIP-GS forward-compatibility rule |

Mode is derived, not directly flagged: `Mode::Verify{sig_file}` if `--verify`
was seen, else `Mode::Sign{key_id}` if `-bsau` was seen, else a fatal parse
error (`lib.rs:262-278`) — "must specify either -bsau <key> ... or --verify
<file>". This exactly matches the two invocation forms `git-sign-nostr`'s
own `README.md` documents:

```
git-sign-nostr --status-fd=2 -bsau <keyid>        # sign
git-sign-nostr --status-fd=1 --verify <sigfile> -  # verify
```

`--status-fd` value is validated by `parse_status_fd` (`lib.rs:284-294`):
must parse as `i32` and be `>= MIN_STATUS_FD (1)` — fd 0 (stdin) is
explicitly rejected since payload is read from stdin (`lib.rs:137,
288-292`). There is no declared upper bound
(`test_parse_status_fd` at `lib.rs:2079-2087` asserts `1024` is accepted).

There is no `--help`/`--version` flag; an unrecognized flag is silently
dropped rather than erroring, by design (NIP-GS spec compatibility
requirement, restated in the doc comment at `lib.rs:270-271`).

#### `git-sign-nostr` stdio/status-fd contract

| Channel | Sign mode | Verify mode |
|---|---|---|
| stdin | git object payload (commit/tag bytes) bounded to `MAX_PAYLOAD = 100 MiB` (`lib.rs:119-121, 1552-1567`) | same payload, re-hashed for comparison |
| stdout | armored signature only, or nothing on error (`lib.rs:1063-1066`) | not used for data (status only, if `--status-fd=1`) |
| status-fd (or stderr fallback) | `[GNUPG:] BEGIN_SIGNING` / `SIG_CREATED ...` (`lib.rs:1076-1077`) | `NEWSIG`/`GOODSIG`/`BADSIG`/`VALIDSIG`/`TRUST_*`/`ERRSIG`/`NOTATION_*` lines |
| exit code | `0` success, `1` any `Error::Fatal`/`Error::VerifyFailed` (`lib.rs:1782-1791`) | same |

If `--status-fd=1` is passed in **sign** mode, the code detects the conflict
with stdout (which must carry the signature) and silently redirects status
output to stderr instead (`lib.rs:1739-1747`) — a defensive override of the
literal `--status-fd` argument the spec's CLI table doesn't call out as an
exception.

#### `git-credential-nostr` CLI argument surface

`run()` (`crates/git-credential-nostr/src/lib.rs:152-264`) reads
`std::env::args().nth(1)` as the git-credential-helper verb:

| `argv[1]` | Behavior |
|---|---|
| `get` (or no argument) | proceeds to read stdin and possibly emit a credential (`lib.rs:154`) |
| anything else (`store`, `erase`, unrecognized) | returns exit `0` immediately, no stdout/stderr output (`lib.rs:155`) |

This matches the standard three-verb git credential-helper protocol
(`get`/`store`/`erase`); `store` and `erase` are both accepted as no-ops,
which is correct for a helper with `ephemeral=true` credentials (nothing to
persist or invalidate) but is not distinguished in code — both verbs, and any
typo, take the identical silent-exit-0 branch (`lib.rs:155`).

#### `git-credential-nostr` stdin protocol (key=value lines)

`parse_stdin()` (`lib.rs:107-132`) reads line-oriented `key=value` pairs
until EOF or a blank line, recognizing:

| Line prefix | Field | Notes |
|---|---|---|
| `capability[]=authtype` (exact match) | `has_authtype_capability` | git ≥ 2.46 sends this to advertise it supports the `authtype`/`credential`/`ephemeral` extension (`lib.rs:116-117`) |
| `protocol=` | `protocol` | e.g. `https` (`lib.rs:118-119`) |
| `host=` | `host` | (`lib.rs:120-121`) |
| `path=` | `path` | only present if `credential.useHttpPath=true` (`lib.rs:122-123`) |
| `wwwauth[]=Nostr ...` | `wwwauth` | first matching line only kept (`lib.rs:124-128`) — other `wwwauth[]=` schemes (e.g. `Basic`) are ignored |

Any other line is silently ignored (no `else` branch falls through).

#### `git-credential-nostr` stdout protocol (credential response)

On success, the full ordered output (`lib.rs:258-263`) is:

```
capability[]=authtype
authtype=Nostr
credential=<base64(json(kind:27235 event))>
ephemeral=true
quit=true
<blank line>
```

`ephemeral=true` tells git not to cache/store this credential (matches the
per-request nature of a signed NIP-98 token); `quit=true` tells git not to
try any further credential helpers in the chain. On any early decline (see
Business Rules doc for the decision tree), the output is either nothing, or
a bare blank line (`lib.rs:161-164` — printed when `!has_authtype_capability`).

#### `git-credential-nostr` exit codes

| Code | Meaning | Site |
|---|---|---|
| `0` | success (credential emitted), or a deliberate silent decline (non-`get` verb, no authtype capability, no `wwwauth`/method hint) | `lib.rs:155, 163, 176, 182` |
| `1` | hard error: missing protocol/host/path, invalid key, invalid/malformed auth tag, signing/serialization failure | `lib.rs:172, 211, 220, 231, 243, 251` |

No other exit codes are used — this is a simple binary success/failure
model, unlike `buzz-cli`'s 6-value exit-code convention documented in
`AGENTS.md` (0/1/2/3/4/5). Neither crate follows that convention (expected —
they implement git's protocols, not Buzz's CLI convention).

#### Public Rust library surface

| Crate | Public item | Signature | Site |
|---|---|---|---|
| `git-sign-nostr` | `pub fn run() -> i32` | entry point, returns process exit code | `crates/git-sign-nostr/src/lib.rs:1726` |
| `git-credential-nostr` | `pub fn run() -> i32` | entry point, returns process exit code | `crates/git-credential-nostr/src/lib.rs:152` |

Both crate roots have module doc comments (`//!`, `git-sign-nostr/src/lib.rs:1-70`,
`git-credential-nostr/src/lib.rs:1-6`) but **the `pub fn run()` itself has no
`///` doc comment** in either crate — the preceding comments
(`git-sign-nostr/src/lib.rs:1724-1725`, a two-line `///`-style block, is
actually present for `git-sign-nostr`; `git-credential-nostr/src/lib.rs:151`
has a one-line `///` comment) — see Conventions doc for the precise
doc-comment-discipline finding, since the two crates are not symmetric here.

`main.rs` in both crates is a 3-line multicall shim:
```rust
fn main() {
    std::process::exit(git_sign_nostr::run());     // or git_credential_nostr::run()
}
```
(`crates/git-sign-nostr/src/main.rs:1-3`,
`crates/git-credential-nostr/src/main.rs:1-3`). No other public items exist
in either crate — every helper function (`load_key`, `parse_args`,
`compute_signing_hash`, etc.) is private to its crate, confirmed by the
absence of any `pub` modifier on them in the `grep`-derived function list for
`git-sign-nostr/src/lib.rs` (only `pub fn run` at `:1726` is `pub`) and for
`git-credential-nostr/src/lib.rs` (only `pub fn run` at `:152`).

#### No shell completions, man pages, or `--help`

Neither crate integrates `clap` or any CLI-argument-parsing framework —
both hand-roll argv scanning (`git-sign-nostr/src/lib.rs:196-280`,
`git-credential-nostr/src/lib.rs` reads only `args().nth(1)` at `:153`).
Consequently there is no `--help`, no `--version`, no shell-completion
generation, and no usage string beyond the fatal-error messages
(e.g. `"must specify either -bsau <key> (sign) or --verify <file> (verify)"`,
`git-sign-nostr/src/lib.rs:277`). This is consistent with both being
non-interactive helpers invoked exclusively by `git`, never typed by a human,
so the absence is unsurprising rather than a defect — flagged here as a
factual API-surface property, not a recommendation to add one.
