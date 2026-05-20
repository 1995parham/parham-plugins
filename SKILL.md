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
