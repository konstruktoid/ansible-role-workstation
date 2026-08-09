# workstation

[![Ansible Lint](https://github.com/konstruktoid/ansible-role-workstation/actions/workflows/lint.yml/badge.svg)](https://github.com/konstruktoid/ansible-role-workstation/actions/workflows/lint.yml)
[![Molecule testing workflow](https://github.com/konstruktoid/ansible-role-workstation/actions/workflows/molecule.yml/badge.svg)](https://github.com/konstruktoid/ansible-role-workstation/actions/workflows/molecule.yml)
[![OpenSSF Scorecard](https://api.securityscorecards.dev/projects/github.com/konstruktoid/ansible-role-workstation/badge)](https://scorecard.dev/viewer/?uri=github.com/konstruktoid/ansible-role-workstation)

An [Ansible](https://www.ansible.com/) role that installs and hardens a developer workstation running
Ubuntu Resolute (26.04). The role is the only supported target platform; it does not attempt to support
other Ubuntu releases or distributions.

See [OVERVIEW.md](OVERVIEW.md) for a high-level overview of the repository's purpose, scope, and
structure.

The role has two parts. First, it installs the developer tooling that was previously applied as a
sequence of manual shell commands: Docker Engine, [uv](https://docs.astral.sh/uv/), Node.js and npm,
the Claude Code CLI, tox, the GitHub CLI, and the `copilot.vim` plugin. Second, it applies the fifteen
[konstruktoid.hardening](https://galaxy.ansible.com/ui/repo/published/konstruktoid/hardening/) roles
included by `tasks/hardening.yml`.

Hardening is applied last on purpose: the tooling installs pull in apt packages, and an apt
transaction that installs or upgrades `systemd` re-runs `systemd-sysusers`, which would recreate the
accounts `konstruktoid.hardening.delete_users` removes.

## Usage

This repository includes a ready-to-run `playbook.yml` and `inventory.ini` that apply the role to the
local machine over the `local` connection plugin, so no SSH access or remote target is required.

1. Install Ansible-core 2.18 or later and the collections listed under Requirements.
2. Review `defaults/main.yml` and override any variable that does not match the target account or
   the intended set of hardening roles and tools.
3. Run the role against the local machine:

   ```shell
   ansible-playbook -K -i inventory.ini playbook.yml
   ```

   `-K` is required unless the account has passwordless sudo; see
   [Privilege escalation and `-K`](#privilege-escalation-and--k).

To apply the role to a remote host instead, add the host to an inventory and reference the role from a
playbook as shown in [Playbook example](#playbook-example), or run a subset of the role using the tags
declared in `tasks/main.yml`, for example `--tags docker,uv`.

### Privilege escalation and `-K`

Run the playbook as the unprivileged account that the workstation is being set up for, not as root,
and pass `-K` (`--ask-become-pass`) so ansible-core can supply that account's password to sudo. Without
it, the first task that elevates fails with `Missing sudo password`, and the run stops partway through.
`-K` can be omitted only where sudo is configured `NOPASSWD` for the account, as it is in the Molecule
scenarios.

The prompt is for the invoking account's own password, so the account must already be able to elevate
through sudo. On Ubuntu that means membership in the `sudo` group.

Most of the role never elevates. `-K` is needed because these parts do:

| Elevates | Why |
| --- | --- |
| The fifteen `konstruktoid.hardening.*` roles | They edit system configuration under `/etc` and manage systemd units; each role handles its own `become` internally. |
| `tasks/packages.yml`, `tasks/docker.yml`, `tasks/github_cli.yml` | apt package installation, and the GPG keyrings and `deb822` repository definitions under `/etc/apt`. |
| The `docker` group-membership task in `tasks/docker.yml` | Modifying an account's supplementary groups is a root operation. |

Everything else, namely `uv`, tox, the Claude Code CLI, and the `copilot.vim` clone, only writes to the
connecting user's home directory and is pinned to `become: false` on its import in `tasks/main.yml`, so
those tasks stay unprivileged even when the role is included from a playbook that sets `become: true`
for the whole play. Running the entire play as root is not supported: the tools would be installed
into root's home directory rather than the intended account's.

A run with every elevating part disabled (`workstation_harden_*`, `workstation_docker_install` and
`workstation_gh_install` set to `false`, and `tasks/packages.yml` skipped with
`--skip-tags packages`) needs no `-K` at all.

### sudo-rs and `become_exe`

`playbook.yml` sets `become_exe: sudo.ws`, and the [Playbook example](#playbook-example) below does the
same. This is required, not incidental. Ubuntu Resolute points `/usr/bin/sudo` at
[sudo-rs](https://github.com/trifectatechfoundation/sudo-rs), whose password prompt ansible-core does
not recognize, so any task using `become` hangs until it fails with `Timed out waiting for become
success`. `sudo.ws` is the classic sudo binary, registered as an `update-alternatives` entry by the
`sudo` package, and pointing `become_exe` at it avoids the prompt mismatch.

The upstream fix, [ansible/ansible#86175](https://github.com/ansible/ansible/pull/86175), is merged in
`devel` but has not been backported, so no ansible-core release up to 2.21.x contains it. Keep
`become_exe` in place until you are running a version that does; see
[ansible/ansible#85837](https://github.com/ansible/ansible/issues/85837) for the background. Setting
`ANSIBLE_BECOME_EXE=sudo.ws`, or `become_exe = sudo.ws` under `[privilege_escalation]` in
`ansible.cfg`, works equally well if you would rather not set it per play.

This does not affect the Molecule scenarios, which either connect as root or use a `NOPASSWD` sudoers
entry, so no password prompt is ever produced.

## Requirements

- Ansible-core 2.18 or later.
- Ubuntu Resolute (26.04). The role does not gate its tasks on `ansible_facts.distribution`, so running
  it against another operating system produces undefined results.
- An unprivileged account that can elevate through sudo, and `-K` on the command line to supply its
  password unless sudo is configured `NOPASSWD` for that account. See
  [Privilege escalation and `-K`](#privilege-escalation-and--k).
- The collections listed in `requirements.yml`:

  ```shell
  ansible-galaxy install -r requirements.yml
  ```

## Design principles

- **Security first.** Package and repository management use checksum-verified downloads and signed
  apt repositories instead of curl-piped-to-shell installers. The fifteen hardening roles are enabled
  by default and can be disabled individually, not collectively.
- **Minimal `become`.** Tasks run without elevated privileges unless the underlying operation requires
  root: apt package and repository management, Docker group membership, and the hardening role
  includes (which manage their own `become` internally). Tool installations that only need to write to
  the connecting user's home directory, such as `uv`, `tox`, the Claude Code CLI, and `copilot.vim`,
  never use `become`, and are pinned to `become: false` in `tasks/main.yml` so a play-level
  `become: true` in a consuming playbook cannot elevate them. See
  [Privilege escalation and `-K`](#privilege-escalation-and--k) for what this means at run time.
- **Restrictive file permissions.** Files the role manages directly carry an explicit owner, group, and
  mode. The apt keyrings under `/etc/apt/keyrings` are root-owned and mode `0644`; `~/.npmrc` is
  created mode `0600`, because npm also stores registry credentials there.

## Playbook example

```yaml
---
- hosts: all
  become_exe: sudo.ws
  tasks:
    - name: Include the workstation role
      ansible.builtin.import_role:
        name: workstation
```

Run it with `ansible-playbook -K`, and do not set `become: true` on the play; see
[Privilege escalation and `-K`](#privilege-escalation-and--k). See
[sudo-rs and `become_exe`](#sudo-rs-and-become_exe) for why `become_exe` is set.

## Role variables

Defined in `defaults/main.yml` and validated at run time by the role argument specification in
`meta/argument_specs.yml`, which declares the type and default of every variable listed below.

| Variable | Default | Description |
| --- | --- | --- |
| `workstation_user` | `"{{ ansible_facts.user_id }}"` | Account used for Docker group membership. |
| `workstation_harden_apport` | `true` | Enable the `konstruktoid.hardening.apport` role. |
| `workstation_harden_automatic_updates` | `true` | Enable the `konstruktoid.hardening.automatic_updates` role. |
| `workstation_harden_delete_users` | `true` | Enable the `konstruktoid.hardening.delete_users` role. |
| `workstation_harden_issue` | `true` | Enable the `konstruktoid.hardening.issue` role. |
| `workstation_harden_journald` | `true` | Enable the `konstruktoid.hardening.journald` role. |
| `workstation_harden_lock_root` | `true` | Enable the `konstruktoid.hardening.lock_root` role. |
| `workstation_harden_motd_news` | `true` | Enable the `konstruktoid.hardening.motd_news` role. |
| `workstation_harden_package_management` | `true` | Enable the `konstruktoid.hardening.package_management` role. |
| `workstation_harden_paths` | `true` | Enable the `konstruktoid.hardening.paths` role. |
| `workstation_harden_prelink` | `true` | Enable the `konstruktoid.hardening.prelink` role. |
| `workstation_harden_resolvedconf` | `true` | Enable the `konstruktoid.hardening.resolvedconf` role. |
| `workstation_harden_root_access` | `true` | Enable the `konstruktoid.hardening.root_access` role. |
| `workstation_harden_sudo` | `true` | Enable the `konstruktoid.hardening.sudo` role. |
| `workstation_harden_systemdconf` | `true` | Enable the `konstruktoid.hardening.systemdconf` role. |
| `workstation_harden_timesyncd` | `true` | Enable the `konstruktoid.hardening.timesyncd` role. |
| `workstation_packages` | see `defaults/main.yml` | apt packages installed without recommended or suggested extras. |
| `workstation_docker_install` | `true` | Install Docker Engine from the upstream apt repository. |
| `workstation_docker_packages` | see `defaults/main.yml` | Docker Engine apt packages. |
| `workstation_docker_group_membership` | `true` | Add `workstation_user` to the `docker` group. |
| `workstation_uv_install` | `true` | Install `uv`. |
| `workstation_uv_arch` | `"{{ ansible_facts.architecture }}"` | Architecture used to select the `uv` release archive. |
| `workstation_uv_release` | `"0.11.32"` | Pinned `uv` release. |
| `workstation_uv_url` | see `defaults/main.yml` | Base URL for the `uv` release archive. |
| `workstation_npm_global_prefix` | `"{{ ansible_facts.env.HOME }}/.local"` | npm global install prefix, owned by the connecting user. |
| `workstation_tox_install` | `true` | Install tox as a `uv` tool. |
| `workstation_claude_code_install` | `true` | Install the Claude Code CLI through npm. |
| `workstation_claude_code_package` | `"@anthropic-ai/claude-code"` | npm package installed for Claude Code. |
| `workstation_gh_install` | `true` | Install the GitHub CLI from the upstream apt repository. |
| `workstation_copilot_vim_install` | `true` | Clone `copilot.vim`. |
| `workstation_copilot_vim_repo` | `"https://github.com/github/copilot.vim.git"` | Repository cloned for `copilot.vim`. |
| `workstation_copilot_vim_version` | `release` | Git ref checked out for `copilot.vim`. |
| `workstation_copilot_vim_dest` | `"{{ ansible_facts.env.HOME }}/.config/nvim/pack/github/start/copilot.vim"` | Destination path for the `copilot.vim` clone. |
| `shasums` | see `defaults/main.yml` | Checksums for pinned tool releases, keyed by architecture. |

Every `workstation_harden_*` and `workstation_*_install` variable can be set to `false` independently,
so a specific hardening role or tool installation can be skipped without disabling the rest.

## Docker group membership

If `workstation_docker_group_membership` is `true` (the default), `workstation_user` is added to the
`docker` group. Membership in that group is equivalent to root access on the host, because it grants
unrestricted access to the Docker daemon socket. Set `workstation_docker_group_membership: false` if
that trade-off is not acceptable, and use `sudo docker` or an equivalent access control mechanism
instead.

## Checksum verification

`uv` is installed from a release archive downloaded over HTTPS and verified against the SHA-256
checksum recorded in `shasums.workstation_uv_release`, rather than by executing the upstream install
script. When `workstation_uv_release` is updated, update the corresponding checksums for both the
`x86_64` and `aarch64` architectures, and verify the new checksums against the values published by the
[uv release](https://github.com/astral-sh/uv/releases) before committing them.

Docker Engine and the GitHub CLI are installed from their respective upstream apt repositories, added
using a GPG-signed keyring under `/etc/apt/keyrings` and the `ansible.builtin.deb822_repository`
module, following the same pattern used by both projects' official installation instructions.

## Testing with molecule

Two [Molecule](https://ansible.readthedocs.io/projects/molecule/) scenarios exercise this role, both
against a single Ubuntu Resolute target:

- the `default` scenario boots a QEMU/UEFI cloud image using Molecule's ansible-native driver
  (`driver: name: default`, no Docker or Vagrant plugin required), and
- the `docker` scenario provisions a privileged container through the `community.docker` collection.

Both scenarios converge by running the role, and `molecule/default/verify.yml` (reused by the `docker`
scenario) asserts that the expected packages are installed, the connecting user is a member of the
`docker` group, Docker Engine is running, `uv` and tox are installed and functional, the Claude Code
CLI and `copilot.vim` are present, `~/.npmrc` and the apt keyrings have the expected owner and mode,
and a representative subset of the hardening changes took effect
(root account locked, Apport disabled, unnecessary system users removed, PATH hardening in place,
`systemd-timesyncd` and `systemd-resolved` configured).

```shell
uv pip install -r requirements-dev.txt
ansible-galaxy install --force -r requirements.yml
molecule test
molecule test -s docker
```

The `default` scenario requires `qemu-system-x86_64`, `qemu-img`, `genisoimage`, and OVMF firmware
(`/usr/share/OVMF/OVMF_{CODE,VARS}_4M.fd`) on the host running the tests; the base cloud image is
downloaded once and cached under `~/.cache/molecule-qemu/images`. The `docker` scenario requires a
Docker daemon the invoking user can access.

`tox -l` lists all available `tox` test environments. `tox -e docker` installs the collections listed
in `requirements.yml`, runs `ansible-lint`, and then runs `molecule test -s docker`.

## Continuous integration

Six GitHub Actions workflows run against this repository. Every job that runs on a runner starts with
[`step-security/harden-runner`](https://github.com/step-security/harden-runner), declares an explicit
`timeout-minutes`, and requests the narrowest `permissions` it needs. Third-party actions are pinned to
a commit SHA and kept current by Dependabot; the only exception is the reusable
`slsa-framework/slsa-github-generator` workflow, which has to be referenced by release tag.

| Workflow | Trigger | Purpose |
| --- | --- | --- |
| `lint.yml` | push, pull request, every third day | Runs `ansible-lint` at `profile: production`. |
| `molecule.yml` | push, pull request, every third day, manual | Runs `tox -e docker` and `tox -e docker-upstream`, that is the `docker` Molecule scenario against released and against upstream Ansible. |
| `slsa.yml` | push, release | Hashes the role content, then generates SLSA build provenance and attaches the checksum file to tagged releases. |
| `scorecards.yml` | push to `main`, weekly, branch protection change | Runs the OpenSSF Scorecard analysis and uploads the results to code scanning. |
| `dependency-review.yml` | pull request | Blocks pull requests that introduce known-vulnerable dependencies. |
| `issues.yml` | issue opened | Assigns new issues to the maintainer. |

The `default` Molecule scenario is not exercised in CI, because it boots a QEMU virtual machine and
depends on host virtualization support. Run it locally as described under
[Testing with molecule](#testing-with-molecule).

## Contributing

Contributions are welcome, regardless of size. If something looks wrong, open an issue; if you have a
fix, open a pull request.

## License

Apache License Version 2.0

## Author Information

[https://github.com/konstruktoid](https://github.com/konstruktoid "github.com/konstruktoid")
