<!-- markdownlint-disable -->

# Hardening Report: pkgxdev--setup/v3.2.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **pkgxdev--setup/v3.2.0** was hardened automatically. 3 finding(s) were identified and resolved across 4 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference Actions using mutable tags instead of pinned full-length SHA commit hashes, making them vulnerable to supply-chain attacks if the tag is moved.

Failing references:
- ci.action.yml: `actions/checkout@v4`, `actions/setup-node@v4`
- ci.installer.yml: `actions/checkout@v4`, `reviewdog/action-shellcheck@v1`
- cd.deployment.yml: `pkgxdev/setup@latest`
- cd.installer.yml: `bobheadxi/deployments@v1`, `actions/checkout@v4`, `aws-actions/configure-aws-credentials@v4`
- cd.vx.yml: `actions/checkout@v4`, `fischerscode/tagger@v0`

Locations:

- `.github/workflows/ci.action.yml:21`
- `.github/workflows/ci.action.yml:34`
- `.github/workflows/ci.installer.yml:20`
- `.github/workflows/ci.installer.yml:25`
- `.github/workflows/cd.deployment.yml:12`
- `.github/workflows/cd.installer.yml:19`
- `.github/workflows/cd.installer.yml:25`
- `.github/workflows/cd.installer.yml:27`
- `.github/workflows/cd.vx.yml:18`
- `.github/workflows/cd.vx.yml:19`

### script-injection (severity: high)

Direct GitHub Actions expression interpolation inside `run:` shell command strings allows script injection.

(a) `.github/workflows/cd.vx.yml` — the `run:` block at the `vtagger` job step interpolates `${{ secrets.GITHUB_TOKEN }}`, `${{ github.repository }}`, and `${{ github.event.release.tag_name }}` directly into shell commands. Even though `secrets.GITHUB_TOKEN` is GitHub-controlled, all three expressions are substituted into the shell string before the shell parses it, which is the script-injection pattern.

Offending lines:
  `curl -s -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" ...`
  `https://api.github.com/repos/${{ github.repository }}/releases/latest`
  `if [ "${latest_release}" = "${{ github.event.release.tag_name }}" ]; then`

(b) `.github/workflows/ci.action.yml` — the `plus-pkgs` job interpolates `${{ matrix.prefix }}` directly into a `run:` shell command:
  `run: test -f '${{ matrix.prefix }}/curl.se/v*/bin/curl'`

Locations:

- `.github/workflows/cd.vx.yml:24`
- `.github/workflows/cd.vx.yml:26`
- `.github/workflows/cd.vx.yml:29`
- `.github/workflows/ci.action.yml:55`

### missing-permissions (severity: medium)

Several workflow files have no top-level `permissions:` key and at least one job also lacks a job-level `permissions:` key, meaning the workflow runs with the default (potentially broad) token permissions.

- `ci.action.yml`: No top-level `permissions:` block and none of the four jobs (`std`, `plus-pkgs`, `multiple-apply-is-ok`, `dist`) define job-level permissions.
- `cd.deployment.yml`: No top-level `permissions:` block and the single job `test-deployment` has no job-level permissions.
- `cd.qa.yml`: No top-level `permissions:` block and neither reusable-workflow-call job defines permissions.
- `cd.installer.yml`: No top-level `permissions:` block; the `qa` job (which calls a reusable workflow) has no job-level permissions (only the `put` job has `permissions: deployments: write`).

Locations:

- `.github/workflows/ci.action.yml:1`
- `.github/workflows/cd.deployment.yml:1`
- `.github/workflows/cd.qa.yml:1`
- `.github/workflows/cd.installer.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, missing-permissions

**Notes:**

Fixed all three finding categories across five workflow files:

1. unpinned-uses: Pinned all 10 mutable action references to full SHA hashes with tag comments preserved: actions/checkout@v4, actions/setup-node@v4, reviewdog/action-shellcheck@v1, pkgxdev/setup@latest, bobheadxi/deployments@v1 (×2), aws-actions/configure-aws-credentials@v4, fischerscode/tagger@v0.

2. script-injection: (a) cd.vx.yml — moved secrets.GITHUB_TOKEN, github.repository, and github.event.release.tag_name into the step env block as GITHUB_TOKEN, GITHUB_REPOSITORY, RELEASE_TAG_NAME; shell script now uses plain env var references. (b) ci.action.yml — moved matrix.prefix into MATRIX_PREFIX env var; changed single-quoted glob path to double-quoted $MATRIX_PREFIX reference.

3. missing-permissions: Added top-level `permissions: contents: read` to ci.action.yml, cd.deployment.yml, cd.qa.yml, and cd.installer.yml. The cd.installer.yml put job retains its existing `deployments: write` job-level permission.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed script injection in .github/workflows/ci.installer.yml at the no-dirs job. Moved `${{ matrix.dir }}` out of the `run:` shell string into an `env:` block as `DIR: ${{ matrix.dir }}`, and updated the shell command from `sudo mv ${{ matrix.dir }} $(mktemp -d)` to `sudo mv "$DIR" $(mktemp -d)`. The value is now passed through an environment variable and properly double-quoted, preventing shell injection.

### Iteration 3

**Fixes applied:** script-injection

**Notes:**

Fixed script injection in hardened/action/.github/workflows/cd.installer.yml at line 52. Moved `${{ secrets.AWS_CF_DISTRIBUTION_ID }}` out of the `run:` block into an `env:` block as `DIST_ID`, and updated the cloudfront create-invalidation command to reference it as `"$DIST_ID"` instead of the raw expression.

### Iteration 4

**Fixes applied:** suspicious-run-content

**Notes:**

Replaced all `eval "$(...)"` patterns in .github/workflows/ci.installer.yml across the eval-bash, eval-zsh, and eval-posix-sh jobs:

1. `eval "$(cat ./installer.sh)"` → `. ./installer.sh` (POSIX dot/source command, works in bash, zsh, and sh)
2. `eval "$(pkgx +duf)"` in eval-bash and eval-zsh → `source <(pkgx +duf)` (process substitution, supported by bash and zsh)
3. `eval "$(pkgx +duf)"` in eval-posix-sh → write output to a temp file and dot-source it (`_pkgx_env=$(mktemp); pkgx +duf > "$_pkgx_env"; . "$_pkgx_env"; rm -f "$_pkgx_env"`) since POSIX sh does not support process substitution

All 9 flagged locations have been remediated. No eval with command substitution patterns remain in the file.

