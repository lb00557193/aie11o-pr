---
name: pr-dev
description: "Create a Bitbucket pull request from the current feature branch to a target integration branch (e.g. dev). Trigger on `/pr-dev`. Target branch comes from $ARGUMENTS (or is prompted). Source is the current HEAD if it matches a feature-style branch prefix (feature/feat/fix/bugfix/hotfix/improvement/improvements/chore/refactor/docs/test/release/ci); otherwise the user must specify both source and target. Title is generated from the latest commit subject prefixed with the Jira ticket extracted from the branch name. Description is generated from the diff, following the pr_generation template, with empty sections omitted. Default language is Traditional Chinese (zh); pass `--lang en` for English. If a `jira` section exists in ~/.aie11o/aie11o.json, also posts the description as a Jira comment with marker support (overwrites prior marker comment on re-iteration with a holistic regenerated description)."
---

# /pr-dev — Create PR (feature → integration)

Promote the current feature branch to a target integration branch
(e.g. `dev`, `develop`, `main`).

## Inputs

- **Target branch** — first positional arg in `$ARGUMENTS`. If missing,
  ask the user.
- **Language flag** — optional `--lang en` / `--lang zh` anywhere in
  `$ARGUMENTS`. Default `zh`.
- **Source branch** — current HEAD, but only if it matches:
  `^(feature|feat|fix|bugfix|hotfix|improvement|improvements|chore|refactor|docs|test|release|ci)/`.
  Otherwise STOP and ask the user to specify both source and target
  explicitly.

Examples:
- `/pr-dev dev`
- `/pr-dev main --lang en`

## Procedure

Follow the shared workflow at
`~/.claude/skills/pr-dev/WORKFLOW.md`, sections 1–8.

For step 6 (title + description), use **Mode A** (`/pr-dev`).
