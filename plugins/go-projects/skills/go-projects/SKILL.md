---
name: go-projects
description: Build production-grade Go CLI projects — koochooloo-style layout (cmd/<bin>/main.go + internal/cmd + internal/infra + internal/domain), urfave/cli v3, koanf v2 config, justfile, golangci-lint, and GitHub Actions. Captures pitfalls learned in the wild.
---

# Go projects (CLI + config)

Use this skill when you are scaffolding a new Go CLI, adding a subcommand,
wiring configuration, or making library-choice decisions for any of those.
Defaults below are opinionated and reflect real production decisions, not
survey-style "best practices."

The canonical reference project is **[1995parham/koochooloo](https://github.com/1995parham/koochooloo)**.
When the recipe below contradicts your gut, mirror koochooloo — it's the
load-bearing example.

## Defaults

| Decision | Pick | Reason |
| --- | --- | --- |
| CLI framework | `github.com/urfave/cli/v3` | Stdlib `context.Context` plumbing built in; smaller surface than cobra; no codegen. (koochooloo uses cobra; both are acceptable, but new repos pick urfave v3.) |
| Config | `github.com/knadh/koanf/v2` | Provider/parser split keeps the binary small. Layers defaults → file → env → flags cleanly. |
| Config format | TOML (koochooloo) or YAML | TOML matches the koochooloo precedent; YAML is fine for human-edited config. Both via koanf parsers. |
| Project layout | `cmd/<binary>/main.go` + `internal/cmd` + `internal/infra` + `internal/domain` | Entrypoint thin; CLI tree + infra adapters + domain logic each in their own tree. |
| Version stamp | `github.com/carlmjohnson/versioninfo` | `--version` shows VCS commit + dirty flag with zero ldflags. |
| Linter | `golangci-lint` v2 with `default: all` + targeted disables | Fail fast on the lint server. Disables list ports straight from koochooloo. |
| Task runner | `justfile` | Discoverable (`just --list`), no Makefile-isms, ships with one binary. |
| CI | GitHub Actions: lint, test, build, codeql, dependabot | Mirrors koochooloo's `.github/`. |
| License | Apache-2.0 (`LICENSE` at repo root) | Standard for new OSS Go tooling; matches NATS / Synadia / HashiCorp ecosystem. |
| Module path | `github.com/<owner>/<repo>` | Matches the canonical Go tooling assumption; avoids vanity-URL maintenance. |

## Repository skeleton

```
<repo>/
├── LICENSE
├── README.md
├── .gitignore                       # /<repo>, /dist/, *.test, *.out, coverage.*, .DS_Store, /.idea, /.vscode
├── .golangci.yml                    # linter config (see below)
├── justfile                         # build/test/lint/tidy/update recipes
├── go.mod
├── go.sum
├── cmd/
│   └── <binary>/
│       └── main.go                  # 5-line entrypoint that calls internal/cmd.Execute()
├── internal/
│   ├── cmd/                         # CLI command tree (urfave/cli v3 *Command)
│   │   ├── root.go                  # root command, wires subcommand groups
│   │   └── <group>/                 # one package per subcommand group
│   │       ├── <group>.go           # returns *cli.Command containing leaf commands
│   │       └── <verb>.go            # one file per leaf command
│   ├── domain/                      # pure logic — no external I/O
│   │   ├── model/                   # entity types (rarely needed in CLI tools)
│   │   └── <area>/                  # domain services, classifiers, validators
│   └── infra/                       # adapters that talk to the outside world
│       ├── config/                  # koanf wrapper
│       ├── logger/                  # zap setup (when needed)
│       └── <client>/                # external API clients, DB drivers, etc.
├── api/                             # OpenAPI / Proto / k6 scripts (when relevant)
├── configs/                         # config.example.toml
├── deployments/                     # docker-compose.yml, k8s/, helm/ (when relevant)
├── build/package/                   # Dockerfile, docker-bake.json (when relevant)
├── docs/                            # ROADMAP.md, architecture notes
└── .github/
    ├── workflows/
    │   ├── test.yaml                # lint + test (codecov) + build
    │   └── codeql.yml               # security scan, weekly schedule
    └── dependabot.yml               # gomod + github-actions weekly
```

Drop `api/`, `configs/`, `deployments/`, `build/package/` for a CLI-only
project. Keep them as soon as you ship a server, container, or config file.

## The thin entrypoint

`cmd/<binary>/main.go` should be small enough to read in one breath:

```go
package main

import "github.com/<owner>/<repo>/internal/cmd"

func main() {
    cmd.Execute()
}
```

`internal/cmd/root.go` does the work — it builds the command tree, registers
subcommand groups (one factory per group), prints VCS info, and calls Run.
Keeping `main` thin is what lets you test the command wiring from another
binary or from `go test`.

## The domain / infra / cmd split

The three internal trees have **strict directionality**:

- `internal/cmd` may import `internal/domain` *and* `internal/infra`.
- `internal/infra` may import `internal/domain` (to fulfill domain interfaces).
- `internal/domain` may import *nothing else in the project*.

If `internal/domain` ends up importing `github.com/nats-io/...`, you've
mis-classified — that file belongs in `internal/infra/<name>`. The litmus
test: can this package be unit-tested with no network, no filesystem, no
clock, no external client? If yes, it's domain.

For tiny CLIs (one verb, no business logic) you can collapse this to just
`internal/cmd` + `internal/infra` — see "When to break the defaults" below.

## urfave/cli v3 — the parts that actually matter

v3 is a breaking rewrite of v2. Notable shifts:

- `cli.NewApp()` is gone. Use `&cli.Command{...}` as the root.
- `*cli.Context` is gone. Action signature is `func(ctx context.Context, cmd *cli.Command) error`.
- `Subcommands` field renamed to `Commands`.
- Hooks: `BeforeFunc = func(context.Context, *cli.Command) (context.Context, error)`.

Read flag values via methods on the `*cli.Command` passed to the action:
`cmd.String("flag")`, `cmd.Int("flag")`, `cmd.Duration("flag")`, `cmd.Bool("flag")`.
Check whether a flag was explicitly set with `cmd.IsSet("flag")` — this is the
hook that lets config defaults coexist with per-invocation overrides.

Wire env-var fallback declaratively:

```go
&cli.StringFlag{
    Name:    "config",
    Aliases: []string{"c"},
    Usage:   "path to a config file",
    Sources: cli.EnvVars("APP_CONFIG"),
}
```

Mark required at declaration time (avoid runtime checks):

```go
&cli.StringFlag{Name: "context", Required: true, ...}
```

Subcommand group pattern — one factory function per group, leaf commands as
unexported helpers in the same package:

```go
// internal/cmd/<group>/<group>.go
func Command() *cli.Command {
    return &cli.Command{
        Name:  "<group>",
        Usage: "...",
        Commands: []*cli.Command{
            scanCommand(),
            applyCommand(),
        },
    }
}
```

`int` flag caveat: `cmd.Int(...)` returns plain `int`, not `int64`. Cast at
the boundary if the consuming struct uses `int64`.

### cobra alternative

koochooloo uses cobra with a `<group>.Register(root)` pattern instead of
a `Command()` factory:

```go
// internal/cmd/server/main.go
func Register(root *cobra.Command) {
    root.AddCommand(&cobra.Command{ ... })
}
```

Both patterns are fine; pick one per repo and stay consistent.

## koanf v2 — the parts that actually matter

The provider/parser split is the *whole* design. The core module knows nothing
about YAML/TOML/env vars/files — you compose it. That means `go mod tidy` only
pulls what you actually use.

Layered load order: **defaults (via `structs.Provider`) → file → env → flags**.
Last `Load`/`Merge` wins.

```go
k := koanf.New(".")
_ = k.Load(structs.Provider(Default(), "koanf"), nil)   // hardcoded defaults
_ = k.Load(file.Provider("config.toml"), toml.Parser()) // file overrides
_ = k.Load(env.Provider(".", env.Opt{                   // env overrides
    Prefix: "APP_",
    TransformFunc: func(key, val string) (string, any) {
        return strings.ReplaceAll(
            strings.ToLower(strings.TrimPrefix(key, "APP_")), "__", "."), val
    },
}), nil)

var cfg Config
_ = k.Unmarshal("", &cfg)
```

The koochooloo pattern uses `__` (double underscore) as the env-var nesting
separator so single underscores can appear in field names. Use it if your
config has snake_case keys.

### Pitfalls

1. **Sub-module versioning.** `github.com/knadh/koanf/providers/env/v2` is a
   separate Go module from `github.com/knadh/koanf/v2`. `go mod tidy` won't
   discover it on its own — `go get github.com/knadh/koanf/providers/env/v2@v2.0.0`
   explicitly. Same applies to any provider that has a `/v2` suffix; the
   non-`/v2` providers (`file`, `parsers/yaml`, `parsers/toml/v2`) live where
   their published path says and are picked up automatically.
2. **`@latest` lies for sub-modules.** Asking for `@latest` on the parent
   module won't resolve the `/v2`-suffixed sub-modules — Go's module proxy
   treats them as different modules. Pin a version.
3. **Don't fight the provider for flag merging.** koanf has `posflag`/`basicflag`
   providers, but they assume `pflag`/stdlib `flag`. With urfave/cli v3 it's
   cleaner to load defaults+file+env into koanf, then in the Action apply CLI
   overrides by reading `cmd.IsSet("name")` and `cmd.String("name")` etc.
   directly into the same struct.

Config struct: lean on `koanf:` tags. koochooloo also adds `json:` for the
"loaded configuration" debug print:

```go
type Config struct {
    Defaults Defaults                  `json:"defaults" koanf:"defaults"`
    Contexts map[string]ContextOptions `json:"contexts" koanf:"contexts"`
}
```

Hardcoded defaults belong in a `Default()` function fed via `structs.Provider`,
not on the struct literal at Unmarshal time — that way `--print-config` shows
exactly the same shape no matter where a value came from.

## Tooling

### justfile

Default `just` invocation lists recipes:

```just
default:
    @just --list

build:
    @echo '{{ BOLD + CYAN }}Building <binary>{{ NORMAL }}'
    go build -o <binary> ./cmd/<binary>

update:
    @cd ./cmd/<binary> && go get -u

test:
    go test -race -v ./... -covermode=atomic -coverprofile=coverage.out

lint:
    golangci-lint run -c .golangci.yml

tidy:
    go mod tidy
```

For server/daemon projects, add the koochooloo `dev` recipe that drives
`docker compose -f deployments/docker-compose.yml` and a `seed`/`database`
recipe that runs migrations against the dev environment.

### .golangci.yml

Turn on every linter and disable a small, deliberate set that historically
produced churn:

```yaml
---
version: "2"
linters:
  default: all
  disable:
    - gomodguard
    - wsl
    - depguard
    - tagliatelle
    - nolintlint
    - varnamelen
    - ireturn
    - revive
    - wrapcheck
    - noinlineerr
  settings:
    wrapcheck:
      ignore-sigs:
        - .Errorf(
        - errors.New(
        - errors.Unwrap(
        - .Wrap(
        - .Wrapf(
    wsl_v5:
      allow-first-in-block: true
      allow-whole-block: false
      branch-max-lines: 2
```

Reasoning: `wsl`, `varnamelen`, `revive`, `wrapcheck` all flag idiomatic Go
that's been fine for years; `nolintlint` punishes legitimate suppression
comments; `depguard`/`gomodguard` are over-strict without a curated allowlist.

### .github/workflows

Two workflows in koochooloo:

- `test.yaml` — lint job, test job (with codecov upload), and a build job
  that runs after both pass. Use `actions/setup-go@v6` with
  `go-version-file: "go.mod"` so the runner tracks `go.mod` automatically.
- `codeql.yml` — security analysis on push, pull request, and a weekly cron
  (`0 6 * * 1`).

`dependabot.yml` watches `gomod` and `github-actions` weekly.

For service repos add a `docker` job (needs: lint+test) that does
`docker/bake-action@v7` against `build/package/docker-bake.json`.

### versioninfo

```go
import "github.com/carlmjohnson/versioninfo"

root := &cli.Command{
    Name:    "<binary>",
    Version: versioninfo.Short(), // urfave/cli v3 Version field
}
// or for cobra:
// root := &cobra.Command{Use: "<binary>", Version: versioninfo.Short()}
```

`versioninfo.Short()` reads the VCS info Go embeds at build time — no
`-ldflags` needed. Dirty workspaces are flagged automatically.

## Go module hygiene

- `go get pkg@vX.Y.Z` for new deps. Pin a version, never blind `@latest`.
- `go get -u` only when you have time to run the test suite afterwards.
- `go mod tidy` after every meaningful import change.
- `go build ./cmd/<binary>` for a single binary; `go build ./...` won't
  work with `-o /path` because it tries to write *N* outputs to one path.
- gopls "module not in workspace" warnings on a fresh repo are noise — they
  go away once the directory is opened as its own workspace or referenced from
  a `go.work` file. Don't paper over by importing wrong paths.

## jsm.go / nats.go ergonomics

When working with JetStream from Go:

- Use `Consumer.LatestState()` returning `api.ConsumerInfo`. There is no
  `State()` or `Info()` method.
- `api.SequenceInfo.Last` is `*time.Time`, **nil for consumers that have
  never acked**. Always nil-check before dereferencing.
- `api.ConsumerInfo.Created` is `time.Time` (not a pointer). Use it when the
  ack floor is nil and you still want an "age."
- For consumer-not-found, check via `api.IsNatsError(err, 10014)` — there's
  no sentinel `ErrConsumerNotFound`.
- For mass operations the CLI rejects (e.g. consumer names starting with `-`),
  fall through to the JetStream API subject directly:
  `nats req '$JS.API.CONSUMER.DELETE.<stream>.<consumer>' ""`.

## Pitfalls table

| Symptom | Real cause | Fix |
| --- | --- | --- |
| `go mod tidy` keeps dropping a sub-module | Root module doesn't contain `/v2` package | `go get <full-path>@<exact-version>` |
| `cmd.Int(...)` won't assign to `int64` field | v3 returns `int`, not `int64` | Cast: `int64(cmd.Int("x"))` |
| Config defaults disappear after Unmarshal | Defaults set after `Unmarshal` | Feed defaults through `structs.Provider` *before* Unmarshal |
| `--help` shows `default: 0s` for a duration | No `Value:` on the flag | Either set a `Value:` or document via config defaults |
| gopls red squiggles on every import | New module not in any `go.work` | Add to workspace or ignore — `go build` is the truth |
| `cli.Context` undefined | Following v2 examples in v3 code | Action signature is `(context.Context, *cli.Command)` |
| Linter complains about wsl/varnamelen everywhere | Default golangci-lint config | Disable in `.golangci.yml` (see config above) |
| Binary at repo root instead of `./cmd/<binary>` | `main.go` left at top level | Move to `cmd/<binary>/main.go`; update justfile target |

## Anti-patterns to refuse

- **Manual `os.Args` parsing in subcommands.** v3's `cmd.Args()` is enough.
- **`init()` for CLI wiring.** Build the command tree in a factory function (`Execute()`); easier to test.
- **`os.Exit` deep inside a subcommand.** Return an error; the root prints + exits.
- **`internal/domain` importing third-party clients.** That file belongs in `internal/infra/<name>` instead. Domain stays pure.
- **`Makefile` instead of `justfile`.** Make's tab/space rules and POSIX-shell-isms keep hurting new contributors. `just` recipes read like recipes.
- **`viper` in new code.** koanf does the same job with one-tenth the dependency tree. Choose viper only when extending an existing viper-based project.
- **Wide-open `replace` directives.** Pin via `go.mod` `require` lines; only use `replace` for genuinely forked dependencies.
- **Skipping the linter on CI.** golangci-lint catches what reviewers don't.

## When to break the defaults

- **Tiny one-binary tool with three flags.** Skip koanf entirely; bind flags
  directly to a config struct. Skip `internal/domain` — everything can live
  in `internal/<area>`. The overhead isn't worth it.
- **Long-running daemon with reload.** Add `fsnotify` watch on the koanf
  file provider; emit a signal on change rather than re-reading every request.
- **Library, not binary.** Drop `cmd/`; export from the repo root. Don't
  make consumers traverse `internal/` paths.
- **HTTP service.** Add `internal/infra/http` with handlers, plus
  `internal/domain/service` for the business interfaces those handlers call.
  Wire via go.uber.org/fx (see koochooloo) if you have more than a handful
  of dependencies.
- **TUI.** Most of this still applies, but the CLI framework choice flips to
  `bubbletea` (or `tview`) and the root command lives inside the program loop.

## Reference

- **[1995parham/koochooloo](https://github.com/1995parham/koochooloo)** —
  full reference project: cobra root, koanf TOML config with `structs`
  defaults, MongoDB infra under `internal/infra/db`, OTel telemetry, fx DI,
  k6 load tests, docker-bake, codecov.
- **[1995parham/natsie](https://github.com/1995parham/natsie)** — small CLI
  applying this skill: urfave/cli v3 + koanf YAML + carlmjohnson/versioninfo.
- [urfave/cli v3 migration guide](https://cli.urfave.org/migrate-v2-to-v3/)
- [knadh/koanf README](https://github.com/knadh/koanf)
- [golangci-lint v2 linter list](https://golangci-lint.run/usage/linters/)
