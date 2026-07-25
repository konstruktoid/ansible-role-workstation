# Repository Instructions for GitHub Copilot

This repository is a single Ansible role (`workstation`) that installs and hardens a developer
workstation running Ubuntu Resolute (26.04), the only platform this role supports. Prefer
secure-by-default, operationally reliable, maintainable, and auditable changes.

## Mission and baseline

- Preserve the security-first intent of the role unless an explicit deviation is requested.
- Ubuntu Resolute (26.04) is the only supported platform. Do not add OS-conditional branches for other
  distributions or reintroduce multi-OS support without an explicit request.
- The role has two halves: `tasks/hardening.yml`, which includes the fifteen
  `konstruktoid.hardening.*` roles listed in `ansible-roles`, and `tasks/{packages,docker,uv,npm,tox,
  github_cli,copilot_vim}.yml`, which install the developer tooling captured in `install-log`. Keep
  that separation; do not fold hardening logic into the tool-installation tasks or vice versa.
- Every hardening role and every tool installation must remain independently toggleable through its
  own `workstation_harden_*` or `workstation_*_install` variable. Never collapse these into a single
  switch.
- Prefer minimal, reviewable, reversible diffs.

## Engineering expectations

- Use declarative, idempotent Ansible solutions.
- Prefer built-in and well-supported modules over ad hoc shell/command usage.
- Use clear task names, explicit conditions, and deterministic behavior.
- Favor maintainability over clever one-liners.
- Binary and archive downloads (currently `uv`) must keep checksum verification against the `shasums`
  values in `defaults/main.yml` — never fetch and install a release without verifying it. Prefer a
  vendor's signed apt repository (as used for Docker Engine and the GitHub CLI) over piping an install
  script through a shell.

## Security requirements

- Enforce least privilege; do not add `become: true` beyond what a specific operation requires. Most
  of this role's own tasks (`uv`, `tox`, npm global installs, the `copilot.vim` clone) are deliberately
  user-scoped and must stay that way. `become` is only acceptable for apt package/repository
  management, the Docker group-membership change, and the `konstruktoid.hardening.*` role includes
  (which manage their own `become` internally).
- Use restrictive ownership and permissions by default for any file this role manages directly (npm
  prefix configuration, downloaded archives, cloned repositories).
- Never hardcode secrets, credentials, keys, tokens, or passwords.
- Treat these as high-sensitivity areas: the fifteen `konstruktoid.hardening.*` role includes and their
  toggles, the Docker apt repository and GPG key setup, the `docker` group-membership task (equivalent
  to root access on the host), and the GitHub CLI apt repository and GPG key setup.
- Avoid security relaxations (disabling a hardening role by default, skipping checksum verification,
  broadening `become`) unless explicitly requested and documented.

## Preferred patterns

- Module-first authoring (`ansible.builtin.*`, `community.docker.*`, `community.general.*`,
  `community.crypto.*` per `requirements.yml`).
- Always use the fully-qualified collection name (FQCN) for every module, with no exceptions in this
  codebase.
- Explicit `owner`, `group`, and quoted octal string `mode` values (for example `mode: "0644"`) for
  managed files.
- Double-quoted YAML strings; use single quotes only when the value itself contains a double quote.
- Variable names prefixed `workstation_` in `defaults/main.yml`, matching existing convention.
- `ansible.builtin.deb822_repository` plus a GPG key fetched with `ansible.builtin.get_url` into
  `/etc/apt/keyrings` for any new apt-repository-based install, following the pattern already used for
  Docker Engine and the GitHub CLI.
- Keep `meta/main.yml` `galaxy_info.platforms` accurate; it should continue to list only Ubuntu
  Resolute.

## Discouraged patterns

- `shell`/`command` when an Ansible module exists.
- Non-idempotent logic without proper guards (`creates`, a version-check block, or an equivalent).
- curl-pipe-to-shell installers. Replace them with a checksum-verified archive download or a signed
  apt repository, as already done for `uv`, Docker Engine, and the GitHub CLI.
- Broad `become` scope, world-writable modes, or implicit behavior that reduces auditability.
- Skipping checksum verification on downloaded archives.

## Template and Jinja guidance

- This role currently has no `templates/` directory. If one is introduced, keep templates
  deterministic and explicit, never embed secrets, and avoid permissive fallback values unless
  explicitly required.

## Testing and validation

- Changes must pass `ansible-lint` at `profile: production` (see `.ansible-lint`); do not silence
  findings introduced by new work.
- Validate role changes with `tox -e docker` (installs `requirements.yml`, runs `ansible-lint`, then
  `molecule test -s docker` against a privileged Ubuntu Resolute container, including an idempotence
  check).
- The `molecule/default` scenario boots a QEMU/UEFI Ubuntu Resolute cloud image (no Vagrant/VirtualBox)
  and requires `qemu-system-x86_64`, `qemu-img`, `genisoimage`, and OVMF firmware on the host; it is
  not exercised by CI (`.github/workflows/molecule.yml` only runs the `docker` scenario) and depends on
  host virtualization support, so treat local failures there as environment-dependent rather than
  assuming role code is at fault.
- On some local/dev machines, `tox -e docker` / `molecule test -s docker` itself fails at the
  container-`create` step with a `community.docker`-driver/`runc` error, or the Docker socket is not
  accessible to the invoking user, unrelated to role content — a known local environment issue, not a
  regression from role changes. `ansible-lint` (production profile) remains the reliable fast local
  check in that case; let CI provide the authoritative pass/fail for the `docker` scenario.
- Prefer fixing lint/test failures over suppressing them; treat suppression as a last resort requiring
  justification.

## Documentation guidance

- Explain security rationale and operational impact for sensitive changes.
- Note any intentional deviation from the security-first, minimal-`become` intent of the role.
- Keep the README's "Role variables" table in sync with `defaults/main.yml`.

## Review expectations

Prioritize findings on:
- security misconfiguration,
- privilege escalation (`become` scope beyond what an operation requires),
- file ownership/mode,
- idempotency,
- unsafe module choice,
- missing checksum verification on downloaded archives,
- a curl-pipe-to-shell pattern reintroduced where a checksum-verified download or signed apt
  repository should be used instead.

For meaningful findings, provide: Finding, Risk (Critical/High/Medium/Low), Location, Recommendation,
and safer example as a short code snippet.

## Change safety rules

- Do not disable a hardening role by default or weaken its configuration without explicit instruction.
- Do not silently broaden `become` scope or grant `workstation_docker_group_membership` implications
  without noting the root-equivalence trade-off.
- Flag high-impact changes affecting the hardening role set, the Docker or GitHub CLI apt repository
  and GPG key setup, or checksum verification.
