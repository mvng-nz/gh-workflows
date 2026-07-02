# mvng-nz/gh-workflows

Reusable GitHub Actions workflows for all MVNG app and package repositories.

## Critical lessons

These workflows encode hard-won lessons from brand-hub debugging:

- **PAT required for GitHub Packages**: Every workflow that installs `@mvng-nz` packages needs `NODE_AUTH_TOKEN` from a PAT secret. The auto `GITHUB_TOKEN` is repo-scoped and will 403 on cross-repo GitHub Packages downloads.
- **Permissions block replaces defaults**: Callers must set `permissions: contents: read, packages: read` minimum. Adding a `permissions` block **replaces** the default permissions — if you omit `contents: read`, `actions/checkout` fails with "Repository not found".
- **Always `yarn turbo run ...`**: Never use bare `turbo` — it is not on PATH. Use `yarn turbo run <task>`.
- **Node 24, corepack, immutable**: All workflows use Node 24, `corepack enable`, and `yarn install --immutable`.

> **Note**: Private repos on GitHub Free cannot use organization-level secrets. `PACKAGES_TOKEN` must be added per-repo.

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

Builds Storybook, runs component tests with Playwright, then deploys to Chromatic.

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
    uses: mvng-nz/gh-workflows/.github/workflows/changesets-release.yml@v1
    with:
      build-command: yarn turbo run build
    secrets:
      packages-token: ${{ secrets.PACKAGES_TOKEN }}
```
