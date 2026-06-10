# Remotes & Collaboration

> How your local repo talks to GitHub, why `fetch` and `pull` differ, tracking branches, and the pull-request review flow that real teams run on.

## TL;DR

- A **remote** is a named reference to another repo; `origin` is the conventional name for the one you cloned/pushed to.
- **`git fetch`** downloads remote commits/refs but **doesn't touch your branches** — safe, read-only-to-your-work.
- **`git pull`** = `fetch` + **integrate** (merge by default, or rebase with `--rebase`) into your current branch.
- A **tracking (upstream) branch** links your local branch to a remote one so bare `push`/`pull`/`status` know the target and show ahead/behind counts.
- **Pull Requests** are a GitHub (not Git) concept: propose a branch for review/CI before it merges into `main`.
- Keep a feature branch current with `git pull --rebase origin main` (or fetch + rebase) to stay ahead of drift.
- **`git push --force-with-lease`** is the safe force-push; bare `--force` can clobber teammates' commits.

## Deep Dive

### Remotes
`git remote -v` lists configured remotes and their URLs. `origin` is just a default name; you can have several (e.g. `origin` your fork, `upstream` the source repo in the fork workflow). Remote-tracking branches like `origin/main` are local read-only mirrors of the remote's state as of your last fetch — they're how Git knows you're "2 ahead, 1 behind."

### fetch vs pull (the key distinction)
- **`git fetch`** contacts the remote and updates your remote-tracking refs (`origin/*`) and downloads new objects. Your own branches and working tree are untouched. This lets you inspect what changed (`git log main..origin/main`) before integrating — the cautious choice.
- **`git pull`** does a `fetch` then immediately **integrates** `origin/<branch>` into your current branch. By default that's a **merge** (possibly creating a merge commit); `git pull --rebase` replays your local commits on top instead, keeping history linear.

So `pull` can surprise you with merge commits or conflicts; `fetch` never changes your work. Many teams set `pull.rebase = true` or use `--rebase` to avoid "Merge branch 'main' of origin..." noise.

### Pushing & tracking branches
`git push -u origin feature` publishes `feature` and sets the **upstream** so future `git push`/`git pull` need no arguments. `git push` sends your local commits to the remote; it's rejected (non-fast-forward) if the remote has commits you don't — you must integrate first (`pull`/`fetch`+`rebase`).

### Keeping a feature branch up to date
While your PR is open, `main` keeps moving. Periodically update:
```bash
git fetch origin
git rebase origin/main      # linear; or: git merge origin/main
```
Rebasing keeps your branch a clean continuation of `main` and shrinks the eventual diff/conflicts — but since rebase rewrites your branch's commits, you then `push --force-with-lease` to your *own* branch (fine; it's yours).

### Pull Requests & review flow
A PR (Merge Request in GitLab) is a platform feature that proposes merging your branch into a base branch. The typical flow:
1. Push your feature branch, open a PR against `main`.
2. CI runs (build, lint, **the test suite** — directly relevant to a TA role); reviewers comment.
3. Address feedback with more commits (or amend + force-with-lease).
4. Merge via **merge commit**, **squash-and-merge** (one tidy commit — most common), or **rebase-and-merge** (linear, no merge commit).
5. Delete the branch.

**Branch protection** rules (covered more in [[workflows-and-strategies]]) gate merges on green CI and approvals — the mechanism that makes automated tests a release gate.

### Tags
`git tag -a v1.2.0 -m "Release 1.2.0"` marks a specific commit (typically a release). Tags aren't pushed by default: `git push origin v1.2.0` or `git push --tags`. Lightweight tags are bare pointers; annotated tags (`-a`) store tagger/date/message and are preferred for releases.

## Code Examples

```bash
# Inspect remotes
git remote -v
git remote add upstream https://github.com/ORG/repo.git   # fork workflow

# See what's incoming WITHOUT changing your branch
git fetch origin
git log --oneline main..origin/main      # commits on remote you don't have yet

# Integrate
git pull                 # fetch + merge current branch's upstream
git pull --rebase        # fetch + rebase (linear history)

# Publish a new branch and set upstream
git switch -c feature/api-tests
git push -u origin feature/api-tests

# Keep the feature branch fresh against main
git fetch origin
git rebase origin/main
git push --force-with-lease      # safe force-push of YOUR branch after rebase

# Tags / releases
git tag -a v1.2.0 -m "Release 1.2.0"
git push origin v1.2.0
```

## Interview Q&A

**Q:** What's the difference between `git fetch` and `git pull`?

**A:** `fetch` downloads new commits and updates remote-tracking branches (`origin/main`) but leaves my local branches and working tree untouched — so I can review what changed before integrating. `pull` is `fetch` plus an immediate integration into my current branch — a merge by default, or a rebase with `--rebase`. I often `fetch` first when I want to see incoming changes before they touch my work.

**Q:** What is a tracking/upstream branch?

**A:** A link between a local branch and a remote branch (e.g. local `main` ↔ `origin/main`), set with `push -u` or `branch --set-upstream-to`. It lets bare `push`/`pull` know the target and lets `git status` report how many commits I'm ahead/behind.

**Q:** Your push is rejected as non-fast-forward. What happened and what do you do?

**A:** The remote branch has commits I don't have locally, so my push isn't a clean fast-forward. I `git fetch` and integrate — usually `git rebase origin/<branch>` (or `pull --rebase`) to replay my commits on top — resolve any conflicts, then push. I never `--force` over it unless it's my own branch and I know I'm intentionally rewriting it, in which case `--force-with-lease`.

**Q:** Is a Pull Request a Git feature?

**A:** No — it's a feature of hosting platforms like GitHub/GitLab built on top of Git branches. Git itself just has branches, pushes, and merges. The PR adds the review, discussion, CI checks, and merge-button workflow around proposing one branch be merged into another.

**Q:** How do you keep a long-running feature branch from drifting too far from `main`?

**A:** Regularly `git fetch origin` and `git rebase origin/main` (or merge if the branch is shared). Rebasing keeps the branch a clean continuation of latest `main` and minimizes the final conflict surface; afterward I `push --force-with-lease` since rebasing rewrote my branch's commits.

**Q:** Squash-and-merge vs merge commit vs rebase-and-merge for a PR?

**A:** Squash-and-merge collapses the branch into one commit on `main` — cleanest history, loses intermediate commits (great for noisy feature branches). A merge commit preserves all commits and records the merge point. Rebase-and-merge replays the commits linearly with no merge commit. I pick based on the team's history conventions; squash is the most common default.

## Gotchas & Anti-patterns

- **`git push --force` on a shared branch** — overwrites others' commits. Use `--force-with-lease`, which refuses if the remote moved since your last fetch, and only on branches you own.
- **Blind `git pull` creating merge-commit noise** — on a branch you've committed to, a plain pull can spawn "Merge branch 'main'..." commits; prefer `pull --rebase` or fetch-then-review.
- **Forgetting tags aren't pushed** — `git push` doesn't send tags; release tags silently stay local until `git push --tags`/`push origin <tag>`.
- **Confusing `origin/main` with `main`** — `origin/main` is a snapshot from your last fetch, not live; a stale fetch makes you think you're up to date.
- **Working directly on `main` and pushing** — bypasses review/CI/branch protection; always branch + PR in a team setting.
- **Resolving the "non-fast-forward" rejection by force-pushing** — destroys the remote commits you were missing; integrate instead.

## Further Reading

- [Pro Git — Working with Remotes](https://git-scm.com/book/en/v2/Git-Basics-Working-with-Remotes)
- [Pro Git — Tagging](https://git-scm.com/book/en/v2/Git-Basics-Tagging)
- [GitHub Docs — About pull requests](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/about-pull-requests)
- [git-push: `--force-with-lease`](https://git-scm.com/docs/git-push#Documentation/git-push.txt---force-with-lease)
