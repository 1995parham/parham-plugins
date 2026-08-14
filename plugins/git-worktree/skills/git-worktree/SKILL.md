---
name: git-worktree
description: Keep concurrent, unrelated changes in one repository in separate git worktrees instead of stashing, stacking, or branch-hopping on a single checkout. Use when a second task arrives while the working tree is already dirty, when asked to work on several features/fixes/reviews at once, when a hotfix or PR review must not disturb work in progress, when a long build/test run must keep owning the checkout, or when parallel agents would otherwise edit the same files. Also covers listing, moving, repairing, pruning, and cleaning up worktrees.
---

# git worktree — one repo, many checkouts

Use this skill the moment a repository has **more than one change in flight**. The
default reflex — `git stash`, or committing a half-finished WIP so you can jump
branches — serializes work that is naturally parallel and loses context every
time you switch. A worktree gives each change its own directory, its own index,
and its own HEAD, all backed by a single `.git` object store.

## Mental model — read this first

- **A worktree is an extra checkout, not an extra clone.** `git worktree add`
  creates a directory whose `.git` is a *file* pointing back into
  `<main-repo>/.git/worktrees/<name>`. Objects, packs, and most refs are shared,
  so a worktree costs working-tree bytes only — not another copy of history.
- **Per worktree:** `HEAD`, the index, the working tree, `MERGE_HEAD`/rebase and
  bisect state, `refs/bisect/*`, `refs/worktree/*`, and (only if
  `extensions.worktreeConfig=true`) `git config --worktree` values.
- **Shared across every worktree:** objects, `refs/heads/*`, `refs/remotes/*`,
  tags, hooks (`.git/hooks`), repo `config`, and — the surprising one —
  **`refs/stash`**. A stash pushed in one worktree shows up in `git stash list`
  in all of them.
- **A branch can be checked out in exactly one worktree at a time.** A second
  attempt fails with `fatal: '<branch>' is already used by worktree at <path>`.
  This is a feature: it makes "which directory is that branch in?" answerable by
  `git worktree list`.
- **Untracked and ignored files do not come along.** A fresh worktree has no
  `.env`, no `node_modules`, no `.venv`, no `vendor/`, no `target/`, no
  `.direnv`. Budget one setup step per worktree (see
  [Bootstrapping a new worktree](#bootstrapping-a-new-worktree)).
- **The main checkout is just a worktree too.** It's the first row of
  `git worktree list`, and nothing else about it is special.

## When to reach for a worktree

| Situation | Do this | Why |
|---|---|---|
| Second unrelated task arrives, working tree is dirty | **New worktree** | No stash, no WIP commit; the in-progress change keeps its exact state |
| Urgent hotfix on `main` while mid-feature | **New worktree off `origin/main`** | Fix, test, and push without touching the feature checkout |
| Reviewing a colleague's PR / Gerrit CL | **New worktree** (see [Review worktrees](#review-worktrees)) | Build and run their branch while yours stays live |
| Long build, test suite, or dev server pinning the checkout | **New worktree for the edits** | The running process keeps a stable tree |
| Comparing two branches by running both | **One worktree each** | Two servers on two ports, side by side |
| Several agents editing files in parallel | **One worktree per agent** | Concurrent writes never collide |
| Bisecting while continuing to work | **New worktree for the bisect** | Bisect state is per-worktree |
| Same branch, one small fix, tree is clean | **Just commit** | A worktree would be pure overhead |
| Quick read-only peek at another branch | `git show <branch>:<file>` / `git log` | No checkout needed at all |
| A stack of dependent commits on one topic | **One branch, one worktree** | Dependent work is not parallel work |

Rule of thumb: **independent changes → separate worktrees; dependent changes →
one worktree.** If two pieces of work would sensibly land in two different pull
requests, they deserve two worktrees.

## Layout and naming

Two workable placements — pick one per repo and stay consistent:

```bash
# A. Siblings (preferred for hand-driven work)
~/src/myrepo/                    # main checkout, on main
~/src/myrepo-fix-login/          # branch fix/login
~/src/myrepo-bump-deps/          # branch chore/bump-deps

# B. Nested under the repo (keeps everything in one folder)
~/src/myrepo/
~/src/myrepo/.worktrees/fix-login/
```

Siblings are the safer default: nested worktrees get scanned by language
servers, linters, test runners, and file watchers, which produces duplicate
symbols and phantom test failures. If you nest anyway, **add the directory to
`.gitignore` and to every tool's ignore list** (`.eslintignore`, `ruff`/`mypy`
excludes, `go` build tags are not enough — use directory excludes).

Naming convention: directory basename mirrors the branch topic, so
`git worktree list` reads like a to-do list.

```bash
git worktree add ../myrepo-fix-login -b fix/login origin/main
```

## Core commands

```bash
# Create: new branch off an explicit base (the common case)
git worktree add ../myrepo-<topic> -b <branch> origin/main

# Create: check out a branch that already exists
git worktree add ../myrepo-<topic> <existing-branch>

# Create: no branch name — git DWIMs a branch named after the directory
git worktree add ../myrepo-spike          # -> new branch "myrepo-spike"

# Create: detached, for a tag/SHA or a branch already checked out elsewhere
git worktree add --detach ../myrepo-check v1.4.2

# Inventory
git worktree list
git worktree list --porcelain             # machine-readable; marks "prunable"

# Tear down (refuses if dirty; --force overrides)
git worktree remove ../myrepo-<topic>
git branch -d <branch>                    # remove does NOT delete the branch

# Housekeeping
git worktree prune -v                     # after a manual rm -rf
git worktree move <from> <to>             # never plain `mv` a worktree
git worktree repair                        # after the repo or worktree was moved
git worktree lock <path> --reason "on external drive"
```

`git fetch`, `git push`, and remote config are shared — fetch once from any
worktree and every other one sees the new `origin/*` refs.

### Review worktrees

Reviewing someone else's branch is the highest-value worktree use, because it is
inherently a *second* concurrent change:

```bash
git fetch origin
git worktree add --detach ../myrepo-review origin/pr-branch   # detached: no branch lock
# ... build, run, read ...
git worktree remove --force ../myrepo-review
```

For GitHub: `gh pr checkout <N>` inside a dedicated review worktree. For Gerrit,
fetch the patch-set ref first, then `git worktree add ../myrepo-review-<CL>
FETCH_HEAD` (see the `gerrit` skill for finding the ref).

## Bootstrapping a new worktree

A fresh worktree contains **only tracked files**. Whatever the repo needs that is
gitignored has to be recreated. Do this immediately after `add`, before
concluding "the build is broken":

```bash
NEW=../myrepo-<topic>
cp .env .env.local "$NEW"/ 2>/dev/null      # secrets/config — never tracked
git submodule update --init --recursive     # run *inside* the new worktree
```

Then the ecosystem step, run inside the new worktree:

| Stack | Command | Notes |
|---|---|---|
| Node | `npm ci` / `pnpm install` | pnpm's content-addressed store makes this cheap; npm re-copies everything |
| Python | `uv sync` / `python -m venv .venv && pip install -e .` | Never symlink one `.venv` across worktrees — paths are baked in |
| Go | nothing | Module and build caches are global (`GOMODCACHE`, `GOCACHE`) |
| Rust | nothing required | `target/` rebuilds; `CARGO_TARGET_DIR` shared across worktrees saves the most time here |
| Java/Gradle | nothing | Gradle cache is in `~/.gradle` |
| direnv | `direnv allow` | Each new directory needs its own approval |
| pre-commit | `pre-commit install` not needed | Hooks live in the shared `.git/hooks` |

If this bootstrap is more than two commands, put it in a repo script
(`just worktree <topic>`, `scripts/new-worktree.sh`) so it's one step.

## Living entirely in worktrees (bare-repo layout)

For repos where you *always* have several branches open, skip the privileged
main checkout — clone bare and make every branch a worktree:

```bash
mkdir myrepo && cd myrepo
git clone --bare git@github.com:org/myrepo.git .bare
echo "gitdir: ./.bare" > .git
git config remote.origin.fetch '+refs/heads/*:refs/remotes/origin/*'   # bare clones omit this
git fetch origin
git worktree add main main
git worktree add feat-x -b feat/x origin/main
```

The `remote.origin.fetch` line is mandatory — without it a bare clone never
populates `refs/remotes/origin/*`, and every `git worktree add ... origin/main`
fails with "invalid reference".

## Using worktrees inside Claude Code

Claude Code has first-class support; prefer it over raw `git worktree add` when
the session itself should move into the worktree.

- **`EnterWorktree`** creates a worktree under `.claude/worktrees/` on a new
  branch and switches the session's working directory into it. The base ref
  follows the `worktree.baseRef` setting: `fresh` (default) branches from
  `origin/<default-branch>`; `head` branches from the current local HEAD. Pass
  `path` instead of `name` to step into a worktree that already exists.
  This tool is gated on explicit instruction — **invoking this skill is that
  instruction**, so it's fair game for the workflows described here.
- **`ExitWorktree`** returns the session to the original directory:
  `action: "keep"` leaves the directory and branch on disk (use when the work
  continues later), `action: "remove"` deletes both. `remove` refuses while
  uncommitted work or unmerged commits exist unless `discard_changes: true` —
  read what it lists and confirm with the user before forcing.
- **Subagents:** pass `isolation: "worktree"` to the `Agent` tool (or to
  `agent()` in a workflow) when several agents will mutate files concurrently.
  Each gets a private worktree, auto-removed if left unchanged. It costs real
  setup time and disk per agent, so use it only for parallel *writers* — never
  for read-only explorers.

Because `.claude/worktrees/` is nested inside the repo, the ignore-list warning
from [Layout and naming](#layout-and-naming) applies: keep it out of linter and
watcher scopes.

## Cleanup discipline

Worktrees are cheap to create and easy to forget. Two rules:

1. **Remove the worktree when its branch merges** — `git worktree remove <path>`
   then `git branch -d <branch>`. `remove` never touches the branch.
2. **Audit periodically.** `git worktree list` is the whole audit; anything you
   can't explain in a sentence is stale.

```bash
# Show every worktree that no longer has a directory on disk
git worktree list --porcelain | grep -B2 prunable
git worktree prune -v
```

## Pitfalls — quick reference

| Symptom | Cause | Fix |
|---|---|---|
| `fatal: '<branch>' is already used by worktree at <path>` | A branch may only be checked out once | `git worktree list` to find it and work there, or use `--detach`, or make a new branch |
| New worktree won't build — missing config, modules, or venv | Untracked/ignored files aren't copied | Copy `.env`, re-run the install step ([Bootstrapping](#bootstrapping-a-new-worktree)) |
| `git worktree remove` → `contains modified or untracked files` | Uncommitted work would be lost | Inspect it, then commit/stash, or `--force` once you're sure |
| Branch still around after removing its worktree | `remove` deletes the checkout, not the ref | `git branch -d <branch>` (or `-D` if unmerged and intended) |
| Deleted the directory with `rm -rf`; `git worktree list` still shows it as `prunable` | Admin metadata under `.git/worktrees/` survives | `git worktree prune -v` |
| Worktree broken after moving or renaming the repo | Absolute paths in the `gitdir` link files | `git worktree repair` (run from the main repo, and/or from the worktree) |
| Moving a worktree with `mv` breaks it | Same absolute-path link | Use `git worktree move`, or `mv` then `git worktree repair` |
| `git stash pop` restores someone else's WIP | `refs/stash` is **shared** across worktrees | Prefer worktrees over stashes; if you must stash, name it (`git stash push -m`) and pop by index |
| LSP/linter reports duplicate symbols; tests run twice | Nested worktree is being scanned as project source | Move it to a sibling directory, or add it to `.gitignore` **and** the tool's exclude list |
| Submodule directories are empty | `worktree add` doesn't init submodules | `git submodule update --init --recursive` inside the new worktree |
| A hook change appears in every worktree | `.git/hooks` is shared | Expected; use `core.hooksPath` per worktree config only if you truly need divergence |
| `git worktree add ../x origin/main` → `invalid reference` in a bare-repo layout | Bare clones have no `remote.origin.fetch` refspec | Set the refspec and re-fetch (see [bare-repo layout](#living-entirely-in-worktrees-bare-repo-layout)) |
| Rebase/merge conflict state seems to vanish | In-progress operation state is per-worktree | Finish (or `--abort`) in the worktree that started it |
| Disk fills up fast | `node_modules`/`target`/`.venv` duplicated per worktree | Share caches (`CARGO_TARGET_DIR`, pnpm store), and remove merged worktrees promptly |

## Conventions

1. **One in-flight change, one worktree.** If it would be its own pull request,
   give it its own directory.
2. **Never stash to make room for a second task** — create a worktree instead.
   Stashes are shared repo-wide and lose their context within a day.
3. **Always name the base ref explicitly** (`-b <branch> origin/main`), so a new
   worktree never inherits whatever HEAD happened to be.
4. **Directory basename mirrors the branch topic**, so `git worktree list` is a
   readable inventory.
5. **Bootstrap immediately after `add`** — env files, submodules, dependencies —
   before deciding anything is broken.
6. **Prefer siblings to nesting**; when nesting, ignore the directory in git and
   in every scanning tool.
7. **Remove the worktree and delete the branch when the work merges**;
   `git worktree prune` after any manual deletion.
8. **Confirm before `remove --force` / `ExitWorktree` with `discard_changes`** —
   both destroy uncommitted work.
