---
name: go-projects
description: Build production-grade Go CLI projects with urfave/cli v3, koanf v2 config, and the standard cmd/ + internal/ layout. Captures pitfalls learned in the wild (modular providers, jsm.go state API, etc.).
---

# Go projects (CLI + config)

Use this skill when you are scaffolding a new Go CLI, adding a subcommand, wiring configuration, or making library-choice decisions for either of those. Defaults below are opinionated and reflect real production decisions, not survey-style "best practices."

## Defaults

| Decision | Pick | Reason |
| --- | --- | --- |
| CLI framework | `github.com/urfave/cli/v3` | Stdlib `context.Context` plumbing built in; smaller surface than cobra; no codegen. |
| Config | `github.com/knadh/koanf/v2` | Provider/parser split keeps the binary small. Layers file → env → flags cleanly. |
| Project layout | `main.go` + `cmd/<group>/<command>.go` + `internal/<pkg>` | Each subcommand group is its own package; `internal/` enforces visibility. |
| License | Apache-2.0 (`LICENSE` at repo root) | Standard for new OSS Go tooling; matches NATS / Synadia / HashiCorp ecosystem. |
| Module path | `github.com/<owner>/<repo>` | Matches the canonical Go tooling assumption; avoids vanity-URL maintenance. |

## Repository skeleton

```
<repo>/
├── LICENSE
├── README.md
├── .gitignore           # /<repo>, /dist/, /bin/, *.test, *.out, coverage.*, .DS_Store
├── go.mod
├── go.sum
├── main.go              # thin: error-print + os.Exit
├── cmd/
│   ├── root.go          # builds the root *cli.Command, wires subcommand groups
│   └── <group>/
│       ├── <group>.go   # returns *cli.Command with sub-Commands
│       └── <verb>.go    # one file per leaf command
├── internal/
│   ├── <domain>/        # business logic — pure functions, no CLI types
│   ├── config/          # koanf wrapper
│   └── ...
└── docs/
    └── ROADMAP.md       # ship in slices; document what's deferred
```

`main.go` stays small enough to read in one breath:

```go
package main

import (
    "fmt"
    "os"

    "github.com/<owner>/<repo>/cmd"
)

func main() {
    if err := cmd.Execute(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

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
    Usage:   "path to a YAML config file",
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
// cmd/<group>/<group>.go
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

## koanf v2 — the parts that actually matter

The provider/parser split is the *whole* design. The core module knows nothing
about YAML, env vars, or files — you compose it. That means `go mod tidy` only
pulls what you actually use.

Layered load order: **file → env → flags**. Last `Load`/`Merge` wins.

```go
k := koanf.New(".")
_ = k.Load(file.Provider(path), yaml.Parser())   // base
_ = k.Load(env.Provider(".", env.Opt{
    Prefix: "APP_",
    TransformFunc: func(key, val string) (string, any) {
        return strings.ReplaceAll(
            strings.ToLower(strings.TrimPrefix(key, "APP_")), "_", "."), val
    },
}), nil)                                          // override

var cfg Config
_ = k.Unmarshal("", &cfg)
```

Pitfalls:

1. **Sub-module versioning.** `github.com/knadh/koanf/providers/env/v2` is a
   separate Go module from `github.com/knadh/koanf/v2`. `go mod tidy` won't
   discover it on its own — `go get github.com/knadh/koanf/providers/env/v2@v2.0.0`
   explicitly. Same applies to any provider that has a `/v2` suffix; the
   non-`/v2` providers (`file`, `parsers/yaml`) live in the root module and
   are picked up automatically.
2. **`@latest` lies for sub-modules.** Asking for `@latest` on the parent
   module won't resolve the `/v2`-suffixed sub-modules — Go's module proxy
   treats them as different modules. Pin a version.
3. **Don't fight the provider for flag merging.** koanf has `posflag`/`basicflag`
   providers, but they assume `pflag`/stdlib `flag`. With urfave/cli v3 it's
   cleaner to load file+env into koanf, then in the Action apply CLI overrides
   by reading `cmd.IsSet("name")` and `cmd.String("name")` etc. directly into
   the same struct.

Config struct: lean on `koanf:` tags, not `mapstructure:` or `yaml:`:

```go
type Config struct {
    Defaults Defaults                  `koanf:"defaults"`
    Contexts map[string]ContextOptions `koanf:"contexts"`
}
type Defaults struct {
    MinPending int64         `koanf:"min_pending"`
    MinIdle    time.Duration `koanf:"min_idle"`
}
```

Defaults belong on the struct literal *before* Unmarshal; koanf only fills
fields it sees in the merged map.

## Go module hygiene

- `go get pkg@vX.Y.Z` for new deps. Pin a version, never blind `@latest`.
- `go get -u` only when you have time to run the test suite afterwards.
- `go mod tidy` after every meaningful import change.
- `go build .` for a single binary at the repo root; `go build ./...` won't
  work with `-o /path` because it tries to write *N* outputs to one path.
- gopls "module not in workspace" warnings on a fresh repo are noise — they
  go away once the directory is opened as its own workspace or referenced from
  a `go.work` file. Don't paper over by importing wrong paths.

## jsm.go / nats.go ergonomics

When working with JetStream from Go (likely if you got to this skill via
NATS work):

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
| Config defaults disappear after Unmarshal | Defaults set after `Unmarshal` | Set defaults on the struct *before* `Unmarshal` |
| `--help` shows `default: 0s` for a duration | No `Value:` on the flag | Either set a `Value:` or accept that the docs read odd |
| gopls red squiggles on every import | New module not in any `go.work` | Add to workspace or ignore — `go build` is the truth |
| `cli.Context` undefined | Following v2 examples in v3 code | Action signature is `(context.Context, *cli.Command)` |

## Anti-patterns to refuse

- **Manual `os.Args` parsing in subcommands.** v3's `cmd.Args()` is enough.
- **`init()` for CLI wiring.** Build the command tree in a factory function (`Execute()`); easier to test.
- **`os.Exit` deep inside a subcommand.** Return an error; the root prints + exits.
- **One mega-package.** If `internal/scanner` and `internal/config` share types, introduce a third `internal/types` rather than collapsing them. Subcommand packages should know nothing about each other.
- **`viper` in new code.** koanf does the same job with one-tenth the dependency tree. Choose viper only when extending an existing viper-based project.

## When to break the defaults

- **Tiny one-binary tool with three flags.** Skip koanf entirely; bind flags directly to a config struct. The overhead isn't worth it.
- **Long-running daemon with reload.** Add `fsnotify` watch on the koanf file provider; emit a signal on change rather than re-reading every request.
- **Library, not binary.** Drop `cmd/`; export from the repo root. Don't make consumers traverse `internal/` paths.
