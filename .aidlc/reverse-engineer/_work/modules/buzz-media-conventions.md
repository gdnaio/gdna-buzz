## Module: buzz-media (`crates/buzz-media`)

### Aspect: Conventions

### 1. Module organization

Flat, one concern per file, all modules `pub` and re-exported selectively (`crates/buzz-media/src/lib.rs:5-28`):

| File | LOC | Concern |
|---|---|---|
| `src/lib.rs` | 29 | module declarations + curated re-exports |
| `src/types.rs` | 31 | wire response type (`BlobDescriptor`) |
| `src/thumbnail.rs` | 51 | sync CPU-bound derived artifacts |
| `src/config.rs` | 187 | config struct + startup validation |
| `src/error.rs` | 198 | error enum + HTTP mapping |
| `src/storage.rs` | 404 | S3 client and key builders |
| `src/upload_record.rs` | 419 | moderation side-channel records |
| `src/auth.rs` | 552 | Blossom kind-24242 verification |
| `src/upload.rs` | 732 | upload pipelines (orchestration) |
| `src/bucket_index.rs` | 755 | key taxonomy + pure accounting fold |
| `src/validation.rs` | 2594 | all content validation |
| `tests/static_creds_minio.rs` | 75 | live MinIO round-trip (`#[ignore]`) |

Layering is one-directional: `upload.rs` orchestrates and depends on `auth`, `config`, `error`, `storage`, `thumbnail`, `types`, `upload_record`, `validation` (`crates/buzz-media/src/upload.rs:1-20`); `validation.rs` and `bucket_index.rs` have no S3 dependency at all (`crates/buzz-media/src/bucket_index.rs:4-6`).

Notable structural convention: `bucket_index.rs` is explicitly written as I/O-free pure logic with an injected page-fetcher so it can be driven by synthetic listings in tests (`crates/buzz-media/src/bucket_index.rs:4-14`).

---

### 2. Naming

| Pattern | Examples |
|---|---|
| `validate_*` for fail-closed checks returning `Result` | `validate_content`, `validate_file_content`, `validate_video_file`, `validate_image_metadata_free`, `validate_jpeg_metadata_free`, `validate_mp4_metadata_free` (`crates/buzz-media/src/validation.rs:159`, `:238`, `:289`, `:492`, `:502`, `:831`) |
| `looks_like_*` for boolean structural probes | `looks_like_iso_bmff`, `looks_like_mp4_iso_bmff` (`crates/buzz-media/src/validation.rs:48`, `:52`) |
| `verify_blossom_*` for auth | `verify_blossom_auth_event_for_verb`, `verify_blossom_auth_event`, `verify_blossom_upload_auth`, `verify_blossom_get_auth` (`crates/buzz-media/src/auth.rs:31`, `:147`, `:175`, `:207`) |
| `process_*_upload` for pipelines | `process_upload`, `process_file_upload`, `process_video_upload` (`crates/buzz-media/src/upload.rs:207`, `:245`, `:292`) |
| `*_key` for object-key builders | `sidecar_key`, `ctx_sidecar_key`, `upload_record_key` (`crates/buzz-media/src/storage.rs:183`, `:188`, `crates/buzz-media/src/upload_record.rs:181`) |
| `parse_*` returning `Option` for lenient parses | `parse_public_ip`, `parse_port`, `parse_thumb_key`, `parse_blob_key`, `parse_sidecar_key`, `parse_auxiliary_key`, `parse_canonical_uuid` (`crates/buzz-media/src/upload_record.rs:191`, `:197`, `crates/buzz-media/src/bucket_index.rs:129`-`:172`, `:112`) |
| `is_*` predicates | `is_sha256`, `is_blob_ext`, `is_ulid_charset`, `is_public_ip`, `is_snapshot_text_chunk` (`crates/buzz-media/src/bucket_index.rs:75`, `:84`, `:93`, `crates/buzz-media/src/upload_record.rs:207`, `crates/buzz-media/src/validation.rs:584`) |
| `_sync` suffix marks CPU-bound functions meant for `spawn_blocking` | `generate_image_metadata_sync` (`crates/buzz-media/src/thumbnail.rs:15`) |
| SCREAMING_SNAKE consts for policy tables and bounds | `ALLOWED_MIME_TYPES`, `BLOCKED_FILE_MIME_TYPES`, `MP4_BRANDS`, `PNG_SNAPSHOT_KEYWORDS`, `MAX_PIXELS`, `MAX_ATOMS`, `MAX_BOXES`, `MAX_BOX_DEPTH`, `MIN_SNIFF_BYTES`, `BUF`, `UPLOAD_RECORD_VERSION` |

---

### 3. Error handling

Single crate error enum `MediaError` (`crates/buzz-media/src/error.rs:8-86`), `thiserror`-derived, 35 variants, plus a separate `SweepError` for the accounting fold (`crates/buzz-media/src/bucket_index.rs:341-362`).

| Variant | Message | HTTP status |
|---|---|---|
| `UnknownContentType` | `unknown content type` | 415 |
| `DisallowedContentType(String)` | `disallowed content type: {0}` | 415 |
| `FileTooLarge { size: u64, max: u64 }` | `file too large: {size} bytes (max {max})` | 413 |
| `ImageTooLarge` | `image dimensions too large` | 413 |
| `InvalidImage` | `invalid image data` | 422 |
| `MetadataForbidden` | `media contains metadata or a non-canonical metadata channel` | 422 |
| `InvalidSignature` | `invalid signature` | 401 (generic) |
| `InvalidAuthKind` | `invalid auth event kind` | 401 |
| `InvalidAuthVerb` | `invalid auth verb` | 401 |
| `MissingTag(&'static str)` | `missing required tag: {0}` | 401 |
| `HashMismatch` | `hash mismatch` | 401 |
| `ServerMismatch` | `server mismatch` | 401 |
| `TokenExpired` | `token expired` | 401 |
| `TimestampOutOfWindow` | `timestamp out of window` | 401 |
| `StorageError(String)` | `storage error: {0}` | 500 (body flattened) |
| `Internal` | `internal error` | 500 |
| `NotFound` | `not found` | 404 |
| `MissingAuth` | `missing authorization header` | 401 |
| `InvalidAuthScheme` | `invalid authorization scheme` | 401 |
| `InvalidBase64` | `invalid base64 encoding` | 401 |
| `InvalidAuthEvent` | `invalid auth event` | 401 |
| `Unauthorized` | `unauthorized` | 401 |
| `InsufficientScope` | `insufficient scope` | 403 |
| `RelayMembershipRequired` | `relay membership required` | 403 |
| `TokenRevoked` | `token revoked` | 401 |
| `PubkeyMismatch` | `pubkey mismatch` | 401 |
| `UploadRateLimitExceeded` | `upload rate limit exceeded` | 429 |
| `UploadConcurrencyLimitReached` | `upload concurrency limit reached` | 429 |
| `WrongCodec` | `unsupported media codec: only H.264 video and AAC audio are accepted` | 415 |
| `DurationTooLong` | `video too long: duration exceeds 600 seconds` | 422 |
| `ResolutionTooHigh` | `video resolution too high: maximum is 3840x2160` | 422 |
| `MoovNotAtFront` | `moov atom not at front of file (not fast-start)` | 422 |
| `UnsupportedContainer` | `unsupported container: only MP4 is accepted` | 415 |
| `InvalidVideo` | `invalid video data` | 422 |
| `Io(String)` | `io error: {0}` | 500 |

Conventions visible in the mapping (`crates/buzz-media/src/error.rs:106-160`):
- All 15 authentication-failure variants collapse to a single `401 "authentication failed"` body, explicitly "to prevent oracle enumeration"; `InsufficientScope` stays 403 because it is authorization, not authentication (`crates/buzz-media/src/error.rs:120-146`).
- 5xx bodies are flattened to `"internal error"` and logged at `error` (`crates/buzz-media/src/error.rs:154-158`).
- Errors are converted, never `unwrap`ped: three `From` impls (`image::ImageError` → `InvalidImage`, `S3Error` → `StorageError`, `serde_json::Error` → `StorageError`) at `crates/buzz-media/src/error.rs:88-104`.
- `MediaConfig::validate` returns `Result<(), String>` (plain strings, not `MediaError`) because it is a startup check (`crates/buzz-media/src/config.rs:66`).
- Some variants are declared here but not constructed in this crate (`Unauthorized`, `TokenRevoked`, `PubkeyMismatch`, `RelayMembershipRequired`, `MissingAuth`, `InvalidAuthScheme`, `InvalidBase64`, `UploadRateLimitExceeded`, `UploadConcurrencyLimitReached`) — they exist for relay handlers that share the type; see the Debt aspect.

---

### 4. Async patterns

| Pattern | Usage |
|---|---|
| CPU-bound work always inside `tokio::task::spawn_blocking` | validation+hash+auth (`crates/buzz-media/src/upload.rs:79-89`), video auth (`crates/buzz-media/src/upload.rs:410-414`), MP4 validation (`crates/buzz-media/src/upload.rs:416-419`), thumbnail (`crates/buzz-media/src/upload.rs:518-524`) |
| Join errors mapped, never unwrapped | `.map_err(\|_\| MediaError::Internal)??` (double `?` over join + inner result) at `crates/buzz-media/src/upload.rs:87-88`, `:414`, `:419`, `:524` |
| Owned inputs cloned into blocking closures; `Bytes` clones are refcount bumps (documented) | `crates/buzz-media/src/upload.rs:193-201` |
| Generic closure injection instead of trait objects for the two variable steps of the buffered pipeline | `crates/buzz-media/src/upload.rs:54-63` |
| Streaming rather than buffering for large payloads | `StreamReader` + 64 KiB chunks to temp file (`crates/buzz-media/src/upload.rs:325-395`), 8 MiB `BufReader` upload (`crates/buzz-media/src/storage.rs:91-101`), `ByteStream` download (`crates/buzz-media/src/storage.rs:131-146`) |
| Sync-blocking `std::fs` used deliberately inside blocking contexts | `crates/buzz-media/src/validation.rs:295`, `:415`, `:921` |
| Async page fetcher expressed as `FnMut(Option<String>) -> Future` | `crates/buzz-media/src/bucket_index.rs:377-383` |

---

### 5. Documentation conventions

- Every file opens with a `//!` module doc (`crates/buzz-media/src/lib.rs:1-3`, `crates/buzz-media/src/upload_record.rs:1-48`, `crates/buzz-media/src/bucket_index.rs:1-19`).
- All public items carry `///` doc comments; several document *why* a rule exists, including consumer contracts (`crates/buzz-media/src/upload_record.rs:29-48`) and threat rationale (`crates/buzz-media/src/validation.rs:66-86`).
- A markdown table inside module docs describes the key taxonomy (`crates/buzz-media/src/bucket_index.rs:12-19`).
- Spec references are inline: BUD-01 (`crates/buzz-media/src/auth.rs:201-205`, `crates/buzz-media/src/storage.rs:378`), BUD-02 (`crates/buzz-media/src/types.rs:1`), BUD-11 §5/§6 (`crates/buzz-media/src/auth.rs:47`, `:124`, `:189`).

---

### 6. Testing patterns

| Metric | Count |
|---|---|
| `#[test]` (sync) in `src/` | **98** |
| `#[tokio::test]` | **6** (5 in `crates/buzz-media/src/bucket_index.rs:538-661`, 1 in `crates/buzz-media/tests/static_creds_minio.rs:44`) |
| Total tests | **104** |
| `#[ignore]` | **1** — `crates/buzz-media/tests/static_creds_minio.rs:45` (`"requires a live MinIO (docker compose up -d minio minio-init)"`) |
| Integration test files | 1 (`tests/static_creds_minio.rs`) |
| Binary fixtures | 12 PNG/JPEG files under `crates/buzz-media/tests/fixtures/{android,ios}` |

Per-file distribution: `validation.rs` 47, `bucket_index.rs` 16 + 5 async, `auth.rs` 14, `upload_record.rs` 7, `config.rs` 4, `storage.rs` 4, `upload.rs` 4, `error.rs` 2; `lib.rs`, `types.rs`, `thumbnail.rs` have **zero**.

Patterns:
- `#[cfg(test)] mod tests` at the bottom of each file with a local `test_config()`/`valid_config()`/`storage_config()` builder (`crates/buzz-media/src/validation.rs:945-983`, `crates/buzz-media/src/config.rs:128-145`, `crates/buzz-media/src/storage.rs:281-301`, `crates/buzz-media/src/upload.rs:566-584`).
- Hand-built binary fixtures as consts (`TINY_JPEG`, `TINY_PNG`, `TINY_GIF`, `MP4_FTYP_MAGIC`, `TINY_PDF`, `TINY_ZIP`) plus ~30 MP4 box-builder helpers (`crates/buzz-media/src/validation.rs:1651-2150`).
- Real-device fixtures compiled in with `include_bytes!` to pin the mobile-sanitizer contract (`crates/buzz-media/src/validation.rs:1164-1280`).
- "Prove the fixture" pattern: an independent parser asserts the fixture really contains GPS EXIF before the validator is exercised (`crates/buzz-media/src/validation.rs:1044-1075`).
- Negative-first assertions via `matches!(result, Err(MediaError::X))` (`crates/buzz-media/src/validation.rs:1298-1305`).
- Mutation-resistance tests named for the property they defend, e.g. `same_sha_sidecars_do_not_bleed_between_communities` (`crates/buzz-media/src/storage.rs:352-377`) and `malformed_uploads_key_is_unknown_not_auxiliary` (`crates/buzz-media/src/bucket_index.rs:490-506`).
- Async tests drive the sweep fold through canned `Page`s and an `Arc<Mutex<Vec<Page>>>` script (`crates/buzz-media/src/bucket_index.rs:551-600`).
- Env-overridable integration config (`BUZZ_S3_ENDPOINT`/`ACCESS_KEY`/`SECRET_KEY`/`BUCKET`) in the ignored test only (`crates/buzz-media/tests/static_creds_minio.rs:22-34`).

---

### 7. Repo-guideline compliance

| Guideline (AGENTS.md) | Status in this crate |
|---|---|
| No `unsafe` | Satisfied — zero occurrences of `unsafe` across all 12 files |
| No new `unwrap()`/`expect()` in production paths | Mostly satisfied: 9 remaining non-test `.unwrap()` calls, all `try_into()` on fixed-size slices behind explicit length checks (`crates/buzz-media/src/validation.rs:604`, `:605`, `:673`, `:674`, `:697`, `:706`, `:707`, `:885`, `:886`); plus one `unwrap_or_default()` that intentionally swallows blurhash failure (`crates/buzz-media/src/thumbnail.rs:37`) |
| Public API must have doc comments | Satisfied for all public fns/types reviewed |
