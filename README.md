# parham-plugins

A [Claude Code](https://docs.claude.com/en/docs/claude-code) plugin marketplace. Currently ships four plugins:

- **`gopass`** — work with the [gopass](https://www.gopass.pw/) password store: show / insert / generate / move / search secrets, manage GPG recipients across single- and multi-store setups, handle git-backed sync, recover from wedged merges, and resolve `gopass://` cross-secret references. Includes a pitfalls table covering invalid-key warnings, misleading recipient listings, full-fingerprint requirements, and trailing-newline traps.
- **`go-projects`** — scaffold and grow production-grade Go CLIs using the [koochooloo](https://github.com/1995parham/koochooloo) layout (`cmd/<bin>/main.go` + `internal/cmd` + `internal/infra` + `internal/domain`), [urfave/cli v3](https://cli.urfave.org/), [koanf v2](https://github.com/knadh/koanf) config, `justfile`, `golangci-lint` v2 (`default: all`), and the koochooloo GitHub Actions setup. Codifies modern Go defaults (Go 1.26, `log/slog`, generics over `any`, `pgx`/`echo`/`fx` for services) and the pitfalls that come with them.
- **`gerrit`** — work with [Gerrit](https://www.gerritcodereview.com/) code review over its SSH API and the `refs/changes` git workflow: fetch and check out a change by CL number, query the change list, read change-level and inline review comments (`gerrit query --comments --patch-sets` with a `jq` formatter), cast `Code-Review`/`Verified` votes, and manage reviewers. Includes a pitfalls table covering the missing-`Change-Id` hook, the `--patch-sets` requirement for inline comments, the streaming-JSON `stats` row, and the `29418` SSH port.
- **`git-worktree`** — keep concurrent, unrelated changes in one repository in separate [git worktrees](https://git-scm.com/docs/git-worktree) instead of stashing, stacking, or branch-hopping on a single checkout: a decision table for worktree vs. stash vs. plain commit, layout and naming conventions, review worktrees, the per-worktree bootstrap step (env files, submodules, dependencies), the bare-repo layout, and Claude Code's `EnterWorktree` / `ExitWorktree` / subagent `isolation: "worktree"` integration. Includes a pitfalls table covering the one-worktree-per-branch lock, the **shared** `refs/stash`, `prune` after a manual `rm -rf`, `repair` after a move, and nested worktrees confusing linters.

## Install

Add the marketplace once, then install the plugin(s) you want from it:

```shell
/plugin marketplace add 1995parham/parham-plugins
/plugin install gopass@parham-plugins
/plugin install go-projects@parham-plugins
/plugin install gerrit@parham-plugins
/plugin install git-worktree@parham-plugins
```

Update later with:

```shell
/plugin marketplace update parham-plugins
```

All skills auto-activate when Claude detects relevant work in the conversation:

- `gopass` triggers on secrets, recipients, the password store, or any `gopass` command. Invoke explicitly with `/gopass:gopass`.
- `go-projects` triggers when scaffolding a Go CLI, adding a subcommand, wiring config, or making library-choice decisions for a Go project. Invoke explicitly with `/go-projects:go-projects`.
- `gerrit` triggers on a CL or change number, a patch set, `Change-Id`, or any request to pull / review / approve a change or read its comments. Invoke explicitly with `/gerrit:gerrit`.
- `git-worktree` triggers when a second task arrives while the working tree is dirty, when several features / fixes / reviews are in flight at once, or when parallel agents would edit the same checkout. Invoke explicitly with `/git-worktree:git-worktree`.

## Repository layout

```
.
├── .claude-plugin/
│   └── marketplace.json              # marketplace catalog
├── plugins/
│   ├── gopass/
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json           # plugin manifest
│   │   └── skills/
│   │       └── gopass/
│   │           └── SKILL.md          # the skill content
│   ├── go-projects/
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json
│   │   └── skills/
│   │       └── go-projects/
│   │           └── SKILL.md
│   ├── gerrit/
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json
│   │   └── skills/
│   │       └── gerrit/
│   │           └── SKILL.md
│   └── git-worktree/
│       ├── .claude-plugin/
│       │   └── plugin.json
│       └── skills/
│           └── git-worktree/
│               └── SKILL.md
├── LICENSE
└── README.md
```

## What each plugin covers

### `gopass`

Content: [`plugins/gopass/skills/gopass/SKILL.md`](plugins/gopass/skills/gopass/SKILL.md).

- Mental model: secrets are files, stores are directories with their own recipient list and git remote
- Common commands: `show` / `insert` / `generate` / `find` / `mv` / `rm` / `audit` / `fsck`
- Secret formats: Key-Value (default), YAML (legacy), Plain
- `gopass://` references and `core.follow-references`
- Multi-store mounts
- Recipient lifecycle, including the "delete expired key locally" trick that avoids a store-wide re-encryption
- Sync hygiene and merge-conflict recovery
- TOTP, audit, env injection, environment templates

### `go-projects`

Content: [`plugins/go-projects/skills/go-projects/SKILL.md`](plugins/go-projects/skills/go-projects/SKILL.md).

- Opinionated defaults table — Go version, logger, CLI framework, config, HTTP framework, Postgres driver, metrics, tracing, DI, linter, task runner, CI
- Reference project: [1995parham/koochooloo](https://github.com/1995parham/koochooloo); small CLI example: [1995parham/natsie](https://github.com/1995parham/natsie)
- Repository skeleton following [golang-standards/project-layout](https://github.com/golang-standards/project-layout) with the `domain` / `infra` / `cmd` import direction rules
- Modern Go (1.21–1.26): `log/slog`, `slices`/`maps`/`cmp`, `errors.Join`, `sync.OnceValue`, `iter.Seq`, `synctest`, generic type aliases, `tool` directive, `new(expr)`, `errors.AsType[T]`, `go fix` modernizers
- urfave/cli v3 specifics (v2 → v3 migration gotchas, env-var sources, `cmd.IsSet`) plus the cobra alternative koochooloo uses
- koanf v2 layered loading (defaults → file → env → flags) and sub-module versioning pitfalls
- `justfile`, `.golangci.yml` (`default: all`, line-level `//nolint:<linter> // <why>` discipline), and the koochooloo GitHub Actions workflows (mandatory `lint` job, CodeQL, Dependabot)
- HTTP service stack (echo/fiber, pgx vs GORM, prometheus, OpenTelemetry, fx) and when to break each default
- Pitfalls table covering sub-module versioning, `cmd.Int` vs `int64`, config defaults disappearing, lint suppression hygiene, `go.mod` toolchain drift, and more

### `gerrit`

Content: [`plugins/gerrit/skills/gerrit/SKILL.md`](plugins/gerrit/skills/gerrit/SKILL.md).

- Mental model: a change is one commit (identified by `Change-Id` or CL number); each push is a new patch set exposed as `refs/changes/<NN>/<change>/<patchset>`; review is label votes, not comments
- Setup: SSH access, the `commit-msg` hook, and `~/.ssh/config` with port `29418`
- Fetch & check out a change: discover the latest patch set with `git ls-remote "refs/changes/*/<CL>/*"`, then `git fetch` + `review/<CL>` branch
- Query the change list with `gerrit query` operators and output flags (`--format JSON`, `--current-patch-set`, `--patch-sets`, `--comments`, ...)
- The headline workflow — reading comments: cover messages at top-level `.comments[]`, inline file comments under `.patchSets[].comments[]`, with a colorized `jq` formatter and the `--patch-sets` requirement
- Voting (`gerrit review --code-review/--verified --submit`) and reviewer management (`gerrit set-reviewers`)
- The helper-wrapper subcommand pattern (`pull` / `comments` / `approve` / `assign` / `mine` / `to-review` / `list`)
- Pitfalls table: missing `Change-Id`, inline comments needing `--patch-sets`, `jq` null-iteration guards, the streaming `stats` row, line-less file comments, wrong SSH port, and per-version option drift

### `git-worktree`

Content: [`plugins/git-worktree/skills/git-worktree/SKILL.md`](plugins/git-worktree/skills/git-worktree/SKILL.md).

- Mental model: a worktree is an extra checkout, not an extra clone — what is per-worktree (`HEAD`, index, working tree, rebase/bisect state) vs. shared (objects, branches, remotes, hooks, config, and `refs/stash`)
- Decision table: worktree vs. stash vs. plain commit vs. `git show`, with the rule "independent changes → separate worktrees; dependent changes → one worktree"
- Layout and naming: siblings vs. nesting, and why nested worktrees confuse language servers, linters, and watchers
- Core commands: `add` (with `-b <branch> origin/main`, `--detach`, the directory-name DWIM), `list --porcelain`, `remove`, `prune`, `move`, `repair`, `lock`
- Review worktrees for PRs and Gerrit CLs, using `--detach` to dodge the branch lock
- The per-worktree bootstrap step — env files, submodules, and a per-ecosystem table (Node / Python / Go / Rust / Gradle / direnv)
- The bare-repo layout, including the mandatory `remote.origin.fetch` refspec fixup
- Claude Code integration: `EnterWorktree` / `ExitWorktree` (`worktree.baseRef`, keep vs. remove, `discard_changes`) and subagent `isolation: "worktree"` for parallel writers
- Cleanup discipline and pitfalls table: the one-worktree-per-branch lock, missing untracked files, dirty-removal refusal, branches surviving `remove`, `prunable` entries, `repair` after a move, shared stashes, empty submodules, and disk blowup from duplicated dependency trees

## Validate locally

Before pushing changes to the marketplace:

```bash
claude plugin validate .
```

## License

MIT — see [`LICENSE`](LICENSE).
