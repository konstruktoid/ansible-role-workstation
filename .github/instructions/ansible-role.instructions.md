---
applyTo: "{tasks,defaults,meta,molecule}/**/*.{yml,yaml}"
---

# Workstation Role Instructions

The rules for Ansible YAML changes in this role live in
[`.agents/instructions/ansible-role.md`](../../.agents/instructions/ansible-role.md), and extend the
repository-wide instructions in [`AGENTS.md`](../../AGENTS.md). Read both; they apply in full here.

This file exists only to declare the path scope above for tools that read
`.github/instructions/*.instructions.md`. It is an entry point, not a copy: make every content change
in `.agents/instructions/ansible-role.md` rather than here.
