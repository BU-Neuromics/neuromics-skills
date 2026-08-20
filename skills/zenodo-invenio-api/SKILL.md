---
name: zenodo-invenio-api
description: "Use when writing or debugging code that talks to the Zenodo or InvenioRDM REST API: creating drafts, uploading files, publishing records, minting or reserving DOIs, importing files into a new version, handling rate limits and retries, or verifying downloaded checksums. Contains empirically verified API behavior that contradicts the documentation in several places. Invoke when the user is scripting a Zenodo deposit, hits a 400/404/429 from Zenodo, asks about sandbox.zenodo.org, files-import, concept versus version DOIs, oc-checksum, or automating dataset publication to an InvenioRDM instance."
metadata:
  version: "0.3.0"
---

# Zenodo / InvenioRDM API behavior

Empirical findings from a fixture-backed spike against `sandbox.zenodo.org`
(2026-08-10), harvested from `BU-Neuromics/datapin` `docs/zenodo-notes.md`.
Several of these **contradict the documented behavior** — they are recorded
because they cost real debugging time to discover.

**Caveat carried from the source:** sandbox can lag or lead production Zenodo.
Anything below marked ⚠ should be re-verified against production before relying
on it in either direction.

## Rules that prevent data-losing bugs

**Never blind-retry a publish.** After a successful publish (`202`) the draft
resource is *gone*, and POSTing publish again returns **`404 "Not found."`** —
not an error naming the record. So a publish that times out or 5xxes must be
reconciled by `GET /api/records/{id}` and checking `status == "published"`.
Treating the 404 as "the record vanished" will make a client destroy or
re-create a record that published successfully. Publish can 504 while
succeeding.

**`oc-checksum` strips leading zeros.** Downloads do **not** send
`Content-MD5`; they send **`oc-checksum: MD5:<hex>`**. That hex has leading
zeros stripped — an observed header of `da8f…` (31 chars) corresponded to a
true stream MD5 of `0da8f…`. **Left-pad to 32 characters before comparing**, or
verification will spuriously fail on roughly 1 in 16 files. Downloads do
support `Accept-Ranges: bytes`.

**`Retry-After` appears on `200` responses too.** A retry layer must key on
**status code**, never on header presence, or it will stall on successful
requests. Honor `Retry-After` first, then `X-RateLimit-Reset`.

## Rate limits

Buckets differ per endpoint class (sandbox observations):

| Endpoint class | Limit |
|---|---|
| general API (records CRUD) | ~133/min |
| **search** (`/api/records?q=`) | **30/min** — 429 at the 31st request |
| file content download | ~1000/min |

A real 429 carried both `retry-after: 49` and `x-ratelimit-reset`, body
`{"message":"30 per 1 minute","status":429}`. Search is the bucket you will
actually exhaust; back off there first.

## Publishing

- **`metadata.publisher` is required to publish** (DOI registration). Omitting
  it returns `400` with a structured `errors[]`. For Zenodo remotes, "Zenodo"
  is what the web UI defaults to.
- Validation errors arrive as `{status, message, errors[]}` where each entry is
  `{field, messages[]}`. **Surface `errors[]` verbatim** — it names the exact
  field, and paraphrasing it destroys the only actionable information.
- **A registered-but-not-uploaded file blocks publish**: `400` with
  `files: ["One or more files have not completed their transfer, please
  wait."]`. Preflight by listing draft files and `DELETE`ing entries with
  `status: "pending"` (leftovers from a crashed upload); deleting a pending
  entry returns `204`.
- **Publish responses are legacy-shaped.** They carry top-level `doi`,
  `conceptdoi`, `recid`, `state` rather than the InvenioRDM-reference
  `pids.doi.identifier`. Draft-create is a hybrid too. **Read DOIs
  defensively:** try `pids.doi.identifier`, then top-level `doi` /
  `conceptdoi`.

## Versions and DOIs

- **`POST …/versions` is idempotent** — called twice it returns `201` with the
  *same* draft id. Safe to re-run after a crash; no existence pre-check needed.
- A fresh version draft starts with **zero files**.
- **Concept DOI plus per-version DOIs.** The version chain is readable via
  `GET /api/records/{id}/versions`, with
  `metadata.relations.version[0] {index, is_last}`. `10.5072` is the
  sandbox/test prefix and those DOIs never resolve.
- **Reserving a DOI pre-publish works**: `POST …/draft/pids/doi` → `201` with
  the DOI at top-level `doi` and `doi_url`.
- **`GET …/versions/latest` returns a `301`** — follow redirects, or use
  `links.latest` from a fresh GET.
- **`DELETE …/draft` → `204`** and the id then 404s with "The persistent
  identifier does not exist." Discard leaves no trace.

## Files

Three-step flow: register the key(s), upload content, then commit — the commit
response carries the real `checksum: "md5:<hex>"`.

- **The file-count cap is enforced at registration, atomically.** Registering
  101 keys in one `POST …/draft/files` returns `400 "Uploading selected files
  will result in exceeding the max amount per record."` with **zero entries
  registered**. 100 keys register cleanly; one more key on a full draft fails
  the same way. The limit surfaces before any bytes move — a client-side
  preflight only improves the message.
- ⚠ **Zero-byte files are NOT rejected**, contradicting the documented
  expectation. They register, upload (`200`), commit (`200`, `size: 0`,
  `md5:d41d8cd98f00b204e9800998ecf8427e`) and publish (`202`). Keep an
  empty-file check as a *client lint* — empty files are almost never intended —
  but do not model it as a server-enforced failure.
- **Multipart is available**: register with
  `{"key":…,"size":…,"transfer":{"type":"M","parts":N,"part_size":…}}` → `201`
  with per-part URLs under `links.parts[]`. Those URLs expire in **~14 days**;
  use them promptly and re-register if stale.

## `files-import` — copy-by-reference, all-or-nothing, empty-draft-only

`POST …/draft/actions/files-import` copies **all** of the previous version's
files by reference — `201`, same `file_id`, fresh `bucket_id`, checksums
preserved, **no bytes moved**. It is the cheap path to a new version.

Constraints that dictate the whole flow:

- Running it on a **non-empty** draft returns `400` with
  `files.enabled: ["Please remove all files first."]`.
- There is **no partial or selective import**.

So the changed-files flow is necessarily: **import all → `DELETE` the changed
keys (`204`) → re-register, upload, commit those → publish.**

## Listing drafts

`GET /api/user/records` returns the **latest version per concept** with a
`status` field (`"draft"` / `"published"`).

⚠ **The documented-looking `?is_published=false` query param is silently
ignored** — it returns identical results. The **search-query form works**:
`?q=is_published:false`. Preflight cleanup of crashed pushes must use the `q=`
form *and* double-check `status`, and remember search is the 30/min bucket.

## Other

- **Resource types are instance-defined**: `GET /api/vocabularies/
  resourcetypes?size=100` returned 43 entries on sandbox. **Probe, don't
  hardcode.**
- Published records **cannot be deleted** via the API, and sandbox is
  periodically wiped — tests must create their own records and never assume
  persistence.

## Not probed by the source spike

Treat these as unknown rather than working: community submission flows,
**embargo and restricted-access behavior**, `files-import` when the previous
version has zero files, and the 30-day post-publish edit window (documented for
the UI, never API-verified).

For choosing between repositories and the publication workflow itself, see the
`research-data-publication` skill.
