---
name: spr-stacked-pr
description: >
  Create and manage stacked pull requests with the `git spr` CLI (ejoffe/spr), where
  each commit becomes its own PR. Use when the user wants to split work into a stack of
  small, dependent PRs, or mentions stacked diffs, stacked PRs, `git spr`, or
  commit-per-PR review. Covers creating the stack, updating it after changes, adding
  reviewers, syncing after merges, and merge guidance.
metadata:
  author: bendzae
  version: "0.1.0"
  tool: ejoffe/spr 0.17.x (https://github.com/ejoffe/spr)
---

# spr-stacked-pr

`git spr` ([ejoffe/spr](https://github.com/ejoffe/spr)) manages **stacked pull requests** on GitHub. Its model is fundamentally different from branch-per-PR tools:

> **Each commit on your branch becomes one pull request.** You work on a single branch, commit normally, and `git spr update` turns every commit into a separate PR — stacked in order, each PR based on the one below it.

```
main (trunk)
 ●  commit "Add user model"      → PR #1  (base: main)            ← bottom (merges first)
 ●  commit "Add user API"        → PR #2  (base: PR #1 branch)
 ●  commit "Add user dashboard"  → PR #3  (base: PR #2 branch)    ← top (merges last)
HEAD
```

The **bottom** of the stack is the oldest commit (closest to trunk, merges first); the **top** is the newest commit (HEAD, merges last). spr tracks which commit owns which PR by appending a hidden `commit-id: <8 hex>` trailer to each commit message on first `update` — this survives amends and rebases, so editing a commit updates the *same* PR.

There are no branches to juggle: you reorder, insert, amend, or drop PRs simply by editing commits with ordinary `git rebase`/`git commit --amend`, then re-running `git spr update`.

## When to use this skill

Use this when the user wants to:

- Split one body of work into a chain of small, independently reviewable PRs
- Create, update, or restructure a stack where **each commit is a PR**
- Add reviewers to stacked PRs, check their status, or sync after some have merged
- Work with `git spr` / ejoffe-style stacked diffs specifically

If the user's tool keeps **one branch per PR** (e.g. `gh stack`, Graphite branches, `av`), this is the wrong skill — that is a different model.

## Prerequisites

`git spr` must be installed and on `PATH` (invoked as `git spr ...`):

```bash
brew install ejoffe/tap/spr     # macOS / Linux
# or: nix profile install github:ejoffe/spr
git spr version                 # verify
```

**Authentication** — spr reuses existing GitHub credentials, in this order:

1. `GITHUB_TOKEN` environment variable, or
2. the GitHub CLI login (`gh auth login`) — spr reads `gh`'s `hosts.yml`.

If `git spr status` errors with `401 Unauthorized`, run `gh auth status` (or set `GITHUB_TOKEN`) before retrying. The token needs `repo` scope.

**Repository config** is auto-created as `.spr.yml` on first run, auto-detecting the owner/repo from the `origin` remote. No manual setup is needed for a normal `origin`-backed GitHub repo. If the trunk is not `main`, set `githubBranch` (see [Configuration](#configuration)).

## Agent rules

These rules keep every `git spr` invocation non-interactive. Following them, the core commands (`update`, `status`, `merge`, `sync`) never prompt.

1. **Disable the GitHub-star prompt before the first `git spr` call.** spr occasionally prints `enjoying git spr? [Y/n]` and waits on stdin, which hangs an agent. Suppress it once per machine by writing `stargazer: true` to `~/.spr.yml`:
   ```bash
   grep -qs '^stargazer:' ~/.spr.yml || echo 'stargazer: true' >> ~/.spr.yml
   ```
2. **Never run `git spr amend` or `git spr edit` — they open interactive pickers** (`Commit to amend [1-3]:`) and an interactive rebase that an agent cannot drive. Amend commits with **native git instead** (see [Modifying a commit](#modify-a-commit-in-the-stack)). Reserve `git spr` for `update`, `status`, `merge`, and `sync`.
3. **One commit = one PR. Make each commit a discrete, reviewable unit.** Plan the stack by dependency order *before* committing: foundational changes (models, schema, shared utils) go in **earlier (lower)** commits; dependents (API, UI, consumers) go in **later (higher)** commits. If B depends on A, A must be the same commit or an earlier one.
4. **The commit subject is the PR title; the commit body is the PR description.** Write real commit messages — they are user-facing PR content. Single-line messages produce title-only PRs.
5. **Never edit or remove the `commit-id:` trailer** spr adds to commit bodies. Removing it orphans the PR (spr creates a duplicate). Native `git commit --amend` and `git rebase` preserve it automatically — that's why they're safe.
6. **Parse status with `git spr status --text`** (`URL : title` per line) or `git spr status` with `--detail` for the status bits. Plain `git spr status` is already non-interactive.
7. **Stop after `git spr update` unless the user explicitly asks to merge.** `update` creates/updates real PRs on GitHub (an outward-facing action) — confirm intent if not already authorized. Do **not** run `git spr merge` by default: with the defaults `requireApproval: true` / `requireChecks: true`, and because GitHub forbids approving your own PR, merge will report `no mergeable pull requests found` until a human reviews. See [Merging](#merging).
8. **Tell humans to merge with `git spr merge`, not the GitHub "Merge" button.** spr writes `Do not merge manually using the UI` into each PR body for a reason: stack branches are chained, and the UI merge breaks the chain. (`git spr` itself enforces correct order.)
9. **Rebase onto trunk with `git rebase`, never `git merge`.** Merge commits corrupt the stack (each commit is its own PR; conflicts must be resolved per-commit). After fetching, `git rebase origin/<trunk>` then `git spr update`, or use `git spr sync`.
10. **Prefix a commit subject with `WIP` to skip PR creation for that commit.** Useful for in-progress work at the top of the stack; remove the prefix and re-run `update` when ready.

## Quick reference

| Task | Command |
|------|---------|
| Disable star prompt (once) | `echo 'stargazer: true' >> ~/.spr.yml` |
| Create / update all PRs in the stack | `git spr update` |
| Update only the bottom N PRs | `git spr update --count N` |
| Create PRs with reviewers | `git spr update -r alice -r bob` |
| Update without rebasing onto trunk | `git spr update --no-rebase` |
| Show stack status (human) | `git spr status` |
| Show stack status with check/approval bits | `git spr status --detail` |
| Show stack status (parseable `URL : title`) | `git spr status --text` |
| Sync local stack with remote (after merges) | `git spr sync` |
| Amend a commit (native git, see rules) | `git commit --amend` / `git rebase -i origin/<trunk>` |
| Skip PR for a commit | prefix its subject with `WIP` |
| Merge ready PRs (human / explicit ask only) | `git spr merge` |
| Start a fresh stack | `git checkout -b new_stack @{push}` |

---

## Workflows

### Create a stack from scratch

```bash
# 0. One-time: silence the star prompt so spr never blocks on stdin
grep -qs '^stargazer:' ~/.spr.yml || echo 'stargazer: true' >> ~/.spr.yml

# 1. Start from an up-to-date trunk
git checkout main
git pull

# 2. Make one commit per reviewable unit, in dependency order (bottom → top).
#    Stage deliberately so each commit is cohesive. The subject becomes the PR
#    title; the body becomes the PR description.
git add internal/models/user.go
git commit -m "Add user model" -m "Introduces the User type and its persistence layer."

git add internal/api/users.go
git commit -m "Add user API endpoints" -m "CRUD endpoints backed by the user model."

git add web/dashboard.tsx
git commit -m "Add user dashboard" -m "Frontend that consumes the user API."

# 3. Turn every commit into a stacked PR
git spr update
#   [✅✅✅✅] 3: Add user dashboard
#   [✅✅✅✅] 2: Add user API endpoints
#   [✅✅✅✅] 1: Add user model

# 4. Inspect the result
git spr status --detail
```

`git spr update` pushes a branch per commit, opens/updates one PR per commit, bases each PR on the one below it, and links them into a stack. New commits get new PRs; existing commits update their PR in place.

### Add another PR on top

Just commit on top of the stack and update again — no branch commands:

```bash
git add internal/api/audit.go
git commit -m "Add audit logging"
git spr update          # creates PR #4 on top, based on PR #3
```

### Modify a commit in the stack

To change an already-submitted commit (e.g. addressing review feedback), edit the **commit**, not a branch. **Do not use `git spr amend`/`edit`** (interactive) — use native git, then `git spr update`.

**Top commit** — amend directly:

```bash
git add internal/api/users.go
git commit --amend --no-edit      # keeps subject/body AND the commit-id trailer
git spr update                    # updates only the affected PR(s)
```

**A lower commit** — use a non-interactive rebase with `--fixup` + autosquash (avoids the interactive editor that plain `git rebase -i` opens):

```bash
# 1. Make the fix in the working tree, then create a fixup commit targeting the
#    commit that owns the PR. Find its SHA from `git log --oneline origin/main..HEAD`.
git add internal/models/user.go
git commit --fixup=<sha-of-user-model-commit>

# 2. Autosquash the fixup into its target, non-interactively
GIT_SEQUENCE_EDITOR=: git rebase -i --autosquash origin/main

# 3. Re-submit — the rebase preserved every commit-id, so PRs update in place
git spr update
```

> Why this matters: putting a fix in the wrong commit lands it in the wrong PR. Always fix the commit that owns the change. Because the `commit-id` trailer rides along through amends and rebases, spr maps the edited commit back to its existing PR.

### Reorder, insert, or drop PRs

The stack is just your commit order, so restructuring is ordinary git history editing:

```bash
GIT_SEQUENCE_EDITOR='<your-edit-script>' git rebase -i origin/main
# reorder lines to reorder PRs; delete a line to drop that PR;
# add a new commit anywhere to insert a PR at that position
git spr update
```

After `update`, spr reparents and re-bases the affected PRs automatically. A dropped commit's PR is closed on the next `update`/`sync`.

### Add or change reviewers

Reviewers are added to **newly created** PRs via `-r/--reviewer` (repeatable):

```bash
git spr update -r alice -r bob
```

For PRs that already exist, request reviewers with `gh`:

```bash
gh pr edit <number> --add-reviewer alice
```

(Or set `defaultReviewers` in `.spr.yml` to apply reviewers to every new PR — see [Configuration](#configuration).)

### Check status and parse it

```bash
# Human-readable, with the four status bits
git spr status --detail

# Machine-readable: one "URL : title" per line
git spr status --text
```

Extract the PR URLs:

```bash
git spr status --text | sed 's/ : .*//'
```

**Status bits** (shown by `status --detail` and after `update`/`merge`) are four symbols per PR, e.g. `[✅✅✅✅]`:

| Position | Meaning | ✅ | ❌ | ⌛ | ➖ |
|----------|---------|----|----|----|----|
| 1 | Checks   | pass        | failed         | pending | not required |
| 2 | Approval | approved    | not approved   | —       | not required |
| 3 | Conflicts| none        | has conflicts  | —       | — |
| 4 | Stack    | all clear   | blocked below  | —       | — |

A PR is mergeable only when all four are ✅ (or ➖). The 4th bit means every PR *below* it in the stack is also ready.

### Sync after PRs merge or trunk moves

When some PRs have merged on GitHub, or trunk has advanced:

```bash
git spr sync        # fetch trunk, rebase the remaining stack onto it, update PRs
```

`sync` pulls remote changes, rebases your local stack onto the updated trunk, and reconciles PR state. If it reports a conflict, resolve it like any rebase conflict (edit files → `git add` → `git rebase --continue`), then run `git spr update`.

### Start a new, unrelated stack

Don't pile unrelated work onto the current stack. Branch off the last pushed state:

```bash
git checkout -b another-feature @{push}
# ... commit, then git spr update
```

### Handle a rebase conflict

```bash
# A `git spr sync`/`git rebase origin/main` reported a conflict:
# 1. Resolve markers in the conflicted files
git add path/to/resolved-file
git rebase --continue            # repeat until the rebase completes
# 2. Re-submit the stack
git spr update
# If unrecoverable:
git rebase --abort
```

---

## Merging

By default merging is a **human step**, and the agent should not perform it (see Agent rule 7).

- `.spr.yml` defaults are `requireApproval: true` and `requireChecks: true`. spr will not merge a PR until it is approved and its checks pass.
- GitHub does not allow a user to approve their own PR, so an agent that authored the PRs cannot make them mergeable on its own.
- When a human (or CI) has approved and checks are green, merge **in stack order** with:
  ```bash
  git spr merge            # merges every PR that is ready, bottom-up
  git spr merge --count 1  # merge only the bottom-most ready PR
  ```
  spr finds the highest mergeable PR, combines the commits up to it into one merge, merges via the configured `mergeMethod` (default `rebase`), and closes the intermediate PRs — avoiding redundant CI runs.
- **Always merge through `git spr merge`, never the GitHub UI.** The UI breaks the chained bases between stack branches.

If the user explicitly asks the agent to merge anyway, surface the approval/checks requirement first, and only proceed if they confirm they have lowered those requirements or the PRs are genuinely ready.

---

## Commands

| Command | Aliases | Purpose | Key flags |
|---------|---------|---------|-----------|
| `git spr update` | `u`, `up` | Create/update one PR per commit | `-c/--count N`, `-r/--reviewer <user>` (repeatable), `--no-rebase`/`--nr`, `-f/--fetch`, `--nf/--no-fetch` |
| `git spr status` | `s`, `st` | Show stack + PR status | `--detail`, `-t/--text` |
| `git spr sync` |  | Fetch trunk, rebase stack, reconcile PRs | — |
| `git spr merge` |  | Merge mergeable PRs bottom-up (human step) | `-c/--count N` |
| `git spr amend` | `a` | ⚠️ Interactive picker — **avoid**; use native git | — |
| `git spr edit` | `e` | ⚠️ Interactive rebase — **avoid**; use native git | `--done`, `--abort`, `-u/--update` |
| `git spr check` |  | Run the pre-merge command from `mergeCheck` | — |
| `git spr version` |  | Show version | — |

Global flags: `--detail` (status-bit headers), `--verbose` (log git/GitHub calls), `--debug`, `--profile`.

`update` env toggles: `SPR_NOREBASE=1` (disable rebase), `SPR_FETCH` / `SPR_NOFETCH`.

---

## Configuration

### Repository config — `.spr.yml` (repo root, auto-created)

| Setting | Default | Description |
|---------|---------|-------------|
| `githubRepoOwner` / `githubRepoName` | auto | Auto-detected from the git remote |
| `githubRemote` | `origin` | Git remote to use |
| `githubBranch` | `main` | Target/trunk branch for PRs — **set this if your trunk isn't `main`** |
| `githubHost` | `github.com` | Change for GitHub Enterprise |
| `requireChecks` | `true` | Require checks to pass before merge |
| `requiredChecks` | — | List of specific check names; when set, only these are evaluated |
| `requireApproval` | `true` | Require approval before merge |
| `mergeMethod` | `rebase` | `rebase`, `squash`, or `merge` |
| `mergeQueue` | `false` | Use GitHub merge queue |
| `defaultReviewers` | — | Reviewers added to every new PR |
| `mergeCheck` | — | Command run by `git spr check` before merging |
| `showPrTitlesInStack` | `false` | List PR titles in each PR's stack section |

Example `.spr.yml`:

```yaml
githubBranch: main
requireChecks: true
requiredChecks:
  - "ci/test"
requireApproval: true
mergeMethod: squash
defaultReviewers:
  - teammate
```

### User config — `~/.spr.yml`

| Setting | Default | Description |
|---------|---------|-------------|
| `stargazer` | `false` | **Set to `true` to silence the interactive star prompt** (see Agent rule 1) |
| `createDraftPRs` | `false` | Create new PRs as drafts |
| `preserveTitleAndBody` | `false` | Don't overwrite PR title/body from the commit on update |
| `deleteMergedBranches` | `false` | Delete stack branches after their PRs merge |
| `shortPRLink` | `false` | Show `PR-<number>` instead of the full URL |
| `branchPrefix` | `spr` | Prefix for spr-managed branch names |

---

## Gotchas

- **Hang on `enjoying git spr? [Y/n]`** → you skipped Agent rule 1. Set `stargazer: true` in `~/.spr.yml`.
- **`unable to auto configure repository owner`** → no GitHub `origin` remote, or run outside a git repo. Add the remote or set `githubRepoOwner`/`githubRepoName` in `.spr.yml`.
- **`401 Unauthorized` / `error: 401`** → no valid token. `gh auth login` or export `GITHUB_TOKEN`.
- **`no mergeable pull requests found`** → PRs aren't approved/green (expected for agent-authored PRs). This is normal; leave merging to a human.
- **A duplicate PR appeared after editing a commit** → the `commit-id` trailer was stripped (e.g. by a `--amend` that rewrote the whole body, or a squash that dropped it). Restore the original `commit-id:` line in the commit body and re-run `update`.
- **Stack looks scrambled after a manual UI merge** → someone merged via the GitHub button. Run `git spr sync` to rebuild the chain; in future merge with `git spr merge`.
- **Commit didn't become a PR** → its subject starts with `WIP`. Remove the prefix and re-run `update`.
