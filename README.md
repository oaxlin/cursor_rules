# cursor_config

Personal [Cursor](https://cursor.com) configuration: rules and agent skills that shape how the AI behaves in your editor.

## Installation

Copy the contents of this repository into `~/.cursor`. That is the entire install—no build step, package manager, or extra tooling.

```bash
# Clone (or download) this repo, then copy rules and skills into ~/.cursor
cp -r rules skills ~/.cursor/
```

If you already have files in `~/.cursor/rules` or `~/.cursor/skills`, merge carefully so you do not overwrite anything you want to keep. You can copy individual files instead of whole directories.

Cursor reads rules from `~/.cursor/rules/` and skills from `~/.cursor/skills/` automatically.

## Contents

### Rules (`rules/`)

| File | Scope |
|------|-------|
| `karpathy-guidelines.mdc` | Always applied — behavioral guidelines for coding and review |
| `branch-issue-review.mdc` | Always applied — routes branch/PR review requests to the matching skill |
| `perl-style.mdc` | Applied to Perl files (`*.pm`, `*.t`, `*.pl`, etc.) |

### Skills (`skills/`)

| Skill | Purpose |
|-------|---------|
| `branch-issue-review` | Branch-scoped code review in the current workspace git repo |

## Updating

Pull the latest changes from this repo and copy the updated `rules/` and `skills/` directories into `~/.cursor/` again.
