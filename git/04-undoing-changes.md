# Undoing Changes

> The "oh no" toolkit: reset vs revert vs restore, stashing, cherry-picking, and the reflog safety net. Knowing exactly which one to reach for is a strong senior signal.

## TL;DR

- **`git restore`** — discard working-tree/staged changes for files (didn't touch history). `--staged` unstages; plain `restore` throws away edits.
- **`git reset`** — move the current branch pointer. `--soft` (keep changes staged), `--mixed` (default, keep changes unstaged), `--hard` (discard everything). Rewrites local history.
- **`git revert`** — create a *new* commit that undoes a previous one. **Safe on shared branches** because it adds history rather than rewriting it.
- **reset vs revert:** reset rewrites/removes commits (local only); revert is a forward, non-destructive undo (use on `main`/shared).
- **`git stash`** — shelve uncommitted work to switch context, reapply later with `stash pop`.
- **`git cherry-pick`** — copy a specific commit from one branch onto another.
- **`git reflog`** — the log of where `HEAD` has been; your recovery net for "lost" commits after a bad reset/rebase.

## Deep Dive

### Discarding uncommitted work — `restore`
`git restore <file>` overwrites a file in the working tree with the version from the index/HEAD — i.e. throw away unstaged edits. `git restore --staged <file>` removes it from the index (unstages) but keeps your edits. These replace the confusing old `git checkout -- <file>` and `git reset HEAD <file>` forms.

### Moving the branch pointer — `reset`
`git reset <commit>` repoints the current branch to `<commit>`. The flag controls what happens to your changes:
- **`--soft`** — branch moves; index and working tree untouched. The undone commits' changes are left **staged**. Great for "recommit these differently."
- **`--mixed`** (default) — branch moves; index reset to match; working tree untouched. Changes left **unstaged**.
- **`--hard`** — branch moves; index *and* working tree reset. **Changes are discarded** — destructive.

`git reset --soft HEAD~1` is the idiomatic "uncommit the last commit but keep my work staged." Because reset rewrites where the branch points, it's a **local-history** operation — don't reset commits you've already pushed and others have pulled.

### Forward, safe undo — `revert`
`git revert <commit>` computes the inverse of that commit and records it as a **new** commit. History is preserved and only grows, so it's the correct way to undo something on a **shared** branch (e.g. backing out a bad commit already on `main`). Reverting a merge commit needs `-m` to pick which parent line to keep.

### reset vs revert (the classic question)
- **reset** = "pretend those commits never happened" by moving the pointer back. Rewrites history → only for local/unpushed work.
- **revert** = "apply the opposite of that commit" as a new commit. Non-destructive → safe for public branches.

### Shelving work — `stash`
`git stash` saves your uncommitted changes (tracked by default) and reverts the working tree to clean, so you can switch branches or pull. `git stash pop` reapplies and drops the stash; `git stash apply` reapplies but keeps it. Use `-u` to include untracked files. `git stash list` / `git stash show -p` to inspect. Stash is a quick shelf, not a substitute for committing real work.

### Copying a commit — `cherry-pick`
`git cherry-pick <hash>` applies the change introduced by one commit onto your current branch as a new commit. Useful for hotfixes (apply a fix to a release branch and to `main`) or grabbing one commit without merging a whole branch. Can conflict like a merge.

### The safety net — `reflog`
`git reflog` records every move of `HEAD` (commits, checkouts, resets, rebases) for ~90 days. After a `reset --hard` or a botched rebase that "lost" commits, the commits aren't gone — find them in the reflog and `git reset --hard HEAD@{n}` or `git checkout <hash>` to recover. This is why "I lost my work" is almost never literally true if it was ever committed.

## Code Examples

```bash
# Discard uncommitted changes
git restore app.ts              # throw away unstaged edits to app.ts
git restore --staged app.ts     # unstage (keep edits)
git restore .                   # discard ALL unstaged working-tree changes

# Undo the last commit, keeping the work
git reset --soft HEAD~1         # uncommit, changes stay STAGED
git reset HEAD~1                # (--mixed) uncommit, changes UNSTAGED
git reset --hard HEAD~1         # uncommit AND discard the changes (destructive)

# Safely undo a pushed commit on a shared branch
git revert <bad-commit-hash>    # creates a new "undo" commit; then push

# Stash to switch context
git stash -u                    # shelve tracked + untracked work
git switch main && git pull
git switch feature && git stash pop

# Bring one commit from another branch
git cherry-pick 4f2a9c1

# Recover from a bad reset/rebase
git reflog                      # find the lost HEAD position, e.g. HEAD@{3}
git reset --hard HEAD@{3}
```

## Interview Q&A

**Q:** What's the difference between `git reset` and `git revert`?

**A:** `reset` moves the branch pointer to an earlier commit, effectively erasing the later commits from that branch — it rewrites history, so it's only safe on local/unpushed work. `revert` creates a new commit that applies the inverse of a target commit, leaving all history intact — that makes it the safe way to undo something already pushed to a shared branch like `main`.

**Q:** Explain `--soft`, `--mixed`, and `--hard` reset.

**A:** All three move the branch pointer; they differ in what they do to the index and working tree. `--soft` leaves both alone, so the undone changes stay staged. `--mixed` (default) resets the index but keeps the working tree, leaving changes unstaged. `--hard` resets both, discarding the changes entirely — the only destructive one.

**Q:** I ran `git reset --hard` and lost commits. Are they recoverable?

**A:** Almost certainly yes, if they were committed. `git reflog` shows every position `HEAD` has held, including the commit before the reset. I find that entry and `git reset --hard HEAD@{n}` (or check out the hash). Reflog keeps entries for ~90 days. Uncommitted changes that were `--hard`-discarded, though, are genuinely gone.

**Q:** How do you undo a single bad commit that's already on `main` and shared?

**A:** `git revert <hash>` — it adds a new commit undoing that change without rewriting history, so collaborators just pull the revert normally. I'd avoid `reset` here because rewriting shared history breaks everyone downstream.

**Q:** When would you use `cherry-pick`?

**A:** To copy a specific commit across branches without merging the whole branch — classically, applying a hotfix to both a release branch and `main`, or pulling one useful commit out of an abandoned feature branch.

**Q:** `git restore` vs `git reset` for unstaging — what changed?

**A:** Modern Git split the overloaded `checkout`/`reset` UX: `git restore --staged <file>` unstages, `git restore <file>` discards working-tree edits. `reset` is now reserved for moving branch pointers. Older muscle memory (`git reset HEAD <file>`, `git checkout -- <file>`) still works but the `restore` verbs are clearer.

## Gotchas & Anti-patterns

- **`reset --hard` without thinking** — discards uncommitted work irrecoverably (reflog only saves *committed* states). Stash or commit first.
- **`reset` on pushed commits** — rewrites shared history; use `revert` instead for anything others have pulled.
- **Forgetting `stash pop` conflicts** — popping onto a changed working tree can conflict; resolve like a merge. A forgotten stash lingers in `stash list`.
- **Cherry-pick creating duplicate commits** — the picked commit gets a new hash; if the branch is later merged you can get the change twice. Be deliberate.
- **Assuming work is lost** — before declaring defeat, always check `git reflog`; most "lost" commits are one command away.
- **Reverting a merge commit without `-m`** — fails or does the wrong thing; you must specify the parent to keep.

## Further Reading

- [Pro Git — Undoing Things](https://git-scm.com/book/en/v2/Git-Basics-Undoing-Things)
- [Pro Git — Reset Demystified](https://git-scm.com/book/en/v2/Git-Tools-Reset-Demystified)
- [Pro Git — Stashing and Cleaning](https://git-scm.com/book/en/v2/Git-Tools-Stashing-and-Cleaning)
- [git-reflog documentation](https://git-scm.com/docs/git-reflog)
