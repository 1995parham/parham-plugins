# parham-plugins

A [Claude Code](https://docs.claude.com/en/docs/claude-code) plugin marketplace. Currently ships one plugin:

- **`gopass`** — work with the [gopass](https://www.gopass.pw/) password store: show / insert / generate / move / search secrets, manage GPG recipients across single- and multi-store setups, handle git-backed sync, recover from wedged merges, and resolve `gopass://` cross-secret references. Includes a pitfalls table covering invalid-key warnings, misleading recipient listings, full-fingerprint requirements, and trailing-newline traps.

## Install

Add the marketplace once, then install the plugin from it:

```shell
/plugin marketplace add 1995parham/parham-plugins
/plugin install gopass@parham-plugins
```

Update later with:

```shell
/plugin marketplace update parham-plugins
```

The `gopass` skill auto-activates when Claude detects gopass-related work in the conversation (e.g. you mention secrets, recipients, the password store, or run a `gopass` command). You can also invoke it explicitly with `/gopass:gopass`.

## Repository layout

```
.
├── .claude-plugin/
│   └── marketplace.json              # marketplace catalog
├── plugins/
│   └── gopass/
│       ├── .claude-plugin/
│       │   └── plugin.json           # plugin manifest
│       └── skills/
│           └── gopass/
│               └── SKILL.md          # the skill content
├── LICENSE
└── README.md
```

## What `gopass` covers

The skill content lives in [`plugins/gopass/skills/gopass/SKILL.md`](plugins/gopass/skills/gopass/SKILL.md). At a glance:

- Mental model: secrets are files, stores are directories with their own recipient list and git remote
- Common commands: `show` / `insert` / `generate` / `find` / `mv` / `rm` / `audit` / `fsck`
- Secret formats: Key-Value (default), YAML (legacy), Plain
- `gopass://` references and `core.follow-references`
- Multi-store mounts
- Recipient lifecycle, including the "delete expired key locally" trick that avoids a store-wide re-encryption
- Sync hygiene and merge-conflict recovery
- TOTP, audit, env injection, environment templates

## Validate locally

Before pushing changes to the marketplace:

```bash
claude plugin validate .
```

## License

MIT — see [`LICENSE`](LICENSE).
