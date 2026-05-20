# Git Reference

A personal cheat sheet of the git commands I'll actually use, organised by how often I'll use them. Skim this when stuck; commit it to muscle memory by using the daily core constantly.

Commands are grouped by frequency, not topic, because the realistic question is "what do I reach for right now?" not "what's the theoretical category?".

---

## Tier 1 — Daily core

The handful I'll use many times every working day. Internalise these first.

### `git status`

Shows what's changed, what's staged for the next commit, and what branch I'm on. Run this every few minutes during active work — it's free and tells me exactly where the repo stands.

```bash
git status
```

Output sections to read:
- **"Changes to be committed"** (green) — staged, ready for `git commit`
- **"Changes not staged for commit"** (red) — modified but not staged
- **"Untracked files"** (red) — new files git hasn't seen before

If `git status` says "nothing to commit, working tree clean", everything is saved into history.

### `git diff`

Shows the line-by-line changes in unstaged files. "What did I change since the last commit?"

```bash
git diff                  # all unstaged changes
git diff path/to/file     # changes to one file
```

Lines prefixed `-` are removals, `+` are additions. Press `q` to exit the pager.

### `git diff --staged`

Same idea but shows what's been *staged* — i.e. what I'm about to commit. Run this before every commit as a sanity check.

```bash
git diff --staged
```

(Also written `git diff --cached` — identical command, two names.)

### `git add`

Moves changes into the staging area for the next commit.

```bash
git add path/to/file        # stage one file
git add file1.py file2.py   # stage several specific files
git add .                   # stage everything in current dir (recursive)
git add -A                  # stage everything in the entire repo
```

Prefer naming files for normal commits — it forces me to think about whether each change belongs in this commit. Use `git add .` only for sweeping changes where I genuinely want everything.

### `git commit -m "message"`

Saves staged changes into history with a message. Always use `-m` unless I genuinely want a multi-line message (which opens an editor).

```bash
git commit -m "Add HSV tuner skeleton"
```

Commit message conventions:
- Imperative mood, present tense — "Add X", not "Added X" or "Adds X"
- Specific — "Fix integral windup in pan controller" not "Fix bug"
- No `WIP`, `stuff`, `updates`, `more changes` on `main`

The mental rule: the message completes the sentence *"If applied, this commit will ___."*

### `git push`

Sends commits from local to GitHub. After the first `git push -u origin main`, plain `git push` is all I need.

```bash
git push
```

If git complains the remote is ahead of local, run `git pull` first to bring local up to date, then push.

### `git pull`

Fetches changes from GitHub and merges them into the current branch. Run at the start of a work session, especially if I've pushed from another machine or accepted PRs on the web.

```bash
git pull
```

### `git log --oneline`

Shows commit history, one line per commit. Useful for "what have I done recently?" or finding a commit hash.

```bash
git log --oneline           # compact list
git log                     # full detail, with author + date
git log --oneline -10       # just the last 10
```

Press `q` to exit the pager.

---

## Tier 2 — Weekly / branching

Less frequent but still part of normal workflow, especially once v2 work starts (per RFP §12.1, v2 lives on a feature branch merged via PR).

### `git branch`

Lists local branches and marks the current one with `*`.

```bash
git branch
```

### `git checkout -b <name>` (or `git switch -c <name>`)

Creates a new branch and switches to it. The two forms are equivalent; `switch` is the newer, more focused command.

```bash
git checkout -b v2-csrt-tracker
git switch -c v2-csrt-tracker     # modern equivalent
```

Use case: starting v2 work without touching `main`.

### `git checkout <name>` (or `git switch <name>`)

Switches to an existing branch.

```bash
git checkout main
git switch main                   # modern equivalent
```

### `git merge <branch>`

Merges another branch into the current one. For solo work, often done via Pull Requests on GitHub instead — better practice and matches what team workflows look like.

```bash
git checkout main
git merge v2-csrt-tracker
```

### `git push -u origin <branch>`

First push of a new branch needs `-u` to set up tracking. After that, `git push` alone works.

```bash
git push -u origin v2-csrt-tracker
```

---

## Tier 3 — Inspecting and orienting

Used when I need to understand the state of the repo or look up something specific.

### `git log --oneline --graph --all`

Visual tree of branch history. Good for "what does my repo actually look like?".

```bash
git log --oneline --graph --all
```

### `git show <commit-hash>`

Shows what changed in a specific commit. Hash can be the short 7-character form from `git log --oneline`.

```bash
git show 99e5712
```

### `git remote -v`

Lists configured remotes with their URLs. Useful for "where am I pushing to?".

```bash
git remote -v
```

### `git diff <commit1> <commit2>`

Shows differences between two commits. Useful for "what changed between yesterday and today?".

```bash
git diff HEAD~3 HEAD              # last 3 commits' worth of changes
git diff abc1234 def5678          # between two specific commits
```

`HEAD` means the current commit; `HEAD~3` means three commits before that.

---

## Tier 4 — Recovery (rare but vital)

Used rarely. When I need them, I really need them. Worth knowing they exist; don't try to memorise the syntax — look it up when the moment comes.

### `git restore <file>`

**Destructive.** Discards unstaged changes to a file, reverting it to the last committed version. The changes are gone.

```bash
git restore path/to/file
```

Use when I've made changes I regret and want to wipe them.

### `git restore --staged <file>`

Non-destructive. Unstages a file — moves it back from staged to modified. The changes are still in the file, just not staged for the next commit.

```bash
git restore --staged path/to/file
```

### `git commit --amend -m "new message"`

Edits the **most recent** commit's message. Useful for fixing typos in the message.

```bash
git commit --amend -m "Fix typo in previous message"
```

Can also be used to add forgotten files to the last commit:

```bash
git add forgotten_file.py
git commit --amend --no-edit   # keep the existing message
```

**Don't use after pushing.** Amending rewrites history, which breaks anything else that already has the old version.

### `git reset HEAD~1`

Undoes the last commit, keeping the changes in the working directory so I can re-commit differently. By default, the changes are unstaged.

```bash
git reset HEAD~1            # undo last commit, keep changes unstaged
git reset --soft HEAD~1     # undo last commit, keep changes staged
git reset --hard HEAD~1     # undo last commit AND discard changes (destructive)
```

**Don't use after pushing.** Same reason as `--amend`.

### `git stash` / `git stash pop`

Temporarily shelves uncommitted changes so I can do something else (e.g. switch branches), then restores them later.

```bash
git stash                   # shelve current changes
git stash pop               # restore them
git stash list              # see all stashed sets
```

### `git revert <commit-hash>`

Creates a new commit that undoes the changes from a specific past commit. **Safe to use after pushing** because it adds history rather than rewriting it.

```bash
git revert abc1234
```

Use when I need to undo a commit that's already on GitHub.

---

## Tier 5 — Configuration (set once, forget)

One-time setup. Already done on this Mac, but listed for reference / future machines.

```bash
git config --global user.name "Ahmed Elganady"
git config --global user.email "your-github-email@example.com"
git config --global init.defaultBranch main
git config --global pull.rebase false
```

Check current config:

```bash
git config --global --list
```

---

## The standard daily loop

This is the rhythm to internalise. One pass per logical unit of work — not per file save, not per day.

```bash
git status                          # what's changed?
git diff                            # what specifically?
git add file1.py file2.py           # stage what belongs together
git status                          # confirm staging is right
git diff --staged                   # last look before committing
git commit -m "Add HSV calibration script"
git push                            # send to GitHub
```

A logical unit of work is something I could describe in one sentence: "added the HSV tuner", "fixed servo angle limits", "wrote PID unit tests".

---

## Frequency hierarchy (realistic estimate)

| Tier | Commands | Frequency |
|---|---|---|
| 1 | `status`, `add`, `commit`, `push`, `diff`, `log` | Several times a day |
| 2 | `pull`, `branch`, `checkout`/`switch`, `merge` | Few times a week |
| 3 | `log --graph`, `show`, `remote -v`, `diff <commits>` | Few times a week |
| 4 | `restore`, `stash`, `commit --amend`, `reset`, `revert` | Few times a month |
| 5 | `config` | Once per machine |

---

## Mistakes I'll make and how to think about recovery

- **Committed too early / wrong message** → `git commit --amend -m "..."` if not yet pushed.
- **Staged something I didn't want** → `git restore --staged <file>`.
- **Trashed a file with edits I wanted to keep** → if not yet `git add`'d, the changes are gone. Commit early and often.
- **Made a mess on `main`** → branch out (`git switch -c rescue`), fix on the branch, decide later.
- **"Detached HEAD" warning** → I've checked out a commit instead of a branch. `git switch main` (or whatever branch) gets back.

When in doubt: `git status` first. It tells me where I am and usually suggests the next command.

---

## What good commit history looks like

A `git log --oneline` on a well-disciplined repo reads like a changelog:

```
4a3f291  Add VT-7 latency measurement script
9c8d122  Fix servo angle clamp off-by-one in servo_node
b71e009  Tune Kp to 0.8, Ki to 0.1 — see results.md VT-3
e3d5fa4  Add controller_node with PI math + unit tests
99e5712  Initial Commit
```

Each line tells the reader what happened. That's the goal.
