# neuromics-skills

Agent skills for computational research projects, from the
[BU Neuromics lab](https://github.com/BU-Neuromics).

These encode decisions that are expensive to rediscover — where research data
should live, how it becomes citable without leaking before publication, how
FAIR archive APIs actually behave as opposed to how they are documented, and
how to structure a dataset so that downstream analysis projects can reproduce
what they consumed.

## Install

Any agent (Claude Code, Cursor, Codex, Copilot, and others):

```bash
npx skills add BU-Neuromics/neuromics-skills
```

Add `--list` to see the skills without installing, or
`-s <name>` to pick individual ones.

Claude Code, as a plugin:

```
/plugin marketplace add BU-Neuromics/neuromics-skills
/plugin install research-data@neuromics-skills
```

## Skills

### `research-data-publication`

Choosing where data lives and how it becomes citable. The workspace/archive
split, why "private until publication" is usually the absence of a publish
rather than a privacy feature, granting journal reviewers access to unpublished
data, repository limits and DOI models compared across Zenodo, Dataverse, and
Figshare, and the controlled-access question to settle with an IRB *before*
designing a pipeline.

### `zenodo-invenio-api`

Empirically verified Zenodo and InvenioRDM REST API behavior, several points of
which contradict the documentation: publish-twice returns 404 so a retry must
reconcile rather than re-publish, `oc-checksum` strips leading zeros,
`Retry-After` appears on 200 responses, zero-byte files are accepted,
`files-import` is all-or-nothing and empty-draft-only, and
`?is_published=false` is silently ignored.

### `datalad-project-data`

DataLad dataset design for pipelines whose outputs feed other projects.
Versioning versus provenance, thin-dataset versus one-repo shape, keeping
clinical files out on purpose, where content bytes live and why a host must
never become part of a dataset's identity, `datalad run` granularity when a
workflow engine owns execution, and the `annex`/`filetree` mode choice when
publishing to Dataverse.

## Provenance of the API findings

The `zenodo-invenio-api` skill is harvested from `docs/zenodo-notes.md` in
[`BU-Neuromics/datapin`](https://github.com/BU-Neuromics/datapin), a
fixture-backed spike run against `sandbox.zenodo.org` on 2026-08-10. `datapin`
itself is no longer developed — mature tooling covers its ground — but the API
findings outlived it, which is why they live here as a skill rather than as
code.

## Contributing

Skills live in `skills/<name>/SKILL.md` with YAML frontmatter carrying `name`
and `description`. The `description` is what an agent matches against, so it
should enumerate concrete triggers rather than describe the topic abstractly.

Prefer recording what was *verified* and marking what was not. A skill that
distinguishes "confirmed against a live service" from "documented but untested"
is far more useful than one that reads uniformly confident.

## License

MIT — see [LICENSE](LICENSE).
