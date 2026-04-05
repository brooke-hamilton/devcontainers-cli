# Implementation Plan: Lockfile Graduation

Implementation plan for [lockfile-graduation.md](lockfile-graduation.md).

## Source Files

| File | Role |
| ---- | ---- |
| `src/spec-configuration/lockfile.ts` | Core lockfile read/write/generate logic |
| `src/spec-configuration/containerFeaturesConfiguration.ts` | Feature processing pipeline; calls `readLockfile()`, `writeLockfile()`, `generateLockfile()` |
| `src/spec-node/devContainersSpecCLI.ts` | CLI command definitions (`up`, `build`); flag parsing and wiring |
| `src/spec-configuration/containerCollectionsOCI.ts` | OCI manifest fetching; uses lockfile digest for pinned resolution |
| `src/spec-configuration/containerFeaturesOCI.ts` | OCI feature manifest resolution; passes lockfile integrity to manifest fetch |

## Implementation Approach

This implementation follows a **test-first approach**. All tests are written and committed before any production code changes. The tests will initially fail (red), then the implementation makes them pass (green).

### Phase 1: Write tests

Update and add tests in `src/test/container-features/lockfile.test.ts`. All new tests target the new behavior and will fail against the current code.

#### 1a. Update existing tests to use new flags

- [ ] Replace `--experimental-frozen-lockfile` with `--frozen-lockfile` in the `frozen lockfile` and `outdated lockfile` tests.
- [ ] Remove `--experimental-lockfile` from the `write lockfile` and `lockfile with dependencies` tests (these should pass with no flag once auto-generation is implemented).

These tests will fail until the new flags are wired up.

#### 1b. Add auto-generation test

- [ ] Add test: `build` with no lockfile flags on a workspace that has no lockfile → verify a lockfile is created.

This test will fail because the current code requires `--experimental-lockfile` to create a lockfile.

#### 1c. Add opt-out tests

- [ ] Add test: `build --no-lockfile` on a workspace with no lockfile → verify no lockfile is created.
- [ ] Add test: `build --no-lockfile` on a workspace with an existing lockfile → verify the lockfile on disk is not modified and features resolve from the registry (not pinned to lockfile digests).

These tests will fail because `--no-lockfile` does not exist yet.

#### 1d. Add deprecation warning tests

- [ ] Add test: `build --experimental-lockfile` → verify build succeeds and stderr contains a deprecation warning.
- [ ] Add test: `build --experimental-frozen-lockfile` with a valid lockfile → verify build succeeds and stderr contains a deprecation warning.

These tests will fail because the current code does not emit deprecation warnings.

#### 1e. Verify existing tests still define expected behavior

- [ ] Confirm all existing tests (frozen, outdated, upgrade, integrity, empty file init, OCI integrity, tarball integrity) are unchanged and still passing at this point. They exercise behavior that should not change.

### Phase 2: Implement production changes

Make the failing tests pass, one step at a time.

#### 2a. Add new CLI flags (`devContainersSpecCLI.ts`)

- [ ] Add `--lockfile` (type: `boolean`, default: `true`, description: `Read and write the lockfile`) to both `build` and `up` command option definitions.
- [ ] Add `--frozen-lockfile` (type: `boolean`, default: `false`, description: `Require lockfile to exist and match exactly`) to both `build` and `up` command option definitions.
- [ ] Keep `--experimental-lockfile` and `--experimental-frozen-lockfile` as hidden aliases (`hidden: true`).
- [ ] In the handler functions (`provision()`, `doBuild()`), destructure both old and new flags. Map experimental flags to new flags and emit deprecation warnings when the experimental flags are used.

#### 2b. Update internal interfaces (`containerFeaturesConfiguration.ts`)

- [ ] Add `lockfile` (boolean) and `frozenLockfile` (boolean) to `ContainerFeatureInternalParams`.
- [ ] Keep `experimentalLockfile` and `experimentalFrozenLockfile` on the interface during the deprecation period.
- [ ] In the CLI handler, merge: if `experimentalLockfile` is `true`, set `lockfile = true` and emit warning. If `experimentalFrozenLockfile` is `true`, set `frozenLockfile = true` and emit warning.

#### 2c. Wire flags through params (`devContainersSpecCLI.ts`)

- [ ] In `provision()` and `doBuild()`, pass the new `lockfile` and `frozenLockfile` values into the options objects that flow to `createDockerParams()`.
- [ ] Ensure `ProvisionOptions` and the build options type include the new fields.

#### 2d. Update lockfile reading (`containerFeaturesConfiguration.ts`)

- [ ] In `generateFeaturesConfig()` (~line 488), gate the `readLockfile()` call on the `lockfile` flag:
  - If `lockfile` is `false` (`--no-lockfile`), skip `readLockfile()` entirely — pass `undefined` for `lockfile` to `processFeatureIdentifier()`, `computeDependsOnInstallationOrder()`, and `fetchFeatures()`.
  - If `lockfile` is `true` (default), read as today.

#### 2e. Update lockfile writing (`lockfile.ts`)

- [ ] In `writeLockfile()`, change the early-return guard (currently line 52):
  - **Current:** `if (!forceInitLockfile && !oldLockfileContent && !params.experimentalLockfile && !params.experimentalFrozenLockfile) { return; }`
  - **New:** `if (!params.lockfile) { return; }` — when `--no-lockfile`, skip writing entirely (regardless of `forceInitLockfile` or existing content).
  - When `params.lockfile` is `true` (default), always proceed to write logic — removing the guard that required the experimental flag for initial creation.
- [ ] Replace `params.experimentalFrozenLockfile` checks with `params.frozenLockfile`.

#### 2f. Deprecation warnings

- [ ] When `--experimental-lockfile` is passed, log to stderr: `Warning: --experimental-lockfile is deprecated. Lockfiles are now enabled by default.`
- [ ] When `--experimental-frozen-lockfile` is passed, log to stderr: `Warning: --experimental-frozen-lockfile is deprecated. Use --frozen-lockfile instead.`

### Phase 3: Documentation and changelog

- [ ] Update CLI help text descriptions for the new flags.
- [ ] Add `outdated` and `upgrade` to the README.md command list.
- [ ] Add a brief lockfile mention to README.md.
- [ ] Add CHANGELOG.md entry documenting the lockfile graduation, new flags, and deprecation of experimental flags.

## VS Code Extension Coordination

The VS Code Dev Containers extension (closed-source, `microsoft/vscode-remote-release` for issues) has a `dev.containers.experimentalLockfile` user setting that passes `--experimental-lockfile` to the CLI. This requires a separate coordinated change:

- **No blocker:** The CLI changes are backward compatible. The extension can continue passing `--experimental-lockfile` — it will trigger a deprecation warning but still work.
- **Recommended extension changes:**
  - Rename `dev.containers.experimentalLockfile` → `dev.containers.lockfile` (default: `true`).
  - Add `dev.containers.frozenLockfile` (default: `false`) for users who want enforcement.
  - Emit a VS Code deprecation notice if the old setting is detected.
- **File an issue** on `microsoft/vscode-remote-release` to request alignment after CLI changes ship.
