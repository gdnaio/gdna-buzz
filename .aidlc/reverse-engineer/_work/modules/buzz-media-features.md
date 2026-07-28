## Module: buzz-media (`crates/buzz-media`)

### Aspect: Features

Completeness is judged against what the code does, not against the Blossom spec's full endpoint set.

| Feature | Completeness | Evidence |
|---|---|---|
| Content-addressed blob upload (image path) — sniff, validate, hash, store, thumbnail, blurhash, sidecar | Full | `crates/buzz-media/src/upload.rs:207-236`, `crates/buzz-media/src/thumbnail.rs:15-50` |
| Generic-file upload path (documents, archives, text, data) | Full | `crates/buzz-media/src/upload.rs:245-274`, `crates/buzz-media/src/validation.rs:159-209` |
| Streaming video upload (never fully buffered in RAM) | Full | `crates/buzz-media/src/upload.rs:292-511` |
| Server-side SHA-256 + `x`-tag hash verification | Full | `crates/buzz-media/src/upload.rs:84-86`, `crates/buzz-media/src/auth.rs:190-196` |
| Blossom kind-24242 auth verification (upload + get verbs) | Full for the two verbs implemented; `delete`/`list`/`mirror` verbs absent | `crates/buzz-media/src/auth.rs:6-10`, `crates/buzz-media/src/auth.rs:31-236` |
| Per-community sidecar metadata as tenant read gate | Full | `crates/buzz-media/src/storage.rs:177-221` |
| Idempotent re-upload short-circuit | Full | `crates/buzz-media/src/upload.rs:97-128`, `crates/buzz-media/src/upload.rs:424-453` |
| Thumbnail generation (320px JPEG) + 4×3 blurhash | Full for images; **none for video** ("no thumbnail — desktop handles that") | `crates/buzz-media/src/thumbnail.rs:29-38`, `crates/buzz-media/src/upload.rs:437-448` |
| Metadata/EXIF policy enforcement for JPEG/PNG/WebP/GIF/MP4 | Full as a *rejection* policy; no stripping/sanitizing is performed server-side | `crates/buzz-media/src/validation.rs:487-928` |
| Image-bomb protection (25 MP pre-decode cap, fail-closed on unparseable geometry) | Full | `crates/buzz-media/src/validation.rs:264-273` |
| MP4 structural validation (fast-start, codec, duration, resolution, box allowlist) | Full | `crates/buzz-media/src/validation.rs:289-395`, `crates/buzz-media/src/validation.rs:408-484`, `crates/buzz-media/src/validation.rs:831-928` |
| S3 range reads and streaming reads (HTTP 206 / large video serving support) | Full | `crates/buzz-media/src/storage.rs:118-146` |
| HEAD with size metadata | Partial — only `content_length` is surfaced; no ETag, last-modified, or content-type | `crates/buzz-media/src/storage.rs:167-175`, `crates/buzz-media/src/storage.rs:379-381` |
| Per-upload moderation records (`_uploads/` side channel) with opt-in IP/port capture | Full, off by default | `crates/buzz-media/src/upload_record.rs:139-199`, `crates/buzz-media/src/config.rs:45-61` |
| Bucket key taxonomy + storage-accounting sweep (physical/logical/orphan/anomaly gauges) | Full as a pure fold; the S3 driver is one page-listing method and the scheduling lives in the relay | `crates/buzz-media/src/bucket_index.rs:54-411`, `crates/buzz-media/src/storage.rs:242-265` |
| AWS credential-chain support (IRSA / Pod Identity) alongside static keys | Full | `crates/buzz-media/src/storage.rs:25-65` |
| Blob deletion / GC / orphan sweeping (actual deletes) | **Stubbed** — only a raw `delete(key)` primitive; no GC job, no cascade, no retention | `crates/buzz-media/src/storage.rs:158-164`, `crates/buzz-media/src/upload.rs:122-127` |
| Blossom BUD-02 `/list`, `DELETE /{sha256}`, BUD-04 mirror | **Absent** in this crate | no such fn in `crates/buzz-media/src/lib.rs:17-28`; relay routes only upload/get/head (`crates/buzz-relay/src/router.rs:38-45`) |
| Signed/presigned URL generation | **Absent** — no `presign_*` call anywhere | `crates/buzz-media/src/storage.rs:73-265` |
| Multipart upload | **Absent** — `put_object_stream_with_content_type` is used; no explicit multipart API | `crates/buzz-media/src/storage.rs:96-101` |
| Audio support | Deliberately **not supported**: sniffed `audio/*` is rejected on the generic path "until Buzz has an explicit sanitizer and location-metadata validator for its container" | `crates/buzz-media/src/validation.rs:186-196` |
| PDF inline preview | Deliberately deferred — "inline PDF preview is a planned fast-follow" | `crates/buzz-media/src/validation.rs:211-215` |
| Transcoding / re-encoding of stored media | **Absent** by design (validation rejects non-canonical inputs instead) | `crates/buzz-media/src/validation.rs:487-491` |

---

### TODO / FIXME / HACK / XXX comments

**None.** A repo-relative scan of all 12 `.rs` files in the crate returns zero matches for `TODO`, `FIXME`, `HACK`, or `XXX` (verified across `crates/buzz-media/src/*.rs` and `crates/buzz-media/tests/static_creds_minio.rs`).

Deferred work is instead recorded in prose comments. The forward-looking ones, quoted verbatim:

- "A V2 background GC job can sweep blobs with no matching sidecar after a grace period." — `crates/buzz-media/src/upload.rs:126-127`
- "PDF is intentionally *not* inline yet — inline PDF preview is a planned fast-follow; until the renderer handles it, force download like any other file." — `crates/buzz-media/src/validation.rs:213-215`
- "audio is rejected until Buzz has an explicit sanitizer and location-metadata validator for its container." — `crates/buzz-media/src/validation.rs:188-190`
- "Revert to crates.io once #449 lands upstream." (workspace-level, about the `aws-creds` fork this crate depends on) — `Cargo.toml:169`

For contrast, the adjacent relay handler *does* carry a TODO for the quota gap this crate does not cover: "TODO(v2): Add persistent per-pubkey storage quotas. Admission limits below bound active parser/storage work, but they do not cap durable bytes stored." — `crates/buzz-relay/src/api/media.rs:302-304`.
