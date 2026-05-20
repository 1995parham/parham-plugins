# gopass-skill

A [Claude Code](https://docs.anthropic.com/claude-code) skill for working with the [gopass](https://www.gopass.pw/) password store.

When invoked, the skill teaches Claude how to:

- Show, insert, generate, move, search, and delete secrets safely
- Manage GPG recipients on single- and multi-store setups
- Handle git-backed sync, recover from wedged merges, and reason about auto-push behaviour
- Use cross-secret references (`gopass://`) instead of duplicating credentials
- Diagnose common pitfalls — invalid-key warnings, misleading recipient listings, full-fingerprint requirements, trailing-newline traps

## Install

Clone into your Claude Code skills directory:

```bash
git clone https://github.com/1995parham/gopass-skill.git ~/.claude/skills/gopass
```

Or, if you maintain a plugin marketplace, add this repo as a plugin source.

The skill auto-activates when Claude detects gopass-related work in the conversation (e.g. you mention secrets, recipients, the password store, or run a `gopass` command). You can also invoke it explicitly with `/gopass` if your harness exposes skills as slash commands.

## What's covered

The skill content lives in [`SKILL.md`](SKILL.md). At a glance:

- Mental model: secrets are files, stores are directories with their own recipient list and git remote
- Common commands: `show` / `insert` / `generate` / `find` / `mv` / `rm` / `audit` / `fsck`
- Secret formats: Key-Value (default), YAML (legacy), Plain
- `gopass://` references and `core.follow-references`
- Multi-store mounts
- Recipient lifecycle, including the "delete expired key locally" trick that avoids a store-wide re-encryption
- Sync hygiene and merge-conflict recovery
- TOTP, audit, env injection, templates

## License

MIT — see [`LICENSE`](LICENSE).
