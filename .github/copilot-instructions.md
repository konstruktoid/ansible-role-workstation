# Repository Instructions for Coding Agents

The instructions for this repository live in [`AGENTS.md`](../AGENTS.md). Read that file; it applies in
full here.

This file exists only because some tools look for `.github/copilot-instructions.md` specifically. It is
an entry point, not a copy: `AGENTS.md` is the single source of truth, so make every content change
there rather than here, and do not restate its rules in this file.

Path-scoped instructions that extend `AGENTS.md`:

- [`.agents/instructions/ansible-role.md`](../.agents/instructions/ansible-role.md) for
  `{tasks,defaults,meta,molecule}/**/*.{yml,yaml}`, mirrored as an entry point at
  [`.github/instructions/ansible-role.instructions.md`](instructions/ansible-role.instructions.md).
- [`.agents/instructions/github-actions.md`](../.agents/instructions/github-actions.md) for
  `.github/workflows/**`, mirrored as an entry point at
  [`.github/instructions/github-actions.instructions.md`](instructions/github-actions.instructions.md).
- [`.agents/skills/ansible-verification-loop/SKILL.md`](../.agents/skills/ansible-verification-loop/SKILL.md)
  for the bounded lint/test/idempotence loop every change to the role has to clear before it is
  reported as done.
