## Module: buzz-media (`crates/buzz-media`)

### Aspect: Security

### 0. Summary table

| # | Concern | Status in this crate | Key file:line |
|---|---|---|---|
| S-1 | Content-type spoofing | Mitigated — MIME always sniffed, header never read | `crates/buzz-media/src/validation.rs:239-247` |
| S-2 | Pixel bomb / decompression bomb | Mitigated — 25 MP pre-decode cap, fail-closed | `crates/buzz-media/src/validation.rs:264-273` |
| S-3 | Memory exhaustion (images/files) | Partially mitigated — caps bound RAM but the whole body is buffered | `crates/buzz-media/src/validation.rs:167-172`, `crates/buzz-relay/src/api/media.rs:359-372` |
| S-4 | Memory exhaustion (video) | Mitigated — streamed to disk, 64 KiB chunks | `crates/buzz-media/src/upload.rs:325-395` |
| S-5 | Disk exhaustion (video temp files) | Partially mitigated — per-request bound only | `crates/buzz-media/src/upload.rs:307-320` |
| S-6 | Path traversal in object keys | Mitigated in this crate by construction | `crates/buzz-media/src/upload.rs:94`, `crates/buzz-media/src/storage.rs:183-185` |
| S-7 | SSRF | Not applicable — no remote fetch exists | whole crate |
| S-8 | Hash verification of uploaded bytes | Mitigated — server computes the hash, client hash must match | `crates/buzz-media/src/upload.rs:84-86`, `crates/buzz-media/src/auth.rs:190-196` |
| S-9 | Integrity on read | **Gap** — bytes are never re-verified on read | `crates/buzz-media/src/storage.rs:105-146` |
| S-10 | EXIF/metadata privacy | Mitigated by rejection, not stripping | `crates/buzz-media/src/validation.rs:487-928` |
| S-11 | Read authentication | **Not in this crate** — verifier provided, gate lives in relay | `crates/buzz-media/src/auth.rs:207-236`, `crates/buzz-relay/src/api/media.rs:489-515` |
| S-12 | Signed URLs | Not implemented anywhere in the crate | `crates/buzz-media/src/storage.rs:73-265` |
| S-13 | Tenant isolation of blobs | Sidecar-scoped; raw CAS bytes are shared | `crates/buzz-media/src/storage.rs:177-185` |
| S-14 | `unsafe` | **None** — zero occurrences | all 12 files |
| S-15 | Panic surface in parsers | Bounded; 9 infallible `try_into().unwrap()` + unchecked `offset += atom_size` | `crates/buzz-media/src/validation.rs:481`, `:604-886` |
| S-16 | Durable-bytes quota | **Gap** — no per-pubkey/per-community storage quota | `crates/buzz-relay/src/api/media.rs:302-304` (TODO) |

---

### 1. Content-type spoofing

The declared HTTP `Content-Type` is never consulted in this crate. Both validators start from `infer::get` on the actual bytes, with the comment "Magic bytes — never trust Content-Type header" (`crates/buzz-media/src/validation.rs:239-242`).

Cross-path spoofing is closed in both directions:
- MP4 submitted to the image path → `infer` reports `video/mp4`, which is absent from `ALLOWED_MIME_TYPES` → `DisallowedContentType` (`crates/buzz-media/src/validation.rs:9-13`, `:245-247`; test `crates/buzz-media/src/validation.rs:1626-1648`).
- Media submitted to the generic-file path → sniffed `image/*`, `video/*`, `audio/*` all rejected (`crates/buzz-media/src/validation.rs:186-196`; tests `crates/buzz-media/src/validation.rs:1507-1555`).
- MP4 with a proprietary major brand that `infer` cannot classify → still caught by the structural `ftyp` check, so it cannot fall through the generic path as an opaque attachment (`crates/buzz-media/src/validation.rs:174-181`; test asserts `infer::get(...)` is `None` yet the file is rejected, `crates/buzz-media/src/validation.rs:1518-1525`).

Stored-XSS defence for generic files is layered: a deny-list here (`text/html`, `application/xhtml+xml`, `image/svg+xml`, `application/javascript`, `text/javascript`, plus 9 executable/installer types — `crates/buzz-media/src/validation.rs:75-95`) and `serve_inline` returning false for everything except `image/*`/`video/*` (`crates/buzz-media/src/validation.rs:216-218`). The comment records that the response headers (`Content-Disposition: attachment`, `nosniff`, `CSP: default-src 'none'`) already neutralise these and the list is defence in depth against a future header regression (`crates/buzz-media/src/validation.rs:66-74`). Those headers are set in the relay (`crates/buzz-relay/src/api/media.rs:660-700`), not here.

Residual: unsniffable bytes on the generic path are accepted as `application/octet-stream` (`crates/buzz-media/src/validation.rs:207`). An HTML/JS payload with no magic signature is therefore stored — safe only because of the attachment/nosniff headers plus the `serve_inline` policy, i.e. this is a defence-in-depth dependency on the relay's header behaviour.

---

### 2. Decompression / pixel bombs and malformed media

| Guard | Value | file:line |
|---|---|---|
| Byte cap before any parse (image) | `max_image_bytes` / `max_gif_bytes` | `crates/buzz-media/src/validation.rs:259-268` |
| Pixel cap before full decode | `25_000_000` px; comment "25 megapixels — 100MB max RGBA decode" | `crates/buzz-media/src/validation.rs:269-273` |
| Fail-closed on unparseable geometry | `imagesize::blob_size(...).map_err(\|_\| InvalidImage)?` — unknown-geometry images never reach the decoder | `crates/buzz-media/src/validation.rs:264-270` |
| Structural pre-checks before decode | full container walk (`validate_image_metadata_free`) runs before the pixel check and before decode | `crates/buzz-media/src/validation.rs:263`, `:492-500` |
| MP4 top-level atom scan cap | `MAX_ATOMS = 1024`, header-only reads, fail-closed on exceeding | `crates/buzz-media/src/validation.rs:411-433` |
| MP4 box-walk caps | `MAX_BOXES = 100_000`, `MAX_BOX_DEPTH = 32` | `crates/buzz-media/src/validation.rs:832-833`, `:871-878` |
| MP4 division-by-zero guard | `timescale() == 0` rejected before `duration()` ("Fail fast before it panics") | `crates/buzz-media/src/validation.rs:346-351` |
| GIF/PNG/WebP/JPEG walks | all use `checked_add` + explicit `<= len` filters, no unchecked indexing into attacker-controlled offsets | `crates/buzz-media/src/validation.rs:533-538`, `:596-600`, `:672-682`, `:739-748` |

Order matters and is correct: the pixel guard runs **before** `image::load_from_memory`, and the decode happens in a separate function called from `spawn_blocking` (`crates/buzz-media/src/validation.rs:264-273` → `crates/buzz-media/src/thumbnail.rs:26` via `crates/buzz-media/src/upload.rs:518-524`).

Residuals:
1. The 25 MP cap bounds a single RGBA decode to ~100 MB, but the *concurrent* count is not bounded here — concurrency limits live in the relay (`crates/buzz-relay/src/config.rs:659-676`).
2. Animated GIF/WebP are decoded via `image::load_from_memory`, which yields a single frame; the only bound on animation complexity is `max_gif_bytes` (`crates/buzz-media/src/validation.rs:259-262`).
3. `check_moov_before_mdat` advances with `offset += atom_size` where `atom_size` can be an attacker-supplied 64-bit value (`crates/buzz-media/src/validation.rs:463-481`). No `checked_add` — this is an overflow-wrap in release builds (workspace release profile has `panic = "abort"`, `Cargo.toml:159-162`) and a debug-build panic. Impact is bounded by `MAX_ATOMS = 1024` iterations and by the fact that a wrapped offset yields a failed `read_exact` or a `MoovNotAtFront`; the sibling walker in `validate_mp4_metadata_free` does use `checked_add` (`crates/buzz-media/src/validation.rs:891-893`), so this is an internal inconsistency worth closing.

---

### 3. Memory and disk exhaustion

| Path | Peak RAM | file:line |
|---|---|---|
| Image | full body (≤ `max_image_bytes`) + decoded RGBA (≤ ~100 MB) + thumbnail | `crates/buzz-media/src/upload.rs:207-236`, `crates/buzz-media/src/thumbnail.rs:26-32` |
| Generic file | full body (≤ `max_file_bytes`); the body is cloned into the blocking closure but `Bytes` clones are refcount bumps, documented | `crates/buzz-media/src/upload.rs:76-78`, `:193-201` |
| Video | 64 KiB read buffer + 4 KiB sniff buffer; body never fully in RAM | `crates/buzz-media/src/upload.rs:349-353` |
| Video S3 upload | 8 MiB `BufReader`, streamed from disk | `crates/buzz-media/src/storage.rs:91-101` |
| Blob read | `get`/`get_range` materialise into `Vec<u8>`; `get_stream` does not | `crates/buzz-media/src/storage.rs:105-146` |
| Sweep | only per-sha/per-binding aggregates, never the full listing (documented) | `crates/buzz-media/src/bucket_index.rs:230-246` |

Video disk usage: a `NamedTempFile` is created per request and dropped immediately after the S3 upload (`crates/buzz-media/src/upload.rs:307-309`, `:435`). Per-request size is capped three times (Content-Length pre-check, running total, on-disk size — `crates/buzz-media/src/upload.rs:311-320`, `:372-378`, `crates/buzz-media/src/validation.rs:298-304`), but **total concurrent temp-file bytes are not bounded in this crate**; that depends on the relay's upload concurrency semaphore (`crates/buzz-relay/src/config.rs:659-668`). With the defaults (8 concurrent × 500 MB) the worst case is ~4 GB of temp space.

A body-limit error raised by axum's layer mid-stream is translated to `FileTooLarge`/413 rather than a 500, via substring matching on three known Display forms with a test that guards against all three breaking at once (`crates/buzz-media/src/upload.rs:328-341`, `:358-366`, test `crates/buzz-media/src/upload.rs:685-712`). This is fragile by construction (acknowledged in the comment) but fails toward a correct-status rejection, not toward acceptance.

---

### 4. Path traversal in object keys

Every key written by this crate is built from server-controlled components:

| Key | Components | file:line |
|---|---|---|
| blob | server-computed sha256 hex + ext from a fixed MIME→ext map | `crates/buzz-media/src/upload.rs:94`, `crates/buzz-media/src/validation.rs:930-939` |
| thumb | server sha256 + literal `.thumb.jpg` | `crates/buzz-media/src/upload.rs:531` |
| sidecar | `_meta/` + `TenantContext::community()` UUID + server sha256 | `crates/buzz-media/src/storage.rs:183-185` |
| record | `_uploads/` + community UUID + server sha256 + generated ULID | `crates/buzz-media/src/upload_record.rs:181-183` |

No client string ever reaches a key. The one path with a partly-external component is the generic-file extension: when `file_mime_to_ext` has no mapping it falls back to `infer`'s `kind.extension()` (`crates/buzz-media/src/validation.rs:203-206`) — a static string from the `infer` crate's own table, not user input, though it is not re-validated against a charset here. The sweep's classifier independently enforces a 1–8 alphanumeric ext when reading keys back (`crates/buzz-media/src/bucket_index.rs:84-87`), and the relay's `validate_media_path` constrains the read path (`crates/buzz-relay/src/api/media.rs:547`, tests `crates/buzz-relay/src/api/media.rs:1206-1215`).

`MediaStorage::get`/`get_stream`/`get_range`/`delete` take an arbitrary `&str` key with **no validation inside this crate** (`crates/buzz-media/src/storage.rs:105-164`) — key sanitation for read paths is entirely the caller's responsibility.

---

### 5. SSRF

No outbound fetch of user-supplied URLs exists anywhere in the crate. The only network egress is to the configured S3 endpoint via `rust-s3` (`crates/buzz-media/src/storage.rs:34-70`); there is no mirror/BUD-04 implementation, no URL field consumed as an input, and no HTTP client dependency. `public_base_url` is used only for string formatting of response URLs (`crates/buzz-media/src/upload.rs:548`, `crates/buzz-media/src/thumbnail.rs:44`).

Adjacent note: the `aws-creds` fork the workspace pins reads container-credential URIs from the environment and, per the patch rationale, includes "a loopback allowlist for the auth token" (`Cargo.toml:167-169`) — the relevant SSRF-adjacent surface is in that dependency, not in this crate's code.

---

### 6. Hash verification of uploaded bytes — verified

The hash in the key/URL is always the server's own digest of the bytes actually received:

1. Buffered: `hex::encode(Sha256::digest(&bytes))` over the exact `Bytes` that are later PUT (`crates/buzz-media/src/upload.rs:84`, PUT at `:129`).
2. Streamed: `hasher.update(&buf[..n])` on each chunk immediately before `file.write_all(&buf[..n])`, so hash and file content cannot diverge (`crates/buzz-media/src/upload.rs:379-382`); the same temp file is then uploaded (`crates/buzz-media/src/upload.rs:434`).
3. The client's declared hash is checked *against* the computed one — `verify_blossom_upload_auth` requires at least one `x` tag equal to `sha256` else `HashMismatch` (`crates/buzz-media/src/auth.rs:189-196`).
4. The key uses the computed hash (`crates/buzz-media/src/upload.rs:94`, `:421`), so even a matching-but-irrelevant `x` tag cannot rename a blob.

Gaps: no post-PUT read-back verification, and no verification on the read path (`crates/buzz-media/src/storage.rs:105-146`) — a bucket-level tamper (direct S3 write with the relay's credentials, or a compromised store) would be served without detection. The relay does not re-hash either (`crates/buzz-relay/src/api/media.rs:619-760`).

---

### 7. EXIF / metadata privacy

The crate's stance is **reject, not strip** (`crates/buzz-media/src/validation.rs:487-491`): "This is deliberately a structural allowlist rather than an EXIF-tag denylist: location can also live in XMP, comments, PNG text, ICC descriptions, or private chunks. Client encoders remove these before upload."

Coverage: JPEG `APP1..APP13/APP15/COM` and non-canonical `APP0`/`APP14` (`crates/buzz-media/src/validation.rs:539-563`); PNG `eXIf`/`zTXt`/`iTXt`/`iCCP` + all unknown ancillary chunks + `pHYs` (excluded as "an identity channel") (`crates/buzz-media/src/validation.rs:614-651`); WebP `EXIF`/`XMP `/`ICCP`/unknown chunks + `VP8X` metadata presence flags even when the chunk is absent (`crates/buzz-media/src/validation.rs:690-732`); GIF comment/plain-text/non-loop application extensions (`crates/buzz-media/src/validation.rs:783-826`); MP4 `meta ilst keys data uuid xml  bxml loci ©xyz name chap`, non-allowlisted boxes, alternate tracks, timed-metadata tracks (`crates/buzz-media/src/validation.rs:834-846`, `:895-908`, `:325-386`). Trailing-bytes-after-terminator is forbidden in all four image containers, closing the "motion photo"/appended-payload channel (`crates/buzz-media/src/validation.rs:528-532`, `:653-656`, `:698-701`, `:819-823`).

Tests prove the policy against real metadata rather than synthetic markers: a hand-built EXIF GPS IFD verified by an independent parser (`crates/buzz-media/src/validation.rs:1011-1075`), real XMP GPS strings (`crates/buzz-media/src/validation.rs:1112-1162`), QuickTime ISO-6709 `©xyz` coordinates (`crates/buzz-media/src/validation.rs:2411-2432`), and unsanitized iOS/Android encoder output (`crates/buzz-media/src/validation.rs:1203-1280`).

Two deliberate exceptions to note in a privacy review:
- Exactly one PNG `tEXt` chunk keyed `buzz_agent_snapshot` or `buzz_team_snapshot` is permitted, i.e. a sanctioned in-image data channel for agent/team sharing (`crates/buzz-media/src/validation.rs:575-590`, `:601-611`). Duplicate or near-miss keywords are rejected (`crates/buzz-media/src/validation.rs:1328-1366`).
- One exact 53-byte ffmpeg empty-`udta` byte sequence is permitted verbatim in MP4 (`crates/buzz-media/src/validation.rs:834-840`, `:895-903`).

Consequence: privacy depends on **clients** sanitizing. A user uploading an untouched camera JPEG gets a 422, not a stripped image — a usability/robustness tradeoff, not a leak.

---

### 8. Read access, authentication, and signed URLs

- This crate provides the *verifier* `verify_blossom_get_auth` but never calls it (`crates/buzz-media/src/auth.rs:207-236`). Nothing in `MediaStorage`, `upload.rs`, or `validation.rs` gates reads.
- The relay owns the gate: `authenticate_media_read` returns early when `require_media_get_auth` is false, otherwise calls `buzz_media::auth::verify_blossom_get_auth(..., 3600)` (`crates/buzz-relay/src/api/media.rs:489-515`), fed by `BUZZ_REQUIRE_MEDIA_GET_AUTH`, default **false** (`crates/buzz-relay/src/config.rs:677-685`).
- Documented consequence of BUD-01 server-scoped get tokens: they "intentionally grant reads for all blobs on the host until expiration; callers must still apply relay membership after this verifier returns" (`crates/buzz-media/src/auth.rs:201-205`). The `RelayMembershipRequired` variant exists (`crates/buzz-media/src/error.rs:69-70`) but is never constructed here — that check must exist in the relay.
- **No presigned URLs** anywhere: no `presign_get`/`presign_put` call in `crates/buzz-media/src/storage.rs:73-265`. Blobs are served by the relay proxying S3 bytes, so bucket credentials never reach clients.
- Auth replay window: `created_at` freshness is bounded (600 s buffered, 3600 s video — `crates/buzz-media/src/upload.rs:85`, `:412`) *in addition to* the `expiration` tag, explicitly to bound replay (`crates/buzz-media/src/auth.rs:112-122`). Within that window a captured token is replayable; no nonce/jti store exists (the `TokenRevoked` variant is declared but unused here).
- Auth error responses are uniform 401 `"authentication failed"` to avoid an enumeration oracle (`crates/buzz-media/src/error.rs:120-146`).

---

### 9. Tenant isolation of blobs

| Property | Finding |
|---|---|
| Raw blob bytes | **Not** community-scoped — key is `{sha}.{ext}` globally (`crates/buzz-media/src/upload.rs:94`) |
| Metadata sidecar | Community-scoped and described as "the tenant read gate": "A blob in another community must never be observable through a global `_meta/{sha}.json` lookup" (`crates/buzz-media/src/storage.rs:177-182`) |
| Community source | Always `TenantContext` (server-resolved); comment: "Callers must never derive the community from client-supplied blob metadata, URLs, or event tags" (`crates/buzz-media/src/storage.rs:204-209`) |
| Existence-oracle handling | `read_sidecar_mime` collapses read failure and absence to `None` so an A-bound request cannot distinguish a B-only blob from a missing blob (`crates/buzz-media/src/storage.rs:222-233`) |
| Blossom `server` tag | Validated against the *per-request* bound tenant host, not a process-global domain, with `normalize_host` equivalence; unknown host → reject (`crates/buzz-media/src/auth.rs:124-143`) |
| Regression tests | `sidecar_keys_are_community_scoped`, `same_sha_sidecars_do_not_bleed_between_communities` (`crates/buzz-media/src/storage.rs:333-377`), `test_server_tag_normalized_against_bound_host` (`crates/buzz-media/src/auth.rs:470-530`) |

Structural consequence: isolation is *metadata-level*, not *byte-level*. Anyone who can guess or learn a sha256 and reach a relay path that bypasses the sidecar lookup would hit shared bytes. The read path in the relay does perform the sidecar lookup before serving (`crates/buzz-relay/src/api/media.rs:625-660`), so the gate holds for the shipped handler — but the property depends on every future reader honouring it. A second consequence: cross-community dedup is observable as a timing/behaviour difference on upload (the idempotent short-circuit needs the *community's* sidecar, so it does not leak across tenants — `crates/buzz-media/src/upload.rs:97-103`).

---

### 10. `unsafe`

**Verified none.** Zero occurrences of the `unsafe` keyword across `crates/buzz-media/src/{lib,types,thumbnail,config,error,storage,upload_record,auth,upload,bucket_index,validation}.rs` and `crates/buzz-media/tests/static_creds_minio.rs`. All binary parsing uses safe slicing with explicit bounds checks and `try_into()` on fixed-size ranges.

---

### 11. Input-validation gaps and other residual findings

| # | Finding | file:line |
|---|---|---|
| G-1 | No integrity check on read — stored bytes are never re-hashed against their key | `crates/buzz-media/src/storage.rs:105-146` |
| G-2 | `MediaStorage` key-taking methods accept arbitrary keys without validation; callers must sanitize | `crates/buzz-media/src/storage.rs:105-164` |
| G-3 | Unchecked `offset += atom_size` in the moov scanner (attacker-controlled 64-bit size), inconsistent with the `checked_add` used in the sibling box walker | `crates/buzz-media/src/validation.rs:481` vs `:891-893` |
| G-4 | Unsniffable generic uploads accepted as `application/octet-stream`; safety rests on relay response headers | `crates/buzz-media/src/validation.rs:207`, `crates/buzz-media/src/validation.rs:66-74` |
| G-5 | No durable storage quota per pubkey/community; admission limits bound only in-flight work (relay TODO acknowledges it) | `crates/buzz-relay/src/api/media.rs:302-304` |
| G-6 | Orphan blobs are deliberately never cleaned up and no GC exists, so failed/partial uploads accumulate bytes an attacker can grow (bounded per-request, unbounded in aggregate) | `crates/buzz-media/src/upload.rs:122-127` |
| G-7 | `max_image_bytes`/`max_gif_bytes` have no serde default; a config source that omits them fails to deserialize rather than defaulting safely (fail-closed, but easy to misread as "50 MB is the crate default") | `crates/buzz-media/src/config.rs:38-44` |
| G-8 | Blurhash failure is silently swallowed (`unwrap_or_default()`), producing an empty blurhash rather than an error | `crates/buzz-media/src/thumbnail.rs:36-37` |
| G-9 | `head_with_metadata` coerces a missing `content_length` to `0`, which a caller could misread as a zero-byte object | `crates/buzz-media/src/storage.rs:170-172` |
| G-10 | Aggregation counters use unchecked `+=` on `u64` (`physical_bytes += size`); overflow-wrap only, no correctness guard | `crates/buzz-media/src/bucket_index.rs:251-252` |
| G-11 | IP collection is opt-in, fail-empty, and scoped to the `_uploads/` record only — never blob metadata, response, or audit log (privacy-positive, listed for completeness) | `crates/buzz-media/src/upload_record.rs:30-40`, `:191-256` |
| G-12 | `_uploads/` records are stated to be unreachable via the media serve path "by construction" because `validate_media_path` requires a bare 64-hex first segment — a cross-crate invariant, enforced in the relay, not here | `crates/buzz-media/src/upload_record.rs:24-27`, `crates/buzz-relay/src/api/media.rs:547` |
