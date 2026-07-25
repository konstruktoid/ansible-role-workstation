---
applyTo: "{tasks,defaults,meta}/**/*.{yml,yaml}"
---

# Workstation Role Instructions

Apply these rules to Ansible YAML changes in this role.

## Authoring priorities

- Module-first implementation; avoid `shell`/`command` unless no safe module exists (the version-check
  commands in `tasks/uv.yml` and `tasks/tox.yml` are acceptable precedent for querying an installed
  tool's version or invoking a `uv`/`command`-only workflow, not a license to add more).
- Always use the fully-qualified collection name (FQCN, for example `ansible.builtin.apt`,
  `ansible.builtin.deb822_repository`, `community.general.npm`) for every module call.
- Declarative, idempotent tasks with descriptive names.
- Explicit `owner`, `group`, and restrictive quoted octal string `mode` values (for example
  `mode: "0644"`) for managed files.
- Double-quoted YAML strings; variables prefixed `workstation_` (for example
  `workstation_docker_packages`, `workstation_npm_global_prefix`), matching `defaults/main.yml`.
- Minimal `become`: only apt package/repository management, the `docker` group-membership task, and
  the `konstruktoid.hardening.*` role includes may use it. Every other task in this role's own
  `tasks/*.yml` files runs as the connecting user.
- Every hardening role include in `tasks/hardening.yml` and every tool-install task file imported from
  `tasks/main.yml` must stay behind its own boolean toggle in `defaults/main.yml`, so a single role or
  tool can be disabled without affecting the others.
- This role targets Ubuntu Resolute (26.04) exclusively; do not add `ansible_facts.os_family` /
  `ansible_facts.distribution` branches for other platforms.
- New or changed pinned tool releases (currently `workstation_uv_release`) must update the matching
  entry under `shasums:` in `defaults/main.yml` and keep checksum verification in the download task.

## Conservative security handling

Treat these domains as high sensitivity and change conservatively:
- The fifteen `konstruktoid.hardening.*` role includes in `tasks/hardening.yml` and their
  `workstation_harden_*` toggles.
- The Docker Engine and GitHub CLI apt repository and GPG key setup (`tasks/docker.yml`,
  `tasks/github_cli.yml`), including the keyring paths under `/etc/apt/keyrings`.
- The `docker` group-membership task in `tasks/docker.yml`, since group membership is equivalent to
  root access on the host.
- Checksum verification logic and pinned release versions in `defaults/main.yml` and `tasks/uv.yml`.
- The npm global-prefix configuration in `tasks/npm.yml`, which exists specifically to avoid requiring
  `become` for global package installs; do not reintroduce a root-owned global prefix.

## Compliance-aware behavior

- Favor patterns aligned with the upstream security guidance of each installed tool (Docker's official
  apt-repository install method, the GitHub CLI's official apt-repository install method) over
  ad hoc or curl-pipe-to-shell alternatives.
- Improve auditability, traceability, and enforcement consistency.
- Reference exact upstream documentation or CVEs only when verified in repository context; otherwise
  cite likely risk areas and rationale.

## Review priorities

1. Security regression or weakening of a hardening role's effect
2. Over-privileged execution (`become` used beyond what a task requires)
3. Non-idempotent logic or risky shell pipelines
4. Missing explicit ownership/mode on managed files, or missing checksum verification on downloaded
   archives
5. Operational reliability issues (missing `creates`/version guard, brittle conditions)

## Risk levels

- Critical: clear security bypass, credential exposure, or severe privilege expansion (for example
  disabling checksum verification, or granting `become: true` where none is required)
- High: a hardening role disabled by default, or a curl-pipe-to-shell pattern reintroduced
- Medium: moderate hardening gap or reliability risk
- Low: maintainability/readability issue with minimal security impact
