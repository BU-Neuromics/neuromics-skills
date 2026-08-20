---
name: dataverse-api
description: "Use when scripting or debugging the Dataverse native REST API: creating datasets and minting DOIs, generating Preview/Private URLs for reviewer access, publishing datasets, discovering required metadata fields and controlled vocabularies, or handling differences between Dataverse instances and versions. Invoke when the user automates a Dataverse deposit, mentions dataverse.harvard.edu or demo.dataverse.org, asks about X-Dataverse-key, dataset-json, previewUrl or privateUrl, anonymizedAccess, :publish, dataset collections and aliases, or needs to give journal reviewers access to an unpublished Dataverse dataset."
metadata:
  version: "0.3.0"
---

# Dataverse native API

Verified against `demo.dataverse.org` (v6.11) and `dataverse.harvard.edu`
(v6.10.1) on 2026-08-20, by real authenticated runs against both — including a
full deposit of a ~95 MB dataset. Where the two instances disagree, that is
called out; those disagreements are the most expensive surprises here.

## Start by fetching the instance's own spec

Every Dataverse instance serves its complete OpenAPI spec:

```bash
curl -sS 'https://demo.dataverse.org/openapi?format=json'   # 455 paths on 6.11
```

Do this **first**, against the instance you are actually targeting. It is
authoritative, it is current for that deployment, and it resolves endpoint
questions faster than the guides — whose native-API page is large enough that
extraction from it is unreliable.

Check the version too, because instances drift:

```bash
curl -sS 'https://<host>/api/info/version'
```

Harvard trailed demo by a minor version when this was written. **Never assume
an endpoint exists on a target instance because it exists on demo.**

## Authentication

Pass the token as the **`X-Dataverse-key` header**. The API also accepts a
`key` query parameter — avoid it, since query strings leak into access logs,
shell history, and proxy records.

Never write a token into a repository, a script, or a transcript. Read it from
a `chmod 600` file or an environment variable at call time.

## Creating a dataset

```
POST /api/dataverses/{collection-alias}/datasets
```

Accepts `application/json` (dataset-json), `application/ld+json`, or
`application/json-ld`. Optional `doNotValidate` query param. Datasets are
always created **inside a collection**; `root` is the fallback when the user
has no collection of their own.

A created dataset is a **private draft that already carries a DOI**, which is
what makes "reserve the identifier now, publish later" work.

**Required citation fields** — confirmed from the live metadata block, not the
docs:

`title`, `author`, `datasetContact`, `dsDescription`, `subject`

Discover them for any instance rather than trusting this list to stay current:

```bash
curl -sS 'https://<host>/api/metadatablocks/citation'
```

## Controlled vocabularies — probe, don't hardcode

`subject` is a controlled vocabulary. On 6.11 it holds: Agricultural Sciences,
Arts and Humanities, Astronomy and Astrophysics, Business and Management,
Chemistry, Computer and Information Science, Earth and Environmental Sciences,
Engineering, Law, Mathematical Sciences, Medicine Health and Life Sciences,
Physics, Social Sciences, Other — plus **`Demo Only`, which exists on demo and
not on production instances.** A value that validates on demo can therefore
fail on Harvard; this is the concrete reason to re-probe per instance.

Licenses come from their own endpoint:

```bash
curl -sS 'https://<host>/api/licenses'    # filter on .active
```

Active on 6.11: CC0 1.0, CC BY 4.0, CC BY-NC 4.0, CC BY-NC-ND 4.0,
CC BY-NC-SA 4.0, CC BY-ND 4.0, CC BY-SA 4.0, PDDL-1.0, ODC-By 1.0, ODbL 1.0,
OGL UK 3.0.

## Reviewer access — fully automatable

```
POST   /api/datasets/{id}/previewUrl?anonymizedAccess=true
GET    /api/datasets/{id}/previewUrl
DELETE /api/datasets/{id}/previewUrl
```

**The anonymized variant is just a query parameter** — the double-blind review
link needs no UI interaction. `DELETE` revokes it.

`privateUrl` exists at the same three verbs and is the pre-6.x name, retained
for compatibility. Prefer `previewUrl` on 6.x and fall back on older instances.

The link lets a holder view and download files, including restricted and
embargoed ones, **without a Dataverse account** — which is what makes it the
right mechanism for journal review of unpublished data. Verify the
anonymization by opening the link yourself; there is a history of identity
leaking via page HTML and via the hosting collection.

## Publishing

```
POST /api/datasets/{id}/actions/:publish?type=major
```

Also accepts `type=minor` and an `assureIsIndexed` flag.

**Publication is irreversible.** Never wire this into an automated pipeline or
a script that runs unattended. Require an explicit human confirmation in the
same action that calls it.

## Dataverse's version and DOI model

Two ways Dataverse differs from Zenodo and Figshare, both of which break
assumptions carried over from those platforms:

- **One DOI covers all versions.** There is no per-version DOI; the landing
  page carries a version picker. Code that expects a fresh DOI per version
  will misreport citations.
- **A new version is a mutable account draft that retains the published
  files.** So a "new version" flow does not start from empty — it starts from
  whatever the previous version held, and same-name uploads replace entries.

Harvard Dataverse deposit limits: **2.5 GB per file, 1 TB per researcher**,
open to researchers of any affiliation.

## Two 403s that mean completely different things

**A JSON 403 is your metadata.** Dataverse reports metadata-validation failures
as HTTP **403**, not 400, with the detail in the JSON `message`:
`"Validation Failed: Point of Contact E-mail is required. ... Author Name is
required. ..."`. Read the body; the status code alone is misleading.

**An HTML 403 is Harvard's WAF.** `dataverse.harvard.edu` sits behind a WAF that
returns a bare nginx `403` with an HTML body. Two distinct triggers, both
verified:

1. **The client.** Python's `urllib` is refused outright — including plain
   authenticated GETs that succeed from the curl binary against the same URL
   with the same token. Use an HTTP client that is not urllib.
   `demo.dataverse.org` does not do this at all, so code that works against
   demo can fail against production for reasons unrelated to your logic.
2. **Uploaded file content.** The WAF inspects file bodies and appears to score
   them cumulatively rather than matching a single rule. A markdown document
   discussing shell commands and HTTP status codes was refused, while every
   individual line from it — and the bare words `curl` and `wget` — uploaded
   fine on their own. Bisecting found a "first blocked prefix", and the
   offending line changed after edits, which is the signature of an anomaly
   score crossing a threshold, not a signature match. If a text file is refused
   while its neighbours in the same directory succeed, this is why. Do not
   waste time on permissions.

## Tabular ingest: CSV files gain a second representation

Dataverse runs **tabular ingest** on CSV, TSV, Stata, SPSS, R and Excel
deposits, parsing them into its own tab-delimited archival form to enable
preview, variable-level metadata and external tools. Be precise about what this
does, because the alarming description is the wrong one:

**Nothing is overwritten.** The original is retained as a first-class object.
The file metadata records `originalFileName`, `originalFileFormat` and
`originalFileSize`, and — importantly — the `md5`/`checksum` Dataverse reports
is the **original's**, not the derived file's. `?format=original` returns the
original byte-for-byte, and the web UI's download menu offers "the original
file" among its formats.

**But the default download is the derived file**, and it differs semantically.
On one 125-column clinical table, ingest rewrote `NA` to empty string in 3,614
of 26,375 cells; the rest of the delta was commas becoming tabs and 2,110 added
quote characters, accounting exactly for the byte difference. No values were
altered — but for a "not assessed" code, collapsing `NA` to blank is a real
change, and it is invisible to anyone who does not know to ask.

Worse, the derived form is **not reproducible across instances**: identical
input produced 128,506 bytes on one instance and 127,366 on another. So the
`.tab` cannot be treated as canonical, and its checksum will not match the
`md5` displayed beside it.

### Controlling it

- **At upload: `tabIngest: "false"`** in the `jsonData` field of
  `/datasets/:persistentId/add` prevents ingest. Verified from an ordinary
  account — **no superuser needed**. This is the only reliable control a
  depositor has.
- **Ingest is asynchronous.** The upload response *always* reports
  `tabularData: false`; re-query the file listing to see the real outcome.
  This produces convincing false negatives.
- **`uningest` is superuser-only.** An ordinary depositor gets
  `User @… is not permitted to perform requested action`, so there is **no
  after-the-fact remedy**. It must be prevented at upload.
- **There is no per-collection setting.** The only installation-level control
  is `:TabularIngestSizeLimit`, a global superuser database setting that sets a
  size *threshold*, not an on/off switch — which is why the same file may be
  ingested on one instance and skipped on another.
- Compressed files (`.csv.gz`) are not ingested, which is a workaround when you
  cannot set `tabIngest`.

If a client library uploads for you, check whether it can pass `tabIngest`
before assuming you have control.

## Publishing requires the parent collection to be published

Verified, with this exact error:

> This dataset may not be published because its host dataverse (NAME) has not
> been published.

So a freshly created collection blocks publication of everything inside it.
Publishing the collection exposes only its own name and description — no draft
datasets — but it is a step people discover at the worst possible moment.
Check `isReleased` on the collection early.

**Reviewer access is unaffected.** An anonymized Preview URL works on a draft
inside an *unpublished* collection, so nothing needs publishing for peer
review.

## Collection creation is permission-gated

`root` may report `permissionRoot: true`, in which case an ordinary account
cannot create collections directly under it (`User NAME is not permitted to
perform requested action`). Create sub-collections under a collection you
already own. Do not assume demo's permission model matches production's.

## Response shapes

- **Create dataset** returns `{"status":"OK","data":{"id":<int>,
  "persistentId":"doi:..."}}`. The DOI exists immediately, on a private draft,
  even inside an unpublished collection.
- **Preview URL** returns `token`, `link`
  (`<host>/previewurl.xhtml?token=...`), and a `roleAssignment` granting the
  `member` role, described as "A person who can view both unpublished
  dataverses and datasets."
- **Download a file**: `/api/access/datafile/{id}` serves the derived form;
  add `?format=original` for the deposited bytes. Both answer **`303`** with a
  redirect to a presigned object-store URL carrying a short expiry (one hour
  observed) — so follow redirects, and if you record the URL anywhere, record
  the API endpoint, never the presigned link.
- **Delete a file**: `DELETE /api/files/{id}` → `{"status":"OK","data":true}`.
  Useful for cleaning up after failed or exploratory uploads; file ids come
  from the version's file listing.

## Still unverified

- Embargo and restricted-access behavior end to end
- Per-file restriction via API (documented; not exercised here)
- The 30-day post-publish edit window
- Whether the WAF rules apply to other Dataverse installations, or are
  Harvard-specific — assume Harvard-specific until tested

For choosing between repositories and the surrounding publication workflow, see
`research-data-publication`. For Zenodo/InvenioRDM, see `zenodo-invenio-api`.
