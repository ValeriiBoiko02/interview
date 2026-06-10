# Workflows & Branching Strategies

> How teams organize branches and ship code: trunk-based vs GitHub Flow vs GitFlow, commit hygiene, and the branch-protection + CI gating that makes your automated test suite a real release gate.

## TL;DR

- **Trunk-based development**: everyone integrates into `main` constantly via short-lived branches; feature flags hide unfinished work. Best for CI/CD and fast delivery.
- **GitHub Flow**: one long-lived `main` + short feature branches → PR → merge → deploy. Simple, the de-facto default for web apps.
- **GitFlow**: long-lived `main` + `develop`, plus `feature/`, `release/`, `hotfix/` branches. Heavyweight; fits versioned/released software, often overkill today.
- **Short-lived branches** beat long-lived ones — they reduce merge pain and keep `main` releasable.
- **Conventional Commits** (`feat:`, `fix:`, `test:` …) give readable history and enable automated changelogs/versioning.
- **Branch protection** on `main` (require PR, green CI, approvals) is what turns "tests pass" into "you can't merge unless tests pass" — central to a test-automation role.

## Deep Dive

### Trunk-based development
Developers commit small changes to `main` (the "trunk") at least daily, using very short-lived branches (hours to a day) merged behind green CI. Incomplete features hide behind **feature flags** rather than living on a long branch. Pros: minimal merge conflicts, always-releasable trunk, ideal for continuous deployment. Cons: demands strong CI, test discipline, and flag hygiene. This is the model most high-throughput teams and the DevOps research (DORA) favor.

### GitHub Flow
A pragmatic middle ground: `main` is always deployable; you branch off for a change, push, open a PR, get review + CI, merge, deploy. No `develop` branch, no release branches. Simple and effective for products that deploy continuously (most web/SaaS). It's effectively trunk-based with PRs as the gate.

### GitFlow
A structured model with two permanent branches — `main` (released code) and `develop` (integration) — plus supporting branches: `feature/*` off develop, `release/*` to stabilize a version, `hotfix/*` off main for emergency fixes. Powerful for software with **explicit versioned releases** (desktop apps, libraries with multiple supported versions). For continuously deployed web apps it's usually too heavy and slows integration. Knowing *when it's overkill* is the senior signal.

### Choosing
| Strategy | Branches | Best for | Cost |
|---|---|---|---|
| Trunk-based | `main` + tiny short-lived | CI/CD, fast teams | Needs flags + strong tests |
| GitHub Flow | `main` + feature | Most web/SaaS apps | Low |
| GitFlow | `main`+`develop`+release/hotfix/feature | Versioned/released products | High ceremony |

Across all of them, the constant advice: **keep branches short-lived and small** to minimize divergence and conflicts.

### Commit hygiene & Conventional Commits
Good history is a feature: atomic commits (one logical change), imperative present-tense messages ("Add retry to flaky login test", not "added stuff"). **Conventional Commits** prefix the type — `feat:`, `fix:`, `test:`, `refactor:`, `chore:`, `docs:` — optionally with a scope: `test(login): stabilize auto-wait`. This enables tooling (semantic-release, automated changelogs, semver bumps) and makes `git log` scannable. Squash noisy WIP commits before merge (see [[branching-and-merging]]).

### Branch protection & CI gating (the TA angle)
On the hosting platform you protect `main`: require a PR (no direct pushes), require **status checks to pass** (your Playwright suite, lint, typecheck), require ≥1 approval, and optionally require the branch be up to date with `main` before merge. This is the mechanism that makes automated tests an enforced **release gate** — a failing test literally blocks the merge button. As a senior test-automation engineer you're often the one defining which checks are required, keeping them fast and non-flaky (flaky required checks erode trust and get bypassed), and deciding what runs on PR vs nightly.

## Code Examples

```bash
# Short-lived feature branch, GitHub Flow style
git switch -c feat/api-contract-tests
# ...small, focused commits...
git push -u origin feat/api-contract-tests
# open PR -> CI (Playwright suite) runs -> review -> squash-and-merge -> delete branch
```

```text
# Conventional Commit messages
feat(checkout): add guest checkout flow
fix(login): handle expired-token redirect
test(cart): cover quantity-update edge cases
chore(ci): shard Playwright run across 4 workers
```

```yaml
# Illustrative: GitHub Actions check that becomes a REQUIRED status check on main
name: e2e
on: pull_request
jobs:
  playwright:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npx playwright test            # failing tests block the merge
```

## Interview Q&A

**Q:** Compare trunk-based development and GitFlow. When would you pick each?

**A:** Trunk-based keeps one `main` with tiny short-lived branches merged continuously behind green CI, using feature flags to hide unfinished work — ideal for continuous delivery and minimizing merge conflicts, but it demands strong tests and flag discipline. GitFlow adds `develop` plus release/hotfix/feature branches and suits software with explicit versioned releases needing parallel maintenance. For a continuously deployed web app I'd choose trunk-based or GitHub Flow; GitFlow's ceremony is usually overkill there.

**Q:** Why prefer short-lived branches?

**A:** The longer a branch lives, the more `main` drifts from it, so conflicts and integration risk grow non-linearly. Short branches merge cleanly, keep `main` releasable, get reviewed while context is fresh, and surface integration problems early. Long-lived branches are where painful "big bang" merges come from.

**Q:** How do automated tests actually gate releases in a Git workflow?

**A:** Through branch protection on `main`: require a PR, require status checks (the test suite, lint, typecheck) to pass, and require approvals. The CI runs the suite on every PR; a failing required check disables the merge button. So the tests aren't advisory — they're an enforced gate. My job is also to keep those required checks fast and non-flaky, since a flaky required check trains people to bypass or rerun blindly.

**Q:** What are Conventional Commits and why use them?

**A:** A commit-message convention prefixing the type — `feat:`, `fix:`, `test:`, etc., optionally with a scope. They make history readable and machine-parseable, enabling automated changelogs and semantic-version bumps (e.g. `feat` → minor, `fix` → patch, `BREAKING CHANGE` → major). They also nudge people toward atomic, single-purpose commits.

**Q:** A required CI check is flaky and frequently red on unrelated PRs. What do you do?

**A:** Treat it as a priority bug, not background noise. Quarantine the flaky test (mark it non-blocking / move it off the required set) so it stops blocking unrelated work, investigate the root cause (timing, test isolation, shared state, real auto-wait misuse), fix and re-promote it. Leaving a flaky check required erodes trust in the whole gate and trains people to merge red.

## Gotchas & Anti-patterns

- **Long-lived feature branches** — the top source of merge hell; integrate small and often instead.
- **Adopting GitFlow by default** — its release/develop overhead slows continuously-deployed products; match the model to the delivery cadence.
- **Committing directly to `main`** — bypasses review/CI; protect the branch so it can't happen.
- **Required checks that are slow or flaky** — people start ignoring or rerunning them; keep the required set fast, deterministic, and meaningful (push slow/flaky suites to nightly).
- **Vague commit history** — "fix", "update", "wip" commits make `git log`, `bisect`, and `revert` useless; enforce atomic, conventional commits and squash WIP before merge.
- **Feature flags that never get removed** — trunk-based relies on flags, but stale flags become tech debt and test-matrix explosion; track and retire them.

## Further Reading

- [Trunk Based Development](https://trunkbaseddevelopment.com/)
- [GitHub Flow](https://docs.github.com/en/get-started/using-github/github-flow)
- [Atlassian — Gitflow Workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [GitHub Docs — About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
