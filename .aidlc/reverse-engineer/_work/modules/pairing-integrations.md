## Module: buzz-pair-relay & buzz-pairing-cli (`crates/buzz-pair-relay`, `crates/buzz-pairing-cli`)
### Aspect: Integrations

#### buzz-pair-relay's relationship to buzz-relay: fully separate process, zero shared code

`buzz-pair-relay` is **not** a proxy or forwarder in front of the main `buzz-relay` — it is a completely independent binary with its own `hyper` + `tokio-tungstenite` HTTP/WebSocket stack (`crates/buzz-pair-relay/Cargo.toml:18-30`), built from lower-level primitives rather than reusing `buzz-relay`'s connection-handling code. Confirmed by grepping the entire `crates/buzz-relay` source tree for "pair" (case-insensitive): zero matches — the main relay has no route, module, or handler that mentions pairing at all, and a targeted grep of `crates/buzz-relay/src/router.rs` for `/pair` also returns zero matches. The two relays share:
- **The wire format only** (NIP-01 `REQ`/`EVENT`/`CLOSE`/`OK`/`EOSE`/`NOTICE`/`CLOSED`, kind:24134 events) — not any Rust type or trait.
- **Nothing from `buzz-core`** — `buzz-pair-relay`'s `Cargo.toml` does not depend on `buzz-core` at all (absent from its dependency list, `Cargo.toml:18-30`), so it cannot and does not import `buzz_core::kind::KIND_PAIRING`; instead it hardcodes the same value locally as `const KIND_PAIR: u64 = 24134` (`lib.rs:63`). It also reimplements its own NIP-44 envelope sanity checker (`validate_nip44_content`, `lib.rs:343-411`) rather than depending on `buzz_core::observer::content_looks_like_nip44` (`crates/buzz-core/src/observer.rs:21-23`, `53-55`) — see Debt for how this duplication plays out across a third crate too.
- **Cryptographic primitives directly from `secp256k1`/`sha2`**, not through `buzz-core::pairing::crypto` or `buzz-core::verification` — `buzz-pair-relay` implements its own hex decode/encode and its own NIP-01 commitment-hash + Schnorr verification (`verify_event_sig`, `lib.rs:521-555`), entirely independent of `buzz_core::verification` (which `buzz-core::pairing::session::validate_event_basics` uses via `event.verify()`, `session.rs:632-634`). This is a defensible design choice — `buzz-pair-relay` deliberately avoids depending on `buzz-core` to keep the sidecar's footprint minimal and independently auditable, and its own doc comment frames "Signature verification" as a first-class part of its security model (`lib.rs:19-21`) — but it means any future change to NIP-01 verification semantics in `buzz-core::verification` would not automatically propagate here.

Net: the two relays are architecturally siblings, not parent/child. Per the Helm chart, `buzz-pair-relay` is deployed as its own `Deployment`/`Service` (`deploy/charts/buzz/templates/pairing-relay.yaml:3-4`, `53-54`), built as a separate binary in the same container image (`Dockerfile:67-72`, `140`), and the main relay only *references* it by URL — `buzz-relay`'s config carries an optional `pairing_relay_url: Option<String>` read from `BUZZ_PAIRING_RELAY_URL` (`crates/buzz-relay/src/config.rs:70`, `430-446`) purely to **advertise** it in the NIP-11 document (`crates/buzz-relay/src/nip11.rs:52-54`, `243`). The main relay never opens a connection to the pairing relay — it is advertisement-only.

#### buzz-pairing-cli's dependency graph: this is where the real coupling lives

Unlike `buzz-pair-relay`, `buzz-pairing-cli` depends directly on `buzz-core` (`crates/buzz-pairing-cli/Cargo.toml:15`) and re-exports almost none of its own logic — it is a thin CLI/IO shell around `buzz_core::pairing`:

| Dependency | Role | Citation |
|---|---|---|
| `buzz-core` (workspace path dep) | `kind::KIND_PAIRING`, the entire `pairing::{session, crypto, qr, types}` module + `PairingError` | `main.rs:18-23`, `Cargo.toml:15` |
| `nostr` (workspace, `0.44`) | `Event`, `EventBuilder`, `Keys`, `RelayUrl`, `SecretKey`, `ToBech32` — building/signing/parsing Nostr events, including the NIP-42 `EventBuilder::auth` helper | `main.rs:25-26`, used at `main.rs:441`, `454` |
| `tokio-tungstenite` (workspace) | Raw WebSocket client transport to whichever relay URL is supplied | `main.rs:27`, `main.rs:132`, `212` |
| `clap` (direct, version `"4"`, not workspace-pinned) | CLI parsing | `Cargo.toml:19` |
| `zeroize` (workspace) | Wraps secrets in `Zeroizing<String>` | `main.rs:28`, used in `resolve_payload` |
| `thiserror` (workspace) | `CliError` enum | `main.rs:74-97` |
| `hex` (workspace) | test-vectors hex encode/decode | `main.rs:405-407`+ |

Notably **absent**: no dependency on `buzz-pair-relay` itself, and no dependency on `buzz-ws-client` — the shared NIP-42 WebSocket client crate `AGENTS.md`'s repo-structure table describes as "Shared NIP-42 WebSocket client (connect, auth, publish)." `buzz-pairing-cli` reimplements its own NIP-42 auth handshake (`handle_nip42_auth`, `main.rs:416-478`) and its own EOSE/event-wait loops (`wait_for_eose`, `main.rs:529-558`; `wait_for_event`, `main.rs:503-527`) from raw `tokio_tungstenite` primitives rather than reusing `buzz-ws-client`. This diverges from the pattern `AGENTS.md` describes for `buzz-cli` ("wire the REST/WebSocket call in `client.rs`") — `buzz-pairing-cli` has no equivalent shared client module.

#### Cross-referencing the desktop and mobile pairing implementations (outside this scope, but load-bearing context)

Both `desktop/src-tauri/src/commands/pairing.rs` and `mobile/lib/features/pairing/*.dart` implement **their own** versions of the NIP-AB transport plumbing:

- **Desktop** reuses `buzz_core_pkg::pairing::session::PairingSession` directly — `desktop/src-tauri/Cargo.toml` depends on `buzz-core` under the local alias `buzz_core_pkg = { package = "buzz-core", path = "../../crates/buzz-core" }`, and `pairing.rs` imports `buzz_core_pkg::pairing::{qr::encode_qr, session::PairingSession, types::{AbortReason, PayloadType}}` (`desktop/src-tauri/src/commands/pairing.rs:5-8`). So desktop and `buzz-pairing-cli` genuinely share the state-machine and crypto implementation — only the transport/NIP-42 glue code is separately written (see below).
- **Mobile** does not link any Rust code — it reimplements the HKDF derivations, SAS formatting, and NIP-42 handshake natively in Dart. `mobile/lib/features/pairing/pairing_crypto.dart` exists as the crypto reimplementation, and `mobile/lib/features/pairing/pairing_socket.dart` independently reimplements the NIP-42 challenge/response and EOSE-wait choreography, with its own timeout constants: `_desktopPairingAuthChallengeGrace = Duration(seconds: 3)` and `_pairingAuthOkTimeout = Duration(seconds: 8)` (`pairing_socket.dart:9-10`). This is a legitimate necessity — Dart cannot link a Rust crate directly — but it means the NIP-AB protocol has two independent implementations of the same cryptographic derivations and handshake timing in the codebase, and nothing enforces that they stay in sync beyond the shared `NIP-AB.md` test vectors (which the Rust side pins in unit tests, `crates/buzz-core/src/pairing/crypto.rs:239-273`, but the Dart implementation was not confirmed to carry an equivalent test-vector assertion in this pass).

#### NIP-42 handshake timing across the three Rust/Dart implementations — confirmed values

| Implementation | Challenge-wait timeout | OK-wait timeout | Citation |
|---|---|---|---|
| `buzz-pairing-cli` | 3s | 5s | `crates/buzz-pairing-cli/src/main.rs:428`, `461` |
| Desktop (`pairing.rs`) | 3s | 5s | `desktop/src-tauri/src/commands/pairing.rs:393`, `431` |
| Mobile (`pairing_socket.dart`) | 3s | **8s** | `mobile/lib/features/pairing/pairing_socket.dart:9-10` |

The CLI and desktop agree exactly (3s/5s); the mobile OK-wait timeout (8s) differs from both. This is a small, real behavioral drift between three independently-maintained copies of what should be the same handshake — see Debt.

#### Deployment-level integration

- **Docker image**: `buzz-pair-relay` is compiled and shipped in the same multi-stage image as `buzz-relay`/`buzz-admin` (`Dockerfile:67-72`, copied to `/usr/local/bin/buzz-pair-relay` at `Dockerfile:140`) — same release artifact, different entrypoint.
- **Helm chart**: an optional, independently-scaled `Deployment` + `Service` gated by `pairingRelay.enabled` (default `false`, `deploy/charts/buzz/values.yaml:194`), with its own resource requests/limits (`values.yaml:205-211`) separate from the main relay's. `BUZZ_PAIR_RELAY_BIND_ADDR` is injected as `0.0.0.0:{{ .Values.pairingRelay.service.port }}` (`deploy/charts/buzz/templates/pairing-relay.yaml:37-38`).
- **Main relay awareness**: purely advertisement-level via NIP-11's `pairing_relay_url` field (see above). The desktop client (`desktop/src-tauri/src/commands/pairing.rs:482-512`, `probe_pairing_relay`/`pairing_relay_from_nip11`) probes the main relay's NIP-11 document to decide between a configured pairing-relay URL, a "legacy `/pair`" same-host path, or the main relay itself. **The "legacy `/pair` path" fallback that the desktop code and the Helm chart README both describe does not actually exist in `buzz-relay`'s router** — confirmed by reading `crates/buzz-relay/src/router.rs` and finding no `/pair` route registered anywhere, and by a repo-wide case-insensitive grep for "pair" inside `crates/buzz-relay/` returning zero matches. This is a real contradiction between documented/coded client-side fallback behavior and server-side reality; see Debt.
- **`buzz-pairing-cli` has no deployment artifact of its own** — it is not copied into the runtime Docker image (absent from `Dockerfile`'s `COPY --from=builder` lines, which list only `buzz-relay`, `buzz-admin`, `buzz-pair-relay`) and is not referenced anywhere under `deploy/`. It exists purely as a developer/interop-testing tool, matching its README's stated purpose.

#### What this module group does *not* integrate with

No database (`buzz-db`), no Redis (`buzz-pubsub`), no auth crate (`buzz-auth`), no search (`buzz-search`), no audit log (`buzz-audit`), no media storage (`buzz-media`) — confirmed by the absence of any of these in either crate's `Cargo.toml`. This is consistent with the "ephemeral, stateless" framing both crates use to describe themselves.
