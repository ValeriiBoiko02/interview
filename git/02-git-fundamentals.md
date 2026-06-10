# Git Fundamentals

> The mental model every senior engineer should be able to draw on a whiteboard: what Git actually stores, where changes live before they're permanent, and what `HEAD`, branches, and commits really are.

## TL;DR

- Git is a **distributed** version control system: every clone is a full repository with complete history, not a thin checkout.
- Three states a file moves through: **working tree** (your edits) → **staging area / index** (`git add`) → **repository** (`git commit`, permanent snapshot). The **stash** is a fourth, off-to-the-side shelf.
- A **commit** is an immutable snapshot of the whole tree + metadata (author, message, parent commit). It's identified by a SHA-1/SHA-256 hash.
- A **branch** is just a movable pointer (a 41-byte file) to a commit. **`HEAD`** points to your current branch (or directly to a commit when "detached").
- History is a **DAG** of commits linked by parent pointers; a merge commit has two parents.
- `git status`, `git diff`, `git log` are your three "where am I?" commands — use them constantly.

## Deep Dive

### Distributed, not centralized
Unlike SVN, there is no single authoritative server in Git's model. `git clone` copies the entire history locally, so commits, branching, diffs, and log are all **local and offline**. "The remote" (GitHub) is a convention for collaboration, not a technical requirement. This is why most Git operations are instant — no network round-trip.

### The three areas (+ stash)
1. **Working tree** — the actual files on disk you edit.
2. **Staging area (index)** — a draft of your next commit. `git add` copies a file's current state into the index. This lets you craft a commit from *part* of your changes (e.g. stage one file, leave another).
3. **Repository (`.git/`)** — `git commit` takes whatever is in the index and writes a permanent snapshot.

The **stash** is a separate shelf for temporarily parking uncommitted work (see [[undoing-changes]]).

Understanding the index is what separates a confident Git user from someone who only memorized `add`/`commit`. `git status` literally describes the gap between these areas: "Changes to be committed" (index vs last commit) and "Changes not staged" (working tree vs index).

### What a commit actually is
A commit object stores: a pointer to a **tree** (the directory snapshot), the **parent** commit hash(es), author/committer + timestamps, and the message. The hash is computed from all of that, so a commit is **immutable** — "editing" a commit (amend, rebase) actually creates a *new* commit with a new hash. This immutability is the root of why rewriting shared history is dangerous (see [[branching-and-merging]]).

Commits are **snapshots, not diffs**. Git stores the full tree each commit (deduplicated by content hash), and computes diffs on demand for display. This is a common interview gotcha — people assume Git stores deltas.

### Branches, HEAD, and refs
- A **branch** is a lightweight, movable pointer to a commit, stored as a file under `.git/refs/heads/`. Creating a branch is O(1) — it just writes a hash to a file. This cheapness is why branch-heavy workflows are idiomatic in Git.
- **`HEAD`** is a pointer to "where you are" — normally it points at a branch (e.g. `refs/heads/main`), and that branch points at a commit. When you commit, the current branch pointer advances; `HEAD` follows.
- **Detached HEAD**: if you check out a specific commit/tag instead of a branch, `HEAD` points directly at a commit. New commits there aren't on any branch and can be lost — Git warns you.

### The everyday loop
`edit → git add → git commit`, with `git status`/`git diff`/`git log` to orient. `git diff` shows working-tree-vs-index; `git diff --staged` shows index-vs-last-commit. `git log --oneline --graph --decorate` visualizes the DAG and where branch pointers sit.

## Code Examples

```bash
# Orientation trio
git status                 # what's staged / unstaged / untracked
git diff                   # working tree vs index (unstaged changes)
git diff --staged          # index vs HEAD (what the next commit will contain)
git log --oneline --graph --decorate --all   # the commit DAG + branch pointers

# Staging precisely (craft a focused commit)
git add src/login.ts       # stage just one file
git add -p                 # interactively stage selected hunks
git commit -m "Add login flow"

# Inspect a commit object
git cat-file -p HEAD       # shows tree, parent, author, message of current commit
```

```bash
# HEAD and branches are just pointers — prove it:
cat .git/HEAD              # -> "ref: refs/heads/main"
cat .git/refs/heads/main  # -> the 40-char commit SHA that 'main' points to
```

## Interview Q&A

**Q:** What's the difference between the working directory, the staging area, and the repository?

**A:** The working directory is your files on disk. The staging area (index) is a buffer holding the exact content of the *next* commit — `git add` puts changes there. The repository is the committed history; `git commit` writes the staged snapshot permanently. The staging area is what lets me build a clean, focused commit out of a messy working directory.

**Q:** Does Git store diffs or snapshots?

**A:** Snapshots. Each commit references a full tree of the project at that point, with content deduplicated by hash so unchanged files aren't re-stored. Diffs are computed on demand for display. (Internally, packfiles do delta-compress for storage efficiency, but the conceptual model is snapshots.)

**Q:** What is `HEAD`?

**A:** A pointer to your current location in history — usually a symbolic ref to the branch you're on (which in turn points to a commit). When I commit, the branch advances and `HEAD` moves with it. If I check out a raw commit instead of a branch, I get a "detached HEAD" where new commits aren't attached to any branch.

**Q:** Why is creating a branch in Git so cheap compared to older VCS?

**A:** Because a branch is just a 41-byte file containing a commit hash — no file copying involved. Switching branches updates `HEAD` and reconciles the working tree. That cheapness is why short-lived feature branches are the norm.

**Q:** What does it mean that commits are immutable?

**A:** A commit's hash is derived from its content (tree + parents + metadata), so you can never change a commit in place — operations like `--amend` or `rebase` create new commits with new hashes and leave the originals orphaned. This is why rewriting history that others have already pulled is disruptive.

## Gotchas & Anti-patterns

- **Thinking `git add` saves your work** — it only stages; an unrecovered crash before `commit` loses staged-but-uncommitted content conceptually (though it's in the index). Commit early.
- **Confusing `git diff` and `git diff --staged`** — plain `diff` hides already-staged changes, surprising people who think it shows "all my changes."
- **Detached HEAD surprise** — checking out a tag/commit then committing creates orphan commits; if you switch away without branching, `reflog` is your only recovery.
- **Assuming Git needs the server** — branching, committing, diffing, and history browsing are all local; treating Git like SVN (constant server contact) misses the point.
- **Giant catch-all commits** — not using the index to make atomic, reviewable commits makes history (and `git bisect`/`revert`) far less useful.

## Further Reading

- [Pro Git — Git Basics & The Three States](https://git-scm.com/book/en/v2/Getting-Started-What-is-Git%3F)
- [Pro Git — Git Internals: Git Objects](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)
- [Pro Git — Git Branching in a Nutshell](https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell)
