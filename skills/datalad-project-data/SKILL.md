---
name: datalad-project-data
description: "Use when designing DataLad datasets for research pipelines, especially when one project produces processed data that downstream analysis projects consume: dataset shape, what to annex versus commit to git, keeping clinical or PHI files out, siblings and where content bytes live, moving data between an HPC cluster and a laptop, recording provenance with datalad run, and pinning upstream data from a downstream repo. Invoke when the user mentions DataLad, git-annex, datalad get/push/clone, subdatasets, RIA stores, annex special remotes, migrating off DVC, or asks how a downstream project should consume another project's outputs reproducibly. Also covers choosing a storage backend (S3/MinIO, WebDAV, rclone, rsync, encrypted remotes, self-hosting trade-offs), why an archival repository is the wrong place for a working store, publishing exactly one DOI, and registering published download URLs as annex sources with git annex registerurl so one citable record serves both humans and DataLad."
metadata:
  version: "0.5.0"
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

## Two storage roles, chosen independently

The most common design mistake is treating "where does the data live" as one
question. It is two, with different requirements and usually different answers:

| | **Working store** | **Published archive** |
|---|---|---|
| Lifetime | while the work is active | forever |
| Mutability | overwritten constantly | immutable once published |
| Visibility | private | public (or controlled) |
| Versions | all of them | the one behind the paper |
| Identity | a config value | a DOI you cite |

Keep these separate in your head and in your configuration. The git layer —
filenames, checksums, history — is small text and lives in a git host
regardless; only *content* needs a store.

Because content location is a **mutable, many-valued property** of a
content-addressed dataset, you can add, remove, and re-order stores forever
without changing the dataset or what consumers type. Never let a host become
part of the dataset's identity. This is what turns a provider shutting down
into a config change instead of a migration.

### Working store: pick per environment

git-annex ships many remote types — `S3`, `rsync`, `webdav`, `rclone`,
`directory`, `gcrypt`, `git-lfs`, `httpalso`, `external` and more. Check what
your build offers with `git annex version`. Practical picks:

- **S3-compatible** — the best-supported path. Points at anything speaking the
  S3 API, not just AWS: set `host=`, `protocol=https`, `requeststyle=path`,
  and `signature=v4` or `v2` to target MinIO or similar. Credentials from
  `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`.
- **WebDAV** — simplest thing to stand up. `url=`, credentials from
  `WEBDAV_USERNAME` / `WEBDAV_PASSWORD`; works against Nextcloud/ownCloud and
  friends. Notably it accepts `exporttree=yes` *and* `annexobjects=yes`, so one
  remote can hold a readable tree and annex objects together.
- **rclone** — reaches dozens of consumer and institutional cloud services,
  which often means using storage your institution already pays for.
- **rsync over SSH** — free and obvious when you control a reachable host.
  **Check direction and reachability before designing around it:** a host
  reachable *from* your cluster is not necessarily reachable from a laptop
  outside the institutional network, and a cluster requiring interactive 2FA
  for inbound SSH makes "pull from the cluster" miserable. Prefer pushing out
  from the cluster to somewhere both ends can reach.
- **`encryption=shared`** (or `hybrid`/`pubkey`) on any of the above — content
  is GPG-encrypted at rest, so an untrusted or even public host holds bytes it
  cannot read. This is the right answer for sensitive data on infrastructure
  you do not control, and it is available on S3, WebDAV, rsync and more.

**On self-hosting:** MinIO (single binary, S3 API) or a small WebDAV server are
both easy to run and work natively. But weigh it honestly — standing up a
TLS-terminated public service means certificates, patching, backups and uptime,
and it makes your own box a single point of failure for data recovery, which is
usually the thing you were trying to de-risk. For modest volumes, managed
object storage with a free tier is less work and more durable. Self-host when
you need data sovereignty or already run the infrastructure, not to save money
on gigabytes.

### Provisioning an S3 working store

`templates/s3-annex-store.yaml` in this skill directory is a CloudFormation
template that provisions a private bucket plus a least-privilege IAM user for
exactly this role. Offer it when someone is standing up a working store on AWS.
Prerequisite is an authenticated AWS CLI.

```bash
aws cloudformation deploy --template-file s3-annex-store.yaml \
  --stack-name annex-store --parameter-overrides BucketName=NAME \
  --capabilities CAPABILITY_NAMED_IAM        # required: it creates a named IAM user

aws iam create-access-key --user-name git-annex-store   # secret shown once

git annex initremote s3 type=S3 bucket=NAME \
    datacenter=us-east-1 encryption=shared partsize=1GiB
```

**`datacenter=` is git-annex's name for the AWS region.** `region=` only
applies when pointing at a non-AWS S3-compatible host, where you also set
`host=` and usually `requeststyle=path`.

Five decisions in that template are worth carrying into any equivalent you
write by hand, because each addresses a specific way this goes wrong:

- **`DeletionPolicy: Retain` on the bucket.** A stack teardown must never be
  the thing that destroys data.
- **Do not create the access key in the template.** CloudFormation stack
  outputs are readable by anyone holding `cloudformation:DescribeStacks`, so a
  secret placed there is far more exposed than it appears. Output the *command*
  that mints the key instead.
- **Abort incomplete multipart uploads on a lifecycle rule.** Interrupted
  multipart uploads do not appear in an object listing but are still billed,
  and git-annex uses multipart for large files — so a failed transfer can cost
  money indefinitely and invisibly.
- **Grant `s3:CreateBucket`, scoped to the single bucket.** git-annex's
  `initremote` attempts to create the bucket even when it already exists;
  without this, setup fails with a confusing AccessDenied.
- **Server-side encryption is not a substitute for `encryption=shared`.** SSE
  means the provider encrypts data it can still read. Client-side annex
  encryption means it never holds plaintext. For sensitive content you want the
  latter — and then the key lives in the dataset's `git-annex` branch, so
  losing that branch loses the data.

Versioning with a short noncurrent-expiry window is also worth enabling: it
buys a recovery window against an accidental `git annex drop --from`, without
paying to retain every version forever.

**One bucket per dataset, not one per lab.** S3 charges nothing for a bucket —
you pay for storage, requests and egress — so consolidating saves no money and
the choice is purely about boundaries. Per-bucket wins on three:

- **IAM blast radius** is per-project by construction. A shared bucket needs
  prefix-scoped policies (`Resource: .../project/*` plus an `s3:prefix`
  condition on `ListBucket`), which is achievable but must be authored
  correctly for every project, and a mistake fails open silently.
- **Cost attribution.** S3 cost allocation tags apply to buckets, not to
  prefixes inside them. Tag `Project` and `DataClassification` at creation.
- **Retention and compliance.** Study retention is administered per study, so a
  per-project bucket can be audited and eventually deleted wholesale; a prefix
  cannot.

The default quota is 100 buckets per account (raisable to 1,000), which is not
a constraint at lab scale. Bucket names are globally unique across all of AWS,
so adopt a convention like `<org>-<project>-annex`.

A shared bucket is reasonable for many small datasets in a single sensitivity
tier managed by one person — set `fileprefix=<project>/` per remote, which is
exactly what that parameter is for. A useful middle path is one bucket per
*sensitivity tier* with `fileprefix=` per project: it keeps the boundary that
matters for audit while reducing stack count, and gives up per-project cost
attribution.

Note that with `encryption=shared`, each dataset's content is encrypted under a
key held in *its own* `git-annex` branch — so a leaked bucket credential yields
ciphertext for datasets whose git repos the holder cannot read. That is a real
mitigation, but do not treat it as the access boundary; scope IAM properly
regardless.

### Do not use an archival repository as your working store

It is tempting, because a repository draft is private, network-reachable, and
free. It also works. But it is off-label, and the platform will tell you so:

- Repositories run **content transformations** on deposit. One silently
  converted CSV files to its own tab-delimited format — including opaque
  annex-key blobs it had no reason to touch — storing a derived copy of an
  82 MB key nobody will ever read.
- **Web application firewalls** score uploaded content. A markdown file
  discussing shell commands was refused outright while its neighbours uploaded
  fine.
- Multi-version storage **grows monotonically**, so an unpublished draft
  accumulates every version of every file, consuming a free community
  allocation for content that will never be browsed.

It is a curated-deposit system being fed blob traffic. Use it for the archive
role it was built for, and put the working store on infrastructure meant for
mutable object storage.

### Publish exactly one DOI, ever

If a workflow leaves you with two records for one dataset, publish one and keep
the other **reserved but never published**. A draft's identifier does not
resolve, is not indexed, and cannot be cited, so the ambiguity never reaches
the world. Write down which is which, and why, or someone will later publish
the second "for completeness" and you will have two citable identifiers for one
set of bytes.

The working store is infrastructure, not a citable object. Nobody cites their
object store.

### At publication, register the published URLs as annex sources

This is what makes one published DOI serve both audiences, and it is the payoff
for content-addressed storage. After the archive record is public, tell
git-annex that each key is also available there:

```bash
git annex registerurl MD5E-s82214524--b63a48ec…  \
  'https://<host>/api/access/datafile/<id>?format=original'
```

The `web` remote is enabled by default, so a consumer who clones the git layer
can now `datalad get` content straight from the citable record — no token, no
access to your private store, no dependence on your account continuing to
exist. The retrieval command never changed; only where it resolves.

Four things to get right:

- **Register the stable API URL, not whatever it redirects to.** These
  endpoints commonly answer `303` with a presigned object-store link carrying a
  short expiry (one hour in the case I measured). Register the redirecting URL,
  which re-signs on every fetch.
- **The `web` remote sends no authentication.** This works only for openly
  accessible files. Restricted or embargoed content still needs the
  authenticated remote, which is coherent — restricted data should not be
  anonymously fetchable — but it means the story is partial for a
  mixed-access dataset.
- **File identifiers change per version.** Where URLs are id-based rather than
  path-based, a new published version means new ids, so URL registration is a
  step in every release checklist, not a one-time action.
- **URL claims live in the `git-annex` branch**, so they propagate to every
  clone once pushed. Registering them is a publishing act in itself.

### Choosing the key backend

`MD5E` versus `SHA256E` is worth a deliberate decision rather than a default.
`SHA256E` is cryptographically stronger and the better default. But `MD5E`
embeds size and md5 in the key, and md5 is what most data repositories report
in their file metadata — so with `MD5E` you can verify annex keys directly
against a repository's own records, and against checksums recorded by whatever
tool you are migrating from. If reconciling with external checksums is part of
your workflow, match their algorithm. Changing later means `git annex migrate`,
which rewrites keys — a migration, not a config change.

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

## Publishing to Dataverse — vendor specifics

`datalad-dataverse` is the extension for this. Check its activity before
depending on it: as of 2026-08 the last release and last commit were both
2024-10-29, so treat it as stable-but-dormant rather than actively maintained.

Create the Dataverse dataset first — it is a **draft with a DOI already
assigned** — then `datalad add-sibling-dataverse` with the instance URL and that
DOI, and push. The sibling is usable immediately; publication is not a
prerequisite. Creation is scriptable via the native API, so no web-UI step is
required (see `dataverse-api`).

**`--mode` takes five values**, not two: `annex`, `filetree`, `annex-only`,
`filetree-only`, `git-only`. The two that matter:

- **`annex`** (default) — a sibling *tandem*: git history plus multi-version
  content storage. Content lands under `annex/<hash>/<hash>/` as git-annex
  keys, so `datalad get` works perfectly and a human browsing the web UI sees
  hashed blobs.
- **`filetree`** — a matching human-readable directory tree
  (`counts/all_counts.csv`), browsable and downloadable without DataLad.
  **It also deposits the git history and supports cloning** — it is not a dumb
  snapshot. Its real limitation is that only one snapshot of *content* is
  retained, so a consumer who pinned an older commit cannot fetch that
  version's bytes from it.

So the two modes look like the two roles above — annex for durable
multi-version storage, filetree for the readable citable artifact. Resist the
symmetry. Per **Publish exactly one DOI, ever**, only one of them should ever
become public, and the working-store role belongs on mutable object storage
rather than a second repository record.

**A trap specific to this platform: `filetree` mode cannot produce a faithful
readable record for tabular data.** Dataverse runs tabular ingest on deposit,
converting CSV/TSV to its own tab-delimited representation — so the readable
tree that was the entire point of the mode shows `all_counts.tab`, and the
default download is the derived file. The native API accepts a `tabIngest:
"false"` field to prevent this, but the extension does not send it (it builds a
fixed pyDataverse `Datafile` payload), and `uningest` is superuser-only, so a
depositor cannot fix it afterwards. If exact bytes matter in the published
record — and for a citable dataset they do — deposit the human-readable
snapshot by direct API call with `tabIngest` disabled, and let DataLad own the
versioned side only. Filed upstream as datalad/datalad-dataverse#340.

When a mode creates a storage and a regular sibling together, DataLad
**configures the publication dependency automatically**; no manual
`--publish-depends` is needed within a mode.

### The push sequence, which is not what you would guess

Three things bite in order, all verified:

```bash
# 1. First push to a sibling whose deposit does not exist yet MUST be plain
#    git. `datalad push` fetches before pushing and aborts with
#    "couldn't find remote refs (repository deposit does not exist...)".
git push <sibling> main git-annex

# 2. annex-mode content
datalad push --to <sibling>

# 3. filetree-mode content: `datalad push` uses `git annex copy`, which
#    refuses an exporttree=yes remote ("use git-annex export to store content
#    on it"). Export is the verb.
git annex export main --to <sibling>-storage
```

Repeated exports **replace rather than accumulate** — each run emits
`unexport` then `export` for changed paths. Verified over three
modify-export cycles: the remote file count stayed constant, content changed
each time, and checking out the original branch removed the test file
entirely. This was the risk worth testing, and it holds.

To modify an annexed file you must `datalad unlock` it first, or writes fail
with `Permission denied` against the read-only annex object.

### Credentials on a headless cluster

git-annex's external special remote resolves the API token through DataLad's
credential manager, which persists via `python-keyring`. The default
`SecretService` backend fails on a login node with `Prompt dismissed` — there
is no session to create a keyring collection in. Select a file backend:

```bash
export PYTHON_KEYRING_BACKEND=keyrings.alt.file.PlaintextKeyring
```

Inject the token by environment so it never lands in a config file. The secret
field is **`secret`**, not `token` — `..._TOKEN` is silently ignored and you
get "No suitable credential found":

```bash
export DATALAD_CREDENTIAL_<NAME>_SECRET="$(cat ~/.config/.../token)"
export DATALAD_CREDENTIAL_<NAME>_TYPE=token
export DATALAD_CREDENTIAL_<NAME>_REALM="https://<host>/dataverse"
```

Note that `PlaintextKeyring` writes the token to disk in cleartext under
`~/.local/share/python_keyring/`; restrict that directory, and treat it as
equivalent in sensitivity to the token file itself.

## Two traps that cost real time

**`text2git` is wrong for a data dataset.** It routes text files into git, and
a count matrix is text. Creating a dataset with `-c text2git` would commit an
79 MB CSV and a subject-level metadata table into git proper — bloating every
clone and putting sensitive columns permanently in history where they cannot be
withdrawn.

**`.gitattributes` can annex itself, and then nothing works.** Within
gitattributes the *last* matching line wins, so a catch-all placed after your
exceptions captures the attributes file too. It becomes a symlink, git reports
`unable to access '.gitattributes': Too many levels of symbolic links`, and
**every attribute silently becomes `unspecified`** — so all files get annexed
with the default backend and no rule you wrote applies. Put the catch-all
first, exceptions after, and add `.gitattributes annex.largefiles=nothing`
explicitly. Verify with `git check-attr annex.largefiles -- <paths>` before
saving any data, and consider a plain `git add` for the attributes file so it
lands in git regardless.

```
* annex.backend=MD5E
* annex.largefiles=anything
*.md annex.largefiles=nothing
*.sh annex.largefiles=nothing
*.py annex.largefiles=nothing
.gitattributes annex.largefiles=nothing
```

Using the `MD5E` backend has a side benefit: annex keys embed size and md5, so
they can be checked directly against checksums recorded by whatever tool you
are migrating from.

Unrelated but wasteful: `-q` is a *global* datalad option. `datalad save -q -m
MSG PATH` prints usage and does nothing; write `datalad -q save ...`.

## Downstream consumption

```bash
datalad clone -d . <data-repo-url> inputs/data
datalad get inputs/data/counts.csv
```

The subdataset records a commit SHA, so the downstream repo states exactly
which data version it used — the property that makes the pair reproducible.
Have downstream projects commit that pin rather than tracking the upstream
default branch.

**After cloning from a Dataverse record, the storage sibling is not
auto-enabled.** The clone succeeds and the file tree is visible, but the first
`datalad get` fails with `not available`. DataLad prints the fix; tell
consumers about it up front:

```bash
datalad siblings enable -s <sibling>-storage
```

Documentation kept in git rather than the annex is readable in a fresh clone
with no `get` at all, which is why a README and a provenance record belong
there.

For choosing an archive and the publication workflow, see the
`research-data-publication` skill. For scripting deposits, see
`zenodo-invenio-api` and `dataverse-api` — the latter covers creating the
draft dataset and DOI that `add-sibling-dataverse` requires.
