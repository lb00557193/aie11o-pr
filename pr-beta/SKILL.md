---
name: pr-beta
description: "Create a Bitbucket pull request from a dev-equivalent branch to a beta-equivalent target. Trigger on `/pr-beta`. Takes two positional args: `<source> <target>` (e.g. `/pr-beta dev beta`). If either is missing, the skill asks for it. Title and description are copied verbatim from the most recently MERGED PR whose destination is the source branch; if none exists, falls back to a diff-based description."
---

# /pr-beta — Cascade PR (dev-equivalent → beta-equivalent)

Promote a dev-equivalent branch to a beta-equivalent target.

## Inputs

- **Source branch** — first positional arg in `$ARGUMENTS`. If missing,
  ask the user.
- **Target branch** — second positional arg in `$ARGUMENTS`. If missing,
  ask the user.

Example: `/pr-beta dev beta`

## Procedure

Follow the shared workflow at
`~/.claude/skills/pr-dev/WORKFLOW.md`, sections 1–6.

For step 5 (title + description), use **Mode B** (cascade promotion) —
copy from the most recently MERGED PR targeting the source branch.
