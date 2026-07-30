---
name: branch-issue-review
description: >-
  Branch-scoped code review in the CURRENT workspace git repository only: check for issues,
  code review, audit/review the branch, review this PR, PR review (without a file list). Do not search other
  repos. Branch-vs-base review requires a clean tree; uncommitted/working-tree/staged scope
  uses local diff instead. Then git fetch origin and git pull
  when @{u} exists so remotes are current; optional checkout when extra message text names a branch;
  diff vs default base (origin/main, origin/master, main, master), review changed files plus
  minimum supporting files for call-chain issues,
  workspace rules by file type, large-diff policy, test/CI awareness (report-only), hidden-items rollup.
---

# Branch-scoped issue review

## Scope

This skill runs entirely inside **one git repository**: the **current Cursor workspace’s repo** (the tree returned by `git rev-parse --show-toplevel` when run from the workspace root). It is meant to work in **whatever** project you have open—not tied to a fixed path—but **never** look for branches, run review commands, or read files **outside** that repo (no scanning `~/repos`, sibling folders, or multi-repo search unless the user explicitly opens that other repo as the workspace).

Use this whenever the user asks for branch-scoped review without listing only specific files (unless **Preconditions** below still apply).

When triggered (including phrases like **check for issues**, **code review**, **review for problems**, **audit the branch**, **review this branch**, **review this PR**, **PR review**, **review uncommitted**, **review working tree**, **review staged changes**):

## Working-tree review (uncommitted / staged / local changes)

When the user clearly wants to review **local uncommitted changes** (not branch-vs-base), use this path **instead of** precondition **1** (clean tree), precondition **2** (fetch/pull), and git steps **4-5**. Phrases include **uncommitted**, **uncommitted diff**, **working tree**, **working-tree**, **local changes**, **staged**, **staged changes**, **unstaged**, and similar.

**Precedence:** When both branch-review phrasing (**code review**, **review this branch**, etc.) and working-tree phrases appear in the same message (e.g. `code review uncommitted diff`), **working-tree review wins**—use this path, not branch-vs-base.

- A **dirty tree is expected**; do not ask the user to commit, stash, or discard first.
- **Skip** fetch/pull and branch-vs-base diff.
- **Optional branch checkout** (precondition 3) applies only when extra text names a branch ref.

**List changed paths** for the requested scope:

- **All uncommitted** (default when scope is ambiguous): `git diff HEAD --name-only` (staged + unstaged vs `HEAD`)
- **Staged only:** `git diff --staged --name-only`
- **Unstaged only:** `git diff --name-only`

If the user named specific paths, use those (optionally intersect with the diff path list). If the path list is empty, report that there are no changes in that scope and stop.

Read the matching diff for review content: `git diff HEAD`, `git diff --staged`, or `git diff` as appropriate. Then follow steps **6-10** and **Reporting conventions**.

## Preconditions (apply first)

All git commands and branch resolution below run **only in the current workspace git repo**. **Skip preconditions 1-2** when **Working-tree review** applies.

1. **Stop if dirty tree** (branch-vs-base reviews only): `git status --porcelain` must be empty. If not, **stop** the review; ask the user to commit, stash, or discard. Do not continue until clean. **Exception:** **Working-tree review** (above)—a dirty tree is expected.

2. **Fetch / pull** (branch-vs-base only): Run **`git fetch origin`** so **`origin/*`** updates and newly pushed remote branches become visible. Use **`git fetch --all --prune`** instead when this repo relies on multiple remotes or you need pruned stale refs. Then run **`git pull`** when the **current** branch has an upstream (`@{u}` resolves)—this completes the usual “update before review” flow after fetch. If **`git pull`** stops with conflicts or errors, **stop** and tell the user to fix their branch before reviewing. **Exception:** **Working-tree review**—skip fetch/pull.

3. **Optional branch checkout from extra text:** If the message contains **extra text** beyond the review phrase (same line or next line), parse the trimmed remainder using **Checkout guardrails** below. When checkout applies, run **`git checkout`** (plain name first, then **`origin/<name>`** if needed). Example: `code review issue/PROJ-1234` → checkout `issue/PROJ-1234`, then `origin/issue/PROJ-1234` if needed. If checkout was required but the ref still does not resolve **after** fetch, **stop** and report—not a valid branch in this repository (do **not** hunt for it in other projects).

   **Checkout guardrails**

   - **Do not checkout** when the remainder is clearly **scope or description**, not a ref: file paths (`src/foo.rb`), broad topics (`the auth refactor`, `login flow`), or filler (`before merge`, `my changes`). Review the **current** branch (after fetch/pull).
   - **Do checkout** when any of these apply:
     - User names the branch explicitly (`branch issue/PROJ-1234`, `on feature/foo`).
     - The token looks like a ref (`issue/PROJ-1234`, `feature/foo`, `PROJ-1234`) **and** matches a local or `origin/*` branch after fetch (`git branch -a --list '*<token>*'`, or `git show-ref --verify --quiet refs/heads/<token>` / `refs/remotes/origin/<token>`).
   - **Ambiguous remainder** (could be description or ref): check whether it matches an existing branch after fetch. Match → checkout; no match → **do not checkout**; review the current branch. Do **not** **stop** solely because text was ambiguous.
   - **Clear branch intent, no matching ref** (explicit `branch`/`on`, or ref-shaped token the user clearly meant as a branch): **stop** and report the ref is missing in this repo.

## Scoped review (when user has not listed exact paths)

- Run git steps **4-6** below first.
- Regardless of scope, follow **7-10** and **Reporting conventions**.

4. **Resolve the default base branch**. Prefer in order: `origin/main`, `origin/master`, `main`, `master` — use the first ref that exists locally or on `origin` (remotes were refreshed in precondition 2).

5. **List changed paths** for commits on the current branch only:
   - `git merge-base <base> HEAD` then `git diff --name-only <merge-base>..HEAD`
   - or: `git diff --name-only <base>...HEAD` (three-dot)

6. **Review the changed files** (from step 5, **Working-tree review**, or paths the user names), and **the minimum supporting files** needed to validate behavior that spans files. Start from the diff; expand scope only when a finding depends on a caller, callee, outer transaction, or shared helper not in the diff. Examples: `is_scan` / reconcile logic in `Domain.pm` may require reading the matching `*::Scan` reconcile entrypoint; indirect `q_job` queueing may require tracing into `unassign`, `sync`, or `add_sync_q_job` in related modules. Read the **smallest** set of extra files that closes the chain—typically one or two, not a directory sweep. Do not do a repo-wide review unless they ask. List any supporting files you read beyond the diff in the report summary.

7. **Apply workspace rules by file type.** After you have the review path list (changed paths from step 5 or **Working-tree review**, paths the user named, plus any supporting files per step **6** when needed), read and follow matching rules before the generic checklist below.
   - **Discover rules** in `<repo>/.cursor/rules/*.mdc` (project) and `~/.cursor/rules/*.mdc` (user). Project rules win when they conflict with user rules.
   - **Match by path:** use each rule's `globs` when present. Rules with `alwaysApply: true` apply to every review in that workspace.
   - **Precedence:** workspace rules override generic items in **Things to look for** when they conflict.
   - **Skip as checklist:** `branch-issue-review.mdc` only routes to this skill - do not treat it as extra review criteria.
   - **Examples** (read the actual files; do not assume names): `**/*.{pm,t,pl}` may map to `perl-style.mdc` or a repo `perl-review.mdc`; test-related rules may prescribe how to run or recommend tests.

8. **Large-diff policy.** When the changed path list is large, scope the review explicitly instead of pretending every file got equal depth.
   - **Threshold:** treat **more than 30 changed files** or a very large diff (rough guide: thousands of lines changed) as a large diff. State that in the report summary.
   - **Summarize scope:** group changed paths by top-level directory or module; call out areas you did **not** read line-by-line.
   - **Prioritize deep review** for high-risk paths when present: authn/authz, crypto/secrets, migrations/schema, concurrency/async, public APIs or RPC handlers, config/feature flags, dependency manifest changes, and security-sensitive parsers.
   - **Breadth pass:** for remaining files, look for obvious regressions and cross-cutting issues (broken imports, renamed symbols, missing tests for new behavior) rather than full style passes.
   - **Offer follow-up:** if the branch is too large for one pass, say which directories or files deserve a focused second review.

9. **Report** with file-scoped findings. Say if there are no changes in scope (branch matches base, or working tree is empty). Include a **Test plan** subsection when relevant (see **Test and CI awareness**). Always include a **hidden-items rollup** at the end of the review (see **Reporting conventions** below).

10. **Severity for style/readability**: Multi-condition `unless` statements (e.g. `unless A && B`, `unless $x || $y`, chained `unless` with multiple tests) are hard to read and easy to mis-edit. When flagging them, treat them as **low priority** (polish/refactor), not correctness bugs, unless they clearly change behavior.

**Skip** the git-scope steps (resolve base branch, diff changed paths—steps **4-5**) if the user already listed exact files/paths, is clearly asking about a single known file, or **Working-tree review** applies—**but** if they used branch-review phrasing (**code review**, **review this branch**, etc.) **without** working-tree scope, **still** enforce **clean working tree**, **fetch/pull**, and optional **branch checkout** from extra message text before reviewing those paths. **Still apply step 6** and steps **7-10** to whatever path list you review.

**Still follow Reporting conventions**, including the hidden-items rollup.

## Things to look for when doing a review

1. **Understand the change** - read the files or diff to understand what the code is supposed to do. Identify the scope (new feature, bug fix, refactor). Apply **workspace rules** (step 7) before this generic list; skip checklist items that do not apply to the changed languages or frameworks.

2. **Check correctness**
   - Does the code handle edge cases (empty input, null, zero, negative numbers)?
   - Are error states handled (try/catch, error boundaries, fallback UI)?
   - Does async code handle race conditions, cancellation, and timeouts?
   - Are there off-by-one errors in loops or array access?

3. **Check maintainability**
   - Are functions focused on a single responsibility?
   - Are variable and function names descriptive?
   - Is there unnecessary duplication that should be extracted?
   - Are magic numbers replaced with named constants?
   - Is the code complexity reasonable (deeply nested conditionals, long functions)?

4. **Check performance**
   - Are there N+1 query patterns in database access?
   - Are expensive computations or API calls happening in render loops?
   - Are large lists missing virtualization or pagination?
   - Are there missing indexes for common database queries?
   - Is memoization used appropriately (not over-applied)?

5. **Check type safety** (TypeScript projects)
   - Are there `any` types that should be narrowed?
   - Are function return types explicit for public APIs?
   - Are union types handled exhaustively?

6. **Check testing**
   - Are there tests for the new/changed code?
   - Do tests cover the happy path AND error cases?
   - Are tests isolated (no shared mutable state)?
   - See **Test and CI awareness (report-only)** below for how to surface gaps and suggested commands without running tests.

7. **Provide feedback** - organize findings by severity:
   - **Must fix**: bugs, security issues, data loss risks
   - **Should fix**: performance issues, maintainability concerns
   - **Nit**: style preferences, minor suggestions

## Test and CI awareness (report-only)

**Do not run tests or CI** unless the user explicitly asks. This section is for **reporting gaps and a suggested verification plan**, not for executing the suite during review.

1. **Test coverage gaps** - call out when changed behavior appears untested or under-tested:
   - new features, bug fixes, or refactors with no corresponding test file changes
   - logic changes where existing tests do not obviously cover new branches or error paths
   - deleted or weakened tests without explanation

2. **Repo test entry points** - when suggesting verification, prefer commands documented in the repo or workspace rules (read `Makefile`, `package.json` scripts, `README`, `.cursor/rules/`, CI workflow files). Cite the **documented** command; do not invent runners. Example pattern: `make test`, `npm test`, `./t/fast_test --no-core-lib`.

3. **CI config changes** - when `.github/workflows/`, `.gitlab-ci.yml`, or similar files are in the diff, note whether workflow changes look aligned with the code changes (new steps, env vars, services, job triggers). Flag obvious mismatches; do not claim pipelines passed.

4. **Report subsection** - include a **Test plan** section in the review (see **Reporting conventions**). Omit it only when there is nothing useful to say (no test gaps and no CI files touched).

## Reporting conventions

- Be constructive - explain *why* something is a problem, not just that it is.
- **Test plan (when relevant):** After findings (or as its own short subsection), list suggested verification steps using repo-documented commands. Example:
  ```markdown
  ## Test plan
  - Run `./t/fast_test --no-core-lib t/addon/foo.t` after the query change in `lib/Addon/Foo.pm`.
  - No test gaps noted for the config-only edits.
  ```
  Do not state that tests passed or failed unless you ran them.
- **Hidden items rollup (required):** After the main findings, append a compact line stating how many findings were **not** fully expanded (and optional one-line buckets). Examples: "Hidden items: **3** (2 low-priority style, 1 speculative)" or "Hidden items: **0**".
- Treat as **hidden** when you abbreviated, deferred, summarized, or withheld full detail—but the item still matters for completeness: low-severity/readability/style, speculative risks without evidence, omissions for brevity, nice-to-have refactors out of scope, or anything nodded at without a dedicated paragraph. Multi-condition `unless` called out briefly as polish counts here unless dissected inline.
- Do **not** count pure filler (“no findings”) or tautological closings—only substantive items deserving a tally.
- Format the items so they are readable by CLI, but can be easily copied/pasted into a JIRA. Make sure to include the filename and line number so we can easily identify where to attach the review comment on a pull request.
- Do not use the en-dash character (`–`); use a hyphen (`-`) instead.
