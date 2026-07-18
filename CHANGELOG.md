# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [2.0.0] - 2026-07-19

### Added

- `react-native-package-ci.yml` — reusable CI workflow for React Native packages using Node 24, Corepack, `yarn install --immutable`, and Yarn Turbo (`lint`, `typecheck`, `test`).
- `react-native-release.yml` — reusable release workflow that creates GitHub releases from tags and extracts release notes from `CHANGELOG.md`.
- `cloudflare-pages-deploy.yml` — reusable deployment workflow for Cloudflare Pages using `cloudflare/wrangler-action@v4`, with optional GitHub Deployments integration.

### Removed

- `flutter-package-ci.yml` — no longer maintained as MVNG moves from Flutter to React Native.
- `flutter-release.yml` — no longer maintained as MVNG moves from Flutter to React Native.

### Changed

- Updated `README.md` for the `v2` major version: added React Native and Cloudflare Pages sections, refreshed the required-secrets table, added a migration note, and updated all caller examples to `@v2`.
- Updated `.gitignore` to ignore common editor directories and local documentation directories (`docs/plans`, `docs/brainstorms`).

### Migration

This release is a breaking change for consumers of the Flutter workflows.

- To keep using the Flutter workflows, pin callers to the `v1` major tag:
  ```yaml
  uses: mvng-nz/gh-workflows/.github/workflows/flutter-package-ci.yml@v1
  ```
- To migrate to the new major version, replace Flutter callers with `react-native-package-ci.yml` and `react-native-release.yml`, and ensure your `turbo.json` defines `lint`, `typecheck`, and `test` tasks.
- When migrating deployments from Netlify to Cloudflare Pages, pass `branch: <production-branch>` and configure the Pages project production branch instead of using `--prod`.

[Unreleased]: https://github.com/mvng-nz/gh-workflows/compare/v2.0.0...HEAD
[2.0.0]: https://github.com/mvng-nz/gh-workflows/compare/v1.3.1...v2.0.0
