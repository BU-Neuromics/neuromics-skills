---
name: dataverse-api
description: "Use when scripting or debugging the Dataverse native REST API: creating datasets and minting DOIs, generating Preview/Private URLs for reviewer access, publishing datasets, discovering required metadata fields and controlled vocabularies, or handling differences between Dataverse instances and versions. Invoke when the user automates a Dataverse deposit, mentions dataverse.harvard.edu or demo.dataverse.org, asks about X-Dataverse-key, dataset-json, previewUrl or privateUrl, anonymizedAccess, :publish, dataset collections and aliases, or needs to give journal reviewers access to an unpublished Dataverse dataset."
metadata:
  version: "0.1.0"
---

# Dataverse native API

Verified against `demo.dataverse.org` (v6.11) and `dataverse.harvard.edu`
(v6.10.1) on 2026-08-20.

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

## Not yet verified — treat as unknown

The OpenAPI spec documents **every response as a bare `OK`**, so response
bodies are not established from it:

- Whether dataset creation returns the DOI in `data.persistentId` (documented
  behavior, unconfirmed here)
- The exact shape of preview-URL and publish responses
- The file upload flow end to end
- Whether Harvard 6.10.1 diverges from demo 6.11 on any of the above

Confirm these against a **demo instance first** — it is a separate service with
its own account and token, and its DOIs are test identifiers. Update this skill
with the observed shapes once a real run has happened.

For choosing between repositories and the surrounding publication workflow, see
`research-data-publication`. For Zenodo/InvenioRDM, see `zenodo-invenio-api`.
