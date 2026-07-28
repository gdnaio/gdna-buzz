## Module: buzz-media (`crates/buzz-media`)

### Aspect: Business Rules

Every rule below was read in the source; the "Trigger" column names the entry point that enforces it.

---

### 1. Content addressing and integrity

| # | Rule | Enforcement | Trigger |
|---|---|---|---|
| BR-1 | Blob key is `{sha256}.{ext}` where sha256 is computed server-side over the received bytes | `crates/buzz-media/src/upload.rs:84`, `crates/buzz-media/src/upload.rs:94` | every buffered upload |
| BR-2 | Streaming (video) hash is computed incrementally over the same bytes written to the temp file | `crates/buzz-media/src/upload.rs:365-381`, `crates/buzz-media/src/upload.rs:397` | `process_video_upload` |
| BR-3 | The client-declared hash (Blossom `x` tag) must equal the server-computed hash, else `HashMismatch` | `crates/buzz-media/src/auth.rs:190-196` | `verify_blossom_upload_auth`, called at `crates/buzz-media/src/upload.rs:85` and `crates/buzz-media/src/upload.rs:412` |
| BR-4 | The stored key is derived from the *server-computed* hash, not the `x` tag — a spoofed `x` cannot mis-key a blob | `crates/buzz-media/src/upload.rs:94`, `crates/buzz-media/src/upload.rs:421` | all upload paths |
| BR-5 | Extension is derived from sniffed MIME only; no client filename is consulted anywhere in the crate | `crates/buzz-media/src/validation.rs:930-939`, `crates/buzz-media/src/validation.rs:203-206`, `crates/buzz-media/src/upload.rs:420` | all upload paths |

There is **no re-read-and-verify** after the S3 PUT, and no verification on read: `MediaStorage::get`/`get_stream`/`get_range` return bytes without re-hashing (`crates/buzz-media/src/storage.rs:105-146`).

---

### 2. Size limits

| # | Rule | Value | file:line |
|---|---|---|---|
| BR-6 | Image upload cap (non-GIF) | `config.max_image_bytes` (no in-crate default; required serde field) | `crates/buzz-media/src/validation.rs:259-268`, `crates/buzz-media/src/config.rs:41` |
| BR-7 | Animated-GIF cap, applied when sniffed MIME == `image/gif` | `config.max_gif_bytes` | `crates/buzz-media/src/validation.rs:259-262` |
| BR-8 | Generic-file cap | `config.max_file_bytes`, default **104_857_600 (100 MB)** | `crates/buzz-media/src/validation.rs:167-172`, `crates/buzz-media/src/config.rs:7-9` |
| BR-9 | Video cap, pre-checked against `Content-Length` before streaming | `config.max_video_bytes`, default **524_288_000 (500 MB)** | `crates/buzz-media/src/upload.rs:311-320`, `crates/buzz-media/src/config.rs:3-5` |
| BR-10 | Video cap re-checked per read chunk during streaming (running total) | same | `crates/buzz-media/src/upload.rs:372-378` |
| BR-11 | Video cap re-checked a third time against the temp file's on-disk size | same | `crates/buzz-media/src/validation.rs:298-304` |
| BR-12 | Config coherence: all four caps must be > 0 and `max_gif_bytes <= max_image_bytes` | startup failure | `crates/buzz-media/src/config.rs:76-90` |
| BR-13 | Pixel-count cap (image bomb): `width * height <= 25_000_000` | 25 MP | `crates/buzz-media/src/validation.rs:269-273` |
| BR-14 | Video duration `> 600.0 s` → `DurationTooLong`; `<= 0.0` → `InvalidVideo` | 600 s | `crates/buzz-media/src/validation.rs:354-359` |
| BR-15 | Video resolution `> 3840 × 2160` → `ResolutionTooHigh` | 4K | `crates/buzz-media/src/validation.rs:364-366` |

`max_image_bytes` and `max_gif_bytes` have **no `#[serde(default)]`** — the caller must supply them (`crates/buzz-media/src/config.rs:38-44`); the relay defaults them to 50 MB / 10 MB (`crates/buzz-relay/src/config.rs:626-632`).

---

### 3. MIME sniffing vs declared content type

| # | Rule | file:line |
|---|---|---|
| BR-16 | MIME is always sniffed from magic bytes via `infer`; the HTTP `Content-Type` header is never read in this crate | `crates/buzz-media/src/validation.rs:239-242` ("never trust Content-Type header"), `crates/buzz-media/src/validation.rs:176-201` |
| BR-17 | Image path allowlist: exactly `image/jpeg`, `image/png`, `image/gif`, `image/webp` | `crates/buzz-media/src/validation.rs:15`, enforced `crates/buzz-media/src/validation.rs:245-247` |
| BR-18 | Unsniffable bytes on the image path → `UnknownContentType` (fail closed) | `crates/buzz-media/src/validation.rs:239-242` |
| BR-19 | `video/mp4` uploaded through the image path is rejected (`DisallowedContentType`) — spoofing an MP4 as an image cannot skip video validation | `crates/buzz-media/src/validation.rs:9-13`, `crates/buzz-media/src/validation.rs:245-247` |
| BR-20 | Generic path: any sniffed `image/*`, `video/*`, or `audio/*` is rejected so recognized media cannot bypass its format-specific validator; audio is rejected outright pending a sanitizer | `crates/buzz-media/src/validation.rs:186-196` |
| BR-21 | Generic path: any structurally valid ISO-BMFF `ftyp` container is rejected even when `infer` doesn't recognize the brand | `crates/buzz-media/src/validation.rs:174-181` |
| BR-22 | Generic path deny-list (defence in depth against stored XSS / malware): `text/html`, `application/xhtml+xml`, `image/svg+xml`, `application/javascript`, `text/javascript`, `application/x-msdownload`, `application/x-executable`, `application/vnd.microsoft.portable-executable`, `application/x-mach-binary`, `application/x-sharedlib`, `application/x-elf`, `application/x-msi`, `application/vnd.android.package-archive`, `application/x-apple-diskimage` | `crates/buzz-media/src/validation.rs:75-95`, enforced `crates/buzz-media/src/validation.rs:198-200` |
| BR-23 | Unsniffable bytes on the generic path are accepted as `application/octet-stream` / ext `bin` (text, CSV, JSON, source code have no magic bytes) | `crates/buzz-media/src/validation.rs:207` |
| BR-24 | Serve policy: only `image/*` and `video/*` are inline; everything else (including PDF) is an attachment | `crates/buzz-media/src/validation.rs:216-218` |

---

### 4. Metadata-free (privacy) enforcement — no transformation, reject instead

The crate does **not strip** EXIF/XMP/ICC; it *rejects* any file carrying a metadata channel, on the stated premise that client encoders sanitize before upload (`crates/buzz-media/src/validation.rs:487-491`).

| # | Container | Rule | file:line |
|---|---|---|---|
| BR-25 | JPEG | Only canonical JFIF `APP0` (fixed length formula) and 12-byte `Adobe` `APP14` allowed; `APP1`–`APP13`, `APP15`, and `COM` → `MetadataForbidden`; trailing bytes after `EOI` → `MetadataForbidden` | `crates/buzz-media/src/validation.rs:539-563`, `crates/buzz-media/src/validation.rs:528-532` |
| BR-26 | PNG | `eXIf`/`zTXt`/`iTXt`/`iCCP` forbidden; unknown ancillary chunks forbidden; only `cHRM gAMA sBIT sRGB bKGD hIST tRNS sPLT acTL fcTL fdAT` allowed; `pHYs` deliberately excluded; bytes after `IEND` forbidden | `crates/buzz-media/src/validation.rs:614-651` |
| BR-27 | PNG (product exception) | Exactly one `tEXt` chunk whose keyword is `buzz_agent_snapshot` or `buzz_team_snapshot` followed by NUL is permitted (agent/team snapshot manifests); a second such chunk or any other keyword is forbidden | `crates/buzz-media/src/validation.rs:579-590`, `crates/buzz-media/src/validation.rs:601-611` |
| BR-28 | WebP | RIFF declared size must equal `len-8`; only `VP8 `/`VP8L`/`VP8X`/`ALPH`/`ANIM`/`ANMF` chunks; `VP8X` ICC/EXIF/XMP flags (`0x20/0x08/0x04`) forbidden; `ANMF` sub-chunks recursively restricted to `ALPH`/`VP8 `/`VP8L` | `crates/buzz-media/src/validation.rs:690-732`, `crates/buzz-media/src/validation.rs:660-687` |
| BR-29 | GIF | Only Graphic Control Extensions with exact spec shape and `NETSCAPE2.0`/`ANIMEXTS1.0` loop application extensions; comment/plain-text/other app extensions forbidden; trailing bytes after `0x3B` forbidden | `crates/buzz-media/src/validation.rs:734-829` |
| BR-30 | MP4 | Box allowlist (35 types); explicit forbidden list `meta ilst keys data uuid xml  bxml loci ©xyz name chap`; any box not in the allowlist → `MetadataForbidden`; `udta` permitted **only** as the exact 53-byte empty-ffmpeg `udta` payload | `crates/buzz-media/src/validation.rs:831-861`, `crates/buzz-media/src/validation.rs:895-908` |
| BR-31 | MP4 tracks | Exactly one video track; ≥2 video or ≥2 audio tracks → `MetadataForbidden`; any non-video/non-audio track type (e.g. timed metadata) → `MetadataForbidden` | `crates/buzz-media/src/validation.rs:325-331`, `crates/buzz-media/src/validation.rs:369-386` |

Only transformation performed anywhere in the crate: JPEG **thumbnail** generation (≤320×320, aspect preserved) plus a 4×3 blurhash, both derived artifacts written to separate keys; the original bytes are stored unmodified (`crates/buzz-media/src/thumbnail.rs:30-38`, `crates/buzz-media/src/upload.rs:129`). No transcoding, no re-encoding of the stored blob, no EXIF stripping.

---

### 5. Codec / container policy (video)

| # | Rule | file:line |
|---|---|---|
| BR-32 | Container must be structurally ISO-BMFF `ftyp` with an MP4 brand (major **or** any compatible brand in `isom iso2..iso9 mp41 mp42 avc1 dash M4V `) — proprietary major brands accepted when a compatible brand matches | `crates/buzz-media/src/validation.rs:17-20`, `crates/buzz-media/src/validation.rs:52-62`, called `crates/buzz-media/src/upload.rs:403-407` |
| BR-33 | QuickTime major brand `qt  ` → `UnsupportedContainer` | `crates/buzz-media/src/validation.rs:316-322` |
| BR-34 | Video codec must be H.264 (`mp4::MediaType::H264`), else `WrongCodec` (rejects HEVC/VP9/AV1) | `crates/buzz-media/src/validation.rs:335-341` |
| BR-35 | Audio codec must be AAC (`mp4::MediaType::AAC`), else `WrongCodec` | `crates/buzz-media/src/validation.rs:378-382` |
| BR-36 | `moov` must precede `mdat` (fast-start), verified by a header-only top-level atom scan before the MP4 parser runs | `crates/buzz-media/src/validation.rs:291-293`, `crates/buzz-media/src/validation.rs:408-484` |
| BR-37 | Audio-only MP4 (no video track, has audio) → `DisallowedContentType("audio/mp4")`; no tracks at all → `InvalidVideo` | `crates/buzz-media/src/validation.rs:388-395` |
| BR-38 | `track.timescale() == 0` rejected before calling `duration()` to avoid a division-by-zero panic in the `mp4` crate | `crates/buzz-media/src/validation.rs:346-351` |

---

### 6. Blossom auth rules (kind 24242)

| # | Rule | file:line |
|---|---|---|
| BR-39 | Schnorr signature must verify | `crates/buzz-media/src/auth.rs:37-39` |
| BR-40 | Kind must be exactly 24242 | `crates/buzz-media/src/auth.rs:42-44` |
| BR-41 | `content` must be non-empty after trim (BUD-11 human-readable string) | `crates/buzz-media/src/auth.rs:47-49` |
| BR-42 | `t` tag required and must equal the expected verb (`upload`/`get`) | `crates/buzz-media/src/auth.rs:59-66`, `crates/buzz-media/src/auth.rs:97-99` |
| BR-43 | `expiration` tag required and must be strictly in the future | `crates/buzz-media/src/auth.rs:101-110` |
| BR-44 | `created_at` must not be > 5 s in the future | `crates/buzz-media/src/auth.rs:116-119` |
| BR-45 | `created_at` must be within `max_age_secs`: **600 s** for buffered image/file uploads, **3600 s** for video uploads | `crates/buzz-media/src/auth.rs:120-122`; windows at `crates/buzz-media/src/upload.rs:85` and `crates/buzz-media/src/upload.rs:412` |
| BR-46 | If any `server` tag is present, the bound tenant host must match one of them after `normalize_host` (case, trailing dot, default port, scheme/path tolerant); if the bound host is unknown (`None`) → reject (fail closed) | `crates/buzz-media/src/auth.rs:135-143`, `crates/buzz-media/src/auth.rs:163-170` |
| BR-47 | Upload: at least one `x` tag must equal the computed sha256 | `crates/buzz-media/src/auth.rs:190-196` |
| BR-48 | Get: `x` match **or** `server` match suffices; neither → `InsufficientScope`. A `server`-scoped get token grants reads of all blobs on that host until expiry (documented, deliberate) | `crates/buzz-media/src/auth.rs:201-236` |

---

### 7. Deduplication / idempotency

| # | Rule | file:line |
|---|---|---|
| BR-49 | Upload short-circuits only when **both** sidecar and blob exist; a sidecar without its blob falls through to re-upload | `crates/buzz-media/src/upload.rs:97-103`, `crates/buzz-media/src/upload.rs:424-427` |
| BR-50 | Short-circuit returns the descriptor built from the existing sidecar with the **original** `uploaded_at`, not now | `crates/buzz-media/src/upload.rs:120-128` |
| BR-51 | Identical bytes across communities share one blob object but get separate sidecars (dedup is global, visibility is per-community) | `crates/buzz-media/src/storage.rs:177-185`, test `crates/buzz-media/src/storage.rs:352-377` |
| BR-52 | A re-upload of known bytes still writes an upload record when records are enabled, because no blob PUT occurs and the event would otherwise be invisible to moderation | `crates/buzz-media/src/upload.rs:104-119`, `crates/buzz-media/src/upload.rs:428-443` |

---

### 8. Write ordering (publish gate)

| # | Rule | file:line |
|---|---|---|
| BR-53 | Order is: blob → derived artifacts (thumb) → upload record → sidecar. The sidecar is the serve gate, so a failed moderation record cannot publish media | `crates/buzz-media/src/upload.rs:129-172`, `crates/buzz-media/src/upload.rs:434-475` |
| BR-54 | Record existence implies the referenced objects are readable (record written after blob durability) | `crates/buzz-media/src/upload_record.rs:35-40`, `crates/buzz-media/src/upload.rs:158-172` |
| BR-55 | On metadata-generation failure the orphan blob is deliberately **not** deleted (concurrent same-hash uploads could race); logged at `warn` and left for GC | `crates/buzz-media/src/upload.rs:122-127`, `crates/buzz-media/src/upload.rs:132-141` |

---

### 9. Deletion / GC / orphan sweeping / retention

| # | Rule | file:line |
|---|---|---|
| BR-56 | The crate exposes a raw `delete(key)` and nothing else — no blob-lifecycle delete, no cascade to sidecar/thumb/record, no `DELETE /{sha256}` implementation | `crates/buzz-media/src/storage.rs:158-164` |
| BR-57 | GC is **not implemented**; a background job is named as future work ("A V2 background GC job can sweep blobs with no matching sidecar after a grace period") | `crates/buzz-media/src/upload.rs:122-127` |
| BR-58 | Orphan *accounting* exists: a blob sha with no sidecar binding in any community counts as an orphan blob; a sidecar binding whose sha has no blob counts as an orphan sidecar | `crates/buzz-media/src/bucket_index.rs:294-315` |
| BR-59 | Multi-variant anomaly: one sha with >1 blob extension is counted and all its variant bytes are billed | `crates/buzz-media/src/bucket_index.rs:298-306` |
| BR-60 | Logical billing double-counts a blob bound to N communities (documented as intentional); thumb bytes are attributed to the blob's sha | `crates/buzz-media/src/bucket_index.rs:210-213`, `crates/buzz-media/src/bucket_index.rs:317-334` |
| BR-61 | `_uploads/` auxiliary objects count toward physical totals only, never logical/orphan math | `crates/buzz-media/src/bucket_index.rs:266-268` |
| BR-62 | Sweep is bounded: cumulative object count checked **before** folding each page; breach → `CapExceeded` and the whole sweep fails (no partial snapshot) | `crates/buzz-media/src/bucket_index.rs:394-398`, `crates/buzz-media/src/bucket_index.rs:339-341` |
| BR-63 | `is_truncated == true` with no continuation token → `MalformedPage` (fail, don't return partial data) | `crates/buzz-media/src/bucket_index.rs:404-409` |
| BR-64 | Key classification is strict: 64-char lowercase hex sha, 1–8 alphanumeric ext, canonical lowercase UUID, uppercase-only 26-char ULID; anything malformed falls to `Unknown` rather than being coerced | `crates/buzz-media/src/bucket_index.rs:75-195` |

No retention/TTL/expiry logic exists anywhere in the crate (no lifecycle-policy code, no `expires_at` field).

---

### 10. Upload attribution / IP collection rules

| # | Rule | file:line |
|---|---|---|
| BR-65 | Upload records are off unless `upload_records_enabled` | `crates/buzz-media/src/config.rs:49-50`, gating at `crates/buzz-media/src/upload.rs:104`, `:158`, `:428`, `:464` |
| BR-66 | IP recording is fail-empty: only a single syntactically valid **public** address is recorded; private, loopback, link-local, CGNAT, documentation, ULA, Teredo, 6to4, v4-mapped, comma lists, `ip:port` all record nothing | `crates/buzz-media/src/upload_record.rs:191-256` |
| BR-67 | A port is recorded only alongside a valid IP (`ip.and(port)`) | `crates/buzz-media/src/upload_record.rs:151-153` |
| BR-68 | Startup fails if an IP header is configured without records enabled, or a port header without an IP header, or if either header name is not a valid HTTP header name | `crates/buzz-media/src/config.rs:91-124` |
| BR-69 | Community id/host and uploader pubkey in the record are server-resolved (`TenantContext`, verified auth event pubkey), never client-supplied | `crates/buzz-media/src/upload_record.rs:154-168`, `crates/buzz-media/src/storage.rs:204-209` |
