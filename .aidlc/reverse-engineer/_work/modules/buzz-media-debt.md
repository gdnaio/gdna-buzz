## Module: buzz-media (`crates/buzz-media`)

### Aspect: Technical Debt

### 1. Complexity hotspots

| File | LOC | Non-test share | Note |
|---|---|---|---|
| `src/validation.rs` | 2594 | ~940 lines of logic; the `#[cfg(test)] mod tests` starts at `crates/buzz-media/src/validation.rs:941` and runs to EOF (~1650 lines, 64% of the file) | Single file holds five hand-written binary-format parsers plus three public validators |
| `src/bucket_index.rs` | 755 | logic ends ~`crates/buzz-media/src/bucket_index.rs:411`; tests `:413-661` | Well-separated pure logic |
| `src/upload.rs` | 732 | logic ends at `crates/buzz-media/src/upload.rs:560`; tests `:562-731` | Contains the longest function in the crate |
| `src/auth.rs` | 552 | logic `:1-236`; tests `:241-551` | |
| `src/upload_record.rs` | 419 | logic `:1-256`; tests `:258-418` | |
| `src/storage.rs` | 404 | logic + tests interleaved (tests at `:269-377`, then two public structs at `:378-403`) | Type definitions placed *after* the test module — unusual layout |

Largest functions:

| Function | Approx. lines | file:line |
|---|---|---|
| `process_video_upload` | ~220 | `crates/buzz-media/src/upload.rs:292-511` |
| `validate_video_file` | ~107 | `crates/buzz-media/src/validation.rs:289-395` |
| `validate_mp4_metadata_free` (incl. nested `walk`) | ~98 | `crates/buzz-media/src/validation.rs:831-928` |
| `verify_blossom_auth_event_for_verb` | ~115 | `crates/buzz-media/src/auth.rs:31-145` |
| `check_moov_before_mdat` | ~77 | `crates/buzz-media/src/validation.rs:408-484` |
| `validate_jpeg_metadata_free` | ~77 | `crates/buzz-media/src/validation.rs:502-578` |
| `validate_webp_metadata_free` (incl. nested fn) | ~74 | `crates/buzz-media/src/validation.rs:659-732` |
| `validate_gif_metadata_free` (incl. nested fn) | ~96 | `crates/buzz-media/src/validation.rs:734-829` |
| `process_buffered_upload` | ~139 | `crates/buzz-media/src/upload.rs:54-192` |
| `BucketAggregate::finish` | ~61 | `crates/buzz-media/src/bucket_index.rs:277-337` |

`process_video_upload` is a 7-step inline pipeline (stream+hash, container check, auth, validation, idempotency, store, sidecar) with a large nested block for step 1 (`crates/buzz-media/src/upload.rs:322-398`) — the natural extraction candidate, and the one path *not* covered by the shared `process_buffered_upload` refactor that unified the other two.

---

### 2. Dead / unused code

| Item | Evidence |
|---|---|
| 9 `MediaError` variants are never constructed in this crate: `MissingAuth`, `InvalidAuthScheme`, `InvalidBase64`, `Unauthorized`, `RelayMembershipRequired`, `TokenRevoked`, `PubkeyMismatch`, `UploadRateLimitExceeded`, `UploadConcurrencyLimitReached` (`crates/buzz-media/src/error.rs:44-63`, `:67-74`) — they exist for relay handlers that reuse the type (e.g. `crates/buzz-relay/src/api/media.rs:88-140`, `:887`). Legitimate shared-error-type coupling, but it means the crate's error enum is not self-contained. |
| `MediaError::TokenRevoked` in particular has **no revocation mechanism anywhere in this crate** — no nonce/jti store exists (`crates/buzz-media/src/auth.rs:31-236`). |
| `MediaStorage::delete` has no in-crate caller — no upload path, GC, or sweep uses it (`crates/buzz-media/src/storage.rs:158-164`); only the ignored MinIO test exercises it (`crates/buzz-media/tests/static_creds_minio.rs:71-74`). |
| `MediaStorage::get` has no in-crate caller either; `get_sidecar` uses `bucket.get_object` directly rather than going through it (`crates/buzz-media/src/storage.rs:197-199`) — a small inconsistency that duplicates the 404-mapping logic omission. |
| `MediaStorage::head_with_metadata`, `get_range`, `get_stream`, `list_page` are all relay-only consumers — fine, but nothing in-crate tests them (see coverage gaps). |
| `BlossomVerb::Get` / `verify_blossom_get_auth` are unused within the crate (`crates/buzz-media/src/auth.rs:207-236`); only the relay calls them (`crates/buzz-relay/src/api/media.rs:502`). |
| `MediaConfig::validate` is never called inside the crate (`crates/buzz-media/src/config.rs:66`) — startup enforcement depends on the relay invoking it. |
| `looks_like_iso_bmff` is public and used by the relay (`crates/buzz-relay/src/api/media.rs:50`), while `looks_like_mp4_iso_bmff` is `pub(crate)` — the two near-identical names with different visibility is a readability trap (`crates/buzz-media/src/validation.rs:48-62`). |

---

### 3. Test coverage gaps

104 tests total (98 `#[test]`, 6 `#[tokio::test]`, 1 `#[ignore]`). Distribution is heavily skewed toward validation.

| Gap | Evidence |
|---|---|
| **No test executes any upload pipeline.** `process_upload`, `process_file_upload`, `process_video_upload` have zero direct tests — `upload.rs`'s 4 tests cover only `build_descriptor` (×3) and the body-limit substring heuristic (`crates/buzz-media/src/upload.rs:565-730`). The 7-step ordering contract (blob → thumb → record → sidecar), the idempotent short-circuit, and the orphan-blob-on-failure behaviour are therefore untested in-crate. |
| **No test exercises `MediaStorage` against a store** except the `#[ignore]`d MinIO round-trip (`crates/buzz-media/tests/static_creds_minio.rs:44-75`). The 4 in-crate `storage.rs` tests cover credential selection and key formatting only (`crates/buzz-media/src/storage.rs:302-377`). |
| **404/error mapping untested**: the `HttpFailWithBody(404, _)` branches in `get`, `get_range`, `head`, `head_with_metadata`, and the `status_code == 404` check in `get_stream` have no coverage (`crates/buzz-media/src/storage.rs:105-175`). |
| `get_range`, `get_stream`, `put_file`, `list_page` have **no tests at all** (`crates/buzz-media/src/storage.rs:85-265`); `list_page`'s conversion into `Page` is the one place where the otherwise-pure sweep meets S3 and it is unverified. |
| `thumbnail.rs` has **zero tests** (`crates/buzz-media/src/thumbnail.rs:1-51`) — the 320px/blurhash/thumb-URL contract and the non-image early return are unverified. |
| `types.rs` and `lib.rs` have zero tests (serde renames like `type`/skip-if-none are covered indirectly via `upload.rs`'s descriptor tests). |
| `record_upload_event` itself is untested (`crates/buzz-media/src/upload_record.rs:139-178`); only the record's serialization shape, key format, and IP/port parsing are (`crates/buzz-media/src/upload_record.rs:258-417`). |
| `MediaError::into_response` covers only the 415 and 422 groups (`crates/buzz-media/src/error.rs:163-198`) — the collapsed-401 behaviour, the 403 split, 413, 429, and the 5xx body flattening are untested despite being explicit security decisions. |
| Video-path negative tests exist for `validate_video_file` but not for the streaming wrapper: the `MIN_SNIFF_BYTES` accumulation, the Content-Length fast-fail, and the mid-stream running-total cap are untested (`crates/buzz-media/src/upload.rs:311-395`). |
| The ignored MinIO test is the only integration coverage; the AWS-credential-chain branch (`Credentials::default()`) is intentionally never exercised (`crates/buzz-media/src/storage.rs:51-56`). |

---

### 4. Fragile / risky constructs

| # | Item | file:line |
|---|---|---|
| D-1 | Body-limit detection by matching three Display substrings, because "axum wraps LengthLimitError in its error chain but doesn't expose the inner type for downcasting" — a dependency-wording dependency. The guard test only proves the current strings, not that axum still produces one of them | `crates/buzz-media/src/upload.rs:328-341`, test `crates/buzz-media/src/upload.rs:685-712` |
| D-2 | `EMPTY_FFMPEG_UDTA` hardcodes 53 exact bytes of a specific ffmpeg output; any encoder change breaks uploads with a 422 | `crates/buzz-media/src/validation.rs:834-839`, `:895-903` |
| D-3 | Metadata policy is a strict allowlist over five binary formats, hand-implemented. Any new legitimate chunk/box from a client encoder version bump becomes a hard rejection (already visible in the fixture pairs: unsanitized iOS/Android output is rejected, sanitized accepted — `crates/buzz-media/src/validation.rs:1203-1280`) | `crates/buzz-media/src/validation.rs:487-928` |
| D-4 | `check_moov_before_mdat` uses unchecked `offset += atom_size` on an attacker-supplied 64-bit size, while the sibling `validate_mp4_metadata_free` walker uses `checked_add` — inconsistent overflow discipline | `crates/buzz-media/src/validation.rs:481` vs `:891-893` |
| D-5 | Two independent MP4 traversals (`check_moov_before_mdat` and `validate_mp4_metadata_free::walk`) each re-open and re-scan the file, plus a third parse by the `mp4` crate — three passes over the same temp file per video upload | `crates/buzz-media/src/validation.rs:291-307`, `:408`, `:831` |
| D-6 | Duration (`600.0`) and resolution (`3840`/`2160`) limits are inline literals rather than named constants or config, unlike every byte cap | `crates/buzz-media/src/validation.rs:357`, `:364` |
| D-7 | `max_image_bytes`/`max_gif_bytes` lack `#[serde(default)]` while `max_video_bytes`/`max_file_bytes`/`s3_region` have one — inconsistent defaulting policy inside one struct | `crates/buzz-media/src/config.rs:38-50` |
| D-8 | Auth windows (600 s / 3600 s) are magic numbers passed at call sites rather than config or named constants | `crates/buzz-media/src/upload.rs:85`, `:412` |
| D-9 | `BlobHeadMeta`/`BlobMeta` are declared *after* the `#[cfg(test)] mod tests` block in `storage.rs`, splitting the type definitions from the impl | `crates/buzz-media/src/storage.rs:268-403` |
| D-10 | 9 non-test `.unwrap()` calls on `try_into()` of fixed-size slices. All are provably infallible given the preceding bounds checks, but they violate the repo's "no new unwrap in production paths" guideline and would be free to convert | `crates/buzz-media/src/validation.rs:604`, `:605`, `:673`, `:674`, `:697`, `:706`, `:707`, `:885`, `:886` |
| D-11 | `blurhash::encode(...).unwrap_or_default()` silently substitutes an empty blurhash on failure | `crates/buzz-media/src/thumbnail.rs:36-37` |
| D-12 | `content_length.unwrap_or(0)` in `head_with_metadata` conflates "absent header" with "zero bytes" | `crates/buzz-media/src/storage.rs:170-172` |
| D-13 | Unchecked `+=` accumulation of `u64` byte totals in the sweep aggregate | `crates/buzz-media/src/bucket_index.rs:251-252` |
| D-14 | `SweepError::Timeout` is defined here but only ever constructed by the relay — an error variant whose producer lives in another crate (documented as deliberate, still a coupling smell) | `crates/buzz-media/src/bucket_index.rs:349-357` |
| D-15 | `get_sidecar` bypasses `MediaStorage::get`, so it does not get the 404→`NotFound` mapping and surfaces a generic `StorageError` for a missing sidecar | `crates/buzz-media/src/storage.rs:193-202` vs `:105-111` |
| D-16 | `read_sidecar_mime` splits on `'.'` and falls back to the whole string (`split('.').next().unwrap_or(sha256_ext)`) — tolerant parsing at a security-relevant seam (the tenant gate), relying on the relay's `validate_media_path` upstream | `crates/buzz-media/src/storage.rs:227-228` |

---

### 5. Functional gaps carried as debt

| Gap | Evidence |
|---|---|
| No GC / orphan cleanup — orphan blobs are knowingly accumulated ("A V2 background GC job can sweep blobs with no matching sidecar after a grace period") | `crates/buzz-media/src/upload.rs:122-127` |
| No retention/TTL logic of any kind | no lifecycle code in `crates/buzz-media/src/storage.rs:34-265` |
| No durable storage quota per pubkey or community; only in-flight admission limits exist, and they live in the relay | `crates/buzz-relay/src/api/media.rs:302-304` (TODO) |
| No Blossom `/list`, `DELETE /{sha256}`, or mirror support | `crates/buzz-media/src/lib.rs:17-28` (no such exports) |
| Audio uploads unsupported pending a sanitizer | `crates/buzz-media/src/validation.rs:186-196` |
| Inline PDF preview deferred | `crates/buzz-media/src/validation.rs:211-215` |
| No S3 retry or timeout policy | `crates/buzz-media/src/storage.rs:34-265` |
| No integrity re-verification on read | `crates/buzz-media/src/storage.rs:105-146` |
| Video thumbnails delegated to the desktop client rather than generated server-side | `crates/buzz-media/src/upload.rs:437-448` |

---

### 6. Deprecated API usage

None found. `rust-s3` 0.37's manual `list_page` is used deliberately over the auto-paginating `list` ("NOT the auto-paginating `list`, which has no cap" — `crates/buzz-media/src/storage.rs:235-241`). No `#[deprecated]` items, no `#[allow(deprecated)]`, no compiler-deprecation suppressions appear in the crate. The one non-standard dependency situation is the temporary workspace `[patch.crates-io]` pin of `aws-creds` to a fork, explicitly marked "Revert to crates.io once #449 lands upstream" (`Cargo.toml:163-171`).

---

### 7. Inconsistencies worth flagging

| # | Inconsistency |
|---|---|
| I-1 | Three different 404 detection mechanisms: `HttpFailWithBody(404, _)` matching (`crates/buzz-media/src/storage.rs:107`), `response.status_code == 404` (`crates/buzz-media/src/storage.rs:136-139`), and no check at all in `get_sidecar` (`crates/buzz-media/src/storage.rs:197-199`) |
| I-2 | Byte caps are configurable; pixel, duration, resolution, atom, box, and depth caps are hardcoded |
| I-3 | Overflow discipline differs between the two MP4 walkers (D-4) |
| I-4 | The buffered image and file paths were unified behind `process_buffered_upload`, but the video path duplicates all shared steps inline (`crates/buzz-media/src/upload.rs:54-192` vs `:292-511`) |
| I-5 | `ext` derivation has three separate implementations: `mime_to_ext` (`crates/buzz-media/src/validation.rs:930-939`), `file_mime_to_ext` + `infer` fallback (`crates/buzz-media/src/validation.rs:99-156`, `:203-206`), and a hardcoded `"mp4"` (`crates/buzz-media/src/upload.rs:420`) — and a fourth, independent notion of a valid ext in the sweep classifier (`crates/buzz-media/src/bucket_index.rs:84-87`) |
| I-6 | `_meta/` and `_uploads/` key builders live in different modules (`storage.rs` vs `upload_record.rs`) while their parsers both live in `bucket_index.rs` — the writer/reader pair for the same layout is split across three files, so a layout change must be made in three places (the sweep tests are the only cross-check) |
