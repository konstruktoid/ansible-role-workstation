# Repository Overview: `workstation`

## Introduction

`workstation` is an [Ansible](https://www.ansible.com/) role that configures a single Ubuntu
Resolute (26.04) machine as a hardened developer workstation. It targets exactly one platform,
Ubuntu Resolute, and does not attempt to support other Ubuntu releases or Linux distributions.

The repository addresses a specific problem: preparing a fresh Ubuntu Resolute installation for
software development work while reducing its exposed attack surface. This preparation previously
involved an ad hoc sequence of manual shell commands and curl-piped install scripts, a process that
is slow to repeat, difficult to audit, and easy to get wrong.

The intended audience is individual developers and small teams who provision their own Ubuntu
Resolute workstations and want that provisioning to be repeatable, reviewable, and aligned with
recognized security hardening guidance rather than assembled from memory each time a machine is
rebuilt.

## Purpose

The repository exists to combine workstation setup and workstation hardening into a single,
version-controlled Ansible role. The two tasks are addressed together because they compete for the
same resource, namely root privileges on the target machine: development tooling has to be
installed, and the underlying operating system has to be locked down. Treating them as one role
keeps the trade-offs between convenience and security visible in one place instead of split across
unrelated scripts.

The guiding principles are security first and minimal use of elevated privileges. Package and
repository management rely on checksum-verified downloads and signed apt repositories rather than
curl-piped-to-shell installers, and the fifteen hardening roles the project depends on are enabled
by default. Tasks run without elevated privileges unless the underlying operation strictly requires
root, such as apt package and repository management or Docker group membership; tool installations
that only write to the connecting user's home directory never elevate privileges.

The result is a workstation that starts from a known, auditable, and hardened baseline. Every
hardening control and every tool installation is individually toggleable, so a user can opt out of
any single behavior without disabling the rest.

## Major Components

**Hardening integration** applies a curated set of independent hardening roles published by the
`konstruktoid.hardening` collection, covering areas such as automatic updates, journald and logging
configuration, root account locking, sudo policy, systemd configuration, and time synchronization.
Each hardening role is included conditionally, controlled by its own dedicated toggle, so any one
control can be disabled without affecting the others.

**Developer tooling installation** installs the software a developer is expected to need on a
workstation: Docker Engine, the `uv` Python package and project manager, Node.js and npm, the Claude
Code command-line interface (CLI), tox, the GitHub CLI, and the `copilot.vim` plugin. Each tool is
installed through a mechanism appropriate to its trust model, for example a checksum-verified release
archive for `uv`, or a signed upstream apt repository for Docker Engine and the GitHub CLI, rather
than through a generic install script executed with elevated trust.

**Role variables and defaults** define every toggle and configuration point exposed by the role,
including which account tooling is installed for, which packages are installed, and the pinned
versions and checksums used for integrity verification. These variables allow hardening and tooling to
be selectively enabled, disabled, or reconfigured without modifying task logic.

**Test harness** exercises the role through two independent Molecule scenarios, one using a
QEMU-booted virtual machine and the other using a privileged container, both converging the same role
and verifying the same outcomes: expected packages present, Docker group membership granted, Docker
Engine running, `uv` and tox functional, the Claude Code CLI and `copilot.vim` present, and a
representative sample of the hardening changes in effect. The test harness confirms that hardening and
tooling continue to behave correctly as they change. Continuous integration runs the linting and the
container-based scenario automatically, against both the released and the development version of
Ansible, and complements them with supply-chain checks: dependency review on pull requests, an
OpenSSF Scorecard analysis, and SLSA build provenance for the role content. The virtual-machine
scenario is left to local runs, because it depends on host virtualization support.

## Scope

The repository is scoped to provisioning a single Ubuntu Resolute machine, whether a physical
workstation or a virtual machine, into a hardened state suitable for software development. Included
functionality covers the fifteen hardening domains supplied by the `konstruktoid.hardening`
collection and the installation of the specific developer tools listed above.

Intended use cases include applying the role to a freshly installed Ubuntu Resolute machine as part
of initial setup, re-running the role against an already-configured machine to bring it back to the
declared baseline, and applying the role with a subset of its hardening controls or tool
installations disabled to accommodate local requirements. Each hardening role and each tool
installation is expected to be idempotent and safe to re-run.

## Out of Scope

The repository does not define or invent hardening controls of its own. It depends on and applies
the hardening roles published by the separate `konstruktoid.hardening` collection, and defers to that
collection's own scope and guidance for the content of each control.

It does not support any operating system other than Ubuntu Resolute. The role does not inspect
`ansible_facts.distribution` and does not gate its tasks on the detected platform, so applying it to a
different operating system or Ubuntu release produces undefined results rather than a controlled
error.

It does not manage the broader fleet-level concerns that a configuration management system for many
hosts would need, such as inventory management, secrets distribution, or orchestration across
multiple machines. It is scoped to a single target host at a time.

It does not select or install an arbitrary set of developer tools. The tooling installed is fixed to
the specific list captured from prior manual setup, and adding new tools requires an intentional
change to the role rather than a general-purpose package installation mechanism.

## Architecture Summary

The role has no orchestration layer beyond the standard Ansible role structure. A single entry point,
`tasks/main.yml`, imports one task file for the hardening integration and one task file for each
piece of developer tooling, with each import gated by its own boolean variable and tagged so it can
be targeted individually. The hardening integration in turn includes each of the fifteen
`konstruktoid.hardening` roles, again gated individually, delegating their internal logic and their
own use of elevated privileges to those roles rather than duplicating it.

Configuration flows in one direction: variables declared in `defaults/main.yml` determine which task
files execute and how they behave, and no task file writes back to those defaults or to another task
file's state. The role can be included directly by a playbook running against a remote host, or
invoked through the repository's own `playbook.yml` and `inventory.ini` against the local machine,
without any change to the role itself.
