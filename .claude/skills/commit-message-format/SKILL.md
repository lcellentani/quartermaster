---
name: commit-message-format
description: "Reference for the commit message format: [Tn] type(scope): subject. Not user-invocable directly — used internally by the /commit-message command."
user-invocable: false
---
  
# Commit format

`[Tn] type(scope): subject`

- `[Tn]` is the delegation tier prefix: [T1], [T2], [T3], or [T4]
- `type(scope)` follows Conventional Commits
- `Subject` uses imperative mood, ≤72 chars, no trailing period

# Example

[T2] feat(hooks): detect Copilot CLI vs Claude Code config path

Adds the shared plumbing that inspects the environment to decide which
agent's config directory to write to, so hooks install to the right
place without the user specifying it manually.
