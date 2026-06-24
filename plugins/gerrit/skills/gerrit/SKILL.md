---
name: gerrit
description: Work with Gerrit code review over its SSH API and the `refs/changes` git workflow — fetch and check out a change, query CLs, read change-level and inline comments, vote (Code-Review/Verified), and add reviewers. Use whenever the user references Gerrit, a CL or change number, a patch set, `Change-Id`, asks to pull/review/approve a change, or wants to read review comments on a change.
---

# gerrit

Operate on a [Gerrit](https://www.gerritcodereview.com/) server from the command line: fetch and check out changes, query the change list, read change-level and inline review comments, cast review votes, and manage reviewers.

Gerrit is driven two ways, used together:

- **git** against the project remote, for fetching change refs (`refs/changes/...`).
- **the Gerrit SSH API** — `ssh <gerrit-host> gerrit <subcommand>` (SSH port is usually `29418`) — for querying, reviewing, and reviewer management.

Voting (`gerrit review`) and reviewer changes mutate server state and are visible to the whole team. Confirm the vote/label and message with the user before running `gerrit review`, especially `+2`/`-2` and `--submit`/`--abandon`.

## Mental model — read this first

- **A change is one logical commit.** It is identified two ways: the **`Change-Id`** (an `I`-prefixed 40-hex string in the commit message footer, stable across rebases) and the **legacy number / CL number** (e.g. `1051`, shown in the change URL `/c/<project>/+/1051`). Either works wherever a change is referenced.
- **Each push to `refs/for/<branch>` creates a new patch set.** Patch sets are numbered `1, 2, 3, …` within a change. They are exposed as git refs: `refs/changes/<NN>/<change>/<patchset>`, where `<NN>` is the last two digits of the change number (zero-padded). Example: change `1051`, patch set `3` → `refs/changes/51/1051/3`.
- **The `Change-Id` is created by the `commit-msg` hook.** If a push is rejected with `missing Change-Id in commit message footer`, install the hook (see Setup) and `git commit --amend`.
- **Review is expressed as label votes**, not comments. Standard labels: `Code-Review` (`-2..+2`) and `Verified` (`-1..+1`); a change is submittable when it has the required positive votes and no blocking `-2`. Comments (cover messages and inline file comments) are separate from votes.
- **The SSH API is the scriptable surface.** `ssh <host> gerrit query|review|set-reviewers|...`. Run `ssh <host> gerrit <cmd> --help` to see exact options for *your* server version — option availability varies across Gerrit releases.

## Setup / connection

```bash
# Verify SSH access and server version
ssh <gerrit-host> gerrit version

# One-time: install the commit-msg hook so commits get a Change-Id
gitdir=$(git rev-parse --git-dir)
scp -p -P 29418 <gerrit-host>:hooks/commit-msg "$gitdir/hooks/"
# (or: curl -Lo "$gitdir/hooks/commit-msg" https://<gerrit-host>/tools/hooks/commit-msg && chmod +x ...)
```

Put the host in `~/.ssh/config` so the port/user are implicit:

```sshconfig
Host <gerrit-host>
    Port 29418
    User <your-gerrit-username>
```

## Fetching & checking out a change

You rarely know the patch-set ref by hand — discover it from the change number with `git ls-remote`, then fetch:

```bash
# List every patch-set ref for change 1051
git ls-remote origin "refs/changes/*/1051/*"
# -> <sha>  refs/changes/51/1051/1
#    <sha>  refs/changes/51/1051/2  ...  (highest number = latest patch set)

# Fetch the latest patch set and check it out on a review branch
git fetch origin refs/changes/51/1051/2
git checkout -b review/1051 FETCH_HEAD
```

Conventions that keep this clean:

- Branch-name the checkout `review/<change-number>` so review branches are obvious and disposable.
- The **latest** patch set is the highest trailing number; sort the `ls-remote` output and take the last.
- After reviewing, switch away and `git branch -D review/<n>`.

## Authoring a change — always on a branch

**Never author a CL on the default branch (`master`/`main`). Create a dedicated branch first**, even for a one-commit change. The push target is `refs/for/<base>` regardless, but committing on a throwaway branch keeps the local default branch clean, makes the work easy to abandon or amend, and avoids accidentally entangling unrelated local commits in the push.

```bash
# 1. Branch off the up-to-date base (one branch per change/topic)
git fetch origin master
git checkout -b <topic> origin/master        # e.g. grafana-admin-secret

# 2. Make edits, then commit. The commit-msg hook adds the Change-Id;
#    if it's missing, install the hook (see Setup) and `git commit --amend`.
git add -p && git commit

# 3. Push the commit as a change to the base branch
git push --no-thin origin HEAD:refs/for/master
#   -> remote prints the change URL, e.g. https://<host>/c/<project>/+/<NN>
```

- **One logical change per branch.** A push to `refs/for/<base>` turns each commit on the branch into a separate change — keep the branch to the single commit you intend to review.
- **Amend, don't add, for revisions.** To upload a new patch set, `git commit --amend` (keeping the same `Change-Id`) and push again to `refs/for/<base>`.
- **Optional push options:** `refs/for/master%wip` (work-in-progress/draft), `%topic=<name>`, `%r=<reviewer>` — append after the ref.
- Confirm before pushing if the change is sensitive or you're unsure of the base branch — a pushed change is team-visible (though it can be abandoned).

## Querying changes — `gerrit query`

```bash
ssh <host> gerrit query status:open owner:self          # your open changes
ssh <host> gerrit query status:open reviewer:self       # changes awaiting your review
ssh <host> gerrit query status:merged limit:20          # recent merged
ssh <host> gerrit query change:1051                     # one specific change
```

Common operators: `status:{open,merged,abandoned}`, `owner:self`, `reviewer:self`, `project:<name>`, `branch:<name>`, `is:wip`, `change:<num-or-Id>`, `limit:N`.

Output flags (combine as needed):

| Flag | Adds |
|---|---|
| `--format JSON` | machine-readable, one JSON object per line + a trailing `stats` row |
| `--current-patch-set` | current patch set details + approvals |
| `--patch-sets` | info on **all** patch sets |
| `--all-approvals` | every patch set's votes |
| `--all-reviewers` | full reviewer list |
| `--comments` | **change-level (cover) and inline comments** |
| `--files` | file list per patch set |
| `--commit-message` | full commit message |
| `--dependencies` | depends-on / needed-by |

`--format JSON` streams **one change per line plus a final `{"type":"stats",...}` row**. When querying a single change, take the first line (`head -n1`). A query that matches nothing returns only the stats row — detect "not found" by checking the first line has no `.number`.

## Reading comments on a CL — the headline workflow

`--comments` returns *both* kinds of comment, in different places in the JSON:

- **Change-level / cover comments** → top-level `.comments[]`. These are the "Patch Set N: Code-Review+2", "Uploaded patch set N", and free-form review summaries.
- **Inline file comments** → `.patchSets[].comments[]`, each with `file`, `line`, `reviewer`, and `message`.

Add `--patch-sets` so inline comments from every patch set are included (not just the current one):

```bash
ssh <host> gerrit query --comments --patch-sets --format JSON change:1051 | head -n1 | jq .
```

Comment object shape:

```jsonc
// top-level .comments[]  (cover / change messages)
{ "timestamp": 1780137802,
  "reviewer": { "name": "...", "email": "...", "username": "..." },
  "message": "Patch Set 2: Code-Review-1\n\n(1 comment)" }

// .patchSets[].comments[]  (inline)
{ "file": "path/to/file.yaml", "line": 77,
  "reviewer": { "name": "...", "email": "..." },
  "message": "Let's keep request == limit here." }
```

A readable, colorized dump with `jq` (cover messages, then inline grouped by patch set):

```bash
ssh <host> gerrit query --comments --patch-sets --format JSON change:1051 | head -n1 | jq -r '
  "Change \(.number): \(.subject)",
  "\(.url)   [\(.status)]",
  "",
  "── Change messages ──",
  (.comments[]? | "\(.timestamp|todate)  \(.reviewer.name)\n\(.message)\n"),
  "── Inline comments ──",
  (.patchSets[]? | select((.comments // []) | length > 0) |
    "Patch set \(.number)",
    ((.comments // [])[] | "  \(.file):\(.line // "(file)")  \(.reviewer.name)\n    \(.message)\n"))
'
```

Notes:
- `(.comments // [])` guards patch sets that have no `comments` key; `.line // "(file)"` handles file-level (line-less) comments.
- `todate` renders the epoch `timestamp` as ISO-8601 UTC.
- Inline comments persist per patch set — a comment on patch set 1 stays attached to patch set 1 even after newer patch sets land, so iterate all of `.patchSets[]`.

## Reviewing / voting — `gerrit review`

Identify the patch set by **commit SHA** (simplest from a checked-out review branch) or by `<change>,<patchset>`:

```bash
# Vote +2 with a message on the currently checked-out commit
ssh <host> gerrit review "$(git rev-parse HEAD)" --code-review +2 -m "LGTM"

# Vote on a specific change/patch set without checking it out
ssh <host> gerrit review 1051,2 --code-review -1 -m "see inline"

# Verified label, and submit
ssh <host> gerrit review <sha> --verified +1
ssh <host> gerrit review <sha> --code-review +2 --submit
```

Useful flags: `--code-review {-2..+2}`, `--verified {-1..+1}`, `-m "<message>"` (cover message), `--submit`, `--abandon`, `--restore`, `--rebase`. The flag form of `gerrit review` posts only a **cover** comment/vote. To post **inline file comments** from the CLI, pipe a `ReviewInput` JSON to `gerrit review --json` (see next section) — you do *not* need the web UI or the REST `drafts` API.

## Posting inline comments — `gerrit review --json`

`gerrit review <change>,<patchset> --json` reads a Gerrit [`ReviewInput`](https://gerrit-review.googlesource.com/Documentation/rest-api-changes.html#review-input) object from **stdin**. This is the scriptable way to leave inline file comments (the flag form can't). Inline comments go under `comments`, keyed by file path, each entry a `CommentInput`:

```bash
cat <<'JSON' | ssh <host> gerrit review 1051,3 --json
{
  "message": "Optional cover message.",
  "comments": {
    "path/to/file.go": [
      { "line": 38, "unresolved": true, "message": "This builds the request with an empty URL." }
    ]
  }
}
JSON
```

Key fields per comment:

- `line` — 1-based line on the **new** side of that patch set. Omit `line` (or use `"range"`) for a file-level comment.
- `side` — `REVISION` (new, the default) or `PARENT` (left/base side). Only set when commenting on the base.
- **`unresolved`** — **THE GOTCHA: this defaults to `false`, i.e. the comment lands already-resolved.** When you're doing a review and want the author to act on a note, you **must** set `"unresolved": true` on every comment, or it shows up pre-resolved and is easy to miss. Set it explicitly on *every* review comment.
- `in_reply_to` — UUID of an existing comment to thread under / reopen. The thread's resolved state follows its **last** comment, so replying with `"unresolved": true` reopens a resolved thread.

Caveats:
- The path must match the patch set exactly (run `gerrit query --files` if unsure); a wrong path silently drops the comment.
- The SSH `gerrit query --comments` output does **not** expose comment UUIDs, so you can't get an `in_reply_to` target over SSH. To reopen a specific existing thread you need the REST API (`GET /changes/<id>/comments`, which requires an HTTP password) — otherwise just **re-post the note as a new comment with `"unresolved": true`** (a minor duplicate, but it stands as an open review item).
- Reading comments back: the JSON query exposes a comment's resolved state as `unresolved` (it may render as `null` for older comments that predate the field — treat `null`/absent as resolved).
- `--json` and `-m/-l` are mutually exclusive in spirit: put the cover message (`message`) and label votes (`labels`) inside the JSON instead.

## Managing reviewers — `gerrit set-reviewers`

```bash
ssh <host> gerrit set-reviewers -a alice@example.com -a bob@example.com 1051   # add
ssh <host> gerrit set-reviewers -r alice@example.com 1051                      # remove
```

The change can be the number or the `Change-Id`. Repeat `-a` per reviewer.

## Helper-wrapper pattern

These commands are verbose; wrapping them in a small `git-gerrit` script with subcommands keeps day-to-day use ergonomic. A useful command set:

| Subcommand | Wraps |
|---|---|
| `pull <CL>` | `git ls-remote` to find the latest patch set → `git fetch` → `git checkout -b review/<CL>` |
| `comments [CL]` | `gerrit query --comments --patch-sets --format JSON change:<CL>` + the `jq` formatter above (defaults to the current commit's `Change-Id`) |
| `approve "<msg>"` | `gerrit review $(git rev-parse HEAD) --code-review +2 -m "<msg>"` |
| `assign` | `fzf`-pick reviewers → `gerrit set-reviewers -a ... <Change-Id>` |
| `mine` / `to-review` | `gerrit query status:open owner:self` / `reviewer:self` |
| `list [status]` | `gerrit query status:<status>` |

Resolve the current change's `Change-Id` from the commit footer when no CL is given:

```bash
git log -1 --format=%B | grep -E '^Change-Id:' | awk '{print $2}' | head -n1
```

## Pitfalls — quick reference

| Symptom | Cause | Fix |
|---|---|---|
| Push rejected: `missing Change-Id in commit message footer` | `commit-msg` hook not installed | Install the hook (see Setup), then `git commit --amend` |
| `gerrit query --comments` shows only cover messages, no inline | Missing `--patch-sets`; inline comments live under patch sets | Add `--patch-sets`; read `.patchSets[].comments[]` |
| `jq: Cannot iterate over null` on `.patchSets[].comments` | Some patch sets have no `comments` key | Guard with `(.comments // [])` |
| "Change not found" when the change exists | Parsed the wrong JSON line, or matched the trailing `stats` row | Take `head -n1`; treat a first line with no `.number` as not-found |
| Inline comment has no `line` | File-level (whole-file) comment | Fall back to `.line // "(file)"` |
| `gerrit review` says it can't find the patch set | Stale SHA, or change/patchset typo | Use `git rev-parse HEAD` on the checked-out review branch, or `<change>,<patchset>` |
| Reviewing the wrong code | Fetched an old patch set | Take the **highest** trailing number from `git ls-remote "refs/changes/*/<CL>/*"` |
| `gerrit review --code-review +2` rejected | Insufficient label permission on the project | You may only hold `+1`; an approver must cast `+2` |
| SSH `connection refused` / hangs | Wrong port — Gerrit SSH is `29418`, not `22` | Set `Port 29418` in `~/.ssh/config` for the host |
| Option not recognized | Option set varies by Gerrit version | `ssh <host> gerrit <cmd> --help` to see what your server supports |
| Inline review comment lands already-resolved | `unresolved` defaults to `false` in `ReviewInput` | Set `"unresolved": true` on every review comment in the `--json` payload |
| Can't reopen / reply to a specific existing comment over SSH | SSH query doesn't expose comment UUIDs for `in_reply_to` | Use REST `GET /changes/<id>/comments` (needs HTTP password), or just re-post the note with `"unresolved": true` |
| Inline comment silently doesn't appear | File path in `comments` doesn't match the patch set | Verify the exact path with `gerrit query --files` |

## Conventions for write operations

1. **Always author a CL on a dedicated branch, never on `master`/`main`** — `git checkout -b <topic> origin/master` before committing, one logical change per branch, then push to `refs/for/<base>`.
2. **Confirm vote + message before `gerrit review`**, especially `+2`/`-2`, `--submit`, and `--abandon` — they're team-visible and hard to walk back.
3. **Fetch the latest patch set** before reviewing; never assume patch set 1.
4. **Use `review/<CL>` branches** and delete them when done, so review checkouts never get confused with real work.
5. **Read comments with `--patch-sets`** so you see inline threads from every patch set, not just the current one.
6. **Check `--help` per server** — Gerrit's SSH options differ across versions; don't assume a flag exists.
7. **Set `"unresolved": true` on inline review comments** — they default to resolved, which buries them. Only leave a comment resolved when it's truly informational and needs no author action.
