---
name: gopass
description: Work with the gopass password store — show, insert, generate, move, search secrets; manage recipients and GPG encryption; handle multi-store mounts; sync with git remotes; resolve cross-secret references via `gopass://`. Use whenever the user references gopass, mentions the password store, asks to read/write a secret, manage recipients, or troubleshoot encryption/sync issues.
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
gopass generate <path> [length]                                        # cryptic password, default ~24 chars
gopass generate -s <path> 32                                           # include symbols
gopass generate -g xkcd --xkcd-capitalize --xkcd-numbers <path>        # xkcd-style: capitalized words + a number
gopass generate <path> <key> <length>                                  # populate a body key instead of the password
gopass generate -p <path>                                              # print the password after generating
gopass generate -c <path>                                              # copy the password to clipboard
gopass generate -f <path>                                              # force-overwrite an existing entry without prompting
```

`gopass generate` replaces only the first line (the password) and preserves any body key:value pairs. To regenerate an entire secret from scratch, delete it first or use `gopass insert -f`.

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

## Secret formats

### Key-Value (default, recommended)

Line 1 is the password. Subsequent lines parse as `key: value` pairs.

```text
super-secret-token
url: https://api.example.com
username: svc-account
notes: rotate quarterly
```

Read with:
```bash
gopass show -o <path>           # just the password
gopass show <path> url          # just the value of `url`
```

### YAML (legacy, discouraged)

A body containing a `---` separator triggers the YAML parser. Avoid — unquoted numbers like `1234` get parsed as integers, and phone-number-shaped strings can become octal. If you must inspect a YAML-shaped secret, use `gopass show -n <path>`.

### Plain

Anything without `key: value` lines is just a multi-line password. `gopass show -o` returns the first line; `gopass show` returns everything.

## Cross-secret references — `gopass://`

A secret whose first line is a `gopass://` URI references another secret. When `core.follow-references` is enabled, `gopass show` transparently resolves the reference at read time, returning the target's password.

Enable once per machine:
```bash
gopass config core.follow-references true
```

Use it to avoid duplicating a canonical credential:

```text
# Canonical secret at services/db/postgres
the-real-password
url: postgres://db.example.com:5432/app

# Reference at apps/billing/db-creds
gopass://services/db/postgres
note: shares creds with the billing service
```

Then both `gopass show -o services/db/postgres` and `gopass show -o apps/billing/db-creds` return `the-real-password`. Rotating the canonical secret updates every reference automatically.

Limits: references resolve at read time only; circular references error out; the URI must be the password line, not a body field. To see the literal reference rather than the resolved value, use `gopass show -n <path>`.

## Multi-store / mounts

```bash
gopass mounts                                       # list mounts
gopass mounts add <name> <path>                     # mount an existing store
gopass mounts remove <name>                         # unmount (does not delete)
gopass clone <git-url> <name>                       # clone a remote store and mount it
```

A mount is just a directory marker — the actual store lives at `<XDG_DATA_HOME>/gopass/stores/<name>`. Secrets under the mount path use that store's `.gpg-id`, not the root's.

To check which store a given path uses:
```bash
gopass mounts | grep <mount-name>
```

## Recipients & encryption

### List

```bash
gopass recipients                                   # all stores
gopass recipients --store=<mount-name>              # single store
```

The display shows `email => 0xKEYID` when a recipient is identified by email. **This `=>` resolution is what GPG returns at *display* time and is not necessarily the key actually used at encryption time.** When the listed key is expired or otherwise invalid, gopass prints `Not using invalid key 0xXXX` and silently falls back to another valid key matching the same email (e.g. a newer key the user pushed). The listing won't update to reflect this until the expired key is removed from your local keyring.

To see the **actual** recipients of a stored secret:
```bash
gpg --list-packets <store>/<path>.gpg | grep keyid
```

### Add a recipient

```bash
gopass recipients add --store=<mount> <full-40-char-fingerprint>
```

The short `0x...` form returns `Error: no key added`. Always use the full 40-character fingerprint:
```bash
gpg --list-keys --with-colons <email-or-keyid> | awk -F: '/^fpr/{print $10; exit}'
```

After adding, gopass re-encrypts every secret in the store to include the new recipient.

### Remove a recipient

```bash
gopass recipients remove --store=<mount> <email-or-fingerprint>
```

Whatever identifier appears in `.gpg-id` works (email if added by email, fingerprint if added by fingerprint). The remove re-encrypts every secret to the *remaining* recipients.

Caveat: removed recipients can still decrypt old git revisions and any local copies they have. The only way to truly revoke is to rotate every secret.

### Refresh public keys from the store

When a teammate publishes a new public key, they typically commit it to `.public-keys/<email>`. Pull the latest and import:

```bash
cd <store-dir> && git pull
gpg --import .public-keys/<email>
# or for everything new:
gopass sync --store=<mount>
```

### The "delete expired key locally" trick

When `gopass recipients` shows `email => 0xEXPIRED` for someone who has *also* published a newer valid key with the same email UID, you don't need to swap the recipient in `.gpg-id`. Just delete the expired key from your local keyring:

```bash
gpg --batch --yes --delete-keys <expired-key-fingerprint>
```

Once only the valid key matches the email in your keyring, GPG resolves the email to it, the `Not using invalid key` warning goes away, and the display updates on next listing. This is a per-developer fix that doesn't require a store-wide re-encryption.

This trick does **not** help when the expired key has no valid replacement — that user's new secrets won't be encrypted to them at all until they publish a new key. Confirm coverage with `gpg --list-packets` on a recently re-encrypted secret.

## Sync (git)

### Normal flow

Most write operations (insert, rm, mv, recipients add/remove) auto-commit and auto-push for git-backed stores. Force a manual sync with:

```bash
gopass sync                                        # all stores
gopass sync --store=<mount>                        # one store
gopass --nosync show <path>                        # one-off skip auto-sync
```

`sync` does: pull → fetch teammates' new public keys → push.

### Recovering from a wedged merge

If `gopass sync` reports `Pulling is not possible because you have unmerged files`, the store's git working tree has a conflict from a prior partial merge. To get back to a clean state:

```bash
cd <store-dir>
git status                                         # confirm the conflict
git diff --name-only --diff-filter=U               # list conflicted files
git merge --abort                                  # back out the merge

# Decide: keep local commits and rebase, or discard them and match remote?
# To discard local changes:
git reset --hard origin/<branch>
```

`git reset --hard` discards any local commits ahead of the remote — including any `recipients remove` or secret edits made since the last successful sync. Run `git log @{u}..HEAD --oneline` first to see what's about to be thrown away. After the reset, redo any work that was lost.

### When local diverges and push is rejected

```text
! [rejected]  master -> master (non-fast-forward)
```

Means your local store has commits the remote doesn't, *and* the remote has commits you don't. Pull first (`git pull` in the store dir), resolve any conflicts, then push. Avoid `git push --force` on shared stores — you'll overwrite teammates' commits.

## OTP / TOTP

Store an `otpauth://` URL in a secret (typically on a `totp:` body line or as the password) and gopass generates time-based tokens:

```bash
gopass insert <path>                               # paste otpauth://... as the value
gopass otp <path>                                  # print current TOTP
gopass otp -c <path>                               # copy to clipboard
gopass otp -p <path>                               # chain TOTP after password
```

## Audit and fsck

```bash
gopass audit                                       # decrypt all secrets, check for weak/leaked passwords
gopass audit --format=html -o report.html          # write an HTML report
gopass fsck --store=<mount>                        # check store integrity, fix permissions, normalize
gopass fsck --decrypt --store=<mount>              # also re-encrypt every secret (slow)
```

`fsck --decrypt` is the canonical "re-encrypt the world after recipient churn" command. It will silently drop recipients whose keys are invalid in your local keyring at the moment it runs — verify your keyring is current before running.

## Environment injection

Run a subprocess with a secret's body exposed as environment variables:

```bash
# A secret with body:
#   url: postgres://...
#   user: app
#   password: hunter2

gopass env services/db/postgres -- ./run-migration.sh
# inside run-migration.sh: $URL, $USER, $PASSWORD are set
```

`--keep-case` preserves the original key casing instead of upper-casing.

## Pitfalls — quick reference

| Symptom | Cause | Fix |
|---|---|---|
| `Error: no key added` from `recipients add` | Short fingerprint passed | Use full 40-char fingerprint (`fpr` field from `gpg --with-colons`) |
| `Not using invalid key 0xXXX` warnings | Recipient's key expired/invalid | If they have a valid alternative, delete the expired one from your local keyring. If not, the user is silently being dropped from new encryptions |
| `gopass recipients` shows expired key for an email | Local keyring has both expired and new keys | `gpg --delete-keys <expired-fpr>` — display will resolve to the valid key |
| `Pulling is not possible because you have unmerged files` | Store git repo wedged on prior merge | `git merge --abort` then decide: reset to origin or finish merge manually |
| `! [rejected] non-fast-forward` on push | Local and remote both have new commits | Pull in the store dir, resolve, push |
| Trailing newline in inserted password | Used `echo` to pipe value | Use `printf '%s'` (no `\n`) instead |
| `gopass show` returns the literal `gopass://...` string | `core.follow-references` not enabled | `gopass config core.follow-references true` |
| Multi-line insert eats the password | First line wasn't the password; used `-m` without the right structure | Ensure line 1 is the password and body follows |
| Secret with `---` line parses weird | Triggered YAML parser | `gopass show -n` to view raw, or rewrite without `---` |
| `recipients remove` succeeded but nothing pushed | Auto-push failed silently | Check the store git repo: `git log @{u}..HEAD` for unpushed commits, then `git push` |

## Conventions for write operations

1. **Confirm before destructive ops** on git-backed stores: `recipients remove`, `rm`, `fsck --decrypt`, `git reset --hard` in a store dir. These re-encrypt or rewrite history that teammates also pull.
2. **Use full fingerprints** in `.gpg-id` rather than emails when you want to pin a specific key. Emails are easier to maintain but resolve to whatever GPG picks locally.
3. **Verify coverage after recipient changes** with `gpg --list-packets <file>.gpg` on a representative secret, especially after a key rotation.
4. **Don't unset auto-sync** unless you have a reason; the default keeps teammates' keyrings and `.public-keys/` in sync.
5. **Store URLs and usernames alongside the credential** as body fields (`url:`, `username:`) so the secret is self-describing.
