# Branching & Merging

> Branches, how histories come back together, and the question you *will* be asked: merge vs rebase. Plus how to actually resolve a conflict without panic.

## TL;DR

- A branch is a movable pointer; `git switch -c feature` creates and moves onto a new one.
- **Fast-forward merge**: target branch had no new commits, so Git just slides the pointer forward — no merge commit.
- **3-way (true) merge**: both branches advanced; Git creates a **merge commit** with two parents, preserving the actual history.
- **Rebase**: replays your commits on top of another branch, producing a **linear** history — but creates *new* commits (new hashes).
- **Golden rule:** rebase local/unpushed work freely; **don't rebase commits others have already pulled** (shared history).
- A **merge conflict** happens when both sides change the same lines; Git marks them with `<<<<<<< ======= >>>>>>>`, you edit to the desired result, then `git add` + continue.
- `git switch`/`git restore` are the modern, purpose-specific replacements for the overloaded `git checkout`.

## Deep Dive

### Branching
`git branch` lists; `git switch -c name` creates+switches (modern), equivalent to `git checkout -b name`. Because a branch is just a pointer, you create them liberally for features/fixes and delete them after merge (`git branch -d name`).

### Fast-forward vs 3-way merge
When you merge `feature` into `main`:
- If `main` hasn't moved since `feature` branched off, `feature`'s commits are a direct continuation. Git can just **fast-forward** `main`'s pointer to `feature`'s tip — no new commit, perfectly linear.
- If `main` *has* advanced independently, the histories diverged. Git performs a **3-way merge** using the two tips and their common ancestor (merge base), creating a **merge commit** with two parents. This faithfully records that two lines of work joined.

`--no-ff` forces a merge commit even when a fast-forward was possible — teams use this to keep feature boundaries visible in history.

### Rebase
`git rebase main` (while on `feature`) finds the merge base, sets `feature` aside, fast-forwards to `main`, then **replays each of your commits on top** one by one. Result: linear history, as if you'd started your work from the latest `main`. Crucially, replayed commits are **brand-new commits with new hashes** — the originals are discarded.

**Merge vs rebase — the trade-off:**
| | Merge | Rebase |
|---|---|---|
| History shape | Non-linear, preserves true topology | Linear, easy to read/bisect |
| Adds a merge commit | Yes (unless FF) | No |
| Rewrites commits | No | Yes (new hashes) |
| Safe on shared branches | Yes | No |
| Conflict resolution | Once, at the merge | Possibly per replayed commit |

**Golden rule of rebasing:** never rebase commits that exist outside your local repo (already pushed/pulled by others). Rebasing rewrites history; collaborators who based work on the old commits get a divergent, conflicting history. Rebase is for *cleaning up local work before sharing*.

### Interactive rebase & squash
`git rebase -i HEAD~5` opens an editor to **reword**, **squash**/**fixup** (combine commits), **reorder**, **edit**, or **drop** commits — the standard way to tidy a messy local branch into a few clean commits before opening a PR. Squashing turns "wip", "fix typo", "really fix it" into one coherent commit.

### Resolving merge conflicts
A conflict arises when merge/rebase can't auto-combine because both sides edited the same region. Git pauses and writes conflict markers into the file:

```
<<<<<<< HEAD
your current branch's version
=======
the incoming branch's version
>>>>>>> feature
```

Process: open each conflicted file (`git status` lists them), edit to the correct final content, **remove the markers**, `git add` the resolved file, then `git merge --continue` (or `git rebase --continue`). `git merge --abort` / `git rebase --abort` bails out completely and restores the pre-merge state. Tools: `git mergetool`, or VS Code's inline "Accept Current/Incoming/Both".

## Code Examples

```bash
# Create and work on a feature branch
git switch -c feature/login        # modern; = git checkout -b feature/login
# ...commit work...

# Integrate latest main into your feature before opening a PR
git switch feature/login
git rebase main                    # linear; replays your commits on top of main
# resolve conflicts if any, then:
#   git rebase --continue   (or --abort)

# Merge the feature into main (force a merge commit to keep the feature boundary)
git switch main
git merge --no-ff feature/login
git branch -d feature/login        # delete the merged branch

# Tidy local history before sharing
git rebase -i HEAD~3               # squash/reword/reorder the last 3 commits
```

```bash
# A conflict in action
git merge feature
# Auto-merging app.ts
# CONFLICT (content): Merge conflict in app.ts
git status                         # shows app.ts as "both modified"
# ...edit app.ts, remove <<<<<<< ======= >>>>>>> markers...
git add app.ts
git merge --continue
```

## Interview Q&A

**Q:** Merge vs rebase — when do you use each?

**A:** Merge preserves true history and is safe on shared branches; I use it to integrate a finished feature into `main` (often `--no-ff` to keep the feature visible). Rebase produces a clean linear history but rewrites commits, so I use it only on my **local, unpushed** work — e.g. `rebase main` to update my feature branch, or interactive rebase to squash WIP commits before a PR. The rule: never rebase commits others already have.

**Q:** What is a fast-forward merge?

**A:** When the branch I'm merging into hasn't diverged — the target's tip is an ancestor of the source — Git can just move the pointer forward to the source tip with no merge commit. If the branches diverged, a fast-forward isn't possible and Git makes a 3-way merge commit instead.

**Q:** Why is rebasing a shared/public branch dangerous?

**A:** Rebase creates new commits with new hashes and discards the originals. Anyone who already pulled the old commits now has history that conflicts with the rewritten version; their next pull diverges and they get messy conflicts or duplicate commits. So I only rebase history that lives solely in my local repo.

**Q:** Walk me through resolving a merge conflict.

**A:** `git status` to list conflicted files. Open each, find the `<<<<<<< / ======= / >>>>>>>` markers, edit the section to the intended final result, and delete the markers. `git add` each resolved file, then `git merge --continue` (or `--rebase --continue`). If it's going badly, `git merge --abort` resets to the pre-merge state. I lean on a merge tool or the editor's 3-way view for big conflicts.

**Q:** How do you squash several commits into one?

**A:** `git rebase -i HEAD~N`, then mark the commits to fold in as `squash` (keeps messages) or `fixup` (discards them). Common before opening a PR to turn noisy WIP commits into one clean, reviewable commit. On an already-pushed branch I'd then force-push *my own* branch with `--force-with-lease`.

## Gotchas & Anti-patterns

- **Rebasing shared history** — the cardinal sin; rewrites commits others depend on. If you must force-push a rebased *personal* branch, use `--force-with-lease` (refuses if someone else pushed meanwhile), never bare `--force`.
- **Leaving conflict markers in the file** — committing `<<<<<<<` markers compiles to broken code; always search for them before staging.
- **Deleting an unmerged branch with `-d`** — Git refuses (`-d` is safe); `-D` force-deletes and can lose commits. Don't reach for `-D` reflexively.
- **Assuming rebase has no conflicts** — it can conflict *per replayed commit*, so you may resolve the "same" area multiple times. `git rebase --abort` is the escape hatch.
- **Merge-commit noise** — fast-forwarding everything erases feature boundaries; never-fast-forwarding clutters history. Pick a team convention (`--no-ff` for features is common).
- **Detached HEAD commits during rebase confusion** — if you end up detached, `git reflog` finds your lost commit tips.

## Further Reading

- [Pro Git — Basic Branching and Merging](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging)
- [Pro Git — Rebasing](https://git-scm.com/book/en/v2/Git-Branching-Rebasing)
- [Atlassian — Merging vs Rebasing](https://www.atlassian.com/git/tutorials/merging-vs-rebasing)
