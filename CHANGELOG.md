# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- `node-runner.yml` — canonical reusable Node CI workflow consolidating shared checkout, Node setup, Corepack, `yarn install --immutable`, command execution, and optional coverage upload.
- `playwright-e2e.yml` — reusable Playwright end-to-end test workflow with `.nvmrc` support, browser selection, and optional artifact upload.
- `node-version-file` input and `.nvmrc` resolution across `node-ci.yml`, `react-native-package-ci.yml`, `cloudflare-pages-deploy.yml`, `storybook-test-deploy.yml`, and `changesets-release.yml`. Node version selection now prefers `.nvmrc`, falls back to an explicit `node-version`, then defaults to `24`.
- `ref` input to `cloudflare-pages-deploy.yml` so callers can deploy a fixed ref; defaults to the triggering ref when omitted.
- Modernized `changesets-release.yml`:
  - Upgraded to `changesets/action@v2.0.0-next.3`.
  - Exposed `version-script`, `publish-script`, `create-github-releases`, `push-git-tags`, `setup-git-user`, `github-token`, `node-version`, `node-version-file`, and `node-options` inputs.
  - Replaced hardcoded `NPM_TOKEN` with an optional `npm-token` secret; omitting it enables tag-only private-package releases.

### Changed

- `node-ci.yml` and `react-native-package-ci.yml` are now legacy shims that delegate to `node-runner.yml`; preserved for backward compatibility under `@v1`.
- `node-ci.yml` and `react-native-package-ci.yml` default `node-version` input is now `''` so `.nvmrc` is respected when present.
- `storybook-test-deploy.yml` now accepts `node-version` and `node-version-file` inputs instead of hardcoding Node 24.
- Bumped `actions/setup-node` to `v7` and `actions/upload-artifact` to `v7` across all workflows.
- Removed `docs/plans` and `docs/brainstorms` from `.gitignore` so planning documents can be tracked in the repo.
- Updated `README.md` with new workflow sections, input tables, `.nvmrc` guidance, and caller examples.

## [1.4.0] - 2026-07-19

### Added

- `react-native-package-ci.yml` — reusable CI workflow for React Native packages using Node 24, Corepack, `yarn install --immutable`, and Yarn Turbo (`lint`, `typecheck`, `test`).
- `react-native-release.yml` — reusable release workflow that creates GitHub releases from tags and extracts release notes from `CHANGELOG.md`.
- `cloudflare-pages-deploy.yml` — reusable deployment workflow for Cloudflare Pages using `cloudflare/wrangler-action@v4`, with optional GitHub Deployments integration.

### Removed

- `flutter-package-ci.yml` — no longer maintained as MVNG moves from Flutter to React Native.
- `flutter-release.yml` — no longer maintained as MVNG moves from Flutter to React Native.

### Changed

- Updated `README.md` for the `v1.4.0` release: added React Native and Cloudflare Pages sections, refreshed the required-secrets table, added a migration note, and updated caller examples to `@v1` and `v1.4.0`.
- Updated `.gitignore` to ignore common editor directories and local documentation directories (`docs/plans`, `docs/brainstorms`).

### Migration

Flutter workflows were removed because they are no longer used within MVNG.

- If you still need the Flutter workflows, pin your caller to the last `v1.3.x` release that included them:
  ```yaml
  uses: mvng-nz/gh-workflows/.github/workflows/flutter-package-ci.yml@v1.3.1
  ```
- To use the new React Native workflows, replace Flutter callers with `react-native-package-ci.yml` and `react-native-release.yml`, and ensure your `turbo.json` defines `lint`, `typecheck`, and `test` tasks.
- When migrating deployments from Netlify to Cloudflare Pages, pass `branch: <production-branch>` and configure the Pages project production branch instead of using `--prod`.

[Unreleased]: https://github.com/mvng-nz/gh-workflows/compare/v1.4.0...HEAD
[1.4.0]: https://github.com/mvng-nz/gh-workflows/compare/v1.3.1...v1.4.0
