# workstation

An [Ansible](https://www.ansible.com/) role that installs and hardens a developer workstation running
Ubuntu Resolute (26.04). The role is the only supported target platform; it does not attempt to support
other Ubuntu releases or distributions.

See [OVERVIEW.md](OVERVIEW.md) for a high-level overview of the repository's purpose, scope, and
structure.

The role has two parts. First, it applies the fifteen [konstruktoid.hardening](https://galaxy.ansible.com/ui/repo/published/konstruktoid/hardening/)
roles listed in `ansible-roles`. Second, it installs the developer tooling previously captured as
manual shell commands in `install-log`: Docker Engine, [uv](https://docs.astral.sh/uv/), Node.js and
npm, the Claude Code CLI, tox, the GitHub CLI, and the `copilot.vim` plugin.

## Usage

This repository includes a ready-to-run `playbook.yml` and `inventory.ini` that apply the role to the
local machine over the `local` connection plugin, so no SSH access or remote target is required.

1. Install Ansible-core 2.18 or later and the collections listed under Requirements.
2. Review `defaults/main.yml` and override any variable that does not match the target account or
   the intended set of hardening roles and tools.
3. Run the role against the local machine:

   ```shell
   ansible-playbook -i inventory.ini playbook.yml
   ```

To apply the role to a remote host instead, add the host to an inventory and reference the role from a
playbook as shown in [Playbook example](#playbook-example), or run a subset of the role using the tags
declared in `tasks/main.yml`, for example `--tags docker,uv`.

## Requirements

- Ansible-core 2.18 or later.
- Ubuntu Resolute (26.04). The role does not gate its tasks on `ansible_facts.distribution`, so running
  it against another operating system produces undefined results.
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
  never use `become`.

## Playbook example

```yaml
---
- hosts: all
  tasks:
    - name: Include the workstation role
      ansible.builtin.import_role:
        name: workstation
```

## Role variables

Defined in `defaults/main.yml`.

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
CLI and `copilot.vim` are present, and a representative subset of the hardening changes took effect
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

## Contributing

Contributions are welcome, regardless of size. If something looks wrong, open an issue; if you have a
fix, open a pull request.

## License

Apache License Version 2.0

## Author Information

[https://github.com/konstruktoid](https://github.com/konstruktoid "github.com/konstruktoid")
