# Product Spec: Take the Lockfile Out of Preview

**Status:** Draft  
**Scope:** Dev Container CLI  
**Spec change required:** No — the [lockfile spec](https://github.com/devcontainers/spec/blob/main/docs/specs/devcontainer-lockfile.md) already exists and is not marked as experimental.

## Background

The `devcontainer-lock.json` file records the exact resolved version, OCI digest, and SHA-256 integrity checksum for every Feature in `devcontainer.json`. It provides reproducibility, integrity verification, and reviewable diffs when Features change.

The lockfile was introduced as a preview feature in the CLI ([PR #495](https://github.com/devcontainers/cli/pull/495)) behind `--experimental-lockfile` and `--experimental-frozen-lockfile` flags. It has been stable and in use — notably by the [devcontainers/images](https://github.com/devcontainers/images) repo — but the experimental flags create adoption friction.

### References

- [Lockfile spec](https://github.com/devcontainers/spec/blob/main/docs/specs/devcontainer-lockfile.md)
- [CLI implementation tracking issue](https://github.com/devcontainers/cli/issues/564)
- [Discussion #237](https://github.com/orgs/devcontainers/discussions/237) (community feedback)
- [Supply chain security proposal](https://gist.github.com/brooke-hamilton/05baf8f9ab7b1b2dd6a987072fb13f8a)

## Goals

1. Make the lockfile a first-class, non-experimental feature of the CLI.
2. Default to security: generate and update lockfiles automatically.
3. Preserve backward compatibility where possible; justify any breaking change with a security rationale.
4. Provide a clean migration path from the experimental flags.

## Current State

### Commands with lockfile flags

| Command | `--experimental-lockfile` | `--experimental-frozen-lockfile` |
|---------|:------------------------:|:-------------------------------:|
| `up`    | Yes | Yes |
| `build` | Yes | Yes |

Both flags are `boolean`, default `false`, and `hidden: true`.

### Commands that use the lockfile without flags

| Command | Behavior |
|---------|----------|
| `outdated` | Reads the existing lockfile to resolve current versions. No flags needed. |
| `upgrade` | Generates a new lockfile and writes it. Accepts `--dry-run`, `--feature`, `--target-version`. No lockfile flags. |

### Commands with no lockfile interaction

`exec`, `read-configuration`, `run-user-commands`, `set-up`, `features test`, `features package`, `features publish`, `features info`, `features resolve-dependencies`, `features generate-docs`, `templates apply`, `templates publish`, `templates generate-docs`.

### Current behavior: when is the lockfile read?

The lockfile is read at the start of feature processing in `build` and `up`, regardless of any flags:

1. **OCI features:** The lockfile's `integrity` digest is passed to manifest resolution. This causes the manifest to be fetched **by digest** instead of by tag — pinning to the exact artifact recorded in the lockfile. If the fetched manifest's computed digest doesn't match, the build fails with `"Digest did not match"`.
2. **Tarball features:** The lockfile's `integrity` digest is passed to the tarball downloader. After downloading, the SHA-256 of the tarball is computed and compared. If it doesn't match, the build fails with `"Digest did not match"`.

**This means: if a lockfile exists, it is always used to pin and verify Features — no flag is required for this.** The lockfile flags only control whether the lockfile is *written*, not whether it is *read*.

### Current behavior: when is the lockfile written?

The lockfile is written at the end of feature processing, after all Features have been resolved and fetched. The `writeLockfile()` function decides whether to write based on these conditions:

| Condition | Lockfile written? | Notes |
|-----------|:-:|-------|
| No lockfile exists, no flags | **No** | Early return — lockfile is not created. |
| No lockfile exists, `--experimental-lockfile` | **Yes** | Lockfile is created from the resolved Features. |
| No lockfile exists, `--experimental-frozen-lockfile` | **Error** | `"Lockfile does not exist."` |
| Populated lockfile exists, `devcontainer.json` unchanged | **No** | Features are resolved by lockfile digest, not by tag. The generated lockfile matches the existing file byte-for-byte, so no write occurs. **Upstream releases do not cause the lockfile to change.** |
| Populated lockfile exists, `devcontainer.json` changed, no flags | **Yes** | See below for what changes trigger this. |
| Populated lockfile exists, `devcontainer.json` changed, `--experimental-frozen-lockfile` | **Error** | `"Lockfile does not match."` |
| Empty lockfile exists (touch workaround), no flags | **Yes** | Empty file triggers `initLockfile` → force write. |

**Why upstream releases don't change the lockfile:** When a lockfile entry exists for a feature (e.g., `featureABC:1` with `integrity: "sha256:olddigest"`), the resolution code in `fetchOCIManifestIfExists()` fetches the manifest **by digest** (`/manifests/sha256:olddigest`), not by tag (`/manifests/1`). The `:1` tag is never consulted. The registry returns the exact pinned artifact, the generated lockfile contains the same digest, and `writeLockfile()` detects the content is identical and skips writing. A new version published under the `:1` tag is invisible until the developer explicitly runs `devcontainer upgrade`.

**What changes in `devcontainer.json` cause the lockfile to be updated:**
- A feature was added — the new feature has no lockfile entry, so it resolves by tag and gets a new lockfile entry.
- A feature was removed — the lockfile is regenerated without the removed feature's entry.
- A feature's version constraint was changed (e.g., `:1` → `:2`) — the lockfile entry key changes, so it resolves fresh by the new tag.
- A feature's `dependsOn` graph changed (new transitive dependency).

**What does NOT cause the lockfile to be updated:**
- Upstream releases of features already pinned in the lockfile.
- Registry tag changes (e.g., `:1` now points to a newer artifact).
- Changes to a feature's install script at the same version (the digest would differ, but the old digest is still used for resolution).

### Key observation

When a lockfile already exists, `build` and `up` already read it for pinning/integrity and update it if features change — no flag required. The only gap is **initial creation** — today you must pass `--experimental-lockfile` or use the `touch` workaround to bootstrap the lockfile.

## Proposed Changes

### 1. Auto-generate lockfile by default (`build`, `up`)

**Change:** When `build` or `up` processes Features and no lockfile exists, create one automatically.

**Rationale:** This is the single most impactful change. It makes lockfiles the default for all users, aligning with the npm/yarn/cargo pattern where `install` always produces a lockfile. Since the lockfile is a new file written alongside `devcontainer.json`, this does not change the container build output or break any existing workflow — it only adds a file.

**Detail:**
- In `writeLockfile()`, remove the early-return condition that skips writing when no lockfile exists and no experimental flag is set.
- The lockfile is written after feature resolution succeeds, so there is no impact on build failure behavior.

### 2. Add `--lockfile` and `--frozen-lockfile` flags (`build`, `up`)

**Change:** Add non-hidden `--lockfile` and `--frozen-lockfile` boolean flags to `build` and `up`.

| Flag | Default | Behavior |
|------|---------|----------|
| `--lockfile` | `true` | Write/update the lockfile after feature resolution. |
| `--frozen-lockfile` | `false` | Require lockfile to exist and match exactly; fail otherwise. Implies `--lockfile`. |

**Detail:**
- `--lockfile` defaults to `true`, reflecting the new default behavior from change #1. Users can pass `--no-lockfile` to suppress lockfile creation/update.
- `--frozen-lockfile` replaces `--experimental-frozen-lockfile` with identical semantics.

### 3. Deprecate experimental flags (`build`, `up`)

**Change:** Keep `--experimental-lockfile` and `--experimental-frozen-lockfile` as hidden aliases. Emit a deprecation warning when they are used.

**Rationale:** This avoids breaking existing CI/CD pipelines immediately while guiding users to the new flags.

**Detail:**
- When `--experimental-lockfile` is passed, map it to `--lockfile` and log: `Warning: --experimental-lockfile is deprecated. Lockfiles are now enabled by default.`
- When `--experimental-frozen-lockfile` is passed, map it to `--frozen-lockfile` and log: `Warning: --experimental-frozen-lockfile is deprecated. Use --frozen-lockfile instead.`
- Plan to remove the experimental flags in a future major version. Announce the timeline in the changelog.

### 4. No changes to `outdated` or `upgrade`

These commands already operate without experimental flags. No changes needed.

| Command | Status |
|---------|--------|
| `outdated` | No change. Reads existing lockfile. Works today. |
| `upgrade` | No change. Writes updated lockfile. Works today. |

### 5. No changes to other commands

Commands like `exec`, `read-configuration`, `features test`, etc. do not process Features at build time and have no lockfile interaction. No changes needed.

## Behavior Summary (After Changes)

**Read behavior (unchanged — lockfile is always read when present):**

| Scenario | Read behavior |
|----------|--------------|
| No lockfile exists | Features resolve from registry by tag/version as normal. |
| Populated lockfile exists | Features resolve using lockfile digests. Integrity verified. Build fails if digest doesn't match (`"Digest did not match"`). |

Reading is independent of all flags. If a lockfile exists, it is used.

**Write behavior (after changes):**

| Scenario | Write behavior |
|----------|---------------|
| No lockfile, default flags | **New:** Lockfile is created after successful feature resolution. |
| No lockfile, `--no-lockfile` | No lockfile created (opt out). |
| Populated lockfile, `devcontainer.json` unchanged | No write (lockfile digests pin resolution; generated content is byte-identical). |
| Populated lockfile, `devcontainer.json` changed, default flags | Lockfile overwritten with new content (same as today). |
| Populated lockfile, `devcontainer.json` changed, `--no-lockfile` | Lockfile is **not** updated. |
| Populated lockfile, `devcontainer.json` changed, `--frozen-lockfile` | **Error:** `"Lockfile does not match."` (same as today's `--experimental-frozen-lockfile`). |
| No lockfile, `--frozen-lockfile` | **Error:** `"Lockfile does not exist."` |
| Empty lockfile (touch workaround) | Lockfile initialized and populated (same as today). |
| `--experimental-lockfile` passed | Deprecation warning; behaves like default (`--lockfile`). |
| `--experimental-frozen-lockfile` passed | Deprecation warning; behaves like `--frozen-lockfile`. |

**What causes lockfile content to differ:** Only changes to `devcontainer.json` — features added/removed, version constraints changed, or `dependsOn` graph changed. Upstream feature releases do NOT cause the lockfile to differ because the lockfile digests pin resolution (see "Current behavior: when is the lockfile read?" above). To pick up new upstream versions, use `devcontainer upgrade`.

## Breaking Changes

| Change | Impact | Justification |
|--------|--------|---------------|
| Lockfile auto-generated on `build`/`up` | A new `devcontainer-lock.json` file appears in the workspace. Users may see new untracked files in git. | Security by default. The lockfile provides integrity verification for all Feature artifacts. Users who don't want it can pass `--no-lockfile`. This mirrors the behavior of every major package manager. |
| `--experimental-lockfile` triggers deprecation warning | CI logs include a new warning line. | Standard deprecation practice. No functional change. |

These are low-risk changes. No existing container build will fail. No existing lockfile will be invalidated.

## Migration Guide

| Current usage | Migration |
|---------------|-----------|
| `devcontainer build --experimental-lockfile` | `devcontainer build` (lockfile is now default) |
| `devcontainer up --experimental-lockfile` | `devcontainer up` (lockfile is now default) |
| `devcontainer build --experimental-frozen-lockfile` | `devcontainer build --frozen-lockfile` |
| `devcontainer up --experimental-frozen-lockfile` | `devcontainer up --frozen-lockfile` |
| `touch .devcontainer-lock.json` (Codespaces workaround) | No change needed; still works. But the lockfile is now created automatically, so the touch workaround is no longer necessary. |
| No lockfile usage | Lockfile is created automatically. Commit it to source control for reproducibility. Pass `--no-lockfile` to opt out. |

## Out of Scope

- **`devcontainer.json` properties for lockfile configuration** — Adding `lockfile` / `locked` properties to `devcontainer.json` (as suggested in [Discussion #237](https://github.com/orgs/devcontainers/discussions/237#discussioncomment-16265065)) would require a spec change. This is a good idea for a future iteration, but out of scope for this phase. The goal here is to remove the experimental flags from the CLI without a spec change.
- **VS Code settings** — The VS Code `dev.containers.experimentalLockfile` setting is owned by the VS Code Dev Containers extension. Coordination needed, but not part of this CLI spec.
- **Signature verification, provenance, SBOMs** — These are later phases of the supply chain security roadmap.

## Test Plan

Update existing tests in `src/test/container-features/lockfile.test.ts`:

1. **Update flag references** — Replace `--experimental-lockfile` with default behavior (remove the flag from tests that only need lockfile generation). Replace `--experimental-frozen-lockfile` with `--frozen-lockfile`.
2. **Auto-generation test** — Add a test that runs `build` with no lockfile flags and verifies a lockfile is created.
3. **Opt-out test** — Add a test that runs `build --no-lockfile` and verifies no lockfile is created.
4. **Deprecation warning tests** — Verify that `--experimental-lockfile` and `--experimental-frozen-lockfile` still work but emit deprecation warnings.
5. **Existing behavior preserved** — All existing lockfile tests (frozen, outdated, upgrade, integrity, empty file init) continue to pass.

## Implementation Checklist

- [ ] Modify `writeLockfile()` in `src/spec-configuration/lockfile.ts` — remove the guard that skips writing when no lockfile exists and no experimental flag is set.
- [ ] Add `--lockfile` (default: `true`) and `--frozen-lockfile` (default: `false`) to `build` and `up` command option definitions in `src/spec-node/devContainersSpecCLI.ts`.
- [ ] Add `--no-lockfile` support (yargs handles `--no-` prefix for boolean flags automatically).
- [ ] Wire the new flags through `createDockerParams` / `ProvisionOptions` to `writeLockfile()`.
- [ ] Keep `--experimental-lockfile` and `--experimental-frozen-lockfile` as hidden aliases; emit deprecation warnings when used.
- [ ] Update `ContainerFeatureInternalParams` interface to use the new property names.
- [ ] Update tests.
- [ ] Update CHANGELOG.md.
