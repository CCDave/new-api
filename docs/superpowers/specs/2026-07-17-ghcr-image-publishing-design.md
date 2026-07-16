# GHCR Image Publishing Design

## Goal

Prepare the fork for lightweight development and publish its Docker image to GitHub Container Registry without reusing the upstream project's release workflows or image namespace.

## Branch configuration

The GitHub repository's default branch will be `llmnex/main`.

The only protection applied to `llmnex/main` will be deletion protection. Direct pushes, force pushes, pull requests, approvals, and required status checks remain unrestricted while the development team is small.

These repository settings are configured in GitHub and do not require tracked repository files.

## Workflow isolation

The inherited upstream GitHub Actions workflows remain disabled in the fork. The fork adds one independent workflow:

```text
.github/workflows/llmnex-ghcr.yml
```

The new workflow does not modify or replace upstream workflow files, reducing conflicts when future upstream releases are merged.

## Triggers

The workflow publishes only in either of these cases:

1. A Git tag matching `llmnex-v*` is pushed.
2. A user starts the workflow manually and supplies an existing Git tag matching `llmnex-v*`.

Normal branch pushes do not build or publish images. A manual request is rejected when the supplied tag does not exist or does not use the `llmnex-v` prefix.

## Image build and publication

The workflow checks out the selected tag, writes that tag to `VERSION` in the temporary CI workspace, and builds the repository's existing Dockerfile for `linux/amd64`.

It authenticates to GHCR with the workflow-provided `GITHUB_TOKEN`. The job receives only these explicit permissions:

```yaml
contents: read
packages: write
```

A successful build publishes exactly two image tags:

```text
ghcr.io/cddave/llmnex-new-api:<Git tag>
ghcr.io/cddave/llmnex-new-api:latest
```

For example, Git tag `llmnex-v1.0.0-rc.21.1` publishes:

```text
ghcr.io/cddave/llmnex-new-api:llmnex-v1.0.0-rc.21.1
ghcr.io/cddave/llmnex-new-api:latest
```

The workflow uses the GitHub Actions build cache. It does not build ARM images, create a multi-architecture manifest, sign images, publish to Docker Hub, create a GitHub Release, or generate additional SHA tags.

## Failure behavior

Input validation, checkout, authentication, or image build failures stop the job before publication completes. `latest` is updated only as part of a successful image push. A failed run must not fall back to a branch or silently substitute another tag.

## First publication

The first intended release tag is:

```text
llmnex-v1.0.0-rc.21.1
```

After the first successful push creates the GHCR package, its visibility is changed to Public in the GitHub package settings. No Docker Hub credentials or additional repository secrets are required.

## Verification

Before merging, verify that:

1. The workflow YAML parses successfully.
2. Its automatic trigger accepts only `llmnex-v*` tags.
3. Its manual path rejects missing, nonexistent, and incorrectly prefixed tags.
4. The Docker build targets `linux/amd64` and uses the existing Dockerfile.
5. The workflow references only `ghcr.io/cddave/llmnex-new-api`.
6. The permissions are limited to `contents: read` and `packages: write`.
7. Normal branch pushes do not trigger publication.
