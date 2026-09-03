---
name: file-storage-media
description: Use when designing or reviewing file upload/download features, object storage usage (S3/GCS/R2), presigned URLs, user-generated media (images, video, documents), export/report generation, file processing pipelines, or storage lifecycle/retention. Complements security-boundaries (upload validation) and distributed-reliability (async processing).
---

# File Storage & Media

Two structural rules. **Files don't flow through app servers:** uploads go client → object storage via presigned URLs, downloads go storage/CDN → client — proxying a 100MB file through your API ties up a worker for the whole transfer and turns Little's law against you (`capacity-estimation`: concurrency = rate × duration, and duration just exploded). **The database owns the truth about files, storage owns the bytes:** every file has a metadata row (owner, key, state, size, type), and all authorization happens against that row (`security-boundaries`) — the bucket is a dumb, private byte store.

## Decision 1: Where files live

Object storage (S3/GCS/R2-class), full stop, for anything user-generated or generated-for-users. Local disk dies with the instance and breaks the moment there are two nodes; the database is for the *metadata row*, not the bytes (blobs in Postgres bloat backups, WAL, and vacuum for zero benefit — `data-modeling`). Buckets are **private by default, always**; public access is granted per-object through a CDN or presigned URLs, never by bucket policy. Egress pricing is the sleeper cost: S3-class egress is ~$0.09/GB while R2-class is free — for download-heavy products (exports, media serving) this dominates the bill, so run the `capacity-estimation` bandwidth template before choosing.

## Decision 2: Upload flow — presigned, direct, confirmed

The standard sequence:

1. Client asks your API to start an upload (name, type, size).
2. API authorizes (quota, entitlements, type/size policy), creates a metadata row with state `pending`, returns a **presigned URL** — constrained to a server-generated key (never a client-chosen path), an expiry of minutes, and content-type/size conditions (presigned POST policies enforce these; plain PUT trusts the client more, prefer POST policies for user uploads).
3. Client uploads directly to storage.
4. Completion is confirmed server-side — a confirm call that verifies the object exists (HEAD) or a storage-event notification — and the row moves `pending → uploaded`. **Never trust the client's "done."**

Presigned URLs are bearer tokens: short expiry, HTTPS only, never logged (`security-boundaries` secrets rules). Files >~100MB use **multipart** (parallel parts, resumable) — and every bucket gets a lifecycle rule to **abort incomplete multipart uploads** after ~7 days, because abandoned parts bill invisibly forever.

Validate after upload, before use: type by **magic bytes** (not extension, not client Content-Type), size against policy, malware scan where the threat model warrants; images get re-encoded (strips EXIF/GPS and most polyglot-file attacks) — the `security-boundaries` upload rules, applied at the `uploaded → ready` transition.

## Decision 3: The metadata row and the state machine

`pending → uploaded → processing → ready` (+ `failed`, + `deleted`): the row carries id, owner/tenant, bucket+key, size, verified content type, state, and timestamps. Consequences of doing this properly:

- **Orphan cleanup is a query**, not an S3 listing: rows stuck in `pending`/`uploaded` past a TTL are abandoned uploads — a cron (Redis-locked on multi-node) deletes object + row. The reverse orphan (object with no row) shouldn't be possible if keys are only ever minted by the API.
- **Authorization is object-level** — every download/delete checks the row's owner/tenant, never "knows the URL" (an unguessable key is not access control — same argument as `data-modeling`'s non-enumerable IDs).
- Deletion policy is explicit: soft-delete the row, lifecycle-expire the bytes, and document the retention promise (GDPR deletion means the *object* goes, not just the row).

## Decision 4: Serving — downloads and media

- **Private files** (documents, exports): short-lived presigned GET, issued per request after the authz check. `response-content-disposition=attachment` for anything a browser shouldn't render inline (HTML/SVG uploads served inline from your domain = stored XSS — `security-boundaries`).
- **Public/semi-public media** (avatars, product images): CDN in front of the private bucket (origin-access, so the bucket stays sealed), long `max-age` + content-hashed or versioned keys so invalidation is a key change (`caching-strategy` Decision 6).
- **Images**: don't pre-generate every size — an on-the-fly transform service (image CDN or self-hosted transformer behind the CDN) turns "we need a 320px WebP now" into a URL parameter, with the CDN as the cache (`web-performance` Decision 2 consumes these).
- **Video** is its own domain (transcoding ladders, HLS/DASH): buy (Mux/Cloudflare Stream-class) unless video *is* the product.

## Decision 5: Processing pipelines

Anything beyond a HEAD check (thumbnails, PDF rendering, virus scan, transcoding, export generation) runs **async in a worker queue**, never in the request path: jobs are idempotent (keyed on file id + operation — retries are certain, `distributed-reliability`), write derived objects under deterministic keys (re-runs overwrite, not duplicate), update the state machine, and have a DLQ + a `failed` state the UI can show. Exports/reports are the same pattern in reverse: generate to object storage under a lifecycle-expiring prefix, hand the user a presigned GET; the `capacity-estimation` worked example (concurrency = peak rate × 30s generation) is exactly this feature.

## Decision 6: Lifecycle and cost

Storage grows monotonically unless you decide otherwise, and the bill is storage + requests + **egress**:

- Lifecycle rules per prefix, set at bucket creation: expire temp/export prefixes (days), abort incomplete MPUs (always), transition cold data to infrequent-access/archive tiers only when access patterns justify the retrieval costs.
- Retention is a *product* decision recorded per file class (user files: until deleted; exports: 30–90 days; logs/artifacts: policy) — `data-modeling` growth rules; state it in the design doc.
- Instrument: bucket size by prefix, upload failure rate, orphan-cleanup counts, egress GB (`observability-slo` business metrics).

## Review checklist

- Uploads direct-to-storage via presigned URL — nothing streaming through app servers?
- Presigned: server-minted key, short expiry, type/size conditions, never logged?
- Completion verified server-side; metadata state machine with orphan cleanup?
- Post-upload validation: magic bytes, re-encode images, disposition on serve?
- Every download authorized against the metadata row (owner/tenant)?
- Bucket private; public serving only via CDN with origin access?
- Multipart for large files + abort-incomplete lifecycle rule?
- Processing async, idempotent, DLQ'd? Exports lifecycle-expired?
- Egress cost estimated for the read pattern?

## Anti-patterns

- Files uploaded through / downloaded through the API server "because it was easier"
- Public bucket, or unguessable-URL-as-authorization
- Trusting client-reported completion, filename, or Content-Type
- Client-chosen storage keys (path traversal into other tenants' prefixes)
- Blobs in the database; metadata only in the bucket (no queryable row)
- No cleanup story: pending rows, abandoned multiparts, expired exports accumulating forever
- Serving user-uploaded HTML/SVG inline from the app domain
- Thumbnailing in the request handler; non-idempotent processing jobs
- Pre-generating every conceivable image size instead of transforming on demand
- Discovering egress pricing from the first big invoice
