# mvng-nz/gh-workflows

Reusable GitHub Actions workflows for MVNG app and package repositories.

## Overview

Shared workflow definitions for CI, deployment, releases, and Storybook testing. All workflows use:

- **Node 24** with Corepack and `yarn install --immutable`
- **GitHub Packages auth** via PAT (`NODE_AUTH_TOKEN`) with `scope: '@mvng-nz'`
- **Explicit `permissions` blocks** — callers must grant appropriate scopes
- **Yarn Turbo** for monorepo task orchestration (`yarn turbo run <task>`)

> **Note**: `PACKAGES_TOKEN` must be added as a repository secret in each consuming repo. A GitHub PAT with `packages:read` is required for all workflows; `packages:write` is additionally required for publish workflows.

## Versioning

This repo uses tagged releases. Consumers should pin to a major version tag (e.g. `@v1`) to get patch and minor updates automatically without breaking changes. For full reproducibility, pin to a specific version (e.g. `@v1.0.0`).

```yaml
# Recommended: major version pin
uses: mvng-nz/gh-workflows/.github/workflows/node-ci.yml@v1

# Fully pinned
uses: mvng-nz/gh-workflows/.github/workflows/node-ci.yml@v1.0.0
```

**Do not use `@main`** — any push to `main` immediately changes behavior for all consumers.

### Creating a new release

```bash
git tag v1.0.0
git push origin v1.0.0
# Update the moving major tag
git tag -f v1
git push origin v1 --force
```

> **Note**: This repo does not include a caller workflow. The reusable workflows require a `yarn.lock` and project dependencies to run, which don't exist in this infrastructure repo. Use the examples below as wiring templates in your consumer repos.

## Development

This repo uses Nix for the developer environment. After entering the shell (`nix develop` or with `direnv`), [Lefthook](https://github.com/evilmartians/lefthook) is installed automatically and registers Git hooks from `lefthook.yml`.

Pre-commit hooks run on staged files:

- `actionlint` — lint GitHub Actions workflow files under `.github/workflows/`

## Workflows

### node-ci.yml

Runs lint, build, and test, then uploads coverage artifacts.

**Inputs**

| Name              | Type    | Default                          | Description                          |
| ----------------- | ------- | -------------------------------- | ------------------------------------ |
| `node-version`    | string  | `'24'`                           | Node.js version                      |
| `run-command`     | string  | `yarn turbo run lint build test` | CI command to run                    |
| `upload-coverage` | boolean | `true`                           | Whether to upload coverage artifacts |

**Secrets**

| Name             | Required | Description                     |
| ---------------- | -------- | ------------------------------- |
| `packages-token` | yes      | GitHub PAT with `packages:read` |

**Example**

```yaml
jobs:
  ci:
    uses: mvng-nz/gh-workflows/.github/workflows/node-ci.yml@v1
    secrets:
      packages-token: ${{ secrets.PACKAGES_TOKEN }}
```

With custom inputs:

```yaml
jobs:
  ci:
    uses: mvng-nz/gh-workflows/.github/workflows/node-ci.yml@v1
    with:
      node-version: '22'
      run-command: yarn turbo run lint test
    secrets:
      packages-token: ${{ secrets.PACKAGES_TOKEN }}
```

---

### netlify-deploy.yml

Builds the project and deploys to Netlify.

**Inputs**

| Name            | Type    | Required | Default | Description                        |
| --------------- | ------- | -------- | ------- | ---------------------------------- |
| `build-command` | string  | yes      | —       | Command to build the project       |
| `publish-dir`   | string  | yes      | —       | Directory to publish (e.g. `dist`) |
| `production`    | boolean | no       | `false` | Deploy as production (`--prod`)    |

**Secrets**

| Name                 | Required | Description                     |
| -------------------- | -------- | ------------------------------- |
| `packages-token`     | yes      | GitHub PAT with `packages:read` |
| `netlify-auth-token` | yes      | Netlify personal access token   |
| `netlify-site-id`    | yes      | Netlify site ID                 |

**Example**

```yaml
jobs:
  deploy:
    uses: mvng-nz/gh-workflows/.github/workflows/netlify-deploy.yml@v1
    with:
      build-command: yarn turbo run build
      publish-dir: dist
      production: true
    secrets:
      packages-token: ${{ secrets.PACKAGES_TOKEN }}
      netlify-auth-token: ${{ secrets.NETLIFY_AUTH_TOKEN }}
      netlify-site-id: ${{ secrets.NETLIFY_SITE_ID }}
```

---

### changesets-release.yml

Creates a release pull request or publishes packages using changesets.

**Inputs**

| Name            | Type   | Required | Description                                 |
| --------------- | ------ | -------- | ------------------------------------------- |
| `build-command` | string | yes      | Command to build packages before publishing |

**Secrets**

| Name             | Required | Description                                     |
| ---------------- | -------- | ----------------------------------------------- |
| `packages-token` | yes      | GitHub PAT with `packages:write` for publishing |

**Example**

```yaml
jobs:
  release:
    uses: mvng-nz/gh-workflows/.github/workflows/changesets-release.yml@v1
    with:
      build-command: yarn turbo run build
    secrets:
      packages-token: ${{ secrets.PACKAGES_TOKEN }}
```

> The caller workflow must grant `permissions: contents: write, packages: write` for version commits and publishing.

---

### storybook-test-deploy.yml

Builds Storybook, runs component tests via `yarn turbo run test-storybook` (Storybook v10 turbo-compatible test runner), then deploys to Chromatic using the official [`chromaui/action`](https://github.com/chromaui/action) GitHub Action. The Action forwards the `pull_request`/`push` event context to Chromatic, fixing branch detection and PR status checks. Includes workflow-level `concurrency` to cancel superseded runs.

**Secrets**

| Name                      | Required | Description                     |
| ------------------------- | -------- | ------------------------------- |
| `packages-token`          | yes      | GitHub PAT with `packages:read` |
| `chromatic-project-token` | yes      | Chromatic project token         |

**Example**

```yaml
jobs:
  storybook:
    uses: mvng-nz/gh-workflows/.github/workflows/storybook-test-deploy.yml@v1
    secrets:
      packages-token: ${{ secrets.PACKAGES_TOKEN }}
      chromatic-project-token: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
```

---

### flutter-package-ci.yml

Runs `flutter analyze`, `dart format --set-exit-if-changed`, and `flutter test` across a matrix of Flutter versions. Uses `subosito/flutter-action` to install Flutter.

**Inputs**

| Name               | Type   | Default                                            | Description                                                                           |
| ------------------ | ------ | -------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `flutter-versions` | string | `'[{"version": "3.41.0"}, {"channel": "stable"}]'` | JSON array of objects with `version` (e.g. `"3.41.0"`) or `channel` (e.g. `"stable"`) |
| `analyze-flags`    | string | `--fatal-infos`                                    | Flags passed to `flutter analyze`                                                     |
| `runs-on`          | string | `ubuntu-latest`                                    | Runner image to use                                                                   |

**Example**

```yaml
jobs:
  ci:
    uses: mvng-nz/gh-workflows/.github/workflows/flutter-package-ci.yml@v1
```

With custom inputs:

```yaml
jobs:
  ci:
    uses: mvng-nz/gh-workflows/.github/workflows/flutter-package-ci.yml@v1
    with:
      flutter-versions: '[{"channel": "stable"}, {"channel": "beta"}]'
      analyze-flags: '--fatal-infos --fatal-warnings'
      runs-on: macos-latest
```

---

### flutter-release.yml

Creates a GitHub Release from a git tag, extracting the release notes from the `## [version]` section of `CHANGELOG.md`. Uses the `gh` CLI (preinstalled on GitHub-hosted runners) — no third-party action dependency.

**Example**

```yaml
on:
  push:
    tags:
      - 'v*'

permissions:
  contents: write

jobs:
  release:
    uses: mvng-nz/gh-workflows/.github/workflows/flutter-release.yml@v1
```

> The caller workflow must grant `permissions: contents: write` for release creation.

---

### package-publish.yml

Publishes a single package to GitHub Packages on release. Validates that the git tag matches `package.json` version before publishing.

**Secrets**

| Name             | Required | Description                                     |
| ---------------- | -------- | ----------------------------------------------- |
| `packages-token` | yes      | GitHub PAT with `packages:write` for publishing |

**Example**

```yaml
on:
  release:
    types: [published]

permissions:
  contents: read
  packages: write

jobs:
  publish:
    uses: mvng-nz/gh-workflows/.github/workflows/package-publish.yml@v1
    secrets:
      packages-token: ${{ secrets.PACKAGES_TOKEN }}
```

---

## Config package usage

Simple config packages (e.g. `prettier-config`, `eslint-config`) that don't use turbo or have tests can still use `node-ci.yml` with custom inputs:

```yaml
jobs:
  ci:
    uses: mvng-nz/gh-workflows/.github/workflows/node-ci.yml@v1
    with:
      run-command: yarn lint
      upload-coverage: false
    secrets:
      packages-token: ${{ secrets.PACKAGES_TOKEN }}
```

For publishing, use `package-publish.yml` triggered on GitHub release:

```yaml
on:
  release:
    types: [published]

permissions:
  contents: read
  packages: write

jobs:
  publish:
    uses: mvng-nz/gh-workflows/.github/workflows/package-publish.yml@v1
    secrets:
      packages-token: ${{ secrets.PACKAGES_TOKEN }}
```

## Required secrets

All workflows require `PACKAGES_TOKEN` — a GitHub Personal Access Token with at least `packages:read` (and `packages:write` for the changesets-release and package-publish workflows). Add it as a repository secret in each consuming repo.

| Secret                    | Used by                   |
| ------------------------- | ------------------------- |
| `PACKAGES_TOKEN`          | All workflows             |
| `NETLIFY_AUTH_TOKEN`      | netlify-deploy.yml        |
| `NETLIFY_SITE_ID`         | netlify-deploy.yml        |
| `CHROMATIC_PROJECT_TOKEN` | storybook-test-deploy.yml |

## Caller permissions

Every caller workflow must include a `permissions` block. At minimum:

```yaml
permissions:
  contents: read
  packages: read
```

For `changesets-release.yml` and `package-publish.yml`, the caller needs:

```yaml
permissions:
  contents: write
  packages: write
```

For `changesets-release.yml`, also add `pull-requests: write`:

```yaml
permissions:
  contents: write
  packages: write
  pull-requests: write
```

If a caller workflow mixes read-only and write jobs, use per-job `permissions` overrides:

```yaml
permissions:
  contents: read
  packages: read

jobs:
  ci:
    uses: mvng-nz/gh-workflows/.github/workflows/node-ci.yml@v1
    secrets:
      packages-token: ${{ secrets.PACKAGES_TOKEN }}

  release:
    permissions:
      contents: write
      packages: write
      pull-requests: write
    uses: mvng-nz/gh-workflows/.github/workflows/changesets-release.yml@v1
    with:
      build-command: yarn turbo run build
    secrets:
      packages-token: ${{ secrets.PACKAGES_TOKEN }}
```

## Repository settings

Consumers of `changesets-release.yml` must enable GitHub Actions to create pull requests:

1. Go to **Settings → Actions → General**
2. Scroll to **Workflow permissions**
3. Check **"Allow GitHub Actions to create and approve pull requests"**
4. Save

This is required because `changesets/action` creates a "Version Packages" PR using `GITHUB_TOKEN`, which GitHub blocks by default. The caller workflow must also grant `pull-requests: write` in its `permissions` block — without it, the checkbox may not appear.

> **Org restriction**: If the checkbox is missing, an org admin may have disabled it at **Organization Settings → Actions → General → Workflow permissions**.
