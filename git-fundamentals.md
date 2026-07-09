# Git — The Reference You Should Know (for experienced engineers)

Dense, practical, scannable. Assumes you know what version control is. Includes the setup we covered for Claude Code.

---

## 1. Setup & config

Config has three scopes, narrowest wins: **local** (`.git/config`) > **global** (`~/.gitconfig`) > **system** (`/etc/gitconfig`).

```bash
# Identity (global default)
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Per-repo override (run inside the repo, no --global)
git config user.email "you@work.com"

# Inspect
git config --list --show-origin     # every setting + which file it came from
git config --global --list          # global only
git config --local --list           # this repo only
git config --global --edit          # open ~/.gitconfig
git config user.email               # read one key

# Sensible globals
git config --global init.defaultBranch main
git config --global pull.rebase true          # rebase instead of merge on pull
git config --global push.autoSetupRemote true # auto -u on first push
git config --global core.editor "code --wait"
git config --global rerere.enabled true       # remember conflict resolutions
git config --global fetch.prune true          # drop deleted remote branches on fetch
```

Handy aliases:
```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.last "log -1 HEAD --stat"
git config --global alias.unstage "reset HEAD --"
```

---

## 2. Starting a repo

```bash
git init                       # new repo in current dir
git clone <url>                # clone
git clone <url> dir            # clone into named dir
git clone --depth 1 <url>      # shallow (fast, no full history)
git remote add origin <url>    # attach a remote
git remote -v                  # list remotes
git remote set-url origin <url>
```

---

## 3. The daily loop

```bash
git status                     # what's changed / staged
git status -s                  # short form
git add <file>                 # stage
git add -p                     # stage interactively, hunk by hunk (learn this)
git add -A                     # stage everything (incl. deletions)
git commit -m "msg"
git commit -am "msg"           # stage tracked + commit
git commit --amend             # rewrite last commit (message or add staged changes)
git commit --amend --no-edit   # add staged changes, keep message
git restore <file>             # discard unstaged changes to a file
git restore --staged <file>    # unstage (keep changes)
git restore --source=HEAD~2 <file>  # restore file from an older commit
```

**Mental model:** working dir → (`add`) → staging area / index → (`commit`) → local repo → (`push`) → remote.

---

## 4. Inspecting

```bash
git log --oneline --graph --decorate --all   # the view you want 90% of the time
git log -p <file>              # history with diffs for a file
git log --follow <file>        # follow across renames
git log -S "someString"        # commits that added/removed that string (pickaxe)
git log --author="name" --since="2 weeks ago"
git show <commit>              # a commit's diff
git diff                       # unstaged changes
git diff --staged              # staged changes (vs HEAD)
git diff main..feature         # between branches
git diff --stat                # summary of changes
git blame <file>               # who last touched each line
git shortlog -sn               # commit counts per author
```

---

## 5. Branches

```bash
git branch                     # list local
git branch -a                  # incl. remote-tracking
git switch <branch>            # move to branch (modern)
git switch -c <branch>         # create + switch
git checkout <branch>          # older equivalent of switch
git branch -d <branch>         # delete (safe; refuses if unmerged)
git branch -D <branch>         # force delete
git branch -m old new          # rename
git branch --merged            # branches merged into current (safe to delete)
```

---

## 6. Merging vs rebasing

```bash
git merge <branch>             # merge branch into current (creates merge commit)
git merge --no-ff <branch>     # force a merge commit even if fast-forward
git merge --squash <branch>    # combine into staged changes, one commit

git rebase main                # replay current branch's commits on top of main
git rebase -i HEAD~5           # interactive: squash/reword/reorder/drop last 5
git rebase --onto A B feature  # advanced: move feature from B onto A
```

Rebase for a **clean linear history** before merging; merge to **preserve context**. Golden rule: **don't rebase commits you've already pushed & shared** (rewrites history others may have).

Interactive rebase actions: `pick`, `reword`, `edit`, `squash` (combine, keep messages), `fixup` (combine, drop message), `drop`, reorder lines.

Conflict flow (merge or rebase):
```bash
# edit conflicted files, then:
git add <resolved-files>
git rebase --continue    # or: git merge --continue
git rebase --abort       # bail out entirely
```

---

## 7. Syncing with remotes

```bash
git fetch                      # download refs, don't touch working dir
git fetch --prune              # + drop deleted remote branches
git pull                       # fetch + merge (or rebase if configured)
git pull --rebase              # fetch + rebase (linear)
git push                       # push current branch
git push -u origin <branch>    # push + set upstream (first time)
git push --force-with-lease    # safe force push (won't clobber others' work)
git push origin --delete <branch>   # delete remote branch
```

Prefer **`--force-with-lease`** over `--force` — it refuses if the remote moved since your last fetch.

---

## 8. Undo & recovery (the safety net)

```bash
git reset --soft HEAD~1        # undo commit, keep changes staged
git reset --mixed HEAD~1       # undo commit, keep changes unstaged (default)
git reset --hard HEAD~1        # undo commit + discard changes (destructive)
git revert <commit>            # new commit that undoes another (safe for shared history)

git reflog                     # log of everywhere HEAD has been — your undo history
git reset --hard <reflog-sha>  # recover a "lost" commit/branch after a bad reset

git clean -n                   # preview untracked files that would be deleted
git clean -fd                  # actually delete untracked files + dirs
```

**`git reflog` is the thing that saves you** after a mistaken reset/rebase — commits aren't truly gone for ~30–90 days.

---

## 9. Stash

```bash
git stash                      # shelve working changes
git stash -u                   # include untracked files
git stash push -m "wip: auth"  # named stash
git stash list
git stash pop                  # apply + drop most recent
git stash apply stash@{2}      # apply a specific one, keep it
git stash drop stash@{0}
```

---

## 10. Cherry-pick, tags, bisect, worktrees

```bash
git cherry-pick <commit>       # apply one commit onto current branch
git cherry-pick A..B           # a range

git tag v1.2.0                 # lightweight tag
git tag -a v1.2.0 -m "msg"     # annotated (preferred for releases)
git push origin v1.2.0         # tags don't push by default
git push --tags

git bisect start
git bisect bad                 # current is broken
git bisect good <sha>          # this one worked
# git checks out midpoints; you mark good/bad until it finds the culprit
git bisect reset

# Worktrees: multiple branches checked out at once, separate dirs
git worktree add ../feature-x feature-x
git worktree list
git worktree remove ../feature-x
```

Worktrees are what Claude Code Desktop uses under the hood for **parallel sessions** — each session gets an isolated working dir on its own branch, so they don't step on each other.

---

## 11. .gitignore

```bash
# patterns
build/            # dir
*.log             # extension
!keep.log         # negate (un-ignore)
/secret.env       # only at repo root

git rm -r --cached <path>      # stop tracking something already committed
git check-ignore -v <path>     # why is this file ignored?
```

---

## 12. GitHub CLI (`gh`) — needed for Claude Code PR features

```bash
brew install gh                # macOS (cli.github.com for Windows/Linux)
gh auth login                  # authenticate
gh auth status
gh repo clone owner/repo
gh pr create                   # open a PR from current branch
gh pr status
gh pr checks                   # CI status
gh pr view --web
gh pr merge --squash --delete-branch
gh issue list
```

Without `gh` authenticated, Claude Code's **PR creation, CI monitoring, and auto-merge** features won't work.

---

## 13. Git + Claude Code (project-level setup)

The git config above is machine/repo-level. The files that shape Claude Code sit alongside git and **should be committed** so your team shares them:

```
CLAUDE.md                        # project memory — commit
.claude/settings.json            # hooks, permissions (shared) — commit
.claude/launch.json              # dev-server / preview config — commit
.claude/skills/<name>/SKILL.md   # shared skills — commit
.claude/agents/<name>.md         # shared subagents — commit
.claude/settings.local.json      # personal overrides / secrets — GITIGNORE
```

Add to `.gitignore`:
```
.claude/settings.local.json
```

Everything git-related (`.git/config`, worktrees, diffs, PR status) is what the Desktop app's **diff viewer, worktree-isolated parallel sessions, and CI status bar** are reading from. No repo → no visual review or PR monitoring.

---

## 14. Quick-recall table

| Goal | Command |
|---|---|
| Undo last commit, keep work | `git reset --soft HEAD~1` |
| Safely undo a pushed commit | `git revert <sha>` |
| Recover after a bad reset | `git reflog` → `git reset --hard <sha>` |
| Stage selectively | `git add -p` |
| Clean history before merge | `git rebase -i main` |
| Safe force push | `git push --force-with-lease` |
| Find which commit broke it | `git bisect` |
| Who wrote this line | `git blame <file>` |
| Shelve work temporarily | `git stash` |
| Find commit that touched a string | `git log -S "text"` |
| Multiple branches at once | `git worktree add` |

---
*Git fundamentals are stable; the Claude Code `.claude/` conventions reflect the app as of mid-2026.*
