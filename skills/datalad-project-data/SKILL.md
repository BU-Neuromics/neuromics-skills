---
name: datalad-project-data
description: "Use when designing DataLad datasets for research pipelines, especially when one project produces processed data that downstream analysis projects consume: dataset shape, what to annex versus commit to git, keeping clinical or PHI files out, siblings and where content bytes live, moving data between an HPC cluster and a laptop, recording provenance with datalad run, and pinning upstream data from a downstream repo. Invoke when the user mentions DataLad, git-annex, datalad get/push/clone, subdatasets, RIA stores, annex special remotes, migrating off DVC, or asks how a downstream project should consume another project's outputs reproducibly."
metadata:
  version: "0.1.0"
---

# DataLad for pipeline output data

Designing DataLad datasets when one project produces data and others consume
it.

## Why DataLad rather than a data-versioning tool

Be precise about what the user needs, because the answer differs:

- **Versioning** pins bytes — file, checksum, version. It records *what* the
  data was. DVC in `dvc add` mode, manifest-based tools, and plain checksums
  all do this.
- **Provenance** records derivation. `datalad run` captures the command, its
  inputs, and its outputs in the commit; `datalad rerun` re-executes it. And
  subdataset pinning gives a real dependency edge — a downstream repo records
  the exact commit SHA of the upstream data it consumed.

If the user only needs "get the same bytes back," simpler tools suffice. Choose
DataLad when they want to answer *which pipeline version produced this file*
and *what did the downstream analysis actually consume*. That second question
is the one that makes an upstream/downstream project pair reproducible.

## Installing on HPC without admin help

`datalad` and `git-annex` are frequently absent from cluster module systems,
but both are on conda-forge and need no admin request:

```bash
conda create -n datalad -c conda-forge datalad=1.6.2 git-annex=10.20260601
```

`git-annex` ships `nodep` builds on conda-forge, so **no Haskell toolchain is
required**. Pin both — an unpinned data-recovery dependency is a liability.

If the project is migrating off a DVC setup that depended on personal forks or
unreleased plugins, retiring that dependency is usually a larger win than the
migration itself. Say so.

## Dataset shape: thin data dataset versus one repo

Two defensible shapes when a pipeline repo produces publishable artifacts.

**Thin data dataset** — a separate dataset holding only the publishable
outputs, which the pipeline pushes to and downstream projects consume.

**One repo** — the pipeline repo *is* the dataset; downstream projects install
it as a subdataset and `datalad get` files out of its output directory.

Weigh these:

| | Thin dataset | One repo |
|---|---|---|
| Accidental-disclosure surface | only files you deliberately copied in | the whole worktree |
| Citable unit | "the data for this study" | "the pipeline repo" |
| Pin churn | SHA moves only when data moves | every code edit moves the SHA |
| Provenance link | must be recorded explicitly | free — same commit holds code and data |
| Machinery | second repo plus a push step | none |

**Default to the thin dataset when the data will be published**, and close its
one weakness by writing the producing pipeline's commit SHA into the data
dataset on every push. Without that, the code→data link is gone.

**Choose one repo when minimizing moving parts matters more** — but then
excluding sensitive directories is step one, not a later cleanup.

## Keep clinical and PHI files out — deliberately

A directory that is untracked *and* un-gitignored is the dangerous state: it
looks inert, and a routine `datalad save .` at the repo root stages it. If a
project holds clinical files (SPSS `.sav`, subject-level spreadsheets,
genotypes), the first commit of the migration should add them to `.gitignore`,
before anything else.

Do not rely on "I'll remember not to save that." And be aware that an
export-style publish snapshots the *worktree*, so an un-ignored directory
reaches the archive even if it was never committed.

Set annex policy explicitly rather than inheriting a default:

```
# .gitattributes
* annex.largefiles=(largerthan=10MB)
*.md annex.largefiles=nothing
*.csv annex.largefiles=anything
```

## Where the bytes live

The git layer and the content layer go to different places, and this is the
feature. Filenames, checksums, and history are small text and can live in a
private GitHub repo — reachable from anywhere. Only content needs a store.

Because content location is a **mutable, many-valued property** of a
content-addressed dataset, you can add, remove, and re-order stores forever
without changing the dataset or what consumers type. Design accordingly: never
let a host become part of the dataset's identity. This is what makes a
sunsetting provider a config change rather than a migration.

Common stores:

- **RIA store** — `datalad create-sibling-ria -s store ria+ssh://host:/path`.
  Good on shared cluster filesystems. Check whether that filesystem is actually
  backed up; many HPC "project" partitions are explicitly not.
- **rsync special remote over SSH** — takes an `ssh.example.com:/path` target,
  SSH transport by default. Supports `encryption=none|shared|hybrid|pubkey`,
  so an untrusted host can hold encrypted content. This is the answer when
  someone asks whether git-annex works over scp.
- **S3** — mature, first-class special remote support.

**Direction matters more than reachability.** If the cluster requires
interactive 2FA for inbound SSH, do not design a workflow where a laptop pulls
*from* the cluster — an authentication prompt in the middle of a `datalad get`
is miserable. Push *out* from the cluster to a store the other environment can
reach.

## Provenance with a workflow engine

When Nextflow, Snakemake, or similar owns execution, do not wrap individual
processes — the engine's own trace and DAG do that better. Wrap the **whole
invocation** with `datalad run` so one commit ties the pipeline's commit SHA,
its parameters, and its outputs together. That is the granularity that answers
"which pipeline version made this."

Note that published outputs must be real files, not symlinks into a scratch
work directory, for the dataset to be self-contained. Pipelines that publish by
copy satisfy this; ones that symlink do not, and their scratch directory then
cannot be deleted.

## Publishing to Dataverse

`datalad-dataverse` is the maintained extension. Create the Dataverse dataset
in the web UI first — it is a **draft with a DOI already assigned** — then
`datalad add-sibling-dataverse` with the instance URL and that DOI, and
`datalad push`. The sibling is usable immediately; publication is not a
prerequisite.

**The `--mode` choice matters and is not reversible in place:**

- **`annex`** (default) — git history plus annexed content. Full versioning and
  `datalad clone` works, but content sits under mangled annex-key paths. A
  human browsing the record sees hashed blobs, **not** filenames.
- **`filetree`** — a single snapshot as a readable directory tree, browsable and
  downloadable from the web UI by people not using DataLad. No version history,
  so content for older commits is not retained.

If both are wanted, they need two Dataverse datasets, and therefore two DOIs —
publish only one to avoid citation ambiguity. Chain them with
`datalad siblings --publish-depends <other>` (which sets
`remote.<name>.datalad-publish-depends`) so a single push updates both in order,
rather than leaving the second as a step someone must remember at the end of a
multi-year project.

Two caveats to state plainly:

- **Dataverse mints one DOI for all versions**, unlike Zenodo and Figshare.
  Do not assume per-version DOI semantics.
- **Repeated pushes to a `filetree` sibling are not documented.** The
  extension's tutorial covers a one-off export and stops; it does not cover
  repeated re-export, combining modes, or `publish-depends` against Dataverse.
  Validate on a throwaway Dataverse dataset with three or four
  push-modify-push cycles, confirming files are replaced rather than
  duplicated, **before** trusting it with real data.

## Downstream consumption

```bash
datalad clone -d . <data-repo-url> inputs/data
datalad get inputs/data/counts.csv
```

The subdataset records a commit SHA, so the downstream repo states exactly
which data version it used — the property that makes the pair reproducible.
Have downstream projects commit that pin rather than tracking the upstream
default branch.

For choosing an archive and the publication workflow, see the
`research-data-publication` skill. For scripting Zenodo deposits, see
`zenodo-invenio-api`.
