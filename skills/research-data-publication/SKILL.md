---
name: research-data-publication
description: "Use when planning where research data will live and how it becomes citable: choosing between Zenodo, Dataverse, and Figshare; keeping data private until publication; granting journal reviewers access to unpublished data; minting DOIs; embargoes and restricted access; controlled-access requirements for human-subject data; or migrating off a sunsetting data host. Invoke when the user asks where to host processed data, how to publish a dataset, how to share data with reviewers, what repository to deposit in, how to keep data private until a paper is accepted, whether data needs dbGaP or controlled access, or mentions DOI, FAIR, data availability statement, embargo, InvenioRDM, OSF, or supplemental data."
metadata:
  version: "0.1.0"
---

# Research data publication

Deciding where research data lives, and how it becomes citable without leaking
before publication.

## The reframe that resolves most confusion

Users usually ask a single question — "where should the data go?" — that is
really two questions with different answers:

| | **Workspace track** | **Archive track** |
|---|---|---|
| For | results in flight, regenerated often | the finished, citable dataset |
| Consumers | you, collaborators, downstream pipelines | readers, reviewers, reusers |
| Mutability | overwrite freely | published versions are immutable |
| Identity | a path or a name | a DOI that resolves forever |
| Privacy | access control on the store | public, embargoed, or restricted |

Most tooling frustration comes from trying to make one system do both jobs.

**The most important consequence: "private until publication" is usually the
absence of a publish, not a privacy feature.** Keep data on the workspace
track and deposit it when the paper is ready. Do not reach for draft records,
embargo flags, or restricted access to solve a problem that "haven't published
yet" already solves. Reach for those only when something external — a
reviewer, a funder mandate, a submission requirement — forces a record to exist
before the data can be public.

## Gate this before designing anything

If the data are human-subject, settle whether they can be openly deposited
**before** designing the pipeline. Per-subject clinical variables combined with
genotype or expression data are frequently restricted to controlled access
(dbGaP or equivalent) by IRB or funder terms, and **an embargo does not satisfy
a controlled-access requirement** — embargoes end in open access, which is
precisely what the requirement forbids.

Ask explicitly. If the answer is "not sure," treat controlled access as the
working assumption and design so the open-deposit path can be added later:
publish derived, non-identifying summaries openly and keep per-subject data on
the workspace track. Do not silently assume open deposit is available because
it makes the plan simpler.

## Choosing an archive

Verified limits as of 2026-08; re-check before relying on them.

| | **Zenodo** | **Harvard Dataverse** | **Figshare** |
|---|---|---|---|
| Per file | part of the record cap | 2.5 GB | 20 GB free / up to 5 TB institutional |
| Per record | 50 GB, 100 files (200 GB on request) | 1 TB per researcher | 20 GB private free tier |
| Eligibility | anyone | anyone, any institution | anyone |
| DOI model | **one per version** | **one DOI for all versions** | one per version |
| Draft sharing | metadata always public once published | Preview URL, incl. anonymized | private item + reviewer link |

Decision shortcuts:

- **Default to Zenodo** for open deposit of a finished dataset. Widely
  understood, generous limits, per-version DOIs.
- **Choose Dataverse when reviewers need pre-publication access**, because its
  Preview URL is the best-designed mechanism for that (see below).
- **Watch the DOI model.** Dataverse issuing one DOI across all versions
  changes how you cite a specific version — do not assume Zenodo's per-version
  semantics generalize.
- **Check the file-count cap, not just total size.** A per-sample output
  directory can blow through Zenodo's 100-file limit while being small; tar it.

## Reviewer access before publication

Journals often require supplemental data at review while the work is still
unpublished. Dataverse handles this directly: generate a **Preview URL** on an
unpublished draft and anyone holding it can view *and download* files —
including restricted and embargoed ones — with **no account**. There is a
**"Create Anonymous Preview URL"** variant for double-blind review that strips
author, depositor, contact, producer, production place, and distributor from
metadata, citations, and version history. Access is revocable by disabling the
URL.

Two things to tell the user:

1. **A reviewer clicks download links in a web UI.** They will not install a
   CLI or use an API token. So the record they are pointed at must contain
   real filenames in a readable tree. Any storage layout that writes
   content-addressed or hashed object names is useless for review even though
   it is perfect for machines. If a workflow can produce either, the
   human-readable one must exist *at submission*, not at publication.
2. **Verify the anonymization yourself.** Dataverse's own guides advise opening
   your anonymous link and confirming it reveals nothing, and that advice
   exists because there have been real leaks — author names recoverable from
   dataset page HTML, and identity inferable from the collection or
   installation hosting the draft. If the journal is genuinely double-blind,
   test before sending.

Zenodo is weaker here: **record metadata is always public**, even for
restricted and embargoed records. Creating a Zenodo record early to have a
stable target publishes the dataset's title, description, and authorship while
the work is unpublished. Zenodo's restricted access with access requests suits
"public record, gated files" after publication — not pre-publication secrecy.

## Rules that prevent expensive mistakes

- **Publication is irreversible.** Zenodo and Dataverse both refuse to
  un-publish. Say this out loud before any publish step, and never run one
  without explicit confirmation in the same turn.
- **Rehearse on a sandbox.** `sandbox.zenodo.org` is a separate service with
  its own account and token, minting `10.5072` DOIs that never resolve — which
  is exactly what a dry run needs. Do the first end-to-end run there.
- **Do not mint two DOIs for one dataset.** If a workflow maintains both a
  machine-readable and a human-readable record, publish one and either keep the
  other unpublished or link it explicitly as a companion. Two published DOIs
  for the same data creates real citation ambiguity.
- **After publication, syncing stops being free.** Pushing to a published
  record creates a new unpublished draft version needing another publish
  action. This is correct — the record behind a paper should not mutate
  silently — but it means automation that "kept everything in sync"
  pre-publication no longer does.
- **A host you do not control can disappear.** OSF sunsetted its projects
  product; plan so the deposit target is swappable. Prefer designs where
  content location is configuration rather than identity.

## What to leave behind for reproducers

A dataset is only reusable if someone can tell what produced it. Record, in
the dataset or its metadata: the pipeline repository URL and the **exact commit
SHA** that produced these files, the parameters used, and checksums per file.
If the archive record is separate from the code repository, that SHA is the
only link between them — omit it and the provenance is gone.

For DataLad-managed datasets and the upstream/downstream project relationship,
see the `datalad-project-data` skill. For Zenodo and InvenioRDM API behavior
when scripting deposits, see the `zenodo-invenio-api` skill.
