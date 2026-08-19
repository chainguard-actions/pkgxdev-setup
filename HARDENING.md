<!-- markdownlint-disable -->

# Hardening Report: pkgxdev--setup/v3.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **pkgxdev--setup/v3.0.0** was hardened automatically. 5 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All `uses:` references across workflow files use mutable tag refs (e.g. @v1, @v4, @v0) instead of immutable 40-character commit SHAs. This exposes the workflow to supply-chain attacks if any referenced action's tag is moved or compromised. Affected actions include: bobheadxi/deployments@v1, actions/checkout@v4, aws-actions/configure-aws-credentials@v4, actions/setup-node@v4, fischerscode/tagger@v0.

Locations:

- `.github/workflows/cd.installer.yml:16`
- `.github/workflows/cd.installer.yml:22`
- `.github/workflows/cd.installer.yml:23`
- `.github/workflows/cd.installer.yml:38`
- `.github/workflows/cd.vx.yml:17`
- `.github/workflows/cd.vx.yml:18`
- `.github/workflows/ci.action.yml:22`
- `.github/workflows/ci.action.yml:43`
- `.github/workflows/ci.action.yml:44`
- `.github/workflows/ci.installer.yml:24`
- `.github/workflows/ci.installer.yml:40`
- `.github/workflows/ci.installer.yml:41`

### script-injection (severity: high)

Sub-rule (a): GitHub Actions expression `${{ matrix.prefix }}` is interpolated directly inside a `run:` shell command string: `run: test -f '${{ matrix.prefix }}/deno.land/v*/bin/deno'`. The matrix value flows through YAML template substitution before the shell sees it, allowing an attacker who controls the matrix (e.g. via workflow_call or pull_request) to inject arbitrary shell commands.

Locations:

- `.github/workflows/ci.action.yml:47`

### script-injection (severity: high)

Sub-rule (a): GitHub Actions expression `${{ matrix.dir }}` is interpolated directly inside a `run:` shell command string: `run: sudo mv ${{ matrix.dir }} $(mktemp -d)`. The matrix value flows through YAML template substitution before the shell sees it, allowing an attacker who controls the matrix to inject arbitrary shell commands — made especially dangerous here because the command runs with `sudo`.

Locations:

- `.github/workflows/ci.installer.yml:87`

### missing-permissions (severity: medium)

Workflow file has no top-level `permissions:` key and no job-level `permissions:` key on any of its jobs (`qa-installer`, `qa-action`). Without explicit permissions the workflow inherits the repository's default token permissions, which may be overly broad (write-all on some repositories).

Locations:

- `.github/workflows/cd.qa.yml:1`

### missing-permissions (severity: medium)

Workflow file has no top-level `permissions:` key and no job-level `permissions:` key on any of its jobs (`std`, `plus-pkgs`, `multiple-apply-is-ok`, `dist`). Without explicit permissions the workflow inherits the repository's default token permissions, which may be overly broad.

Locations:

- `.github/workflows/ci.action.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, missing-permissions

**Notes:**

Fixed all 5 findings across 5 workflow files:

1. **unpinned-uses**: Pinned all action references to full commit SHAs with tag comments preserved: bobheadxi/deployments@v1→648679e8..., actions/checkout@v4→11d5960a..., aws-actions/configure-aws-credentials@v4→7474bc46..., actions/setup-node@v4→49933ea5..., fischerscode/tagger@v0→5ca3fa63...

2. **script-injection (ci.action.yml)**: Moved `${{ matrix.prefix }}` into an `env:` block as `PKGX_PREFIX` and referenced it as `"$PKGX_PREFIX"` in the shell command.

3. **script-injection (ci.installer.yml)**: Moved `${{ matrix.dir }}` into an `env:` block as `MATRIX_DIR` and referenced it as `"$MATRIX_DIR"` in the sudo mv command.

4. **missing-permissions (cd.qa.yml)**: Added `permissions: contents: read` top-level block.

5. **missing-permissions (ci.action.yml)**: Added `permissions: contents: read` top-level block.

### Iteration 2

**Fixes applied:** missing-permissions

**Notes:**

Added `permissions: {}` to the `qa` job in `.github/workflows/cd.installer.yml`. The job only calls a reusable CI workflow (`./.github/workflows/ci.installer.yml`) for testing and requires no special token permissions. The `put` job already had `permissions: { deployments: write }`. Both jobs now have explicit, minimal permissions blocks.

