# Plan: Consolidate node-ci.yml and react-native-package-ci.yml

## Purpose

Reduce duplication between `node-ci.yml` and `react-native-package-ci.yml`, which are identical except for their default `run-command`. Introduce a single `node-runner.yml` reusable workflow and keep the existing workflows as thin backward-compatible shims until a future major release can remove them.

## Scope

**In scope:**
- Create a new `.github/workflows/node-runner.yml` that consolidates the shared CI behavior.
- Refactor `.github/workflows/node-ci.yml` to be a thin caller/shim of `node-runner.yml`.
- Refactor `.github/workflows/react-native-package-ci.yml` to be a thin caller/shim of `node-runner.yml`.
- Update `README.md` to document `node-runner.yml` and mark `node-ci.yml`/`react-native-package-ci.yml` as legacy shims.
- Update `CHANGELOG.md` under `[Unreleased]`.

**Out of scope:**
- Functional changes to the CI steps themselves (e.g. adding new test runners, changing coverage paths, or altering Node/Corepack/Yarn behavior).
- Removing `node-ci.yml` or `react-native-package-ci.yml` until a future major version.
- Changes to `cloudflare-pages-deploy.yml`, `storybook-test-deploy.yml`, `changesets-release.yml`, `package-publish.yml`, or `react-native-release.yml`.

## Assumptions

- The two workflows are intentionally kept separate only because their default `run-command` differs. [verified: source of both files]
- Consumers reference `node-ci.yml` or `react-native-package-ci.yml` by `@v1` and expect stable behavior. [verified: `README.md` examples]
- The shared steps (checkout, Node setup, Corepack, `yarn install --immutable`, artifact upload) are stable and unlikely to diverge soon. [verified: source of both files]
- `.nvmrc`/`node-version` handling will be in place before this consolidation, because `node-runner.yml` should inherit the same pattern. [dependency: main reusability plan]

## Dependencies

- This plan must be executed only after the main reusability plan (`2607210053-improve-mvng-gh-workflows-reusability.md`) has completed and its `node-version-file` logic has been merged. If it is not complete, defer this plan until it is.
- Pre-flight: Read `.github/workflows/node-ci.yml` and confirm it contains a `node-version-file` input and the expected `Resolve Node version` step. If the dependency plan's changes are not actually present in the working tree, halt and complete that plan first.

## Implementation steps

1. **Create `node-runner.yml`** — `.github/workflows/node-runner.yml`
   - Define `workflow_call` inputs in this order: `node-version`, `node-version-file`, `node-options`, `run-command` (required, no default), `upload-coverage`. GitHub Actions displays inputs in definition order, so required inputs are listed first for clarity.
   - Secret: `packages-token` (required).
   - Implement the shared steps: checkout, Node setup, Corepack enable, `yarn install --immutable`, run the supplied command, optionally upload coverage.
   - Permissions: `contents: read`, `packages: read`.
   - Copy the `node-version-file` resolution logic from the modified `node-ci.yml` after the dependency plan completes; do not reimplement it from scratch.

2. **Refactor `node-ci.yml` to a shim** — `.github/workflows/node-ci.yml`
   - Keep the same `workflow_call` inputs and defaults, preserving the original input order for backward compatibility: `node-version`, `run-command`, `upload-coverage`, `node-options`, plus `node-version-file` if it was added by the dependency plan. Pass all inputs through to `node-runner.yml`.
   - Replace the job steps with a single `uses: ./.github/workflows/node-runner.yml` call, passing through all inputs and explicitly passing `packages-token: ${{ secrets.packages-token }}`.

3. **Refactor `react-native-package-ci.yml` to a shim** — `.github/workflows/react-native-package-ci.yml`
   - Keep the same `workflow_call` inputs and defaults, preserving the original input order for backward compatibility: `node-version`, `node-options`, `run-command`, `upload-coverage`, plus `node-version-file` if it was added by the dependency plan. Pass all inputs through to `node-runner.yml`.
   - Replace the job steps with a single `uses: ./.github/workflows/node-runner.yml` call, passing through all inputs and explicitly passing `packages-token: ${{ secrets.packages-token }}`.

4. **Update documentation** — `README.md`, `CHANGELOG.md`
   - Document `node-runner.yml` as the canonical public CI workflow in the README: add an input table matching the format of existing workflow sections, and an example caller showing the required `run-command` input.
   - Update the `node-ci.yml` and `react-native-package-ci.yml` sections to note they are legacy shims that delegate to `node-runner.yml` and are preserved for backward compatibility. If `node-version-file` was added by the dependency plan, include it in the shim input tables as well.
   - Add release notes under `[Unreleased]`.

## Files to modify/create

- `.github/workflows/node-runner.yml` — new reusable CI workflow containing the shared implementation.
- `.github/workflows/node-ci.yml` — refactor into a backward-compatible shim calling `node-runner.yml`.
- `.github/workflows/react-native-package-ci.yml` — refactor into a backward-compatible shim calling `node-runner.yml`.
- `README.md` — document `node-runner.yml` and the shim status of the existing workflows.
- `CHANGELOG.md` — add release notes.

## Verification

- Run `actionlint .github/workflows/*.yml` (or `lefthook run actionlint`) on the modified workflows.
- Test the shims in `mvng-nz/super-q` (or another known consumer) to confirm existing callers still work unchanged. Note: `act` has limited support for reusable workflows calling reusable workflows, so it may not be a reliable verification method.
- Verify that both workflows still upload coverage artifacts and respect `upload-coverage: false`.
- Confirm that `node-version`, `node-version-file`, and `node-options` pass through correctly.
- Test the `node-version-file` resolution logic in `node-runner.yml` by running it with `.nvmrc` present, with `.nvmrc` absent, and with explicit `node-version` override to confirm the copied logic works correctly.
- Confirm that `node-runner.yml` inputs display in the specified order in the GitHub Actions UI by checking the workflow file after creation.

## Success criteria

- There is one source of truth for the shared CI implementation (`node-runner.yml`).
- Existing callers of `node-ci.yml` and `react-native-package-ci.yml` continue to work without changes.
- The README clearly distinguishes the canonical workflow from the legacy shims.

## Rollback / reversibility

- Because this is a pure refactor into shims, callers can revert by pinning to the previous `v1.x` tag.
- If a shim regresses, the corresponding legacy workflow can be temporarily restored by reverting the shim file to its previous inline implementation.
- Removing the legacy shims is a breaking change and should only happen in a future major release (e.g. `v2`).
- This backward-compatible refactor can be released under the existing `@v1` moving tag without requiring a new minor version, since callers using the legacy shims experience no breaking changes.

## Risks & considerations

- Reusable workflows calling reusable workflows (`uses: ./.github/workflows/node-runner.yml`) add one layer of indirection; ensure input names and secret passthrough match exactly. GitHub allows up to 4 levels of nesting; `node-ci` → `node-runner` is one level and safe.
- A single `node-runner.yml` with no default `run-command` is more flexible but makes the workflow less self-describing; documentation is important.
- If future CI steps diverge (e.g. React Native needs platform-specific setup), the shared abstraction may need to be split again. This plan assumes the current shared steps remain stable.
