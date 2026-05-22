# parham-plugins

A [Claude Code](https://docs.claude.com/en/docs/claude-code) plugin marketplace. Currently ships two plugins:

- **`gopass`** — work with the [gopass](https://www.gopass.pw/) password store: show / insert / generate / move / search secrets, manage GPG recipients across single- and multi-store setups, handle git-backed sync, recover from wedged merges, and resolve `gopass://` cross-secret references. Includes a pitfalls table covering invalid-key warnings, misleading recipient listings, full-fingerprint requirements, and trailing-newline traps.
- **`go-projects`** — scaffold and grow production-grade Go CLIs using the [koochooloo](https://github.com/1995parham/koochooloo) layout (`cmd/<bin>/main.go` + `internal/cmd` + `internal/infra` + `internal/domain`), [urfave/cli v3](https://cli.urfave.org/), [koanf v2](https://github.com/knadh/koanf) config, `justfile`, `golangci-lint` v2 (`default: all`), and the koochooloo GitHub Actions setup. Codifies modern Go defaults (Go 1.26, `log/slog`, generics over `any`, `pgx`/`echo`/`fx` for services) and the pitfalls that come with them.

## Install

Add the marketplace once, then install the plugin(s) you want from it:

```shell
/plugin marketplace add 1995parham/parham-plugins
/plugin install gopass@parham-plugins
/plugin install go-projects@parham-plugins
```

Update later with:

```shell
/plugin marketplace update parham-plugins
```

Both skills auto-activate when Claude detects relevant work in the conversation:

- `gopass` triggers on secrets, recipients, the password store, or any `gopass` command. Invoke explicitly with `/gopass:gopass`.
- `go-projects` triggers when scaffolding a Go CLI, adding a subcommand, wiring config, or making library-choice decisions for a Go project. Invoke explicitly with `/go-projects:go-projects`.

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
│   └── go-projects/
│       ├── .claude-plugin/
│       │   └── plugin.json
│       └── skills/
│           └── go-projects/
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

## Validate locally

Before pushing changes to the marketplace:

```bash
claude plugin validate .
```

## License

MIT — see [`LICENSE`](LICENSE).
