## Module: buzz-core (`crates/buzz-core`)

### Aspect: Configuration

---

### 1. Environment variables

**None.** A recursive search of `crates/buzz-core/src` for `env::var` and `std::env` returns zero matches. The crate performs no environment reads, no file reads, and no config parsing — consistent with the zero-I/O charter stated at `crates/buzz-core/Cargo.toml:29`.

Runtime configuration for anything built on this crate (relay URL, keys, DB, Redis) is read by the consuming crates, not here.

---

### 2. Cargo features

| Feature | Declared | Default? | What it gates | Enabled by |
|---------|----------|----------|---------------|-----------|
| `test-utils` | `crates/buzz-core/Cargo.toml:10-11` (`test-utils = []`) | no (no `default` key exists) | `pub mod test_helpers` — `make_event` (`src/lib.rs:55`), `make_event_with_keys` (`:64`), `make_stored_event` (`:72`); the gate is `#[cfg(any(test, feature = "test-utils"))]` at `src/lib.rs:47` | `crates/buzz-relay/Cargo.toml:89`, inside that crate's `[dev-dependencies]` (`:86`) |

No other `#[cfg(feature = …)]` appears anywhere in `src/` — the only conditional compilation in the crate is `#[cfg(test)]` and the single `#[cfg(any(test, feature = "test-utils"))]`.

Notably, this crate has **no `dev` feature** (unlike `buzz-auth`, whose `dev` feature is switched on from `crates/buzz-relay/Cargo.toml:84`/`:90`).

---

### 3. Package metadata (all inherited from the workspace)

| Key | Value / source | file:line |
|-----|----------------|-----------|
| `name` | `buzz-core` | `crates/buzz-core/Cargo.toml:2` |
| `version` | `version.workspace = true` → `0.1.0` | `Cargo.toml:3`; root `Cargo.toml:35` |
| `edition` | `edition.workspace = true` → `2021` | `Cargo.toml:4`; root `Cargo.toml:36` |
| `rust-version` | `rust-version.workspace = true` → `1.88.0` | `Cargo.toml:5`; root `Cargo.toml:37` |
| `license` | `license.workspace = true` → `Apache-2.0` | `Cargo.toml:6`; root `Cargo.toml:38` |
| `repository` | `repository.workspace = true` → `https://github.com/block/sprout` | `Cargo.toml:7`; root `Cargo.toml:39` |
| `description` | "Core types, event verification, and filter matching for Buzz" | `Cargo.toml:8` |

Dependency versions are all workspace-inherited except one local pin: `percent-encoding = "2.3"` (`crates/buzz-core/Cargo.toml:26`).

---

### 4. Compile-time configuration (lints and assertions)

| Setting | Effect | file:line |
|---------|--------|-----------|
| `#![deny(unsafe_code)]` | any `unsafe` in the crate is a hard compile error | `src/lib.rs:1` |
| `#![warn(missing_docs)]` | undocumented public items warn | `src/lib.rs:2` |
| `const _: () = assert!(...)` × 25 | kind-range and u16-fit invariants are enforced at compile time; violating one fails the build | `src/kind.rs:707-744` |

No `[lints]` table exists in this crate's manifest and no `[workspace.lints]` exists in the root manifest, so lint configuration is entirely via these crate-level attributes.

---

### 5. Tunable constants (the crate's de-facto configuration surface)

These are compile-time constants, not runtime config. Changing one requires a rebuild of every dependent crate.

| Constant | Value | Governs | file:line |
|---|---|---|---|
| `EPHEMERAL_KIND_MIN` / `MAX` | 20000 / 29999 | ephemeral classification | `src/kind.rs:321`, `:323` |
| `PARAM_REPLACEABLE_KIND_MIN` / `MAX` | 30000 / 39999 | NIP-33 addressable classification | `src/kind.rs:316`, `:318` |
| `NIP44_MIN_CONTENT_LEN` | 132 | minimum accepted NIP-44 ciphertext length | `src/observer.rs:21` |
| `NIP44_MAX_CONTENT_LEN` | 87_472 | maximum accepted NIP-44 ciphertext length | `src/observer.rs:23` |
| `OBSERVER_MAX_PLAINTEXT_LEN` | 65_535 | observer plaintext cap | `src/observer.rs:25` |
| `OBSERVER_AGENT_TAG` / `OBSERVER_FRAME_TAG` | `"agent"` / `"frame"` | observer frame tag names | `src/observer.rs:13`, `:15` |
| `OBSERVER_FRAME_TELEMETRY` / `OBSERVER_FRAME_CONTROL` | `"telemetry"` / `"control"` | observer frame direction values | `src/observer.rs:17`, `:19` |
| `CORE_SLUG` | `"core"` | reserved engram slug | `src/engram.rs:20` |
| `D_TAG_DOMAIN` | `b"agent-memory/v1/d-tag"` | HMAC domain separator; doc says future revisions MUST change it | `src/engram.rs:22-24` |
| `NIP44_PLAINTEXT_MAX` | 65_535 | engram body cap | `src/engram.rs:28` |
| `SLUG_MAX_LEN` | 255 | engram slug byte cap (per-segment cap of 64 is an inline literal at `src/engram.rs:98`) | `src/engram.rs:31` |
| `MEM_PREFIX` (private) | `"mem/"` | engram slug namespace | `src/engram.rs:34` |
| `MAX_PROTECTION_RULES` | 50 | `buzz-protect` rules per repo | `src/git_perms.rs:19` |
| `MAX_PATTERN_LENGTH` | 256 | ref-pattern character cap | `src/git_perms.rs:21` |
| `MAX_WILDCARDS_PER_PATTERN` | 3 | wildcard segments per pattern | `src/git_perms.rs:23` |
| `ZERO_OID` (private, fn-local) | 40 `0` chars | create/delete detection | `src/git_perms.rs:213` |
| `DEFAULT_TIMEOUT` (private) | `Duration::from_secs(120)` | pairing session lifetime | `src/pairing/session.rs:43` |
| `PAIRING_KIND` (private) | `KIND_PAIRING as u16` = 24134 | pairing event kind | `src/pairing/session.rs:46` |
| `INFO_SESSION_ID` / `INFO_SAS` / `INFO_TRANSCRIPT` (private) | `"nostr-pair-session-id"`, `"nostr-pair-sas-v1"`, `"nostr-pair-transcript-v1"` | HKDF domain separation | `src/pairing/crypto.rs:23-25` |
| SAS modulus (inline literal) | `1_000_000` | 6-digit SAS space | `src/pairing/crypto.rs:73` |
| QR URI length cap (inline literal) | 2048 | max scanned URI length | `src/pairing/qr.rs:106` |
| pairing content-length window (inline literals) | `132..=87472` | inbound ciphertext gate — duplicates `observer::NIP44_MIN/MAX_CONTENT_LEN` rather than referencing them | `src/pairing/session.rs:595` |
| pairing plaintext cap (inline literal) | `65_535` | decrypted plaintext gate — duplicates `OBSERVER_MAX_PLAINTEXT_LEN` / `NIP44_PLAINTEXT_MAX` | `src/pairing/session.rs:609` |
| jitter window (inline literal) | `% 31` → 0–30 s | `created_at` metadata jitter | `src/pairing/session.rs:578` |

Non-configurable by design: nothing in the crate reads these from the environment, and there is no builder or options struct that would let a consumer override them at runtime.
