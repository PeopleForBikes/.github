# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is the PeopleForBikes organization-wide `.github` repository. It holds reusable GitHub Actions workflows, and `profile/README.md`, which is rendered on the organization's GitHub page. Community health files added here become the defaults for the organization's repositories.

There is nothing to build, and no test suite, lint configuration or CI for this repository itself. The workflows can be checked locally with `uvx --from actionlint-py actionlint` and `uvx zizmor --offline .github/workflows`.

## Reusable workflows

Every file in `.github/workflows/` is triggered only by `workflow_call`: none of them run on pushes or PRs here. They execute in the context of the calling repository, so checkouts, `just` recipes and file paths all refer to the caller. Changes can only be exercised end to end from a calling repository that references the branch under test:

```yaml
jobs:
  ci:
    uses: PeopleForBikes/.github/.github/workflows/ci-python-uv.yml@<branch>
```

The repository has no tags and callers reference `@main`, so a merge takes effect everywhere immediately. Treat workflow file names, inputs, secrets and the caller contract below as a public API.

### Caller contract

The workflows rely on these conventions, which are not declared as inputs:

- **Python CI and release** (`ci-python*`, `release-python*`): a justfile with `lint-md`, `lint-python` and `docs` recipes, plus `test` for CI. The release workflows expect `just docs` to write to `docs/build/html/`, which is deployed to GitHub Pages, and a changelog whose entry becomes the GitHub release body.
- **uv variants**: `uv.lock` must be in sync (`uv lock --check`), and dependencies are installed with `--all-extras --dev`.
- **Zola docs** (`docs-zola`): a justfile with `lint-sass`, `lint-md` and `lint-spellcheck` recipes, and a site built in `<build-dir>/public`. The deploy job publishes the artifact of the build job, only on `refs/heads/main`.
- **Rust CI** (`ci-rust`): calls cargo directly on the whole workspace, without `just`. Clippy warnings are errors, tests and examples included. `clippy-exclude` skips packages such as generated code.
- **Node CI** (`ci-node`): npm or pnpm with an up-to-date lock file, and `lint` and `build` scripts in `package.json`, plus an optional `test`. With pnpm, the version comes from the `packageManager` field or from `pnpm-version`, never both. `artifact-path` uploads the build as the `site` artifact for the caller's own publishing job.
- **Docs linting** (`lint-docs`): a justfile with `lint-md` and `lint-spellcheck` recipes.
- **Lambda deployment** (`lambda-deploy-rust`): a committed `Cargo.lock`, and lambdas built with `cargo lambda` from `working-directory`. `stage` selects the concurrency group: callers must not add a group shared by several stages, or their deployments cancel each other. The region is an input because callers cannot use secrets in `with`.
- **Docker** (`docker-build-publish-ghcr`): despite its name, publishes to GHCR and/or AWS ECR, depending on `push-to-ghcr` and `push-to-ecr`. If neither is set, it builds without pushing. ECR publishing needs the `aws-region` input and the `github-role` secret (an AWS role ARN assumed through OIDC), and the ECR repository must have the same name as the GitHub repository. Images are built for amd64 and arm64, unless `platforms` says otherwise. `use-cache` stores layers in GHCR rather than the GitHub Actions cache, because callers run on tags and an Actions cache created on a tag is not visible to other tags. It is opt-in since cached `apt-get` or `git clone` layers are not refreshed.

The spelling recipe is always named `lint-spellcheck`: do not make it configurable.

### Permissions

A called workflow cannot request more than its caller has: an excessive request fails the caller at startup, even for a job that would be skipped. Hence:

- The CI workflows request `contents: read`.
- The release workflows request `contents: write` for their deploy and release jobs, so callers must grant it.
- `lambda-deploy-rust` requests `id-token: write` and `contents: read`, so callers must grant them.
- The `docs-zola` deploy job and the Docker workflow inherit the caller's permissions. They need `contents: write` for the former, and `packages: write` (GHCR) or `id-token: write` (ECR) for the latter.

Checkouts use `persist-credentials: false`.

### Caching

The CI workflows and `lambda-deploy-rust` (`use-cache`) cache dependencies, the release workflows deliberately do not: a cache created on a tag is not visible to other tags, and releases should not be built from files left by an earlier job.

### Paired files

The Python workflows come in Poetry (`ci-python.yml`, `release-python.yml`) and uv (`ci-python-uv.yml`, `release-python-uv.yml`) variants that are otherwise near duplicates. The setup steps are also repeated across jobs within a file, `ci-node.yml` included. When changing a shared step or bumping an action, apply it to every occurrence.

## Action pinning

Actions are pinned to a full commit SHA followed by a version comment:

```yaml
- uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
```

When bumping an action manually, update both the SHA and the comment. `dtolnay/rust-toolchain` has no usable release tags: pin it to a commit of its `master` branch (commits of its other branches are garbage collected) and pass `toolchain` explicitly. Dependabot proposes action updates weekly, with a 7-day cooldown.

Dependabot does not update the tool versions set in the workflows: `just-version` in every `setup-just` step and the default of `cargo-lambda-version`.

`git grep -hoE 'uses: .*' -- .github/workflows | sort | uniq -c` lists every pin, which makes version drift and unpinned actions easy to spot.
