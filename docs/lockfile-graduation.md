# Product Spec: Lockfile Graduation

## Summary

The net change in this spec is to make lockfile generation the default behavior for `build` and `up`, similar to how `npm install` automatically generates a `package-lock.json`. Today, lockfile creation requires passing `--experimental-lockfile` or using a `touch` workaround. After this change, `devcontainer-lock.json` is created automatically on the first build and kept up to date as `devcontainer.json` changes.

This proposal defines a behavior change, which some could consider to be a breaking change: builds that previously produced no lockfile will now produce an additional `devcontainer-lock.json` file. In practice, this is low-impact — no existing builds will fail, no container output changes, and the only visible effect is a new file in the workspace. Users who don't want it can opt out with `--no-lockfile`. The experimental flags are preserved as hidden, deprecated aliases so existing CI pipelines continue to work. The commands `devcontainer outdated` and `devcontainer upgrade` (which already read/write lockfiles) will be unaffected by this change.

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
| --------- | :------------------------: | :-------------------------------: |
| `up` | Yes | Yes |
| `build` | Yes | Yes |

Both flags are `boolean`, default `false`, and `hidden: true`.

### Commands that use the lockfile without flags

| Command | Behavior |
| --------- | ---------- |
| `outdated` | Reads the existing lockfile to resolve current versions. No flags needed. |
| `upgrade` | Generates a new lockfile and writes it. Accepts `--dry-run`, `--feature`, `--target-version`. No lockfile flags. |

### Current behavior: when is the lockfile read?

The lockfile is read at the start of feature processing in `build` and `up`, regardless of any flags:

1. **OCI features:** The lockfile's `integrity` digest is passed to manifest resolution. This causes the manifest to be fetched **by digest** instead of by tag — pinning to the exact artifact recorded in the lockfile. If the fetched manifest's computed digest doesn't match, the build fails with `"Digest did not match"`.
2. **Tarball features:** The lockfile's `integrity` digest is passed to the tarball downloader. After downloading, the SHA-256 of the tarball is computed and compared. If it doesn't match, the build fails with `"Digest did not match"`.

**This means: if a lockfile exists, it is always used to pin and verify Features — no flag is required for this.** The lockfile flags only control whether the lockfile is *written*, not whether it is *read*.

### Current behavior: when is the lockfile written?

The lockfile is written at the end of feature processing, after all Features have been resolved and fetched. The `writeLockfile()` function decides whether to write based on these conditions:

| Condition | Lockfile written? | Notes |
| ----------- | :-: | ------- |
| No lockfile exists, no flags | **No** | Early return — lockfile is not created. |
| No lockfile exists, `--experimental-lockfile` | **Yes** | Lockfile is created from the resolved Features. |
| No lockfile exists, `--experimental-frozen-lockfile` | **Error** | `"Lockfile does not exist."` |
| Populated lockfile exists, `devcontainer.json` unchanged | **No** | Features are resolved by lockfile digest, not by tag. The generated lockfile matches the existing file byte-for-byte, so no write occurs. **Upstream feature releases do not cause the lockfile to change.** |
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
| ------ | --------- | ---------- |
| `--lockfile` | `true` | Read and write the lockfile. When `true` (default), the lockfile is read for pinning/integrity during feature resolution, and written/updated afterward. |
| `--no-lockfile` | — | Skip both reading and writing the lockfile. Features resolve from the registry by tag as if no lockfile exists. The lockfile on disk (if any) is neither consulted nor modified. Matches `npm install --no-package-lock` / `pnpm install --no-lockfile` semantics. |
| `--frozen-lockfile` | `false` | Require lockfile to exist and match exactly; fail otherwise. Implies `--lockfile`. |

**Detail:**

- `--lockfile` defaults to `true`, reflecting the new default behavior from change #1. Users can pass `--no-lockfile` to opt out entirely.
- `--no-lockfile` skips both reading and writing. This is the escape hatch for users who don't want lockfile behavior at all. It matches npm/pnpm convention where `--no-lockfile` means "pretend the lockfile doesn't exist."
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
| --------- | -------- |
| `outdated` | No change. Reads existing lockfile. Works today. |
| `upgrade` | No change. Writes updated lockfile. Works today. |

### 5. No changes to other commands

Commands like `exec`, `read-configuration`, `features test`, etc. do not process Features at build time and have no lockfile interaction. No changes needed.

## Behavior Summary (After Changes)

**Read behavior:**

| Scenario | Read behavior |
| ---------- | -------------- |
| No lockfile exists | Features resolve from registry by tag/version as normal. |
| Populated lockfile exists, default flags | Features resolve using lockfile digests. Integrity verified. Build fails if digest doesn't match (`"Digest did not match"`). |
| Populated lockfile exists, `--no-lockfile` | Lockfile is **ignored**. Features resolve from registry by tag/version as if no lockfile exists. |

Reading is independent of all flags **except** `--no-lockfile`, which skips reading entirely.

**Write behavior (after changes):**

| Scenario | Write behavior |
| ---------- | --------------- |
| No lockfile, default flags | **New:** Lockfile is created after successful feature resolution. |
| Any state, `--no-lockfile` | No lockfile created or updated. Existing lockfile on disk is untouched. |
| Populated lockfile, `devcontainer.json` unchanged | No write (lockfile digests pin resolution; generated content is byte-identical). |
| Populated lockfile, `devcontainer.json` changed, default flags | Lockfile overwritten with new content (same as today). |
| Populated lockfile, `devcontainer.json` changed, `--frozen-lockfile` | **Error:** `"Lockfile does not match."` (same as today's `--experimental-frozen-lockfile`). |
| No lockfile, `--frozen-lockfile` | **Error:** `"Lockfile does not exist."` |
| Empty lockfile (touch workaround) | Lockfile initialized and populated (same as today). |
| `--experimental-lockfile` passed | Deprecation warning; behaves like default (`--lockfile`). |
| `--experimental-frozen-lockfile` passed | Deprecation warning; behaves like `--frozen-lockfile`. |

**What causes lockfile content to differ:** Only changes to `devcontainer.json` — features added/removed, version constraints changed, or `dependsOn` graph changed. Upstream feature releases do NOT cause the lockfile to differ because the lockfile digests pin resolution (see "Current behavior: when is the lockfile read?" above). To pick up new upstream versions, use `devcontainer upgrade`.

## Breaking Changes

| Change | Impact | Justification |
| -------- | -------- | --------------- |
| Lockfile auto-generated on `build`/`up` | A new `devcontainer-lock.json` file appears in the workspace. Users may see new untracked files in git. | Security by default. The lockfile provides integrity verification for all Feature artifacts. Users who don't want it can pass `--no-lockfile`. This mirrors the behavior of every major package manager. |
| `--experimental-lockfile` triggers deprecation warning | CI logs include a new warning line. | Standard deprecation practice. No functional change. |

These are low-risk changes. No existing container build will fail. No existing lockfile will be invalidated. However, any CI/CD processes that enforce a lack of changes during build (e.g., by checking for a clean git state) may need to be updated to allow the new lockfile or the deprecation warning.

## Migration Guide

| Current usage | Migration |
| --------------- | ----------- |
| `devcontainer build --experimental-lockfile` | `devcontainer build` (lockfile is now default) |
| `devcontainer up --experimental-lockfile` | `devcontainer up` (lockfile is now default) |
| `devcontainer build --experimental-frozen-lockfile` | `devcontainer build --frozen-lockfile` |
| `devcontainer up --experimental-frozen-lockfile` | `devcontainer up --frozen-lockfile` |
| `touch .devcontainer-lock.json` (Codespaces workaround) | No change needed; still works. But the lockfile is now created automatically, so the touch workaround is no longer necessary. |
| No lockfile usage | Lockfile is created automatically. Commit it to source control for reproducibility. Pass `--no-lockfile` to opt out. |

## Documentation

The following user-facing documentation is in scope for this change:

- **CLI help text** — The `--lockfile`, `--no-lockfile`, and `--frozen-lockfile` flags on `build` and `up` need visible, non-hidden descriptions in the yargs command definitions. The deprecated `--experimental-lockfile` and `--experimental-frozen-lockfile` flags remain hidden.
- **README.md** — The command list in the README does not currently include `outdated` or `upgrade`. These commands are already shipped and should be added to the command list. The README should also briefly mention that lockfiles are generated by default on `build` and `up`.
- **CHANGELOG.md** — Add an entry documenting the lockfile graduation, new flags, and deprecation of experimental flags. Follow the existing format (month/year header, version number, bulleted descriptions with PR links).

## Out of Scope

- **`devcontainer.json` properties for lockfile configuration** — Adding `lockfile` / `locked` properties to `devcontainer.json` (as suggested in [Discussion #237](https://github.com/orgs/devcontainers/discussions/237#discussioncomment-16265065)) would require a spec change. This is a good idea for a future iteration, but out of scope for this phase. The goal here is to remove the experimental flags from the CLI without a spec change.
- **VS Code settings** — The VS Code `dev.containers.experimentalLockfile` setting is owned by the VS Code Dev Containers extension (closed-source; `microsoft/vscode-remote-release` for issues). Coordination needed, but not a blocker — the CLI changes are backward compatible, and the extension can continue passing `--experimental-lockfile` until it is updated separately.
- **Signature verification, provenance, SBOMs** — These are later phases of the supply chain security roadmap.

## Implementation

See [lockfile-graduation-implementation.md](lockfile-graduation-implementation.md) for the implementation plan, test plan, and VS Code extension coordination details.
