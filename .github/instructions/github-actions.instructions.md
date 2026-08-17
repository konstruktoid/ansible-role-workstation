---
applyTo: ".github/workflows/**/*.yml,.github/workflows/**/*.yaml"
---

# GitHub Actions Security Instructions

The rules for workflow definitions live in
[`.agents/instructions/github-actions.md`](../../.agents/instructions/github-actions.md), and extend the
repository-wide instructions in [`AGENTS.md`](../../AGENTS.md). Read both; they apply in full here.

This file exists only to declare the path scope above for tools that read
`.github/instructions/*.instructions.md`. It is an entry point, not a copy: make every content change
in `.agents/instructions/github-actions.md` rather than here.
