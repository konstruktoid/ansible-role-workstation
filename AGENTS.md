# Repository Instructions for Coding Agents

This repository is a single Ansible role (`workstation`) that installs and hardens a developer
workstation running Ubuntu Resolute (26.04), the only platform this role supports. Prefer
secure-by-default, operationally reliable, maintainable, and auditable changes.

These instructions are tool-neutral and apply to any coding agent or assistant working in this
repository, as well as to human contributors. Nothing here is specific to one vendor's product; if
an agent needs its own entry point, point that entry point at this file rather than copying it, so
there is a single source of truth to keep current.

## Further instructions

- [`.agents/instructions/ansible-role.md`](.agents/instructions/ansible-role.md): rules for the
  role's own YAML, applying to `{tasks,defaults,meta,molecule}/**/*.{yml,yaml}`.
- [`.agents/instructions/github-actions.md`](.agents/instructions/github-actions.md): rules for
  workflow definitions, applying to `.github/workflows/**`.
- [`.agents/skills/ansible-verification-loop/SKILL.md`](.agents/skills/ansible-verification-loop/SKILL.md):
  the bounded lint/test/idempotence loop every change to the role has to clear before it is reported
  as done, and the repository conventions it checks against.

## Vendor entry points

This file and everything under `.agents/` are the canonical, tool-neutral instructions. The paths below
exist only because specific tools look for them; each one points at a file above and holds no rules of
its own. Change the canonical file, never the entry point, and if a tool needs a new entry point, add a
pointer rather than a copy.

| Entry point | Points at |
| --- | --- |
| `CLAUDE.md` | `AGENTS.md` |
| `.github/copilot-instructions.md` | `AGENTS.md` |
| `.github/instructions/ansible-role.instructions.md` | `.agents/instructions/ansible-role.md` |
| `.github/instructions/github-actions.instructions.md` | `.agents/instructions/github-actions.md` |
| `.claude/skills/ansible-verification-loop` | `.agents/skills/ansible-verification-loop` (symlink) |

The two `.github/instructions/*.instructions.md` files keep their `applyTo` front matter, because that
is what scopes them to a path; the rules themselves stay in `.agents/instructions/`.

## Mission and baseline

- Preserve the security-first intent of the role unless an explicit deviation is requested.
- Ubuntu Resolute (26.04) is the only supported platform. Do not add OS-conditional branches for other
  distributions or reintroduce multi-OS support without an explicit request.
- The role has three parts: `tasks/hardening.yml`, which includes the fifteen
  `konstruktoid.hardening.*` roles;
  `tasks/{packages,docker,uv,npm,tox,opencode,github_cli,copilot_vim}.yml`, which install the developer
  tooling; and `tasks/verify.yml`, which asserts that the command-line clients the role installed are
  present, executable, owned by the connecting user, and reporting the expected version. Keep that
  separation; do not fold hardening logic into the tool-installation tasks or vice versa, and do not
  move installation work into `verify.yml`. `tasks/hardening.yml` and `tasks/main.yml` are the
  authoritative lists of what the role applies; there is no separate manifest file.
- Every hardening role and every tool installation must remain independently toggleable through its
  own `workstation_harden_*` or `workstation_*_install` variable. Never collapse these into a single
  switch. `tasks/packages.yml` is the one exception: it has no boolean toggle and installs whatever
  `workstation_packages` lists, so it is disabled by emptying that list or by skipping the `packages`
  tag.
- `playbook.yml` sets `become_exe: sudo.ws` deliberately. Ubuntu Resolute points `/usr/bin/sudo` at
  `sudo-rs`, whose password prompt ansible-core does not recognize, so `become` tasks hang and fail
  with "Timed out waiting for become success"; `sudo.ws` is the classic sudo binary the `sudo` package
  registers with `update-alternatives`. It is not a typo for `sudo`. Do not "correct" it, and keep it
  in the README's playbook example. The upstream fix (ansible/ansible#86175, for issue
  ansible/ansible#85837) is merged in `devel` but not backported, so it is absent from every
  ansible-core release this role supports; the setting can be dropped once the minimum supported
  version includes it.
- Prefer minimal, reviewable, reversible diffs.

## Engineering expectations

- Use declarative, idempotent Ansible solutions.
- Prefer built-in and well-supported modules over ad hoc shell/command usage.
- Use clear task names, explicit conditions, and deterministic behavior.
- Favor maintainability over clever one-liners.
- Binary and archive downloads (currently `uv`, opencode, and the GitHub CLI) must keep checksum
  verification against the `shasums` values in `defaults/main.yml`. Never fetch and install a release
  without verifying it. Prefer a checksum-verified release archive, or a vendor's signed apt repository
  (as used for Docker Engine), over piping an install script through a shell.
- Extract release archives with `--no-same-owner`. They record the uid and gid of the upstream build
  account, which `tar` restores when the play connects as root, leaving a binary owned by an account
  that does not exist on the workstation.

## Security requirements

- Enforce least privilege; do not add `become: true` beyond what a specific operation requires. Most
  of this role's own tasks (`uv`, `tox`, npm global installs, the opencode and GitHub CLI archive
  installs, the `copilot.vim` clone) are deliberately user-scoped and must stay that way. `become` is
  only acceptable for apt package/repository management, the Docker group-membership change, the
  removal of the system-wide GitHub CLI install in `tasks/github_cli.yml`, and the
  `konstruktoid.hardening.*` role includes (which manage their own `become` internally).
- Command-line clients (the Claude Code CLI, opencode, the GitHub CLI) are installed under the
  connecting user's home directory, never system-wide. Do not convert one back to an apt package or a
  root-owned prefix, and keep the ownership assertions in `tasks/verify.yml` intact.
- Use restrictive ownership and permissions by default for any file this role manages directly (npm
  prefix configuration, downloaded archives, cloned repositories).
- Never hardcode secrets, credentials, keys, tokens, or passwords.
- Treat these as high-sensitivity areas: the fifteen `konstruktoid.hardening.*` role includes and their
  toggles, the Docker apt repository and GPG key setup, the `docker` group-membership task (equivalent
  to root access on the host), the checksum-verified release archive downloads in `tasks/uv.yml`,
  `tasks/opencode.yml`, and `tasks/github_cli.yml`, and the removal of the system-wide GitHub CLI
  install in `tasks/github_cli.yml`.
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
  Docker Engine. A command-line client a single account uses is installed into that account's
  `~/.local/bin` from a checksum-verified release archive instead, as done for `uv`, opencode, and the
  GitHub CLI, so it needs no `become`.
- Keep `meta/main.yml` `galaxy_info.platforms` accurate; it should continue to list only Ubuntu
  Resolute.

## Discouraged patterns

- `shell`/`command` when an Ansible module exists.
- Non-idempotent logic without proper guards (`creates`, a version-check block, or an equivalent).
- curl-pipe-to-shell installers. Replace them with a checksum-verified archive download or a signed
  apt repository, as already done for `uv`, opencode, the GitHub CLI, and Docker Engine.
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
  accessible to the invoking user. Both are known local environment issues unrelated to role content,
  not regressions from role changes. `ansible-lint` (production profile) remains the reliable fast
  local check in that case; let CI provide the authoritative pass/fail for the `docker` scenario.
- Prefer fixing lint/test failures over suppressing them; treat suppression as a last resort requiring
  justification.

## Documentation guidance

- Explain security rationale and operational impact for sensitive changes.
- Note any intentional deviation from the security-first, minimal-`become` intent of the role.
- Every role variable is declared in three places that must be changed together: `defaults/main.yml`,
  the matching option in `meta/argument_specs.yml` (type and default), and the README's "Role
  variables" table. A variable added or renamed in only one of them is a defect: `argument_specs.yml`
  is validated at run time, so a stale entry there fails the play rather than merely drifting.
- `molecule/default/verify.yml` re-declares several role and `konstruktoid.hardening.*` defaults in its
  own `vars:` block, because verify runs as a separate `ansible-playbook` invocation that cannot see
  them. Changing a value it mirrors (notably `workstation_uv_release`,
  `workstation_opencode_release`, `workstation_gh_release`, `workstation_packages`, and
  `workstation_docker_packages`) requires updating that block in the same change.

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
- Flag high-impact changes affecting the hardening role set, the Docker apt repository and GPG key
  setup, checksum verification, or whether a command-line client is installed per user or
  system-wide.
