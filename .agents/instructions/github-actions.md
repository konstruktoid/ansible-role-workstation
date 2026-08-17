# GitHub Actions Security Instructions

**Applies to:** `.github/workflows/**/*.{yml,yaml}`

Apply these rules to workflow definitions (`lint.yml`, `molecule.yml`, `issues.yml`, `scorecards.yml`,
`dependency-review.yml`, `slsa.yml`). They extend the repository-wide instructions in
[`AGENTS.md`](../../AGENTS.md).

## Security-first workflow design

- Use least-privilege `permissions`, denying by default. Every workflow declares `permissions: {}` at
  the top level and grants scopes per job, so one compromised job cannot reach another job's token
  scope. Never grant a scope at workflow level. Add only the scopes a job's steps actually call, with
  a trailing comment naming what needs each one, as `issues.yml` does for `issues: write` and
  `slsa.yml`'s provenance and release jobs do for `id-token: write` and `contents: write`. No scope
  implies another: a job needing a write scope must still list `contents: read` if it checks out.
- Keep a job holding a write scope separate from any job that builds or tests untrusted code, so the
  write token is never present while that code runs. Gate privileged jobs on the ref that actually
  needs them — `slsa.yml`'s provenance job runs only for `refs/tags/`, so branch pushes never mint an
  OIDC token.
- Keep triggers narrow (`branches`, `paths`, event types) and avoid unnecessary execution scope.
- Pin third-party actions to a commit SHA (existing pattern: `uses: owner/action@<sha> # vX.Y.Z`).
  Never introduce an unpinned `@main`/`@vX` reference for a third-party action. Every third-party
  action in this repo is currently SHA-pinned and kept current by the `github-actions` Dependabot
  ecosystem in `.github/dependabot.yml`; the reusable `slsa-github-generator` workflow is the sole
  tag-referenced exception, since a reusable workflow call must resolve a ref the generator supports.
  Pin to the latest published release rather than to whatever the file already used, looking the
  version up at the time of the change. Dependabot applies a deliberate 7-day cooldown, so a release
  published within the last week is not yet a pin this repo adopts.
- Prefer a tool preinstalled on the runner over a third-party action when it does the same job,
  especially inside a job holding a write scope: `slsa.yml` uploads its release asset with
  `gh release upload` rather than an action, removing one dependency from the `contents: write` path.
- Treat all PR metadata, issue content, artifact contents, and external inputs as untrusted. Do not
  interpolate `${{ github.* }}` expressions directly into a `run:` script. Read the equivalent
  `GITHUB_*` environment variable instead, as the `Resolve the repository name` steps in `slsa.yml` do.
- Keep `step-security/harden-runner` as the first step of every job, as done throughout this repo.
- Set `persist-credentials: false` on `actions/checkout` unless a later step genuinely needs to push
  with the job's credentials; no workflow in this repo does.

## Secrets and token handling
- Never hardcode secrets or tokens.
- Use GitHub secrets/variables and mask sensitive output.
- Avoid printing env/context objects that may contain credentials.
- Minimize `GITHUB_TOKEN` privileges and step exposure.

## High-risk event and runner handling
- Use extreme caution with `pull_request_target` and `workflow_run`; never execute untrusted code with
  elevated context. This repo currently uses only `pull_request`, `push`, `schedule`, and
  `workflow_dispatch`. Do not switch to `pull_request_target` without a clear, documented need.
- Validate artifact provenance before reuse; avoid cross-trust artifact promotion. Be especially
  careful in `slsa.yml`, which produces the release provenance consumers rely on.
- Treat caches as potentially attacker-influenced; scope keys defensively.

## Reliability and safety defaults
- Set explicit `timeout-minutes` on every job; all current jobs declare one, so a new job without it
  is a regression rather than an omission.
- Use `concurrency` to prevent unsafe overlap. `lint.yml`, `molecule.yml` and `dependency-review.yml`
  group on `${{ github.workflow }}-${{ github.ref }}` with `cancel-in-progress: true`; `slsa.yml` uses
  the same group with `cancel-in-progress: false`, so a release-provenance run is never cancelled
  midway.
- Keep scripts short, fail-fast, and explicit (`set -euo pipefail` for bash steps where appropriate).
- Do not assume a directory exists because the standard Ansible role layout defines it. This role has
  no `handlers/` or `templates/` directory, which is what previously broke the `slsa.yml` build step.
  Guard filesystem traversal with an existence check.
- Avoid curl-pipe-to-shell patterns; download, verify, then execute. Mirror the checksum-verification
  discipline used for the `uv` release archive in `tasks/uv.yml` and the `shasums` values in
  `defaults/main.yml`, or the signed apt-repository pattern used for Docker Engine.
  In workflows, prefer a SHA-pinned setup action over an upstream install script: `molecule.yml`
  installs `uv` through `astral-sh/setup-uv` rather than piping `astral.sh/uv/install.sh` into a shell.

## Review priorities
1. Privilege model (`permissions`, token use, OIDC/release scope in `slsa.yml`)
2. Trigger safety and untrusted input handling
3. Third-party dependency trust (pinning, provenance)
4. Secret exposure risks in logs, outputs, artifacts, and caches
5. Runner isolation and escalation paths

## Risk levels
- Critical: direct secret exfiltration path or untrusted-code execution in privileged context
- High: broad token permissions, unsafe event usage, or unpinned high-trust action usage
- Medium: missing safety controls (`timeout-minutes`, `concurrency`, validation gaps)
- Low: style/maintainability issues with limited security impact
