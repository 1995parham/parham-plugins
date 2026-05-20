---
name: gopass
description: Work with the gopass password store — show, insert, generate, move, search secrets; manage recipients and GPG encryption; handle multi-store mounts; sync with git remotes; resolve cross-secret references via `gopass://`. Use whenever the user references gopass, mentions the password store, asks to read/write a secret, manage recipients, or troubleshoot encryption/sync issues.
license: MIT
---

# gopass

Operate on a [gopass](https://www.gopass.pw/) password store: show / insert / move / generate secrets, manage recipients and GPG keys, handle multi-store mounts, sync with git remotes, and resolve cross-secret references.

This skill is read+write — it changes encrypted files, mutates `.gpg-id` recipient lists, and triggers git commits and pushes when stores are git-backed. Confirm with the user before destructive operations (`rm`, `recipients remove`, force-resync) on stores that have a remote.

## Mental model — read this first

- **A secret is a file.** Each entry is one encrypted file on disk. Paths use `/` like a filesystem: `services/db/postgres` is a single secret at `<store>/services/db/postgres.gpg`.
- **A store is a directory** with its own `.gpg-id` recipient list and (optionally) its own git repo. The default "root" store usually lives at `~/.local/share/gopass/stores/root`.
- **Multiple stores can be mounted** under a path inside the root store. List with `gopass mounts`. A path like `team/foo/bar` may actually live in a submount with a different recipient list and a different git remote — treat each mount as an independent unit.
- **Recipients are GPG identities** (fingerprints or emails) listed in the store's `.gpg-id`. Every secret in the store is encrypted to every listed recipient. Public keys exported into `.public-keys/<email>` are auto-imported on `gopass sync`.
- **Git-backed stores auto-commit and auto-push** for most write operations (insert / rm / mv / recipients add|remove). A pull-before-push happens silently; merge conflicts wedge the store and require manual recovery (see below).

## Common operations

### Show

```bash
gopass show <path>              # full secret (password + body), with parsing for key-value
gopass show -o <path>           # password only (first line) — safe for piping
gopass show <path> <key>        # value of <key> from key-value body
gopass show -n <path>           # raw, no parsing (debugging YAML or `gopass://` refs)
gopass show -c <path>           # copy password to clipboard, don't print
gopass show -r -2 <path>        # second-most-recent revision (or use `gopass history`)
```

The first line is the **password**. Lines after the first are the **body** and are parsed for `key: value` pairs (see [Secret formats](#secret-formats)).

### Insert

```bash
# Single line (interactive prompt for the value):
gopass insert <path>

# Single line from stdin:
printf '%s' "$VALUE" | gopass insert -f <path>

# Multi-line (first line = password, rest = key: value body), from stdin:
printf '%s\nurl: %s\nuser: %s\n' "$TOKEN" "$URL" "$USER" | gopass insert -m -f <path>

# Append rather than overwrite:
printf '\nextra: data' | gopass insert -a <path>
```

- `-f` / `--force` overwrites without prompting; required for non-interactive scripts.
- `-m` / `--multiline` reads multiple lines until EOF (when piped) or opens `$EDITOR` (when interactive).
- Use `printf '%s'` (no `\n`) for single-line values to avoid a trailing newline in the password field. `echo` always appends `\n`.

### Generate

```bash
gopass generate <path> [length]                # cryptic password, default ~24 chars
gopass generate -s <path> 32                   # include symbols
gopass generate -x -n -c <path>                # xkcd-style: words + a number, capitalized
gopass generate <path> <key> <length>          # populate a body key instead of the password
gopass generate -p <path>                      # print after generating
```

The default replaces only the first line (password); body is preserved. Use `-t` / `--force-regen` to overwrite the entire secret.

### Search, list, find

```bash
gopass ls                       # tree view of all stores
gopass ls --flat                # flat list of secret paths
gopass find <term>              # fuzzy search by path
gopass grep <term>              # decrypt every secret and grep the contents (slow!)
```

### Move, copy, delete

```bash
gopass mv old/path new/path     # rename / relocate (re-encrypts if crossing stores)
gopass cp src dst               # copy
gopass rm -f <path>             # delete; -f skips confirmation
```

`mv` and `cp` across mounts re-encrypt the secret with the destination store's recipient list.
