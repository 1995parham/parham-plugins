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
| Go version | Latest stable (1.26 as of 2026-05). Pin `go 1.26` in `go.mod`; let `toolchain` auto-upgrade patch releases. | Modern stdlib (`slog`, `slices`, `maps`, `iter`, `cmp`, `synctest`) and language features (range-over-int, range-over-func, generic type aliases, `new(expr)`) are only available on recent toolchains. Bump within a release of upstream. |
| Logger | `log/slog` (stdlib) | Structured logging in the stdlib since 1.21; no third-party dependency. Use `zap`/`zerolog` only when an existing repo already commits to it. |
| CLI framework | `github.com/urfave/cli/v3` | Stdlib `context.Context` plumbing built in; smaller surface than cobra; no codegen. (koochooloo uses cobra; both are acceptable, but new repos pick urfave v3.) |
| Config | `github.com/knadh/koanf/v2` | Provider/parser split keeps the binary small. Layers defaults → file → env → flags cleanly. |
| Config format | TOML (koochooloo) or YAML | TOML matches the koochooloo precedent; YAML is fine for human-edited config. Both via koanf parsers. |
| Project layout | [golang-standards/project-layout](https://github.com/golang-standards/project-layout) — `cmd/<binary>/main.go` + `internal/cmd` + `internal/infra` + `internal/domain`, plus `api/`, `configs/`, `deployments/`, `build/package/`, `docs/` as needed. | Community-consensus layout; predictable for outside contributors. Note it is community, not officially blessed by the Go team — but the convention is broad enough that following it is the path of least friction. |
| DI | none for small CLIs; **`go.uber.org/fx`** once the dependency graph grows (HTTP server + DB + queue + scheduler + telemetry). | fx gives lifecycle hooks (`OnStart`/`OnStop`), graceful shutdown ordering, and graph validation at startup. Constructor-injection style fits Go better than tags-on-structs. Reach for it when you find yourself writing a 40-line `main` that wires components by hand. |
| HTTP framework | `github.com/labstack/echo/v4` (default) or `github.com/gofiber/fiber/v3` (when raw throughput matters more than stdlib compatibility) | Both have first-class middleware, route groups, validators, and OpenTelemetry instrumentation. Pick one per repo; do not mix. Echo's `net/http` compatibility makes it the safer default; fiber's fasthttp engine wins on benchmarks but breaks the `http.Handler` contract. Stdlib `net/http` + `ServeMux` (1.22+) is fine for the smallest services. |
| Postgres driver | `github.com/jackc/pgx/v5` (direct or via `database/sql` adapter) — **not** GORM. | Native Postgres types (arrays, JSON, ranges, LISTEN/NOTIFY), no reflection-heavy ORM layer, predictable SQL. Pair with `sqlc` for type-safe queries or write SQL by hand. GORM hides cost, surfaces footguns (N+1, auto-migrate in prod), and the abstraction leaks when you need anything Postgres-specific. |
| Metrics | `github.com/prometheus/client_golang/prometheus` + `/metrics` HTTP endpoint | Prometheus is the de-facto standard; the official client is well-maintained. Register collectors in `internal/infra/metrics`; expose via the same HTTP server or a sidecar mux. |
| Tracing | OpenTelemetry — `go.opentelemetry.io/otel` + an OTLP exporter | Vendor-neutral; switches backends (Jaeger, Tempo, Honeycomb, Datadog) by swapping the exporter. Use the contrib instrumentations for echo/fiber/pgx/grpc; don't hand-instrument what's already covered. |
| Version stamp | `github.com/carlmjohnson/versioninfo` | `--version` shows VCS commit + dirty flag with zero ldflags. |
| Linter | `golangci-lint` v2 with `default: all` and the disable list kept *short*; suppress noise per-line with `//nolint:<linter> // <reason>` rather than disabling globally. | Fail fast on the lint server. A small global disable list is fine; everything else gets an inline opt-out with a documented reason so reviewers can audit it. |
| Task runner | `justfile` | Discoverable (`just --list`), no Makefile-isms, ships with one binary. |
| CI | GitHub Actions: **lint (golangci-lint, required)**, test, build, codeql, dependabot | `golangci-lint` runs on every push and PR — no exceptions. Mirrors koochooloo's `.github/`. |
| Tool deps | `tool` directive in `go.mod` (Go 1.24+) | Replaces the old `tools.go` + blank-import trick. `go tool <name>` runs them. |
| License | Apache-2.0 (`LICENSE` at repo root) | Standard for new OSS Go tooling; matches NATS / Synadia / HashiCorp ecosystem. |
| Module path | `github.com/<owner>/<repo>` | Matches the canonical Go tooling assumption; avoids vanity-URL maintenance. |

## Repository skeleton

Follows [golang-standards/project-layout](https://github.com/golang-standards/project-layout)
— `cmd/` for entrypoints, `internal/` for private code, `api/` for protocol
definitions, `configs/` for sample config, `deployments/` for compose/k8s/helm,
`build/package/` for Dockerfiles, `docs/` for prose. The layout is community
convention, not Go-team blessed, but new contributors recognize it on sight.

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

## Modern Go — what to reach for first

The latest stable Go is **1.26** (released 2026-02-10; 1.26.3 is the current
patch). Pin `go 1.26` in `go.mod`. The standard library has absorbed most of
what people used to pull in third-party packages for — reach for stdlib first.

Features worth using by default:

- **`log/slog`** (1.21) — structured logging. Replaces zap/zerolog for new
  code. Use `slog.SetDefault(slog.New(slog.NewJSONHandler(os.Stdout, nil)))`
  in `main`, then `slog.InfoContext(ctx, "msg", "key", value)` everywhere.
- **`slices`, `maps`, `cmp`** (1.21) — generic helpers (`slices.Sort`,
  `slices.Contains`, `maps.Keys`, `cmp.Or`, `cmp.Compare`). Stop hand-rolling
  these.
- **`min`, `max`, `clear`** (1.21 builtins) — `clear(m)` empties a map without
  allocation.
- **`errors.Join`** (1.20) + **`%w` wrapping** + **`errors.Is/As`** — wrap with
  context (`fmt.Errorf("loading config: %w", err)`); never return bare errors
  from infra. Use `errors.Join(err1, err2)` for "both happened."
- **`sync.OnceFunc` / `OnceValue` / `OnceValues`** (1.21) — replaces the
  `sync.Once` + closure boilerplate.
- **`context.WithoutCancel`, `WithDeadlineCause`, `AfterFunc`** (1.21) — use
  `WithoutCancel` when fanning out cleanup that must outlive the request.
- **Range over int** (1.22) — `for range 10 { ... }` instead of a C-style loop.
- **Loop variable per-iteration scope** (1.22) — the `i := i` shadow dance
  is no longer needed inside `for ... range`.
- **`net/http.ServeMux` patterns** (1.22) — `mux.Handle("GET /users/{id}", h)`
  with method matching and path wildcards. Skip third-party routers for simple
  services.
- **`math/rand/v2`** (1.22) — properly seeded by default; no `rand.Seed(...)`.
- **Range over functions** (1.23) — define iterators as `iter.Seq[T]` /
  `iter.Seq2[K,V]` and consume with `for v := range seq { ... }`.
- **`unique.Make`** (1.23) — canonicalize repeated values (string interning).
- **`testing/synctest`** (1.25, stable in 1.26) — synthetic clock for testing
  concurrent code. Replaces brittle `time.Sleep` in tests.
- **`encoding/json` `omitzero`** (1.24) — emits nothing for zero-valued fields,
  fixing the `omitempty`-vs-zero ambiguity for time/struct fields.
- **Generic type aliases** (1.24) — `type Set[T comparable] = map[T]struct{}`
  finally works.
- **`tool` directive in `go.mod`** (1.24) — declare build/test tooling
  (`go tool add github.com/google/wire/cmd/wire`); run with `go tool wire`.
  Retire `tools.go` and the blank-import workaround.
- **`new(expr)`** (1.26) — `p := new(int64(300))` returns `*int64` pointing at
  300. Drop the temporary variable.
- **`errors.AsType[T]`** (1.26) — type-safe error inspection without the
  `var x *T; errors.As(err, &x)` dance.
- **`go fix` modernizers** (1.26) — `go fix ./...` will rewrite legacy patterns
  (`interface{}` → `any`, `sort.Slice` → `slices.Sort`, etc.) to their modern
  equivalents. Run it after a Go upgrade.

Things to stop reaching for:

- `any` / `interface{}` as a parameter or field type. If you reach for `any`,
  reach for **generics** first. A function that takes `any` and switches on
  `.(type)` is almost always a generic function in disguise — `func Sum[T
  Number](xs []T) T` beats `func Sum(xs []any) any` on safety, performance
  (no boxing), and call-site ergonomics (no cast on the result). Reserve
  `any` for the genuinely heterogeneous edges: JSON decode targets,
  `slog.Attr` values, plugin/reflection boundaries.
- `interface{}` spelled out — use `any` (1.18+) on the rare lines where `any`
  is genuinely the right choice.
- `ioutil` — every helper moved to `io` or `os` in 1.16.
- `github.com/pkg/errors` — `errors.Is/As/Join` + `%w` cover it.
- `github.com/sirupsen/logrus` and similar — `log/slog` covers it.
- Hand-rolled `Min`/`Max`/`Contains` generics — `slices`/`min`/`max`.
- `tools.go` with blank imports — `tool` directive in `go.mod`.

## Comments on exported identifiers

Every exported type, function, method, constant, and package-level variable
**gets a godoc comment**. The comment starts with the identifier's name and
is one or more complete sentences:

```go
// Config holds the loaded configuration tree, layered from defaults,
// the on-disk config file, environment variables, and CLI flags.
type Config struct { ... }

// Load reads cfg from the file at path. It returns the first non-nil
// error from any provider; partial loads are reported but not fatal.
func Load(path string) (*Config, error) { ... }
```

Package docs go in `doc.go` (or above `package xyz` in the smallest file)
and start with `Package xyz ...`.

This is enforced by `revive`'s `exported` rule (which is part of the default
golangci-lint set). Don't disable revive globally to silence it — fix the
missing comment, or annotate the one legitimate exception with
`//nolint:revive // <reason>`.

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

**Posture:** `default: all`, disable as little as possible globally, and
silence the few legitimate exceptions per-line with
`//nolint:<linter> // <why>`. A line-level opt-out is reviewable; a global
disable is invisible. If a linter fires often enough that you're tempted to
disable it, fix the code first.

```yaml
---
version: "2"
linters:
  default: all
  disable:
    # Keep this list short. Each entry needs a one-line "why" comment.
    - depguard       # noisy without a curated allowlist; revisit when one exists
    - gomodguard     # same reasoning as depguard
    - tagliatelle    # struct tags often follow external API naming, not Go style
    - varnamelen     # short receiver / loop names are idiomatic Go
    - wsl            # whitespace cop produces too much churn for too little signal
  settings:
    revive:
      # `revive` stays enabled — it is what enforces godoc comments on
      # exported names. Tune its rules here rather than disabling it wholesale.
      rules:
        - name: exported
          arguments: ["checkPrivateReceivers", "disableStutteringCheck"]
        - name: package-comments
        - name: var-naming
    wrapcheck:
      # Don't double-wrap stdlib-style sentinels.
      ignore-sigs:
        - .Errorf(
        - errors.New(
        - errors.Unwrap(
        - errors.Join(
        - .Wrap(
        - .Wrapf(
issues:
  # `nolintlint` stays on (it's in `default: all`) so every suppression must
  # name the linter and carry a reason. This is the contract.
  max-issues-per-linter: 0
  max-same-issues: 0
```

#### Line-level suppression — the only acceptable shape

```go
// One linter, with a reason. The reason is mandatory; `nolintlint` rejects
// bare `//nolint` and `//nolint:foo` without `//` reason.
sum := 0 //nolint:revive // accumulator name is fine in a 4-line loop

// Multiple linters in one directive:
if err := f.Close(); err != nil { //nolint:errcheck,gosec // best-effort close on shutdown
    // ...
}

// Whole-block suppression — put the directive on the block opener:
//nolint:gocyclo // state machine, splitting it hurts readability more than the cyclo score
func dispatch(ev Event) error {
    // ...
}
```

Rules of thumb:

- **Never** use a bare `//nolint` or a file-wide `//nolint:all`. Reviewers
  can't tell what was being silenced or why.
- The reason text after `//` must explain *why this code is the exception*,
  not *what the linter does*. "errcheck — Close in defer" is bad;
  "best-effort close on shutdown, nothing to do on error" is good.
- If you find yourself writing the same `//nolint` reason three times in a
  package, that's a signal to *fix the code* (extract a helper, restructure)
  or open a discussion about adding the linter to the global disable list —
  not to keep stamping suppressions.
- `wrapcheck`, `revive`, `errcheck` are kept on because they catch real bugs.
  Suppress per-line when you have a reason, not globally.

### .github/workflows

Two workflows in koochooloo:

- `test.yaml` — **`lint` job is mandatory** and runs on every push/PR via
  `golangci/golangci-lint-action@v8` (or current major). It must block merges.
  Then a `test` job (with codecov upload), and a `build` job that runs after
  both pass. Use `actions/setup-go@v6` with `go-version-file: "go.mod"` so
  the runner tracks `go.mod` automatically — and bump `go.mod` to the latest
  stable Go (`go 1.26` at time of writing) so the runner picks it up.
- `codeql.yml` — security analysis on push, pull request, and a weekly cron
  (`0 6 * * 1`).

```yaml
# .github/workflows/test.yaml — the lint job
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-go@v6
        with:
          go-version-file: "go.mod"
      - uses: golangci/golangci-lint-action@v8
        with:
          version: latest
          args: --timeout=5m
```

`dependabot.yml` watches `gomod` and `github-actions` weekly so the toolchain,
linter version, and module pins stay current automatically.

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
| Linter complains about wsl/varnamelen everywhere | Default golangci-lint config | Disable *that linter only* in `.golangci.yml` with a one-line "why" comment; suppress single sites with `//nolint:<linter> // <reason>` |
| `revive` flags every exported type/function | Missing godoc comments | Add `// Name ...` godoc comments; do NOT disable revive |
| `nolintlint` rejects your suppression | Bare `//nolint` or missing reason | Use `//nolint:<linter> // <why this code is the exception>` |
| `go.mod` stuck on an old `go 1.X` | Never bumped after install | Set `go 1.26` (latest stable) and let `toolchain` pin the patch; CI's `go-version-file: go.mod` follows |
| `tools.go` blank imports drift from CI versions | Pre-1.24 tooling pattern | Migrate to `tool` directive in `go.mod` (`go tool add <path>`) and `go tool <name>` to run |
| Binary at repo root instead of `./cmd/<binary>` | `main.go` left at top level | Move to `cmd/<binary>/main.go`; update justfile target |

## Anti-patterns to refuse

- **Manual `os.Args` parsing in subcommands.** v3's `cmd.Args()` is enough.
- **`init()` for CLI wiring.** Build the command tree in a factory function (`Execute()`); easier to test.
- **`os.Exit` deep inside a subcommand.** Return an error; the root prints + exits.
- **`internal/domain` importing third-party clients.** That file belongs in `internal/infra/<name>` instead. Domain stays pure.
- **`Makefile` instead of `justfile`.** Make's tab/space rules and POSIX-shell-isms keep hurting new contributors. `just` recipes read like recipes.
- **`viper` in new code.** koanf does the same job with one-tenth the dependency tree. Choose viper only when extending an existing viper-based project.
- **Wide-open `replace` directives.** Pin via `go.mod` `require` lines; only use `replace` for genuinely forked dependencies.
- **Skipping the linter on CI.** golangci-lint runs on every push and PR; it's a *required* check, not advisory.
- **Disabling a linter globally to make red squiggles go away.** Disable globally only with a one-line justification in `.golangci.yml`; otherwise suppress per-line with `//nolint:<linter> // <reason>` and a real reason.
- **Bare `//nolint` or file-wide `//nolint:all`.** Every suppression names the linter *and* explains why. `nolintlint` enforces this — leave it on.
- **Undocumented exported names.** Every exported type/function/constant/variable needs a godoc comment starting with its name. `revive`'s `exported` rule catches this; don't disable it.
- **Pinning to an EOL Go version.** Track the latest stable Go in `go.mod`. Two releases behind = no security fixes upstream.
- **Reaching for third-party packages the stdlib now ships.** Default to `log/slog`, `slices`/`maps`/`cmp`, `errors.Join`, `sync.OnceValue`, `iter.Seq` — only pull in `zap`/`zerolog`/`lo`/`pkg/errors` when an existing repo already does.
- **GORM for new Postgres code.** Use `pgx/v5` directly, or `database/sql` over pgx, optionally with `sqlc` for typed queries. GORM's reflection layer hides cost, auto-migrate is dangerous in prod, and the API papers over Postgres features you'll eventually need.
- **`any` / `interface{}` to dodge writing types.** Prefer concrete types, named interfaces with method sets, or generics (`[T any]`, `[T comparable]`, `[T Number]`). A `map[string]any` config blob or an `any`-returning helper is almost always a generic in disguise. Reserve `any` for genuinely heterogeneous boundaries (JSON decode targets, `slog.Attr`, reflection-based plugin loaders).
- **`switch v.(type)` chains over `any`.** If you're type-switching on more than two branches, lift the dispatch into a generic function or a closed interface (sealed via an unexported method). Type switches that grow over time are how callers end up panicking on a case that was never added.
- **Hand-rolled DI past ~10 constructors.** Reach for `go.uber.org/fx` — its lifecycle hooks give you correct shutdown ordering and the graph is validated at startup, not at the first request.
- **Bespoke tracing or metrics formats.** Default to OpenTelemetry (any backend) and Prometheus (scrape endpoint). Don't reinvent context propagation or write a custom `/health` JSON when stdlib + the common contrib instrumentations cover it.

## When to break the defaults

- **Tiny one-binary tool with three flags.** Skip koanf entirely; bind flags
  directly to a config struct. Skip `internal/domain` — everything can live
  in `internal/<area>`. The overhead isn't worth it.
- **Long-running daemon with reload.** Add `fsnotify` watch on the koanf
  file provider; emit a signal on change rather than re-reading every request.
- **Library, not binary.** Drop `cmd/`; export from the repo root. Don't
  make consumers traverse `internal/` paths.
- **HTTP service.** Default stack:
  - **`echo/v4`** (or `fiber/v3` if you've benchmarked and need fasthttp).
    Mount under `internal/infra/http` with handlers; route registration goes
    in a `Register(e *echo.Echo, deps ...)` per group.
  - **`pgx/v5`** for Postgres in `internal/infra/db` — never GORM. Pair with
    `sqlc` if you want type-safe generated query code.
  - **`prometheus`** collectors in `internal/infra/metrics`, exposed at
    `/metrics`.
  - **OpenTelemetry** SDK + OTLP exporter in `internal/infra/telemetry`. Use
    contrib instrumentation for echo/fiber/pgx so spans propagate without
    hand-rolling.
  - **`log/slog`** with a JSON handler in production, text in dev. Attach the
    trace/span IDs from the OTel context to every log via a `slog.Handler`
    wrapper so logs and traces correlate.
  - **`go.uber.org/fx`** for DI once the constructor list outgrows a
    readable `main` (see koochooloo for the canonical wiring). Lifecycle
    hooks (`OnStart`/`OnStop`) give you graceful shutdown ordering for
    free — HTTP server stops *before* the DB pool closes.
  - Business interfaces live in `internal/domain/service`; handlers depend
    on the interface, infra provides the implementation.
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
