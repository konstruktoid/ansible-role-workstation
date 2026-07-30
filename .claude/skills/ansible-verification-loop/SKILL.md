---
name: ansible-verification-loop
description: This skill should be used when reviewing, writing, or modifying anything under this repository's `workstation` Ansible role, including `tasks/`, `defaults/`, `vars/`, `meta/`, `handlers/`, `templates/`, `requirements.yml`, `README.md`'s "Role variables" section, and any `molecule/` scenario. Always consult it before editing role YAML or test fixtures, even for a single-line change, since it defines both the repo's security/quality conventions (FQCN, quoting, `become` scope, checksum pinning) and the bounded lint/test/idempotence verification loop that every change to the role must pass before it is reported done.
---

# ansible-verification-loop

## Purpose
Provide a structured approach for reviewing and modifying the `workstation` role in this repository.
It ensures changes are made consistently, verified thoroughly through a bounded fix/re-verify loop, and
documented clearly.

## When to use this
- When reviewing or modifying `tasks/`, `defaults/`, `vars/`, `meta/`, `requirements.yml`, or the
  `molecule/` scenarios.
- When you need to ensure that changes are made consistently and verified thoroughly.

## When NOT to use this
- When making changes that do not involve this role's Ansible content (e.g. only README prose,
  `.github/` workflow tweaks, or repo metadata unrelated to the role's behavior).

## Before changing anything
1. Read `defaults/main.yml`, `tasks/main.yml`, and `meta/main.yml`, plus `requirements.yml` for
   collection dependencies. `tasks/main.yml` imports the other `tasks/*.yml` files in order. Trace
   which ones apply to the change:
   - `hardening.yml` includes the fifteen `konstruktoid.hardening.*` roles, each gated by its own
     `workstation_harden_*` toggle.
   - `packages.yml`, `docker.yml`, `uv.yml`, `npm.yml`, `tox.yml`, `github_cli.yml`, and
     `copilot_vim.yml` install the developer tooling, each gated by its own `workstation_*_install`
     toggle.
   These two files are the authoritative inventory of what the role does; there is no separate
   manifest of hardening roles or tools to cross-check against.
2. This role targets Ubuntu Resolute (26.04) exclusively. Do not add OS-conditional branches for
   other distributions or reintroduce multi-OS support without an explicit request.

## While making the change
3. Follow `.github/copilot-instructions.md` and `.github/instructions/*.instructions.md`, the
   authoritative security/quality rules for this repo (FQCN only, double-quoted strings, quoted octal
   `mode` with explicit `owner`/`group`, `workstation_`-prefixed variable names, checksum verification
   on downloaded binaries such as `uv`). Treat the fifteen hardening role toggles, the Docker apt
   repository and group-membership tasks, and any GPG-key/apt-repository setup as high-sensitivity.
4. Minimize `become`. Most of this role's own tasks (uv, tox, npm global installs, the Git clone of
   copilot.vim) are deliberately user-scoped and must stay that way. `become: true` is only acceptable
   where the underlying operation genuinely requires root: apt package/repository management, the
   `docker` group membership change, and the `konstruktoid.hardening.*` role includes (which manage
   their own `become` internally). Do not add `become: true` to a task, block, or role default beyond
   what the specific operation requires.
5. Read the [YAML 1.2.2 specification](https://yaml.org/spec/1.2.2/) before writing or reviewing YAML
   content; it is the authoritative reference for scalar resolution, quoting, and syntax that
   `ansible-lint`/`yamllint` don't fully enforce. In particular, watch for:
   - Ambiguous plain scalars that the core schema would resolve as boolean/null instead of a string
     (`y`/`n`/`yes`/`no`/`on`/`off`/`null`/`~`). Quote them if a string is intended.
   - Numeric-looking plain scalars that could be misread as int/float/octal/sexagesimal (this repo
     already quotes octal `mode` values, e.g. `mode: "0755"`, and version strings such as
     `workstation_uv_release`; keep doing that for any new scalar that looks numeric but must stay a
     string).
   - Tabs used for indentation (YAML block structure requires spaces).
   - Anchors/aliases (`&`/`*`) or explicit tags (`!!`). This repo currently uses neither; avoid
     introducing them unless there is a clear benefit, since they reduce readability of task files.
   Do not change quoting/formatting purely for spec-purity if it would fight `ansible-lint`'s
   `production` profile or the repo's `.ansible-lint` rules; those take precedence on any conflict.
6. Follow the existing conventions and patterns in the codebase: naming, file structure, and style.
7. If a `defaults/main.yml` variable is added, renamed, retyped, or removed, update both the matching
   option in `meta/argument_specs.yml` and the "Role variables" table in `README.md` to match. The
   argument specification is validated at run time, so a default that has drifted from its spec fails
   the play instead of merely being out of date.
8. If a pinned tool release (`workstation_uv_release`, or a similar version pin added later) changes,
   update the matching `shasums` entry in `defaults/main.yml` alongside the version/URL variables.
   Verify new checksums against the upstream project's published checksums before committing them.
9. Add or update test coverage for the change:
   - `molecule/default/converge.yml` and `molecule/docker/molecule.yml` (which reuses
     `../default/prepare.yml`, `../default/converge.yml`, and `../default/verify.yml`) both exercise
     this single role; there is no per-scenario role split. The two scenarios differ only in how the
     target is provisioned: `molecule/default/create.yml` boots a QEMU virtual machine, while
     `molecule/docker/create.yml` starts a privileged container from `molecule_yml.platforms`.
   - Extend `molecule/default/verify.yml` with assertions for new/changed behavior (installed
     packages, file ownership/mode, group membership, service state), following the existing
     `ansible.builtin.assert` pattern. Because verify is a separate `ansible-playbook` run, it cannot
     see `defaults/main.yml` or the `konstruktoid.hardening.*` defaults and instead mirrors the values
     it needs in its own `vars:` block. Update that block whenever a value it mirrors changes.
   - Assertions that depend on a running init system (`systemd_service` state, timers) must stay gated
     on `ansible_facts.virtualization_type not in ["container", "docker", "podman"]`, since the
     `docker` scenario's container runs `sleep 1d` as PID 1 rather than systemd.
   - If new scenario-specific variables are needed, add them under the relevant host in
     `molecule/default/inventory/host_vars/resolute/main.yml` (and
     `molecule/docker/inventory/host_vars/resolute/main.yml` if the `docker` scenario needs different
     values).

## Verification loop
Never declare a change done based on the edit alone. The change is only done once it has cleared the
loop below. "Done" means every box in the Verification checklist is checked, not that the edit compiles
or looks right.

**Order checks cheapest-first, and stop at the first failure.** Each retry of the full `molecule test`
cycle costs minutes (it boots/upgrades a container); `ansible-lint` costs seconds. Re-running an
expensive check against code that a cheap check would have already rejected wastes the loop's attempt
budget on noise instead of on real fixes:
1. `ansible-lint` (see the command below): fast, catches most convention violations.
2. `molecule converge -s docker` / `molecule verify -s docker`: fast iteration once lint is clean; use
   this while actively fixing converge/verify failures instead of the full `test` cycle.
3. A full `molecule test -s docker` (or `tox -e docker`): the authoritative gate. Only run this once
   converge/verify are passing, since it re-does the create/converge/idempotence/verify/destroy cycle
   from scratch.

**Bound the loop to 3 full attempts.** An attempt is: make a fix, then re-run the checks above from
step 1. If a checklist item still fails after 3 attempts, stop. Do not keep guessing, silently drop
the requirement, or ship with a known-failing gate. Report to the user instead (see below). Track what
changed between attempts, in scratch notes if needed, so the final report can state what was tried and
why it did not resolve the failure, rather than only that the check still fails.

**Treat different gate failures as different problems.** A lint failure, an idempotence failure (second
converge reports changes), and a `verify.yml` assertion failure point at unrelated root causes. Fix
the specific one that failed rather than re-touching unrelated code each retry.

### Commands
- Run `ansible-lint` and confirm a clean exit / expected output (`profile: production`, see
  `.ansible-lint`). This is the primary quality gate; do not add suppressions to silence findings
  from new changes.
- Run `tox -e docker` and confirm exit code 0. This installs role dependencies (`requirements.yml`),
  runs `ansible-lint`, then invokes `molecule test -s docker` to converge and verify the role in a
  privileged Ubuntu Resolute container, including an idempotence check.
- Running `molecule test -s docker` directly skips the dependency install and lint steps that
  `tox -e docker` performs first. Run `ansible-galaxy install --force -r requirements.yml` and
  `ansible-lint` yourself beforehand if you use this form instead of `tox`.
- The `molecule/default` scenario (driven by `tox -e devel`/`upstream`) boots a QEMU/UEFI Ubuntu
  Resolute cloud image (no Vagrant/VirtualBox) and requires `qemu-system-x86_64`, `qemu-img`,
  `genisoimage`, and OVMF firmware on the host. It is not run in CI (`.github/workflows/molecule.yml`
  only runs `tox -e docker` and `tox -e docker-upstream`), so local `create`/`prepare`/`converge`
  failures there may reflect host virtualization support rather than role content. Verify host
  tooling before assuming a regression.
- The `package_management` hardening role sets `system_upgrade: true` by default, which runs a full
  `apt` upgrade on every converge. Expect the first converge in a scenario to take noticeably longer
  than a typical role; this is expected behavior, not a hang. Do not count it as a stalled attempt.
- On some local/dev machines, `tox -e docker` / `molecule test -s docker` itself fails at the
  container-`create` step with a `community.docker`-driver/`runc` error unrelated to role content.
  This is a known pre-existing local environment issue. If that happens, `ansible-lint` (production
  profile) is the reliable fast local check, and CI provides the authoritative pass/fail for the
  `docker` scenario. Say so explicitly in the report rather than counting it as an unresolved attempt
  against the 3-try budget.

### If the loop exhausts its 3 attempts
Report the issue instead of proceeding or silently giving up:
- Which checklist item(s) remain unresolved.
- What was tried across the attempts and why each did not resolve it.
- Detailed reproduction steps and the relevant lint/molecule output or logs.

## Verification checklist
- [ ] Lint passes: `ansible-lint`
- [ ] Tests pass: `tox -e docker` / `molecule test -s docker`
- [ ] Idempotence holds (no changes reported on molecule's second converge)
- [ ] `molecule/default/verify.yml` updated if the role's observable behavior changed, including its
      `vars:` block if a value it mirrors from `defaults/main.yml` changed
- [ ] `molecule/*/inventory/host_vars/resolute/main.yml` updated if new scenario-specific variables are
      needed
- [ ] `meta/argument_specs.yml` matches `defaults/main.yml` (same option names, types, and defaults)
- [ ] `README.md` "Role variables" table matches `defaults/main.yml`
- [ ] `shasums` in `defaults/main.yml` updated if a pinned tool release changed, with checksums
      verified against upstream
- [ ] `become` is not used more broadly than the specific task requires
- [ ] No unrelated files changed
- [ ] New/changed YAML scalars are unambiguous per the
      [YAML 1.2.2 spec](https://yaml.org/spec/1.2.2/) (no unquoted `y`/`n`/`on`/`off`/`null`-like
      strings, no unquoted numeric-looking strings that must stay strings, no tabs)
