## Module: git-sign-nostr & git-credential-nostr (`crates/git-sign-nostr`, `crates/git-credential-nostr`)
### Aspect: Configuration

#### Overview

Configuration for both crates is entirely env-var- and git-config-driven —
neither reads a dedicated config file, neither has a CLI flag for
configuration values (the CLI flags each accepts are protocol arguments from
git, documented in the API Surface doc, not user-facing config). This doc
enumerates every env var and git-config key actually read in code, with
default, read site, and documentation status.

#### Environment variables

| Variable | Default (if unset) | Read in `git-sign-nostr` | Read in `git-credential-nostr` | Documented in `.env.example` | Documented in `AGENTS.md` | Documented in `NOSTR.md` | Documented in crate README |
|---|---|---|---|---|---|---|---|
| `NOSTR_PRIVATE_KEY` | none — falls through to next source | `load_key`, `crates/git-sign-nostr/src/lib.rs:399-415` | `load_key`, `crates/git-credential-nostr/src/lib.rs:51-56` | no | no | no | yes, both crate READMEs (`git-sign-nostr/README.md` § Usage, `git-credential-nostr/README.md` § CI/CD) |
| `BUZZ_PRIVATE_KEY` | none — falls through to next source | `load_key`, `git-sign-nostr/src/lib.rs:417-436` | **not read at all** | no | yes, `AGENTS.md:164` (as an ACP-harness var, not specifically this crate) | no | no (neither crate README mentions it) |
| `BUZZ_AUTH_TAG` | none — `oa` omitted | `load_auth_tag`, `git-sign-nostr/src/lib.rs:468-471` | `load_auth_tag`, `git-credential-nostr/src/lib.rs:79-81` | no | yes, `AGENTS.md:164` | no | yes, `git-sign-nostr/README.md` § Usage (as an example export) |

Every one of these three env vars is consumed via a plain
`std::env::var("...")` call at the sites above — confirmed by grep, no other
env var is referenced in either crate's `src/`.

#### Git config keys

| Key | Default | Read in `git-sign-nostr` | Read in `git-credential-nostr` | Documented in `.env.example` | Documented in `AGENTS.md` | Documented in `NOSTR.md` | Documented in crate README |
|---|---|---|---|---|---|---|---|
| `nostr.keyfile` | unset → error "no key available"/"no nostr key configured" | `load_key`, `git-sign-nostr/src/lib.rs:437-441` | `load_key`, `git-credential-nostr/src/lib.rs:58` | no | no | no | yes, both READMEs (`git-sign-nostr/README.md` § Key Loading Priority, `git-credential-nostr/README.md` § Setup) |
| `nostr.authtag` | unset → `oa` omitted | `load_auth_tag` via `git_config_strict`, `git-sign-nostr/src/lib.rs:472-473` | `load_auth_tag` via `git_config`, `git-credential-nostr/src/lib.rs:82` | no | no | no | no (neither README documents this fallback explicitly, though `git-sign-nostr/README.md`'s "Optional: NIP-OA owner attestation" section only shows the env-var form) |
| `user.signingkey` | unset → `TRUST_UNDEFINED` (verify); ignored if empty (sign) | `determine_trust`, `git-sign-nostr/src/lib.rs:1673-1682`; also the `-u <key>` argv value in `do_sign`, `lib.rs:980-993` | not read | n/a (standard git config) | n/a | n/a | yes, `git-sign-nostr/README.md` § Usage |

Note on read mechanism: both crates read git config by shelling out to
`git config --get <key>` (`git_config` functions,
`git-sign-nostr/src/lib.rs:661-680`,
`git-credential-nostr/src/lib.rs:16-25`) rather than parsing `.git/config`
directly — this is standard practice (respects git's own config layering:
system/global/local/worktree/`GIT_CONFIG_*` env overrides) but means both
crates' "configuration" is transitively also configurable via any mechanism
git itself supports, including the `GIT_CONFIG_COUNT`/`GIT_CONFIG_KEY_*`/
`GIT_CONFIG_VALUE_*` env-var scheme NIP-GS explicitly recommends for
ephemeral per-process config (`docs/nips/NIP-GS.md` § Git Configuration) —
this is exactly the mechanism `buzz-dev-mcp`'s `shim.rs:186-213` uses to set
`nostr.keyfile`, `user.signingkey`, etc. without touching any actual
`.gitconfig` file.

#### Parsed-but-never-read / undocumented config: none found

No config value is parsed and then discarded without use in either crate —
every env var and git-config key listed above is read exactly where it
affects behavior, and every read site's result is consumed (either used
directly or checked and reported as an error). No dead configuration
variable was found by inspection of both `src/` trees.

The closest thing to "undocumented config" is `nostr.authtag`: it is a real,
functioning fallback in both crates' `load_auth_tag` (see table above), but
it is not mentioned in either crate's `README.md`, nor in `NOSTR.md`, nor in
`AGENTS.md` — only the `BUZZ_AUTH_TAG` env-var form is documented anywhere.
A user relying solely on the published READMEs would not discover that
`git config nostr.authtag '["auth",...]'` is a valid alternative to setting
the environment variable. This is flagged as a genuine documentation gap
distinct from a code defect.

#### `.env.example` coverage

Checked directly: `grep -n -i 'NOSTR_PRIVATE_KEY\|nostr\.keyfile\|BUZZ_AUTH_TAG\|nostr\.authtag'
.env.example` returns **zero matches** — none of the three env vars these
two crates read, and none of the three git-config keys, appear anywhere in
the repo's root `.env.example`. This is consistent with `.env.example`'s own
scope (it documents relay/backend/ACP-harness configuration —
`DATABASE_URL`, `REDIS_URL`, `BUZZ_ACP_*`, etc. — none of which these two
git-helper binaries read), so the absence is expected rather than an
oversight: these two crates are invoked by `git`, not by the relay/ACP
process tree `.env.example` configures, and are documented instead in their
own crate-local `README.md` files. `BUZZ_AUTH_TAG` specifically is
documented in `AGENTS.md:164` as one of three vars "auto-injected by the ACP
harness into managed agent subprocesses" — accurate for the ecosystem as a
whole (an ACP-managed agent does get `BUZZ_AUTH_TAG` injected, and
`git-sign-nostr`/`git-credential-nostr` do consume it if present), but that
line in `AGENTS.md` is about the ACP harness's injection behavior, not
specifically about these two crates, so it should not be read as this
module's own configuration documentation.

#### Compile-time configuration (`Cargo.toml` features)

Neither crate exposes a Cargo feature flag (confirmed: no `[features]`
section in either `Cargo.toml`). `git-sign-nostr`'s `Cargo.toml` enables the
`zeroize` workspace dependency's `derive` feature explicitly
(`features = ["derive"]`, `crates/git-sign-nostr/Cargo.toml:26`) — see Debt
doc for whether this feature is actually exercised (no `#[derive(Zeroize)]`
was found anywhere in the crate; only `Zeroizing<T>`/`.zeroize()` method
calls are used, which come from `zeroize`'s base functionality, not the
`derive` feature). `git-credential-nostr`'s `Cargo.toml` depends on
`zeroize = { workspace = true }` with no extra features
(`crates/git-credential-nostr/Cargo.toml:11`).

#### Platform-conditional configuration

Both crates compile different code on Unix vs. non-Unix via
`#[cfg(unix)]`/`#[cfg(not(unix))]`, but this is a compile-time target
selection, not a runtime-configurable value — listed here for completeness
since it does affect *behavior* per-platform:

- `git-sign-nostr/Cargo.toml:38-42` declares `libc = "0.2"` as a
  `[target.'cfg(unix)'.dependencies]`-scoped dependency — on non-Unix
  targets the crate would fail to build the fd-passing status-fd logic at
  all in the same way (the module doc states the crate is "Unix-only",
  `lib.rs:13`); there is no non-Unix `#[cfg(not(unix))]` arm for
  `StatusWriter::new`'s fd-handling path (`lib.rs:302-354`; the
  `#[cfg(not(unix))]` arm at `:352-355` simply drops to `None`/stderr, so the
  crate *does* compile on non-Unix but loses the real status-fd
  functionality git relies on for GnuPG-protocol signaling).
- `git-credential-nostr` has a real (if reduced) non-Unix behavior for
  keyfile permission checking: `check_keyfile_permissions`
  (`crates/git-credential-nostr/src/lib.rs:41-44`) prints a warning and
  returns `Ok(())` unconditionally on non-Unix, rather than failing to
  compile or silently skipping the check with no signal.

#### Summary of gaps

- **`nostr.authtag` git-config fallback is real but undocumented** in any
  README, `AGENTS.md`, or `NOSTR.md` — only the `BUZZ_AUTH_TAG` env var form
  is published.
- **`BUZZ_PRIVATE_KEY` is asymmetric**: read by `git-sign-nostr`, not read by
  `git-credential-nostr` — neither crate's README calls out this difference,
  and a reader of `AGENTS.md`'s ecosystem-level "auto-injected" env-var list
  (`BUZZ_RELAY_URL`, `BUZZ_PRIVATE_KEY`, `BUZZ_AUTH_TAG`,
  `AGENTS.md:163-164`) would not learn that `BUZZ_PRIVATE_KEY` only helps
  half of this module's two crates.
- **No config value found parsed-but-unused** in either crate.
