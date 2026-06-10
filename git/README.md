# Git

Version control with Git — from publishing your first repo to the conceptual and workflow questions a Senior Test Automation Engineer is expected to answer fluently. Coverage is tuned to interviews: the "why" and trade-offs, history-rewriting safety, conflict handling, and how Git workflows + branch protection turn an automated test suite into an enforced release gate.

## Topics (recommended reading order)

1. [Repo Setup & Publishing to GitHub](./01-repo-setup-and-github.md) — local folder → GitHub, end to end: install/config, `init`, `.gitignore`, first commit, remote, push, and HTTPS-token vs SSH auth.
2. [Git Fundamentals](./02-git-fundamentals.md) — the mental model: distributed VCS, working tree / staging / repo, commits as snapshots, branches, `HEAD`, refs.
3. [Branching & Merging](./03-branching-and-merging.md) — branches, fast-forward vs 3-way merge, **merge vs rebase**, interactive rebase/squash, resolving conflicts.
4. [Undoing Changes](./04-undoing-changes.md) — `restore`, `reset` (soft/mixed/hard) vs `revert`, `stash`, `cherry-pick`, and `reflog` as the safety net.
5. [Remotes & Collaboration](./05-remotes-and-collaboration.md) — remotes, `fetch` vs `pull`, tracking branches, pushing safely, Pull Requests, tags.
6. [Workflows & Branching Strategies](./06-workflows-and-strategies.md) — trunk-based vs GitHub Flow vs GitFlow, commit hygiene, branch protection & CI gating.
