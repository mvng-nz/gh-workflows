# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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
