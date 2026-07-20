# Plan: Improve mvng-nz/gh-workflows reusability for super-q

## Purpose

Refactor the upstream reusable workflows so `mvng-nz/super-q` can stop inlining steps in its `.github/workflows/*.yml` files and call reusable workflows from this repo. The immediate focus is a private-package Changesets release flow, a Playwright E2E reusable workflow, and a consistent `.nvmrc`-based Node setup across all reusable workflows.

## Scope

**In scope:**
- Modernize `.github/workflows/changesets-release.yml` for private-package tag-based releases.
- Add a new `.github/workflows/playwright-e2e.yml` reusable workflow.
- Add `node-version-file` support to `.github/workflows/node-ci.yml`, `.github/workflows/react-native-package-ci.yml`, `.github/workflows/cloudflare-pages-deploy.yml`, `.github/workflows/storybook-test-deploy.yml`, and the modernized `.github/workflows/changesets-release.yml`.
- Add an optional `ref` input to `.github/workflows/cloudflare-pages-deploy.yml`.
- Update `README.md` and `CHANGELOG.md` to document the new and changed workflows and the release tag.

**Out of scope:**
- Any changes to `mvng-nz/super-q` application code or tests.
- New `super-q` app features.
- Changes to `.github/workflows/package-publish.yml` or `.github/workflows/react-native-release.yml` unless required by the `changesets-release.yml` refactor. `package-publish.yml` is excluded because it is a single-package npm publish workflow that validates its tag against `package.json` and has no downstream consumer requesting `.nvmrc` support; it will be addressed separately if needed.
- Consolidation of `node-ci.yml` and `react-native-package-ci.yml` (tracked in a separate deferred plan).

## Assumptions

- `mvng-nz/gh-workflows` uses semantic-version tags (`v1`, `v1.x`) and downstream callers reference `@v1`. [verified: `README.md`]
- `changesets/action@v2` (currently `v2.0.0-next.3`) exposes `version-script`, `publish-script`, `setup-git-user`, `create-github-releases`, and `push-git-tags`. [verified in `v2.0.0-next.3`; confirm before relying on a stable release]
- Downstream consumers use Changesets v3 or later, which is required for `changesets/action@v2` compatibility. [unverified]
- Downstream consumers that need private-package releases configure `privatePackages: { "version": true, "tag": true }` in their Changesets config so `yarn changeset publish` produces git tags and GitHub releases without npm publishing. [unverified]
- Consumers have `.nvmrc` at the repo root; falling back to Node `24` is acceptable when `.nvmrc` is absent and no override is supplied. [verified: `README.md` states Node 24 default]
- Existing `@v1` callers of `changesets-release.yml` must not break until a new major version is published; use backward-compatible defaults. [requirement]
- `actionlint` and `lefthook` are available in the Nix dev shell. [verified: `lefthook.yml`, `README.md`]

## Dependencies

- None within this repo. Downstream `super-q` caller updates (`release.yml`, `web-e2e.yml`) happen after the new tag is published.

## Implementation steps

1. **Add `.nvmrc`-based Node setup across existing reusable workflows** — `.github/workflows/node-ci.yml`, `.github/workflows/react-native-package-ci.yml`, `.github/workflows/cloudflare-pages-deploy.yml`, `.github/workflows/storybook-test-deploy.yml`
   - Add `node-version` and `node-version-file` inputs; default `node-version-file` to `.nvmrc` and keep a `node-version` fallback of `'24'`.
   - Implement the resolution in each affected workflow with a step like. Note that `.github/workflows/storybook-test-deploy.yml` has separate `test` and `deploy` jobs; apply the resolution to both.
     ```yaml
     - name: Resolve Node version
       id: resolve-node
       run: |
         if [ -n "${{ inputs.node-version }}" ]; then
           echo "version=${{ inputs.node-version }}" >> "$GITHUB_OUTPUT"
         elif [ -f .nvmrc ]; then
           echo "version=$(cat .nvmrc)" >> "$GITHUB_OUTPUT"
         else
           echo "version=24" >> "$GITHUB_OUTPUT"
         fi
     - name: Setup Node
       uses: actions/setup-node@v6
       with:
         node-version: ${{ steps.resolve-node.outputs.version }}
         cache: yarn
         registry-url: https://npm.pkg.github.com
         scope: '@mvng-nz'
     ```
   - Preserve existing behavior for callers that already pass `node-version`.

2. **Modernize `changesets-release.yml`** — `.github/workflows/changesets-release.yml`
   - Before starting, verify that downstream consumers (especially `super-q`) are using Changesets v3 or later, because `changesets/action@v2` requires Changesets v3 compatibility. Verify by checking the consumer's lockfile or `yarn why @changesets/cli` output to confirm the resolved version is 3 or higher.
   - Confirm the downstream Changesets config enables `privatePackages: { "version": true, "tag": true }` so private packages get git tags and GitHub releases without npm publishing.
   - Upgrade `changesets/action@v1` to `changesets/action@v2.0.0-next.3` (or the latest stable `v2` once released and tested).
   - Add `workflow_call` inputs: `version-script`, `publish-script`, `create-github-releases`, `push-git-tags`, `setup-git-user`, `github-token`, `node-version`, `node-version-file`, `node-options`, `build-command`. The `github-token` input is provided for callers who need to override with a custom PAT, although `changesets/action@v2` defaults to `${{ github.token }}`. Pass the `changesets/action` inputs through directly in the `with:` block; the workflow inputs already use the same kebab-case names the action expects (`setup-git-user`, `create-github-releases`, `push-git-tags`).
   - Replace the hardcoded `NPM_TOKEN` env var with an optional `npm-token` secret; only inject it when provided. The workflow secret uses kebab-case (`npm-token`) following GitHub Actions conventions and is passed to the action as the `NPM_TOKEN` environment variable. `changesets/action@v2` skips `.npmrc` token injection when `NPM_TOKEN` is absent, enabling tag-only private-package releases. Before merging, search `mvng-nz/super-q` (and any other known consumer) `.github/workflows/*.yml` for `NPM_TOKEN` or `npm-token` to confirm callers do not reference it directly.
   - Keep `packages: write` in the job permissions. The reusable workflow's permissions define the effective permissions for its jobs and cannot elevate above the caller, but omitting `packages: write` would prevent npm/GitHub Packages publishing even when the caller grants it.
   - Keep `contents: write` and `pull-requests: write`.
   - Preserve `fetch-depth: 0` in the checkout step; Changesets requires full git history to determine version changes and create proper commits/tags.
   - Use the same `.nvmrc` resolution pattern from Step 1; allow `node-version` to override.

3. **Add `playwright-e2e.yml`** — `.github/workflows/playwright-e2e.yml`
   - Encapsulate the current inline pattern used in `super-q/.github/workflows/web-e2e.yml`: checkout, Node setup from `.nvmrc`, `corepack enable`, `yarn install --immutable` with `NODE_AUTH_TOKEN`, Playwright browser install (`npx playwright install --with-deps ${{ inputs.browser }}`), run the e2e command, optional artifact upload.
   - Inputs: `run-command` (required), `working-directory` (default `.`), `browser` (default `chromium`), `node-version`, `node-version-file`, `node-options`, `upload-artifacts` (boolean, default `false`), `artifact-name`, `artifact-path`, `retention-days`.
   - Secret: `packages-token` (required).
   - Let `super-q/.github/workflows/web-e2e.yml` become a clean ~20-line caller.

4. **Enhance `cloudflare-pages-deploy.yml`** — `.github/workflows/cloudflare-pages-deploy.yml`
   - Add a new optional `ref` input; implement as `ref: ${{ inputs.ref || github.ref }}` so an empty value (default or explicit) falls back to `github.ref`. This supports callers that want to deploy a fixed ref (e.g., always deploy from a release tag) regardless of the triggering event, while keeping the default behavior of checking out the triggering ref for existing callers.

5. **Update documentation** — `README.md`, `CHANGELOG.md`
   - Add a `playwright-e2e.yml` section with an input table and example caller.
   - Update the `changesets-release.yml` section for new inputs and private-package-only usage.
   - Add release notes under `[Unreleased]` for the next `v1.x` tag.
   - Update caller examples that are affected by new defaults.
   - Add `.nvmrc` guidance: "Consumers using `.nvmrc` should place it at the repository root with a single line containing the Node version (e.g., `24`). If `.nvmrc` is absent, workflows default to Node 24."

## Files to modify/create

- `.github/workflows/changesets-release.yml` — upgrade action version, expose inputs, optional npm token, `.nvmrc` support.
- `.github/workflows/playwright-e2e.yml` — new reusable E2E workflow.
- `.github/workflows/node-ci.yml` — add `node-version-file` and fallback logic.
- `.github/workflows/react-native-package-ci.yml` — add `node-version-file` and fallback logic.
- `.github/workflows/cloudflare-pages-deploy.yml` — add `node-version-file` and `ref`.
- `.github/workflows/storybook-test-deploy.yml` — add `node-version-file` and fallback logic.
- `README.md` — document new inputs, the new workflow, and caller examples.
- `CHANGELOG.md` — document changes under `[Unreleased]`.

## Verification

- Run `actionlint .github/workflows/*.yml` (or `lefthook run actionlint`) on every modified workflow.
- Before tagging, create a test caller workflow in a fork that mimics the current `changesets-release.yml` `@v1` usage (`build-command` only, no new inputs) and verify it still passes. This confirms backward compatibility.
- Test the `.nvmrc` fallback by running callers with `.nvmrc` present, with `.nvmrc` absent, and with `node-version` explicitly overridden.
- Test `changesets-release.yml` and `playwright-e2e.yml` against a downstream fork or with `act` before cutting the release tag.
- Test `changesets-release.yml` with and without the optional `github-token` input to confirm the default `${{ github.token }}` behavior and a custom PAT override both work.
- Test `cloudflare-pages-deploy.yml` with `ref` omitted, `ref` explicitly empty, and `ref` set to a specific branch/tag to verify the fallback to `github.ref`.
- Before moving the `v1` tag, test the actual `super-q` caller workflows (`release.yml`, `web-e2e.yml`, `deploy-web.yml`) against the new tag in a fork or test repo to catch integration issues early.
- After tagging, validate `super-q` callers:
  - `deploy-web.yml` still calls `cloudflare-pages-deploy.yml` with no regressions.
  - `release.yml` becomes a caller of the modernized `changesets-release.yml`.
  - `web-e2e.yml` becomes a caller of `playwright-e2e.yml`.
- Confirm `yarn changeset status`, `yarn turbo run lint build typecheck test`, and `yarn format-check` pass in `super-q` after the caller updates.
- Confirm the downstream Changesets config enables `privatePackages: { "version": true, "tag": true }` before relying on tag-only releases.

## Success criteria

- `super-q` no longer inlines the steps now covered by these reusable workflows.
- A private-only Changesets release produces per-package git tags (e.g. `web@x.y.z`, `mobile@x.y.z`) and GitHub releases without npm publishing.
- All modified workflows still pass `actionlint` and behave correctly for existing `@v1` callers.
- `.nvmrc` at the repo root drives Node version selection unless the caller explicitly overrides it.

## Rollback / reversibility

- Ship changes under a new minor/patch tag (e.g. `v1.5.0`) and move the `v1` tag only after validation.
- Downstream callers can be reverted to their previous inline definitions in a single commit.
- Removing or changing inputs later would be a breaking change; keep defaults backward-compatible within `v1`.

## Risks & considerations

- `changesets/action@v2` is currently a pre-release (`v2.0.0-next.3`) with no guaranteed stable release date. Pin to `changesets/action@v2.0.0-next.3` initially, and only move to a stable `v2` tag after it is released and tested. If a stable release is delayed or introduces breaking changes, roll back to `v1` and add custom git tag steps. It also requires Changesets v3 in downstream consumers.
- The reusable workflow's permissions define the effective permissions for its jobs and cannot elevate above the caller. `packages: write` is kept in `changesets-release.yml` to avoid breaking existing npm/GitHub Packages publishers; it is harmless for private-only consumers.
- Adding `node-version-file` to workflows that currently default to `node-version: '24'` must not change behavior for existing callers; the fallback chain must be implemented exactly as specified in Step 1.
- `cloudflare/wrangler-action` v3 and v4 have different input schemas; keeping the workflow pinned to `v4` avoids runtime surprises.
