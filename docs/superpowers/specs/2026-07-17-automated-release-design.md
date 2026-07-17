# Automated Release Design

## Goal

Add a zero-input GitHub Actions entry point that creates the next fork Release from the current `llmnex/main` commit, generates its metadata, and starts the existing GHCR image publisher.

## Workflow isolation

Add one new workflow:

```text
.github/workflows/llmnex-release.yml
```

Keep `.github/workflows/llmnex-ghcr.yml` unchanged. The new workflow owns version calculation and GitHub Release creation, then starts the existing workflow through its `workflow_dispatch` input. The existing workflow remains available for Releases created through the GitHub UI and for manually rebuilding an existing tag.

## Trigger and branch

The new workflow has a zero-input `workflow_dispatch` trigger. It rejects runs whose selected ref is not `llmnex/main`, so every automatic Release points at the current commit of the fork's stable branch.

A concurrency group permits only one automated Release run at a time and does not cancel an in-progress Release.

## Version calculation

The workflow finds the closest official tag reachable from `llmnex/main` using tags that begin with `v`. For official base tag `v1.0.0-rc.21`, the fork Release prefix is:

```text
llmnex-v1.0.0-rc.21.
```

It scans existing tags with that exact prefix, finds the largest numeric suffix, and adds one. With no existing fork Release for that base, the first version is:

```text
llmnex-v1.0.0-rc.21.1
```

Subsequent versions are `.2`, `.3`, and so on. After the codebase moves to official tag `v1.0.0-rc.22`, the sequence automatically restarts at `llmnex-v1.0.0-rc.22.1`.

Tags whose suffix is not a positive integer are ignored for revision calculation. The calculated tag must not already exist.

## Release metadata

The workflow creates the Git tag and GitHub Release at the workflow's `llmnex/main` commit using GitHub CLI and `GITHUB_TOKEN`.

The Release title equals the calculated tag. GitHub generates the Release Notes automatically. Notes begin at the preceding fork Release for the same official base, or at the official base tag for the first fork Release.

If no commits exist between the notes start tag and the current commit, the workflow stops instead of creating an empty Release.

## Image publication handoff

Events created with `GITHUB_TOKEN` generally do not start another workflow automatically. After creating the Release, the new workflow explicitly dispatches `.github/workflows/llmnex-ghcr.yml` on `llmnex/main` and passes the new tag through its existing required `tag` input.

The GHCR workflow remains responsible for validating the tag and publishing:

```text
ghcr.io/ccdave/llmnex-new-api:<calculated tag>
ghcr.io/ccdave/llmnex-new-api:latest
```

## Permissions

The new workflow receives only:

```yaml
contents: write
actions: write
```

`contents: write` creates the tag and Release. `actions: write` starts the existing image workflow. The existing GHCR workflow keeps its current permissions.

## Failure behavior

The workflow stops without creating a Release when it is run from the wrong branch, no official base tag exists, no new commits exist, or the calculated tag already exists.

Concurrency prevents two zero-input runs from selecting the same revision simultaneously. If Release creation succeeds but image dispatch fails, the Release remains valid and the existing GHCR workflow can be run manually with that Release tag; the workflow does not delete published history.

## Verification

Before merging, verify that:

1. Both workflow YAML files pass `actionlint`.
2. The existing GHCR workflow has no diff.
3. The new workflow has no inputs and accepts only `workflow_dispatch`.
4. The branch guard requires `llmnex/main`.
5. Version calculation produces `.1` without prior fork tags and increments the highest numeric suffix.
6. A run with no new commits is rejected.
7. Release Notes use the correct preceding tag.
8. The new Release tag is passed to `llmnex-ghcr.yml` through `workflow_dispatch`.
