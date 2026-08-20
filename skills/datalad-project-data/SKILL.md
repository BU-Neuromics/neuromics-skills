---
name: datalad-project-data
description: "Use when designing DataLad datasets for research pipelines, especially when one project produces processed data that downstream analysis projects consume: dataset shape, what to annex versus commit to git, keeping clinical or PHI files out, siblings and where content bytes live, moving data between an HPC cluster and a laptop, recording provenance with datalad run, and pinning upstream data from a downstream repo. Invoke when the user mentions DataLad, git-annex, datalad get/push/clone, subdatasets, RIA stores, annex special remotes, migrating off DVC, or asks how a downstream project should consume another project's outputs reproducibly."
metadata:
  version: "0.2.0"
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

That last point is the actual argument for maintaining both: the annex record
is durable multi-version storage, the filetree record is the readable citable
artifact. If you keep both, publish only one — two DOIs for one dataset creates
citation ambiguity.

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
