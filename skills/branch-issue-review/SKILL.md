---
name: branch-issue-review
description: >-
  Branch-scoped code review in the CURRENT workspace git repository only: check for issues,
  code review, audit/review the branch, review this PR, PR review (without a file list). Do not search other
  repos. Enforces clean working tree (stop if dirty), then git fetch origin and git pull
  when @{u} exists so remotes are current; optional checkout when extra message text names a branch;
  diff vs default base (origin/main, origin/master, main, master), review only changed files, hidden-items rollup.
---

# Branch-scoped issue review

## Scope

This skill runs entirely inside **one git repository**: the **current Cursor workspace’s repo** (the tree returned by `git rev-parse --show-toplevel` when run from the workspace root). It is meant to work in **whatever** project you have open—not tied to a fixed path—but **never** look for branches, run review commands, or read files **outside** that repo (no scanning `~/repos`, sibling folders, or multi-repo search unless the user explicitly opens that other repo as the workspace).

Use this whenever the user asks for branch-scoped review without listing only specific files (unless **Preconditions** below still apply).

When triggered (including phrases like **check for issues**, **code review**, **review for problems**, **audit the branch**, **review this branch**, **review this PR**, **PR review**):

## Preconditions (apply first)

All git commands and branch resolution below run **only in the current workspace git repo**.

1. **Stop if dirty tree:** `git status --porcelain` must be empty. If not, **stop** the review; ask the user to commit, stash, or discard. Do not continue until clean.

2. **Sync remotes (discover new branches):** Run **`git fetch origin`** so **`origin/*`** updates and newly pushed remote branches become visible. Use **`git fetch --all --prune`** instead when this repo relies on multiple remotes or you need pruned stale refs. Then run **`git pull`** when the **current** branch has an upstream (`@{u}` resolves)—this completes the usual “update before review” flow after fetch. If **`git pull`** stops with conflicts or errors, **stop** and tell the user to fix their branch before reviewing.

3. **Optional branch checkout from extra text:** If the message contains **extra text** beyond the review phrase (same line or next line), parse the trimmed remainder using **Checkout guardrails** below. When checkout applies, run **`git checkout`** (plain name first, then **`origin/<name>`** if needed). Example: `code review issue/PROJ-1234` → checkout `issue/PROJ-1234`, then `origin/issue/PROJ-1234` if needed. If checkout was required but the ref still does not resolve **after** fetch, **stop** and report—not a valid branch in this repository (do **not** hunt for it in other projects).

   **Checkout guardrails**

   - **Do not checkout** when the remainder is clearly **scope or description**, not a ref: file paths (`src/foo.rb`), broad topics (`the auth refactor`, `login flow`), or filler (`before merge`, `my changes`). Review the **current** branch (after fetch/pull).
   - **Do checkout** when any of these apply:
     - User names the branch explicitly (`branch issue/PROJ-1234`, `on feature/foo`).
     - The token looks like a ref (`issue/PROJ-1234`, `feature/foo`, `PROJ-1234`) **and** matches a local or `origin/*` branch after fetch (`git branch -a --list '*<token>*'`, or `git show-ref --verify --quiet refs/heads/<token>` / `refs/remotes/origin/<token>`).
   - **Ambiguous remainder** (could be description or ref): check whether it matches an existing branch after fetch. Match → checkout; no match → **do not checkout**; review the current branch. Do **not** **stop** solely because text was ambiguous.
   - **Clear branch intent, no matching ref** (explicit `branch`/`on`, or ref-shaped token the user clearly meant as a branch): **stop** and report the ref is missing in this repo.

## Scoped review (when user has not listed exact paths)

- Run git steps **4–6** below first.
- Regardless of scope, follow **7–8** and **Reporting conventions**.

4. **Resolve the default base branch**. Prefer in order: `origin/main`, `origin/master`, `main`, `master` — use the first ref that exists locally or on `origin` (remotes were refreshed in precondition 2).

5. **List changed paths** for commits on the current branch only:
   - `git merge-base <base> HEAD` then `git diff --name-only <merge-base>..HEAD`
   - or: `git diff --name-only <base>...HEAD` (three-dot)

6. **Review only those files** (plus any path the user additionally names). Do not do a repo-wide review unless they ask.

7. **Report** with file-scoped findings. Say if the branch matches the base (no changed files). Always include a **hidden-items rollup** at the end of the review (see **Reporting conventions** below).

8. **Severity for style/readability**: Multi-condition `unless` statements (e.g. `unless A && B`, `unless $x || $y`, chained `unless` with multiple tests) are hard to read and easy to mis-edit. When flagging them, treat them as **low priority** (polish/refactor), not correctness bugs, unless they clearly change behavior.

**Skip** the git-scope steps (resolve base branch, diff changed paths) if the user already listed exact files/paths, or is clearly asking about a single known file—**but** if they used branch-review phrasing (**code review**, **review this branch**, etc.), **still** enforce **clean working tree**, **fetch/pull**, and optional **branch checkout** from extra message text before reviewing those paths.

**Still follow Reporting conventions**, including the hidden-items rollup.

## Reporting conventions

**Hidden items rollup (required):** After the main findings, append a compact line stating how many findings were **not** fully expanded (and optional one-line buckets). Examples: “Hidden items: **3** (2 low-priority style, 1 speculative)” or “Hidden items: **0**”.

Treat as **hidden** when you abbreviated, deferred, summarized, or withheld full detail—but the item still matters for completeness: low-severity/readability/style, speculative risks without evidence, omissions for brevity, nice-to-have refactors out of scope, or anything nodded at without a dedicated paragraph. Multi-condition `unless` called out briefly as polish counts here unless dissected inline.

Do **not** count pure filler (“no findings”) or tautological closings—only substantive items deserving a tally.
