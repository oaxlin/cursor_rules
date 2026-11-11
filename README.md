# cursor_config

Personal [Cursor](https://cursor.com) configuration: rules and agent skills that shape how the AI behaves in your editor.

## Installation

Clone (or pull) this repo under `~/repos/cursor_config`, then merge **rules** and **skills** into `~/.cursor/`. No build step.

```bash
cd ~/repos/cursor_config
git pull

cp -a rules/. ~/.cursor/rules/
cp -a skills/. ~/.cursor/skills/
```

`cp -a` overwrites files that exist in both places and leaves **extra** files in `~/.cursor/rules/` alone (see below).

Cursor reads rules from `~/.cursor/rules/` and skills from `~/.cursor/skills/` automatically.

## Updating

Same as installation after `git pull`:

```bash
cd ~/repos/cursor_config && git pull
cp -a rules/. ~/.cursor/rules/
cp -a skills/. ~/.cursor/skills/
```

A new Agent chat (or window reload) helps pick up rule changes.

## Contents

### Rules in this repo (`rules/`)

| File | Scope |
|------|-------|
| `karpathy-guidelines.mdc` | Always applied — behavioral guidelines for coding and review |
| `branch-issue-review.mdc` | Always applied — routes review requests to the `branch-issue-review` skill (branch vs base, or uncommitted/working-tree) |
| `commit-message-approval.mdc` | Always applied — propose commit message and wait for approval before `git commit` |
| `kubectl-readonly.mdc` | Always applied — prefer `~/.krew/bin/kubectl-readonly` for read-only cluster checks |
| `perl-style.mdc` | Always applied — Perl layout, refs, DBI `0E0`, writing guidance; pairs with `perl-review.mdc` |
| `perl-review.mdc` | Always applied — Perl **review** policy (undef `->[]` / `->{}` chains are not crashes); repo-local `team/perl-review.mdc` stubs are deprecated |
| `perl-run-scripts-directly.mdc` | Always applied — run executable `*.pl` via `./script` when shebang is present |

### Rules only on some machines (`~/.cursor/rules/`)

This repository is the canonical set for the table above. You may keep **additional** personal rules only under `~/.cursor/rules/` (for example machine-specific or per-workspace rules). A plain `cp -a rules/. ~/.cursor/rules/` does **not** delete those extras.

### Skills (`skills/`)

| Skill | Purpose |
|-------|---------|
| `branch-issue-review` | Code review in the current workspace git repo by default, or explicitly named repositories for multi-repo reviews: **branch vs base** (`code review`, `review this PR`) requires a clean tree; **working tree** (`review uncommitted diff`, `review staged changes`) reviews local changes without commit. Applies workspace rules by file type, large-diff policy, and report-only test/CI guidance. |
